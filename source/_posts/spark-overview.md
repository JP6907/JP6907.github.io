---
title: Spark Overview
catalog: true
date: 2019-08-31 10:06:40
subtitle: Spark集群架构
header-img: "/img/article_header/article_header.png"
tags: 
- spark
- 大数据
categories:
  - Spark
---



# Spark集群架构
> Spark版本：2.4.0


## 1. Spark运行架构
&emsp;Spark集群中的Spark Application的运行架构由两部分组成：包含SparkContext的Driver Program（驱动程序）和在Executor中执行计算的程序。Spark Application一般都是在集群上以独立的进程集合运行。
![spark运行架构图](https://github.com/JP6907/Pic/blob/master/spark/cluster-overview.png?raw=true)
&emsp;Spark有多种**运行模式**，比如standalone(spark自身单独的集群资源管理器), Mesos 和 YARN，这些资源管理器负责计算资源的管理和分配。


&emsp;根据 Spark Application是否在集群资源中运行，Spark Application的**运行方式**又可分为Cluster模式和Client模式，如果是Cluster模式，Spark会把Application代码发送到Executor，即Driver会在集群中某个节点运行；Client模式下Driver会在提交的机器上运行。  

&emsp;每个Application都有专属它的executor进行，该进程在application运行过程中一直在内存驻留，并以多线程的方式运行task。这种Application隔离机制无论是从调度角度看（每个driver调度它自己的任务），还是从运行角度（来自不同的Application的Task运行在不同的JVM中）都很有优势。这意味着Spark Application不能跨Application共享数据，除非将数据写入外部储存系统。

&emsp;Spark与集群资源管理器无关，只要能获取到Executor进程，并能保持相互通信就可以。

&emsp;提交Application的Client应该靠近Worker节点，最好是在同一个Rack（局域网）中，因为Application的Driver和Executor之间有大量的信息交换；如果想在远程集群运行，最好使用**RPC**(Remote Procedure Call Protocol，即远程过程调用协议)将Spark Aplication提交给集群(Cluster模式)，**不要远离Worker运行Driver！**


## 2. 集群管理模式
&emsp;Spark系统当前支持三种外部集群资源管理器：
- [Standalone](http://spark.apache.org/docs/2.4.0/spark-standalone.html) - Spark自身提供的集群资源管理器
- [Mesos](http://spark.apache.org/docs/2.4.0/running-on-mesos.html)
- [Yarn](http://spark.apache.org/docs/2.4.0/running-on-yarn.html)
- [Kubernetes](http://spark.apache.org/docs/2.4.0/running-on-kubernetes.html)


## 3. 监控
&emsp;每一个Driver程序都有一个WebUI监控器，一般是4040端口，可以看到有关Spark Application 运行的任务、程序和储存空间大小等信息，方便我们监控Spark Application在集群中的运行状态。
```
http://<driver-node>:4040
```
详细使用指南可见：http://spark.apache.org/docs/2.4.0/monitoring.html

## 4. 作业和任务调度
&emsp;Spark作业和任务调度是Spark的核心，Spark有多种运行模式，下面以Spark自身的Standalone模式，并且Driver运行在客户端的Client模式来介绍。

&emsp;下图是Spark的作业和任务调度系统整体概况：
![spark-schedule](https://github.com/JP6907/Pic/raw/master/spark/spark-schedule)
根据该图，具体的调度过程为：
1. Spark应用程序经过RDD的各种transform操作计算，最后通过RDD的action操作触发job，图中的join、groupby、和filter操作都是transform操作。
2. 提交之后首先根据RDD之间的依赖关系**构建DAG**(Directed Acyclic Graph)有向无环图，然后将DAG图提交给DAGScheduler进行解析，就进入了DAGScheduler阶段。
3. **DAGScheduler**是面向Stage的高级层的调度器，DAGScheduler把DAG拆分成很多个Tasks，每组的Tasks是一个Stage，解析时是以Shuffle为边界反向解析构建Stage，每次遇到Shuffle就会产生新的Stage，然后以一个个TaskSet（TaskSet等同于Stage，是对Stage的一次封装）的形式提交给底层调度器TaskScheduler。DAGScheduler还要记录哪些RDD被存入磁盘等物化动作，还需要监视因为Shuffle输出导致的失败，如果发现这个Stage失败，可能就要重新提交该Stage。
4. 一个**TaskScheduler**只为一个SparkContext实例服务，TaskScheduler接收来自DAGScheduler发送过来的TaskSet，TaskScheduler收到TaskSet后负责把TaskSet以Task的形式一个个发到集群Worker节点中的Executor中去运行。如果某个Task运行失败，TaskScheduler要负责重试，如果发现某个Task一直未运行完，就可能在其它节点启动同样的任务运行同一个Task，哪个任务先运行完就用哪个任务的结果。
5. **Executor**收到TaskScheduler发送过来的Task后，以多线程的方式运行，每一个线程负责一个Task。Task运行结束后要返回给DAGScheudler，不同类型的Task返回的方式不同。ShuffleMapTask返回的是一个MapStatue对象，而不是结果本身；ResultTask根据结果大小不同，返回的方式又可分为两类。


## 5. 相关术语
下面是一些常见的术语：
Term | Meaning
-|-
Application | 用户程序，包含Driver程序和Executor程序
Application jar | 在Application中的Jar包，包含用户程序的一些依赖jar包，注意不需要包含Hadoop或Spark的jar包
Driver program | Application运行main函数并创建SparkContext的进程
Cluster manager | 负责集群资源调度和管理
Deploy mode | 区分Driver运行的位置，cluster模式：driver在集群内部运行，client模式：driver在集群外部运行
Worker node | 集群中运行application程序的节点
Executor | Worker节点中运行application程序的进程，会运行tasks，将数据储存在内存或磁盘中。每个Application都有属于它自己的Executors
Task | Executor上运行任务的单位
Job | 用户程序中提交的作业，包含多个task
Stage | 每个job可以被分解为更小的task集，每个task集成为一个stage，stage之间存在一定的依赖关系



> 参考：http://spark.apache.org/docs/2.4.0/index.html

>本文作者：ZJP
版权声明：本文为博主原创文章，遵循 [CC 4.0 BY-SA](http://creativecommons.org/licenses/by-sa/4.0/) 版权协议，转载请附上原文出处链接和本声明。
本文链接：http://zhoujiapeng.top/Spark/spark-overview/