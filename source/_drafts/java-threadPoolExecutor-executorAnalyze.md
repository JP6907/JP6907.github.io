---
title: 线程池 之 ThreadPoolExecutor 执行原理
catalog: true
subtitle: ThreadPoolExecutor 执行原理
header-img: "/img/article_header/article_header.png"
tags:
- java
- 编程语言
- 并发
categories:
- java
---

# ThreadPoolExecutor 执行原理

&emsp;ThreadPoolExecutor 的实现实际是一个生产消费模型，当用户添加任务到任务池时相当于生产者生产元素，workers线程工作集中的线程直接执行任务或者从任务队列里面获取任务时则相当于消费者消费元素。
&emsp;用户线程提交任务时的execute方法如下：
```java
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        // 1、工作线程 < 核心线程 ，创建线程执行
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        // 2、运行态，并尝试将任务加入队列
        if (isRunning(c) && workQueue.offer(command)) {
            //二次检查
            int recheck = ctl.get();
            //如果不是RUNNING则从队列中删除任务，并执行拒绝策略
            if (! isRunning(recheck) && remove(command))
                reject(command);
            //如果当前线程池为空，则添加一个线程
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        } // 3、队列已满，新增线程使用尝试使用最大线程运行
        else if (!addWorker(command, false))
            reject(command);
    }
```
&emsp;execute 的判断逻辑可用下图表示:
![线程任务处理流程](https://github.com/JP6907/Pic/blob/master/java/thread-process-task.png?raw=true)
1. 如果当前线程池任务线程数量小于核心线程池数量，会向 wokers 里面新增一个核心线程执行该任务。
2. 如果当前线程池任务线程数量大于核心线程池数量，如果处于 RUNNING 状态，则尝试将任务添加到任务队列，这里需要重新判断线程池状态是因为这里 exexute 不是原子操作，此时可能已经处于非 RUNNING 状态，如果处于非 RUNNING 状态则需要抛弃该新任务，将任务从队列中移出。
3. 如果当前线程池任务线程数量大于核心线程池数量，且任务队列已满，将会创建一个新的任务线程，直到超出 maximumPoolSize，如果超出 maximumPoolSize，则任务将会被拒绝，通过设定的拒绝策略来处理。

&emsp;在execute方法中，用到了double-check的思想，我们看到上述代码中并没有同步控制，都是基于乐观的check，如果任务可以创建则进入addWorker(Runnable firstTask, boolean core)方法，注意上述代码中的三种传参方式：
- addWorker(command, true)： 创建核心线程执行任务；
- addWorker(command, false)：创建非核心线程执行任务；
- addWorker(null, false)： 创建非核心线程，当前任务为空(会从任务队列去取)；

&emsp;addWorker的返回值是boolean，不保证操作成功。下面详看addWorker方法：
```java
private boolean addWorker(Runnable firstTask, boolean core) {
    // 第一部分：自旋、CAS、重读ctl 等结合，直到确定是否可以创建worker，
    // 可以则跳出循环继续操作，否则返回false
    retry:
    for (;;) {
        int c = ctl.get();
        //获取当前状态  c由runState和workercount组成
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        //非 running，非running状态都不接受新任务
        //SHUTDOWN可以处理队列中的任务
        //代码(1)
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        //代码(2)
        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            //CAS增加线程数量
            if (compareAndIncrementWorkerCount(c)) // CAS增长workerCount，成功则跳出循环
                break retry;
            c = ctl.get();  // Re-read ctl 重新获取ctl
            //代码2.1  判断线程池状态是否发生变化
            if (runStateOf(c) != rs) // 状态改变则继续外层循环，否则在内层循环
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }
    // 第二部分：创建worker，这部分使用ReentrantLock锁
    boolean workerStarted = false; // 线程启动标志位
    boolean workerAdded = false;  // 线程是否加入workers 标志位
    Worker w = null;
    try {
        w = new Worker(firstTask); //创建worker
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // 获取到锁以后仍需检查ctl，可能在上一个获取到锁处理的线程可能会改变runState
                // 如 ThreadFactory 创建失败 或线程池被 shut down等
                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive())
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                t.start(); // 启动线程
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w); // 失败操作
    }
    return workerStarted;
}
```
&emsp;addWorker的工作可分为两个部分：
- 第一部分：原子操作，判断是否可以创建worker。通过自旋、CAS、ctl 等操作，判断继续创建还是返回false，自旋周期一般很短。
- 第二部分：同步创建workder，并启动线程。

&emsp;我们看一下第一部分的一段代码(1)：
```java
if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
    return false;
```
&emsp;这个判断条件比较复杂，可以转换一下：
```java
if (rs >= SHUTDOWN &&
            (rs != SHUTDOWN ||
            firstTask != null ||
            workQueue.isEmpty()))
    return false;
```
&emsp;也就是代码(1)在下面三种情况会返回false（对应 || 的三种情况）：
- (1) 当前线程池状态为 STOP、TIDYING和TERMINATED。
- (2) 当前线程池为 SHUTDOWN 并且有了第一个任务。
- (3) 当前线程池为 SHUTDOWN 并且任务队列为空。

&emsp;代码(2)的循环的作用是使用CAS操作增加线程数量，成功则退出循环，进入第二部分。注意这里需要判断线程池的状态是否发生变化(代码2.1)，如果变了，需要回到第一部分的开头重新获取线程池状态。

&emsp;执行到第二部分，说明已经使用SCAS成功增加了线程数量，接下来使用全局的独占锁来控制把新增的 Worker 添加到工作集 workers 中。如果添加成功，则调用  t.start() 启动工作线程。
&emsp;addWorker(Runnable firstTask, boolean core) 参数中 core 指明了线程数量限制，true 对应 corePoolSize，false 对应 maximumPoolSize。另外，这里的 firstTask 参数可能为 null，这并不影响，也不需要在这里增加额外的判断，Worker 在 run 的时候，如果判断 firstTask 为 null，则会调用 getTask() 方法从任务队列中获取新任务，后面会详细介绍。


&emsp;理清 addWorker 之后，我们来看一下 Worker 类：
![Worker类图](https://github.com/JP6907/Pic/blob/master/java/Worker.png?raw=true)

&emsp;Worker是ThreadPoolExecutor的内部类，实现了 AbstractQueuedSynchronizer 并继承了 Runnable。
```java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable
{
    private static final long serialVersionUID = 6138294804551838833L;

    /** 每个worker有自己的内部线程，ThreadFactory创建失败时是null */
    final Thread thread;
    /** 初始化任务，可能是null */
    Runnable firstTask;
    /** 每个worker的完成任务数 */
    volatile long completedTasks;

    Worker(Runnable firstTask) {
        setState(-1); // 禁止线程在启动前被打断
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    /** 重要的执行方法  */
    public void run() {
        runWorker(this);
    }

    // state = 0 代表未锁；state = 1 代表已锁

    protected boolean isHeldExclusively() {
        return getState() != 0;
    }

    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }

    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }

    public void lock()        { acquire(1); }
    public boolean tryLock()  { return tryAcquire(1); }
    public void unlock()      { release(1); }
    public boolean isLocked() { return isHeldExclusively(); }
    // interrupt已启动线程
    void interruptIfStarted() {
        Thread t;
        // 初始化是 state = -1，不会被interrupt
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {
            }
        }
    }
}
```
&emsp;在构造函数内首先设置 Woker 状态为 -1，这是为了避免在调用 runWorker 方法前被打断（当其它线程调用了线程池的shutdownNow方法时，如果Worker状态>=-1则会中断该线程）。
&emsp;Worker 实现了简单的 非重入互斥锁，互斥容易理解，非重入是为了避免线程池的一些控制方法获得重入锁，比如setCorePoolSize操作。注意 Worker 实现锁的目的与传统锁的意义不太一样。其主要是为了控制线程是否可interrupt，以及其他的监控，如线程是否 active（正在执行任务）。
> 线程池里线程是否处于运行状态与普通线程不一样，普通线程可以调用 Thread.currentThread().isAlive() 方法来判断，而线程池，在run方法中可能在等待获取新任务，这期间线程线程是 alive 但是却不是 active。

&emsp;Worker 的 run 方法调用了 runWorker，代码如下：
```java
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        //Worker初始化时同步状态置为-1，此处进行解锁操作目的是将同步状态置为0，允许中断。
        w.unlock(); 
        boolean completedAbruptly = true;
        try {
            // loop 直至 task = null （线程池关闭、超时等）
            // 注意这里的getTask()方法，我们配置的阻塞队列会在这里起作用
            while (task != null || (task = getTask()) != null) {
                w.lock();  // 执行任务前上锁
                // 如果线程池停止，确保线程中断; 如果没有，确保线程不中断。这需要在第二种情况下进行重新获取ctl，以便在清除中断时处理shutdownNow竞争
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task); // 扩展点
                    Throwable thrown = null;
                    try {
                        task.run(); // 真正执行run方法
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown); // 扩展点
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly); // 线程退出工作
        }
    }
```
&emsp;runWorker的主要任务就是一直loop循环，来一个任务处理一个任务，没有任务就去getTask()，getTask()可能会阻塞，代码如下：
```java
private Runnable getTask() {
    boolean timedOut = false; // 上一次 poll() 是否超时

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 是否继续处理任务 可以参见上一篇的状态控制
        //两种情况：
        //rs >= SHUTDOWN && rs >= STOP  STOP以上，不处理任务
        //rs >= SHUTDOWN && workQueue.isEmpty()  SHUTDOWN态不接收新任务，会处理队列中的任务，但此时队列为空
        //代码1
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // 是否允许超时
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            Runnable r = timed ?
                //代码2
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```
&emsp;代码1处指明了两种情况无法获取到新task：
- rs >= SHUTDOWN && rs >= STOP  线程池状态为STOP、TIDYING或TERMINATED，不处理任何任务，包括任务队列中的任务
- rs >= SHUTDOWN && workQueue.isEmpty()  SHUTDOWN态不接收新任务，会处理队列中的任务，但此时队列为空，没有新任务 

&emsp;getTask()方法里面主要用我们配置的workQueue来工作，其阻塞原理与超时原理基于阻塞队列实现，这里不做详解。代码2处 workQueue.poll 方法是一个阻塞方法。

&emsp;总结，ThreadPoolExecutor的执行主要围绕Worker，Worker 实现了 AbstractQueuedSynchronizer 并继承了 Runnable，其对锁的妙运用，值得思考。

&nbsp;
> 参考：
https://www.jianshu.com/p/23cb8b903d2c
https://www.cnblogs.com/zaizhoumo/p/7794818.html
《Java并发编程之美》
