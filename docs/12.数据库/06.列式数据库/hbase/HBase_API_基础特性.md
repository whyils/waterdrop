---
icon: logos:hbase
title: HBase Java API 基础特性
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
permalink: /pages/f62adb4e/
---

# HBase Java API 基础特性

## HBase Client API

### HBase Java API 示例

引入依赖

```xml
<dependency>
    <groupId>org.apache.hbase</groupId>
    <artifactId>hbase-client</artifactId>
    <version>2.1.4</version>
</dependency>
```

示例

```java
public class HBaseUtils {

    private static Connection connection;

    static {
        Configuration configuration = HBaseConfiguration.create();
        configuration.set("hbase.zookeeper.property.clientPort", "2181");
        // 如果是集群 则主机名用逗号分隔
        configuration.set("hbase.zookeeper.quorum", "hadoop001");
        try {
            connection = ConnectionFactory.createConnection(configuration);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * 创建 HBase 表
     *
     * @param tableName      表名
     * @param columnFamilies 列族的数组
     */
    public static boolean createTable(String tableName, List<String> columnFamilies) {
        try {
            HBaseAdmin admin = (HBaseAdmin) connection.getAdmin();
            if (admin.tableExists(TableName.valueOf(tableName))) {
                return false;
            }
            TableDescriptorBuilder tableDescriptor = TableDescriptorBuilder.newBuilder(TableName.valueOf(tableName));
            columnFamilies.forEach(columnFamily -> {
                ColumnFamilyDescriptorBuilder cfDescriptorBuilder = ColumnFamilyDescriptorBuilder.newBuilder(Bytes.toBytes(columnFamily));
                cfDescriptorBuilder.setMaxVersions(1);
                ColumnFamilyDescriptor familyDescriptor = cfDescriptorBuilder.build();
                tableDescriptor.setColumnFamily(familyDescriptor);
            });
            admin.createTable(tableDescriptor.build());
        } catch (IOException e) {
            e.printStackTrace();
        }
        return true;
    }

    /**
     * 删除 hBase 表
     *
     * @param tableName 表名
     */
    public static boolean deleteTable(String tableName) {
        try {
            HBaseAdmin admin = (HBaseAdmin) connection.getAdmin();
            // 删除表前需要先禁用表
            admin.disableTable(TableName.valueOf(tableName));
            admin.deleteTable(TableName.valueOf(tableName));
        } catch (Exception e) {
            e.printStackTrace();
        }
        return true;
    }

    /**
     * 插入数据
     *
     * @param tableName        表名
     * @param rowKey           唯一标识
     * @param columnFamilyName 列族名
     * @param qualifier        列标识
     * @param value            数据
     */
    public static boolean putRow(String tableName, String rowKey, String columnFamilyName, String qualifier,
                                 String value) {
        try {
            Table table = connection.getTable(TableName.valueOf(tableName));
            Put put = new Put(Bytes.toBytes(rowKey));
            put.addColumn(Bytes.toBytes(columnFamilyName), Bytes.toBytes(qualifier), Bytes.toBytes(value));
            table.put(put);
            table.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return true;
    }

    /**
     * 插入数据
     *
     * @param tableName        表名
     * @param rowKey           唯一标识
     * @param columnFamilyName 列族名
     * @param pairList         列标识和值的集合
     */
    public static boolean putRow(String tableName, String rowKey, String columnFamilyName, List<Pair<String, String>> pairList) {
        try {
            Table table = connection.getTable(TableName.valueOf(tableName));
            Put put = new Put(Bytes.toBytes(rowKey));
            pairList.forEach(pair -> put.addColumn(Bytes.toBytes(columnFamilyName), Bytes.toBytes(pair.getKey()), Bytes.toBytes(pair.getValue())));
            table.put(put);
            table.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return true;
    }

    /**
     * 根据 rowKey 获取指定行的数据
     *
     * @param tableName 表名
     * @param rowKey    唯一标识
     */
    public static Result getRow(String tableName, String rowKey) {
        try {
            Table table = connection.getTable(TableName.valueOf(tableName));
            Get get = new Get(Bytes.toBytes(rowKey));
            return table.get(get);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }

    /**
     * 获取指定行指定列 (cell) 的最新版本的数据
     *
     * @param tableName    表名
     * @param rowKey       唯一标识
     * @param columnFamily 列族
     * @param qualifier    列标识
     */
    public static String getCell(String tableName, String rowKey, String columnFamily, String qualifier) {
        try {
            Table table = connection.getTable(TableName.valueOf(tableName));
            Get get = new Get(Bytes.toBytes(rowKey));
            if (!get.isCheckExistenceOnly()) {
                get.addColumn(Bytes.toBytes(columnFamily), Bytes.toBytes(qualifier));
                Result result = table.get(get);
                byte[] resultValue = result.getValue(Bytes.toBytes(columnFamily), Bytes.toBytes(qualifier));
                return Bytes.toString(resultValue);
            } else {
                return null;
            }

        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }

    /**
     * 检索全表
     *
     * @param tableName 表名
     */
    public static ResultScanner getScanner(String tableName) {
        try {
            Table table = connection.getTable(TableName.valueOf(tableName));
            Scan scan = new Scan();
            return table.getScanner(scan);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }

    /**
     * 检索表中指定数据
     *
     * @param tableName  表名
     * @param filterList 过滤器
     */

    public static ResultScanner getScanner(String tableName, FilterList filterList) {
        try {
            Table table = connection.getTable(TableName.valueOf(tableName));
            Scan scan = new Scan();
            scan.setFilter(filterList);
            return table.getScanner(scan);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }

    /**
     * 检索表中指定数据
     *
     * @param tableName   表名
     * @param startRowKey 起始 RowKey
     * @param endRowKey   终止 RowKey
     * @param filterList  过滤器
     */

    public static ResultScanner getScanner(String tableName, String startRowKey, String endRowKey,
                                           FilterList filterList) {
        try {
            Table table = connection.getTable(TableName.valueOf(tableName));
            Scan scan = new Scan();
            scan.withStartRow(Bytes.toBytes(startRowKey));
            scan.withStopRow(Bytes.toBytes(endRowKey));
            scan.setFilter(filterList);
            return table.getScanner(scan);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }

    /**
     * 删除指定行记录
     *
     * @param tableName 表名
     * @param rowKey    唯一标识
     */
    public static boolean deleteRow(String tableName, String rowKey) {
        try {
            Table table = connection.getTable(TableName.valueOf(tableName));
            Delete delete = new Delete(Bytes.toBytes(rowKey));
            table.delete(delete);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return true;
    }

    /**
     * 删除指定行指定列
     *
     * @param tableName  表名
     * @param rowKey     唯一标识
     * @param familyName 列族
     * @param qualifier  列标识
     */
    public static boolean deleteColumn(String tableName, String rowKey, String familyName,
                                          String qualifier) {
        try {
            Table table = connection.getTable(TableName.valueOf(tableName));
            Delete delete = new Delete(Bytes.toBytes(rowKey));
            delete.addColumn(Bytes.toBytes(familyName), Bytes.toBytes(qualifier));
            table.delete(delete);
            table.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return true;
    }

}
```

## 数据库连接

在上面的代码中，在类加载时就初始化了 Connection 连接，并且之后的方法都是复用这个 Connection，这时我们可能会考虑是否可以使用自定义连接池来获取更好的性能表现？实际上这是没有必要的。

首先官方对于 `Connection` 的使用说明如下：

```
Connection Pooling For applications which require high-end multithreaded
access (e.g., web-servers or  application servers  that may serve many
application threads in a single JVM), you can pre-create a Connection,
as shown in the following example:

对于高并发多线程访问的应用程序（例如，在单个 JVM 中存在的为多个线程服务的 Web 服务器或应用程序服务器），
您只需要预先创建一个 Connection。例子如下：

// Create a connection to the cluster.
Configuration conf = HBaseConfiguration.create();
try (Connection connection = ConnectionFactory.createConnection(conf);
     Table table = connection.getTable(TableName.valueOf(tablename))) {
  // use table as needed, the table returned is lightweight
}
```

之所以能这样使用，这是因为 Connection 并不是一个简单的 socket 连接，[接口文档](https://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Connection.html) 中对 Connection 的表述是：

```
A cluster connection encapsulating lower level individual connections to actual servers and a
connection to zookeeper.  Connections are instantiated through the ConnectionFactory class.
The lifecycle of the connection is managed by the caller,  who has to close() the connection
to release the resources.

Connection 是一个集群连接，封装了与多台服务器（Matser/Region Server）的底层连接以及与 zookeeper 的连接。
连接通过 ConnectionFactory  类实例化。连接的生命周期由调用者管理，调用者必须使用 close() 关闭连接以释放资源。
```

之所以封装这些连接，是因为 HBase 客户端需要连接三个不同的服务角色：

- **Zookeeper** ：主要用于获取 `meta` 表的位置信息，Master 的信息；
- **HBase Master** ：主要用于执行 HBaseAdmin 接口的一些操作，例如建表等；
- **HBase RegionServer** ：用于读、写数据。

![](https://raw.githubusercontent.com/dunwu/images/master/snap/20230315202403.png)

Connection 对象和实际的 Socket 连接之间的对应关系如下图：

![](https://raw.githubusercontent.com/dunwu/images/master/snap/20230315202426.png)

在 HBase 客户端代码中，真正对应 Socket 连接的是 `RpcConnection` 对象。HBase 使用 `PoolMap` 这种数据结构来存储客户端到 HBase 服务器之间的连接。`PoolMap` 的内部有一个 `ConcurrentHashMap` 实例，其 key 是 `ConnectionId`（封装了服务器地址和用户 ticket)，value 是一个 `RpcConnection` 对象的资源池。当 HBase 需要连接一个服务器时，首先会根据 `ConnectionId` 找到对应的连接池，然后从连接池中取出一个连接对象。

```
@InterfaceAudience.Private
public class PoolMap<K, V> implements Map<K, V> {
  private PoolType poolType;

  private int poolMaxSize;

  private Map<K, Pool<V>> pools = new ConcurrentHashMap<>();

  public PoolMap(PoolType poolType) {
    this.poolType = poolType;
  }
  .....
```

HBase 中提供了三种资源池的实现，分别是 `Reusable`，`RoundRobin` 和 `ThreadLocal`。具体实现可以通 `hbase.client.ipc.pool.type` 配置项指定，默认为 `Reusable`。连接池的大小也可以通过 `hbase.client.ipc.pool.size` 配置项指定，默认为 1，即每个 Server 1 个连接。也可以通过修改配置实现：

```
config.set("hbase.client.ipc.pool.type",...);
config.set("hbase.client.ipc.pool.size",...);
connection = ConnectionFactory.createConnection(config);
```

由此可以看出 HBase 中 Connection 类已经实现了对连接的管理功能，所以我们不必在 Connection 上在做额外的管理。

另外，Connection 是线程安全的，但 Table 和 Admin 却不是线程安全的，因此正确的做法是一个进程共用一个 Connection 对象，而在不同的线程中使用单独的 Table 和 Admin 对象。Table 和 Admin 的获取操作 `getTable()` 和 `getAdmin()` 都是轻量级，所以不必担心性能的消耗，同时建议在使用完成后显示的调用 `close()` 方法来关闭它们。

## 概述

HBase 的主要客户端操作是由 `org.apache.hadoop.hbase.client.HTable` 提供的。创建 HTable 实例非常耗时，所以，建议每个线程只创建一次 HTable 实例。

HBase 所有修改数据的操作都保证了行级别的原子性。要么读到最新的修改，要么等待系统允许写入改行修改

用户要尽量使用批处理 (batch) 更新来减少单独操作同一行数据的次数

写操作中设计的列的数目并不会影响该行数据的原子性，行原子性会同时保护到所有列

创建 HTable 实例（指的是在 java 中新建该类），每个实例都要扫描。META. 表，以检查该表是否存在，推荐用户只创建一次 HTable 实例，而且是每个线程创建一个

如果用户需要多个 HTable 实例，建议使用 HTablePool 类（类似连接池）

## CRUD 操作

### put

`Table` 接口提供了两个 `put` 方法

```java
// 写入单行 put
void put(Put put) throws IOException;
// 批量写入 put
void put(List<Put> puts) throws IOException;
```

Put 类提供了多种构造器方法用来初始化实例。

Put 类还提供了一系列有用的方法：

多个 `add` 方法：用于添加指定的列数据。

`has` 方法：用于检查是否存在特定的单元格，而不需要遍历整个集合

`getFamilyMap` 方法：可以遍历 Put 实例中每一个可用的 KeyValue 实例

getRow 方法：用于获取 rowkey
Put.heapSize() 可以计算当前 Put 实例所需的堆大小，既包含其中的数据，也包含内部数据结构所需的空间

#### KeyValue 类

特定单元格的数据以及坐标，坐标包括行键、列族名、列限定符以及时间戳
`KeyValue(byte[] row, int roffset, int rlength, byte[] family, int foffoset, int flength, byte[] qualifier, int qoffset, int qlength, long timestamp, Type type, byte[] value, int voffset, int vlength)`
每一个字节数组都有一个 offset 参数和一个 length 参数，允许用户提交一个已经存在的字节数组进行字节级别操作。
行目前来说指的是行键，即 Put 构造器里的 row 参数。

#### 客户端的写缓冲区

每一个 put 操作实际上都是一个 RPC 操作，它将客户端数据传送到服务器然后返回。

HBase 的 API 配备了一个客户端的写缓冲区，缓冲区负责收集 put 操作，然后调用 RPC 操作一次性将 put 送往服务器。

```java
void setAutoFlush(boolean autoFlush)
boolean isAutoFlush()
```

默认情况下，客户端缓冲区是禁用的。可以通过 `table.setAutoFlush(false)` 来激活缓冲区。

#### Put 列表

批量提交 `put` 列表：

```java
void put(List<Put> puts) throws IOException
```

注意：批量提交可能会有部分修改失败。

#### 原子性操作 compare-and-set

`checkAndPut` 方法提供了 CAS 机制来保证 put 操作的原子性。

### get

```
Result get(Get get) throws IOException
```

```csharp
Get(byte[] row)
Get(byte[] row, RowLock rowLock)
Get addColumn(byte[] family, byte[] qualifier)
Get addFamily(byte[] family)
```

#### Result 类

当用户使用 `get()` 方法获取数据，HBase 返回的结果包含所有匹配的单元格数据，这些数据被封装在一个 `Result` 实例中返回给用户。

Result 类提供的方法如下：

```java
byte[] getValue(byte[] family, byte[] qualifier)
byte[] value()
byte[] getRow()
int size()
boolean isEmpty()
KeyValue[] raw()
List<KeyValue> list()
```

## delete

```
void delete(Delete delete) throws IOException
```

```csharp
Delte(byte[] row)
Delete(byte[] row, long timestamp, RowLock rowLock)
```

```csharp
Delete deleteFamily(byte[] family)
Delete deleteFamily(byte[] family, long timestamp)
Delete deleteColumns(byte[] family, byte[] qualifier)
Delete deleteColumn(byte[] family, byte[] qualifier) // 只删除最新版本
```

## 批处理操作

Row 是 Put、Get、Delete 的父类。

```java
void batch(List<Row> actions, Object[] results) throws IOException, InterruptedException
Object batch(List<Row> actions) throws IOException, InterruptedException
```

## 行锁

region 服务器提供了行锁特性，这个特性保证了只有一个客户端能获取一行数据相应的锁，同时对该行进行修改。

如果不显示指定锁，服务器会隐式加锁。

## 扫描

scan，类似数据库系统中的 cursor，利用了 HBase 提供的底层顺序存储的数据结构。

调用 HTable 的 getScanner 就可以返回扫描器

```java
ResultScanner getScanner(Scan scan) throws IOException
ResultScanner getScanner(byte[] family) throws IOException
```

Scan 类构造器可以有 startRow，区间一般为 [startRow, stopRow)

```csharp
Scan(byte[] startRow, Filter filter)
Scan(byte[] startRow)
```

### ResultScanner

以行为单位进行返回

```java
Result next() throws IOException
Result[] next(int nbRows) throws IOException
void close()
```

### 缓存与批量处理

每一个 next() 调用都会为每行数据生成一个单独的 RPC 请求

可以设置扫描器缓存

```cpp
void setScannerCaching(itn scannerCaching)
int getScannerCaching()
```

缓存是面向行一级操作，批量是面向列一级操作

```cpp
void setBatch(int batch)
int getBatch
```

RPC 请求的次数=（行数、*每行列数）/Min（每行的列数，批量大小）/扫描器缓存

## 各种特性

`Bytes` 类提供了一系列将原生 Java 类型和字节数组互转的方法。

## 参考资料

- [《HBase 权威指南》](https://item.jd.com/11321037.html)
- [《HBase 权威指南》官方源码](https://github.com/larsgeorge/hbase-book)
