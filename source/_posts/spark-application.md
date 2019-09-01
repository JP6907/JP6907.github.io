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

> 未完待续

>本文作者：ZJP
版权声明：本文为博主原创文章，遵循 [CC 4.0 BY-SA](http://creativecommons.org/licenses/by-sa/4.0/) 版权协议，转载请附上原文出处链接和本声明。
本文链接：http://zhoujiapeng.top/article/spark-application/