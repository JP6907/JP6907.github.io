---
title: Java 并发编程 之 锁的概述
subtitle: 锁的概述
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - java
  - 编程语言
  - 并发
categories:
  - java
date: 2019-10-13 17:17:22
---


# 乐观锁与悲观锁
&emsp;乐观锁和悲观锁是在数据库中引入的名词，但是在并发包锁里面也引入了类似的思想。
&emsp;悲观锁指对数据被外界修改持保守态度，认为数据很容易就会被其他线程修改，所以在数据被处理前先对数据进行加锁，并在整个数据处理过程中，使数据处于锁定状态。悲观锁的实现往往依靠数据库提供的锁机制，即在数据库中，在对数据记录操作前给记录加排它锁。如果获取锁失败， 则说明数据正在被其他线程修改， 当前线程则等待或者抛出异常。如果获取锁成功，则对记录进行操作，然后提交事务后释放排它锁。
&emsp;下面看一个例子，使用悲观锁来避免多线程同时对一个记录进行修改：
```java
public int updateEntry(long id) {
    // (1)使用悲观锁获取指定记录
    EntryObject entry= query("select * from tablel where id = #{id} for update",id);
    //(2)修改记录内容， 根据计算修改entry记录的属性
    String name = generatorName(entry) ;
    entry.setName(name);
    //(3)update操作
    int count = update("update tablel set name ＝ #{name} , age=#{age} where id =#{id}", entry);
    return count;
```
&emsp;这里的 updateEntry 整个函数作为一个事务提交，多个线程调用 updateEntry 方法，并且传递的是同一个id时，只有一个线程会执行成功，其它线程阻塞。

&emsp;乐观锁是相对悲观锁来说的，它认为数据在一般情况下不会造成冲突，所以在访问记录前不会加排它锁，而是在进行数据提交更新时，才会正式对数据冲突与否进行检测。具体来说，根据update 返回的行数让用户决定如何去做。
&emsp;乐观锁一般会使用**版本号机制**或**CAS算法**实现。
&emsp;首先看一下版本号方法，将上面的例子改为使用乐观锁的代码如下。
```java
public int updateEntry(long id) {
    // (1)使用乐观锁获取指定记录
    EntryObject entry= query("select * from tablel where id = #{id} for update",id);
    //(2)修改记录内容， version字段不能被修改
    String name = generatorName(entry) ;
    entry.setName(name);
    //(3)update操作
    int count = update("update tablel set name ＝ #{name} , age=#{age}, version=${version}+1 where id =#{id} and version=#{version}", entry);
    return count;
```
&emsp;在调用update之前可能会有多个线程都调用了updateEntry，因此版本号 version 可能已经被修改了，所以 update 的时候需要判断 version 是否为原来的 version，如果是，说明该数据没有被其它线程修改，可以更新，version+1，count 会返回1，否则返回0，表示失败，失败的处理策略由用户决定，这有点CAS操作的意思。

&emsp;接下来看一下CAS方法：
```java
public boolean updateEntry(long id) {
    boolean result = false;
    int retryNum = 5;
    while(retryNum>0){
        // (1)使用乐观锁获取指定记录
        EntryObject entry= query("select * from tablel where id = #{id} for update",id);
        //(2)修改记录内容， version字段不能被修改
        String name = generatorName(entry) ;
        entry.setName(name);
        //(3)update操作
        int count = update("update tablel set name ＝ #{name} , age=#{age}, version=${version}+1 where id =#{id} and version=#{version}", entry);
        if(count ==1){
            result = true;
            break;
        }
        retryNum--;
    }
    return count;
```
&emsp;如上代码使用retryNum 设置更新失败后的重试次数,如果update更新失败，则会重新尝试，这类似于CAS的自旋操作，只是这里没有使用死循环，而是指定了尝试次数。

# 公平锁与非公平锁
&emsp;根据线程获取锁的抢占机制，锁可以分为公平锁和非公平锁，公平锁表示线程获取锁的顺序是按照线程请求锁的时间早晚来决定的，也就是最早请求锁的线程将最早获取到锁。而非公平锁则在运行时闯入，也就是先来不一定先得。
&emsp;ReentrantLock 提供了公平和非公平锁的实现,如果构造函数不传递参数，则默认是非公平锁。
- 公平锁： ReentrantLock pairLock = new ReentrantLock(true)。
- 非公平锁： ReentrantLock pairLock =new ReentrantLock(false)。

**在没有公平性需求的前提下尽量使用非公平锁，因为公平锁会带来性能开销。**


# 独占锁与共享锁
&emsp;根据锁只能被单个线程持有还是能被多个线程共同持有，锁可以分为独占锁和共享锁。
独占锁保证任何时候都只有一个线程能得到锁， ReentrantLock 就是以独占方式实现的。共享锁则可以同时由多个线程持有，例如 ReadWriteLock 读写锁，它允许一个资源可以被多线程同时进行读操作。
&emsp;独占锁是一种悲观锁，由于每次访问资源都先加上互斥锁，这限制了并发性，因为读操作并不会影响数据的一致性，而独占锁只允许在同一时间由一个线程读取数据，其他线程必须等待当前线程释放锁才能进行读取。
&emsp;共享锁则是一种乐观锁，它放宽了加锁的条件，允许多个线程同时进行读操作。

# 可重入锁
&emsp;当一个线程要获取一个被其他线程持有的独占锁时，该线程会被阻塞，那么当一个线程再次获取它自己己经获取的锁时是否会被阻塞呢？如果不被阻塞，那么我们说该锁可重入的。
&emsp;实际上，synchronized 内部锁是可重入锁。可重入锁的原理是在锁内部维护一个线程标示，用来标示该锁目前被哪个线程占用，然后关联一个计数器。一开始计数器值为o,说明该锁没有被任何线程占用。当一个钱程获取了该锁时，计数器的值会变成1 ，这时其他线程再来获取该锁时会发现锁的所有者不是自己而被阻塞挂起。
但是当获取了该锁的线程再次获取锁时发现锁拥有者是自己，就会把计数器值加+1,当释放锁后计数器值-1 。当计数器值为0 时－，锁里面的线程标示被重置为null ， 这时候被阻塞的线程会被唤醒来竞争获取该锁。

# 自选锁
&emsp;由于Java 中的线程是与操作系统中的线程一一对应的，所以当一个线程在获取锁（比如独占锁）失败后，会被切换到内核状态而被挂起。当该线程获取到锁时又需要将其切换到内核状态而唤醒该线程。而从用户状态切换到内核状态的开销是比较大的，在一定程度上会影响并发性能。自旋锁则是，当前线程在获取锁时，如果发现锁已经被其他线程占有，它不马上阻塞自己，在不放弃CPU 使用权的情况下，**多次尝试获取**（默认次数是10 ，可以使用-XX :PreBlockSpinsh 参数设置该值），很有可能在后面几次尝试中其他线程己经释放了锁。如果尝试指定的次数后仍没有获取到锁则当前线程才会被阻塞挂起。由此看来 **自旋锁是使用CPU时间换取线程阻塞与调度的开销**，但是很有可能这些CPU 时间白白浪费了。

&nbsp;
> 参考：
《Java 并发编程之美》