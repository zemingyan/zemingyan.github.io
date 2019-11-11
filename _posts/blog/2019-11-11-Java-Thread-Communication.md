---
layout: post
title:  Java线程通信
categories: [Thread]
description: Java线程通信
keywords: Thread,Thread Pool
---
Java线程通信
---



在操作系统中，线程之间通信方式最常见的有四种: 临界区，信号量，互斥量，事件。

在java中临界区是用的最多的，在同一个进程内共享资源，效率最高。

同步互斥关系-----同步是互斥更进一步的约束。　



##### **场景一 :  线程交替执行**

​		同步例子，生产-消费者模式。　同步锁(wai,notify)和重入锁(await,signal)的实现方式，和操作系统中的pv操作思想很像。

​		也可以使用阻塞队列，思想都是一样的，只是阻塞队列已经实现了临界状态下的处理(满的时候不能在put,空的时候不能再take)。　信号量的思想和读写锁类似。

​		关于两个线程之间交替执行的场景，只是生产消费者模式的一种微型变化。

```java
// 以下是同步锁
public class Test {
    public static void main(String[] args) {
        Test test = new Test();
        new Thread(new Producer()).start();
        new Thread(new Consumer()).start();
        new Thread(new Producer()).start();
        new Thread(new Consumer()).start();
    }
}
class Utils{
    public static  Integer count = 0;

    public static final Integer FULL = 10;
    public static String LOCK = "lock";
}
class Producer implements Runnable {
    @Override
    public void run() {
        for (int i = 0; i < 10; i++) {
            try {
                Thread.sleep(3000);
            } catch (Exception e) {
                e.printStackTrace();
            }
            synchronized (Utils.LOCK) {
                while (Utils.count == Utils.FULL) {
                    try {
                        Utils.LOCK.wait();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
                Utils.count++;
                System.out.println(Thread.currentThread().getName() + "生产者生产，目前总共有" + Utils.count);
                Utils.LOCK.notifyAll();
            }
        }
    }
}
class Consumer implements Runnable {
    @Override
    public void run() {
        for (int i = 0; i < 10; i++) {
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            synchronized (Utils.LOCK) {
                while (Utils.count == 0) {
                    try {
                        Utils.LOCK.wait();
                    } catch (Exception e) {
                    }
                }
                Utils.count--;
                System.out.println(Thread.currentThread().getName() + "消费者消费，目前总共有" + Utils.count);
                Utils.LOCK.notifyAll();
            }
        }
    }
}
```

```java
//重入锁
public class Test2 {
    public static void main(String[] args) {
        Test2 test2 = new Test2();
        new Thread(new Producer()).start();
        new Thread(new Consumer()).start();
        new Thread(new Producer()).start();
        new Thread(new Consumer()).start();

    }
}
class Utils2 {
    public static Integer count = 0;
    public static final Integer FULL = 10;
    public static Lock lock = new ReentrantLock();
    public static final Condition notFull = lock.newCondition();
    public static final Condition notEmpty = lock.newCondition();
}
class Producer implements Runnable {
    @Override
    public void run() {
        for (int i = 0; i < 10; i++) {
            try {
                Thread.sleep(3000);
            } catch (Exception e) {
                e.printStackTrace();
            }
            Utils2.lock.lock();
            try {
                while (Utils2.count == Utils2.FULL) {
                    try {
                        Utils2.notFull.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                Utils2.count++;
                System.out.println(Thread.currentThread().getName()
                        + "生产者生产，目前总共有" + Utils2.count);
                //唤醒消费者
                Utils2.notEmpty.signal();
            } finally {
                //释放锁
                Utils2.lock.unlock();
            }
        }
    }
}


class Consumer implements Runnable {
    @Override
    public void run() {
        for (int i = 0; i < 10; i++) {
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e1) {
                e1.printStackTrace();
            }
            Utils2.lock.lock();
            try {
                while (Utils2.count == 0) {
                    try {
                        Utils2.notEmpty.await();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
                Utils2.count--;
                System.out.println(Thread.currentThread().getName()
                        + "消费者消费，目前总共有" + Utils2.count);
                Utils2.notFull.signal();
            } finally {
                Utils2.lock.unlock();
            }
        }
    }
}
```



​		关于线程的join方法之前有说过，当前线程等待调用t.join方法的线程(t)的锁。　也就是t线程执行释放锁以后在执行当前线程，当前线程可以是main线程，也可以是其他线程，当有三个或者以上线程并发的时候，调用join方法还会有多个线程在并发。两个线程时候调用join就变成了串行。



##### **场景二：某线程等待多个线程执行完毕以后在执行**

​	 比如多线程的希尔排序，后一轮的排序需要前面一轮排序结束。

第一轮步长为len/2的时候，len/2个线程分别排序。等len/2个线程排序结束以后，第二轮重新选步长，再开始排序用多个线程排序。



```java
public class Test {
    static ExecutorService es = Executors.newFixedThreadPool(2);
    static int[] array = {1, 3, 4, 5, 2};

    public static class shellSort implements Runnable {
        static int d;
        static int start;
        static CountDownLatch latch;

        public shellSort(int d, int start, CountDownLatch latch) {
            this.d = d; // 步长
            this.start = start;
            this.latch = latch;
        }

        @Override
        public void run() {
            for (int i = start; i < array.length; i++) {
                int number = array[i];
                int j = i - d;
                while (j >= 0 && array[j] > number) {
                    array[j + d] = array[j];
                    j -= d;
                }
                array[j + d] = number;
                latch.countDown();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException{
        int d = array.length;
        while (d > 0) {
            d /= 2;
            CountDownLatch latch = new CountDownLatch(d);
            for (int i = 0; i < d; i++) {
                es.submit(new shellSort(d, i, latch));
            }
            latch.await();
        }
    }
}
```



##### **场景三　一组线程都准备好以后在同时执行**

​		和上一个场景最大的不同之处是:上面希尔排序中，一个步长的排序线程组结束后，会循环迭代。而且需要重新开线程调整步长再排序。　　　

​		线程A预处理需要200毫米左右，Ｂ线程需要400毫秒左右，C线程需要500毫秒左右。当它们全部预处理完毕以后再一起执行后续操作。这里预处理操作和后续操作可以放在同一个线程中，如果用场景二那样，则变成了，全部预处理结束，再全部执行。　弊端：1.原本一个线程可以完成的操作变成了预处理线程和执行线程两个线程。　2.新的执行操作的线程，并不是同时开始的。因为在调用start方法的时候，是有先后顺序的，如果拆分，后续操作显然也就有了先后顺序，在时效性很强的场合下会出问题（比如运动会比赛，多个参赛者必须可以在同一时间开始，而不是人为的规定了先后顺序）。

```java

public class TestCyclic {
    public static void main(String[] args){
        cyclicBarrierTest();
    }
    private static void cyclicBarrierTest(){
        int runner = 3;
        CyclicBarrier cyclicBarrier = new CyclicBarrier(runner);
        final Random random = new Random();

        for(char runnerName = 'A'; runnerName <= 'C'; runnerName++){
            final String rName = String.valueOf(runnerName);

            new Thread(new Runnable(){

                public void run() {
                    long prepareTime = random.nextInt(10000) + 100;
                    System.out.println(rName + ":预处理的时间：" + prepareTime);
                    try {
                        Thread.sleep(prepareTime);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println(rName + "预处理完毕，等待其他");
                    try {
                        cyclicBarrier.await();
                    } catch (InterruptedException | BrokenBarrierException e) {
                        e.printStackTrace();
                    }
                    //所有线程预处理完毕
                    System.out.println(rName + "开始执行后续操作");
                }
            }).start();
        }
    }
}
```



##### **场景四　:异步执行，获取子线程返回结果**

​		除了Thread,Runnable以外，Callable也可以开新的线程实现异步，并且得到返回结果，获取返回结果的方式有两种，Future和FutureTask。

```java
public class TestCyclic {
    public static void main(String[] args){
        future();
    }
    private static void future(){

        Callable<Integer> callable = new Callable<Integer>(){
            public Integer call() throws Exception {
                System.out.println("任务开始");
                Thread.sleep(1000);
                int result = 0;
                for(int i=0;i<=100;i++){
                    result += i;
                }
                System.out.println("任务结束并且返回结果");
                return result;
            }
        };

        FutureTask<Integer> futureTask = new FutureTask<>(callable);
        new Thread(futureTask).start();


        try {
            System.out.println("主线程调用 futureTask.get()之前");
            System.out.println("Result:" + futureTask.get());
            System.out.println("主线程调用 futureTask.get()之后");
            //也可以用线程池.submit来获得Future
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
    }

}

```







