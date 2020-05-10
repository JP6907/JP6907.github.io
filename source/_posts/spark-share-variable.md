---
title: Spark源码阅读 之 共享变量
subtitle: Spark 共享变量
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - spark
  - 大数据
  - 源码阅读
categories:
  - Spark
date: 2019-09-05 19:36:45
---


# Spark 共享变量

> Spark版本：2.4.0

&emsp;Spark一个非常重要的特性就是共享变量。通常情况下，传递给 Spark 操作（例如 map 或 reduce）的函数是在远程集群节点上执行的，函数中使用的变量，在多个节点上执行时是同一变量的多个副本。这些变量被拷贝到每台机器上，并且在远程机器上对变量的更新不会回传给驱动程序。在任务之间支持通用的，可读写的共享变量是效率是非常低的。
&emsp;Spark 提供了两种类型的共享变量 : 广播变量（broadcast variables）和 累加器（accumulators）。Broadcast Variable会将使用到的变量，仅仅为每个节点拷贝一份，更大的用处是优化性能，减少网络传输以及内存消耗。Accumulator则可以让多个task共同操作一份变量，主要可以进行累加操作。


## 1. 广播变量
&emsp;广播变量允许用户保留一个只读的**变量**，缓存在每一台机器上，而不用在任务（Task）之间传递变量。广播变量可以被用于有效地给**每个节点**一个大输入数据集的副本而**非每个任务**保存一份复制。同时 Spark 采用 **P2P** 的方式来传播广播变量，从而减少通信开销。
&emsp;广播变量的实例对象的 value 值不能在广播后被修改，这样可以保证所有节点收到的都是一样的广播值。
&emsp;广播变量通常使用方式包括共享配置文件、map数据集、树形数据结构等，为能够更好更快速为 Task 使用相关变量。
&emsp;接下来从三个方面解析广播变量：初始化、创建（写入）、使用（读取）。

### 1.1 初始化
&emsp;SparkEnv 中会有一个 BroadcastManager 对象被创建：
```scala
//SparkEnv
val broadcastManager = new BroadcastManager(isDriver, conf, securityManager)
```
&emsp;BroadcastManager 的初始化函数 initialize 中会创建一个 broadcastFactory，factory 负责创建我们所需要的 Broadcast 对象。BroadcastFactory 是一个 trait，但是在 Spark 2.4.0 版本中，BroadcastFactory 只有一种实现(1.X版本中有 HttpBroadcastFactory 方式)，也就是 BroadcastManager 中创建的 broadcastFactory 类型 --- TorrentBroadcastFactory。TorrentBroadcastFactory 使用类似于 BitTorrent 的协议（P2P方式）在 Executors 之间传输和广播变量。
```scala
//BroadcastManager
private def initialize() {
    synchronized {
      if (!initialized) {
        broadcastFactory = new TorrentBroadcastFactory
        broadcastFactory.initialize(isDriver, conf, securityManager)
        initialized = true
      }
    }
  }
```
&emsp;调用 SparkContext.broadcast(value) 可以创建一个 Broadcast 实例，它的值可以通过调用 Broadcast 实例的 value() 方法获得。
```scala
//SparkContext
def broadcast[T: ClassTag](value: T): Broadcast[T] = {
    assertNotStopped()
    require(!classOf[RDD[_]].isAssignableFrom(classTag[T].runtimeClass),
      "Can not directly broadcast RDDs; instead, call collect() and broadcast the result.")
    val bc = env.broadcastManager.newBroadcast[T](value, isLocal)
    val callSite = getCallSite
    logInfo("Created broadcast " + bc.id + " from " + callSite.shortForm)
    cleaner.foreach(_.registerBroadcastForCleanup(bc))
    bc
  }
```
&emsp;这里调用 env.broadcastManager 的 newBroadcast 方法来创建一个新的 Broadcast 对象，env 里面的 broadcastManager 是 TorrentBroadcastFactory，所以创建的是 TorrentBroadcast。
```scala
//TorrentBroadcastFactory
override def newBroadcast[T: ClassTag](value_ : T, isLocal: Boolean, id: Long): Broadcast[T] = {
    new TorrentBroadcast[T](value_, id)
  }
```
&emsp;在 TorrentBroadcast 的初始化过程中，会调用 setConf() 方法将 SparkConf 对象注入到 TorrentBroadcast 中，setConf() 方法里面会设置压缩方式。并调用 writeBlocks() 方法将数据切分储存。
```scala
//TorrentBroadcast
setConf(SparkEnv.get.conf)

/** Total number of blocks this broadcast variable contains. */
private val numBlocks: Int = writeBlocks(obj)
```

### 1.2 创建（写入）
&emsp;上面讲到 TorrentBroadcast 实例创建过程，会调用 writeBlocks 方法，这个方法的主要功能是按照定义的广播块大小切分数据，切分块大小默认是4M（spark.broadcast.blockSize），在 setConf 方法中被设置。然后调用 blockManager 将切分后的数据信息储存到磁盘中。

```scala
//TorrentBroadcast
  /**
   * Divide the object into multiple blocks and put those blocks in the block manager.
   *
   * @param value the object to divide
   * @return number of blocks this broadcast variable is divided into
   */
  private def writeBlocks(value: T): Int = {
    import StorageLevel._
    // Store a copy of the broadcast variable in the driver so that tasks run on the driver
    // do not create a duplicate copy of the broadcast variable's value.
    val blockManager = SparkEnv.get.blockManager

    //持久化广播对象，blockid是broadcastId
    if (!blockManager.putSingle(broadcastId, value, MEMORY_AND_DISK, tellMaster = false)) {
      throw new SparkException(s"Failed to store $broadcastId in BlockManager")
    }

    //切分成块
    val blocks =
      TorrentBroadcast.blockifyObject(value, blockSize, SparkEnv.get.serializer, compressionCodec)
    if (checksumEnabled) {
      checksums = new Array[Int](blocks.length)
    }
    blocks.zipWithIndex.foreach { case (block, i) =>
      if (checksumEnabled) {
        checksums(i) = calcChecksum(block)
      }

      //序列化方式储存每个切分块，blockid是pieceId
      val pieceId = BroadcastBlockId(id, "piece" + i)
      val bytes = new ChunkedByteBuffer(block.duplicate())
      if (!blockManager.putBytes(pieceId, bytes, MEMORY_AND_DISK_SER, tellMaster = true)) {
        throw new SparkException(s"Failed to store $pieceId of $broadcastId in local BlockManager")
      }
    }
    blocks.length
  }
```
&emsp;writeBlocks 在储存切分的广播块之前需要先调用 blockManager.putSingle 将广播变量以非序列化方式持久化，然后再将广播变量切分成块，调用 putBytes 将切分好的块以**序列化**的方式储存。putBytes 根据储存策略会优先写入本地，既然序列化数据是本地储存，由此而来的问题是读取问题，BlockManager 储存数据不会像 HDFS 依据备份策略储存多份数据放置不同节点，如果没有备份数据，那么必然产生数个问题：
- 节点故障，无法访问节点数据；
- 数据热点，所有任务都会使用该数据；
- 网络传输，所有节点频繁访问单节点；

### 1.3 使用（读取）
&emsp;为了解决广播变量读取的问题，Spark 采用 P2P 点对点方式，只要使用过广播变量数据，则在本节点储存数据，由此变成新的数据源，随着数据源不断增加，传输数据的速度也会越来越快。广播变量的值通过调用 Broadcast.value 来获取：
```scala
//Broadcast
def value: T = {
    assertValid()
    getValue()
  }
//TorrentBroadcast
override protected def getValue() = {
    _value
}
@transient private lazy val _value: T = readBroadcastBlock()
```

```scala
//TorrentBroadcast
private def readBroadcastBlock(): T = Utils.tryOrIOException {
    TorrentBroadcast.synchronized {
      val broadcastCache = SparkEnv.get.broadcastManager.cachedValues

      Option(broadcastCache.get(broadcastId)).map(_.asInstanceOf[T]).getOrElse {
        setConf(SparkEnv.get.conf)
        val blockManager = SparkEnv.get.blockManager

        //尝试从本地读取
        blockManager.getLocalValues(broadcastId) match {
          case Some(blockResult) =>
            if (blockResult.data.hasNext) {
              ...
          case None =>
            logInfo("Started reading broadcast variable " + id)
            val startTimeMs = System.currentTimeMillis()

            //从其它节点获取
            val blocks = readBlocks()
            logInfo("Reading broadcast variable " + id + " took" + Utils.getUsedTimeMs(startTimeMs))

            try {
              ...
              val storageLevel = StorageLevel.MEMORY_AND_DISK

              //储存到本地，并汇报给 master
              if (!blockManager.putSingle(broadcastId, obj, storageLevel, tellMaster = false)) {
                throw new SparkException(s"Failed to store $broadcastId in BlockManager")
              }

              ...
              blocks.foreach(_.dispose())
            }
        }
      }
    }
  }
```
&emsp;readBlocks 方法会尝试从其它节点获取广播块：
```scala
//TorrentBroadcast
private def readBlocks(): Array[BlockData] = {
    // Fetch chunks of data. Note that all these chunks are stored in the BlockManager and reported
    // to the driver, so other executors can pull these chunks from this executor as well.
    val blocks = new Array[BlockData](numBlocks)
    val bm = SparkEnv.get.blockManager

    //循环遍历所有块，避免访问热点，随即顺序读取数据
    for (pid <- Random.shuffle(Seq.range(0, numBlocks))) {
      val pieceId = BroadcastBlockId(id, "piece" + pid)
      logDebug(s"Reading piece $pieceId of $broadcastId")
      // First try getLocalBytes because there is a chance that previous attempts to fetch the
      // broadcast blocks have already fetched some of the blocks. In that case, some blocks
      // would be available locally (on this executor).
      bm.getLocalBytes(pieceId) match {   //获取本地数据成功
        case Some(block) =>
          blocks(pid) = block
          releaseLock(pieceId)
        case None =>                    
          bm.getRemoteBytes(pieceId) match {  
            case Some(b) =>
              if (checksumEnabled) {
                val sum = calcChecksum(b.chunks(0))
                if (sum != checksums(pid)) {
                  throw new SparkException(s"corrupt remote block $pieceId of $broadcastId:" +
                    s" $sum != ${checksums(pid)}")
                }
              }
              // We found the block from remote executors/driver's BlockManager, so put the block
              // in this executor's BlockManager.
              
              //储存到本地，并汇报Master
              if (!bm.putBytes(pieceId, b, StorageLevel.MEMORY_AND_DISK_SER, tellMaster = true)) {
                throw new SparkException(
                  s"Failed to store $pieceId of $broadcastId in local BlockManager")
              }
              blocks(pid) = new ByteBufferBlockData(b, true)
            case None =>
              throw new SparkException(s"Failed to get $pieceId of $broadcastId")
          }
      }
    }
    blocks
  }
```
&emsp;在 readBlocks 方法中，为了避免访问热点，会以随机顺序读取数据。读取某个块的时候，会先再一次尝试从本地读取，这是因为在上一次获取的时候可能已经成功获取到某些块。如果这一次还是无法从本地获取到数据，则调用 getRemoteBytes 从其它节点获取，getRemoteBytes 方法在[《Spark储存体系》](http://zhoujiapeng.top/Spark/spark-storage/#532-get-1)文章中已经介绍过了，这里不再赘述。

### 1.4 广播变量总结
&emsp;到这里我们知道，TorrentBroadcast 首先将广播变量数据分块，并存到 BlockManager 中；每个节点需要读取广播变量时，是分块读取，对每一块都读取其位置信息，然后选择其中一个存有此块的节点进行读取；每个节点读取后会将包含的广播块信息报告给 mBlockadeanagerMaster，这样本地节点就成了这个广播网络中的一个 peer。


### 1.5 一个特殊例子
&emsp;广播变量有时候可以用来解决由于数据切片造成的程序逻辑错误问题。
&emsp;举例来说，有两个表，ip规则表和日志表，ip规则表记录了ip和省份的映射关系，日志表记录了页面访问地址以及访问ip，现在需要使用这两个表进行匹配聚合，统计出各个省份的访问ip总数量，两个表的数据都是储存在HDFS中的。数据示例如下：
```
//访问日志示例
20090121000132095572000|125.213.100.123|show.51.com|/shoplist.php?phpfile=shoplist2.php&style=1&sex=137|Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; SV1; Mozilla/4.0(Compatible Mozilla/4.0(Compatible-EmbeddedWB 14.59 http://bsalsa.com/ EmbeddedWB- 14.59  from: http://bsalsa.com/ )|http://show.51.com/main.php|
```
```
//ip规则表示例，数据按照ip有序存放
1.0.1.0|1.0.3.255|16777472|16778239|亚洲|中国|福建|福州||电信|350100|China|CN|119.306239|26.075302
```
&emsp;稍微不注意的话，或许我们会这么做，从HDFS中将数据读取到 ipRulesRDD、和 logRDD 之后，遍历 logRDD，逐一使用二分法去比对 ipRulesRDD，通过这种方式查找出每一条日志记录中ip对应的省份。但是这种做法是错误的，这里忽略了一个问题，如果数据是存放在 HDFS 的话，那么读取数据的时候日志表和**ip规则表都会被分区**，每个 Executor 节点都只能读取到部分的ip规则表而不是完整的ip规则表，因此 Executor 在查找ip规则表的时候，就只能使用本地获取到的那部分ip规则表进行比对，这样势必会造成某些ip无法查找到对应的省份。
&emsp;为了解决这个问题，我们需要让每一个 Executor 节点都拥有完整的ip规则表，这时候广播变量就发挥作用了。我们可以先调用 RDD.collect 方法将完整的数据收集到 Driver 端，然后再调用 sc.broadcast 将数据ip规则表广播到所有的 Executor 节点上，这样的话，每一个 Executor 都能够持有一个完整的ip规则表的数据副本。
```scala
//从hdfs读取ip规则数据
val rulesLines = sc.textFile(args(0))
//整理ip规则数据
//整理ip规则数据
val ipRulesRDD: RDD[(Long, Long, String)] = rulesLines.map(line => {
  val fields = line.split("[|]")
  val startNum = fields(2).toLong
  val endNum = fields(3).toLong
  val province = fields(6)
  (startNum, endNum, province)
})
//从各个executor收集ip规则到driver端
val rulesInDriver : Array[(Long,Long,String)] = ipRulesRDD.collect()
//从driver端广播ip规则到各个executor
val broadcastRef = sc.broadcast(rulesInDriver)
```


## 2. 累加器
&emsp;累加器是一种只能通过关联操作进行“加”操作的变量，因此它能够高效的应用于并行操作中。它们能够用来实现计数器和求和器。Spark原生支持Int和Double类型的累加器，开发者可以自己添加支持的类型。运行在集群上的任务可以给累加器增加值，但是它们不能读取这个值，**只有驱动程序（Driver端程序）可以使用 Accumulators 对象的 value 方法来读取累加器的值**。
&emsp;我们可以创建命名或未命名的累加器。命名累加器会在 Web UI(4040端口) 中展示。Spark在Tasks 任务表中显示由任务修改的每个累加器的值。比如创建一个 longAccumulator 类型的累加器：
```scala
val longAccumulator = sc.longAccumulator("long-account")
sc.parallelize(Array(1,2,3,4)).foreach(x => longAccumulator.add(x))
```
&emsp;在 WebUI 中可以看到：
![accumulators](https://gitee.com/JP6907/Pic/raw/master/spark/accumulators.png?raw=true)
&emsp;跟踪 WebUI 中的累加器对于理解运行的 stage　的进度很有用。&nbsp;
&emsp;下面我们来看一下累加器的实现，以 longAccumulator 为例，当调用 sc.longAccumulator 的时候：
```scala
//SparkContext
def longAccumulator(name: String): LongAccumulator = {
    val acc = new LongAccumulator
    register(acc, name)
    acc
  }
def register(acc: AccumulatorV2[_, _], name: String): Unit = {
    acc.register(this, name = Option(name))
  }

//AccumulatorV2
private[spark] def register(
      sc: SparkContext,
      name: Option[String] = None,
      countFailedValues: Boolean = false): Unit = {
    if (this.metadata != null) {
      throw new IllegalStateException("Cannot register an Accumulator twice.")
    }
    this.metadata = AccumulatorMetadata(AccumulatorContext.newId(), name, countFailedValues)
    AccumulatorContext.register(this)
    sc.cleaner.foreach(_.registerAccumulatorForCleanup(this))
  }
```
&emsp;最后会调用 AccumulatorV2 的 register 方法，可以看到，这个方法里面将累加器注册到了 AccumulatorContext 里面：
```scala
//AccumulatorContext
def register(a: AccumulatorV2[_, _]): Unit = {
    originals.putIfAbsent(a.id, new jl.ref.WeakReference[AccumulatorV2[_, _]](a))
  }

private val originals = new ConcurrentHashMap[Long, jl.ref.WeakReference[AccumulatorV2[_, _]]]
```
&emsp;在 AccumulatorContext 中注册累加器其实就是将累加器添加到 originals 中，originals 是一个 HashMap。因此，Driver 端是通过 AccumulatorContext 里面的一个 HashMap 变量来维护所有的累加器。
&emsp;当调用累加器的 value 方法时，直接返回累加器的和：
```scala
//LongAccumulator
override def value: jl.Long = _sum
override def add(v: jl.Long): Unit = {
    _sum += v
    _count += 1
  }
```


&nbsp;
&nbsp;
>本文作者：ZJP
版权声明：本文为博主原创文章，遵循 [CC 4.0 BY-SA](http://creativecommons.org/licenses/by-sa/4.0/) 版权协议，转载请附上原文出处链接和本声明。
本文链接：http://zhoujiapeng.top/Spark/spark-share-variable/