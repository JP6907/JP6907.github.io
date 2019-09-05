---
title: Spark源码阅读 之 Spark Application 的提交
catalog: true
date: 2019-08-31 15:08:21
subtitle: Spark Application 的提交
header-img: "/img/article_header/article_header.png"
tags:
- spark
- 大数据
- 源码阅读
categories:
  - Spark
---

# Spark Application 提交

> Spark版本：2.4.0


&emsp;下面分析过程为Spark Standalone运行模式，Spark Standalone是一种典型的Master-Slave架构，在这种模式下，主要包括三个组件：Master、Worker、Driver，这里的Driver我们以运行在客户端的Client模式。

## 1.启动脚本
&emsp;首先我们会使用spark的sbin目录下"start-all"脚本来启动集群：
```shell
if [ -z "${SPARK_HOME}" ]; then
  export SPARK_HOME="$(cd "`dirname "$0"`"/..; pwd)"
fi

# Load the Spark configuration
. "${SPARK_HOME}/sbin/spark-config.sh"

# Start Master
"${SPARK_HOME}/sbin"/start-master.sh

# Start Workers
"${SPARK_HOME}/sbin"/start-slaves.sh
```
可以看到分别调用了start-master.sh脚本和start-slaves.sh脚本，查看start-master.sh脚本：
```shell
# Starts the master on the machine this script is executed on.

if [ -z "${SPARK_HOME}" ]; then
  export SPARK_HOME="$(cd "`dirname "$0"`"/..; pwd)"
fi

# NOTE: This exact class name is matched downstream by SparkSubmit.
# Any changes need to be reflected there.
CLASS="org.apache.spark.deploy.master.Master"

# ......

if [ "$SPARK_MASTER_WEBUI_PORT" = "" ]; then
  SPARK_MASTER_WEBUI_PORT=8080
fi

"${SPARK_HOME}/sbin"/spark-daemon.sh start $CLASS 1 \
  --host $SPARK_MASTER_HOST --port $SPARK_MASTER_PORT --webui-port $SPARK_MASTER_WEBUI_PORT \
  $ORIGINAL_ARGS

```
该脚本启动了**org.apache.spark.deploy.master.Master**类，同样，查看start-slaves.sh脚本，可以看到调用了start-slave.sh脚本，而start-slave.sh脚本启动了**org.apache.spark.deploy.worker.Worker**类。
```shell
if [ -z "${SPARK_HOME}" ]; then
  export SPARK_HOME="$(cd "`dirname "$0"`"/..; pwd)"
fi

# NOTE: This exact class name is matched downstream by SparkSubmit.
# Any changes need to be reflected there.
CLASS="org.apache.spark.deploy.worker.Worker"

# ...

  "${SPARK_HOME}/sbin"/spark-daemon.sh start $CLASS $WORKER_NUM \
     --webui-port "$WEBUI_PORT" $PORT_FLAG $PORT_NUM $MASTER "$@"
}

# ...
```
这时，Spark集群的Master节点和Worker已经全部启动，Worker会向Master完成注册，并定时向Master发送心跳，使得Master节点可以知道该Worker节点属于活跃状态。

## 2.SparkSubmit
&emsp;打开 spark-submit 脚本，可以看到这个脚本最终启动了 org.apache.spark.deploy.SparkSubmit 这个类。
```shell
if [ -z "${SPARK_HOME}" ]; then
  source "$(dirname "$0")"/find-spark-home
fi

# disable randomized hash for string in Python 3.3+
export PYTHONHASHSEED=0

exec "${SPARK_HOME}"/bin/spark-class org.apache.spark.deploy.SparkSubmit "$@"
```
&emsp; Object SparkSubmit类继承自 CommandLineUtils ，名字浅显易懂，就是命令行解析工具类，它负责解析输入的命令和参数，并启动一个子类执行具体的业务逻辑。
```scala
override def main(args: Array[String]): Unit = {
    val submit = new SparkSubmit() {
      self =>

      override protected def parseArguments(args: Array[String]): SparkSubmitArguments = {
        new SparkSubmitArguments(args) {
          override protected def logInfo(msg: => String): Unit = self.logInfo(msg)

          override protected def logWarning(msg: => String): Unit = self.logWarning(msg)
        }
      }

      override protected def logInfo(msg: => String): Unit = printMessage(msg)

      override protected def logWarning(msg: => String): Unit = printMessage(s"Warning: $msg")

      override def doSubmit(args: Array[String]): Unit = {
        try {
          super.doSubmit(args)
        } catch {
          case e: SparkUserAppException =>
            exitFn(e.exitCode)
        }
      }

    }
    submit.doSubmit(args)
  }
```

SparkSubmit 的面函数里面重写了解析参数，打印日志等函数，最后执行 submit.doSubmit(args)，所以 doSubmit 函数是主要的入口：
```scala
def doSubmit(args: Array[String]): Unit = {
    // Initialize logging if it hasn't been done yet. Keep track of whether logging needs to
    // be reset before the application starts.
    val uninitLog = initializeLogIfNecessary(true, silent = true)

    val appArgs = parseArguments(args)
    if (appArgs.verbose) {
      logInfo(appArgs.toString)
    }
    appArgs.action match {
      case SparkSubmitAction.SUBMIT => submit(appArgs, uninitLog)
      case SparkSubmitAction.KILL => kill(appArgs)
      case SparkSubmitAction.REQUEST_STATUS => requestStatus(appArgs)
      case SparkSubmitAction.PRINT_VERSION => printVersion()
    }
  }
```
doSubmit 首先调用 parseArguments 解析输入的参数，然后根据具体的参数再调用相应的处理函数，我们具体看一下submit的逻辑。

submit 函数负责根据输入参数提交 Application，执行分为两步：      

1. 准备环境，设置classpath、系统参数、程序参数等：
```scala
//org.apache.spark.deploy.SparkSubmit.submit()
val (childArgs, childClasspath, sparkConf, childMainClass) = prepareSubmitEnvironment(args)
```
2. 运行，通过反射机制启动 Spark Application 程序 mainClass 中的 main 方法：
```scala
runMain(childArgs, childClasspath, sparkConf, childMainClass, args.verbose)
```

```scala
private def runMain(
      childArgs: Seq[String],
      childClasspath: Seq[String],
      sparkConf: SparkConf,
      childMainClass: String,
      verbose: Boolean): Unit = {
    
    /// ...

    var mainClass: Class[_] = null

    try {
      mainClass = Utils.classForName(childMainClass)
    } catch {
      /// ...
    }

    val app: SparkApplication = if (classOf[SparkApplication].isAssignableFrom(mainClass)) {
      mainClass.newInstance().asInstanceOf[SparkApplication]
    } else {
      // SPARK-4170
      if (classOf[scala.App].isAssignableFrom(mainClass)) {
        logWarning("Subclasses of scala.App may not work correctly. Use a main() method instead.")
      }
      new JavaMainApplication(mainClass)
    }

    /// ...
  }
```
上面启动的 Spark Application 程序（用户程序）会创建 SparkContext，每个 Application 都会对应着唯一一个 SparkContext ，main方法里面会完成 SparkContext的初始化。SparkContext 是 Spark 应用创建时的上下文对象，是一个重要的入口类，在内部会进行一系列重要的操作，其中最重要的是创建 TaskScheduler 和 DAGScheduler实例。

## 3.SparkContext
&emsp;SparkoContext 是 Spark 应用创建时的上下文对象，是一个重要的入口类，在内部会进行一系列重要的操作，其中最重要的是创建 TaskScheduler 和 DAGScheduler实例。

### 3.1 SparkConf
&emsp;SparkConf负责管理Spark应用中的属性设置，它通过一个HashMap容器来管理key/alue类型的属性。
```scala
class SparkConf(loadDefaults: Boolean) extends Cloneable with Logging with Serializable {

  import SparkConf._

  /** Create a SparkConf that loads defaults from system properties and the classpath */
  def this() = this(true)

  private val settings = new ConcurrentHashMap[String, String]()

  /// ...
}
```
&emsp;注意，在SparkContext中会对传入的SparkConf克隆并验证，即一旦SparkConf传给SparkContext，就不能在修改了，Spark不支持运行时修改配置。
```scala
//org.apache.spark.SparkContext
try {
    _conf = config.clone()
    _conf.validateSettings()

    ...
```

### 3.2 LiveListenerBus
&emsp;这里使用典型的**观察者模式**，SparkListener 向LiveListenerBus类注册不同类型的 SparkListenerEvent事件，LiveListenerBus会遍历它的所有监听者SparkListener， 然后使用对应事件接口进行相应。
```scala
//org.apache.spark.SparkContext
_listenerBus = new LiveListenerBus(_conf)

...

setupAndStartListenerBus()
postEnvironmentUpdate()
postApplicationStart()
```
当task scheduler准备好了之后，就会通过postEnvironmentUpdate通知listenerBus
```scala
/** Post the environment update event once the task scheduler is ready */
  private def postEnvironmentUpdate() {
    if (taskScheduler != null) {
      val schedulingMode = getSchedulingMode.toString
      val addedJarPaths = addedJars.keys.toSeq
      val addedFilePaths = addedFiles.keys.toSeq
      val environmentDetails = SparkEnv.environmentDetails(conf, schedulingMode, addedJarPaths,
        addedFilePaths)
      val environmentUpdate = SparkListenerEnvironmentUpdate(environmentDetails)
      listenerBus.post(environmentUpdate) //观察者模式
    }
  }
```

### 3.3 SparkEnv
&emsp;SparkEnv是运行环境，封装了所有Spark运行时的环境对象，如Serializer、ShuffleManager、BroadcastManager、BlockManager、MemoryManager等。
```scala
// Create the Spark execution environment (cache, map output tracker, etc)
    _env = createSparkEnv(_conf, isLocal, listenerBus)
    SparkEnv.set(_env)
```

### 3.4 SparkUI
&emsp;通过SparkUI可以观察Spark集群的运行情况和Spark Application的运行情况。
```scala
_ui =
      if (conf.getBoolean("spark.ui.enabled", true)) {
        Some(SparkUI.create(Some(this), _statusStore, _conf, _env.securityManager, appName, "",
          startTime))
      } else {
        // For tests, do not enable the UI
        None
      }
    // Bind the UI before starting the task scheduler to communicate
    // the bound port to the cluster manager properly
    _ui.foreach(_.bind())
```

### 3.5 EventLoggingListener
&emsp;EventLoggingListener默认是关闭的，可以通过spark.eventLog.enabled配置开启，它的主要功能是以json格式记录发生的事件。
```scala
_eventLogger =
      if (isEventLogEnabled) {
        val logger =
          new EventLoggingListener(_applicationId, _applicationAttemptId, _eventLogDir.get,
            _conf, _hadoopConfiguration)
        logger.start()
        listenerBus.addToEventLogQueue(logger)
        Some(logger)
      } else {
        None
      }
```

### 3.6 TaskScheduler
&emsp;SparkContext会创建 TaskScheduler 和 DAGScheduler 调度器：
```scala
// Create and start the scheduler
    val (sched, ts) = SparkContext.createTaskScheduler(this, master, deployMode)
    _schedulerBackend = sched
    _taskScheduler = ts
    _dagScheduler = new DAGScheduler(this)
```
TaskScheduler 通过不同的 SchedulerBackend 来调度和管理任务，它实现了FIFO调度和FAIR调度。SparkContext.createTaskScheduler 会根据不同的部署模式选择不同的 SchedulerBackend
```scala
/**
   * Create a task scheduler based on a given master URL.
   * Return a 2-tuple of the scheduler backend and the task scheduler.
   */
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

    ...

      case SPARK_REGEX(sparkUrl) =>
        val scheduler = new TaskSchedulerImpl(sc)
        val masterUrls = sparkUrl.split(",").map("spark://" + _)
        val backend = new StandaloneSchedulerBackend(scheduler, sc, masterUrls)
        scheduler.initialize(backend)
        (backend, scheduler)

    ...
  }
```

### 3.7 StandaloneAppClient
&emsp; standalone 模式对应的是 SPARK_REGEX，这里的 SchedulerBackend 实现的是 StandaloneSchedulerBackend，它是CoarseGrainedSchedulerBackend 的子类，CoarseGrainedSchedulerBackend 实现了 SchedulerBackend 特质。StandaloneSchedulerBackend 会创建并启动 StandaloneAppClient，client 启动之后会向 Master注册 Application。
```scala
//org.apache.spark.scheduler.cluster.StandaloneSchedulerBackend
client = new StandaloneAppClient(sc.env.rpcEnv, masters, appDesc, this, conf)
client.start()

//StandaloneAppClient.scala
override def onStart(): Unit = {
      try {
        registerWithMaster(1)
      } catch {
        case e: Exception =>
          logWarning("Failed to connect to master", e)
          markDisconnected()
          stop()
      }
    }
```
&emsp;client 会尝试向所有的Master（可能有多个）注册，直到成功连接上其中一个 Master。发送注册消息的过程中，如果超过20s没有接收到注册成功的消息，那么会重新注册，如果重试超过3次仍未成功，那么本次提交就以失败结束。
```scala
//StandaloneAppClient.scala
private def registerWithMaster(nthRetry: Int) {
      registerMasterFutures.set(tryRegisterAllMasters())
      registrationRetryTimer.set(registrationRetryThread.schedule(new Runnable {
        override def run(): Unit = {
          if (registered.get) {
            registerMasterFutures.get.foreach(_.cancel(true))
            registerMasterThreadPool.shutdownNow()
          } else if (nthRetry >= REGISTRATION_RETRIES) {
            markDead("All masters are unresponsive! Giving up.")
          } else {
            registerMasterFutures.get.foreach(_.cancel(true))
            registerWithMaster(nthRetry + 1)
          }
        }
      }, REGISTRATION_TIMEOUT_SECONDS, TimeUnit.SECONDS))
    }

private def tryRegisterAllMasters(): Array[JFuture[_]] = {
      for (masterAddress <- masterRpcAddresses) yield {
        registerMasterThreadPool.submit(new Runnable {
          override def run(): Unit = try {
            if (registered.get) {
              return
            }
            logInfo("Connecting to master " + masterAddress.toSparkURL + "...")
            val masterRef = rpcEnv.setupEndpointRef(masterAddress, Master.ENDPOINT_NAME)
            masterRef.send(RegisterApplication(appDescription, self))
          } catch {
            case ie: InterruptedException => // Cancelled
            case NonFatal(e) => logWarning(s"Failed to connect to master $masterAddress", e)
          }
        })
      }
    }
```
tryRegisterAllMasters 函数通过 rpcEnv 向 Master 发送一个 RegisterApplication 注册的消息，StandaloneAppClient 的 receive 方法会接收 Master 的返回消息：
```scala
override def receive: PartialFunction[Any, Unit] = {
      case RegisteredApplication(appId_, masterRef) =>
        // FIXME How to handle the following cases?
        // 1. A master receives multiple registrations and sends back multiple
        // RegisteredApplications due to an unstable network.
        // 2. Receive multiple RegisteredApplication from different masters because the master is
        // changing.
        appId.set(appId_)
        registered.set(true)
        master = Some(masterRef)
        listener.connected(appId.get)

    ...
```

### 3.8 Master 和 Worker
Master 接收到 AppClient 的 RegisterApplication 消息后，会创建并注册对应的 Applicaton。

```scala
//org.apache.spark.deploy.master
override def receive: PartialFunction[Any, Unit] = {
    case ElectedLeader =>
      val (storedApps, storedDrivers, storedWorkers) = persistenceEngine.readPersistedData(rpcEnv)
      
      ...

    case RegisterApplication(description, driver) =>
      // TODO Prevent repeated registrations from some driver
      if (state == RecoveryState.STANDBY) {
        // ignore, don't send response
      } else {
        logInfo("Registering app " + description.name)
        val app = createApplication(description, driver)
        registerApplication(app)
        logInfo("Registered app " + description.name + " with ID " + app.id)
        persistenceEngine.addApplication(app)
        driver.send(RegisteredApplication(app.id, self))
        schedule()
      }
     
     ...
```
在registerApplication函数里面，会通过applicationMetricsSystem度量系统**为该Application注册资源**。Spark基于Metric构建了自己的度量系统，提供完整的系统监控功能，如可测性、性能优化、运维评估、数据统计等。
```scala
applicationMetricsSystem.registerSource(app.appSource)
```
在注册完成之后，会调用schedule函数，为 **worker** 进行资源的调度分配：
```scala
private def schedule(): Unit = {
    if (state != RecoveryState.ALIVE) {
      return
    }
    // Drivers take strict precedence over executors
    val shuffledAliveWorkers = Random.shuffle(workers.toSeq.filter(_.state == WorkerState.ALIVE))
    val numWorkersAlive = shuffledAliveWorkers.size
    var curPos = 0
    for (driver <- waitingDrivers.toList) { // iterate over a copy of waitingDrivers
      // We assign workers to each waiting driver in a round-robin fashion. For each driver, we
      // start from the last worker that was assigned a driver, and continue onwards until we have
      // explored all alive workers.
      var launched = false
      var numWorkersVisited = 0
      while (numWorkersVisited < numWorkersAlive && !launched) {
        val worker = shuffledAliveWorkers(curPos)
        numWorkersVisited += 1
        if (worker.memoryFree >= driver.desc.mem && worker.coresFree >= driver.desc.cores) {
          launchDriver(worker, driver)
          waitingDrivers -= driver
          launched = true
        }
        curPos = (curPos + 1) % numWorkersAlive
      }
    }
    startExecutorsOnWorkers()
  }
```
scheduler 会首先打散所有 worker，然后再选出符合条件的 worker 节点，使得一个 Application 尽可能多得分配到不同的节点，最后通过startExecutorsOnWorkers 在相应的worder分配资源并启动 Executor 进程：
```scala
private def startExecutorsOnWorkers(): Unit = {
    // Right now this is a very simple FIFO scheduler. We keep trying to fit in the first app
    // in the queue, then the second app, etc.
    for (app <- waitingApps) {
      val coresPerExecutor = app.desc.coresPerExecutor.getOrElse(1)
      // If the cores left is less than the coresPerExecutor,the cores left will not be allocated
      if (app.coresLeft >= coresPerExecutor) {
        // Filter out workers that don't have enough resources to launch an executor
        val usableWorkers = workers.toArray.filter(_.state == WorkerState.ALIVE)
          .filter(worker => worker.memoryFree >= app.desc.memoryPerExecutorMB &&
            worker.coresFree >= coresPerExecutor)
          .sortBy(_.coresFree).reverse
        val assignedCores = scheduleExecutorsOnWorkers(app, usableWorkers, spreadOutApps)

        // Now that we've decided how many cores to allocate on each worker, let's allocate them
        for (pos <- 0 until usableWorkers.length if assignedCores(pos) > 0) {
          allocateWorkerResourceToExecutors(
            app, assignedCores(pos), app.desc.coresPerExecutor, usableWorkers(pos))
        }
      }
    }
  }
```
allocateWorkerResourceToExecutors 函数会发送一个 LaunchExecutor消息给对应 Worker，Worker 根据 Master 的资源分配结果类创建 Executor 进程。

```scala
//org.apache.spark.deploy.worker
override def receive: PartialFunction[Any, Unit] = synchronized {
  ...

  case LaunchExecutor(masterUrl, appId, execId, appDesc, cores_, memory_) =>
      if (masterUrl != activeMasterUrl) {
        logWarning("Invalid Master (" + masterUrl + ") attempted to launch executor.")
      } else {
        try {
          logInfo("Asked to launch executor %s/%d for %s".format(appId, execId, appDesc.name))

          // Create the executor's working directory
          val executorDir = new File(workDir, appId + "/" + execId)
          if (!executorDir.mkdirs()) {
            throw new IOException("Failed to create directory " + executorDir)
          }

          // Create local dirs for the executor. These are passed to the executor via the
          // SPARK_EXECUTOR_DIRS environment variable, and deleted by the Worker when the
          // application finishes.
          val appLocalDirs = appDirectories.getOrElse(appId, {
            val localRootDirs = Utils.getOrCreateLocalRootDirs(conf)
            val dirs = localRootDirs.flatMap { dir =>
              try {
                val appDir = Utils.createDirectory(dir, namePrefix = "executor")
                Utils.chmod700(appDir)
                Some(appDir.getAbsolutePath())
              } catch {
                case e: IOException =>
                  logWarning(s"${e.getMessage}. Ignoring this directory.")
                  None
              }
            }.toSeq
            if (dirs.isEmpty) {
              throw new IOException("No subfolder can be created in " +
                s"${localRootDirs.mkString(",")}.")
            }
            dirs
          })
          appDirectories(appId) = appLocalDirs
          val manager = new ExecutorRunner(
            appId,
            execId,
            appDesc.copy(command = Worker.maybeUpdateSSLSettings(appDesc.command, conf)),
            cores_,
            memory_,
            self,
            workerId,
            host,
            webUi.boundPort,
            publicAddress,
            sparkHome,
            executorDir,
            workerUri,
            conf,
            appLocalDirs, ExecutorState.RUNNING)
          executors(appId + "/" + execId) = manager
          manager.start()
          coresUsed += cores_
          memoryUsed += memory_
          sendToMaster(ExecutorStateChanged(appId, execId, manager.state, None, None))
        } catch {
          case e: Exception =>
            logError(s"Failed to launch executor $appId/$execId for ${appDesc.name}.", e)
            if (executors.contains(appId + "/" + execId)) {
              executors(appId + "/" + execId).kill()
              executors -= appId + "/" + execId
            }
            sendToMaster(ExecutorStateChanged(appId, execId, ExecutorState.FAILED,
              Some(e.toString), None))
        }
      }

  ...

}
```
Worker 接收到 LaunchExecutor 消息后会创建相应的工作目录，并启动 ExecutorRunner。Worker 会记录本身资源的使用情况，包括CPU、内存等，但这个统计只是为了 WebUI 的展现。Master 本身会记录 Worker 的资源使用情况，无需 Worker 汇报。**Worker 和 Master 之间传送的心跳的目的仅仅是汇报存活状态，不会携带其它信息**。

ExecutorRunner 会启动一个线程来执行任务:
```scala
private[worker] def start() {
    workerThread = new Thread("ExecutorRunner for " + fullId) {
      override def run() { fetchAndRunExecutor() }
    }
    workerThread.start()
    // Shutdown hook that kills actors on shutdown.
    shutdownHook = ShutdownHookManager.addShutdownHook { () =>
      // It's possible that we arrive here before calling `fetchAndRunExecutor`, then `state` will
      // be `ExecutorState.RUNNING`. In this case, we should set `state` to `FAILED`.
      if (state == ExecutorState.RUNNING) {
        state = ExecutorState.FAILED
      }
      killProcess(Some("Worker shutting down")) }
  }
```
主要的线程逻辑在 fetchAndRunExecutor 函数中，里面首先根据ApplicationDescription设置一些环境参数、创建工作目录，最后再执行相应的程序。

## 4.总结
&emsp;现总结 **Standalone+Client** 模式下，Spark Application 提交后整个系统的运行流程：
1. (client/driver端) 使用 spark-submit 脚本提交 Application；
2. (client/driver端) spark-submit 脚本执行 org.apache.spark.deploy.SparkSubmit 类的主函数入口，如果脚本执行submit命令，则 SparkSubmit 首先解析传入参数，然后使用 Java 的反射机制运行参数中指定的 mainClass 主类的main函数，即用户程序中的 main 函数；
3. (client/driver端) main函数中会创建 SparkContext，它程序的重要入口，包含系统运行的上下文信息，SparkContext 会根据传入的 SparkConf创建一系列组建，包括 LiveListenerBus、SparkEnv、RpcEndpointRef、TaskScheduler、DAGScheduler等；
4. (client/driver端) SparkContext.createTaskScheduler 会根据部署模式创建对应的 SchedulerBackend，Standalone 模式下是 StandaloneSchedulerBackend，StandaloneSchedulerBackend 会创建 StandaloneAppClient，并启动 client；
5. (client/driver端) StandaloneAppClient 启动过程中会向 Master 注册 Application,通过 rpcEnv 发送一个 RegisterApplication 信息；
6. (Master节点) Master 接收到 RegisterApplication 信息后，会在 MetricsSystem 度量系统中注册 Applicaiton 需要的资源，然后调用 schedule 函数执行调度过程，schedule函数会为 worker 进行计算资源的调度分配，最后发送 LaunchExecutor 给各个 Worker 通知其启动相应的 Executor 进程；
7. (Worker节点) Worker 接收到 LaunchExecutor 消息后会创建相应的工作目录，并启动 ExecutorRunner，ExecutorRunner 会启动一个线程来执行具体的任务；


&nbsp;
&nbsp;
>本文作者：ZJP
版权声明：本文为博主原创文章，遵循 [CC 4.0 BY-SA](http://creativecommons.org/licenses/by-sa/4.0/) 版权协议，转载请附上原文出处链接和本声明。
本文链接：http://zhoujiapeng.top/Spark/spark-application/