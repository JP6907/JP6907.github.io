---
title: 深入理解 HashMap(二)
catalog: true
subtitle: HashMap
header-img: /img/article_header/article_header.png
tags:
  - java
  - 编程语言
  - jdk
categories:
  - java
---

# 深入理解 HashMap(二)
接着上一篇文章[《深入理解 HashMap(一)》](http://zhoujiapeng.top/java/java-HashMap)，我们继续分析 HashMap。

本文首先介绍 HashMap 中的 hash 算法和索引计算方法，它们都经过了一定的优化，在 get 和 put 函数中均会用到。接着会详细介绍 get 和 put 的过程，理解了这两个过程，就基本上掌握了 HashMap 的整体流程。

## Hash 算法
普通的 Hash 表可能会使用
```java
key.hashCode() 
```
来计算key的hash，但是 HashMap 有自己的散列算法：
```java
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```
我们知道, int类型是32位的, h ^ h >>> 16 其实就是将hashCode的高16位和低16位进行异或, 这充分利用了高半位和低半位的信息, 对低位进行了扰动, 目的是为了使该hashCode映射成数组下标时可以更均匀, 详细的解释可以参考[这里](https://www.zhihu.com/question/20733617/answer/111577937)。

另外, 从这个函数中, 我们还可以得到一个意外收获:
HashMap中key值可以为null, 且null值一定存储在数组的第一个位置.



## 优化的索引计算
对于一般的 Hash 表，可能会这样来将 key 映射成 index：
```java
h % n
```
但是在 HashMap 中会通过这样的方式来计算索引：
```java
(n - 1) & hash
```
这个位运算，实际上是对取余运算的优化。由于hash桶数组的大小一定是2的幂次方，因此能够这样优化，即：
```java
h % n = (n - 1) & hash
```

思路是这样的，bi是b二进制第i位的值：

b % 2<sup>i</sup> = (2<sup>N</sup>b<sub>N</sub> + 2<sup>N-1</sup>b<sub>N-1</sub>+ ... + 2<sup>i</sup>b<sub>i</sub> + ... 2<sup>0</sup>b<sub>0</sub>) % 2<sup>i</sup>

设x >= i，则一定有 2<sup>x</sup>b<sub>x</sub> % 2i = 0

所以，上面的式子展开后就是：
b % 2<sup>i</sup> = 2<sup>i-1</sup>b<sub>i-1</sub> + 2<sup>i-2</sup>b<sub>i-1</sub> + ... 2<sup>0</sup>b<sub>0</sub>
结果就是只保留低于第i位的所有数据。

反映到二进制上来说，以8位二进制举个例子：

对于 h % n：
显然2的幂次方N的二进制位是只有一个1的。8的二进制为000001000，1在第3位。
任何一个数B对N求余，反映二进制上，就是高于等于第3位的置0，低于的保留。如10111010 % 00001000 = 00000010
对于 n-1：
00001000 - 1 = 00000111，这样减一之后，需要保留的对应位为全是1，需要置0的对应位全都是0。把它与B作与运算，就是只保留低于第3位的数据，就能得到结果。



由于取模运算是十分耗时的，而索引计算在 HashMap 中是非常常见的操作，将取模操作转化成&运算能够很大程度上提高效率。

## get函数
```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```
get 函数先是调用 hash 对 key 做一次 hash 运算之后再调用 getNode 函数。
hash 运算在上面已经分析过了，我们看一下 getNode 函数：
```java
final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        //如果table不为空，且对应索引上存在数据
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            //key相等，在该索引位置上的第一个节点就是要找的节点
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            //需要往后找
            if ((e = first.next) != null) {
                //如果是红黑树
                if (first instanceof TreeNode)
                    //在红黑树上查找
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {//链表
                    //遍历链表
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```
getNode 的思路为：
- table 非空且 key 在 table 中对应位置上存在数据
    - 否：返回 null
    - 是：
        - 在该索引位置上的第一个节点就是要找的节点，即 key 相等，直接返回第一个节点
        - 否则，判断该位置上的结构
            - 红黑树：在红黑树上查找
            - 链表：遍历链表查找


## put函数


```java
public V put(K key, V value) {
        //提前计算好key的hash
        return putVal(hash(key), key, value, false, true);
    }
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        //onlyIfAbsent含义是如果那个位置已经有值了，是否替换
        //evict
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        //1.如果table为空
        if ((tab = table) == null || (n = tab.length) == 0)
            //resize的作用为扩容或初始化这里为初始化
            n = (tab = resize()).length;
        //2.i = (n - 1) & hash]计算索引，判断该位置上是否有数据
        if ((p = tab[i = (n - 1) & hash]) == null)
            //2.1 如果没有则直接添加新节点
            tab[i] = newNode(hash, key, value, null);
        else {//2.2 该位置上已经有数据
            //e会储存新插入的节点，或者如果key已经存在，则储存已经存在的节点
            Node<K,V> e; K k;
            //此时p为table表中该索引位置上的第一个数据
            //2.2.1 判断第一个位置上的key是否相同
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            //2.2.2 如果当前链是红黑树
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {//2.2.3 链表
                //遍历链表，binCount用来统计链表长度
                for (int binCount = 0; ; ++binCount) {
                    //这里已经将p.next赋值给了e，后面使用e和p来遍历链表
                    //情况1：最后一个节点
                    if ((e = p.next) == null) {
                        //插入新的节点
                        p.next = newNode(hash, key, value, null);
                        //如果链表长度大于8，则转化为红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    //情况2：不是最后一个节点，存在key相同的节点
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    //链表的下一个节点，前面已经将p.next赋值给了e
                    p = e;
                }
            }
            //情况2的处理，情况1时e==null
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                //如果允许覆盖
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        //储存的数据快要达到饱和时扩容
        if (++size > threshold)
            //扩容
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```


思路大概是这样的逻辑：

- 1.判断table是否分配，如果没有就先分配空间，和前面提到的“延时分配”对应起来。
- 2.同样，根据hash值定位hash桶数组的位置。然后：
    - 2.2 该位置为null。直接创建一个节点插入。
    - 2.3 该位置为平衡树。调用TreeNode的一个方法完成插入，具体逻辑在这个方法里。
    - 2.4 该位置为链表。遍历链表，进行插入。会出现两种情况：
        - 遍历到链表尾，说明这个key不存在，应该直接在链表尾插入。但是这导致链表增长，需要触发链表重构成红黑树的判断逻辑。
        - 在链表上找到一个key相同的节点，根据 onlyIfAbsent 参数决定是否要使用新数据覆盖旧数据
- 3. 判断插入数据后 size 是否会超过阈值，如果超过，执行扩容操作。


&nbsp;
> 参考：
https://segmentfault.com/a/1190000015631344
https://segmentfault.com/a/1190000015798586