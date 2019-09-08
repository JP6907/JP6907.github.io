---
title: Spark源码阅读 之 Partitioner 详解
subtitle: Spark Partitioner 详解
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - spark
  - 大数据
  - 源码阅读
categories:
  - Spark
date: 2019-09-08 10:49:21
---



# Spark Partitioner 详解

> Spark版本：2.4.0

&emsp;本文主要根据源码剖析 Spark Partitioner 分区器的实现原理，以及介绍如何自定义分区器。

## 1.概述
&emsp;Partitioner 是 shuffle 过程中 key 重分区时的策略，即计算 key 决定 k-v 属于哪个分区，Transformation 是宽依赖的算子时，父 RDD 和子 RDD 之间会进行 shuffle 操作，shuffle 涉及到网络开销，由于父 RDD 和子 RDD 中的 partition 是多对多关系，所以容易造成partition中数据分配不均匀，导致数据的倾斜。


## 2.Shuffle
&emsp;在 MapReduce 框架中，Shuffle 是连接 Map 和和 Reduce 之间的桥梁，Map 的输出要用到 Reduce 中必须经过 Shuffle这个环节，Spark 作为 MapReduce 框架的一种实现，自然也实现了 shuffle 的逻辑。
&emsp;Shuffle 描述的是一个过程，表现多对多的依赖关系，是 Map 和 Reduce 两个阶段的纽带，是对数据重新分区的过程，将经过 mapTask 后，key 值相同的数据重新划分到同一个 partition 中。
&emsp;在 Spark2.4.0 版本中 Shuffle 的实现只保留了 SortShuffleManager，也可以自定义。在本文中，只讨论 Shuffle 过程中 Partitioner 的作用，Partitioner 在 shuffle 的 map 端发挥作用，根据 map 端的 key 值，按照不同 Partitioner 的逻辑计算出 reduce 端的 partitionId，以实现对相同 key 值数据的重新聚合。

## 3.Partitioner
&emsp;Partitioner 是一个抽象类，有两个方法，numPartitions 和 getPartition。numPartitions 返回分区数量，getPartition 根据 key 返回分区 id。
```scala
abstract class Partitioner extends Serializable {
  def numPartitions: Int
  def getPartition(key: Any): Int
}
```

## 4.defaultPartitioner
&emsp;defaultPartitioner 是 Partitioner 伴生对象里面定义的一个方法，该函数定义了 Partitioner 的默认分区策略。如果设置了 spark.default.parallelism，则使用该值作为默认 partitions 分区数量，否则使用上游 RDD (参数中的RDD)中 partitions 最大的数作为默认 partitions。过滤出上游 RDD 中包含 partitioner 的 RDD，选择包含有最大 partitions 并且符合 isEligiblePartitioner 条件 的 RDD，将该 RDD 中的partitioner 设置为分区策略，否则返回一个带有默认分区数量的 HashPartitioner 作为 Partitioner。
```scala
def defaultPartitioner(rdd: RDD[_], others: RDD[_]*): Partitioner = {
    val rdds = (Seq(rdd) ++ others)
    val hasPartitioner = rdds.filter(_.partitioner.exists(_.numPartitions > 0))

    val hasMaxPartitioner: Option[RDD[_]] = if (hasPartitioner.nonEmpty) {
      Some(hasPartitioner.maxBy(_.partitions.length))
    } else {
      None
    }

    val defaultNumPartitions = if (rdd.context.conf.contains("spark.default.parallelism")) {
      rdd.context.defaultParallelism
    } else {
      rdds.map(_.partitions.length).max
    }

    // If the existing max partitioner is an eligible one, or its partitions number is larger
    // than the default number of partitions, use the existing partitioner.
    if (hasMaxPartitioner.nonEmpty && (isEligiblePartitioner(hasMaxPartitioner.get, rdds) ||
        defaultNumPartitions < hasMaxPartitioner.get.getNumPartitions)) {
      hasMaxPartitioner.get.partitioner.get
    } else {
      new HashPartitioner(defaultNumPartitions)
    }
  }
```

## 5.Partitioner实现类
&emsp;Partitioner 主要有两个实现类：HashPartitioner 和 RangePartitioner，上面提到过 HashPartitioner 是默认的重分区方式.

### 5.1 HashPartitioner
&emsp;HashPartitioner 将 key 的 hashCode 与分区数量求模得到分区号。由于 Java Array 的 hashcode 依赖于索引而不是 Array 的内容，因此如果尝试使用 HashPartitioner 对 RDD[Array[_]] 或 RDD[(Array[_], _)] 进行分区，它将不会对 key 值进行重分区，不会得到我们想要的结果。
```scala
class HashPartitioner(partitions: Int) extends Partitioner {
  require(partitions >= 0, s"Number of partitions ($partitions) cannot be negative.")

  def numPartitions: Int = partitions

  def getPartition(key: Any): Int = key match {
    case null => 0
    case _ => Utils.nonNegativeMod(key.hashCode, numPartitions)
  }

  override def equals(other: Any): Boolean = other match {
    case h: HashPartitioner =>
      h.numPartitions == numPartitions
    case _ =>
      false
  }

  override def hashCode: Int = numPartitions
}
```

### 5.2 RangePartitioner
&emsp;RangePartitioner 主要用于 RDD 的数据排序相关API中，比如 sortByKey 底层使用的数据分区器就是 RangePartitioner 分区器。

![rangePartitioner](https://github.com/JP6907/Pic/blob/master/spark/rangePartitioner.png?raw=true)

&emsp;RangePartitioner 分区器的实现方式主要是通过两个步骤来实现的，第一步：先从整个 RDD 中抽取出样本数据，将样本数据排序，计算出每个分区的最大key值，形成一个 Array[KEY] 类型的数组变量 rangeBounds；第二步：判断 key 在 rangeBounds 中所处的范围，给出该 key 值在下一个 RDD 中的分区id号；该分区器要求 RDD 中的 KEY 类型必须是可以排序的。
&emsp;RangePartitioner 的 rangeBounds 按照 key 排序 划分出长度大致相等的几个范围，最终返回 （partitions - 1）个有序的边界值。
```scala
private var rangeBounds: Array[K] = {
    if (partitions <= 1) {
      Array.empty
    } else {
      // This is the sample size we need to have roughly balanced output partitions, capped at 1M.
      // Cast to double to avoid overflowing ints or longs
      val sampleSize = math.min(samplePointsPerPartitionHint.toDouble * partitions, 1e6)
      // Assume the input partitions are roughly balanced and over-sample a little bit.
      val sampleSizePerPartition = math.ceil(3.0 * sampleSize / rdd.partitions.length).toInt
      val (numItems, sketched) = RangePartitioner.sketch(rdd.map(_._1), sampleSizePerPartition)
      if (numItems == 0L) {
        Array.empty
      } else {
        // If a partition contains much more than the average number of items, we re-sample from it
        // to ensure that enough items are collected from that partition.
        val fraction = math.min(sampleSize / math.max(numItems, 1L), 1.0)
        val candidates = ArrayBuffer.empty[(K, Float)]
        val imbalancedPartitions = mutable.Set.empty[Int]
        sketched.foreach { case (idx, n, sample) =>
          if (fraction * n > sampleSizePerPartition) {
            imbalancedPartitions += idx
          } else {
            // The weight is 1 over the sampling probability.
            val weight = (n.toDouble / sample.length).toFloat
            for (key <- sample) {
              candidates += ((key, weight))
            }
          }
        }
        if (imbalancedPartitions.nonEmpty) {
          // Re-sample imbalanced partitions with the desired sampling probability.
          val imbalanced = new PartitionPruningRDD(rdd.map(_._1), imbalancedPartitions.contains)
          val seed = byteswap32(-rdd.id - 1)
          val reSampled = imbalanced.sample(withReplacement = false, fraction, seed).collect()
          val weight = (1.0 / fraction).toFloat
          candidates ++= reSampled.map(x => (x, weight))
        }
        RangePartitioner.determineBounds(candidates, math.min(partitions, candidates.size))
      }
    }
  }
```
&emsp;分区数量为边界值数量+1
```scala
def numPartitions: Int = rangeBounds.length + 1
```
&emsp;我们再来看一下 getPartition 方法：
```scala
def getPartition(key: Any): Int = {
    val k = key.asInstanceOf[K]
    var partition = 0
    if (rangeBounds.length <= 128) {
      // If we have less than 128 partitions naive search
      
      ///如果分区数量少于128，直接遍历
      while (partition < rangeBounds.length && ordering.gt(k, rangeBounds(partition))) {
        partition += 1
      }
    } else {

      //否则使用二分查找
      // Determine which binary search method to use only once.
      partition = binarySearch(rangeBounds, k)
      // binarySearch either returns the match location or -[insertion point]-1
      if (partition < 0) {
        partition = -partition-1
      }
      if (partition > rangeBounds.length) {
        partition = rangeBounds.length
      }
    }
    if (ascending) {
      partition
    } else {
      rangeBounds.length - partition
    }
  }
```
&emsp;getPartition 先判断间隔符的个数，如果小于128则直接遍历比较key和分隔符得到 PartitionId，否则使用二分查找，并对边界条件进行了判断，最后根据构造 RangePartitioner 时传入的ascending参数确定是升序或降序返回 PartitionId。


## 6.自定义 Partitioner
&emsp;在 Spark，默认使用 HashPartitioner 对 rdd 进行分区。在有些情况下，可能会产生数据倾斜等情况，这个时候就需要我们自定义分区器。自定义分区器最主要的是重写 numPartitions 、 getPartition 和 equals 方法。
&emsp;比如我们想对以以下数据集根据第一个字段分区：
```
cookieid,createtime,pv
cookie1,2015-04-10,1
cookie1,2015-04-11,5
cookie2,2015-04-10,2
cookie2,2015-04-11,3
.....
```
定义分区器：
```scala
class UDFPartitioner(args: Array[String]) extends Partitioner {
 
  private val partitionMap: HashMap[String, Int] = new HashMap[String, Int]()
  var parId = 0
  for (arg <- args) {
    if (!partitionMap.contains(arg)) {
      partitionMap(arg) = parId
      parId += 1
    }
  }
 
  override def numPartitions: Int = partitionMap.valuesIterator.length
 
  override def getPartition(key: Any): Int = {
    val keys: String = key.asInstanceOf[String]
    val sub = keys
    partitionMap(sub)
  }
}
```
测试：
```scala
object UDFPartitionerMain {
  def main(args: Array[String]): Unit = {
    val conf = new SparkConf().setMaster("local[2]").setAppName(this.getClass.getSimpleName)
    val ssc = SparkSession
      .builder()
      .config(conf)
      .getOrCreate()
    val sc = ssc.sparkContext
    sc.setLogLevel("WARN")
 
    val rdd = ssc.sparkContext.textFile("file:///E:\\TestFile\\analyfuncdata.txt")
    val transform = rdd.filter(_.split(",").length == 3).map(x => {
      val arr = x.split(",")
      (arr(0), (arr(1), arr(2)))
    })
    val keys: Array[String] = transform.map(_._1).collect()
    val partiion = transform.partitionBy(new UDFPartitioner(keys))
    partiion.foreachPartition(iter => {
      println(s"**********分区号：${TaskContext.getPartitionId()}***************")
      iter.foreach(r => {
        println(s"分区:${TaskContext.getPartitionId()}###" + r._1 + "\t" + r._2 + "::" + r._2._1)
      })
    })
    ssc.stop()
  }
}
```





&nbsp;
&nbsp;
>本文作者：ZJP
版权声明：本文为博主原创文章，遵循 [CC 4.0 BY-SA](http://creativecommons.org/licenses/by-sa/4.0/) 版权协议，转载请附上原文出处链接和本声明。
本文链接：http://zhoujiapeng.top/Spark/spark-partitioner/