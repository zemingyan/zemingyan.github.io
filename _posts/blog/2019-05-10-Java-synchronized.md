---
layout: post
title: 浅学 Synchronized
categories: [Java]
description: Synchronized 是并发编程中重要的使用工具之一
keywords: Synchronized
---

Synchronized 是并发编程中重要的使用工具之一

---

#### 前言

JVM 自带的关键字，可在需要线程安全的业务场景中使用，来保证线程安全。

#### 用法

> * 按照锁的对象区分可以分为对象锁和类锁
> * 按照在代码中的位置区分可以分为方法形式和代码块形式

##### 对象锁

锁对象为当前 this 或者说是当前类的实例对象

``` 
// 对象锁方法形式
public synchronized void method() {
	...
}

// 对象锁代码块形式
public void method() {
	synchronized (this) {
		...
	}
}
```
##### 类锁

锁的是当前类或者指定类的Class对象。

一个类可能有多个实例对象，但它只可能有一个Class对象。

``` 
public static void synchronized method() {
    System.out.println("我是静态方法形式的类锁");
}

public void method() {
    synchronized(*.class) {
        System.out.println("我是代码块形式的类锁");
    }
}
```


#### 简单举例

可以参看[深入理解synchronized关键字][1]、[Java并发编程：Synchronized及其实现原理][2]。

原理也可以参看上面两个文章。

#### 总结

一把锁只能同时被一个线程获取，没有拿到锁的线程必须等待；

每个实例都对应有自己的一把锁，不同实例之间互不影响；

锁对象是*.class以及synchronized修饰的static方法时，所有对象共用一把类锁；

无论是方法正常执行完毕或者方法抛出异常，都会释放锁；

使用 synchronized 修饰的方法都是可重入的

``` 
public class SynchronizedRecursion {
    int a = 0;
    int b = 0;
    private void method1() {
        System.out.println("method1正在执行，a = " + a);
        if (a == 0) {
            a ++;
            method1();
        }
        System.out.println("method1执行结束，a = " + a);
    }

    private synchronized void method2() {
        System.out.println("method2正在执行，b = " + b);
        if (b == 0) {
            b ++;
            method2();
        }
        System.out.println("method2执行结束，b = " + b);
    }

    public static void main(String[] args) {
        SynchronizedRecursion synchronizedRecursion = new SynchronizedRecursion();
        synchronizedRecursion.method1();
        synchronizedRecursion.method2();
    }
}
结果为：
    method1正在执行，a = 0
    method1正在执行，a = 1
    method1执行结束，a = 1
    method1执行结束，a = 1

    method2正在执行，b = 0
    method2正在执行，b = 1
    method2执行结束，b = 1
    method2执行结束，b = 1
```

可以看到method1()与method2()的执行结果一样的，method2()在获取到对象锁以后，在递归调用时不需要等上一次调用先释放后再获取，而是直接进入，这说明了synchronized的可重入性。

当然，除了递归调用，调用同类的其它同步方法，调用父类同步方法，都是可重入的，前提是同一对象去调用，这里就不一一列举了.

---
参考链接
* [深入理解synchronized关键字][1]
* [Java并发编程：Synchronized及其实现原理][2]



[1]: https://mp.weixin.qq.com/s/Zwl3fUyO4igP6wE5W5IwYw
[2]: https://www.cnblogs.com/paddix/p/5367116.html


