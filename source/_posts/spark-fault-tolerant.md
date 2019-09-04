---
title: Spark源码阅读 之 容错机制
catalog: true
date: 2019-09-04 08:54:32
subtitle: Spark 容错机制
header-img: "/img/article_header/article_header.png"
tags:
- spark
- 大数据
- 源码阅读
---

# Spark 容错机制

> Spark版本：2.4.0

&emsp;分布式数据集实现容错的方式有两种，一种是记录更新，另一种是检查点(checkpoint)。Spark 采用两种方式类实现数据集的容错：第一种是Lineage(血统)机制，即记录更新，基于 RDD，Spark 实现了一系列的 transformation 算子。这些算子可以同时施加于同一数据，这样就形成了以了 Lineage 链。当某个 RDD 的分区数据丢失时，就可以依据这个链进行回溯，重新计算并还原丢失的数据；第二种是 Checkpoint(检查点)机制，当 RDD 的计算链过长时，一旦某个分区的数据丢失后就需要进行大量的重复计算来恢复，这样不仅消耗了时间资源，也可能会造成某些数据的冗余计算，为了避免这种情况的发生，可以使用 Checkpoint 机制，将数据储存到磁盘来实现容错。


## 1.Lineage机制
&emsp;RDD 提供基于粗粒度转换(coarse-grained-transformation)的接口，如map、join等可以将同一操作施加到许多数据项上。RDD 通过所谓的血统关系（Lineage）记录它是如何从其它 RDD 演变过来的。当某个 RDD 部分分区数据丢失时，可以通过重新计算来还原丢失的数据，避免数据复制的高开销。
&emsp;在 RDD 中将依赖划分成两种类型：窄依赖（Narrow Dependencies）和宽依赖（Wide/Shuffle Dependencies）。窄依赖是指父 RDD 的每一个分区最多被一个子 RDD 的分区所用，表现为一个父 RDD 的分区对应于一个子 RDD 的分区或多个父 RDD 的分区对应于一个子 RDD 的分区，也就是说一个父 RDD 的一个分区不可能对应一个子 RDD 的多个分区；宽依赖是指子 RDD 的分区依赖于父 RDD 的多个分区或所有分区，也就是说存在一个父 RDD 的一个分区对应一个子 RDD 的多个分区。例如，map就是一种窄依赖，而join则会导致宽依赖，除非父 RDD 是 hash-partitioned。窄依赖对于数据重新计算的开销远小于宽依赖的数据重新计算开销。
&emsp;这种划分在容错机制中有两个用处。首先，窄依赖支持在一个节点上管道化执行，即 pipeline 操作；其次，窄依赖支持更高效的故障还原，对于窄依赖，只有丢失的父 RDD 的分区需要重新计算，不依赖于其它分区，并不存在冗余计算，而宽依赖可能会导致重新计算并不会丢失的子 RDD 的分区数据。因此对于宽依赖，Spark 通过一定的数据持久化来简化故障还原。


## 2.Checkpoint机制
&emsp;Lineage 机制可以用来故障恢复，但对于 Lineage 链较长的 RDD 来说，这种恢复可能很耗时，尤其是在做一些迭代操作的时候。Checkpoint 机制通过冗余数据来缓存数据，避免重复计算。Checkpoint 机制没有在 Job 第一次计算得到结果就储存，而是等到 Job 结束后启动**专门的 Job** 去完成 Checkpoint。也就是说，需要 Checkpoint 的 RDD 会被计算两次。因此，在使用 rdd.Checkpoint() 的时候，建议加上 rdd.cache()，这样第二次运行的 job 可能直接从缓存读取数据而不需要重复计算。另外，RDD 提供了 rdd.persist(StorageLevel.DISK_ONLY) 这样的方法，相当于缓存到磁盘，这样就可以做到 RDD 在第一次计算的时候就将结果储存到磁盘。
&emsp;在checkpoint之前，我们需要先设置checkpoint数据保存的目录:
```scala
sc.setCheckpointDir("path")
```
&emsp;当一个RDD需要被 checkpoint 的时候，一般会经过四个阶段：initialized、Mark for checkpointing、checkpoinging in progress、checkpointed。

### 2.1 标记 checkpoint
&emsp;当我们调用 rdd.checkpoint() 的时候，实际上并不会真正触发 checkpoint 操作，而只是将当前 RDD 标识为需要 checkpoint。
```scala
def checkpoint(): Unit = RDDCheckpointData.synchronized {
    // NOTE: we use a global lock here due to complexities downstream with ensuring
    // children RDD partitions point to the correct parent partitions. In the future
    // we should revisit this consideration.
    if (context.checkpointDir.isEmpty) {
      throw new SparkException("Checkpoint directory has not been set in the SparkContext")
    } else if (checkpointData.isEmpty) {
        //类型是 ReliableRDDCheckpointData
      checkpointData = Some(new ReliableRDDCheckpointData(this))
    }
  }
```
&emsp;这里会根据当前 RDD 给 checkpointData 赋值，注意，这里给 checkpointData 赋的值是 **ReliableRDDCheckpointData** 对象。当 checkpointData 不为空的时候，表明这个 RDD 需要被 checkpoint，即被标记。

### 2.2 触发 checkpoint 操作
&emsp;checkpoint 真正执行操作会被推迟到 action 算子触发 runJob 的时候。
```scala
//Sparkcontext.runJob
def runJob[T, U: ClassTag](
      rdd: RDD[T],
      func: (TaskContext, Iterator[T]) => U,
      partitions: Seq[Int],
      resultHandler: (Int, U) => Unit): Unit = {
    if (stopped.get()) {
      throw new IllegalStateException("SparkContext has been shutdown")
    }
    val callSite = getCallSite
    val cleanedFunc = clean(func)
    logInfo("Starting job: " + callSite.shortForm)
    if (conf.getBoolean("spark.logLineage", false)) {
      logInfo("RDD's recursive dependencies:\n" + rdd.toDebugString)
    }
    dagScheduler.runJob(rdd, cleanedFunc, partitions, callSite, resultHandler, localProperties.get)
    progressBar.foreach(_.finishAll())
    rdd.doCheckpoint()
  }
```
最后会执行 rdd.doCheckpoint()，这里真正触发 checkpoint 操作。
```scala
private[spark] def doCheckpoint(): Unit = {
    RDDOperationScope.withScope(sc, "checkpoint", allowNesting = false, ignoreParent = true) {
      if (!doCheckpointCalled) {
        doCheckpointCalled = true
        if (checkpointData.isDefined) {
          if (checkpointAllMarkedAncestors) {
            // TODO We can collect all the RDDs that needs to be checkpointed, and then checkpoint
            // them in parallel.
            // Checkpoint parents first because our lineage will be truncated after we
            // checkpoint ourselves
            dependencies.foreach(_.rdd.doCheckpoint())
          }
          checkpointData.get.checkpoint()
        } else {
          dependencies.foreach(_.rdd.doCheckpoint())
        }
      }
    }
  }
```
&emsp;doCheckpoint函数首先会判断 checkpointData 是否被定义，即当前 RDD 是否已经被标记为需要 checkpoint，然后 finalRDD 会顺着依赖关系回溯扫描，先将需要的父 RDD checkpoint。之后调用 checkpointData.get.checkpoint() 对当前 RDD 做 checkpoint 操作。

### 2.3 checkpointData.checkpoint
```scala
//org.apache.spark.rdd.RDDCheckpointData
final def checkpoint(): Unit = {
    // Guard against multiple threads checkpointing the same RDD by
    // atomically flipping the state of this RDDCheckpointData
    RDDCheckpointData.synchronized {
      if (cpState == Initialized) {
        cpState = CheckpointingInProgress
      } else {
        return
      }
    }
    //持久化
    val newRDD = doCheckpoint()

    // Update our state and truncate the RDD lineage
    RDDCheckpointData.synchronized {
      cpRDD = Some(newRDD)
      cpState = Checkpointed
      //标记，切断依赖关系
      rdd.markCheckpointed()
    }
  }
```
&emsp;这里首先会将状态修改为 CheckpointingInProgress(正在checkpoint)，然后调用 val newRDD = doCheckpoint() 将 RDD 持久化，之后将状态置为 Checkpointed(已经checkpoint)，最后调用 rdd.markCheckpointed() 将当前 rdd 标志为已经 checkpoint。

&emsp;这里先讲解 rdd.markCheckpointed()，然后再介绍 doCheckpoint。
&emsp;markCheckpointed 会清除所有当前 rdd 的所有依赖信息，切断依赖关系。
```scala
private[spark] def markCheckpointed(): Unit = {
    clearDependencies()
    partitions_ = null
    deps = null    // Forget the constructor argument for dependencies too
  }
protected def clearDependencies(): Unit = {
    dependencies_ = null
  }
```

&emsp;RDDCheckpointData 是一个抽象类，checkpointData 在赋值的时候是赋以一个 ReliableRDDCheckpointData 对象，所以这里的 doCheckpoint() 调用的是 ReliableRDDCheckpointData 的 doCheckpoint()函数。
```scala
protected override def doCheckpoint(): CheckpointRDD[T] = {
    val newRDD = ReliableCheckpointRDD.writeRDDToCheckpointDirectory(rdd, cpDir)

    // Optionally clean our checkpoint files if the reference is out of scope
    if (rdd.conf.getBoolean("spark.cleaner.referenceTracking.cleanCheckpoints", false)) {
      rdd.context.cleaner.foreach { cleaner =>
        cleaner.registerRDDCheckpointDataForCleanup(newRDD, rdd.id)
      }
    }

    logInfo(s"Done checkpointing RDD ${rdd.id} to $cpDir, new parent is RDD ${newRDD.id}")
    newRDD
  }
```

### 2.4 持久化

&emsp;我们再打开 ReliableCheckpointRDD.writeRDDToCheckpointDirectory 函数，这个函数实现了将 RDD 持久化的操作，最后返回一个新的 ReliableCheckpointRDD 类型的 RDD。
&emsp;注意，在这个函数里面，调用了 sc.runJob，启动了一个**专门的 Job** 来完成 Checkpoint，这里的代价是比较大的，会重新计算原来的 RDD。
```scala
def writeRDDToCheckpointDirectory[T: ClassTag](
      originalRDD: RDD[T],
      checkpointDir: String,
      blockSize: Int = -1): ReliableCheckpointRDD[T] = {
    val checkpointStartTimeNs = System.nanoTime()

    //......

    //将写磁盘需要的配置文件（如core-site.xml等）brocast到其他worker节点上的BlockManager
    val broadcastedConf = sc.broadcast(
      new SerializableConfiguration(sc.hadoopConfiguration))
    // TODO: This is expensive because it computes the RDD again unnecessarily (SPARK-8582)

    //启动一个Job来完成Checkpoint
    sc.runJob(originalRDD,
      writePartitionToCheckpointFile[T](checkpointDirPath.toString, broadcastedConf) _)

    if (originalRDD.partitioner.nonEmpty) {
      writePartitionerToCheckpointDir(sc, originalRDD.partitioner.get, checkpointDirPath)
    }

    val checkpointDurationMs =
      TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - checkpointStartTimeNs)
    logInfo(s"Checkpointing took $checkpointDurationMs ms.")

    val newRDD = new ReliableCheckpointRDD[T](
      sc, checkpointDirPath.toString, originalRDD.partitioner)
    if (newRDD.partitions.length != originalRDD.partitions.length) {
      throw new SparkException(
        "Checkpoint RDD has a different number of partitions from original RDD. Original " +
          s"RDD [ID: ${originalRDD.id}, num of partitions: ${originalRDD.partitions.length}]; " +
          s"Checkpoint RDD [ID: ${newRDD.id}, num of partitions: " +
          s"${newRDD.partitions.length}].")
    }
    newRDD
  }
```
&emsp;至此，checkpoint 操作完成。

### 2.5 获取RDD

&emsp;在后面，当需要获取某个 RDD 的数据的时候，系统就会尝试从磁盘中读取数据，获取不到再重新计算。
&emsp;我们可以看到，rdd.getOrCompute 函数调用了 computeOrReadCheckpoint 方法，在rdd.computeOrReadCheckpoint方法中，如果看到已经完成 Checkpoint，就直接从 firstParent 中读取数据。
```scala
private[spark] def computeOrReadCheckpoint(split: Partition, context: TaskContext): Iterator[T] =
  {
    if (isCheckpointedAndMaterialized) {
      firstParent[T].iterator(split, context)
    } else {
      compute(split, context)
    }
  }

private[spark] def isCheckpointedAndMaterialized: Boolean =
    checkpointData.exists(_.isCheckpointed)
```
firstParent 其实就是当前 RDD 的第一个父 RDD，iterator 就会尝试通过 blockManager 从磁盘读取数据
```scala
/** Returns the first parent RDD */
  protected[spark] def firstParent[U: ClassTag]: RDD[U] = {
    dependencies.head.rdd.asInstanceOf[RDD[U]]
  }
final def iterator(split: Partition, context: TaskContext): Iterator[T] = {
    if (storageLevel != StorageLevel.NONE) {
      getOrCompute(split, context)
    } else {
      computeOrReadCheckpoint(split, context)
    }
  }

private[spark] def getOrCompute(partition: Partition, context: TaskContext): Iterator[T] = {
    val blockId = RDDBlockId(id, partition.index)
    var readCachedBlock = true
    // This method is called on executors, so we need call SparkEnv.get instead of sc.env.

    //尝试通过blockManager从磁盘读取
    SparkEnv.get.blockManager.getOrElseUpdate(blockId, storageLevel, elementClassTag, () => {
      readCachedBlock = false
      computeOrReadCheckpoint(partition, context)
    }) match {
      case Left(blockResult) =>
        if (readCachedBlock) {
          val existingMetrics = context.taskMetrics().inputMetrics
          existingMetrics.incBytesRead(blockResult.bytes)
          new InterruptibleIterator[T](context, blockResult.data.asInstanceOf[Iterator[T]]) {
            override def next(): T = {
              existingMetrics.incRecordsRead(1)
              delegate.next()
            }
          }
        } else {
          new InterruptibleIterator(context, blockResult.data.asInstanceOf[Iterator[T]])
        }
      case Right(iter) =>
        new InterruptibleIterator(context, iter.asInstanceOf[Iterator[T]])
    }
  }
```


## 3.总结
&emsp;下面总结 checkpoint 机制的完整流程：
1. 用户程序调用rdd.checkpoint(),将当前 rdd 标志为需要 checkpoint；
2. action 算子触发 sc.runJob()，runJob 调用 rdd.doCheckpoint()，触发 checkpoint 操作；
3. ReliableRDDCheckpointData 启动一个专门的 Job 完成 RDD 的持久化操作；
4. 持久化完成之后，当前 RDD 会清除所有依赖信息，切断依赖关系，结束 checkpoint。



&nbsp;
&nbsp;
>本文作者：ZJP
版权声明：本文为博主原创文章，遵循 [CC 4.0 BY-SA](http://creativecommons.org/licenses/by-sa/4.0/) 版权协议，转载请附上原文出处链接和本声明。
本文链接：http://zhoujiapeng.top/article/spark-fault-tolerant/
