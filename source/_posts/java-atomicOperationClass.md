---
title: Java 并发编程 之 原子操作类
subtitle: 原子操作类
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - java
  - 编程语言
  - 并发
categories:
  - java
date: 2019-09-30 10:29:30
---



# Java 并发包中的原子操作类
## 1. 原子变量操作类
&emsp;JUC 并发包中包含有 Atomiclnteger、AtomicLong 和 AtomicBoolean 等原子性操作类,本文以 AtomicLong 类为例，AtomicLong 是原子性递增或者递减类,其内部使用 Unsafe 来实现,我们看下面的代码：
```java
public class AtomicLong extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 1927816293512124184L;

    // setup to use Unsafe.compareAndSwapLong for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;
    static final boolean VM_SUPPORTS_LONG_CAS = VMSupportsCS8();

    private static native boolean VMSupportsCS8();

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicLong.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile long value;

    public AtomicLong(long initialValue) {
        value = initialValue;
    }
    ...
```
&emsp;在[《Java 并发编程 之 Unsafe 类解析》](http://zhoujiapeng.top/java/java-unsafe)文章中介绍过，并不是所有的类内部都可以通过 Unsafe.getUnsafe() 来获取 Unsafe 的实例，这里可以这样做是因为 AtomicLong 是在 rt.jar 包下面，是通过 BootStrap 类加载器进行加载的。
&emsp;下面看一下 AtomicLong 中的主要函数：
```java
//调用unsafe方法，原子性设置value值为原始值+1，返回值为递增后的值
public final long incrementAndGet() {
        return unsafe.getAndAddLong(this, valueOffset, 1L) + 1L;
    }

public final long decrementAndGet() {
        return unsafe.getAndAddLong(this, valueOffset, -1L) - 1L;
    }

public final long getAndIncrement() {
        return unsafe.getAndAddLong(this, valueOffset, 1L);
    }

public final long getAndDecrement() {
        return unsafe.getAndAddLong(this, valueOffset, -1L);
    }
...
```
&emsp;AtomicLong 没有使用 synchronized 或其它锁机制来实现同步，这些都是阻塞算法，对性能有一定的损耗，而是使用 CAS 非阻塞算法，但是在高并发情况下还是会存在一定的问题。
&emsp;我们先来看一下 getAndIncrement 方法，它调用了 unsafe.getAndAddLong 方法，我们来看一下这个方法：
```java
public final long getAndAddLong(Object var1, long var2, long var4) {
        long var6;
        do {
            var6 = this.getLongVolatile(var1, var2);
        } while(!this.compareAndSwapLong(var1, var2, var6, var6 + var4));

        return var6;
    }
```
&emsp;这里使用了 CAS **自旋**操作来设置新值，跟[《Java 并发编程 之 ThreadLocalRandom》](http://zhoujiapeng.top/java/java-threadLocalRandom/#1-random)文章中讲到的 Random 类的局限性类似，如果大量线程同时调用这个方法，**大量线程进行自旋重试,这会降低并发性能**，JDK8 提供了一个在高并发下性能更好的 LongAdder 类。

## 2. JDK8 新增的原子操作类 LongAdder
&emsp;既然 AtomicLong 的性能瓶颈是由于过多线程同时去竞争一个变量的更新而产生的,那么如果把一个变量分解为多个变量,让同样多的线程去竞争多个资源,是不是就解决了性能问题？是的,LongAdder 就是这个思路。

![AtomicLong](https://gitee.com/JP6907/Pic/raw/master/java/AtomicLong.png)
&emsp;如图，AtomicLong 多个线程同时竞争同一个原子变量。

![LongAdder](https://gitee.com/JP6907/Pic/raw/master/java/LongAdder.png)
&emsp;使用 LongAdder 时,则是在内部维护多个 Cell 变量,每个 Cell 里面有一个初始值为 0 的long型变量,这样,在同等并发量的情况下,争夺单个变量更新操作的线程量会减少,这变相地减少了争夺共享资源的并发量。另外,多个线程在争夺同一个 Cell 原子变量时如果失败了,它并不是在当前 Cell 变量上一直自旋 CAS 重试,而是尝试在其他 Cell 的变量上进行 CAS 尝试 ,这个改变增加了当前线程重试 CAS 成功的可能性。最后,在获取 LongAdder 当前值时, 是把所有 Cell 变量的 value 值累加后再加上 base 返回的。

&emsp;LongAdder 继承了 Striped64，base 和 Cell 是封装在 Striped64 里面的，我们首先看一下 Cell 的结构：
```java
@sun.misc.Contended static final class Cell {
        volatile long value;
        Cell(long x) { value = x; }
        final boolean cas(long cmp, long val) {
            return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
        }

        // Unsafe mechanics
        private static final sun.misc.Unsafe UNSAFE;
        private static final long valueOffset;
        static {
            try {
                UNSAFE = sun.misc.Unsafe.getUnsafe();
                Class<?> ak = Cell.class;
                valueOffset = UNSAFE.objectFieldOffset
                    (ak.getDeclaredField("value"));
            } catch (Exception e) {
                throw new Error(e);
            }
        }
    }
```
&emsp;Cell 类的结构很简单，就是封装了一个 long 类型的 value，以及对这个 value的 CAS 操作，保证对 value 操作的原子性。注意这里使用 @sun.misc.Contended 注解对 Cell 类进行字节填充，这防止了 Cell[] 数组中多个元素共享一个缓存行,在性能上是一个提升。
```java
public long sum() {
        Cell[] as = cells; Cell a;
        long sum = base;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        return sum;
    }
```
&emsp;sum()返回当前的值,内部操作是累加所有 Cell 内部的 value 值后再累加 base。由于计算总和时没有对 Cell 数组进行加锁,所以在累加过程中可能有其他线程对 Cell中的值进行了修改,也有可能对数组进行了扩容,所以**sum返回的值并不是非常精确**的, 其返回值并不是一个调用 sum 方法时的原子快照值。
```java
public void reset() {
        Cell[] as = cells; Cell a;
        base = 0L;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    a.value = 0L;
            }
        }
    }
public long sumThenReset() {
        Cell[] as = cells; Cell a;
        long sum = base;
        base = 0L;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null) {
                    sum += a.value;
                    a.value = 0L;
                }
            }
        }
        return sum;
    }
```
&emsp;reset和sumThenReset的实现比较简单，我们来看一下最关键的 add 方法的实现：
```java
public void add(long x) {
        Cell[] as; long b, v; int m; Cell a;
        if ((as = cells) != null || !casBase(b = base, b + x)) {//(1)
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 || //(2)
                (a = as[getProbe() & m]) == null || //(3)
                !(uncontended = a.cas(v = a.value, v + x))) //(4)
                longAccumulate(x, null, uncontended); //(5)
        }
    }
```
&emsp;代码(1)首先看 cells 是否为空，如果为空，则在 base 上进行累加，如果 cells 不为空或者在 base 上累加失败则执行根据(2)(3)决定应该访问 cells 数组里面的哪一个 cell 元素。(2)的意思是判断 cells 数组是否为空，以及它的长度，如果不为空且长度至少为1，也就是 cells 存在，则判断(3)，(3)根据当前 cells 的长度决定应该访问cells 数组里面的哪一个 cell 元素(getProbe() & m)，(4) 则是对这个 Cell 元素执行 CAS 操作进行累加。如果执行失败，则需要调用 longAccumulate 函数，这是 Cells 数组被初始化和扩容的地方。
```java
final void longAccumulate(long x, LongBinaryOperator fn,
                              boolean wasUncontended) {
        //(1)初始化当前线程的变量 LocalRandomProbe 的值
        int h;
        if ((h = getProbe()) == 0) {
            ThreadLocalRandom.current(); // force initialization
            h = getProbe();
            wasUncontended = true;
        }
        boolean collide = false;                // True if last slot nonempty
        for (;;) {
            Cell[] as; Cell a; int n; long v;
            //(2)cells不为空
            if ((as = cells) != null && (n = as.length) > 0) {
                //(2.1)如果当前线程对应的 cell 不存在，则创建
                if ((a = as[(n - 1) & h]) == null) {
                    if (cellsBusy == 0) {       // Try to attach new Cell
                        Cell r = new Cell(x);   // Optimistically create
                        //新建cell元素，需要调用casCellsBusy将cellsBusy标志设置为1
                        if (cellsBusy == 0 && casCellsBusy()) {
                            boolean created = false;
                            try {               // Recheck under lock
                                Cell[] rs; int m, j;
                                if ((rs = cells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {
                                cellsBusy = 0;
                            }
                            if (created)
                                break;
                            continue;           // Slot is now non-empty
                        }
                    } 
                    //cellsBusy当前忙，但是不会冲突，循环重试
                    collide = false;
                }
                else if (!wasUncontended)       // CAS already known to fail
                    wasUncontended = true;      // Continue after rehash
                //(2.2)当前Cell存在，执行CAS进行累加
                else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                             fn.applyAsLong(v, x))))
                    break;
                //(2.3)累加失败，如果当前cell数组元素个数大于CPU个数，说明不会发生冲突，循环重试
                else if (n >= NCPU || cells != as)
                    collide = false;            // At max size or stale
                //(2.4)当前cells的元素小于当前机器CPU个数，会发生冲突
                else if (!collide)
                    collide = true;
                //有冲突，即当前cells的元素小于当前机器CPU个数，并且当前多个线程访问了cells同一个元素
                //如果Cell数组个数没有达到CPU个数并且有冲突则扩容
                //(2.5)如果会发生冲突，则进行 Cell 数组扩容
                else if (cellsBusy == 0 && casCellsBusy()) {
                    try {
                        if (cells == as) {      // Expand table unless stale
                            //扩充为两倍
                            Cell[] rs = new Cell[n << 1];
                            //拷贝原来的值
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];
                            cells = rs;
                        }
                    } finally {
                        //重置标志
                        cellsBusy = 0;
                    }
                    collide = false;
                    continue;                   // Retry with expanded table
                }
                //h的改变会改变当前线程对应访问的cell位置
                h = advanceProbe(h);
            }
            //(3)cells为空，初始化 cell 数组
            else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
                boolean init = false;
                try {                           // Initialize table
                    if (cells == as) {
                        //初始化 cells数组长度为 2
                        Cell[] rs = new Cell[2];
                        //h = getProbe()，计算当前线程应该访问cell数组哪个位置
                        rs[h & 1] = new Cell(x);
                        cells = rs;
                        //标志已经初始化
                        init = true;
                    }
                } finally {
                    //重置 cellsBusy 标志，不需要 cas操作，因为 cellsBusy 是 volatile 类型
                    cellsBusy = 0;
                }
                if (init)
                    break;
            }
            else if (casBase(v = base, ((fn == null) ? v + x :
                                        fn.applyAsLong(v, x))))
                break;                          // Fall back on using base
        }
    }
```
&emsp;LongAdder 维护了一个延迟初始化的原子性更新数组(默认情况 下 Cell 数组是 null〕和一个基值变量 base 。由于 Cells 占用的内存是相对比较大的,所以一开始并不创建它,而是在需要时创建,也就是惰性加载。当一开始判断 Cell 数组是 null 并且并发线程较少时,所有的累加操作都是对 base 变量进行的 。保持 Cell 数组的大小为 2 的N次方,在初始化时 Cell 数组中的 Cell 元素个数为2。
&emsp;longAccumulate 方法的基本思路为：
- (1)初始化当前线程的变量 LocalRandomProbe 的值
- 循环
    - (2)如果cells不为空
        - (2.1)如果当前线程对应的 cell 不存在，则创建
        - (2.2)如果当前Cell存在，执行CAS进行累加
        - (2.3)如果累加失败，如果当前cell数组元素个数大于CPU个数，说明不会发生冲突，循环重试
        - (2.4)如果当前cells的元素小于当前机器CPU个数，会发生冲突
        - (2.5)如果会发生冲突，则进行 Cell 数组扩容
    - (3)cells为空，初始化 cell 数组

&emsp;需要注意的是，在进行 Cell 数组初始化和扩容的时候，需要调用 casCellsBusy 将 cellsBusy 标志设置为1，防止多个线程同时初始化或扩容。

&emsp;总结一下，LongAdder 类通过内部 cells 数组分担了高并发下多线程同时对一个原子变量进行更新时的竞争量,让多个线程可以同时对 cells 数组里面的元素进行并行的更新操作。另外,数组元素 Cell 使用 @sun.misc.Contended 注解进行修饰,这避免了 cells 数组内多个原子变量被放入同一个缓存行,也就是避免了伪共享,这对性能也是一个提升。


## 3. LongAccumulator 类
&emsp;LongAdder 类是 LongAccumulator 的一个特例,LongAccumulator 比 LongAdder 的功能更强大。看一下它的构造函数：
```java
public LongAccumulator(LongBinaryOperator accumulatorFunction,
                           long identity) {
        this.function = accumulatorFunction;
        base = this.identity = identity;
    }
```
&emsp;identity 其实就是 base 值，LongBinaryOperator 可以传入一个累加规则，LongBinaryOperator 是一个接口，只定义了一个方法：
```java
@FunctionalInterface
public interface LongBinaryOperator {
    long applyAsLong(long left, long right);
}
```
&emsp;调用 LongAdder 就相当于使用下面的方式调用 LongAccumulator：
```java
LongAdder adder = new LongAdder();
LongAccumulator accumulator = new LongAccumulator(new LongBinaryOperator() {
    @Override
    public long applyAsLong(long left, long right) {
        return left + right;
    }
},0);
```
&emsp;LongAdder 的 add 方法相当于 LongAccumulator 的 accumulate 方法：
```java
//LongAdder.add
public void add(long x) {
        Cell[] as; long b, v; int m; Cell a;
        if ((as = cells) != null || !casBase(b = base, b + x)) {//(1)
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 || //(2)
                (a = as[getProbe() & m]) == null || //(3)
                !(uncontended = a.cas(v = a.value, v + x))) //(4)
                longAccumulate(x, null, uncontended); //(5)
        }
    }
//LongAccumulator.accumulate
public void accumulate(long x) {
        Cell[] as; long b, v, r; int m; Cell a;
        if ((as = cells) != null ||
            (r = function.applyAsLong(b = base, x)) != b && !casBase(b, r)) {
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[getProbe() & m]) == null ||
                !(uncontended =
                  (r = function.applyAsLong(v = a.value, x)) == v ||
                  a.cas(v, r)))
                longAccumulate(x, function, uncontended);
        }
    }
```
&emsp;对比可以发现，这两个方法基本上是一致的，唯一不同的是，add 方法的累加规则默认使用 b+x，而 accumulate 调用的是 LongBinaryOperator.applyAsLong，
我们可以看一下 longAccumulate 方法：
```java
final void longAccumulate(long x, LongBinaryOperator fn,
                              boolean wasUncontended) {
        ...
            else if (casBase(v = base, ((fn == null) ? v + x :
                                        fn.applyAsLong(v, x))))
                break;                          // Fall back on using base
        }
    }
```
&emsp;可以看到，如果 LongBinaryOperator 参数为 null，则默认使用 v+x 规则，否则调用 fn.applyAsLong。

&emsp;总结一下，LongAdder 类是 LongAccumulator 的一个特例,只是后者提供了更加强大的功能, 可以让用户自定义累加规则。


## 4. 解决CAS自旋在高并发情况下的性能问题的思路比较
&emsp;在[《Java 并发编程 之 ThreadLocalRandom》](http://zhoujiapeng.top/java/java-threadLocalRandom)文章中以及本文都有提到，使用 CAS 自旋来设置变量的值可以避免加锁和线程调度的开销，但这可能导致大量线程进行自旋重试,从而降低并发性能。对此，ThreadLocalRandom 和 LongAdder 都有自己的解决方法，我们来对比一下：
- ThreadLocalRandom 的方法是让每一个线程都维护自己随机数种子，而生成随机数的算法是线程无关的，从而实现每个线程可以在不加锁的情况下随意调用随机数生成方法。
- LongAdder 的情况和 ThreadLocalRandom 不同的是，LongAdder 的 value 值是需要对所有线程共享的，所以不可能让每一个线程独自维护一个 value，它的方法是在内部维护多个 Cell 变量，多个线程对 value 的 CAS 操作可以分散到多个 Cell 变量上，减少了争夺共享资源的并发量，最后,在获取 LongAdder 当前值时, 是把所有 Cell 变量的 value 值累加后再加上 base 返回。



&nbsp;
&nbsp;
> 参考：
《Java并发编程之美》