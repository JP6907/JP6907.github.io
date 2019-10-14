---
title: Java 并发编程 之 ReentrantReadWriteLock
subtitle: ReentrantReadWriteLock 详解
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - java
  - 编程语言
  - 并发
categories:
  - java
date: 2019-10-14 08:54:58
---


# ReentrantReadWriteLock 详解

## 1. 概述
&emsp;ReentrantLock 是一个排他锁，同一时间只允许一个线程访问，而 ReentrantReadWriteLock 允许多个读线程同时访问，但不允许写线程和读线程、写线程和写线程同时访问。相对于排他锁，提高了并发性。在实际应用中，大部分情况下对共享数据（如缓存）的访问都是读操作远多于写操作，这时ReentrantReadWriteLock能够提供比排他锁更好的并发性和吞吐量。
&emsp;读写锁的内部维护了一个 ReadLock 和一个 WriteLock ，它们依赖 Sync 实现具体功能。而 Sync 继承自AQS ，并且也提供了公平和非公平的实现。我们知道 AQS 中只维护了一个state 状态，而 ReentrantReadWriteLock 则需要维护读状态和写状态， 一个 state 怎么表示写和读两种状态呢？ReentrantReadWriteLock 巧妙地使用 state 的高16 位表示读状态，也就是获取到读锁的次数；使用低16 位表示获取到写锁的线程的可重入次数。

```java
static final int SHARED_SHIFT   = 16;
//共享读锁状态，高16位
static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
//共享锁线程最大数65535
static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
//排他锁低16位
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

//返回持有读锁线程锁
/** Returns the number of shared holds represented in count  */
static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
//返回写锁可重入个数
/** Returns the number of exclusive holds represented in count  */
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```
&emsp;其中 firstReader 用来记录第一个获取到读锁的线程， firstReaderHoldCount 则记录第一个获取到读锁的线程获取读锁的可重入次数。cachedHoldCounter 用来记录最后一个获取读锁的线程获取读锁的可重入次数。
```java
private transient HoldCounter cachedHoldCounter;
private transient Thread firstReader = null;
private transient int firstReaderHoldCount;

static final class HoldCounter {
            int count = 0;
            // Use id, not reference, to avoid garbage retention
            final long tid = getThreadId(Thread.currentThread());
        }
```

## 2. 写锁的获取与释放
&emsp;在 ReentrantReadWriteLock 中写锁使用 WriteLock 来实现。
- lock()

&emsp;写锁是个独占锁， 某时只有一个线程可以获取该锁。如果当前没有线程获取到读锁和写锁， 则当前线程可以获取到写锁然后返回。如果当前己经有线程获取到读锁和写锁，则当前请求写锁的线程会被阻塞挂起。另外， 写锁是可重入锁，如果当前线程己经获取了该锁，再次获取只是简单地把可重入次数加 1 后直接返回。
```java
public void lock() {
            sync.acquire(1);
        }
public final void acquire(int arg) {
        //sync重写的tryAcquire方法
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
protected final boolean tryAcquire(int acquires) {
            Thread current = Thread.currentThread();
            int c = getState();
            int w = exclusiveCount(c);
            //(1 )说明读锁或者写锁已经被某线程获取
            if (c != 0) {
                // (Note: if c != 0 and w == 0 then shared count != 0)
                //(2) 结果为真的两种情况：
                //1. w==0，写锁没有被获取，说明被获取的是读锁，而当前想要获取写锁，失败，直接返回false
                //2. w!=0 && current != getExclusiveOwnerThread()，当前写锁已经被获取，而且不是被当前线程获取，false
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                //(3 )到这里说明当前线程已经获取了写锁，判断可重入次数
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // Reentrant acquire
                //(4) 设置可重入次数
                setState(c + acquires);
                return true;
            }
            //(5) c==0，第一个写线程获取写锁
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
            //(6) 获取写锁成功，设置标志
            setExclusiveOwnerThread(current);
            return true;
        }
```
&emsp;获取写锁的思路为：
- 1. 当前写锁或读锁已经被某线程持有
    - 1.1 读锁已经被持有，当前尝试获取写锁失败
    - 1.2 写锁已经被持有，并且写锁持有者不是当前线程，获取失败
    - 1.3 写锁已经被当前线程持有，判断可重入数量是否超出最大值
- 2. 写锁和读锁均没有被任何线程获取，则根据公平策略来获取写锁并设置标志

&emsp;这里的根据公平策略来获取写锁由代码(5) 处体现，writerShouldBlock() 函数公平锁和非公平锁有不同的实现。
&emsp;公平锁的实现为：
```java
final boolean writerShouldBlock() {
            return hasQueuedPredecessors();
        }
```
&emsp;这里使用 hasQueuedPredecessors 来判断当前线程节点是否有前驱节点等待获取锁，如果有则当前线程放弃获取写锁的权限。

&emsp;非公平锁的实现为：
```java
final boolean writerShouldBlock() {
            return false; // writers can always barge
        }
```
&emsp;这里直接返回 false，直接参与锁的竞争。

- tryLock()

&emsp;尝试获取写锁，如果当前没有其他线程持有写锁或者读锁，则当前线程获取写锁会成功， 然后返回 true 。如果当前己经有其他线程持有写锁或者读锁则该方法直接返回false,且当前线程并 **不会被阻塞**。如果当前线程已经持有了该写锁则简单增加AQS 的状态值后直接返回true。
```java
public boolean tryLock( ) {
            return sync.tryWriteLock();
        }
final boolean tryWriteLock() {
            Thread current = Thread.currentThread();
            int c = getState();
            if (c != 0) {
                int w = exclusiveCount(c);
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                if (w == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
            }
            if (!compareAndSetState(c, c + 1))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }
```
&emsp;tryWriteLock 方法和 tryAcquire 方法类似，不再赘述。

- unlock()

&emsp;尝试释放锁，如果当前线程持有该锁，调用该方法会让该线程对该线程持有的AQS状态值减1 ，如果减去1后当前状态值为0 则当前线程会释放该锁，否则仅仅减l而己。如果当前线程没有持有该锁而调用了该方法则会抛出Illega!MonitorStateException 异常，代码如下:
```java
public void unlock() {
            sync.release(1);
        }
public final boolean release(int arg) {
        if (tryRelease(arg)) {
            //激活阻塞队列里面的一个线程
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
protected final boolean tryRelease(int releases) {
            //是否写锁拥有者调用的unlock
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            int nextc = getState() - releases;
            //释放后写锁持有可重入数量是否为0
            boolean free = exclusiveCount(nextc) == 0;
            if (free)
                setExclusiveOwnerThread(null);
            setState(nextc);
            return free;
        }
```

## 3. 读锁的获取与释放
&emsp;在 ReentrantReadWriteLock 中写锁使用 ReadLock 来实现。

- lock()

&emsp;获取读锁，如果当前没有其他线程持有写锁，则当前线程可以获取读锁，AQS 的状态值 state 的高16位的值会增加l，然后方法返回。否则如果其他一个线程持有写锁， 则当前线程会被阻塞。
```java
public void lock() {
            sync.acquireShared(1);
        }
public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            //放进阻塞队列
            doAcquireShared(arg);
    }
protected final int tryAcquireShared(int unused) {
            //(1)获取当前状态值
            Thread current = Thread.currentThread();
            int c = getState();
            //(2)如果写锁被其它线程持有
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
            int r = sharedCount(c);
            //(3)尝试获取锁，多个读线程只有一个会成功，不成功的进入fullTryAcquireShared进行重试
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
                //第一个线程获取读锁
                if (r == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                //如果是第一个获取读锁的线程
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
            }
            //类似 tryAcquireShhared，但是是自旋获取
            //因为读锁是可共享的，只需要自旋获取，不能放进阻塞队列
            return fullTryAcquireShared(current);
        }
```
&emsp;获取读锁的过程为：
- 1. 当前写锁被其它线程持有，获取读锁失败；
- 2. 尝试获取读锁，失败则自旋重试；

&emsp;如果当前要获取读锁的线程己经持有了写锁， 则也可以获取读锁。但是需要注意，当一个线程先获取了写锁，然后获取了读锁处理事情完毕后，要记得把读锁和写锁都释放掉，不能只释放写锁。
代码(3)处的readerShouldBlock方法公平锁和非公平锁有不同的实现。
&emsp;非公平锁的实现为：
```java
final boolean readerShouldBlock() {
            return apparentlyFirstQueuedIsExclusive();
        }
final boolean apparentlyFirstQueuedIsExclusive() {
        Node h, s;
        return (h = head) != null &&
            (s = h.next)  != null &&
            !s.isShared()         &&
            s.thread != null;
    }
```
&emsp;apparentlyFirstQueuedIsExclusive方法的作用为，，如果队列里面存在一个元素，则判断第一个元素是不是正在尝试获取写锁，如果不是， 则当前线程判断当前获取读锁的线程是否达到了最大值，也就是说如果当前阻塞队列的第一个线程获取的不是写锁，则当前线程可以有机会竞争获得锁。
最后执行CAS 操作将AQS 状态值的高16 位值增加l。

&emsp;公平锁的实现为：
```java
final boolean readerShouldBlock() {
            return hasQueuedPredecessors();
        }
```
&emsp;如果当前阻塞队列还有线程，则当前线程放弃获取锁的权利。

&emsp;我们再看一下 tryAcquireShared 方法里面最后的 fullTryAcquireShared 方法：
```java
final int fullTryAcquireShared(Thread current) {
            /*
             * This code is in part redundant with that in
             * tryAcquireShared but is simpler overall by not
             * complicating tryAcquireShared with interactions between
             * retries and lazily reading hold counts.
             */
            HoldCounter rh = null;
            for (;;) {
                int c = getState();
                if (exclusiveCount(c) != 0) {
                    if (getExclusiveOwnerThread() != current)
                        return -1;
                    // else we hold the exclusive lock; blocking here
                    // would cause deadlock.
                } else if (readerShouldBlock()) {
                    // Make sure we're not acquiring read lock reentrantly
                    if (firstReader == current) {
                        // assert firstReaderHoldCount > 0;
                    } else {
                        if (rh == null) {
                            rh = cachedHoldCounter;
                            if (rh == null || rh.tid != getThreadId(current)) {
                                rh = readHolds.get();
                                if (rh.count == 0)
                                    readHolds.remove();
                            }
                        }
                        if (rh.count == 0)
                            return -1;
                    }
                }
                if (sharedCount(c) == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    if (sharedCount(c) == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                        cachedHoldCounter = rh; // cache for release
                    }
                    return 1;
                }
            }
        }
```
&emsp;fullTryAcquireShared 方法和 tryAcquireShared 类似，它们的不同之处在于，前者通过循环自旋获取。之所以以自旋的方式获取，是因为读锁是可共享的，只需要自旋获取，不能放进阻塞队列。

- unlock()
```java
public void unlock() {
            sync.releaseShared(1);
        }
public final boolean releaseShared(int arg) {
        //释放锁
        //返回结果为读锁持有数量是否为0，为0则唤醒阻塞线程
        if (tryReleaseShared(arg)) {
            //唤醒阻塞线程
            doReleaseShared();
            return true;
        }
        return false;
    }
protected final boolean tryReleaseShared(int unused) {
            Thread current = Thread.currentThread();
            ...
            //CAS自旋更新
            for (;;) {
                int c = getState();
                int nextc = c - SHARED_UNIT;
                if (compareAndSetState(c, nextc))
                    // Releasing the read lock has no effect on readers,
                    // but it may allow waiting writers to proceed if
                    // both read and write locks are now free.
                    return nextc == 0;
            }
        }
```

> 参考：
《Java 并发编程之美》

> 相关文章推荐：
[《Java 并发编程 之 AbstractQueuedSynchronizer》](http://zhoujiapeng.top/java/java-AbstractQueuedSynchronizer)
