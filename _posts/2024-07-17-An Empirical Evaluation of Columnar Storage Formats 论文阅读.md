> [An Empirical Evaluation of Columnar Storage Formats](https://15721.courses.cs.cmu.edu/spring2024/papers/02-data1/p148-zeng.pdf) 论文阅读

### 0. 概述

文章认为十年前设计的Parquet与ORC在当前的硬件、工作负载下可能存在性能问题，于是通过系统化的比较两个编码格式，设计不同特征的数据集评估Parquet与ORC，分析列存的原理与性能，探索下一代列存格式需要解决的问题。

文章总结了3个结论

* 在当前硬件、数据特征下，对性能显著提升的设计，如字典编码、数值编码算法倾向于解码速度而不是压缩比，可选的压缩比选项、更精细的辅助数据结构
* 机器学习工作负载&GPU解码，格式设计的低效
* 下一代格式设计时需要考虑的因素

### 1. 简介

Parquet & ORC解决的问题

* 无关属性跳过
* 高效的数据压缩
* 矢量查询处理能力
* 统一数据格式、共享数据

新的挑战

* 存储介质每秒千兆的性能
* 数据湖下存储数据到S3等对象存储，高宽带高延迟特点
* 新的算法实现：轻量级的压缩算法、索引、过滤等技术

本文的内容

* parquet&orc的格式、设计分析
* 一个新的基准测试
* 对核心组件的全面分析：编码、块压缩、元数据组织、索引和过滤，嵌套数据
* 机器学习的性能问题&GPU下的性能分析

本文的主要贡献

* 为 Parquet 和 ORC 等列式存储格式创建了特征分类法。
* 设计了一个基准测试来对这些格式进行压力测试，并在不同工作负载下确定它们的性能与空间权衡
* 利用基准测试在 Parquet 和 ORC 上进行了一系列全面的实验，并总结了未来格式设计的经验教训。

### 2. 背景

![image-20240716231946452](/assets/images/image-20240716231946452.png)

### 3. 特征分类

![image-20240716232121625](/assets/images/image-20240716232121625.png)

#### 格式布局

通过将行数据切分成行组，按列进行存储，每个属性组成一个列块。利用矢量查询能力及降低了元组的重建成本。

同时都进行了编解码、压缩算法、页脚模块

|          | Parquet              | ORC        |
| -------- | -------------------- | ---------- |
| 行组大小 | 以行数为主（如100w） | 64M        |
| 压缩比   | 尽可能高的压缩能力   | 支持配置化 |
|          |                      |            |

![img](/assets/images/2024_07_15_94ad8f505ff391a297d0g-04.jpg)

![img](/assets/images/2024_07_15_94ad8f505ff391a297d0g-04-20240717070236585.jpg)

#### 编码

减少存储和网络成本，常见方式：字典编码（Dictionary Encoding）、游程编码（RLE）和位填充（Bitpacking）

|          | Parquet                           | ORC                |
| -------- | --------------------------------- | ------------------ |
| 字典编码 | 所有列                            | 字符串列           |
| 字典大小 | 1M                                | NDV阀值            |
| 整数列   | 字典编码--> RLE & Bitpacking (>8) | 四种算法，贪婪使用 |

#### 压缩

文章提出：块压缩应用于列存储格式对现代硬件的端到端查询速度没有帮助

* 默认开启
* 压缩算法对类型不可知，以字节流统一处理
* 提供压缩比 & 压缩\解压速度的设置能力

#### 类型系统

|          | Parquet                  | ORC                        |
| -------- | ------------------------ | -------------------------- |
| 基础类型 | INT32, FLOAT, BYTE_ARRAY |                            |
| 其他类型 | 由基础类型实现           | 单独实现                   |
| 复杂类型 | Struct、List 和 Map      | Struct、List 、 Map、UNION |

#### 索引&过滤

区域映射（zone map）

* 最小值、最大值和行计数,
* 有利于谓词下沉
* physical page & 10000行
* PageIndex 解决 parquet中`页面头与每个页面共存且在存储上是不连续的`

Bloom Filters
不同厂商的选择不同

```
According to our survey, Arrow and DuckDB only adopt zone maps at the row group level for Parquet, while InfluxDB and Spark enable PageIndex and Bloom Filters to trade space for better selection performance [46]. When writing ORC files, Arrow, Spark, and Presto enable row indexes but disable Bloom Filters by default.
```

####  Nested Data Model

![image-20240717073212114](/assets/images/image-20240717073212114.png)

### 4. 列存储基准

#### 列属性

* NDV Ratio
* Null Ratio
* Value Range
* Sortedness
* Skew Pattern
  * Uniform
  * Gentle Zipf
  * Hotspot
  * Single/Binary

#### 实际数据参数分布

![image-20240717073823960](/assets/images/image-20240717073823960.png)

#### 预定义工作负载

![image-20240717073914148](/assets/images/image-20240717073914148.png)

### 5.实验评估

#### 5.1 实验设置

使用将parquet&orc文件解压到一个固定大小的缓存区的方式，进行测试

#### 5.2 Benchmark Result Overview 

* 在文件大小方面，Parquet 和 ORC 之间没有明显的优胜者
* Parquet 在扫描方面比 ORC 更快
* 查询的平均延迟Parquet  > ORC

#### 5.3 Encoding Analysis 

5.3.1 Compression Ratio

5.3.2 Decoding Speed

#### 5.4 Block Compression

#### 5.5 Wide-Table Projection

#### 5.6 Indexes and Filters

#### 5.7 Nested Data Model

#### 5.8 Machine Learning Workloads

#### 5.9 GPU Decoding

### 6 LESSONS AND FUTURE DIRECTIONS

* 字典编码在各种数据类型（甚至浮点值）中都是有效的
* 在列格式中保持编码方案简单以确保高效的解码性能至关重要
* 查询处理的瓶颈正在从存储转移到现代硬件上的（CPU）计算
* 未来格式中的元数据布局应该是集中的，并且友好地支持随机访问
  * ML 训练的宽（特征）表
  * I/O 块的大小应该针对高延迟云存储进行优化
* 随着存储成本的降低，未来的格式可以承担更复杂的索引和过滤结构，以加快查询处理速度
* 嵌套数据模型应设计为与现代内存格式具有亲和性，以减少翻译开销

* 高效地支持宽表投影和低选择性选择
  * 机器学习工作负载
  * 更好的元数据组织和更有效的索引
  * 二进制数据的存储
* 未来的格式应考虑与 GPU 的解码效率

### 

