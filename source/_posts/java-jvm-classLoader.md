---
title: 深入理解Java虚拟机 之 虚拟机类加载机制
subtitle: 虚拟机类加载机制
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - java
  - 编程语言
  - jvm
categories:
  - java
date: 2019-10-22 10:26:02
---



# 虚拟机类加载机制

## 1. 概述
&emsp;在Java语言里面，类型的加载、连接和初始化过程都是在程序运行期间完成的，这种策略虽然会另类加载时稍微增加一些性能开销，但是会为Java应用程序提供高度的灵活性，Java里天生可以动态扩展的语言特性就是依赖**运行期动态加载**和**动态连接**这个特点实现的。

## 2. 类加载的时机
&emsp;JVM类加载机制流程大致分为以下七个步骤：
- 1.加载 Loading
- 2.验证 Verification
- 3.准备 Preparation
- 4.解析 Resolution
- 5.初始化 Initialization
- 6.使用 Using
- 7.卸载 Unloading

![类的生命周期](https://gitee.com/JP6907/Pic/raw/master/java/jvm/class-life.webp)

&emsp;其中，加载、验证、准备、初始化和卸载这5个阶段的顺序是确定的，类的加载过程必须按照这种顺序按部就班地开始，而解析阶段则不一定：在某些情况下可以在初始化阶段之后再开始，这是为了支持Java语言的运行时绑定（动态绑定/晚期绑定）。



## 3. 类加载的过程

### 3.1 加载
&emsp;“加载”是“类加载”（Class Loading）过程的一个阶段。在加载阶段，虚拟机需要完成以下3件事情：
1. 通过一个类的全限定名来获取定义此类的**二进制字节流**。
2. 将这个字节流所代表的静态存储结构转化为**方法区**的运行时数据结构。
3. 在内存中生成一个代表这个类的**java.lang.Class**对象，作为方法区这个类的各种数据的访问入口。

&emsp;注意：这里不一定非得要从一个Class文件获取，也可以通过其它方式获得：
- 从zip包中读取（比如从jar包和war包中读取）
- 运行时计算生成（动态代理），ProxyGenerator.generateProxyClass
- 由其它文件生成（比如将JSP文件转换成对应的CLass类）
- 数据库中读取，把程序安装到数据库中来完成程序代码在集群间的分发

&emsp;相对于类加载过程的其他阶段，一个非数组类的加载阶段是开发人员可控性最强的，因为加载阶段既可以使用系统提供的引导类加载器来完成，也可以由用户自定义的类加载器去完成。
&emsp;对于数组类，数组类本身不通过类加载器创建，它是由Java虚拟机直接创建的，但是数组类的元素类型最终还是要靠类加载器去创建。
- 如果数组的组件类型是引用类型，那么递归采用上述的加载过程去加载这个组件类型，该数组在加载该组建类型的类加载期器的类命名空间上被标识；
- 如果数组的组件类型不是引用类型，Java虚拟机会把数组标记为与引导类加载期关联；

### 3.2 验证
&emsp;Class文件并不一定要求用Java源码编译而来，可以使用任何途径产生。虚拟机如果不检查输入的字节流，很可能会因为载入了有害的字节流而导致系统崩溃，因此验证阶段是非常重要的，它包含下面4个检验动作：
- 文件格式验证
- 元数据验证
- 字节码验证
- 符号引用验证

1. 文件格式验证
&emsp;验证字节流是否符合Class文件格式的规范，并且能被当前版本的虚拟机处理：
- 是否以魔数0xCAFEBABE开头
- 主次版本号是否在当前虚拟机处理范围之内
- 常量池中的常量是否有不被支持的常量类型
- 指向常量的各种索引值中是否有指向不存在的常量或不符合类型的常量
- CONSTANTUtf8info型的常量中是否有不符合UTF8编码的数据
- Class文件中各个部分及文件本身是否有被删除的或附加的其他信息
- ...

&emsp;该阶段的主要目的是保证输入的字节流能正确地解析并储存与方法区之内，格式上符合一个Java类型信息的要求，后面的3个阶段都是基于方法区的储存结构进行，不会再直接操作字节流。

2. 元数据验证
&emsp;对字节码信息进行语义分析，保证其描述的信息符合Java语言规范的要求：
- 这个类是否有父类
- 这个类的父类是否继承了不准许被继承的类
- 如果这个类不是抽象类,是否实现了其父类或者接口之中要求实现的所有方法
- 类中的字段方法是否与父类产生矛盾

主要目的是对类的元数据信息进行语义检验，保证不存在不符合Java语言规范的元数据信息。


3. 字节码验证
&emsp;主要目的是通过数据流和控制流分析，确定程序语义是合法的、符合逻辑的。
- 保证任意时刻操作数栈的数据类型与指令代码序列都能配合工作
- 保证跳转指令不会跳转到方法体以外的字节码指令上
- 保证方法体中的类型转换是有效的
- ...

&emsp;即使字节码验证之中进行了大量的检查，也不能保证一定是安全的。这涉及到离散数学一个很著名的问题”Halting Problem“：通过程序去检验程序逻辑是无法做到绝对准确的-不能通过程序准确检查出程序是否能在有限的时间之内结束运行。

&emsp;在文章[《深入理解Java虚拟机 之 类文件结构》](http://zhoujiapeng.top/java/java-jvm-class-file-struct/#37)中有介绍到：在类加载的验证阶段，由于数据流验证的高复杂性，虚拟机设计团队为了避免过多的时间消耗在字节码验证阶段，在JDK1.6之后的 Javac编译器和 Java 虚拟机中进行了一项优化，给方法体的 Code 属性的属性表中增加了一项名为"**StackMapTable**"的属性，这项属性描述了方法体中所有基本块(Basic Block，按照控制流拆分的代码块)开始时本地变量表和操作数栈应有的状态，在字节码验证期间，就不需要根据程序推导这些状态的合法性，只需要检查 StackMapTable 属性中的记录是否合法即可，这样将字节码验证的类型推导转变为类型检查从而节省一些时间。


4. 符号引用验证
&emsp;这个阶段发生在虚拟机将符号引用转化为直接引用的时候-解析阶段发生。
- 符号引用中通过字符串描述的全限定名是否找到相应的类
- 在指定的类中是否存在符合方法的字段描述符以及简单名称说描述的方法和字段
- 符号引用中的类、字段、方法的访问性是否被当前类访问

&emsp;符号引用验证的目的是确保解析动作正常执行。


### 3.3 准备
&emsp;准备阶段是正式为**类变量**分配内存并设置类变量的**初始值**阶段，即在方法区中分配这些变量所使用的内存空间。这里需要注意两个问题：
- 这时候进行内存分配的仅包括类变量(static 修饰的变量),而不包括实例变量，实例变量将会在对象实例化时随着对象一起分配在java堆中
- 这里所说的初始值“通常情况”下是数据类型的零值，假设一个类变量定义为: public static int value = 12; 那么变量value在准备阶段过后的初始值为0而不是12，把value赋值为123的 putstatic 指令是程序被编译后，存放于类构造器\<clinit>()方法之中，所以把value赋值为12的动作将在初始化阶段才会被执行。

&emsp;另外，如果类字段的字段属性表中存在 **ConstantValue** 属性(同时被final和static修饰)，那么在准备阶段就会被初始化为 ConstantValue 所指定的值:
&emsp;public static **final** int value = 123;
&emsp;那么在准备阶段value会被设置为123。

### 3.4 解析
&emsp;解析阶段是虚拟机常量池内的符号引用替换为直接引用的过程。 
- 符号引用：符号引用是一组符号来描述所引用的目标对象，符号可以是任何形式的字面量，只要使用时能无歧义地定位到目标即可。符号引用与虚拟机实现的内存布局无关，引用的目标对象并不一定已经加载到内存中。 
- 直接引用：直接引用可以是直接指向目标对象的指针、相对偏移量或是一个能间接定位到目标的句柄。直接引用是与虚拟机内存布局实现相关的，同一个符号引用在不同虚拟机实例上翻译出来的直接引用一般不会相同，如果有了直接引用，那引用的目标必定已经在内存中存在。

&emsp;虚拟机规范并没有规定解析阶段发生的具体时间，只要求了在执行anewarry、checkcast、getfield、instanceof、invokeinterface、invokespecial、invokestatic、invokevirtual、multianewarray、new、putfield和putstatic这13个用于操作符号引用的字节码指令之前，先对它们使用的符号引用进行解析，所以虚拟机实现会根据需要来判断，到底是在类被加载器加载时就对常量池中的符号引用进行解析，还是等到一个符号引用将要被使用前才去解析它。

&emsp;解析的动作主要针对类或接口、字段、类方法、接口方法四类符号引用进行。分别对应编译后常量池内的CONSTANT_Class_Info、CONSTANT_Fieldref_Info、CONSTANT_Methodef_Info、CONSTANT_InterfaceMethoder_Info四种常量类型。

1.类、接口的解析 
2.字段解析 
3.类方法解析 
4.接口方法解析



### 3.5 初始化
&emsp;类的初始化阶段是类加载过程的最后一步，在准备阶段，类变量已赋过一次系统要求的初始值，而在初始化阶段，则是根据程序员通过程序制定的主观计划去初始化类变量和其他资源，也就是执行类构造器\<clinit>()方法的过程。关于clinit方法有下面几点需要注意：
- 类构造器< clinit >()方法是由编译器自动收集类中的所有**类变量的赋值动作**和**静态语句块**(static{}块)中的语句合并产生的。
- 类构造器< clinit >()方法与类的构造函数(实例构造函数< init >()方法)不同，它不需要显式调用父类构造，虚拟机会保证在子类< clinit >()方法执行之前，父类的< clinit >()方法已经执行完毕。因此在虚拟机中的第一个执行的< clinit >()方法的类肯定是java.lang.Object。
- 由于父类的< clinit >()方法先执行，也就意味着父类中定义的静态语句快要优先于子类的变量赋值操作。
- < clinit >()方法对于类或接口来说并不是必须的，如果一个类中没有静态语句，也没有变量赋值的操作，那么编译器可以不为这个类生成< clinit >()方法。
- 接口中不能使用静态语句块，但接口与类不太能够的是，执行接口的< clinit >()方法不需要先执行父接口的< clinit >()方法。只有当父接口中定义的变量被使用时，父接口才会被初始化。另外，接口的实现类在初始化时也一样不会执行接口的< clinit >()方法。
- 虚拟机会保证一个类的< clinit >()方法在多线程环境中被正确加锁和同步，如果多个线程同时去初始化一个类，那么只会有一个线程执行这个类的< clinit >()方法，其他线程都需要阻塞等待，直到活动线程执行< clinit >()方法完毕。如果一个类的< clinit >()方法中有耗时很长的操作，那就可能造成多个进程阻塞。

&emsp;虚拟机规范严格规定**有且只有**5种情况必须立即对类进行”初始化“：
- 1.遇到new、getstatic、putstatic或invokestatic这4条字节码指令时，如果类没有进行过初始化，则需先触发其初始化。生成这4条指令的最常见的java代码场景是：使用new关键字实例化对象、读取或设置一个类的静态字段(被final修饰、已在编译器把结果放入常量池的静态字段除外)的时候，以及调用类的静态方法的时候。
- 2.使用java.lang.reflect包的方法对类进行反射调用的时候。
- 3.当初始化一个类的时候，如果发现其父类还没有进行过初始化、则需要先出发其父类的初始化。
- 4.jvm启动时，用户指定一个执行的主类(包含main方法的那个类)，虚拟机会先初始化这个类。

&emsp;这种场景称为对一个类进行**主动引用**,除此之外所有引用类的所有方式都不会触发初始化，称为**被动引用**：
- 通过子类引用父类的静态字段(static)，不会导致子类初始化，对于静态字段，只有直接定义这个字段的类才会被初始化。
- 通过数组定义来引用类，不会触发此类的初始化，前面讲到，数组类本身不通过类加载器创建，它是由Java虚拟机直接创建的。
- 常量(static final)在编译阶段会存入**调用类**的常量池，本质上没有直接引用到定义常量的类。

## 4. 类加载器
&emsp;虚拟机设计团队把类加载阶段中的“**通过一个类的全限定名来获取描述此类的二进制字节流**”这个动作放到Java虚拟机外部去实现，以便让应用程序自己决定如何去获取所需要的类。实现这个动作的代码模块称为“类加载器”。

### 4.1 类与类加载器
&emsp;对于任意一个类，都需要由加载它的**类加载器和这个类本身一同确立其在Java虚拟机中的唯一性**。如果两个类来源于同一个Class文件，只要加载它们的类加载器不同，那么这两个类就必定不相等。
```java
public class JVM7_8 {

    public static void main(String[] args) throws Exception{
        //自定义类加载器
        ClassLoader myLoader = new ClassLoader() {
            @Override
            public Class<?> loadClass(String name) throws ClassNotFoundException {
                try {
                    String fileName = name.substring(name.lastIndexOf(".") + 1) + ".class";
                    InputStream is = getClass().getResourceAsStream(fileName);
                    if (is == null) {
                        return super.loadClass(name);
                    }
                    byte[] b = new byte[is.available()];
                    is.read(b);
                    return defineClass(name, b, 0, b.length);
                } catch (IOException e) {
                    throw new ClassNotFoundException(name);
                }
            }
        };

        Object obj = myLoader.loadClass("com.jp.jvm.ch07.JVM7_8").newInstance();
        System.out.println(obj.getClass());
        //不属于同一个类加载器，所以不一样
        //虚拟机中存在了两个ClassLoader
        //一个是由系统应用程序类加载器加载的
        //另一个是自定义的类加载器
        System.out.println(obj instanceof com.jp.jvm.ch07.JVM7_8);
    }
}
```
&emsp;输出结果：
```
class com.jp.jvm.ch07.JVM7_8
false
```

### 4.2 双亲委派模型
&emsp;从虚拟机的角度来说，只存在两种不同的类加载器：一种是启动类加载器（Bootstrap ClassLoader），该类加载器使用C++语言实现，属于虚拟机自身的一部分。另外一种就是所有其它的类加载器，这些类加载器是由Java语言实现，独立于JVM外部，并且全部继承自抽象类java.lang.ClassLoader。
&emsp;从Java开发人员的角度来看，大部分Java程序一般会使用到以下三种系统提供的类加载器：
1. 启动类加载器（Bootstrap ClassLoader）：负责加载 JAVA_HOME\lib 目录中并且能被虚拟机识别的类库到JVM内存中，如果名称不符合的类库即使放在lib目录中也不会被加载。该类加载器无法被Java程序直接引用。
2. 扩展类加载器（Extension ClassLoader）：该加载器主要是负责加载 JAVA_HOME\lib\ext 目录中，或者被 java.ext.dirs 系统变量所定义的路径中的所有类库，该加载器可以被开发者直接使用。
3. 应用程序类加载器（Application ClassLoader）：该类加载器也称为系统类加载器，它负责加载用户类路径（Classpath）上所指定的类库，开发者可以直接使用该类加载器，如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。我们的应用程序都是由这三类加载器互相配合进行加载的，另外可以自己定义类加载器。 
4. 自定义类加载器必须继承 ClassLoader。 

&emsp;这些类加载器之间的关系如下图所示：
![双亲委派模型](https://gitee.com/JP6907/Pic/raw/master/java/jvm/Parents-Delegation-Model.webp)

&emsp;加载器之间的这种层次关系，就称为类加载器的**双亲委派模型**（Parent Delegation Model）。该模型要求除了顶层的启动类加载器外，其余的类加载器都应当有自己的父类加载器。子类加载器和父类加载器不是以继承（Inheritance）的关系来实现，而是通过**组合**（Composition）关系来复用父加载器的代码。

&emsp;双亲委派模型的工作过程为：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的加载器都是如此，因此所有的类加载请求都会传给顶层的启动类加载器，只有当父加载器反馈自己无法完成该加载请求（该加载器的搜索范围中没有找到对应的类）时，子加载器才会尝试自己去加载。

&emsp;使用这种模型来组织类加载器之间的关系的好处是Java类随着它的类加载器一起具备了一种带有优先级的层次关系(**各个类加载器的基础类统一问题**)。例如java.lang.Object类，无论哪个类加载器去加载该类，最终都是由启动类加载器进行加载，因此Object类在程序的各种类加载器环境中都是同一个类。否则的话，如果不使用该模型的话，如果用户自定义一个java.lang.Object类且存放在classpath中，那么系统中将会出现多个Object类，应用程序也会变得很混乱。如果我们自定义一个rt.jar中已有类的同名Java类，会发现JVM可以正常编译，但该类永远无法被加载运行。

&emsp;下面是双亲委派模型的实现：
```java
//ClassLoader.java
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            //首先检查请求的类是否已经被加载
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    //如果父类加载器抛出 ClassNotFoundException
                    //说明父类加载器无法完成加载请求
                }

                if (c == null) {
                    //在父类加载器无法加载的时候
                    //再调用本身的 findlClass 方法来进行加载
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```



&nbsp;
> 参考：
《深入理解Java虚拟机》