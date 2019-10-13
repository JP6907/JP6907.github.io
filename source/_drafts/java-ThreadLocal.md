---
title: Java 并发编程 之 ThreadLocal
catalog: true
subtitle: ThreadLocal 详解
header-img: /img/article_header/article_header.png
tags:
  - java
  - 编程语言
  - 并发
categories:
  - java
---

# ThreadLocal 详解

## 1. 前言
在我们日常 Java Web 开发中难免遇到需要把一个参数层层的传递到最内层，然后中间层根本不需要使用这个参数，或者是仅仅在特定的工具类中使用，这样我们完全没有必要在每一个方法里面都传递这样一个通用的参数。如果有一个办法能够在任何一个类里面想用的时候直接拿来使用就太好了。Java的Web项目大部分都是基于Tomcat，每次访问都是一个新的线程，这样让我们联想到了ThreadLocal，每一个线程都独享一个ThreadLocal，在接收请求的时候set特定内容，在需要的时候get这个值。下面我们就进入主题。

## 2. 概述
多线程访问同一个共享变量时很容易出现并发问题，ThreadLocal 提供了线程本地变量，如果创建一个 ThreadLocal 变量。那么访问这个变量的每个线程都会有这个变量的一个本地**副本**，当多线程操作这个变量时，实际操作的是自己本地内存里面的变量，从而避免了线程安全问题。
```java
public static ThreadLocal<String> threadLocal = new ThreadLocal<String>();
threadLocal.set("Hello World");
threadLocal.remove();
```

## 3. 实现原理

可以看到 ThreadLocal有一个内部类 ThreadLocalMap，这是一个定制化的 Hashmap，可以储存一些键值对
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
另外，在 Thread 类内部，有两个 ThreadLocalMap 类型的属性：
```java
//Thread.java
ThreadLocal.ThreadLocalMap threadLocals = null;
ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
```
这两个变量就是本文的重点了，其实每个线程的本地变量不是存放在 ThreadLocal 实例里面的，而是存放在 Thread.threadLocals 里面，也就是说，ThreadLocal 类型的本地变量存放在具体的线程内存空间中，ThreadLocal 只是一个工具壳，封装了操作这些副本变量的方法。

下面分析一些 ThreadLocal 的 set、get 以及 remove 方法的实现逻辑
- set 方法
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
set的逻辑很简单，获取当前线程内部的 threadLocals，ThreadLocal 对象作为key，变量值作为 value，保存到 threadLocals 里面。