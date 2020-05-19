---
title: Java 并发容器 之 ConcurrentHashMap(二)
subtitle: ConcurrentHashMap
catalog: true
comments: true
indexing: true
header-img: ../../../../img/article_header/article_header.png
top: false
tocnum: true
tags:
  - java
  - 编程语言
  - 并发
categories:
  - java
date: 2020-05-14 19:36:01
---



# ConcurrentHashMap
&emsp;接着上一篇文章《Java 并发容器 之 ConcurrentHashMap(一)》，我们继续分析 ConcurrentHashMap。

## 8. transfer()
&emsp;真正执行数据迁移的是 transfer 函数，transfer 函数非常的长：
```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        /**
         * 第一部分
         * 计算单个线程能处理的最少桶结点个数和一些属性的初始化操作。
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

            /**
             * 2.1 计算当前线程当前允许操作的桶索引区间
             */
            //while 循环计算负责迁移的桶的区间
            // advance 为 true 表示可以进行下一个位置的迁移了
            // 简单理解结局：i 指向了 transferIndex，bound 指向了 transferIndex-stride
            while (advance) {
                int nextIndex, nextBound;
                /**
                 * 2.1.1 i还在区间内
                 */
                //--i，向下一个位置迁移
                //移动后判断是否超出边界
                if (--i >= bound || finishing)
                    //还在区间内，暂时置为false，当前节点迁移结束才会变成true
                    advance = false;
                //transferIndex <= 0 说明已经没有需要迁移的桶了
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                /**
                 * 2.1.2 更新操作区间
                 */
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
            /**
             * 2.2 当前线程的所有任务完成
             */
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                /**
                 * 2.2.1 如果整张表已经完成迁移，更新 table 和 sizeCtl
                 */
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
                    /**
                     * 2.2.2 其它线程还在扩容，只更新 sizeCtl 标志
                     */
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
            /**
             * 第三部分：桶处理过程
             */
            /**
             * 3.1 待迁移桶为空，那么在此位置 CAS 添加 ForwardingNode 结点标识该桶已经被处理过了
             */
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            /**
             * 3.2 如果扫描到 ForwardingNode，说明此桶已经被处理过了，跳过即可
             */
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            /**
             * 3.3 进行实际的迁移节点操作
             */
            else {
                //加锁
                //f为首节点
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        // 将原链表一分为二，插入到新的table
                        // 找到原链表中的 lastRun，然后 lastRun 及其之后的节点是一起进行迁移的
                        // lastRun 之前的节点需要进行克隆，然后分到两个链表中
                        if (fh >= 0) {//链表的迁移操作
                            //n=2^k 只有一位是1
                            int runBit = fh & n;//0或1，用来帮助将原链表拆分成两条子链表
                            Node<K,V> lastRun = f;
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
                            //将lastRun前面的节点插入到子链表中
                            //lastRun后面的节点，已经连接到 ln 或者 hn 上了
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

&emsp;transfer 的整体过程为：
- 1.计算单个线程能处理的最少桶结点个数和一些属性的初始化操作。
- 2.并发扩容控制的核心。
    - 2.1 计算当前线程当前允许操作的桶索引区间
        - 2.1.1 i还在区间内，索引向下一个位置移动
        - 2.1.2 超出区间，更新操作区间
    - 2.2 如果当前线程的所有任务完成
        - 2.2.1 如果整张表已经完成迁移，更新 table 和 sizeCtl
        - 2.2.2 其它线程还在扩容，只更新 sizeCtl 标志
- 3.桶处理过程
    - 3.1 待迁移桶为空，那么在此位置 CAS 添加 ForwardingNode 结点标识该桶已经被处理过了
    - 3.2 如果扫描到 ForwardingNode，说明此桶已经被处理过了，跳过即可
    - 3.3 加锁，进行实际的迁移节点操作，分为链表和红黑树两种情况，迁移结束后设置该位置为 ForwardingNode，标识该桶已经被处理

&emsp;上面的代码比较长，我们分段来分析。
&emsp;第一部分是计算单个线程能处理的最少桶结点个数(步长stribe)和一些属性的初始化操作。
```java
/**
         * 第一部分
         * 计算单个线程能处理的最少桶结点个数和一些属性的初始化操作。
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
```
&emsp;这里需要注意的是，transferIndex 会在 nextTab 初始化的时候被设置 transferIndex = n，指向旧 table 的最后一个位置。transferIndex 是一个全局的变量，桶迁移过程是从最后一个位置往前遍历的，transferIndex 指向最新完成迁移(或者已经被分配给某个线程，将被处理)的桶位置。

&emsp;下面看一下第二部分的代码，首先看2.1，计算操作区间：
```java
        transferIndex = n;
        ......
/**
             * 2.1 计算当前线程当前允许操作的桶索引区间
             */
            //while 循环计算负责迁移的桶的区间
            // advance 为 true 表示可以进行下一个位置的迁移了
            // 简单理解结局：i 指向了 transferIndex，bound 指向了 transferIndex-stride
            while (advance) {
                int nextIndex, nextBound;
                /**
                 * 2.1.1 i还在区间内
                 */
                //--i，向下一个位置迁移
                //移动后判断是否超出边界
                if (--i >= bound || finishing)
                    //还在区间内，暂时置为false，当前节点迁移结束才会变成true
                    advance = false;
                //transferIndex <= 0 说明已经没有需要迁移的桶了
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                /**
                 * 2.1.2 更新操作区间
                 */
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
```

&emsp;在进行实际的数据迁移之前，要先计算当前线程允许操作的桶索引区间，单个线程允许处理的最少table桶首节点个数，不能小于 16，这个数量称为 stribe，步长。i 作为操作的桶索引，每一次允许操作的桶索引区间为（nextBound,nextIndex），其中 nextBound = nextIndex - stride。nextIndex 一开始等于 transferIndex，每个线程领取一个长度为 stribe 的区间，然后将 transferIndex 设置为 transferIndex - stribe：
```java
//这里设置了nextIndex
else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
//这里使用 UnSafe 设置 全局 transferIndex 为 transferIndex - stribe
else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    //这里确定了操作区间为（bound,nextIndexs）
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
```
&emsp;举例来说，原table的长度 n = 128，stribe = 16
&emsp;线程1首先进入扩容，设置 transferIndex = 128，transferIndex > 0，则领取操作区间 (112,128],然后设置 transferIndex 为 112
&emsp;接着线程2进入协助扩容，此时 transferIndex = 112 > 0,领取操作区间(96,112],然后设置 transferIndex 为 96
&emsp;接着线程1完成(112,128]区间的数据迁移，再次尝试领取操作区间，此时 transferIndex = 96 > 0 ，领取区间 (80,96]，然后设置 transferIndex 为 80
&emsp;....
&emsp;直至 transferIndex <= 0
&emsp;通过这种方式，就能够实现多个线程同时迁移不同的区间，做到并行化。

&emsp;接下来看代码2.2，当线程完成当前区间任务，且领取不到新的区间时，此时有两种情况：
- 1.所有线程已经完成了所有区间的数据迁移
- 2.还有其它线程正在迁移某个区间

&emsp;是否还存在其它线程正在迁移是通过 sizeCtl 来判断的
&emsp;sizeCtl = resizeStamp(n) << RESIZE_STAMP_SHIFT + 2 表示只有一个线程在参与迁移，因此如果 (sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT 则说明还存在其它线程
```java
 /**
             * 2.2 当前线程的所有任务完成
             */
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                /**
                 * 2.2.1 如果整张表已经完成迁移，更新 table 和 sizeCtl
                 */
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
                    /**
                     * 2.2.2 其它线程还在扩容，只更新 sizeCtl 标志
                     */
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
```

&emsp;接下来看一下第三部分具体的桶处理过程，3.1和3.2比较简单，我们看一下3.3实际的迁移节点操作：
```java
/**
             * 3.3 进行实际的迁移节点操作
             */
            else {
                //加锁
                //f为首节点
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        // 将原链表一分为二，插入到新的table
                        // 找到原链表中的 lastRun，然后 lastRun 及其之后的节点是一起进行迁移的
                        // lastRun 之前的节点需要进行克隆，然后分到两个链表中
                        if (fh >= 0) {//链表的迁移操作
                            //n=2^k 只有一位是1
                            int runBit = fh & n;//0或1，用来帮助将原链表拆分成两条子链表
                            Node<K,V> lastRun = f;
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
                            //将lastRun前面的节点插入到子链表中
                            //lastRun后面的节点，已经连接到 ln 或者 hn 上了
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
                            ...
                    }
                }
            }
        }
```
&emsp;看一下链表的情况，跟 HashMap 的扩容类似，这里也是将原链表拆分成两条子链表然后插入到新的链表去，只是这里的实际操作有点不一样
&emsp;runBit = fh & n，n=2^k 只有一位是1，做与操作，所以 runBit 为0或1，fh 是一个 hash 值，所以 runBit 的值应该是一个随机值0或1
接下遍历整个链表:
```java
    Node<K,V> lastRun = f;
        //整个for循环是为了找到最后一段连续节点 p.hash & n 都相同的节点
        for (Node<K,V> p = f.next; p != null; p = p.next) {
        int b = p.hash & n;
            //是否与前驱节点相同
        if (b != runBit) {
                runBit = b;
                lastRun = p;
            }
        }
```
&emsp;这里整个for循环是为了找到最后一段连续节点 p.hash & n 都相同的节点
&emsp;举例来说，runBit = 1，链表上的每个节点依次取 p.hash & n 之后为：0101011110001110101111，最后一段连续的是最后的 1111 
&emsp;最终结果是 runBit = 1，lastRun 值向倒数第4个1的位置，也就是说，lastRun后面的所有节点的 p.hash & n 都为1.
&emsp;具体为什么要这么做需要看到后面才能理解。
&emsp;紧接着：
```java
    //lastRun连接着后面其余节点的连接
        if (runBit == 0) {
            ln = lastRun;
            hn = null;
        }
        else {
            hn = lastRun;
            ln = null;
        }
```
&emsp;这时最后四个节点被连接到了 hn 链表上去
```java
//将lastRun前面的节点插入到子链表中
        //lastRun后面的节点，已经连接到 ln 或者 hn 上了
        for (Node<K,V> p = f; p != lastRun; p = p.next) {
            int ph = p.hash; K pk = p.key; V pv = p.val;
            if ((ph & n) == 0)
                //前插法
                ln = new Node<K,V>(ph, pk, pv, ln);
            else
                hn = new Node<K,V>(ph, pk, pv, hn);
        }
```
&emsp;这里从头遍历 lastRun 前面的节点，如果 (ph & n) == 0，则插入到 ln 链表，否则插入到 hn 链表，这里插入都是使用头插法。
&emsp;到这里我们可以知道，原链表 lastRun 前面的节点中 (ph & n) == 0 的节点都被分配到 ln，不等于0的节点都被分配到 hn，而 lastRun 后面的节点已经被提前加到 hn 上面了，并且它们都是  (ph & n) == 1，不需要再进行多余的判断了。所以前面先寻找 lastRun，将 lastRun 后面的节点一次性加入到 hn，能够省去后面很多的判断和链表操作。
&emsp;不得不感叹，源码作者的设计真的是非常的细致！

&emsp;最后，将 ln 和 hn 放到新 table 的 i 位置和 i+n 位置，然后把原 table 的该位置设置为 ForwardingNode，表示该位置已经完成数据迁移。
```java
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
```



## 9. addCount() && size

&emsp;在 putVal 函数中，当我们成功的添加完成一个结点，最后是需要判断添加操作后是否会导致哈希表达到它的阈值，并针对不同情况决定是否需要进行扩容，还有 CAS 式更新哈希表实际存储的键值对数量。这些操作都封装在 addCount 这个方法中，下面我们看看该方法的具体实现，该方法主要做两个事情。一是更新数量，二是判断是否需要扩容，如果如果是增加节点(check>=0)，则有可能超过阈值，就需要扩容。关于扩容部分不再赘述。这里更新数量的思想和 [LongAdder](http://zhoujiapeng.top/java/java-atomicOperationClass/) 一样。它的方法是在内部维护多个 Cell 变量，多个线程对 value 的 CAS 操作可以分散到多个 Cell 变量上，减少了争夺共享资源的并发量，最后,在获取数量当前值时, 是把所有 Cell 变量的 value 值累加后再加上 base 返回。
```java
//x为需要增加的数量
//check>=0 增加节点，，check为更新后的数量
//check<0，删除节点
private final void addCount(long x, int check) {
        /**
         * 第一部分，更新数量
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
        //说明是增加节点，则有可能使得节点数量超过阈值
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
            //s = sumCount() 超过阈值 sizeCtl，需要扩容
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                //获取扩容检验标识
                int rs = resizeStamp(n);
                if (sc < 0) { //如果正在初始化或扩容
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    //协助扩容
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                //首先发起扩容
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
                s = sumCount();
            }
        }
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

## 10. remove()
&emsp;remove一个节点的思路为：
- 1.如果table为空，或者找不到key，直接返回
- 2.table 正在扩容，协助迁移
- 3.找到对应的桶，锁住头节点，找到对应的节点进行移除操作
- 4.调用 addCount 减少数量
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

## 11. clear()
&emsp;clear 方法将删除整张哈希表中所有的键值对，同样，如果发现table正在扩容，则需要先协助扩容，再进行hash表的清除操作，清除过程是一个桶一个桶删除，删除每一个桶之前会获取头节点的监视器锁，然后再删除hash表对该桶的引用，该桶就能够被自动回收了。
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
                //协助扩容
                tab = helpTransfer(tab, f);
                i = 0; // restart
            }
            else {
                //获取头节点的监视器锁
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
                        //删除对桶的引用
                        setTabAt(tab, i++, null);
                    }
                }
            }
        }
        if (delta != 0L)
            addCount(delta, -1);
    }
```


## 12. get()
&emsp;get 方法可以根据指定的键，返回对应的键值对，由于是读操作，可以不需要加锁。
get 方法的思路为：
- 1.计算 hash 值 
- 2.根据 hash 值找到数组对应位置的桶: (n - 1) & h 
- 3.在该桶上找到对应的节点
- 4.如果找不到对应节点，返回null


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

## 13. 总结

&emsp;操作HashMap的时候遇到其它线程正在扩容的，不是阻塞等待，而是主动加入帮助扩容
说实话，Java8 ConcurrentHashMap 源码真心不简单，最难的在于扩容，数据迁移操作不容易看懂。
切分链表 runBit  lastRun

&nbsp;
>参考：
https://www.cnblogs.com/zaizhoumo/p/7709755.html
https://www.cnblogs.com/yangming1996/p/8031199.html
https://blog.csdn.net/sihai12345/article/details/79383766