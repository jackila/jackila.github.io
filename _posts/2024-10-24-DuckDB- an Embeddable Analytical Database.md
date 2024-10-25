---
tags: 论文
---

## ABSTRACT

SQLite的流行说明对于进程内数据管理的需求，但是现在没有一款进程级别的分析型数据库。本文介绍的DuckDB就是一款**嵌入到**另一个进程能够执行**分析型SQL查询作业**的数据管理系统

## 1 INTRODUCTION

#### 需求

embedded into other processes

* 数据库系统作为一个linked library 
* 完全运行在主进程中

<img src="/assets/images/image-20241025085612440.png" alt="image-20241025085612440" style="zoom:50%;" />

#### SQLite的特点

* 专注于 transactional (OLTP) workloads
*  面向行的执行引擎
* 使用B-Tree存储格式
* OLAP的性能比较差

#### 嵌入式OLAP分析引擎

##### Interactive data analysis（交互式数据分析）

* 常见的处理方式是通过R或者python环境下借助一些组件(dplyr [14], Pandas)实现
* 这些组件提供了丰富的数据操作能力，比如WHERE、SELECT、GROUP BY、JOIN、ORDER BY等
* 但是他们没有**全查询优化** 能力与**事务性存储**能力

##### “edge” computing（边缘计算）

比如电力计量设备，会将数据汇总后进行分析，存在下面2个问题

* 带宽问题
* 隐私问题

上述两个场景都需要可移植性、资源利用是设计中的至关重要的两个点

MonetDBLite虽然具有了嵌入式分析系统的能力，但是它又一些难以解决的问题

一个真正的嵌入式分析系统需要具备以下特点

#### 高效的处理OLAP作业，并且不会完全牺牲OLTP性能

同时支持数据更新、数据展示

#### 高度稳定

* 资源耗尽时，自动杀死查询
* 优雅自适应的解决资源冲突问题

#### 高效的读写数据

* 相同进程、地址空间，可以在数据处理时充分利用数据共享

#### 有效的嵌入能力与可移植性

* 不要依赖外部包(openssl)
* 不允许进行信号处理、调用 `exit()` 函数以及修改某些单一进程状态（如区域设置（locale）、工作目录等）的操作

## 2 DESIGN AND IMPLEMENTATION

<img src="/assets/images/image-20241025002133602.png" alt="image-20241025002133602" style="zoom:50%;" />

#### SQL parser 

parse tree of C++ classes

* 尽可能的精简Postgres’ SQL parser
* 先解析成一个parse tree of C structures，后解析成parse tree of C++ classes
  limit the reach of Postgres’ data structures

### logical planner

 fully type-resolved logical query plan

#### binder

将表达式解析成schema 对象(table、view、column name、type)

#### plan generator

* 将parse tree  转换成基本的逻辑查询算子(logical query operators )
  scan, filter, project
* 指标信息会被存储起来得以利用
  * 被优化器利用
  * 防止一些数据转换的溢出

#### optimizer

optimized logical plan 

* 采用动态规划的方式进行**连接顺序优化**，并且在面对复杂的连接图时，会使用一种贪婪算法作为备用方案
* 任意子查询的扁平化
* 利用一些重写规则简化表达式树
  * 公共子表达式消去
  * 常量合并
* 使用样本和HyperLogLog进行基数评估

physical planner

* 将logical plan转换成physical plan
* 选择合适的实现
  * 使用索引
  * 根据join等式选择hash join or merge join 

#### execution engine 

##### 向量化解释执行引擎

* JIT compilation 的问题
  * 依赖大量的编译包(LLVM)
* vector使用固定大小的row(1024)
* 固定长度的类型采用原生数组存储
* 变长类型(string)采用一个指针的原生数组、分离的string heap
* null value通过一个独立的bit 向量存储
* 为了避免向量中的数据频繁移动，采用了一个选择向量协助一些像filter之类的算子
* 包含一个广泛的向量操作库支持关系算子，并且通过借助c++ template扩展代码支持所有的数据类型

##### “Vector Volcano” model

* 从root算子，拉去第一个chunk
* 递归整个计划，直至到一个scan算子
* 直至root数据库受到的chunk为空，query结束

#### 基于MVCC的ACID特性

* implement HyPer’s serializable variant of MVCC 
* updates data in-place immediately
* keeps **previous states** stored in **a separate undo buffer** for concurrent transactions and aborts
* 为了兼顾并发能力，所以没有采用Optimistic Concurrency Control 

####  persistent storage

使用**read-optimized DataBlocks storage layout**

* 逻辑表被按行分成一个个数据库，然后按行进行存储，使用轻量的压缩方法
* 每个区块包含max/min的索引
* 每个区块使用一种轻量级别的索引，提高检索效率

## 3 DEMONSTRATION SCENARIO

#### “teaser” scenario

小数据集，4款数据库都表现较好的性能

##### 大数据集的差异

* SQLite 受限与列处理引擎模型
* MonetDBLite 由于批处理导致产生大量的中间数据

* HyPer 性能很高，但是需要client的数据传输

#### “drilldown” scenario

## 4 CURRENT STATE AND NEXT STEPS

* 完成DataBlocks存储方案和子查询折叠
* buffer manager
* 已经实现查询间并行(inter-query parallelism)，将要实现 查询内的并行度(intra-query parallelism)
* work stealing scheduler
* allow balancing resource usage with the host application
* 支持database APIs of R and Python

##### self checking

edge computing

* 在所有的位置存储checksum
   keep checksums on all persistent and inter- mediate data and piggy-back checksum verification on scan operators
* 定时检测指标
  periodically run sanity check computation to ensure correct operation of CPU and RAM