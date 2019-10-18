---
title: 深入理解Java虚拟机 之 内存分配与回收策略
catalog: true
subtitle: 内存分配与回收策略
header-img: /img/article_header/article_header.png
tags:
  - java
  - 编程语言
  - jvm
categories:
  - java
---

# 内存分配与回收策略
&emsp;Java技术体系中所提倡的自动内存管理最终可以归结于为了自动化的解决两个问题：**给对象分配内存**和**回收分配给对象的内存**。
对象的内存分配，往大的方向上讲，就是在堆上分配(但也可能经过JIT编译后被拆散为标量类型并间接地在栈上分配)，对象主要分配在新生代的Eden区上，如果启动了本地线程分配缓冲，将按线程优先在TLAB上分配。少数情况下也可能会直接分配在老年代中，分配的规则并不是百分之百固定的，其细节决定于当前使用的是哪种垃圾收集器组合，当然还有虚拟机中与内存相关的参数。
&emsp;本文介绍几条最普遍的内存分配规则。


## 1. 对象优先在Eden分配
&emsp;大多数情况下，对象在新生代Eden区中分配。当Eden区没有足够的空间进行分配时，虚拟机将发起一次Minor GC。
&emsp;注意区分一下以下两种GC：
- 新生代GC（Minor GC）：指发生在新生代的垃圾收集动作，因为Java对象大多具备朝生夕灭的特性，所以 Minor GC 非常频繁，一般回收速度也比较快。
- 老年代GC（Major GC/Full GC）：指发生在老年代的 GC，出现了 Major GC，经常会伴随至少一次 Minor GC（不绝对，Parallel Scavenge收集器的收集策略就有直接进行 Major GC 的策略选择过程）。Major GC 的速度一般会比 Minor GC 慢 10 倍以上。

## 2. 大对象直接进入老年代
&emsp;所谓的大对象是指，需要大量连续内存空间的Java对象，最典型的大对象就是那种很长的字符串及数组。**大对象对虚拟机的内存分配来说就是一个坏消息（比遇到一个大对象更加坏的消息就是遇到一群”朝生夕灭“的”短命的大对象“)**，所以写程序时应避免在频繁调用的方法中创建大对象），经常出现大对象容易导致内存还有不少空间时就提前出发垃圾回收以获取足够的连续空间来“安置”它们。
&emsp;虚拟机提供一个 -XX:PretenureSizeThreshold 参数，大于这个设置的对象会直接在老年代分配。这样做的目的是为了避免在Eden区和两个Survivor区之间发生大量的内存复制(新生代采用复制算法收集内存)。
&emsp;虚拟机参数配置示例：
```
“-XX:+PrintGCDetails -Xms20M -Xmx20M -Xmn10M -XX:SurvivorRatio=8 -XX:PretenureSizeThreshold=3145728”
```
- -XX:+PrintGCDetails：打印内存回收日志
- -Xms：Java堆大小
- -Xmx: Java堆最大可扩展大小
- -Xmn：给新生代分配的堆内存，剩下的分配给老年代
- -XX:SurvivorRatio：决定新生代 Survivor 区和 Eden 区的比例，8 表示Eden：Survivor为8：1
- -XX:PretenureSizeThreshold：大小超过这个阈值的对象直接在老年代分配

## 3. 长期存活的对象将进入老年代
&emsp;既然虚拟机采用了分代收集的思想来管理内存，那内存回收时就必须能识别哪些对象应放入新生代，那些应放入老年代中。为了做到这一点，虚拟机为每个对象定义了一个年龄（Age）计数器。如果对象在 Eden 出生并经过了一次 Minor GC 后仍然存活，并且能被 Survivor区域容纳，将被移动至Survivor空间中，并且age设置为1，对象在 Survivor 区中每“熬过”一次 Minor GC 后，age都会加1，当年龄达到一定程度（默认是15）时，就会晋升到老年代中。对象晋升到老年代的年龄阈值，可以通过参数  -XX:MaxTenuringThreshold 设置。
```
-Xms20M -Xmx20M -Xmn10M 
-XX:SurvivorRatio=8 
XX:+PrintGCDetails
-XX:+UseSerialGC
-XX:MaxTenuringThreshold=1
-XX:+PrintTenuringDistribution
```


## 4. 动态对象年龄判定
&emsp;为了能更好的适应不同程序的内存状况，虚拟机并不总是要求对象必须达到一定年龄才能晋升老年代，如果Survivor空间中相同年龄的对象大小超过Survivor空间的一半，年龄大于或者等于改年龄的对象就可以直接进入老年代中，无需等到特定要求的年龄。


## 5. 空间分配担保
&emsp;在发生Minor GC时，虚拟机会检测之前每次晋升到老年代的平均大小是否大于老年代中的剩余空间大小，如果大于，则改为直接进行一次Full GC。如果小于，则查看设置是否允许担保失败：如果允许，那先尝试进行一次 Minor GC，尽管这次 Minor GC 是有风险的；如果不允许，则要改为进行一次Full GC。
&emsp;新生代使用复制收集算法，但为了内存利用率，只使用其中一个Survivor空间来作为轮换备份，因此当出现大量对象在Minor GC后仍然存活的情况（最极端的就是内存回收后新生代中的所有对象都存活），Survivor空间无法容纳所有的存活对象，就需要老年代进行分配担保，把Survivor无法容纳的对象直接进入老年代。
&emsp;取平均值进行比较其实仍然是一种动态概率的手段，也就是说，如果某次Minor GC存活后的对象突增，远远高于平均值的话，依然会导致担保失败（Handle Promotion Failure），如果出现了担保失败，那就只好再失败后重新发起一次Full GC。虽然担保失败失败绕的圈子是最大的，但大部分情况下都还是会把 -XX:+HandlePromotionFailure 开关打开，避免 Full GC 过于频繁。
&emsp;需要注意的是，在 JDK6 Update 24 之后规则变为**只要老年代的连续空间大于新生代对象总大小或者历次晋升的平均大小就会进行 Minor GC，否则将进行 Full GC**。虽然源码中还定义了 HandlePromotionFailure 参数，但是在代码中已经不会再使用它了。
