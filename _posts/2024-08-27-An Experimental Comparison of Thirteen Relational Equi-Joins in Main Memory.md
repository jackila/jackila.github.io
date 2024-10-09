---
tags: 论文
---
[**An Experimental Comparison of Thirteen Relational Equi-Joins in Main Memory**](https://15721.courses.cs.cmu.edu/spring2024/papers/09-hashjoins/schuh-sigmod2016.pdf)

# ABSTRACT

鉴于不同文章对不同的equi-joins算法的不同评价，本文尝试回答一个常见的问题

#### what is the fastest join algorithm in 2015

* an end-to-end black box comparison of the most important methods
* inspect the internals of these algorithms in a white box comparison
* Improve join algorithms by optimizations 
  * software- write combine buffers
  * various hash table implementations
  * NUMA-awareness (data placement  & scheduling)
* inspect various radix partitioning strategies

#### factor in scaling effects 

* size of the input datasets
* number of threads
* different page sizes
* data distributions

> there is no single best join algorithm. Each algo- rithm has its strength and weaknesses and shines in different areas of the parameter space

# 1.INTRODUCTION

不同研究，矛盾的结论

* [4]： hash-based approach > sort based approaches,
* the PRB-family  vs the NOP-family
  * 2011 [7] :  NOP > PRB
  * 2013  [4]:  PRB > NOP
  * 2013 [14]:  PRB < NOP

上述论文之间冲突的原因

* different implementations
* different optimizations
* different performance metrics
  * number of join results produced per second
  * the sum of the relation sizes （this paper）

* different machines
* difference of micro-benchmarks and real queries

本文的一些默认设置

* 更加关注 hash-based join algorithms
  * 大量的研究表明 sort-based approach 的性能不够好
* the context of a modern NUMA architecture
* 大部分算法基于：[17, 4, 14, 5]这几篇论文

本文的核心贡献

* Black box Comparison
* White box Comparison
* Optimizing Join Algorithms
* Comprehensive Comparison of Joins
* Effect on real queries
* Key Lessons Learned

# 2.RELATED WORK

* [13]  comparing a hash join and a sort-merge join
  * parallel radix hash join outperforms the sort-merge join by a factor of two
  * sort-merge joins can be optimized
    * *Wider SIMD instructions* with scaled near-linearly 
    * *Limited per-core bandwidth*
      * hash join algorithm  has limited number of TLB cache entries and  three trips to main memory
      * sort-merge join only needs two
* [7] introducing no-partitioning hash join
  * no-partitioning hash join outperforms all partition-based hash joins for almost all data distributions
* [3] sort-merge join 、focus on optimizing for NUMA systems
  * uses a carefully tuned memory access pattern 
  * avoids inter-thread synchroniza- tion as much as possible
  * faster than no-partitioning hash join [7] and radix hash join [15]
* [5]  variants of parallel radix hash join and no-partitioning hash join algorithms
  * improving the cache efficiency of the hash table implementations for all
  * a better skew handling mechanism for parallel radix hash join
  * parallel radix hash join outperforms the no-partitioning hash join
* [14] NUMA-aware no-partitioning hash join
  * their no- partitioning hash join method outperforms the parallel radix hash join 
* [4] improved the sort-merge join and their own parallel radix join 
  *  sort-merge join 
    * uses wider SIMD instructions 
    * uses range partitioning to allow for efficient multithreading without heavy syn- chronization
  * parallel radix hash join still always outperforms sort-merge join
*  [17] studied the memory-efficiency of hash join methods
  * proposed a highly memory efficient linear probing hash table called concise hash table
  * reduce the memory usage by one to three orders
* others
  * GPUs
  * hybrid CPU-GPU archi- tectures
  * coprocessors attached through PCI express cards like Intel’s Xeon Phi

# 3. FUNDAMENTAL REPRESENTATIVES OF MAIN-MEMORY JOIN ALGORITHMS

常见的三种join algorithms算法

* partition-based hash joins
* no-partitioning hash joins
* sort-merge joins

![image-20240824170603537](/assets/images/image-20240824170603537.png)

## 3.1 Partition-based Hash Joins

#### 核心思想

* 将两个数据源，分割成多个分区，组成一对对的联合分区
* 尽可能的其中一个分区适合cache大小

目标是：*minimize the number of cache misses*

问题：excessive TLB misses

##### PRB

> uses two-pass partition- ing to guarantee that the number of partitions does not exceed the number of TLB entries

##### The first partitioning pass

1. assigning equal-sized regions (chunks) to each thread

   > 将数据集分成大小相同的区域

2. precomputes the output memory ranges of each target partition by building histograms
   * each thread knows where and how much to write for each partition without the need for further syn- chronization

3. each thread scans the input relation and writes each tuple to its destination region
4. produces a considerable number of partitions

#####  the second partitioning pass

5. entire sub-partitions are assigned to worker threads by using a task queue

##### In the join phase

6. each thread takes one co-partition at a time
7. runs a textbook hash join algorithm on it using a chained hash table

## 3.2 No-partitioning Hash Joins

#### 核心思想

* 构建一个全局的hash table
* 并发多线程、 *out-of-order execution* 
  * *hide cache miss penalties*

> no need to tune *hardware cache sizes or number of TLB entries* 

#### NOP

* global linear probing hash table 

  * lock-free synchronization mechanism （compare-and-swap）

* assigning equal-sized regions (chunks) to each thread

  > 将数据集分成大小相同的区域

* Each thread then inserts its chunk of the build relation into the global hash table

* each thread starts probing its chunk of the probe relation

##### 优化

* each thread uses an atomic Compare-and-Swap (CAS) operation

* interleave hash table allocation among all available NUMA nodes for better memory bandwidth utilization

* 使用CHTJ

  * four major components
    * an array *A* of size *n*
    * a hash function *hash* 
    *  a bitmap *B* of size
    * a population count array *PC* of size *n*/4 with a running sum of the population count of the bitmap
      记录每个32位的bitmap的实例数量
  * 如何查找一个元素
    * 先根据bitmap 查看是否存在记录：*B*(*hash*(*key*)) 
    * 根据PC数据结构获取，当前的实例PC之前的总数：x= ⌊*hash*(*key*)/32⌋ − 1 
    * 遍历bitmap，从⌊*hash*(*key*)/32⌋·32，一直到*hash*(*key*)−1，就能计算出 差值 y = [⌊*hash*(*key*)/32⌋·32，*hash*(*key*)−1]
    * 则返回记录 array[x+y]

  * CHTJ的使用
    * radix-partitions the build input into a small number of partitions very similar to PRB
    * global CHT is allocated where each thread bulkloads its partition to a disjoint region in that CHT
    * the probe relation is handled similarly to NOP: each thread probes one chunk of the probe rela- tion against the global CHT

## 3.3 Sort-merge Joins

#### 核心思想

将数据进行排序，然后通过merge的方式查询所有匹配的记录。虽然单个sort merge join比hash join的性能差，但是对于一个复杂的join查询，会有更好的性能

#### 优化

* MWAY is the m-way sort merge join
  * partitions the data very similar to PRB
    * using only a single partition- ing phase
    *  creating only few partitions
* software write-combine buffers (see Section 5.1) are used
* After partitioning, each partition will be merge-sorted independently by a sepa- rate thread
* implemented with bitonic sorting- and merge-networks
* Both sort- and merge-networks are vector- ized using SIMD instructions
* multi-way merging is used to save memory bandwidth

# 4.BLACK BOX COMPARISONS

![image-20240825125212073](/assets/images/image-20240825125212073.png)

> * this:proposed in this paper
> * Original:  use the implementation from authors of the paper
> * “Modified” means we modify the orig- inal implementation to get the new variant
> * “Own” means we implemented the algorithm from scratch

![image-20240825125128055](/assets/images/image-20240825125128055.png)

# 5. WHITE BOX COMPARISONS

在黑盒测试中parallel radix join的性能存在疑问，下文提出了一些优化方式，提升性能

## 5.1 Optimizing Radix Partitioning

#### NUMA-Awareness

> -basic-numa flag

* allocates the partition buffer on all NUMA nodes

* Memory Allocation Locality

#### Software Write-Combine Buffers.

> -enable-swwc-part

通过将多个tuple写入同一个cache line中，等cache line满了，一起flush，伪代码如下

```c++
for all tuple ∈ relation do
	partition ← hash(tuple.key);
	pos ← slots[partition] mod TuplePerCacheline; slots[partition]++;
	buffer[partition].data[pos] = tuple;
	if pos == TuplePerCacheline - 1 then
		dest ← slots[partition] - TuplePerCacheline; 
		copy buffer[partition].data to output[dest];
```

通过这种方式，可以有效的降低TLB missing

* Non-temporal streaming

> write half a cache line to DRAM bypassing all caches

通过优化上面的特性，我们获得一个更高效的算法，PRO

另外还可以通过将two-pass radix- partitioning修改为Single-pass Partitioning的方式，获得更高的性能

![image-20240825131030677](/assets/images/image-20240825131030677.png)

## 5.2 Choice of Hash Method

常见的三种方式

* Linear
  * NOP 
  * PRL
* Chained
  * PRO 
* CHT
  * CHTJ
* Arrays
  * NOPA
  * PRA

#### Performance Comparison

*  PRO  >  NOP 
*  PRA, PRO, and PRL 似乎性能相同，但是下文会给出一个相反的结论

![image-20240825131613109](/assets/images/image-20240825131613109.png)

>  parallel radix partition- ing joins PR* are providing the highest throughput 

# 6. OPTIMIZING PARALLEL RADIX JOIN

针对parallel radix partition- ing joins 分别从partition phase and the join phase 进行优化

## 6.1 NUMA-aware Partitioning

![image-20240825141323016](/assets/images/image-20240825141323016.png)

#### The Parallel Radix Partitioning algorithm【Figure 4(a)】

> 类似 Grace Hash Join ，只不过是在NUMA级别上处理的

1. every thread sequentially reads a horizontal chunk of the input relation to create a **local histogram**
2. a global thread merges the local histograms into a global histogram
3. each thread reads again its horizontal local chunk of the input relation and partitions the data into its corresponding SWWCB

##### For the probe relation

*  (1)–(3) are executed similarly using the same partitioning function
* each pair of corresponding parti- tions is joined independently

#### Chunked Parallel Radix partition (CPRL) algorithm【Figure 4(c)】

1. the histogram phase (1) is the same as in PRO

2. proceed directly with phase (3)
   * each thread partitions its data locally within its chunk only based on its local histogram

#####  probe relation phases

* (1)–(3) are executed similarly using the same partitioning function

 each pair of corresponding par- titions, i.e. each co-partition, is joined independently

* neither have physically contiguous probe nor build partitions available
* read the different chunks belonging to the build input
  * directly load that data into a local hash table
  * read the different chunks belonging to the probe input from its (possibly NUMA-remote) sources and probe them directly against the hash table

特点： CPRL trades small random writes to remote memory for large sequential reads from remote memory

hash table 区别

*  CPRL ： linear probing hash table
* CPRA： arrays

![image-20240825143952884](/assets/images/image-20240825143952884.png)

* CPR*-algorithms outperform the PR*-algorithms by ∼20%
* the partitioning times of the CPR*- algorithms are indeed reduced as expected
* the join time is reduced

## 6.2 NUMA-aware Scheduling

All PR*- and CPR*-algorithms build co-partitions which eventually have to be joined independently

* How are those individual joins scheduled
* What effect does this schedule abort performance

#### how

* all co- partitions are put into a LIFO-task queue
*  inserted into the queue in ascending sequential in- dices order
* one quarter of each input relation is physically allocated on one of the NUMA-regions

这种方式导致的性能问题

* threads removing tasks from the queue will have to read their input data from **the same NUMA-region**

![image-20240825145653162](/assets/images/image-20240825145653162.png)

* 大部分时间， PRO uses only a single NUMA-region
* PROiS、PRLiS、PRAiS all NUMA nodes are used at the same time
* CPRL 基本不受到影响，every parti- tion has to be read from all NUMA nodes anyhow

解决这个问题的方案有2种

* insert co-partitions into the task queue in a round-robin manner.
* use a different queue for each NUMA-region

![image-20240825150221846](/assets/images/image-20240825150221846.png)

* The improved scheduling results in a speedup of the join phase of PRL and PRA by more than a factor of 2
* the join phase of the CPR*- algorithms 的性能比PR*iS-algorithms 要差
* 但是，in total the CPR*-algorithms are slightly faster than the PR*iS-algorithms
* we can now clearly observe that different hash table implementations have an effect on the runtime of the algorithms

# 7.PUTTING IT ALL TOGETHER

### 7.2 Varying Page Sizes

>  evaluate pages of 4 KB and 2 MB 

![image-20240826214408770](/assets/images/image-20240826214408770.png)

* all algorithms except PRB improve by using huge pages

### 7.3 Scalability in Dataset Size

#### two workloads

* the probe relation is ten times the size of the build relation
* both relations have the same size

#### Fine-tuning the partition-based joins.

*  radix-based algorithms are very sensitive to the number of bits used for partitioning

> 猜想：doubling the data size, we take one additional bit for partitioning

![image-20240826215034703](/assets/images/image-20240826215034703.png)

* the size of the software write-combine buffers
* choosing the number of bits such that the partitions fit into L2 cache is close to the optimal choice
* partitioning costs increase sharply
  * is better to balance the partitioning cost with the join costs

##### the minimal number of bits such that the partitions still fit into the shared LLC

### Predicting the optimal number of radix bits

![image-20240826224249483](/assets/images/image-20240826224249483.png)

#####  use Equation (1) to set the bits of all PR*- and CPR*-algorithms

![image-20240826230641018](/assets/images/image-20240826230641018.png)

* for very small input sizes, i.e. up to 4M tuples, the various algorithms show similar performance
* to larger sizes, we observe that the PR*- and CPR*-algorithms outperform the NOP*-algorithms, CHTJ, and MWAY.
* the bigger the build relation, the higher the probability for an LLC miss as well a TLB-miss

# 8.EFFECTS ON REAL QUERIES

![image-20240826231113492](/assets/images/image-20240826231113492.png)

##### a major part of the runtime is spent in the non-join parts of the query

* the join key column is a primary dense key and the Part table is even generated in sorted order according to this key
* scanning and filtering 600 M tuples from Lineitem down 21.42 M tuples simply eats up some time
* Q19 we have to access several attributes other than the join key

# 9. LESSONS LEARNED

* Don’t use CPR* algorithms on small inputs
* Clearly specify all options used in experiments
* If in doubt, use a partition-based algorithm for large scale joins
* Use huge pages
* Use Software-write combine buffer
* Use the right number of partition bits for partition-based algorithms
* Use a simple algorithm when possible
* Be sure to make your algorithm NUMA-aware
* Be aware that join runtime̸=query time

# 引用

[3]  Martina-CezaraAlbutiu,AlfonsKemper,andThomasNeumann.

Massively Parallel Sort-Merge Joins in Main Memory Multi-Core

Database Systems. *PVLDB*, 5(10):1064–1075, 2012.

[4]  CagriBalkesen,GustavoAlonso,JensTeubner,andMTamerOzsu. Multi-Core, Main-Memory Joins: Sort vs. Hash Revisited. *PVLDB*,7(1):85–96, 2013.

[5]  CagriBalkesen,JensTeubner,GustavoAlonso,andMTamerOzsu.

Main-Memory Hash Joins on Multi-Core CPUs: Tuning to the

Underlying Hardware. *ICDE*, pages 362–373, 2013.

[6]  CagriBalkesen,JensTeubner,GustavoAlonso,andMTamerOzsu.

Main-Memory Hash Joins on Modern Processor Architectures.

*TKDE*, 27(7):1754–1766, 2015.

[7] Spyros Blanas, Yinan Li, and Jignesh M Patel. Design and Evaluation of Main Memory Hash Join Algorithms for Multi-Core CPUs. SIGMOD, pages 37–48, 2011.

[14] Harald Lang, Viktor Leis, Martina-Cezara Albutiu, Thomas Neumann, and Alfons Kemper. Massively Parallel NUMA-aware Hash Joins. IMDM, pages 3–14, 2013.

[17] R Barber G Lohman I Pandis, V Raman R Sidle, G Attaluri N Chainani S Lightstone, and D Sharpe. Memory-Efficient Hash Joins. PVLDB, 8(4):353–364, 2014.