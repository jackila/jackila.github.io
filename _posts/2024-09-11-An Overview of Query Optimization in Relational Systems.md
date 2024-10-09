---
tags: 论文
---
[An Overview of Query Optimization in Relational Systems](https://15721.courses.cs.cmu.edu/spring2024/papers/13-optimizer1/chaudhuri-pods1998.pdf)

## OBJECTIVE

focus primarily on **the optimization of SQL queries** in **relational database systems** and present my biased and incomplete view of this field, The goal of this article is not to be comprehensive,but rather to explain **the foundations and present samplings** of significant work in this area

## INTRODUCTION

 Two key components of the query evaluation component

* query optimizer 
* query execution engine

简单介绍了一下query execution engine，深入学习可以参考[[20]](GraefeG.Query Evaluation Techniques for Large Databases In. ACMComputingSurveys:Vol25. No2..June1993)

对于一个给定的SQL，通过优化器可以有非常多的operator tree

* 代数表达可以相互之间转换，

  >  Join(Join(A,B),C)=Join(Join(B,C),A)

* 同一个代数表达式又可以有很多operator tree的实现

为了选择高效的实现

* A space of plans(search space)
* a cost estimation technical 
* an enumeration  algorithm

一个完美的优化器应该包含

* the search space includes plans that have low cost
* the costing technique is accurate
* the enumeration algorithm is efficent

## AN EXAMPLE: SYSTEM-R OPTIMIZER

present a subset of those [important ideas](Selinger,P.G.,Astrahan,M-M., Chamberlin.D.D., Lorie. R.A., Price T.G. Access Path Selection in a Relational Database SystemI.n Readingsin DatabaseSystemsM. organKaufma) here in the context of Select-Project-Join(SPJ)queries

* join 算子具有交换性
*  cost model 
  * A set of statistics maintained on relations and indexes
  * Formulas to estimate **selectivity of predicates** and to project **the size of the output data** stream for every operator node
  * Formulas to estimate the **CPU and IO costs of query execution** for every operator
* enumeration algorithm for System-R optimizer
  * dynamic programming 
  * use of interesting orders.

> the idea of interesting order was later generalizedto **physical properties** in [22] and is used extensively in modem optimizers

#### 针对像physical properties导致违反最优性原则的情况，可以采用一些简单机制处理

System-R优化器的三个核心特性：cost-based optimization, dynamic programming and interesting orders

## 4.SEARCH SPACE

the search space for optimization

* the set of **algebraic transformations** that preserve equivalence
* the set of **physical operators** supported in an optimizer

常见的两种模式

* algebraic expression

  > algebraic expression -------->  the parse tree ------> logical operator trees (query trees) ---------> operator tree

* “calculus-oriented” representation

### 4.1 Commuting Between Operators

#### Generalizing Join Sequencing

Join sequencing可以有2种方式：linear sequence （大部分系统采用的方式） 、Bushy join sequence

另外，常见的优化方式还有：Deferring Cartesian products

有一些优化器会提供一个adaptor，决定是否使用Bushy join sequence或者Deferring Cartesian products

#### Outerjoin and Join

joins and outerjoins maybe reordere

Join(R, S LOJ T) = Join (R,S) LOJ T

#### Group-By and Join

![image-20240912201541619](/assets/images/image-20240912201541619.png)

### 4.2 Reducing Mu&Block Queries to Single-Block

it is possibleto **collapsea multi-block SQL query** into **a single block SQL query**

#### Merging Views

* unfolded the view definitions to obtain a single block SQL query

  > Q = Join(R,V) and viewV = Join(S,T) ===> Join(R, Join(S,T) )

* when contains a group by operator

  > requires the ability to **pull-up** the **group-by** operator and then to freely reorder **not only the joins** but also the **group-by operator** to ensure optimality

#### Merging Nested Subqueries

![image-20240912202143047](/assets/images/image-20240912202143047.png)

![image-20240912202155171](/assets/images/image-20240912202155171.png)

更复杂的一个转换

![image-20240912202248732](/assets/images/image-20240912202248732.png)

![image-20240912202255764](/assets/images/image-20240912202255764.png)



### 4.3 Using Semijoin Like Techniques for Optimizing Multi-Block Queries

![image-20240912202424090](/assets/images/image-20240912202424090.png)

转换成

![image-20240912202433035](/assets/images/image-20240912202433035.png)

![image-20240912202443121](/assets/images/image-20240912202443121.png)

## 5. STATISTICS AND COST ESTIMATION

The basic estimation framework

* Collect statistical summariesof data that has been stored
* Given an operator and the statistical summary for each of its input data streams,determine the
  * Statistical summary of the output datastream
  * Estimated cost of executing the operation

### 5.1 Statistical Summaries of Data

#### Statistical Information on Base Data

* the number of tupIes
* the number of physical pages
* Statistical information on columns
* information on the data distribution on a column is provided by histograms

#### Estimating Statistics on Base Data

* estimate the statistical parameters accurately and efficiently
  * sample data

#### Propagation of Statistical Information

propagate the statistical information through operator

### 5.2 Cost Computation

*  estimates CPU, I/O
* communication costs
* modeling buffer utilization

## 6. ENUMERATION ARCHITECTURES

the enumerator面对如下变化，可以轻松的调整

* the addition of new transformation
* the addition of new physical operator
* changes in the cost estimation technique

常见的2种实现方式：Sturburst and Volcuno &Cascades 

* Use of generalizedcost functions and physical properties with operator nodes
* Use of a rule engine that allows transformations to modify the query expressionor the operator trees
* Many exposed“knobs” thatcanbeusedto tune the behavior of the system

### Starburst

begins with a **structural representation** of the SQL query that is used throughout the lifecycle of optimization

####  query rewtite phase of optimization

* transform 
* A forward chaining rule engine governs the rules

#### plan optimization.

In **computing such derivations,** comparable plans that represent the same **physical and logical properties** but have higher costs, **are prune**

### Volcano/Cascades

 rules are used universally to represen **the knowledge of search space**

*  The transformation rules 

  > map an algebraic expression into another

* The implementation rules

  >  an algebraic expression into an operator tree

与starburst的核心区别

* These systems do not use two **distinct optimization phases** because all transformations are **algebrnlc and cost-base**
* The mapping from algebraic to physical operators **occurs in a single step**
* Insteadof applying rules in a **forward chaining fashion**, as in the Starburst query rewrlte phase,Volcano/Cascades **does goal-driven application of rules**

## 7.BEYOND THE FUNDAMENTALS

#### Distributed and Parallel Databases

* Distributed database introduce issues of communication costs

* Parallel databas exploit multiple processingelement

#### User-Defined Functions

#### Materialized Views

#### Other Optimization Issues

*  being able to defer generation of complete plans subjec to availability of rumtime information
* considering other resourcese,specially memory
* .....