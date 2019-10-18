---
title: java-ConcurrentHashMap
catalog: true
subtitle:
header-img:
tags:
---



````java
private transient volatile int sizeCtl;
```
这是一个重要的属性，无论是初始化哈希表，还是扩容 rehash 的过程，都是需要依赖这个关键属性的。该属性有以下几种取值：
- 0：默认值
- -1：代表哈希表正在进行初始化
- 大于0：相当于 HashMap 中的 threshold，表示阈值
- 小于-1：代表有多个线程正在进行扩容

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
                        if (fh >= 0) { // 头结点的 hash 值大于 0，说明是链表？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？
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
            while (advance) {
                int nextIndex, nextBound;
                if (--i >= bound || finishing)
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
                    nextTable = null;
                    table = nextTab;
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
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
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        if (fh >= 0) {//链表的迁移操作
                            int runBit = fh & n;//这东西是什么？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？
                            Node<K,V> lastRun = f;
                            //整个 for 循环为了找到整个桶中最后连续的 fh & n 不变的结点？？？？？？？？？？？？？？？？？？？？
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
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
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            //把两条链表整体迁移到nextTab中
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            //将原桶标识位已经处理
                            setTabAt(tab, i, fwd);
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
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
        }
    }
```


```java
private final void addCount(long x, int check) {
        CounterCell[] as; long b, s;
        if ((as = counterCells) != null ||
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            CounterCell a; long v; int m;
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
            s = sumCount();
        }
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


https://www.cnblogs.com/zaizhoumo/p/7709755.html
https://www.cnblogs.com/yangming1996/p/8031199.html
https://blog.csdn.net/sihai12345/article/details/79383766