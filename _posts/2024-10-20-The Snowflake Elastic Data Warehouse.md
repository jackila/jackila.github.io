---
tags: 论文
---

## ABSTRACT

snowflake出现的背景是，当时没有一个整合了云资源、saas服务理念的数仓工具。

##### 传统的数仓工具的弊端

* 基于固定资源、没法利用云资源的扩展能力
* 依赖复杂的ETL、物理调优
* 无法解决半结构化数据、快速响应需求迭代

##### SnowFlake的特点

* 多租户（multi-tenant）
* 事务能力(transactional)
* 安全(secure)
* 高扩展、灵活
* 支持SQL
* 支持半结构、无模式(schema-less)数据

> 用户只需要将数据上传到snowflake就能通过一些通用查询工具查询数据

##### 本文核心

* Snowflake的设计以及多集群、共享数据架构(multi-cluster、shared-data architecture)

* 核心特性
  * 高扩展性、可用性(extreme elasticity and availability)
  * 支持半结构、无模式(schema-less)数据
  * 数据回溯能力(time travel)
  * 端到端数据安全(end-to-end security)
* 开发中的经验与未来的工作方向

# 1. INTRODUCTION

##### 传统数仓的困境

* 无法利用云平台共享基础服务的特性

  * 高扩展性、可用性
  * 按需支付

* 无法解析复杂的数据结构、快速响应多样的数据内容

  > predictable, slow-moving, and easily categorized data from largely internal sources

  * 大数据量
  * schema-less, semi-structured formats

##### 过去的解决方案：大数据平台(Hadoop or Spark)

* 虽然一直在改进，但是仍然缺少传统数仓的效率与大量特性
* 需要大量功能有待实现完成

snowflake基于上述的问题，作为一个机遇云资源的数仓服务而开发。他没有依赖hadoop、postgresql等服务，几乎所有的模块都是从零实现。它的核心特性如下

#### 纯粹的SAAS服务（Pure Software-as-a-Service (SaaS) Experience）

* 使用云上资源，无需自建
* 云上数据或将数据上传到云上后，马上即可以通过web ui界面查询到
* 用户无需关心性能参数、物理存储结构(索引、分区、表空间)、存储管理任务(清理、压缩、归档)

#### 关系数据库(Relational)

支持SQL、ACID事务

#### 支持半结构数据(Semi-Structured)

* 通过内嵌函数、SQL扩展使用户支持遍历、打平、嵌套半结构数据，比如 JSON、avro
* schema的自动发现、列存储使得操作半结构、schema-less数据与普通关系型数据的性能一样的快，而且无需用户调优

#### 弹性服务(Elastic)

存储、计算可以互不影响的扩缩容，并且不会影响数据的可用性与正在查询中SQL的性能

#### 高可用性(Highly Available)

* 支持node、cluster、数据中心的崩溃
* 软硬件的升级都不会有停机影响

#### 持久化数据(Durable)

通过**数据克隆**、**撤销删除**和**跨区域备份**实现极高的耐久性，防止数据丢失

#### 成本高效(Cost-efficient)

* 高效的计算能力、压缩特性
* 按需支付存储、计算成本

#### 安全(Secure)

* 所有的数据(设置临时数据、网络传输)都被端到端的加密
* 没有用户数据被暴露到云平台
* RBAC保证用户在SQL查询中的细粒度访问控制

# 2. STORAGE VERSUS COMPUTE

Shared-nothing architectures 之所以在高性能数仓中占主要地位，是由于它的高扩展性与商用硬件(scalability and commodity hardware)

* 有利于星型查询(starschema queries)
* 由于数据的分区特性，避免数据与资源的竞争，所以不需要昂贵的、定制化硬件
* 由于每个节点的角色是相同的，所以除了设计简单，还带来了一些额外的特性
  * 性能、可扩展性、维护性

但是同时，一个纯粹的Shared-nothing architectures有一个及其严重的弊端：计算、存储资源紧密耦合

在以下场景下会带来问题

* 异构工作负载(Heterogeneous Workload)

  * 同一个系统配置需要同时处理high I/O band-width, light compute 与low I/O bandwidth, heavy compute 两种完全迥异的任务
  * 导致较低的资源利用率

* 节点的变化(Membership Changes)

  * 节点崩溃、扩缩容都会导致这种场景
  * 大量数据需要reshuffle
  * 而这些节点即需要处理reshuffle、又需要处理查询负载，可以观察到明显的性能影响，限制了扩展能力与可用性

  > 可以通过replicate做到在性能上实现一定的缓解

* 在线更新(Online Upgrade)

  * 原则上，一个节点一个节点的升级是可以实现无感升级，但事实上在所有事物都耦合并且要求同质的前提下，很难实现

上面的场景在内部环境是可以容忍的，毕竟节点的变动、升级迭代是可控的。

##### 云资源是不可接受的

* 不同的节点类型
* 节点容易崩溃、性能也不同
* Membership Changes是一个常见的事情
* 必须支持的版本迭代、扩缩容操作
  * 快速的开发、迭代更新
  * 按需使用资源

基于上述问题，snowflake实现了解偶了计算与存储资源(**multi-cluster, shared-data architecture**)

* 计算又snowflake专有的无共享(share-nothing)引擎支持
* 存储由s3支持
* 本地磁盘只会存储warm data

# 3. ARCHITECTURE

<img src="/assets/images/image-20241020192858324.png" alt="image-20241020192858324" style="zoom:50%;" />

* Data Storage

  存储数据，一般是S3

* Virtual Warehouses
  处理查询作业

* Cloud Services
  管理VM、查询、并发、元数据等

## 3.1 Data Storage

#### 影响snowflake设计的S3特性(对比本地磁盘)

* 高延迟、更高的cpu开销
  需要处理网络协议、(反)序列化、连接管理、远程连接
* 简单的API，只支持PUT/GET/DELETE
* 只能通过file写入(覆盖)，不支持修改、append
* 需要提前将upload-size写入请求中
* GET 请求支持获取文件的部分内容

#### Snowflake基于上述特性的设计

* 影响了table file format与并发控制的设计
* 将一张表水平切分成一个个的较大、不可变的文件
* 数据被拆分成一列一列，然后进行存储、压缩(PAX or other)
* 每个文件都一个元数据头
* 请求时只需要请求文件元数据头与部分相关列

#### Snowflake不仅仅存储数据，还存储以下类型数据

* 临时文件
  有利于大查询场景
* 查询结果
  避免传统数据库服务端的cursors，提供更简单的数据交互

元数据存储方式

* 组成内容：catalog objects、statistics、locks、 transaction logs
* Cloud Services 层中的一个可扩展的、事务型KV存储引擎

## 3.2 Virtual Warehouses

Snowflak为每个用户分配一个VM，所有的VM组成了这里的Virtual Warehouses layer，每个VM由一个个的EC2节点组成。每个VM内EC2的数量不同

### *3.2.1 伸缩与隔离能力(Elasticity and Isolation)*

* VM对于用户是一个资源单元，可以创建、删除、调整大小
* 一个查询任务只会在一个VM中运行
* 由于文件的不可变性，一个work process不会出现外部可见的影响
  系统在整理process完成之前，不会使中间数据对用户可见，所以用户是可以对修改无感的
* 通过隔离的VM，可以实现用户之间的任务互相不影响
* 失败任务支持在同一个VM中重试，不支持节点级别的重试
* 伸缩能力甚至可以在相同价格下实现更好的性能
  4个节点跑15个小时 vs 32个节点跑2小时

### *3.2.2 本地缓存能力与文件分摊(Local Caching and File Stealing)*

* 每个工作节点(work node)在本地磁盘维护一张表的缓存
  文件头信息+上次查询的相关列数据
* 可以被并发线程、后续的查询(进程)共享
* 通过LRU进行管理
* 基于表名采用一致性hash算法
  * 提供cache命中、避免重复cache同一个表文件
  * 同一个表文件会缓存在同一个工作节点上
* 一致性hash算法采用lazy策略
  * 节点崩溃，不会马上做cache迁移
* 解决数据倾斜(skew)
  * 在scan层处理数据倾斜
  * 先下载完分配的file的work node，会从同事中请求file，然后接管这些文件的下载
  * 高性能节点会直接从s3下载数据，而不是从低性能节点获取

### *3.2.3 执行引擎(Execution Engine)*

自研的高性能SQL执行引擎特点

#### Columnar

对比row-wise

* more effective use of CPU caches and SIMD instructions
*  more opportunities for (light- weight) compression 

#### Vectorized

对比MapReduce

* 避免中间结果的物化
* pipeline 模式(pipelined fashion)
* 批处理上千行记录的行格式
* 降低IO、提高了cache效率

#### Push-based

对比Volcano-style 的pull模式

* 向下游算子推送结果
* improves cache efficiency
* efficiently process DAG-shaped plans

#### 其他特点

* 执行过程中不需要事务管理
* 不需要buffer pool
* 支持所有类型算子(join, group by, sort) spill 到磁盘上
  * 虽然内存型引擎更精简、快速，但是分析作业需要处理更大的数据量

## 3.3 Cloud Services

对比传统数据库每个用户一个管理系统，Snowflake采用了多租户的方式，并且保证每个子服务(access control, query optimizer, transaction manager, and others)都是高可用、可扩展的

#### *3.3.1 查询管理器与优化器(Query Management and Optimization)*

* 所有查询语句都会经过，并且执行查询早期的所有环节：parsing, object resolution, access control, and plan optimization
* typical Cascades- style approach：top-down cost-based optimization
* 维护所有的指标数据
* 由于没有索引，所以查询计划的范围比较小
* 很多决定被推迟到执行时处理，进一步减少需要检索的查询计划
  * 提高了程序的健壮性
  * 损失了一些极致性能
  * 更易用
* 分发查询计划到各个work节点，并将执行状态反馈给用户，实时追踪

#### *3.3.2 并发控制(Concurrency Control)*

* 任务特点：large reads, bulk or trickle inserts, and bulk updates
* 使用Snapshot Isolation (SI)实现ACID transactions
  * 只能看见事务开始前的所有版本数据
  * 基于MVCC实现与S3的文件不可变性实现
  * 将文件的变更维护在metadata中

#### *3.3.3 数据剪切(Pruning)*

传统数据库通过索引、B+tree的方式对查询进行数据剪切，但是这种方式对于snowflake这种系统不合适

所以snowflak采用：min-max based pruning、small materialized aggregates、zone maps 、data skipping等方式

简单说，就是维护每个文件中的数据信息，支持快速的过滤能力

除了这种static pruning，也支持运行时的dynamic pruning，比如join时

# 4. FEATURE HIGHLIGHTS

snowflake实现了对于关系数据库很普通的特性：完整的sql能力、acid事务、标准的接口、稳定安全、客户支持、高性能与可扩展性。除了上述的这些特性，本节介绍了一些snowflake独特的特性

### 4.1 纯粹的SAAS服务(Pure Software-as-a-Service Experience)

* 强大的web ui能力
* 用户只需要关心数据、查询，不需要关心： failure modes、tuning knobs、physical design、storage grooming tasks

### 4.2 高可用性(Continuous Availability)

传统的数仓与业务系统的耦合度比较低，所以对可用性要求不高。但是作为一个saas系统，snowflake需要满足相同的可用性。本节主要从下面2个特征说明这一问题

#### *4.2.1 自恢复能力(Fault Resilience)*

<img src="/assets/images/image-20241021211233512.png" alt="image-20241021211233512" style="zoom:50%;" />

* S3、Snowflake’s metadata store 都是在多个可用区备份，借助load balancer可以直接可用区级别的容灾能力
* VM在可用性与性能方面，选择单AZ运行，所以如果出现问题需要马上启动另一个可用区。但是这种场景及其的少

#### *4.2.2 无感知升级迭代(Online Upgrade)*

<img src="/assets/images/image-20241021211822335.png" alt="image-20241021211822335" style="zoom:50%;" />

* 在Cloud Services组件与VM组件，都实现了多版本部署
* 大部分服务都是无状态的，metadata store处理元数据版本与schema迭代，并保证向后兼容
* 多版本Cloud Services共享metadata store，同时不同版本的work node也可以共享缓存
* 这种升级迭代的便利，增加了版本的迭代与bug的修复频率，甚至能快速的版本降级

### 4.3 半结构、无结构数据的支持(Semi-Structured and Schema-Less Data)

* snowflake使用VARIANT类型管理Semi-Structured and Schema-Less Data
  * ARRAY and OBJECT 是variant的特殊结构
* VARIANT 使得snowflake从ETL类型转换成了一种ELT模式
  * 简单说就是先将数据加载进去，通过variant表示复杂结构，然后通过数据库自带的命令、甚至UDF解析数据

#### *4.3.1 Post-relational Operations*

##### 提取能力(extraction)

通过SQL函数、类js编程语言将VARIANT类型提取、转换成标准的SQL类型

##### 扁平(flattening)

将一个嵌套结构拍平称多行记录，或者将多行记录通过新的聚合、分析函数(ARRAY_AGG \ OBJECT_AGG)整合成一行

#### *4.3.2 Columnar Storage and Processing*

很明显半结构数据使用列式存储要比行存性能更好

Impala与Dremel处理Semi-Structured需要用户提前定义好表结构，snowflake为了提高灵活性与性能，采用了自动推断字段类型与列存的方式

snowflake采用了混合存储类型的方式，通过自动检测，对于常见的字段，会被单独提取出来，使用列存，甚至会统计它的指标，用于后续的截切

在扫描过程中，多个列也可以被整合成一个VARIANT结构，将查询不想关的字段过滤

上述的选择性的处理列信息，可能会导致某些查询的性能影响

* 比如某些过滤条件不足以让系统在metadata中产生相关的指标

snowflake通过对文档中paths构造bloom filter解决这个问题

#### *4.3.3 乐观转换（Optimistic Conversion）*

date/time values 常被表示为字符串类型，对于查询的剪切与转换都是一个麻烦

如果在写入时直接转换，有可能会损失原始数据中的信息

snowflake采用乐观数据转换的方式解决这个问题，并保存2钟格式的数据

#### *4.3.4 Performance*

整合columnar storage、optimistic conversion、pruning三个效果，验证semi-structured data的查询性能

<img src="/assets/images/image-20241021220309915.png" alt="image-20241021220309915" style="zoom:50%;" />

* 大部分的查询性能会有10%的性能损耗，除了Q9 and Q17 over SF1000

对于相对稳定、简单的半结构数据处理引擎几乎与传统的关系数据库有相同的性能

### 4.4 回溯与clone能力(Time Travel and Cloning)

snowflake会保存过去90天内过期的数据，以至于可以借助AT、BEFORE实现数据回溯的能力

```sql
SELECT * FROM my_table AT(TIMESTAMP =>
  ’Mon, 01 May 2015 16:20:00 -0700’::timestamp);
SELECT * FROM my_table AT(OFFSET => -60*5); -- 5 min ago
SELECT * FROM my_table BEFORE(STATEMENT =>
  ’8e5d0ca9-005e-44e6-b858-a8f5b37c5726’);
```

并且可以在同一个查询中访问不同版本的数据

```sql
SELECT new.key, new.value, old.value FROM my_table new
JOIN my_table AT(OFFSET => -86400) old -- 1 day ago
ON new.key = old.key WHERE new.value <> old.value;

```

甚至可以快速的恢复删除的表

```sql
DROP DATABASE important_db; -- whoops!
UNDROP DATABASE important_db;
```

设置在不用创建任务数据文件的前提下，即可快速的clone一个表的数据，只需要在元数据中复制部分信息即可

```sql
CREATE DATABASE recovered_db CLONE important_db BEFORE(
  STATEMENT => ’8e5d0ca9-005e-44e6-b858-a8f5b37c5726’);
```

### 4.5 Security

snowflake为实现完全的端到端的加密与安全性，实现了以下能力

* two-factor authentication(2FA)

* (client-side) encrypted data import and export
* secure data transfer and storage
* role-based access control (RBAC [26]) for database objects

#### *4.5.1 Key Hierarchy*

使用强大的 **AES 256位加密**、采用 **AWS CloudHSM** 进行密钥管理、自动轮换和重新加密密钥，以及遵循 **NIST 800-57** 标准的密钥管理生命周期。并且用户完全无感知

<img src="/assets/images/image-20241021233112880.png" alt="image-20241021233112880" style="zoom:50%;" />

hierarchical key model 保护了多租户下的数据安全

#### *4.5.2 Key Life Cycle*

snowflake不仅在数据数量、范围级别做了限制，而且在key的有效期上做了约束。加密key有以下四个阶段

* **Pre-operational creation phase**（预操作创建阶段）

* **Operational phase**（操作阶段）
  * 加密、解密
* **Post-operational phase**（后操作阶段）
  * 存档或记录、进行审计时使用
* **Destroyed phase**（销毁阶段）
  * 密钥会被永久删除，以防止任何潜在的安全风险

密钥轮换和重新加密，Snowflake 有效地控制了密钥的使用时长

#### *4.5.3 End-to-End Security*

除了数据加密，还有以下措施

* S3的访问策略、存储隔离
* RBAC级别的查询能力
* 加密数据的上传下载
* 2FA的访问控制

# 5.RELATED WORK

### Cloud-based Parallel Database Systems

Redshift 

* 采用了传统的shared-nothing architecture，所以新增、减少计算资源会导致数据重分配
* 今观redshift可以处理json格式，但是Snowflake借助colunmn storage，性能更强

BigQuery

* a SQL-like language
* append-only and re- quire schemas.
*  Snowflake offers full DML ，ACID transactions， no require schema definitions 

Microsoft SQL Data Warehouse (Azure SQL DW)

* 存算分离
* 并发度有上限，32个
* Snowflake no need choosing appropriate distribution keys and other administrative tasks
* 不支持semi-structured data

### Document Stores and Big Data

Document stores(MongoDB、Couchbase Server、Apache Cassandra )

* difficulty to express more complex queries

为了解决这个问题出现了一些组件协助完成这些任务

“Big Data” ： Apache Hive [9], Apache Spark [11], Apache Drill [7], Cloudera Impala [21], and Facebook Presto

支持查询嵌套结构

但是snowflake通过schema inference, optimistic conversions, and columnar storage，将上面不同系统的灵活性与关系型列式数据库的存储效率和执行速度结合起来

# 6. LESSONS LEARNED AND OUTLOOK

2012年时，大部分的人的注意力都在SQL on Hadoop，很少人关注如何开发一款适合云服务的数据管理系统。

hadoop并没有替代mysql，只是作为一个补充

Snowflake不仅代替了事务数据库而且是hadoop集群

在开发过程中，走了很多弯路

* 一些关系型操作符实现过于简单
* 没有及时支持所有数据类型
* 对资源管理的关注不足
* 在全面的日期和时间功能方面的工作被推迟

通过避免复杂的调优设置，Snowflake 最终实现了用户友好的性能选择

接下来的挑战

* Building a metadata layer that can support hundreds of users concur- rently

* Handling various types of node failures, network failures, and support- ing services 
*  Security has been and will continue to be a big topic

The biggest future challenge for Snowflake is the transi- tion to a full self-service model

