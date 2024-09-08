---
title: ClickHouse Lightning Fast Analytics for Everyone
layout: post
categories: [ClickHouse]
image: /assets/img/clickhouse-logo.jpg
customexcerpt: "ClickHouse VLDB"
---

`CK`的第一篇论文，介绍了开源代码部分的方方面面，闭源部分比如`light-weight update`和`shared-merge-tree`没有涉及。

单独提一点，`CK`编译之后的可执行文件完全静态链接，没有第三方依赖，对于`C++`来说还是挺方便的。
<img title="" src="https://wudidapaopao.github.io/assets/img/2024-09-06-ck-lightning-fast-analytics-for-everyone/1.jpeg" alt="" data-align="center" width="660">

# 1. Storage Layer

## 1.1 On-Disk Format

`CK`主要的持久化存储结构是`merge tree`，包含若干`data part`，每次写入都会生成一个`data part`。后台会定期做合并，当`data part`大于`150GB`，不会再参与合并。和`lsm-tree`不一样的是合并不需要严格按照写入顺序。

- 同步写入：立刻生成一个`data part`。

- 异步写入：缓存写入的数据，当数据量或者时间超过阈值，再异步生成`data part`。

每个`data part`对应一个目录，目录内包含`schema`信息，每个`column`对应一个文件，当`data part`小于`10MB`时，所有`column`放在同一个文件。

`Data part`内部再按照`8192`行划分成一个个`granule`，按照`1M`大小聚集多个`granule`位一个`block`，`block`用`lz4`压缩。为了随机定位，会单独存储每个`granule`所在`compressed block`在文件的偏移以及`granule`在`uncompressed block`内的偏移。

每个`table`可以按照`range`，`hash`，`round-robin`以任意的分区表达式进行分区，每个分区会存储分区表达式的最小值和最大值，便于分区裁剪。用户还可以创建`column statistics`用于基数预测。

<br>

## 1.2 Data Pruning

- `sparse primary key index`：所有行按照`PK`列排序，每个`graunle`的第一个`PK`值记录在索引文件，查询时加载整个文件到内存，然后二分查找定位到`graunle`。

- `projection`：按照指定的不同于`pk`的顺序对行进行排列，对于已经存在的`data part`会延迟添加`projection`。

- `skipping index`：为相邻的若干`graunle`添加元数据，比如`min-max indices`，`set indices`，`bloom filter indices(build for row / token / n-gram values)`。

<br>

## 1.3 Merge-time Data Transformation

利用后台`merge`过程处理一些额外工作，`merge tree`衍生出了一些变种：

- `replacing merges`：每行数据添加一个`timestamp`，相同`PK`只保留最新的几个版本的行。

- `aggregating merges`：相同`PK`的行在`merge`时会聚合成一个新的聚合行，一般用于物化视图。

- `TTL (time-to-live) merges`：`merge`时删除过期的行，一次处理一个`data part`。

<br>

## 1.4 Updates and Delets

- `update`或`delete`不会阻塞并行写入。

- 因为`mutation`需要重写整个`data part`，避免临时文件过大，整个过程非原子，一个个`data part`依次进行`mutation`。

- `lightweight deletes`只更新内部的`bitmap column`，更加轻量。

- 同一个table的`update`或`delete`认为是少见的，所以串行操作。

<br>

## 1.5 Idempotent Inserts

为了避免客户端重复写入值，`CK`给最近写入的`data part`计算并存储`hash`值，存放在本地或者`keeper`上。

<br>

## 1.6 Data Replication

`CK`的多副本间复制，增量日志存储在`keeper`上，决定全局的顺序。需要同步的操作包括`insert`，`mutation`，`merge`，`DDL`。

- `insert`的同步时直接从写入节点下载拉取`data part`。

- `merge`的同步可以本地`merge`，也可以从远端下载，比如两个副本位于不同的数据中心，可以选择本地`merge`。

- 支持多`leader`更新`replication log`。

- 默认异步复制，也支持同步复制模式，即`quorum`节点接受新状态才算成功。

- 日志回放可以并行，比如同一个表的写入或者不同表的操作。

<img title="" src="https://wudidapaopao.github.io/assets/img/2024-09-06-ck-lightning-fast-analytics-for-everyone/2.jpeg" alt="" data-align="center" width="660">
<br>

<br>

## 1.7 ACID Compliance

- 基于`data part`的`version`进行`MVCC`读。

- 不强制`fsync`，容忍小概率丢数据风险。

<br>

## 2. Query Processing Layer

## 2.1 SIMD Parallelization

- 编译成多平台的指令(`SSE 4.2 / AVX2 / AVX-512`)，运行时根据`cpuid`选择。

- 手写指令或者编译器`auto-vectorization`。

<br>

## 2.2 Multi-Core Parallelization

- `SQL`转化成一个`physical plan operators`的`DAG`。每个`physical plan operators`按照配置的并发度生成若干个`execution lanes`。

- 工作线程持续遍历`DAG`中的`physical plan operators`，执行状态转换(`need-chunk` / `ready` / `done`)。

- 执行过程中根据系统的实际负载，可以调节工作线程的数量以改变并发度。

- 执行过程中还可以改变`operators`，比如内存超限后切换到`external aggregation / sort / join`。当等待远端数据时，将`operators`加入到异步队列。

- 当执行`filter`后，不同`lane`的数据可能不均衡，可以添加`repartition operator`进行数据均衡。

- 有些`operator`会阻塞并发，比如下图中的`GroupStateMerge`。

<br>

## 2.3 Multi-Node Parallelization

多个`shard`并行执行，最后将结果发给`initiator node`进行`merge`。

<br>

<img title="" src="https://wudidapaopao.github.io/assets/img/2024-09-06-ck-lightning-fast-analytics-for-everyone/3.jpeg" alt="" data-align="center" width="660">
<br>

<br>

## 2.4 Holistic Performance Optimization

### 2.4.1 Query Optimization

- 优化`AST`：比如`concat(lower(’a’),upper(’b’))`转化为` ’aB’`，`sum(a*2)` 转化为`2 * sum(a)`，`x=c OR x=d` 转化为  `x IN (c,d)`。之后`AST`转化为`logical operator plan`。

- 优化`logical operator plan`：比如`filter pushdown`，`reordering function
  evaluation and sorting steps`。之后`logical operator plan`转化为`physical operator plan`。

- 优化`logical physical plan`：利用`merge tree`的特性，比如`PK`的有序性，可以加速`order by`或者`group by`的速度。`Group by`可以使用`sort aggregation`，降低内存开销并且`aggregate value` 可以被立刻传递给下一个`operator`。

<br>

### 2.4.2 Query Compilation

当查询被多次执行，使用`LLVM`将多个`plan operators`转化位`1`个。：

- `a * b + c + 1` 从`3`个`operator`转化为`1`个。对`group by`的多个聚合函数只执行一次。

- 好处是可以降低虚函数调用次数，降低分支预测失败，保持数据在`cpu cache`，利用编译器的能力用上最新硬件指令。

<br>

### 2.4.3 Primary Key Index Evaluation

利用`merge tree`的`PK`有序，`toYear(k) =  2024 可以替换为 k >= 2024-01-01 && k < 2025-01-01`，将全表扫描转为部分扫描。

<br>

### 2.4.4 Data Skipping

利用`skipping index`跳过数据。

<br>

### 2.4.5 Hash Tables

哈希表是各种`operator`最常用的数据结构，所以`CK`测试了市面上的各种`hash table`的优化，对不同场景使用不同的`hash table`。

<br>

### 2.4.6 Joins

本人暂时不熟悉也不关注`join`，略。

<br>

## 2.5 Workload Isolation

- `CPU`：根据当前可用的`CPU cores`，查询运行时可以动态调整工作线程数量。

- `memory`：`server / user / query`级别的内存使用情况会被追踪。当空闲内存较多时，查询也可以使用超出限制的内存。内存不够时也会使用`external sort / group / join`。 

- `I/O`：允许用户限制本地和远程的`disk`访问。

<br>

## 3. Intergration Layer

`ETL`是将远程数据`push`到目标系统，`CK`使用`pull`以使用远程数据。

- `external connectivity`：`CK`内置了`MySQL`，`Kafka`，`Redis`，`S3`等各种外部数据的访问。
  
  - `temporary access with integration table functions`：临时地对外部数据进行查询或写入。
  
  - `persisted access`：外部数据拉取到`CK`进行读写。

- `data formats`：`CK`支持`CSV`，`JSON`，`Parquet`，`PB`等各种格式的读写。`Parquet` 也被在查询执行过程中利用，`filter`可以直接在压缩数据上执行。

- `compatibility interfaces`：客户端和`CK`可以通过`MySQL`，`PG`，`HTTP`等各种协议交互。

<br>

## 4. Built-in Performance Analysis Tools

用户可以通过系统表和`CK`进行交互，以获取查询或者后台操作的性能瓶颈。

- `server and query metrics`：活跃的`data part`数量，网络吞吐，缓存命中率，查询读取的`block`数量，索引使用情况等。

- `sampling profiler`：`server threads`的调用栈可以被收集，然后导出到外部工具。

- `OpenTelemetry integration`：集成了`OpenTelemetry`。

- `explain query`：查看查询的`AST`， ` logical and physical operator plans`，`execution-time behavior`。

<br>

# 5. Related Work

和`DuckDB`的区别：

- `DuckDB`使用轻量级压缩，比如`order-preserving dictionaries `或者`frame-of-reference`，而`CK`使用`LZ4`。

- DuckDB使用`serializable transactions based on Hyper’s MVCC scheme`，对于事务支持更完善，而`CK`只提供基本的`snapshot isolation`。

上面两点可以看出`DuckDB`对于混合负载，即有一定`insert / update`的场景更友好，而`CK`还是比较适合批量写入，更新较少的场景。

<br>

# 6. Conclusion and Outlook

未来的计划：

- 对`transaction`更好的支持。

- 支持`PromQL`作为查询语言。

- 支持`JSON`数据类型。

- 为`join`生成更好的`plan`。

- 实现`light-weight updates`。

<br>

# 参考资料

- https://www.vldb.org/pvldb/vol17/p3731-schulze.pdf


