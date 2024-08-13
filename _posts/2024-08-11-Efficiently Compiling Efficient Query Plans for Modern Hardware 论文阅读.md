>  [Efficiently Compiling Efficient Query Plans for Modern Hardware](https://15721.courses.cs.cmu.edu/spring2024/papers/07-compilation/p539-neumann.pdf) 论文阅读

## 概述

为了提高cpu的利用，有一些技术已经非常有效，比如批处理或向量化处理。本文提出一种新颖的编译策略用来提高cpu的性能，通过LLVM编译框架将query转换为一个精简并且高效的机器码

## 1.简介

常规的火山模型适合IO成本较高的查询，对于CPU为主的query，可能会有以下问题

* next方法与被调用的记录数量有关，可能达到数百万次
* call a next通常是一次virtual call or  a call via a function pointer
  * 比regular call性能差
  * 分支预测上会出现降级情况
* 较差的代码局部性、复杂的book-keeping
  * 比如对压缩数据的scan

常见批处理的优化方式无法将数据在不copy 的情况下传递到parent operator

本文的算法特点

* Processing is data centric and not operator centric
  * 尽可能的将数据keep in cpu register 
* Data is not pulled by operators but pushed towards the operators
  *  better code and data locality
* Queries are compiled into native machine code using the optimizing [LLVM compiler framework](LLVM: A compilation framework for lifelong program analysis & transformation. )

LLVM不仅性能上与hand-code相近，而且后续可以借助框架未来的迭代，使用其带来的code optimization, and hardware improvements

## 2.相关工作

* 当前流行的volcano system【4】

  *  dominated by disk I/O 
  * a. The Volcano optimizer generator: Extensibility and efficient search

* 通过在算子之间传递一批数据，reduce the high calling costs of the iterator model

  * Block oriented processing of relational database operations in modern computer architectures

* MonetDB system 物化所有intermediate results & vectorized manner 

  > 即便如此，根据压测也是没有达到hand-code的性能

  * Database architecture evolution: Mammals flourished long before dinosaurs became extinct.
  * Optimizing database architecture for the new bottleneck: memory access.

* compile the query into some kind of executable format, instead of using interpreter structures
  * Compiled query execution engine using JVM
* compiling the query into C code using code templates for each operator
  * K. Krikellas, S. Viglas, and M. Cintra. Generating code for holistic query evaluation. In ICDE, pages 613–624, 2010
* reducing the impact of branching
  * K. A. Ross. Conjunctive selection conditions in main memory. In PODS, pages 109–120, 2002
* using SIMD instructions
  * V. Raman, G. Swart, L. Qiao, F. Reiss, V. Dialani, D. Kossmann, I. Narang, and R. Sidle. Constant-time query processing. In ICDE, pages 60–69, 2008
  * M. Zukowski, P. A. Boncz, N. Nes, and S. H´eman. MonetDB/X100 - a DBMS in the CPU cache. IEEE Data Eng. Bull., 28(2):17–22, 2005

## 3.THE QUERY COMPILER

### 3.1. Query Processing Architecture

>  pipeline-breaking： takes an incoming tuple out of the CPU registers

基本思路：While pushing tuples, we continue pushing until we reach the next pipeline-breaker

* cheap to compute for tuple in cpu register
* 复杂的逻辑控制流（分支、条件判断和复杂决策）放在 tight loops外
* 对于不可避免的pipeline-breakers，通过优化尽可能的降低对内存的访问

![image-20240812082537723](/assets/images/image-20240812082537723.png)

![image-20240812082558380](/assets/images/image-20240812082558380.png)

本文最主要的挑战就是将figure3中的query转换为figure4中的代码块

* keep tuples in CPU registers and only access memory to retrieve new tuples or to materialize their results
*  very good code locality as small code fragments are working on large amounts of data in tight loops

### 3.2 Compiling Algebraic Expressions

编译代数表达式是困难的

* 算子之间的边界模糊
* 一个算子的逻辑可能会被分散到多个代码块中
* code fragment是一个非常规的结构

each operator offers two functions( a mental model)

* produce()

  >  the produce function asks the operator to produce its result tuples, which are then pushed towards the consuming operator by calling their consume functions.

* consume(attributes,source)

![image-20240813085908946](/assets/images/image-20240813085908946.png)

![image-20240813085445461](/assets/images/image-20240813085445461.png)

## 4. CODE GENERATION

### 4.1 Generating Machine Code

接下来，将要讨论如何将代数表达式编译成机器码。

最初，我们尝试生成c++代码，然后在运行时通过shard library被compiler加载

* 耗时过长
* c++代码控制力不够，不能提供更高效的性能，比如overflow flag的配置

取而代之，使用了LLVM compiler framework产生可移植的汇编代码（直接被optimizing JIT compiler执行）

* more robust than write it manually
  * hides the problem of register allocation by offering an unbounded number of registers

* LLVM汇编语言具有可移植性，通过JIT适配各种不同的计算机架构
* LLVM assembler是一种强类型，避免生成原始text 类型的c++代码
* LLVM is a full strength optimizing compiler
  * fast machine code
  * few milliseconds for query compilation

为了避免使用LLVM assember 完成所有的查询逻辑

* LLVM 与C++可以互相调用
* 复杂的查询逻辑使用C++，不同算子之间的联系使用LLVM code
* 核心部分（C++ 部分）会提前编译，LLVM `chain` 动态生成 

![image-20240813221017748](/assets/images/image-20240813221017748.png)

* 两种模式的切换是由LLVM控制
* hot path必须由pure LLVM执行

### 4.2 Complex Operators

对于复杂的算子，不可能也没有必要编译成一个函数

* 大多数情况下，LLVM code会调用C++代码
* 可能会导致代码的膨胀

所以，可以在LLVM中定义函数，合适的时机调用，但是必须保证

* hot path does not a function boundary

### 4.3 Performance Tuning

LLVM code 借助tight loop 提供了高效的memory prefetching 以及准确的分支预测

* 甚至code fragment 都已经不是瓶颈，由于其他代码的slow （hash）

* query compiler必须提供一个好的分支预测

##### 高效的内存存取

* 尽可能的延迟加载字段
* 使用pointer表示string类型

##### 准确的分支预测

```c++
Entry* iter=hashTable[hash];
 while (iter) {
   ... // inspect the entry
   iter=iter->next;
 }
```

问题：mixes up two things： 

* 判断hash中是否有值，（大多数情况为true）
* 判断是否是数组的最后一个 （大多数情况为false）

````c++
Entry* iter=hashTable[hash];
 if (iter) do {
   ... // inspect the entry
   iter=iter->next;
 } while (iter);
````

虽然看起来上面的问题比较复杂，但是避免这些问题并不麻烦，针对SQL-92的生成的代码总共也就11000行

## 5.ADVANCED PARALLELIZATION TECHNIQUES

尽可能的将数据保留在寄存器并不意味着我们只能线性的处理数据，其他高级处理技术可以很自然集成在一起

* Traditional block-wise processing & SIMD registers
  * LLVM directly allows for modeling SIMD values as vector types
* multi-core architectures for inter-query parallelism
  * solved by partitioning the input of operators
  * LLVM can support with nearly no code changes

## 6. EVALUATION

### 6.1 Systems Comparison

![image-20240813231632641](/assets/images/image-20240813231632641.png)

* OLTP workloads： 性能相似，但是编译时间差距在10倍以上
* OLAP part
  *  DB X is clearly much slower
  *  LLVM  整体上是其他的2-4倍
  * LLVM的编译时间非常的短

### 6.2 Code Quality

branching and cache effects

![image-20240813232223661](/assets/images/image-20240813232223661.png)

* LLVM code contains far less branches 
* mispredictions is significantly lower for the LLVM code
* LLVM code shows a very good data locality and has less cache misses than MonetDB
*  generated LLVM code is much more compact than the MonetDB code



## 7.CONCLUSION

* data-centric query pro- cessing is a very efficient query execution model
* machine code using the optimizing LLVM compiler  rivals hand-written C++ code
* compact and maintainable
* benefits from future compiler and processor improvements without re-engineering the query engine
