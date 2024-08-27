---
title: ClickHouse Serialization  
layout: post
categories: [ClickHouse]
image: /assets/img/clickhouse-logo.jpg
customexcerpt: "ClickHouse序列化"
---

`ClickHouse`关于序列化相关的代码位于`src/DataTypes/Serializations`。

# 1. ISerialization

`ISerialization`是不同序列化实现类的公共父类。

- 每个`DataType`有自己默认的`serialization`。

- `Serialization`可以被包装到特别种类的`serialization`，目前只有sparse这一种。

- 每个`serialization`有自己对应的反序列化后的`IColumn`。

- 每种`serialization`包含若干`stream`，大部分`serialization`只有一条`stream`，不同`stream`用`path`识别。
  
  - 对于单条`stream`，`serializeBinaryBulk`和`deserializeBinaryBulk`作为序列化和反序列化函数。
  
  - 对于多条`stream`，`serializeBinaryBulkWithMultipleStreams`，`deserializeBinaryBulkWithMultipleStreams`和`enumerateStreams`作为序列化和反序列化函数。
  
  - `enumerateStreams`是告诉外部当前的`serialization`对应有哪些`stream`，外部可以提前打开预取这些`stream`。

<br>

# 2. SimpleTextSerialization

定义了一些公共接口，如何以text，json，csv等格式进行序列化/反序列化。

<br>

# 3. SerializationNumber

- `SerializationNumber`针对的数据类型是整数，浮点数。

- `Stream`：单条`stream`。

- 传输格式：以`UInt32`为例，直接以小端方式连续存储数据。

- 内存格式：`ColumnVector<>`，可以直接随机访问。如果是大端机器，无论序列化和反序列化，都需要做转化。如果是小端机器，直接内存复制。

<br>

# 4. SerializationSparse

- `SerializationSparse`内部包含另一个`serialization`，主要针对一列数据中大部分是`default`值的情况。

- `Stream`：`offset stream`和`data stream`。

- 传输格式：`offset stream`是连续的变长`Uint64`，表示非`default`值在原始列中的`position`，`data stream`取决于内部`serialization`具体的种类。`Position`的变长`Uint64`读上来的最高`bit`位表示当前是不是`granule`的末尾。
  
  反序列化时，`DeserializeStateSparse`会为维护上次反序列化之后的状态，比如上次解析到了第`30`行，但limit是`10`，那么`DeserializeStateSparse`会标记`10`行之后的`20`行也是`default`值，供下次继续反序列化时使用。

- 内存格式：`ColumnSparse`，包含`ColumnVector<Uint64>`的`offset`数组，以及内部`serialization`具体对应的`IColumn`(第`0`个元素是`default`值)。访问某个下标的数据时，首先需要在`offset`数组二分查找，确定当前元素是否是`defualt`，或者在内部`serialization`中的位置。

<br>

# 5. SerializationArray

- `SerializationArray`主要针对`nested`类型。

- `Stream`：`size stream`和`data stream`。

- 传输格式：
  
  - `size stream`：小端连续存储Uint64，对应的是`data stream`中的元素`size`。
  
  - `data stream`：按照具体的类型存储。

- 内存格式：`ColumnArray`，包含一个`ColumnVector<Uint64>`作为offset以及具体的数据内容`IColumn`。注意这里的`offset`存放的是元素`offset`，所以可以随机定位

<br>

# 6. SerializationLowCardinality

- `SerializationLowCardinality`对于低基数列进行字典编码。

- `Stream`：
  
  - `key stream`：去重的列数据。
  
  - `index stream`：字典码数组。

- 传输格式：
  
  - `key stream`：
    
    - `8`字节的版本号，目前只有一种版本，即`SharedDictionariesWithAdditionalKeys`。
    
    - 连续的`granule`可能共享一份全局字典。
  
  - `index stream`：
    
    - `IndexesSerializationType`：每个`granule`的开头会把标记当前`granule`是否有额外的`key`，是否使用全局字典，是否需要重新从`key stream`更新全局字典。以及字典码的存储长度，`Uint8/Uint16/Uint32/Uint64`。
    
    - 字典码的数组。

- 内存格式：
  
  - 反序列化后生成`ColumnLowCardinality`。`ColumnLowCardinality`包含`ColumnUnique`和`ColumnVector`。`ColumnUnique`维护字典值，包括字典码到字典值以及字典值到字典码的映射。`ColumnVector`存放的是实际存储的字典码列表。

<br>

# 参考资料

- https://clickhouse.com/docs/en/sql-reference/data-types

- [聊聊ClickHouse中的低基数LowCardinality类型-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/1953125)


