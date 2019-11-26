---
layout: post
title:  CSV文件导入MYSQL遇到的事
categories:  [Java]
description: csv文件导入mysql, spring jpa 自动建表字段乱序
keywords: Jpa,Hibernate,Mysql
---
CSV文件导入MYSQL遇到的事
---




### 			　　　　　			CSV文件导入MYSQL遇到的事

​		撸到几十万条数据，拿来写实验的时候测试。but... 全是csv格式的，而且，没有关系模型。

​		瞬间后悔，当初在公司的时候怎么没有把开发和测试环境下mysql的表和数据全部用mysqldump导出来。　不过想来就算导出来也理不清，表太多，表内数据量太大(也因为后期项目变大才用微服务重构)。然后现在有两件事要做，自己建表，然后把csv文件内的数据导入到mysql。

​		总的环境是ubuntu18.04

##### 		1. 建表

​	　自己写create建表，然后嫌麻烦，表数量不少，每个表字段也不少，还有考虑数据类型（比如交易金额这些不能用double,只能用bigdecimal），长度等等，好麻烦。还是用hibernate自动建表偷个懒。

　　哔哩啪啦写完，字段长度约束什么的，懒得弄了。

```java
@Entity
@Table(name = "olist_order_items_dataset")
public class OlistOrderItemsDataset {
    @Id
    @Column(name = "order_id")
    private String orderId;
    // order_item_id
    @Column(name = "order_item_id")
    private Integer orderItemId;
    // product_id
    @Column(name = "product_id")
    private String productId;
    // seller_id
    @Column(name = "seller_id")
    private String sellerId;
    //price
    @Column(name = "price")
    private BigDecimal price;
    // freight_value
    @Column(name = "freight_value")
    private BigDecimal freightValue;
}

```

​		然后springboot项目启动一跑，表建完了。

#### 数据导入

​		然后就是把csv文件内的数据导入到mysql，　进入mysql 然后切换到olist数据库，导入数据。

```mysql
LOAD DATA LOCAL INFILE '/home/zemingyan/Downloads/data/olist/olist_order_items_dataset.csv'
INTO TABLE olist_order_items_dataset
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;
```

然后，报错

```mysql
ERROR 1290 (HY000): The MySQL server is running with the –secure-file-priv option so it cannot execute this statement
```

​		在mysql中secure-file-priv默认的值为 /var/lib/mysql-files　也就是说如果要讲这些文件(json,csv都可以)导入到mysql，必须要放到/var目录下去。　多麻烦，而且/var目录下的文件比较随意存放一些变量，删掉也没事，平时dpkg安装东西失败的时候lock文件就在/var下，不删掉的话后续就一直被锁了。

​		然后还是觉得改一下secure-file-priv参数，在mysql的my.cnf下追加

```mys
[mysqld]
secure-file-priv = ""
```

​	　然后，这里有点不一样，在ubuntu16.04的时候，[mysql]和[mysqld]都是放在my.cnf下，但是在18.04就放在不同文件下了。　 　

```shell
 cat  /etc/mysql/my.cnf
```

得到的结果

```shell
# The MySQL database server configuration file.
# You can copy this to one of:
# - "/etc/mysql/my.cnf" to set global options,
# - "~/.my.cnf" to set user-specific options.
# One can use all long options that the program supports.
# Run program with --help to get a list of available options and with
# --print-defaults to see which it would actually understand and use.
# For explanations see
# http://dev.mysql.com/doc/mysql/en/server-system-variables.html

# * IMPORTANT: Additional settings that can override those from this file!
#   The files must end with '.cnf', otherwise they'll be ignored.
!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mysql.conf.d/
```

[mysqld]在 /etc/mysql/mysql.conf.d/mysqld.cnf下，　修改完重启mysql 

```shell
service mysql restart
```

然后查看　属性 secure_file_priv

```mysql
show variables like 'secure_file_priv';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| secure_file_priv |       |
+------------------+-------+
1 row in set (0.00 sec)
```



然后再执行上面的导入命令。　注：　如果如果导入的时候不加LOCAL，则默认导入的数据文件是在服务端，就会出现以下错误。

```mysq
ERROR 1290 (HY000): The MySQL server is running with the –secure-file-priv option so it cannot execute this statement
```



然后，就可以正常的导入数据了，

```mysql
mysql> LOAD DATA LOCAL INFILE '/home/zemingyan/Downloads/data/olist/olist_order_items_dataset.csv'
    -> INTO TABLE olist_order_items_dataset
    -> FIELDS TERMINATED BY ','
    -> ENCLOSED BY '"'
    -> LINES TERMINATED BY '\n'
    -> IGNORE 1 ROWS;
Query OK, 98666 rows affected, 13984 warnings (1.11 sec)
Records: 112650  Deleted: 0  Skipped: 13984  Warnings: 13984
```

然后看一下导入的数据(另一张有问题表的数据)

```mysql
mysql> select * from olist_geolocation_dataset limit 5;
+------+---------------------+-----------------+-----------------+-------------------+-----------------------------+
| gid  | geolocation_city    | geolocation_lat | geolocation_lng | geolocation_state | geolocation_zip_code_prefix |
+------+---------------------+-----------------+-----------------+-------------------+-----------------------------+
| 1001 | -23.54929199999999  |          -46.63 |            0.00 | SP                | NULL                        |
| 1002 | -23.54831797807146  |          -46.64 |            0.00 | SP                | NULL                        |
| 1003 | -23.54903244546711  |          -46.64 |            0.00 | SP                | NULL                        |
| 1004 | -23.550115903139222 |          -46.64 |            0.00 | SP                | NULL                        |
| 1005 | -23.549819091869107 |          -46.64 |            0.00 | SP                | NULL                        |
+------+---------------------+-----------------+-----------------+-------------------+-----------------------------+
```

wc,这数据，居然有0，还有null，这是什么破玩意儿。　然后查看一下mysql里面的表，看一下建表语句。

```mysql
  mysql> show create table olist_order_items_dataset;
 olist_order_items_dataset | CREATE TABLE `olist_order_items_dataset` (
  `order_id` varchar(255) NOT NULL,
  `freight_value` decimal(19,2) DEFAULT NULL,
  `order_item_id` int(11) DEFAULT NULL,
  `price` decimal(19,2) DEFAULT NULL,
  `product_id` varchar(255) DEFAULT NULL,
  `seller_id` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`order_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 |
```

​		Entity中定义的时候，列的顺序明明是　order_item_id   product_id   seller_id ....   然后建表的时候居然变了，根本就没有根据你定义的顺序来。　　然后load data local 导入的时候，又是按照列来的，接着就全乱套了，bigdecimal被导入字符串的话，就直接null了。　　

​		what？本来还想偷个懒，结果直接凉了，难道又要重新去自己手动建表，那还不如用mybatis，那岂不是白整了，浪费时间？　不行，作为程序员，得继续解决问题，就像之前说的，碰到问题不去排查直接换操作系统的程序员是最low的程序员。

​		一顿google，原来在hibernate开发的时候，用来装载这些字段的数据结构是TreeMap，然后顺序就变了，变成了字典序，跟你自己定义的顺序没有半毛钱关系。然后需要自己修改hibernate的源码，what？　修改源码？那我要自己下下来改完然后重新打成jar导入进去？而且我用的是maven，又不是像go一样直接自己找包。又一顿谷歌，发现其实编译完以后不需要打jar包重新替换（虽然打成jar替换也可以）。

​	

#### 重新编译



​		在依赖下找到PropertyContainer类，在org.hibernate.cfg包下（所幸hibernate是开源的，要是碰到闭源的，还要自己去反编译），把代码拷贝出来（这些类都是只读的，改不了）然后自己新建一个PropertyContainer类，把里面的TreeMap改成LinkedHashMap。

```java
package org.hibernate.cfg; //这里的包名必须和hibernate下的一样
class PropertyContainer {
    // 源码中的TreeMap 修改成LinkedHashMap 　然后装载的时候顺序就是你定义的顺序
    //private final TreeMap<String, XProperty> fieldAccessMap; 
    private final LinkedHashMap<String, XProperty> persistentAttributeMap;
    //还有两个内部方法，原本的参数是TreeMap, 都修改成LinkedHashMap
    this.collectPersistentAttributesUsingLocalAccessType(this.persistentAttributeMap, persistentAttributesFromGetters, fields, getters);
            this.collectPersistentAttributesUsingClassLevelAccessType(this.persistentAttributeMap, persistentAttributesFromGetters, fields, getters);
}
```

​	关于包名，要和hibernate下的一样，如果是springboot项目，不能放在启动类下，通常在创建项目的时候，都是以com.xxx.xx（根据公司项目组和业务定名字）开头，直接丢在启动类下，包名就不是org.hibernate.cfg了。必须自己在java目录下创建对应的包名，如果是平时偷懒的，直接把代码放在和启动类同一目录下（spirngboot默认的扫描路径）。也没有配置＠ComponentScan，然后这个时候PropertyContainer又不在启动类所在目录，就会扫描不到，压根就没有被JVM加载，那和没有改是一回事，还是按照字典序来创建列的顺序。

​	　然后重启项目，查看mysql的建表语句

```mysql
mysql> show create table olist_order_items_dataset;

| olist_order_items_dataset | CREATE TABLE `olist_order_items_dataset` (
  `order_id` varchar(255) NOT NULL,
  `order_item_id` int(11) DEFAULT NULL,
  `product_id` varchar(255) DEFAULT NULL,
  `seller_id` varchar(255) DEFAULT NULL,
  `price` decimal(19,2) DEFAULT NULL,
  `freight_value` decimal(19,2) DEFAULT NULL,
  PRIMARY KEY (`order_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 |
```

　　字段的顺序和自己在Entity中定义的是一样的，再将csv文件中的数据重新导入mysql，查看数据。

```mysql
mysql> select * from olist_order_items_dataset limit 5;
+----------------------------------+---------------+----------------------------------+----------------------------------+--------+---------------+
| order_id                         | order_item_id | product_id                       | seller_id                        | price  | freight_value |
+----------------------------------+---------------+----------------------------------+----------------------------------+--------+---------------+
| 00010242fe8c5a6d1ba2dd792cb16214 |             1 | 4244733e06e7ecb4970a6e2683c13e61 | 48436dade18ac8b2bce089ec2a041202 |  58.90 |         13.29 |
| 00018f77f2f0320c557190d7a144bdd3 |             1 | e5f2d52b802189ee658865ca93d83a8f | dd7ddc04e1b6c2c614352b383efe2d36 | 239.90 |         19.93 |
| 000229ec398224ef6ca0657da4fc703e |             1 | c777355d18b72b67abbeef9df44fd0fd | 5b51032eddd242adc84c38acab88f23d | 199.00 |         17.87 |
| 00024acbcdf0a6daa1e931b038114c75 |             1 | 7634da152a4610f1595efa32f14722fc | 9d7a1d34a5052409006425275ba1c2b4 |  12.99 |         12.79 |
| 00042b26cf59d7ce69dfabb4e55b4fd9 |             1 | ac6c3623068f30de03045865e4e10089 | df560393f3a51e74553ab94004ba5c87 | 199.90 |         18.14 |
+----------------------------------+---------------+----------------------------------+----------------------------------+--------+---------------+
5 rows in set (0.00 sec)
```





思考：　

​	至于为什么自己创建相同的包名.类名，然后将需要修改的代码替换，再启动就编译完成？　

　这就和类加载器和加载机制有关了，JVM用的是双亲委托的方式。具体加载机制后续再展开。



相关知识点和面试题：

​		jvm类加载机制，加载过程(装载，验证，准备，解析，初始化)（一个java类的生命周期）。启动类加载器，扩展类加载器，应用类加载器。

​		tomcat类加载机制

