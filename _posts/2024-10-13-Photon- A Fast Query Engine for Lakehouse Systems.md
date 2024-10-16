---
tags: 论文
---

## ABSTRACT

业界现在流行一种新范式：湖仓，在非结构化的数据湖之上实现了结构化数据数仓的功能。

> 数据湖：HDSF、S3等
>
> 湖仓（lakehourse）：hudi、delta lake、iceberg

这时的查询引擎，要求分析非结构数据性能好，分析列存一样的结构数据性能强

Photon的特点

* 向量查询引擎
* 面对数仓的SQL查询性能高、能够分析原始数据
* 支持spark api

本文讨论了photon的一些设计选择(向量化、代码生成)，描述了与sql、spark runtime的集成能力，任务模型和内存管理能力

## 1 INTRODUCTION

##### 当前大部分的公司采用这样的数据处理范式

1. 将原始、非结构数据存储到数据湖(S3、Azure Data Lake Storage、Google Cloud Storage)
   一般是列式格式：Apache Parquet or Delta Lake 
2. 通过spark 、presto分析数据
3. 对于大部分的SQL需求，将部分数据迁移到数仓中
   * 高性能(high performance)
   * 可管理的(governance)
   * 并发性(concurrency)

##### 上述的2层架构存在以下问题

* 复杂、昂贵
* 只有部分数据被数仓管理，可用
* 数仓数据由于ETL等原因不同步

##### 解决方案：湖仓

湖仓特性：governance、ACID transactions and rich SQL support directly over a data lake

优点：简化数据管理、更少的ETL与查询引擎

湖仓（ Delta Lake）不断的在演进

* 新增了一些数仓的特性：transactions and time travel
* 优化存储层（ storage layer）访问的工具： data cluster、skipping indices

DataBrick为了提高查询性能，开发了Photon，主要有以下两个挑战

#### Challenge 1: Supporting raw, uncurated data

湖仓的丰富的数据类型，增加了查询引擎的挑战性，比如湖仓中会存在两种截然不同的数据类型

1. 经过精心设计的数据结构，包括数据约束、索引、统计等信息
2. 不理想的数据存储，比如小文件、很多列、数据稀疏、数据值较大等，以及缺少集群信息、统计信息
3. 字符串类型非常普遍，而且稍稍空值表示、编码类型(ASCII 、 UTF-8)等

基于上述的特点，lakehouse的查询引擎应该具有以下灵活性

* 能够处理**未经预处理的复杂数据**
* 在**优化过的、符合最佳实践的数据**上表现出**极高的性能**
  * multidimensional clustering 
    通过对数据进行多维度聚类，能够优化查询性能，特别是在多维分析场景下
  * reasonable file sizes
  * appropriate data types

解决上述问题的设计方案

* 使用vectorized-interpreted model 

  * runtime adaptivity

    通过对微批数据分析，选择合适优化方案

  * easier to build, profile, debug, and operate at scale

  * 保留抽象边界(query operator)可以收集丰富的指标，帮助终端用户更好理解查询行为

* 使用C++实现

  * 基于JVM的DBR存在性能上限
  * JIT的转换限制(方法大小)，导致优化失败时，可能会导致非常大的性能落差
  * native engine比JVM更容易解释性能问题
  * native engine除了性能的提升，还增大了处理记录的数量、使查询计划更简单

#### Challenge 2: Supporting existing Spark APIs.

##### 目的

* 将photon嵌入到spark 引擎中，并同时支持spark、pure sql语义的作业
* 需要与user-define code共享资源，保持与spark语义相同

##### 方案

photon在算子级别嵌入到DBR中，支持可选替换能力。是否选择使用photon执行，对用户是无感的，DBR根据具体情况决定使用photon执行还是旧算子执行

## 2 BACKGROUND

本节主要是介绍了湖仓系统

### 2.1 Databricks’ Lakehouse Architecture

Databricks湖仓系统的组成

#### 数据湖存储系统(Data Lake Storage)

* 存算分离
* 支持多种底成本存储(S3, ADLS, GCS)
* 用户无需数据迁移
* 支持开源数据存储格式，Apache Parquet

#### 数据自管理系统(Automatic Data Management)

Databricks使用Delta lake作为数据湖的存储层，他支持以下特性

* ACID事务(ACID transactions)

* time travel

* 审计日志(audit logging)

*  fast metadata operations

* 使用parquet存储数据与元数据

* 使用合理的访问层，数仓的很多存储优化也适用于开源格式(parquet)

* 其他性能优化(automatic data clustering and caching)

  > clustering records based on common query predicates to enable data skipping and reduce I/O

#### 弹性的执行层(Elastic Execution Layer)

<img src="/assets/images/image-20241013195508310.png" alt="image-20241013195508310" style="zoom:50%;" />

* Execution layer执行了所有任务
  * “internal” queries：auto data-clustering and metadata access
  * customer queries：ETL jobs, machine learning, and SQL
* 执行层特性：scalable, reliable, and deliver excellent performance

### 2.2 The Databricks Runtime

* 将一个作业(job)，解析成多个stage，每个stage根据数据情况变成多个task
* stage与stage之间是有届的
* task运行在executor中
* DBR只有一个driver node ，负责调度、查询计划、其他中心任务，并且管理着一个或多个executor
* task executor process是一个多线程、包含一个任务调度器、一个线程池，并行的执行由driver提交的独立的task
* SQL query经过DataFrame 对象最终会被转换成一个query plan
* 一个查询计划(query plan)是一个有SQL算子组成的树状结构，这些算子其实组成了一个个的stage
* photon就是在executor执行task的一种执行引擎，之前的执行引擎是spark sql提供的

## 3 EXECUTION ENGINE DESIGN DECISIONS

本节先描述了photon的整体架构，然后深入讨论了查询引擎的设计选择

### 3.1 Overview

photon架构的一些细节

* 是一个shared library，DBR可以直接invoke，在JVM进程中作为一个单线程任务运行
* Photon structures 的算子使用HasNext()/GetNext() API，既可以通过JNI从java中获取数据，也可以被java调用响应数据
* 使用向量方法操作列式存储的数据

### 3.2 JVM vs. Native Execution

> 这里有一个误区
> 上文中描述 `Photon runs as part of a single-threaded task in DBR, within an executor’s JVM process` 但是在本节中`JVM vs. Native Execution` 比较时，又说采用了`Native Execution`，看起来像前后矛盾了
>
> 答：
>
> 并不矛盾，只是两个纬度的描述。photon确实是jvm中的一个线程，但是其中的逻辑是C++实现的，通过JVM的JNI方式调用

1. 观察发现，大部分的负载都是CPU密集型(CPU-bound)
   *  **low- level optimizations** such as **local NVMe SSD caching** [38] and **auto- optimized shuffle** [55] have significantly **reduced IO latency**
   * techniques such as **data clustering**, allow queries to more aggressively **skip unneeded data** via **file pruning** [32] ，further reducing IO wait times
   * Lakehouse新增大量的工作负载，处理非规范的数据(un-normalized data)、large strings、非结构嵌套数据类型(unstructured nested data types)
2. 在JVM上进行JIT级别的优化是及其困难的，而且对低级别的优化操作(memory pipelining and custom SIMD kernels)的不可控,也导致了性能上线
3. 我们还发现，在生产环境中，我们也开始遇到JVM查询的性能悬崖
   * 堆内存超过64G，GC的性能会被严重影响
   * 为了解决这个问题必须对堆外内存内存进行管理，代码难以实现、维护
   * Java code generation受限于生成method的大小、代码cache大小，会降级成一个很慢的Volcano-style interpreted code path
     * 对于100个列的大宽表，经常会触发这一问题

考虑到JVM性能瓶颈、可扩展性，采用了native execution

### 3.3 Interpreted Vectorization vs. Code-Gen

|          | Interpreted Vectorization                                    | Code-Gen                                |
| -------- | ------------------------------------------------------------ | --------------------------------------- |
| 代表产品 | MonetDB/X100 system                                          | Spark SQL, HyPer [41], or Apache Impala |
| 优点     | 1. 动态调度根据数据情况选择代码执行<br />2. 批处理均摊虚拟方法调用成本<br />3. **enable SIMD vectorization**, and **better utilize the CPU pipeline** and **memory hierarchy** | 1. 通过生成专门代码，避免动态调用成本   |

选择Interpreted Vectorization的原因如下

##### 1. Easier to develop and scale

Code-Gen难以debug

##### 2. Observability is easier

由于code-gen会折叠、合并operator，导致无法通过采集指标，确认每个opertor的损耗

##### 3. Easier to adapt to changing data

根据运行时数据动态调整运行时状态

##### 4. Specialization is still possible

通过创建special- ized fused operators 实现相同的效果

### 3.4 Row vs. Column-Oriented Execution

* 更适配SIMD
* enable more efficient data pipelining and prefetching
*  more efficient data serialization
* 可以直接读写列式文件
* 可以通过维护字典，降低内存使用
  * 字符串、长数据

实际上，某些场景photon也会使用行数据，比如hash table

### 3.5 Partial Rollout

支持动态的切换photon与spark executor

* 与开源社区保持一致，版本不落后

接下来2个章节主要介绍上述设计在photon如何实现

## 4 VECTORIZED EXECUTION IN PHOTON

本节主要介绍向量列式查询引擎

### 4.1 Batched Columnar Data Layout

Photon的数据的最小单元：column vector，

column vector的组成： a contiguous buffer of values、a byte vector to indicate the NULL-ness of each value、batch-level metadata

column batch  的组成：多个column vector、 position list show `active` row

<img src="/assets/images/image-20241015083412145.png" alt="image-20241015083412145" style="zoom:50%;" />

验证记录是否活动、非活动采用byte vector也是常规方案，但是从实际效果与后续的研究发现，这种方案的性能不好

* 适合SIMD、但是需要遍历所有数据(O(batch size)),但是positon list O(active rows)

算子之间的数据处理基于column batch 粒度

### 4.2 Vectorized Execution Kernels

Photon将函数称为execution kernels，并针对每个函数使用vector的方式实现，比如

* expressions、probes into a hash table、serialization for data exchange、runtime statistics calculation

常规的优化方式有

* hand-coded SIMD intrinsics
* rely on the compiler to auto-vectorize the kernel (常用方式)

截止C++的template，kernel还可以针对不同data type做针对性的优化

data vectors + position list ====>> vector as output

一个kernel 的例子
<img src="/assets/images/image-20241015085331719.png" alt="image-20241015085331719" style="zoom:50%;" /> 

4.3 Filters and Conditionals

4.4 Vectorized Hash Table

4.5 Vector Memory Management

4.6 Adaptive Execution

## 5 INTEGRATION WITH DATABRICKS RUNTIME

Photon与其他传统数仓技术不同，它与DBR 、湖仓架构共享资源。并且与spark executor共同协作，在不支持Photon时，采用spark executor。所以photon必须与DBR、内存管理紧密结合

### 5.1 Converting Spark Plans to Photon Plans

Photon借助new rule in Catalyst 将spark使用的physical plan转换为photon使用的算子

转换的过程有以下几个特点

* photon不会从plan的中间开始
* 转换会spark时，会新增一个transition node ，做行列转换
* 对scan node转换时，新增一个adapter node，即可以实现行列转换，而且避免了数据copy

### 5.2 Executing Photon Plans

#### 简单的处理流程

1. 将photon plan转换成一个Protobuf  message.通过JNI传递给Photon C++ library
2. 算子使用了传统的范式： HasNext()/GetNext()
3. 在处理data change时，photon会将shuffle file按照Spark’s shuffle protocol 写入，并将shuffle metadata传递给spark
4. 但是处理shuffle时，photon写入的数据必须由photon算子读取

#### Adapter node to read Scan data

Adapter node的执行过程如下

* GetNext将2个pointer传递给JVM
  * vector of column value 
  * null values
* JVM 产生堆外列式数据，并按照原始数据存储
* 此时adapter node即获得对应的数据

#### Transition node to pass Photon data to Spark

将列式数据转换成行式数据

### 5.3 Unified Memory Management

photon与spark使用相同的资源，所以为了避免os或jvm将进程杀死，photon必须与能修改spark的内存

photon既可以要求spark释放算子，也可以被其他spark 算子要求释放内存(通过spill到磁盘上的方式)

使用的策略：对内存使用的算子排序，选择刚刚合适的算子进行资源释放

### 5.4 Managing On-heap vs. Off-heap memory

photon使用的内存主要是off-heap类型，无法被GC管理，所以如果photon依赖查询中的堆内存会产生问题，比如使用spark中broadcast的数据。photon通过一个监听器管理、清理Photon-specific state

### 5.5 Interaction with Other SQL Features

即时引入了photon，DBR、spark很多重要的性能优化都被保留下来。比如

adaptive query execution、shuffle/exchange/subquery reuse and dynamic file pruning、提供指标信息

### 5.6 Ensuring Semantics Consistency

photon通过三种方式实现与spark的语义一致

* Unit tests.
* End-to-end tests
* Fuzz tests

## 6 EXPERIMENTAL EVALUATION

本节主要解释下面三个问题

* 什么类型的查询从photon中获益最多
* 在端到端的查询语句中，photon与现在的查询引擎之间的差异如何
* 一些局部性优化的效果如何，比如adaptivity

### 6.1 Which Queries will Photon Benefit?

优化场景

* CPU密集型的作业：joins, aggregations, and SQL expression evaluation
  * native code, columnar, vectorized execution, and runtime adaptivity
* 其他一些operator：data exchanges and writes（Parquet Writes）
  * 借助编解码、内存处理等效果

没什么效果的场景

* IO or network bound

<img src="/assets/images/image-20241015215600613.png" alt="image-20241015215600613" style="zoom:50%;" />

### 6.2 Comparison vs. DBR on TPC-H

Photon可以实现最多23倍，平均4倍左右的性能提升

<img src="/assets/images/image-20241015215544647.png" alt="image-20241015215544647" style="zoom:50%;" />

### 6.3 Overhead of JVM Transitions

photon与jvm通过JNI进行互相调用，经过验证JNI调用的损耗及其少。大概只占0.06%，从adapter node获取数据损耗在0.2%，将数据序列化成scala对象的损耗占了95%，，列转行的操作没有额外的损耗。批处理后，均摊到每条记录的负载就更少了

### 6.4 Benefits of Runtime Adaptivity

#### Adapting to Sparse Batches

查询hash table之前，photon会先对列批数据进行压缩，以提高查询的并发度。这种处理方式大概会有1.5倍的性能提升

<img src="/assets/images/image-20241015220739575.png" alt="image-20241015220739575" style="zoom:50%;" />

#### Adaptive String Encoding Microbenchmark

photon采用了将uuid字符串编码成128-bit的数字方式，这种方案在时间上没有很高的提升，但是可以降低2倍数据存储，有利于数据的shuffle

<img src="/assets/images/image-20241015221011127.png" alt="image-20241015221011127" style="zoom:50%;" />