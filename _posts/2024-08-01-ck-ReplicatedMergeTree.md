---
title: ClickHouse ReplicatedMergeTree  
layout: post
categories: [ClickHouse]
image: /assets/img/airplane.jpg
customexcerpt: "ClickHouse副本同步机制"
---

# 1. MergeTree

`CK`最基本的单机存储引擎是`MergeTree`，每个`MergeTree`包含若干`partition`，每个`partition`的每一次写入会生成一个或多个`data part`。

每个`data part`包括`compact`和`wide`两种方式的列存存储形式。当数据量不大时会选择`compact`形式将所有列存放在一个文件中，当数据量大时则会选择`wide`形式，每个列单独存放。

`MergeTree`还有`skipping index`，`projection`等存储结构，本文不详细解释。

<br>

# 2. MergeTree Write Path

`MergeTree`在内存中不存储数据，`MergeTree`写入时直接在磁盘上写入`data part`，而`update`和`delete`通过`mutation`的方式异步更新`data part`，`mutation`过程不保证原子性。后来也支持了`light update`。

当磁盘上每写入一个`data part`，都会递增赋予`block number`，如果后台发生`merge`，新生成的`data part`会包含`merge`前`data part`中最小的`block number`和最大的`block number`。输入`merge`的`data part`列表一定是相邻的，所以`merge`后的`block number`依旧可以表示数据的版本(新旧)。

当磁盘上发生`mutation(update/delete)`，`mutation`也会赋予当前递增最新的`block number`，并且只会对版本号比当前`mutation`小的`data part`进行重写，重写后的`data part`会记录当前这次`mutation`的版本作为自己的版本号。

总结下，没有发生过`mutation`的`data part`，`min block number`是自己的`data version`，发生`mutation`过的`data part`，`mutation`对应的版本是自己的`data version`。

<br>

# 3. Data Replication

`CK`是支持多副本的，副本间的数据同步通过`ReplicatedMergeTree`实现，比较有意思的两点：

- 副本间同步不是通过常见的`master-slave`模式，而是通过中心模块`keeper`(类似`zookeeper`，`CK`自己用`RAFT`实现)方式进行协调和同步。

- 支持多leader写入，通过`keeper`协调。

为什么`CK`的`data replication`这么特别？

首先因为`CK`同步的日志不是数据本身，比如有新的写入，在`keeper`上只是协调通知，副本间数据的同步最终是去其他节点上进行`get data part`的方式实现。又比如`mutation`，`keeper`上只记录`update/delete`的条件以及`mutation`的`block number`，实际的操作还需要在副本本地进行。

除了`mutation`，`merge`也是通过`keeper + log`的方式，协调所有副本进行。

至于为什么支持多点写入，因为`CK`对更新写入没有什么约束，只要最终的写入操作在所有副本上是顺序进行即可。

<br>

# 4. ReplicatedMergeTree

接下来详细看下`ReplicatedMergeTree`是怎么做的。

`Keeper`上包含一些路径，用于存放日志信息和相关辅助信息：

- `zookeeper_path`：多副本共享的公共目录。

- `replica_path`：副本的私有目录。

- `zookeeper_path/log`：包含副本间需要同步的日志，最简单的比如`GET_PART`，用于通知其他副本去远程获取新写入的`data part`。

- `replica_path/queue`：副本从`zookeeper_path/log`拉取日志，除了写到内存，还会写到自己在`keeper`上的私有目录，然后再执行消费日志。看起来有点多此一举，直接消费不就好了。

- `replica_path/log_pointer`：副本自己的目录下还会记录当前消费到的点位，遇到重启时，后面可以从这个点位开始消费。

- `zookeeper_path/mutations`：`mutation`操作会写到公共目录，多个`leader replica`可以并发写入，同时往这个目录下进行写入。

- `replica_path/mutation_pointer`：某个副本的`mutation`的进度，小于`mutation_pointer`的都已经完成。

<br>

`ReplicatedMergeTree`内部会包含若干任务(`BackgroundSchedulePool::TaskHolder`)，用于执行数据同步的流程。

### 4.1 Queue Updating Task

从`zookeeper_path/log`拉去日志到内存并写入到`replica_path/queue`。

**触发时机**

`Keeper上注册`watch，目录下有新数据时则会触发调度`queue updating task`。

**相关函数**

- `StorageReplicatedMergeTree::queueUpdatingTask`

- `ReplicatedMergeTreeQueue::pullLogsToQueue`

<br>

### 4.2 Queue Executing Task

通过`ReplicatedMergeTreeQueue::selectEntryToProcess`选择可以执行的`log`，然后根据不同`log`类型(比如`mutate`是生成`MutateFromLogEntryTask（MutateTask）`，选择不同的线程池执行，然后继续调度执行`queue executing task`。

从上面可以看出，从`queue executing task`的视角看，日志任务的执行可以并发。

对于非`mutate/merge`的`log`，`StorageReplicatedMergeTree::processQueueEntry`执行完可以直接删除对应`keeper`上节点。

**触发时机**

`Queue updating task`执行后会触发`queue executing task`。

**相关函数**

- `BackgroundJobsAssignee::threadFunc`

- `StorageReplicatedMergeTree::scheduleDataProcessingJob`

- `ReplicatedMergeTreeQueue::selectEntryToProcess`

- `BackgroundJobsAssignee::scheduleMergeMutateTask`

<br>

### 4.3 Mutations Updating Task

从`keeper`的`mutation`目录，拉取`mutation log`更新到内存。

根据当前磁盘上的`data part`以及`log queue`中待生成的`data part`，确定`mutation`对应的`data part`(根据`data part`的`data version`确定是否要`mutate`)。

**触发时机**

`Keeper`的`mutation`目录注册`watch`，有新节点时触发。

**相关函数**

- `StorageReplicatedMergeTree::mutationsUpdatingTask`

- `ReplicatedMergeTreeQueue::updateMutations`

<br>

### 4.4 Merge/Mutation Selecting Task

选择可以执行的`mutation`或者`merge`，通过`createLogEntryToMutatePart`写`log`到`keeper`。对于`mutation`，每次只会选一个`data part`，然后写到`log`。

为什么允许多个副本同时执行`merge/mutation selecting task`，因为更新`keeper`的`log`先读取`keeper`的`log`目录，拿到版本，写入采用乐观更新的方式，可能写入失败。失败后重新读取`keeper`的`log`目录，就不会导致多个副本同时写入相同`mutation`的情况。

**触发时机**

-  当`Commit`新的的`data part`。

- 当`mutations updating task`执行完，发现有新的`mutation`。

**相关函数**

- `StorageReplicatedMergeTree::mergeSelectingTask`

- `StorageReplicatedMergeTree::createLogEntryToMutatePart`

<br>

### 4.5 Mutations Finalizing Task

标记已经完成的`mutation`操作。

**相关函数**

- `StorageReplicatedMergeTree::mutationsFinalizingTask`

- `ReplicatedMergeTreeQueue::tryFinalizeMutations`

<br>

### 4.6 Cleanup Task

清理`keeper`上不需要的`log`，`mutation`。

主要逻辑在`ReplicatedMergeTreeCleanupThread`中。

<br>

# 参考资料

- https://zhuanlan.zhihu.com/p/144020199

- https://www.jianshu.com/p/11ce7a327c62

- https://github.com/ClickHouse/ClickHouse
