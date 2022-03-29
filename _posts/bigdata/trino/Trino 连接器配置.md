---
title: Trino 连接器配置
date: 2022/02/14 22:59:59
updated: 2022/02/14 22:59:59
permalink: /bigdata/trino/connectors.html
categories:
- bigdata
tags:
- Trino
---

## Hive connector

### Overview

Trino 仅使用 Hive 的数据和元数据，不使用 HiveQL 或 Hive 的执行环境的任何部分。

可以使用 Hive 连接器查询包括 HDFS，Amazon S3 或 S3 兼容系统，Google 云存储和 Azure 存储等许多分布式存储系统。

Coordinator 和所有 Worker 必须能通过网络访问 Hive Metastore 和文件系统，Trino 与 Hive Metastore 通信使用 Thrift 协议，默认端口 9083。

<!--more-->

### 支持的文件格式

Hive Connector支持以下文件类型：

- ORC
- Parquet
- Avro
- RCText（使用 `ColumnarSerDe` 的 RCFile）
- RCBinary（使用 `LazyBinaryColumnarSerDe` 的 RCFile）
- SequenceFile
- JSON（使用 `org.apache.hive.hcatalog.data.JsonSerDe` ）
- CSV（使用 `org.apache.hadoop.hive.serde2.OpenCsvSerDe`）
- TextFile

### 物化视图

Hive Connector 支持读取 Hive 物化视图。在 Trino 中，这些物化视图显示为常规的只读表。

### 视图

Hive 的视图通过 HiveQL 定义并存储在 Hive Metastore 服务中。Trino 可以分析并读取到数据。

Hive Connector 支持以三种不同模式读取 Hive 视图：Disable、Legacy、Experimental。可以在配置文件中配置使用哪种模式。

- Disabled

  - 默认，Hive 视图在 Trino 中不可用。

- Legacy

  - 执行 Hive 视图的一个简单的实现，允许对视图中的数据读取访问。

  - 通过 `hive.translate-hive-views=true` 和 `hive.legacy-hive-view-translation=true` 配置项开启

  - 可以通过在 SQL 前加入 `SET legacy_hive_view_translation = true` ，临时使用 Legacy 方式的 Hive 视图。

  - 这种模式只是将 Hive 视图的 HiveQL 语言定义语句直接替换到查询 SQL 中，没有做任何转换。因此，在复杂查询中可能会出现结果不同或查询失败的情况。

- Experimental

  - 重新设计并可能比 Legacy 模式拥有更大的潜力，它可以分析，处理和重写包括表达式和语句的 Hive 视图。
  - 它支持 Hive 视图的以下功能：
    - `UNION [DISTINCT]` 和 `UNION ALL`
    - 嵌套的 `GROUP BY` 子句
    - `current_user()`
    - `LATERAL VIEW OUTER EXPLODE`
    - array 数据类型的 `LATERAL VIEW [OUTER] EXPLODE` 
    - `LATERAL VIEW json_tuple`
  - 通过 `hive.translate-hive-views=true` 配置开启 Experimental 模式的 Hive 视图支持。同时要确保 `hive.legacy-hive-view-translation` 配置被移除或被置为 `false`
  - 注意：有很多的特性 Experimental 模式还没有支持，以下是缺少功能的不完整列表：
    - HiveQL中的 `current_date`, `current_timestamp` 等
    - Hive 的方法调用，包括：`translate()`、窗口方法等
    - 表表达式和简单大小写表达式
    - 时间表达式经度设置
    - 支持所有 Hive 数据类型并正确映射到 Trino 类型
    - 执行 UDF 的能力

### 配置

在 Trino 根目录中创建 `etc/catalog/hive.properties` 配置文件

```bash
$ sudo mkdir -p etc/catalog
$ sudo vim etc/catalog/hive.properties
```

在 `hive.properties` 中写入以下配置：

```properties
# 连接器名称
connector.name=hive
# Hive Metastore 地址,如果 HMS 配置了HA，多个地址使用『,』隔开
hive.metastore.uri=thrift://manage2.us-east1-b.c.ironmeta.internal:9083
```

#### HDFS 配置

对于基本设置，Trino 会自动配置 HDFS 客户端，不需要任何配置文件。但如果使用联合 HDFS 或 NameNode 配置了高可用时，需要指定 HDFS 客户端配置文件，以便 Trino 访问 HDFS 群集。

添加 hive.config.resources 属性以引用 HDFS 配置文件：

```properties
# 指定 hdfs 配置文件位置
hive.config.resources=/opt/package/trino/etc/catalog/hdfs-conf/core-site.xml,/opt/package/trino/etc/catalog/hdfs-conf/hdfs-site.xml
```

