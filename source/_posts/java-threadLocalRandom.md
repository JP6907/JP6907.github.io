---
title: Java 并发编程 之 ThreadLocalRandom
subtitle: ThreadLocalRandom
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - java
  - 编程语言
  - 并发
categories:
  - java
date: 2019-09-29 20:09:54
---



# Java 并发编程 之 ThreadLocalRandom

## 1. Random 类及其局限性
&emsp;java.util.Random 是使用比较广泛的随机数生成工具类，而且 java.lang.Math 中的随机数也是使用 java.util.Random 的实例。我们看一下 Random 的 nextInt 方法：
```java
public int nextInt(int bound) {
        //参数检查
        if (bound <= 0)
            throw new IllegalArgumentException(BadBound);

        //1.根据老的种子生成新的种子
        int r = next(31);
        int m = bound - 1;
        //2.根据新的种子计算随机数
        if ((bound & m) == 0)  // i.e., bound is a power of 2
            r = (int)((bound * (long)r) >> 31);
        else {
            for (int u = r;
                 u - (r = u % bound) + m < 0;
                 u = next(31))
                ;
        }
        return r;
    }
```
&emsp;由此可见，新的随机数的生成需要两个步骤
- 首先根据老的种子生成新的种子
- 根据新的种子来计算新的随机数

&emsp;我们看一下这里的 next 方法：
```java
protected int next(int bits) {
        long oldseed, nextseed;
        AtomicLong seed = this.seed;
        do {
            oldseed = seed.get();
            nextseed = (oldseed * multiplier + addend) & mask;
        } while (!seed.compareAndSet(oldseed, nextseed));
        return (int)(nextseed >>> (48 - bits));
    }
```
&emsp;在 next 方法里面，使用 CAS 操作，使用新的种子去更新老的种子，因此这里是线程安全的。但是，如果多个线程同时执行到 compareAndSet 函数这里，CAS 操作会保证只有一个线程可以更新种子，失败的线程会通过循环重新获取更新后的种子作为当前种子去
计算老的种子,保证了随机数的随机性。
&emsp;总结:每个 Random 实例里面都有一个原子性的种子变量用来记录当前的种子值,当要生成新的随机数时需 要根据当前种子计算新的种子并更新回原子变量。在多线程下使用单个 Random 实例生成随机数时,当多个线程同时计算随机数来计算新的种子时, 多个线程会竞争同一个原子变量的更新操作,由于原子变量的更新是 CAS 操作,同时只有一个线程会成功,所以会造成**大量线程进行自旋重试,这会降低并发性能**,所以 ThreadLocalRandom 应运而生。

## 2. ThreadLocalRandom
&emsp;为了弥补多线程高并发情况下 Random 的缺陷 , 在且 JUC 包下新增了 ThreadLocalRandom类,下面首先看下如何使用它：
```java
public class ThreadLocalRandomTest {
    public static void main(String[] args){
        ThreadLocalRandom random = ThreadLocalRandom.current();

        for(int i=0;i<10;i++){
            System.out.println(random.nextInt(5));
        }
    }
}
```
&emsp;Random 的缺点是多个线程会使用同一个原子性种子变量, 从而导致对原子变量更新的竞争。ThreadLocalRandom 的原理是让**每个线程都维护一个种子变量**,则每个线程生成随机数时都根据自己老的种子计算新的种子,并使用新种子更新老的种子,再根据新种子计算随机数,就不会存在竞争问题了,这会大大提高并发性能。
![ThreadLocalRandom](https://github.com/JP6907/Pic/blob/master/java/ThreadLocalRandom.png?raw=true)

&emsp;下面从源码具体分析一下。
```java
//ThreadLocalRandom.java
    private static final sun.misc.Unsafe UNSAFE;
    private static final long SEED;
    private static final long PROBE;
    private static final long SECONDARY;
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> tk = Thread.class;
            SEED = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("threadLocalRandomSeed"));
            PROBE = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("threadLocalRandomProbe"));
            SECONDARY = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("threadLocalRandomSecondarySeed"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }
```
&emsp;这里的 SEED 是代表 Thread 类里面 threadLocalRandomSeed 变量的偏移量，Unsafe可以通过这个偏移量来获取到储存在每个 Thread 实例中的 seed，而 threadLocalRandomProbe 在这里是用来判断是否初始化使用的，在其它地方会有其他用途（如 [LongAdder](http://zhoujiapeng.top/java/java-atomicOperationClass) 类中用来决定应该访问cells数组的哪个元素）。
&emsp;下面是获取 ThreadLocalRandom 实例的 current() 方法：
```java
static final ThreadLocalRandom instance = new ThreadLocalRandom();
public static ThreadLocalRandom current() {
        if (UNSAFE.getInt(Thread.currentThread(), PROBE) == 0)
            localInit();
        return instance;
    }
static final void localInit() {
        int p = probeGenerator.addAndGet(PROBE_INCREMENT);
        int probe = (p == 0) ? 1 : p; // skip 0
        long seed = mix64(seeder.getAndAdd(SEEDER_INCREMENT));
        Thread t = Thread.currentThread();
        UNSAFE.putLong(t, SEED, seed);
        UNSAFE.putInt(t, PROBE, probe);
    }
```
&emsp;注意这里的 instance 是 static 的一个 ThreadLocalRandom 实例，current 方法返回的是 instance。因此当多线程通过 current 方法获取 ThreadLocalRandom 实例的时候其实获取到的是同一个实例。由于具体的 seed 是放在线程里面的，ThreadLocalRandom 的实例里面只包含与线程无关的通用算法，所以它是线程安全的。
&emsp;默认情况下，Thread 里面的 threadLocalRandomProbe 变量值是0，如果判断这个值为0，则说明当前线程是第一次调用 ThreadLocalRandom 的 current 方法，那么需要调用 localInit 方法进行初始化。这里使用了延迟初始化，在不需要使用随机数功能的时候就不初始化 Thread 里面的种子变量，这是一种优化。
&emsp;接下来看一下获取随机数的 next 方法：
```java
public int nextInt(int bound) {
        //参数检验
        if (bound <= 0)
            throw new IllegalArgumentException(BadBound);
        //1.根据当前线程中的种子计算新种子
        int r = mix32(nextSeed());
        //2.根据新种子计算随机数
        int m = bound - 1;
        if ((bound & m) == 0) // power of two
            r &= m;
        else { // reject over-represented candidates
            for (int u = r >>> 1;
                 u + m - (r = u % bound) < 0;
                 u = mix32(nextSeed()) >>> 1)
                ;
        }
        return r;
    }
```
&emsp;生成随机数的逻辑和 Random 类似，我们重点看一下生成新种子的 nextSeed 方法：
```java
final long nextSeed() {
        Thread t; long r; // read and update per-thread seed
        UNSAFE.putLong(t = Thread.currentThread(), SEED,
                       r = UNSAFE.getLong(t, SEED) + GAMMA);
        return r;
    }
```
&emsp;上面代码使用 r = UNSAFE.getLong(t, SEED) 获取当前线程中的 threadLocalRandomSeed 变量的值，然后在种子的基础上累加 GAMMA 值作为新的种子重新放回 threadLocalRandomSeed 中，这里不会涉及多线程竞争的问题。

## 3. 总结
&emsp;本文首先讲解了 Random 的实现原理以及 Random 在多线程下需要竞争种子原子变量更新操作的缺点,从而引出 ThreadLocalRandom 类。ThreadLoca!Random 使用 ThreadLocal 的原理,让每个线程都持有一个本地的种子变量,该种子变量只有在使用随机数时才会被
初始化。在多线程下计算新种子时是根据自己线程内维护的种子变量进行更新,从而避免了竞争。

&nbsp;
&nbsp;
> 参考：
《Java并发编程之美》