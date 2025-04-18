---
icon: logos:hbase
title: HBase Schema 设计
date: 2023-03-15 20:28:32
categories:
  - 数据库
  - 列式数据库
  - hbase
tags:
  - 数据库
  - 列式数据库
  - 大数据
  - hbase
permalink: /pages/2197e992/
---

# HBase Schema 设计

## HBase Schema 设计要素

- 这个表应该有多少 Column Family
- Column Family 使用什么数据
- 每个 Column Family 有有多少列
- 列名是什么，尽管列名不必在建表时定义，但读写数据是要知道的
- 单元应该存放什么数据
- 每个单元存储多少时间版本
- 行健(rowKey)结构是什么，应该包含什么信息

## Row Key 设计

### Row Key 的作用

在 HBase 中，所有对表的访问都要通过 Row Key，有三种访问方式：

- 使用 `get` 命令，查询指定的 Row Key，即精确查找。
- 使用 scan 命令，根据 Row Key 进行范围查找。
- 全表扫描，即直接扫描表中所有行记录。

此外，在 HBase 中，表中的行，是按照 Row Key 的字典序进行排序的。

由此，可见，Row Key 的良好设计对于 HBase CRUD 的性能至关重要。

### Row Key 的设计原则

**长度原则**

RowKey 是一个二进制码流，可以是任意字符串，最大长度为 64kb，实际应用中一般为 10-100byte，以 byte[]形式保存，一般设计成定长。建议越短越好，不要超过 16 个字节，原因如下：

1. 数据的持久化文件 HFile 中时按照 Key-Value 存储的，如果 RowKey 过长，例如超过 100byte，那么 1000w 行的记录，仅 RowKey 就需占用近 1GB 的空间。这样会极大影响 HFile 的存储效率。
2. MemStore 会缓存部分数据到内存中，若 RowKey 字段过长，内存的有效利用率就会降低，就不能缓存更多的数据，从而降低检索效率。
3. 目前操作系统都是 64 位系统，内存 8 字节对齐，控制在 16 字节，8 字节的整数倍利用了操作系统的最佳特性。

**唯一原则**

必须在设计上保证 RowKey 的唯一性。由于在 HBase 中数据存储是 Key-Value 形式，若向 HBase 中同一张表插入相同 RowKey 的数据，则原先存在的数据会被新的数据覆盖。

**排序原则**

HBase 的 RowKey 是按照 ASCII 有序排序的，因此我们在设计 RowKey 的时候要充分利用这点。

**散列原则**

设计的 RowKey 应均匀的分布在各个 HBase 节点上。

### 热点问题

Region 是在 HBase 集群上分布数据的最小单位。每个 Region 由它所属的表的起始范围来表示（即起始 Row Key 和结束 Row Key）。

如果，Row Key 使用单调递增的整数或时间戳，就会产生一个问题：因为 Hbase 的 Row Key 是就近存储的，这会导致一段时间内大部分读写集中在某一个 Region 或少数 Region 上（根据二八原则，最近产生的数据，往往是读写频率最高的数据），即所谓 **热点问题**。

#### 反转（Reversing）

第一种咱们要分析的方法是反转，顾名思义它就是把固定长度或者数字格式的 RowKey 进行反转，反转分为一般数据反转和时间戳反转，其中以时间戳反转较常见。

- **反转固定格式的数值** - 以手机号为例，手机号的前缀变化比较少（如 `152、185` 等），但后半部分变化很多。如果将它反转过来，可以有效地避免热点。不过其缺点就是失去了有序性。
- **反转时间** - 如果数据访问以查找最近的数据为主，可以将时间戳存储为反向时间戳（例如： `timestamp = Long.MAX_VALUE – timestamp`），这样有利于扫描最近的数据。

#### 加盐（Salting）

这里的“加盐”与密码学中的“加盐”不是一回事。它是指在 RowKey 的前面增加一些前缀，加盐的前缀种类越多，RowKey 就被打得越散。

需要注意的是分配的随机前缀的种类数量应该和我们想把数据分散到的那些 region 的数量一致。只有这样，加盐之后的 rowkey 才会根据随机生成的前缀分散到各个 region 中，避免了热点现象。

#### 哈希（Hashing）

其实哈希和加盐的适用场景类似，但我们前缀不可以是随机的，因为必须要让客户端能够完整地重构 RowKey。所以一般会拿原 RowKey 或其一部分计算 Hash 值，然后再对 Hash 值做运算作为前缀。

## HBase Schema 设计规则

### Column Family 设计

HBase 不能很好处理 2 ~ 3 个以上的 Column Family，所以 **HBase 表应尽可能减少 Column Family 数**。如果可以，请只使用一个列族，只有需要经常执行 Column 范围查询时，才引入多列族。也就是说，尽量避免同时查询多个列族。

- **Column Family 数量多，会影响数据刷新**。HBase 的数据刷新是在每个 Region 的基础上完成的。因此，如果一个 Column Family 携带大量导致刷新的数据，那么相邻的列族即使携带的数据量很小，也会被刷新。当存在许多 Column Family 时，刷新交互会导致一堆不必要的 IO。 此外，在表/区域级别的压缩操作也会在每个存储中发生。
- **Column Family 数量多，会影响查找效率**。如：Column Family A 有 100 万行，Column Family B 有 10 亿行，那么 Column Family A 的数据可能会分布在很多很多区域（和 RegionServers）。 这会降低 Column Family A 的批量扫描效率。

Column Family 名尽量简短，最好是一个字符。Column Family 会在列限定符中被频繁使用，缩短长度有利于节省空间并提升效率。

### Row 设计

**HBase 中的 Row 按 Row Key 的字典顺序排序**。

- **不要将 Row Key 设计为单调递增的**，例如：递增的整数或时间戳

  - 问题：因为 Hbase 的 Row Key 是就近存储的，这样会导致一段时间内大部分写入集中在某一个 Region 上，即所谓热点问题。

  - 解决方法一、加盐：这里的不是指密码学的加盐，而是指将随机分配的前缀添加到行键的开头。这么做是为了避免相同前缀的 Row Key 数据被存储在相邻位置，从而导致热点问题。示例如下：

    - ```
      foo0001
      foo0002
      foo0003
      foo0004

      改为

      a-foo0003
      b-foo0001
      c-foo0003
      c-foo0004
      d-foo0002
      ```

  - 解决方法二、Hash：Row Key 的前缀使用 Hash

- **尽量减少行和列的长度**

- **反向时间戳**：反向时间戳可以极大地帮助快速找到值的最新版本。

- **行健不能改变**：唯一可以改变的方式是先删除后插入。

- **Row Key 和 Column Family**：Row Key 从属于 Column Family，因此，相同的 Row Key 可以存在每一个 Column Family 中而不会出现冲突。

### Version 设计

最大、最小 Row 版本号：表示 HBase 会保留的版本号数的上下限。均可以通过 HColumnDescriptor 对每个列族进行配置

Row 版本号过大，会大大增加 StoreFile 的大小；所以，最大 Row 版本号应按需设置。HBase 会在主要压缩时，删除多余的版本。

### TTL 设计

Column Family 会设置一个以秒为单位的 TTL，一旦达到 TTL 时，HBase 会自动删除行记录。

仅包含过期行的存储文件在次要压缩时被删除。 将 hbase.store.delete.expired.storefile 设置为 false 会禁用此功能。将最小版本数设置为 0 以外的值也会禁用此功能。

在较新版本的 HBase 上，还支持在 Cell 上设置 TTL，与 Column Family 的 TTL 不同的是，单位是毫秒。

### Column Family 属性配置

- HFile 数据块，默认是 64KB，数据库的大小影响数据块索引的大小。数据块大的话一次加载进内存的数据越多，扫描查询效果越好。但是数据块小的话，随机查询性能更好

```
> create 'mytable',{NAME => 'cf1', BLOCKSIZE => '65536'}
复制代码
```

- 数据块缓存，数据块缓存默认是打开的，如果一些比较少访问的数据可以选择关闭缓存

```
> create 'mytable',{NAME => 'cf1', BLOCKCACHE => 'FALSE'}
复制代码
```

- 数据压缩，压缩会提高磁盘利用率，但是会增加 CPU 的负载，看情况进行控制

```
> create 'mytable',{NAME => 'cf1', COMPRESSION => 'SNAPPY'}
复制代码
```

Hbase 表设计是和需求相关的，但是遵守表设计的一些硬性指标对性能的提升还是很有帮助的，这里整理了一些设计时用到的要点。

## Schema 设计案例

### 案例：日志数据和时序数据

假设采集以下数据

- Hostname
- Timestamp
- Log event
- Value/message

应该如何设计 Row Key？

（1）Timestamp 在 Row Key 头部

如果 Row Key 设计为 `[timestamp][hostname][log-event]` 形式，会出现热点问题。

如果针对时间的扫描很重要，可以采用时间戳分桶策略，即

```
bucket = timestamp % bucketNum
```

计算出桶号后，将 Row Key 指定为：`[bucket][timestamp][hostname][log-event]`

如上所述，要为特定时间范围选择数据，需要对每个桶执行扫描。 例如，100 个桶将在键空间中提供广泛的分布，但需要 100 次扫描才能获取单个时间戳的数据，因此需要权衡取舍。

（2）Hostname 在 Row Key 头部

如果主机样本量很大，将 Row Key 设计为 `[hostname][log-event][timestamp]`，这样有利于扫描 hostname。

（3）Timestamp 还是反向 Timestamp

如果数据访问以查找最近的数据为主，可以将时间戳存储为反向时间戳（例如： `timestamp = Long.MAX_VALUE – timestamp`），这样有利于扫描最近的数据。

（4）Row Key 是可变长度还是固定长度

拼接 Row Key 的关键字长度不一定是固定的，例如 hostname 有可能很长，也有可能很短。如果想要统一长度，可以参考以下做法：

- 将关键字 Hash 编码：使用某种 Hash 算法计算关键字，并取固定长度的值（例如：8 位或 16 位）。
- 使用数字替代关键字：例如：使用事件类型 Code 替换事件类型；hostname 如果是 IP，可以转换为 long
- 截取关键字：截取后的关键字需要有足够的辨识度，长度大小根据具体情况权衡。

（5）时间分片

```
[hostname][log-event][timestamp1]
[hostname][log-event][timestamp2]
[hostname][log-event][timestamp3]
```

上面的例子中，每个详细事件都有单独的行键，可以重写如下，即每个时间段存储一次：

```
[hostname][log-event][timerange]
```

## 参考资料

- [HBase 官方文档之 HBase and Schema Design](https://hbase.apache.org/book.html#schema)
