---
tags: 论文
---

## ABSTRACT

Google Dremel是第一批将当前云分析工具流行的架构原则应用实践的应用，这些实践包括

* 存算分离(disaggregated storage and compute)
* 原位分析(in situ analysis)
* 半结构数据的列示存储

本文主要介绍这些特征在过去10年如何发展，并最终成为google bigquery的基础

## 1. INTRODUCTION

2010年Dremel第一次出现在某个论文中，同年big query以Dremel为基础开始研发

BigQuery是一个 **fully-managed**, **serverless data warehouse** that enables scalable analytics over **petabytes of data**

本文主要关注Dremel的核心思想与架构原则

* SQL：Dremel通过一个通用、开源的框架支持SQL范式的查询

* *Disaggregated compute and storage*：架构支持存算分离，可以在各自纬度上独自扩展

* *In situ analysis*：Dremel借助分布式文件与一些通用的数据访问工具，允许与MR&其他数据处理工具在SQL层面互相操作

  > 比如多个数据源的join操作

* *Serverless computing*：Dremel不需要提前分配资源，可以自动扩容

* *Columnar storage*：Dremel通过一种创新的编码方式，提高了嵌套结构、半结构数据的易操作性

另外本文介绍了Dremel降低延迟的方式

## 2. 拥抱SQL

起初，google内部相信SQL doesn’t scale，在解决大数据方面的问题提出了很多解决方案，如GFS、MapReduce、BigTable等，而且通过一个新语言Sawzall更简单的让实现业务需求

SQL的优点：允许用户快速的实现、运行、迭代查询语句

<img src="/assets/images/image-20241008205047094.png" alt="image-20241008205047094" style="zoom:50%;" />

F1当前的发展方向

* HTAP功能
* 跨多个专门化存储系统进行联邦查询

ANSI standard for SQL存在诸多问题，所以google开发了一套新的SQL框架：GoogleSQL

其他基于hadoop、mapreduce、其他NOSQL系统的公司会通过HiveSQL、SparkSQL、Presto解决出现的复杂性与迭代速度慢的问题

## 3.解耦

### 3.1 Disaggregated storage

##### 最初的设计

* 几百台shared-nothing服务器运行
* 每台服务器在本地存储互不重复的数据
* 提高性能的方式只能通过：专用硬件(dedicated hardware) 与 直接连接的磁盘（direct-attached disks） 实现

##### 问题

* 负载增加后，很难通过部分专用服务器进行管理

##### 解决方案

* 迁移到Borg上
  * 解决了查询负载的问题，并提高了共享资源的利用

##### 新问题

* 共用硬盘

##### 解决方案

* 备份：一个表分别存储在3个不同本地磁盘，并被不同的服务器管理

通过集群管理、数据备份，Dremel大幅提升了它的可扩展性以及速度，可处理数据从 petabyte-sized到trillion- row tables

问题：备份导致计算与存储的耦合性更紧密

* 增加新特性需要考虑备份
* 系统无法在不迁移数据的情况扩缩容
* 扩容存储，同时要求增加服务，升级CPU
* 无法被非Dremel系统访问

方案：上述方案的解决方案与GFS及其的相似，以至于可以直接复用

##### 新问题：第一个版本的 GFS-based Dremel，性能低了一个数量级

* 扫描一个table时，由于大量的tablet导致的大量的文件，导致耗费大量时间
* 最初设计的metadata格式是以磁盘查找为基础而不是网络请求

优化细节第7节会详细描述，大概包含：存储格式、元数据表示、查询亲和性(query affinity)、预读(prefetching)

#### disaggregated storage其他优点

* 使数据更开放自由、降低了复杂度

* improved the SLOs and robustness of Dremel

  > GFS

* 不需要从GFS迁移通用表到Dremel

* 其他团队使用Dremel更简单

### 3.2 Disaggregated memory

Dremel通过shuffle实现distribution join的能力，利用本地RAM与disk存储临时结果。

将计算节点与中间结果shuffle存储耦合造成了扩展性的瓶颈

* 无法解决 shuffle操作导致成本或复杂性将呈现非线性、平方级的快速增长
* 资源碎片化、容易被闲置、较差的隔离性，成为了扩展性（scalability）和多租户（multi-tenancy）环境中的一个主要瓶颈

##### 解决方案

* 2012年，通过Colossus distributed file system 实现了一个disaggregated shuffle infrastructure
  * encountered all of the challenges described in [38, 47]
*  2014， settled on **the shuffle infrastructure** which supported completely **in-memory query execution** 

<img src="/assets/images/image-20241008221514522.png" alt="image-20241008221514522" style="zoom:50%;" />

* managed separately in a distributed transient storage system

新shuffle 实现的效果

* 降低了一个数量级的延迟
* 最大支持的shuffle数据量提高了一个数量级
* 降低资源消耗超过20%

### 3.3 Observations

分离已经是一个非常重要的趋势，主要有以下几个特性

* 经济规模(economies of scale): 存储层的发展路线，RAID、SAN、GFS、云仓库级（Warehouse-scale computing）
* 统一性(*Universality*): 分析系统与事务系统都倾向于支持存储分离，如Span- ner [17], AWS Aurora [44], Snowflake [18], and Azure SQL Hyperscale
* *Higher-level APIs*：分离后的资源可以通过高级别的API访问，封装了如权限控制、静态加密、客户管理的密钥、负载均衡、元数据管理，甚至是过滤聚合等操作
* 增值重新打包（*Value-added repackaging*）：原始资源被封装到某个服务中，提供特定的能力。比如上文提到的shuffle服务对RAM的封装

## 4.原位分析

原位数据处理是指：不需要提前数据加载、转换，可以在原始位置访问数据。比较经典的实现是mapreduce

在数据仓库向数据湖转换中，有三个核心组成已经被dermel应用

* 使用不同的数据源的数据
* 移除从事务系统到数据仓库的ETL过程
* 支持大量计算引擎操作数据

### 4.1 Dremel’s evolution to in situ analysis

初版本：需要数据加载、指定数据格式、数据无法被其他工具访问

迭代：迁移到GFS后

* 通过基础包开源了存储格式
  * 列式存储，自描述
  * 每个文件不仅存储了一个表的分区数据，还包括精准的元数据（schema、衍生信息（数据的范围））
  * 自我描述的存储格式支持转换工具与sql分析工具的互相操作

简单描述比如，数据可以通过map reduce作用在列存数据、将结果落盘最后被dremel查询到

#### 迭代方向

* 新增文件格式，如 record-based
* 支持更多的联邦查询
  * 直接访问远端文件系统如：Google Cloud Storage and Google Drive
  * 通过其他查询引擎API访问数据，如 F1, MySQL, and BigTable
  * 增加可连接数据的种类与范围
  * 利用原有系统的特性进行查询，如bigtable的 raw key

### 4.2 Drawbacks of in situ analysis

* 数据没有那么安全、保密
* 无法优化存储层性能或者统计信息

上述问题通过BigQuery Managed Storage进行了解决

> 有一些产品尝试将 in situ 与 managed storage 整合在一起，如NoDB and Delta Lake

## 5.无服务计算(serverless compute)

本节主要介绍在serverless方面，Dremel与工业界采用了哪些不同方式、serverless的核心点、以及工业界如何采纳这些想法

### 5.1 Serverless roots

3个核心思想

* 解耦

  按需分配、存算资源分离、较低的成本执行

* 容错与可重启能力(*Fault Tolerance and Restartability*)

  * 每个子任务都是确定的、可重复的
  * 调度器允许调度多次重复的子任务，缓解未响应的任务

* 虚拟调度单元：借助slot概念分配资源

### 5.2 Evolution of serverless architecture

#### 中心化调度（*Centralized Scheduling*）

<img src="/assets/images/image-20241009205653736.png" alt="image-20241009205653736" style="zoom:50%;" />

通过分配资源给上图的intermediate servers，使资源利用率更高并保证隔离性

#### shuffle持久层(*Shuffle Persistence Layer*)

通过解偶调度、不同查询阶段的执行，将shuffle的中间结果作为查询执行状态的一个checkpoint，调度器就可以灵活的运用动态抢占作业的能力，解决资源不足的问题

#### 灵活的执行DAG(*Flexible Execution DAGs*)

> 固定的执行树对聚合操作有益，但是对复杂的查询并不理想

1. 接收sql节点，作为协调节点，生成对应的查询执行树，根据调度器分配的workers，进行编排
2. workers作为一个无状态的池子，根据协调者，获取分配到的执行树（ ready-to-execute local query execution plan (tree) ）。于是从低向上的开始执行，通过Shuffle Persistence Layer存储中间层结果

例子如下

<img src="/assets/images/image-20241009211857825.png" alt="image-20241009211857825" style="zoom:50%;" />

#### 动态查询引擎（*Dynamic Query Execution.*）

根据数据情况选择不同的优化策略

一般情况下，很难获取准确的基数估计指标，这种问题很容易经过join算子后指数级的扩大。基于shuffle persistence layer、cen- tralized query orchestration，Dremel采用了根据运行时采集的指标数据，动态改变查询计划的方式

## 6.嵌套类型数据的列示存储

常见的实现方式有两种：repetition/definition、length/presence 

#### repetition/definition

##### 产品：Dremel、Parquet 

弊端：占用较多的存储空间

优点：查询时可以一次性获取父节点，不需要多次查询

#### length/presence 

##### 产品：ORC、Arrow

2014年，Dremel使用了新的编码方式，Capacitor，更多特性如下

### 6.1 嵌入式执行器(Embedded evaluation)

将filter能力嵌入进Capacitor的数据访问包中，并使用下面的技术提升filter的能力

* 分区与谓词裁剪(*Partition and predicate pruning*)
  * 大量的数据指标被维护，可以直接借助这些指标过滤不满足的分区
* 向量化处理(*Vectorization*)
  可以使用大部分的向量处理技术
* Skip-indexes
  数据写入Capacitor时，将column value组成一个个segment，并在column header中维护一个索引，索引指向对应的segment。通过filter可以快速定位指定的索引，避免其他索引的解压操作
* 谓词重排序(*Predicate reordering*)
  针对谓词重排序的常见的优化算法，依赖谓词的选择性与消耗情况，这是很难预估的。Capacitor借助一系列启发式规则解决这个问题，常见的方式有dictionary usage、unique value cardinality、NULL density、expression complexity

### 6.2 Row reordering

Capacitor使用了多种数据压缩方式，包括dictionary and run-length encodings (RLE)。

为了使RLE具有更好的压缩算法，capacitor可以通过调整记录的顺序，提高压缩效果。例子如下

<img src="/assets/images/image-20241009220321274.png" alt="image-20241009220321274" style="zoom:50%;" />

但是这种重排序属于一种NP问题，大部分情况下效果并不理想。capacitor借助数据采样、启发法实现了一个近似模型。可以将存储成本整体降低17%，有一些数据集可以达到40%，甚至达到70%，如下图

<img src="/assets/images/image-20241009220625059.png" alt="image-20241009220625059" style="zoom:50%;" />

### 6.3 More complex schemas

最初的Dremel版本无法支持无限嵌套的数据结构，Capacitor实现了这个能力

对于复杂的schema，另一种挑战是，没有严格的schema，比如新字段可以随意出现，或者字段类型发生了变化。Protocol Buffers彻底解决了这些问题，但是Capacitor只部分解决了这个问题

## 7.大数据下交互查询的延迟

上文提到存算分离、原位分析、无状态节点都与我们传统认为的优化性能的方式相反，本节主要介绍除了列存方式外，其他降低查询延迟的方法

##### *Stand-by server pool.*

随时处于运行时的服务器资源，减少了机器分配、二进制包复制分配、启动延迟等

##### *Speculative execution*

任务的长尾效应导致查询的延迟变大，Dremel可以通过将查询分解成上千个子任务、高性能节点多执行任务、重复执行慢任务等方式解决这个问题

##### *Multi-level execution trees*

上百个节点执行一个query一般需要一个协调者，但是Dremel借助Google search中树状结构，将执行流从root到leaf再从leaf到root进行执行。在并行执行请求的分发与查询结果的组合上有很好的效果

##### *Column-oriented schema representation*

Dremel存储格式被设计成自描述，如数据分区中存储了嵌入的schema

##### *Balancing CPU and IO with lightweight compression*

压缩操作需要权衡cpu、IO的负载情况，需要均衡两种损耗

##### *Approximate results*

针对一些操作，比如top-k、count-distinct，通过使用近似值可以有效的提高查询性能，Dremel支持设置处理数据百分比的方式实现这个效果，当时设置为98%时，查询延迟可以降低2-3x

##### *Query latency tiers*

面对不同延迟的查询与不同优先级的查询，Dremel通过调度器保证资源的公平调度，抢占式的处理任务，避免任务发生饥饿现象

##### *Reuse of file operations*

大量的文件处理是延迟的一种常见瓶颈，Dremel通过2个主要方式解决这个问题

* 统一在root server获取、复用元数据，并将元数据传递到leaf server
* 尽可能的将一张表写到一个大文件中

##### *Guaranteed capacity*

Dremel专门预留了部分资源给低延迟任务，平时这些资源可以被其他任务使用，但是一旦出现低延迟任务，这些资源就会马上被释放用于处理低延迟任务

##### *Adaptive query scaling*

DAG支持动态调整有利于降低延迟，具体的执行DAG可以根据不同的查询计划进行调整

## 8.结论

重新强调了Dremel使用的技术在过去10年对工业界的影响，很多技术在不同领域中得到了发展

