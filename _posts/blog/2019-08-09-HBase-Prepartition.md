---
layout: post
title: HBase 预分区
categories: [HBase]
description: HBase 预分区
keywords: HBase
---

HBase 提供了预分区功能，即用户可以在创建表的时候对表按照一定的规则分区

---

#### 前言

HBase 存在热点问题。热点问题首选解决方法是 Rowkey 的散列与预分区设计

Rowkey 散列可以参看[HBase Rowkey 设计指南](https://lihuimintu.github.io/2019/08/07/HBase-Rowkey/)

#### 概述

**什么是预分区？**

HBase 表在刚刚被创建时，只有1个分区（region），当一个 region 过大（达到 hbase.hregion.max.filesize 属性中定义的阈值，默认10GB）时，

表将会进行split，分裂为2个分区。表在进行split的时候，会耗费大量的资源，频繁的分区对HBase的性能有巨大的影响。

HBase提供了预分区功能，即用户可以在创建表的时候对表按照一定的规则分区。

**预分区的目的是什么？**

减少由于 region split 带来的资源消耗。从而提高HBase的性能。

#### 方法

##### shell

通过 HBase shell 来创建

在命令中指定分区的 Rowkey

``` 
create 't1', 'f1', SPLITS => ['10', '20', '30', '40']

create 't1', {NAME =>'f1', TTL => 180}, SPLITS => ['10', '20', '30', '40']

create 't1', {NAME =>'f1', TTL => 180}, {NAME => 'f2', TTL => 240}, SPLITS => ['10', '20', '30', '40']
```

还可以通过 HBase shell 指定文件来创建

在任意路径下创建一个保存分区 key 的文件，一行代表一个 Rowkey

``` 
create 't1', 'f1', SPLITS_FILE => '/home/hadmin/hbase-1.3.1/txt/splits.txt'

create 't1', {NAME =>'f1', TTL => 180}, SPLITS_FILE => '/home/hadmin/hbase-1.3.1/txt/splits.txt'

create 't1', {NAME =>'f1', TTL => 180}, {NAME => 'f2', TTL => 240}, SPLITS_FILE => '/home/hadmin/hbase-1.3.1/txt/splits.txt'
```

##### api

``` 
package api;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.HColumnDescriptor;
import org.apache.hadoop.hbase.HTableDescriptor;
import org.apache.hadoop.hbase.TableName;
import org.apache.hadoop.hbase.client.Admin;
import org.apache.hadoop.hbase.client.Connection;
import org.apache.hadoop.hbase.client.ConnectionFactory;
import org.apache.hadoop.hbase.util.Bytes;

public class create_table_sample2 {
    public static void main(String[] args) throws Exception {
        Configuration conf = HBaseConfiguration.create();
        conf.set("hbase.zookeeper.quorum", "192.168.1.80,192.168.1.81,192.168.1.82");
        Connection connection = ConnectionFactory.createConnection(conf);
        Admin admin = connection.getAdmin();

        TableName table_name = TableName.valueOf("TEST1");
        if (admin.tableExists(table_name)) {
            admin.disableTable(table_name);
            admin.deleteTable(table_name);
        }

        HTableDescriptor desc = new HTableDescriptor(table_name);
        HColumnDescriptor family1 = new HColumnDescriptor(constants.COLUMN_FAMILY_DF.getBytes());
        family1.setTimeToLive(3 * 60 * 60 * 24);     //过期时间
        family1.setMaxVersions(3);                   //版本数
        desc.addFamily(family1);

        byte[][] splitKeys = {
            Bytes.toBytes("row01"),
            Bytes.toBytes("row02"),
        };

        admin.createTable(desc, splitKeys);
        admin.close();
        connection.close();
    }
}
```

---
参考链接
* [HBase预分区方法](https://www.cnblogs.com/quchunhui/p/7543385.html)
* [HBase Rowkey的散列与预分区设计](https://www.cnblogs.com/bdifn/p/3801737.html)


