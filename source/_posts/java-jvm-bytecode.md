---
title: 深入理解Java虚拟机 之 字节码指令
subtitle: 字节码指令
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - java
  - 编程语言
  - jvm
categories:
  - java
date: 2019-10-21 10:34:53
---


# 字节码指令简介
&emsp;Java虚拟机指令由一个字节长度、代表某种特定操作含义的数字(操作码Opcode)，以及跟随在其后的零至多个代表此操作所需的参数(操作数Operands)构成的。由于**Java虚拟机是面向操作数栈而不是寄存器的架构**，所以大多数指令都只有操作码，而没有操作数。

&emsp;字节码指令集是一种具有鲜明特点、优劣势都很突出的指令集架构：
1. 由于限定了Java虚拟机操作码的长度为1个字节，指令集的操作码不能超过256条。
2. Class文件格式放弃了编译后代码中操作数长度对齐，这就意味者虚拟机处理那些超过一个字节数据的时候，不得不在运行的时候从字节码中重建出具体数据的结构。

&emsp;这种操作在一定程度上会降低一些性能，但这样做的优势也非常的明显：
1. 放弃了操作数长度对齐，就意味着可以省略很多填充和间隔符号。
2. 用一个字节来表示操作码，也是为了获取短小精悍的代码。

&emsp;这种追求尽可能小数据量，高传输效率的设计是由Java语言之初面向网络、智能家电技术背景决定的。
&emsp;Java虚拟机解释器执行简单模型如下：
```java
do{
	计算PC寄存器的值+1;
	根据PC寄存器只是位置，从字节码流中取出操作码;
	if(存在操作数) 从字节码中取出操作数;
	执行操作码定义的操作;
}while(字节码长度>0);
```

## 1. 字节码与数据类型
&emsp;在Java虚拟机的指令集中，大多数的指令都包含了其操作所对应的数据类型信息。例如：iload指令用于从局部变量表中加载int型的数据到操作数栈中，而fload指令加载的则是float类型的数据。这两条指令的操作在虚拟机内部可能会是由同一段代码来实现的，但在Class文件中它们必须拥有各自独立的操作码。

&emsp;对于大部分与数据类型相关的字节码指令，它们的操作码助记符中都有特殊的字符来表明专门为哪种数据类型服务：i代表对int类型的数据操作，ｌ代表long，s代表short，b代表byte，c代表char，f代表float，d代表double，a代表reference。注意并不是所有的字节码指令都有数据类型相关，比goto无条件跳转指令就与数据类型无关。这里需要注意的是，并非每种数据类型和每一种操作都有对应的指令。这是因为Java虚拟机的操作码长度只有一个字节，如果每一种与数据类型相关的指令都支持Java虚拟机所有运行时数据类型的话，那指令的数量就会超出一个字节所能表示的数量范围，所以Java虚拟机的指令集对于特定的操作只提供了有限的类型相关指令去支持它。

&emsp;大部分的指令都没有支持整数类型byte、char和short，甚至没有任何指令支持boolean类型。编译器会在编译期或运行期将byte和short类型的数据带符号扩展为相应的int类型数据。与之类似，在处理boolean、byte、short和char类型的数组时，也会转换为使用对应的int类型的字节码指令来处理。因此，大多数对于boolean、byte、short和char类型数据的操作，实际上都是使用相应的int类型作为运算类型。

![Java虚拟机所支持的数据类型1](https://github.com/JP6907/Pic/raw/master/java/jvm/bytecode1)
![Java虚拟机所支持的数据类型2](https://github.com/JP6907/Pic/raw/master/java/jvm/bytecode2)



## 2. 加载和储存指令
&emsp;加载和存储指令用于将数据在栈帧中的局部变量表和操作数栈(见第2章关于内存区域的介绍)之间来回传输,这类指令包括如下内容:
- 将一个局部变量加载到操作栈:iload、iload_\<n>、lload、lload_\<n>、fload、fload_\<n>、dload、dload_\<n>、aload、aload_\<n> 。
- 将一个数值从操作数栈存储到局部变量表 :istore、istore_\<n>、lstore、lstore_\<n>、fstore、fstore_\<n> 、dstore、dstore_\<n> 、astore、astore_\<n> 。
- 将一个常量加载到操作数栈:bipush、sipush、ldc、 ldc_w、ldc2_w、 aconst_null、iconst_ml、iconst_、lconst_\<l>、fconst_\<f>、dconst_\<d>。
- 扩充局部变量表的访问索引的指令 : wide。

&emsp;存储数据的操作数栈和局部变量表主要就是由加载和存储指令进行操作,除此之外,还有少量指令,如访问对象的字段或数组元素的指令也会向操作数栈传输数据。


## 3. 运算指令
&emsp;运算或算术指令用于对两个操作数栈上的值进行某种特定运算,并把结果重新存入到操作栈顶。大体上算术指令可以分为两种:对整型数据进行运算的指令与对浮点型数据进行运算的指令,无论是哪种算术指令,都使用Java虚拟机的数据类型,由于没有直接支持byte、 short、char和boolean类型的算术指令,对于这类数据的运算,应使用操作int类型的指令代替。整数与浮点数的算术指令在溢出和被零除的时候也有各自不同的行为表现,所有的算术指令如下:
- 加法指令:iadd、ladd、fadd、dadd。
- 减法指令:isub、lsub、fsub、dsub。
- 乘法指令:imul、lmul、fmul、dmul。
- 除法指令:idiv、ldiv、fdiv、ddiv。
- 求余指令:irem、lrem、frem、drem。
- 取反指令 : ineg、lneg、fneg、dneg。
- 位移指令:ishl、ishr、iushr、lshl、lshr、lushr。
- 按位或指令:ior、lor。
- 按位与指令:iand、land。
- 按位异或指令:ixor、lxor。
- 局部变量自增指令:iinc。
- 比较指令:dcmpg、dcmpl、fcmpg、fcmpl、lcmp。

&emsp;Java虚拟机规范没有明确定义过整型数据溢出的具体运算结果，仅规定了在处理整型数据时，只有除法指令（idv和ldv）以及求余指令（irem和lrem）中当出现除数为0时会导致虚拟机抛出 ArithmeticExcption 异常，其余任何整型数运算场景都不应该抛出运行时异常。

&emsp;Java虚拟机规范要求虚拟机实现在处理浮点数时,必须严格遵循**IEEE 754**规范中所规定的行为和限制。也就是说,Java虚拟机必须完全支持IEEE 754中定义的非正规浮点数值 ( Denormalized Floating-Point Numbers ) 和逐级下溢( Gradual Underflow ) 的运算规则。这些特征将会使某些数值算法处理起来变得相对容易一些。

&emsp;Java虚拟机要求在进行浮点数运算时,所有的运算结果都必须舍入到适当的精度,非精确的结果必须被舍入为可被表示的最接近的精确值,如果有两种可表示的形式与该值一样接近,将优先选择最低有效位为零的。这种舍入模式也是IEEE 754规范中的默认舍入模式,称为向**最接近数舍入模式**。

&emsp;在把浮点数转换为整数时,Java虚拟机使用正EE 754标准中的**向零舍入模式**,这种模式的舍入结果会导致数字被截断,所有小数部分的有效字节都会被丢弃掉。向零舍入模式将在目标数值类型中选择一个最接近但是不大于原值的数字来作为最精确的舍入结果。

&emsp;另外 ,Java虚拟机在处理浮点数运算时,不会拋出任何运行时异常(这里所讲的是Java 语言中的异常,请读者勿与正EE 754规范中的浮点异常互相混淆, IEEE 754的浮点异常是一种运算信号),当一个操作产生溢出时,将会使用有符号的无穷大来表示,如果某个操作结果没有明确的数学定义的话,将会使用NaN值来表示。所有使用NaN值作为操作数的算术操作 ,结果都会返回NaN。

&emsp;在对long类型数值进行比较时,虚拟机采用带符号的比较方式,而对浮点数值进行比较时 (dcmpg、dcmpl、fcmpg、 fcmpl ) ,虚拟机会采用IEEE 754规范所定义的无信号比较( Nonsignaling Comparisons ) 方式。

## 4. 类型转换指令
&emsp;类型转换指令可以将两种不同的数值类型进行相互转换,这些转换操作一般用于实现用户代码中的显式类型转换操作,或者用来处理本节开篇所提到的字节码指令集中数据类型相关指令无法与数据类型一一对应的问题。
Java 虚拟机直接支持(即转换时无需显式的转换指令)以下数值类型的**宽化类型转换** ( WideningNumeric Conversions , 即小范围类型向大范围类型的安全转换):
- int类型到long、float或者double类型。
- long类型到float、double类型。
- float类型到double类型。

&emsp;相对的,处理**窄化类型转换**( Narrowing Numeric Conversions ) 时 ,必须显式地使用转换指令来完成,这些转换指令包括:i2b、i2c、i2s、l2i、f2i、f2l、d2i、d2l和d2f。窄化类型转换可能会导致转换结果产生不同的正负号、不同的数量级的情况,转换过程很可能会导致数值的精度丢失。

&emsp;在将int或long类型窄化转换为整数类型T的时候,转换过程仅仅是简单地丢弃除最低位N个字节以外的内容(v丢弃高位),N是类型T的数据类型长度,这将可能导致转换结果与输入值有不同的正负号。

&emsp;在将一个浮点值窄化转换为整数类型T ( T限于int或long类型之一 ) 的时候,将遵循以下转换规则:
- 如果浮点值是NaN ,那转换结果就是int或long类型的0。
- 如果浮点值不是无穷大的话,浮点值使用IEEE 754的向零舍入模式取整,获得整数值v , 如果v在目标类型T ( int或long ) 的表示范围之内,那转换结果就是v。
- 否则,将根据v的符号,转换为T所能表示的最大或者最小正数。


## 5. 对象创建与访问指令
- 创建类实例的指令:new。
- 创建数组的指令 : newarray、anewarray、multianewarray。
- 访问类字段(static字段 ,或者称为类变量)和实例字段(非static字段 ,或者称为实例变量 )的指令:getfield、putfield、getstatic、putstatic。
- 把一个数组元素加载到操作数栈的指令:baload、caload、saload、iaload、laload、faloads daload、aaload。
- 将一个操作数栈的值存储到数组元素中的指令:bastore、castore、sastore、iastore、fastore、dastore、aastore。
- 检查类实例类型的指令:instanceof、checkcast。


## 6. 操作数栈管理指令
- 将操作数栈的栈顶一个或两个元素出栈:pop、pop2。
- 复制栈顶一个或两个数值并将复制值或双份的复制值重新压入栈顶:dup、dup2、 dup_x1、dup2_x1、dup_x2、dup2_x2。
- 将栈最顶端的两个数值互换 : swap。


## 7. 控制转移指令
- 条件分支:ifeq、iflt、ifle、ifne、ifgt、ifge、ifnull、ifnonnull、if_icmpeq、if_icmpne、if_icmplt、 if_icmpgt、if_icmple、if_icmpge、if_acmpeq和if_acmpne。
- 复合条件分支:tableswitch、 lookupswitch。
- 无条件分支:goto、goto_w、jsr、jsr_w、ret。

&emsp;在Java虚拟机中有专门的指令集用来处理int和reference类型的条件分支比较操作,为了可以无须明显标识一个实体值是否null,也有专门的指令用来检测null值。

&emsp;与前面算术运算时的规则一致,对于boolean类型、byte类型、char类型和short类型的条件分支比较操作,都是使用int类型的比较指令来完成,而对于long类型、float类型和double类型的条件分支比较操作,则会先执行相应类型的比较运算指令(dcmpg、dcmpl、fcmpg、 fcmpl、lcmp,见6.4.3节),运算指令会返回一个整型值到操作数栈中,随后再执行int类型的条件分支比较操作来完成整个分支跳转。由于**各种类型的比较最终都会转化为int类型的比较操作** ,int类型比较是否方便完善就显得尤为重要,所以Java虚拟机提供的int类型的条件分支指令是最为丰富和强大的。


## 8. 方法调用和返回指令
- invokevirtual 指令用于调用对象的实例方法,根据对象的实际类型进行分派(虚方法分派),这也是Java语言中最常见的方法分派方式。
- invokeinterface 指令用于调用接口方法,它会在运行时搜索一个实现了这个接口方法的对象 ,找出适合的方法进行调用。
- invokespecial 指令用于调用一些需要特殊处理的实例方法,包括实例初始化方法、私有方法和父类方法。
- invokestatic 指令用于调用类方法(static方法)。
- invokedynamic 指令用于在运行时动态解析出调用点限定符所引用的方法,并执行该方法 ,前面4条调用指令的分派逻辑都固化在Java虚拟机内部,而invokedynamic指令的分派逻辑是由用户所设定的引导方法决定的。


## 9. 异常处理指令
&emsp;在Java程序中显式拋出异常的操作(**throw**语句)都由**athrow**指令来实现,除了用throw语句显式拋出异常情况之外,Java虛拟机规范还规定了许多运行时异常会在其他Java虚拟机指令检测到异常状况时自动拋出。例如 ,在前面介绍的整数运算中,当除数为零时,虚拟机会在idiv或ldiv指令中拋出ArithmeticException异常。

&emsp;而在Java虚拟机中,处理异常(catch语句)不是由字节码指令来实现的(很久之前曾经使用jsr和ret指令来实现,现在已经不用了),而是使用**异常表**来完成的。


## 10. 同步指令
&emsp;Java虚拟机可以支持方法级的同步和方法内部一段指令序列的同步,这两种同步结构都是使用**管程(Monitor)**来支持的。

&emsp;方法级的同步是隐式的,即无须通过字节码指令来控制,它实现在方法调用和返回操作之中。虛拟机可以从方法常量池的方法表结构中的**ACC_SYNCHRONIZED**访问标志得知一个方法是否声明为同步方法。当方法调用时,调用指令将会检查方法的ACC_SYNCHRONIZED 访问标志是否被设置,如果设置了,执行线程就要求先成功持有管程,然后才能执行方法, 最后当方法完成(无论是正常完成还是非正常完成)时释放管程。在方法执行期间,执行线程持有了管程,其他任何线程都无法再获取到同一个管程。如果一个同步方法执行期间拋出了异常,并且在方法内部无法处理此异常,那么这个同步方法所持有的管程将在异常拋到同步方法之外时**自动释放**。

&emsp;同步一段指令集序列通常是由Java语言中的synchronized语句块来表示的,Java虚拟机的指令集中有**monitorenter**和**monitorexit**两条指令来支持synchronized关键字的语义,正确实现synchronized关键字需要Javac编译器与Java虚拟机两者共同协作支持。
```java
public void onlyMe(Foo f){
    synchronized (f){
        doSomething();
    }
}
```
&emsp;编译之后生成的字节码序列：
```
public void onlyMe(com.jp.jvm.ch06.TestClass$Foo);
        descriptor: (Lcom/jp/jvm/ch06/TestClass$Foo;)V
        flags: ACC_PUBLIC
        Code:
        stack=2, locals=4, args_size=2
        0: aload_1                       //将对象f入栈
        1: dup                           //复制栈顶元素（即f的引用）
        2: astore_2                      //将栈顶元素储存到局部变量表Slot2中
        3: monitorenter                  //已栈顶元素(f)作为锁，开始同步
        4: aload_0                       //将局部变量Slot0（即this指针）的元素入栈
        5: invokevirtual #3              //调用this.doSomething方法
        8: aload_2                       //将局部变量Slot2（即f）入栈
        9: monitorexit                   //释放f的监视器锁，退出同步
        10: goto          18             //方法正常结束，跳转到18返回  
        13: astore_3                     //从这步开始是异常路径，，见下面异常表Taget13
        14: aload_2                      //将局部变量Slot2（即f）入栈
        15: monitorexit                  //退出同步
        16: aload_3                      //将局部变量Slot3的元素（即异常对象）入栈
        17: athrow                       //把异常抛出
        18: return                       //方法正常返回
        Exception table:                 //异常表
        from    to  target type
        4    10    13   any
        13    16    13   any
```
&emsp;编译器必须确保无论方法通过何种方式完成,方法中调用过的每条monitorenter指令都必须执行其对应的monitorexit指令 ,而无论这个方法是正常结束还是异常结束。

&emsp;从上面字节码序列中可以看到,为了保证在方法异常完成时momtorenter和monitorexit指令依然可以正确配对执行,编译器会自动产生一个异常处理器,这个异常处理器声明可处理所有的异常,它的目的就是用来执行momtorexit指令。


&nbsp;
> 参考：
《深入理解Java虚拟机》