---
title: Spark源码阅读 之 Dependency 详解
subtitle: Spark Dependency 详解
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - spark
  - 大数据
  - 源码阅读
categories:
  - Spark
date: 2019-09-08 11:28:21
---


# Spark Dependency 实现

> Spark版本：2.4.0

&emsp;在[《Spark DAGScheduler 详解》](http://zhoujiapeng.top/Spark/spark-dagscheduler/)中讲到，DADScheduler 根据 ShuffleDependency 来切分 Stage，那么 ShuffleDependency 是怎么实现的，Dependency 又有那些类型？本文根据源码详细剖析 Spark 中 Dependency 的实现原理。

## 1. Dependency 的类型
&emsp;通过源码，我们可以知道，Dependency 是一个抽象类，内部只有一个方法，返回类型是 RDD[T]，因此 RDD 的 Dependency 也是 RDD。
```scala
abstract class Dependency[T] extends Serializable {
  def rdd: RDD[T]
}
```
&emsp;Dependency 只有两个实现类，ShuffleDependency 和 NarrowDependency。NarrowDependency 有三个子类，OneToOneDependency、RangeDependency 和 PruneDependency。因此，总的来说，Dependency 只有两大类型：**ShuffleDependency** 和 **NarrowDependency**，也就是我们常说的宽依赖和窄依赖。

## 2. 获取 RDD 的 依赖

&emsp;接下来我们来看一下如何获得一个 rdd 的依赖信息。我们可以看到，在DAGScheduler 中 getMissingParentStages 方法 (寻找当前Stage的父Stage) 调用 rdd.dependencies 来获取 RDD 的依赖：
```scala
//RDD
  /**
   * Get the list of dependencies of this RDD, taking into account whether the
   * RDD is checkpointed or not.
   */
  final def dependencies: Seq[Dependency[_]] = {
    checkpointRDD.map(r => List(new OneToOneDependency(r))).getOrElse {
      if (dependencies_ == null) {
        dependencies_ = getDependencies
      }
      dependencies_
    }
  }
  ```
&emsp;这里将 checkpoint 也考虑进去，之所以要考虑 checkpoint，是因为当 rdd 被 checkpoint 之后， 依赖关系会被切断，不能再通过调用 getDependencies 来获取依赖信息。
&emsp;关于 checkpoint 的原理可以参考[《Spark 容错机制》](http://zhoujiapeng.top/Spark/spark-fault-tolerant/)，这里只做简要概述。
&emsp;如果 rdd 被 checkpoint 后， checkpointData 不为空，被赋值为一个 ReliableRDDCheckpointData 对象。执行 checkpointRDD.map(r => List(new OneToOneDependency(r))) 会生成返回一个OneToOneDependency 类型的 List，OneToOneDependency 是 NarrowDependency 的子类，因此调用 rdd.dependencies 会返回一个 NarrowDependency 列表。
&emsp;如果 rdd 没有被checkpoint，则会返回 getOrElse 后面的东西，这里面调用了 getDependencies 来获取依赖信息。由于 RDD 的实现类型很多，我们以 MapPartitionsRDD 和 ShuffledRDD 为例，来说明 getDependencies 。


## 3. MapPartitionsRDD

&emsp;有很多 RDD 算子会返回 MapPartitionsRDD 对象，我们这里以 map 方法为例子，它会返回一个  MapPartitionsRDD 对象。
```scala
def map[U: ClassTag](f: T => U): RDD[U] = withScope {
    val cleanF = sc.clean(f)
    new MapPartitionsRDD[U, T](this, (context, pid, iter) => iter.map(cleanF))
  }
```

&emsp;当前的 RDD（父RDD）作为 MapPartitionsRDD 构造参数中的 prev RDD 传进去。
&emsp;我们再看一下 MapPartitionsRDD 的构造：
```scala
//MapPartitionsRDD
  private[spark] class MapPartitionsRDD[U: ClassTag, T: ClassTag](
    var prev: RDD[T],
    f: (TaskContext, Int, Iterator[T]) => Iterator[U],  // (TaskContext, partition index, iterator)
    preservesPartitioning: Boolean = false,
    isFromBarrier: Boolean = false,
    isOrderSensitive: Boolean = false)
  extends RDD[U](prev)
          ↓↓↓↓↓↓↓
//RDD[U](prev)
def this(@transient oneParent: RDD[_]) =
    this(oneParent.context, List(new OneToOneDependency(oneParent)))
    ↓↓↓↓↓↓↓
abstract class RDD[T: ClassTag](
    @transient private var _sc: SparkContext,
    @transient private var deps: Seq[Dependency[_]]
  ) extends Serializable with Logging 
```
第二个参数是deps，传入的实现类型是 OneToOneDependency，即 NarrowDependency。

我们看一下 getDependencies 方法，返回的正是这个参数，因此 rdd.dependencies 返回的是 NarrowDependency。
```scala
protected def getDependencies: Seq[Dependency[_]] = deps
```

## 4. ShuffledRDD

&emsp;接下来看一下 ShuffledRDD。以 reduceByKey 为例，它会返回一个 MapPartitionsRDD 对象或 ShuffledRDD 对象。

```scala
def reduceByKey(func: (V, V) => V): RDD[(K, V)] = self.withScope {
    //defaultPartitioner构造默认分区器
    reduceByKey(defaultPartitioner(self), func)
  } 
```
&emsp;这里面调用了 defaultPartitioner 方法，它会根据参数(父RDD)构造一个 Partitioner 分区器，这个分区器是 reduce 过程重分区使用的。关于 defaultPartitioner 可以参考[《Spark Partitioner 详解》](http://zhoujiapeng.top/Spark/spark-partitioner/#4defaultpartitioner-1)。reduceByKey 方法最终会调用 combineByKeyWithClassTag 方法：
```scala
def combineByKeyWithClassTag[C](
      createCombiner: V => C,
      mergeValue: (C, V) => C,
      mergeCombiners: (C, C) => C,
      partitioner: Partitioner,
      mapSideCombine: Boolean = true,
      serializer: Serializer = null)(implicit ct: ClassTag[C]): RDD[(K, C)] = self.withScope {
    require(mergeCombiners != null, "mergeCombiners must be defined") // required as of Spark 0.9.0
    if (keyClass.isArray) {
      if (mapSideCombine) {
        throw new SparkException("Cannot use map-side combining with array keys.")
      }
      if (partitioner.isInstanceOf[HashPartitioner]) {
        throw new SparkException("HashPartitioner cannot partition array keys.")
      }
    }
    val aggregator = new Aggregator[K, V, C](
      self.context.clean(createCombiner),
      self.context.clean(mergeValue),
      self.context.clean(mergeCombiners))

    //判断父分区和子分区器是否一样
    if (self.partitioner == Some(partitioner)) {
      //分区器一样，则不需要 shuffle，返回 MapPartitionsRDD
      self.mapPartitions(iter => {
        val context = TaskContext.get()
        new InterruptibleIterator(context, aggregator.combineValuesByKey(iter, context))
      }, preservesPartitioning = true)
    } else {
      //分区器不一样，需要 shuffle，返回 ShuffledRDD
      new ShuffledRDD[K, V, C](self, partitioner)
        .setSerializer(serializer)
        .setAggregator(aggregator)
        .setMapSideCombine(mapSideCombine)
    }
  }
```
&emsp;combineByKeyWithClassTag 方法会根据 self.partitioner == Some(partitioner)  条件来决定要生成 MapPartitionsRDD 还是 ShuffledRDD。如何理解这个条件？我们假设 reduceByKey 是在某个 RDD 上被调用的，设此 RDD 为A，调用 reduceByKey 生成的 RDD 为B，这个条件里面的 partitioner 就是 reduceByKey 方法里面使用 defaultPartitioner(self) 生成的分区器，也就是 B 的 分区器，self.partitioner 是指 A 的分区器。分区器的作用在于决定 RDD 中每个 KV 对应哪个分区。如果 A 和 B 的分区器一样，它们的分区规则一样，因此 A 中的数据经过计算之后的分区号会保持不变，数据不需要 shuffle，因此会生成一个 MapPartitionsRDD。如果分区器不一样，分区规则不一样，意味着需要 shuffle，所以生成一个 ShuffledRDD。

> 注意：在特定情况下 reduceByKey 不会导致shuffle，因为在 combineByKey 中会根据 Partitioner 决定需要生成的RDD。


&emsp;MapPartitionsRDD 的 getDependencies 方法在前面已经介绍过了，我们再来看一下 ShuffledRDD 的 getDependencies 方法。ShuffledRDD 重写了 getDependencies 方法：
```scala
override def getDependencies: Seq[Dependency[_]] = {
    val serializer = userSpecifiedSerializer.getOrElse {
      val serializerManager = SparkEnv.get.serializerManager
      if (mapSideCombine) {
        serializerManager.getSerializer(implicitly[ClassTag[K]], implicitly[ClassTag[C]])
      } else {
        serializerManager.getSerializer(implicitly[ClassTag[K]], implicitly[ClassTag[V]])
      }
    }
    List(new ShuffleDependency(prev, part, serializer, keyOrdering, aggregator, mapSideCombine))
  }

```
这个方法会返回 ShuffleDependency。


## 总结
&emsp;从上面两个例子可以看到，getDependencies 方法返回的依赖类型是 NarrowDependency 或 ShuffleDependency，事实上 Dependency 的实现类也只有这两种。
&emsp;通过调用 rdd.dependencies 可以获取某个 RDD 的依赖。如果 rdd 已经被 checkpoint，那么它的依赖关系会被清除，rdd.dependencies 会默认返回 OneToOneDependency，即 NarrowDependency。 reduceByKey 在某些情况下不会导致 shuffle，因为在 combineByKey 中会根据 Partitioner 决定需要生成的 RDD，如果 reduceByKey 使用的分区规则和调用者的分区规则一致，会返回一个 MapPartitionsRDD，不需要经过 shuffle。


&nbsp;
&nbsp;
>本文作者：ZJP
版权声明：本文为博主原创文章，遵循 [CC 4.0 BY-SA](http://creativecommons.org/licenses/by-sa/4.0/) 版权协议，转载请附上原文出处链接和本声明。
本文链接：http://zhoujiapeng.top/Spark/spark-dependency/

