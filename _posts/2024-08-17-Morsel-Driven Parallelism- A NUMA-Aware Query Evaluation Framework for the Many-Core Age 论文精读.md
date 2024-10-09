---
tags: 论文
---
> [Morsel-Driven Parallelism: A NUMA-Aware Query Evaluation Framework for the Many-Core Age](https://15721.courses.cs.cmu.edu/spring2024/papers/08-scheduling/p743-leis.pdf)

## 0. 概述

并发查询的性能受限于下面几个问题，无法充分充分利用最新的计算机架构的性能

* 充分利用多核处理器，将查询任务均匀的分散到上百个线程中
* 有序cpu的乱序特性，无法将均匀的将工作平均分配，即便是在有准确的数据统计下
* 非一致性内存访问（NUMA）的存在

当前常见的 plan-driven 并发的方式受限于

* 线程切换
* 数据的负载均衡（load balance）

本文的morsel-driven 框架具有以下特定：细粒度、运行时、NUMA-aware

* 将data 分割成small fragment，分发给worker thread
* 每个work thread都执行一个完整的operator pipeline （until next operator breaker）
* 并行度根据实际执行速度调整，非固定的
* 对于新查询请求，可以动态调整资源执行作业
* 调度器感知 NUMA-local morsels 以及 operator state 的数据本地性，将大部分的执行任务发生在NUMA-local memory 状态中

## 1. INTRODUCTION

90年待，针对火山模型，并行能力一般使用plan-driven方式：在query compile-time 时，优化器决定线程运行数量、每个线程实例化一个查询算子计划，通过exchange operators 将他们连接起来

![image-20240817175446615](/assets/images/image-20240817175446615.png)

* 一个query会分成多个segment，每个segement执行一个morsel（10000 tuples），在下一个pipeline breaker 实例化
* NUMA local processing： 线程在同一个core中执行读取、写入
* 固定线程数（一般是机器内核数），避免创建新的线程
* 线程pin到指定的core，避免切换线程导致NUMA-local的失效

1. morsel-drivern的一个核心特性是fullly elastic

* uncertain size distributions of intermediate results
* hard-to-predict performance of modern CPU cores 

2. morsel-driven 是一个完整的查询执行框架，所有的物理算子在每个执行阶段都需要支持morsel-wise的并发能力

3.  awareness of data locality

   * locality of the input morsels and materialized output buffers
   * the state created and accessed by the operators

   > maximize NUMA-local execution
   >
   > * 减少 remote NUMA access
   > * memory latency is optimized
   > * minimized cross-socket memory traffic

4. 动态分区实现的数据本地性  can be achieved by our locality-aware dispatcher

   * 火山模型隐藏了并行性，所以不需要share state

   * exchange operators 执行on-the-fly data partitioning 

     > 这种方式并非总是高效

   * 有些系统支持每个算子都并行执行，但它增加了算子之间的不必要同步

5. morsel-wise framework 可以集成到现有系统中

   * 比如exchange operator集成 morsel-wise scheduling
   * 引入hash-table sharing

6. 支持JIT code compilation

本文对主要贡献是提出了一个查询引擎的架构蓝图，包括

* Morsel-driven query execution
  * distributes work between threads dynamically using work-stealing
  * prevents unused CPU resources 
  * *elasticity*
* A set of fast parallel algorithms for the most important relational operators
* method to integrating NUMA-awareness

## 2. MORSEL-DRIVEN EXECUTION

> pipeline parallelization  vs morsels

基于如下的查询计划，下面分别描火山模型与morsel-driven的区别

![image-20240818081652143](/assets/images/image-20240818081652143.png)

![image-20240818081834476](/assets/images/image-20240818081834476.png)

常见的执行情况

* HyPer通过JIT会将上述的执行计划，编译成one code fragment
* Vectorwise 进一步采用 vector-at-a- time 的方式优化

morsel-driven 执行情况

* 生成一个dispatcher，第三个阶段必须在前两个阶段结束后执行
* 每个阶段都是相同大小的morsel，而不是通过morsel边界，避免数据倾斜
* 线程数不变
* 中间结果写入numa-local storage area

阶段一：

![image-20240818082846910](/assets/images/image-20240818082846910.png)

* morsel-driver完全的弹性： 由于每个线程每次只处理一个morsel 数据
*  enables perfect sizing of the global hash table
* a lock-free implementation is essential

阶段二

![image-20240818083540216](/assets/images/image-20240818083540216.png)

##### 与火山模型对比

* the pipeline is not independent
* share data structures
*  the operators are aware of parallel execution
* must perform synchronization 
* fully elastic
  *  different pipeline segments 
  *  the same pipeline segment

## 3. DISPATCHER: SCHEDULING PARALLEL PIPELINE TASKS

Morsel-driven 通过固定物理线程数避免线程的切换、终止，通过分配morsel的方式动态的调整并行度。一个task由a pipeline job 和 a particular morsel 组成。 通过验证，固定的10000个一批比动态实时调整大小性能会更好

3个核心目标

* Preserving (NUMA-)locality 
  * 分配的morsel会尽可能一直在同一个core中
* Full elasticity
* Load balancing
  * 避免 fast core wait for slow core

![image-20240818085228040](/assets/images/image-20240818085228040.png)

> each of the active queries is controlled by a *QEPobject*
>
> * 即dispatcher 只会接受到可以执行的pipeline job

### 3.1 Elasticity

通过 a morsel at a time 可以实现完全的弹性并发的能力，甚至可以实现处理优先级调度的组件，支持优先执行某些查询

* 每一个core都维护着一个独立的任务列表
* 每个pipeline job 维护一个待执行的morsel

### 3.2 Implementation Overview

* we maintain storage area boundaries for each core/socket and segment these large storage areas into morsels on demand

* the dispatcher is implemented as a lock-free data structure only

* QEproject也是一个被动状态机

  * 通过dispatcher调用
  * 在原本执行请求任务的线程上执行

* elasticity、load balancing and skew resistance

  * “steal work” from another core

* support *bushy parallelism* （树状并发）

  > 左深（Left-deep）平行计划\右深（Right-deep）平行计划\Bushy 平行计划

  性能提升一般，并且会破坏data locality，so e currently avoid to execute multiple pipelines from one query in parallel

* support query canceling

### 3.3 Morsel Size

* 比起vector/strides系统，there is no performance penalty if a morsel does not fit into cache

![image-20240820072935542](/assets/images/image-20240820072935542.png)

避免shared data structure成为瓶颈

* lock-free
* the total work is initially split between all threads
  * cache line align each range，conflicts at the cache line level are unlikely
* if more than one query is executed concurrently, the pressure on the data structure is further reduced
* it is always possible to increase the morsel size
  * fewer accesses to the work-stealing data structure
  * `enough concurrent queries` to solve `too large morsel size results in underutilized threads`

## 4. PARALLEL OPERATOR DETAILS

### 4.1 Hash Join

分为2个阶段

* 阶段一：the build input tuples are materialized into a thread-local storage area
* 阶段二：the parallel build phase each thread scans its storage area and inserts pointers to its tuples using the atomic compare- and-swap instruction
* outer join会增加一个marker 结构

一般情况下，highly-optimized radix join can achieve higher per- formance than a single-table join

However in comparison with radix join `our single-table hash join`

* fully pipelined for the larger input relation，uses less space 
* 在数据查询中，多个小的维度表可以像“一个团队”那样，通过大事实表的探测管道同时进行连接，而不是一个个关联维表
* is very efficient if the two input cardinalities differ strongly, as is very often the case in practice
* can benefit from skewed key distributions
* is insensitive to tuple size
* has no hardware-specific parameters

对于复杂查询，性能也是更好

>  97.4%  的 TPC-H benchmark 会执行probe 操作，Star Schema Bench- mark 这一比例达到99.5%。

同时也支持 radix join implementation

### 4.2 Lock-Free Tagged Hash Table

#### 独特的tag hashing 算法

hash table use an early-filtering optimization,核心思想有点类似位图与布隆过滤器

> tag a hash bucket list with a small filter into which all elements of that partic- ular list are “hashed” to set their 1-bit

![image-20240820081010625](/assets/images/image-20240820081010625.png)

文中的tagging算法比Bloom filter有以下几个优势

* for large tables，Bloom filter may not fit into cache
* Besides join，tagging is also very beneficial during aggregation when most keys are unique

####  only stores pointers, and not the tuples themselves

#### use large virtual memory pages (2MB) both for the hash ta- ble and the tuple storage areas

* The number of TLB misses is reduced，too many kernel page faults  during the build phase are avoided

* allocate the hash table using the Unix *mmap* system call

  * no need to manually initialize the hash table to zero  in an additional phase

  * the table is adaptively distributed over the NUMA nodes

    > 请求的线程会在local node 构建相应的page

### 4.3 NUMA-Aware Table Partitioning

* round-robin assignment

* hash value of some “important” attribute

  > 通过业务字段指定分区
  >
  >  this is more a performance hint than a hard partitioning

* hash函数保证了每个分区内数量的大致相同

> co-location scheme is beneficial but not decisive for the high performance 

### 4.4 Grouping/Aggregation

similar to IBM BLU’s aggregation

![image-20240820085319838](/assets/images/image-20240820085319838.png)

#### 阶段一

使用一个hread-local, fixed-sized hash table 进行本地预聚合，when full ， flushed， 所有的数据会被partitioned，然后在多个线程中进行交换

#### 阶段二

scanning a partition ---->  aggregating it into a thread-local hash table ----> fully aggregated ---> pushed into the following operator  before processing any other partitions

### 4.5 Sorting

![image-20240820090449280](/assets/images/image-20240820090449280.png)

* 本地排序后，计算出相应的separators，然后进行merge

## 5. EVALUATION

![image-20240820223752810](/assets/images/image-20240820223752810.png)

* For most queries, HyPer reaches a speedup close to 30
* overall performance is severely limited by its low speedup（加速比）,，Vectorwise is often less than 10
* query 6 具有明显的负载不均衡的问题
* 当我们关闭一些重要的特性，会影响程序性能
  * disable explicit NUMA-awareness
  * disable adaptive morsel-wise processing 
  * hash tagging

![image-20240820225934379](/assets/images/image-20240820225934379.png)

* Because of NUMA-aware processing, most data is accessed locally, which results in lower latency and higher bandwidth
* The table also shows that Vectorwise is not NUMA optimized
  * most queries have high percentages of re- motely accessed memory

![image-20240820231257381](/assets/images/image-20240820231257381.png)

* the default placement of the operating system is sub- optimal

![image-20240820231704420](/assets/images/image-20240820231704420.png)

*  throughput stays high even if few streams (but many cores per stream) are used
* This al- lows to minimize response time for high priority queries without sacrificing too much throughput

#### Star Schema Benchmark

* The scalability is higher than on TPC-H, because TPC-H is a much more complex and chal- lenging workload
* complex joins is beneficial

## 6.RELATED WORK

* radix hash join
* abort NUMA
  * pinpoints the relevance of NUMA-locality
* multi-core servers for parallel query processing.
  * IBM BLU query engine
  * Microsoft’s Apollo project
  * Oracle  reliance on query optimizer estimates
* Porobic et al. inves- tigated [29] and improved NUMA-placement in OLTP systems by partitioning the data and internal data structures in a NUMA-aware way [28]
* presented a hardware-oblivious approach to parallelization that allows operators to be compiled to different hardware platforms
* ex- ploit common work from multiple queries.
* vector-wise execution model  & the batch-mode execution 
* compiled query evaluation
* horizontal Volcano parallelism 

## 7. CONCLUSIONS AND FUTURE WORK

It is targeted at solving the major bot- tlenecks for analytical query performance in the many-core age

* load-balancing
* thread synchronization
* memory access locality
* resource elasticity