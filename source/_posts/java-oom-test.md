---
title: 深入理解Java虚拟机 之 OutOfMemoryError 异常实战
subtitle: OutOfMemoryError 异常实战
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - java
  - 编程语言
  - jvm
categories:
  - java
date: 2019-10-16 11:25:11
---


# OutOfMemoryError 异常实战

&emsp;本节内容的目的有两个:
- 第一 ,通过代码验证Java虚拟机规范中描述的各个运行时区域存储的内容;
- 第二 ,希望读者在工作中遇到实际的内存溢出异常时 ,能根据异 常的信息快速判断是哪个区域的内存溢出,知道什么样的代码可能会导致这些区域内存溢出,以及出现这些异常后该如何处理。

## 1. Java 堆溢出
&emsp;Java堆用于存储对象实例,只要不断地创建对象,并且保证GC Roots到对象之间有可达路径来避免垃圾回收机制清除这些对象,那么在对象数量到达最大堆的容量限制后就会产生内存溢出异常。

&emsp;下面代码中限制Java堆的大小为20MB,不可扩展(将堆的最小值-Xms参数与最大值-Xmx参数设置为一样即可避免堆自动扩展),通过参数-XX:+HeapDumpOnOutOfMemoryError可以让虚拟机在出现内存溢出异常时Dump出当前的内存堆转储快照以便事后进行分析。
```java
/**
 * VM Args：-Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
 * @author zzm
 */
public class HeapOOM {

    static class OOMObject {
    }

    public static void main(String[] args) {
        List<OOMObject> list = new ArrayList<OOMObject>();

        while (true) {
            list.add(new OOMObject());
        }
    }
}
```
运行结果：
```
java.lang.OutOfMemoryError :Java heap space
Dumping heap to java_pid3404.hprof.
Heap dump file created[22045981 bytes in 0.663 secs]
```
&emsp;Java堆内存的OOM异常是实际应用中常见的内存溢出异常情况。当出现Java堆内存溢出时 ,异常堆栈信息“java.lang.OutOfMemoryError”会跟着进一步提示“Java heap space”。

&emsp;要解决这个区域的异常,一般的手段是先通过内存映像分析工具(如Eclipse Memory Analyzer)对Dump出来的堆转储快照进行分析,重点是确认内存中的对象是否是必要的,也就是要先分清楚到底是出现了内存泄漏(Memory Leak)还是内存溢出(Memory Overflow)。
&emsp;如果是内存泄露,可进一步通过工具查看泄露对象到GC Roots的引用链。于是就能找到泄露对象是通过怎样的路径与GC Roots相关联并导致垃圾收集器无法自动回收它们的。掌握了泄露对象的类型信息及GC Roots引用链的信息,就可以比较准确地定位出泄露代码的位置。

&emsp;如果不存在泄露,换句话说,就是内存中的对象确实都还必须存活着,那就应当检查虛拟机的堆参数(-Xmx与-Xms ) ,与机器物理内存对比看是否还可以调大,从代码上检查是否存在某些对象生命周期过长、持有状态时间过长的情况,尝试减少程序运行期的内存消耗。


## 2. 虚拟机栈和本地方法栈溢出
&emsp;HotSpot虚拟机中并不区分虚拟机栈和本地方法栈,因此 ,对于HotSpot来 说 ,虽然-Xoss参数 (设置本地方法栈大小)存在 ,但实际上是无效的,栈容量只由-Xss参数设定。 关于虚拟机栈和本地方法栈,在Java虚拟机规范中描述了两种异常:
- 如果线程请求的栈深度大于虚拟机所允许的最大深度,将拋出StackOverflowError异常。
- 如果虚拟机在扩展栈时无法申请到足够的内存空间,则拋出OutOMemoryError异常。

&emsp;这里把异常分成两种情况,看似更加严谨,但却存在着一些互相重叠的地方:当栈空间无法继续分配时,到底是内存太小,还是已使用的栈空间太大,其本质上只是对同一件事情的两种描述而已。

- 使用-Xss参数减少栈内存容量。结果 :拋出StackOverflowError异常 ,异常出现时输出的堆栈深度相应缩小。
- 定义了大量的本地变量,增大此方法帧中本地变量表的长度。结果 :拋出 StackOverflowError异常时输出的堆栈深度相应缩小。

StackOverflowError异常:
```java
/**
 * VM Args：-Xss128k
 * @author zzm
 */
public class JavaVMStackSOF {

    private int stackLength = 1;

    public void stackLeak() {
        stackLength++;
        stackLeak();
    }

    public static void main(String[] args) throws Throwable {
        JavaVMStackSOF oom = new JavaVMStackSOF();
        try {
            oom.stackLeak();
        } catch (Throwable e) {
            System.out.println("stack length:" + oom.stackLength);
            throw e;
        }
    }
}
```
运行结果：
```
stack length :2402
Exception in thread"main"java.lang.StackOverflowError
at org.fenixsoft.oom.VMStackSOF.leak (WIStackSOF.java :20) 
at org.fenixsoft.oom.VMStackSOF.leak (WIStackSOF.java :21) 
at org.fenixsoft.oom.VMStackSOF.leak (WIStackSOF.iava :21) 
.....后续异常堆栈信息省略
```
&emsp;实验结果表明:在单个线程下,无论是由于栈帧（一个方法中包含的本地变量数）太大还是虚拟机栈容量（-Xss参数减少每个线程栈内存容量）太小,当内存无法分配的时候,虚拟机拋出的都是StackOverflowError异常。

OutOMemoryError异常：
&emsp;通过不断地建立线程的方式倒是可以产生内存溢出异常，但是这样产生的内存溢出异常与栈空间是否足够大并不存在任何联系,或者准确地说，在这种情况下，为每个线程的栈分配的内存越大，反而越容易产生内存溢出异常。

&emsp;其实原因不难理解,操作系统分配给每个进程的内存是有限制的,譬如32位的Windows 限制为2GB。虚拟机提供了参数来控制Java堆和方法区的这两部分内存的最大值。剩余的内存为2GB ( 操作系统限制)减去Xmx ( 最大堆容量),再减去MaxPermSize (最大方法区容量 ),程序计数器消耗内存很小,可以忽略掉。如果虚拟机进程本身耗费的内存不计算在内 ,剩下的内存就由虚拟机栈和本地方法栈“瓜分” 了。每个线程分配到的栈容量越大,可以建立的线程数量自然就越少,建立线程时就越容易把剩下的内存耗尽。

&emsp;这一点读者需要在开发多线程的应用时特别注意,出现StackOverflowError异常时有错误堆栈可以阅读,相对来说,比较容易找到问题的所在。而且 ,如果使用虚拟机默认参数,栈深度在大多数情况下(因为每个方法压入栈的帧大小并不是一样的,所以只能说在大多数情况下)达到1000〜2000完全没有问题,对于正常的方法调用(包括递归),这个深度应该完全够用了。但是 ,**如果是建立过多线程导致的内存溢出,在不能减少线程数或者更换64位虚拟机的情况下,就只能通过减少最大堆和减少栈容量来换取更多的线程**。如果没有这方面的处理经验,这种通过“减少内存”的手段来解决内存溢出的方式会比较难以想到。
```java
/**
 * VM Args：-Xss2M （这时候不妨设大些）
 * @author zzm
 */
public class JavaVMStackOOM {
       private void dontStop() {
              while (true) {
              }
       }
       public void stackLeakByThread() {
              while (true) {
                     Thread thread = new Thread(new Runnable() {
                            @Override
                            public void run() {
                                   dontStop();
                            }
                     });
                     thread.start();
              }
       }

       public static void main(String[] args) throws Throwable {
              JavaVMStackOOM oom = new JavaVMStackOOM();
              oom.stackLeakByThread();
       }
}
```
```
Exception in thread"main"java.lang.OutOfMemoryError :unable to create new native thread
```

## 3. 方法区和运行时常量池溢出
&emsp;由于运行时常量池是方法区的一部分,因此这两个区域的溢出测试就放在一起进行。前面提到JDK 1.7开始逐步“去永久代”的事情,在此就以测试代码观察一下这件事对程序的实际影响。

### 3.1 常量池
&emsp;String.intern() 是一个Native方法,它的作用是:如果字符串常量池中已经包含一个等于此String对象的字符串,则返回代表池中这个字符串的String对 象 ;否则 ,将此String对象包含的字符串添加到常量池中,并且返回此String对象的引用。在JDK 1.6及之前的版本中,由于常量池分配在永久代内,我们可以通过-XX : PermSize和-XX : MaxPermSize限制方法区大小 ,从而间接限制其中常量池的容量。
```java
/**
 * VM Args：-XX:PermSize=10M -XX:MaxPermSize=10M
 * @author zzm
 */
public class RuntimeConstantPoolOOM {

    public static void main(String[] args) {
        // 使用List保持着常量池引用，避免Full GC回收常量池行为
        List<String> list = new ArrayList<String>();
        // 10MB的PermSize在integer范围内足够产生OOM了
        int i = 0; 
        while (true) {
            list.add(String.valueOf(i++).intern());
        }
    }
}
```
```
Exception in thread"main"java.lang.OutOfMemoryError :PermGen space
at java.lang.String, intern (Native Method)
at org.fenixsoft.oom.RuntimeConstantPoolOOM.main(RuntimeConstantPoolOOM.java:18)
```
&emsp;从运行结果中可以看到,运行时常量池溢出,在OutOfMemoryError后面跟随的提示信息是“**PermGen space**” ,说明运行时常量池属于方法区(HotSpot虚拟机中的永久代)的一部 分。
而使用JDK 1.7/1.8运行这段程序就不会得到相同的结果,while循环将一直进行下去。
&emsp;关于这个字符串常量池的实现问题,还可以引申出一个更有意思的影响,String.intern()返回引用的测试:
```java
public class RuntimeConstantPoolOOM {

    public static void main(String[] args) {
        public static void main(String[] args) {
        String str1 = new StringBuilder("中国").append("钓鱼岛").toString();
        System.out.println(str1.intern() == str1);

        String str2 = new StringBuilder("ja").append("va").toString();
        System.out.println(str2.intern() == str2);
    }   }
}
```
&emsp;这段代码在JDK 1.6中运行,会得到两个false,而在JDK 1.7/1.8中运行,会得到一个true和一 个false。产生差异的原因是:在JDK 1.6中,intern() 方法会把首次遇到的字符串实例复制到永久代中,返回的也是永久代中这个字符串实例的引用,而由StringBuilder创建的字符串实例在Java堆上,所以必然不是同一个引用,将返回false。而JDK 1.7/1.8 (以及部分其他虚拟机 ,例如JRockit) 的intern() 实现不会再复制实例,只是**在常量池中记录首次出现的实例引用**，因此intern() 返回的引用和由StringBuilder(）创建的那个字符串实例是同一个。对str2比较返回false是因为“java”这个字符串在执行StringBuilder.toString() 之前已经出现过,字符串常量池中已经有它的引用了,不符合“ 首次出现” 的原则 ,而“计算机软件”这个字符串则是首次出现的,因此返回true。

### 3.2 方法区
&emsp;方法区用于存放Class的相关信息,如类名、访问修饰符、常量池、字段描述、方法描述 等。对于这些区域的测试,基本的思路是运行时产生大量的类去填满方法区,直到溢出。虽然直接使用Java SE API也可以动态产生类(如反射时的GeneratedConstmctorAccessor和动态代理等),但在本次实验中操作起来比较麻烦。在下面代码中,借助CGLib直接操作字节码运行时生成了大量的动态类。

&emsp;值得特别注意的是,我们在这个例子中模拟的场景并非纯粹是一个实验,这样的应用经常会出现在实际应用中:当前的很多主流框架,如Spring、Hibernate ,在对类进行增强时, 都会使用到CGLib这类字节码技术,增强的类越多,就需要越大的方法区来保证动态生成的 Class可以加载入内存。另外,JVM上的动态语言(例如Groovy等 )通常都会持续创建类来实现语言的动态性,随着这类语言的流行,也越来越容易遇到与下面相似的溢出场景。

借助CGLib使方法区出现内存溢出异常:
```java
/**
 * VM Args： -XX:PermSize=10M -XX:MaxPermSize=10M
 * @author zzm
 */
public class JavaMethodAreaOOM {

    public static void main(String[] args) {
        while (true) {
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(OOMObject.class);
            enhancer.setUseCache(false);
            enhancer.setCallback(new MethodInterceptor() {
                public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
                    return proxy.invokeSuper(obj, args);
                }
            });
            enhancer.create();
        }
    }

    static class OOMObject {

    }
}
```
```
Caused by :java.lang.OutOfMemoryError :PermGen space
at java.lang.ClassLoader.defineClassl (Native Method)
at java.lang.ClassLoader.defineClassCond (ClassLoader. java :632 ) at java.lang.ClassLoader.defineClass (ClassLoader.java :616 )
— 8 more
```
&emsp;方法区溢出也是一种常见的内存溢出异常,一个类要被垃圾收集器回收掉,判定条件是比较苛刻的。**在经常动态生成大量Class的应用中,需要特别注意类的回收状况**。这类场景除了上面提到的程序使用了**CGLib**字节码增强和动态语言之外,常见的还有:大量**JSP**或动态产生JSP文件的应用(JSP第一次运行时需要编译为Java类 )、基于**OSGi**的应用(即使是同一个类文件,被不同的加载器加载也会视为不同的类)等。


## 4. 本机直接内存溢出
&emsp;DirectMemory容量可通过-XX : MaxDirectMemorySize指定,如果不指定,则默认与Java堆最大值(-Xmx指定)一样 ,下面代码越过了DirectByteBuffer类 ,直接通过反射获取Unsafe实例进行内存分配(Unsafe类的getUnsafe()方法限制了只有引导类加载器才会返回实例,也就是设计者希望只有rt.jar中的类才能使用Unsafe的功能)。因为,虽然使用 DirectByteBuffer分配内存也会拋出内存溢由异常,但它抛出异常时并没有真正向操作系统申请分配内存,而是通过计算得知内存无法分配,于是手动拋出异常,真正申请分配内存的方法unsafe.allocateMemory()。
使用unsafe分配本机内存：
```java
/**
 * VM Args：-Xmx20M -XX:MaxDirectMemorySize=10M
 * @author zzm
 */
public class DirectMemoryOOM {

    private static final int _1MB = 1024 * 1024;

    public static void main(String[] args) throws Exception {
        Field unsafeField = Unsafe.class.getDeclaredFields()[0];
        unsafeField.setAccessible(true);
        Unsafe unsafe = (Unsafe) unsafeField.get(null);
        while (true) {
            unsafe.allocateMemory(_1MB);
        }
    }
}
```
```
Exception in thread"main"java.lang.OutOfMemoryError at sun.misc.Unsafe .allocateMemory (Native Method ) at org. fenixsoft. oom.DMOOM.main (DMOOM.java :20 )
```
&emsp;由DirectMemory导致的内存溢出,一个明显的特征是在Heap Dump文件中不会看见明显的异常,如果读者发现OOM之后Dump文件很小,而程序中又直接或间接使用了NIO,那就可以考虑检查一下是不是这方面的原因。

&nbsp;
>参考：
《深入理解Java虚拟机》
