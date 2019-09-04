---
title: spark-storage
catalog: true
subtitle:
header-img:
tags:
---

# Spark储存体系

## 1.储存体系架构
&emsp;简单来说，Spark 储存体系是各个 Driver 和 Executor 实例中的 BlockManager 所组成的，但从一个整体上看，Spark 储存体系包含很多组件，如图所示。
![storage-architecture](https://github.com/JP6907/Pic/blob/master/spark/storage-architecture.png?raw=true)

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
其中会先创建 BlockManagerMaster，在 registerOrLookupEndpoint 中根据是否为 Driver 节点来创建 BlockManagerMasterEndpoint 的 RpcEndpointRef：
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
![DiskBlockManager-dir](https://github.com/JP6907/Pic/blob/master/spark/DiskBlockManager-dir.png?raw=true)
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

### 5.3 Partition和Block的关系