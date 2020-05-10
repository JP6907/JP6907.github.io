---
title: Java 并发编程 之 ABA 问题
subtitle: ABA 问题
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - java
  - 编程语言
  - 并发
categories:
  - java
date: 2019-09-27 16:16:56
---


# Java 并发编程 之 ABA 问题
## 1. 基本的ABA问题
&emsp;在CAS算法中，需要取出内存中某时刻的数据(由用户完成)，在下一时刻比较并交换(CPU保证原子操作)，这个时间差会导致数据的变化。假设有以下顺序事件：
1. 线程1从内存位置V中取出A
2. 线程2从内存位置V中取出A
3. 线程2进行了写操作，将B写入内存位置V
4. 线程2将A再次写入内存位置V
5. 线程1进行CAS操作，发现V中仍然是A，交换成功

&emsp;尽管线程1的CAS操作成功，但线程1并不知道内存位置V的数据发生过改变。

## 2. ABA问题模拟
```java
public class ABADemo {
    
    private static AtomicReference<Integer> atomicReference = new AtomicReference<Integer>(100);

    public static void main(String[] args) {
        new Thread(() -> {
            atomicReference.compareAndSet(100, 101);
            atomicReference.compareAndSet(101, 100);
        },"t1").start();
        
        new Thread(() -> {
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(atomicReference.compareAndSet(100, 2019) + "\t修改后的值:" + atomicReference.get());
        },"t2").start();
    }
}
```
&emsp;初始值为100，线程t1将100改成101，然后又将101改回100，线程t2先睡眠1秒，等待t1操作完成，然后t2线程将值改成2019，可以看到，线程2修改成功:
```
true    修改后的值:2019
```

## 3. ABA问题的严重后果
&emsp;上面的Demo不会产生太大的影响，但有时候，ABA造成的后果很严重。我们再来看一下另外一个例子，在《Java并发实战》一书的第15章中有一个用原子变量实现的并发栈，代码如下：
```java
public class Node {
	public final String item;
	public Node next;
	public Node(String item){
		this.item = item;
	}
}
public class ConcurrentStack {
	AtomicReference<Node> top = new AtomicReference<Node>();
	public void push(String item){
		Node newTop = new Node(item);
		Node oldTop;
		do{
			oldTop = top.get();
			newTop.next = oldTop;
		}
		while(!top.compareAndSet(oldTop, newTop));
	}
	public String pop(){
		Node newTop;
		Node oldTop;
		do{
			oldTop = top.get();
			if(oldTop == null){
				return null;
			}
			newTop = oldTop.next;
		}
		while(!top.compareAndSet(oldTop, newTop));
		return oldTop.item;
	}
}
```
&emsp;这个例子并不会引发ABA问题，至于为什么不会，后面再讲解，下面将并发栈的例子修改一下，看看ABA会造成什么问题：
```java
public class Node {
	public final String item;
	public Node next;
	public Node(String item){
		this.item = item;
	}
}
public class ConcurrentStack {
	AtomicReference<Node> top = new AtomicReference<Node>();
	public void push(Node node){
		Node oldTop;
		do{
			oldTop = top.get();
			node.next = oldTop;
		}
		while(!top.compareAndSet(oldTop, node));
	}
	public Node pop(int time){
		Node newTop;
		Node oldTop;
		do{
			oldTop = top.get();
			if(oldTop == null){
				return null;
			}
			newTop = oldTop.next;
			TimeUnit.SECONDS.sleep(time);
		}
		while(!top.compareAndSet(oldTop, newTop));
		return oldTop;
	}
}
```
&emsp;这里只是简单做了一点修改：
1. push方法：原来是使用内容构造Node，现在直接传入Node，这样就符合了“在算法中的节点可以被循环使用”这个要求；
2. pop方法的sleep，这是模拟线程的执行情况，以便观察结果；

&emsp;下面是main方法：
```java
public static void main(String[] args) throws InterruptedException {
        ConcurrentStack stack = new ConcurrentStack();
        stack.push(new Node("B"));
        stack.push(new Node("A"));

        //让B出栈
        Thread threadA = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    stack.pop(3);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });

        //先然NodeA和NodeB出栈，然后让NodeD，NodeC，NodeA入栈（NodeA在栈顶）
        Thread threadB = new Thread(new Runnable() {
            @Override
            public void run() {
                Node A = null;
                try {
                    A = stack.pop(0);
                    stack.pop(0);
                    stack.push(new Node("D"));
                    stack.push(new Node("C"));
                    stack.push(A);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });

        threadA.start();
        threadB.start();

        //按照想法，执行之后stack里面应该会有CD
        // 但是实际执行之后，stack里面之后B
        threadA.join();
        threadB.join();
        Node top = stack.pop(0); //第一次有数据
        if(top!=null) 
            System.out.println(top.item); 
        else
            System.out.println("the stack is empty");
        top = stack.pop(0); //第二次没数据
        if(top!=null)
            System.out.println(top.item);
        else
            System.out.println("the stack is empty");
    }
```
&emsp;按照想法，执行之后stack里面应该会有CD，但是实际执行之后，stack里面只有B。

&emsp;我们来分析一下这个过程。

![ABA-1](https://gitee.com/JP6907/Pic/raw/master/java/ABA-1.png)

&emsp;线程A调用
```java
oldTop = top.get();
if(oldTop == null){
	return null;
}
newTop = oldTop.next;
```
&emsp;此时 oldTop 对应A，newTop对应B，接下来准备调用 compareAndSet(oldTop, newTop) 将栈顶设置为B，但在这之前 sleep 了3秒，此时A还未真正出栈。
&emsp;在线程A sleep 的时候，线程B介入，将A、B出栈，再pushD、C、A，此时堆栈结构如下图，而对象B此时处于游离状态，栈顶仍然为NodeA：

![ABA-2](https://gitee.com/JP6907/Pic/raw/master/java/ABA-2.png)

&emsp;此时轮到线程A执行CAS操作，检测发现栈顶仍为A，所以CAS成功，栈顶变为B，但实际上B.next为null，所以此时的情况变为：

![ABA-3](https://gitee.com/JP6907/Pic/raw/master/java/ABA-3.png)

&emsp;所以这时候栈中只有B一个元素，C和D组成的链表不再存在于堆栈中，平白无故就把C、D丢掉了。以上就是由于ABA问题带来的隐患。
&emsp;我们再回过头来看一下修改之前的并发栈例子，push的时候传入的是一个item，在push函数内部会根据item构造Node，compareAndSet函数比较的是Node，对于新的Node，compareAndSet会失败，而修改之后的例子里面，push函数传入的是Node，线程B将NodeA出栈，后面又将NodeA重新入栈，重复利用原来的Node，compareAndSet才能执行成功，这也导致了ABA问题的产生。

## 4. ABA问题的解决
&emsp;要解决ABA问题，可以增加一个版本号，当内存位置V的值每次被修改后，版本号都加1。Java中提供了AtomicStampedReference和AtomicMarkableReference来解决ABA问题。

### 4.1 AtomicStampedReference
&emsp;AtomicStampedReference内部维护了对象值和版本号，在创建AtomicStampedReference对象时，需要传入初始值和初始版本号，当AtomicStampedReference设置对象值时，对象值以及状态戳都必须满足期望值，写入才会成功。
```java
public class ABADemo {
    
    private static AtomicStampedReference<Integer> atomicStampedReference = new AtomicStampedReference<Integer>(100,1);

    public static void main(String[] args) {
        new Thread(() -> {
            System.out.println("t1拿到的初始版本号:" + atomicStampedReference.getStamp());
            
            //睡眠1秒，是为了让t2线程也拿到同样的初始版本号
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            atomicStampedReference.compareAndSet(100, 101,atomicStampedReference.getStamp(),atomicStampedReference.getStamp()+1);
            atomicStampedReference.compareAndSet(101, 100,atomicStampedReference.getStamp(),atomicStampedReference.getStamp()+1);
        },"t1").start();
        
        new Thread(() -> {
            int stamp = atomicStampedReference.getStamp();
            System.out.println("t2拿到的初始版本号:" + stamp);
            
            //睡眠3秒，是为了让t1线程完成ABA操作
            try {
                TimeUnit.SECONDS.sleep(3);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("最新版本号:" + atomicStampedReference.getStamp());
            System.out.println(atomicStampedReference.compareAndSet(100, 2019,stamp,atomicStampedReference.getStamp() + 1) + "\t当前 值:" + atomicStampedReference.getReference());
        },"t2").start();
    }
}
```
&emsp;输出结果：
```
t1拿到的初始版本号:1
t2拿到的初始版本号:1
最新版本号:3
false	当前 值:100
```
&emsp;分析过程：
1. 初始值100，初始版本号1
2. 线程t1和t2拿到一样的初始版本号
3. 线程t1完成ABA操作，版本号递增到3
4. 线程t2完成CAS操作，最新版本号已经变成3，跟线程t2之前拿到的版本号1不相等，操作失败

### 4.2 AtomicMarkableReference
&emsp;AtomicStampedReference可以给引用加上版本号，追踪引用的整个变化过程，如：A -> B -> C -> D - > A，通过AtomicStampedReference，我们可以知道，引用变量中途被更改了3次。但是，有时候，我们并不关心引用变量更改了几次，只是单纯的关心是否更改过，所以就有了AtomicMarkableReference。AtomicMarkableReference的唯一区别就是不再用int标识引用，而是使用boolean变量——表示引用变量是否被更改过。
```java
public class ABADemo2 {

    private static AtomicMarkableReference<Integer> atomicMarkableReference = new AtomicMarkableReference<Integer>(100,false);

    public static void main(String[] args) {
        new Thread(() -> {
            System.out.println("t1版本号是否被更改:" + atomicMarkableReference.isMarked());

            //睡眠1秒，是为了让t2线程也拿到同样的初始版本号
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            atomicMarkableReference.compareAndSet(100, 101,atomicMarkableReference.isMarked(),true);
            atomicMarkableReference.compareAndSet(101, 100,atomicMarkableReference.isMarked(),true);
        },"t1").start();

        new Thread(() -> {
            boolean isMarked = atomicMarkableReference.isMarked();
            System.out.println("t2版本号是否被更改:" + isMarked);

            //睡眠3秒，是为了让t1线程完成ABA操作
            try {
                TimeUnit.SECONDS.sleep(3);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("是否更改过:" + atomicMarkableReference.isMarked());
            System.out.println(atomicMarkableReference.compareAndSet(100, 2019,isMarked,true) + "\t当前 值:" + atomicMarkableReference.getReference());
        },"t2").start();
    }
}
```
&emsp;输出结果：
```
t1版本号是否被更改:false
t2版本号是否被更改:false
是否更改过:true
false	当前 值:100
```
&emsp;分析过程：
1. 初始值100，初始版本号未被修改 false
2. 线程t1和t2拿到一样的初始版本号都未被修改 false
3. 线程t1完成ABA操作，版本号被修改 true
4. 线程t2完成CAS操作，版本号已经变成true，跟线程t2之前拿到的版本号false不相等，操作失败



&nbsp;
&nbsp;
> 参考：
https://www.cnblogs.com/lmj612/p/10836912.html
https://www.jb51.net/article/132977.htm
https://www.cnblogs.com/john8169/p/9780574.html