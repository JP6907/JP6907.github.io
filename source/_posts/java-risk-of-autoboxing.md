---
title: Java 自动装箱的陷阱
catalog: true
date: 2019-09-18 19:42:02
subtitle: 自动装箱
header-img: /img/article_header/article_header.png
tags:
- java
- 编程语言
- jvm
categories:
- java
---

# Java 自动装箱的陷阱

## 1. 自动装箱
&emsp;自动装箱、拆箱是Java语言中使用的最多的语法糖之一。装箱就是自动将基本数据类型转换为包装器类型；拆箱就是自动将包装器类型转换为基本数据类型。
```java
public class Main {
    public static void main(String[] args) {
    //自动装箱
    Integer total = 99;
    //自定拆箱
    int totalprim = total;
    }
}
```
&emsp;下面看一下需要装箱拆箱的类型有哪些：
![java-auto-boxing-type](https://gitee.com/JP6907/Pic/raw/master/java/java-auto-boxing-type)

## 2. 自动装箱的过程
&emsp;下面我们看一下自动装箱、拆箱的具体过程。执行下面语句，查看 main 函数的字节码
```shell
javac AutoBoxing.java
javap -verbose AutoBoxing
```
&emsp;下面是 main 函数的字节码：
```
public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
            flags: ACC_PUBLIC, ACC_STATIC
            Code:
            stack=1, locals=3, args_size=1
            0: bipush        99
            2: invokestatic  #2                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
            5: astore_1
            6: aload_1
            7: invokevirtual #3                  // Method java/lang/Integer.intValue:()I
            10: istore_2
            11: return
            LineNumberTable:
            line 7: 0
            line 9: 6
            line 10: 11
```
&emsp;bipush 将 99 放到栈顶，接下来使用 invokestatic，这里会调用了 Integer.valueOf 的静态方法，栈顶的数据作为参数，即这里实际执行的是 Integer.valueOf(99) 语句，这就是自动装箱的过程。接下来 astore_1 将 total 的引用放到栈顶，aload_1 将栈顶total的引用加载到了 slot1，接下来 invokevirtual 会调用 Integer.intValue 方法，即 total.intValue() 方法，istore_2 将该值放到了 slot2，即赋值给了totalprim，这里就完成了拆箱的过程。

## 3. 自动装箱的陷阱
&emsp;自动装箱、拆箱这些语法糖看似很简单，但这里面有很多我们需要注意的地方，下面代码演示了自动装箱的一些错误用法：
```java
public static void main(String[] args){
        Integer a = 1;  
        Integer b = 2;  
        Integer c = 3;  
        Integer d = 3;  
        Integer e = 128; 
        Integer f = 128;  
        Long g = 3L;
        System.out.println(c == d); //true  
        System.out.println(e == f); //false  
        System.out.println(c == (a+b)); //true  
        System.out.println(c.equals(a+b)); //true  
        System.out.println(g == (a+b)); //true 
        System.out.println(g.equals(a+b)); //false 
    }
```
&emsp;这样的答案可能会出乎很多人的意料，接下来一一分析。
&emsp;首先明确一下 “\==” 和 equals 方法的用法。
&emsp;"\=="：如果是基本数据类型，则直接对值进行比较，如果是引用数据类型，则是对他们的地址进行比较（但是只能比较相同类型的对象，或者比较父类对象和子类对象。类型不同的两个对象不能使用==）。
&emsp;equals：装箱类型重写了 equals 方法，比较规则为：如果两个对象的类型一致，并且内容一致，则返回true。以 Integer 为例：
```java
//Integer.java
public boolean equals(Object obj) {
        if (obj instanceof Integer) {
            return value == ((Integer)obj).intValue();
        }
        return false;
    }
```
&emsp;由前面分析可知，自动装箱的时候实际调用了 Integer.valueOf 方法：
```java
//Integer.java
public static Integer valueOf(int i) {
        if (i >= IntegerCache.low && i <= IntegerCache.high)
            return IntegerCache.cache[i + (-IntegerCache.low)];
        return new Integer(i);
    }
```
&emsp;IntegerCache 是 Integer 内部定义的一个私有静态内部类：
```java
private static class IntegerCache {
        static final int low = -128;
        static final int high;
        static final Integer cache[];

        static {
            // high value may be configured by property
            int h = 127;
            String integerCacheHighPropValue =
                sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
            if (integerCacheHighPropValue != null) {
                try {
                    int i = parseInt(integerCacheHighPropValue);
                    i = Math.max(i, 127);
                    // Maximum array size is Integer.MAX_VALUE
                    h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
                } catch( NumberFormatException nfe) {
                    // If the property cannot be parsed into an int, ignore it.
                }
            }
            high = h;

            cache = new Integer[(high - low) + 1];
            int j = low;
            for(int k = 0; k < cache.length; k++)
                cache[k] = new Integer(j++);

            // range [-128, 127] must be interned (JLS7 5.1.7)
            assert IntegerCache.high >= 127;
        }

        private IntegerCache() {}
    }
```
&emsp;通过观察上面的代码我们可以发现，Integer使用一个内部静态类中的一个静态数组（Integer cache[]）保存了-128-127范围内的数据，静态数组在类加载以后是存在方法区的，并不是什么常量池。在自动装箱的时候，首先判断要装箱的数字的范围，如果在-128-127的范围则直接返回缓存中已有的对象，否则new一个新的对象。其他的包装类也有类似的实现方式。 
&emsp;需要注意几点：
- 包装类的 == 运算在不遇到算术运算的情况下不会自动拆箱
- 类之间 == 比较的是地址
- 包装类的 equals 方法不处理数据转型的关系

&emsp;因此，我们有以下分析：
```java
Integer a = 1;  //直接获取IntegerCache内部的对象
Integer b = 2;  //直接获取IntegerCache内部的对象
Integer c = 3;  //直接获取IntegerCache内部的对象
Integer d = 3;  //直接获取IntegerCache内部的对象
Integer e = 128; //new一个新的对象
Integer f = 128;  //new一个新的对象
Long g = 3L;
```
```java
System.out.println(c == d); //true  是同一个对象
```
c 和 d 指向的是 IntegerCache 内部同一个对象，地址一样，== 比较的是地址。
```java
System.out.println(e == f); //false   //128开始就false，不同的对象
```
e 和 f 大于127，都是 new 生成的新对象，值相同，但是对应不同的内存地址。
```java
System.out.println(c == (a+b)); //true  //拆箱变成基本类型
```
(a+b) 触发自动拆箱，自动拆箱后是基本类型，它们的值是相等的。
```java
System.out.println(c.equals(a+b)); //true  //Integer和Integer类型一致，数值也一样
```
equals 的参数是 Object，(a+b)会触发自动拆箱，结果变成基本类型，传进 equals 方法的时候会执行自动装箱，结果和 c 的值相同，而且类型相同，返回true。
```java
System.out.println(g == (a+b)); //true  //拆箱子
```
(a+b) 触发自动拆箱，自动拆箱后是基本类型，它们的值是相等的。
```java
System.out.println(g.equals(a+b)); //false  //Long和Integer类型不一样
```
(a+b)传进 equals 方法，自动装箱变成 Integer，值相同但是类型不同，返回false。

## 4. 总结
&emsp;鉴于包装类的“\==”运算在不遇到算数运算的情况下不会自动拆箱，以及它们的equals方法不处理数据转型的关系，我们在实际编码的时候应该尽量避免这样使用自动装箱与拆箱。


&nbsp;
&nbsp;
>本文作者：ZJP
版权声明：本文为博主原创文章，遵循 [CC 4.0 BY-SA](http://creativecommons.org/licenses/by-sa/4.0/) 版权协议，转载请附上原文出处链接和本声明。
本文链接：http://zhoujiapeng.top/java/java-risk-of-autoboxing/