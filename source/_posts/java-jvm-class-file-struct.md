---
title: 深入理解Java虚拟机 之 类文件结构
subtitle: .class
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - java
  - 编程语言
  - jvm
categories:
  - java
date: 2019-10-20 21:36:25
---



# 类文件结构
&emsp;代码编译的结果从本地机器码转变为字节码，是存储格式发展的一小步，却是编程语言发展的一大步。
&emsp;在JDK的bin目录中，Oracle公司已经为我们准备好一个专门用于分析Class文件字节码的工具：javap，使用javap工具的-verbose参数可以输出输出 class 文件字节码内容。

## 1. 概述
&emsp;近10年来虚拟机以及大量建立在虚拟机之上的程序语言如雨后春笋般出现并蓬勃发展，将我们编写的程序编译成二进制本地机器码（Native Code）已不再是唯一的选择，越来越多的程序语言选择了与操作系统和机器指令集无关的、平台中立的格式作为程序编译后的存储格式。

## 2. 无关性的基石
&emsp;Java在刚刚诞生之时曾经提出过一个非常著名的宣传口号：“一次编写，到处运行（Write Once,Run Anywhere）”，这句话充分表达了软件开发人员对冲破平台界限的渴求。在无时无刻不充满竞争的IT领域，不可能只有Wintel存在，我们也不希望只有Wintel存在，各种不同的硬件体系结构和不同的操作系统肯定会长期并存发展。“与平台无关”的理想最终实现在操作系统的应用层上：Sun公司以及其他虚拟机提供商发布了许多可以运行在各种不同平台上的虚拟机，这些虚拟机都可以载入和执行同一种平台无关的字节码，从而实现了程序的“一次编写，到处运行”。

&emsp;各种不同平台的虚拟机与所有平台都统一使用的程序存储格式——**字节码（ByteCode）是构成平台无关性的基石**。到目前为止，或许大部分程序员都还认为Java虚拟机执行Java程序是一件理所当然和天经地义的事情。但在Java发展之初，设计者就曾经考虑过并实现了让其他语言运行在Java虚拟机之上的可能性，他们在发布规范文档的时候，也刻意把Java的规范拆分成了Java语言规范《The Java Language Specification》及Java虚拟机规范《The Java Virtual Machine Specification》。并且在1997年发布的第一版Java虚拟机规范中就曾经承诺过：“In the future,we will consider bounded extensions to the Java virtual machine to provide better support for other languages”（在未来，我们会对Java虚拟机进行适当的扩展，以便更好地支持其他语言运行于JVM之上），当Java虚拟机发展到JDK 1.7～1.8的时候，JVM设计者通过JSR-292基本兑现了这个承诺。

&emsp;**实现语言无关性的基础仍然是虚拟机和字节码存储格式**。Java虚拟机不和包括Java在内的任何语言绑定，它**只与“Class文件”这种特定的二进制文件格式所关联**，Class文件中包含了Java虚拟机指令集和符号表以及若干其他辅助信息。基于安全方面的考虑，Java虚拟机规范要求在Class文件中使用许多强制性的语法和结构化约束，但任一门功能性语言都可以表示为一个能被Java虚拟机所接受的有效的Class文件。作为一个通用的、机器无关的执行平台，任何其他语言的实现者都可以将Java虚拟机作为语言的产品交付媒介。例如，使用Java编译器可以把Java代码编译为存储字节码的Class文件，使用JRuby等其他语言的编译器一样可以把程序代码编译成Class文件，虚拟机并不关心Class的来源是何种语言，如图6所示。

![language independent](https://github.com/JP6907/Pic/raw/master/java/class/language-independence)

&emsp;Java语言中的各种变量、关键字和运算符号的语义最终都是由多条字节码命令组合而成的，因此字节码命令所能提供的语义描述能力肯定会比Java语言本身更加强大。因此，有一些Java语言本身无法有效支持的语言特性不代表字节码本身无法有效支持，这也为其他语言实现一些有别于Java的语言特性提供了基础。


## 3. Class类文件的结构
&emsp;本文关于Class文件结构的讲解中，我们将以《Java虚拟机规范（第2版）》（1999年发布，对应于JDK 1.4时代的Java虚拟机）中的定义为主线，这部分内容虽然古老，但它所包含的指令、属性是Class文件中最重要和最基础的。

&emsp;注意: **任何一个Class文件都对应着唯一一个类或接口的定义信息，但反过来说，类或接口并不一定都得定义在文件里**（譬如类或接口也可以通过类加载器直接生成）。本章中，笔者只是通俗地将任意一个有效的类或接口所应当满足的格式称为“Class文件格式”，实际上它并不一定以磁盘文件的形式存在。

&emsp;Class文件是一组以8位字节为基础单位的二进制流，各个数据项目严格按照顺序紧凑地排列在Class文件之中，中间没有添加任何分隔符，这使得整个Class文件中存储的内容几乎全部是程序运行的必要数据，没有空隙存在。当遇到需要占用8位字节以上空间的数据项时，则会按照高位在前的方式分割成若干个8位字节进行存储。

&emsp;根据Java虚拟机规范的规定，Class文件格式采用一种类似于C语言结构体的伪结构来存储数据，这种伪结构中只有两种数据类型：**无符号数和表**，后面的解析都要以这两种数据类型为基础，所以这里要先介绍这两个概念。

&emsp;**无符号数**属于基本的数据类型，以u1、u2、u4、u8来分别代表1个字节、2个字节、4个字节和8个字节的无符号数，无符号数可以用来描述数字、索引引用、数量值或者按照UTF-8编码构成字符串值。

&emsp;**表**是由多个无符号数或者其他表作为数据项构成的复合数据类型，所有表都习惯性地以“_info”结尾。表用于描述有层次关系的复合结构的数据，整个Class文件本质上就是一张表，它由表6-1所示的数据项构成。

![Class文件格式](https://github.com/JP6907/Pic/raw/master/java/class/class-file-struct)

&emsp;无论是无符号数还是表，当需要描述同一类型但数量不定的多个数据时，经常会使用一个**前置的容量计数器**加若干个连续的数据项的形式，这时称这一系列连续的某一类型的数据为某一类型的集合。图中的数据项，无论是顺序还是数量，甚至于数据存储的字节序（Byte Ordering,Class文件中字节序为Big-Endian）这样的细节，都是被严格限定的，哪个字节代表什么含义，长度是多少，先后顺序如何，都不允许改变。


### 3.1 魔数与Class文件的版本
&emsp;每个Class文件的头4个字节称为魔数（Magic Number），它的唯一作用是确定这个文件是否为一个能被虚拟机接受的Class文件。很多文件存储标准中都使用魔数来进行身份识别，Class文件的魔数的获得很有“浪漫气息”，值为：0xCAFEBABE（咖啡宝贝？）

&emsp;紧接着魔数的4个字节存储的是Class文件的版本号：第5和第6个字节是次版本号（Minor Version），第7和第8个字节是主版本号（Major Version）。Java的版本号是从45开始的，JDK 1.1之后的每个JDK大版本发布主版本号向上加1（JDK 1.0～1.1使用了45.0～45.3的版本号），高版本的JDK能向下兼容以前版本的Class文件，但不能运行以后版本的Class文件，即使文件格式并未发生任何变化，虚拟机也必须拒绝执行超过其版本号的Class文件。

&emsp;例如，JDK 1.1能支持版本号为45.0～45.65535的Class文件，无法执行版本号为46.0以上的Class文件，而JDK 1.2则能支持45.0～46.65535的Class文件。现在，最新的JDK版本为1.7，可生成的Class文件主版本号最大值为51.0。

&emsp;下表列出了从JDK 1.1到JDK 1.7，主流JDK版本编译器输出的默认和可支持的Class文件版本号。
![Class文件版本号](https://github.com/JP6907/Pic/raw/master/java/class/class-file-version)


### 3.2 常量池
&emsp;紧接着主次版本号之后的是常量池入口，常量池可以理解为Class文件之中的资源仓库，它是Class文件结构中与其他项目关联最多的数据类型，也是占用Class文件空间最大的数据项目之一，同时它还是在Class文件中第一个出现的表类型数据项目。

&emsp;由于常量池中常量的数量是不固定的，所以在常量池的入口需要放置一项u2类型的数据，代表常量池容量计数值（constant_pool_count）。与Java中语言习惯不一样的是，这个**容量计数是从1而不是0开始的**，设计者将第0项常量空出来是有特殊考虑的，这样做的目的在于满足后面某些指向常量池的索引值的数据在特定情况下需要表达“不引用任何一个常量池项目”的含义，这种情况就可以把索引值置为0来表示。Class文件结构中只有常量池的容量计数是从1开始，对于其他集合类型，包括接口索引集合、字段表集合、方法表集合等的容量计数都与一般习惯相同，是从0开始的。 

&emsp;常量池中主要存放两大类常量：字面量（Literal）和符号引用（Symbolic References）。字面量比较接近于Java语言层面的常量概念，如文本字符串、声明为final的常量值等。而符号引用则属于编译原理方面的概念，包括了下面三类常量：
- 类和接口的全限定名（Fully Qualified Name）
- 字段的名称和描述符（Descriptor）
- 方法的名称和描述符

&emsp;Java代码在进行Javac编译的时候，并不像C和C++那样有“连接”这一步骤，而是**在虚拟机加载Class文件的时候进行动态连接**。也就是说，在Class文件中不会保存各个方法、字段的最终内存布局信息，因此这些字段、方法的符号引用不经过运行期转换的话无法得到真正的内存入口地址，也就无法直接被虚拟机使用。当虚拟机运行时，需要从常量池获得对应的符号引用，再在类创建时或运行时解析、翻译到具体的内存地址之中。

&emsp;常量池中每一项常量都是一个表,表都有一个共同的特点，就是表开始的第一位是一个u1类型的标志位（tag，取值见表6-3中标志列），代表当前这个常量属于哪种常量类型。
![常量池的项目类型](https://github.com/JP6907/Pic/raw/master/java/class/constant-pool)

&emsp;每种常量类型各自均有自己的结构。我们需要根据表的第一位确定常量类型之后才能知道这个表的具体大小。比如常量池的某项常量，它的标志位（偏移地址：0x0000000A）是0x07，查表的标志列发现这个常量属于CONSTANT_Class_info类型，此类型的常量代表一个类或者接口的符号引用。CONSTANT_Class_info的结构比较简单：
![CONSTANT_Class_info](https://github.com/JP6907/Pic/raw/master/java/class/CONSTANT_Class_info)

&emsp;tag是标志位，上面已经讲过了，它用于区分常量类型；name_index是一个索引值，它指向常量池中一个CONSTANT_Utf8_info类型常量，此常量代表了这个类（或者接口）的全限定名，这里name_index值（偏移地址：0x0000000B）为0x0002，也即是指向了常量池中的第二项常量。继续从图6-3中查找第二项常量，它的标志位（地址：0x0000000D）是0x01，查表6-3可知确实是一个CONSTANT_Utf8_info类型的常量。CONSTANT_Utf8_info类型的结构见表6-5。 
![CONSTANT_Utf8_info](https://github.com/JP6907/Pic/raw/master/java/class/CONSTANT_Utf8_info)

&emsp;length值说明了这个UTF-8编码的字符串长度是多少字节，它后面紧跟着的长度为length字节的连续数据是一个使用UTF-8缩略编码表示的字符串。UTF-8缩略编码与普通UTF-8编码的区别是：从’\u0001’到’\u007f’之间的字符（相当于1～127的ASCII码）的缩略编码使用一个字节表示，从’\u0080’到’\u07ff’之间的所有字符的缩略编码用两个字节表示，从’\u0800’到’\uffff’之间的所有字符的缩略编码就按照普通UTF-8编码规则使用三个字节表示。

![常量池中14种常量项的结构总表](https://github.com/JP6907/Pic/raw/master/java/class/constant-pool-table)

### 3.3 访问标志
&emsp;访问标志（access_flags），这个标志用于识别一些类或者接口层次的访问信息，包括：这个Class是类还是接口；是否定义为public类型；是否定义为abstract类型；如果是类的话，是否被声明为final等。具体的标志位以及标志的含义见表6-7。 
![访问标志](https://github.com/JP6907/Pic/raw/master/java/class/access-flag.png)

### 3.4 类索引、父类索引与接口索引集合
&emsp;类索引、父类索引和接口索引集合都按顺序排列在访问标志之后。
&emsp;类索引（this_class）和父类索引（super_class）都是一个u2类型的数据，而接口索引集合（interfaces）是一组u2类型的数据的集合，Class文件中由这三项数据来确定这个类的继承关系。类索引用于确定这个类的全限定名，父类索引用于确定这个类的父类的全限定名，接口索引集合就用来描述这个类实现了哪些接口，这些被实现的接口将按implements语句（如果这个类本身是一个接口，则应当是extends语句）后的接口顺序从左到右排列在接口索引集合中。
![类索引查找全限定名过程](https://github.com/JP6907/Pic/raw/master/java/class/class-search-name.png)

### 3.5 字段表集合
&emsp;字段表（field_info）用于描述接口或者类中声明的**变量**。字段（field）包括类级变量以及实例级变量，但不包括在方法内部声明的局部变量。字段叫什么名字、字段被定义为什么数据类型，这些都是无法固定的，只能引用常量池中的常量来描述。
![字段表结构](https://github.com/JP6907/Pic/raw/master/java/class/field_info.png)

&emsp;字段修饰符放在access_flags项目中，它与类中的access_flags项目是非常类似的，都是一个u2的数据类型，其中可以设置的标志位和含义见表6-9。 
![字段访问标志](https://github.com/JP6907/Pic/raw/master/java/class/field_info-access-flag.png)

&emsp;name_index和descriptor_index。它们都是对常量池的引用，分别代表着字段的简单名称以及字段和方法的描述符。现在需要解释一下“简单名称”、“描述符”以及前面出现过多次的“全限定名”这三种特殊字符串的概念:
- 全限定名：org/fenixsoft/clazz/TestClass
- 简单名称：inc
- 描述符：描述字段的数据类型、方法的参数列表（包括数量、类型以及顺序）和返回值。

![描述符标识字符含义](https://github.com/JP6907/Pic/raw/master/java/class/descriptor.png)
&emsp;对于数组类型，每一维度将使用一个前置的“[”字符来描述，如一个定义为“java.lang.String[][]”类型的二维数组，将被记录为：“[[Ljava/lang/String；”，一个整型数组“int[]”将被记录为“[I”。

&emsp;用描述符来描述方法时，按照先参数列表，后返回值的顺序描述，参数列表按照参数的严格顺序放在一组小括号“（）”之内。如方法void inc（）的描述符为“（）V”，方法java.lang.String toString（）的描述符为“（）Ljava/lang/String；”，方法int indexOf（char[]source,int sourceOffset,int sourceCount,char[]target,int targetOffset,int targetCount,int fromIndex）的描述符为“（[CII[CIII）I”。

&emsp;字段表都包含的固定数据项目到descriptor_index为止就结束了，不过在descriptor_index之后跟随着一个属性表集合用于存储一些额外的信息，字段都可以在属性表中描述零至多项的额外信息。对于本例中的字段m，它的属性表计数器为0，也就是没有需要额外描述的信息，但是，如果将字段m的声明改为“final static int m=123；”，那就可能会存在一项名称为ConstantValue的属性，其值指向常量123。关于attribute_info的其他内容，将在6.3.7节介绍属性表的数据项目时再进一步讲解。 

&emsp;字段表集合中不会列出从超类或者父接口中继承而来的字段，但有可能列出原本Java代码之中不存在的字段，譬如在内部类中为了保持对外部类的访问性，会自动添加指向外部类实例的字段。另外，**在Java语言中字段是无法重载的，两个字段的数据类型、修饰符不管是否相同，都必须使用不一样的名称，但是对于字节码来讲，如果两个字段的描述符不一致，那字段重名就是合法的**。

### 3.6 方法表集合
&emsp;Class文件存储格式中对方法的描述与对字段的描述几乎采用了完全一致的方式，方法表的结构如同字段表一样，依次包括了访问标志（access_flags）、名称索引（name_index）、描述符索引（descriptor_index）、属性表集合（attributes）几项。这些数据项目的含义也非常类似，仅在访问标志和属性表集合的可选项中有所区别。 
![方法表结构](https://github.com/JP6907/Pic/raw/master/java/class/method-table.png)

&emsp;因为volatile关键字和transient关键字不能修饰方法，所以方法表的访问标志中没有了ACC_VOLATILE标志和ACC_TRANSIENT标志。与之相对的，synchronized、native、strictfp和abstract关键字可以修饰方法，所以方法表的访问标志中增加了ACC_SYNCHRONIZED、ACC_NATIVE、ACC_STRICTFP和ACC_ABSTRACT标志。对于方法表，所有标志位及其取值可参见表
![方法表访问标志](https://github.com/JP6907/Pic/raw/master/java/class/method-access-flag.png)

&emsp;方法里的Java代码，经过编译器编译成字节码指令后，存放在方法属性表集合中一个名为“Code”的属性里面，属性表是Class文件格式中最具扩展性的一种数据项目。

&emsp;在Java语言中，要重载（Overload）一个方法，除了要与原方法具有相同的简单名称之外，还要求必须拥有一个与原方法不同的特征签名，**特征签名就是一个方法中各个参数在常量池中的字段符号引用的集合，也就是因为返回值不会包含在特征签名中，因此Java语言里面是无法仅仅依靠返回值的不同来对一个已有方法进行重载的**。但是在Class文件格式中，特征签名的范围更大一些，只要描述符不是完全一致的两个方法也可以共存。也就是说，**如果两个方法有相同的名称和特征签名，但返回值不同，那么也是可以合法共存于同一个Class文件中的**。（**Java代码的方法特征签名只包括了方法名称、参数顺序及参数类型，而字节码的特征签名还包括方法返回值以及受查异常表**）。

### 3.7 属性表集合
&emsp;在Class文件、字段表、方法表都可以携带自己的属性表集合，以用于描述某些场景专有的信息。
&emsp;与Class文件中其他的数据项目要求严格的顺序、长度和内容不同，属性表集合的限制稍微宽松了一些，不再要求各个属性表具有严格顺序，并且只要不与已有属性名重复，任何人实现的编译器都可以向属性表中写入自己定义的属性信息，Java虚拟机运行时会忽略掉它不认识的属性。
![虚拟机规范预定义的属性](https://github.com/JP6907/Pic/raw/master/java/class/attribute_info)

&emsp;对于每个属性，它的名称需要从常量池中引用一个CONSTANT_Utf8_info类型的常量来表示，而属性值的结构则是完全自定义的，只需要通过一个u4的长度属性去说明属性值所占用的位数即可。一个符合规则的属性表应该满足表6-14中所定义的结构。 
![属性表结构](https://github.com/JP6907/Pic/raw/master/java/class/attribute_info-struct.png)


1. Code属性

&emsp;Java程序方法体中的代码经过Javac编译器处理后，最终变为**字节码指令**存储在Code属性内。Code属性出现在方法表的属性集合之中，但并非所有的方法表都必须存在这个属性，譬如接口或者抽象类中的方法就不存在Code属性，如果方法表有Code属性存在，那么它的结构将如下：
![Code属性表结构](https://github.com/JP6907/Pic/raw/master/java/class/Code.png)

- attribute_name_index：指向CONSTANT_Utf8_info型常量的索引，常量值固定为“Code”，它代表了该属性的属性名称。
- attribute_length：指示了属性值的长度，由于属性名称索引与属性长度一共为6字节，所以属性值的长度固定为整个属性表长度减去6个字节。
- max_stack：操作数栈（Operand Stacks）深度的最大值。
- max_locals：局部变量表所需的存储空间，单位是Slot,Slot是虚拟机为局部变量分配内存所使用的最小单位。对于byte、char、float、int、short、boolean和returnAddress等长度不超过32位的数据类型，每个局部变量占用1个Slot，而double和long这两种64位的数据类型则需要两个Slot来存放。方法参数（包括实例方法中的隐藏参数“this”）、显式异常处理器的参数（Exception Handler Parameter，就是try-catch语句中catch块所定义的异常）、方法体中定义的局部变量都需要使用局部变量表来存放。
- code_length：字节码长度。为 u4 类型，理论上最大值可以达到2（32次方）-1，但是虚拟机规范中明确限制了一个方法不允许超过65535条字节码指令，即它实际只使用了u2的长度，如果超过这个限制，Javac编译器也会拒绝编译，一般一个方法很难超过这个长度，某些特殊情况，例如在编译一个很复杂的JSP文件时，某些JSP编译器会把JSP内容和页面输出的信息归并于一个方法之中，就可能因为方法生成字节码超长的原因而导致编译失败。
- code：存储字节码指令的一系列字节流。为 u1 类型，最多可以表达256条指令，目前，Java虚拟机规范已经定义了其中约200条编码值对应的指令含义。

2. Exceptions属性

&emsp;Exceptions属性并不是异常表，Exceptions属性的作用是**列举出方法中可能抛出的受查异常**（Checked Excepitons），也就是方法描述时在throws关键字后面列举的异常。
![Exceptions属性表结构](https://github.com/JP6907/Pic/raw/master/java/class/Exceptions.png)


3. LineNumberTable属性

&emsp;LineNumberTable属性用于描述**Java源码行号与字节码行号**（字节码的偏移量）之间的对应关系。它并不是运行时必需的属性，但默认会生成到Class文件之中，可以在Javac中分别使用-g：none或-g：lines选项来取消或要求生成这项信息。如果选择不生成LineNumberTable属性，对程序运行产生的最主要的影响就是当抛出异常时，堆栈中将不会显示出错的行号，并且在调试程序的时候，也无法按照源码行来设置断点。
![LineNumberTable属性表结构](https://github.com/JP6907/Pic/raw/master/java/class/LineNumberTable.png)

示例：
```
LineNumberTable:
line 3: 0
```
前者是字节码行号，后者是Java源码行号。

4. LocalVariableTable属性

&emsp;LocalVariableTable属性用于描述**栈帧中局部变量表中的变量与Java源码中定义的变量之间的关系**，它也不是运行时必需的属性，但默认会生成到Class文件之中，可以在Javac中分别使用-g：none或-g：vars选项来取消或要求生成这项信息。如果没有生成这项属性，最大的影响就是当其他人引用这个方法时，所有的参数名称都将会丢失，IDE将会使用诸如arg0、arg1之类的占位符代替原有的参数名，这对程序运行没有影响，但是会对代码编写带来较大不便，而且在调试期间无法根据参数名称从上下文中获得参数值。
![LocalVariableTable属性表结构](https://github.com/JP6907/Pic/raw/master/java/class/LocalVariableTable.png)


5. SourceFile属性

&emsp;SourceFile属性用于记录生成这个**Class文件的源码文件名称**。这个属性也是可选的，可以分别使用Javac的-g：none或-g：source选项来关闭或要求生成这项信息。在Java中，对于大多数的类来说，类名和文件名是一致的，但是有一些特殊情况（如内部类）例外。如果不生成这项属性，当抛出异常时，堆栈中将不会显示出错代码所属的文件名。这个属性是一个定长的属性。
![SourceFile属性表结构](https://github.com/JP6907/Pic/raw/master/java/class/LocalVariableTable.png)


6. ConstantValue属性

&emsp;ConstantValue属性的作用是**通知虚拟机自动为静态变量赋值**。只有被**static**关键字修饰的变量（类变量）才可以使用这项属性。类似“int x=123”和“static int x=123”这样的变量定义在Java程序中是非常常见的事情，但虚拟机对这两种变量赋值的方式和时刻都有所不同。对于非static类型的变量（也就是实例变量）的赋值是在实例构造器＜init＞方法中进行的；而对于类变量，则有两种方式可以选择：在类构造器＜clinit＞方法中或者使用ConstantValue属性。目前Sun Javac编译器的选择是：如果同时使用final和static来修饰一个变量（按照习惯，这里称“常量”更贴切），并且这个变量的数据类型是基本类型或者java.lang.String的话，就生成ConstantValue属性来进行初始化，如果这个变量没有被**final**修饰，或者并非基本类型及字符串，则将会选择在＜clinit＞方法中进行初始化。

&emsp;虽然有final关键字才更符合“ConstantValue”的语义，但虚拟机规范中并没有强制要求字段必须设置了ACC_FINAL标志，只要求了有ConstantValue属性的字段必须设置ACC_STATIC标志而已，对final关键字的要求是Javac编译器自己加入的限制。而对ConstantValue的属性值只能限于基本类型和String。
![ConstantValue属性表结构](https://github.com/JP6907/Pic/raw/master/java/class/ConstantValue.png)


7. InnerClasses属性

&emsp;InnerClasses属性用于记录**内部类与宿主类之间的关联**。如果一个类中定义了内部类，那编译器将会为它以及它所包含的内部类生成InnerClasses属性。
![InnerClasses属性表结构](https://github.com/JP6907/Pic/raw/master/java/class/InnerClasses.png)
数据项number_of_classes代表需要记录多少个内部类信息，每一个内部类的信息都由一个inner_classes_info表进行描述。
![inner_classes_info表结构](https://github.com/JP6907/Pic/raw/master/java/class/inner_classes_info.png)


8. Deprecated及Synthetic属性

&emsp;Deprecated和Synthetic两个属性都属于标志类型的布尔属性，只存在有和没有的区别，没有属性值的概念。
&emsp;Deprecated属性用于表示某个类、字段或者方法，已经被程序作者定为**不再推荐使用**，它可以通过在代码中使用@deprecated注释进行设置。

&emsp;Synthetic属性代表此**字段或者方法并不是由Java源码直接产生的**，而是由编译器自行添加的，在JDK 1.5之后，标识一个类、字段或者方法是编译器自动产生的，也可以设置它们访问标志中的ACC_SYNTHETIC标志位，其中最典型的例子就是Bridge Method。所有由非用户代码产生的类、方法及字段都应当至少设置Synthetic属性和ACC_SYNTHETIC标志位中的一项，唯一的例外是实例构造器“＜init＞”方法和类构造器“＜clinit＞”方法。
![Deprecated及Synthetic属性表结构](https://github.com/JP6907/Pic/raw/master/java/class/Deprecated-Synthetic.png)


9. StackMapTable属性

&emsp;这个属性会在虚拟机类加载的字节码验证阶段被新类型检查验证器（Type Checker）使用（见7.3.2节），目的在于代替以前比较消耗性能的基于数据流分析的类型推导验证器。
![StackMapTable属性表结构](https://github.com/JP6907/Pic/raw/master/java/class/StackMapTable.png)



10. Signature属性

&emsp;Signature是一个可选的定长属性，可以出现于类、属性表和方法表结构的属性表中。任何类、接口、初始化方法或成员的泛型签名如果包含了类型变量（Type Variables）或参数化类型（Parameterized Types），则Signature属性会为它记录泛型签名信息。之所以要专门使用这样一个属性去 **记录泛型类型**，是因为Java语言的泛型采用的是 **擦除法实现的伪泛型**，在字节码（Code属性）中，泛型信息编译（类型变量、参数化类型）之后都通通被擦除掉。使用擦除法的好处是实现简单（主要修改Javac编译器，虚拟机内部只做了很少的改动）、非常容易实现Backport，运行期也能够节省一些类型所占的内存空间。但坏处是运行期就无法像C#等有真泛型支持的语言那样，将泛型类型与用户定义的普通类型同等对待，例如**运行期做反射时无法获得到泛型信息。Signature属性就是为了弥补这个缺陷而增设的，现在Java的反射API能够获取泛型类型，最终的数据来源也就是这个属性**。
![Signature属性表结构](https://github.com/JP6907/Pic/raw/master/java/class/Signature.png)


11. BootstrapMethods属性

&emsp;BootstrapMethods是一个复杂的变长属性，位于类文件的属性表中。这个属性用于保存invokedynamic指令引用的引导方法限定符。《Java虚拟机规范（Java SE 7版）》规定，如果某个类文件结构的常量池中曾经出现过CONSTANT_InvokeDynamic_info类型的常量，那么这个类文件的属性表中必须存在一个明确的BootstrapMethods属性，另外，即使CONSTANT_InvokeDynamic_info类型的常量在常量池中出现过多次，类文件的属性表中最多也只能有一个BootstrapMethods属性。
BootstrapMethods属性与JSR-292中的InvokeDynamic指令和java.lang.Invoke包关系非常密切，要介绍这个属性的作用，必须先弄清楚 **InovkeDynamic**指令的运作原理。
![BootstrapMethods属性表结构](https://github.com/JP6907/Pic/raw/master/java/class/BootstrapMethods.png)


&nbsp;
> 参考：
《深入理解Java虚拟机》