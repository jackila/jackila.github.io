---
tags: 论文
---
> [Plan Stitch: Harnessing the Best of Many Plans](https://15721.courses.cs.cmu.edu/spring2024/papers/15-optimizer3/p1123-ding.pdf)

## ABSTRACT

查询计划回归(Query performance regression)是指优化器选择了一个性能更差的执行计划。常见的商业DB会通过**RBPC(**reversion-based plan correction)从历史执行过的计划中，选择一个之前性能更好的计划。

However, this approach **ignores** potentially **valuable information** of **efficient *subplans*** collected from *other* previously-executed plans

本文提出了一个新的方案plan stitch：**automatically** and **opportunistically** combines **efficient subplans** of previously-executed plans into a valid new plan

## 1. INTRODUCTION

有时新执行计划会选择一些higher execution cost 比之前的计划。参考：[5, 6, 31]

**APC(Automatic plan correction)** 是常见的解决方案，可以自动的监控、修正查询计划，并且大概率是一个有效计划。

RPBC将自己限制在只能处理一个完整的、已执行过的计划。但是通过算子级别的执行消耗指标，Plan Stitch可以低风险组装出一个性能更高的执行计划

#### Plan Stitch challenge

* 从大量的算子、子计划中发现高效的子计划
* combine **the access paths**, **physical operators**, and **join orders**  from **these subplans** to a **single valid** plan

#### plan stitch solution

formulating the problem as a **plan search problem** similar to traditional query optimization, but in a **constrained search space.**

####  constrained search space

* Every **physical operator**  must appear in a previously-executed plan of the **same query** with **exactly the same logical expression**
* Every **physical operator**  must be **valid** in **current configuration**

#### 特性：automatic, low-overhead, and low-risk

> plan stitch 从执行过的算子中获取执行损耗，因此是low-rist，这一点至关重要

#### 其他

* applied even when there is **no re- gression**
* For parameterized queries (query templates) or stored procedures 

### 本文重点

* **propose Plan Stitch**：fully-automated, low-overhead technique construct new plans that are cheaper 
* describe an efficient, dynamic **programming-based approach** to construct a plan with the cheapest execution cost.
* implement Plan Stitch as a **component layered on top** of Microsoft SQL Server

## 2. OVERVIEW

#### Problem statement

Plan Stitch会从输入 (⟨q, P, C⟩) 计算出新的output(p)

#### Architectural overview

![image-20240927084648797](/assets/images/image-20240927084648797.png)

Plan Stitch 使用heuristics invalidate the execution data 

## 3. PLAN STITCH

#### two major components

* 构建constrained search space
* construct the stitched plan 

### 3.1 Constrained Search Space

挑战

* identifying **equivalent subplans** from different plans pi
* **compactly** encoding these equivalent subplans **in a structure** to **allow efficient search**

#### Identifying equivalent subplans

Every node  represents a **logical expression** with the **required physical properties**

相同的标准

* the same logical expression
* the required physical properties

如何实现

**Previous work** has proposed **tests and greedy algorithms** to match equivalent logical expressions [23,39] to enable the **query optimizer** to **match views and detect duplicate expressions**

* use similar heuristics
* use the optimizer to ensure the correctness 

#### Encoding the constrained search space

an **AND-OR graph**

* consists of AND and OR nodes
* AND node corresponds to a **physical operator** in a plan
  * Hash Join
* OR node represents **a logical expression** with the **required physical properties**
* the children of an AND node are OR nodes
* The children of an OR node are AND nodes

如何构建一个AND-OR graph

a physical operator in pi [**AND node**]--------> all the equivalent subplans from pj ∈ P ------------> create an OR node [**OR node**]---------> root physical operator of each subplan [**AND node**]

是否可以加入AND-OR graph，需要满足该operator 是否符合当前的配置

### 3.2 Constructing the Stitched Plan

##### AND-OR graph 两个核心特性

* acyclic，有向无环图
* at least one OR node

**Stitching plans** construct **from leaf** AND nodes **to the root** OR node using **dynamic programming**

**Costing stitched plans** 

Plan Stitch **combines** the **observed execution cost** of the operators in the stitched subplan

`stitchedSubUnitCost(opCost, execCount, {(childSubUnitCost, childExecCount)})`

公式： op算子的执行损耗、次数，以及子算子的损耗、次数

##### Assumptions in costing

* 一个算子的执行消耗，在不同计划中相同
* 多次执行的计划，均摊执行消耗，并且忽视首次启动损耗

### 3.3 Stitch Algorithm

<img src="/assets/images/image-20240927212907792.png" alt="image-20240927212907792" style="zoom:50%;" />

* **初始化和排序（第1行）**

  将AND-OR图中的子计划组（subplan groups）从**底部到顶部**排序

* **遍历OR节点，初始化成本（第2-4行）**

* **处理AND节点（第5-9行）**

  * 叶子结点处理

* **递归处理非叶节点（第10-15行）**

  * 遍历and结点下的每一个or结点，拼接到op结点后计算、比较厚获取bestSubPlan
  * 计算best plan的成本

* **更新最佳子计划（第16-18行）**

* **返回结果（第21行）**

#### Time complexity

>  *The worst-case running time for the plan stitch algorithm is* Θ((NM)2)*.*

虽然时间复杂度很高，但是基于以下优势，算法很少能到达**worse-case running time**

* 实际的查询，with a few hundred of operators and a handful of plans to stitch
* the equivalent subplan matches are usually sparse（较少的）

## 4. IMPLEMENTATION

作者基于Figure 2在Microsoft SQL Server  实现了Plan Stitch ，并且可以借助查询hints强制使用stitch plan

* implement Plan Stitch **external to** SQL Server
* using **heuristics** for subplan match
* the existing mechanism to **force and validate** the stitched plan

#### Matching equivalent subplans

1. 排除绝对不会相同子计划

   * different joined tables

   * not matching interesting orders

2. 只考虑满足必要条件的候选匹配

   * the joined tables（相同的表）

   * sort order of output columns（输出顺序相同）

3. 通过比较表达式树，尽可能匹配查询中计算的表达式
   * 详细比较表达式结构
4. 其他方式
   1. sort orders
   2. consider the serial or parallel mode

#### Forcing the stitched plan

the **optimizer** must **ensure** that stitched plan is correct

* 约束搜索空间，仅考虑符合特定计划结构的执行方案
* 优化器需要找到计划中的**表达式与原始查询中的表达式等效**的部分
* relies on **heuristics** to perform this match
* SQL Server’s query optimizer uses a **large collection of transformation rules**
* 会导致false positive 但是不会导致 false negatives
  * 错误判断不等效，但是不会错误判断等效

#### Handling errors in validation

handles such failed validations with ***two-stage stitch***

* a second attempt with ***sparse stitch***，比如通过移除一些复杂算子（bitmap，Compute Scalar）
* 如果还是invalid，uses the cheapest previously-executed plan which is valid
*  does not necessarily sacrifice the quality of the stitched plan

## 5. EXPERIMENT

本文将从以下几个方面分析stitch plan的效果

* Plan Quality：性能提升如何，带来的风险如何
* Cost Estimation：stitch plan估算的执行损耗比起真是损耗的差距如何
* Coverage：stitch plan优化plan的覆盖面有多少
* Overhead： 新增多少额外成本
* Stitched Plan Analysis 
  * **How different** is the stitched plan compared to the optimizer’s plan
  * How many previously- executed plans **are used for** the stitched plan
  * **Why** does the optimizer miss the cheaper stitched plan in its optimization
* Parameterized Queries
  * How much does Plan Stitch improve in **aggregated execution cost** of query instances
* Data Changes 
  * How much does **cost estimation** in Plan Stitch **degrade** when data changes

### 5.2 Plan Quality Improvement

* Plan Stitch further reduces the execution cost by **at least 10% for at least 40%** of stitched plans across all the workloads
* the percent of stitched plans that regress more than 10% compared to RBPC is **less than 2.7%**

### 5.3 Cost Estimation

由于我们在前文中一些假设，会导致预估的cost与真实的cost之间存在部分差异

* the estimation is **within 20%** for at least 70% of stitched plans, with a maximum of 85% in Cust1 workload
* In all our real-world customer workloads, the misestimate is **less than 50%**

有一些超过50%的误判，主要是由于**implementation artifacts** of SQL Server

* 由于plan forcing导致的变化，优化器的影响
  * Bitmap and Parallelism are ignored
  * Sort and Compute Scalar operators can be rearranged
* the variance of runtime factors can lead to errors in cost estimation
  * 比如内存不足导致的spills to disk

### 5.4 Coverage

* Plan Stitch improves up to **20×** more plans compared with RBPC

* 不同配置下，至少一个超过10%的的提升，Plan Stitch covers up to **6× additional queries** compared with RBPC

### 5.5 Overhead

**overhead of Plan Stitch** is **less than 88%** of the time compared with **optimizing the corresponding query with the optimizer**

### 5.6 Stitched Plan Analysis

Stitched Plan变更情况

* at least 94% of stitched plans, the leaf operators change
* at least 83% of stitched plans, the internal operators are different
* more than 63% have plan structural changes
  *  including more than 33% with different join orders

Stitched plan 使用的query情况

* more than 53% from up to 4 plans across our workloads

为什么优化器没有优化成stitched plan

* a cost misestimate（绝大部分）
* search strategy issue （10%- 22%）

### 5.7 Parametric Queries

Plan Stitch improves the plan quality without introducing more variance of execution cost compared to RBPC

### 5.8 Data Changes

Plan Stitch can **regress less** in the changed databases than in the original data- base compared with RBPC

## 6.RELATED WORK

### Execution feedback to improve plan quality

常见类型

* using **observed cardinality** of a query’s expressions to **improve optimization** of the **same query**
* using **observed cardinality** to improve **summary statistics** to **help many queries**
* improving the **optimizer’s cost model**

### Plan regression correction

最常见的优化方式

### Query hinting

query hints can influence the join order, ac- cess paths, up to providing the entire plan

### Exploring alternative plans

AND-OR graph [24] represents **a search space** that **allows the exploration of alternative plans**

* modify leaf AND nodes  
* reuse the internal plan structure and the query optimizer’s cost,