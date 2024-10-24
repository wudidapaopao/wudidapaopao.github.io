---
title: Serializable Snapshot Isolation in PostgreSQL
layout: post
categories: [Paper]
image: /assets/img/postgresql-logo.jpeg
customexcerpt: "PostgreSQL SSI"
---

# 1. Snaphot Isolation Versus Serializable

SI的异常行为：

+ Write Skew。
+ Batch Processing。  
  如果没有只读事务T1，是没有异常行为的。

<img title="" src="https://wudidapaopao.github.io/assets/img/2024-10-22-serializable-snapshot-isolation-in-postgresql/1.jpeg" alt="" data-align="center" width="660">

为什么需要Serializable：

+ 方便应用开发者避免异常行为，这些异常行为难以发现，有时是静默错误。
+ 提前分析所有执行的事务，复杂度太大，且无法处理ad hoc查询。



# 2. Serializable Snaphot Isolation

SSI是基于SI实现的serializable，性能比strict two phase locking好。

SI的serialization graph：没有cycle才是serializable。

<img title="" src="https://wudidapaopao.github.io/assets/img/2024-10-22-serializable-snapshot-isolation-in-postgresql/2.jpeg" alt="" data-align="center" width="660">

**SI的anomalies对应的serialization history graph一定包含两条相邻的rw edge，且T3在这个cycle中是第一个commit的。**

<img title="" src="https://wudidapaopao.github.io/assets/img/2024-10-22-serializable-snapshot-isolation-in-postgresql/3.jpeg" alt="" data-align="center" width="660">

## SSI

基于上述理论，当在serialization history graph找到两条同一方向的相邻的rw边，则认为出现了危险结构，需要abort其中一个事务，虽然有误判的可能。

相比于S2PL或者OCC，SSI是更快的，因为它的限制更少，允许一定程度的并发读写。

**变种SSI**：

对于下图的危险结构，当T1或者T2的commit time小于T3时，则认为没有必要abort事务，可以降低误判的概率。

<img title="" src="https://wudidapaopao.github.io/assets/img/2024-10-22-serializable-snapshot-isolation-in-postgresql/4.jpeg" alt="" data-align="center" width="300">

PSSI(Precisely SSI)，是除了rw，也追踪ww，wr依赖以确定是否有cycle的完全准确的判定，但是需要额外的付出更多内存开销，所以不被采用。



# 3. Read only Optimization

我们可以知道， 当T1为只读事务时，T3的提交版本必须小于T1的读版本，否则T3和T1之间无法形成依赖，从而无法成环(注意，如果是危险结构，T3的提交版本一定是最小的)。

<img title="" src="https://wudidapaopao.github.io/assets/img/2024-10-22-serializable-snapshot-isolation-in-postgresql/4.jpeg" alt="" data-align="center" width="300">

## Deferrable Transaction

基于上述推论，当一个只读事务开始时，可以延迟等待获取一个安全的读版本(小于该读版本的事务都已经提交)，那么该读事务可以安全地执行，不需要去进行SSI相关的依赖检查。



# 4. Implementing SSI In PostgresSQL

## PG Background

SI和RC的实现区别是，RC在每次query前都获取一次读版本。

实现SSI之后，SI作为RR，SSI作为serializable。

PG的MVCC是在数据页内标记删除旧版本，写入新版本，而Oracle/MySQL是原地更新，旧版本写入rollback log中。

PG中的锁：

+ lightweight locks：Latch。
+ heavyweight locks：表锁，控制DML和DDL的并发，存放在内存的lock stable中。
+ Tuple Locks：为了节省内存，没有冲突的情况下，在tuple header做标记，记录持有该lock的事务ID。当发生写写冲突时，复用heavyweight lock manager及其死锁检测。

## Detecing Conflicts

在`PG`故有的`Lock Manager`之外，单独实现一个`SSI Lock Manager`。事务读操作需要在`SSI Lock Manager`中添加`Tuple`级或者`Predicate`级的`SIREAD Lock`，`SIREAD Lock`不会造成`blocking`，主要用于`RW`依赖判定。`Non-Blocking`也意味着不需要进行死锁检测。

`SIREAD Lock`不会阻塞`DDL`，但`DDL`会导致`Tuple`或者`Index-Gap`类型的`Lock`失效，此时`SIREAD Lock`需要升级至表级。

当`RW`依赖中的两个并发事务，如果写比读先发生，那么快照读的机制会发现正在读的记录有未提交的写，则探测到了`RW`依赖。如果读比写先发生，读的事务需要获取`SIREAD Lock`，写的事务需要检查`SSI Lock Manager`探测是否有`RW`依赖。`SIREAD Lock`需要一直持有，直到和自身事务并发的事务都已经提交，才可以释放。

## Tracking Conflicts

每个事务维护两个列表，一个`RW`依赖的`In`列表，一个`RW`依赖的`Out`列表。

## Resolving Conflicts：Safe Retry

<img title="" src="https://wudidapaopao.github.io/assets/img/2024-10-22-serializable-snapshot-isolation-in-postgresql/4.jpeg" alt="" data-align="center" width="300">

当出现如上危险结构时，需要选择一个事务abort，选择的策略：

+ 等到T3提交再去选择事务abort，因为只有当T3最先提交，才可能出现依赖cycle。
+ 当T3提交后，如果T2还没提交，优先选择T2进行abort，因为当T2重试时，T2和已经提交的T3一定不是并发，则该危险结构不会再出现。
+ 如果T2和T3都已经提交，那么选择T1进行重试也是安全的。

所以当危险结构出现时，不会立刻重试事务。而是当事务提交时，去检查是否存在危险结构。



# 5. Memory Usage Mitigation

SSI降低内存开销的措施：

1. 只读事务获取一个足够大的安全快照，后续操作不需要做检查。
2. 细粒度的锁提升为粗粒度的锁，比如DDL发生时，tuple级的`SIREAD Lock`提升为表级的`SIREAD Lock`。
3. 清理对象：  
   清理`SIREAD Lock`：当一个事务提交后，如果与之并发的事务都已经提交，那么这个事务获取的`SIREAD Lock`都可以清理。  
   清理`Transaction Node`：当一个事务提交后，当与之并发的事务都已经提交，该事务的事务节点如果直接清理，可能存在一种情况，T1->T2->T3，当T2和T3都已经提交，如果T1之后发生了与T2的RW冲突，则T1需要知道T2与T3发生了RW冲突，且还需要知道T3的提交版本，为了处理这种情况，T2需要记录与之RW冲突的事务中的最小提交版本。  
   当活跃事务只剩下只读事务时，所有的`SIREAD Lock`都可以清理，因为之后不会再有并发事务发生RW冲突，需要用到`SIREAD Lock`的情况。`SIREAD Lock`是用来发生写操作时，查看并发事务是否有对应`SIREAD Lock`的情况。此外，当活跃事务只剩下只读事务，对于`T1->T2`且T2已经提交的情况，对于`T2`来说，T1的入边可以删掉，因为已经不可能出现并发事务T3，并且T3写了T2读过的记录。
4. Summarizing Committed Transactions：为了避免维护过多已经提交事务的`dependency graph`和`SIREAD Lock`，对于可以替代的情况，会将多个已提交事务用一个代替。



# 6. Feature Interactions

## Two-Phase Commit

<img title="" src="https://wudidapaopao.github.io/assets/img/2024-10-22-serializable-snapshot-isolation-in-postgresql/5.jpeg" alt="" data-align="center" width="300">

+ Prepare时，SIREAD Lock也要持久化，以便在崩溃回复时重建。
+ `dependency graph`过大不方便持久化，且prepare之后也会变，所以保守地认为崩溃回复后认为prepared的事务有in的RW冲突，也有out的RW冲突。
+ 一般来说，为了避免重复冲突，会abort中间的事务，但prepared的事务不能abort，所以只能抛弃左边的事务。

## Streaming Replication

Slave上的只读查询会影响到SSI的正确性，但是如果每次只读查询都把read set同步发到master上，开销不可接受。

PG目前支持在slave上读，但不支持SSI的读。未来可以：

+ 在slave上直接根据回放的日志本地选择一个stale的safe snapshot version进行读。
+ 等待下一个safe snapshot version进行读。

## Savepoints and Subtransactions

当一个事务先获取`SIREAD Lock`，之后对同一个tuple修改时，会删除`SIREAD Lock`，直接加`Write Lock`，因为加了`Write Lock`已经保证了之后不会有`RW`冲突。

但是在PG使用subtransaction作为savepoint的场景，回滚subtransaction会释放`Write Lock`，从而可能释放隐藏的`SIREAD Lock`，对于subtransaction的回滚，我们不希望`SIREAD Lock`也被释放，因为此时用户可能在subtransaction读到了数据，需要`SIREAD Lock`保证SSI。所以在子事务中，不能使用`Write Lock`替代`SIREAD Lock`的方式。

## Index Types

目前PG只有B+Tree支持`Predicate Lock`，其他的index想要获取`Predicate Lock`只能退化到表锁。



# 7. Conclusion

这是第一个在之前不支持serializable的数据库中实现SSI的产品：

+ 引入了新的in-memory的提供SIREAD Lock的SSI Lock Manager，追踪RW依赖。
+ 尽可能让内存使用控制在合理范围，比如summarize committed transaction。
+ 针对只读事务做优化，比如获取安全的读快照。



# 参考资料

- https://drkp.net/papers/ssi-vldb12.pdf

