---
title: ClickHouse Create Table  
layout: post
categories: [ClickHouse]
image: /assets/img/clickhouse-logo.jpg
customexcerpt: "ClickHouse创建表"
---

简述下`ClickHouse`创建表的流程，比较粗糙，后面需要了再细看。

# 1. InterpreterCreateQuery

经过语法解析后生成`ASTCreateQuery`，然后进入`InterpreterCreateQuery::execute`。

- `Context::checkAccess`进行权限检查。

- 通过`DDLGuard`让同一个表的所有DDL只能串行执行，`DDLGuard`内部维护了`table`和`database`基本的锁。

- `DatabaseWithOwnTablesBase::isTableExist`检查创建的表是否已经存在，`DatabaseWithOwnTablesBase`维护了`table name`到`StorageMergeTree`的映射。

- `DatabaseOnDisk::checkMetadataFilenameAvailability`检查`metadata`目录下是否已经有该表的元数据，元数据其实是一条`create table`的`SQL`。这种情况是因为一个表被`detach`了，对于非`attach`的情况，要报错。

- 同样地，还会检查`data`目录下该表是否已经存在，对于非`attach`的情况，要报错。

- `StorageFactory::get`，新建`StorageMergeTree`。

- `DatabaseOnDisk::createTable`，把`StorageMergeTree`加入到`DatabaseOnDisk`。内存中用一个`map`维护，磁盘上会在`metadata`目录下写入`create table`的`SQL`文件。为了保证原子性，流程如下：
  
  - `creating the .sql.tmp file`;
  
  - `adding a table to tables`;
  
  - `rename .sql.tmp to .sql`;

- `StorageMergeTree::startup`，初始化`StorageMergeTree`中的一些复杂逻辑。

- `fillTableIfNeeded`，如果是`create select`的`SQL`，则还会插入数据。

<br>

# 2. StorageMergeTree

## 2.1 StorageMergeTree::StorageMergeTree

对于`StorageMergeTree`的构造函数，处理逻辑包括：

- `initializeDirectoriesAndFormatVersion`，在`data`目录下新建表的目录下，初始化`format_version.txt`，写入格式版本号。如果不是`create table`的场景，则进行`format_version.txt`的校验。

- `loadDataParts`，非`create table`的场景，加载该表磁盘上的`data part`。

- `loadMutations`，非`create table`的场景，加载该表待执行的`mutation`文件，文件名形如`mutation_1.txt`。

- `loadDeduplicationLog`：用来导数去重。

<br>

## 2.2 StorageMergeTree::startup

新建`StorageMergeTree`后，还需要调用`startup`才能正常工作。

- `clearEmptyParts`，清理没有数据的`data part`。

- `clearOldTemporaryDirectories`，清理一些可能由于突然重启导致残留的临时目录。

- 启动若干后台任务，包括周期性清理临时文件，`mutation/merge`后台任务，数据移动的后台任务(如果有多块盘，不同磁盘间移动`data part`)。


















