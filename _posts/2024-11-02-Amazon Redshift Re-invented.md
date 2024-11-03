---
tags: 论文
---

## ABSTRACT

2013 Redshift改变整个数仓行业，作为第一个全管理、PB级别、企业级的云数仓产品。

##### 传统预制的数仓

* 昂贵、没有弹性、需要有经验的调优、管理

##### RedShift最近的一些迭代特性

* 分层存储
* 自动扩缩容的多集群
* 多集群下数据共享
* AQUA 查询加速层
* 自动化(Autonomics)
* 支持 Serverless
  * 不需要数仓基础设施即可进行数据分析
* 与AWS的其他组件的无缝衔接
  * 通过Spectrum查询数据湖
  * 使用PartiQL写入、查询半结构数据
  * 通过Kinesis and MSK 消费实时数据，写入redshift
  * Redshift ML支持构建、训练、部署机器学习
  * federated queries 支持Aurora and RDS operational databases的联邦查询
  * federated materialized views 支持物化视图

## 1 INTRODUCTION

redshift专注于以下4个用户需求

##### 面对不断复杂的分析语句的高性能的执行能力

* 通过代码生成，将操作符(过滤、连接、聚合等)合并到每个查询片段
* 预取(prefetching)、向量执行
* 支持线性增长(scale linearly)

##### 处理更多数据、支持更多用户的能力

* 存算分离
* 支持动态调整集群大小
* 支持集群根据用户负载自动的新增、删除，以消除峰值
* 用户支持多个独立的集群消费同一批数据

##### 用户希望更易于使用

* redshift使用结合了机器学习的自洽系统，根据每个用户的独特需求进行调优各个集群
* 自动化工作负载管理、物理优化、MVs的刷新，以及根据MV对查询语句进行重写的预处理

##### 希望无缝融入AWS的生态，可以使用AWS专有工具

* 联邦查询能力
  * 事务数据库(DynamoDB\Aurora)
  * S3
  *  ML services of Amazon Sagemaker
* 通过Glue Elastic Views，可以创建事务数据库((DynamoDB\Aurora))的物化视图
  * 增量刷新(incrementally refreshed on updates)
* 借助 SUPER type and PartiQL可以写入、读取半结构数据

本文的结构组成如下

Section 2：系统概览、数据组成、查询处理流程，以及其他优化策略

Section 3：Redshift Managed Storage (RMS)，高性能事务存储层

Section 4：介绍计算层

Section 5：自动化能力

Section 6：与其他组件的无缝衔接能力

## 2 PERFORMANCE THAT MATTERS

### 2.1 Overview

![image-20241102140923450](/assets/images/image-20241102140923450.png)

#### 组成

##### 一个协调节点(a single coordinator (leader) node)

##### 多个计算节点(multiple worker (compute) nodes)

#####  Redshift Managed Storage

数据存储层

* 备份在S3
* 计算节点缓存数据(compressed column-oriented format)在本地的SSD中

##### 数据表存储的方式

* 每个计算节点备份
* 分区成多个bucket，分散在计算节点
  * round-robin、hash、base on 特定key

#### 扩展能力

##### Concurrency Scaling

##### Data Sharing

多个独立的集群提供数据分析能力

##### AQUA

借助FPGAs提升性能

##### Compilation-As-A-Service

缓存着查询片段优化后生成的代码

#### 查询方式

##### JDBC/ODBC connection

##### Data API 

#### 查询流程

![image-20241102143243871](/assets/images/image-20241102143243871.png)

* leader node 接受查询语句

* 执行2，parsed,rewritten,optimized

  * 优化器根据cluster’s topology、cost of data movement between compute nodes 、其他cost选择最优的查询计划

  * 计划根据分区Key尽可能避免数据的迁移

    比如如果是一个join，在相同的partition key进行join，那么可以直接在本地的分区下进行关联

* 通过workload management组件控制提交plan到执行阶段

* 一旦提交，the optimized plan会被分成一个个执行单元

  * end with a blocking pipeline-breaking operation
  * return the final result to the user

* 阶段4: 每个执行单元都会生成对应的optimized C++ code，分发到计算节点

* 阶段5:获取列数据从本地缓存或Redshift Managed Storage

#### 优化策略

##### 降低扫描数据块的方式

* evaluates query predicates over **zone maps**
* small hash tables that contain the **min/max values per block**
* leverages **late materialization**

##### 经过过滤后的数据会被分割成共享作业单元，可以被均衡的并行执行

##### 向量扫描、SIMD处理

数据解压、谓词处理

##### Bloom filters

created when building hash tables

##### 预取(Prefetching )

utilize hash tables more efficiently

#### Price-Performance

![image-20241102150519461](/assets/images/image-20241102150519461.png)

* 未优化版本可以有3倍的优势
* 优化后有1.5倍优势
*  linear scaling 能力，对用户来说成本可预测的

接下来介绍几个特殊的重写、优化与执行模型

### 2.2 Introduction to Redshift Code Generation

Redshift将query plan、schema生成对应的c++代码，然后编译成二进制，分发到计算节点

比如下面的sql

```sql
SELECT sum(R.val) FROM R, S WHERE R.key = S.key AND R.val < 50
```

会产生如下的代码

<img src="/assets/images/image-20241102152212275.png" alt="image-20241102152212275" style="zoom:50%;" />

* 这个代码块包含scans base table R (lines 3-6), applies the filter (line 8), probes the hash table of S (line 10) and computes the aggregate sum() (line 13)
* keeping the working set **as close to the CPU** 
  * kept in CPU registers

这种模式不会使用任何形式的解释代码(interpreted code)

##### standard Volcano execution model 的弊端

* 每个算子被被实现成一个个迭代器
* 函数指针或虚函数在每个执行步骤被动态选择对应的算子

弊端：产生编译代码会产生相应的延迟

### 2.3 Vectorized Scans

#### 图4的问题

*  𝑔𝑒𝑡_𝑛𝑒𝑥𝑡() 使用了pull-based模式，一个记录存在过多行时，会导致cpu register被耗尽，成本很高
* predicate evaluation (line 8) 会导致错误的判断分支，暂缓数据处理流
* 如果使用较复杂的压缩代码，会降低宽表的编译速度

#### 方案

* SIMD-vectorized scan layer
  * accesses the data blocks
  * evalu- ates predicates 
* the vectorized scan functions 会被提前编译，包含所有数据类型、及他们支持的编码、压缩方案
  * 通过谓词筛选，符合条件的记录被存储到栈上的局部数组上，并被后续步骤使用
  * SIMD降低了register的压力
  * SIMD减少了代码量，提高了宽表数个数量级的编译时间
* 这种设计结合了scan阶段的column-at-a-time execution 与join、聚合阶段的tuple-at-a-time 
* chunks的大小根据下面2个因素决定
  * total width of the columns
  *  the thread-private (L2) CPU cache

### 2.4 Reducing Memory Stalls with Prefetching

#### 问题

* Redshift’s pipelined execution避免了join、aggregate的中间结果的物化，使其一直在cpu寄存器中
* hash join的处理hash table，aggregations更新hash table产生完整的missing 开销
* 在push model中Memory stall 尤其明显，并且可能会抵消中间结果物化的成本

#### 探究

* 有一种方案是将数据分区，直到符合CPU cache，避免cache missing
  当数据量较大时，是不可行的
* redshift会将必要的列向下游传递，如果hash table比cpu cache大，会增加cache missing的延迟

#### 解决cache missing 的延迟

采用prefetch机制

* 在L1中维护一个环形缓存(circular buffer)
* 新记录到达时，会执行prefetches，push到缓存中，并将上一个pop
* 正常情况下会缓存多个记录，同时缓存、预处理。如果buffer满了，就会单个处理
* 面对大宽表，会产生多个阶段的prefetch，尽可能的让数据符合L1 cache

### 2.5 Inline Expression Functions

#### 问题

* 支持复杂的数据类型与表达式函数

#### 方案

* 生成的代码包含预编译头文件，对所有基本算子进行了内联处理，比如hash、字符串比较
* 根据查询的复杂度，表量函数会被解析成内联或常规函数
* 大部分函数都是标量的，但是其内部可能是SIMD-向量优化的
* 很多string函数会被定制化为SIMD code

### 2.6 Compilation Service

* 使用local cache、external code cache 查找对应的compiled segments
* 借助external compilation service 的并发能力降低编译的延迟
  99.5%

### 2.7 CPU-Friendly Encoding

* 支持通用的面相字节的压缩方法：LZO and ZSTD
* 可以针对特定类型使用相应的优化算法
  *  AZ64 algorithm：covers numeric and date/time data types
  * AZ64 与ZSTD相同的压缩效率，更快的解压能力
    3TB 42% 

### 2.8 Adaptive Execution

运行时,根据运行的指标(execution statistics),改变生成的代码(generated code)以及运行属性(runtime properties)

> 文中通过Bloom filter与hash join之间的例子解释这种动态转换

### 2.9 AQUA for Amazon Redshift

Advanced Query Accelerator (AQUA) 

* 多租户服务
* 集群外缓存层(off-cluster caching layer )
* 扫描、聚合的下推加速器

### 2.10 Query Rewriting Framework

#### 采用DSL-based Query Rewriting Framework (QRF)

* 重写、优化的速度非常的快
* 引入重写规则，优化unions, joins and aggregations的执行顺序
* 能够将嵌套的子查询转换为非嵌套的形式

* 支持生成脚本维护增量物化视图，并且通过物化视图替换query
* 重写的方式，简单到实习生即可实现相关的替换
  * pattern matcher
  * generator 创建新的query
* 引入与嵌套和半结构化数据处理相关的重写
* 扩大了物化视图的范围

## 3 SCALING STORAGE

#### 存储层的组成

* memory、local storage、cloud object storage
*  data lifecycle operations (commit, caching, prefetching, snapshot/restore, replication, and disaster-recovery)

#### 存储层的特性

##### Durability and Availability

每次commit都会将数据保存到S3，通过S3的特性保证

##### Scalability

s3提供了无限的可扩展性

RMS自动优化数据存储和性能

* data block temperature(冷热)

* data block age(访问的频率)

* workload patterns(访问模式)

  高峰访问之类的

##### Performance

内存、算法优化

* prefetching
*  sizes the in-memory cache
*  optimizes the commit protocol to be incremental

### 3.1 Redshift Managed Storage

* RMS管理了用户数据、事务元数据
* 通过多可用去(AZ)保证11个9的持久性、4个9的可用性
* 基于AWS Nitro System开发
* SSD作为本地缓存
  * 自动细粒度数据清理
  * 智能的预取能力

![image-20241103213440312](/assets/images/image-20241103213440312.png)

* s3中的数据快照

  * 支持集群、表 从任何可用恢复点 的快速恢复数据
  * s3是数据共享、机器学习的数据管道与原始来源

* prefetching方案提升性能

* RMS tunes cache replacement 

  可以将缓存情况，反馈给用户，供用户抉择是否扩缩容

* RMS就地调整集群大小，只是一个简单的元数据调整

  * 计算节点的无状态
  * 总能通过RMS访问到数据

* RMS是依赖于元数据管理所以比较容易扩容

* RMS的分层特性让SSD(cache)更换硬件更容易

  * balancing performance and memory needs of queries.

#### 表存储

* 分区成数据分片，组成blocks的逻辑链

* block header：identity, table ownership and slice information

* superblock：indexed of block header
  * 通过zone maps 找到指定的super block进行遍历
  * query tracking information

#### 事务请求被同步的commit到S3

* 保证多个集群的请求可以访问实时、事务一致性的数据

#### 跨集群写入S3(并发写入、读取)

##### 写入

* 批量数据写入
* 通过同步屏障来隐藏或减少写入操作的延迟

##### 读取

* 状态由一个节点控制
  * 状态（state）指的是数据的实际所有权和管理责任
* 查询和写操作在多个集群中并行执行，提供动态扩展的计算能力以支持大规模查询负载
* 查询基于快照隔离（snapshot isolation）
* 按需优先获取数据(cache or s3)

### 3.2 Decoupling Metadata from Data

这个特性使Redshift支持Elastic Resize and Cross- Instance Restore

Elastic Resize： 集群的增加计算节点、存储资源

Cross-Instance Restore： 从另一个集群恢复镜像

#### 实现方案

* 生成一个最小迁移数据的计划，并且导致一种balanced cluster
* 配置前，生成集群的记录、checksum，事后进行校验
  *  number of tables, blocks, rows, bytes used
  * data distribution, along with a snapshot

### 3.3 Expand Beyond Local Capacity

redshift通过S3扩展存储容量、利用SSD、本地磁盘作为缓存，为了实现这种转换，需要有下面的一些改变

* upgrading **superblock** to **support larger capacities**
* modifying **local layout** to support more metadata
* modifying **how snapshots are taken**
* transforming how to rehydrate and evict data
  缓存数据的重新加载与清理

本节主要描述tiered- storage cache and dynamic buffer cache

##### The tiered-storage cache

two-level clock-based cache replacement policy to track data blocks

分为冷热数据两个缓存层，数据先在冷层缓存，然后升级到热层，或者被清理

集群reconfiguration后，通过使用tiered-storage cache 实现rehydration

* reconfiguration (e.g., Elastic Resize, cluster restore, hardware failures)
* 大概20%的rehydration就能实现80%的缓存命中

##### dynamic disk- cache

*  the hottest blocks
* other blocks created by queries 
  * new data blocks and query-specific temporary blocks
* 根据内存情况自动清理

### 3.4 Incremental Commits

通过Redshift’s log-based commit protocol 替换redirect-on-write protocol，提升系统的性能

### 3.5 Concurrency Control

* 借助MVVC特性实现
* enforces serializable isolation

#####  graph-based mechanism

* track dependencies between transactions to avoid cycles and en- force serializability

**Serial Safety Net (SSN)**

* certifier on top of Snapshot Isolation 

## 4 SCALING COMPUTE

#### 4.1 Cluster Size Scaling

* light-weight metadata operation
* decouples compute parallelism from data partitions

#### 4.2 Concurrency Scaling

* resources are fully utilized and new queries start queuing
* automatically attaches additional Concurrency Scaling compute clusters and routes the queued queries to them.

#### 4.3 Compute Isolation

* securely and easily share live data
* queries a shared object, one or more metadata requests are issued
* authorized to access a data share.

## 5 AUTOMATED TUNING AND OPERATIONS

simplified many aspects of traditional data warehousing

*  cluster maintenance, patching, monitoring, resize, backups and encryption

其他待自动的作业

* routine maintenance tasks
  * schedule maintenance tasks (e.g., vacuum)
* performance knobs 
  * distribution keys

##### 自动化

* analyze or the refresh of materialized views in the background 
*  chooses query concurrency and memory assignment based on workload characteristics
*  au- tomatically applying distribution and sort key recommendations.
* make additional nodes available as soon as possible for node failures, cluster resumption and concurrency scaling
* offers a serverless option 

### 5.1 Automatic Table Optimizations

* Choosing appropriate distribution and sort keys 
* Automatic Table Optimization (ATO)  fully automated it
  * through the console users manually apply recommendations through simple DDLs
  * automatic background workers periodically apply beneficial rec- ommendations 

![image-20241104000006173](/assets/images/image-20241104000006173.png)

### 5.2 Automatic Workload Management

提交query的数量多少会影响集群的性能，太多、太少都是不合理的，redshift通过机器学习调整并发查询的数量

#### Redshift’s Automatic Workload Manager (AutoWLM) 

admission control、scheduling 、resource allocation

* converts its **execution plan** and **optimizer-derived statistics** into **a feature vector**
* estimate metrics like execution time, memory consumption and compilation time
*  finds its place in the execution queue

*  monitors the utilization of cluster’s resources using a feedback mechanism
* scheduling higher priority queries more often than low priority ones

![image-20241104000732690](/assets/images/image-20241104000732690.png)

adjust the **concurrency level** in tandem with the number of query arrivals leading to minimum queuing and execution time

### 5.3 Query Predictor Framework

These models are maintained by Redshift’s Query Predictor Framework

* predict the memory consumption and the execution time of a query
* collects training data, trains an XGBOOST model and permits inference whenever required

### 5.4 Materialized Views

Redshift automates the efficient maintenance and use of MVs in three ways

##### 1.  it incrementally maintains filter, projec- tion, grouping and join in materialized views to reflect changes on base tables

##### 2. Redshift can automate the timing of the maintenance

刷新的优先级

* the utility of a materialized view in the query workload
* the cost of refreshing the materialized view

95%的视图会在15分钟内刷新

##### 3.使用视图重写query

### 5.5 Smart Warmpools, Gray Failure Detection and Auto-Remediation

#### smart warmpool architecture

* prompt replacements of faulty nodes

* rapid resumption of paused clusters
* automatic concurrency scaling
* failover
*  many other critical operations

##### forecast how many EC2 instances are required for a given warm- pool at any time

*  built a machine learning model

#### gray failures

* outlier detection algorithms   that identify with confi- dence sub-performing components (e.g., slow disks, NICs, etc.) 
* automatically trigger the corresponding remediation actions

### 5.6 Serverless Compute Experience

automated provision- ing, sizing and scaling of Redshift compute resources.

* Serverless offers a near-zero touch inter- face.
*  pay only for the seconds they have queries running.
* Serverless maintains the rich analytics capabilities

## 6 USING THE BEST TOOL FOR THE JOB

### 6.1 Data in Open File Formats in Amazon S3

#### Spectrum

* access data in open file formats in Amazon S3 
* cost effective with pay-as-you-go billing based on amount of data scanned
* provides massive scale-out processing
* performing scans and aggregations of data in Parquet, Text, ORC and AVRO formats
* multi-tenant Spectrum nodes
* leverages 1:10 fan-out ratio from Redshift compute slice to Spectrum instance
* acquired during query execution and released subsequently.

如何使用

* register their external tables in either Hive Metastore, AWS Glue or AWS Lake Formation catalog
* 将 Spectrum 表的外部数据**本地化（localized）**到临时表中，内部表现为一个external table
* queries are rewritten and isolated to Spectrum sub-queries
  * pushdown filters and aggregation

##### 过程

* leader node generates scan ranges

  Either through S3 listing or from manifests belonging to partitions

* scan ranges are sent over to compute nodes.

  Along with the serialized plan

*  Spectrum instance retrieve S3 objects

  * result cache 
  * materialized views

### 6.2 Redshift ML with Amazon Sagemaker

![image-20241104003639961](/assets/images/image-20241104003639961.png)



### 6.3 OLTP Sources with Federated Query and Glue Elastic Views

针对AWS的OLTP数据库，常见的处理有两种方式

#### 借助Redshift’s Federated Query对数据原地分析

* sending subqueries with filters and aggregations into the OLTP source database 
  * speed up query performance 
  * re- duce data transfer over the network

#### 通过Glue Elastic Views将数据copy、同步到Redshift

ingestion of data from OLTP sources into Redshift.

*  enables the definition of views over AWS sources

* The views are defined in PartiQL
* GEV offers a journal of changes to the view, i.e., a stream of insert and delete changes
* the user can define Redshift materialized views that reflect the data of the GEV views

有点像CDC处理器

### 6.4 Redshift’s SUPER Schemaless Processing

SUPER半结构类型

* Redshift string 、num- ber scalars、arrays and structs

#### 使用场景

*  low latency and flexible insertion of JSON data
* 无需对接入数据的格式进行验证，任意格式
* 查询时不需要指定schema，支持filtering, join and aggregation
* 可以使用PartiQL materialized views分解成结构物化视图

### 6.5 Redshift with Lambda

支持基于AWS lambda代码的UDF函数

#### 使用场景如下

* 从外部数据源、数据API丰富数据
* 从外部数据提供者屏蔽或标记数据
* 迁移历史遗留的java c c++代码

