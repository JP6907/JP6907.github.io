---
title: plan
catalog: true
subtitle:
header-img:
tags:
---

### 内存池模型


### shuffle机制

### shuffle  join

### 数据本地性 公平调度算法

### JVM 内存回收策略


Driver、Master

local模式下 Driver、Master、Worker运行在同一个 JVM 中

运行在独立的 JVM 中
standlone 模式下，StandaloneSchedulerBackend 会创建一个 Driver：StandaloneAppClient
Worker 创建 额外 Executor 进程去运行 task


scala　　闭包