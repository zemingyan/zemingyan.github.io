---
layout: post
title: 使用 Flink 时遇到的问题
categories: [Flink]
description: 使用 Flink 时遇到的问题
keywords: Flink
---

使用Flink时遇到的问题（不断更新中）

---

#### 前言

作为公司第一个吃 Flink 这支螃蟹的人肯定免不了踩坑，因此通过文字记录的方式记录踩坑经历

#### Could not build the program from JAR file.

用 Flink1.8 提交 jar 包到 Yarn 集群运行时报错

``` 
Could not build the program from JAR file.

Use the help option (-h or --help) to get help on the command.
```

这是需要配置 Hadoop Classpaths。 [Configuring Flink with Hadoop Classpaths](https://ci.apache.org/projects/flink/flink-docs-stable/ops/deployment/hadoop.html#configuring-flink-with-hadoop-classpaths)

``` 
export HADOOP_CLASSPATH=`hadoop classpath`
```

#### UnsupportedClassVersionError

用 Flink1.8 提交 jar 包到 Yarn 集群运行时报 UnsupportedClassVersionError 错误

``` 
Exception in thread "main" java.lang.UnsupportedClassVersionError: org/apache/flink/client/cli/CliFrontend : Unsupported major.minor version 52.0
        at java.lang.ClassLoader.defineClass1(Native Method)
        at java.lang.ClassLoader.defineClass(ClassLoader.java:800)
        at java.security.SecureClassLoader.defineClass(SecureClassLoader.java:142)
        at java.net.URLClassLoader.defineClass(URLClassLoader.java:449)
        at java.net.URLClassLoader.access$100(URLClassLoader.java:71)
        at java.net.URLClassLoader$1.run(URLClassLoader.java:361)
        at java.net.URLClassLoader$1.run(URLClassLoader.java:355)
        at java.security.AccessController.doPrivileged(Native Method)
        at java.net.URLClassLoader.findClass(URLClassLoader.java:354)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:425)
        at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:308)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:358)
        at sun.launcher.LauncherHelper.checkAndLoadMain(LauncherHelper.java:482)
```

Java 版本不对, Flink1.8 需要 JDK1.8 以上。修改环境变量 JAVA_HOME





