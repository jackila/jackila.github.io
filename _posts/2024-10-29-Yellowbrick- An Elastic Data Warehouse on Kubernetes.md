---
tags: 论文
---

## **ABSTRACT**

Yellowbrick Data Warehouse是一个高效、可扩展、伸缩的数仓系统，可以在云环境、私有数据中心运行。

* 它有一系列的k8s管理的微服务组成
* k8s是唯一控制源，处理系统配置、状态
* 管理数仓的所有生命周期操作
  * 计算资源、共享服务的创建、伸缩、释放
* 提供一套SQL操作接口供用户直接操作k8s
* 借助DPDK实现了一个可靠的网络协议

本文介绍了Yellowbrick的概览、微服务方法、已经相关的优化策略，并在最后总结了一些经验与未来的计划

## **1** **INTRODUCTION**

##### Prior to 2010

* 私有数据中心
* 在固定的资源占用下，追求效率与性能

##### 公有云环境

* 弹性
* 存算分离
* SAAS服务般的用户体验

k8s已经成为微服务事实意义上的标准编排框架，虽然有些数仓声称有能力在k8s上部署

* 并不是由精细的微服务构成的，依然是传统的架构，不能有效的利用k8s的特性
* 没有提供一套统一的SQL接口管理K8S

## **2** **OVERVIEW OF YELLOWBRICK**

##### Yellowbrick产品特性

* 满足ACID特性(ACID-compliant)
* MPP架构
* SQL类型的关系数仓

##### Yellowbrick技术特性

* 即时弹性(instant elasticity)
* 可扩展(scalability)
* 高性能(performance)
* 高效的(efficiency)
* 高并发(high concurrency)
* 高可用的(availability)

##### Yellowbrick的组成

* *data warehouse manager*

  管理多个、独立的data warehouse instances的控制面板

* *data warehouse instance*

  管理一系列的数据库

* *compute clusters*

  * 弹性、可伸缩
  * 根据不同的负载情况，向*data warehouse instance*添加计算资源

<img src="/assets/images/image-20241029211921007.png" alt="image-20241029211921007" style="zoom:50%;" />

##### 读算分离

每一个计算节点都会挂在一个NVMe SSD-based 共享缓存，通过缓存分片数据提高查询性能

##### 可伸缩性

* compute cluster可以一次增加1到64个节点
* Compute clusters可以通过配置，选择自动根据查询负载选择挂起、恢复，以实现底层计算资源的释放、申请
* 被一个instance管理的database，对每一个Compute clusters可见
* 最多可以有3000个worker(mpp process)可以分配给一个instance，被打包成一个cluster
* 单一用户可以分配一个或多个cluster，并通过负载均衡的方式分发作业

##### 支持多租户

### **2.1** **Microservices Architecture**

<img src="/assets/images/image-20241029214152932.png" alt="image-20241029214152932" style="zoom:50%;" />

* 所有的组件都是由一个个的微服务，通过k8s管理可用性与扩展性
* data warehouse instance 是服务的入口
  * manages connections、query parsing、query plan caching、row store、metadata management、transaction management
  * singleton StatefulSet pod
  * Compute intensive tasks（bulk data loading、query compilation） 可以委派给水平扩展能力的ReplicaSet pods
* data warehouse manager 有一组pod组成
  * UI, authentication, monitoring, configuration management and workflow services
* 一个 compute node会执行一个worker process(MPP process)
* 支持SQL接口管理K8s资源

### **2.2** **Deployment Approach**

##### 如何让部署过程足够的简单

AWS上借助AWS CloudFormation service实现

* VPC、load balancers、subnets、security groups、Elastic Kubernetes Service cluster
* 镜像可以从 Elastic Container Registry自动获取
* 通过data warehouse manager UI可以创建data warehouse instances

##### 效果

* work flow自动为compiler and bulk loader 安装StatefulSet instance pod、ReplicaSet pods
* EKS 自动扩缩容计算资源
* Persistent volume claims 自动获取必要的存储资源
* 通过SQL、UI，compute clusters可以从EKS中分配

## **3** **SOFTWARE OPTIMIZATIONS**

yellowbreck不但在数据库管理软件层面上实现了很多优化，还通过os bypass的技术，解决了操作系统在存储、网络、内存管理、调度等方面的低效。并且自动化了管理、维护数据库的很多任务

### **3.1** **Database Optimizations**

Yellowbrick实现了大部分MPP数仓所有具有标准SQL优化与算法

* parallel query plans（并行的查询计划）
* cost-based optimization（基于代价的优化）
* workload management（负载管理）
* parallel query execution（并行查询执行能力）
* Query Plan被转换成C++ code ，通过编译微服务编译，并分发到worker节点

##### The SQL parser and planner 

* 基于PostgreSQL 9.5
* 支持hash, sort-merge and loop joins
* SQL rewrites
  * 下推、消除、推断、谓词与join语句的简化
* planning joins, aggregates and scans 时，会进行代价估算
* 主键、外键约束会被用于在join算子中的指标的基数估算
* 基数估算基于HyperLogLog算法

无共享(share-nothing)数据库，处理数据采用下面**三种分布式策略**(row)

 基于query execution plan进行选择

* hash values in a specified column
* randomly
* replicated across workers

workers由执行引擎(execution engine)与存储引擎(storage engine)组成

#### execution engine

* 使用基于信用的框架控制每个查询消耗的资源
* 受负载管理规则约束
* 管理内存、线程、调度、与其他work的通信以及查询的整个生命周期
* 执行查询计划的代码实例(an object code instantiation of a query plan)
  * LLVM

<img src="/assets/images/image-20241030204246768.png" alt="image-20241030204246768" style="zoom:50%;" />

* 执行引擎处理一个查询图，它的节点与 SQL planner生成的抽象查询计划中的节点一一对应
* 执行开始，每个线程会获取一个信用点(credit)
* 这些信用点用于根据我们的**工作负载管理系统**设定的**限制**来控制每个查询**使用的内存和临时磁盘资源**
* **信用点**向查询图的**叶节点流动**，**数据包**向上流动
* 图节点只有在拥有信用点的情况下才能处理数据包
* Link关联节点之间的连接，负责信用点的计算，可以将数据分发到不同的线程(同步、异步)
* 叶子结点负责从存储中读取数据

* distribution operator也使用了相同原理的流控与反压，以至于从单个查询执行图扩展到所有worker。优化了数据流动和内存使用

* 图节点只能主动释放控制权，不能被其他查询打断

* 根据所涉及的图节点类型，以不同方式处理数据是最佳选择
  * 选择它们能够处理的数据包格式类型
  * 转置节点被注入到执行计划中
* 支持多核并且NUMA-aware
  * primarily core- local and secondarily NUMA-node-local
  * 数据倾斜时，会进行reallocation

#### storage engine

* 管理列存分片数据文件
* 执行查询计划中的 table scanning leaf nodes 
* 过滤条件下推
  * skip a shard file entirely
  * skip components of it
  * 根据文件名、头信息、列元数据等过滤数据
  * 解压数据后，还可以使用Dynamically-created Bloom 过滤
* 基本的处理流程
  * 通过PCIs bus从NVMe cache中读取数据
  * 解压、数据格式转换、通过SIMD过滤数据，传递给查询引擎
  * 每个数据包256k，正好适合L3 cache大小
  * 采用自定义开发的NVMe驱动，在用户内存操作数据，避免内核开销

#### 资源分配方式

一个compute cluster可以有多种workload management ，计算、内存、临时存储被分为不同的pool，根据query中的特征(用户、角色、应用程序、数据库、查询标签),选择不同pool。query会被赋予不同优先级、限流，并在资源超额时自动取消重启

允许混合工作负载（例如数据加载和查询）

#### 查询生命周期

<img src="/assets/images/image-20241030214011966.png" alt="image-20241030214011966" style="zoom:50%;" />

### **3.2** **Operating System Optimizations**

#### 核心目标

数据直接从NVMe SSDs中读取出来，然后加载到CPU caches中，使得后续的查询直接在L3 cache中处理数据，而不是在主内存

#### 方式

替换linux系统内的内存管理与任务调度

##### 内存管理器

* avoid kernel swapping
* avoid memory fragmentation
  Memory allocations are grouped by query lifetime
* largely lock-free
* NUMA-aware
* 分配器使用的所有内存都在一个连续的虚拟地址区域中进行 mmap
* 使用 2 MB 或 1 GB 的 HugePage 块中的内存
* 这些初始页面被 mlocked，强制它们保持在其初始物理地址的内存中
* 内存分配器完全在这个连续的虚拟地址空间内工作
  * 实现更少位数的寻址
  * 节省内存元数据存储的空间

##### task scheduler

* 运行在用户空间
* context switch between queries in ～100 nanoseconds
* 一个查询的执行在不同节点是同步的，并且保证同时在同一个阶段
  * 当发生数据分发时，网络层数据包可以避免等待而发生堆积，导致数据无法存储在L3 cache，转而存储到了main memory
* 时间被划分成同步的里秒slot
* 在一个slot中，整个集群只会执行一个query，所有的cpu资源都被用于这一个query，slot结束会切换到另一个query
* 能够处理任务的优先级，优先处理新任务而不是长时间作业
* 能够协调集群的所有节点资源

### **3.3** **Networking Optimization using DPDK**

采用DPDK(Data Plane Development Kit )在节点间交换数据

* Low latency
*  high bandwidth
* bypass the kernel network stack
* avoid intermediate copies and system calls
* directly address the network device from within user space

基于UDP的网络协议

* reliable, ordered packet delivery and minimize CPU overhead

Seastar虽然通过DPDK实现了用户态的网络处理，但是它是基于TCP协议，没有考虑数据包的可靠性

我们的实现中，一个worker的每个vcpu的线程可以与另一个worker的任意一个线程连接，并独立维护接受、发送队列，独立处理数据包，并且通过Receive side scaling技术将接受端数据负载均衡到多个线程中

<img src="/assets/images/image-20241030225927693.png" alt="image-20241030225927693" style="zoom:50%;" />

* 使用DPDK后，性能提升可以到20%

<img src="/assets/images/image-20241031210203652.png" alt="image-20241031210203652" style="zoom:50%;" />

* 横轴是网络IO的数据量
* 网络IO增大会有效的提升性能，其他Q50这个查询语句尤其明显

### **3.4** **Storage Optimizations**

采用了混合的存储引擎设计，前端使用行存、后端使用列存。行存由 data warehouse instance 管理

* 数据会立刻插入行存，并自动的刷新到列存中
* Bulk load则会直接插入列存，跳过行存过程
* 使用一个**共享的事务日志**、设置**“read committed”隔离级别**、以及使用**多版本并发控制**，数据库能够在**行存储和列存储**之间保持**ACID属性**，确保数据的安全性和一致性
* shard file是不可修改的，delete record通过bitmap进行标记，系统定时进行合并操作
* workers从对象存储中一次性读256k数据，并将其在SSD中缓存。结合了块读与预读的优化策略
* 采用了一种LRU策略对缓存管理
  * 新数据会写到列表的最下面，只有再读时才会设置到头部
* 数据加载时，记录会直接写入到对象存储中，节点负责的文件会根据情况改变所有者
* 每个shard文件100M左右，每次写入时为2M block
* 绕过NVMe cache 直接写入对象存储，只会通过读进行缓存填充
* 自主研发了C++ S3 connection library

## **4** **PERFORMANCE COMPARISON**

<img src="/assets/images/image-20241031212916719.png" alt="image-20241031212916719" style="zoom:50%;" />

Yellowbrick cluster executes the TPC-DS 1 TB workload 2-5x faster than the other platforms

<img src="/assets/images/image-20241031213103298.png" alt="image-20241031213103298" style="zoom:50%;" />

* 相对成本会有2-3倍的提升
* 除了yellowbrick，其他的数据库采用了相同的优化手段，所以性能相近

### **4.1** **Scaling Compute and Data**

<img src="/assets/images/image-20241031213453499.png" alt="image-20241031213453499" style="zoom:50%;" />

* 表4显示price-performance 是线性的，可观察，可预测的
* 图6显示给定数据量与运行节点的比例，运行时间相同

### **4.2 Concurrency Scaling**

<img src="/assets/images/image-20241031215247845.png" alt="image-20241031215247845" style="zoom:50%;" />

* 相对运行时间随着并行度的增加、即负载的增加，线性增长
* 实际执行时间要稍稍犹豫预期，可能是由于缓存复用的效果

![image-20241031215606841](/assets/images/image-20241031215606841.png)

相同的查询负载，通过增加集群性能，可以看到明显的性能提升

### **4.3 Multi-cluster Scaling**

支持通过负载均衡将查询分发到不同的cluster中

![image-20241031215901855](/assets/images/image-20241031215901855.png)

* 负载被均匀的分散到各个集群中

用户既可以在一个集群中，一个节点一个节点的增加集群性能，也可以提供多个集群以支持业务量的增加

## **5** **CONCLUSIONS AND FURTHER WORK**

* Yellowbrick通过k8s管理资源、生命周期，可以让我们更关注于数据库的性能与新特性
* k8s没有影响到我们使用os bypass的特性
* os bypass极大的提升的数据库的性能
* yellowbrick的扩展性是线性的，多个维度
  * compute cluster size
  * number of compute clusters
  * data volume
  * degree of query concurrency
* 使用lz4压缩可以降低50%的流量大小
* 其他在进行中的优化
  * extending our range-based filtering to support multiple ranges per column
  * support for more **complex Bloom filter expressions** and the selective application of **filters based on cost**
  * automated SQL query rewrites in the planner