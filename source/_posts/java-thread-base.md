---
title: Java 并发编程 之 线程基础
subtitle: 线程基础
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - java
  - 编程语言
  - 并发
categories:
  - java
date: 2019-10-13 10:19:45
---



## 1. 线程的创建与运行
&emsp;Java 中由三种线程创建方式，分别为实现 Runnable 接口的run方法，继承 Thread 类并重写run方法，使用 FutureTask 方式。
### 1.1 Thread 方式
示例
```java
import java.util.concurrent.Callable;
import java.util.concurrent.FutureTask;

/**
 * 线程创建的三种方式
 */
public class ThreadCreate {

    public static class MyThread extends Thread{

        @Override
        public void run() {
            System.out.println("I am a child thread");
        }
    }
    public static void main(String[] args){
        MyThread thread = new MyThread();
        thread.start();
    }
}

```

### 1.2 Runnable 方式
示例
```java
import java.util.concurrent.Callable;
import java.util.concurrent.FutureTask;

/**
 * 线程创建的三种方式
 */
public class ThreadCreate {

    public static class RunableTask implements Runnable{

        @Override
        public void run() {
            System.out.println("I am a child thread");
        }
    }
    public static void main(String[] args){
        RunableTask task = new RunableTask();
        new Thread(task).start();
        new Thread(task).start();
    }
}

```

### 1.3 FutureTask 方式
示例
```java
import java.util.concurrent.Callable;
import java.util.concurrent.FutureTask;

/**
 * 线程创建的三种方式
 */
public class ThreadCreate {

    public static class CallerTask implements Callable<String>{
        @Override
        public String call() throws Exception {
            //返回值
            return "hello";
        }
    }
    public static void main(String[] args){

        FutureTask<String> futureTask = new FutureTask<>(new CallerTask());
        new Thread(futureTask).start();
        try{
            //等待任务结束，并返回结果
            String result = futureTask.get();
            System.out.println(result);
        }catch (Exception e){
            e.printStackTrace();
        }
    }
}

```

## 2. 线程通知与等待
&emsp;Object 类包含了通知和等待系列函数，由于继承机制，Java 中所有类都带有这类函数。

### 2.1 wait 函数
&emsp;当一个线程调用一个共享变量的wait方法时，该线程会被阻塞挂起，直到发生以下情况：
- 其它线程调用该共享变量的 notify 方法 或 notifyAll 方法；
- 其它线程调用了该线程的 interrupt 方法，该线程抛出 InterruptedException 异常返回；

&emsp;需要注意的是，调用 wait 方法需要事**先获取该对象的监视器锁**，否则会抛出 IllegalMonitorStateException 异常。
&emsp;获取监视器锁的方法：
(1) 执行 synchronized 同步代码块，使用该共享变量作为参数
```java
synchronized(共享变量){

}
```
(2) 调用该共享变量的方法，并且该方法使用了 synchronized 修饰
```java
synchronized void add(int a,int b){

}
```
&emsp;另外需要注意的是，一个线程可以从挂起状态变为运行状态（也就是被唤醒），即使其它线程没有调用 notify 方法或 notifyAll 方法，或者被中断，或者等待超时，这就是所谓的**虚假唤醒**。虽然虚假唤醒在实际中很少发生，但要防范于未然，做法就是不停去测试该线程被唤醒的条件是否满足，不满足则继续等待。
```java
synchronized(obi){
    while(条件不满足)
        obj.wait();
}
```
&emsp;线程调用 wait 方法之后，会释放锁该变量的监视器锁，然后阻塞一直等待到被唤醒，并且重新获得监视器锁才会返回。
&emsp;需要**注意**的是，调用 wait 方法只会释放当前共享变量上的锁，如果当前线程还持有其它共享变量的锁，则这些锁是**不会被释放**的。
```

```

当一个线程调用wait被阻塞后，其它线程中断了该线程，则该线程会抛出 InterruptedException 异常并返回。

wait(long timeout)和 wait(long timeout,int nanos)相比 wait 函数多了超时参数，wait 方法就等于wait(0),函数会因为超时而返回，不会一直等待。



### 2.2 notify 函数
&emsp;一个线程调用 notify 方法，会唤醒一个在该共享变量上调用 wait 方法后被挂起的线程。一个共享变量上可能会有多个线程在等待，具体唤醒哪个等待的线程是随机的。此外，被唤醒的线程必须在重新获取该共享对象的监视器锁后才会返回，因为被唤醒的线程可能还需要和其它线程竞争该监视器锁，所以被唤醒的线程不一定能够继续执行。
&emsp;notifyAll 方法则会唤醒所有在该共享变量上调用wait被阻塞的线程。
&emsp;类似wait函数，只有当前线程**获取到了共享变量的监视器锁才能调用notify方法**。
```java
public class WaitTest {
    private static volatile Object resourceA = new Object();
    private static volatile Object resourceB = new Object();

    public static class RunableA implements Runnable{

        @Override
        public void run() {
            try {
                synchronized (resourceA) {
                    System.out.println("Thread A get resourceA lock");
                    synchronized (resourceB) {
                        System.out.println("Thread A get resourceB lock");

                        //线程A阻塞并释放 resourceA 上的锁
                        resourceA.wait();
                    }
                }
            }catch (Exception e){
                e.printStackTrace();
            }
        }
    }

    public static class RunableB implements Runnable{

        @Override
        public void run() {
            try{
                //休眠，保证ThreadA先拿到锁
                Thread.sleep(1000);
                synchronized (resourceA) {
                    System.out.println("Thread B get resourceA lock");
                    System.out.println("Thread B try to get resourceB lock...");
                    synchronized (resourceB) {
                        System.out.println("Thread B get resourceB lock");
                        //阻塞并释放 resourceA 上的锁
                        resourceA.wait();
                    }
                }
            }catch (Exception e){

            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Thread threadA = new Thread(new RunableA());
        Thread threadB = new Thread(new RunableB());
        threadA.start();
        threadB.start();

        threadA.join();
        threadB.join();

        System.out.println("main over");
    }
}
```
&emsp;上述例子threadA在调用wait之后释放了resourceA上的锁，但是没有释放resourceB上的锁，threadA阻塞在 resourceA.wait()处，threadB 阻塞在 synchronized (resourceB) 处。


## 3. join 方法
&emsp;Thread类的join方法可以使得当前线程被阻塞，等待某个线程结束再继续往下执行，join是无参且返回值为void的方法。另外，线程A调用线程B的join方法后被阻塞，当其它线程调用了线程A的interrupt方法中断了线程A，线程A会抛出 InterruptedException 异常而返回。

## 4. sleep 方法
&emsp;sleep是Thread类的静态方法，当一个线程调用 Thread.sleep 后，调用线程会暂时让出指定时间的执行权，在这段时间内不参与CPU调度。
&emsp;当时该线程持有的监视器资源，比如**锁是保持持有不让出的**。在睡眠期间其它线程调用了该线程的 interrupt 方法中断该线程，则该线程会抛出 InterruptedException 异常而返回。

## 5. yield 方法
&emsp;yield是Thread的静态方法，当一个线程调用 yield 方法时，当前线程会让出 CPU 使用权，然后处于就绪状态，线程调度器会从线程就绪队列中获取一个线程优先级最高的线程，当然也可能会调度到刚刚让出 CPU 的那个线程来获取 CPU 执行权。

## 6. 线程中断
&emsp;Java中的线程中断是一种线程间协作模式，实际是通过**设置线程的中断标志，并不能直接终止该线程的执行，而需要被中断线程根据中断状态自行处理**。
&emsp;关于中断有几个方法需要区分：
- void interrupt()：该方法会设置被调用线程的中断标志为true，实际并不会直接中断该线程。如果线程A因为调用 wait、join 或 sleep 方法而被阻塞，其它线程调用线程A的interrupt方法会导致线程A在调用这些方法的地方抛出 InterruptedException 异常而返回。
```java
//Thread.java
public void interrupt() {
        //检查权限
        if (this != Thread.currentThread())
            checkAccess();

        synchronized (blockerLock) {
            Interruptible b = blocker;
            if (b != null) {
                //设置中断标志
                interrupt0();           // Just to set the interrupt flag
                b.interrupt(this);
                return;
            }
        }
        interrupt0();
    }
private native void interrupt0();    
```
- boolean isInterrupted()：检测**当前被调用线程**是否被中断，返回中断标志状态
```java
//Thread.java
public boolean isInterrupted() {
        return isInterrupted(false);
    }
private native boolean isInterrupted(boolean ClearInterrupted);
```
- boolean interrupted()：静态方法，检测**当前线程**是否被中断，与 isInterrupted 不同的是，该方法如果发现当前线程被中断，则会清除中断标志
```java
//Thread.java
public static boolean interrupted() {
        return currentThread().isInterrupted(true);
    }
```

下面看一个线程使用 Interrupted 优雅退出的例子：
```java
//Thread.java
public void run(){
    try{
        while(!Thread.currentThread().isInterrupted()&& more work to do){
            //do more work
        }
    }catch(InterruptedException e){
        // thread was interrupted during sleep or wait
    }finally{
        //cleanup,if required
    }
}
```
&emsp;最后使用一个例子区分总结一下这三个方法的区别：
```java
public class InterruptTest {

    public static void main(String[] args) throws InterruptedException {

        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                for(;;){

                }
//                while(!Thread.interrupted()){  //中断标志为true时会退出循环，并清除中断标志
//
//                }
//                System.out.println("in isInterrupted:" + Thread.currentThread().isInterrupted());
            }
        });

        //启动
        thread.start();
        //设置中断标志
        thread.interrupt();
        //获取中断标志
        System.out.println("isInterrupted:" + thread.isInterrupted()); //判断thread
        //获取中断标志并重置
        //获取的是当前线程(主线程)，不是thread线程
        System.out.println("isInterrupted:" + thread.interrupted());   //判断currentThread

        System.out.println("isInterrupted:" + Thread.interrupted());   //判断currentThread

        System.out.println("isInterrupted:" + thread.isInterrupted());  //判断thread

        thread.join();

        System.out.println("main thread is over");
    }
}
```
运行结果：
```
isInterrupted:true
isInterrupted:false
isInterrupted:false
isInterrupted:true
```
如果上述例子的for空循环替换成注释部分，可以验证 interrupted 函数是否会真的清除标志位，执行结果为：
```
isInterrupted:true
isInterrupted:false
isInterrupted:false
isInterrupted:true  //可能为true或false，这里运行时可能在标志位清除之前或之后
in isInterrupted:false //标志位被清除
main thread is over
```


## 7. 守护线程与用户线程
&emsp;Java中的线程分为两类，分别为 daemon线程（守护线程）和 user线程（用户线程）。JVM启动时会调用main函数，main函数所在的线程就是一个用户线程，其实在JVM内部同时启动了好多守护线程，比如垃圾回收线程。守护线程和非守护线程的区别在于：当最后一个非守护线程结束时，JVM会正常退出，而不管当前是否有守护线程，也就是说守护线程是否结束不会影响JVM的退出，只要有一个用户线程还没结束，JVM就不会退出。
注意：
- 正在运行的常规线程不应该设置为守护线程；
- thread.setDaemon(true)必须在thread.start()之前设置，否则会跑出一个IllegalThreadStateException异常；
- 在Daemon线程中产生的新线程也是Daemon的（这里要和linux的区分，linux中守护进程fork()出来的子进程不再是守护进程）；
- 根据自己的场景使用（在应用中，有可能你的Daemon Thread还没来的及进行操作时，虚拟机可能已经退出了）；

&nbsp;
> 参考：
《Java 并发编程之美》