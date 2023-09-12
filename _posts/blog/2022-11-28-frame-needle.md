---
layout: post
title:  frame-needle
categories:  [Java]
description: 包装框架理解
keywords: frame,needle
---

frame-needle

---


needle框架大致理解

所谓needle，就是在springboot上封装了一层，主要包括这几个方面，依赖，工具，常用组件的通用包装。

依赖：会定期去维护一个版本，将springboot中某个固定版本所需要的依赖包固定下来。再默认置入常用的依赖，常见的meaven依赖问题，在这个里面直接就解决了。不过当遇到包装的一层本身依赖了带有漏洞的组件时，需要自己去调成新的无漏洞组件。

工具： 比如JSON，之前fastJson出过漏洞，大概看了一下，jackJson的封装。加密，指针或者集合判空等。

常用的组件类： JPA建表时记录的时间，雪花ID等。

BaseEntity

needle中的baseentity。 createTime updateTime, 雪花id

在orm的entity中，以表扩展的方式（orm处理继承的时候有好几种），来扩展这三个字段，其中两个时间字段所用注解的注解位于Javax中，并不是自己写的处理方式。

在本次项目中的可扩展处（数据源，数据集，数据服务，在分页处，每一个都需要获取其创建、更新的时间和人，在继承BaseEntity之前，自己写一个类继承BaseEntity，然后从UserInfo获取用户信息，则后续不需要在不同模块的业务代码中手动维护这几项）。

```java
package javax.persistence;
/**
 * Specifies a callback method for the corresponding 
 * lifecycle event. This annotation may be applied to methods 
 * of an entity class, a mapped superclass, or a callback 
 * listener class.
 *
 * @since 1.0
 */
@Target({METHOD}) 
@Retention(RUNTIME)
public @interface PrePersist {}
/**
 * Specifies a callback method for the corresponding 
 * lifecycle event. This annotation may be applied to methods 
 * of an entity class, a mapped superclass, or a callback 
 * listener class.
 *
 * @since 1.0
 */
@Target({METHOD}) 
@Retention(RUNTIME)
public @interface PreUpdate {}
```

雪花ID

数字位在内的64位二进制，1+41+10+12。 学习雪花算法的应用场景以及原理。

雪花算法在needle中的应用。

```java
int dcId = RandomUtil.randomInt(0, 31);
int workerId = RandomUtil.randomInt(0, 31);
snowflake = IdUtil.getSnowflake(workerId, dcId); 
//hutool的单例   
//IDUtils中原有的dataSetId,workerId分别用的mac和mac+进程pid  通过&取对应的二进制位, 如workid的(mpid.toString().hashCode() & 0xffff) % (maxWorkerId + 1);
public static Snowflake getSnowflake(long workerId) {
	return Singleton.get(Snowflake.class, workerId);
}
```

在本机上，是直接取的两个32以内的随机数，当成work,dataset的id，然后传给单例工厂。

Singleton关键代码如下：

```java
public final class Singleton {

	private static final SafeConcurrentHashMap<String, Object> POOL = new SafeConcurrentHashMap<>();
// SafeConcurrentHashMap这里用的是Safe的同步map，和其他封装了指针的java Safe类不一样，
//该类只重写了 同步hashmap的computeIfAbsent方法，防止分段锁极端情况下产生的死循环
	private Singleton() {
	}

    public static <T> T get(Class<T> clazz, Object... params) {
		Assert.notNull(clazz, "Class must be not null !");
		final String key = buildKey(clazz.getName(), params);
		return get(key, () -> ReflectUtil.newInstance(clazz, params));
	}

    private static String buildKey(String className, Object... params) {
       if (ArrayUtil.isEmpty(params)) {
          return className;
       }
       return StrUtil.format("{}#{}", className, ArrayUtil.join(params, "_"));
    }// 此处将class和参数列表（参数加顺序）作为唯一标识
    public static <T> T get(String key, Func0<T> supplier) {
		return (T) POOL.computeIfAbsent(key, (k)-> supplier.callWithRuntimeException());
	}
```

从单例工厂获取snowflake对象之后，再生成雪花ID，代码如下。雪花算法利用的时间戳，依赖于系统的时间，一旦系统时间出现错误，所生成的id要么跳段，要么重复（最常见的，一台物理机，win和linux双系统互存的时候，一定有个系统时间是乱的。 在hutool中，为了保证时间的正确性，不会使用本地的时间，而是从卫星或其他地方获取一个准确的时间，而由此会产生时间校验，也就多出了timeOffset参数，如果用的是本地时间，该参数没什么必要。

```java
public synchronized long nextId() {
   long timestamp = genTime();
   if (timestamp < this.lastTimestamp) {
      if (this.lastTimestamp - timestamp < timeOffset) {
         timestamp = lastTimestamp;
      } else {
         throw new IllegalStateException(StrUtil.format("Clock moved backwards. Refusing to generate id for {}ms", lastTimestamp - timestamp));
      }
   }

   if (timestamp == this.lastTimestamp) {
      final long sequence = (this.sequence + 1) & SEQUENCE_MASK;
      if (sequence == 0) {
         timestamp = tilNextMillis(lastTimestamp);
      }
      this.sequence = sequence;
   } else {
       // randomSequenceLimit 低频率下，一秒内只生成一个ID，sequence一直都是0，
       //所生成的雪花id尾数一定是偶数，通过该参数，再生成一个随机数，避免该情况
      if (randomSequenceLimit > 1) {
         sequence = RandomUtil.randomLong(randomSequenceLimit);
      } else {
         sequence = 0L;
      }
   }

   lastTimestamp = timestamp;

   return ((timestamp - twepoch) << TIMESTAMP_LEFT_SHIFT)
         | (dataCenterId << DATA_CENTER_ID_SHIFT)
         | (workerId << WORKER_ID_SHIFT)
         | sequence;
}
```

