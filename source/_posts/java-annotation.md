---
title: Java Annotation
subtitle: Annotation
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - java
  - 编程语言
  - jdk
categories:
  - java
date: 2019-10-20 11:24:58
---


# Java Annotation
## 1. 概述
&emsp;Java 注解（Annotation）又称 Java 标注，是 JDK5.0 引入的一种注释机制。
&emsp;Java 语言中的类、方法、变量、参数和包等都可以被标注。和 Javadoc 不同，Java 标注可以通过反射获取标注内容。在编译器生成类文件时，标注可以被嵌入到字节码中。Java 虚拟机可以保留标注内容，在运行时可以获取到标注内容 。 当然它也支持自定义 Java 标注。

&emsp;Java 内置定义了一套注解，共有 7 个，3 个在 java.lang 中，剩下 4 个在 java.lang.annotation 中。

&emsp;作用在代码的注解是：
- @Override - 检查该方法是否是重载方法。如果发现其父类，或者是引用的接口中并没有该方法时，会报编译错误。
- @Deprecated - 标记过时方法。如果使用该方法，会报编译警告。
- @SuppressWarnings - 指示编译器去忽略注解中声明的警告。

&emsp;作用在其他注解的注解(或者说 元注解)是:
- @Retention - 标识这个注解怎么保存，是只在代码中，还是编入class文件中，或者是在运行时可以通过反射访问。
- @Documented - 标记这些注解是否包含在用户文档中。
- @Target - 标记这个注解应该是哪种 Java 成员。
- @Inherited - 标记这个注解是继承于哪个注解类(默认 注解并没有继承于任何子类)

&emsp;从 Java 7 开始，额外添加了 3 个注解:
- @SafeVarargs - Java 7 开始支持，忽略任何使用参数为泛型变量的方法或构造函数调用产生的警告。
- @FunctionalInterface - Java 8 开始支持，标识一个匿名函数或函数式接口。
- @Repeatable - Java 8 开始支持，标识某注解可以在同一个声明上使用多次


## 2. Annotation 架构
![Annotation架构](https://gitee.com/JP6907/Pic/raw/master/java/annotation.jpg)

&emsp;从中，我们可以看出：
- 1 个 Annotation 和 1 个 RetentionPolicy 关联。
可以理解为：每1个Annotation对象，都会有唯一的RetentionPolicy属性。
- 1 个 Annotation 和 1~n 个 ElementType 关联。
可以理解为：对于每 1 个 Annotation 对象，可以有若干个 ElementType 属性。
- Annotation 有许多实现类，包括：Deprecated, Documented, Inherited, Override 等等。

&emsp;Annotation 的每一个实现类，都 "和 1 个 RetentionPolicy 关联" 并且 " 和 1~n 个 ElementType 关联"。
&emsp;图的左半部分是 Annotation 的组成部分，右半部分是 Annotation 的实现类。


## 3. Annotation 组成部分
&emsp;Java Annotation 的组成中，有 3 个非常重要的主干类。它们分别是：
### 3.1 Annotation.java
```java
package java.lang.annotation;
public interface Annotation {

    boolean equals(Object obj);

    int hashCode();

    String toString();

    Class<? extends Annotation> annotationType();
}
```
&emsp;Annotation 就是个接口。
&emsp;"每 1 个 Annotation" 都与 "1 个 RetentionPolicy" 关联，并且与 "1～n 个 ElementType" 关联。可以通俗的理解为：每 1 个 Annotation 对象，都会有唯一的 RetentionPolicy 属性；至于 ElementType 属性，则有 1~n 个。

### 3.2 ElementType.java
```java
package java.lang.annotation;
public enum ElementType {
    TYPE,               /* 类、接口（包括注释类型）或枚举声明  */

    FIELD,              /* 字段声明（包括枚举常量）  */

    METHOD,             /* 方法声明  */

    PARAMETER,          /* 参数声明  */

    CONSTRUCTOR,        /* 构造方法声明  */

    LOCAL_VARIABLE,     /* 局部变量声明  */

    ANNOTATION_TYPE,    /* 注释类型声明  */

    PACKAGE             /* 包声明  */
}
```
&emsp;ElementType 是 Enum 枚举类型，它用来指定 Annotation 的类型，表示该注解可以用来修饰什么，如类、方法、属性等。
&emsp;"每 1 个 Annotation" 都与 "1～n 个 ElementType" 关联。当 Annotation 与某个 ElementType 关联时，就意味着：Annotation有了某种用途。例如，若一个 Annotation 对象是 METHOD 类型，则该 Annotation 只能用来修饰方法。

### 3.3 RetentionPolicy.java
```java
package java.lang.annotation;
public enum RetentionPolicy {
    SOURCE,            /* Annotation信息仅存在于编译器处理期间，编译器处理完之后就没有该Annotation信息了  */

    CLASS,             /* 编译器将Annotation存储于类对应的.class文件中。默认行为  */

    RUNTIME            /* 编译器将Annotation存储于class文件中，并且可由JVM读入 */
}
```
&emsp;RetentionPolicy 是 Enum 枚举类型，它用来指定 Annotation 的策略。通俗点说，就是不同 RetentionPolicy 类型的 Annotation 的作用域不同。
"每 1 个 Annotation" 都与 "1 个 RetentionPolicy" 关联。
- a) 若 Annotation 的类型为 SOURCE，则意味着：Annotation 仅存在于编译器处理期间，编译器处理完之后，该 Annotation 就没用了。 例如，" @Override" 标志就是一个 Annotation。当它修饰一个方法的时候，就意味着该方法覆盖父类的方法；并且在编译期间会进行语法检查！编译器处理完后，"@Override" 就没有任何作用了。
- b) 若 Annotation 的类型为 CLASS，则意味着：编译器将 Annotation 存储于类对应的 .class 文件中，它是 Annotation 的默认行为。
- c) 若 Annotation 的类型为 RUNTIME，则意味着：编译器将 Annotation 存储于 class 文件中，并且可由JVM读入，会在运行时起作用。


## 4. Annotation 实现类
&emsp;&emsp;理解了上面的3个类的作用之后，我们接下来可以讲解Annotation实现类的语法定义了。

### 4.1 Annotation 通用定义
我们可以这样定义一个注解：
```java
@Documented
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface MyAnnotation1 {
    public String key() default "Hello";
}
```
&emsp;上面的作用是定义一个 Annotation，它的名字是 MyAnnotation1。定义了 MyAnnotation1 之后，我们可以在代码中通过 "@MyAnnotation1" 来使用它。 其它的，@Documented, @Target, @Retention, @interface 都是来修饰 MyAnnotation1 的。下面分别说说它们的含义：

1. @interface

&emsp;使用 @interface 定义注解时，意味着它实现了 java.lang.annotation.Annotation 接口，即该注解就是一个Annotation。
定义 Annotation 时，@interface 是必须的。
注意：它和我们通常的 implemented 实现接口的方法不同。Annotation 接口的实现细节都由编译器完成。通过 @interface 定义注解后，该注解不能继承其他的注解或接口。

2. @Documented

&emsp;类和方法的注释在缺省情况下是不出现在 javadoc 中的。如果使用 @Documented 修饰该 Annotation，则表示它可以出现在 javadoc 中。
定义 Annotation 时，@Documented 可有可无；若没有定义，则 注释不会出现在 javadoc 中。

3. @Target(ElementType.TYPE)

&emsp;前面我们说过，ElementType 是 Annotation 的类型属性。而 @Target 的作用，就是来指定 Annotation 的类型属性。
@Target(ElementType.TYPE) 的意思就是指定该 Annotation 的类型是 ElementType.TYPE。这就意味着，MyAnnotation1 是来修饰"类、接口（包括注释类型）或枚举声明"的注解。
定义 Annotation 时，@Target 可有可无。若有 @Target，则该 Annotation 只能用于它所指定的地方；若没有 @Target，则该 Annotation 可以用于任何地方。

4. @Retention(RetentionPolicy.RUNTIME)

&emsp;前面我们说过，RetentionPolicy 是 Annotation 的策略属性，而 @Retention 的作用，就是指定 Annotation 的策略属性。
@Retention(RetentionPolicy.RUNTIME) 的意思就是指定该 Annotation 的策略是 RetentionPolicy.RUNTIME。这就意味着，编译器会将该 Annotation 信息保留在 .class 文件中，并且能被虚拟机读取。
定义 Annotation 时，@Retention 可有可无。若没有 @Retention，则默认是 RetentionPolicy.CLASS。


### 4.2 Java 自带的 Annotation
&emsp;通过上面的示例，我们能理解：@interface 用来声明 Annotation，@Documented 用来表示该 Annotation 是否会出现在 javadoc 中， @Target 用来指定 Annotation 的类型，@Retention 用来指定 Annotation 的策略。

&emsp;java 常用的 Annotation：
```
@Deprecated  -- @Deprecated 所标注内容，不再被建议使用。
@Override    -- @Override 只能标注方法，表示该方法覆盖父类中的方法。
@Documented  -- @Documented 所标注内容，可以出现在javadoc中。
@Inherited   -- @Inherited只能被用来标注“Annotation类型”，它所标注的Annotation具有继承性，父类的Annotation可以被子类继承。
@Retention   -- @Retention只能被用来标注“Annotation类型”，而且它被用来指定Annotation的RetentionPolicy属性。
@Target      -- @Target只能被用来标注“Annotation类型”，而且它被用来指定Annotation的ElementType属性。
@SuppressWarnings -- @SuppressWarnings 所标注内容产生的警告，编译器会对这些警告保持静默。
```

1. @Deprecated
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(value={CONSTRUCTOR, FIELD, LOCAL_VARIABLE, METHOD, PACKAGE, PARAMETER, TYPE})
public @interface Deprecated {
}
```
&emsp;若某个方法被 @Deprecated 标注，则该方法不再被建议使用。如果有开发人员试图使用或重写被 @Deprecated 标示的方法，编译器会给相应的提示信息。示例如下:
![](https://gitee.com/JP6907/Pic/raw/master/java/annotation-Deprecated.jpg)


2. @Inherited
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Inherited {
}
```
&emsp;假设，我们定义了某个 Annotaion，它的名称是 MyAnnotation，并且 MyAnnotation 被标注为 @Inherited。现在，某个类 Base 使用了MyAnnotation，则 Base 具有了"具有了注解 MyAnnotation"；现在，Sub 继承了 Base，由于 MyAnnotation 是 @Inherited的(具有继承性)，所以，Sub 也 "具有了注解 MyAnnotation"。
```java
import java.lang.annotation.Target;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Inherited;

/**
 * 自定义的Annotation。
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@interface Inheritable
{
}

@Inheritable
class InheritableFather
{
    public InheritableFather() {
        // InheritableBase是否具有 Inheritable Annotation
        System.out.println("InheritableFather:"+InheritableFather.class.isAnnotationPresent(Inheritable.class));
    }
}

/**
 * InheritableSon 类只是继承于 InheritableFather，
 */
public class InheritableSon extends InheritableFather
{
    public InheritableSon() {
        super();    // 调用父类的构造函数
        // InheritableSon类是否具有 Inheritable Annotation
        System.out.println("InheritableSon:"+InheritableSon.class.isAnnotationPresent(Inheritable.class));
    }
    
    public static void main(String[] args)
    {
        InheritableSon is = new InheritableSon();
    }
}
```
&emsp;运行结果：
```java
InheritableFather:true
InheritableSon:true  //如果去掉Inheritable的@Inherited，则为false
```

3. @SuppressWarnings
```java
@Target({TYPE, FIELD, METHOD, PARAMETER, CONSTRUCTOR, LOCAL_VARIABLE})
@Retention(RetentionPolicy.SOURCE)
public @interface SuppressWarnings {
    String[] value();
}
```
&emsp;SuppressWarnings 的作用是，让编译器对"它所标注的内容"的某些警告保持静默。例如，"@SuppressWarnings(value={"deprecation", "unchecked"})" 表示对"它所标注的内容"中的 "SuppressWarnings 不再建议使用警告"和"未检查的转换时的警告"保持沉默。示例如下：

![](https://gitee.com/JP6907/Pic/raw/master/javaannotation-SuppressWarnings1.jpg)
![](https://gitee.com/JP6907/Pic/raw/master/java/annotation-SuppressWarnings2.jpg)

&emsp;如果没有使用 @SuppressWarnings(value={"deprecation"}) , 而 Date 属于 java 不再建议使用的类。因此，调用 Date 的 API 时，会产生警告。如果使用了 @SuppressWarnings(value={"deprecation"})。编译器对"调用 Date 的 API 产生的警告"保持沉默。

&emsp;SuppressWarnings 常用的关键字的表格：
```
deprecation  -- 使用了不赞成使用的类或方法时的警告
unchecked    -- 执行了未检查的转换时的警告，例如当使用集合时没有用泛型 (Generics) 来指定集合保存的类型。
fallthrough  -- 当 Switch 程序块直接通往下一种情况而没有 Break 时的警告。
path         -- 在类路径、源文件路径等中有不存在的路径时的警告。
serial       -- 当在可序列化的类上缺少 serialVersionUID 定义时的警告。
finally      -- 任何 finally 子句不能正常完成时的警告。
all          -- 关于以上所有情况的警告。
```

4. @Override
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.SOURCE)
public @interface Override {
}
```
&emsp;若某个方法被 @Override 的标注，则意味着该方法会覆盖父类中的同名方法。如果有方法被 @Override 标示，但父类中却没有"被 @Override 标注"的同名方法，则编译器会报错。

## 4. Annotation 的作用
&emsp;Annotation 是一个辅助类，它在 Junit、Struts、Spring 等工具框架中被广泛使用。
&emsp;我们在编程中经常会使用到的 Annotation 作用有：

### 4.1 编译检查
&emsp;Annotation 具有"让编译器进行编译检查的作用"。
例如，@SuppressWarnings, @Deprecated 和 @Override 都具有编译检查作用。

1. 关于 @SuppressWarnings 和 @Deprecated，已经在"第3部分"中详细介绍过了。这里就不再举例说明了。
2. 若某个方法被 @Override 的标注，则意味着该方法会覆盖父类中的同名方法。如果有方法被 @Override 标示，但父类中却没有"被 @Override 标注"的同名方法，则编译器会报错。


### 4.2 在反射中使用 Annotation
&emsp;在反射的 Class, Method, Field 等函数中，有许多于 Annotation 相关的接口。

&emsp;这也意味着，我们可以在反射中解析并使用 Annotation。
```java
import java.lang.annotation.Annotation;
import java.lang.annotation.Target;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Inherited;
import java.lang.reflect.Method;

/**
 * Annotation在反射函数中的使用示例
 */
@Retention(RetentionPolicy.RUNTIME)
@interface MyAnnotation {
    String[] value() default "unknown";
}

/**
 * Person类。它会使用MyAnnotation注解。
 */
class Person {
    
    /**
     * empty()方法同时被 "@Deprecated" 和 "@MyAnnotation(value={"a","b"})"所标注 
     * (01) @Deprecated，意味着empty()方法，不再被建议使用
     * (02) @MyAnnotation, 意味着empty() 方法对应的MyAnnotation的value值是默认值"unknown"
     */
    @MyAnnotation
    @Deprecated
    public void empty(){
        System.out.println("\nempty");
    }
    
    /**
     * sombody() 被 @MyAnnotation(value={"girl","boy"}) 所标注，
     * @MyAnnotation(value={"girl","boy"}), 意味着MyAnnotation的value值是{"girl","boy"}
     */
    @MyAnnotation(value={"girl","boy"})
    public void somebody(String name, int age){
        System.out.println("\nsomebody: "+name+", "+age);
    }
}

public class AnnotationTest {

    public static void main(String[] args) throws Exception {
        
        // 新建Person
        Person person = new Person();
        // 获取Person的Class实例
        Class<Person> c = Person.class;
        // 获取 somebody() 方法的Method实例
        Method mSomebody = c.getMethod("somebody", new Class[]{String.class, int.class});
        // 执行该方法
        mSomebody.invoke(person, new Object[]{"lily", 18});
        iteratorAnnotations(mSomebody);
        

        // 获取 somebody() 方法的Method实例
        Method mEmpty = c.getMethod("empty", new Class[]{});
        // 执行该方法
        mEmpty.invoke(person, new Object[]{});        
        iteratorAnnotations(mEmpty);
    }
    
    public static void iteratorAnnotations(Method method) {

        // 判断 somebody() 方法是否包含MyAnnotation注解
        if(method.isAnnotationPresent(MyAnnotation.class)){
            // 获取该方法的MyAnnotation注解实例
            MyAnnotation myAnnotation = method.getAnnotation(MyAnnotation.class);
            // 获取 myAnnotation的值，并打印出来
            String[] values = myAnnotation.value();
            for (String str:values)
                System.out.printf(str+", ");
            System.out.println();
        }
        
        // 获取方法上的所有注解，并打印出来
        Annotation[] annotations = method.getAnnotations();
        for(Annotation annotation : annotations){
            System.out.println(annotation);
        }
    }
}
```
&emsp;运行结果：
```
somebody: lily, 18
girl, boy, 
@com.skywang.annotation.MyAnnotation(value=[girl, boy])

empty
unknown, 
@com.skywang.annotation.MyAnnotation(value=[unknown])
@java.lang.Deprecated()
```

### 4.3 根据 Annotation 生成帮助文档
&emsp;通过给 Annotation 注解加上 @Documented 标签，能使该 Annotation 标签出现在 javadoc 中。
```java
@Documented
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface MyAnnotation1 {
    public String key() default "Hello";
}

/**
 * 本方法是覆写Object的toString方法
 */
@MyAnnotation
public String toString(){
    return "hello"
}
```
加入文档注释后，会在doc文档中出现(使用javadoc文档生成)。


### 4.4 能够帮忙查看查看代码
&emsp;通过 @Override, @Deprecated 等，我们能很方便的了解程序的大致结构。
&emsp;另外，我们也可以通过自定义 Annotation 来实现一些功能。


&nbsp;
> 参考：
https://www.runoob.com/w3cnote/java-annotation.html