---
layout: post
title:  Java 线程＆线程池
categories: [Thread]
description: Java 线程＆线程池
keywords: Thread,Thread Pool
---
Java线程池

---
# 线程&线程池

java中线程的状态　(具体的英文注释删了，太多占位置)　和操作系统中的进程有点像，但是不一样（因为线程是在java虚拟机里面定义的，而进程是在操作系统内定义的。是两个不同层面的问题）

```java
 public enum State {
        NEW, 
        RUNNABLE,
        BLOCKED,
        WAITING,
        TIMED_WAITING,
        TERMINATED;
    }
```

![](/images/blog/2019-10-21-java-thread.webp){:height="80%" width="80%"}

**NEW **:当使用new Thread()创建一个新的线程，又还没有开始执行（not yet started）它的时候就处于NEW状态。
**TERMINATED**:终止状态

**RUNNABLE** :正在Java虚拟机中执行。

**BLOCKED**：等待监monitor锁　(在同步锁中，会在编译期在花括号两端加上monitor  exit monitor。 重入锁则是用户自己通过 lock.lock()  lock.unlock() ，然后生成)

**WAITING**：无限期等待另一个线程执行一个特别的动作。

**TIMED_WAITING**：限时等待另一个线程执行一个动作。



在操作系统层面的ready,running状态在java线程中没有。因为线程是为了更好的实现并发，在线程中一个线程进行上下文切换的时候非常迅速，如果一个cpu时间片在10-20ms，在不计算其他io阻塞，线程挂起睡眠的情况下，每秒钟需要在ready和running之间切换几十次上百次。　这种频繁变动不止耗费资源，线程监督服务甚至捕捉不到它的状态，最少人通过工具去查看的时候看不到实时状态。JVM 实现都把 Java 线程一一映射到操作系统底层的线程上，把调度委托给了操作系统，我们在虚拟机层面看到的状态实质是对底层状态的映射及包装。



##### 常用的方法：

Thread.run()  普通方法，线程执行的代码块

Thread.start() c++写的本地方法。必须通过start()来启动一个线程，如果直接调用run,就变成了平常的方法调用

Thread.sleep()/sleep(long millis)

Thread.yield()  让出CPU的使用权，给其他线程执行机会、让同等优先权的线程运行（但并不保证当前线程会被JVM再次调度、使该线程重新进入Running状态），如果没有同等优先权的线程，那么yield()方法将不会起作用

Thread.join()  使用该方法的线程会在此之间执行完毕后再往下继续执行(类似于插队)。

​					join方法的底层原理是(调用了wait)，当在当前线程(比如main线程)里面调用另一个线程(比如t线程)的			join方法时，当前线程(main线程)会去等待另一线程(t1线程)的锁---相当于main线程调用了t1.wait();需要等			到t1线程执行完毕才能继续。

​           　　join实例:  如下，如果将t1.join注释掉， t1,t2线程内的run方法会并发运行。

```java
Thread t1 = new Thread(new Runnable(){...});
Thread t2 = new Thread(new Runnable(){...});
t1.start();　// main线程开出了t1线程去执行，当t1的run方法执行以后，会出现main和t1并发执行的情况
t1.join();　// main程会等待t1线程执行完毕，然后继续执行
t2.start();　// main线程开启另一个支线程t2
...  
    //最终结果是，t1线程的run方法先执行，然后main才会开启t2线程。　接着t2线程执行run方法
    
     //事例二　　当改成以下代码的时候
t1.start();  // main线程启动t1
t2.start();  // main线程启动t2
t1.join();   // main线程等待t1执行完毕　
...           
    //　由于main线程开启了t1和t2，在main线程调用t1.join之前，是三个线程的并发情况。　之后虽然调用了t1.join()但是t2线程并不会进入等待状态，也就是说t1,t2在并发，main在等待t1。　　　
    //如果想要　t1的run方法执行一般以后执行t2，然后再执行t1. 可以在t1的run方法中间调用t2.join()
    
```



```java
public　class Thread implements Runnable {
@Deprecated　public final void suspend()
@Deprecated　public final void resume() 
}
```

两个方法配套使用，suspend()使得线程进入阻塞状态，并且不会自动恢复，必须其对应的resume() 被调用，才能使得线程重新进入可执行状态。典型地，suspend() 和 resume() 被用在等待另一个线程产生的结果的情形：测试发现结果还没有产生后，让线程阻塞，另一个线程产生了结果后，调用 resume() 使其恢复。      线程不安全，已经过时





Object.wait()  当一个线程执行到wait()方法时，他就进入到一个和该对象相关的等待池(Waiting Pool)中，同时失去了对象的机锁—暂时的，wait后还要返还对象锁。当前线程必须拥有当前对象的锁，如果当前线程不是此锁的拥有者，会抛出IllegalMonitorStateException异常,所以wait()必须在synchronized block中调用。

Object.notify()/notifyAll() 唤醒在当前对象等待池中等待的第一个线程/所有线程。notify()/notifyAll()也必须拥有相同对象锁，否则也会抛出IllegalMonitorStateException异常。



实现生产－消费者模式的时候，可以用的wait notify（区别一下同步锁，　同步锁是提供线程之间的互斥，保证线程安全。而wait notify是线程之间的通讯方式) (*这两个方法效率低，在jdk1.5以后提供了Lock接口和Condition对象。Condition中的await(), signal().signalAll()代替Object中的wait(),notify(),notifyAll()*)

###### **常见面试题：　wait notify 为什么是属于Object而不是Thread**

因为这些方法在操作同步线程时，都必须要标识它们操作线程的锁，只有同一个锁上的被wait线程，可以被同一个锁上的notify唤醒，不可以对不同锁中的线程进行唤醒。也就是说，等待和唤醒必须是同一个锁。而锁可以是任意对象，所以可以被任意对象调用的方法是定义在object类中。

##### 关于这些方法的使用，在锁里面最好体现，如多线程的希尔排序





### 　　　　　　　　　　　　　　　　　　线程池

​	面试时Leader问我的问题－－线程池类型

我的回答：cache,single,fix，但是阿里java开发手册里面推介参数齐全的ThreadPoolExecutor，前面三个线程池都是调用的ThreadPoolExecutor.

```java
public ThreadPoolExecutor(int corePoolSize, 
                          int maximumPoolSize, 
                          long keepAliveTime,
                          TimeUnit unit,                          
                          BlockingQueue<Runnable> workQueue,                         		 						   ThreadFactory threadFactory,                          									   RejectedExecutionHandler handler)
    
```



这是TheadPoolExecutor参数最齐全的构造函数，

###### corePoolSize

线程池的核心线程数。在没有设置allowCoreThreadTimeOut为true的情况下，核心线程会在线程池中一直存活，即使处于闲置状态。

###### maximumPoolSize

线程池允许创建的最大线程数。如果队列满了，并且已创建的线程数小于最大线程数，则线程池会再创建新的线程执行任务。值得注意的是如果使用了无界的任务队列这个参数就没什么效果。
如果运行的线程少于 corePoolSize，则 Executor始终首选添加新的线程，而不进行排队。（如果当前运行的线程小于corePoolSize，则任务根本不会存放，添加到queue中，而是直接抄家伙（thread）开始运行）
如果运行的线程等于或多于 corePoolSize，则 Executor始终首选将请求加入队列，而不添加新的线程。
如果无法将请求加入队列，则创建新的线程，除非创建此线程超出 maximumPoolSize，在这种情况下，任务将被拒绝。

###### keepAliveTime

非核心线程闲置时的超时时长。超过该时长，非核心线程就会被回收。若allowCoreThreadTimeOut属性为true时，该时长同样会作用于核心线程。

###### unit

keepAliveTime的时间单位。

###### workQueue

线程池中的任务队列，通过线程池的execute()方法提交的Runnable对象会存储在该队列中

###### threadFactory

线程工厂，为线程池提供创建新线程的功能。ThreadFactory是一个接口。Executors中提供了DefaultThreadFactory。

###### RejectedExecutionHandler

the handler to use when execution is blocked because the thread bounds and queue capacities are reached。当任务无法被执行时(超过线程最大容量maximum并且queue已经被排满了)的处理策略。默认为AbortPolicy，直接抛出异常。当队列和线程池都满了，说明线程池处于饱和状态，那么必须采取一种策略处理提交的新任务。

#### 总的来说，关于线程池中线程的优先级：

#### corePoolSize > workQueue >maximumPoolSize 

#### 如果全部都满了，就会执行拒绝策略，默认是直接抛出异常。

```java
public static ExecutorService newFixedThreadPool(int nThreads) {   
    return new ThreadPoolExecutor(nThreads,
                                  nThreads,                                  
                                  0L, 
                                  TimeUnit.MILLISECONDS,                                  									  new LinkedBlockingQueue<Runnable>());}

public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService(
            		new ThreadPoolExecutor(1,
                                           1,
                                    	   0L, 
                                           TimeUnit.MILLISECONDS,
                                    	   new LinkedBlockingQueue<Runnable>()));
    }

public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, 
                                      Integer.MAX_VALUE,
                                      60L, 
                                      TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```

其中当队列，线程工厂和拒绝策略没有制定的时候都是使用默认的值：workQueue,     threadFactory, defaultHandler。





关于Executors家族谱，　线程引出的锁问题，以及AQS(抽象队列同步器),之后再讲。