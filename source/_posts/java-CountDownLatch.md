---
title: Java 线程同步器 之 CountDownLatch 原理剖析
subtitle: CountDownLatch 原理剖析
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - java
  - 编程语言
  - 并发
categories:
  - java
date: 2019-10-12 14:41:55
---


# CountDownLatch 原理剖析

## 1. 概述
&emsp;CountDownLatch是一个同步工具类，它允许一个或多个线程一直等待，直到其他线程的操作执行完后再执行。
&emsp;CountDownLatch使用一个计数器count实现，构建CountDownLatch时需要使用给定的count初始化CountDownLatch。在count到达0之前，调用await()方法的线程将一直阻塞，当count到达0时，会唤醒所有阻塞的线程。注意：计数器count**无法被重置**，即只能实现一次这种功能，这也是CountDownLatch与CyclicBarrier的区别。


## 2. 使用
&emsp;CountDownLatch 是一个通用同步工具，它有很多用途。将计数 1 初始化的 CountDownLatch 用作一个简单的开/关锁存器，或入口：在通过调用 countDown() 的线程打开入口前，所有调用 await 的线程都一直在入口处等待。用 N 初始化的 CountDownLatch 可以使一个线程在 N 个线程完成某项操作之前一直等待，或者使其在某项操作完成 N 次之前一直等待。

&emsp;提供的方法：
```java
//使当前线程阻塞直到计数器count变为0，除非被中断
public void await() throws InterruptedException
//使当前线程阻塞直到计数器count变为0，除非被中断或超过了指定时间
public boolean await(long timeout, TimeUnit unit) throws InterruptedException
//将计数器count递减，若count变为0则唤醒所有等待的线程
public void countDown()
//返回计数器count值
public long getCount()
```
&emsp;使用示例：
```java
/**
 * CountDownLatch 线程同步器测试
 */
public class CountDownLatchTest {
    private static volatile CountDownLatch countDownLatch = new CountDownLatch(2);

    static class MyRunnable implements Runnable{
        private String name;
        MyRunnable(String name){
            this.name = name;
        }
        @Override
        public void run() {
            try{
                Thread.sleep(1000);
            }catch (InterruptedException e){
                e.printStackTrace();
            }finally {
                System.out.println(name + " over");
                countDownLatch.countDown();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {

        ExecutorService executorService = Executors.newFixedThreadPool(2);
        executorService.submit(new MyRunnable("ThreadOne"));
        executorService.submit(new MyRunnable("ThreadTwo"));

        System.out.println("wait all child thread over!");

        //等待子线程执行完毕，返回
        countDownLatch.await();

        System.out.println("all child thread over!");

        executorService.shutdown();
    }
}
```

## 3. 实现原理
&emsp;CountDownLatch基于AQS实现，使用AQS的同步状态state表示计数器count。
&emsp;先看一下CountDownLatch的内部类Syns的实现，Syns继承了 **AbstractQueuedSynchronizer**：
```java
private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {
            setState(count);  //设置同步状态state为count
        }

        int getCount() {
            return getState();  //查询同步状态
        }
        //重写AQS的共享式获取同步状态的方法。当state=0时返回1，获取成功；当state=1时返回-1，获取失败。
        //acquires实际无用
        //该方法会被 await 方法调用，判断count是否减至0
        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }
        //重写AQS的共享式释放同步状态的方法。基于自旋CAS递减同步状态
        //会被countDown调用，计数器减1
        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            //如果state=0，那么直接返回false
            //如果state>0，那么递减state。若更新后的state=0则返回true，释放同步状态成功；反之，返回false。
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }
```
&emsp;由Sync源码可以看出，CountDownLatch基于AQS的共享式获取和释放同步状态的机制实现。

&emsp;接下来看一下await()方法，当线程调用 await 方法后，当前线程会被阻塞，下面的情况之一发生才返回：
- 计数器减为0
- 其它线程调用当前线程的 interrupt 方法中断当前线程，当前线程抛出 InterruptedException 异常，然后返回
```java
//调用AQS提供的共享式可中断获取同步状态方法。
//若获取成功（state=0），继续执行后续代码；否则（state>0）,阻塞当前线程。
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);    
}
public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        //响应中断
        //如果线程被中断则抛出异常
        if (Thread.interrupted())
            throw new InterruptedException();
        //判断计数器是否为0，为0则直接返回，否则进入AQS的队列等待    
        if (tryAcquireShared(arg) < 0)
            //阻塞，进入队列等待
            doAcquireSharedInterruptibly(arg);
    }
protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }
```
&emsp;await(long timeout, TimeUnit unit)方法和await方法类似，差别是设置的超时时间到了，会超时返回false。

&emsp;接下来看一下 countDown 方法，线程调用该方法，计数器的值递减，递减之后如果计数器值为0则唤醒所有因调用await方法而被阻塞的线程，否则什么都不做：
```java
public void countDown() {
        sync.releaseShared(1);
    }
public final boolean releaseShared(int arg) {
        //计数器减1
        if (tryReleaseShared(arg)) {
            //AQS释放资源
            doReleaseShared();
            return true;
        }
        return false;
    }
protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            //如果state=0，那么直接返回false
            //如果state>0，那么递减state。若更新后的state=0则返回true，释放同步状态成功；反之，返回false。
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
```
&emsp;如果当前状态值为0，则直接返回false，否则使用CAS循环重试将计数值减1，之后该需要唤醒因调用await方法而被阻塞的线程，具体在AQS的doReleaseShared方法中实现。

## 4. 总结 
&emsp;CountDownLatch 是使用 AQS 实现的，使用 AQS 的状态变量来存放计数器的值。首先在初始化 CountDownLatch 时设置状态值（计数器值），当多个线程调用 countdown 方法时实际是原子性递减 AQS 的状态值。当线程调用 await 方法后当前线程会被放入 AQS 的阻塞队列待计数器为0后被唤醒返回。其它线程调用 countdown 方法让计数器值递减1，当计数器值变为0时，当前线程还要调用 AQS 的 doReleaseShared 方法来激活由于调用 await 方法而被阻塞的线程。

&nbsp;
&nbsp;
> 参考：
https://www.cnblogs.com/zaizhoumo/p/7786893.html
《Java 并发编程之美》


---
相关推荐：
Java 并发包中线程同步器系列：
- [《Java 线程同步器 之 CountDownLatch 原理剖析》](http://zhoujiapeng.top/java/java-CountDownLatch)
- [《Java 线程同步器 之 CyclicBarrier 原理剖析》](http://zhoujiapeng.top/java/java-CyclicBarrier)
- [《Java 线程同步器 之 Semaphore 原理剖析》](http://zhoujiapeng.top/java/java-Semaphore)

&emsp;使用同步器有助于我们大大减少在Java中使用wait、noify等来实现线程同步的代码量，在日常开发中需要进行线程同步时使用这些同步类会节省很多代码并且保证正确性。