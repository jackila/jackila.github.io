---
tags: 论文
---
**[Froid: Optimization of Imperative Programs in a Relational Database](https://15721.courses.cs.cmu.edu/spring2024/papers/11-udfs/p432-ramachandra.pdf)**

## ABSTRACT

imperative functions and procedures 的性能极差，本文介绍了Froid框架，通过UDF转换为关系代数表达式，并嵌入到被调用的SQL query，这个框架有以下特点

* support cost-based optimization
* efficient, set-oriented, parallel plans 
* compiler optimizations with no additional implementation effort

## 1. INTRODUCTION

UDFs and procedures offer many advantages over standard SQL ，but has  a huge performance penalty

Froid using a novel technique to **automatically** convert **imperative programs** into **equivalent relational algebraic** forms 

* models **blocks** of imperative code as relational expressions
* **combines** them into a **single expression** using the **Apply operator**

* compiler optimizations(dead code elimination、program slicing、constant folding)
* not only T-SQL UDFs but also other imperative languages

##### following contributions in this paper

*  challenges in optimization of imperative code and reason
* describe the novel techniques underlying Froid
* show how several compiler optimizations used
* discuss the design and implementation of Froid and experimental evaluation

## 2. BACKGROUND

本文主要关注Scalar T-SQL UDFs

### Scalar UDF Example

![image-20240904233155331](/assets/images/image-20240904233155331.png)

`select c name, dbo.total price(c custkey) from customer;`

![image-20240905083744566](/assets/images/image-20240905083744566.png)

性能问题

* Iterative invocation

  > a lot of context switching

* Lack of costing

* Interpreted execution

  > No cross-statement optimizations are car- ried out, unlike in compiled languages

* Limitation on parallelism

## 3.THE FROID FRAMEWORK

### Intuition

`If the entire body of an imperative UDF can be expressed as a single relational expression R, then any query that invokes this UDF can be transformed into a query with R as a nested sub-query in place of the UDF`

### The APPLY operator

**combine** multiple relational expressions into a **single expression**, and SQL Server’s **query optimizer** can **remove the Apply operator** and enable the use of **set-oriented relational operations**

### Overview of Approach

![image-20240906025413943](/assets/images/image-20240906025413943.png)

### Supported UDFs and queries

![image-20240906025727656](/assets/images/image-20240906025727656.png)

## 4.UDF ALGEBRIZATION

### Construction of Regions

**Basic blocks**

> sequential regions

**if-else blocks**

> conditional regions

**loops**

>  loop regions

### Relational Expressions for Regions

#### *4.2.1 Imperative statements to relational expressions*

* Variable declarations and assignments
* Conditional statements
* Return statements
* Function invocations
* Others

#### *4.2.2 Derived table representation*

* Froid constructs the expression of each region as a derived table as follows

![image-20240906032353706](/assets/images/image-20240906032353706.png)

### Combining expressions using APPLY

![image-20240906032937131](/assets/images/image-20240906032937131.png)

## 5.SUBSTITUTION AND OPTIMIZATION

* set-oriented plans
* expensive operations inside the UDF **are now visible** to the optimizer, and **are now visible** 
* the UDF is **no longer interpreted** since it is now a single relational expression
* the limitation on **parallelism no longer holds** since the entire query including the UDF is now **in the same execution context**

One of the key advantages of Froid’s approach is that it requires **no changes to the query optimizer**

## 6.COMPILER OPTIMIZATIONS

#### Dynamic Slicing

relational algebraic transformations such as **projection-push- down** and **apply-removal** that Froid uses

#### Constant Folding and Propagation

SQL Server’s existing **scalar simplification mechanisms** simplify the expression

#### Dead Code Elimination

removed by the optimizer using **projection pushdown**

#### Summary

* the semantics of the Apply operator allows **the query optimizer** to **move and reuse operations** as necessary, while preserving correlation dependencies
* due to the way Froid is designed ,these techniques are automatically applied across nested function invocations, resulting in increased benefits due to **interprocedural optimization**

## 7. DESIGN AND IMPLEMENTATION

#### Cost-based Substitution

we chose to perform inlining **during binding** due to these reasons

* inlined version performs better in almost all cases
* requiring no changes to the query optimizer
* Certain optimizations such as constant folding are performed during binding

#### Imposing Constraints

Algebrization can increase the size and complexity of the resulting query 

* simplify the query tree reducing its size when possible
* restrict the size of algebrized query tree(restricts the size of UDFs)

Nested and Recursive functions

* controlling the **inlining depth** based on the size of the algebrized tree

#### Supporting additional languages

adding support for additional languages only re- quires

* plugging in a parser for that language
* providing a language-specific implementation for each sup- ported construct
* data type semantics need to be taken into account

#### Implementation Details

##### Security and Permissions

* Froid enlists the UDF for permission checks
* Others, Froid handles as the way view permissions 

##### Plan cache implications

* Protect cache plan 

##### Type casting and conversions

Froid **explicitly** inserts **appropriate type casts** for actual parameters and the return value

##### Non-deterministic intrinsics

certain non-deterministic functions such as GETDATE(),Froid **disable transforming** such UDFs

## 8.EVALUATION

### 8.1 Applicability of Froid

![image-20240906041734183](/assets/images/image-20240906041734183.png)

### 8.2 Performance improvements

#### *8.2.1 Number of UDF invocations*

![image-20240906042657158](/assets/images/image-20240906042657158.png)

With Froid enabled, we see an improvement of one to three orders of magnitude

#### *8.2.2 Impact of parallelism*

* For this particular UDF, SQL Server **switches to a parallel plan** when the cardinality of the table is greater than 10000 
* without parallelism, Froid achieves improvements up to **two orders of magnitude**

#### *8.2.3 Compile time overhead*

![image-20240906043104376](/assets/images/image-20240906043104376.png)

* We observe gains of more than an order of magnitude for all these UDFs
* Note that the compilation time of each of these UDFs is less than 10 seconds

#### *8.2.4 Complex Analytical Queries With UDFs*

![image-20240906043249192](/assets/images/image-20240906043249192.png)

* Froid leads to **improvements** of multiple orders of magnitude (compare (b) vs. (c))
* in most cases, there is no overhead to using UDFs when Froid is enabled (see (a) vs. (c))

#### *8.2.5 Factor of improvement*

![image-20240906043549414](/assets/images/image-20240906043549414.png)

* observe improvements in the range of 5x-1000x across both workloads
* there were 5 UDFs that showed no improvement or performed slightly worse due to Froid
  * complex recursive functions
  * can be handled by appropriately tuning the constraints 
* invoke expensive TVFs

#### *8.2.6 Columnstore indexes*

![image-20240906044019622](/assets/images/image-20240906044019622.png)

get about 5x improvement in performance 

#### *8.2.7 Natively compiled queries and UDFs*

![image-20240906044229196](/assets/images/image-20240906044229196.png)

* the classic mode of interpreted T-SQL, we see a 20x improvement due to Froid
* natively compiled both the UDF and the query, giving an additional 4.6x improvement over native compilation.

### 8.3 Resource consumption

![image-20240906044522500](/assets/images/image-20240906044522500.png)

* Reduce CPU time by 1-3 orders of magnitude for all UDFs
* Froid also reduces I/O costs



## 9. RELATED WORK

Optimization of SQL queries containing sub-queries \ optimize nested sub-queries

Optimization of imperative programs in the compilers community

perform sub-program inlining which applies only to nested function calls & cache function results

* Can not fix all drawbacks[Section 2.3]

use **programming languages techniques** to optimize database-backed applications

* transforms fragments of code into SQL using Query-By-Synthesis (QBS)
* QBS is based on **program synthesis**, whereas Froid uses a **program transformation** based approach
* it is limited in its scalability to large functions
  * none of those are larger than 100 lines of code
* QBS suffers from potentially very long optimization times

The StatusQuo system include

* a program analysis that identifies blocks of imperative logic that can be translated to SQL
* a **program partitioning method** to move application logic into imperative stored procedures
* The SQL translation in StatusQuo uses QBS[4] to extract equivalent SQL

static analysis based approach with similar goals.

* generated SQL turns out to be larger and more complex compared to Froid

decorrelate queries in UDFs using extensions to the Apply operator

* Froid does not require any new operators or operator ex- tensions 
* their trans- formation rules are designed to be a part of a **cost based optimizer**
*  they do not address vital issues
  * handling multiple return statements
  * avoiding redundant computation of predicate expressions

