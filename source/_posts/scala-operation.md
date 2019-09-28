---
title: Scala 强大的集合操作
subtitle: Operations
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - scala
  - 编程语言
categories:
  - scala
date: 2019-09-28 10:21:31
---


# Scala 强大的集合操作

### Map
- map[B](f: (A) ⇒ B): List[B] 
定义一个变换,把该变换应用到列表的每个元素中,原列表不变，返回一个新的列表数据
```scala
val text = List("Homeway,23,Male","Jpne,23,Female")
//map
val userList = text.map(line => {
    val fields = line.split(",")
    (fields(0),fields(1),fields(2))
})
println(userList)

=> List((Homeway,23,Male), (Jpne,23,Female))
```

### flatten
- flatten[B]: List[B]
对列表的列表进行平坦化操作 
```scala
val textFlatten = List("Homeway,23,Male","Jpne,23,Female").map(_.split(",").toList).flatten
println(textFlatten)

=> List(Homeway, 23, Male, Jpne, 23, Female)
```

### flatMap 
- flatMap : flatMap[B](f: (A) ⇒ GenTraversableOnce[B]): List[B] 
map之后对结果进行flatten，定义一个变换f, 把f应用列表的每个元素中，每个f返回一个列表，最终把所有列表连结起来。
```scala
val userFlatMap = List("Homeway,23,Male","Jpne,23,Female").flatMap(_.split(","))
println(userFlatMap)

=> List(Homeway, 23, Male, Jpne, 23, Female)
```


### reduce，reduceLeft，reduceRight
- reduce[A1 >: A](op: (A1, A1) ⇒ A1): A1
定义一个变换f, f把两个列表的元素合成一个，遍历列表，最终把列表合并成单一元素
```scala
val sums = List(1,2,3,4).reduce((a,b) => a+b)
println(sums)

=> 10
```
- reduceLeft: reduceLeft[B >: A](f: (B, A) ⇒ B): B
从列表的左边往右边应用reduce函数
```scala
//reduceLeft
val sums1 = List(1,2,3,4).reduceLeft((a,b) => {
    println((a,b))
    a+b
})
println(sums1)

(1,2)
(3,3)
(6,4)
10
```
- reduceRight: reduceRight[B >: A](op: (A, B) ⇒ B): B
从列表的右边往左边应用reduce函数
```scala
//reduceRight
val sums2 = List(1,2,3,4).reduceRight((a,b) => {
    println((a,b))
    a+b
})
println(sums2)

(3,4)
(2,7)
(1,9)
10
```


### fold，foldleft，foldRight
- fold: fold[A1 >: A](z: A1)(op: (A1, A1) ⇒ A1) A1
带有初始值的reduce,从一个初始值开始，从左向右将两个元素合并成一个，最终把列表合并成单一元素。
```scala
println(Array(1,2,3,4,5).fold(-1)((a,b) => {
      println(a,b)
      a+b
    }))

(-1,1)
(0,2)
(2,3)
(5,4)
(9,5)
14
```
- foldLeft: foldLeft[B](z: B)(f: (B, A) ⇒ B)
B 带有初始值的reduceLeft
```scala
println(Array(1,2,3,4,5).foldLeft(-1)((a,b) => {
      println(a,b)
      a+b
    }))

(-1,1)
(0,2)
(2,3)
(5,4)
(9,5)
14
```
- foldRight: foldRight[B](z: B)(op: (A, B) ⇒ B)
B 带有初始值的reduceRight
```scala
println(Array(1,2,3,4,5).foldRight(-1)((a,b) => {
      println(a,b)
      a+b
    }))

(5,-1)
(4,4)
(3,8)
(2,11)
(1,13)
14
```


### sortBy,sortWith,sorted
- sortBy: sortBy[B](f: (A) ⇒ B)(implicit ord: math.Ordering[B]): List[A]
按照应用函数f之后产生的元素进行排序
```scala
val users = List(("aa",22),("bb",66),("cc",44),("dd",34),("ee",11))
println(users.sortBy({case(name,age) => age}))

List((ee,11), (aa,22), (dd,34), (cc,44), (bb,66))
```
- sorted： sorted[B >: A](implicit ord: math.Ordering[B]): List[A]
按照元素自身进行排序
```scala
println(List(2,4,1,6,-1).sorted)

List(-1, 1, 2, 4, 6)
```
- sortWith： sortWith(lt: (A, A) ⇒ Boolean): List[A]
使用自定义的比较函数进行排序
```scala
println(users.sortWith({case(user1,user2) => user1._2 > user2._2}))

List((bb,66), (cc,44), (dd,34), (aa,22), (ee,11))
```


### filter,filterNot
- filter: filter(p: (A) ⇒ Boolean): List[A]
保留列表中符合条件p的列表元素
```scala
println(List(1,2,3,4,5,6).filter(_%2==0))

List(2, 4, 6)
```
- filterNot: filterNot(p: (A) ⇒ Boolean): List[A]
保留列表中不符合条件p的列表元素
```scala
println(List(1,2,3,4,5,6).filterNot(_%2==0))

List(1, 3, 5)
```


### count
- count(p: (A) ⇒ Boolean): Int
计算列表中所有满足条件p的元素的个数，等价于 filter(p).length
```scala
println(List(1,2,3,4,5,6).count(_>3))

3
```


### diff, union, intersect
- diff:diff(that: collection.Seq[A]): List[A]
保存列表中那些不在另外一个列表中的元素，即从集合中减去与另外一个集合的交集
```scala
println(List(1,2,3).diff(List(2,3,4)))

List(1)
```
- union : union(that: collection.Seq[A]): List[A]
与另外一个列表进行连结
```scala
println(List(1,2,3).union(List(2,3,4)))
println(List(1,2,3).++(List(2,3,4)))

List(1, 2, 3, 2, 3, 4)
List(1, 2, 3, 2, 3, 4)
```
- intersect: intersect(that: collection.Seq[A]): List[A]
与另外一个集合的交集
```scala
println(List(1,2,3).intersect(List(2,3,4)))

List(2, 3)
```


### distinct
- distinct: List[A]
保留列表中非重复的元素，相同的元素只会被保留一次
```scala
println(List(1,2,3,3,3,4,4,5,6,6).distinct)

List(1, 2, 3, 4, 5, 6)
```


### groupBy,grouped
- groupBy : groupBy[K](f: (A) ⇒ K): Map[K, List[A]]
将列表进行分组，分组的依据是应用f在元素上后产生的新元素
```scala
println(List(("John","Man"),("Li","Feman"),("Fed","Man"),("Ka","Feman")).groupBy(_._2))

Map(Man -> List((John,Man), (Fed,Man)), Feman -> List((Li,Feman), (Ka,Feman)))
```
- grouped: grouped(size: Int): Iterator[List[A]]
按列表按照固定的大小进行分组
```scala
List(("John","Man"),("Li","Feman"),("Fed","Man"),("Ka","Feman")).grouped(1).foreach(println)

List((John,Man), (Li,Feman))
List((Fed,Man), (Ka,Feman))
```



### scan，scanLeft,scanRight
- scan[B >: A, That](z: B)(op: (B, B) ⇒ B)(implicit cbf: CanBuildFrom[List[A], B, That]): That
由一个初始值开始，从左向右，进行积累的op操作，和fold函数类似，区别在于，fold的结果只有最终的累加值，而scan的结果包含每一次累加的值
```scala
println(List(1,2,3,4).scan(10)((x,y)=>{
      println((x,y))
      x+y
    }))

(10,1)
(11,2)
(13,3)
(16,4)
List(10, 11, 13, 16, 20)
```
- scanLeft: scanLeft[B, That](z: B)(op: (B, A) ⇒ B)(implicit bf: CanBuildFrom[List[A], B, That]): That
从左向右进行scan函数的操作
```scala
println(List(1,2,3,4).scanLeft(10)((x,y)=>{
      println((x,y))
      x+y
    }))

(10,1)
(11,2)
(13,3)
(16,4)
List(10, 11, 13, 16, 20)
```
- scanRight: scanRight[B, That](z: B)(op: (A, B) ⇒ B)(implicit bf: CanBuildFrom[List[A], B, That]): That
从右向左进行scan函数的操作
```scala
println(List(1,2,3,4).scanRight(10)((x,y)=>{
      println((x,y))
      x+y
    }))

(4,10)
(3,14)
(2,17)
(1,19)
List(20, 19, 17, 14, 10)
```



### take,takeRight,takeWhile
- take : take(n: Int): List[A]
提取列表的前n个元素 
```scala
println(List(1,2,3,4,5).take(3))

List(1, 2, 3)
```
- takeRight: takeRight(n: Int): List[A]
提取列表的最后n个元素
```scala
println(List(1,2,3,4,5).takeRight(3))

List(3, 4, 5)
```
- takeWhile: takeWhile(p: (A) ⇒ Boolean): List[A]
从左向右提取列表的元素，直到条件p不成立(遍历过程只要遇到一个不成立就停止)
```scala
println(List(1,2,3,4,5).takeWhile(_ < 3))

List(1, 2)
```


### drop,dropRight,dropWhile
- drop: drop(n: Int): List[A]
丢弃前n个元素，返回剩下的元素
```scala
println(List(1,2,3,4,5).drop(3))

List(4, 5)
```
- dropRight: dropRight(n: Int): List[A]
丢弃最后n个元素，返回剩下的元素
```scala
println(List(1,2,3,4,5).dropRight(3))

List(1, 2)
```
- dropWhile: dropWhile(p: (A) ⇒ Boolean): List[A]
从左向右丢弃元素，直到条件p不成立
```scala
println(List(1,2,3,4,5,1).dropWhile(_<3))

List(3, 4, 5, 1)
```


### span,splitAt,partition
- span : span(p: (A) ⇒ Boolean): (List[A], List[A])
从左向右应用条件p进行判断，直到条件p不成立(只要遇到一个不成立就停止)，此时将列表分为两个列表
```scala
println(List(1,1,2,3,4,5,1).span(_==1))

(List(1, 1),List(2, 3, 4, 5, 1))
```
- splitAt: splitAt(n: Int): (List[A], List[A])
将列表分为前n个，与，剩下的部分
```scala
println(List(1,2,3,4,5,6).splitAt(3))

(List(1, 2, 3),List(4, 5, 6))
```
- partition: partition(p: (A) ⇒ Boolean): (List[A], List[A])
将列表分为两部分，第一部分为满足条件p的元素，第二部分为不满足条件p的元素，注意和span的区别，partition会对每个元素应用判断条件
```scala
println(List(1,2,3,1,4,2).partition(_==2))

(List(2, 2),List(1, 3, 1, 4))
```


### padTo
- padTo(len: Int, elem: A): List[A]
将列表扩展到指定长度，长度不够的时候，使用elem进行填充，否则不做任何操作。
```scala
println(List(1,2,3,4,5).padTo(7,10))

List(1, 2, 3, 4, 5, 10, 10)
```



### combinations,permutations
- combinations: combinations(n: Int): Iterator[List[A]]
取列表中的n个元素进行组合，返回**不重复**的组合列表，结果一个迭代器
```scala
List(1,1,3,4).combinations(2).foreach(println)
List(1,1,3,4).combinations(3).foreach(println)

List(1, 1)
List(1, 3)
List(1, 4)
List(3, 4)

List(1, 1, 3)
List(1, 1, 4)
List(1, 3, 4)
```
- permutations: permutations: Iterator[List[A]]
对列表中的元素进行排列，返回**不重复**的排列列表，结果是一个迭代器
```scala
List(1,1,3,4).permutations.foreach(println)

List(1, 1, 3, 4)
List(1, 1, 4, 3)
List(1, 3, 1, 4)
List(1, 3, 4, 1)
List(1, 4, 1, 3)
List(1, 4, 3, 1)
List(3, 1, 1, 4)
List(3, 1, 4, 1)
List(3, 4, 1, 1)
List(4, 1, 1, 3)
List(4, 1, 3, 1)
List(4, 3, 1, 1)
```


### zip, zipAll, zipWithIndex, unzip,unzip3
- zip: zip[B](that: GenIterable[B]): List[(A, B)] 
与另外一个列表进行拉链操作，将对应位置的元素组成一个pair，返回的列表长度为两个列表中短的那个
```scala
println(List(1,2,3,4).zip(List("a","b","c","d","e")))

List((1,a), (2,b), (3,c), (4,d))
```
- zipAll: zipAll[B](that: collection.Iterable[B], thisElem: A, thatElem: B): List[(A, B)]
与另外一个列表进行拉链操作，将对应位置的元素组成一个pair，若列表长度不一致，自身列表比较短的话使用thisElem进行填充，对方列表较短的话使用thatElem进行填充
```scala
println(List(1,2,3,4).zipAll(List("a","b","c","d","e"),0,"y"))
println(List(1,2,3,4).zipAll(List("a","b","c"),0,"y"))

List((1,a), (2,b), (3,c), (4,d), (0,e))
List((1,a), (2,b), (3,c), (4,y))
```
- zipWithIndex：zipWithIndex: List[(A, Int)]
将列表元素与其索引进行拉链操作，组成一个pair
```scala
println(List("a","b","c").zipWithIndex)

List((a,0), (b,1), (c,2))
```
- unzip: unzip[A1, A2](implicit asPair: (A) ⇒ (A1, A2)): (List[A1], List[A2])
解开拉链操作
```scala
println(List((1,"a"),(2,"b"),(3,"c")).unzip)

(List(1, 2, 3),List(a, b, c))
```
- unzip3: unzip3[A1, A2, A3](implicit asTriple: (A) ⇒ (A1, A2, A3)): (List[A1], List[A2], List[A3])
3个元素的解拉链操作
```scala
println(List((1,"a","x"),(2,"b","y"),(3,"c","z")).unzip3)

(List(1, 2, 3),List(a, b, c),List(x, y, z))
```


### slice
- slice(from: Int, until: Int): List[A]
提取列表中从位置from到位置until(不含该位置)的元素列表
```scala
println(List(1,2,3,4,5).slice(1,3))

List(2, 3)
```


### sliding
- sliding(size: Int, step: Int): Iterator[List[A]]
将列表按照固定大小size进行分组，步进为step，step默认为1,返回结果为迭代器
```scala
println(List(1,2,3,4,5,6,7,8,9).sliding(2,3).toList)

List(List(1, 2), List(4, 5), List(7, 8))
```


### updated
- updated(index: Int, elem: A): List[A]
对列表中的某个元素进行更新操作
```scala
println(List(1,2,3,4,5,6).updated(2,10))

List(1, 2, 10, 4, 5, 6)
```


&nbsp;
&nbsp;
> 参考：
《Scala 编程实战》