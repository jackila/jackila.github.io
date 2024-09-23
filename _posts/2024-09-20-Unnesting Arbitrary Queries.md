> Unnesting Arbitrary Queries

**Abstract**

nested queries 一般会导致很差的性能，当前系统使用**a number of heuristics** to ***unnest*** these queries。但是这种方式不通用，本文提出一种通用方案**unnesting arbitrary queries**

## 1 Introduction

#### nested query 问题描述

##### 原始SQL

```sql
select s.name,e.course
       from   students s,exams e
where  s.id=e.sid and
              e.grade=(select min(e2.grade)
                       from exams e2
                       where s.id=e2.sid)
```

##### 优化后SQL

```sql
select s.name,e.course
        from   students s,exams e,
               (select e2.sid as id, min(e2.grade) as best
                from exams e2
                group by e2.sid) m
        where  s.id=e.sid and m.id=s.id and
               e.grade=m.best
```

常见的系统可以通过：`外部查询提供的属性用来替代内部查询中的自由变量` 解决

但是，对下面的sql，无能为力

```sql
select s.name, e.course
   from   students s, exams e
   where  s.id=e.sid and
     (s.major = ’CS’ or s.major = ’Games Eng’) and
     e.grade>=(select avg(e2.grade)+1 --one grade worse
               from exams e2          --than the average grade
               where s.id=e2.sid or   --of exams taken by
                     (e2.curriculum=s.major and --him/her or taken
                     s.year>e2.date))           --by elder peers

```



unnest query的性能提升

*  算法复杂度 ： **O(n*n) algorithm (nested loop join)**  提升到 **an O(n) algorithm (hash join, joining keys)**
* get a factor 10 or even 100 performance improvement

## 2 Preliminaries

$$ {ddd}
T_1 ⋈_p T_2 := σ_p(T_1 × T_2)
$$

1. $T_1 ⋈_p T_2 := σ_p(T_1 × T_2)$

> 笛卡尔集合后，通过谓词p过滤

#### 2. dependence join 公式

![image-20240920214201146](/assets/images/image-20240920214201146.png)

> the right hand side **is evaluated** for **every tuple** of the left hand side

3. A(T_1) : T_1表的属性字段

4. F(T_2) : T_2 表的 free variables

> 自由变量是在内部查询中依赖于外部查询的变量。也就是说，这些变量的值是由外部查询确定的

5. F(T2) ⊆ A(T1) : 一般dependence join都会有这样的约束

>  如果F(T)∩A(D)=∅，那么可以将dependence join转换为 regular join(下文会提到)

6. 

<img src="/assets/images/image-20240920215259101.png" alt="image-20240920215259101" style="zoom:50%;" />

> 最左的join 与 最右的join之间，通过**谓词p**与**C表中字段**进行关联

7. 

<img src="/assets/images/image-20240920215641142.png" alt="image-20240920215641142" style="zoom:50%;" />

> group 表达式
>
> e： 集合
>
> a：映射字段 如，sum(*) as a
>
> A： group by A
>
> $\forall$: 所有，每一个

8. 

<img src="/assets/images/image-20240920220016703.png" alt="image-20240920220016703" style="zoom:50%;" />

> 将函数f计算的结果映射到字段a中，类似 sum(1) as a

9. 

<img src="/assets/images/image-20240920220136023.png" alt="image-20240920220136023" style="zoom:50%;" />

> t1表与t2表字段a是相同的

## 3 Unnesting

### 3.1 Simple Unnesting

```sql
select ...
   from   lineitem l1 ...
   where  exists (select *
                  from lineitem l2
                  where l2.l_orderkey = l1.l_orderkey)
...
```

#### 代数表示:

<img src="/assets/images/image-20240920220356748.png" alt="image-20240920220356748" style="zoom:50%;" />

将 `dependent predicates ` 尽可能的不断上移，直至所有的attributes在输入中都存在，则可以转换为 regular join，

<img src="/assets/images/image-20240920220831416.png" alt="image-20240920220831416" style="zoom:50%;" />

### 3.2 General Unnesting

如果上述方式无效，则执行如下操作

1. 按照如下的公式，转换 **dependent join** 

<img src="/assets/images/image-20240920221117393.png" alt="image-20240920221117393" style="zoom:50%;" />

> transformed a **generic dependent join** into a dependent join of **a *set*** (i.e., **a relation without duplicates**)

#### 示例：

##### 原始

<img src="/assets/images/image-20240921071244227.png" alt="image-20240921071244227" style="zoom:50%;" />

##### 转换后

<img src="/assets/images/image-20240921071342145.png" alt="image-20240921071342145" style="zoom:50%;" />

2. **push the new dependent join down** into the query until we can transform it into a regular join

#### 例子

<img src="/assets/images/image-20240921071547188.png" alt="image-20240921071547188" style="zoom:50%;" />

最终我们的目的是如下的转换

<img src="/assets/images/image-20240921071635696.png" alt="image-20240921071635696" style="zoom:50%;" />

> 如果T中的free variable 不再依赖 临时表D，那么 dependence join 可以直接转换为 regular join

#### 转化算法

##### selections

<img src="/assets/images/image-20240921071918857.png" alt="image-20240921071918857" style="zoom:50%;" />

##### join

<img src="/assets/images/image-20240921072029962.png" alt="image-20240921072029962" style="zoom:50%;" />

##### *outer joins*

<img src="/assets/images/image-20240921072137324.png" alt="image-20240921072137324" style="zoom:50%;" />

##### *semi join* and *anti join*

<img src="/assets/images/image-20240921075527805.png" alt="image-20240921075527805" style="zoom:50%;" />

##### *group-by* operator

<img src="/assets/images/image-20240921075618467.png" alt="image-20240921075618467" style="zoom:50%;" />

##### Project operator

<img src="/assets/images/image-20240921075659657.png" alt="image-20240921075659657" style="zoom:50%;" />

##### set operations

<img src="/assets/images/image-20240921075743273.png" alt="image-20240921075743273" style="zoom:50%;" />



### 3.3 Optimization of Example Query Q1

![image-20240921081809078](/assets/images/image-20240921081809078.png)

#### decouple both sides

If **all attributes** of the domain are **(equi-)joined** with existing attributes, we can instead **derive the domain** from the existing attributes

<img src="/assets/images/image-20240921082039801.png" alt="image-20240921082039801" style="zoom:50%;" />

* **map operator** that substitutes d.id with e2.sid

## 4 Optimizations

上述的转换是一种保守的实现，接下来我们介绍一下性能更好的转换方式：移除表D

* find **equivalence classes** that are induced by the join and filter conditions
* identified the equivalence classes C
  <img src="/assets/images/image-20240921085259374.png" alt="image-20240921085259374" style="zoom:50%;" />

> instead of **joining with D**, we can **extend T** (using the map operator) and **compute the implied attribute value** from D by using the equivalent attributes.

由于这种替换会导致中间表数据的增长，所以需要根据cost判断是否执行该操作

