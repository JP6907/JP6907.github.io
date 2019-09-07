---
title: Spark源码阅读 之 Dependency 实现
catalog: true
subtitle: Spark Dependency 实现
header-img: "/img/article_header/article_header.png"
tags:
- spark
- 大数据
- 源码阅读
categories:
  - Spark
---

# Spark Dependency 实现

> Spark版本：2.4.0

&emsp;DADScheduler 根据 ShuffleDependency 来切分 Stage，那么 ShuffleDependency 是怎么实现的，Dependency 又有那些类型？本文根据源码详细剖析 Spark 中 Dependency 的实现原理。


  引发shuffle的transformation会生成特殊的RDD，此RDD会是shuffle中子Stage的起点，当这些RDD的compute方法被调用时，就会触发reduce端操作的执行。这种特殊的RDD有两类：

ShuffledRDD, 它只有一个父RDD，是对一个RDD进行shuffle的结果。

CoGroupedRDD, 它有多个RDD，是对多个RDD进行shuffle的结果。

ShuffleDependency 是怎么生成的
以 reduceByKey 为例 生成的是 ShuffledRDD    


## 1. 获取 RDD 的 依赖

&emsp;我们可以看到，在DAGScheduler 中 getMissingParentStages 方法调用 rdd.dependencies 来获取 RDD 的依赖：
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
&emsp;当 rdd 被 checkpoint 后， checkpointData 不为空，被赋值为一个ReliableRDDCheckpointData 对象，执行 checkpointRDD.map(r => List(new OneToOneDependency(r))) 会生成返回一个OneToOneDependency 列表，OneToOneDependency是 NarrowDependency 的子类，因此调用 rdd.dependencies 会返回一个 NarrowDependency 列表。
&emsp;如果 rdd 没有被checkpoint，则会返回 getOrElse 后面的东西，这里面调用了 getDependencies 来获取依赖信息。由于 RDD 的类型很多，我们以 MapPartitionsRDD 和 ShuffledRDD 为例，来说明 Dependency 的实现。

以 MapPartitionsRDD 和 ShuffledRDD 为例


MapPartitionsRDD

首先以 map 方法   map 方法会返回一个 MapPartitionsRDD 对象
```scala
def map[U: ClassTag](f: T => U): RDD[U] = withScope {
    val cleanF = sc.clean(f)
    new MapPartitionsRDD[U, T](this, (context, pid, iter) => iter.map(cleanF))
  }
```

&emsp;当前的 RDD 作为 MapPartitionsRDD 构造参数中的 prev RDD
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

//RDD[U](prev)
def this(@transient oneParent: RDD[_]) =
    this(oneParent.context, List(new OneToOneDependency(oneParent)))

abstract class RDD[T: ClassTag](
    @transient private var _sc: SparkContext,
    @transient private var deps: Seq[Dependency[_]]
  ) extends Serializable with Logging 
```
第二个参数是deps，传入的实现类型是 OneToOneDependency，即 NarrowDependency

我们看一下 getDependencies 方法，返回的正是这个参数，因此 rdd.dependencies 获得的是 NarrowDependency 类型
```scala
protected def getDependencies: Seq[Dependency[_]] = deps
```


接下来看一下 ShuffledRDD



ShuffledRDD

reduceByKey
```scala
def reduceByKey(func: (V, V) => V): RDD[(K, V)] = self.withScope {
    reduceByKey(defaultPartitioner(self), func)
  } 
```

defaultPartitioner 方法会构造一个 Partitioner 分区器，这个分区器是用来 reduce 过程分区用的。reduceByKey 方法最终会调用 combineByKeyWithClassTag 方法：

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
    if (self.partitioner == Some(partitioner)) {
      self.mapPartitions(iter => {
        val context = TaskContext.get()
        new InterruptibleIterator(context, aggregator.combineValuesByKey(iter, context))
      }, preservesPartitioning = true)
    } else {
      new ShuffledRDD[K, V, C](self, partitioner)
        .setSerializer(serializer)
        .setAggregator(aggregator)
        .setMapSideCombine(mapSideCombine)
    }
  }
```
combineByKeyWithClassTag 方法会根据 self.partitioner == Some(partitioner)  条件来决定要生成 MapPartitionsRDD 还是 ShuffledRDD。如何理解这个条件？我们假设 reduceByKey 是在某个 RDD 上被调用的，设此 RDD 为A，调用 reduceByKey 生成的 RDD 为B，这个条件里面的 partitioner 就是 reduceByKey 方法里面使用 defaultPartitioner(self) 生成的分区器，也就是 B 的 分区器，self.partitioner 是指 A 的分区器。分区器的作用在于决定 RDD 中每个 KV 对应哪个分区。如果 A 和 B 的分区器一样，它们的分区规则一样，因此 A 中的数据经过计算之后的分区号会保持不变，数据不需要 shuffle，因此会生成一个 MapPartitionsRDD。如果分区器不一样，分区规则不一样，意味着需要 shuffle，所以生成一个 ShuffledRDD。

> 注意：在特定情况下 reduceByKey 不会导致shuffle，因为在 combineByKey 中会根据 Partitioner 决定需要生成的RDD。

self.partitioner == Some(partitioner) 介绍

&emsp;我们再来看一下 ShuffledRDD 的 getDependencies 方法。ShuffledRDD 重写了 getDependencies 方法：
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


从上面两个例子可以看到，getDependencies 方法返回的依赖类型是 NarrowDependency 或 ShuffleDependency，事实上 Dependency 的实现类也只有这两种。

## 总结
&emsp;RDD 的 Dependency 类型只有两种，NarrowDependency 和 ShuffleDependency。通过调用 rdd.dependencies 可以获取某个 RDD 的依赖。如果 rdd 已经被 checkpoint，那么它的依赖关系会被清除，rdd.dependencies 会返回 OneToOneDependency，即 NarrowDependency。 reduceByKey 在某些情况下不会导致 shuffle，因为在 combineByKey 中会根据 Partitioner 决定需要生成的 RDD，如果 reduceByKey 使用的分区规则和调用者的分区规则一致，会返回一个 MapPartitionsRDD，不需要经过 shuffle。


&nbsp;
&nbsp;
>本文作者：ZJP
版权声明：本文为博主原创文章，遵循 [CC 4.0 BY-SA](http://creativecommons.org/licenses/by-sa/4.0/) 版权协议，转载请附上原文出处链接和本声明。
本文链接：http://zhoujiapeng.top/Spark/spark-dependency/

