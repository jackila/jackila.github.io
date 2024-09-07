[**Don’t Hold My Data Hostage – A Case For Client Protocol Redesign**](https://15721.courses.cs.cmu.edu/spring2024/papers/12-networking/p1022-muehleisen.pdf)

## ABSTRACT

Transferring **a large amount of data** from a database to **a client program** is a surprisingly **expensive operation**

1. **explore and analyse** the **result set serialization** design space

   * present **experimental results** from a large chunk of the database market

   * show the **inefficiencies** of current approaches

2. propose a **columnar serialization method**

   * improves transmission performance by **an order of magnitude**

## 1. INTRODUCTION

Result set serialization（RSS）的传输问题 在数据科学中经常遇见，比如 采用 `SELECT * FROM lineitem`将所有数据传输到一些机器学习工具中，进行统计、计算、分析工作。常见的解决方式是`performing the computations in the database`避免`exporting the data`,但他的实现与优化难度很高

本文分析了现有的`serialization formats`

* how they perform when transferring various data sets in **different network scenarios** 
* examine why they show this performance

本文的主要贡献如下

* We **benchmark** the result set serialization methods used by **major database systems**, and measure how they perform when transferring large amounts of data in **different network environments**
  * how these methods perform 
  * what the deficiencies of their designs 
* We **explore the design space** of result set serialization and **investigate numerous techniques** that can be used to create an efficient serialization method
  * benchmark these techniques
  * discuss their advantages and disadvantages
* We **propose** a new **column-based serialization method** that **is suitable for** exporting large result sets
  * performs an order of magnitude better 

## 2. STATE OF THE ART

C-S模型

![image-20240907124756688](/assets/images/image-20240907124756688.png)

1. the **server** has to **serialize the data** to the result set format
2. the converted message has to be **sent over the socket** to the client
3. the client has to **deserialize the result set** so it can use the actual data

### 2.1 Overview

测试时的一些默认设置

* use the **ODBC client** connectors for each of the databases

  > For Hive, we use the JDBC client

* use the lineitem table of the TPC-H benchmark

* Both the server and the client ran on the same machine

* measured after a “warm-up run”

* using the netcat (nc) [12] utility as baseline

![image-20240907123811955](/assets/images/image-20240907123811955.png)

* the **dominant cost** of this query is the cost of **result set (de)serialization** and **transferring the data**

![image-20240907123719005](/assets/images/image-20240907123719005.png)

* **none of the systems** come close to the performance of our baseline
* the compressed version of the MySQL client protocol transferred **the least amount of data**
* MongoDB requires transferring ca. **six times the CSV size**
* most systems with an **uncompressed protocol** transfer **more data** than the CSV file,but not an order of magnitude more

### 2.2 Network Impact

核心指标：bandwidth & latency

##### Latency

![image-20240907125513101](/assets/images/image-20240907125513101.png)

理论上， **the transfer** of a large result set **should not be influenced** by the **latency** as the server can send the entire result set **without needing to wait** for any confirmation

但是，higher latencies have on the different protocols

* both DB2 and DBMS X perform significantly worse

  > send explicit **confirmation messages** from the client to the server
  >
  > the messages are **cheap** with a low latency, but become very **costly** when the latency increases

* performance of all systems is heavily influenced by a high latency

  > the underlying **TCP/IP layer** does send acknowledgement messages when data is received

**Throughput**

The more we restrict the throughput, the more protocols that send a lot of data are penalized

![image-20240907130154928](/assets/images/image-20240907130154928.png)

* When the **bandwidth is reduced**, protocols that send **a lot of data** start **performing worse** than protocols that **send a lower amount of data**

* the throughput decreases **compression** becomes more effective

### 2.3 Result Set Serialization

#### PostgreSQL

| INT32       | VARCHAR10 |
| ----------- | --------- |
| 100,000,000 | OK        |
| NULL        | DPFKG     |

![image-20240907130945630](/assets/images/image-20240907130945630.png)

组成：total length、the amount of fields、 each field its length (−1 if the value is NULL) 、the data

* the amount of per-row **metadata is greater** than the actual data w.r.t. the amount of bytes
* a lot of information is **repetitive and redundant**
  * the **amount of fields** is constant for an entire result set
  * the message type marker redundant
* requires so **many bytes**  but **low serialization and deserialization costs**

#### MySQL



![image-20240907133316817](/assets/images/image-20240907133316817.png)

组成：three-byte data length、a packet sequence number (0-256, wrapping around)、 Field lengths、Field data

> * Field lengths are encoded as **variable-length integers**. **NULL values** are encoded with a special field length,0xFB
>
> * Field data is transferred in **ASCII format**

* **binary encoding** for metadata, and **text** for actual field data
* The **number of fields** in a row is constant and defined in the **result set header**
* The **sequence number is redundant** 

#### DBMS X

![image-20240907135345183](/assets/images/image-20240907135345183.png)

组成：packet header、length in bytes、the values

> This length is transferred as a **variable-length integer**

* On a lower layer, DBMS X uses a **fixed network message length** for batch transfers, which can be configure

#### MonetDB

![image-20240907135955898](/assets/images/image-20240907135955898.png)

* **text-based** result serialization format,**the ASCII representations** of values are transferred
  * 避免了：endian-ness、transfer of leading zeroes、variable-length strings
* Values are **delimited** similar to CSV files
  * Missing values are encoded as the **string literal NULL**

* tabs and spaces is historic reasons
* While it is simple, converting the **internally used** binary value representations to strings and back is an **expensive operation**

#### Hive

![image-20240907140551996](/assets/images/image-20240907140551996.png)

* Hive and Spark SQL use a **Thrift-based protocol** to transfer result sets

* Thrift contains a serialization method for **generic complex structured messages**
  * due to the **columnar nature** of the format, **these overheads** are not dependent on the number of rows
  * The only per-value overheads are the lengths of the **string values** and the **NULL mask.**
* Hive performs **very poorly** on our benchmark
  * expensive **variable-length encoding** of **each individual value** in **integer columns**

### 3.PROTOCOL DESIGN SPACE

#### 数据集特点

##### lineitem

* be similar to **real-world data warehouse fact tables**
* It contains 16 columns, with the types of either **INTEGER, DECIMAL, DATE and VARCHAR**
* **no missing values**
* 60 million rows and is 7.2GB in CSV format

##### American Community Survey (ACS)

* consists of 274 columns
* majority of type INTEGER
* 16.1% of the fields contain missing values
* 9.1 million rows、

##### Airline On-Time Statistics

* The most frequent types in the 109 columns are **DECIMAL and VARCHAR**
* 55.2% of the fields contain missing values
* 10 million rows、3.6GB in CSV format

### 3.1 Protocol Design Choices

#### Row/Column-wise

vector-based protocol：  chunks of rows are encoded in column-major format

#### Chunk Size

![image-20240907161117652](/assets/images/image-20240907161117652.png)

* For each dataset, the protocol **performs poorly** when the chunk size **is very small**
* the performance is **optimal** when the chunk size is **around 1MB**

#### Data Compression

make different trade-offs in terms of the **(de)compression costs** versus the achieved **compression ratio**

![image-20240907165053352](/assets/images/image-20240907165053352.png)

* even when using **generic, data-agnostic compression methods** the **column-wise files** always compress significantly **better**
* The best compression method **depends on** how expensive it is to **transfer bytes**
  * on a **fast network connection** a **lightweight** compression method
  * On a **slower network** connection use **heavyweight** compression

network connection with **different throughput limitations**

![image-20240907165513852](/assets/images/image-20240907165513852.png)

* **not compressing** the data **performs best** when the server and client are located **on the same machine**
* **Lightweight compression** becomes worthwhile when the server and client are **using a gigabit or worse connection**
* **only when** we move to a very **slow network connection (10Mbit/s)** that heavier compression performs better than lightweight compression
  * Even in this case, however, **the very heavy XZ** still performs **poorly**。because it **takes too lon**g to **compress/decompress the data**

so best compression method **depends entirely on the connection speed** between the server and the client

* choose to use a **simple heuristic** for determining which compression method to use
  * same machine，we **do not use any compression**
  * Otherwise, we use **lightweight compression**

#### Column-Specific Compression

Using these **specialized compression algorithms** we could achieve a **higher compression ratio** at a **lower cost** than when using data-agnostic compression algorithms

* **run-length encoding** or **delta encoding** could be used on **numeric columns**
* have **statistics** on a column which would allow for **additional optimizations in column compression**

**Integer values** in particular can be compressed at a very **high speed** using **vectorized binpacking** or **PFOR [21] compression algorithms**

![image-20240907170809863](/assets/images/image-20240907170809863.png)

* For the **lineitem table**, we see that **both PFOR and binpacking** achieve a **higher compression ratio** than **Snappy** at a **lower performance cost**
* perform **better than Snappy** in all scenarios
* **not compressing performs better** in the local- host scenario
* When **transferring the ACS dataset** the column-specific compression methods perform significantly **worse than Snappy**
* **PFOR compression algorithm** performs significantly **better than binpacking** on the ACS data
* **Both specialized compression algorithms** perform very **poorly** on the ontime dataset

we have chosen **not to use column-specific compression algorithms**

#### Data Serialization

TCP 数据常见的三种方式

* custom text/binary serializations
* generic serialization libraries ：Protocol Buffers、Thrift

the closer the serialized format is to the n**ative data storage layout**, the **less the computational overhead** required for their (de)serialization

![image-20240907183454916](/assets/images/image-20240907183454916.png)

* **custom result set serialization format** > protobuf serialization
* protobuf messages are smaller than our custom format

As a result of these high serialization costs, we have chosen to use a **custom serialization format**

#### String handling

There are **three main options** for character transfer

* **Null-Termination**, where every string is **suffixed with a 0 byte** to indicate the end of the string
* **Length-Prefixing**, where every string is prefixed with its length
* **Fixed Width**, where every string has a fixed width as described in its SQL type

![image-20240907184104258](/assets/images/image-20240907184104258.png)

* a fixed-width representation performs extremely well while transferring a small string column

![image-20240907184136694](/assets/images/image-20240907184136694.png)

* the fixed-width approach transfers a significantly higher num- ber of bytes

* unnecessary padding can have on performance in the worst case

结论

* we have chosen to conservatively **use the fixed-width** representation only when transferring columns of **type VARCHAR(1)**
* others，we use the **null-termination method** because of its better compressibility

## 4.IMPLEMENTATION & RESULTS

### 4.1 MonetDB Implementation

![image-20240907184741743](/assets/images/image-20240907184741743.png)

核心内容

* Each chunk is **prefixed by the amount of rows** in that particular chunk
* the columns of the result set follow in the same order as they appear **in the result set header**
* Columns with **fixed-length types**, such as four-byte integers, **do not have any additional stored** before them.
* Columns with **variable-length types**, such as VARCHAR columns, are **prefixed with the total length of the column** in bytes
* **Missing values** are encoded **as a special value** within the domain of the type being transferred
  * 2_31 is used to represent **the NULL value** for four-byte integers
* The maximum size of the chunks is **specified in bytes**

### 4.2 PostgreSQL Implementation

![image-20240907185659918](/assets/images/image-20240907185659918.png)

* missing values are encoded differently,each column is prefixed with **a bitmask** 
* we store the bitmask per column, can ignore some time
* convert row-major format to columnar result set format
* the **cost of this random access** pattern is mitigated because the chunks are small and generally **fit in the L3 cache of a CPU**

### 4.3 Evaluation

网络描述

Local：The server and client reside on the **same machine**

LAN Connection：using a **gigabit ethernet** connection with **1000 Mb/s throughput** and **0.3ms latency**

WAN Connection：**100 Mbit/s throughput**, **25ms latency** and **1.0% uniform random packet loss**

#### Lineitem

![image-20240907190658618](/assets/images/image-20240907190658618.png)

* our **uncompressed protocol** performs **best** in the **localhost scenario**, and **our compressed protocol** performs the **best** in the **LAN and WAN scenarios**.
* implementation in **MonetDB** performs **better** than the implementation in **PostgreSQL**.
* Our columnar representation transfers less data，than DBMS X
*  the timings for **MySQL with compression** **do not change significantly** when **network limitations** are introduced
  * the time spend **compress- ing the data dominates the data transfer time**
  * our compressed protocol transfers less data
* The performance of our uncompressed protocol **degrades significantly** when **network limitations** are introduced because it is **bound by the network speed**
* DBMS X and DB2 degrade significantly when even more network limitations are introduced
  * both have explicit **confirmation** messages

#### ACS Data

When transferring the ACS data, we again see that **our uncompressed protocol performs bes**t in the localhost scenario and the **compressed protocol performs best** with network limitations

#### Ontime Data

## 5.CONCLUSION AND FUTURE WORK

We then evaluated our protocol against the state of the art protocols on **various real-life data sets,** and found **an order of magnitude faster performance** when exporting large datasets

优化方向

* use the network speed as a heuristic for which compression method to use
* use different columnar compression methods depending on the data distribution within a column
* serializing the result set in parallel