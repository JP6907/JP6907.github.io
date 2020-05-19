---
title: synchronized 的实现原理
catalog: true
subtitle: synchronized 的实现原理
header-img: /img/article_header/article_header.png
tags:
  - java
  - 编程语言
  - 并发
categories:
  - java
---

# synchronized 的实现原理

## 1. synchronized 常见三种使用方法：　
- 1）普通同步方法，锁是当前实例；
- 2）静态同步方法，锁是当前类的Class实例，Class数据存在永久代中，是该类的一个全局锁；
- 3）对于同步代码块，锁是synchronized括号里配置的对象。

Java中的每个对象都可以作为锁。当一个线程访问同步代码块时，需要首先获取锁，退出代码块或抛出异常时必须释放锁。


## 2. “锁”到底是什么？
首先通过源代码和反汇编代码研究锁的实现原理。

**1）同步代码块源代码：**
```java
public class SynchronizedBlock {
                public void method() {
                synchronized (this) {
                    System.out.println("...");
                }
                System.out.println("...");
                }
            }
```
字节码（javap -c）：
```
public class SynchronizedBlock {
  
              // Method descriptor #6 ()V
              // Stack: 1, Locals: 1
              public SynchronizedBlock();
 aload_0 [this]
 invokespecial java.lang.Object() [8]
 return
                  Line numbers:
                [pc: 0, line: 5]
                  Local variable table:
                [pc: 0, pc: 5] local: this index: 0 type: SynchronizedBlock
              
              // Method descriptor #6 ()V
              // Stack: 2, Locals: 2
              public void method();
 aload_0 [this]
 dup
 astore_1
 monitorenter    //在同步块开始位置插入monitorenter指令
 getstatic java.lang.System.out : java.io.PrintStream [15]
 ldc <String "..."> [21]
 invokevirtual java.io.PrintStream.println(java.lang.String) : void [23]
 aload_1
 monitorexit    //在同步块结束位置插入
 goto 20
 aload_1
 monitorexit    //在抛出异常位置释放锁
 athrow        //抛出异常指令
 getstatic java.lang.System.out : java.io.PrintStream [15]
 ldc <String "..."> [21]
 invokevirtual java.io.PrintStream.println(java.lang.String) : void [23]
 return
                  Exception Table:
                [pc: 4, pc: 14] -> 17 when : any
                [pc: 17, pc: 19] -> 17 when : any
                  Line numbers:
                [pc: 0, line: 7]
                [pc: 4, line: 8]
                [pc: 12, line: 7]
                [pc: 20, line: 10]
                [pc: 28, line: 11]
                  Local variable table:
                [pc: 0, pc: 29] local: this index: 0 type: SynchronizedBlock
                  Stack map table: number of frames 2
                [pc: 17, full, stack: {java.lang.Throwable}, locals: {SynchronizedBlock, SynchronizedBlock}]
                [pc: 20, chop 1 local(s)]
            }
```

**2）同步方法源代码：**
```java
public class SynchronizedMethod {
                    //普通同步方法
                    public synchronized void method1() {
                    System.out.println("...");
                    }
                    //静态同步方法
                    public synchronized static void method2() {
                    System.out.println("...");
                    }
                }
```
字节码（javap -v）：
```
public class SynchronizedMethod
              SourceFile: "SynchronizedMethod.java"
              minor version: 0
              major version: 51
              flags: ACC_PUBLIC, ACC_SUPER
            Constant pool:
               #1 = Class              #2             //  SynchronizedMethod
               #2 = Utf8               SynchronizedMethod
               #3 = Class              #4             //  java/lang/Object
               #4 = Utf8               java/lang/Object
               #5 = Utf8               <init>
               #6 = Utf8               ()V
               #7 = Utf8               Code
               #8 = Methodref          #3.#9          //  java/lang/Object."<init>":()V
               #9 = NameAndType        #5:#6          //  "<init>":()V
              #10 = Utf8               LineNumberTable
              #11 = Utf8               LocalVariableTable
              #12 = Utf8               this
              #13 = Utf8               LSynchronizedMethod;
              #14 = Utf8               method1
              #15 = Fieldref           #16.#18        //  java/lang/System.out:Ljava/io/PrintStream;
              #16 = Class              #17            //  java/lang/System
              #17 = Utf8               java/lang/System
              #18 = NameAndType        #19:#20        //  out:Ljava/io/PrintStream;
              #19 = Utf8               out
              #20 = Utf8               Ljava/io/PrintStream;
              #21 = String             #22            //  ...
              #22 = Utf8               ...
              #23 = Methodref          #24.#26        //  java/io/PrintStream.println:(Ljava/lang/String;)V
              #24 = Class              #25            //  java/io/PrintStream
              #25 = Utf8               java/io/PrintStream
              #26 = NameAndType        #27:#28        //  println:(Ljava/lang/String;)V
              #27 = Utf8               println
              #28 = Utf8               (Ljava/lang/String;)V
              #29 = Utf8               method2
              #30 = Utf8               SourceFile
              #31 = Utf8               SynchronizedMethod.java
            {
              public SynchronizedMethod();
                flags: ACC_PUBLIC
                Code:
                  stack=1, locals=1, args_size=1
                 0: aload_0       
                 1: invokespecial #8                  // Method java/lang/Object."<init>":()V
                 4: return        
                  LineNumberTable:
                line 3: 0
                  LocalVariableTable:
                Start  Length  Slot  Name   Signature
      5     0  this   LSynchronizedMethod;

              public synchronized void method1();
                flags: ACC_PUBLIC, ACC_SYNCHRONIZED        //同步方法增加了ACC_SYNCHRONIZED标志
                Code:
                  stack=2, locals=1, args_size=1
                 0: getstatic     #15                 // Field java/lang/System.out:Ljava/io/PrintStream;
                 3: ldc           #21                 // String ...
                 5: invokevirtual #23                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
                 8: return        
                  LineNumberTable:
                line 6: 0
                line 7: 8
                  LocalVariableTable:
                Start  Length  Slot  Name   Signature
      9     0  this   LSynchronizedMethod;

              public static synchronized void method2();
                flags: ACC_PUBLIC, ACC_STATIC, ACC_SYNCHRONIZED
                Code:
                  stack=2, locals=0, args_size=0
                 0: getstatic     #15                 // Field java/lang/System.out:Ljava/io/PrintStream;
                 3: ldc           #21                 // String ...
                 5: invokevirtual #23                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
                 8: return        
                  LineNumberTable:
                line 10: 0
                line 11: 8
                  LocalVariableTable:
                Start  Length  Slot  Name   Signature
            }
```
&emsp;通过反汇编代码可以观察到：
&emsp;同步代码块是使用 **MonitorEnter** 和 **MoniterExit** 指令实现的，在编译时，MonitorEnter指令被插入到同步代码块的开始位置，MoniterExit指令被插入到同步代码块的结束位置和异常位置。任何对象都有一个Monitor与之关联，当Monitor被持有后将处于锁定状态。MonitorEnter指令会尝试获取Monitor的持有权，即尝试获取锁。
&emsp;同步方法依赖flags标志 **ACC_SYNCHRONIZED** 实现，字节码中没有具体的逻辑，可能需要查看JVM的底层实现（同步方法也可以通过Monitor指令实现）。ACC_SYNCHRONIZED标志表示方法为同步方法，如果为非静态方法（没有 **ACC_STATIC** 标志），使用调用该方法的对象作为锁对象；如果为静态方法（有ACC_STATIC标志），使用该方法所属的Class类在JVM的内部对象表示Klass作为锁对象。

&emsp;下面是摘自《Java虚拟机规范》的话：
&emsp;*Java虚拟机可以支持方法级的同步和方法内部一段指令序列的同步，这两种同步结构都是使用管程（Monitor）来支持的。*
&emsp;*方法级的同步是隐式，即无需通过字节码指令来控制的，它实现在方法调用和返回操作之中。虚拟机可以从方法常量池中的方法表结构中的ACC_SYNCHRONIZED访问标志区分一个方法是否同步方法。当方法调用时，调用指令将会检查方法的ACC_SYNCHRONIZED访问标志是否被设置，如果设置了，执行线程将先持有管程，然后再执行方法，最后在方法完成（无论是正常完成还是非正常完成）时释放管程。在方法执行期间，执行线程持有了管程，其他任何线程都无法再获得同一个管程。如果一个同步方法执行期间抛出了异常，并且在方法内部无法处理此异常，那这个同步方法所持有的管程将在异常抛到同步方法之外时自动释放。*
&emsp;*同步一段指令集序列通常是由Java语言中的synchronized块来表示的，Java虚拟机的指令集中有monitorenter和monitorexit两条指令来支持synchronized关键字的语义，正确实现synchronized关键字需要编译器与Java虚拟机两者协作支持。*
&emsp;*Java虚拟机中的同步（Synchronization）基于进入和退出管程（Monitor）对象实现。无论是显式同步（有明确的monitorenter和monitorexit指令）还是隐式同步（依赖方法调用和返回指令实现的）都是如此。*
&emsp;*编译器必须确保无论方法通过何种方式完成，方法中调用过的每条monitorenter指令都必须有执行其对应monitorexit指令，而无论这个方法是正常结束还是异常结束。为了保证在方法异常完成时monitorenter和monitorexit指令依然可以正确配对执行，编译器会自动产生一个异常处理器，这个异常处理器声明可处理所有的异常，它的目的就是用来执行monitorexit指令。*

 
> 那么Monitor到底是什么？一个作为锁的对象是什么样的工作机制？（Java对象头和Monitor）
 
## 3. Java对象头
通常说的synchronized在方法或块上加锁，这里的锁就是对象锁（当然也可以在类上面），或者叫重量锁，在JVM中又叫对象监视器（Monitor），就是对象来监视线程的互斥。
![](https://gitee.com/JP6907/Pic/raw/master/img/objectHead.png)
&emsp;对象头含有三部分：Mark Word（存储对象自身运行时数据）、Class Metadata Address（存储类元数据的指针）、Array length（数组长度，只有数组类型才有）。
&emsp;重点在Mark Word部分，Mark Word数据结构被设计成非固定的，会随着对象的不同状态而变化，如下表所示。
<escape>
<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;}
.tg td{font-family:Arial, sans-serif;font-size:14px;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:black;}
.tg th{font-family:Arial, sans-serif;font-size:14px;font-weight:normal;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:black;}
.tg .tg-c3ow{border-color:inherit;text-align:center;vertical-align:top}
</style>
<table class="tg">
  <tr>
    <th class="tg-c3ow">锁状态</th>
    <th class="tg-c3ow">25bit</th>
    <th class="tg-c3ow">25bit</th>
    <th class="tg-c3ow">4bit</th>
    <th class="tg-c3ow">1bit是否是偏向锁</th>
    <th class="tg-c3ow">2bit锁标志位</th>
  </tr>
  <tr>
    <td class="tg-c3ow">无锁状态</td>
    <td class="tg-c3ow">hashCode</td>
    <td class="tg-c3ow">hashCode</td>
    <td class="tg-c3ow">age</td>
    <td class="tg-c3ow">0</td>
    <td class="tg-c3ow">01</td>
  </tr>
  <tr>
    <td class="tg-c3ow">轻量级锁</td>
    <td class="tg-c3ow" colspan="4">执行栈中锁记录的指针</td>
    <td class="tg-c3ow">00</td>
  </tr>
  <tr>
    <td class="tg-c3ow">重量级锁</td>
    <td class="tg-c3ow" colspan="4">执行栈中锁记录的指针</td>
    <td class="tg-c3ow">10</td>
  </tr>
  <tr>
    <td class="tg-c3ow">GC标记</td>
    <td class="tg-c3ow" colspan="4">空</td>
    <td class="tg-c3ow">11</td>
  </tr>
  <tr>
    <td class="tg-c3ow">偏向锁</td>
    <td class="tg-c3ow">线程ID</td>
    <td class="tg-c3ow">Epoch</td>
    <td class="tg-c3ow">age</td>
    <td class="tg-c3ow">1</td>
    <td class="tg-c3ow">01</td>
  </tr>
</table>
</escape>

其中Tag的2bit用来显示锁类型。通常我们说synchronized的对象锁，就是这里Tag=10时的monitor对象，这里的Monitor address就是这个monitor对象（就是重量锁）的地址。

当多个线程同时请求synchronized方法或块时，monitor会设置几个虚拟逻辑数据结构来管理这些多线程。下图是简化了的管理结构。
![](https://gitee.com/JP6907/Pic/raw/master/img/monitorManager.png)


&emsp;锁的级别从低到高：无锁、偏向锁、轻量级锁、重量级锁。

## 4. Monitor
&emsp;Monitor可以理解为一种同步工具，也可理解为一种同步机制，常常被描述为一个Java对象。
- 1）互斥：一个Monitor在一个时刻只能被一个线程持有，即Monitor中的所有方法都是互斥的。
- 2）signal机制：如果条件变量不满足，允许一个正在持有Monitor的线程暂时释放持有权，当条件变量满足时，当前线程可以唤醒正在等待该条件变量的线程，然后重新获取Monitor的持有权。

&emsp;所有的Java对象是天生的Monitor，每一个Java对象都有成为Monitor的潜质，因为在Java的设计中 ，每一个Java对象自打娘胎里出来就带了一把看不见的锁，它叫做内部锁或者**Monitor锁**。
&emsp;**Monitor的本质是依赖于底层操作系统的Mutex Lock实现，操作系统实现线程之间的切换需要从用户态到内核态的转换，成本非常高。**

synchronized的同步过程主要通过成对的MoniterEnter和MonitorEixt指令来实现:
1. MonitorEnter指令：插入在同步代码块的开始位置，当代码执行到该指令时，将会尝试获取该对象Monitor的所有权，即尝试获得该对象的锁；
2. MonitorExit指令：插入在方法结束处和异常处，JVM保证每个MonitorEnter必须有对应的MonitorExit；

　　
## 5. 锁的优化
### 5.1 重量级锁
  重量级锁基于Monitor实现，成本高。

### 5.2 轻量级锁
当多线程环境进入synchronized区域的线程没竞争时，JVM并不会马上创建对象锁，而是用轻量锁或偏向锁。

不过需要明确的是，轻量锁和偏向锁，都不能代替重量锁，只不过是在没有多线程竞争时，没必要用重量锁而无畏的消耗资源。但是一旦出现了多线程竞争时，synchronized区域的轻量锁或偏向锁都会立即升级为重量锁。

轻量锁或偏向锁使用的条件是进入synchronized区域时没有其他任何其他线程在使用。

这时线程t访问对象的synchronized区域时，对象头的标志位Tag状态为01，以及还有1位的偏向信息用于记录这个对象是否可用偏向锁。然后t在对象上申请轻量锁时，若偏向信息为0，表明当前对象还未加锁，或加过偏向锁（加过，注意是加过偏向锁的对象只能被同样的线程加锁，如果不同的线程想要获取锁，需要先将偏向锁升级为轻量锁，稍后会讲到），在判断对当前对象确实没有被任何其他线程锁住后，即可以在该对象上加轻量锁。

加轻量锁的过程很简单：在当前线程的栈帧（stack frame）中生成一个锁记录（lock record），这个锁记录比前面说的那个对象锁（管理线程队列的monitor）简单多了，它只是对象头的一个拷贝。然后把对象头里的tag改成00，并把这个栈帧里的lock record地址放入对象头里。若操作成功，那就完成了轻量锁操作。如果不成功，说明有线程在竞争，则需要在当前对象上生成重量锁来进行多线程同步，然后将Tag状态改为10，并生成Monitor对象（重量锁对象），对象头里也会放入Monitor对象的地址。最后将当前线程t排队队列中。

轻量锁的解锁过程也很简单就是把栈帧里刚才的那个lock record拷贝到对象头里，若替换成功，则解锁完成，若替换不成功，表示在当前线程持有锁的这段时间内，其他线程也竞争过锁，并且发生了锁升级为重量锁，这时需要去Monitor的等待队列中唤醒一个线程去重新竞争锁。

总结一下，获取轻量级锁的步骤：
1. 判断是否处于无锁状态，若是，则JVM在当前线程的栈帧中创建锁记录（Lock Record）空间，用于存放锁对象中的Mark Word的拷贝，官方称为Displaced Mark Word；否则执行步骤3）。
2. 当前线程尝试利用CAS将锁对象的Mark Word更新为指向锁记录的指针。如果更新成功意味着获取到锁，将锁标志位置为00，执行同步代码块；如果更新失败，执行步骤3）。
3. 判断锁对象的Mark Word是否指向当前线程的栈帧，若是说明当前线程已经获取了锁，执行同步代码，否则说明其他线程已经获取了该锁对象，执行步骤4）。
4. 当前线程尝试使用自旋来获取锁，自旋期间会不断的执行步骤1），直到获取到锁或自旋结束。因为自旋锁会消耗CPU，所以不能无限的自旋。如果自旋期间获取到锁（其他线程释放锁），执行同步块；否则锁膨胀为重量级锁，当前线程阻塞，等待持有锁的线程释放锁时的唤醒。

释放轻量级锁的步骤：
1. 从当前线程的栈帧中取出Displaced Mark Word存储的锁记录的内容。
2. 当前线程尝试使用CAS将锁记录内容更新到锁对象中的Mark Word中。如果更新成功，则释放锁成功，将锁标志位置为01无锁状态；否则，执行3）。
3. CAS更新失败，说明有其他线程尝试获取锁。需要释放锁并同时唤醒等待的线程。

轻量级锁一般在以下情况使用：
1. 关闭偏向锁
2. 由于多线程竞争导致的偏向锁升级为轻量级锁。


### 5.3 偏向锁
偏向锁是比轻量锁还轻量的锁机制。当synchronized区域长期都由同一个线程加锁、解锁时，jvm就用偏向锁来做，它的加锁解锁比轻量锁操作起来指令更加简化。不过一旦有其他线程使用synchronized区域，即使没有线程间竞争，也会把偏向锁升级为轻量锁，当然如果发生线程竞争就再升级为对象锁。
偏向锁的获取过程为，Mark Word中偏向锁的标识，确认是否为可偏向状态，以及偏向锁指示的线程ID是否为当前线程，第一个获取偏向锁的线程会将线程ID设置为当前线程。

    
偏向锁获取的步骤为：
1. 判断锁对象是否是偏向锁（即锁标志位为01，偏向锁位为1），若为偏向锁状态执行2）。
2. 判断锁对象的线程ID是否为当前线程的ID，如果是则说明已经获取到锁，执行代码块；否则执行3）。
3. 当前线程使用CAS更新锁对象的线程ID为当前线程ID。如果成功，获取到锁；否则执行4）
4. 当到达全局安全点，当前线程根据Mark Word中的线程ID通知持有锁的线程挂起，将锁对象Mark Word中的锁对象指针指向当前堆栈中最近的一个锁记录，偏向锁升级为轻量级锁，恢复被挂起的线程。

释放锁步骤：
偏向锁采用一种等到竞争出现时才释放锁的机制。当其他线程尝试竞争偏向锁时，当前线程才会释放释放锁，线程不会主动去释放偏向锁。偏向锁的撤销需要等待全局安全点。
1. 首先暂停持有偏向锁的线程。
2. 撤销偏向锁，恢复到无锁状态或轻量级锁状态。

### 5.4 自旋锁
因为线程阻塞后进入排队队列和唤醒都需要CPU从用户态转为核心态，尤其频繁的阻塞和唤醒对CPU来说是负荷很重的工作。同时统计发现，很多对象锁的锁定状态只会持续很短的一段时间，例如一个线程切换周期，这样的话在很短的时间内阻塞线程又很快唤醒线程显然不值得，所以引入了自旋锁概念。

所谓“自旋”，就monitor并不把线程阻塞放入排队队列，而是去执行一个无意义的循环，循环结束后看看是否锁已释放并直接进行竞争上岗步骤，如果竞争不到继续自旋循环，循环过程中线程的状态一直处于running状态。明显自旋锁使得synchronized的对象锁方式在线程之间引入了不公平。但是这样可以保证大吞吐率和执行效率。

不过虽然自旋锁方式省去了阻塞线程的时间和空间（队列的维护等）开销，但是长时间自旋也是很低效的。所以自旋的次数一般控制在一个范围内，例如10,50等，在超出这个范围后，线程就进入排队队列。

自适应自旋锁，就是自旋的次数是通过JVM在运行时收集的统计信息，动态调整自旋锁的自旋次数上界。


### 5.5 锁优化总结
偏向锁：对象头中Mark Word 为可偏向状态，第一个获取的线程会将偏向锁指向的线程ID设置为当前线程
轻量级锁：在当前线程栈帧中生成一个锁记录，并把锁记录写入对象头中，这种方式不需要通过队列阻塞线程的当时来管理锁，很简单
自旋锁：不阻塞线程，通过有限次的自旋重试获取锁，因为很多线程暂用锁的时间很短，这样能减少CPU的切换
重量级锁：通过队列的方式，阻塞线程，
**在所有的锁都启用的情况下线程进入临界区时会先去获取偏向锁，如果已经存在偏向锁了，则会尝试获取轻量级锁，如果以上两种都失败，则启用自旋锁，如果自旋也没有获取到锁，则使用重量级锁，没有获取到锁的线程阻塞挂起，直到持有锁的线程执行完同步块唤醒他们**；

偏向锁是在无锁争用的情况下使用的，也就是同步开在当前线程没有执行完之前，没有其它线程会执行该同步快，一旦有了第二个线程的争用，偏向锁就会升级为轻量级锁，一点有两个以上线程争用，就会升级为重量级锁；

如果线程争用激烈，那么应该禁用偏向锁。如果CAS获取偏向锁失败，则表示有竞争。当到达全局安全点（safepoint）时获得偏向锁的线程被挂起，偏向锁升级为轻量级锁，然后被阻塞在安全点的线程继续往下执行同步代码。（撤销偏向锁的时候会导致stop the word）


### 5.6 几种锁的对比
|          |                          优点                          |                     缺点                     |              适用情况              |
|:--------:|:------------------------------------------------------:|:--------------------------------------------:|:----------------------------------:|
|  偏向锁  | 加锁和解锁不需要额外的消耗，和执行非同步代码相差无几。 | 如果线程存在锁竞争，需要额外的锁撤销的消耗。 | 适用于只有一个线程访问同步块的情况 |
| 轻量级锁 |           竞争的线程不会阻塞，提高了响应速度           |      长时间得不到锁的线程使用自旋消耗CPU     |  追求响应速度。同步代码执行非常快  |
| 重量级锁 |            线程竞争不会使用自旋，不会消耗CPU           |       线程出现竞争时会阻塞，响应速度慢       |   追求吞吐量。同步代码执行时间长   |

&nbsp;
&nbsp;
>原文链接：https://www.cnblogs.com/zaizhoumo/p/7700161.html
https://www.cnblogs.com/YDDMAX/p/5658607.html

未读完：
https://www.cnblogs.com/aspirant/p/11470858.html