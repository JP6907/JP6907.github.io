---
title: Java 并发容器 之 ConcurrentHashMap(一)
catalog: true
subtitle: ConcurrentHashMap
header-img: /img/article_header/article_header.png
tags:
  - java
  - 编程语言
  - 并发
categories:
  - java
---



# ConcurrentHashMap
&emsp;本文主要分析 JDK1.8 中的 ConcurrentHashMap。

## 1. 概述
&emsp;HashMap 是我们日常最常见的一种容器，它以键值对的形式完成对数据的存储，但众所周知，它在高并发的情境下是不安全的。ConcurrentHashMap 是 HashMap 的并发版本，它是线程安全的，并且在高并发的情境下，性能优于 HashMap 很多。

&emsp;jdk 1.7 采用分段锁技术，整个 Hash 表被分成多个段，每个段中会对应一个 Segment 段锁，段与段之间可以并发访问，但是多线程想要操作同一个段是需要获取锁的。所有的 put，get，remove 等方法都是根据键的 hash 值对应到相应的段中，然后尝试获取锁进行访问。

![HashMap](https://gitee.com/JP6907/Pic/raw/master/java/ConcurrentHashMapJDK7.png)
(图片来源网络)

&emsp;jdk 1.8 取消了基于 Segment 的分段锁思想，改用 CAS + synchronized 控制并发操作，在某些方面提升了性能。并且追随 1.8 版本的 HashMap 底层实现，使用数组+链表+红黑树进行数据存储。本篇主要介绍 1.8 版本的 ConcurrentHashMap 的具体实现。

![HashMap](https://gitee.com/JP6907/Pic/raw/master/java/ConcurrentHashMap.png)
(图片来源网络)

&emsp;毕竟 ConcurrentHaspMap 只在 HashMap 的基础上实现的，两者在思想会有很多重叠的地方，建议先阅读文章[《深入理解 HashMap》](http://zhoujiapeng.top/java/java-HashMap)之后再来阅读本文，可能会理解得比较透彻。

## 2. 重要属性和构造函数
&emsp;首先看一下一些重要的属性
```java
    //代表整个Hash表，容量总是2的n次幂
    transient volatile Node<K,V>[] table;
    //临时表，用于哈希表扩容，扩容完成后会被重置为 null。
    private transient volatile Node<K,V>[] nextTable;

    //baseCount+counterCells用来记录总元素的数量
    private transient volatile long baseCount;
    //2的n次幂
    private transient volatile CounterCell[] counterCells;
    //自旋锁，用于操作CounterCells
    private transient volatile int cellsBusy;

    //非常重要的属性，下面详细介绍
    private transient volatile int sizeCtl;

    //扩容时迁移数据用的，偏移量
    private transient volatile int transferIndex;    
```
&emsp;可以看到，这里用了baseCount+counterCells数组来记录元素总量，它的思想和 LongAddr 一样，主要是为了解决CAS自旋在高并发情况下的性能问题，具体可以参考文章[《Java 并发编程 之 原子操作类》](http://zhoujiapeng.top/java/java-atomicOperationClass)，
LongAddr，这里简单介绍一下：
&emsp;它的方法是在内部维护多个 Cell 变量，多个线程对 value 的 CAS 操作可以分散到多个 Cell 变量上，减少了争夺共享资源的并发量，最后,在获取元素总数量时, 是把所有 Cell 变量的 value 值累加后再加上 baseCount 返回。

&emsp;另外，这里有一个非常重要的属性：sizeCtl。无论是初始化哈希表，还是扩容 rehash 的过程，都是需要依赖这个关键属性的。该属性有以下几种取值：
- 0：默认值
- -1：代表哈希表正在进行初始化
- 大于0：相当于 HashMap 中的 threshold，表示阈值
- 小于-1：代表有多个线程正在进行扩容

&emsp;这个属性的使用是比较复杂的，在后面分析扩容过程结合实际场景再给出更详细的介绍。

&emsp;接下来看一下构造函数：
```java
public ConcurrentHashMap(int initialCapacity) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException();
        int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                   MAXIMUM_CAPACITY :
                   tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
        this.sizeCtl = cap;
    }
```
&emsp;这个初始化方法有点意思，通过提供初始容量，计算了 sizeCtl，sizeCtl = (1.5 * initialCapacity + 1)，然后向上取最近的 2 的 n 次方。这里的 (initialCapacity >>> 1) 就等于 0.5*initialCapacity。例如， initialCapacity 为 10，那么得到 sizeCtl 为 16，如果 initialCapacity 为 11，得到 sizeCtl 为 32。


## 2. put 方法
&emsp;对于 HashMap 来说，多线程并发添加元素会导致数据丢失等并发问题，那么 ConcurrentHashMap 又是如何做到并发添加的呢？

```java
public V put(K key, V value) {
        return putVal(key, value, false);
    }
    /** Implementation for put and putIfAbsent */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        //计算hash
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            //如果table还为初始化，那么初始化它
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            //根据hash找到相应的索引位置
            //如果该位置上为空，则使用CAS向该位置插入一个节点
            //注意这里给f和i赋值了
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    //跳出循环
                    break;                   // no lock when adding to empty bin
            }
            //MOVED说明该桶的首节点是 Forwarding 类型
            //Forwarding 类型说明正在扩容，但是该位置的数据已经完成迁移，需要协助扩容
            //协助扩容完成之后重新循环
            else if ((fh = f.hash) == MOVED)
                //协助扩容
                tab = helpTransfer(tab, f);
            else {//到这里说明，f是该位置的头节点，而且不为空
                V oldVal = null;
                //获取该头节点的监视器锁，准备插入新的节点
                synchronized (f) { 
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) { // 头结点的 hash 值大于 0，说明是链表
                            //binCount随着链表的遍历增长
                            binCount = 1;
                            //遍历链表
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                //发现具有相同key的节点
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    //判断是否可以覆盖
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                //最后一个节点，将新值插入到链表最后面
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {//红黑树
                            Node<K,V> p;
                            //注意这里是2，不会增长
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                //不等于0说明上面插入了新的节点
                if (binCount != 0) {
                    //红黑树的binCount = 2
                    //链表长度超过阈值，只有链表的情况才有可能是条件成立
                    if (binCount >= TREEIFY_THRESHOLD)
                        // 这个方法和 HashMap 中稍微有一点点不同，那就是它不是一定会进行红黑树转换，
                        // 如果当前数组的长度小于 64，那么会选择进行数组扩容，而不是转换为红黑树
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }
```
&emsp;代码中的注释已经说明了大致每一步的操作，下面总结一下整个流程：
- table是否初始化
    - 否，初始化
    - 是，根据key计算索引位置，该位置是否有数据
        - 无，CAS 插入一个新节点
        - 有，是否为 MOVED，即正在扩容？
            - 是，协助扩容
            - 否，锁住头节点，在该桶插入新的节点
- 最后，判断插入新节点后长度是否超过阈值
    - 是，判断Hash表长度是否小于64
        - 是，Hash 表扩容
        - 否，链表转化为红黑树

&emsp;建议对照着这个框架再次走读一下代码，能够更清晰地把握整个脉络。

&emsp;put的主要流程解释完了，但却留下几个问题：
1. 为什么节点的Hash会等于MOVED？(fh = f.hash) == MOVED
2. Hash表的初始化在并发情况下是怎么进行的？initTable()
3. 等于MOVED的时候需要协助扩容，那么协助扩容是怎么进行的？helpTransfer()
4. 什么情况下会触发扩容？treeifyBin()

&emsp;下面具体分析一下这几个问题。


## 3. MOVED & ForwardingNode
&emsp;首先看一下 ForwardingNode 的结构：
```java
static final class ForwardingNode<K,V> extends Node<K,V> {
        final Node<K,V>[] nextTable;
        ForwardingNode(Node<K,V>[] tab) {
            super(MOVED, null, null, null);
            this.nextTable = tab;
        }
        ...
    }
```
&emsp;可以看到，ForwardingNode 的构造函数将 MOVED 作为参数传入了父类的构造函数：
```java
Node(int hash, K key, V val, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.val = val;
            this.next = next;
        }
```
&emsp;Node 的 hash 被设置成了 MOVED，它的意思是标志当前的Hash表正处于扩容的状态，而且当前桶的数据已经迁移到了新的table。
&emsp;在 transfer 函数里面，当 Hash 表中的某个桶的数据迁移结束之后，就会该桶头节点设置为 ForwardingNode：
```java
ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
...
setTabAt(tab, i, fwd);
```
&emsp;其他线程一旦看到该位置的 hash 值为 MOVED，就不会进行迁移，跳过该节点

&emsp;另外，ForwardingNode 里面还有一个属性 nextTable，它是对全局 nextTable 的引用，所有参与扩容的线程都会想将数据迁移到全局的 nextTable，扩容完成之后 nextTable 会被赋给全局 table，然后 nextTable 被置为 null。


## 4. initTable()
&emsp;initTable 方法是一个初始化哈希表的操作，可能会有多个线程同时调用这个方法，但它同时只允许一个线程进行初始化操作。它的并发问题是通过对 sizeCtl 进行一个 CAS 操作来控制的。
```java
private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            //说明其它线程正在初始化或者扩容table
            if ((sc = sizeCtl) < 0)
                //暂时放弃初始化，下一次循环在次重试
                Thread.yield(); // lost initialization race; just spin
            //进入这里可能不是第一次循环，可能其他线程已经完成了初始化
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    //可能其它线程已经完成了table的初始化，需要再次判断
                    if ((tab = table) == null || tab.length == 0) {
                        // DEFAULT_CAPACITY 默认初始容量是 16
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        // 其实就是 0.75 * n，阈值
                        sc = n - (n >>> 2);
                    }
                } finally {
                    //初始化结束之后，设置为阈值
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```
&emsp;该方法的核心思想就是，只允许一个线程对表进行初始化，如果不巧有其他线程进来了，那么会让其他线程交出 CPU 等待下次系统调度。这样，保证了表同时只会被一个线程初始化。sizeCtl 为 -1 的时候就表示有其它线程正在执行初始化操作。


## 5. treeifyBin()
&emsp;在 putVal 方法的最后，当判断插入节点之后数量超过阈值，就有可能触发扩容操作：
```java
if (binCount >= TREEIFY_THRESHOLD)
    treeifyBin(tab, i);
```
&emsp;treeifyBin 这个方法和 HashMap 中稍微有一点点不同，那就是它不是一定会进行红黑树转换，如果当前数组的长度小于 64，那么会选择进行数组扩容，而不是转换为红黑树。
```java
private final void treeifyBin(Node<K,V>[] tab, int index) {
        Node<K,V> b; int n, sc;
        if (tab != null) {
            //长度小于64，进行数组扩容
            if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
                tryPresize(n << 1);
            //转化成红黑树
            else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
                //加锁
                synchronized (b) {
                    if (tabAt(tab, index) == b) {
                        TreeNode<K,V> hd = null, tl = null;
                        for (Node<K,V> e = b; e != null; e = e.next) {
                            TreeNode<K,V> p =
                                new TreeNode<K,V>(e.hash, e.key, e.val,
                                                  null, null);
                            if ((p.prev = tl) == null)
                                hd = p;
                            else
                                tl.next = p;
                            tl = p;
                        }
                        //将红黑树设置到数组相应的位置
                        setTabAt(tab, index, new TreeBin<K,V>(hd));
                    }
                }
            }
        }
    }
```
&emsp;treeifyBin 函数的逻辑比较简单，扩容是调用 tryPresize 函数，扩容后数组容量为原来的 2 倍。
```java
//size为目标容量
private final void tryPresize(int size) {
        /**
         * 1.计算新table的容量
         */
        //取最近的 2 的 n 次方。
        int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
            tableSizeFor(size + (size >>> 1) + 1);
        int sc;
        while ((sc = sizeCtl) >= 0) {//可以执行扩容操作
            /**
             * 2. 初始化table
             */
            Node<K,V>[] tab = table; int n;
            //如果未初始化，则初始化，和 initTable 函数差不多
            if (tab == null || (n = tab.length) == 0) {
                n = (sc > c) ? sc : c;
                if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                    try {
                        if (table == tab) {
                            @SuppressWarnings("unchecked")
                            //next table 新的table
                            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                            table = nt;
                            //新的阈值 0.75n
                            sc = n - (n >>> 2);
                        }
                    } finally {
                        sizeCtl = sc;
                    }
                }
            }
            else if (c <= sc || n >= MAXIMUM_CAPACITY)
                break;
            else if (tab == table) {
                //返回一个 16 位长度的扩容校验标识
                //第16位为1，1~15位和table的长度有关
                int rs = resizeStamp(n);
                /**
                 * 3.1 已经有其它线程正在扩容
                 */
                if (sc < 0) {// sc = sizeCtl 有其它线程正在扩容
                    Node<K,V>[] nt;\
                    //RESIZE_STAMP_SHIFT=16,不带符号右移动16，取高16位，高16位为扩容检验标识
                    //(sc >>> RESIZE_STAMP_SHIFT) != rs 扩容检验标识错误
                    //sc == rs + 1 没有线程正在扩容，前面判断 sc<0，所以表示有线程正在初始化table
                    //sc == rs + MAX_RESIZERS 当前参与扩容的线程数量已经达到上限
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    //sc+1，当前线程加入扩容
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        //数据迁移
                        //nt = nextTable
                        //已经有其它线程开始扩容，所以nt不为null
                        //表示迁移到已经存在的 nextTable
                        //tranfer 内部不需要去初始化 nextTable
                        transfer(tab, nt);
                }
                /**
                 * 3.2 没有其它线程正在扩容
                 */
                //当前没有其它线程在扩容，当前线程是第一个开始扩容的线程
                //将 sizeCtl 设置为 (rs << RESIZE_STAMP_SHIFT) + 2)
                // 参与扩容的线程数量是从+1开始，+2表示只有一个线程参与扩容，也就是当前线程
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                               (rs << RESIZE_STAMP_SHIFT) + 2))
                    //数据迁移
                    //传入的是null
                    //表示当前线程是第一个开始扩容的线程，transfer 内部会初始化 nextTable
                    transfer(tab, null);
            }
        }
    }
```
&emsp;tryPresize 的思路大概为：
- 1.计算新table的容量
- 2.如果table还未初始化，则初始化它
- 3.通过sizeCtl判断是否有其它线程正在扩容
    - 3.1 有其它线程正在扩容，此时nextTable不为null，当前线程加入扩容，协助将数据迁移到nextTable
    - 3.2 没有其它线程正在扩容，当前线程第一个开始扩容，此时nextTable为null，当前线程会在transfer方法里面初始化nextTable

&emsp;注意，当线程加入扩容的时候，需要验证扩容检验标识，每一次扩容，table的每一个新容量都会对应一个独一无二的检验标识，这个标识会被保存在 sizeCtl 里面，这能够确保所有线程都是在协助table扩容到相同大小。
扩容检验标识可以通过 resizeStamp 函数生成，下面我们看一下这个函数。

## 6. resizeStamp(int n)
```java
static final int resizeStamp(int n) {
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}
```
&emsp;这个函数很短，将 Integer.numberOfLeadingZeros(n) 和 (1 << (RESIZE_STAMP_BITS - 1)) 做与运算。
Integer.numberOfLeadingZeros(n) 函数的作用是返回无符号整型n的最高非零位前面的0的个数，包括符号位在内。
&emsp;举例来说：
n = 1 = 0000 0000 .... 0001, 前面有31个0，所以结果是31
n = 8 = 0000 0000 .... 1000, 前面有28个0，所以结果是28
&emsp;需要注意的是，table的容量，也就是n，总是为2的k次幂，n=2^k，也就是 n 的二进制表示只有一位是1，其它位都是0，所以 n 和 Integer.numberOfLeadingZeros(n) 是一一对应的，用这个很小的数来代表 n，可以确保15位以下能够储存。
&emsp;再看一下后面部分，这里的 RESIZE_STAMP_BITS =16，所以 (1 << (RESIZE_STAMP_BITS - 1)) = 1000 0000 0000 0000，用这个数和 Integer.numberOfLeadingZeros(n) 做与运算，resizeStamp 最终返回的是一个16位的整数，并且最高位为1，这个数对每次扩容都是不一样的，所以可以作为每次扩容时的数据检验标识。而这里为什么要将第16位置为1呢？
&emsp;我们知道，当 sizeCtl < -1 的时候，代表有多个线程正在执行扩容操作，此时 sizeCtl 的高16位会作为扩容检验标识，低16位则为 参与扩容的线程数量-1。回到 tryPresize 函数部分，可以看到，在代码3.2 那里，第一个开始扩容的线程会调用 resizeStamp 生成扩容检验标识 rs，然后设置 sizeCtl：
```java
U.compareAndSwapInt(this, SIZECTL, sc, (rs << RESIZE_STAMP_SHIFT) + 2)
```
&emsp;RESIZE_STAMP_SHIFT=16，此时sc的高16位为扩容检验标识rs，低16位为2，并且此时sc的最高位是1，也就是sc是一个负数，低16位表示参与扩容的线程数量+1.如果是从 1 开始，则有可能导致sc=-1，而sc=-1已经有特殊含义，表示正在初始化table，因此低16位是从2开始的。这也就解释了为什么当 sc<-1 时，表示当前正在执行扩容操作。


## 7. helpTransfer()
&emsp;在 tryPresize 函数里面最终会调用 transfer 函数去执行真正的数据迁移操作，transfer 函数比较长，留着[下一篇文章]()介绍，我们先来看一下 helpTransfer 函数，这个函数最终也会调用 transfer 函数。
&emsp;helpTransfer 从字面上理解就是帮助转移，也就时协助扩容：
```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        Node<K,V>[] nextTab; int sc;
        //table不为空，且头节点为ForwardingNode
        //表示当前正在扩容，并且当前索引位置已经完成数据迁移
        //nextTable是新table的引用
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
            //扩容校验标识
            int rs = resizeStamp(tab.length);
            while (nextTab == nextTable && table == tab &&
                   (sc = sizeCtl) < 0) {
                //(sc >>> RESIZE_STAMP_SHIFT) != rs 扩容检验标识错误
                //sc == rs + 1 没有线程正在扩容，前面判断 sc<0，所以表示有线程正在初始化table
                //sc == rs + MAX_RESIZERS 当前参与扩容的线程数量已经达到上限
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || transferIndex <= 0)
                    break;
                //sc+1 标识增加了一个线程进行扩容
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                    //进行扩容
                    transfer(tab, nextTab);
                    break;
                }
            }
            return nextTab;
        }
        return table;
    }
```

## 未完
&emsp;由于篇幅原因，关于 transfer 等其它函数的分析请看《Java 并发容器 之 ConcurrentHashMap(二)》


&nbsp;
>参考：
https://www.cnblogs.com/zaizhoumo/p/7709755.html
https://www.cnblogs.com/yangming1996/p/8031199.html
https://blog.csdn.net/sihai12345/article/details/79383766