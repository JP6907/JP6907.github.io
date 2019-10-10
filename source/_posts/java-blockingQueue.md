---
title: Java 并发编程 之 BlockingQueue
subtitle: 阻塞队列详解
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - java
  - 编程语言
  - 并发
categories:
  - java
date: 2019-10-10 16:06:35
---


# BlockingQueue 详解

## 1. 概述
&emsp;BlockingQueue是支持两个附加操作的Queue：
- 1）当Queue为空时，获取元素线程被阻塞直到Queue变为非空；
- 2）当Queue满时，添加元素线程被阻塞直到Queue不满。

BlockingQueue不允许元素为null，如果入队一个null元素，会抛NullPointerException。常用于生产者消费者模式。
BlockingQueue对于不能满足条件的操作，提供了四种处理方式：
- 1）直接抛异常，抛出异常。如果队列已满，添加元素会抛出IllegalStateException异常；如果队列为空，获取元素会抛出NoSuchElementException异常；
- 2）返回一个特殊值（null或false）；
- 3）在满足条件之前，无限期的阻塞当前线程，当队列满足条件或响应中断退出；
- 4）在有限时间内阻塞当前线程，超时后返回失败。

|  	|抛出异常| 返回特殊值| 阻塞| 超时|
| -- | -- | -- | -- |-- |
|入队 | add(e) | offer(e) | put(e) | offer(e, time, unit)|
|出队 | remove() | poll() | take() |poll(time, unit)|
|检查 | element() | peek()| | | |

JDK提供的阻塞队列：
- ArrayBlockingQueue：一个由数组结构组成的有界阻塞队列，遵循FIFO原则。
- LinkedBlockingQueue：一个由链表结构组成的有界阻塞队列，遵循FIFO原则，默认和最大长度为Integer.MAX_VALUE。
- PriorityBlockingQueue：一个支持优先级排序的无界阻塞队列。
- DelayQueue：一个使用优先级队列实现的无界阻塞队列。
- SynchronousQueue：一个不存储元素的阻塞队列。
- LinkedTransferQueue：一个由链表结构组成的无界阻塞队列。
- LinkedBlockingDeque：一个由链表结构组成的双向阻塞队列。


## 2. 使用
　　示例：生产者-消费者，BlockingQueue 可以安全地与多个生产者和多个使用者一起使用。
```java
class Producer implements Runnable {
    private final BlockingQueue queue;
    Producer(BlockingQueue q) { queue = q; }
    public void run() {
        try {
            while(true) { queue.put(produce()); }    //当队列满时，生产者阻塞等待
        } catch (InterruptedException ex) { ... handle ...}
    }
    Object produce() { ... }
}

//消费者
class Consumer implements Runnable {
    private final BlockingQueue queue;
    Consumer(BlockingQueue q) { queue = q; }
    public void run() {
    try {
        while(true) { consume(queue.take()); }    //当队列空时，消费者阻塞等待
    } catch (InterruptedException ex) { ... handle ...}
    }
    void consume(Object x) { ... }
}

class Setup {
    void main() {
        BlockingQueue q = new SomeQueueImplementation();
        Producer p = new Producer(q);
        Consumer c1 = new Consumer(q);
        Consumer c2 = new Consumer(q);
        new Thread(p).start();
        new Thread(c1).start();
        new Thread(c2).start();
    }
}
```

## 3. 实现原理
&emsp;当队列满时，生产者会一直阻塞，当消费者从队列中取出元素时，如何通知生产者队列可以继续，以ArrayBlockingQueue和LinkedBlockingQueue为例，分析源代码如何实现阻塞队列。它们的阻塞机制都是基于Lock和Condition实现，其中LinkedBlockingQueue还用到了原子变量类。

### 3.1 ArrayBlockingQueue
&emsp;基于数组的**有界**阻塞队列。队列按FIFO原则对元素进行排序，队列头部是在队列中存活时间最长的元素，队尾则是存在时间最短的元素。新元素插入到队列的尾部，队列获取操作则是从队列头部开始获得元素。 这是一个典型的“有界缓存区”，固定大小的数组在其中保持生产者插入的元素和使用者提取的元素。一旦创建了这样的缓存区，就不能再增加其容量。试图向已满队列中放入元素会导致操作受阻塞；试图从空队列中提取元素将导致类似阻塞。ArrayBlockingQueue构造方法可通过设置fairness参数来选择是否采用公平策略，公平性通常会降低吞吐量，但也减少了可变性和避免了“不平衡性”，可根据情况来决策。

域：
```java
/** The queued items */
final Object[] items;
/** items index for next take, poll, peek or remove */
int takeIndex;
/** items index for next put, offer, or add */
int putIndex;
/** Number of elements in the queue */
int count;
/*
 * Concurrency control uses the classic two-condition algorithm
 * found in any textbook.
 */

/** Main lock guarding all access */
final ReentrantLock lock;
/** Condition for waiting takes */
private final Condition notEmpty;
/** Condition for waiting puts */
private final Condition notFull;
```
&emsp;由ArrayBlockingQueue的域可以看出，使用循环数组存储队列中的元素，两个索引takeIndex和putIndex分别指向下一个要出队和入队的数组位置，线程间的通信是使用ReentrantLock和两个Condition实现的。

#### 3.1.1 put(e)&take() 阻塞
&emsp;当不满足入队或出队条件时，当前线程阻塞等待。即当队列满时，生产者会一直阻塞直到被唤醒，当队列空时，消费者会一直阻塞直到被唤醒。

**入队（put）**
```java
//在队列的尾部（当前putIndex指定的位置）插入指定的元素
public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();    //可响应中断获取锁
    try {
        while (count == items.length)    //如果队列满，在入队条件notFull的等待队列上等待。
                                        //这里使用While循环而非if判断，目的是防止过早或意外的通知，只有条件符合才能推出循环
            notFull.await();
        insert(e);
    } finally {
        lock.unlock();    //释放锁，唤醒同步队列中的后继节点
    }
}
//为保证操作线程安全，此方法必须在获取锁的前提下才能被调用
private void insert(E x) {
    items[putIndex] = x;
    putIndex = inc(putIndex);
    ++count;            //元素数量+1
    notEmpty.signal();    //唤醒出队条件的等待队列上的线程
}
//将i增1，当++i等于数组的最大容量时，将i置为0。即通过循环数组的方式
final int inc(int i) {
    return (++i == items.length) ? 0 : i;
}
```
&emsp;从源码可以看出，入队的大致步骤如下：
- 1）首先获取锁，如果获取锁失败，当前线程可能自旋获取锁或被阻塞直到获取到锁，否则执行2）；
- 2）循环判断队列是否满，如果满，那么当前线程被阻塞到notFull条件的等待队列中，并释放锁，等待被唤醒；
- 3）当队列非满或从await方法中返回（此时当前线程从等待队列中被唤醒并重新获取到锁）时，执行插入元素操作。
- 4）入队完成后，释放锁，唤醒同步队列中的后继节点。

**出队（take）**
```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();    //可响应中断获取锁
    try {
        while (count == 0)    //如果队列为空，在出队条件notEmpty的等待队列中等待
            notEmpty.await();
        return extract();
    } finally {
        lock.unlock();    //释放锁
    }
}
//在当前takeIndex指定的位置取出元素，此方法必须在获取锁的前提下才能被调用
private E extract() {
    final Object[] items = this.items;
    E x = this.<E>cast(items[takeIndex]);    //强制类型转换
    items[takeIndex] = null;
    takeIndex = inc(takeIndex);    //出队索引同样采用循环的方式增1
    --count;
    notFull.signal();    //唤醒入队条件的等待队列中的线程
    return x;
}
```
&emsp;从源码可以看出，出队的大致步骤如下：
- 1）首先获取锁，如果获取锁失败，当前线程可能自旋获取锁或被阻塞直到获取到锁成功。
- 2）获取锁成功，循环判断队列是否为空，如果为空，那么当前线程被阻塞到 notEmpty 条件的等待队列中，并释放锁，等待被唤醒；
- 3）当队列非空或从await方法中返回（此时当前线程从等待队列中被唤醒并重新获取到锁）时，执行取出元素操作。
- 4）出队完成后，释放锁，唤醒同步队列的后继节点，

#### 3.1.2 offer(e)&poll() 返回特殊值
&emsp;当不能满足入队或出队条件时，返回特殊值。当队列满时，入队会失败，offer方法直接返回false，反之入队成功，返回true；当队列空时，poll方法返回null。

**入队（offer）**
```java
public boolean offer(E e) {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lock();    //获取锁
    try {
        if (count == items.length)    //如果队列满，与put阻塞当前线程不同的是，offer方法直接返回false
            return false;
        else {
            insert(e);
            return true;
        }
    } finally {
        lock.unlock();    //释放锁
    }
}
```

**出队（poll）**
```java
//出队
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();    //获取锁
    try {
        return (count == 0) ? null : extract();    //如果队列空，与take阻塞当前线程不同的是，poll方法返回null
    } finally {
        lock.unlock();    //释放锁
    }
}
```

#### 3.1.3 add(e)&remove() 抛异常
&emsp;当不能满足入队或出队条件时，直接抛出异常。当队列满时，入队失败，抛IllegalStateException("Queue full")；当队列空时，remove方法抛NoSuchElementException()异常。

**入队（add）**
```java
public boolean add(E e) {
    return super.add(e);
}

//抽象类AbstractQueue提供的方法
public boolean add(E e) {
    //如果offer返回true，那么add方法返回true；如果offer返回false，那么add方法抛IllegalStateException("Queue full")异常
    if (offer(e))
        return true;
    else
        throw new IllegalStateException("Queue full");
}
```

**出队（remove）**
```java
//抽象类AbstractQueue提供的方法
public E remove() {
    E x = poll();
    if (x != null)
        return x;
    else
        throw new NoSuchElementException();
}
```

#### 3.1.4 offer&poll 超时
&emsp;使用Condition的超时等待机制实现，当不满足条件时，只在有限的时间内阻塞，超过超时时间仍然不满足条件才返回false或null。

**入队**（offer(E e, long timeout, TimeUnit unit)）
```java
public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {

    checkNotNull(e);
    long nanos = unit.toNanos(timeout);    //转换为纳秒
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == items.length) {
            if (nanos <= 0)
                return false;
            nanos = notFull.awaitNanos(nanos);    //与offer直接返回false不同，此处使用Condition的超时等待机制实现，超过等待时间如果仍然不满足条件才返回false
        }
        insert(e);
        return true;
    } finally {
        lock.unlock();
    }
}
```

**出队**（poll(long timeout, TimeUnit unit)）
```java
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0) {
            if (nanos <= 0)
                return null;
            nanos = notEmpty.awaitNanos(nanos);    
        }
        return extract();
    } finally {
        lock.unlock();
    }
}
```


### 3.2 LinkedBlockingQueue
&emsp;基于链表的无界(默认Integer.MAX_VALUE，也可自己指定容量)阻塞队列。与ArrayBlockingQueue一样采用FIFO原则对元素进行排序。基于链表的队列吞吐量通常要高于基于数组的队列。
&emsp;**Executors**创建固定大小线程池的代码，就使用了LinkedBlockingQueue来作为任务队列。

域
```java
/** The capacity bound, or Integer.MAX_VALUE if none */
private final int capacity;    //队列最大容量，默认为Integer.MAX_VALUE
/** Current number of elements */
private final AtomicInteger count = new AtomicInteger(0);    //当前元素数量，原子类保证线程安全
/**
 * Head of linked list.
 * Invariant: head.item == null 
 */
private transient Node<E> head;    //队列的首节点，head节点是个空节点，head.item == null，实际存储元素的第一个节点是head.next
/**
 * Tail of linked list.
 * Invariant: last.next == null
 */
private transient Node<E> last;    //队列的尾节点
/** Lock held by take, poll, etc */
private final ReentrantLock takeLock = new ReentrantLock();    //出队锁
/** Wait queue for waiting takes */
private final Condition notEmpty = takeLock.newCondition();    //出队条件
/** Lock held by put, offer, etc */
private final ReentrantLock putLock = new ReentrantLock();    //入队锁
/** Wait queue for waiting puts */
private final Condition notFull = putLock.newCondition();    //入队条件
```

Node类：
```java
static class Node<E> {
    E item;    //元素
    /**
     * One of:
     * - the real successor Node
     * - this Node, meaning the successor is head.next
     * - null, meaning there is no successor (this is the last node)
     */
    Node<E> next;    //后继节点，LinkedBlockingQueue使用的是单向链表
    Node(E x) { item = x; }
}
```
&emsp;由LinkedBlockingQueue的域可以看出，它使用链表存储元素。线程间的通信也是使用ReentrantLock和Condition实现的，与ArrayBlockingQueue不同的是，LinkedBlockingQueue在入队和出队操作时分别使用两个锁putLock和takeLock。

- 思考问题一：为什么使用两把锁？

&emsp;为了提高并发度和吞吐量，使用两把锁，takeLock只负责出队，putLock只负责入队，入队和出队可以同时进行，提高入队和出队操作的效率，增大队列的吞吐量。LinkedBlockingQueue队列的吞吐量通常要高于ArrayBlockingQueue队列，但是在高并发条件下可预测性降低。

- 思考问题二：ArrayBlockingQueue中的count是一个普通的int型变量，LinkedBlockingQueue的count为什么是AtomicInteger类型的？

&emsp;因为ArrayBlockingQueue的入队和出队操作使用同一把锁，对count的修改都是在处于线程获取锁的情况下进行操作，因此不会有线程安全问题。而LinkedBlockingQueue的入队和出队操作使用的是不同的锁，会有对count变量并发修改的情况，所以使用原子变量保证线程安全。

- 思考问题三：像notEmpty、takeLock、count域等都声明为final型，final成员变量有什么特点？

&emsp;1）对于一个final变量，如果是基本数据类型的变量，则其数值一旦在初始化之后便不能更改；如果是引用类型的变量，则在对其初始化之后便不能再让其指向另一个对象。
&emsp;2）对于一个final成员变量，必须在定义时或者构造器中进行初始化赋值，而且final变量一旦被初始化赋值之后，就不能再被赋值了。只要对象是正确构造的（被构造对象的引用在构造函数中没有“逸出”），那么不需要使用同步（指lock和volatile的使用）就可以保证任意线程都能看到这个final域在构造函数中被初始化之后的值。

**初始化**
```java
//指定容量，默认为Integer.MAX_VALUE
public LinkedBlockingQueue(int capacity) {
    if (capacity <= 0) throw new IllegalArgumentException();
    this.capacity = capacity;
    last = head = new Node<E>(null);    //构造元素为null的head节点，并将last指向head节点
}
```

#### 3.2.1 put&take 阻塞

**阻塞式入队（put）**
```java
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    // Note: convention in all put/take/etc is to preset local var 预置本地变量，例如入队锁赋给局部变量putLock
    // holding count negative to indicate failure unless set.
    int c = -1;
    Node<E> node = new Node(e);    //构造新节点
    //预置本地变量putLock和count
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    putLock.lockInterruptibly();    //可中断获取入队锁
    try {
        /*
         * Note that count is used in wait guard even though it is
         * not protected by lock. This works because count can
         * only decrease at this point (all other puts are shut
         * out by lock), and we (or some other waiting put) are
         * signalled if it ever changes from capacity. Similarly
         * for all other uses of count in other wait guards.
         */
        while (count.get() == capacity) {
            notFull.await();
        }
        enqueue(node);    //在队尾插入node
        c = count.getAndIncrement();    //count原子方式增1，返回值c为count增长之前的值
        if (c + 1 < capacity)    //如果队列未满，通知入队线程（notFull条件等待队列中的线程）
            notFull.signal();
    } finally {
        putLock.unlock();    //释放入队锁
    }
    //如果入队该元素之前队列中元素数量为0，那么通知出队线程（notEmpty条件等待队列中的线程）
    if (c == 0)
        signalNotEmpty();
}

//通知出队线程（notEmpty条件等待队列中的线程）
private void signalNotEmpty() {
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();    //获取出队锁，调用notEmpty条件的方法的前提
    try {
        notEmpty.signal();    //唤醒一个等待出队的线程
    } finally {
        takeLock.unlock();    //释放出队锁
    }
}

private void enqueue(Node<E> node) {
    // assert putLock.isHeldByCurrentThread();
    // assert last.next == null;
    last = last.next = node;
}
```

- 思考问题一：为什么要再声明一个final局部变量指向putLock和count，直接使用成员变量不行吗？

&emsp;直接使用成员变量：每次调用putLock的方法，都需要先通过this指针找到Heap中的Queue实例，然后在根据Queue实例的putLock域引用找到Lock实例，最后才能调用Lock的方法（即将相应的方法信息组装成栈帧压入栈顶）。声明一个final局部变量指向putLock：先通过this指针找到Heap中的Queue实例，将Queue实例的putLock域存储的Lock实例的地址赋给局部变量putLock，以后需要调用putLock的方法时，直接使用局部变量putLock引用就可以找到Lock实例。简化了查找Lock实例的过程。count变量也是同样的道理。个人理解应该是为了提升效率。

- 思考问题二：使用两把锁怎么保证元素的可见性？

&emsp;例如：入队线程使用put方法在队列尾部插入一个元素，怎么保证出队线程能看到这个元素？ArrayBlockingQueue的入队和出队使用同一个锁，所以没有可见性问题。
&emsp;在LinkedBlockingQueue中，每次一个元素入队， 都需要获取putLock和更新count，而出队线程为了保证可见性，需要获取fullyLock（fullyLock方法用于一些批量操作,对全局加锁）或者获取takeLock，然后读取count.get()。因为volatile对象的写操作happen-before读操作,也就是写线程先写的操作对随后的读线程是可见的，volatile相当于一个内存屏障,volatile后面的指令不允许重排序到它之前，而count是原子整型类，是基于volatile变量和CAS机制实现。所以就保证了可见性，写线程修改count-->读线程读取count-->读线程。

- 思考问题三：在put方法中，为什么唤醒出队线程的方法signalNotEmpty()要放在释放putLock锁（putLock.unlock()）之后？同样，take也有同样的疑问？

&emsp;避免死锁的发生，因为signalNotEmpty()方法中要获取takeLock锁。如果放在释放putLock之前，相当于在入队线程需要先获取putLock锁，再获取takeLock锁。例如：当入队线程先获取到putLock锁，并尝试获取takeLock锁，出队线程获取到takeLock锁，并尝试获取putLock锁时，就会产生死锁。

- 思考问题四：什么是级联通知？

&emsp;比如put操作会调用notEmpty的notify,只会唤醒一个等待的读线程来take,take之后如果发现还有剩余的元素,会继续调用notify,通知下一个线程来获取。

**阻塞式出队（take）**
```java
public E take() throws InterruptedException {
    E x;
    int c = -1;
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();    //可中断获取出队锁
    try {
        while (count.get() == 0) {    //如果队列为空，阻塞线程同时释放锁
            notEmpty.await();
        }
        x = dequeue();    //从队列头弹出元素
        c = count.getAndDecrement(); //count原子式递减
        //c>1说明本次出队后，队列中还有元素
        if (c > 1)
            notEmpty.signal();    //唤醒一个等待出队的线程
    } finally {
        takeLock.unlock();    //释放出队锁
    }
    //c == capacity说明本次出队之前是满队列，唤醒一个等待NotFull的线程
    if (c == capacity)
        signalNotFull();
    return x;
}

//唤醒一个等待NotFull条件的线程
private void signalNotFull() {
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        notFull.signal();
    } finally {
        putLock.unlock();
    }
}

//从队列头弹出元素
private E dequeue() {
    // assert takeLock.isHeldByCurrentThread();
    // assert head.item == null;
    Node<E> h = head;
    Node<E> first = h.next;
    h.next = h; // help GC
    head = first;
    E x = first.item;
    first.item = null;
    return x;
}
```

#### 3.2.2 remove & contains & Iterator
**1）remove**

&emsp;删除指定元素,**全局加锁**
```java
public boolean remove(Object o) {   //正常用不到remove方法,queue正常只使用入队出队操作.
    if (o == null) return false;
    fullyLock();                    // 两个锁都锁上了,禁止进行入队出队操作.
    try {
        for (Node<E> trail = head, p = trail.next;
             p != null;             // 按顺序一个一个检索,直到p==null
             trail = p, p = p.next) {
            if (o.equals(p.item)) { //找到了,就删除掉
                unlink(p, trail);   //删除操作是一个unlink方法,意思是p从LinkedList链路中解除.
                return true;        //返回删除成功
            }
        }
        return false;
    } finally {
        fullyUnlock();
    }
}
```

**2）contains**

&emsp;判断是否包含指定的元素,**全局加锁**
```java
public boolean contains(Object o) {     //这种需要检索的操作都是对全局加锁的,很影响性能,要小心使用!
    if (o == null) return false;
    fullyLock();
    try {
        for (Node<E> p = head.next; p != null; p = p.next)
            if (o.equals(p.item))
                return true;
        return false;
    } finally {
        fullyUnlock();
    }
}
```
&emsp;**remove和contains这种需要检索的操作都是对全局加锁的,很影响性能,要小心使用!**
&emsp;全局加锁和解锁的方法作为公共的方法供其他需要全局锁的方法调用，避免由于获取锁的顺序不一致导致死锁。另外fullyLock和fullyUnlock两个方法对锁的操作要相反。

```java
/**
 * Lock to prevent both puts and takes.
 */
void fullyLock() {
    putLock.lock();
    takeLock.lock();
}

/**
 * Unlock to allow both puts and takes.
 */
void fullyUnlock() {
    takeLock.unlock();
    putLock.unlock();
}
```

**3）迭代器Iterator**
&emsp;弱一致性，不会不会抛出ConcurrentModificationException异常，不会阻止遍历的时候对queue进行修改操作,可能会遍历到修改操作的结果.

### 3.3 LinkedBlockingQueue和ArrayBlockingQueue对比
&emsp;ArrayBlockingQueue由于其底层基于数组，并且在创建时指定存储的大小，在完成后就会立即在内存分配固定大小容量的数组元素，因此其存储通常有限，故其是一个“有界“的阻塞队列；而LinkedBlockingQueue可以由用户指定最大存储容量，也可以无需指定，如果不指定则最大存储容量将是Integer.MAX_VALUE，即可以看作是一个“无界”的阻塞队列，由于其节点的创建都是动态创建，并且在节点出队列后可以被GC所回收，因此其具有灵活的伸缩性。但是由于ArrayBlockingQueue的有界性，因此其能够更好的对于性能进行预测，而LinkedBlockingQueue由于没有限制大小，当任务非常多的时候，不停地向队列中存储，就有可能导致**内存溢出**的情况发生。
&emsp;其次，ArrayBlockingQueue中在入队列和出队列操作过程中，使用的是同一个lock，所以即使在多核CPU的情况下，其读取和操作的都无法做到并行，而LinkedBlockingQueue的读取和插入操作所使用的锁是两个不同的lock，它们之间的操作互相不受干扰，因此两种操作可以并行完成，故LinkedBlockingQueue的吞吐量要高于ArrayBlockingQueue。


### 3.4 延迟队列 DelayQueue
&emsp;Delayed 元素的一个无界阻塞队列，只有在延迟期满时才能从中提取元素。该队列的头部 是延迟期满后保存时间最长的 Delayed 元素。如果延迟都还没有期满，则队列没有头部，并且 poll 将返回 null。当一个元素的 getDelay(TimeUnit.NANOSECONDS) 方法返回一个小于等于 0 的值时，将发生到期。即使无法使用 take 或 poll 移除未到期的元素，也不会将这些元素作为正常元素对待。例如，size 方法同时返回到期和未到期元素的计数。此队列不允许使用 null 元素。


### 3.5 具有优先级的阻塞队列 PriorityBlockingQueue
&emsp;基于优先级的无界阻塞队列。优先级队列的元素按照其自然顺序进行排序，或者根据构造队列时提供的 Comparator 进行排序，具体取决于所使用的构造方法。优先级队列不允许使用 null 元素。依靠自然顺序的优先级队列还不允许插入不可比较的对象（这样做可能导致 ClassCastException）。虽然此队列逻辑上是无界的，但是资源被耗尽时试图执行 add 操作也将失败（导致 OutOfMemoryError）。
&emsp;一个**无界**阻塞队列，它使用与类 PriorityQueue 相同的顺序规则，并且提供了阻塞获取操作。虽然此队列逻辑上是无界的，但是资源被耗尽时试图执行 add 操作也将失败（导致OutOfMemoryError）。此类不允许使用 null 元素。依赖自然顺序的
优先级队列也不允许插入不可比较的对象（这样做会导致抛出 ClassCastException）。
&emsp;此类及其迭代器可以实现 Collection 和 Iterator 接口的所有可选 方法。iterator() 方法中提供的迭代器并不 保证以特定的顺序遍历 PriorityBlockingQueue 的元素。如果需要有序地进行遍历，则应考虑使用 Arrays.sort(pq.toArray())。此外，可以使用方法 drainTo 按优先级顺序移除 全部或部分元素，并将它们放在另一个 collection 中。
&emsp;在此类上进行的操作不保证具有同等优先级的元素的顺序。如果需要实施某一排序，那么可以定义自定义类或者比较器，比较器可使用修改键断开主优先级值之间的联系。例如，以下是应用先进先出 (first-in-first-out) 规则断开可比较元素之间联系的一个类。要使用该类，则需要插入一个新的 FIFOEntry(anEntry) 来替换普通的条目对象。


### 3.6 同步队列 SynchronousQueue
&emsp;同步的阻塞队列。其中每个插入操作必须等待另一个线程的对应移除操作，等待过程一直处于阻塞状态，同理，每一个移除操作必须等到另一个线程的对应插入操作。SynchronousQueue没有任何容量。不能在同步队列上进行 peek，因为仅在试图要移除元素时，该元素才存在；除非另一个线程试图移除某个元素，否则也不能（使用任何方法）插入元素；也不能迭代队列，因为其中没有元素可用于迭代。Executors.newCachedThreadPool使用了该队列。
&emsp;SynchronousQueue 是一个特殊的队列，它的内部同时只能够容纳单个元素。如果该队列已有一元素的话，试图向队列中插入一个新元素的线程将会阻塞，直到另一个线程将该元素从队列中抽走。同样，如果该队列为空，试图向队列中抽取一个元素的线程将会阻塞，直到另一个线程向队列中插入了一条新的元素。
&emsp;据此，把这个类称作一个队列显然是夸大其词了。它更多像是一个汇合点。


### 3.7 阻塞双端队列 BlockingDeque
&emsp;java.util.concurrent 包里的 BlockingDeque 接口表示一个线程安放入和提取实例的双端队列。
&emsp;BlockingDeque 类是一个双端队列，在不能够插入元素时，它将阻塞住试图插入元素的线程；在不能够抽取元素时，它将阻塞住试图抽取的线程。
&emsp;deque(双端队列) 是 "Double Ended Queue" 的缩写。因此，双端队列是一个你可以从任意一端插入或者抽取元素的队列。



### 3.8 链阻塞双端队列 LinkedBlockingDeque
&emsp;一个基于已链接节点的、任选范围的阻塞双端队列。
&emsp;可选的容量范围构造方法参数是一种防止过度膨胀的方式。如果未指定容量，那么容量将等于 Integer.MAX_VALUE。只要插入元素不会使双端队列超出容量，每次插入后都将动态地创建链接节点。
大多数操作都以固定时间运行（不计阻塞消耗的时间）。异常包括 remove、removeFirstOccurrence、removeLastOccurrence、contains、iterator.remove() 以及批量操作，它们均以线性时间运行。


&nbsp;
&nbsp;
> 参考链接：
https://www.cnblogs.com/zaizhoumo/p/7786793.html
https://blog.csdn.net/wtopps/article/details/80682267
https://blog.csdn.net/qq_42135428/article/details/80285737