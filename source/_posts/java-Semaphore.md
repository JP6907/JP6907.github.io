---
title: Java 线程同步器 之 Semaphore 原理剖析
subtitle: Semaphore 原理剖析
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - java
  - 编程语言
  - 并发
categories:
  - java
date: 2019-10-12 15:43:50
---



# Semaphore 原理剖析

## 1. 概述
&emsp;Semaphore 信号量也是 Java 中的一个同步器，与 CountDownLatch 和 CycleBarrier 不同的是，它内部的计数器是**递增**的，并且在一开始初始化 Semaphore 时可以指定一个初始值，但是并不需要知道同步的线程数量，而是在需要同步的地方调用 acquire 方法时指定需要同步的线程数量。
&emsp;信号量（Semaphore）控制同时访问资源的线程数量，支持公平和非公平两种方式。

提供的方法：
```java
public Semaphore(int permits)    //permits为许可数，默认非公平方式
public Semaphore(int permits, boolean fair)

//获取一个许可。若获取成功，permits-1，直接返回；否则当前线程阻塞直到有permits被释放，除非线程被中断
//如果线程被中断，则抛出 InterruptedException，并且清除当前线程的已中断状态。 
public void acquire() throws InterruptedException
//忽略中断
public void acquireUninterruptibly()
//尝试获取一个许可，成功返回true，否则返回false。
//即使已将此信号量设置为使用公平排序策略，但是调用 tryAcquire() 也将 立即获取许可（如果有一个可用），而不管当前是否有正在等待的线程。
public boolean tryAcquire()
//超时尝试获取一个许可，该方法遵循公平设置
public boolean tryAcquire(long timeout, TimeUnit unit)
//释放一个许可
public void release()

//以上方法都是获取或释放一个许可，每个方法都存在对应的获取或释放指定个数许可的方法。例如public boolean tryAcquire(int permits)
...

```

使用示例：
```java
public class SemaphoreTest {

    private static Semaphore semaphore = new Semaphore(0);

    private static class MyRunnable implements Runnable{

        @Override
        public void run() {
            try {
                System.out.println(Thread.currentThread() + " over");
                semaphore.release();
            }catch (Exception e){
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        ExecutorService executorService = Executors.newFixedThreadPool(2);
        executorService.submit(new MyRunnable());
        executorService.submit(new MyRunnable());
        //等待子线程执行完毕，返回
        semaphore.acquire(2);
        System.out.println("all child thread over");
        //semaphore可复用
        executorService.submit(new MyRunnable());
        executorService.submit(new MyRunnable());
        //等待子线程执行完毕，返回
        semaphore.acquire(2);
        System.out.println("all child thread over");
        executorService.shutdown();
    }
}
```

## 2. 实现原理
&emsp;Semaphore 基于AQS实现，用同步状态（state）表示许可数（permits），使用AQS的共享式获取和释放同步状态来实现permits的获取和释放。Semaphore 内部的 Sync 只是对 AQS 的一个修饰，并且 Sync 有两个实现类，用来制定获取信号量时是否采用公平策略：
```java
public Semaphore(int permits) {
        sync = new NonfairSync(permits); //默认使用非公平策略
    }
public Semaphore(int permits, boolean fair) {
        sync = fair ? new FairSync(permits) : new NonfairSync(permits);
    }
```

&emsp;下面看一下 acquire 方法：
```java
//阻塞式获取一个许可，响应中断
public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);    //调用AQS提供的可响应中断共享式获取同步状态方法
}
public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        //如果被中断，抛出异常
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
```
&emsp;tryAcquireShared 方法由 Sync 的子类实现，这里分别从两方法来讨论，先讨论非公平策略 NonfairSync 的 tryAcquireShared 方法：
```java
protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
        }
final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                //获取当前信号量的值
                int available = getState();
                //计算当前剩余值
                int remaining = available - acquires;
                //如果剩余值小于0或者 CAS 设置成功则返回
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
```
&emsp;如果当前信号量值不满足需要，即返回的remaining为负数，则会调用 doAcquireSharedInterruptibly 方法，会将当前线程放入 AQS 阻塞队列中挂起。
另外，由于 NonfairSync 是非公平获取的，也就是说先调用 acquire 方法的线程不一定比后来者先获取到信号量。考虑下面场景：线程A调用acquire而被阻塞挂起，此时线程 B 调用 release 释放足够的信号量，线程 C 在线程 A 被唤醒前调用了 acquire，则信号量可能被线程C 获取到。
下面看一下公平策略 FairSync 的实现：
```java
protected int tryAcquireShared(int acquires) {
            for (;;) {
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
```
&emsp;可以看到，公平性是由 hasQueuedPredecessors 来保证的，如果当前阻塞队列中有等待的线程，则当前线程无法获得信号量。

&emsp;acquire(int permits) 方法和 acquire 类似，不同的只是可以指定需要信号量数。
```java
public void acquire(int permits) throws InterruptedException {
        if (permits < 0) throw new IllegalArgumentException();
        sync.acquireSharedInterruptibly(permits);
    }
```
&emsp;acquireUninterruptibly 不同于 acquire 方法之处在于该方法对中断不响应
```java
public void acquireUninterruptibly() {
        sync.acquireShared(1);
    }
public final void acquireShared(int arg) {
        //这里没有检测中断标志
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
```

&emsp;接下来看一下 release 方法，该方法的作用是把当前 semaphore 对象的信号量值增加1，如果当前有线程因为调用 acquire 方法而被阻塞放进 AQS 队列，则会根据公平策略选择一个信号量个数能被满足的线程进行激活，激活的线程再次**尝试**获取刚增加的信号量。注意，这里并不一定能够成功获取，如果设置的是非公平策略，则可能被其它线程抢先获取。
```java
public void release(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        sync.releaseShared(permits);
    }
public final boolean releaseShared(int arg) {
        //tryReleaseShared 更新信号量的值
        if (tryReleaseShared(arg)) {
            //唤醒被阻塞的县城
            doReleaseShared();
            return true;
        }
        return false;
    }
protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                //获取当前信号量值
                int current = getState();
                //增加releases个信号量
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                    //使用CAS保证更新信号量的原子性
                if (compareAndSetState(current, next))
                    return true;
            }
        }
```

&nbsp;
&nbsp;
> 参考：
《Java 并发编程之美》


---
相关推荐：
Java 并发包中线程同步器系列：
- [《Java 线程同步器 之 CountDownLatch 原理剖析》](http://zhoujiapeng.top/java/java-CountDownLatch)
- [《Java 线程同步器 之 CyclicBarrier 原理剖析》](http://zhoujiapeng.top/java/java-CyclicBarrier)
- [《Java 线程同步器 之 Semaphore 原理剖析》](http://zhoujiapeng.top/java/java-Semaphore)

&emsp;使用同步器有助于我们大大减少在Java中使用wait、noify等来实现线程同步的代码量，在日常开发中需要进行线程同步时使用这些同步类会节省很多代码并且保证正确性。