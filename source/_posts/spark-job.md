---
title: Spark源码阅读 之 Spark 作业和调度
catalog: true
date: 2019-09-03 10:32:19
subtitle: Spark 作业和调度
header-img: "/img/article_header/article_header.png"
tags:
- spark
- 大数据
- 源码阅读
categories:
  - Spark
---

# Spark 作业和调度

> Spark版本：2.4.0

&emsp;Spark 应用程序由 driver program 和在 Executor 内执行的代码两部分组成，Spark的作业调度主要说的就是这些基于 RDD 的一系列操作算子构成一个 Job，然后在Executor中执行。操作算子分为 transformation 算子和 action 算子，transformation 算子是 lazy 的，即延迟执行，只有出现 action才会真正触发 Job 的提交和调度执行。

## 1.作业(Job)提交
&emsp;以单词计数程序为例：
```scala
val line = sc.textFile("README.md")
val wordcout = line.flatMap(_.split("")).map(x=>(x,1)).reduceByKey(_+_).count()
```
&emsp;这个Job的真正执行是 count() 这个 action 算子触发的，打开该方法，可以看到这个方法调用了 SparkContext.runJob 方法来提交作业：
```scala
def count(): Long = sc.runJob(this, Utils.getIteratorSize _).sum
```
&emsp;对于 RDD 来说，它们会根据彼此之间的依赖关系形成一个DAG有向无环图,然后把这DAG图交给DAGScheduler来处理。SparkContext.runJob 经过几次回调之后会调用 DAGScheduler 的 runJob 方法，这时作业提交就进入了 DAGScheduler 的处理阶段。
```scala
//org.apache.spark.SparkContext
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
## 2.DAGScheduler划分Stage并提交
&emsp;DAGScheduler 是面向 Stage 的任务调度器，负责接收 Spark 应用提交的 Job，并根据 RDD 的依赖关系划分 Stage，并提交 Stage 给 TaskScheduler 调度器。
&emsp;下面是 DAGScheduler 的 runJob 方法实现：
```scala
def runJob[T, U](
      rdd: RDD[T],
      func: (TaskContext, Iterator[T]) => U,
      partitions: Seq[Int],
      callSite: CallSite,
      resultHandler: (Int, U) => Unit,
      properties: Properties): Unit = {
    val start = System.nanoTime
    val waiter = submitJob(rdd, func, partitions, callSite, resultHandler, properties)
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
handleJobSubmitted 方法是个很关键的方法，在其内部会根据 finalRDD 构建一个 Stage，这也意味着 Stage 划分的开始，最后调用 submitStage 方法来提交这个 Stage
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
      finalStage = createResultStage(finalRDD, func, partitions, jobId, callSite)
    } catch {
      
    ...

    finalStage.setActiveJob(job)
    val stageIds = jobIdToStageIds(jobId).toArray
    val stageInfos = stageIds.flatMap(id => stageIdToStage.get(id).map(_.latestInfo))
    listenerBus.post(
      SparkListenerJobStart(job.jobId, jobSubmissionTime, stageInfos, properties))
    submitStage(finalStage)
  }
```
finalRDD 是这 RDD 依赖链的最后一个RDD，即 action 触发作业提交的那个RDD，这里的 finalStage 并不是切分好的 Stage，是根据 finalRDD 生成的一个跟当前作业关联的一个 Stage，后续要使用 finalStage 来进行 Stage 的切分，即 submitStage 方法：
```scala
//org.apache.spark.scheduler.DAGScheduler
/** Submits stage, but first recursively submits any missing parents. */
  private def submitStage(stage: Stage) {
    val jobId = activeJobForStage(stage)
    if (jobId.isDefined) {
      logDebug("submitStage(" + stage + ")")
      if (!waitingStages(stage) && !runningStages(stage) && !failedStages(stage)) {
        val missing = getMissingParentStages(stage).sortBy(_.id)
        logDebug("missing: " + missing)
        if (missing.isEmpty) {
          logInfo("Submitting " + stage + " (" + stage.rdd + "), which has no missing parents")
          submitMissingTasks(stage, jobId.get)
        } else {
          for (parent <- missing) {
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
&emsp;submitStage 会根据依赖关系划分 Stage，通过递归调用从 finalStage 一直往前找到它的父 Stage，直到 Stage 没有父 Stage（missing.isEmpty 为 true） 时就调用 submitMissingTasks 方法提交 Stage 执行该 Stage 中的 tasks；如果有父 Stage，即 missing 不为空，则遍历 missing，回调当前 Stage 的所有父 Stage submitStage(parent)，然后将当前 Stage被放入 waitingStages 中，等待以后执行 (后面 submitMissingTasks 会介绍)。通过这种方式将 job 划分为一个或者多个 Stage，并且能够保证父 Stage 比子 Stage 先执行其中的任务。&nbsp;
&emsp;这里有两个关键的函数，getMissingParentStages 和 submitMissingTasks，下面具体介绍这两个函数。&nbsp;
&emsp;getMissingParentStages 会根据传入的 Stage 中的 finalRDD 的 dependence 类型来创建它的父 Stage。这里以是否为 ShuffleDependency(宽依赖) 作为划分 Stage 的 界限，如果 RDD 的依赖是宽依赖，说明已经找到当前 Stage 的边界，可以进行切分，则调用 getOrCreateShuffleMapStage 生成该 Stage 的父 Stage；如果是窄依赖，则压入 waitingForVisit 栈顶，等待下一次调用 visit 方法时回溯。
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
&emsp;当前 Stage 的 Task 提交之后，最后执行 submitWaitingChildStages(stage)，这里会从之前放入 waitingStages 的 Stage 中过滤出属于当前 Stage 的子 Stage，继续调用 submitStage。
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
## 3.TaskScheduler提交Task
&emsp;前面 DAGScheduler 调用 taskScheduler.submitTasks，将 Task 提交给 TaskScheduler 去处理。TaskSchedulerImpl(TaskScheduler的实现子类) 会初始化一个 TaskSetManager 对其生命周期进行管理，当 TaskSchedulerImpl 得到 Worker 节点上 Executor 计算资源的时候，会通过 TaskSetManager 来发送具体的 Task 到　Executor 上执行计算。
```scala
//org.apache.spark.scheduler.TaskSchedulerImpl
override def submitTasks(taskSet: TaskSet) {
    val tasks = taskSet.tasks
    logInfo("Adding task set " + taskSet.id + " with " + tasks.length + " tasks")
    this.synchronized {
      val manager = createTaskSetManager(taskSet, maxTaskFailures)
      val stage = taskSet.stageId
      val stageTaskSets =
        taskSetsByStageIdAndAttempt.getOrElseUpdate(stage, new HashMap[Int, TaskSetManager])
      stageTaskSets(taskSet.stageAttemptId) = manager
      val conflictingTaskSet = stageTaskSets.exists { case (_, ts) =>
        ts.taskSet != taskSet && !ts.isZombie
      }
      if (conflictingTaskSet) {
        throw new IllegalStateException(s"more than one active taskSet for stage $stage:" +
          s" ${stageTaskSets.toSeq.map{_._2.taskSet.id}.mkString(",")}")
      }
      schedulableBuilder.addTaskSetManager(manager, manager.taskSet.properties)

      if (!isLocal && !hasReceivedTask) {
        starvationTimer.scheduleAtFixedRate(new TimerTask() {
          override def run() {
            if (!hasLaunchedTask) {
              logWarning("Initial job has not accepted any resources; " +
                "check your cluster UI to ensure that workers are registered " +
                "and have sufficient resources")
            } else {
              this.cancel()
            }
          }
        }, STARVATION_TIMEOUT_MS, STARVATION_TIMEOUT_MS)
      }
      hasReceivedTask = true
    }
    backend.reviveOffers()
  }
```
最后调用 backend.reviveOffers(),进行计算资源的分配并启动 Task，这里的 backend 是 StandaloneSchedulerBackend ，它直接从父类 CoarseGrainedSchedulerBackend 继承 reviveOffers 方法。
```scala
//org.apache.spark.scheduler.cluster.CoarseGrainedSchedulerBackend
override def reviveOffers() {
    driverEndpoint.send(ReviveOffers)
  }
```
这里会向 driver 发送 ReviveOffers 消息,从 driverEndpoint 的初始化可以看到实际上接收并处理消息的类是 CoarseGrainedSchedulerBackend 的内嵌类 DriverEndpoint。
```scala
//org.apache.spark.scheduler.cluster.CoarseGrainedSchedulerBackend
override def start() {
    val properties = new ArrayBuffer[(String, String)]
    for ((key, value) <- scheduler.sc.conf.getAll) {
      if (key.startsWith("spark.")) {
        properties += ((key, value))
      }
    }

    // TODO (prashant) send conf instead of properties
    driverEndpoint = createDriverEndpointRef(properties)
  }
protected def createDriverEndpointRef(
    properties: ArrayBuffer[(String, String)]): RpcEndpointRef = {
    rpcEnv.setupEndpoint(ENDPOINT_NAME, createDriverEndpoint(properties))
}
protected def createDriverEndpoint(properties: Seq[(String, String)]): DriverEndpoint = {
    new DriverEndpoint(rpcEnv, properties)
}
```
我们看一下处理消息的逻辑：
```scala
override def receive: PartialFunction[Any, Unit] = {
    ...
    case ReviveOffers =>
        makeOffers()
    ...
}

// Make fake resource offers on all executors
private def makeOffers() {
    // Make sure no executor is killed while some task is launching on it
    val taskDescs = CoarseGrainedSchedulerBackend.this.synchronized {
    // Filter out executors under killing
    val activeExecutors = executorDataMap.filterKeys(executorIsAlive)
    val workOffers = activeExecutors.map {
        case (id, executorData) =>
            new WorkerOffer(id, executorData.executorHost, executorData.freeCores,
              Some(executorData.executorAddress.hostPort))
    }.toIndexedSeq
        scheduler.resourceOffers(workOffers)
    }
    if (!taskDescs.isEmpty) {
        launchTasks(taskDescs)
    }
}

// Launch tasks returned by a set of resource offers
    private def launchTasks(tasks: Seq[Seq[TaskDescription]]) {
      for (task <- tasks.flatten) {
        val serializedTask = TaskDescription.encode(task)
        if (serializedTask.limit() >= maxRpcMessageSize) {
          Option(scheduler.taskIdToTaskSetManager.get(task.taskId)).foreach { taskSetMgr =>
            try {
              var msg = "Serialized task %s:%d was %d bytes, which exceeds max allowed: " +
                "spark.rpc.message.maxSize (%d bytes). Consider increasing " +
                "spark.rpc.message.maxSize or using broadcast variables for large values."
              msg = msg.format(task.taskId, task.index, serializedTask.limit(), maxRpcMessageSize)
              taskSetMgr.abort(msg)
            } catch {
              case e: Exception => logError("Exception in error callback", e)
            }
          }
        }
        else {
          val executorData = executorDataMap(task.executorId)
          executorData.freeCores -= scheduler.CPUS_PER_TASK

          logDebug(s"Launching task ${task.taskId} on executor id: ${task.executorId} hostname: " +
            s"${executorData.executorHost}.")

          executorData.executorEndpoint.send(LaunchTask(new SerializableBuffer(serializedTask)))
        }
      }
    }

````
&emsp;scheduler.resourceOffers(workOffers) 会申请分配计算所需要的资源，最后调用launchTasks(taskDescs)启动任务。&nbsp;
&emsp;在 launchTasks 中，会根据申请的资源用 for 循环把 Task 一个个发送到Worker节点上的 CoarseGrainedSchedulerBackend 内部的 Executor 来执行 Task。
```scala
//CoarseGrainedSchedulerBackend中的Executor集合
private val executorDataMap = new HashMap[String, ExecutorData]
```

## 4.Executor运行Task并返回结果
&emsp;打开Executor的LaunchTask方法,这里使用TaskRunner来管理Task运行时的所有细节，然后把TaskRunner对象放到Java的threadPool去执行。
```scala
def launchTask(context: ExecutorBackend, taskDescription: TaskDescription): Unit = {
    val tr = new TaskRunner(context, taskDescription)
    runningTasks.put(taskDescription.taskId, tr)
    threadPool.execute(tr)
  }
```
查看TaskRunner的run方法，首先会将Driver端发送过来的Task本身和它所依赖的Jar等文件反序列化
```scala
override def run(): Unit = {
      
      ...
        //反序列化
      try {
        // Must be set before updateDependencies() is called, in case fetching dependencies
        // requires access to properties contained within (e.g. for access control).
        Executor.taskDeserializationProps.set(taskDescription.properties)

        updateDependencies(taskDescription.addedFiles, taskDescription.addedJars)
        task = ser.deserialize[Task[Any]](
          taskDescription.serializedTask, Thread.currentThread.getContextClassLoader)
        task.localProperties = taskDescription.properties
        task.setTaskMemoryManager(taskMemoryManager)

    ...
    //运行Task
    val value = Utils.tryWithSafeFinally {
          val res = task.run(
            taskAttemptId = taskId,
            attemptNumber = taskDescription.attemptNumber,
            metricsSystem = env.metricsSystem)
          threwException = false
          res
        }

    ...
    //判断执行结果是否符合要求
    val serializedResult: ByteBuffer = {
        //如果结果大于１G，则丢弃这个结果
          if (maxResultSize > 0 && resultSize > maxResultSize) {
            logWarning(s"Finished $taskName (TID $taskId). Result is larger than maxResultSize " +
              s"(${Utils.bytesToString(resultSize)} > ${Utils.bytesToString(maxResultSize)}), " +
              s"dropping it.")
            ser.serialize(new IndirectTaskResult[Any](TaskResultBlockId(taskId), resultSize))
          } else if (resultSize > maxDirectResultSize) {
            val blockId = TaskResultBlockId(taskId)
            env.blockManager.putBytes(
              blockId,
              new ChunkedByteBuffer(serializedDirectResult.duplicate()),
              StorageLevel.MEMORY_AND_DISK_SER)
            logInfo(
              s"Finished $taskName (TID $taskId). $resultSize bytes result sent via BlockManager)")
            ser.serialize(new IndirectTaskResult[Any](blockId, resultSize))
          } else {
            logInfo(s"Finished $taskName (TID $taskId). $resultSize bytes result sent to driver")
            serializedDirectResult
          }
        }

        //将 Task 的执行状态汇报给 Driver
        setTaskFinishedAndClearInterruptStatus()
        execBackend.statusUpdate(taskId, TaskState.FINISHED, serializedResult)
```
&emsp;Task 的运行是调用 task.run 方法实现的，task.run 的方法内部会调用 Task.runTask　方法。Task是一个抽象类，它有两个实现子类 ShuffleMapTask 和 ResultTask。&nbsp;
&emsp;对于 ShuffleMapTask 而言，它的结果会写到 BlockManager 之中，最终返回给　DAGScheduler 的是一个 MapStatus 对象，该对象管理了 ShuffleMapTask 的运算结果储存到 BlockManager 里的相关储存信息，而不是计算结果，这些储存信息会成为下一阶段的 Task 需要获得的输入数据的依据。
```scala
//org.apache.spark.scheduler.ShuffleMapTask
override def runTask(context: TaskContext): MapStatus = {
    ...

    var writer: ShuffleWriter[Any, Any] = null
    try {
      val manager = SparkEnv.get.shuffleManager
      writer = manager.getWriter[Any, Any](dep.shuffleHandle, partitionId, context)
      writer.write(rdd.iterator(partition, context).asInstanceOf[Iterator[_ <: Product2[Any, Any]]])
      writer.stop(success = true).get
    } 

    ...
  }

```
对于 ResultTask 而言，返回结果是func函数计算结果。
```scala
//org.apache.spark.scheduler.ResultTask
verride def runTask(context: TaskContext): U = {
    ...

    func(context, rdd.iterator(partition, context))
  }
```

## 5.Driver的处理
&emsp;CoarseGrainedExecutorBackend 中有一个 driver 对象，TaskRunner 调用 execBackend.statusUpdate 函数将 Task 的执行状态汇报给 Driver。
```scala
//org.apache.spark.executor.CoarseGrainedExecutorBackend
override def statusUpdate(taskId: Long, state: TaskState, data: ByteBuffer) {
    val msg = StatusUpdate(executorId, taskId, state, data)
    driver match {
      case Some(driverRef) => driverRef.send(msg)
      case None => logWarning(s"Drop $msg because has not yet connected to driver")
    }
  }
```
最后由 TaskSchedulerImpl 的 statusUpdate 进行任务结束的一些处理工作。


## 6.附
### 6.1 TaskScheduler调度类型
&emsp;Spark 目前提供两种调度策略，一种是 FIFO (先进先出)，这是目前的默认模式；另外一种是 FAIR 模式，FAIR 模式可以通过配置作业权重等参数来决定 Job 的优先模式，TaskSchedulerImpl 的 initialize 方法中实现了对 rootPool 根调度池的初始化。
```scala
//org.apache.spark.scheduler.TaskSchedulerImpl
def initialize(backend: SchedulerBackend) {
    this.backend = backend
    schedulableBuilder = {
      schedulingMode match {
        case SchedulingMode.FIFO =>
          new FIFOSchedulableBuilder(rootPool)
        case SchedulingMode.FAIR =>
          new FairSchedulableBuilder(rootPool, conf)
        case _ =>
          throw new IllegalArgumentException(s"Unsupported $SCHEDULER_MODE_PROPERTY: " +
          s"$schedulingMode")
      }
    }
    schedulableBuilder.buildPools()
  }
```

### 6.2 RDD依赖实现
&emsp;RDD 的 action 算子会触发作业提交，transformation 算子则是返回一个新的 RDD，这个新的 RDD 就包含了对旧 RDD 的依赖。&nbsp;
&emsp;以 map 方法为例，map 是一个 transformation 算子，它的实现如下：
```scala
def map[U: ClassTag](f: T => U): RDD[U] = withScope {
    val cleanF = sc.clean(f)
    new MapPartitionsRDD[U, T](this, (context, pid, iter) => iter.map(cleanF))
  }
```
它返回了一个　MapPartitionsRDD 类，该类继承自 RDD 类：
```scala
private[spark] class MapPartitionsRDD[U: ClassTag, T: ClassTag](
    var prev: RDD[T],
    f: (TaskContext, Int, Iterator[T]) => Iterator[U],  // (TaskContext, partition index, iterator)
    preservesPartitioning: Boolean = false,
    isFromBarrier: Boolean = false,
    isOrderSensitive: Boolean = false)
  extends RDD[U](prev)
```
第一个参数是 prev，即父 RDD，我们看一下 RDD U(prev) 的实现：
```scala
/** Construct an RDD with just a one-to-one dependency on one parent */
  def this(@transient oneParent: RDD[_]) =
    this(oneParent.context, List(new OneToOneDependency(oneParent)))
  ==>
abstract class RDD[T: ClassTag](
    @transient private var _sc: SparkContext,
    @transient private var deps: Seq[Dependency[_]]
  ) {
      ...
    protected def getDependencies: Seq[Dependency[_]] = deps
      ...
  }
```
当调用 getDependencies 的时候，就返回构造时传入的 deps 参数，这样就实现了 RDD 的依赖链。

## 7.总结
&emsp;下面总结 **Standalone+Client** 模式下，从 action 算子触发 Job 提交到任务调度的完整运行流程：
1. (driver) action 算子触发调用 SparkContext.runJob， 经过几次回调之后会调用 DAGScheduler 的 runJob 方法，这时作业提交就进入了 DAGScheduler 的处理阶段；
2. (driver) DAGScheduler 根据RDD的依赖关系是否为 ShuffleDependency(宽依赖) 来划分Stage；
3. (driver) DAGScheduler将每个 Stage 中的 Tasks 包装成 TaskSet，调用 taskScheduler.submitTasks，将 TaskSet 交给 TaskScheduler 处理；
4. (driver) TaskScheduler 创建TaskSetManager 对TaskSet的生命周期进行管理；
5. (driver) TaskScheduler 中的 CoarseGrainedSchedulerBackend 对象调用 driverEndpoint.send(ReviveOffers) 请求分配计算资源并启动任务；
6. (driver) driverEndpoint 接收到请求之后调用 scheduler.resourceOffers(workOffers) 分配计算资源，然后向 Executor 发送 LaunchTask 启动任务的消息；
7. (Executor) Executor 执行 Task，并将执行结果发送给 Driver；



&nbsp;
&nbsp;
>本文作者：ZJP
版权声明：本文为博主原创文章，遵循 [CC 4.0 BY-SA](http://creativecommons.org/licenses/by-sa/4.0/) 版权协议，转载请附上原文出处链接和本声明。
本文链接：http://zhoujiapeng.top/Spark/spark-job/