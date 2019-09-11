---
title: Spark 源码阅读 之 部署模式
subtitle: Spark 部署模式
catalog: true
date: 2019-09-11 21:22:58
header-img: "/img/article_header/article_header.png"
tags:
- spark
- 大数据
- 源码阅读
categories:
- Spark
updateDate: 2019-09-11 21:22:58
---


# Spark 部署模式

> Spark版本：2.4.0


## 1. 概述
&emsp;Spark 支持多种集群运行模式：
- Local - 本地模式，也可以使用伪分布式模式
- [Standalone](http://spark.apache.org/docs/2.4.0/spark-standalone.html) - Spark自身提供的集群资源管理器
- [Mesos](http://spark.apache.org/docs/2.4.0/running-on-mesos.html) - 外部集群资源管理器
- [Yarn](http://spark.apache.org/docs/2.4.0/running-on-yarn.html) - 外部集群资源管理器
- [Kubernetes](http://spark.apache.org/docs/2.4.0/running-on-kubernetes.html) - 外部集群资源管理器

&emsp;下图表示 Spark 的基本工作流程架构图：
![cluster-overview](https://github.com/JP6907/Pic/blob/master/spark/cluster-overview.png?raw=true)
&emsp;其中的 Cluster Manager 集群管理器是可插拔的，也就是本文的介绍重点。


&emsp;我们可以在调用 spark-submit 脚本的时候通过参数指定集群的运行模式。spark-submit 脚本会调用 SparkSubmit 类，通过反射启动指定的 mainClass，mainClass 中会创建 SparkContext，在 SprkContext 调用 createTaskScheduler 就会创建我们需要的资源调度器。关于 SparkSubmit 的详细介绍可以参考文章[《Spark Application 提交》](http://zhoujiapeng.top/Spark/spark-application/#2sparksubmit-1)，这里不再多加赘述。具体集群运行模式的匹配是在 SparkContext 的 createTaskScheduler 方法中，我们看一下这个方法：
```scala
//SparkContext.createTaskScheduler
private def createTaskScheduler(
      sc: SparkContext,
      master: String,
      deployMode: String): (SchedulerBackend, TaskScheduler) = {
    import SparkMasterRegex._

    // When running locally, don't try to re-execute tasks on failure.
    val MAX_LOCAL_TASK_FAILURES = 1

    master match {
      case "local" =>
        val scheduler = new TaskSchedulerImpl(sc, MAX_LOCAL_TASK_FAILURES, isLocal = true)
        val backend = new LocalSchedulerBackend(sc.getConf, scheduler, 1)
        scheduler.initialize(backend)
        (backend, scheduler)

      case LOCAL_N_REGEX(threads) =>
        def localCpuCount: Int = Runtime.getRuntime.availableProcessors()
        // local[*] estimates the number of cores on the machine; local[N] uses exactly N threads.
        val threadCount = if (threads == "*") localCpuCount else threads.toInt
        if (threadCount <= 0) {
          throw new SparkException(s"Asked to run locally with $threadCount threads")
        }
        val scheduler = new TaskSchedulerImpl(sc, MAX_LOCAL_TASK_FAILURES, isLocal = true)
        val backend = new LocalSchedulerBackend(sc.getConf, scheduler, threadCount)
        scheduler.initialize(backend)
        (backend, scheduler)

      case LOCAL_N_FAILURES_REGEX(threads, maxFailures) =>
        def localCpuCount: Int = Runtime.getRuntime.availableProcessors()
        // local[*, M] means the number of cores on the computer with M failures
        // local[N, M] means exactly N threads with M failures
        val threadCount = if (threads == "*") localCpuCount else threads.toInt
        val scheduler = new TaskSchedulerImpl(sc, maxFailures.toInt, isLocal = true)
        val backend = new LocalSchedulerBackend(sc.getConf, scheduler, threadCount)
        scheduler.initialize(backend)
        (backend, scheduler)

      case SPARK_REGEX(sparkUrl) =>
        val scheduler = new TaskSchedulerImpl(sc)
        val masterUrls = sparkUrl.split(",").map("spark://" + _)
        val backend = new StandaloneSchedulerBackend(scheduler, sc, masterUrls)
        scheduler.initialize(backend)
        (backend, scheduler)

      case LOCAL_CLUSTER_REGEX(numSlaves, coresPerSlave, memoryPerSlave) =>
        // Check to make sure memory requested <= memoryPerSlave. Otherwise Spark will just hang.
        val memoryPerSlaveInt = memoryPerSlave.toInt
        if (sc.executorMemory > memoryPerSlaveInt) {
          throw new SparkException(
            "Asked to launch cluster with %d MB RAM / worker but requested %d MB/worker".format(
              memoryPerSlaveInt, sc.executorMemory))
        }

        val scheduler = new TaskSchedulerImpl(sc)
        val localCluster = new LocalSparkCluster(
          numSlaves.toInt, coresPerSlave.toInt, memoryPerSlaveInt, sc.conf)
        val masterUrls = localCluster.start()
        val backend = new StandaloneSchedulerBackend(scheduler, sc, masterUrls)
        scheduler.initialize(backend)
        backend.shutdownCallback = (backend: StandaloneSchedulerBackend) => {
          localCluster.stop()
        }
        (backend, scheduler)

      case masterUrl =>
        val cm = getClusterManager(masterUrl) match {
          case Some(clusterMgr) => clusterMgr
          case None => throw new SparkException("Could not parse Master URL: '" + master + "'")
        }
        try {
          val scheduler = cm.createTaskScheduler(sc, masterUrl)
          val backend = cm.createSchedulerBackend(sc, masterUrl, scheduler)
          cm.initialize(scheduler, backend)
          (backend, scheduler)
        } catch {
          case se: SparkException => throw se
          case NonFatal(e) =>
            throw new SparkException("External scheduler cannot be instantiated", e)
        }
    }
  }
```
&emsp;这里通过正则表达式来确定具体指定的集群部署模式，下面我们详细看一下其中的各种模式。

# 2. 各种部署模式

## 2.1 local 
&emsp;在 local 模式下，execuor、backend、master 都运行在同一个 JVM 进程中，executor 会创建多线程来运行 tasks(单节点多线程)。local 模式最大的好处就是很方便我们在单机上调试应用程序。在 SparkContext.createTaskScheduler 方法中我们可以看到，local 模式可以有3种参数指定的方式：

```scala
      case "local" =>
        val scheduler = new TaskSchedulerImpl(sc, MAX_LOCAL_TASK_FAILURES, isLocal = true)
        val backend = new LocalSchedulerBackend(sc.getConf, scheduler, 1)
        scheduler.initialize(backend)
        (backend, scheduler)

      //val LOCAL_N_REGEX = """local\[([0-9]+|\*)\]""".r
      case LOCAL_N_REGEX(threads) =>
        def localCpuCount: Int = Runtime.getRuntime.availableProcessors()
        // local[*] estimates the number of cores on the machine; local[N] uses exactly N threads.
        val threadCount = if (threads == "*") localCpuCount else threads.toInt
        if (threadCount <= 0) {
          throw new SparkException(s"Asked to run locally with $threadCount threads")
        }
        val scheduler = new TaskSchedulerImpl(sc, MAX_LOCAL_TASK_FAILURES, isLocal = true)
        val backend = new LocalSchedulerBackend(sc.getConf, scheduler, threadCount)
        scheduler.initialize(backend)
        (backend, scheduler)
      
      //val LOCAL_N_FAILURES_REGEX = """local\[([0-9]+|\*)\s*,\s*([0-9]+)\]""".r
      case LOCAL_N_FAILURES_REGEX(threads, maxFailures) =>
        def localCpuCount: Int = Runtime.getRuntime.availableProcessors()
        // local[*, M] means the number of cores on the computer with M failures
        // local[N, M] means exactly N threads with M failures
        val threadCount = if (threads == "*") localCpuCount else threads.toInt
        val scheduler = new TaskSchedulerImpl(sc, maxFailures.toInt, isLocal = true)
        val backend = new LocalSchedulerBackend(sc.getConf, scheduler, threadCount)
        scheduler.initialize(backend)
        (backend, scheduler)
```
1. "local" ： 这里的参数 1 指定核数，也就是说，最多只能有一个线程同时运行。
```scala
val backend = new LocalSchedulerBackend(sc.getConf, scheduler, 1)
```
2. LOCAL_N_REGEX（ "local[N]" 或 "local[\*]" ） : local[N] 指定了运行 N 个线程，local[\*] 创建的线程数和当前机器的核数一样。
```scala
val threadCount = if (threads == "*") localCpuCount else threads.toInt
val backend = new LocalSchedulerBackend(sc.getConf, scheduler, threadCount)
```
3. LOCAL_N_FAILURES_REGEX（ "local[\*, M]" 或 "local[N, M]" ）：local[N, M] 表示运行 N 个线程，最多允许有 M 个线程失败，local[\*, M] 创建的线程数和当前机器的核数一样，最多允许有 M 个线程失败。
```scala
val threadCount = if (threads == "*") localCpuCount else threads.toInt
val backend = new LocalSchedulerBackend(sc.getConf, scheduler, threadCount)
```

&emsp;这三种方式都是创建了 LocalSchedulerBackend，在[《Spark 作业和调度》](http://zhoujiapeng.top/Spark/spark-job/#3taskschedulertask)中讲到，当提交 TaskSet 给 TaskScheduler 的时候会调用 backend.reviveOffers() 进行计算资源的分配并启动 Task。我们看一下 LocalSchedulerBackend.reviveOffers() 方法：
```scala
//LocalSchedulerBackend
def reviveOffers() {
    val offers = IndexedSeq(new WorkerOffer(localExecutorId, localExecutorHostname, freeCores,
      Some(rpcEnv.address.hostPort)))
    for (task <- scheduler.resourceOffers(offers).flatten) {
      freeCores -= scheduler.CPUS_PER_TASK

      //启动task
      executor.launchTask(executorBackend, task)
    }
  }

private val executor = new Executor(
    localExecutorId, localExecutorHostname, SparkEnv.get, userClassPath, isLocal = true)
```
&emsp;可以看到，所有的 task 都是由同一个 executor（意味着单进程） 调用 launchTask 来启动的。再看一下 executor.launchTask 方法：
```scala
def launchTask(context: ExecutorBackend, taskDescription: TaskDescription): Unit = {
    val tr = new TaskRunner(context, taskDescription)
    runningTasks.put(taskDescription.taskId, tr)
    threadPool.execute(tr)
  }
```
&emsp;这里是将 task 包装成 TaskRunner，然后放进线程池里面去执行。到这里我们可以知道，

## 2.2 local-cluster
&emsp;local-cluster，本地集群模式，也就是伪分布式模式。Driver、Master 和 Worker 在同一个 JVM 进程，可以存在多个 Worker，每个 Worker 会有多个 Executor，但这些 Executor 都独自存在一个 JVM 进程中。
```scala
//val LOCAL_CLUSTER_REGEX = """local-cluster\[\s*([0-9]+)\s*,\s*([0-9]+)\s*,\s*([0-9]+)\s*]""".r
case LOCAL_CLUSTER_REGEX(numSlaves, coresPerSlave, memoryPerSlave) =>
        // Check to make sure memory requested <= memoryPerSlave. Otherwise Spark will just hang.
        val memoryPerSlaveInt = memoryPerSlave.toInt
        if (sc.executorMemory > memoryPerSlaveInt) {
          throw new SparkException(
            "Asked to launch cluster with %d MB RAM / worker but requested %d MB/worker".format(
              memoryPerSlaveInt, sc.executorMemory))
        }

        val scheduler = new TaskSchedulerImpl(sc)
        val localCluster = new LocalSparkCluster(
          numSlaves.toInt, coresPerSlave.toInt, memoryPerSlaveInt, sc.conf)
        val masterUrls = localCluster.start()
        val backend = new StandaloneSchedulerBackend(scheduler, sc, masterUrls)
        scheduler.initialize(backend)
        backend.shutdownCallback = (backend: StandaloneSchedulerBackend) => {
          localCluster.stop()
        }
        (backend, scheduler)
```
&emsp;这里先是创建了一个 LocalSparkCluster，构建一个本地 Spark 集群环境。接着创建一个 StandaloneSchedulerBackend，注意这里不是 LocalSchedulerBackend。集群环境是在本地运行还是多节点环境对于 StandaloneSchedulerBackend 是透明的。StandaloneSchedulerBackend 只需要知道 Master 节点的地址就可以了，具体的资源调度由 Master 负责就可以。Standalone 模式同样是创建 StandaloneSchedulerBackend。这里将 StandaloneSchedulerBackend 放到 Standalone 模式一起介绍。


## 2.3 standalone
&emsp;local 模式只有 Driver 和 Executor，且都在一个 JVM 进程中；loca-cluster 模式下的 Driver、Master、Worker 都位于一个 JVM 进程中。所以 local 模式和 local-cluster 模式便于开发、测试，也便于源码阅读和调试，但是不适合在生产环境使用。不同于这两种模式，Standalone 部署模式有下面的特点：
- Driver 是单独的进程、可以存在于集群中，也可存在于集群之外，也对 Spark Application 的执行进行驱动；
- Master 是单独的进程，甚至应该在单独的机器节点上。Master 可以有多个，但同时最多只有一个处于激活状态；
- Worker 是单独的进程，推荐在单独的机器节点上部署。
```scala
//val SPARK_REGEX = """spark://(.*)""".r
case SPARK_REGEX(sparkUrl) =>
        val scheduler = new TaskSchedulerImpl(sc)
        val masterUrls = sparkUrl.split(",").map("spark://" + _)
        val backend = new StandaloneSchedulerBackend(scheduler, sc, masterUrls)
        scheduler.initialize(backend)
        (backend, scheduler)
```
&emsp;不同于 local-cluster 模式，Standalone 模式的 masterUrls 是由参数传入的，然后同样是创建一个 StandaloneSchedulerBackend。我们同样看一下 backend.reviveOffers() 方法。StandaloneSchedulerBackend 的 reviveOffers 是由父类 CoarseGrainedSchedulerBackend 实现的：
```scala
//CoarseGrainedSchedulerBackend
override def reviveOffers() {
    driverEndpoint.send(ReviveOffers)
  }

override def receive: PartialFunction[Any, Unit] = {
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
```
&emsp;这里的 executorDataMap 是一个 HashMap，储存了所有的 Executors。makeOffers() 首先是 调用 scheduler.resourceOffers 进行计算资源的分配，然后调用 launchTasks 去启动对应的 task。我们看一下 launchTasks 方法：
```scala
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

          //获取 task 的执行位置
          val executorData = executorDataMap(task.executorId)
          executorData.freeCores -= scheduler.CPUS_PER_TASK

          logDebug(s"Launching task ${task.taskId} on executor id: ${task.executorId} hostname: " +
            s"${executorData.executorHost}.")

          //启动 task
          executorData.executorEndpoint.send(LaunchTask(new SerializableBuffer(serializedTask)))
        }
      }
    }   
```
&emsp;前面已经为所有的 task 分配好了计算资源，对于哪一个 task 会分配到哪一个 executor 上去执行已经是明确的了，并储存在 task.executorId 中。这里遍历每一个 task，然后调用直接在对应的 executor 上启动该 task。到这里就实现了将 不同的 task 分配到不同的 executor 进行中去运行。

## 2.4 Yarn
&emsp;createTaskScheduler 方法中最后一种情况对应于所有的外部集群部署模式（yarn、k8s、Mesos）：
```scala
case masterUrl =>
        val cm = getClusterManager(masterUrl) match {
          case Some(clusterMgr) => clusterMgr
          case None => throw new SparkException("Could not parse Master URL: '" + master + "'")
        }
        try {
          val scheduler = cm.createTaskScheduler(sc, masterUrl)
          val backend = cm.createSchedulerBackend(sc, masterUrl, scheduler)
          cm.initialize(scheduler, backend)
          (backend, scheduler)
        } catch {
          case se: SparkException => throw se
          case NonFatal(e) =>
            throw new SparkException("External scheduler cannot be instantiated", e)
        }
```
&emsp;getClusterManager 方法返回的是一个 ExternalClusterManager 对象，即外部集群管理器。
```scala
private def getClusterManager(url: String): Option[ExternalClusterManager] = {
    val loader = Utils.getContextOrSparkClassLoader
    val serviceLoaders =
      ServiceLoader.load(classOf[ExternalClusterManager], loader).asScala.filter(_.canCreate(url))
    if (serviceLoaders.size > 1) {
      throw new SparkException(
        s"Multiple external cluster managers registered for the url $url: $serviceLoaders")
    }
    serviceLoaders.headOption
  }
```
&emsp;TaskScheduler 和 SchedulerBackend 是通过 ExternalClusterManager 的 createTaskScheduler 方法和 createSchedulerBackend 获得的。ExternalClusterManager 是一个特质，Yarn 模式的实现类是 YarnClusterManager。Yarn 模式根据 driver 的运行位置不同可以分为 cluster 模式和 client 模式：
```scala
//YarnClusterManager
override def createTaskScheduler(sc: SparkContext, masterURL: String): TaskScheduler = {
    sc.deployMode match {
      case "cluster" => new YarnClusterScheduler(sc)
      case "client" => new YarnScheduler(sc)
      case _ => throw new SparkException(s"Unknown deploy mode '${sc.deployMode}' for Yarn")
    }
  }

  override def createSchedulerBackend(sc: SparkContext,
      masterURL: String,
      scheduler: TaskScheduler): SchedulerBackend = {
    sc.deployMode match {
      case "cluster" =>
        new YarnClusterSchedulerBackend(scheduler.asInstanceOf[TaskSchedulerImpl], sc)
      case "client" =>
        new YarnClientSchedulerBackend(scheduler.asInstanceOf[TaskSchedulerImpl], sc)
      case  _ =>
        throw new SparkException(s"Unknown deploy mode '${sc.deployMode}' for Yarn")
    }
  }
```

&emsp;我们再看一下 SparkSubmit 的 prepareSubmitEnvironment 方法
```scala
private[deploy] def prepareSubmitEnvironment(
      args: SparkSubmitArguments,
      conf: Option[HadoopConfiguration] = None)
      : (Seq[String], Seq[String], SparkConf, String) = {
    ...

    // In yarn-cluster mode, use yarn.Client as a wrapper around the user class
    if (isYarnCluster) {
      childMainClass = YARN_CLUSTER_SUBMIT_CLASS  //"org.apache.spark.deploy.yarn.YarnClusterApplication"
      if (args.isPython) {
        childArgs += ("--primary-py-file", args.primaryResource)
        childArgs += ("--class", "org.apache.spark.deploy.PythonRunner")
      } else if (args.isR) {
        val mainFile = new Path(args.primaryResource).getName
        childArgs += ("--primary-r-file", mainFile)
        childArgs += ("--class", "org.apache.spark.deploy.RRunner")
      } else {
        if (args.primaryResource != SparkLauncher.NO_RESOURCE) {
          childArgs += ("--jar", args.primaryResource)
        }
        childArgs += ("--class", args.mainClass)
      }
      if (args.childArgs != null) {
        args.childArgs.foreach { arg => childArgs += ("--arg", arg) }
      }
    }

    ...
}
```
&emsp;可以看到，childMainClass 被设置成了 YarnClusterApplication 类，传入参数则被添加到 childArgs 中，因此 Yarn-cluster 不同于其它方式通过反射(SparkSubmit.runMain)直接运行参数指定的类，而是先创建 YarnClusterApplication，再通过它来提交 Application。
&emsp;Yarn-client 模式和 local 模式一样，都是通过反射直接启动运行参数(SparkSubmit.runMain)指定的类。
```scala
//SparkSubmit.prepareSubmitEnvironment
// yarn-client 模式
if (deployMode == CLIENT) {
      childMainClass = args.mainClass
      if (localPrimaryResource != null && isUserJar(localPrimaryResource)) {
        childClasspath += localPrimaryResource
      }
      if (localJars != null) { childClasspath ++= localJars.split(",") }
    }
```

> 未完待续



&nbsp;
&nbsp;
>本文作者：ZJP
版权声明：本文为博主原创文章，遵循 [CC 4.0 BY-SA](http://creativecommons.org/licenses/by-sa/4.0/) 版权协议，转载请附上原文出处链接和本声明。
本文链接：http://zhoujiapeng.top/Spark/spark-deploy-mode/