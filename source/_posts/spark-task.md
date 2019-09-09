---
title: Spark源码阅读 之 Task 详解
subtitle: Spark Task 详解
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - spark
  - 大数据
  - 源码阅读
categories:
  - Spark
date: 2019-09-09 14:34:55
---



# Spark Task 详解

> Spark版本：2.4.0


## 1. Task

 &emsp;首先我们先来看一下源码中的介绍，Task 抽象类有这么一段注释：
```
/*
 * A Spark job consists of one or more stages. The very last stage in a job consists of multiple
 * ResultTasks, while earlier stages consist of ShuffleMapTasks. A ResultTask executes the task
 * and sends the task output back to the driver application. A ShuffleMapTask executes the task
 * and divides the task output to multiple buckets (based on the task's partitioner).
 */
 private[spark] abstract class Task[T](
     ...
```
&emsp;意思大致为：一个 Spark job 会包含多个 stage，最后一个 stage 由多个 ResultTask 组成，而签名的 stage 由多个 ShuffleMapTask 组成。ResultTask 会将任务执行结果发送给 driver，ShuffleMapTask 会将任务执行结果根据 task 的分区输出到多个 buckets（一些本地文件）。
&emsp;也就是说 Spark 中有两种类型的 Task：ResultTask：对应于 DAG 图中最后一个 Stage（也就是 ResultStage）；ShuffleMapTask：对应于非最后的 Stage（也就是 ShuffleMapStage）。


## 2. TaskRunner

&emsp;我们再来看一下 Task 的运行过程，TaskRunner 类实现了 Task 的主要运行工作原理：

```scala
  class TaskRunner(
      execBackend: ExecutorBackend,
      private val taskDescription: TaskDescription)
    extends Runnable {

    ...


    override def run(): Unit = {
      
      ...

      try {
        // 对task数据，反序列化
        Executor.taskDeserializationProps.set(taskDescription.properties)

        // 将依赖的文件资源、jar拷贝到到task读取文件的对应目录
        updateDependencies(taskDescription.addedFiles, taskDescription.addedJars)

        // 反序列化 Taskset
        task = ser.deserialize[Task[Any]](
          taskDescription.serializedTask, Thread.currentThread.getContextClassLoader)
        task.localProperties = taskDescription.properties
        task.setTaskMemoryManager(taskMemoryManager)

        ...

        val value = Utils.tryWithSafeFinally {

          // 如果是 ShuffleMapTask，这里的 res 就是 MapStatus
          // 如果后面执行的还是一个ShuffleMapTask，就会联系MapOutputTracker去获取上一个ShuffleMapTask的输出结果。 ResultTask也是一样的。
          val res = task.run(
            taskAttemptId = taskId,
            attemptNumber = taskDescription.attemptNumber,
            metricsSystem = env.metricsSystem)
          threwException = false
          res
        } {
        
        ...

        // 对MapStatus 序列化和封装，因为要发送给driver
        val resultSer = env.serializer.newInstance()
        val beforeSerialization = System.currentTimeMillis()
        val valueBytes = resultSer.serialize(value)
        val afterSerialization = System.currentTimeMillis()

        ...

        setTaskFinishedAndClearInterruptStatus()

        // 调用了executor所在的CoraseGrainedBackend的statusUpdate()方法
        execBackend.statusUpdate(taskId, TaskState.FINISHED, serializedResult)

      } 
      ...
    }

  }
```

&emsp;任务的实际执行是通过 task.run ，我们看一下这个方法
```scala
//Task.run
final def run(
      taskAttemptId: Long,
      attemptNumber: Int,
      metricsSystem: MetricsSystem): T = {
    SparkEnv.get.blockManager.registerTask(taskAttemptId)
    
    //task执行上下文，记录了task执行的全局性的数据，例如task重试次数，属于哪个stage，rdd处理的partition等
    val taskContext = new TaskContextImpl(
      stageId,
      stageAttemptId, // stageAttemptId and stageAttemptNumber are semantically equal
      partitionId,
      taskAttemptId,
      attemptNumber,
      taskMemoryManager,
      localProperties,
      metricsSystem,
      metrics)

    ...

    try {
      runTask(context)
    } catch {
      case e: Throwable =>
        // Catch all errors; run task failure callbacks, and rethrow the exception.
        try {
          context.markTaskFailed(e)
        } catch {
          case t: Throwable =>
            e.addSuppressed(t)
        }
        context.markTaskCompleted(Some(e))
        throw e
    } finally {
      ...
    }
  }
```
&emsp;这里调用了 runTask 方法，Task 类中的 runTask 方法并没有定义具体的逻辑，这是因为 ResultTask 和 ShuffleMapTask 的逻辑不一样。

## 3. ShuffleMapTask.runTask

&emsp;我们先看一些 ShuffleMapTask.runTask 方法
ShuffleMapTask runTask
```scala
//ShuffleMapTask runTask
override def runTask(context: TaskContext): MapStatus = {
    // Deserialize the RDD using the broadcast variable.
    val threadMXBean = ManagementFactory.getThreadMXBean
    val deserializeStartTime = System.currentTimeMillis()
    val deserializeStartCpuTime = if (threadMXBean.isCurrentThreadCpuTimeSupported) {
      threadMXBean.getCurrentThreadCpuTime
    } else 0L
    val ser = SparkEnv.get.closureSerializer.newInstance()
    val (rdd, dep) = ser.deserialize[(RDD[_], ShuffleDependency[_, _, _])](
      ByteBuffer.wrap(taskBinary.value), Thread.currentThread.getContextClassLoader)
    _executorDeserializeTime = System.currentTimeMillis() - deserializeStartTime
    _executorDeserializeCpuTime = if (threadMXBean.isCurrentThreadCpuTimeSupported) {
      threadMXBean.getCurrentThreadCpuTime - deserializeStartCpuTime
    } else 0L

    var writer: ShuffleWriter[Any, Any] = null
    try {
      val manager = SparkEnv.get.shuffleManager
      writer = manager.getWriter[Any, Any](dep.shuffleHandle, partitionId, context)

      // 首先，调用rdd的iterator方法来获取数据，并且传入了当前要处理的partition
      // 核心逻辑就在rdd的iterator()方法中
      // write 方法将数据写入到对应的 bucket 中，供下游 task 拉取
      writer.write(rdd.iterator(partition, context).asInstanceOf[Iterator[_ <: Product2[Any, Any]]])

      // 返回结果 MapStatus ，里面封装了ShuffleMapTask存储在哪里，其实就是BlockManager相关信息
      writer.stop(success = true).get
    } catch {
      case e: Exception =>
        try {
          if (writer != null) {
            writer.stop(success = false)
          }
        } catch {
          case e: Exception =>
            log.debug("Could not stop writer", e)
        }
        throw e
    }
  }
```

&emsp;与 ResultTask 对 partition 数据进行计算得到计算结果并汇报给 driver 不同，ShuffleMapTask 的职责是为下游的 RDD 计算出输入数据。更具体的说，ShuffleMapTask 要计算出 partition 数据并通过 shuffle write 写入磁盘（由 BlockManager 来管理）来等待下游的 RDD 通过 shuffle read 读取，其核心流程如下：

1. 从 SparkEnv 中获取 ShuffleManager 对象
2. 从 ShuffleManager 中获取 ShuffleWriter 对象 writer
3. 得到对应 partition 的迭代器后，通过 writer 将数据写入文件系统中
4. 停止 writer 并返回结果

&emsp;MapOutputTrancker 用于跟踪 map 任务的输出状态，即 ShuffleMapTask 的输出 MapStatue，此状态便于 reduce 任务定位到 map 输出结果所在的节点地址，进而获取中间输出结果。

## 4. ResultTask.runTask
&emsp;了解完 ShuffleMapTask，我们再来看一下 ResultTask：


```scala
//ResultTask.runTask
override def runTask(context: TaskContext): U = {
    // Deserialize the RDD and the func using the broadcast variables.
    val threadMXBean = ManagementFactory.getThreadMXBean
    val deserializeStartTime = System.currentTimeMillis()
    val deserializeStartCpuTime = if (threadMXBean.isCurrentThreadCpuTimeSupported) {
      threadMXBean.getCurrentThreadCpuTime
    } else 0L
    val ser = SparkEnv.get.closureSerializer.newInstance()

    //反序列化得到 rdd 及 rdd 的处理方法 func
    val (rdd, func) = ser.deserialize[(RDD[T], (TaskContext, Iterator[T]) => U)](
      ByteBuffer.wrap(taskBinary.value), Thread.currentThread.getContextClassLoader)
    _executorDeserializeTime = System.currentTimeMillis() - deserializeStartTime
    _executorDeserializeCpuTime = if (threadMXBean.isCurrentThreadCpuTimeSupported) {
      threadMXBean.getCurrentThreadCpuTime - deserializeStartCpuTime
    } else 0L

    //对 rdd 指定 partition 的迭代器执行 func 函数
    func(context, rdd.iterator(partition, context))
  }
```
&emsp;ResultTask.runTask 的核心流程为：
1. 反序列化得到 rdd 及 处理方法 func
2. 对 rdd 指定 partition 的迭代器执行 func 函数并返回结果

## 5. Shuffle
&emsp;我们前面多次提到，ShuffleMapTask 会将计算结果写入 bucket 作为上游数据，供下游的 ResultTask 需要时拉取。接下来我们大致看一下这个过程：
&emsp;Shuffle 的写操作（Shuffle writer）是将 MapTask 操作后的数据写入磁盘中，ShuffleMapTask 会为每个 ReduceTask 创建对应的 Bucket，ShuffleMapTask 产生的结果会根据设置的 partitioner（分区）得到对应的 BucketId，然后填充到相应的 Bucket 中。每个 ShuffleMapTask 的输出结果可能包含所有的 ReduceTask 需要的数据，所以每个 ShuffleMapTask 创建的 Bucket 数目是和 ReduceTask 的数目相等的。ShuffleMapTask 创建的 Bucket 对应于磁盘上的一个文件，用来储存结果，此文件也被称为 BlockFile。如下图所示：
![spark-shuffle](https://github.com/JP6907/Pic/blob/master/spark/spark-shuffle1.jpg?raw=true)
&emsp;每个 Map Task 为每个 Reduce Task 生成一个文件，通常会产生大量的文件（即对应为 M*R 个中间文件，其中 M 表示 Map Task 个数，R 表示 Reduce Task 个数），伴随大量的随机磁盘 I/O 操作与大量的内存开销。总结下这里的两个严重问题：
1. 生成大量文件，占用文件描述符，同时引入 DiskObjectWriter 带来的 Writer Handler 的缓存也非常消耗内存；
2. 如果在 Reduce Task 时需要合并操作的话，会把数据放在一个 HashMap 中进行合并，如果数据量较大，很容易引发 OOM。

&emsp;因此，Spark 做了改进，引入了 File Consolidation 机制。
&emsp;一个 Executor 上所有的 Map Task 生成的分区文件只有一份，即将所有的 Map Task 相同的分区文件合并，这样每个 Executor 上最多只生成 N 个分区文件。
![spark-shuffle-aggregate](https://github.com/JP6907/Pic/blob/master/spark/spark-shuffle2.jpg?raw=true)

&emsp;我们再来看一下，shuffle 过程是怎么利用 MapOutputTrancker 记录的 MapStatue：
1. 在 ShuffleRDD 的 compute 方法中，会获取 BlockStoreShuffleReader，然后在 BlockStoreShuffleReader 中，BlockStoreShuffleReader 的 read 方法会调用 mapOutputTracker.getMapSizesByExecutorId 方法获取一组二元组序列Seq[(BlockManagerId, Seq[(BlockId, Long)])]，第一项代表了 BlockManagerId，第二项描述了存储于该 BlockManager上的一组 shuffle blocks。

2. getMapSizesByExecutorId会调用getStatuses方法获取MapStatus集合，然后最后返回MapStatus集合。

3. 最后根据执行的分区范围[startPartition, endPartition]将返回的结果Array[MapStatus]转换成Seq[(BlockManagerId, Seq[(BlockId, Long)])]。

4. 利用这个Seq[(BlockManagerId, Seq[(BlockId, Long)])]，去指定的BlockManager中去拉取对应的Block块的数据用来迭代计算。

```scala
//BlockStoreShuffleReader.read
override def read(): Iterator[Product2[K, C]] = {
    val wrappedStreams = new ShuffleBlockFetcherIterator(
      context,
      blockManager.shuffleClient,
      blockManager,
      mapOutputTracker.getMapSizesByExecutorId(handle.shuffleId, startPartition, endPartition),
      serializerManager.wrapStream,
      // Note: we use getSizeAsMb when no suffix is provided for backwards compatibility
      SparkEnv.get.conf.getSizeAsMb("spark.reducer.maxSizeInFlight", "48m") * 1024 * 1024,
      SparkEnv.get.conf.getInt("spark.reducer.maxReqsInFlight", Int.MaxValue),
      SparkEnv.get.conf.get(config.REDUCER_MAX_BLOCKS_IN_FLIGHT_PER_ADDRESS),
      SparkEnv.get.conf.get(config.MAX_REMOTE_BLOCK_SIZE_FETCH_TO_MEM),
      SparkEnv.get.conf.getBoolean("spark.shuffle.detectCorrupt", true))

    ...
  }
```

&nbsp;
&nbsp;
>本文作者：ZJP
版权声明：本文为博主原创文章，遵循 [CC 4.0 BY-SA](http://creativecommons.org/licenses/by-sa/4.0/) 版权协议，转载请附上原文出处链接和本声明。
本文链接：http://zhoujiapeng.top/Spark/spark-task/