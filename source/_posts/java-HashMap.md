---
title: 深入理解 HashMap(一)
subtitle: HashMap
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - java
  - 编程语言
  - jdk
categories:
  - java
date: 2019-10-17 14:30:05
---



# 深入理解 HashMap(一)
> 本文介绍的 HashMap 基于 JDK1.8

## 1. 概述
&emsp;HashMap能够存储给定的键值对，并且对于给定key的查询和插入都达到平均时间复杂度为O(1)。
&emsp;实现hash表的关键在于：

1. 对于给定的key，如何将其对应到内存中的一个对应位置，这通过hash算法做到。
2. 通过hash算法 hash(K) % N 来将关键字key映射数组对应hash表的位置上。
3. hash算法存在hash冲突，也即多个不同的K被映射到数组的同一个位置上。如何解决hash冲突？有三种方法。

- 分离链表法。即用链表来保存冲突的K。
- 开放定址法。当位置被占用时，通过一定的算法来试选其它位置。hash(i) = (hash(key) + d(i)) % N，i代表第i次试选。常用的有平方探测法，d(i) = i^2。
- 再散列。如果冲突，就再用hash函数再嵌套算一次，直到没有冲突。

&emsp;本文介绍的 JDK1.8 里面的 HashMap 采用方法类似于分离链表法。

## 2. HashMap in JDK1.8

### 2.1 储存结构
&emsp;HashMap采用数组形式来存储key-value对象, 以利用其优良的查找性能, 数组之所以查找迅速, 是因为可以根据索引(数组下标)直接定位到对应的存储桶(数组所存储对象的位置)，时间复杂度为O(1)。如果有多个元素被映射到同一个桶，则优先采用链表来组织具有相同key的元素。

&emsp;我们知道, 链表查找只能通过顺序查找来实现, 因此, 时间复杂度为o(n), 如果很不巧, 我们的key值被Hash算法映射到一个存储桶上, 将会导致存储桶上的链表长度越来越长, 此时, 数组查找退化成链表查找, 则时间复杂度由原来的o(1) 退化成 o(n)。

&emsp;为了解决这一问题, 在java8中,当链表长度超过 8 之后,将会自动将链表转换成红黑树,以实现 o(log n) 的时间复杂度, 从而提升查找性能。

![HashMap](https://github.com/JP6907/Pic/blob/master/java/HashMap.jpg?raw=true)
(图片来源网络)

### 2.2 扩容问题
&emsp;需要注意的是，采用链表本质上只是为了解决hash冲突问题，而不是为了增大容量，红黑树只是为了解决链表查询效率较低的问题。HashMap采用数组形式来存储key-value对象，最理想的情况是每一个桶只储存一个元素。另外。数组的大小是有限的, 在新建的时候就要指定, 如果加入的节点已经到了数组容量的上限, 已经没有位置能够存储key-value键值对了, 此时就需要扩容。
扩容不是等到数组全部满了再进行扩容，而是有一个阈值，这个阈值和负载因子有关，默认是0.75，官方的解释是：
> the default load factor (.75) offers a good tradeoff between time and space costs.

&emsp;确定了扩容的时机后，接下来的问题是每次扩容需要增加多少空间。
&emsp;我们知道, 数组的扩容是一个很耗费CPU资源的动作, 需要将原数组的内容复制到新数组中去, 因此频繁的扩容必然会导致性能降低, 所以不可能数组满了之后, 每多加一个node, 我们就扩容一次。但是, 一次扩容太大, 导致大量的存储空间用不完, 势必又造成很大的浪费, 因此, 必须根据实际情况设定一个合理的扩容大小。
&emsp;在HashMap的实现中, 每次扩容我们都会将新数组的大小设为原数组大小的两倍。

## 3. 源码分析
&emsp;接下来我们从源码的角度来分析HashMap。
### 3.1 Node

&emsp;table是Hash表数组，数组的元素类型是Node，Node里面包含了一个next字段，这表明HashMap采用的是分离链表的方法实现。
```java
//hash表数组
transient Node<K,V>[] table;
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }
    ...
}
```
&emsp;Node 不仅储存了key、value，还有一个hash字段，为key的hash，既然已经储存了key，为何还要多此一举储存key的hash？
仔细一想不难明白，HashMap能够存储任意对象，对象的hash值是由hashCode方法得到，这个方法由所属对象自己定义，里面可能有费时的操作，而hash值在Hash表内部实现会多次用到，因此这里将它保存起来，是一种优化的手段。

### 3.2 TreeNode
&emsp;TreeNode是红黑树节点，里面有一个red属性。HashMap会在链表过长的时候，将其重构成红黑树。

```java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;
        TreeNode(int hash, K key, V val, Node<K,V> next) {
            super(hash, key, val, next);
        }
}
```

### 3.3 属性字段
```java
//初始默认大小16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
static final int MAXIMUM_CAPACITY = 1 << 30;
// 默认负载因子 0.75
static final float DEFAULT_LOAD_FACTOR = 0.75f;
static final int TREEIFY_THRESHOLD = 8;
static final int UNTREEIFY_THRESHOLD = 6;
static final int MIN_TREEIFY_CAPACITY = 64;

transient Node<K,V>[] table;
transient Set<Map.Entry<K,V>> entrySet;
transient int size;
transient int modCount;
int threshold;
final float loadFactor;
```
&emsp;最重要的是table、size、loadFactor这三个字段：
1. table可以看出是个节点数组，也即hash表中用于映射key的数组。由于链表是递归数据结构，这里数组保存的是链表的头节点。
2. size，hash表中元素个数。
3. loadFactor，装填因子，控制HashMap扩容的时机。

&emsp;对于 threshold 字段：
1. 如果table还没有被分配，threshold 为**初始的空间大小**。如果是0，则是默认大小，DEFAULT_INITIAL_CAPACITY。
2. 如果table已经分配了，这个值为**扩容阈值**，也就是table.length * loadFactor。

### 3.4 构造函数
```java
// 指定初始大小和负载因子
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}

public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
// 没有指定时, 使用默认值
// 即默认初始大小16, 默认负载因子 0.75
public HashMap() {
   this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}

// 利用已经存在的map创建HashMap
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```
如果我们在构造函数指定了 initialCapacity，即初始容量，则会调用 tableSizeFor 函数来设置阈值：
```java
this.threshold = tableSizeFor(initialCapacity);

/**
 * Returns a power of two size for the given target capacity.
 */
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```
&emsp;通过注释我们可以知道 tableSizeFor 函数是将给定的数对齐到2的幂，也就是找到大于等于initialCapacity的最小的2的幂。关于这个算法的具体介绍可以参看[这一篇博客](https://blog.csdn.net/fan2012huan/article/details/51097331)。
&emsp;由此可以知道，hash桶数组的初始大小一定是2的幂，后面扩容的时候总是将将容量扩大到原来的两倍，因此hash桶数组大小总是为2的幂。   

&emsp;另外，这里也用到了类似前面ArrayList的“**延迟分配**”的思路，一开始table是null，只有在第一次插入数据时才会真正分配空间。这样，由于实际场景中会出现大量空表，而且很可能一直都不添加元素，这样“延迟分配”的优化技巧能够节约内存空间。这里就体现出threshold的含义了，hash桶数组的空间未分配时它保存的是table初始的大小。


### 3.5 resize 扩容
&emsp;上面我们说到，构造函数中并不会初始化 table 变量, table 变量是在 resize 过程中初始化的。
&emsp;resize 函数的作用有两个：
- 初始化 table
- 在table大小超过threshold之后进行扩容

```java
final Node<K,V>[] resize() {
        //一：确定新table的容量和阈值
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        //原table已经有值
        if (oldCap > 0) {
            //如果超出最大容量，则不再扩容，这里是2的30次方
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            //容量大于16小于最大值时容量扩大2倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        //到此，newCap是新的hash桶数组大小，newThr是新的扩容阈值
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        //二：将旧table的数据拷贝到新table
        //分配新的table
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        //下面把oldTab的数据迁移到新的table来
        if (oldTab != null) {
            //遍历旧table的每个元素
            for (int j = 0; j < oldCap; ++j) {
                //e代表该位置上的第一个元素
                Node<K,V> e;
                //如果该位置上存在数据
                if ((e = oldTab[j]) != null) {
                    //清除引用
                    oldTab[j] = null;
                    //说明只有一个元素，直接拷贝过来就好
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    //红黑树的情况
                    else if (e instanceof TreeNode)
                        //将e为根节点的红黑数切分两条子树，重新分配到newTab中
                        //如果拆分出来的子树太小了，就会重新将其重构回链表。
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    //链表的情况
                    else { // preserve order
                        //将原链表拆分成两条子链表
                        //下面的大致逻辑为：
                        //构建两条链表 loHead 和 hiHead
                        //以此取原链表上的每个节点 e
                        //判断 (e.hash & oldCap)
                        //   如果为 0，将e插到 loHead
                        //   如果为 1，将e插到 hiHead
                        //最后将 loHead 放到 newTab[j]
                        //   将 hiHead 放到 newTab[j + oldCap]
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {//遍历链表
                            next = e.next;
                            //插入到loHead
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            //插入到hiHead
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        //将 loHead 放到 newTab[j]
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        //将 hiHead 放到 newTab[j + oldCap]
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```
&emsp;resize 函数有点长，主要分为两个步骤：
1. 确定新table的容量和阈值
2. 将旧table的数据拷贝到新table

&emsp;第一部分比较简单，我们主要看一下第二部分，第二部分的思路为：
遍历旧table的每个元素e:
- 该位置只有一个元素，直接拷贝该元素到新table相同位置
- 该位置上是红黑树
    - 将e为根节点的红黑数切分两条子树，重新分配到newTab中，如果拆分出来的子树太小了，就会重新将其重构回链表。
- 该位置上是链表
    - 将原链表拆分成两条子链表

&emsp;下面主要看一下链表拆分的情况：
```java
Node<K,V> loHead = null, loTail = null;
Node<K,V> hiHead = null, hiTail = null;
Node<K,V> next;
do {//遍历链表
    next = e.next;
    //插入到loHead
    if ((e.hash & oldCap) == 0) {
        if (loTail == null)
            loHead = e;
        else
            loTail.next = e;
        loTail = e;
    }
    //插入到hiHead
    else {
        if (hiTail == null)
            hiHead = e;
        else
            hiTail.next = e;
        hiTail = e;
    }
} while ((e = next) != null);
//将 loHead 放到 newTab[j]
if (loTail != null) {
        loTail.next = null;
    newTab[j] = loHead;
}
    //将 hiHead 放到 newTab[j + oldCap]
if (hiTail != null) {
    hiTail.next = null;
    newTab[j + oldCap] = hiHead;
}
```
&emsp;这段代码设计得很是精妙，我们一段一段来看：
```java
Node<K,V> loHead = null, loTail = null;
Node<K,V> hiHead = null, hiTail = null;
```
&emsp;这一段定义了4个Node，从变量命名上看，我们初步猜测，这里定义了两条链表 lo链表 和 hi链表, loHead 和 loTail 分别指向 lo链表的头节点和尾节点, hiHead 和 hiTail 分别指向 hi链表的头节点和尾节点。
```java
do {//遍历链表
    next = e.next;
    //插入到loHead
    if ((e.hash & oldCap) == 0) {
        if (loTail == null)
            loHead = e;
        else
            loTail.next = e;
        loTail = e;
    }
    //插入到hiHead
    else {
        if (hiTail == null)
            hiHead = e;
        else
            hiTail.next = e;
        hiTail = e;
    }
} while ((e = next) != null);
```
&emsp;这一段的主要框架为：
```java
do {
    next = e.next;
    ...
} while ((e = next) != null);
```
&emsp;就是遍历链表上的节点，对于每个节点，有两种情况，如果 (e.hash & oldCap) == 0，则执行：
```java
if ((e.hash & oldCap) == 0) {
        if (loTail == null)
            loHead = e;
        else
            loTail.next = e;
        loTail = e;
    }
```
&emsp;这里其实就是使用尾插法将e节点插入到 lo 链表中，如果上面的条件不成立，则将e节点插入到 hi 链表。
我们再看一下最后一段：
```java
//将 loHead 放到 newTab[j]
if (loTail != null) {
        loTail.next = null;
    newTab[j] = loHead;
}
    //将 hiHead 放到 newTab[j + oldCap]
if (hiTail != null) {
    hiTail.next = null;
    newTab[j + oldCap] = hiHead;
}
```
&emsp;这一段做的事就是：
- 如果lo链表非空, 我们就把整个lo链表放到新table的j位置上 
- 如果hi链表非空, 我们就把整个hi链表放到新table的j+oldCap位置上

&emsp;综上我们知道, 这段代码的意义就是将原来的链表拆分成两个链表, 并将这两个链表分别放到新的table的 j 位置和 j+oldCap 上, j位置就是原链表在原table中的位置, 拆分的标准就是:
(e.hash & oldCap) == 0

&emsp;整个过程用一个图表示：
![HashMap-resize-LinkedList](https://github.com/JP6907/Pic/blob/master/java/HashMap-resize-LinkedList.png?raw=true)
(图片来源网络)


&emsp;接下来我们再看一下 (e.hash & oldCap) == 0 这个条件
&emsp;首先我们要明确三点:
- oldCap 一定是2的整数次幂, 这里假设是 2^m；
- newCap 是 oldCap 的两倍, 则会是 2^(m+1)；
- hash 对数组大小取模 (n - 1) & hash 其实就是取 hash 的低 m 位；

例如: 
我们假设 oldCap = 16, 即 2^4, 
16 - 1 = 15, 二进制表示为 0000 0000 0000 0000 0000 0000 0000 1111 
可见除了低4位, 其他位置都是0（简洁起见，高位的0后面就不写了）, 则 (16-1) & hash 自然就是取hash值的低4位,我们假设它为 abcd.

以此类推, 当我们将oldCap扩大两倍后, 新的index的位置就变成了 (32-1) & hash, 其实就是取 hash值的低5位. 那么对于同一个Node, 低5位的值无外乎下面两种情况:
```
0abcd
1abcd
```
其中, 0abcd与原来的index值一致, 而1abcd = 0abcd + 10000 = 0abcd + oldCap

故虽然数组大小扩大了一倍，但是同一个key在新旧table中对应的index却存在一定联系： 要么一致，要么相差一个 oldCap。

而新旧index是否一致就体现在hash值的第4位(我们把最低为称作第0位), 怎么拿到这一位的值呢, 只要:
```
hash & 0000 0000 0000 0000 0000 0000 0001 0000
```
即
```
hash & oldCap
```
故得出结论:
- 如果 (e.hash & oldCap) == 0 则该节点在新表的下标位置与旧表一致都为 j 
- 如果 (e.hash & oldCap) == 1 则该节点在新表的下标位置 j + oldCap

根据这个条件, 我们将原位置的链表拆分成两个链表, 然后一次性将整个链表放到新的Table对应的位置上。

&emsp;总结一下 resize：
1. resize发生在table初始化, 或者table中的节点数超过threshold值的时候, threshold的值一般为负载因子乘以容量大小.
2. 每次扩容都会新建一个table, 新建的table的大小为原大小的2倍.
3. 扩容时,会将原table中的节点re-hash到新的table中, 但节点在新旧table中的位置存在一定联系: 要么下标相同, 要么相差一个oldCap(原table的大小).



## 4. 未完
> 由于篇幅原因，更多 HashMap 的分析请阅读下一篇文章[《深入理解 HashMap(二)》](http://zhoujiapeng.top/java/java-HashMap2)。


## 5. 总结
&emsp;这里先做一个总结.
&emsp;我们设计HashMap的初心是什么呢, 是找到一种方法, 可以存储一组键值对的集合, 并实现快速的查找：
==> 为了实现快速查找, 我们选择了数组而不是链表. 以利用数组的索引实现o(1)复杂度的查找效率.
==> 为了利用索引查找, 我们引入Hash算法, 将 key 映射成数组下标: key -> Index
==> 引入Hash算法又导致了Hash冲突
==> 为了解决Hash冲突, 我们采用链地址法, 在冲突位置转为使用链表存储.
==> 链表存储过多的节点又导致了在链表上节点的查找性能的恶化
==> 为了优化查找性能, 我们在链表长度超过8之后转而将链表转变成红黑树, 以将 o(n)复杂度的查找效率提升至o(log n)

&emsp;可见, 每一次结构的调整, 都是始终围绕我们的初心:**实现快速的查找**。

&nbsp;
>参考：
https://segmentfault.com/a/1190000015796727
https://segmentfault.com/a/1190000015631344
https://segmentfault.com/a/1190000015806050
https://segmentfault.com/a/1190000015812438
