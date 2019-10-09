---
title: Java 并发编程 之 AbstractQueuedSynchronizer
subtitle: AbstractQueuedSynchronizer 详解
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - java
  - 编程语言
  - 并发
categories:
  - java
date: 2019-10-09 16:49:34
---



# AbstractQueuedSynchronizer 详解
&emsp;本文为转载保存，原文链接在文章末尾给出。文章对 AbstractQueuedSynchronizer 进行了细致的分析，非常详细和全面。虽然暂时还未能完全理解这篇文章，但是非常值得保存慢慢学习消化。

## 1. 背景
&emsp;AQS（java.util.concurrent.locks.AbstractQueuedSynchronizer）是Doug Lea大师创作的用来构建锁或者其他同步组件（信号量、事件等）的基础框架类。JDK中许多并发工具类的内部实现都依赖于AQS，如ReentrantLock, Semaphore, CountDownLatch等等。学习AQS的使用与源码实现对深入理解concurrent包中的类有很大的帮助。
本文重点介绍AQS中的基本实现思路，包括独占锁、共享锁的获取和释放实现原理和一些代码细节。

&emsp;对于AQS中ConditionObject的相关实现，可以参考我的另一篇博文[AbstractQueuedSynchronizer源码解读--续篇之Condition](http://www.cnblogs.com/micrari/p/7219751.html)。

## 2. 简介
&emsp;AQS的主要使用方式是继承它作为一个内部辅助类实现同步原语，它可以简化你的并发工具的内部实现，屏蔽同步状态管理、线程的排队、等待与唤醒等底层操作。

&emsp;AQS设计基于模板方法模式，开发者需要继承同步器并且重写指定的方法，将其组合在并发组件的实现中，调用同步器的模板方法，模板方法会调用使用者重写的方法。

## 3. 实现思路
&emsp;下面介绍下AQS具体实现的大致思路。

&emsp;AQS内部维护一个CLH队列来管理锁。线程会首先尝试获取锁，如果失败，则将当前线程以及等待状态等信息包成一个Node节点加到同步队列里。接着会不断循环尝试获取锁（条件是当前节点为head的直接后继才会尝试）,如果失败则会阻塞自己，直至被唤醒；而当持有锁的线程释放锁时，会唤醒队列中的后继线程。

&emsp;下面列举JDK中几种常见使用了AQS的同步组件：

- ReentrantLock: 使用了AQS的独占获取和释放,用state变量记录某个线程获取独占锁的次数,获取锁时+1，释放锁时-1，在获取时会校验线程是否可以获取锁。
- Semaphore: 使用了AQS的共享获取和释放，用state变量作为计数器，只有在大于0时允许线程进入。获取锁时-1，释放锁时+1。
- CountDownLatch: 使用了AQS的共享获取和释放，用state变量作为计数器，在初始化时指定。只要state还大于0，获取共享锁会因为失败而阻塞，直到计数器的值为0时，共享锁才允许获取，所有等待线程会被逐一唤醒。

### 3.1 如何获取锁
&emsp;获取锁的思路很直接：
```
while (不满足获取锁的条件) {
    把当前线程包装成节点插入同步队列
    if (需要阻塞当前线程)
        阻塞当前线程直至被唤醒
}

将当前线程从同步队列中移除
```
&emsp;以上是一个很简单的获取锁的伪代码流程，AQS的具体实现比这个复杂一些，也稍有不同，但思想上是与上述伪代码契合的。
&emsp;通过循环检测是否能够获取到锁，如果不满足，则可能会被阻塞，直至被唤醒。

### 3.2 如何释放锁
&emsp;释放锁的过程设计修改同步状态，以及唤醒后继等待线程：
```
修改同步状态
if (修改后的状态允许其他线程获取到锁)
    唤醒后继线程
```
这只是很简略的释放锁的伪代码示意，AQS具体实现中能看到这个简单的流程模型。

### 3.3 API简介
&emsp;通过上面的AQS大体思路分析，我们可以看到，AQS主要做了三件事情:

- 同步状态的管理
- 线程的阻塞和唤醒
- 同步队列的维护
下面三个protected final方法是AQS中用来访问/修改同步状态的方法:

- int getState(): 获取同步状态
- void setState(): 设置同步状态
- boolean compareAndSetState(int expect, int update)：基于CAS，原子设置当前状态

&emsp;在自定义基于AQS的同步工具时，我们可以选择覆盖实现以下几个方法来实现同步状态的管理：

| 方法 | 描述 | 
| --- | --- |
|boolean tryAcquire(int arg) | 试获取独占锁 |
|boolean tryRelease(int arg) | 试释放独占锁 | 
|int tryAcquireShared(int arg) | 试获取共享锁 |
|boolean tryReleaseShared(int arg) | 试释放共享锁 |
|boolean isHeldExclusively() | 当前线程是否获得了独占锁 |
&emsp;以上的几个试获取/释放锁的方法的具体实现应当是无阻塞的。

&emsp;AQS本身将同步状态的管理用模板方法模式都封装好了，以下列举了AQS中的一些模板方法：

| 方法 | 描述 |
| --- | --- |
|void acquire(int arg) | 获取独占锁。会调用tryAcquire方法，如果未获取成功，则会进入同步队列等待 |
|void acquireInterruptibly(int arg) | 响应中断版本的acquire |
|boolean tryAcquireNanos(int arg,long nanos) | 响应中断+带超时版本的acquire |
|void acquireShared(int arg) | 获取共享锁。会调用tryAcquireShared方法 |
|void acquireSharedInterruptibly(int arg) | 响应中断版本的acquireShared |
|boolean tryAcquireSharedNanos(int arg,long nanos) | 响应中断+带超时版本的acquireShared |
|boolean release(int arg) | 释放独占锁 |
|boolean releaseShared(int arg) | 释放共享锁 |
|Collection getQueuedThreads() | 获取同步队列上的线程集合 |

&emsp;上面看上去很多方法，其实从语义上来区分就是获取和释放，从模式上区分就是独占式和共享式，从中断相应上来看就是支持和不支持。

## 4. 代码解读
### 4.1 数据结构定义
&emsp;首先看一下AQS中的嵌套类Node的定义:
```java
static final class Node {

    /**
     * 用于标记一个节点在共享模式下等待
     */
    static final Node SHARED = new Node();

    /**
     * 用于标记一个节点在独占模式下等待
     */
    static final Node EXCLUSIVE = null;

    /**
     * 等待状态：取消
     */
    static final int CANCELLED = 1;

    /**
     * 等待状态：通知
     */
    static final int SIGNAL = -1;

    /**
     * 等待状态：条件等待
     */
    static final int CONDITION = -2;

    /**
     * 等待状态：传播
     */
    static final int PROPAGATE = -3;

    /**
     * 等待状态
     */
    volatile int waitStatus;

    /**
     * 前驱节点
     */
    volatile Node prev;

    /**
     * 后继节点
     */
    volatile Node next;

    /**
     * 节点对应的线程
     */
    volatile Thread thread;

    /**
     * 等待队列中的后继节点
     */
    Node nextWaiter;

    /**
     * 当前节点是否处于共享模式等待
     */
    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    /**
     * 获取前驱节点，如果为空的话抛出空指针异常
     */
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null) {
            throw new NullPointerException();
        } else {
            return p;
        }
    }

    Node() {
    }

    /**
     * addWaiter会调用此构造函数
     */
    Node(Thread thread, Node mode) {
        this.nextWaiter = mode;
        this.thread = thread;
    }

    /**
     * Condition会用到此构造函数
     */
    Node(Thread thread, int waitStatus) {
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```
&emsp;这里有必要专门梳理一下节点等待状态的定义，因为AQS源码中有大量的状态判断与跃迁。

| 值 | 描述 |
| -- | -- |
|CANCELLED (1) |当前线程因为超时或者中断被取消。这是一个终结态，也就是状态到此为止。| 
|SIGNAL (-1) |当前线程的后继线程被阻塞或者即将被阻塞，当前线程释放锁或者取消后需要唤醒后继线程。这个状态一般都是后继线程来设置前驱节点的。|
|CONDITION (-2) |当前线程在condition队列中。 |
|PROPAGATE (-3) |用于将唤醒后继线程传递下去，这个状态的引入是为了完善和增强共享锁的唤醒机制。在一个节点成为头节点之前，是不会跃迁为此状态的|
|0 |表示无状态。|

&emsp;对于分析AQS中不涉及ConditionObject部分的代码，可以认为队列中的节点状态只会是CANCELLED, SIGNAL, PROPAGATE, 0这几种情况。
![status](https://github.com/JP6907/Pic/blob/master/java/AQS-status.png?raw=true)

&emsp;图为自制的AQS状态的流转图，AQS中0状态和CONDITION状态为始态，CANCELLED状态为终态。0状态同时也可以是节点生命周期的终态。
注意，上图仅表示状态之间流转的可达性，并不代表一定能够从一个状态沿着线随意跃迁。

&emsp;在AQS中包含了head和tail两个Node引用，其中head在逻辑上的含义是当前持有锁的线程，head节点实际上是一个虚节点，本身并不会存储线程信息。
当一个线程无法获取锁而被加入到同步队列时，会用CAS来设置尾节点tail为当前线程对应的Node节点。
head和tail在AQS中是延迟初始化的，也就是在需要的时候才会被初始化，也就意味着在所有线程都能获取到锁的情况下，队列中的head和tail都会是null。

### 4.2 获取独占锁的实现
&emsp;下面来具体看看acquire(int arg)的实现：
```java
/**
 * 获取独占锁，对中断不敏感。
 * 首先尝试获取一次锁，如果成功，则返回；
 * 否则会把当前线程包装成Node插入到队列中，在队列中会检测是否为head的直接后继，并尝试获取锁,
 * 如果获取失败，则会通过LockSupport阻塞当前线程，直至被释放锁的线程唤醒或者被中断，随后再次尝试获取锁，如此反复。
 */
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

/**
 * 在队列中新增一个节点。
 */
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    Node pred = tail;
    // 快速尝试
    if (pred != null) {
        node.prev = pred;
        // 通过CAS在队尾插入当前节点
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    // 初始情况或者在快速尝试失败后插入节点
    enq(node);
    return node;
}

/**
 * 通过循环+CAS在队列中成功插入一个节点后返回。
 */
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        // 初始化head和tail
        if (t == null) {
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            /*
             * AQS的精妙就是体现在很多细节的代码，比如需要用CAS往队尾里增加一个元素
             * 此处的else分支是先在CAS的if前设置node.prev = t，而不是在CAS成功之后再设置。
             * 一方面是基于CAS的双向链表插入目前没有完美的解决方案，另一方面这样子做的好处是：
             * 保证每时每刻tail.prev都不会是一个null值，否则如果node.prev = t
             * 放在下面if的里面，会导致一个瞬间tail.prev = null，这样会使得队列不完整。
             */
            node.prev = t;
            // CAS设置tail为node，成功后把老的tail也就是t连接到node。
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}

/**
 * 在队列中的节点通过此方法获取锁，对中断不敏感。
 */
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            /*
             * 检测当前节点前驱是否head，这是试获取锁的资格。
             * 如果是的话，则调用tryAcquire尝试获取锁,
             * 成功，则将head置为当前节点。
             */
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            /*
             * 如果未成功获取锁则根据前驱节点判断是否要阻塞。
             * 如果阻塞过程中被中断，则置interrupted标志位为true。
             * shouldParkAfterFailedAcquire方法在前驱状态不为SIGNAL的情况下都会循环重试获取锁。
             */
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

/**
 * 根据前驱节点中的waitStatus来判断是否需要阻塞当前线程。
 */
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
         * 前驱节点设置为SIGNAL状态，在释放锁的时候会唤醒后继节点，
         * 所以后继节点（也就是当前节点）现在可以阻塞自己。
         */
        return true;
    if (ws > 0) {
        /*
         * 前驱节点状态为取消,向前遍历，更新当前节点的前驱为往前第一个非取消节点。
         * 当前线程会之后会再次回到循环并尝试获取锁。
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
         /**
          * 等待状态为0或者PROPAGATE(-3)，设置前驱的等待状态为SIGNAL,
          * 并且之后会回到循环再次重试获取锁。
          */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}


/**
 * 该方法实现某个node取消获取锁。
 */
private void cancelAcquire(Node node) {

   if (node == null)
       return;

   node.thread = null;

   // 遍历并更新节点前驱，把node的prev指向前部第一个非取消节点。
   Node pred = node.prev;
   while (pred.waitStatus > 0)
       node.prev = pred = pred.prev;

   // 记录pred节点的后继为predNext，后续CAS会用到。
   Node predNext = pred.next;

   // 直接把当前节点的等待状态置为取消,后继节点即便也在cancel可以跨越node节点。
   node.waitStatus = Node.CANCELLED;

   /*
    * 如果CAS将tail从node置为pred节点了
    * 则剩下要做的事情就是尝试用CAS将pred节点的next更新为null以彻底切断pred和node的联系。
    * 这样一来就断开了pred与pred的所有后继节点，这些节点由于变得不可达，最终会被回收掉。
    * 由于node没有后继节点，所以这种情况到这里整个cancel就算是处理完毕了。
    *
    * 这里的CAS更新pred的next即使失败了也没关系，说明有其它新入队线程或者其它取消线程更新掉了。
    */
   if (node == tail && compareAndSetTail(node, pred)) {
       compareAndSetNext(pred, predNext, null);
   } else {
       // 如果node还有后继节点，这种情况要做的事情是把pred和后继非取消节点拼起来。
       int ws;
       if (pred != head &&
           ((ws = pred.waitStatus) == Node.SIGNAL ||
            (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
           pred.thread != null) {
           Node next = node.next;
           /* 
            * 如果node的后继节点next非取消状态的话，则用CAS尝试把pred的后继置为node的后继节点
            * 这里if条件为false或者CAS失败都没关系，这说明可能有多个线程在取消，总归会有一个能成功的。
            */
           if (next != null && next.waitStatus <= 0)
               compareAndSetNext(pred, predNext, next);
       } else {
           /*
            * 这时说明pred == head或者pred状态取消或者pred.thread == null
            * 在这些情况下为了保证队列的活跃性，需要去唤醒一次后继线程。
            * 举例来说pred == head完全有可能实际上目前已经没有线程持有锁了，
            * 自然就不会有释放锁唤醒后继的动作。如果不唤醒后继，队列就挂掉了。
            * 
            * 这种情况下看似由于没有更新pred的next的操作，队列中可能会留有一大把的取消节点。
            * 实际上不要紧，因为后继线程唤醒之后会走一次试获取锁的过程，
            * 失败的话会走到shouldParkAfterFailedAcquire的逻辑。
            * 那里面的if中有处理前驱节点如果为取消则维护pred/next,踢掉这些取消节点的逻辑。
            */
           unparkSuccessor(node);
       }
       
       /*
        * 取消节点的next之所以设置为自己本身而不是null,
        * 是为了方便AQS中Condition部分的isOnSyncQueue方法,
        * 判断一个原先属于条件队列的节点是否转移到了同步队列。
        *
        * 因为同步队列中会用到节点的next域，取消节点的next也有值的话，
        * 可以断言next域有值的节点一定在同步队列上。
        *
        * 在GC层面，和设置为null具有相同的效果。
        */
       node.next = node; 
   }
}

/**
 * 唤醒后继线程。
 */
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    // 尝试将node的等待状态置为0,这样的话,后继争用线程可以有机会再尝试获取一次锁。
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    Node s = node.next;
    /*
     * 这里的逻辑就是如果node.next存在并且状态不为取消，则直接唤醒s即可
     * 否则需要从tail开始向前找到node之后最近的非取消节点。
     *
     * 这里为什么要从tail开始向前查找也是值得琢磨的:
     * 如果读到s == null，不代表node就为tail，参考addWaiter以及enq函数中的我的注释。
     * 不妨考虑到如下场景：
     * 1. node某时刻为tail
     * 2. 有新线程通过addWaiter中的if分支或者enq方法添加自己
     * 3. compareAndSetTail成功
     * 4. 此时这里的Node s = node.next读出来s == null，但事实上node已经不是tail，它有后继了!
     */
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```
&emsp;AQS独占锁的获取的流程示意如下：
![获取独占锁](https://github.com/JP6907/Pic/blob/master/java/AQS-exclusiveLock.png?raw=true)

### 4.3 释放独占锁的实现
&emsp;上面已经分析了acquire的实现，下面来看看release的实现：
对于释放一个独占锁，首先会调用tryRelease，在完全释放掉独占锁后，这时后继线程是可以获取到独占锁的，因此释放者线程需要做的事情是唤醒一个队列中的后继者线程，让它去尝试获取独占锁。

&emsp;上述所谓完全释放掉锁的含义，简单来说就是当前锁处于无主状态，等待线程有可能可以获取。
举例：对于可重入锁ReentrantLock, 每次tryAcquire后，state会+1，每次tryRelease后，state会-1，如果state变为0了，则此时称独占锁被完全释放了。

&emsp;下面，我们来看一下release的具体代码实现：
```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        /*
         * 此时的head节点可能有3种情况:
         * 1. null (AQS的head延迟初始化+无竞争的情况)
         * 2. 当前线程在获取锁时new出来的节点通过setHead设置的
         * 3. 由于通过tryRelease已经完全释放掉了独占锁，有新的节点在acquireQueued中获取到了独占锁，并设置了head

         * 第三种情况可以再分为两种情况：
         * （一）时刻1:线程A通过acquireQueued，持锁成功，set了head
         *          时刻2:线程B通过tryAcquire试图获取独占锁失败失败，进入acquiredQueued
         *          时刻3:线程A通过tryRelease释放了独占锁
         *          时刻4:线程B通过acquireQueued中的tryAcquire获取到了独占锁并调用setHead
         *          时刻5:线程A读到了此时的head实际上是线程B对应的node
         * （二）时刻1:线程A通过tryAcquire直接持锁成功，head为null
         *          时刻2:线程B通过tryAcquire试图获取独占锁失败失败，入队过程中初始化了head，进入acquiredQueued
         *          时刻3:线程A通过tryRelease释放了独占锁，此时线程B还未开始tryAcquire
         *          时刻4:线程A读到了此时的head实际上是线程B初始化出来的傀儡head
         */
        Node h = head;
        // head节点状态不会是CANCELLED，所以这里h.waitStatus != 0相当于h.waitStatus < 0
        if (h != null && h.waitStatus != 0)
            // 唤醒后继线程，此函数在acquire中已经分析过，不再列举说明
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```
&emsp;整个release做的事情就是:

1. 调用tryRelease
2. 如果tryRelease返回true也就是独占锁被完全释放，唤醒后继线程。

&emsp;这里的唤醒是根据head几点来判断的，上面代码的注释中也分析了head节点的情况，只有在head存在并且等待状态小于零的情况下唤醒。

### 4.4 获取共享锁的实现
&emsp;与获取独占锁的实现不同的关键在于，共享锁允许多个线程持有。如果需要使用AQS中共享锁，在实现tryAcquireShared方法时需要注意，返回负数表示获取失败;返回0表示成功，但是后继争用线程不会成功;返回正数表示获取成功，并且后继争用线程也可能成功。

&emsp;下面来看一下具体的代码实现：
```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}

private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                // 一旦共享获取成功，设置新的头结点，并且唤醒后继线程
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

/**
 * 这个函数做的事情有两件:
 * 1. 在获取共享锁成功后，设置head节点
 * 2. 根据调用tryAcquireShared返回的状态以及节点本身的等待状态来判断是否要需要唤醒后继线程。
 */
private void setHeadAndPropagate(Node node, int propagate) {
    // 把当前的head封闭在方法栈上，用以下面的条件检查。
    Node h = head;
    setHead(node);
    /*
     * propagate是tryAcquireShared的返回值，这是决定是否传播唤醒的依据之一。
     * h.waitStatus为SIGNAL或者PROPAGATE时也根据node的下一个节点共享来决定是否传播唤醒，
     * 这里为什么不能只用propagate > 0来决定是否可以传播在本文下面的思考问题中有相关讲述。
     */
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}

/**
 * 这是共享锁中的核心唤醒函数，主要做的事情就是唤醒下一个线程或者设置传播状态。
 * 后继线程被唤醒后，会尝试获取共享锁，如果成功之后，则又会调用setHeadAndPropagate,将唤醒传播下去。
 * 这个函数的作用是保障在acquire和release存在竞争的情况下，保证队列中处于等待状态的节点能够有办法被唤醒。
 */
private void doReleaseShared() {
    /*
     * 以下的循环做的事情就是，在队列存在后继线程的情况下，唤醒后继线程；
     * 或者由于多线程同时释放共享锁由于处在中间过程，读到head节点等待状态为0的情况下，
     * 虽然不能unparkSuccessor，但为了保证唤醒能够正确稳固传递下去，设置节点状态为PROPAGATE。
     * 这样的话获取锁的线程在执行setHeadAndPropagate时可以读到PROPAGATE，从而由获取锁的线程去释放后继等待线程。
     */
    for (;;) {
        Node h = head;
        // 如果队列中存在后继线程。
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;
                unparkSuccessor(h);
            }
            // 如果h节点的状态为0，需要设置为PROPAGATE用以保证唤醒的传播。
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;
        }
        // 检查h是否仍然是head，如果不是的话需要再进行循环。
        if (h == head)
            break;
    }
}
```

### 4.5 释放共享锁的实现
&emsp;释放共享锁与获取共享锁的代码共享了doReleaseShared，用于实现唤醒的传播。
```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        // doReleaseShared的实现上面获取共享锁已经介绍
        doReleaseShared();
        return true;
    }
    return false;
}
```
&emsp;从中，我们可以看出，共享锁的获取和释放都会涉及到doReleaseShared,也就是后继线程的唤醒。关于PROPAGATE状态的必要性，后文会作进一步介绍。
## 5. 一些思考
&emsp;AQS的代码实在是很精妙，要看懂大致套路并不困难，但是要完全领悟其中的一些细节是一件需要花功夫来仔细琢磨品味的事情。

&emsp;下面列出一些看源码时的问题与思考:

### 5.1 插入节点时的代码顺序
&emsp;addWaiter和enq方法中新增一个节点时为什么要先将新节点的prev置为tail再尝试CAS，而不是CAS成功后来构造节点之间的双向链接?
&emsp;这是因为，双向链表目前没有基于CAS原子插入的手段，如果我们将node.prev = t和t.next = node（t为方法执行时读到的tail，引用封闭在栈上）
&emsp;放到compareAndSetTail(t, node)成功后执行，如下所示：
```java
if (compareAndSetTail(t, node)) {
   node.prev = t;
   t.next = node;
   return t;
}
```
会导致这一瞬间的tail也就是t的prev为null，这就使得这一瞬间队列处于一种不一致的中间状态。

### 5.2 唤醒节点时为什么从tail向前遍历
&emsp;unparkSuccessor方法中为什么唤醒后继节点时要从tail向前查找最接近node的非取消节点，而不是直接从node向后找到第一个后break掉?

&emsp;在上面的代码注释中已经提及到这一点：如果读到s == null，不代表node就为tail。
&emsp;考虑如下场景：node某时刻为tail,有新线程通过addWaiter中的if分支或者enq方法添加自己,compareAndSetTail成功。此时这里的Node s = node.next读出来s == null，但事实上node已经不是tail，它有后继了!


### 5.3 unparkSuccessor有新线程争锁是否存在漏洞
&emsp;unparkSuccessor方法在被release调用时是否存在这样的一个漏洞?
- 时刻1: node -> tail && tail.waitStatus == Node.CANCELLED (node的下一个节点为tail，并且tail处于取消状态)
- 时刻2: unparkSuccessor读到s.waitStatus > 0
- 时刻3: unparkSuccessor从tail开始遍历
- 时刻4: tail节点对应线程执行cancelAcquire方法中的if (node == tail && compareAndSetTail(node, pred)) 返回true,此时tail变为pred(也就是node)
- 时刻5: 有新线程进队列tail变为新节点
- 时刻6: unparkSuccessor没有发现需要唤醒的节点

&emsp;最终新节点阻塞并且前驱节点结束调用，新节点再也无法被unpark这种情况不会发生,确实可能出现从tail向前扫描，没有读到新入队的节点，但别忘了acquireQueued的思想就是不断循环检测是否能够独占获取锁，否则再进行判断是否要阻塞自己，而release的第一步就是tryRelease，它的语义为true表示完全释放独占锁，完全释放之后才会执行后面的逻辑，也就是unpark后继线程。在这种情况下，新入队的线程应当能获取到锁。如果没有获取锁，则必然是在覆盖tryAcquire/tryRelease的实现有问题，导致前驱节点成功释放了独占锁，后继节点获取独占锁仍然失败。也就是说AQS框架的可靠性还在某些程度上依赖于具体子类的实现，子类实现如果有bug，那AQS再精巧也扛不住。

### 5.4 AQS如何保证队列活跃
&emsp;AQS如何保证在节点释放的同时又有新节点入队的情况下，不出现原持锁线程释放锁，后继线程被自己阻塞死的情况,保持同步队列的活跃？
&emsp;回答这个问题，需要理解shouldParkAfterFailedAcquire和unparkSuccessor这两个方法。以独占锁为例，后继争用线程阻塞自己的情况是读到前驱节点的等待状态为SIGNAL,只要不是这种情况都会再试着去争取锁。假设后继线程读到了前驱状态为SIGNAL，说明之前在tryAcquire的时候，前驱持锁线程还没有tryRelease完全释放掉独占锁。此时如果前驱线程完全释放掉了独占锁，则在unparkSuccessor中还没执行完置waitStatus为0的操作，也就是还没执行到下面唤醒后继线程的代码，否则后继线程会再去争取锁。那么就算后继争用线程此时把自己阻塞了，也一定会马上被前驱线程唤醒。
&emsp;那么是否可能持锁线程执行唤醒后继线程的逻辑时，后继线程读到前驱等待状态为SIGNAL把自己给阻塞，再也无法苏醒呢？这个问题在上面的问题3中已经有答案了，确实可能在扫描后继需要唤醒线程时读不到新来的线程，但只要tryRelease语义实现正确，在true时表示完全释放独占锁，
则后继线程理应能够tryAcquire成功，shouldParkAfterFailedAcquire在读到前驱状态不为SIGNAL会给当前线程再一次获取锁的机会的。
&emsp;别看AQS代码写的有些复杂，状态有些多，还真的就是没毛病，各种情况都能覆盖。

### 5.5 PROPAGATE状态存在的意义
&emsp;在setHeadAndPropagate中我们可以看到如下的一段代码:
```java
if (propagate > 0 || h == null || h.waitStatus < 0 ||
       (h = head) == null || h.waitStatus < 0) {
       Node s = node.next;
       if (s == null || s.isShared())
           doReleaseShared();
}
```
&emsp;为什么不只是用propagate > 0来判断呢？我们知道目前AQS代码中的Node.PROPAGATE状态就是为了此处可以读取到h.waitStatus < 0（PROPAGATE值为-3）；如果这里可以只用propagate > 0来判断，是否PROPAGATE状态都没有存在的必要了？

&emsp;我接触JAVA比较晚，接触的时候就已经是JDK8的年代了。这个问题我思考了很久，没有想到很合理的解释来说明PROPAGATE状态存在的必要性。在网上也鲜少有相关方面的资料、博客提及到这些。后来通过浏览Doug Lea的个人网站，发现在很久以前AQS的代码确实是没有PROPAGATE的，PROPAGATE的引入是为了解决共享锁并发释放导致的线程hang住问题。

&emsp;在Doug Lea的JSR 166 repository上，我找到了PROPAGATE最早被引入的那一版。可以看到Revision1.73中，PROPAGATE状态被引入用以修复bug 6801020,让我们来看看这个bug:
```java
import java.util.concurrent.Semaphore;

public class TestSemaphore {

   private static Semaphore sem = new Semaphore(0);

   private static class Thread1 extends Thread {
       @Override
       public void run() {
           sem.acquireUninterruptibly();
       }
   }

   private static class Thread2 extends Thread {
       @Override
       public void run() {
           sem.release();
       }
   }

   public static void main(String[] args) throws InterruptedException {
       for (int i = 0; i < 10000000; i++) {
           Thread t1 = new Thread1();
           Thread t2 = new Thread1();
           Thread t3 = new Thread2();
           Thread t4 = new Thread2();
           t1.start();
           t2.start();
           t3.start();
           t4.start();
           t1.join();
           t2.join();
           t3.join();
           t4.join();
           System.out.println(i);
       }
   }
}
```
&emsp;很显然，这段程序一定能执行结束的，但是会偶现线程hang住的问题。
&emsp;当时的AQS中setHeadAndPropagate是这样的:
![bug 6801020](https://github.com/JP6907/Pic/blob/master/java/javabug6801020.png?raw=true)
&emsp;以上是bug 6801020修复点的对比，左边为修复之前的版本，右边为引入PROPAGATE修复之后的版本。

&emsp;从左边可以看到原先的setHeadAndPropagate相比目前版本要简单很多，而releaseShared的实现也与release基本雷同，这也正是本问题的核心：为什么仅仅用调用的tryAcquireShared
得到的返回值来判断是否需要唤醒不行呢？

&emsp;在PROPAGATE状态出现之前的源码可以[在这里查看](http://gee.cs.oswego.edu/cgi-bin/viewcvs.cgi/jsr166/src/main/java/util/concurrent/locks/AbstractQueuedSynchronizer.java?revision=1.73&view=markup)。


#### 5.5.1 分析
&emsp;让我们来分析一下上面的程序：

&emsp;上面的程序循环中做的事情就是放出4个线程，其中2个线程用于获取信号量，另外2个用于释放信号量。每次循环主线程会等待所有子线程执行完毕。出现bug也就是线程hang住的问题就在于两个获取信号量的线程有一个会没办法被唤醒，队列就死掉了。

&emsp;在AQS的共享锁中，一个被park的线程，不考虑线程中断和前驱节点取消的情况，有两种情况可以被unpark：一种是其他线程释放信号量，调用unparkSuccessor；另一种是其他线程获取共享锁时通过传播机制来唤醒后继节点。

&emsp;我们假设某次循环中队列里排队的节点为情况为:
&emsp;head -> t1的node -> t2的node(也就是tail)

&emsp;信号量释放的顺序为t3先释放，t4后释放:
- 时刻1: t3调用releaseShared，调用了unparkSuccessor(h)，head的等待状态从-1变为0
- 时刻2: t1由于t3释放了信号量，被t3唤醒，调用Semaphore.NonfairSync的tryAcquireShared，返回值为0
- 时刻3: t4调用releaseShared,读到此时h.waitStatus为0(此时读到的head和时刻1中为同一个head)，不满足条件,因此不会调用unparkSuccessor(h)
- 时刻4: t1获取信号量成功，调用setHeadAndPropagate时，因为不满足propagate > 0（时刻2的返回值也就是propagate==0）,从而不会唤醒后继节点

&emsp;这就好比是一个精巧的多米诺骨牌最终由于设计的失误导致动力无法传递下去，至此AQS中的同步队列宣告死亡。

&emsp;那么引入PROPAGATE是怎么解决问题的呢？
&emsp;引入之后，调用releaseShared方法不再简单粗暴地直接unparkSuccessor,而是将传播行为抽了一个doReleaseShared方法出来。
&emsp;再看上面的那种情况:
- 时刻1：t3调用releaseShared -> doReleaseShared -> unparkSuccessor，完了之后head的等待状态为0
- 时刻2：t1由于t3释放了信号量，被t3唤醒，调用Semaphore.NonfairSync的tryAcquireShared，返回值为0
- 时刻3：t4调用releaseShared,读到此时h.waitStatus为0(此时读到的head和时刻1中为同一个head)，将等待状态置为PROPAGATE
- 时刻4：t1获取信号量成功，调用setHeadAndPropagate时，可以读到h.waitStatus < 0，从而可以接下来调用doReleaseShared唤醒t2

&emsp;也就是说，上面会产生线程hang住bug的case在引入PROPAGATE后可以被规避掉。在PROPAGATE引入之前，之所以可能会出现线程hang住的情况，就是在于releaseShared有竞争的情况下，可能会有队列中处于等待状态的节点因为第一个线程完成释放唤醒，第二个线程获取到锁，但还没设置好head，又有新线程释放锁，但是读到老的head状态为0导致释放但不唤醒，最终后一个等待线程既没有被释放线程唤醒，也没有被持锁线程唤醒。
&emsp;所以，仅仅靠tryAcquireShared的返回值来决定是否要将唤醒传递下去是不充分的。

### 5.6 AQS如何防止内存泄露
&emsp;AQS维护了一个FIFO队列，它是如何保证在运行期间不发生内存泄露的？
AQS在无竞争条件下，甚至都不会new出head和tail节点。线程成功获取锁时设置head节点的方法为setHead，由于头节点的thread并不重要，此时会置node的thread和prev为null，完了之后还会置原先head也就是线程对应node的前驱的next为null，从而实现队首元素的安全移出。而在取消节点时，也会令node.thread = null，在node不为tail的情况下，会使node.next = node（之所以这样也是为了isOnSyncQueue实现更加简洁）。

## 6. 总结
&emsp;AQS毫无疑问是Doug Lea大师令人叹为观止的作品，它实现精巧、鲁棒、优雅，很好地封装了同步状态的管理、线程的等待与唤醒，足以满足大多数同步工具的需求。
阅读AQS的源码不是一蹴而就就能完全读懂的，阅读源码大致分为三步：

- 读懂大概思路以及一些重要方法之间的调用关系
- 逐行看代码的具体实现，知道每一段代码是干什么的
- 琢磨参悟某一段代码为什么是这么写的，能否换一种写法，能否前后几行代码调换顺序，作者是怎么想的

&emsp;从Doug Lea大师的论文中，我们也能够看出他设计并实现了AQS本身一方面是本人功力深厚，另一方面也阅读了大量的文献与资料，也做了很多方面的测试。
&emsp;读AQS最难的地方不在于明白套路和思路，而在于代码中点点滴滴的细节。从一行行的代码角度来说，比如改一个值，是否需要CAS，是否一定要CAS成功；读一个值，在多线程环境下含义是什么，有哪些种情况。从一个个方法角度来说，这些方法的调用关系是如何保证框架的正确性、鲁棒性、伸缩性等。
如果能把这些细节都想清楚，明白作者的思路与考虑，才可以源码理解入木三分了。

&emsp;对于PROPAGATE状态，网上大多AQS的介绍也都只是浅显地提及是用来设置传播的，缺少对于这个状态存在必要性的思考。一开始我也想了很久不明白为什么一定需要一个PROPAGATE状态而不是直接根据tryAcquireShared的返回值来判断是否需要传播。后来也是去了Doug Lea的个人网站翻出当时最早引入PROPAGATE状态的提交，看到了原来的代码，以及 http://bugs.java.com/ 上的bug才更厘清PROPAGATE状态引入的前因后果。

&emsp;尽管看懂源码，也可能远远达不到能再造一个能与之媲美的轮子的程度，但是能对同步框架、锁、线程等有更深入的理解，也是很丰硕的收获了。
&emsp;当然，AQS也有其局限性，由于维护的是FIFO队列。如果想要实现一个具有优先级的锁，AQS就派不上什么用处了。

## 7. 参考
Doug Lea的AQS论文:[《The Art of Multiprocessor Programming(多处理器编程的艺术)》](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)



&nbsp;
&nbsp;
> 原文链接：https://www.cnblogs.com/micrari/p/6937995.html