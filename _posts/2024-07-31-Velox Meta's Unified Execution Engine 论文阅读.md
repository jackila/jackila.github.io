---
tags: 论文
---
>  [Velox: Meta's Unified Execution Engine](https://15721.courses.cs.cmu.edu/spring2024/papers/05-execution2/p3372-pedreira.pdf) 论文阅读

## 0.摘要

针对不同的数据场景，出现了不同的计算引擎。为了统一这些引擎，meta开发了velox，一个开源的c++数据处理组件。他有三个特点：向量化、自适应力、复杂的数据结构支持。当前velox已经继承在分析引擎(presto)、流处理平台、消息总线、数据湖预处理组件以及用于机器学习系统的预处理组件

优点

* 不同组件的优化能力相互借鉴，增强了系统的性能
* 对用户而言，不同语义的一致性
* 复用提升了工程效率

## 1.引言

#### 数据处理领域问题

* 事务处理
* 数据分析
* ETL&大数据移动
* 实时流处理
* 监控中的日志、时序数据的处理
* AI & ML
  * 数据预处理
  * 特征工程

#### 存在的问题

* 相似的优化手段
  * 缓存一致性加速器
  * NVRAM
* 支持tensor数据类型
* 不同的函数行为

#### 各引擎的不同点

* 语言前端（SQL, dataframes, and other DSLs）
* 优化器
* 任务在不同节点的分布方式
* IO层

#### 相同的执行引擎

* 类型系统
* 一种表示数据的内存格式
* 表达式evaluation系统
* 算子(sort\aggregation\join)
* 存储、网络的序列化
* 编码格式
* 资源管理单元(resource management primitives)

#### Velox特征

* 高性能、可复用、可扩展的数据执行引擎
* 支持复杂的类型
* 支持矢量化与适应性
  * 在运行时根据环境条件、输入数据或其他参数来动态调整自身行为或资源分配的能力
* 支持可扩展性
* 使用优化完全的查询计划作为输入，在本地节点执行计算
* velox不提供SQL解析器、数据层、DSL、全局优化器

#### Velox特点

* 效率
  * SIMD
  * lazy evalution
  * adaptive re-ordering and pushdown
  * 公共子表达式消除
  * execution over encoded data
  * code generation
* 一致性
* 工程效率

#### 本文的核心贡献

* 描述了与各种组件的集成
* 提供了一些基准测试指标
* 总结了一些经验与教训

## 2.包介绍

#### velox的角色

![image-20240730230054926](/assets/images/image-20240730230054926.png)

#### 核心组件

* 通用的类型系统
* 矢量特性
  * arrow-compatible 列式内存布局模块
  * 多种编码格式
  * lazy materialization pattern 
  *  out-of-order result buffer population
* Expression Eval
  * 完全矢量化的表达式执行引擎
  * 子表达式消除
  * 常量折叠
  * 高效空值传递
  * encoding-aware evaluation
  * dictionary memoization
* Function（自定义）
  * 标量函数
    * row-by-row simple
    * batch-by-batch vectorized
  * aggregate function
  * 与流行的SQL dialect兼容的function package
* Operator
  * TableScan、Project、Filter、Aggregation、Exchange/Merge、OrderBy、HashJoin、MergeJoin、Unnest
* I/O
  * pluggable 
  * file format encoder/decoders
    * ORC
    * Parquet
  * Storage adapters
    * S3
    * HDFS
* Serializers
  * Support different wire protocols
    * PrestoPage
    * Spark's UnsafeRow
* Resource Management
  * memory arenas 
  * buffer management
  * tasks
  * drivers
  * thread pools for CPU and thread execution
  * spilling
  * caching



## 3.Use Case

### 3.1 presto

![image-20240730233617763](/assets/images/image-20240730233617763.png)

### 3.2 spark

![image-20240730235759660](/assets/images/image-20240730235759660.png)

### 3.3 Realtime Data Infrastructure

![image-20240731001758330](/assets/images/image-20240731001758330.png)

#### 3.3.1 Stream Processing

* 批量处理数据，借助velox的矢量执行模型优化
* reuse velox operators
  * projections
  * filters
  * lookup join

* 时间窗口函数：tumbling、hopping、session windows

#### 3.3.2 Messaging Bus

* leverage the full extent of wire serialization formats available in Velox

  * column-oriented encoding
  * easily deserialized to Velox Vectors 

*  pushdown operations 

  * projections
  * filtering 

  > reducing the amount of data read from Scribe
  >
  >  cross data center traffic reduction

* the same semantics 

#### 3.3.3 Data Ingestion

* 数仓
  * ORC格式一致
  * 数据转换逻辑复用
* DB
  * 分区快照高效实现

### 3.4 Machine Learning 

#### 3.4.1 数据预处理

TorchArrow 内部将 dataframe 表示转换为 Velox 计划，并将其委托给 Velox 进行执行

#### 3.4.2 特征工程

## 4. DEEP DIVE

### 4.1 Type System

* 不同精度的整数和浮点数、字符串（varchar 和 varbinary 类型）、日期、时间戳和函数（lambda）
* 复杂类型，如数组、固定大小数组（用于实现 ML 张量）、映射和行/结构
* 支持任意嵌套，并提供序列化/反序列化方法
* 提供特殊数据类型，可以包装任意C++数据结构
* 自定义扩展
  * Presto 的 HyperLogLog
  * Presto 日期/时间特定数据类型

### 4.2 Vectors

* 内存中表示列示存储布局

* 支持各种编码格式

* 扩展了apache arrow格式

  * 向量大小
  * 数据类型
  * 空值位图

* 向量类通用方法

  * user copy
  * resize
  * hash、compare、print

* 支持丰富的类型

  * fixed-size & variable-size
  *  nested in arbitrary way
  * leverage different encoding formats 

* stored using  velox buffer

  * contiguous pieces of memory allocated from a memory pool
  *  support different ownership modes
    * owned & buffer view
  * reference counted
    * a single Buffer can be referenced by multiple Vectors
    * only singly-referenced data is mutable
    * made writable via copy-on-write

* 支持惰性向量

  * joins and conditionals in projections
  * useful when reading Vector data from remote storage (such as S3 or HDFS)
  * pushdown computation (such as aggregations) without having to materialize

* 支持解码矢量特性

  > 针对编码后的向量提供api接口，支持自定义实现，用于提高特定场景下的性能

  问题：针对一个未知编码的输入，在实现scala funtion or operator时，无法对向量创建进行控制

  * 优点：借助编码数据，提高处理的性能
    * dictionary-encoded input --->  distinct values 
  * 缺点：增加了程序的复杂度

  方案：

  * transforms an arbitrarily-encoded Vector into a flat vector and a set of indices
  * exposes a logically consistent AP

#### 4.2.1Arrow Comparison

##### String

`arrow`

buffer-size ｜ string content buffer

buffer-offset ｜ string content buffer

`velox`

```c++
struct StringView 
{ 
uint32_t size_ ; 
char prefix_ [4]; 
union {
	char inlined [8];
	const char∗ data; 
	} value_ ;
}

```

##### Out-of-order Write Support

* 解决IF 、 switch等分支操作
* 使用bitmask标记，每一行的执行分支
* 分别计算后写入同一个输出矢量
* 支持原始类型、字符串、数组、映射等等

##### More encoding

run-length encoding (RLE), and constant encoding



velox提供一个api供引擎使用，用来与arrow 进行相互转换

### 4.3 Expression Eval 

**向量执行引擎使用场景**

* used by the FilterProject operator, to evaluate filter and projection expressions
* TableScan and IO connectors to consistently evaluate predicate pushdown
* standalone component for other engines 

 **expression trees 的node的组成**

* 一个输入列
* 一个常量(字符)
* 函数调用(函数名+参数)
  * AND/OR
  * IF/SWITCH
  * TRY
* cast表达式
* 一个lambda 函数

>  还有一些元数据信息

##### 4.3.1 Compilation

将表达式树转换成可执行的表达式，一些优化细节如下

* Common Subexpression Elimination
  * strpos(upper(a), ‘FOO’) > 0 OR strpos(upper(a), ‘BAR’) > 0 >>>>> 𝑢𝑝𝑝𝑒𝑟 (𝑎) 计算一次
* Constant Folding
  * 𝑢𝑝𝑝𝑒𝑟 (𝑎) = 𝑢𝑝𝑝𝑒𝑟 (‘𝐹𝑜𝑜‘) >>>>> 𝑢𝑝𝑝𝑒𝑟 (𝑎) = ‘𝐹𝑂𝑂‘
* Adaptive Conjunct Reordering
  * AND(AND(AND(a, b), c), AND(d, e))  >>>>> AND(AND(AND(a, b), c), AND(d, e)) 

##### 4.3.2 Evaluation

compiled expression + an input dataset (represented using Velox Vectors)  ====> an output dataset

* a recursive descent of the expression tree
* passing down a row mask identifying the active elements

一些特殊场景，算子时可以不用计算的

* current node is a common subexpression and the result was already calculated
* the expression is marked as propagating nulls, and any of its inputs are null
  * can efficiently be implemented by SIMD

###### Peeling

* 1 k values in the range of [0,2]
* 0 - red, 1 - green, 2 - blue
* upper(color)
*  produce another vector of size 3: [RED, GREEN, BLUE]
* 将原始值（red、green、blue） 转换为 [RED, GREEN, BLUE]

###### Memoization

避免重复计算，将某些计算结果进行缓存，后续批次复用这些结果

总结：很多操作看起来比较简单，但当用于一些复杂数据结构上后，会有显著的性能提升。通过一些实际数据情况，velox分析消耗大量cpu的操作， 基于此做出了谨慎的优化建议

##### 4.3.3 Code Generation

实验性支持，将表达树重写成C++代码

### 4.4 Functions 

Velox 提供 API，允许开发人员构建自定义标量和聚合函数

##### 4.4.1 Scalar Functions

标量函数是接受单行值作为参数并生成单个输出行的函数

* 在velox中，标量函数也是矢量化的
* 输入参数以batch-to-batch提供
*  their nullability buffers & a bitmap describing the set of active rows

##### 优点

常数时间产生结果

* is_null() 借助internal nullability buffer(空值标记缓冲区)
* cardinality()  vector的元数据 （the internal lengths buffer that represents the size of each array in a vector）
* map_keys()/map_values()  使用MapVector 返回the keys or values internal buffer 

##### 缺点

无法利用列格式的其他function，需要处理大量的复杂度

* manually iterate over each input row
* correctly handle the nullability buffers
* different input (and output) encoding formats
* complex nested types
* allocating or reusing output buffers
* 高级字符串和 json 处理、日期和时间转换、数组/映射/结构操作、正则表达式、数据科学的数学函数

##### Simple Functions

* 隐藏底层引擎和数据布局的许多细节
* 提供与矢量化函数相同水平的性能

通过C++代码表达业务逻辑，逐行处理数据

```c++
class MultiplyFunction { 
	void call(
	int64_t& result , 
	const int64_t& a , 
	const int64_t& b) {
		result=a∗b; 
		}
}; 
registerFunction <MultiplyFunction , int64_t ,int64_t ,int64_t >({"multiply" });
```

* 通过DecodedVectors abstraction 隐藏 input data encoding format
* 使用 C++ template metaprogramming，将方法逻辑作用与batches of rows (Vectors)
* 框架通过优化、provides hints给c++ compile，确保大部分情况，所有的逻辑都是inline模式，允许编译器 auto-vectorization
  * automatically generate SIMD code
*  maps all primitive types to their corresponding C++ types
* Non-primitive types using proxy objects to prevent the overhead of materializing and copying data
  * strings, arrays, maps, and rows/structs

![img](/assets/images/2024_07_31_ac72788c78ff2d15f44cg-08.jpg)

> 简单函数实现不仅更容易阅读和编写，而且更高效

function还可以通过禁用4.3中的某些优化特性，提高某些算子的性能，比如rand() & shuffle()

##### Advanced String Processing

Simple function 还可以通过指定callAscii()，避免了默认使用UTF8的开销

另外很多字符串操作，如𝑠𝑢𝑏𝑠𝑡𝑟(), 𝑡𝑟𝑖𝑚(), and other string tokenization function，通过引用的方式，实现zero-copy

##### 4.4.2 Aggregate Functions

###### 计算方式

a. produces intermediate results

b. takes intermediate results and produces the final result

c. single aggregation: 如果根据某个key进行了分区，可以不用shuffle或者中间结果

d. 中间聚合，用于将部分聚合的结果组合在一起(多线程下)

###### 存储方式

```
+-------------------------+
| Row 1                   |
| +---------------------+ |   
| | Grouping Key Values | |
| +---------------------+ |
| | Fixed-size Accumulator| |
| | (inline)            | |
| +---------------------+ |
| | Pointer to Variable- | |
| | size Accumulator    | |
+-------------------------+

+-------------------------+
| Row 2                   |
| +---------------------+ |   
| | Grouping Key Values | |
| +---------------------+ |
| | Fixed-size Accumulator| |
| | (inline)            | |
| +---------------------+ |
| | Pointer to Variable- | |
| | size Accumulator    | |
+-------------------------+

...

+-------------------------+
| Variable-size Accumulator|
| Buffer                  |
+-------------------------+

```

### 4.5 Operators 

###### Query plans

> Filter、Project、TableScan、Aggregation、HashJoin、Exchange 

###### Operators

* 一对一

  Filter === Filter operator

* 多对一

   Filter  + Project === FilterProject Operator

* 多对多

  HashJoin ===== HashProbe & HashBuild

![image-20240801001134422](/assets/images/image-20240801001134422.png)

##### 4.5.1 Table Scans, Filter, and Project

* Table scans 支持 filter pushdown
* 优先处理 Filter 高效的算子、可以对filter算子进行排序，以提高效率
* simple filter operator SIMD 特性支持多行操作 
* Filter results for dictionary-encoded data are cached
* check cache hits by `gather + compare + mask lookup + permute `
* efficient implementation for large IN filters
  * hash join pushdown
  * trigger 4 cache misses at a time
* 优先进行filter 算子处理，减少数据处理

##### 4.5.2 Aggregate and Hash Joins

统一的hash table数据结构

* VectorHasher 分析批量的hash key，根据key ranges and cardinality，判断选择的数据结构
* all keys map to a handful of integers ---> a flat array.
* multiple keys & a single 64 bit normalized key ---->  `index a flat array`  or `a single hash table key depending on the range of this generated key`

* Others  ----> inefficient multipart hash key 
* 上面的逻辑是自适应的，针对不同数据组
* 借助上面的 `key ranges and cardinality`, can be push down to `TableScans and used as efficient IN filters`

### 4.6 Memory Management

* track memory usage via memory pools
* Small objects from C++ heap
  * query plans, expression trees, and other control structures
* larger objects using custom allocator 
  * data cache entries, hash tables for aggregates and hash joins, and other assorted buffers
* Memory consumers may provide recovery mechanisms 
* The default action for exceeding memory limit is to invoke a process wide memory arbiter.
* In order to support spilling, operators need to implement an interface that communicates how much memory could be released by spilling, and the actual spilling method

##### 4.6.1 Caching

对于一个分布式存储架构，velox通过memory&SSD caching降低了远程IO对查询延迟的影响

* 根据地层列存数据集的布局，支持任意大小的memeory cache
* 借助mmap/madvise，降低碎片化问题
* 通过文件的元数据，交错CPU与IO过程

## 5 EXPERIMENTAL RESULTS

性能基数

* 80个节点
* 64GAM & 2*2TBSSD
* 本地缓存
* 3TB ORC的TPC-H

![image-20240801214030932](/assets/images/image-20240801214030932.png)

* 协调器延迟
* shuffle 延迟

![img](/assets/images/2024_07_31_ac72788c78ff2d15f44cg-11.jpg)



总结：c++版本的presto节省了CPU，同时完全相同的工作负载，Prestissimo将服务器减少了3X。（60 vs 20）

## 6 FUTURE DIRECTIONS 

* For AI data to consume
* 计算资源的组件化和专业化

## 7 RELATED WORK

#### DuckDB

* 可嵌入式分析型关系数据库
* 矢量化引擎

velox作为一个查询组件，而DuckDB更多的是一个完整的数据库

#### The Apache Arrow project（Arrow Compute）

相同点：

* 支持 标量、矢量化和聚合函数

不同点

* arrow的场景更小
* 不支持 other SQL operators or resource management 

Gandiva 与arrow compute相同，只不是一个是解释矢量化、一个是即时编译

#### Photon

C++ 向量化执行引擎 与spark深度绑定

#### Optimized Analytics Package (OAP) 

也是以优化Spark为核心

## 8 CONCLUSION

通过velox解决了数据引擎孤岛的问题，统一现有的执行引擎。整合了meta中各种执行引擎

velox后续会统一运营和分析系统，与图形和监控引擎集成，以及进一步与 ML 平台融合。以及其他领域