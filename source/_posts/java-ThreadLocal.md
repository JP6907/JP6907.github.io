---
title: Java 并发编程 之 ThreadLocal
subtitle: ThreadLocal 详解
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - java
  - 编程语言
  - 并发
categories:
  - java
date: 2019-10-13 15:24:41
---


# ThreadLocal 详解

## 1. 前言
&emsp;在我们日常 Java Web 开发中难免遇到需要把一个参数层层的传递到最内层，然后中间层根本不需要使用这个参数，或者是仅仅在特定的工具类中使用，这样我们完全没有必要在每一个方法里面都传递这样一个通用的参数。如果有一个办法能够在任何一个类里面想用的时候直接拿来使用就太好了。Java的Web项目大部分都是基于Tomcat，每次访问都是一个新的线程，这样让我们联想到了ThreadLocal，每一个线程都独享一个ThreadLocal，在接收请求的时候set特定内容，在需要的时候get这个值。下面我们就进入主题。

## 2. 概述
&emsp;多线程访问同一个共享变量时很容易出现并发问题，ThreadLocal 提供了线程本地变量，如果创建一个 ThreadLocal 变量。那么访问这个变量的每个线程都会有这个变量的一个本地**副本**，当多线程操作这个变量时，实际操作的是自己本地内存里面的变量，从而避免了线程安全问题。
```java
public static ThreadLocal<String> threadLocal = new ThreadLocal<String>();
threadLocal.set("Hello World");
threadLocal.remove();
```

## 3. 实现原理
&emsp;可以看到 ThreadLocal有一个内部类 ThreadLocalMap，这是一个定制化的 Hashmap，可以储存一些键值对。
```java
//ThreadLocal.java
static class ThreadLocalMap {
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
        ......
}
```
&emsp;另外，在 Thread 类内部，有两个 ThreadLocalMap 类型的属性：
```java
//Thread.java
ThreadLocal.ThreadLocalMap threadLocals = null;
ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
```
&emsp;这两个变量就是本文的重点了，其实每个线程的本地变量不是存放在 ThreadLocal 实例里面的，而是存放在 Thread.threadLocals 里面，也就是说，ThreadLocal 类型的本地变量存放在具体的线程内存空间中，ThreadLocal 只是一个工具壳，封装了操作这些副本变量的方法。

&emsp;下面分析一些 ThreadLocal 的 set、get 以及 remove 方法的实现逻辑：
### 3.1 set 方法
```java
public void set(T value) {
        //获取当前线程
        Thread t = Thread.currentThread();
        //获取当前线程内部的threadLocals
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
        //如果是第一次调用，则创建当前线程对应的threadLocals
            createMap(t, value);
    }
ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```
&emsp;set的逻辑很简单，获取当前线程内部的 threadLocals，**ThreadLocal 对象**作为key，变量值作为 value，保存到 threadLocals 里面。
&emsp;这里使用了延迟加载的策略，Thread 里面的 threadLocals 默认为 null，只有等到第一次使用到的时候才会创建。

### 3.2 get 方法
```java
public T get() {
        //获取当前线程
        Thread t = Thread.currentThread();
        //获取当前线程内部的 threadlocals
        ThreadLocalMap map = getMap(t);
        //如果 threadlocals 不为 null，则从 threadlocals 里面查找
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        //threadlocals为null或者threadlocals找不到当前 ThreadLocal 对象，则进行初始化
        return setInitialValue();
    }
private T setInitialValue() {
        //value为null
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        //如果当前threadlocals不为null
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
protected T initialValue() {
        return null;
    }
```
&emsp;get 方法的思路和set类似，同样是获取当前线程内部的 threadLocals，然后以当前 ThreadLocals 对象为 key 去查找，若找不到则为当前 ThreadLocals 对象初始化一个 null 的 value。

### 3.3 remove 方法
```java
public void remove() {
        //获取当前线程内部的 threadlocals
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
            //从threadlocals里面移除指定的ThreadLocal实例的本地变量
             m.remove(this);
     }
```
&emsp;需要注意的是，当一个本地变量不再使用时，要调用 remove 方法，将这个本地变量移除，减少内存的占用。

### 3.4 总结
&emsp;每个线程内部都有一个名为 threadLocals 的成员变量，该变量类型为 HashMap，其中 key 为我们定义的 ThreadLocal 对象的 this 引用，value 则为我们使用 set 设置的值。每个线程的本地变量存放在线程自己的内存变量 threadLocals 中，如果当前线程一直不消亡，那么这些本地变量会一直存在，所以可能造成内存溢出，因此使用完毕时候，要调用 remove 方法，将这个本地变量移除，减少内存的占用。

## 4. 举个例子
&emsp;ThreadLocal 会将数据以K-V形式保存在具体的 Thread 中，其中的 Key 是 ThreadLocal 对象，Value 是具体的值，比如：
```java
private ThreadLocal<UserInfo> threadLocal = new ThreadLocal<UserInfo>();
```
&emsp;则 key 为 threadLocal 对象。
&emsp;这里以《Spring5高级编程(第5版)》中 AOP 的一段代码为例，通过使用前置通知来保护方法访问，从而实现对用户身份限制的效果，其中的 ThreadLocal 就用来在当前线程共享登录用户的信息。
&emsp;SecureManager 类封装了 threadLocal，并且 threadLocal 是 static 类型，所以可以在程序的任意地方通过 SecureManager 来访问同一个 threadLocal，从而共享 threadLocal 里面保存的用户信息。
```java
public class SecureManager {
    //注意这里是 static，所有 SecureManager 对象才能共享同一个 threadLocal
   private static ThreadLocal<UserInfo> threadLocal = new ThreadLocal<UserInfo>();

    public void login(String userName,String password){
        threadLocal.set(new UserInfo(userName,password));
    }
    public void logout(){
        threadLocal.set(null);
    }
    public UserInfo getLoggedUserInfo(){
        return threadLocal.get();
    }
}
```
&emsp;SecureBean 里面 writeSecureMessage 方法是我们将会调用的方法，但是希望在调用这个方法之前验证是否已经登录，SecureAdvice 类实现了前置通知的接口。
```java
public class SecureBean {
       public void writeSecureMessage(){
        System.out.println("This is the secure message!");
    }
}
public class SecureAdvice implements MethodBeforeAdvice {
    private SecureManager secureManager;

    public SecureAdvice(){
        this.secureManager = new SecureManager();
        System.out.println("SecureManager in SecureAdvice:" + secureManager);
    }

    public void before(Method method, Object[] objects, Object o) throws Throwable {
        UserInfo userInfo = secureManager.getLoggedUserInfo();

        if(userInfo==null){
            System.out.println("No user authenticated!");
            throw new SecurityException("You must login before attempting to invoke the method:"
                    + method.getName());
        }else if("John".equals(userInfo.getUserName())){
            System.out.println("Logged in user John!");
        }else{
            System.out.println("User " + userInfo.getUserName() + " is not authenticated!");
            throw new SecurityException("User " + userInfo.getUserName() + " is not authenticated!");
        }
    }
}
```
&emsp;接下来将 SecureBean 和 SecureAdvice 的实例绑定在一起，获取 SecureBean 的代理，当我们调用代理的 writeSecureMessage 时，就会触发先执行 SecureAdvice 的 before 方法。
```java
public static SecureBean getSecureBean(){
        SecureBean bean = new SecureBean();
        SecureAdvice advice = new SecureAdvice();
        //创建目标对象的代理
        ProxyFactory proxyFactory = new ProxyFactory();
        //添加通知
        proxyFactory.addAdvice(advice);
        //指定织入目标
        proxyFactory.setTarget(bean);
        //获取代理实例
        SecureBean proxy = (SecureBean)proxyFactory.getProxy();
        return proxy;
    }
```
&emsp;最后是main方法：
```java
public static void main(String[] args) {
        SecureManager secureManager = new SecureManager();
        System.out.println("SecureManager in main:" + secureManager);
        SecureBean secureBean = getSecureBean();

        secureManager.login("John","123456");
        secureBean.writeSecureMessage();
        secureManager.logout();

        try{
            secureManager.login("Invalid user","123");
            secureBean.writeSecureMessage();
        }catch (SecurityException ex){
            System.out.println("Exception caught:" + ex.getMessage());
        }finally {
            secureManager.logout();
        }

        try{
            secureBean.writeSecureMessage();
        }catch (SecurityException ex){
            System.out.println("Exception caught:" + ex.getMessage());
        }finally {
            secureManager.logout();
        }
    }
```
&emsp;注意我们在main方法中创建一个 SecureManager 实例来执行登录操作，而在 getSecureBean 方法中创建一个 SecureAdvice 实例，在 SecureAdvice 里面也会创建一个 SecureManager 实例来获取已登录的用户信息。由于 SecureManager 里面的 threadLocal 是 static，所以实际上我们操作的是同一个 threadLocal 对象，从而达到了共享用户信息的效果。

## 5. ThreadLocal 不支持继承性
&emsp;ThreadLocal 存在的一个问题是，父线程的 ThreadLocal 变量无法被子线程获取到，这是很合理的现象，因为 ThreadLocal 变量是绑定在线程上的，而父线程和子线程是两个不同的的线程，子线程自然而然无法获取到父线程的本地变量。
```java
public class ThreadLocalsTest {

    //创建线程变量
    public static ThreadLocal<String> threadLocal = new ThreadLocal<String>();
    //inheritableThreadLocal 子线程创建的时候会复制父线程的一份副本
    public static ThreadLocal<String> inheritableThreadLocal = new InheritableThreadLocal<String>();

    public static void main(String[] args){
        threadLocal.set("Hello World");
        inheritableThreadLocal.set("Hello World");

        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                //子线程输出线程变量
                System.out.println("thread threadLocal:" + threadLocal.get()); //获取不到
                System.out.println("thread inheritableThreadLocal:" + inheritableThreadLocal.get()); //可以获取到
            }
        });
        thread.start();

        //主线程输出线程变量
        System.out.println("main threadLocal:" + threadLocal.get()); //null
        System.out.println("main inheritableThreadLocal:" + inheritableThreadLocal.get()); //Hello World
    }

}
```
&emsp;为了解决这个问题，InheritableThreadLocal应运而生。

## 6. InheritableThreadLocal 类
&emsp;InheritableThreadLocal 继承自 ThreadLocal，提供了一个特性，就是让子线程可以访问在父线程中设置的本地变量。
```java
public class InheritableThreadLocal<T> extends ThreadLocal<T> {
    protected T childValue(T parentValue) {
        return parentValue;
    }

    ThreadLocalMap getMap(Thread t) {
       return t.inheritableThreadLocals;
    }

    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
}
```
&emsp;InheritableThreadLocal 重写了三个方法。getMap 返回的是 Thread 内部的 inheritableThreadLocals 变量而不是 threadLocals，createMap 也是为 inheritableThreadLocals 创建新的值。简单来说，在 InheritableThreadLocal 的世界里，inheritableThreadLocals 替代了 threadLocals。
&emsp;接下来我们看一下 InheritableThreadLocal 怎么实现子线程继承父线程的 InheritableThreadLocal，这要从 Thread 的创建说起：
```java
public Thread() {
        init(null, null, "Thread-" + nextThreadNum(), 0);
    }
private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize) {
                          //最后一个参数为 inheritThreadLocals=true，表示继承inheritThreadLocals
        init(g, target, name, stackSize, null, true);
    }
private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals) {
        ...
        //获取当前线程，新线程是由当前线程创建的，所以当前线程是新线程的父线程
        Thread parent = currentThread();
        ...
        //如果inheritThreadLocals参数为true且父线程的inheritableThreadLocals不为null
        if (inheritThreadLocals && parent.inheritableThreadLocals != null)
            //设置子线程的inheritableThreadLocals值
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
        /* Stash the specified stack size in case the VM cares */
        this.stackSize = stackSize;

        /* Set thread ID */
        tid = nextThreadID();
    }
static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
        return new ThreadLocalMap(parentMap);
    }
private ThreadLocalMap(ThreadLocalMap parentMap) {
            Entry[] parentTable = parentMap.table;
            int len = parentTable.length;
            setThreshold(len);
            table = new Entry[len];

            for (int j = 0; j < len; j++) {
                Entry e = parentTable[j];
                if (e != null) {
                    @SuppressWarnings("unchecked")
                    ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
                    if (key != null) {
                        //（1）调用重写的方法 childValue
                        Object value = key.childValue(e.value);
                        Entry c = new Entry(key, value);
                        int h = key.threadLocalHashCode & (len - 1);
                        while (table[h] != null)
                            h = nextIndex(h, len);
                        table[h] = c;
                        size++;
                    }
                }
            }
        }
```java
protected T childValue(T parentValue) {
        return parentValue;
    }
```

&emsp;返回一个新的ThreadLocalMap，ThreadLocalMap 的构造函数其实就是把父线程的 inheritThreadLocals 成员变量的值复制到新的 ThreadLocalMap中 ,其中代码（1）处调用了重写的childValue方法。
&emsp;总的来说，InheritableThreadLocal 将本地变量保存到了具体线程的 inheritThreadLocals 变量里面。当父线程创建子线程时，构造函数会把父线程中 inheritThreadLocals 变量里面的本地变量**复制**（注意不是共享）一份保存到子线程的 inheritThreadLocals 变量里面。

&emsp;那么在什么情况下需要子线程可以获取父线程的本地变量呢？比如子线程需要使用存放在 threadLocal 变量的用户登录信息，一些中间件需要把统一的id追踪的整个调用链路记录下来。有了 InheritableThreadLocal，我们就不再需要通过传参的形式来传递这些变量了。

> 参考：
《Java 并发编程之美》

&nbsp;
>本文作者：ZJP
版权声明：本文为博主原创文章，遵循 [CC 4.0 BY-SA](http://creativecommons.org/licenses/by-sa/4.0/) 版权协议，转载请附上原文出处链接和本声明。
本文链接：http://zhoujiapeng.top/java/java-ThreadLocal