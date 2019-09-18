---
title: scala 尾递归优化
subtitle: scala.annotation.tailrec
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - scala
  - 编程语言
categories:
  - scala
date: 2019-09-18 15:28:08
---


# scala 尾递归优化

## 1. 递归
### 1.1 递归的定义
&emsp;一个函数直接或间接的调用它自己本身，就是递归。它通常把一个大型复杂的问题层层转化为一个与原问题相似的规模较小的问题来求解，递归策略只需少量的代码就可以执行多次重复的计算。

### 1.2 递归的条件
&emsp;一般来说，递归需要有边界条件、递归前进段和递归返回段。当边界条件不满足时，递归前进；当边界条件满足时，递归返回。
&emsp;以递归方式实现阶乘函数的实现：
```scala
def factorial(n:Int): Long ={
    if(n <= 0) 1
    else n * factorial(n-1)
}
```
if(n <= 0) 1是递归返回段，else后面部分是递归前进段。它的调用过程大致如下：
```
factorial(5)
5 * factorial(4)
5 * (4 * factorial(3))
5 * (4 * (3 * factorial(2)))
5 * (4 * (3 * (2 * factorial(1))))
5 * (4 * (3 * (2 * 1)))
```

### 1.3 递归的缺点
- 需要保持调用堆栈,如代码1-1,每一次递归都要保存n*factorial(n-1)栈帧信息。如果调用次数太多，可能会导致栈溢出。
- 效率会比较低，递归就是不断的调用自己本身，如果方法本身比较复杂，每次调用自己效率会较低。

## 2. 尾递归
### 2.1 定义
&emsp;尾递归是指递归调用是函数的最后一个语句，而且其结果被直接返回，这是一类特殊的递归调用。由于递归结果总是直接返回，尾递归比较方便转换为循环，因此编译器容易对它进行优化。现在很多编译器都对尾递归有优化，程序员们不必再手动将它们改写为循环。
&emsp;我们可以这样理解尾递归
- 所有递归形式的调用都出现在函数的末尾。
- 递归调用是整个函数体中最后执行的语句且它的返回值不属于表达式的一部分。

&emsp;1.2中代码最后递归调用 factorial(n-1) ,但它是整个函数返回值表达式{n * factorial(n-1)} 的一部分，因此它不是尾递归。
&emsp;我们可以使用尾递归来实现上面的阶乘：
```scala
def factorial(n:Int):Long = {
    @tailrec
    def factorial(n:Int,aggr:Int): Long ={
        if(n <= 0) aggr
        else factorial(n-1,n*aggr)
    }
    
   factorial(n,1)
}
```
&emsp;调用过程大致如下：
```
factorialTailrec(5, 1)
factorialTailrec(4, 5)  // 1 * 5 = 5
factorialTailrec(3, 20) // 5 * 4 = 20
factorialTailrec(3, 60) // 20 * 3 = 60
factorialTailrec(2, 120) // 60 * 2 = 120
factorialTailrec(1, 120) // 120 * 1 = 120120
```
&emsp;以上的调用，由于调用结果都是直接返回，所以之前的递归调用留在堆栈中的数据可以丢弃，只需要保留最后一次的数据，这就是尾递归容易优化的原因所在。
&emsp;尾递归的核心思想是通过参数来传递每一次的调用结果，达到不压栈。它维护着一个迭代器和一个累加器。

### 2.2 循环
&emsp;事实上，scala都是将尾递归直接编译成循环模式的。所以我们可以大胆的说，所有的循环模式都能改写为尾递归的写法模式。我们可以看一下阶乘算法的递归版本
```scala
def fibfor(n:Int): Int ={
    var m = 1
    for (i <- 1 to n) {
        m = m * i
    }
    m
}
```

### 2.2 普通递归转化为尾递归
&emsp;尾递归会维护一个或多个累计值(aggregate)参数和一个迭代参数。
- 累计值参数aggregate将每次循环产生的结果进行累计，然后传到下一次的调用中。
- 迭代器，和普通递归或循环一样，每次递归或循环后，改变一次。(如for(i=0;i<1-;i++)里面的i)。
&emsp;将普通的递归改写为尾递归，关键在于找到合适的累加器。下面我们以斐波那契数列为例，看看如何找到累加器。斐波那契数列，前两项为1，从第三项起，每一项都是它之前的两项和。这个定义就是天然的递归算法，如下:
```scala
def fibonacci(n: Int): Int = {
  if (n <= 2) 1
  else fibonacci(n - 1) + fibonacci(n - 2)
}
```
&emsp;调用过程为：
```scala
fibonacci(5)
fibonacci(4) + fibonacci(3)
(fibonacci(3) + fibonacci(2)) + (fibonacci(2) + fibonacci(1))
((fibonacci(2) + fibonacci(1)) + 1) + (1 + 1)
((1 + 1) + 1) + 2
```
&emsp;上面显然不是尾递归，如何找到累加器将它改造为尾递归？因为需要前两项的和，所以这里需要两个累加器，假设较小的一个为acc1，较大的一个为acc2，需要计算下一项时，将acc2的值赋给acc1，而（acc1+acc2)赋值给acc2，这样，调用堆栈中旧有的数据即可丢弃。
```scala
def fibonacciTailrec(n: Int, acc1: Int, acc2: Int): Int = {
  if (n < 2) acc2
  else fibonacciTailrec(n - 1, acc2, acc1 + acc2)
}
```
&emsp;调用过程为：
```scala
fibonacciTailrec(5,0,1)
fibonacciTailrec(4,1,1)
fibonacciTailrec(3,1,2)
fibonacciTailrec(2,2,3)
fibonacciTailrec(1,3,5)
```

## 3. Scala对尾递归的支持
### 3.1 使用方法
&emsp;Scala对形式上严格的尾递归进行了优化，对于严格的尾递归，可以放心使用，不必担心性能问题。对于是否是严格尾递归，若不能自行判断， 可使用Scala提供的尾递归标注@scala.annotation.tailrec，这个符号除了可以标识尾递归外，更重要的是编译器会检查该函数是否真的尾递归，若不是，会导致如下编译错误:
```
could not optimize @tailrec annotated method fibonacci: it contains a recursive call not in tail position
```

### 3.2 编译器对尾递归的优化
&emsp;当编译器检测到一个函数调用是尾递归的时候，它就覆盖当前的活动记录而不是在栈中去创建一个新的。scala编译器会察觉到尾递归，并对其进行优化，将它编译成循环的模式。

### 3.3 Scala尾递归的限制
- 尾递归有严格的要求，就是最后一个语句是递归调用，因此写法比较严格。
- 尾递归最后调用的必须是它本身，间接的赋值它本身的函数也无法进行优化。

## 4. 性能测试
&emsp;下面进行fibonacci算法的普通递归和尾递归时间测试对比：
&emsp;测试代码：
```scala
def main(args: Array[String]): Unit = {
    val n = 20

    println(s"n = $n")
    var startTime = System.nanoTime              //系统纳米时间
    println(fibonacci(n))
    var endTime =  System.nanoTime
    println(s"fibonacci time:${(endTime - startTime)/1000000d}ms")

    startTime = System.nanoTime
    println(fibonacciTailrec(n,0,1))
    endTime =  System.nanoTime
    println(s"fibonacciTailrec time:${(endTime - startTime)/1000000d}ms")
    
  }

  def fibonacci(n: Int): Int = {
    if (n <= 2) 1
    else fibonacci(n - 1) + fibonacci(n - 2)
  }


  @tailrec
  def fibonacciTailrec(n: Int, acc1: Int, acc2: Int): Int = {
    if (n < 2) acc2
    else fibonacciTailrec(n - 1, acc2, acc1 + acc2)
  }
```
&emsp;测试结果：
```scala
=========
n = 20
6765
fibonacci time:0.611397ms
6765
fibonacciTailrec time:0.025869ms
=========
n = 30
832040
fibonacci time:5.091809ms
832040
fibonacciTailrec time:0.023307ms
=========
n = 40
102334155
fibonacci time:300.614119ms
102334155
fibonacciTailrec time:0.027578ms
=========
```
&emsp;可以看到，随着n的增大，尾递归带来的性能优化是非常明显的。

## 5. 总结
&emsp;循环调用都是一个累计器和一个迭代器的作用。同理，尾递归也是如此，它也是通过累加和迭代将结果赋值给新一轮的调用，使用好尾递归能够给我们的程序带来很大性能上的优化。



&nbsp;
&nbsp;
>本文作者：ZJP
版权声明：本文为博主原创文章，遵循 [CC 4.0 BY-SA](http://creativecommons.org/licenses/by-sa/4.0/) 版权协议，转载请附上原文出处链接和本声明。
本文链接：http://zhoujiapeng.top/scala/scala-tailrec/