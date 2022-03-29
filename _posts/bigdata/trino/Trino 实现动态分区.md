---
title: Trino 动态分区
date: 2022/02/14 22:59:59
updated: 2022/02/14 22:59:59
permalink: /bigdata/trino/partition.html
categories:
- bigdata
tags:
- Trino
---

# Trino 动态分区

Hive 可以动态分区插入，方法如下：

```sql
#开启动态分区，默认是false
set hive.exec.dynamic.partition=true;
#开启允许所有分区都是动态的，否则必须要有静态分区才能使用。
set hive.exec.dynamic.partition.mode=nonstrict;
insert overwrite table tablename partition (分区字段)
select a, b, year from tab;
```

<!--more-->

Trino/Presto 不支持这种写法

1. 不支持 overwrite 关键字，报错：`missing 'INTO' at 'overwrite'`
2. 不支持 partition 关键字，报错：`extraneous input 'partition' expecting {'.', '(', 'SELECT', 'TABLE', 'VALUES', 'WITH'}`

对于动态分区的重写，Trino 只能把表先清空再写入。

1. 使用 Trino 删除数据：

   ```sql
   delete from t
   ```

2. 使用 Trino 写入数据：

   ```SQL
   -- 把分区字段写到最后
   insert into t1 
   select a, b, year from t2
   ```

此外，对于时间数据类型，使用 Presto 查询时需要加上 date 关键字

```sql
SELECT * FROM t where d > date '2022-02-01' 
```

实际测试：

```sql
create table tmp.test_presto1(name string, etl_time timestamp);
insert into tmp.test_presto1 values('john', '2021-01-01 00:00:00');
insert into tmp.test_presto1 values('lucy', '2021-01-02 00:00:00');
insert into tmp.test_presto1 values('tom', '2021-01-03 01:00:00');

create table tmp.test_presto2(name string) partitioned by (etl_date string);

insert into tmp.test_presto2 
select name, cast(format_datetime(etl_time,'yyyy-MM-dd') as varchar(10)) from tmp.test_presto1;

-- 查分区数据
select * from tmp.test_presto2 where etl_date='2021-01-01';
select * from tmp.test_presto2 where etl_date>'2021-01-01';

-- 清空表
delete from tmp.test_presto2;
```

在 hdfs 上查看：

```shell
hdfs dfs -ls /user/hive/warehouse/tmp.db/test_presto2
Found 3 items
/user/hive/warehouse/tmp.db/test_presto2/etl_date=2021-01-01
/user/hive/warehouse/tmp.db/test_presto2/etl_date=2021-01-02
/user/hive/warehouse/tmp.db/test_presto2/etl_date=2021-01-03
```

