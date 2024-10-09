---
tags: 论文
---
>  [Make the most out of your SIMD investments: counter control flow divergence in compiled query pipelines](https://15721.courses.cs.cmu.edu/spring2024/papers/06-vectorization/lang-vldbj2020.pdf)

## 0.摘要

SIMD面临GPU-like一样的问题：控制流分歧导致向量处理单元利用率不足。文章提出一种新的算法，通过将新元组分配给空闲的SIMD通道，解决这个问题。新算法的效果如下

* TPC-H Q1 表扫描语句，解决利用不足方面 性能提升34%
* HashJoin查询，性能提升25%
* 近似地理空间连接查询，性能提升30%

## 1. 简介

SIMD的应用场景： selection , join , partitioning , sorting , CSV parsing , regular expression matching ,and (de-)compression 

SIMD常见于 interpreting database systems（column at a time），对于AVX-512指令集，一个register最多可以同时处理64个tuple

SIMD的不充分利用场景

* some might is disqualified during predicate evaluation
* some may not find a join partner later on and get discarded

当前场景的解决方式是忽视这些disqualified 记录，借助一个bitmap对数据跟踪，这就导致没有充分利用的vector-processing units (VPUs)

另有一种解决方案是通过每个矢量化运算符之后执行一次materialization来解决。从register移动到memory导致了性能的降低

新算法特点：not a pipeline breakers ， applied at the intra-operator level as well as at operator boundaries

## 2. Background

#### Mask instructions

根据一个bitmask，只在特定的lanes执行向量操作

##### Merge masking

`dst = mask_add(src,mask,a,b)`

* a + b
* Mask  为0 的位置，将src的元素 copy到dest

##### Zero masking

`maskz_add(mask,a,b)`  src = [0]

#### Permute

根据一个index vector，对input vector进行shuffles

![image-20240804234123739](/assets/images/image-20240804234123739.png)

#### Compress/Expand

##### compress: 将bitmask标志的记录，依次写入目标寄存器

##### expand: 将input vector的记录，按照write mask依次写入到目标寄存器

![image-20240804234855460](/assets/images/image-20240804234855460.png)

## 3. Vectorized pipelines

![image-20240804235525760](/assets/images/image-20240804235525760.png)

可以看到图右侧，在scan阶段，register被完全利用了，但是随着 后续的filter、join等算子的执行，部分lanes是没有被利用的(x)

为了解决这个问题，文章通过refill，将新元组分配给空闲的lanes

## 4. Refill algorithms

算法通过位图标识活动的lanes，新数据的来源分为2种：source memory address ， in vector register

### 4.1 Memory to register

![image-20240805001423974](/assets/images/image-20240805001423974.png)

* 先bitwise not 确认 write mask
* 然后通过expand，将数据加载到register中

与此同时，scan operator还会将TID加载到一个output vector，用于后续的加载属性数据，算法如下

![image-20240805001709616](/assets/images/image-20240805001709616.png)

### 4.2 Register to register

![image-20240805003245465](/assets/images/image-20240805003245465.png)

* write mask是 目标reigister 标识的不活跃lanes
* 通过1、2 指令，可以计算出permutation indices
* 通过permutation indices，permute指令即可实现从register to register的目标
* permutation mask 是为了，当没有足够的active element可以填充目标register，需要更新write mask 的inactive lanes
* algorithm支持不计算permutation indices，直接移动数据。但是这种迁移操作会涉及多个vector pair，所以均摊下来，计算perm.indices更高效

```c++
[...]
//Prepare the refill.
fill_rr r(src_mask, dst_mask);
//Copy elements from src to dst.
r.apply(src_tid, dst_tid);
r.apply(src_attr_a, dst_attr_a);
r.apply(src_attr_b, dst_attr_b);
r.apply(..., ...);
//Update the destination mask,
r.update_dst_mask(dst_mask);
//and optionally the source mask.
r.update_src_mask(src_mask);
[...]
```

### 4.3 Variants

当vectors的元素处于compressed state，即active element 被依次存储，存在一种效率更高的permutation过程.这个高效算饭避免了 compress/expand 的指令

##### 通用的fullfill 算法

```c++
struct fill_rr {
__mmask8 permutation_mask;
__m512i permutation_idxs;
//Prepare the permutation.
fill_rr(const __mmask8 src_mask,
const __mmask8 dst_mask) {
__m512i src_idxs = _mm512_mask_compress_epi64(
ALL, src_mask, SEQUENCE);
__mmask8 write_mask = _mm512_knot(dst_mask);
permutation_idxs = _mm512_mask_expand_epi64(
ALL, write_mask, src_idxs);
permutation_mask = _mm512_mask_cmpneq_epu64_mask(
write_mask, permutation_idxs, ALL);
}
//Move elements from ’src’ to ’dst’.
void apply(const __m512i src, __m512i& dst) const{
dst = _mm512_mask_permutexvar_epi64(
dst, permutation_mask, permutation_idxs, src);
}
void update_src_mask(__mmask8& src_mask) const {
__mmask8 compressed_mask =
_pext_u32(~0u, permutation_mask);
__m512i a =
_mm512_maskz_mov_epi64(compressed_mask, ALL);
__m512i b =
_mm512_maskz_expand_epi64(src_mask, a);
src_mask =
_mm512_mask_cmpeq_epu64_mask(src_mask, b, ZERO);
}
void update_dst_mask(__mmask8& dst_mask) const {
dst_mask =
_mm512_kor(dst_mask, permutation_mask);
}
};
```

##### Refill algorithm for compressed vectors

```c++
struct fill_cc {
__mmask8 permutation_mask;
__m512i permutation_idxs;
uint32_t cnt;
//Prepare the permutation.
fill_cc(const uint32_t src_cnt,
const uint32_t dst_cnt) {
const auto src_empty_cnt = LANE_CNT - src_cnt;
const auto dst_empty_cnt = LANE_CNT - dst_cnt;
//Determine the number of elements to be moved.
cnt = std::min(src_cnt, dst_empty_cnt);
bool all_fit = (dst_empty_cnt >= src_cnt);
auto d = all_fit ? dst_cnt : src_empty_cnt;
const __m512i d_vec = _mm512_set1_epi64(d);
//Note: No compress/expand instructions required
permutation_idxs =
_mm512_sub_epi64(SEQUENCE, d_vec);
permutation_mask = ((1u << cnt) - 1) << dst_cnt;
}
//Move elements from ’src’ to ’dst’.
void apply(const __m512i src, __m512i& dst) const{
dst = _mm512_mask_permutexvar_epi64(
dst, permutation_mask, permutation_idxs, src);
}
void update_src_cnt(uint32_t& src_cnt) const {
src_cnt -= cnt;
}
void update_dst_cnt(uint32_t& dst_cnt) const {
dst_cnt += cnt;
}
};
```

上面的两个算法涵盖了2个极端情况

* 活跃元素存储在随机位置
* 活跃元素存储在连续位置

根据src & desc的不同情况，总共存在4种不同的算法(2*2),每种算法又有2种变体

* 源寄存器中的所有元素都保证适合目标寄存器
* 不是所有元素都可以移动，因此元素仍然保留在源寄存器中

## 5. Refill strategies

refill algorithms与数据查询管道集成时，会将查询算子管道转化为一个for-loop，相关的算子封装到consumer() 与 producer()两个函数中。

使用SIMD进行数据处理时，会在每个控制运行时tuple的operator中插入一个校验逻辑，比如一个if-statements。只有SIMD register被填充完整后，才会执行算子逻辑

这个判断逻辑有两个基本的策略

#### 5.1 Consume everything

引入一个register作为buffer，利用不足时(<THRESHOLD),缓存数据，并在后续迭代中，应用给未利用的lanes，保证lanes的使用效率

伪代码

```c++
[...]
auto active_lane_cnt = popcount(mask);
if (active_lane_cnt + buffer_cnt < THRESHOLD
    && !flush_pipeline) {
  [...]//Buffer the input.
}
else {
  const auto bail_out_threshold = flush_pipeline ? 0: THRESHOLD;
  while (active_lane_cnt + buffer_cnt >
         bail_out_threshold) {
  if (active_lane_cnt < THRESHOLD) {
[...]//Refill lanes with buffered elements. }}
    //===---------------------------------===//
    //The actual operator code and
    //consume code of subsequent operators.
    [...]
    //===---------------------------------===//
     active_lane_cnt = popcount(mask);
  }
  if (likely(active_lane_cnt != 0)) {
    [...]//Buffer the remaining elements.
  }
}
//All lanes empty (consume everything semantics).
mask = 0;
[...]
```

#### 5.2 Partial consume

当lanes利用不足时，Consumer code延迟执行，重新执行上一算子，并将active lanes设置为保护状态(防止被覆盖)

伪代码

```c++
[...]
auto active_lane_cnt = popcount(mask);
if (active_lane_cnt < THRESHOLD && !flush_pipeline){
 //Take ownership of newly arrived elements.
 this_stage_mask = mask ^ later_stage_mask;
}
else {
 //===---------------------------------===//
 //The actual operator code and
 //consume code of subsequent operators.
 [...]
 //The later_stage_mask is set by the
 //consumer.
 //===---------------------------------===//
//Protect lanes in the preceding operator.
mask = this_stage_mask | later_stage_mask;
[...]
```

![image-20240806003249129](/assets/images/image-20240806003249129.png)

图a：控制流分歧未优化时的SIMD处理情况

图b：紫色是指，由于利用率较低，将数据存储到buffer中，延迟执行，等黑色线时，通过refill算法填充lanes，执行算子

图c:  紫色为保护状态，Divergent下，t=7时，有4个lanes(0,1,5,7)会执行out算子, partial 判断利用率太低，对这四个lanes进行保护，并从scan 算子开始重新执行，填充其他lanes，直到填充数量满足利用率



对于partial consume，他的利用率并不是很高，尤其是当protected lanes之前的算子

#### 5.3 Discussion and implications

两个策略可以相互之间组合使用，并非互斥角色。甚至在某些特定算子中，Divergent状态下性能会更好。具体如何设计不再本文中阐述

consume everything & partial consume各有利弊

* consume everything受限与buffer的影响，如果需要缓存的数据过多，会发生spilling，进而降低了性能
* partial consume 受限于 query pipeline的长度，较短的算子流，有利于它的高效，而且不受数据大小的影响

## 6. Evaluation

性能分析使用下面三种算子：包含table scan的predicate evalution、hash join、an approximate geospatial join

#### 6.1 Table scan

![image-20240806201653365](/assets/images/image-20240806201653365.png)

7a：

1. SKX的向量与标量性能很相近，即在IPC, branch prediction, and out of order execution等方面表现很好

2. materialization在各个阶段的性能表现的一直很好，buffer的方式会无限接近materialization的性能

7b：

1.  (Selectivities > 0.001) divergent SIMD implementation slow than the scalar implementation 
2. 本文的优化算法会有效的提升SIMD的性能

上面的指标是通过优化了threshold的partial&buffered，以及合理buffer size的materialization，下图表示了在固定selectivity=0.01下上面指标的影响

![image-20240806203230830](/assets/images/image-20240806203230830.png)

#### 6.2 Hashjoin

查询hash table是控制流分歧性能问题的主要来源

Divergent：SIMD同时处理8条记录，使用bitmask跟踪非活跃记录

Partial/Buffered： 会将整个k/v在整个pipeline中进行传递

Materialization：算子之间维护一个buffer，在整个pipeline压缩记录

![image-20240806224953452](/assets/images/image-20240806224953452.png)

* HashTable的size（build input size） 分别大于*L* 2 cache at around 1 MiB，*L* 3 cache around 10 MiB，性能都有明显的降低
* 在L1、L2 cache范围内，SIMD > scalar
* 超过L1、L2 cache，scalar ~ SIMD(10 thread), scala > SIMD (20 thread)
* 20 thread:buffered approach >  divergent approach = 32% ，  partial approach > divergent approach = 19% 
* L1 cache buffered approach > materialization approach = 8%
* 超过L1 cache，materialization approach 性能最好

![image-20240806225947603](/assets/images/image-20240806225947603.png)

* 20 thread 的性能比10 thread高出50%
* higher threshold 更高的性能
* buffered approach 对threshold更敏感
* A buffer size between 128 and 1024 results in the best performance of the materialization approach

其他影响因素

* *match probability*
* *load factor*

![image-20240806230546706](/assets/images/image-20240806230546706.png)

fig a:

* buffered approach 的性能最高
*  low match probabilities 时，scalar approach 性能非常有竞争力
  * 能够尽早的结束pipeline
  * divergent approach性能最差
* scalar approach performs best for low match prob- abilities，worst for a match probability of around 50%，cannot fully recover its throughput even for a match prob- ability of 100%
* buffered and partial approaches  在high match probabilities 下能明显的提升性能

fig b

* load factor = key / total number
* low load factor worse VPU utilization in the divergent approach
* high load factors  the scalar approach performs well
* high load factors like 4, there is little difference between all approaches 

#### 6.3 Approximate geospatial join

算法分两个阶段

* check for a *common prefix* of the query point and the indexed cells
* traverse the tree starting from the root node until we hit a leaf node 

##### 6.3.1 Query pipeline

- [x] Scan point data (source)
- [x] Prefix check
- [x] Tree traversal
- [ ] ~~Output point-polygon pairs (sink)~~

##### 6.3.2 Results

![image-20240806234323041](/assets/images/image-20240806234323041.png)

* boroughs or neighborhood  场景下，算法能提升20%的性能
* 在census场景下， divergence 会导致10%的性能损耗
* 正如5.3，大多数场景，partial consume strategy会让divergence 恶化，最坏情况下有53%的损失
* KNL下，materialization approach 性能较差，性能与scalar相似，
* 在SKX有更好的性能，与buffered pipeline相近
  * 小数据性能差，大数据性能好
  * 原因是数据量大了，hide memory latencies through out-of-order execution
* buffered approach最佳threshold 小于SIMD lanes
  * tree traversal need refilling affects five vec- tor registers,
  * hash join experiment, refilling affects three registers
* optimal threshold for the partial approach is 1，从数据源refill的效率并不高

考虑到这里涉及到2个阶段的SIMD过程，文中增加一个组合算法测试的场景，即stage2 与 stage3 使用的算法不同

![image-20240806235755680](/assets/images/image-20240806235755680.png)

* lower selectivities 下 refill算法有30%的性能提升

#### 6.4 Overhead

验证了返回字段多少对性能的影响

![image-20240807000709263](/assets/images/image-20240807000709263.png)

* the partial consume approach shows no per- formance impact when the number of projected attributes increases
* the materialization approach shows an increasing overhead with an increasing number of attributes
* 整体来说，refill维持较稳定的性能

## 7 Summary and discussion

* partial consume strategy 在简单的workload，有很高效的表现，而复杂的workload会导致性能降低
* materialization approach is very sensi- tive to the underlying hardware
* The SIMD lane utilization threshold对buffered approach 影响较大，对partial影响较小

## 8 Conclusions