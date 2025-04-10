---
icon: logos:hbase
title: HBase 快速入门
date: 2020-02-10 14:27:39
categories:
  - 数据库
  - 列式数据库
  - hbase
tags:
  - 数据库
  - 列式数据库
  - 大数据
  - hbase
permalink: /pages/87503572/
---

# HBase 快速入门

## HBase 简介

### 为什么需要 HBase

在介绍 HBase 之前，我们不妨先了解一下为什么需要 HBase，或者说 HBase 是为了达到什么目的而产生。

在 HBase 诞生之前，Hadoop 可以通过 HDFS 来存储结构化、半结构甚至非结构化的数据，它是传统数据库的补充，是海量数据存储的最佳方法，它针对大文件的存储，批量访问和流式访问都做了优化，同时也通过多副本解决了容灾问题。

Hadoop 的缺陷在于：它只能执行批处理，并且只能以顺序方式访问数据。这意味着即使是最简单的工作，也必须搜索整个数据集，即：**Hadoop 无法实现对数据的随机访问**。实现数据的随机访问是传统的关系型数据库所擅长的，但它们却不能用于海量数据的存储。在这种情况下，必须有一种新的方案来**同时解决海量数据存储和随机访问的问题**，HBase 就是其中之一 (HBase，Cassandra，CouchDB，Dynamo 和 MongoDB 都能存储海量数据并支持随机访问)。

> 注：数据结构分类：
>
> - 结构化数据：即以关系型数据库表形式管理的数据；
> - 半结构化数据：非关系模型的，有基本固定结构模式的数据，例如日志文件、XML 文档、JSON 文档、Email 等；
> - 非结构化数据：没有固定模式的数据，如 WORD、PDF、PPT、EXL，各种格式的图片、视频等。

### 什么是 HBase

**HBase 是一个构建在 HDFS（Hadoop 文件系统）之上的列式数据库**。

HBase 是一种类似于 `Google’s Big Table` 的数据模型，它是 Hadoop 生态系统的一部分，它将数据存储在 HDFS 上，客户端可以通过 HBase 实现对 HDFS 上数据的随机访问。

![img](https://raw.githubusercontent.com/dunwu/images/master/snap/20200601170449.png)

HBase 的**核心特性**如下：

- **分布式**
  - **伸缩性**：支持通过增减机器进行水平扩展，以提升整体容量和性能
  - **高可用**：支持 RegionServers 之间的自动故障转移
  - **自动分区**：Region 分散在集群中，当行数增长的时候，Region 也会自动的分区再均衡
- **超大数据集**：HBase 被设计用来读写超大规模的数据集（数十亿行至数百亿行的表）
- **支持结构化、半结构化和非结构化的数据**：由于 HBase 基于 HDFS 构建，所以和 HDFS 一样，支持结构化、半结构化和非结构化的数据
- **非关系型数据库**
  - **不支持标准 SQL 语法**
  - **没有真正的索引**
  - **不支持复杂的事务**：只支持行级事务，即单行数据的读写都是原子性的

HBase 的其他特性

- 读写操作遵循强一致性
- 过滤器支持谓词下推
- 易于使用的 Java 客户端 API
- 它支持线性和模块化可扩展性。
- HBase 表支持 Hadoop MapReduce 作业的便捷基类
- 很容易使用 Java API 进行客户端访问
- 为实时查询提供块缓存 BlockCache 和布隆过滤器
- 它通过服务器端过滤器提供查询谓词下推

### 什么时候使用 HBase

根据上一节对于 HBase 特性的介绍，我们可以梳理出 HBase 适用、不适用的场景：

HBase **不适用场景**：

- 需要索引
- 需要复杂的事务
- 数据量较小（比如：数据量不足几百万行）

HBase **适用场景**：

- 能存储海量数据并支持随机访问（比如：数据量级达到十亿级至百亿级）
- 存储结构化、半结构化数据
- 硬件资源充足

> 一言以蔽之——HBase 适用的场景是：**实时地随机访问超大数据集**。

HBase 的典型应用场景

- 存储监控数据
- 存储用户/车辆 GPS 信息
- 存储用户行为数据
- 存储各种日志数据，如：访问日志、操作日志、推送日志等。
- 存储短信、邮件等消息类数据
- 存储网页数据

### HBase 数据模型简介

前面已经提及，HBase 是一个列式数据库，其数据模型和关系型数据库有所不同。其数据模型的关键术语如下：

- **Table** - HBase 表由多行组成。
- **Row** - HBase 中的一行由一个行键和一个或多个列以及与之关联的值组成。 行在存储时按行键的字母顺序排序。 为此，行键的设计非常重要。 目标是以相关行彼此靠近的方式存储数据。 常见的行键模式是网站域。 如果您的行键是域，您应该将它们反向存储（org.apache.www、org.apache.mail、org.apache.jira）。 这样，所有 Apache 域在表中彼此靠近，而不是根据子域的第一个字母展开。
- **Column** - HBase 中的列由列族和列限定符组成，它们由 :（冒号）字符分隔。
- **Column Family** - 通常出于性能原因，列族在物理上将一组列及其值放在一起。 每个列族都有一组存储属性，例如它的值是否应该缓存在内存中，它的数据是如何压缩的，它的行键是如何编码的，等等。 表中的每一行都有相同的列族，尽管给定的行可能不在给定的列族中存储任何内容。
- **列限定符** - 将列限定符添加到列族以提供给定数据片段的索引。 给定列族内容，列限定符可能是 content:html，另一个可能是 content:pdf。 尽管列族在表创建时是固定的，但列限定符是可变的，并且行之间可能有很大差异。
- **Cell** - 单元格是行、列族和列限定符的组合，包含一个值和一个时间戳，代表值的版本。
- **Timestamp** - 时间戳写在每个值旁边，是给定版本值的标识符。 默认情况下，时间戳表示写入数据时 RegionServer 上的时间，但您可以在将数据放入单元格时指定不同的时间戳值。

![img](https://raw.githubusercontent.com/dunwu/images/master/cs/bigdata/hbase/1551164224778.png)

### 特性比较

#### HBase vs. RDBMS

| RDBMS                                    | HBase                                              |
| ---------------------------------------- | -------------------------------------------------- |
| RDBMS 有它的模式，描述表的整体结构的约束 | HBase 无模式，它不具有固定列模式的概念；仅定义列族 |
| 支持的文件系统有 FAT、NTFS 和 EXT        | 支持的文件系统只有 HDFS                            |
| 使用提交日志来存储日志                   | 使用预写日志 (WAL) 来存储日志                      |
| 使用特定的协调系统来协调集群             | 使用 ZooKeeper 来协调集群                          |
| 存储的都是中小规模的数据表               | 存储的是超大规模的数据表，并且适合存储宽表         |
| 通常支持复杂的事务                       | 仅支持行级事务                                     |
| 适用于结构化数据                         | 适用于半结构化、结构化数据                         |
| 使用主键                                 | 使用 row key                                       |

#### HBase vs. HDFS

| HDFS                                      | HBase                                                |
| ----------------------------------------- | ---------------------------------------------------- |
| HDFS 提供了一个用于分布式存储的文件系统。 | HBase 提供面向表格列的数据存储。                     |
| HDFS 为大文件提供优化存储。               | HBase 为表格数据提供了优化。                         |
| HDFS 使用块文件。                         | HBase 使用键值对数据。                               |
| HDFS 数据模型不灵活。                     | HBase 提供了一个灵活的数据模型。                     |
| HDFS 使用文件系统和处理框架。             | HBase 使用带有内置 Hadoop MapReduce 支持的表格存储。 |
| HDFS 主要针对一次写入多次读取进行了优化。 | HBase 针对读/写许多进行了优化。                      |

#### 行式数据库 vs. 列式数据库

| 行式数据库                     | 列式数据库                     |
| ------------------------------ | ------------------------------ |
| 对于添加/修改操作更高效        | 对于读取操作更高效             |
| 读取整行数据                   | 仅读取必要的列数据             |
| 最适合在线事务处理系统（OLTP） | 不适合在线事务处理系统（OLTP） |
| 将行数据存储在连续的页内存中   | 将列数据存储在非连续的页内存中 |

列式数据库的优点：

- 支持数据压缩
- 支持快速数据检索
- 简化了管理和配置
- 有利于聚合查询（例如 COUNT、SUM、AVG、MIN 和 MAX）的高性能
- 分区效率很高，因为它提供了自动分片机制的功能，可以将较大的区域分配给较小的区域

列式数据库的缺点：

- JOIN 查询和来自多个表的数据未优化
- 必须为频繁的删除和更新创建拆分，因此降低了存储效率
- 由于非关系数据库的特性，分区和索引的设计非常困难

## HBase 安装

HBase 安装可以参考以下文档：

- [独立模式](https://hbase.apache.org/book.html#quickstart)
- [伪分布式模式](https://hbase.apache.org/book.html#quickstart_pseudo)
- [全分布式模式](https://hbase.apache.org/book.html#quickstart_fully_distributed)
- [Docker 部署](https://github.com/big-data-europe/docker-hbase)

## HBase Hello World 示例

（1）连接 HBase

在 HBase 安装目录的 `/bin` 目录下执行 `hbase shell` 命令进入 HBase 控制台。

```shell
$ ./bin/hbase shell
hbase(main):001:0>
```

（2）输入 `help` 可以查看 HBase Shell 命令。

（3）创建表

可以使用 `create` 命令创建一张新表。必须要指定表名和 Column Family。

```shell
hbase(main):001:0> create 'test', 'cf'
0 row(s) in 0.4170 seconds

=> Hbase::Table - test
```

（4）列出表信息

使用 `list` 命令来确认新建的表已存在。

```shell
hbase(main):002:0> list 'test'
TABLE
test
1 row(s) in 0.0180 seconds

=> ["test"]
```

可以使用 `describe` 命令可以查看表的细节信息，包括配置信息

```shell
hbase(main):003:0> describe 'test'
Table test is ENABLED
test
COLUMN FAMILIES DESCRIPTION
{NAME => 'cf', VERSIONS => '1', EVICT_BLOCKS_ON_CLOSE => 'false', NEW_VERSION_BEHAVIOR => 'false', KEEP_DELETED_CELLS => 'FALSE', CACHE_DATA_ON_WRITE =>
'false', DATA_BLOCK_ENCODING => 'NONE', TTL => 'FOREVER', MIN_VERSIONS => '0', REPLICATION_SCOPE => '0', BLOOMFILTER => 'ROW', CACHE_INDEX_ON_WRITE => 'f
alse', IN_MEMORY => 'false', CACHE_BLOOMS_ON_WRITE => 'false', PREFETCH_BLOCKS_ON_OPEN => 'false', COMPRESSION => 'NONE', BLOCKCACHE => 'true', BLOCKSIZE
 => '65536'}
1 row(s)
Took 0.9998 seconds
```

（5）向表中写数据

可以使用 `put` 命令向 HBase 表中写数据。

```shell
hbase(main):003:0> put 'test', 'row1', 'cf:a', 'value1'
0 row(s) in 0.0850 seconds

hbase(main):004:0> put 'test', 'row2', 'cf:b', 'value2'
0 row(s) in 0.0110 seconds

hbase(main):005:0> put 'test', 'row3', 'cf:c', 'value3'
0 row(s) in 0.0100 seconds
```

（6）一次性扫描表的所有数据

使用 `scan` 命令来扫描表数据。

```shell
hbase(main):006:0> scan 'test'
ROW                                      COLUMN+CELL
 row1                                    column=cf:a, timestamp=1421762485768, value=value1
 row2                                    column=cf:b, timestamp=1421762491785, value=value2
 row3                                    column=cf:c, timestamp=1421762496210, value=value3
3 row(s) in 0.0230 seconds
```

（7）查看一行数据

使用 `get` 命令可以查看一行表数据。

```shell
hbase(main):007:0> get 'test', 'row1'
COLUMN                                   CELL
 cf:a                                    timestamp=1421762485768, value=value1
1 row(s) in 0.0350 seconds
```

（8）禁用表

如果想要删除表或修改表设置，必须先使用 `disable` 命令禁用表。如果想再次启用表，可以使用 `enable` 命令。

```shell
hbase(main):008:0> disable 'test'
0 row(s) in 1.1820 seconds

hbase(main):009:0> enable 'test'
0 row(s) in 0.1770 seconds
```

（9）删除表

使用 `drop` 命令可以删除表。

```shell
hbase(main):011:0> drop 'test'
0 row(s) in 0.1370 seconds
```

（10）退出 HBase Shell

使用 `quit` 命令，就能退出 HBase Shell 控制台。

## 参考资料

- **官方**
  - [HBase 官网](http://hbase.apache.org/)
  - [HBase 官方文档](https://hbase.apache.org/book.html)
  - [HBase 官方文档中文版](http://abloz.com/hbase/book.html)
- **书籍**
  - [Hadoop 权威指南](https://book.douban.com/subject/27600204/)
- **文章**
  - [Bigtable: A Distributed Storage System for Structured Data](https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/bigtable-osdi06.pdf)
  - [An In-Depth Look at the HBase Architecture](https://mapr.com/blog/in-depth-look-hbase-architecture)
- **教程**
  - https://github.com/heibaiying/BigData-Notes
  - https://www.cloudduggu.com/hbase/introduction/
