---
title: 线程池 之 ThreadPoolExecutor 状态控制
subtitle: ThreadPoolExecutor 状态控制
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - java
  - 编程语言
  - 并发
categories:
  - java
date: 2019-09-26 09:28:04
---



# 线程池 之 ThreadPoolExecutor 状态控制

&emsp;ThreadPoolExecutor 内部使用了大量位运算，以下是ThreadPoolExecutor状态控制的主要变量和方法：
```java
//ThreadPoolExecutor.java
    //原子状态控制数
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    //29比特位
    private static final int COUNT_BITS = Integer.SIZE - 3;
    //实际容量 2^29-1
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits
    // runState存储在高位中
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;

    // Packing and unpacking ctl 打包和解压ctl

    // 解压runState
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    // 解压workerCount
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    // 打包ctl
    private static int ctlOf(int rs, int wc) { return rs | wc; }
```
&emsp;**线程池使用一个AtomicInteger的ctl变量将 workerCount（工作线程数量）和 runState（运行状态）两个字段压缩在一起**，这种做法在在java源码里经常有出现，如在 ReentrantReadWriteLock 里就将一个int分成高16位和底16位，分别表示读锁状态和写锁状态。ThreadPoolExecutor里也是使用了同样的思想，表现得更加复杂。
&emsp;ThreadPoolExecutor用ctl的最高3个比特位表示runState， 低29个比特位表示workerCount。因此这里需要特别说明的是：
> 确切的说，当最大线程数量配置为Integer.MXA_VAULE时，ThreadPoolExecutor的线程最大数量依然是2^29-1。

&emsp;目前来看这是完全够用的，但随着计算机的不断发展，真的到了不够用的时候可以改变为AtomicLong。这如同32位系统时间戳会在2038年01月19日03时14分07秒耗尽一样，当以后我们的系统线程能够超过2^29-1时，这些代码就需要调整了。对于未来，无限可能。

&emsp;思考一下为什么是29：3呢？
这是因为我们的运营状态有5种，向上取2次方数，2^3 = 8。所以必须要3个比特位来表示各种状态。

&emsp;运行状态解释：
| 状态 | 解释 |
| ---- | --- | 
|RUNNING |运行态，可处理新任务并执行队列中的任务 |
|SHUTDOW |关闭态，不接受新任务，但处理队列中的任务 |
|STOP |停止态，不接受新任务，不处理队列中任务，且打断运行中任务 |
|TIDYING |整理态，所有任务已经结束，workerCount = 0，将执行terminated()方法 |
|TERMINATED |结束态，terminated() 方法已完成 |

&emsp;整个ctl的状态，会在线程池的不同运行阶段进行CAS转换。

线程池运行状态的转换如下：
1. 线程池在RUNNING状态下调用shutdown()方法会进入到SHUTDOWN状态，（finalize()方法也会调用shutdownNow()）。
2. 在RUNNING和SHUTDOWN状态下调用 shutdownNow() 方法会进入到STOP状态。
3. 在SHUTDOWN状态下，当阻塞队列为空且线程数为0时进入TIDYING状态；在STOP状态下，当线程数为0时进入TIDYING状态。
4. 在TIDYING状态，调用terminated()方法完成后进入TERMINATED状态。

![staus](https://gitee.com/JP6907/Pic/raw/master/java/threadpoolexecutor-status.png?raw=true)

&emsp;另外，可以通过线程池的以下属性监控线程池的当前状态：

- getTaskCount：返回曾计划执行的近似任务总数。因为在计算期间任务和线程的状态可能动态改变，所以返回值只是一个近似值。
- getCompletedTaskCount：返回已完成执行的近似任务总数。因为在计算期间任务和线程的状态可能动态改变，所以返回值只是一个近似值，但是该值在整个连续调用过程中不会减少。
- getLargestPoolSize：线程池曾经创建过的最大线程数量，通过这个数据可以知道线程池是否满过。如等于线程池的最大大小，则表示线程池曾经满了。
- getPoolSize：线程池的线程数量。
- getActiveCount：返回主动执行任务的近似线程数。

&emsp;通过扩展线程池进行监控：继承线程池并重写线程池的beforeExecute()，afterExecute()和terminated()方法，可以在任务执行前、后和线程池关闭前自定义行为。如监控任务的平均执行时间，最大执行时间和最小执行时间等。

&emsp;使用ThreadPoolExecutor直接创建线程池时，可以使用第三方的ThreadFactory，或者自己实现ThreadFactory接口，拓展更多的属性，例如设置线程名称、执行开始时间、优先级等等。

```java
class PausableThreadPoolExecutor extends ThreadPoolExecutor {
    private boolean isPaused;    //标志是否被暂停
    private ReentrantLock pauseLock = new ReentrantLock();    //访问isPaused时需要加锁，保证线程安全
    private Condition unpaused = pauseLock.newCondition();

    public PausableThreadPoolExecutor(...) { super(...); }
    
    //beforeExecute为ThreadPoolExecutor提供的hood方法
    protected void beforeExecute(Thread t, Runnable r) {
        super.beforeExecute(t, r);
        pauseLock.lock();
        try {
            while (isPaused) 
                unpaused.await();
        } catch(InterruptedException ie) {
            t.interrupt();
        } finally {
            pauseLock.unlock();
        }
    }
    //暂停
    public void pause() {
        pauseLock.lock();
        try {
            isPaused = true;
        } finally {
            pauseLock.unlock();
        }
    }
    //取消暂停
    public void resume() {
        pauseLock.lock();
        try {
            isPaused = false;
            unpaused.signalAll();
        } finally {
            pauseLock.unlock();
        }
    }
}
```


&nbsp;
&nbsp;
> 参考：
https://www.jianshu.com/p/18065a78178b 
https://blog.csdn.net/wtopps/article/details/80682267
https://www.cnblogs.com/zaizhoumo/p/7794818.html

> (文章只做学习记录，非商业用途。如有侵权,请联系删除！)