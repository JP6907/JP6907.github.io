---
title: Spark源码阅读 之 Storage储存模块
catalog: true
date: 2019-09-05 10:37:47
subtitle: Spark Storage储存模块
header-img: "/img/article_header/article_header.png"
tags:
- spark
- 大数据
- 源码阅读
categories:
  - Spark
---


# Spark储存体系

> Spark版本：2.4.0

## 1.储存体系架构
&emsp;简单来说，Spark 储存体系是各个 Driver 和 Executor 实例中的 BlockManager 所组成的，但从一个整体上看，Spark 储存体系包含很多组件，如图所示。
![storage-architecture](https://gitee.com/JP6907/Pic/raw/master/spark/storage-architecture.png)

- BlockManagerMaster：代理 BlockManager 与 Driver 上的 BlockManagerMasterEndpoint 通信。图中记号1表示 Executor 节点上的 BlockManager 通过 BlockManagerMaster 与 BlockManagerMasterEndpoint 进行通信，记号2表示 Driver 节点上的 BlockManager 通过 BlockManagerMaster 与 BlockManagerMasterEndpoint 进行通信。

- BlockManagerMasterEndpoint：由 Driver 上的 SparkEnv 负责创建和注册到 Driver 的 RpcEnv 中。**BlockManagerMasterEndpoint 只存在 Driver 的 SparkEnv 中**，Driver 或 Executor 上的 BlockManagerMaster 的 driverEndpoint 持有 BlockManagerMasterEndpoint 的 **RpcEndPointRef**。BlockManagerMasterEndpoint 主要对各个节点上的 BlockManager、BlockManager 与 Executor 的映射关系及 Block 位置信息等进行管理。

- BlockManagerSlaveEndpoint：每个 Executor 或 Driver 的 SparkEnv 中的 BlockManager 都会有创建并注册属于自己的 slaveEndpoint。BlockManagerSlaveEndpoint 将接收 BlockManagerMasterEndpoint 下发的命令。图中记号3和4分别表示 BlockManagerMasterEndpoint 向 Driver节点 和 Executor节点上的 BlockManagerSlaveEndpoint 下发命令。

- SerializerManager：序列化管理器。

- MemoryManager：内存管理器，负责对单个节点上内存的分配与回收。

- MapOutputTracker：map任务输出跟踪器。

- ShuffleManager：Shuffle管理器。

- BlockTransferService：块传输服务。

- shuffleClient：Shuffle的客户端，与 BlockTransferService 配合使用。

- SecurityManager：安全管理器。

- DiskBlockManager：磁盘管理器，对磁盘上的文件及目录的读写操作进行管理。

- BlockInfoManager：块信息管理器，负责Block的元数据及锁资源进行管理。

- MemoryStore：内存储存。依赖于MemoryManager，负责对Block的内存储存。

- DiskStore：磁盘储存。依赖于DiskBlockManager，负责对Block的磁盘储存。


## 2.BlockManagerMaster对BlockManager的管理
&emsp;BlockManagerMaster 的作用是对存在于 Executor 或 Driver 上的 BlockManager 进行统一管理。Executor 与 Driver 关于 BlockManager 的交互都依赖于 BlockManagerMaster。Driver 上的 BlockManagerMaster 会实例化并注册 BlockManagerMasterEndpoint。无论是 Driver 还是 Executor，它们的 BlockManagerMaster 的 driverEndpoint 属性都持有 BlockManagerMasterEndpoint 的 RpcEndpointRef。无论是 Driver 还是 Executor，每个 BlockManager 都拥有自己的 BlockManagerSlaveEndpoint 的 RpcEndpointRef。
&emsp;BlockManagerMaster 负责发送消息，BlockManagerMasterEndpoint 负责消息的接收与处理。BlockManagerSlaveEndpoint 则接收 BlockManagerMasterEndpoint 下发的命令。


## 3.BlockManager的创建
&emsp;BlockManager 在 SparkEnv 类的 create 方法中被创建：
```scala
//SparkEnv
private def create(
      ... ): SparkEnv = {
    
    ...

    val blockManagerMaster = new BlockManagerMaster(registerOrLookupEndpoint(
      BlockManagerMaster.DRIVER_ENDPOINT_NAME,
      new BlockManagerMasterEndpoint(rpcEnv, isLocal, conf, listenerBus)),
      conf, isDriver)

    // NB: blockManager is not valid until initialize() is called later.
    val blockManager = new BlockManager(executorId, rpcEnv, blockManagerMaster,
      serializerManager, conf, memoryManager, mapOutputTracker, shuffleManager,
      blockTransferService, securityManager, numUsableCores)
    
    ...
}
```
&emsp;其中会先创建 BlockManagerMaster，在 registerOrLookupEndpoint 中根据是否为 Driver 节点来创建 BlockManagerMasterEndpoint 的 RpcEndpointRef：
```scala
//SparkEnv
def registerOrLookupEndpoint(
        name: String, endpointCreator: => RpcEndpoint):
      RpcEndpointRef = {
      if (isDriver) {
        logInfo("Registering " + name)
        rpcEnv.setupEndpoint(name, endpointCreator)
      } else {
        RpcUtils.makeDriverRef(name, conf, rpcEnv)
      }
    }
```
&emsp;registerOrLookupEndpoint 创建结果会当作参数传进 BlockManagerMaster，并赋值给了 BlockManagerMaster 中的 driverEndpoint 变量。无论当前节点是 Driver 或者 Executor，这个 driverEndpoint 都代表着 BlockManagerMasterEndpoint，当需要向 Driver 发送消息的时候就要调用这个 driverEndpoint。
&emsp;另外，在 BlockManager 内部，会创建一个 slaveEndpoint，用来代表自己，如果当前节点是 Executor，则 Driver 需要调用这个 slaveEndpoint 发送消息。
```scala
//BlockManager
private val slaveEndpoint = rpcEnv.setupEndpoint(
    "BlockManagerEndpoint" + BlockManager.ID_GENERATOR.next,
    new BlockManagerSlaveEndpoint(rpcEnv, this, mapOutputTracker))
```

## 4.通信层
&emsp;Driver 和 Executor 的通信通过 RpcEndpoint 来实现，无论是 Driver 还是 Executor，它们的 BlockManagerMaster 的 driverEndpoint 属性都持有 BlockManagerMasterEndpoint 的 RpcEndpointRef。无论是 Driver 还是 Executor，每个 BlockManager 都拥有自己的 BlockManagerSlaveEndpoint 的 RpcEndpointRef。
&emsp;Executor 与 Driver 关于 BlockManager 的交互都依赖于 BlockManagerMaster。在 BlockManager 中会通过 BlockManagerMaster 向 BlockManagerMasterEndpoint 发送消息，以注册为例：
```scala
//org.apache.spark.storage.BlockManager
def reregister(): Unit = {
    // TODO: We might need to rate limit re-registering.
    logInfo(s"BlockManager $blockManagerId re-registering with master")
    master.registerBlockManager(blockManagerId, maxOnHeapMemory, maxOffHeapMemory, slaveEndpoint)
    reportAllBlocks()
  }
```
这里调用 master 的 registerBlockManager 方法进行注册，查看该方法：
```scala
def registerBlockManager(
      blockManagerId: BlockManagerId,
      maxOnHeapMemSize: Long,
      maxOffHeapMemSize: Long,
      slaveEndpoint: RpcEndpointRef): BlockManagerId = {
    logInfo(s"Registering BlockManager $blockManagerId")
    val updatedId = driverEndpoint.askSync[BlockManagerId](
      RegisterBlockManager(blockManagerId, maxOnHeapMemSize, maxOffHeapMemSize, slaveEndpoint))
    logInfo(s"Registered BlockManager $updatedId")
    updatedId
  }
```
&emsp;这里通过 driverEndpoint 发送了一个 RegisterBlockManager 信息，这里的 driverEndpoint 是 BlockManagerMaster 构造的时候传入的 BlockManagerMasterEndpoint 的 RpcEndpointRef。
&emsp;简单来说，当 Executor 需要向 Driver 发送消息的时候，消息通信由 SparkEnv 的 BlockManagerMaster 负责，向 BlockManagerMaster 中的 **driverEndpoint**，即 BlockManagerMasterEndpoint 的 RpcEndPointRef 发送消息即可。BlockManagerMasterEndpoint 消息处理函数为 receive 函数：
```scala
override def receiveAndReply(context: RpcCallContext): PartialFunction[Any, Unit] = {
    case RegisterBlockManager(blockManagerId, maxOnHeapMemSize, maxOffHeapMemSize, slaveEndpoint) =>
      context.reply(register(blockManagerId, maxOnHeapMemSize, maxOffHeapMemSize, slaveEndpoint))

    ...
}
```
&emsp;由于在 BlockManagerMaster 调用 registerBlockManager 发送消息的时候，使用 driverEndpoint.askSync，即同步方式，返回结果直接赋值给 updatedId，所以不需要有单独的消息处理函数。
&emsp;我们看一下 Driver 向 Executor 发送消息的过程，这时候则需要向 BlockManager 的 **slaveEndpoint** 发送消息，这个 slaveEndpoint 代表指定 Executor 的 BlockManager。
```scala
//BlockManager
private val slaveEndpoint = rpcEnv.setupEndpoint(
    "BlockManagerEndpoint" + BlockManager.ID_GENERATOR.next,
    new BlockManagerSlaveEndpoint(rpcEnv, this, mapOutputTracker))
```
```scala
//BlockManagerMasterEndpoint
private def removeBlockFromWorkers(blockId: BlockId) {
    val locations = blockLocations.get(blockId)
    if (locations != null) {
      locations.foreach { blockManagerId: BlockManagerId =>
        val blockManager = blockManagerInfo.get(blockManagerId)
        if (blockManager.isDefined) {
          // Remove the block from the slave's BlockManager.
          // Doesn't actually wait for a confirmation and the message might get lost.
          // If message loss becomes frequent, we should add retry logic here.
          
          //发送消息
          blockManager.get.slaveEndpoint.ask[Boolean](RemoveBlock(blockId))
        }
      }
    }
  }
```
BlockManagerMasterEndpoint 会向 Executor 对应的 blockManager 的 slaveEndpoint 发送消息，slaveEndpoint处理消息如下：
```scala
//BlockManagerSlaveEndpoint
override def receiveAndReply(context: RpcCallContext): PartialFunction[Any, Unit] = {
    case RemoveBlock(blockId) =>
      doAsync[Boolean]("removing block " + blockId, context) {
        blockManager.removeBlock(blockId)
        true
      }

    ...
}
```

## 5.储存层(缓存实现)
&emsp;在 RDD 层面来说，RDD 是由不同的 partition 组成的，我们所进行的 transformation 或 action 操作 都是在 partition 上面进行的；而在 Storage 模块内部，RDD 由被视为由不同的 block 组成，对 RDD 的存取是以 block 为单位进行的，**本质上 partition 和 block 是等价的**，只是看待角度不一样。在 Spark storage 模块中存取数据的最小单位是 block，所有的操作都是以 block 为单位进行的。
&emsp;BlockManager 对象被创建的时候会创建出 MemoryStore 和 DiskStore，作为实际 Block 的储存位置。
```scala
val diskBlockManager = {
    // Only perform cleanup if an external service is not serving our shuffle files.
    val deleteFilesOnStop =
      !externalShuffleServiceEnabled || executorId == SparkContext.DRIVER_IDENTIFIER
    new DiskBlockManager(conf, deleteFilesOnStop)
  }
// Actual storage of where blocks are kept
  private[spark] val memoryStore =
    new MemoryStore(conf, blockInfoManager, serializerManager, memoryManager, this)
  private[spark] val diskStore = new DiskStore(conf, diskBlockManager, securityManager)
  memoryManager.setMemoryStore(memoryStore)
```

### 5.1 DiskStore
&emsp;磁盘储存可以配置多个本地储存目录，通过配置 spark.local.dir 属性或 系统属性 java.io.tmpdir 制定的目录，并在每个路径下创建以 blockmgr- 为前缀，UUID 为后缀的随机字符串的子目录
```scala
//DiskBlockManager
private def createLocalDirs(conf: SparkConf): Array[File] = {
    Utils.getConfiguredLocalDirs(conf).flatMap { rootDir =>
      try {
        val localDir = Utils.createDirectory(rootDir, "blockmgr")
        logInfo(s"Created local directory at $localDir")
        Some(localDir)
      } catch {
        case e: IOException =>
          logError(s"Failed to create local dir in $rootDir. Ignoring this directory.", e)
          None
      }
    }
  }
```
&emsp;图片表示 DiskBlockManager 管理的文件目录结构，可以看到，每个一级目录，即 localDirs 下面都会有多个 subDirs，每个 subDir 则储存一个或多个 Block 文件。
![DiskBlockManager-dir](https://gitee.com/JP6907/Pic/raw/master/spark/DiskBlockManager-dir.png)
&emsp;实际 Block 是如何映射到磁盘的具体文件的？我们可以看一下下面函数：
```scala
//DiskBlockManager
def getFile(blockId: BlockId): File = getFile(blockId.name)

def getFile(filename: String): File = {
    // Figure out which local directory it hashes to, and which subdirectory in that
    val hash = Utils.nonNegativeHash(filename)
    val dirId = hash % localDirs.length
    val subDirId = (hash / localDirs.length) % subDirsPerLocalDir

    // Create the subdirectory if it doesn't already exist
    val subDir = subDirs(dirId).synchronized {
      val old = subDirs(dirId)(subDirId)
      if (old != null) {
        old
      } else {
        val newDir = new File(localDirs(dirId), "%02x".format(subDirId))
        if (!newDir.exists() && !newDir.mkdir()) {
          throw new IOException(s"Failed to create local dir in $newDir.")
        }
        subDirs(dirId)(subDirId) = newDir
        newDir
      }
    }

    new File(subDir, filename)
  }
```
&emsp;先是获取 blockId 的名字，BlockId 是一个抽象类，它有很多中实现，比如 RDDBlockId,ShuffleBlockId等，以RDDBlockId为例，它的规则为：
```scala
//org.apache.spark.storage.BlockId
case class RDDBlockId(rddId: Int, splitIndex: Int) extends BlockId {
  override def name: String = "rdd_" + rddId + "_" + splitIndex
}
```
&emsp;获取到 blockid 的名字之后，接下来以下面的规则进行 block 到 file 的映射：
1. 获取文件名的 hash；
```scala
val hash = Utils.nonNegativeHash(filename)
```
2. hash 和 localDirs 数组的长度取余获得选中的一级目录；
```scala
val dirId = hash % localDirs.length
```
3. hash 和 localDirs 数组的长度的商和 subDirsPerLocalDir 取余获得二级目录；
```scala
val subDirId = (hash / localDirs.length) % subDirsPerLocalDir
```
如果二级目录不存在，则需要创建二级目录。

### 5.2 MemoryStore
&emsp;MemoryStore 管理 Block 相对于 BlockStore 来说比较简单，MemoryStore 内部维护了一个 Hashmap 来管理所有的 block，以 block id 为 key 将 block 存放到 Hashmap中。block 的内容则被封装在一个结构体 MemoryEntry。
```scala
private val entries = new LinkedHashMap[BlockId, MemoryEntry[_]](32, 0.75f, true)
```
&emsp;在 MemoryStore 中存放 block必须确保内存足够容纳下该 block，JVM 默认 是60%可以被内存缓存用来储存 block。当超过60%之后，Spark 会根据配置的缓存策略来决定是丢弃一些 block 还是将一些 block 储存到磁盘上。


### 5.3 BlockManager存取Block
&emsp;对于 Block 的存取不需要我们直接通过 DiskStore 或 MemoryStore 来进行，BlockManager 提供了相应的 get 和 put 接口。

#### 5.3.1 put
&emsp;在 BlockManager 中，putSingle 方法能够将一个对象写入 Block，putSingle 调用了 putIterator 方法：
```scala
private def doPutIterator[T](
      blockId: BlockId,
      iterator: () => Iterator[T],
      level: StorageLevel,
      classTag: ClassTag[T],
      tellMaster: Boolean = true,
      keepReadLock: Boolean = false): Option[PartiallyUnrolledIterator[T]] = {
    doPut(blockId, level, classTag, tellMaster = tellMaster, keepReadLock = keepReadLock) { info =>
      ...
    }
  }
```
&emsp;doPutIterator 方法里面只调用了 doPut 方法，后面 {} 里面的内容是传入 doPut 方法的一个函数，该函数的参数是 Blockinfo，函数体的具体内容是根据 Blockinfo 执行相应的储存操作。我们先看一下 doPut 方法：
```scala
private def doPut[T](
      blockId: BlockId,
      level: StorageLevel,
      classTag: ClassTag[_],
      tellMaster: Boolean,
      keepReadLock: Boolean)(putBody: BlockInfo => Option[T]): Option[T] = {

    require(blockId != null, "BlockId is null")
    require(level != null && level.isValid, "StorageLevel is null or invalid")

    val putBlockInfo = {
      val newInfo = new BlockInfo(level, classTag, tellMaster)
      if (blockInfoManager.lockNewBlockForWriting(blockId, newInfo)) {
        newInfo
      } else {
        logWarning(s"Block $blockId already exists on this machine; not re-adding it")
        if (!keepReadLock) {
          // lockNewBlockForWriting returned a read lock on the existing block, so we must free it:
          releaseLock(blockId)
        }
        return None
      }
    }

    val startTimeMs = System.currentTimeMillis
    var exceptionWasThrown: Boolean = true
    val result: Option[T] = try {
      val res = putBody(putBlockInfo)
      
      ...

    } 

    ...

    result
  }
```
&emsp;在 doPut 方法里面，先构造一个 BlockInfo，然后将 BlockInfo 放进 putBody，putBody 就是前面 doPutIterator 方法中传入 doPut 方法的函数，我们再来看一下这个函数：
```scala
private def doPutIterator[T](
      blockId: BlockId,
      iterator: () => Iterator[T],
      level: StorageLevel,
      classTag: ClassTag[T],
      tellMaster: Boolean = true,
      keepReadLock: Boolean = false): Option[PartiallyUnrolledIterator[T]] = {
    doPut(blockId, level, classTag, tellMaster = tellMaster, keepReadLock = keepReadLock) { info =>
      val startTimeMs = System.currentTimeMillis
      var iteratorFromFailedMemoryStorePut: Option[PartiallyUnrolledIterator[T]] = None
      // Size of the block in bytes
      var size = 0L

      //根据储存级别储存到本地
      if (level.useMemory) {
        // Put it in memory first, even if it also has useDisk set to true;
        // We will drop it to disk later if the memory store can't hold it.
        if (level.deserialized) {
          memoryStore.putIteratorAsValues(blockId, iterator(), classTag) match {
            case Right(s) =>
              size = s
            case Left(iter) =>
              // Not enough space to unroll this block; drop to disk if applicable
              if (level.useDisk) {
                logWarning(s"Persisting block $blockId to disk instead.")
                diskStore.put(blockId) { channel =>
                  val out = Channels.newOutputStream(channel)
                  serializerManager.dataSerializeStream(blockId, out, iter)(classTag)
                }
                size = diskStore.getSize(blockId)
              } else {
                iteratorFromFailedMemoryStorePut = Some(iter)
              }
          }
        } else { // !level.deserialized
          memoryStore.putIteratorAsBytes(blockId, iterator(), classTag, level.memoryMode) match {
            case Right(s) =>
              size = s
            case Left(partiallySerializedValues) =>
              // Not enough space to unroll this block; drop to disk if applicable
              if (level.useDisk) {
                logWarning(s"Persisting block $blockId to disk instead.")
                diskStore.put(blockId) { channel =>
                  val out = Channels.newOutputStream(channel)
                  partiallySerializedValues.finishWritingToStream(out)
                }
                size = diskStore.getSize(blockId)
              } else {
                iteratorFromFailedMemoryStorePut = Some(partiallySerializedValues.valuesIterator)
              }
          }
        }

      } else if (level.useDisk) {
        diskStore.put(blockId) { channel =>
          val out = Channels.newOutputStream(channel)
          serializerManager.dataSerializeStream(blockId, out, iterator())(classTag)
        }
        size = diskStore.getSize(blockId)
      }

      val putBlockStatus = getCurrentBlockStatus(blockId, info)
      val blockWasSuccessfullyStored = putBlockStatus.storageLevel.isValid
      if (blockWasSuccessfullyStored) {
        // Now that the block is in either the memory or disk store, tell the master about it.
        info.size = size
        if (tellMaster && info.tellMaster) {
          reportBlockStatus(blockId, putBlockStatus)
        }
        addUpdatedBlockStatusToTaskMetrics(blockId, putBlockStatus)
        logDebug("Put block %s locally took %s".format(blockId, Utils.getUsedTimeMs(startTimeMs)))

        //如果副本数量大于1，需要储存到其它节点上
        if (level.replication > 1) {
          val remoteStartTime = System.currentTimeMillis
          val bytesToReplicate = doGetLocalBytes(blockId, info)
          // [SPARK-16550] Erase the typed classTag when using default serialization, since
          // NettyBlockRpcServer crashes when deserializing repl-defined classes.
          // TODO(ekl) remove this once the classloader issue on the remote end is fixed.
          val remoteClassTag = if (!serializerManager.canUseKryo(classTag)) {
            scala.reflect.classTag[Any]
          } else {
            classTag
          }
          try {
            replicate(blockId, bytesToReplicate, level, remoteClassTag)
          } finally {
            bytesToReplicate.dispose()
          }
          logDebug("Put block %s remotely took %s"
            .format(blockId, Utils.getUsedTimeMs(remoteStartTime)))
        }
      }
      assert(blockWasSuccessfullyStored == iteratorFromFailedMemoryStorePut.isEmpty)
      iteratorFromFailedMemoryStorePut
    }
  }
```
&emsp;这个函数体首先根据 block 的 storage level 将 block 储存到内存或者磁盘上，在本地储存成功之后，判断 block 指定的副本数量 level.replication 是否大于1，如果是，则调用 replicate 方法将 block 储存到其它节点上。在 replicate 方法里面会使用 blockTransferService 服务将 Block 传输到其它节点，而节点的选择会根据副本策略 blockReplicationPolicy，副本策略通过 spark.storage.replication.policy 配置来指定，默认使用 RandomBlockReplicationPolicy。
```scala
private def replicate(
      blockId: BlockId,
      data: BlockData,
      level: StorageLevel,
      classTag: ClassTag[_],
      existingReplicas: Set[BlockManagerId] = Set.empty): Unit = {

    ...

    var peersForReplication = blockReplicationPolicy.prioritize(
      blockManagerId,
      initialPeers,
      peersReplicatedTo,
      blockId,
      numPeersToReplicateTo)

    while(numFailures <= maxReplicationFailures &&
      !peersForReplication.isEmpty &&
      peersReplicatedTo.size < numPeersToReplicateTo) {
      val peer = peersForReplication.head
      try {
        
          ...

        blockTransferService.uploadBlockSync(
          peer.host,
          peer.port,
          peer.executorId,
          blockId,
          buffer,
          tLevel,
          classTag)
        logTrace(s"Replicated $blockId of ${data.size} bytes to $peer" +
          s" in ${(System.nanoTime - onePeerStartTime).toDouble / 1e6} ms")
        peersForReplication = peersForReplication.tail
        peersReplicatedTo += peer
      } 

      ...
  }
```

#### 5.3.2 get
&emsp;接下来我们看一下 BlockManager 的 get 方法，get 能够根据 id 获取指定的 Block。
```scala
def get[T: ClassTag](blockId: BlockId): Option[BlockResult] = {
    val local = getLocalValues(blockId)
    if (local.isDefined) {
      logInfo(s"Found block $blockId locally")
      return local
    }
    val remote = getRemoteValues[T](blockId)
    if (remote.isDefined) {
      logInfo(s"Found block $blockId remotely")
      return remote
    }
    None
  }
```
&emsp;get 方法比较简单，判断该 Block 是否在本地，是的话调用 getLocalValues 直接从本地读取，否则调用 getRemoteValues 请求从其它节点获取。这两个方法也相对比较简单，getLocalValues 根据 Block 的储存级别决定从内存或者磁盘读取。getRemoteValues 调用了 getRemoteBytes，getRemoteBytes 会先向 master 节点请求获取有储存该 Block 的节点地址，然后调用 blockTransferService 服务接收该 Block。
```scala
def getRemoteBytes(blockId: BlockId): Option[ChunkedByteBuffer] = {
    
    ...

    val locationsAndStatus = master.getLocationsAndStatus(blockId)
    
    ...

    while (locationIterator.hasNext) {
      val loc = locationIterator.next()
      logDebug(s"Getting remote block $blockId from $loc")
      val data = try {
        blockTransferService.fetchBlockSync(
          loc.host, loc.port, loc.executorId, blockId.toString, tempFileManager)
      } 

      ...
  }
```
### 5.4 Partition和Block的关系
&emsp;在 Storage 模块里面所有的操作都是和 Block 有关的，但是在 RDD 里面所有的运算都是基于 Partition 的，那么 Partition 是如何与 Block 对应上的？在 Spark 中，Partition 是一个逻辑上的概念，而 Block 是一个物理上的数据实体，**一个 RDD 的 Partition 对应于一个 Storage 模块中的 Block**，下面是 RDD 类中 iterator() 方法的实现：
```scala
final def iterator(split: Partition, context: TaskContext): Iterator[T] = {
    if (storageLevel != StorageLevel.NONE) {
      getOrCompute(split, context)
    } else {
      computeOrReadCheckpoint(split, context)
    }
  }
```
&emsp;如果储存级别不是 NONE 的话，表示该 RDD 所包括的一系列 Partition 以 Block 的状态储存在 BlockManager 中，调用 getOrCompute 获取：
```scala
private[spark] def getOrCompute(partition: Partition, context: TaskContext): Iterator[T] = {
    val blockId = RDDBlockId(id, partition.index)
    var readCachedBlock = true
    // This method is called on executors, so we need call SparkEnv.get instead of sc.env.
    SparkEnv.get.blockManager.getOrElseUpdate(blockId, storageLevel, elementClassTag, () => {
      readCachedBlock = false
      computeOrReadCheckpoint(partition, context)
    }) match {
      ...
    }
  }

@DeveloperApi
case class RDDBlockId(rddId: Int, splitIndex: Int) extends BlockId {
  override def name: String = "rdd_" + rddId + "_" + splitIndex
}
```
&emsp;可以看到，blockId 是由 RDD id 和 Partition index构造出来的，每一个 RDD 的一个分区都对应有一个独一无二的 block id，使用这个 block id 就能够从 BlockManager 获取到指定的 Block。

## 6.储存级别
&emsp;Spark 是基于内存进行计算的，但 RDD 的数据集不仅可以储存在内存中，还可以使用 RDD 类的 persist 方法和 cache 方法将数据缓存到 内存、磁盘等媒介中。我们看一下 persist 方法：
```scala
private def persist(newLevel: StorageLevel, allowOverride: Boolean): this.type = {
    // TODO: Handle changes of StorageLevel
    if (storageLevel != StorageLevel.NONE && newLevel != storageLevel && !allowOverride) {
      throw new UnsupportedOperationException(
        "Cannot change storage level of an RDD after it was already assigned a level")
    }
    // If this is the first time this RDD is marked for persisting, register it
    // with the SparkContext for cleanups and accounting. Do this only once.
    if (storageLevel == StorageLevel.NONE) {
      sc.cleaner.foreach(_.registerRDDForCleanup(this))
      sc.persistRDD(this)
    }
    storageLevel = newLevel
    this
  }
```
&emsp;当 RDD 第一次被计算时，persist 方法会根据参数 StorageLevel 设置特定的缓存策略，但它只适用于原本 StorageLevel 为 None 或者参数与原来 StorageLevel 一致的情况，即 RDD 不允许修改已经设置的 储存级别。
&emsp;注意，persist 方法只是修改了 RDD 的 storageLevel 属性，并没有触发真正的储存操作，storageLevel 属性会被当作参数传递给 BlockManager，由 BlockManager 负责具体的储存操作。RDD 的 getOrCompute 函数会接收 storageLevel 参数，接着就会调用 blockManager.getOrElseUpdate 方法，这时候才真正触发了 RDD 的存储操作。
&emsp;对于 cache 方法而言，它只是 persist 方法的一个特例，即 persist 方法参数为 StorageLevel.MEMORY_ONLY 的情况：
```scala
def persist(): this.type = persist(StorageLevel.MEMORY_ONLY)
def cache(): this.type = persist()
```
&emsp;接下来我们看一下储存级别，根据 _useDisk（使用磁盘）、_useMemory（使用内存）、_useOffHeap（使用堆外内存）、_deserialized（反序列化）、_replication（副本数量）五个参数的组合，Spark提供了12中储存级别的缓存策略：
```scala
//StorageLevel
  val NONE = new StorageLevel(false, false, false, false)
  val DISK_ONLY = new StorageLevel(true, false, false, false)  
  val DISK_ONLY_2 = new StorageLevel(true, false, false, false, 2)  
  val MEMORY_ONLY = new StorageLevel(false, true, false, true)  //只储存在内存
  val MEMORY_ONLY_2 = new StorageLevel(false, true, false, true, 2)  //只储存在内存，副本数量为2
  val MEMORY_ONLY_SER = new StorageLevel(false, true, false, false)   //序列化后储存在内存
  val MEMORY_ONLY_SER_2 = new StorageLevel(false, true, false, false, 2)
  val MEMORY_AND_DISK = new StorageLevel(true, true, false, true)
  val MEMORY_AND_DISK_2 = new StorageLevel(true, true, false, true, 2)
  val MEMORY_AND_DISK_SER = new StorageLevel(true, true, false, false)
  val MEMORY_AND_DISK_SER_2 = new StorageLevel(true, true, false, false, 2)
  val OFF_HEAP = new StorageLevel(true, true, true, false, 1)  //储存在外部分布式缓存系统中
```
&emsp;Spark 的默认储存级别是 MEMORY_ONLY，只缓存到内存并且以原生方式存（反序列化）一个副本。采用 SER 序列化的储存级别能够提高空间的利用率，MEMORY_AND_DISK 储存级别在内存足够的时候直接保存到内存，只有当内存不足的时候，才会储存到磁盘。如果想要确保高效的容错，除了依靠 RDD 的 Lineage 重新计算，还可以采用 replication 大于1的储存策略。


## 7.总结
&emsp;本文首先介绍了 Spark 储存系统的整体架构以及各个组件之间的关系，接着介绍了 Storage 通信层和储存层的实现原理，还分析了 RDD 的 partition 与 Storage 中 Block 一对一的关系，对 Spark 的储存系统整体上有了更好的把握。


&nbsp;
&nbsp;
>本文作者：ZJP
版权声明：本文为博主原创文章，遵循 [CC 4.0 BY-SA](http://creativecommons.org/licenses/by-sa/4.0/) 版权协议，转载请附上原文出处链接和本声明。
本文链接：http://zhoujiapeng.top/Spark/spark-storage/