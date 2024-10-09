---
tags: 论文
---
> [MonetDB/X100: Hyper-Pipelining Query Execution](MonetDB/X100: Hyper-Pipelining Query Execution) 论文阅读

## 0. 概述

文章发表于2005年，当时大部分的数据使用较低的IPC效率，文章给予TPC-H探讨了这种情况，并提出一些新的设计查询处理器的方案

文章介绍了基于上述优化方案实现的X100查询引擎，虽然它看起来像Volcano引擎，但是向量化处理数据的方式使其比其他DBMS在100GB规模下，性能高出一两个数量级

## 1. 简介

* 大部分的数据分析工作，由于其independent calculations特性，是有利于高效的IPC
* 现在的DBMS采用的架构阻碍了编译器的优化（super scalar ）
  * Volcano iterator model的逐个处理元组的方式
* MonetDB/MIL 通过使用column-at-a-time,解决了 tuple-at-a-time的问题
  * full column materialization 产生了大量的中间数据，导致内存宽带成为瓶颈，进而降低了cpu
* X100查询引擎，结合了column-wise execution 、 incremental materialization offered by Volcano-style pipelining
* X100采用了矢量化查询引擎
  * 高效的cpu
  * 支持disk-base存储

### 1.1 大纲

* 现有系统的性能问题
* 借助TPC-H query 1 分别分析以下组件的性能：关系数据库、MonetDB、hand-code implement
* 介绍了 MonetDB X100的架构、实现
* 比较MIL&X100的性能
* 相关工作与总结

## 2. CPU如何高效工作

* 摩尔定律下不断升级的CPU能力
* CPU pipelines 能力支持一个周期多个指令同时执行
  * 指令的依赖
  * if -a-then-b-else-c 导致指令的误执行、恢复的损耗
* super-scalar能力，提升数据处理能力

* 特殊的优化策略
  * Very Large Instruction Word
  * 编译器优化能力
  * 分支预测能力
  * 芯片缓存的作用
  * 适合内存随机访问的数据结构

大部分DBMS的IPC在0.7左右，scientific computation (e.g. matrix multiplication) or multimedia processing 的IPC在2左右。



## 3.TPC-H Q1的微基准测试

![image-20240725212130377](/assets/images/image-20240725212130377.png)

上图显示：hand-code > X100 > MIL > Mysql > DBMS X

### 3.1. Q1 on relation data base

##### 特点

* Volcano模型
* 较高自由度的参数设置
* 可以处理任意复杂的表达式
* 数据粒度为一个tuple

##### 代价

* 核心操作执行时间只占10%
* 28% 用于 hash表的查找、创建
* 62% 用于字段的遍历
* Item_func_plus::val 需要38个指令，38/0.8 = 49个周期
  * MIPS=3 cycles

##### 解析

* 没有pipelined loop优化特性，必须等待指令执行结果（5 cycle）

* 函数调用成本(20个周期)，无法被分摊，导致整体成本增加

### 3.2 Query 1 on MonetDB/MIL 

##### 特点

* column-wise
* binary association table（BAT）= [oid,value]
* a fixed number of parameters of a fixed format(all two-column tables or constants)
* 高效的利用率：span more than 99% of elapsed query time
* 内存敏感型设计

##### 代价

* 大量的中间数据，依赖于cpu的吞吐性能，较高的宽带消耗
* volcano pipeline 会在单次传递中完成选择、计算、聚合，而monetDB是分三个阶段处理，每个阶段都产生中间数据
* 在disk-base的磁盘上实现这个模型，基本不可能（除非RAID系统）

### 3.3 Query 1: Baseline Performance 

##### 特点

* 只处理相关的column
* voids、form of a array、restrict pointer ===> loop pipelining
* performed some common subex- pression elimination 

## 4 X100: A Vectorized Query Processor

#### 目标

* 高吞吐&高CPU效率
* 可以扩展到其他应用领域，数据挖掘、多媒体检索

* 可以随disk的扩展而扩展

#### 架构

##### 存储(Disk)

* 列存
* 数据压缩

##### RAM

* 专门的进程执行：memory-to-cache、cache-to-memory 
* 其他优化：SSE prefetching 、data movement assembly instructions
* same vertically partitioned and even compressed disk data layout to save space and bandwidth

##### Cache

* Volvano-like execution pipeline 
* 将数据分割成小批次(1000),批量执行

##### CPU

* 向量化原语通过暴露数据处理的独立性给编译器，是编译器实现更好的优化
* 不止支持Projection而且支持其他算子(aggregation)
* X100支持针对整个表达式子树的向量化原语，而不是单个函数

![image-20240725224552052](/assets/images/image-20240725224552052.png)

### 4.1 Query Language 

* 标准的关系代数作为查询语言
* 在cpu cache中完成向量的转换

### 4.2 Vectorized Primitives 

* column-wise vector layout 的目的不是memory layout in cache 而是 a low degree of freedom
  * 只需要关心自己的columns信息
  * `vectorized primitives operate on restricted (independent) arrays of fixed shape`,编译器apply aggressive loop pipelining

* X100 contains hundreds of vectorized primitives
  * generated from primitive patterns
  * primitive generation is a file with map signature requests
* support compound primitive signatures
  * 更高的性能
*  primitive generator 不仅仅是一个macro expansion script，还可以通过一个优化器动态产生
* provide (source-)code patterns instead of compiled code, allows all ADTs

### 4.3 Data Storage

* 所有表采用列存方式
  * MonetDB 存储在单个连续的文件中
  * ColumnBM 分散在多个文件
* 存储不可变，更新转增量结构，类似LSM
* 支持简单汇总(summary)索引

## 5 TPC-H Experiments

* MonetDB/MIL <  MonetDB/X100

#### 5.1 Query 1 performance

![image-20240725234415255](/assets/images/image-20240725234415255.png)

* 非常低的CPU周期，执行所有基元，包括复杂的基元

  * 聚合：6 cycle/tuple
  * 乘法：2.2 cycle / tuple

  > Mysql need 49 cycle / tuple

* 非常高的宽带
  * MonetDB/MIL : MonetDB/X100 = 500MB/s : 7.5GB/s
* 高效的fetch-join：2 cycle/tuple

#### 5.1.1 Vector Size Impact

![image-20240725234928443](/assets/images/image-20240725234928443.png)

* 太小，无法利用cpu的并行
* 太大，中间结果不适合缓存，超过L1和L2缓存
  * 中间结果被存储在主内存，类似MonetDB/MIL

## 6 Related Work

* 探索Volcano 的并行查询处理
  * 为每个查询操作符分配一个单独进程
* DB2 数据块执行方式，优化聚合与投影

## 7 Conclusion and Future Work

* 建议在火山和 MonetDB 执行模型之间取得平衡
* 添加更多矢量化查询处理运算符
* 将 X100 用作低功耗（嵌入式，移动）环境的高效查询处理系统