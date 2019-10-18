---
title: Java 并发容器 之 ConcurrentHashMap
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

说实话，Java8 ConcurrentHashMap 源码真心不简单，最难的在于扩容，数据迁移操作不容易看懂。

# ConcurrentHashMap
本文主要分析 JDK1.8 中的 ConcurrentHashMap。
## 1. 概述
HashMap 是我们日常最常见的一种容器，它以键值对的形式完成对数据的存储，但众所周知，它在高并发的情境下是不安全的。ConcurrentHashMap 是 HashMap 的并发版本，它是线程安全的，并且在高并发的情境下，性能优于 HashMap 很多。

jdk 1.7 采用分段锁技术，整个 Hash 表被分成多个段，每个段中会对应一个 Segment 段锁，段与段之间可以并发访问，但是多线程想要操作同一个段是需要获取锁的。所有的 put，get，remove 等方法都是根据键的 hash 值对应到相应的段中，然后尝试获取锁进行访问。

![HashMap](https://github.com/JP6907/Pic/blob/master/java/ConcurrentHashMapJDK7.png?raw=true)
(图片来源网络)

jdk 1.8 取消了基于 Segment 的分段锁思想，改用 CAS + synchronized 控制并发操作，在某些方面提升了性能。并且追随 1.8 版本的 HashMap 底层实现，使用数组+链表+红黑树进行数据存储。本篇主要介绍 1.8 版本的 ConcurrentHashMap 的具体实现。

![HashMap](https://github.com/JP6907/Pic/blob/master/java/ConcurrentHashMap.png?raw=true)
(图片来源网络)

毕竟 ConcurrentHaspMap 只在 HashMap 的基础上实现的，两者在思想会有很多重叠的地方，建议先阅读文章[《深入理解 HashMap》](http://zhoujiapeng.top/java/java-HashMap)之后再来阅读本文，可能会理解得比较透彻。

## 2. 重要属性和构造函数
首先看一下一些重要的属性
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
可以看到，这里用了baseCount+counterCells数组来记录元素总量，它的思想和 LongAddr 一样，主要是为了解决CAS自旋在高并发情况下的性能问题，具体可以参考文章[《Java 并发编程 之 原子操作类》](http://zhoujiapeng.top/java/java-atomicOperationClass)，
LongAddr，这里简单介绍一下：
它的方法是在内部维护多个 Cell 变量，多个线程对 value 的 CAS 操作可以分散到多个 Cell 变量上，减少了争夺共享资源的并发量，最后,在获取元素总数量时, 是把所有 Cell 变量的 value 值累加后再加上 baseCount 返回。

另外，这里有一个非常重要的属性：sizeCtl。无论是初始化哈希表，还是扩容 rehash 的过程，都是需要依赖这个关键属性的。该属性有以下几种取值：
- 0：默认值
- -1：代表哈希表正在进行初始化
- 大于0：相当于 HashMap 中的 threshold，表示阈值
- 小于-1：代表有多个线程正在进行扩容

这个属性的使用是比较复杂的，在后面分析扩容过程结合实际场景再给出更详细的介绍。

接下来看一下构造函数：
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
这个初始化方法有点意思，通过提供初始容量，计算了 sizeCtl，sizeCtl = (1.5 * initialCapacity + 1)，然后向上取最近的 2 的 n 次方。这里的 (initialCapacity >>> 1) 就等于 0.5*initialCapacity。例如， initialCapacity 为 10，那么得到 sizeCtl 为 16，如果 initialCapacity 为 11，得到 sizeCtl 为 32。


## 2. put 方法
对于 HashMap 来说，多线程并发添加元素会导致数据丢失等并发问题，那么 ConcurrentHashMap 又是如何做到并发添加的呢？

ForwardingNode 


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
代码中的注释已经说明了大致每一步的操作，下面总结一下整个流程：
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

建议对照着这个框架再次走读一下代码，能够更清晰地把握整个脉络。

put的主要流程解释完了，但却留下几个问题：
1. 为什么节点的Hash会等于MOVED？(fh = f.hash) == MOVED
2. Hash表的初始化在并发情况下是怎么进行的？initTable()
3. 等于MOVED的时候需要协助扩容，那么协助扩容是怎么进行的？helpTransfer()
4. 什么情况下会触发扩容？treeifyBin()

下面具体分析一下这几个问题。


## 3. MOVED & ForwardingNode
首先看一下 ForwardingNode 的结构：
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
可以看到，ForwardingNode 的构造函数将 MOVED 作为参数传入了父类的构造函数：
```java
Node(int hash, K key, V val, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.val = val;
            this.next = next;
        }
```
Node 的 hash 被设置成了 MOVED，它的意思是标志当前的Hash表正处于扩容的状态，而且当前桶的数据已经迁移到了新的table。
在 transfer 函数进行数据迁移的时候有这样的操作：
```java
ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
...
setTabAt(tab, i, fwd);
```
其他线程一旦看到该位置的 hash 值为 MOVED，就不会进行迁移，跳过该节点

另外，ForwardingNode 里面还有一个属性 nextTable，它是对全局 nextTable 的引用，所有参与扩容的线程都会想将数据迁移到全局的 nextTable，扩容完成之后 nextTable 会被赋给全局 table，然后 nextTable 被置为 null。


## 4.
## 5.
## 6.





resizeStamp


put
putVal  initTable    helpTransfer  treeifyBin  addCount

initTable
treeifyBin  tryPresize
tryPresize 扩容  resizeStamp
resizeStamp
transfer
helpTransfer

addCount  -> LongAdder
remove replaceNode
size
sumCount
get
clear




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



这个方法和 HashMap 中稍微有一点点不同，那就是它不是一定会进行红黑树转换，
如果当前数组的长度小于 64，那么会选择进行数组扩容，而不是转换为红黑树
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

扩容操作，扩容后数组容量为原来的 2 倍。
```java
private final void tryPresize(int size) {
        //取最近的 2 的 n 次方。
        int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
            tableSizeFor(size + (size >>> 1) + 1);
        int sc;
        while ((sc = sizeCtl) >= 0) {//可以执行扩容操作
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
                //返回一个 16 位长度的扩容校验标识？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？
                //第16位为1
                int rs = resizeStamp(n);
                if (sc < 0) {// sc = sizeCtl 有其它线程正在扩容
                    Node<K,V>[] nt;\
                    //RESIZE_STAMP_SHIFT=16,不带符号右移动16，取高16位，高16位为数据检验标识
                    //(sc >>> RESIZE_STAMP_SHIFT) != rs 数据检验标识错误
                    //sc == rs + 1 ？？？？？？？？？？？？？？？？？？？？？？？？

                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        //数据迁移
                        //nt = nextTable
                        //已经有其它线程开始扩容，所以nt不为null
                        //表示迁移到已经存在的 nextTable
                        //tranfer 内部不需要去初始化 nextTable
                        transfer(tab, nt);
                }
                //当前没有其它线程在扩容，当前线程是第一个开始扩容的线程
                //将 sizeCtl 设置为 (rs << RESIZE_STAMP_SHIFT) + 2)
                //这是一个比较大的负数，表示参与扩容的线程增加了？？？？？？？？？？？？？？？？？？？？？
                //设置sc的高16位为rs，低16位为 2，表示当前只有一个线程正在工作
                //rs的第16位为1，RESIZE_STAMP_SHIFT=16
                //设置之后sc的最高位为1，所以sc<0
                //因为初始化时设置sc=-1，而 -1(补码) = 1111 111 .... 1111 (32个1)，
                //可能存在 (rs << RESIZE_STAMP_SHIFT) + k = -1 的情况
                //所以从2开始， +2 表示 只有一个线程
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
```java
static final int resizeStamp(int n) {
        //如果 n<2^31:
        //  Integer.numberOfLeadingZeros(n) > 0
        //如果 n=2^31
        //  Integer.numberOfLeadingZeros(n) = 0
        // (1 << (RESIZE_STAMP_BITS - 1) = 0000 0000 0000 0000 1000 0000 0000 0000
        // 做与运算
        //因为 n = 2^k 有且只有一位是1，
        //Integer.numberOfLeadingZeros(n) 返回n最高非零位前面的0的个数，用这个很小的数来代表 n
        //可以确保15位以下能够储存
        return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
    }
```
resizeStamp 返回的是一个不超过16位的整数，且第16位为1，0~15位跟 table 的大小 n 有关
当正在扩容是，resizeStamp 会被设置到 sc(sizeCtl) 的高16位，所以 sc 会成为一个负数
    
我们说过 resizeStamp(n) 返回的是对 n 的一个数据校验标识，占 16 位。而 RESIZE_STAMP_SHIFT 的值为 16，那么位运算后，整个表达式必然在右边空出 16 个零。
也正如我们所说的，sizeCtl 的高 16 位为数据校验标识，低 16 为表示正在进行扩容的线程数量。

(resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2 表示当前只有一个线程正在工作，相对应的，如果 (sc - 2) == resizeStamp(n) << RESIZE_STAMP_SHIFT，说明当前线程就是最后一个还在扩容的线程，那么会将 finishing 标识为 true，并在下一次循环中退出扩容方法。

加上这个数据检验标识能够确保扩容的时候所有线程都是在对相同大小的table进行操作。


helpTransfer 也会调用 transfer
让当前线程去协助扩容
```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        Node<K,V>[] nextTab; int sc;
        //table不为空，且头节点为ForwardingNode
        //表示当前正在扩容，并且当前索引位置已经完成数据迁移
        //nextTable是新table的引用
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
            //返回一个 16 位长度的扩容校验标识
            int rs = resizeStamp(tab.length);
            while (nextTab == nextTable && table == tab &&
                   (sc = sizeCtl) < 0) {
                //sizeCtl 如果处于扩容状态的话
                //前 16 位是数据校验标识，后 16 位是当前正在扩容的线程总数
                //这里判断校验标识是否相等，如果校验符不等或者扩容操作已经完成了，直接退出循环，不用协助它们扩容了
                //？？？？？？？？？？？？？？？？？？？？？？？？？？？？
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



数据迁移：transfer
```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        /**
         * 第一部分
         * 完成单个线程能处理的最少桶结点个数的计算和一些属性的初始化操作。
         */
        int n = tab.length, stride;
        //计算单个线程允许处理的最少table桶首节点个数，不能小于 16
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        //刚开始扩容，初始化nextTab
        if (nextTab == null) {            // initiating
            try {
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;
            //transferIndex 指向最后一个桶，方便从后向前遍历 
            transferIndex = n;
        }
        int nextn = nextTab.length;
        //定义 ForwardingNode 用于标记迁移完成的桶
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);

        /**
         * 第二部分
         * 并发扩容控制的核心
         */
        boolean advance = true;
        //标志整张表是否完成迁移
        boolean finishing = false; // to ensure sweep before committing nextTab
        //i 指向当前桶，bound 指向当前线程需要处理的桶结点的区间下限
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;

            //while 循环计算负责迁移的桶的区间
            // advance 为 true 表示可以进行下一个位置的迁移了
            // 简单理解结局：i 指向了 transferIndex，bound 指向了 transferIndex-stride
            while (advance) {
                int nextIndex, nextBound;
                //--i，向下一个位置迁移
                //移动后判断是否超出边界
                if (--i >= bound || finishing)
                    //暂时置为false，当前节点迁移结束才会变成true
                    advance = false;
                //transferIndex <= 0 说明已经没有需要迁移的桶了
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                //更新 transferIndex
                //为当前线程分配任务，处理的桶结点区间为（nextBound,nextIndex）
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    //(bound,i]
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
            //当前线程的所有任务完成
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                //如果整张表已经完成迁移
                if (finishing) {
                    //清除缓存
                    nextTable = null;
                    table = nextTab;
                    //修改sizeCtl
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                //设置sizeCtl，当前参与迁移的线程数量-1
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    //(resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2 
                    //相等说明只有当前线程在迁移数据，不等说明还有其它线程
                    //如果 (sc - 2) == resizeStamp(n) << RESIZE_STAMP_SHIFT，说明当前线程就是最后一个还在扩容的线程，
                    //还有线程在扩容
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    //没有线程在扩容，设置为true
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
            //待迁移桶为空，那么在此位置 CAS 添加 ForwardingNode 结点标识该桶已经被处理过了
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            //如果扫描到 ForwardingNode，说明此桶已经被处理过了，跳过即可
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            else {//进行实际的迁移节点操作
                //加锁
                //f为首节点
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        // 下面这一块和 Java7 中的 ConcurrentHashMap 迁移是差不多的，
                        // 需要将链表一分为二，
                        // 找到原链表中的 lastRun，然后 lastRun 及其之后的节点是一起进行迁移的
                        // lastRun 之前的节点需要进行克隆，然后分到两个链表中
                        if (fh >= 0) {//链表的迁移操作
                            //n只有一位是1
                            int runBit = fh & n;//这东西是什么？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？
                            Node<K,V> lastRun = f;
                            //整个 for 循环为了找到整个桶中最后连续的 fh & n 不变的结点？？？？？？？？？？？？？？？？？？？？
                            //整个for循环是为了找到最后一段连续节点 p.hash & n 都相同的节点
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                //是否与前驱节点相同
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            //lastRun连接着后面其余节点的连接
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            //？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？
                            //如果fh&n不变的链表的runbit都是0，则nextTab[i]内元素ln前逆序，ln及其之后顺序
                            //否则，nextTab[i+n]内元素全部相对原table逆序
                            //这是通过一个节点一个节点的往nextTab添加
                            //lastRun后面的节点，已经连接到 ln 或者 hn 上了
                            //拷贝 lastRun前面的节点
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    //前插法
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            //把两条链表整体迁移到nextTab中
                            // 其中的一个链表放在新数组的位置 i
                            setTabAt(nextTab, i, ln);
                            // 另一个链表放在新数组的位置 i+n
                            setTabAt(nextTab, i + n, hn);
                            // 将原数组该位置处设置为 fwd，代表该位置已经处理完毕，
                            // 其他线程一旦看到该位置的 hash 值为 MOVED，就不会进行迁移了
                            setTabAt(tab, i, fwd);
                            //迁移完成之后，设置为true，代表该位置已经迁移完毕，允许向下一个节点移动
                            advance = true;
                        }
                        //红黑树的复制算法
                        else if (f instanceof TreeBin) {
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                if ((h & n) == 0) {
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    else
                                        loTail.next = p;
                                    loTail = p;
                                    ++lc;
                                }
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
                             // 如果一分为二后，节点数少于 8，那么将红黑树转换回链表
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            //迁移完成之后，设置为true，允许向下一个节点移动
                            advance = true;
                        }
                    }
                }
            }
        }
    }
```

当我们成功的添加完成一个结点，最后是需要判断添加操作后是否会导致哈希表达到它的阈值，并针对不同情况决定是否需要进行扩容，还有 CAS 式更新哈希表实际存储的键值对数量。这些操作都封装在 addCount 这个方法中，当然 putVal 方法的最后必然会调用该方法进行处理。下面我们看看该方法的具体实现，该方法主要做两个事情。一是更新 baseCount，二是判断是否需要扩容。

更新数量的思想和 LongAdder 一样
http://zhoujiapeng.top/java/java-atomicOperationClass/
它的方法是在内部维护多个 Cell 变量，多个线程对 value 的 CAS 操作可以分散到多个 Cell 变量上，减少了争夺共享资源的并发量，最后,在获取 LongAdder 当前值时, 是把所有 Cell 变量的 value 值累加后再加上 base 返回。
```java
private final void addCount(long x, int check) {
        /**
         * 第一部分，更新 baseCount
         */
        CounterCell[] as; long b, s;
        if ((as = counterCells) != null ||
            //更性 baseCount 为 b+x
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            //如果更新失败
            
            CounterCell a; long v; int m;
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                //高并发下 CAS 失败会执行 fullAddCount 方法
                // See LongAdder version for explanation
                //这里借鉴了 LongAdder 的思想
                //http://zhoujiapeng.top/java/java-atomicOperationClass/
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
            s = sumCount();
        }
        /**
         * 判断是否需要扩容
         */
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                int rs = resizeStamp(n);
                if (sc < 0) {
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
                s = sumCount();
            }
        }
    }
```


```java
public V remove(Object key) {
        return replaceNode(key, null, null);
    }
final V replaceNode(Object key, V value, Object cv) {
        int hash = spread(key.hashCode());
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            //如果table为空，或者找不到key，直接返回
            if (tab == null || (n = tab.length) == 0 ||
                (f = tabAt(tab, i = (n - 1) & hash)) == null)
                break;
            //正在迁移，协助迁移
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {//找到对应的桶
                V oldVal = null;
                boolean validated = false;
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {//链表
                            validated = true;
                            //遍历链表
                            for (Node<K,V> e = f, pred = null;;) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    V ev = e.val;
                                    if (cv == null || cv == ev ||
                                        (ev != null && cv.equals(ev))) {
                                        oldVal = ev;
                                        if (value != null)
                                            e.val = value;
                                        else if (pred != null)
                                            pred.next = e.next;
                                        else
                                            setTabAt(tab, i, e.next);
                                    }
                                    break;
                                }
                                pred = e;
                                if ((e = e.next) == null)
                                    break;
                            }
                        }
                        else if (f instanceof TreeBin) {//树
                            validated = true;
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> r, p;
                            if ((r = t.root) != null &&
                                (p = r.findTreeNode(hash, key, null)) != null) {
                                V pv = p.val;
                                if (cv == null || cv == pv ||
                                    (pv != null && cv.equals(pv))) {
                                    oldVal = pv;
                                    if (value != null)
                                        p.val = value;
                                    else if (t.removeTreeNode(p))
                                        setTabAt(tab, i, untreeify(t.first));
                                }
                            }
                        }
                    }
                }
                if (validated) {
                    if (oldVal != null) {
                        if (value == null)
                            //更新数量
                            addCount(-1L, -1);
                        return oldVal;
                    }
                    break;
                }
            }
        }
        return null;
    }
```


```java
public int size() {
        long n = sumCount();
        return ((n < 0L) ? 0 :
                (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
                (int)n);
    }
//和LongAddr一样，sum=baseCount+CounterCell[]
final long sumCount() {
        CounterCell[] as = counterCells; CounterCell a;
        long sum = baseCount;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        return sum;
    }
```

get 方法可以根据指定的键，返回对应的键值对，由于是读操作，所以不涉及到并发问题。
get 方法从来都是最简单的，这里也不例外：

1、计算 hash 值 
2、根据 hash 值找到数组对应位置: (n - 1) & h 
3、根据该位置处结点性质进行相应查找

如果该位置为 null，那么直接返回 null 就可以了
如果该位置处的节点刚好就是我们需要的，返回该节点的值即可
如果该位置节点的 hash 值小于 0，说明正在扩容，或者是红黑树，后面我们再介绍 find 方法
如果以上 3 条都不满足，那就是链表，进行遍历比对即可

```java
public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        //计算hash
        int h = spread(key.hashCode());
        //定位到对应的桶
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            //头节点就是
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
             // 如果头结点的 hash 小于 0，说明 正在扩容，或者该位置是红黑树
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            //遍历链表
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
```

clear 方法将删除整张哈希表中所有的键值对，删除操作也是一个桶一个桶的进行删除。
```java
public void clear() {
        long delta = 0L; // negative number of deletions
        int i = 0;
        Node<K,V>[] tab = table;
        //遍历表中的所有桶
        while (tab != null && i < tab.length) {
            int fh;
            Node<K,V> f = tabAt(tab, i);
            if (f == null)
                ++i;
            //正在扩容
            else if ((fh = f.hash) == MOVED) {
                tab = helpTransfer(tab, f);
                i = 0; // restart
            }
            else {
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        //首节点
                        Node<K,V> p = (fh >= 0 ? f :
                                       (f instanceof TreeBin) ?
                                       ((TreeBin<K,V>)f).first : null);
                        while (p != null) {
                            --delta;
                            p = p.next;
                        }
                        setTabAt(tab, i++, null);
                    }
                }
            }
        }
        if (delta != 0L)
            addCount(delta, -1);
    }
```


操作HashMap的时候遇到其它线程正在扩容的，不是阻塞等待，而是主动加入帮助扩容

https://www.cnblogs.com/zaizhoumo/p/7709755.html
https://www.cnblogs.com/yangming1996/p/8031199.html
https://blog.csdn.net/sihai12345/article/details/79383766