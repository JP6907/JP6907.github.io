---
title: volatile 的特殊规则和 long&&double 的非原子协定
subtitle: Java 一些变量类型的特殊规则
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - java
  - 编程语言
  - 并发
  - jvm
categories:
  - java
date: 2019-10-15 10:49:39
---


# 1. volatile 型变量的特殊规则
&emsp;关键字volatile可以说是Java虚拟机提供的最轻量级的同步机制，但是它并不容易完全被正确、完整地理解，以至于许多程序员都习惯不去使用它，遇到需要处理多线程数据竞争问题的时候一律使用synchronized来进行同步，了解volatile变量的语义对后面了解多线程操作的其他特性很有意义。
&emsp;当一个变量定义为volatile之后，它将具备两种特性，第一是保证此变量对所有线程的**可见性**，第二个语义是**禁止指令重排序优化**。

## 1.1 可见性

&emsp;这里的“可见性”是指当一条线程修改了这个变量的值，新值对于其他线程来说是可以立即得知的。 而普通变量不能做到这一点，普通变量的值在线程间传递均需要通过主内存来完成，例如，线程A修改一个普通变量的值，然后向主内存进行回写，另外一条线程B在线程A回写完成了之后再从主内存进行读取操作，新变量值才会对线程B可见。

&emsp;关于 volatile 变量的可见性，经常会被开发人员误解，认为以下描述成立：“volatile变量对所有线程是立即可见的，对volatile变量所有的写操作都能立刻反应到其他线程之中，换句话说，volatile变量在各个线程中是一致的，所以基于volatile变量的运算在并发下是安全的”。 这句话的论据部分并没有错，但是其论据并不能得出“基于volatile变量的运算在并发下是安全的”这个结论。volatile变量在各个线程的工作内存中不存在一致性问题（在各个线程的工作内存中，volatile变量也可以存在不一致的情况，但由于每次使用之前都要先刷新，执行引擎看不到不一致的情况，因此可以认为不存在一致性问题），但是**Java里面的运算并非原子操作，导致 volatile 变量的运算在并发下一样是不安全的**，我们可以通过一段简单的演示来说明原因，请看如下代码：
```java
public class VolatileTest {
    public static volatile int race = 0;

    public static void increase() {
        race++;
    }

    private static final int THREADS_COUNT = 20;

    public static void main(String[] args) {
        Thread[] threads = new Thread[THREADS_COUNT];
        for (int i = 0; i < THREADS_COUNT; i++) {
            threads[i] = new Thread(new Runnable() {
                @Override
                public void run() {
                    for (int i = 0; i < 10000; i++) {
                        increase();
                    }
                }
            });
            threads[i].start();
        }
        //等待所有累加线程都结束
        while (Thread.activeCount() > 1)
            Thread.yield();
        System.out.println(race);
    }
}
```
&emsp;这段代码发起了20个线程，每个线程对race变量进行10000次自增操作，如果这段代码能够正确并发的话，最后输出的结果应该是200000。 读者运行完这段代码之后，并不会获得期望的结果，而且会发现每次运行程序，输出的结果都不一样，都是一个小于200000的数字，这是为什么呢？

&emsp;问题就出现在自增运算“race++”之中，我们用Javap反编译这段代码后会得到代码清单，发现只有一行代码的 increase() 方法在Class文件中是由4条字节码指令构成的（return指令不是由race++产生的，这条指令可以不计算），从字节码层面上很容易就分析出并发失败的原因了：当 getstatic 指令把 race 的值取到操作栈顶时，volatile 关键字保证了 race 的值在此时是正确的，但是在执行 iconst_1、iadd 这些指令的时候，其他线程可能已经把 race 的值加大了，而在操作栈顶的值就变成了过期的数据，所以 putstatic 指令执行后就可能把较小的 race 值同步回主内存之中。下面是 VolatileTest 的字节码：
```
public static void increase（）；
Code：
Stack=2，Locals=0，Args_size=0
0：getstatic#13；//Field race：I
3：iconst_1
4：iadd
5：putstatic#13；//Field race：I
8：return
LineNumberTable：
line 14：0
line 15：8
```
&emsp;客观地说，在此使用字节码来分析并发问题，仍然是不严谨的，因为即使编译出来只有一条字节码指令，也并不意味执行这条指令就是一个原子操作。 一条字节码指令在解释执行时，解释器将要运行许多行代码才能实现它的语义，如果是编译执行，一条字节码指令也可能转化成若干条本地机器码指令，此处使用 -XX：+PrintAssembly 参数输出反汇编来分析会更加严谨一些，但考虑到读者阅读的方便，并且字节码已经能说明问题，所以此处使用字节码来分析。

**由于 volatile 变量只能保证可见性，如果符合以下两条规则才能保证原子性**：
- 1.运算结果并不依赖变量的当前值，或者能够确保只有单一的线程修改变量的值。 
- 2.变量不需要与其他的状态变量共同参与不变约束。

&emsp;如果在不符合以下两条规则的运算场景中，我们仍然要通过**加锁（使用synchronized或java.util.concurrent中的原子类）来保证原子性**。

&emsp;而在像如下的代码所示的场景就很适合使用volatile变量来控制并发，当shutdown（）方法被调用时，能保证所有线程中执行的doWork（）方法都立即停下来：
```java
volatile boolean shutdownRequested;

    public void shutdown() {
        shutdownRequested = true; //直接赋值，不依赖于当前值，且没有其他约束
    }

    public void doWork() {
        while (!shutdownRequested) {
        //do stuff
        }
    }
```

## 1.2 禁止指令重排序优化
&emsp;使用volatile变量的第二个语义是禁止指令重排序优化，普通的变量仅仅会保证在该方法的执行过程中所有依赖赋值结果的地方都能获取到正确的结果，而不能保证变量赋值操作的顺序与程序代码中的执行顺序一致。 因为在一个线程的方法执行过程中无法感知到这点，这也就是Java内存模型中描述的所谓的“**线程内表现为串行的语义**”（Within-Thread As-If-SerialSemantics）。 
上面的描述仍然不太容易理解，我们还是继续通过一个例子来看看为何指令重排序会干扰程序的并发执行，演示程序如代码如下所示:
```java
    Map configOptions;
    char[]configText;
    //此变量必须定义为volatile
    volatile boolean initialized=false;
    //假设以下代码在线程A中执行
    //模拟读取配置信息，当读取完成后将initialized设置为true以通知其他线程配置可用
    configOptions=new HashMap();
    configText=readConfigFile(fileName);
    processConfigOptions(configText,configOptions);
    initialized=true;

    //假设以下代码在线程B中执行
    //等待initialized为true，代表线程A已经把配置信息初始化完成
    while(！initialized){
        sleep();
    }
    //使用线程A中初始化好的配置信息
    doSomethingWithConfig();
```
&emsp;以上是一段伪代码，其中描述的场景十分常见，只是我们在处理配置文件时一般不会出现并发而已。 如果定义initialized变量时没有使用volatile修饰，就可能会由于指令重排序的优化，导致位于线程A中最后一句的代码“initialized=true”被提前执行（这里虽然使用Java作为伪代码，但所指的重排序优化是机器级的优化操作，提前执行是指这句话对应的汇编代码被提前执行），这样在线程B中使用配置信息的代码就可能出现错误，而volatile关键字则可以避免此类情况的发生。

&emsp;指令重排序是并发编程中最容易让开发人员产生疑惑的地方，除了上面伪代码的例子之外，再举一个可以实际操作运行的例子来分析volatile关键字是如何禁止指令重排序优化的。下面代码是一段标准的DCL单例代码，可以观察加入volatile和未加入volatile关键字时所生成汇编代码的差别。
```java
public class Singleton {
    private volatile static Singleton instance;

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }

    public static void main(String[] args) {
        Singleton.getInstance();
    }
}
```
&emsp;编译后，这段代码对instance变量赋值部分如下所示：
```
0x01a3de0f：mov$0x3375cdb0，%esi；……beb0cd75 33
；{oop（'Singleton'）}
0x01a3de14：mov%eax，0x150（%esi）；……89865001 0000
0x01a3de1a：shr$0x9，%esi；……c1ee09
0x01a3de1d：movb$0x0，0x1104800（%esi）；……c6860048 100100
0x01a3de24：lock addl$0x0，（%esp）；……f0830424 00
；*putstatic instance
；-
Singleton：getInstance@24
```

&emsp;通过对比发现，关键变化在于有volatile修饰的变量，赋值后（前面mov%eax，0x150（%esi）这句便是赋值操作）多执行了一个“**lock** addl ＄0x0，（%esp）”操作，这个操作相当于一个**内存屏障**（Memory Barrier或Memory Fence，指重排序时不能把后面的指令重排序到内存屏障之前的位置），只有一个CPU访问内存时，并不需要内存屏障；但如果有两个或更多CPU访问同一块内存，且其中有一个在观测另一个，就需要内存屏障来保证一致性了。 这句指令中的“addl ＄0x0，（%esp）”（把ESP寄存器的值加0）显然是一个空操作（采用这个空操作而不是空操作指令nop是因为IA32手册规定lock前缀不允许配合nop指令使用），关键在于lock前缀，查询IA32手册，它的作用是使得本CPU的Cache写入了内存，该写入动作也会引起别的CPU或者别的内核无效化（Invalidate）其Cache，这种操作相当于对Cache中的变量做了一次前面介绍Java内存模式中所说的“store和write”操作[2]。 所以通过这样一个空操作，可让前面volatile变量的修改对其他CPU立即可见。

&emsp;那为何说它禁止指令重排序呢？从硬件架构上讲，指令重排序是指CPU采用了允许将多条指令不按程序规定的顺序分开发送给各相应电路单元处理。 但并不是说指令任意重排，CPU需要能正确处理指令依赖情况以保障程序能得出正确的执行结果。 譬如指令1把地址A中的值加10，指令2把地址A中的值乘以2，指令3把地址B中的值减去3，这时指令1和指令2是有依赖的，它们之间的顺序不能重排——（A+10）*2与A*2+10显然不相等，但指令3可以重排到指令1、 2之前或者中间，只要保证CPU执行后面依赖到A、 B值的操作时能获取到正确的A和B值即可。 所以在本内CPU中，重排序看起来依然是有序的。 因此，lockaddl＄0x0，（%esp）指令把修改同步到内存时，意味着所有之前的操作都已经执行完成，这样便形成了“指令重排序无法越过内存屏障”的效果。

&emsp;解决了volatile的语义问题，再来看看在众多保障并发安全的工具中选用volatile的意义——它能让我们的代码比使用其他的同步工具更快吗？在某些情况下，volatile的同步机制的性能确实要优于锁（使用synchronized关键字或java.util.concurrent包里面的锁），但是由于虚拟机对锁实行的许多消除和优化，使得我们很难量化地认为volatile就会比synchronized快多少。 如果让volatile自己与自己比较，那可以确定一个原则：**volatile变量读操作的性能消耗与普通变量几乎没有什么差别，但是写操作则可能会慢一些，因为它需要在本地代码中插入许多内存屏障指令来保证处理器不发生乱序执行。 不过即便如此，大多数场景下volatile的总开销仍然要比锁低，我们在volatile与锁之中选择的唯一依据仅仅是volatile的语义能否满足使用场景的需求**。
 
> 关于volatile的正确使用可以参考[正确使用 Volatile 变量](https://blog.csdn.net/sted_zxz/article/details/76615583)。


# 2. long 和 double 的非原子协定
&emsp;Java内存模型要求lock、unlock、read、load、use、assign、store、write这8个操作都具有原子性。但是对于64位的数据类型（long和double），在模型中特别定义了一条相对宽松的规定：允许虚拟机将没有被 volatile 修饰的64位数据读写操作划分为2次32位的操作来进行，即**允许虚拟机实现选择可以不保证64位数据类型的read、load、store、write这4个操作的原子性**。这点就是所谓的 **long 和 double 的非原子协定**。

&emsp;如果有多个线程共享一个并未被 volatile 修饰的64位数据类型变量（long或double类型），并且同时对它们进行读取和修改操作，那么某些线程可能会读取到一个既非原值，也不是其他线程修改值的代表了“半个变量”的数值。

&emsp;不过这种读取到“半个变量”的情况非常罕见（目前商用的Java虚拟机中不会出现），因为Java内存模型虽然允许虚拟机不把long和double类型变量的读写实现成原子操作，但允许虚拟机选择把这些操作实现为具有原子性的操作，而且还“强烈建议”虚拟机这样实现。目前各种平台下的商用虚拟机几乎都选择把64位数据的读写操作作为原子操作来对待，因为我们再编写代码时无需把 long 和 double 类型变量专门声明为 volatile。


&nbsp;
> 参考：
《深入理解JAVA虚拟机》