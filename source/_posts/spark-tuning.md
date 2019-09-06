---
title: Spark 性能调优
subtitle: Tuning Spark
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - spark
  - 大数据
categories:
  - Spark
date: 2019-09-06 15:21:46
---


# Spark 性能调优

&emsp;Spark 具有”内存计算”的天性，集群中的所有资源：CPU、网络带宽或者内存都会成为 Spark 程序的瓶颈。这里主要讲解两方面的 Sprk 性能优化：数据序列化和内存优化。

## 1.数据序列化
&emsp;序列化对于提高分布式程序的性能起到非常重要的作用。这里的数据序列化指的是网络传输的序列化，并不包含 RDD 的序列化储存（在第2节），数据序列化不但能提高网络性能还能减少内存使用。一个不好的序列化模式的速度非常慢或者序列化结果非常大。会极大降低计算速度。很多情况下，这是我们优化 Spark 应用的第一选择。我们可以通过 spark.serializer 参数来配置序列化模式，Spark 提供了两个序列化库：
- Java serialization：这是默认的序列化模式，Spark 采用 Java 的 ObjectOutputStream 序列化一个对象，该方式适用于所有实现了 java.io.Serializable 接口的类。Java 序列化非常灵活，但是速度比较慢，在某些情况下也比较大。

- Kryo serialization：Kryo 不但速度极快，而且产生的结果更为紧凑。kryo 的缺点是不支持所有类型，为了更好的性能，需要提前注册程序中所使用的类。
&emsp;我们可以通过调用 conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer") 来设置 Kryo 序列化模式。Kryo 不能成为默认方式的唯一原因是需要用户进行注册。但是，对于任何“网络密集型”（network-intensive）的应用，都建议采用这种方式。

&emsp;下面是注册自定义类型的方法：
```scala
val conf = new SparkConf().setMaster(...).setAppName(...)
conf.registerKryoClasses(Array(classOf[MyClass1], classOf[MyClass2]))
val sc = new SparkContext(conf)
```
&emsp;另外，如果不注册自定义的类，Kryo 仍然可以工作，但是需要为了每一个对象保存其对应的全类名，这是非常浪费的。更多详细介绍可以查看 [Kryo 文档](https://github.com/EsotericSoftware/kryo)。

## 2.内存优化
&emsp;内存优化有三方面的考虑：对象所占用的内存（可能我们希望将所有数据都加载到内存），访问对象的消耗以及内存回收（GC）所占用的开销。
&emsp;通常，Java对象的访问速度更快，但其占用的空间通常比其内部的属性数据大2-5倍，主要由于一下几方面原因：
- 每一个 Java 对象都包含一个对象头，对象头大约16字节；
- Java String 在实际字符串数据之外，还需要大约40字节的额外开销（String 保存在一个Char数组中，需要额外保存类似长度等其它数据）；同时，因为是 Unicode 编码，一个字符需要占用两个字节，所以，一个长度为10的字符串需要占用60字节；
- 通用的集合类，例如 HashMap、LinkedList 等都采用了链表数据结构，每一个条目不仅包含对象头，还包含一个指向下一个条目的指针（通常8字节）；
- 基本数据类型（Primitive Type）的集合通常保存为对应的包装类，例如 java.lang.Integer。

&emsp;下面分析如何估算对象所占用的内存以及如何进行内存优化---通过改变数据结构或采用序列化。

### 2.1 确定内存消耗
&emsp;确定对象所需内存大小的最好方法是创建一个 RDD，然后将其放入缓存，查看 WebUI 的 Storage 页面，可以看到 RDD 的内存占用情况，或者可以查看 Driver 的日志：
```scala
    val conf = new SparkConf().setAppName("test").setMaster("local") 
    val sc = new SparkContext(conf)
    sc.setLogLevel("info") //修改日志级别

    val value = sc.parallelize(Array(1,2,3,4))
    value.cache()
    value.count()   //触发action

    sc.stop()
```
&emsp;注意，如果是在ide中运行，输出日志级别默认是 warn，需要设置成 info 才能看到内存占用情况，日志信息如下：
```
INFO [dispatcher-event-loop-1] - Added rdd_0_0 in memory on [ip]:41353 (size: 32.0 B, free: 871.8 MB)
```
&emsp;表明 rdd_0_0 这个数据块 消耗了32B内存。


### 2.2 优化数据结构
&emsp;减少内存使用的第一个方法是避免使用一些增加额外开销的 Java 特性，例如基于指针的数据结构以对对象进行再包装等：
- 使用对象数组以及类型（primitive type）数组代替 Java 或 Scala 集合类（Collection Class）。[fastutil](http://fastutil.di.unimi.it/) 库为原始数据类型提供了非常方便的集合类，且兼容 Java 标准类库；
- 尽可能避免采用含有很多小对象和指针的嵌套数据结构；
- 考虑采用数字ID或者枚举类型代替 String 类型的主键；
- 如果内存小于32G，可以设置 JVM 参数 -XX:+UseCompressedOops 以便将8字节指针修改成4字节。我们可以将这些选项添加到 [Spark-env.sh](http://spark.apache.org/docs/2.4.0/configuration.html#environment-variables) 中。



### 2.3 序列化RDD储存
&emsp;如果经过上述优化，对象还是太大以至于不能有效存放，还有一个减少内存使用的简单方法---序列化。采用 [RDD persistence API](http://spark.apache.org/docs/2.4.0/rdd-programming-guide.html#rdd-persistence)的序列化储存级别，例如 MEMORY_ONLY_SER，这会将 RDD 的每一个分区都保存为 byte 数组。序列化带来的唯一缺点是会降低访问速度，因为需要将对象反序列化。序列化方式建议采用 [Kryo](https://github.com/EsotericSoftware/kryo)。

### 2.4 优化内存回收（GC）
&emsp;如果 Spark 程序产生大量的 RDD 数据，JVM内存回收就可能成为问题。JVM需要跟踪所有的Java对象以确定哪些对象不再使用。需要注意的一点是：内存回收的代价和对象的数量正相关。因此，使用对象数量更少的数据结构（使用int数组而不是LinkedList）能显著降低这种消耗。
&emsp;task　的工作内存和缓存在节点的 RDD 之间会相互影响，这种影响也会带来内存回收问题。下面讨论如何为 RDD 分配空间以便减少这种影响。
1. 估算内存回收的影响
&emsp;优化内存回收的第一步是获取一些统计信息，包括内存回收频率、内存回收耗费的时间等。我们可以把参数 -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCTimeStamps 添加到JavaOption　中，详细配置方法可以参考[configuration guide](http://spark.apache.org/docs/2.4.0/configuration.html#Dynamically-Loading-Spark-Properties) 。设置之后，我们可以在日志中看到每一次内存回收的信息。注意，这些日志是保存在集群的 Worker 节点，而不是 Driver。

2. 优化缓存大小
&emsp;用多大的内存来缓存RDD是内存回收一个非常重要的配置参数，默认情况下，Spark采用运行内存(spark.executor.memory)60%的空间来进行RDD缓存。这表明，在 task 运行期间，有40%的内存可以用来进行对象创建。
&emsp;如果task运行速度变慢且JVM频繁进行内存回收，或者内存空间不足，那么降低缓存大小设置减少内存消耗，提高可用的运行内存大小。缓存占用大小比例可以通过参数 spark.storage.memoryFraction 来设置。结合序列化缓存，使用较小缓存足够解决内存回收大部分问题。

3. 内存回收高级优化

&emsp;为了进一步优化内存回收，我们需要了解JVM内存管理的一些基本知识。
   - (1) Java堆空间分为两部分，新生代和老年代，新生代用于保存生命周期较短的对象，老年代用于保存生命周期较长的对象；
   - (2) 新生代进一步分为三部分：一块较大的 Eden 空间和两块较小的 Survivor 空间，Eden 空间和 Survivor 空间默认是8:1；
   - (3) 内存回收过程可以简要描述为：每次使用 Eden 和其中一块 Survivor 空间。如果 Eden 空间满则执行 minor GC(新生代GC),然后将当前 Eden 和 Survivor 空间中存活的对象复制到另外一块 Survivor 空间中。但 Survivor 空间不够用时，那么会将存活对象复制到老年代空间中。最终。如果老年代空间也不够用时，则执行 full GC。

&nbsp;
&emsp;Spark 内存回收优化的目标是确保只有长时间存活的RDD才能保存到老年代区域，同时，新生代区域足够大以保存生命周期较短的对象。这样，在任务执行期间可以避免执行full GC。下面是几个有用的步骤：
   - (1) 收集GC信息检查内存回收是否太过频繁。如果在任务结束之前执行了很多次 full GC，则说明任务执行的内存空间不足；
   - (2) 如果发现老年代消耗殆尽，那么可以减少用于缓存的内存空间，通过配置 spark.storage.memoryFraction 参数。通过减少缓存对象来提高执行速度是非常值得的；
   - (3) 如果有过多的 minor GC 而不是 full GC，那么可以为 Eden 空间分配更大的内存。可以为 Eden 分配大于 task 执行所需要的内存空间。如果 Eden 的大小为E，那么可以通过参数 -Xmn=4/3*E 来设置新生代的大小。（4/3是考虑到 survivor 所需要的空间）；
   - (4) 尝试使用 G1 垃圾回收器，通过配置 -XX:+UseG1GC 参数完成。当垃圾回收是瓶颈的时候，在有些情况下 G1 垃圾回收器能够提高性能。注意，如果 Executors 需要大堆内存时，需要设置 -XX:G1HeapRegionSize 参数来提高 G1 的 region 大小；
   - (5) 举一个例子，如果任务从 HDFS 读取数据，那么任务需要的内存空间可以从读取的 block 数量估算出来。解压后的 block 通常是解压前的2-3倍，所以，当我们需要同时执行3个或4个任务，block 的大小为64M，那么 Eden 的大小为 4*3*64M。


## 3.其他优化方法

### 3.1 并行度
&emsp;我们需要为每一个操作都设置足够高的并行度，才能有效利用集群资源。通常来讲，在集群中，建议为每一个CPU核分配2-3个task。

### 3.2 Reduce Task的内存使用
&emsp;有时候我们会碰到 OutOfMemoryError 错误，这不是因为我们的 RDD 不能加载到内存，而是因为 task 执行需要的数据集过大，例如 执行 groupByKey 操作的 reduce task。Spark 的 shuffle 操作（比如sortByKey, groupByKey, reduceByKey, join, etc），为了完成分组需要为每一个 task 创建 hash 表，这个 hash 表可能非常大。最简单的方法就是增加并行度，这样对于每一个 task 的输入会变得更小。Spark 能够非常有效地支持短时间任务（例如200ms），因为它能够在很多个 task 之间复用 Executor JVM，这样能够减少 task 的启动消耗。所以，我们能够放心地使任务的并行度大于集群的CPU核数。

### 3.3 广播 “大变量”
&emsp;广播变量允许用户保留一个只读的变量，缓存在每一台机器上，而不用在任务（Task）之间传递变量。详细介绍可以参考文章[《Spark 共享变量》](http://zhoujiapeng.top/Spark/spark-share-variable/)。广播变量可以有效减少每一个 task 的大小，以及在集群中启动作业的消耗。如果 task 需要用到 driver 端程序中比较大的对象（例如静态查找表），可以考虑将其变成广播变量。Spark 会在 master 打印每一个 task 序列化后的大小，我们可以通过它来检查任务是否过于庞大。通常来说，超过20KB的任务都是值得优化的。

### 3.4 数据本地性
&emsp;数据本地性对于 Spark 任务执行的性能有显著的影响。如果数据和操作的代码都在同一个节点的话，计算会非常块。但是如果数据和代码是分离的，那么必须有一方必须移动到另外一方。通常来讲，传输序列化后的代码肯定比传输数据要快得多。Spark在此原则上建立数据本地性的调度策略，简单来说，就是尽可能将 task 分配到拥有所需要数据的节点上。
&emsp;数据本地性的原理在这里不多加介绍，详细介绍可以关注我后续会编写的文章《Spark 任务调度 之 数据本地性》。


## 4.总结
&emsp;最后总结一下 Spark 程序优化需要关注的几个关键点---最主要的是数据序列化和内存优化。对于大多数情况而言，采用 Kryo 序列化能够解决性能有关的大部分问题。


>参考： http://spark.apache.org/docs/2.4.0/tuning.html

&nbsp;
&nbsp;
>本文作者：ZJP
版权声明：本文为博主原创文章，遵循 [CC 4.0 BY-SA](http://creativecommons.org/licenses/by-sa/4.0/) 版权协议，转载请附上原文出处链接和本声明。
本文链接：http://zhoujiapeng.top/Spark/spark-tuning/