---
layout: post
title: CDH6 安装
categories: [CDH]
description: CDH6 安装
keywords: CDH6
---

CDH6 安装

---

#### 前言

因有业务团队想要自己部署一套CDH6的集群。

前来寻求帮助，应该不难，但是肯定得踩坑。只能撸起袖子干

选择最新 6.1 版本

官方文档链接  https://www.cloudera.com/documentation/enterprise/latest.html 

选择自己要安装的版本。找到版本的官方的安装文档

![](/images/blog/2019-08-02-2.png){:height="80%" width="80%"}

安装之前建议先阅读要求

![](/images/blog/2019-08-02-3.png){:height="80%" width="80%"}

本人在安装过程中只关注了操作系统要求。

硬件是用公司物理机安装的。

数据库的话是用CM自带的 Psql

![](/images/blog/2019-08-02-4.png){:height="80%" width="80%"}

#### 系统环境

操作系统  CentOS 7.2 x64 

#### 安装之前

##### Host

![](/images/blog/2019-08-02-5.png){:height="80%" width="80%"}

主机YZ-JDB-106-38-13的名称中有大写字符。在这种情况下，通过Kerberos进行的身份验证将无法正常工作。

``` 
# 一定要注意 hostname 不能包含下划线
  
# 修改hostname
hostname yz-100-73-18-59  # 直接修改命令
hostname                  # 查看修改后的hostname
  
# 修改网络配置文件
vim /etc/sysconfig/network
HOSTNAME=yz-JDB-106-35-3
  
# 修改hosts文件
# 把集群所有的ip与对应的hostname
vim /etc/hosts     
100.73.41.20 yz-100-73-41-20
100.73.41.21 yz-100-73-41-21
100.73.41.22 yz-100-73-41-22
```

##### ssh免密

手动操作如下的代码块所示。 参考自[CentOS 配置集群机器之间SSH免密码登录](https://www.cnblogs.com/keitsi/p/5653520.html)

``` 
# 集群密码登录
# 集群中的每台主机上执行下面命令，一路回车，可生成本机的 rsa 类型的密钥
ssh-keygen -t rsa
# 执行完之后在 ~/.ssh/ 目录下会生成一个保存有公钥的文件：id_rsa.pub
   
# 把公钥写入 authorized_keys 文件
# 把自己的公钥拷贝到集群中的Master机
ssh-copy-id root@HadoopMaster
   
# 最终Master机上生成~/.ssh/authorized_keys文件   该文件保存所有机器的公钥
# 把~/.ssh/authorized_keys 拷贝到集群的其他机器上
scp ~/.ssh/authorized_keys root@HadoopSlave1:~/.ssh/
scp ~/.ssh/authorized_keys root@HadoopSlave2:~/.ssh/
```

##### 关闭防火墙

``` 
# 查看防火墙
systemctl status firewalld
 
# 关闭防火墙
systemctl stop firewalld
 
# 开机禁用
systemctl disable firewalld
```

##### 禁用 SELinux

``` 
# 关闭SELINUX
# 临时生效
setenforce 0
 
# 永久生效
vim /etc/selinux/config
SELINUX=disabled
```

##### 启用 NTP

这个是 SA 同事指导的。ntp 配置文件是他给的。

``` 
# 安装网络时间组件
yum -y install ntp
 
# 备份原先ntp配置文件
cp -p /etc/ntp.conf /etc/ntp.conf_`date +%F_%H-%M-%S`
 
# 配置ntp配置文件
vim /etc/ntp.conf
....

# 配置其他配置文件
vim /etc/sysconfig/ntpd
# Command line options for ntpd
OPTIONS="-4 -L -g"

# 停止影响的服务
systemctl stop chronyd.service
 
# 启动ntp服务
systemctl start ntpd.service

# 查看ntp服务
systemctl status ntpd.service

# 确认是否同步成功
ntpq -p  (若网络没问题，命令输出显示4台server并其中一台前会标记*号)
```

##### 虚拟内存设置
``` 
sysctl -w vm.swappiness=0
echo vm.swappiness = 0 >> /etc/sysctl.conf
```

##### 透明大页面设置

``` 
在root权限下，使用以下命令
# 编辑文件，增加以下两行
vim /etc/rc.d/rc.local
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag（重启服务器生效）
 
# shell 下
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag（即时生效，重启失效）
```

#### 安装

安装过程就是页面点点点了。这里略过

#### 遇到的问题记录

##### agent 依赖冲突

`Transaction check error:
file /usr/lib/systemd/system/supervisord.service from install of cloudera-manager-agent-6.1.0-769885.el7.x86_64 conflicts with file from package supervisor-3.1.3-3.el7.noarch`

根据报错提示知道依赖冲突。卸载冲突依赖, 然后安装agent 的是会安装它所依赖的 supervisor
``` 
rpm -e --nodeps supervisor-3.1.3-3.el7.noarch
```

##### 升级软件依赖版本

![](/images/blog/2019-08-02-6.png){:height="80%" width="80%"}

``` 
yum -y install python-pip
pip install --upgrade psycopg2
```

##### JDBC 驱动

![](/images/blog/2019-08-02-7.png){:height="80%" width="80%"}

``` 
cp mysql-connector-java-5.1.47-bin.jar /usr/share/java
mv mysql-connector-java-5.1.47-bin.jar mysql-connector-java.jar
```

##### 数据库没有创建

Able to find the Database server, but not the specified database. Please check if the database name is correct and make sure that the user can access the database.

![](/images/blog/2019-08-02-8.png){:height="80%" width="80%"}

解决方法：创建需要的数据库

![](/images/blog/2019-08-02-9.png){:height="80%" width="80%"}

##### 开启 gtid 问题

User cannot run DDL statements on the specified database. Attempt to create and drop a table failed.

因为 mysql 开启了gtid ，所以导致这个问题。

关闭 mysql 的gtid 即可。

修改 mysql的 my.cnf 配置

``` 
# GTID #
gtid-mode = OFF
enforce-gtid-consistency = 0
```

#### Kudu 路径配置

Cloudera 默认没有给 Kudu 配置路径

![](/images/blog/2019-08-02-10.png){:height="80%" width="80%"}

#### Hive Metastore Canary

Hive 需要 jdbc 的包

![](/images/blog/2019-08-02-11.png){:height="80%" width="80%"}

``` 
cp mysql-connector-java.jar /usr/share/java
cp mysql-connector-java.jar /opt/cloudera/parcels/CDH-6.1.0-1.cdh6.1.0.p0.770702/lib/hive/lib/
```

---
参考链接
* [按照CDH时创建 Hive Metastore 数据库表失败](https://segmentfault.com/q/1010000013538821/a-1020000013541726)
* [hive 未初始化元数据库报错](https://www.cnblogs.com/zwgblog/p/6063993.html)







