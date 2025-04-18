---
icon: logos:hbase
title: HBase Java API 高级特性之协处理器
date: 2023-03-16 09:46:37
categories:
  - 数据库
  - 列式数据库
  - hbase
tags:
  - 数据库
  - 列式数据库
  - 大数据
  - hbase
  - API
permalink: /pages/ae009b3b/
---

# HBase Java API 高级特性之协处理器

## 简述

在使用 HBase 时，如果你的数据量达到了数十亿行或数百万列，此时能否在查询中返回大量数据将受制于网络的带宽，即便网络状况允许，但是客户端的计算处理也未必能够满足要求。在这种情况下，协处理器（Coprocessors）应运而生。它允许你将业务计算代码放入在 RegionServer 的协处理器中，将处理好的数据再返回给客户端，这可以极大地降低需要传输的数据量，从而获得性能上的提升。同时协处理器也允许用户扩展实现 HBase 目前所不具备的功能，如权限校验、二级索引、完整性约束等。

## 参考资料

- [《HBase 权威指南》](https://item.jd.com/11321037.html)
- [《HBase 权威指南》官方源码](https://github.com/larsgeorge/hbase-book)
