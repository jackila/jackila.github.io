---
tags: 论文
---
>  How Good Are Query Optimizers, Really?

## ABSTRACT

本文通过Join Order Benchmark (JOB)验证查询优化器的核心组件对查询性能的影响。主要如下内容

* 工业级cardinality estimators 误差比较高
* query performance 太过依赖数据估算时，性能是无法让人满意的
* cost model 对查询性能的影响要比 cardinality estimates 小
* exhaustive dynamic programming 比heuristic algorithms效果更好，即便是较差的 cardinality estimates 

## 1. INTRODUCTION

<img src="/assets/images/image-20241001142114509.png" alt="image-20241001142114509" style="zoom:50%;" />

当前的问题

* the **cardinality estimations** and the **cost model** are not **accurate**
  * **cardinality estimates** are usually computed based on **simplifying assumptions** like uniformity and independence
  * **assumptions are *frequently* wrong**

为了解决什么问题

* cardinality estimators的实际效果如何，差的估算是否会导致慢查询
* cost model对查询优化进程的重要度如何
*  enumerated plan space需要多大？

采用的策略

* 提出一个新颖的方法，分别测试上面三个不同的组件对性能的影响
* using a real- world data set and 113 multi-join queries
* 所有的数据都可以加载到内存中

最终的成果

* 设计了*Join Order Benchmark (*JOB*)* 
* end-to-end study of the **join ordering problem** using **a real- world data set** and **realistic queries**
* **provide guidelines for the complete design** of a query optimizer.

## 2. BACKGROUND AND METHODOLOGY

The goal of this paper is to investigate **the contribution of all relevant query optimizer components** to **end-to-end** query performance in a **realistic setting**

 In this section we **introduce the Join Order Benchmark**, **describe all relevant aspects** of PostgreSQL, and **present our methodology**

### 2.1 The IMDB Data Set

常规的数据集(TPC-H\TPC-DS\SSB)由于其为人为构造数据集，所以大都具有相同的特性(uniformity, independence, principle of inclusion),与真实数据集有较大的区别，所以本文使用IMDB数据集

### 2.2 The JOB Queries

* designed the queries to have between **3 and 16 joins**, with an **average of 8 joins** per query
* Each query **consists of** one **select-project-join block**
* Our query set consists of **33 query structures**, each with **2-6 variants** that differ in their selections only, resulting in a **total of 113 queries**
  * the variants of the same query structure have **different optimal query plans**
  * some queries have more **complex selection predicates** than the example
* Our queries are “realistic” and “ad hoc” in the sense
  * cardinality estimators the queries are challenging
    * significant number of joins  and the correlations
  * no “trick” the query optimizer
    * picking attributes with extreme correlations

### 2.3 PostgreSQL

相关特性

* Join orders using dynamic programming
* The cardinalities of base tables are estimated using **histograms (quantile statistics)**, **most common values with their frequencies**, and **domain cardinalities** (distinct value counts)
* These **per-attribute statistics** are computed by the **analyze command** using **a sample of** the relation
* 对于复杂谓词，无法应用直方图时，系统采用不具理论基础的特定方法（“魔法常数”）
* 为了应用组合同一表的连接谓词，PostgreSQL 简单地假设独立性，并将各个选择性估计的选择性相乘
* 计算中间结果的公式
  <img src="/assets/images/image-20241001151530135.png" alt="image-20241001151530135" style="zoom:50%;" />
* PostgreSQL’s **cardinality estimator** is based on the **following assumptions**
  * 一致性：所有值，除了最频繁的值，假定具有相同数量的元组
  * 独立性：属性（在同一表中或来自连接表）的谓词是独立的
  * 包含原则：连接键的域重叠，使得较小域中的键在较大域中有匹配
* 最重要的访问路径是**全表扫描**和在**非聚集 B+ 树**索引中的查找
* 连接可以使用nested loops (with or without index lookups)、in-memory hash joins、sort- merge joins 
* cannot be changed at runtime

### 2.4 Cardinality Extraction and Injection

* 在5个不同的DBMS中分别获取对应的**cardinality estimates**，获得对应的最佳查询计划，并在 PostgreSQL 中运行这些计划

* 修改了 PostgreSQL，以启用任意连接表达式的基数注入，允许 PostgreSQL 的优化器使用其他系统的估计（或真实基数）

## 3. CARDINALITY ESTIMATION

In this section, we experimentally investigate **the quality of cardinality estimates** in relational database systems by **comparing the estimates with the true cardinalities**

### 3.1 Estimates for Base Tables

*q-error*：an estimate differs from the true cardinality，using the ratio

<img src="/assets/images/image-20241001161301294.png" alt="image-20241001161301294" style="zoom:50%;" />

* q-errors for the 629 base table selections
  * 1表示没有差距
* 大多数选择的估计是正确的
* DBMS A 和 HyPer 通常能够很好地预测复杂谓词，如like
* 当选择在样本中产生零个元组时，系统会退回到一种临时估计方法（“魔法常数”）
* 其他系统的估计更差，似乎基于每个属性的直方图，这对于许多谓词效果不佳，并且无法检测属性之间的（反）相关性

### 3.2 Estimates for Joins



<img src="/assets/images/image-20241001162459812.png" alt="image-20241001162459812" style="zoom:50%;" />

* 大部分DBMS的连接估计误差的总体方差相似
* 所有系统，我们经常观察到1000倍的误估计范围
* 随着连接数量的增加，误差增加
* 所有系统，对于中间结果都是预估较小，而且中间结果随着连接数量的增加而减少，但是小于DBMS的预估范围
* DBMS A之所以误差较小，可能因为引入一个与连接大小相关的阻尼因子
  * 连接表数量越多，独立性越低
* PostgreSQL 的估计器是基于独立性假设的

虽然每个系统的预估能力不同，但是并不说明查询性能差，因为每个系统都会对这些数据进行一定的自适应调整

### 3.3 Estimates for TPC-H

<img src="/assets/images/image-20241001163700291.png" alt="image-20241001163700291" style="zoom:50%;" />

*  TPC-H query workload  基本上没有什么挑战性
* 自定义query，影响比较大，是一个对于cardinality estimation有挑战的benchmark

### 3.4 Better Statistics for PostgreSQL

**错误估计的不同值总数**是否是基数估计的根本问题？

<img src="/assets/images/image-20241001164455191.png" alt="image-20241001164455191" style="zoom:50%;" />

* 轻微的改进了异常的方差
* 但是低估的趋势被扩大
  * 由于错误的估算会导致中间结果变大，所以导致最终估算结果反而会变得更接近真实

## 4.WHEN DO BAD CARDINALITY ESTIMATES LEAD TO SLOW QUERIES?

上一节中说明各种缓解的估算都会产生很大的误差，但是这些误差确不一定会导致性能差异，比如

* 错误估计的表达式可能与查询的其他部分相比成本较低
* 相关的计划替代方案可能也被类似的因素错误估计，从而“抵消”了原始错误

本节将探索，究竟哪些情况下会使错误估算差生一个慢查询

本节分别从仅使用主键索引，多个索引两种情况进行验证

### 4.1 The Risk of Relying on Estimates

将不同系统的估计值注入到PostgreSQL，执行生成的计划

<img src="/assets/images/image-20241001170433815.png" alt="image-20241001170433815" style="zoom:50%;" />

* 当使用估计值时，绝大多数查询的速度更慢
  * DBMS A：**78%** of the queries are **less than 2×** **slower** than using the true cardinalities
  * DBMS B：**only 53% of** the queries
  * 所有的估计器都存在超时问题

#### 如何避免这些性能问题

* disabled nested-loop joins
  * 错误的将基数评估过低，导致优化器使用了 nested loop join
* 运行时调整哈希表的大小
  * 避免冲突链过长

#### 结论

* 单纯基于成本的做法，即不考虑**基数估计的不确定性**和**不同算法选择的渐近复杂性**，可能导致非常糟糕的查询计划

* 很少比更稳健的算法提供显著好处的算法**不应被选择**
* 查询处理算法应尽可能在**运行时**自动确定**其参数**，而不是依赖基数估计

### 4.2 Good Plans Despite Bad Cardinalities

数据库只有主键索引时，禁用了嵌套循环连接并启用了重新哈希，大多数查询的性能接近使用真实基数获得的性能

* 没有外键索引，大多数大型（“事实”）表需要通过全表扫描来进行扫描
* 基数估计通常足够排除所有灾难性的连接顺序决策
* 在主内存中，选择 index nested-loop join 而不是hash join 不会影响性能太大，他们呢之间的差距在2到5倍
* 主内存中，不会出现随机IO导致的性能问题

### 4.3 Complex Access Paths

增加所有外键索引，存在大量性能差异，40% 的查询速度慢了 2 倍

<img src="/assets/images/image-20241001184416890.png" alt="image-20241001184416890" style="zoom:50%;" />

* 整体性能通常会显著提高，但可用的索引越多，查询优化器的工作就越困难

### 4.4 Join-Crossing Correlations

* 单表子查询基数估计质量的实验中，表明保持表样本的系统能够实现几乎完美的估计结果，即使存在关联谓词（在同一表内）

* 关联谓词涉及来自不同表的列，并通过连接连接的查询，评估中间结果就变得非常有挑战性

解决方法：在连接交叉谓词上增加一个分区索引

查询优化的效果总是受到可用访问路径选项的限制，描述（连接交叉）相关性对当前查询处理具有一定的影响

## 5. COST MODELS

传统磁盘系统领域，成本模型考虑了各种微妙的因素

* partially correlated index accesses
*  interesting orders
* tuple sizes

下文中通过对比三个成本模型，评估成本模型对查询性能的影响

1. 面向磁盘的复杂成本模型，即 PostgreSQL 的模型
2. 主内存设置的 PostgreSQL 模型的调优版本，数据都适合 RAM
3. 仅考虑查询评估过程中产生的元组数量，及其简单的成本模型

### 5.1 The PostgreSQL Cost Model

 CPU 和 I/O 成本与某些权重结合在一起

##### 操作符的成本

* 磁盘页面数量（包括顺序和随机）
* 内存中处理的数据量
* 加权总和

权重参数需要反应如下几个点的相对差异

* 随机访问
* 顺序访问
* CPU 成本

调查成本模型对整体查询引擎性能的影响是重要的，而且是非常困难的

### 5.2 Cost and Runtime

<img src="/assets/images/image-20241001190555747.png" alt="image-20241001190555747" style="zoom:50%;" />

* 较差的**基数估计**导致图 8a 中出现**大量异常值**和**非常宽的标准误差区域**
* Using the d**efault cost model of PostgreSQL** and the **true cardinalities**, the median error of the cost model **is 38%**

### 5.3 Tuning the Cost Model for Main Memory

* 将数据加载到可用内存中

* 减少这两组（CPU&&IO）之间的比例，将CPU 成本参数乘以 50 的因子来实现这一点
* 从图B与图D调优改善了成本与运行时间之间的相关性
* 从比较图 8c 和 d，参数调优的改善仍然被估计值和真实基数之间的差异所掩盖

我们观察到调优提高了成本模型的预测能力：**中位数误差从 38%降低到30%**

### 5.4 Are Complex Cost Models Necessary?

不建模I/O 成本，只是计算在查询执行过程中通过每个操作符的元组数量

<img src="/assets/images/image-20241001191412810.png" alt="image-20241001191412810" style="zoom:50%;" />

* 从 8e 和 f 所示，即使是我们简单的成本模型也能够相当**准确地预测使用真实基数的查询运行时间**
* our **tuned cost model** yields 41% faster runtimes than the s**tandard PostgreSQL model**, but even **a simple Cmm** makes queries **34% faster**

##### 基数估计比成本模型更为关键

## 6. PLAN SPACE

在本节中，我们研究了搜索空间需要多大才能找到一个好的计划

使用一个独立的查询优化器进行验证

* 动态规划（DP）
* 多种启发式连接枚举算法

* 允许注入任意的基数估计
* 首先使用**估计值**作为**输入**运行**查询优化器**。然后，我们使用**真实的基数**重新计算生成**计划的成本**
  * 在不运行query的前提下获取执行的成本

### 6.1 How Important Is the Join Order?

使用 Quickpick [40]算法来可视化不同连接顺序的成本

* 随机选择连接边，直到所有连接关系完全连接

* 查询运行算法 10,000 次并计算结果计划的成本

<img src="/assets/images/image-20241001192926258.png" alt="image-20241001192926258" style="zoom:50%;" />

* 25c有许多好的计划，16d很少
* 分布有时很宽（例如，16d），有时又很窄（例如，25c）
* “无索引”和“主键索引”配置的图形非常相似
* 1.5倍性能内，无索引为44%，主键索引为39%，外键索引只有4%
* 最差计划与最佳计划之间的平均比例，即分布的宽度
  * 101× without indexes, 115× with primary key indexes, and 48120× with foreign key indexes

**三种索引配置的搜索空间截然不同**

### 6.2 Are Bushy Trees Necessary?

大多数优化器的做法

* 不枚举所有可能的树形结构
* 忽略了具有交叉乘积的连接顺序
* not considering bushy join trees 【oracle】

下文中，为了量化限制搜索空间对查询性能的影响，仅枚举左深、右深或之字形树

<img src="/assets/images/image-20241001193708575.png" alt="image-20241001193708575" style="zoom:50%;" />

* zig-zag trees 在大多数情况下性能较好，最坏也只有2.54X
* 左深度表现比之字形树更差，但仍然能达到合理的性能
* 右深树的表现远不如其他树形，因此不应单独使用

### 6.3 Are Heuristics Good Enough?

动态规划与随机算法和贪婪启发式算法进行比较

* “Quickpick-1000”启发式算法
  * chooses the cheapest 1000 random plans
* Greedy Operator Ordering (GOO)

![image-20241001194529530](/assets/images/image-20241001194529530.png)

* 动态规划全面检查搜索空间仍然是值得的
* However, the errors introduced by estimation errors **cause larger performance losses than the heuristics**

虽然穷尽式枚举所有"bushy trees"带来的性能提升是中等的，但与部分枚举算法相比，它仍然有优势。而且，**准确的基数估算**对查询性能的影响更大。此外，由于已有快速高效的穷尽式算法，通常**不需要依赖启发式方法**或禁用复杂的"bushy tree"结构，即可以直接使用穷尽式算法来生成最优查询计划

## 7. RELATED WORK

* keep **table samples** for cardinality estimation **predict single-table result sizes** considerably **better than** those which apply the **independence assumption** and **use single-column histograms**