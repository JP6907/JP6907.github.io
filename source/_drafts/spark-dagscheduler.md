---
title: Spark源码阅读 之 DAGScheduler 详解
catalog: true
subtitle: Spark DAGScheduler 详解
header-img: "/img/article_header/article_header.png"
tags:
- spark
- 大数据
- 源码阅读
categories:
- Spark
---


# Spark DAGScheduler 详解

> Spark版本：2.4.0

&emsp;DAGScheduler 是面向 Stage 的任务调度器，负责接收 Spark 应用提交的 Job，并根据 RDD 的依赖关系划分 Stage，并提交 Stage 给 TaskScheduler 调度器。本文将根据源码详细剖析 DAGScheduler 调度的完整流程。

### DAGScheduler
&emsp;DAGScheduler 会在 action 算子执行的时候被触发。action算子触发sc.runJob,经过几次调用之后，会调用：
```scala
//SparkContext
dagScheduler.runJob(rdd, cleanedFunc, partitions, callSite, resultHandler, localProperties.get)
```
&emsp;这里就是 DAGScheduler 开始执行调度的入口：
```scala
//dagScheduler.runJob
def runJob[T, U](
      rdd: RDD[T],
      func: (TaskContext, Iterator[T]) => U,
      partitions: Seq[Int],
      callSite: CallSite,
      resultHandler: (Int, U) => Unit,
      properties: Properties): Unit = {
    val start = System.nanoTime

    //作业提交
    val waiter = submitJob(rdd, func, partitions, callSite, resultHandler, properties)

    //等待作业完成
    ThreadUtils.awaitReady(waiter.completionFuture, Duration.Inf)
    waiter.completionFuture.value.get match {
      case scala.util.Success(_) =>
        logInfo("Job %d finished: %s, took %f s".format
          (waiter.jobId, callSite.shortForm, (System.nanoTime - start) / 1e9))
      case scala.util.Failure(exception) =>
        logInfo("Job %d failed: %s, took %f s".format
          (waiter.jobId, callSite.shortForm, (System.nanoTime - start) / 1e9))
        // SPARK-8644: Include user stack trace in exceptions coming from DAGScheduler.
        val callerStackTrace = Thread.currentThread().getStackTrace.tail
        exception.setStackTrace(exception.getStackTrace ++ callerStackTrace)
        throw exception
    }
  }
```
调用了 submitJob 方法来提交作业，这里会发生阻塞，等待作业完成。
```scala
//DAGScheduler
def submitJob[T, U](
      rdd: RDD[T],
      func: (TaskContext, Iterator[T]) => U,
      partitions: Seq[Int],
      callSite: CallSite,
      resultHandler: (Int, U) => Unit,
      properties: Properties): JobWaiter[U] = {
    // Check to make sure we are not launching a task on a partition that does not exist.
    
    ...

    assert(partitions.size > 0)
    val func2 = func.asInstanceOf[(TaskContext, Iterator[_]) => _]
    val waiter = new JobWaiter(this, jobId, partitions.size, resultHandler)

    //发送JobSubmitted消息
    eventProcessLoop.post(JobSubmitted(
      jobId, rdd, func2, partitions.toArray, callSite, waiter,
      SerializationUtils.clone(properties)))
    waiter
  }
```
submitJob 创建了 JobWaiter 对象，并发送 JobSubmitted 消息给 DAGScheduler 内嵌的 DAGSchedulerEventProcessLoop 类对象 eventProcessLoop 进行处理:
```scala
//org.apache.spark.scheduler.DAGSchedulerEventProcessLoop
private def doOnReceive(event: DAGSchedulerEvent): Unit = event match {
    case JobSubmitted(jobId, rdd, func, partitions, callSite, listener, properties) =>
      dagScheduler.handleJobSubmitted(jobId, rdd, func, partitions, callSite, listener, properties)

      ...
```
dagScheduler.handleJobSubmitted 方法是个很关键的方法，包含了作业提交处理的主要逻辑。
```scala
private[scheduler] def handleJobSubmitted(jobId: Int,
      finalRDD: RDD[_],
      func: (TaskContext, Iterator[_]) => _,
      partitions: Array[Int],
      callSite: CallSite,
      listener: JobListener,
      properties: Properties) {
    var finalStage: ResultStage = null
    try {
      // New stage creation may throw an exception if, for example, jobs are run on a
      // HadoopRDD whose underlying HDFS files have been deleted.

      //创建 ResultStage
      finalStage = createResultStage(finalRDD, func, partitions, jobId, callSite)
    } catch {
      
    ...

    finalStage.setActiveJob(job)
    val stageIds = jobIdToStageIds(jobId).toArray
    val stageInfos = stageIds.flatMap(id => stageIdToStage.get(id).map(_.latestInfo))
    listenerBus.post(
      SparkListenerJobStart(job.jobId, jobSubmissionTime, stageInfos, properties))
    
    //提交 Stage，开始切分Stage
    submitStage(finalStage)
  }
```
&emsp;finalRDD 是这 RDD 依赖链的最后一个RDD，即 action 触发作业提交的那个RDD。有两种类型的 Stage，ResultStage 和 ShuffleMapStage。finalRDD 是最后一个 RDD，包含在最后一个Stage中，结果直接输出，不需要再经过 shuffle，因此 这里创建的是 ResultStage（除最后的stage之外，前面的stage都是shufflemapstage）。最后调用 submitStage(finalStage)，会根据 finalStage 沿着父stage继续往前回溯开始切分stage。
```scala
//org.apache.spark.scheduler.DAGScheduler
/** Submits stage, but first recursively submits any missing parents. */
  private def submitStage(stage: Stage) {
    val jobId = activeJobForStage(stage)
    if (jobId.isDefined) {
      logDebug("submitStage(" + stage + ")")
      if (!waitingStages(stage) && !runningStages(stage) && !failedStages(stage)) {

        //寻找父stage
        val missing = getMissingParentStages(stage).sortBy(_.id)
        logDebug("missing: " + missing)

        //没有父stage，是最开头的stage，可以提交了
        if (missing.isEmpty) {
          logInfo("Submitting " + stage + " (" + stage.rdd + "), which has no missing parents")
          
          //提交这个stage给taskscheduler
          submitMissingTasks(stage, jobId.get)
        } else {
          for (parent <- missing) {
            
            //沿着父stage继续往前回溯
            submitStage(parent)
          }
          waitingStages += stage
        }
      }
    } else {
      abortStage(stage, "No active job for stage " + stage.id, None)
    }
  }
```
&emsp;submitStage 会根据依赖关系划分 Stage，通过递归调用从 finalStage 一直沿着父stage继续往前回溯，直到 Stage 没有父 Stage（missing.isEmpty 为 true） 时就调用 submitMissingTasks 方法提交 Stage 给 taskscheduler 去执行；如果有父 Stage，即 missing 不为空，则遍历 missing，回调当前 Stage 的所有父 Stage submitStage(parent)，然后将当前 Stage被放入 waitingStages 中，等待以后执行 (后面 submitMissingTasks 会介绍)。通过这种方式将 job 划分为一个或者多个 Stage，并且能够保证父 Stage 比子 Stage 先执行其中的任务。&nbsp;
&emsp;这里有两个关键的函数，getMissingParentStages 和 submitMissingTasks，下面具体介绍这两个函数。&nbsp;
&emsp;getMissingParentStages 的功能是寻找传入 stage 的所有父 stage。这里以是否为 **ShuffleDependency(宽依赖)** 作为划分 Stage 的 界限，如果 RDD 的依赖是宽依赖，说明已经找到当前 Stage 的边界，可以进行切分，调用 getOrCreateShuffleMapStage 生成该 Stage 的父 Stage。**注意**，这里创建的是 ShuffleMapStage，dagScheduler.handleJobSubmitted 中创建的是 ResultStage；如果是窄依赖，则压入 waitingForVisit 栈顶，等待下一次调用 visit 方法时回溯。
```scala
private def getMissingParentStages(stage: Stage): List[Stage] = {
    val missing = new HashSet[Stage]
    val visited = new HashSet[RDD[_]]
    // We are manually maintaining a stack here to prevent StackOverflowError
    // caused by recursively visiting
    val waitingForVisit = new ArrayStack[RDD[_]]
    def visit(rdd: RDD[_]) {
      if (!visited(rdd)) {
        visited += rdd
        val rddHasUncachedPartitions = getCacheLocs(rdd).contains(Nil)
        if (rddHasUncachedPartitions) {
          for (dep <- rdd.dependencies) {
            dep match {
              case shufDep: ShuffleDependency[_, _, _] =>
                val mapStage = getOrCreateShuffleMapStage(shufDep, stage.firstJobId)
                if (!mapStage.isAvailable) {
                  missing += mapStage
                }
              case narrowDep: NarrowDependency[_] =>
                waitingForVisit.push(narrowDep.rdd)
            }
          }
        }
      }
    }
    waitingForVisit.push(stage.rdd)
    while (waitingForVisit.nonEmpty) {
      visit(waitingForVisit.pop())
    }
    missing.toList
  }
```
&emsp;submitMissingTasks 会提交当前切分好的 Stage 并执行其中的任务。submitMissingTasks 中将 Stage 封装成 TaskSet 通过 taskScheduler.submitTasks 提交给 TaskScheduler 处理。&nbsp;
&emsp;另外，submitMissingTasks 会首先生成 Tasks 的具体执行位置，然后在根据这些位置创建 TaskSet。Task 有两种类型，ShuffleMapStage 和 ResultStage，ShuffleMapStage 会根据 Stage 所依赖的 RDD 的partition 分布产生和 partition 数量相等的 Task，这些 Task 根据 partition 的 locality 进行分布。
```scala
private def submitMissingTasks(stage: Stage, jobId: Int) {
    logDebug("submitMissingTasks(" + stage + ")")

    ...

    val taskIdToLocations: Map[Int, Seq[TaskLocation]] = try {
      stage match {
        case s: ShuffleMapStage =>
          partitionsToCompute.map { id => (id, getPreferredLocs(stage.rdd, id))}.toMap
        case s: ResultStage =>
          partitionsToCompute.map { id =>
            val p = s.partitions(id)
            (id, getPreferredLocs(stage.rdd, p))
          }.toMap
      }

    ...

    val tasks: Seq[Task[_]] = try {
      val serializedTaskMetrics = closureSerializer.serialize(stage.latestInfo.taskMetrics).array()
      stage match {
        case stage: ShuffleMapStage =>
          stage.pendingPartitions.clear()
          partitionsToCompute.map { id =>
            val locs = taskIdToLocations(id)
            val part = partitions(id)
            stage.pendingPartitions += id
            new ShuffleMapTask(stage.id, stage.latestInfo.attemptNumber,
              taskBinary, part, locs, properties, serializedTaskMetrics, Option(jobId),
              Option(sc.applicationId), sc.applicationAttemptId, stage.rdd.isBarrier())
          }

        case stage: ResultStage =>
          partitionsToCompute.map { id =>
            val p: Int = stage.partitions(id)
            val part = partitions(p)
            val locs = taskIdToLocations(id)
            new ResultTask(stage.id, stage.latestInfo.attemptNumber,
              taskBinary, part, locs, id, properties, serializedTaskMetrics,
              Option(jobId), Option(sc.applicationId), sc.applicationAttemptId,
              stage.rdd.isBarrier())
          }
      }
    } catch {
      case NonFatal(e) =>
        abortStage(stage, s"Task creation failed: $e\n${Utils.exceptionString(e)}", Some(e))
        runningStages -= stage
        return
    }

    if (tasks.size > 0) {
      logInfo(s"Submitting ${tasks.size} missing tasks from $stage (${stage.rdd}) (first 15 " +
        s"tasks are for partitions ${tasks.take(15).map(_.partitionId)})")

        //提交任务
      taskScheduler.submitTasks(new TaskSet(
        tasks.toArray, stage.id, stage.latestInfo.attemptNumber, jobId, properties))
    } else {
      
        ...

      submitWaitingChildStages(stage)
    }
  }
```
&emsp;总共有两种类型的 Stage，ShuffleMapStage 和 ResultStage,只有最后一个 Stage 才是 ResultStage，其它的都是 ShuffleMapStage。这里会根据不同的 Stage 创建不同的 Task，ShuffleMapStage 对应 ShuffleMapTask，ResultStage 对应 ResultTask。当前 Stage 的 Task 提交之后，最后执行 submitWaitingChildStages(stage)，这里会从之前放入 waitingStages 的 Stage 中过滤出属于当前 Stage 的子 Stage，继续调用 submitStage，即父 Stage 执行结束后，开始执行子 Stage。
```scala
private def submitWaitingChildStages(parent: Stage) {
    logTrace(s"Checking if any dependencies of $parent are now runnable")
    logTrace("running: " + runningStages)
    logTrace("waiting: " + waitingStages)
    logTrace("failed: " + failedStages)
    val childStages = waitingStages.filter(_.parents.contains(parent)).toArray
    waitingStages --= childStages
    for (stage <- childStages.sortBy(_.firstJobId)) {
      submitStage(stage)
    }
  }
```


### 总结
&emsp;action 算子会触发 SparkContext.runjob，然后调用 dagScheduler.runJob 触发 DAGScheduler 开始 Stage 切分。有两种类型的 Stage，ResultStage 和 ShuffleMapStage。DAGScheduler 以 RDD 的依赖是否为 ShuffleDependency 来决定是否需要进行 Stage 切分。因此相邻 Stage 之间会存在一个 Shuffle 的过程，所以除了最后一个 Stage 之外所有的 Stage 都是 ShuffleMapStage。最后一个 Stage 只需要将计算结果输出就行，不再需要 Shuffle，所有是 ResultStage。
&emsp;简单来说，DAGScheduler 的 Stage 切分过程为：根据 RDD 依赖链的最后一个 RDD(finalRDD) 创建一个 ResultStage，然后沿着 finalRDD 不断往前遍历，遇到 RDD 的依赖为 ShuffleDependency 时就创建一个 ShuffleMapStage。当无法再继续找到父 Stage 的时候，则提交当前 Stage 给 TaskScheduler，接着提交当前 Stage 的子 Stage，直到所有的 Stage 都被提交。


&nbsp;
&nbsp;
>本文作者：ZJP
版权声明：本文为博主原创文章，遵循 [CC 4.0 BY-SA](http://creativecommons.org/licenses/by-sa/4.0/) 版权协议，转载请附上原文出处链接和本声明。
本文链接：http://zhoujiapeng.top/Spark/spark-dagscheduler/