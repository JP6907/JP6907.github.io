---
title: Java 并发编程 之 Unsafe 类解析
subtitle: Unsafe 类解析
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - java
  - 编程语言
  - 并发
categories:
  - java
date: 2019-09-27 16:47:24
---


# Unsafe 类解析

## 1. CAS 操作
&emsp;Java中的锁提供给我们一种创建原子操作的方法，但是会导致线程上下文的切换和重新调度开销。Java 提供了非阻塞的 volatile 关键字来解决共享变量的可见性问题,这在一定程度上弥补了锁带来的开销问题,但是 volatile 只能保证共享变量的可见性,不能解决读改一写等的原子性问题。
&emsp;CAS 即 Compare and Swap,其是 JDK 提供的非阻塞原子性操作,它通过**硬件**保证了比较更新操作的原子性。JDK 里面的 Unsafe 类提供了一系列的 compareAndSwap*方法,下面以 compareAndSwapLong 方法为例进行简单介绍：
```java
boolean compareAndSwapLong(Object obj, long valueOffset, long expect, long update);
```
&emsp;compareAndSwap 的意思是比较并交换。CAS 有四个操作数,分别为:对象内存位置、对象中的变量的偏移量、变量预期值和新的值。其操作含义是,如果对象 obj 中内存偏移量为 valueOffset 的变量值为 expect ,则 使用新的值 update 替换旧的值 expect。这是处理器提供的一个**原子性**指令。
&emsp;由于在调用 compareAndSwapLong 之前我们需要先获取变量的原有值，这中间就产生了一个时间窗口，会引发CAS操作中经典的**ABA**问题。举例来说:假如线程 I 使用 CAS 修改初始值为 A 的变量X,那么线程 I 会首先去获取当前变量 X 的值(为 A〕,然后使用 CAS 操作尝试修改 X 的值为 B , 如果使用 CAS 操作成功了, 那么程序运行一定是正确的吗 ?其实未必,这是因为有可能在线程 I 获取变量 X 的值 A 后,在执行 CAS 前,线程 II 使用 CAS 修改了 变量 X 的值为 B ,然后又使用 CAS 修改了变量 X 的值为 A 。所以虽然线程 I 执行CAS时 X 的值是 A , 但是这个 A 己经不是线程 I 获取时的 A 了。这就是ABA问题 。ABA 问题的产生是因为变量的状态值产生了环形转换,就是变量的值可以从A到B,然后再从B到A。如果变量的值只能朝着一个方向转换 ,比如A到B,B到C, 不构成环形,就不会存在问题。JDK 中的 AtomicStampedReference 类给每个变量的状态值都配备了一个时间戳, 从而避免了 ABA 问题的产生。更多关于ABA 问题的详细介绍可以参考文章[《Java 并发编程 之 ABA 问题》](http://zhoujiapeng.top/java/java-aba-question)。



## 2. Unsafe 类

### 2.1 Unsafe 类中的重要方法
&emsp;JDK 的 rt.jar 包中的 Unsafe 类提供了**硬件级别的原子性操作**,Unsafe 类中的方法都是 native 方法 ,它们使用 JNI 的方式访问本地 C++实现库。下面我们来了解一下 Unsafe 提供的几个主要的方法以及编程时如何使用 Unsafe 类做一些事情。
- public native long objectFieldOffset(Field field): 返回指定的变量在所属类中的内存偏移地址,该偏移地址仅仅在该 Unsafe 函数中访问指定字段时使用。如下代码使用 Unsafe 类获取变量 value 在AtomicLong 对象中的内存偏移:
```java
    static Unsafe unsafe = sun.misc.Unsafe.getUnsafe();
    static {
        try {
            long valueOffset = unsafe.objectFieldOffset(AtomicLong.class.getDeclaredField("value"));    
        }catch (Exception ex){
            throw new Error(ex);
        }
    }
```
- int arrayBaseOffset(Class<?> arrayClass): 获取数组中第一个元素的地址。
- int arrayIndexScale(Class<?> arrayClass)：获取数组中一个元素占用的字节。
- boolean compareAndSwapLong(Object obj, long valueOffset, long expect, long update)：比较对象 obj 中内存偏移量为 valueOffset 的变量值是否与 expect 相等,相等则使用新的值 update 替换旧的值 expect，然后返回true，否则返回false。
- long getLongVolatile(Object obj, long offset)：获取对象 obj 中偏移量为 offset 的变量对应 volatile 语义的值。
- void putLongVolatile(Object obj, long offset, long value)：设置 obj 对象中 offset偏移的类型为 long 的 field 的值为 value ,支持 volatile 语义。
- void putOrderedLong(Object obj,long offset,long value)：设置 obj 对象中 offset偏移地址对应的 long 型 field 的值为 value。这是一个有延迟的 putLongvolatile 方法,并且不保证值修改对其他线程立刻可见。只有在变量使用 volatile 修饰并且预计会被意外修改时才使用该方法 。
- void park(boolean isAbsoulte,long time)：阻塞当前线程（这个方法在AQS中的使对象等待时会使用到）。isAbsoulte的为是否是绝对时间，如果isAbsoulte为false，其中time为大于0，那么当前线程会在等待time后唤醒。如果isAbsolute为false，其中time为0，那么线程为无限等待。如果isAbosult为true，那么time就是指的是绝对时间也就是换算为ms后的绝对时间。另外，当其他线程调用了当前阻塞线程的**interrupt**方法而中断了当前线程时，当前线程也会返回，而其他线程调用**unPark**方法并且把当前线程作为参数时也会返回。
- void unpark(Object thread)：唤醒调用park线程的线程
- long getAndSetLong(Object obj,long offset,long update)：获取对象obj中偏移量为offset的变量volatile语义的当前值，并设置变量volatile语义的值为update,这里使用了CAS自旋的方法。
```java
public final long getAndSetLong(Object obj,long offset , long update){
    long l;
    do{
        l = getLongVolatile(obj,offset); //（1）
    }while(!compareAndSwapLong(obj,offset,l,update));
    return l;
}
```
&emsp;由以上代码可知 ,首先 (I) 处的 getLongvolatile 获取当前变量的值 ,然后使用CAS 原子操作设置新值。这里使用 while 循环是考虑到,在多个线程同时调用的情况下 CAS 失败时需要重试。
- long getAndAddLong(Object obj,long offset,long add)方法：设置原始值为val+add。
```java
public final long getAndSetLong(Object obj,long offset , long add){
    long l;
    do{
        l = getLongVolatile(obj,offset);
    }while(!compareAndSwapLong(obj,offset,l,l+add));
    return l;
}
```

### 2.2 如何使用 Unsafe 类
&emsp;我们可以看到 AtomicLong 等类其类是这样使用 Unsafe 类的：
```java
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long valueOffset;
static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicLong.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }
```
&emsp;但是如果我们在自己写的类中同样使用这种方式，会报错：
```
Caused by: java.lang.SecurityException: Unsafe
	at sun.misc.Unsafe.getUnsafe(Unsafe.java:90)
	at Unsafe.TestUnsafe.<clinit>(TestUnsafe.java:12)
```
&emsp;为了找出原因，我们查看一下 Unsafe.getUnsafe 方法：
```java
@CallerSensitive
    public static Unsafe getUnsafe() {
        Class localClass = Reflection.getCallerClass();
        if (!VM.isSystemDomainLoader(localClass.getClassLoader())) {
            throw new SecurityException("Unsafe");
        } else {
            return theUnsafe;
        }
    }
    //判断paramClassLoader是不是Bootstrap类加载器
    public static boolean isSystemDomainLoader(ClassLoader paramClassLoader) {
        return paramClassLoader == null;
    }
```
&emsp;getUnsafe 中 Reflection.getCallerClass() 获取的是用户的TestUnsafe类，TestUnsafe 是由 AppClassLoader 加载的，所以这里抛出了异常。
&emsp;我们的启动 main 函数所在 的类是使用 AppClassLoader 加载的,所以在 main 函数里面加载 Unsafe 类的时候,根据委托机制, 会委托 Bootstrap 类加载器加载。
&emsp;Unsafe 类是 rt.jar 包提供的，rt.jar 里面的类是 Bootstrap 类加载器加载的，如果没有加载器的限制，那么我们的应用程序就可以随意使用 Unsafe，而 Unsafe 可以直接操作内存，这是不安全的，所以限制了只能在 rt.jar 包里面的核心类使用 Unsafe 功能。
&emsp;如果真的想实例化Unsafe类，我们可以通过反射机制来实现：
```java
public class TestUnsafe {

    static Unsafe unsafe;
    //记录变量unsafe在类TestUnsafe中的偏移量
    static long stateOffset;
    private volatile long state = 0;

    static {
        try{
            //使用反射机制绕过安全检查获取unsafe
            Field field = Unsafe.class.getDeclaredField("theUnsafe");
            //设置为可存取
            field.setAccessible(true);
            //获取值
            unsafe = (Unsafe)field.get(null);
            stateOffset = unsafe.objectFieldOffset(TestUnsafe.class.getDeclaredField("state"));

        }catch (Exception e){
            System.out.println(e.getLocalizedMessage());
            throw new Error(e);
        }
    }

    public static void main(String[] args){
        TestUnsafe test = new TestUnsafe();
        Boolean success = unsafe.compareAndSwapInt(test,stateOffset,0,1);
        System.out.println(success);
    }
}
```

&nbsp;
&nbsp;
> 参考资料：
《Java并发编程之美》