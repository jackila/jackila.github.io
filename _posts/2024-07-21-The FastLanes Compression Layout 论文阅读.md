> [The FastLanes Compression Layout: Decoding >100 Billion Integers per Second with Scalar Code](https://15721.courses.cs.cmu.edu/spring2024/papers/03-data2/p2132-afroozeh.pdf) 论文阅读

### 概述

通过数据并行提高轻量压缩（LWC）性能，针对压缩布局，本文提出了2种方式

* 在bit-(un)packing操作中，使用虚拟的1024位SIMD寄存器应用值交错技术(interleavng)
* 使用统一转置布局(Unified Transposed Layout)重新排序所有列中的记录,该布局将记录按照 `04261537`排列

另外通过定义一套虚拟的1024位SIMD的指令集，支持所有的SIMD方言，解决异构问题。标量版本通过现代编译器完全自动向量化，降低代码复杂度

### 1. 简介

##### Vectorized execution

借助表达式解释器将一批记录(1024个)的调用成本分摊，并借助编译器对标量代码进行优化生成SIMD指令，如loop-pipelining、code motion、auto-vectorization

##### Vectorized decoding

针对FOR、DICT、DELTA、RLE，在RAM与CPU之间解压缩向量数据

##### Parquet 

它采用的非交错bit packing与传统的RLE不支持数据并行，无法通过SIMD下提高性能

##### Compressed execution

不建议将数据直接解压缩到SQL类型，而是最小类型，比如uint64尽可能使用uint8，既可以降低数据结构大小，又能借助SIMD的特性提高性能

##### FastLanes

* 新的压缩列数据布局
* 数据并行解码
* 支持异构的ISAs
* 可以通过标量代码实现

#### 1.1 挑战与贡献

![image-20240721101333647](/assets/images/image-20240721101333647.png)

#### 1.2 大纲

* 解释了1024-bits interleaved bit-unpacking、The Unified Transposed Layout 
* 基于上述设计的高效的RLE编码
* 基准测试
* 相关工作与未来计划

### 2. fastlanes

#### 布局

![image-20240721102529770](/assets/images/image-20240721102529770.png)

上图是将1024个3bit大小的数据，在bit-packing下，通过interleaved方式的布局，假设当前为1024位的SIMD

#### 统一的ISA指令

![image-20240721102936926](/assets/images/image-20240721102936926.png)

#### 举例说明

##### SIMD指令

![image-20240721103028351](/assets/images/image-20240721103028351.png)

##### 数据运行情况

![image-20240721103113083](/assets/images/image-20240721103113083.png)

#### 结果

* 10条指令，解压了384个数据
* 经测试AVX512 CPU下，一个周期处理了70个数据，即per cycle = 140 billion values per second on one 2GHz core

#### 2.3 Dealing with Sequential Data Dependencies

针对

![image-20240721103755356](/assets/images/image-20240721103755356.png)

通过重新排序

![image-20240721103826161](/assets/images/image-20240721103826161.png)

文章论证了排序对于存储影响极低，可以通过在selection vector重新进行排序

需要注意的是，虽然冲排序后，向量查询会忽视selection vector，但是在SIMD之下，均摊后的成本非常低



#### 2.4 The Unified Transposed Layout

![image-20240721104324109](/assets/images/image-20240721104324109.png)

对于一个128bit SIMD来说，32位的4 values是合理的，能够充分利用SIMD，但是对于16位的数据，就只能利用一半的SIMD性能，于是提出了 The Unified Transposed Layout

将数据统一布局

![image-20240721104740748](/assets/images/image-20240721104740748.png)

而在运算时，根据不同位的数据，进行计算

![image-20240721104828147](/assets/images/image-20240721104828147.png)

##### 优化RLE

![image-20240721105114702](/assets/images/image-20240721105114702.png)

### 3. 性能评估

##### Q1: FastLanes1024- bit interleaved bit-unpacking 的速度如何

8-bits： SIMD / scala code = 40x-70x

64-bits: SIMD / scala code = 3x-4x 

70 tuples per cycle for 8-bits types 

##### Q2: SIMD的宽度是否影响解压性能，在不同平台下的性能差异如何

* Gravitons weaker SIMD
* Wider SIMD  不一定会提升性能
* ILP(instruction level paralellism ) and register width 会提升性能

##### Q3: 标量代码(scalar code)是否能获益

Scalar_T64 会提升scalar code性能

##### Q4: scalar implementation、auto-vectorization、 explicit SIMD intrinsics之间的性能差距如何

auto-vectorization == explicit SIMD intrinsics

##### Q5: Unified Transposed Layout是否对性能有影响，尤其是对于Delta类型的sequential dependencies

* very high performance for Auto-vectorized.
* scalar_T64 profits from data-parallelism in the Unified Transposed Layout
*  Scalar cannot and can be >40x slower than SIMD

##### Q6: FastLanes 在end to end query中的性能如何 

* reading from FastLanes typically makes a query faster, despite the decompression



### 4. 相关工作

#### Bit-packing

#### DELATA

#### RLE

### 5.未来工作