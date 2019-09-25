---
title: 线程池 之 ThreadPoolExecutor 概述
catalog: true
subtitle: ThreadPoolExecutor 概述
header-img: "/img/article_header/article_header.png"
tags:
- java
- 编程语言
- 并发
categories:
- java
---


# ThreadPoolExecutor 概述
&emsp;合理利用线程池能够带来三个好处。第一：降低资源消耗。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。第二：提高响应速度。当任务到达时，任务可以不需要等到线程创建就能立即执行。第三：提高线程的可管理性。线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。但是要做到合理的利用线程池，必须对其原理了如指掌。

## 1. 创建线程池
&emsp;查看源码，我们可以知道 ThreadPoolExecutor 的继承关系如图。
![ThreadPoolExecutor](https://github.com/JP6907/Pic/blob/master/java/ThreadPoolExecutor.png?raw=true)

&emsp;ExecutorService（ThreadPoolExecutor的顶层接口）使用线程池中的线程执行每个提交的任务，通常我们使用Executors的工厂方法来创建ExecutorService。
&emsp;线程池解决了两个不同的问题：
- 提升性能：它们通常在执行大量异步任务时，由于减少了每个任务的调用开销，并且它们提供了一种限制和管理资源（包括线程）的方法，使得性能提升明显；
- 统计信息：每个ThreadPoolExecutor保持一些基本的统计信息，例如完成的任务数量。

&emsp;为了在广泛的上下文中有用，此类提供了许多可调参数和可扩展性钩子。 java 源码中预配置了几种线程池，可以通过 Executors 的工厂方法直接使用。

- Executors.newCachedThreadPool（无界线程池，自动线程回收）
- Executors.newFixedThreadPool（固定大小的线程池）；
- Executors.newSingleThreadExecutor（单一后台线程）；
- Executors.newScheduledThreadPool（创建定时任务）；

## 2. 自定义线程池
&emsp;我们可以通过 ThreadPoolExecutor 来自定义线程池，下面是它的构造方法：
```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler)
```
&emsp;各个参数的含义:
- corePoolSize - 池中所保存的线程数，包括空闲线程，必须大于或等于0。
- maximumPoolSize - 池中允许的最大线程数，必须大于或等于corePoolSize。
- keepAliveTime - 线程存活时间，当线程数大于核心时，此为终止前多余的空闲线程等待新任务的最长时间。
- unit - keepAliveTime 参数的时间单位，必须大于或等于0。
- workQueue - 工作队列，执行前用于保持任务的队列。此队列仅保持由 execute 方法提交的 Runnable 任务。
- threadFactory - 执行程序创建新线程时使用的工厂，默认为DefaultThreadFactory类。
- handler - 拒绝策略，由于超出线程范围和队列容量而使执行被阻塞时所使用的处理程序，默认策略为ThreadPoolExecutor.AbortPolicy。

### 2.1 corePoolSize 和 maximumPoolSize
|  参数   | 翻译  |
|  ----  | ----  |
| corePoolSize  | 核心线程池数量 |
| maximumPoolSize  | 最大线程池数量 |

&emsp;ThreadPoolExecutor 将会根据corePoolSize和maximumPoolSize自动地调整线程池大小。

&emsp;当使用 execute(Runnable) 方法中提交新任务并且少于 corePoolSize 线程正在运行时，即使其他工作线程处于空闲状态，也会创建一个新线程来处理该请求。 如果有多于corePoolSize 但小于 maximumPoolSize 线程正在运行，则仅当队列已满时才会创建新线程。 通过设置 corePoolSize 和 maximumPoolSize 相同，可以创建一个固定大小的线程池。 通过将 maximumPoolSize 设置为基本上无界的值，例如 Integer.MAX_VALUE，可以允许池容纳任意数量的并发任务。 通常，核心和最大池大小仅在构建时设置，但也可以使用 setCorePoolSize 和 setMaximumPoolSize 进行动态更改。

&emsp;这段话详细了描述了线程池对任务的处理流程，这里用个图总结一下:
![线程任务处理流程](https://github.com/JP6907/Pic/blob/master/java/thread-process-task.png?raw=true)

### 2.2 prestartCoreThread 核心线程预启动
&emsp;在默认情况下，只有当新任务到达时，才开始创建和启动核心线程，但是我们可以使用 prestartCoreThread() 和 prestartAllCoreThreads() 方法动态调整。
如果使用非空队列构建池，则可能需要预先启动线程。
|  方法   | 作用  |
|  ----  | ----  |
| prestartCoreThread()  | 创建一个空闲任务线程等待任务的到达 |
| prestartAllCoreThreads()  | 创建核心线程池数量的空闲任务线程等待任务的到达 |

### 2.3 ThreadFactory 线程工厂
&emsp;新线程使用 ThreadFactory 创建。 如果未另行指定，则使用 Executors.defaultThreadFactory 默认工厂，使其全部位于同一个 ThreadGroup 中，并且具有相同的NORM_PRIORITY 优先级和非守护进程状态。

&emsp;通过自定义的 ThreadFactory，我们可以更改线程的名称，线程组，优先级，守护进程状态等。如果 ThreadFactory 的 newThread 方法返回 null，executor 会继续运行，但可能无法执行任何任务。


### 2.4 Keep-alive times 线程存活时间
&emsp;如果线程池当前拥有超过 corePoolSize 的线程，那么多余的线程在空闲时间超过 keepAliveTime 时会被终止。这提供了一种在线程池不活跃时减少资源消耗的方法。
&emsp;如果线程池变得更加活跃，则应构建新线程。 也可以使用方法 setKeepAliveTime(long，TimeUnit) 的方法进行动态调整。为了防止空闲线程在关闭之前终止，可以使用如下方法：
```java
setKeepAliveTime(Long.MAX_VALUE，TimeUnit.NANOSECONDS);
```
&emsp;默认情况下，keep-alive 策略仅适用于存在超过 corePoolSize 线程的情况。 但是，只要 keepAliveTime 值不为零，方法 allowCoreThreadTimeOut(boolean) 也可用于将此超时策略应用于核心线程。

### 2.5 Queuing 队列

&emsp;workQueue 的类型是 BlockingQueue<Runnable>，用于存放提交的任务，队列的实际容量与线程池大小相关联。

- 如果当前线程池任务线程数量小于核心线程池数量，执行器总是优先创建一个任务线程，而不是从线程队列中取一个空闲线程。
- 如果当前线程池任务线程数量大于核心线程池数量，执行器总是优先从线程队列中取一个空闲线程，而不是创建一个任务线程。
- 如果当前线程池任务线程数量大于核心线程池数量，且队列中无空闲任务线程，将会创建一个任务线程，直到超出 maximumPoolSize，如果超出 maximumPoolSize，则任务将会被拒绝。通过设定的拒绝策略来处理（2.6）。

这个过程参考2.1中的线程任务处理流程图。

&emsp;主要有三种队列策略：

- Direct handoffs 直接握手队列
&emsp;Direct handoffs 工作队列的默认选项是**SynchronousQueue**，它将任务直接提交给线程而不保持它们。在此，如果不存在可用于立即运行任务的线程，则试图把任务加入队列将失败，因此会构造一个新的线程。此策略可以避免在处理可能具有内部依赖性的请求集时出现锁。直接提交通常要求无界 maximumPoolSizes以避免拒绝新提交的任务(**线程数量足够多**)。当命令以超过队列所能处理的平均数连续到达时，此策略允许无界线程具有增长的可能性。

- Unbounded queues 无界队列
&emsp;使用无界队列（例如，不具有预定义容量的**LinkedBlockingQueue**）将导致在所有 corePoolSize 线程都忙时新任务在队列中等待。这样，**创建的线程就不会超过 corePoolSize**。（因此，maximumPoolSize 的值也就无效了。）当每个任务完全独立于其他任务，即任务执行互不影响时，适合于使用无界队列；例如，在 Web 页服务器中。这种排队可用于处理瞬态突发请求，当命令以超过队列所能处理的平均数连续到达时，此策略允许**无界线程**具有增长的可能性。

- Bounded queues 有界队列
&emsp;一个有界的队列（例如**ArrayBlockingQueue**）配置有限的 maximumPoolSizes 配置有助于防止资源耗尽，但是难以控制。队列大小和 maximumPoolSizes 需要相互权衡：
&emsp;使用大队列和较小的 maximumPoolSizes 可以最大限度地减少CPU使用率，操作系统资源和上下文切换开销，但会导致人为的低吞吐量。如果任务经常被阻塞（比如I/O限制），那么我们可能需要配置更多的线程提供给系统调用；
&emsp;使用小队列通常需要较大的 maximumPoolSizes，这会使得CPU更繁忙，产生大量的调度开销，这也会降低吞吐量。

### 2.6 Rejected tasks 拒绝任务策略
&emsp;拒绝策略通过RejectedExecutionHandler参数配置。拒绝任务有两种情况：1. 线程池已经被关闭；2. 任务队列已满且 maximumPoolSizes 已满；
无论哪种情况，都会调用RejectedExecutionHandler的rejectedExecution方法。预定义了四种处理策略：
- AbortPolicy：默认的饱和策略，直接抛出RejectedExecutionException异常。
- CallerRunsPolicy：用调用者所在的线程来执行任务，此策略提供简单的反馈控制机制，能够减缓新任务的提交速度。
- DiscardPolicy：直接丢弃新提交的任务；
- DiscardOldestPolicy：如果执行程序尚未关闭，则丢弃阻塞队列中最靠前的任务，然后重试执行新任务（如果再次失败，则重复此过程）。

&emsp;我们可以自己定义RejectedExecutionHandler，以适应特殊的容量和队列策略场景中,但需要非常小心，尤其是当策略仅用于特定容量或排队策略时。

### 2.7 Hook methods 钩子方法
&emsp;ThreadPoolExecutor为提供了每个任务执行前后提供了钩子方法，重写beforeExecute(Thread，Runnable)和afterExecute(Runnable，Throwable)方法来操纵执行环境； 例如，重新初始化ThreadLocals，收集统计信息或记录日志等。此外，terminated()在Executor完全终止后需要完成后会被调用，可以重写此方法，以执行任殊处理。
> 注意：如果hook或回调方法抛出异常，内部的任务线程将会失败并结束。
```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(corePoolSize,maximumPoolSize,keepAliveTime,
                unit,workQueue,threadFactory,handler){
            @Override
            protected void beforeExecute(Thread t, Runnable r) {
                super.beforeExecute(t, r);
            }
            @Override
            protected void afterExecute(Runnable r, Throwable t) {
                super.afterExecute(r, t);
            }
            @Override
            protected void terminated() {
                super.terminated();
            }
        };
```

### 2.8 Queue maintenance 维护队列
&emsp;getQueue()方法可以访问任务队列，一般用于监控和调试。绝不建议将这个方法用于其他目的。当在大量的队列任务被取消时，remove()和purge()方法可用于回收空间。

### 2.9 Finalization 关闭
&emsp;如果程序中不再持有线程池的引用，并且线程池中没有线程时，线程池将会自动关闭。如果希望确保即使用户忘记调用 shutdown()方法也可以回收未引用的线程池，那么必须通过设置适当的 keep-alive times 并设置allowCoreThreadTimeOut(boolean)。一般情况下，线程池启动后建议手动调用shutdown()关闭。

> 参考 : 
ThreadPoolExecutor.java 源码注释
https://www.jianshu.com/p/c41e942bcd64