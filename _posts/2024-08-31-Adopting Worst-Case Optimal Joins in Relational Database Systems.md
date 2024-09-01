>  **Adopting Worst-Case Optimal Joins in Relational Database Systems**

## ABSTRACT

某些查询场景，**Worst-Case Optimal Joins** 比常规的binary joins 执行时间更少、中间结果更少。

当前的几种常见的实现存在几个问题

* focus on a specific problem domain
* extensive precomputation

本文提出的算法，具有通用性，即适合OLTP又适合OLAP

算法的核心有以下两点

* hash-based worst-case optimal join
*  a hybrid query optimizer for binary and multi-way joins within the same query plan

## 1. INTRODUCTION

现有的worst-case op- timal joins存在几个问题

*  require suitable indexes on all permutations of attributes 
  * storage and maintenance overhead
* EmptyHeaded or LevelHeaded rely on specialized read-only indexest
  * expensive precomputation
* LogicBlox 支持mutable data，but orders of magnitude slower 
* 如果没有不断增长的中间结果，multi-way joins比binary joins 慢非常多

基于上面的问题，新算法的两个核心点是

* 能够根据情况选择multi-way joins还是binary join
  * cost-based query optimizers
* 不需要持久化，运行时可构建的 高性能的 indexes structures 
  * hash- based instead of comparison-based

### 2. BACKGROUND

#### Worst-Case Optimal Join Algorithms 伪代码

![image-20240831225516695](/assets/images/image-20240831225516695.png)

* 3: 假设i=1，则获取所有存在v1字段的表
* 4: 不包含v1字段的表
* 5: R_join中所有的表，获取相应的V1字段的值，取交集，即共同存在的v1值，可以理解是一个k的集合
* 5: 遍历k集合： v = k1
* 将R_join中所有的 （v=k1）的tuple取出来，组成一个新的R_next
* 递归执行下一个v2属性

#### Implementation Challenges

*  index structures have to be built on-the-fly during query processing

* 伪代码（5）的交集计算的算法性能至关重要

  * traditional B+-trees or plain sorted lists 易于build，但是计算性能差
  * EmptyHeaded and LevelHeaded on-the-fly build的性能差

  > hash trie index structure

* 能够自适应选择binary join 、 multi-way-join

  > hybrid query optimization approach 

## 3. MULTI-WAY HASH TRIE JOINS

The workhorse of this approach is a novel *hash trie* data structure 

#### 普通nested hash tables

```json
# 顶层节点 (根节点)
hash_trie = {
    h1(Dept1): {  # 部门 ID = Dept1
        h2(Manager): [("E001", "Alice")],  # 职位 = Manager 对应的链表
        h2(Engineer): [("E002", "Bob")]    # 职位 = Engineer 对应的链表
    },
    h1(Dept2): {  # 部门 ID = Dept2
        h2(Manager): [("E003", "Charlie")]  # 职位 = Manager 对应的链表
    }
}
```

#### 性能问题

每次查询产生的间接损耗

* 为了解决hash冲突，成功的查找，都会执行一次key comparison

  > 延迟比较
  >
  > 使用hash value比较

### 3.2 Join Algorithm Description

算法分为2个阶段“build phase、probe phase

#### *3.2.1 Hash Tries*

> join attributes and their order are determined

![image-20240901103842484](/assets/images/image-20240901103842484.png)

#### *3.2.2 Build Phase*

![image-20240901130412839](/assets/images/image-20240901130412839.png)

#### *3.2.3 Probe Phase*

![image-20240901130753310](/assets/images/image-20240901130753310.png)

### 3.3 Implementation Details

#### *3.3.1 Hash Trie Implementation*

![image-20240901134559245](/assets/images/image-20240901134559245.png)

*singleton pruning*： 如上图的(0,1)

*lazy child expansion*

> only create the root nodes of the hash tries in the build phase, and create any nested hash tables on-demand when they are accessed for the first time during the probe phase

#### *3.3.2 Build Phase*

* 相同分区的数据物理上临近
* 相同数据可以复用index结构

#### *3.3.3 Probe Phase*

*  fully unroll the recursion in Algorithm 3 within the generated code
* fully parallelized by splitting the outermost loop

### 3.4 Further Considerations

* 索引可以作为一个插件，避免在写入时马上构建
* on-the-fly的特性，让hash-based approach 比comparison-based approach更高效

### 4. OPTIMIZING HYBRID QUERY PLANS

propose a **heuristic approach** that refines an **optimized binary join plan** by replacing cascades of **potentially growing joins** with **worst-case optimal joins**

* **cardinality estimates** that are used during regular join order optimization

![image-20240901141153000](/assets/images/image-20240901141153000.png)

* it is classified as a **growing join**
*  its **output cardinality** is greater than the maximum of its input cardinalities

### 5. EXPERIMENTS

 Umbra_OHT vs Umbra_EAG vs Umbra

>  Umbra_OHT 具有自适应能力的join实现
>
> Umbra_EAG 全部使用了worst-case join的实现
>
> Umbra 常规实现

#### *Traditional OLAP Workloads*

![image-20240901144104613](/assets/images/image-20240901144104613.png)

>  基准线是：Umbra的性能
>
> JOB: join order benchmark

#### *Relational Workloads with Growing Joins*

> the Umbra_OHT system exhibits the best overall performance, improving over Umbra by a factor of 1.9× and over Umbra_EAG by a factor of 4.2×

![image-20240901144824696](/assets/images/image-20240901144824696.png)

* 总共32个测试
* neither DBMS_ X nor the Umbra_EAG system are able to match the performance of the unmodified version of Umbra
* the Umbra_OHT improves over the performance of Umbra by identifying six queries
* Umbra_OHT system do not incur any timeouts on this benchmark

#### *Graph Pattern Queries*

> worst-case optimal join plans is very good

![image-20240901145415767](/assets/images/image-20240901145415767.png)

> Umbra_LFT: the Leapfrog Triejoin algorithm within Umbra

*  Umbra_OHT system best runtime  and up to two orders of magnitude
* the unmodified version of Umbra matches the performance of our hash trie join implementation on the 3-clique query

![image-20240901145800315](/assets/images/image-20240901145800315.png)

* Same result

### Detailed Evaluation

#### *Applicability of Worst-Case Optimal Joins*

![image-20240901150903535](/assets/images/image-20240901150903535.png)

* as the number of duplicates in the join result is increased，the runtime of binary join plans increases much more rapidly
* r > 10_4, resulting in good performance across the full range of possible query behavior.

#### *Optimizer Evaluation*

![image-20240901152115811](/assets/images/image-20240901152115811.png)

|                 | decision matches（positives） | decision matches（negatives） |
| --------------- | ----------------------------- | ----------------------------- |
| 正确判断(Ture)  | 正确判断、multi-wayjoin       | 正确判断、非multi-wayjoin     |
| 错误判断(False) | 错误判断成multi-wayjoin       | 错误判断成不是multi-wayjoin   |

## 6.RELATED WORK

suboptimal performance of growing intermediate results

>  [10, 19,30,60]

avoiding redundant partitioning steps

> [19,30]

propose a worst-case op- timal join algorithm

> [45,46,47]

* operators beyond joins

  >  [3,25,26,29,32,57]

* stronger optimality guarantees

  > [5, 33,34,45]

* incremental maintenance of the required data structures [27,28]

well-known Leapfrog Triejoin algorithm that is used in the LogicBlox system and can be implemented on top of existing ordered indexes or plain sorted data [13, 54, 56]

* adopted in distributed query pro- cessing [4,6,13,35] 
* graph processing [3,6,22,42,59,62]
* general-purpose query processing [2,8]

上述的实现，属于comparison-based implementations，存在本文之前阐述的问题

但是，上述问题对于distributed query processing（communication costs  >>> computation costs ）而言，是不存在的[13]

Fekete et al. propose an alternative, radix- based algorithm that achieves the same goal, but do not evaluate an actual implementation of their approach [14]

数据结构的发展

* hash array mapped tries [49]
* extendible hashing schemes [20]

当前成熟的几个系统

* LevelHeaded only allows for static data [2, 3]

* commercial LogicBlox system allows for fully dynamic data   through incremental maintenance of the required index structures [8,27]

自适应能力

LogicBlox is reported to also employ a hybrid optimization strategy [2]

Approaches that holistically optimize hybrid join plans have been proposed for graph processing [42,62]

* 指标计算成本很高

introducing all multi-way joins using generalized hash teams into binary join plans [19,21,30]

