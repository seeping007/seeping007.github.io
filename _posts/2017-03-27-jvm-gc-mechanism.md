---
layout: post
title: JVM探秘 | 垃圾回收机制
description: Java按代回收机制与常用垃圾回收类型
category: blog
---

## 垃圾收集是什么

> Automatic garbage collection is the process of looking at heap memory, identifying which objects are in use and which are not, and deleting the unused objects. An in use object, or a referenced object, means that some part of your program still maintains a pointer to that object. An unused object, or unreferenced object, is no longer referenced by any part of your program. So the memory used by an unreferenced object can be reclaimed.

- GC是一个自动的过程，它是一个低优先级的守护进程。
- GC收集的所谓垃圾指的是不可用对象，即不被程序中任何对象引用的对象，其所占据的堆空间可以被回收。



## 按代回收机制

![gc-generation](../images/gc-generation.png)

### Young Generation

- 新生代用来保存那些第一次被创建的对象，它被分为一个Eden空间和两个Survivor空间。
- 绝大多数新创建的对象会存放于Eden空间；Eden每执行一次Minor GC后，存活的对象会被移动到其中一个Survivor空间（暂且称为S1）并不断在此空间中堆积直至空间饱和；S1空间饱和后，还存活的对象会被移动到另一个Survivor空间（暂且称为S2），并清空S1空间；按此步骤重复几次后依然存活的对象会被移动到老年代。

### Old/Tenured Generation

- 老年代用来保存那些长期存活的对象，即从新生代GC中存活下来的对象。
- 老年代空间的Full GC事件基本上是在空间已满时发生，执行的过程根据GC类型的不同而不同。

### Metaspace

> In JDK 8, classes metadata is now stored in the **native heap** and this space is called **Metaspace**.

- Metaspace存放类加载信息、编译后的代码等，可以通过设置`-XX:MetaspaceSize`、`-XX:MaxMetaspaceSize`等参数来调节metaspace的大小。
- Java 8彻底将永久代移除出了Hotspot JVM，将其原有的数据迁移至Java Heap或Metaspace。主要原因有两个：一是，由于PermGen内存经常不够用或发生内存泄露，引发恼人的OOM；二是，移除PermGen可以促进HotSpot JVM与JRockit VM的融合，因为JRockit没有永久代。



## 垃圾收集算法

### 标记-清除算法（Mark-Sweep）

- 其分为“标记”和“清除”两个阶段：首先标记出所有需要回收的对象，然后统一回收所有被标记的对象。
- 主要不足：两个阶段的效率都不高；会产生大量不连续的内存碎片，进而导致之后在程序运行过程中需要分配较大对象时，无法找到足够的连续内存而不得不提前触发一次垃圾收集动作。

### 复制算法（Copying）

- 其将可用内存按容量划分为大小相等的两块，每次只使用其中的一块。当一块内存用完了，就将还存活着的对象复制到另外一块上面，然后把已使用过的内存空间一次清理掉。这样每次都是对整个半区进行回收，内存分配时无需考虑内存碎片等复杂情况，运行高效。
- 代价：将内存缩小为了原来的一半。

### 标记-压缩算法（Mark-Compact）

- 它是针对老年代的回收算法，标记过程和标记-清除算法一样，但后续步骤不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存。

### 分代收集算法（Generational Collection）

- 根据对象存活周期的不同将内存划分为几块，一般是把Java堆分为新生代和老年代，这样就可以根据各个年代的特点采用最适当的收集算法。



## 垃圾收集器

### Serial GC

> -XX:+UseSerialGC

- 串行回收，新生代复制算法、老年代标记-压缩算法。

### ParNew GC

> -XX:+UseParNewGC

- ParNew其实就是Serial的多线程版本，可以通过`-XX:ParallelGCThreads`限制线程数量。
- 新生代并行，老年代串行；新生代复制算法、老年代标记-压缩算法。

### Parallel GC

> -XX:+UseParallelGC

- Parallel也叫Parallel Scavenge，类似于ParNew，新生代复制算法、老年代标记-压缩算法。
- 它更关注系统的吞吐量，可以通过参数来打开自适应调节策略（`-XX:+UseAdaptiveSizePolicy`等）。

### Parallel Old GC

> -XX:+UseParallelOldGC

- Parallel Old是Parallel Scavenge的老年代版本，使用多线程和标记-压缩算法。

### CMS GC

> -XX:+UseConcMarkSweepGC

- CMS GC（Concurrent Mark Sweep）是一个老年代收集器，新生代使用Parallel Scavenge。 它是基于标记-清除算法实现的，运作过程分为4个步骤：初始标记（initial mark）、并发标记（concurrent mark）、重新标记（remark）、并发清除（concurrent sweep）。
- CMS GC也被称为低延迟GC，它是一种以获取最短回收停顿时间为目标的收集器，经常被用在对于响应时间要求十分苛刻的应用上。
- 缺点：比其他GC类型占用更多的内存和CPU；默认情况下不支持压缩步骤，容易产生大量空间碎片。若因为内存碎片过多而导致压缩任务不得不执行，那么stop-the-world的时间会比其他任何GC类型都长，所以需要考虑压缩任务的发生频率以及执行时间。

### G1 GC

- G1肩负的使命是在将来替换掉CMS。
- 与CMS相比，G1有以下特点：空间整合，采用标记-整理算法，不会产生内存空间碎片；可预测停顿，能让使用者明确指定在一个长度为N毫秒的时间片段内，消耗在垃圾收集上的时间不得超过N毫秒。
- 使用G1时，Java堆的内存布局与其他收集器有很大差别，它将整个Java堆划分为多个大小相等的独立区域，虽然还保留有新生代和老年代的概念，但新生代和老年代不再是物理隔阂了，它们都是一部分（可以不连续）区域的集合。



## 参考资料

- [Java Garbage Collection Basic][1]
- [Java SE HotSpot at a Glance][2]
- [《深入理解Java虚拟机:JVM高级特性与最佳实践（第2版）》][3]
- [JEP 122: Remove the Permanent Generation][4]

[1]: https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html
[2]: https://www.oracle.com/technetwork/java/javase/tech/index-jsp-136373.html
[3]: https://www.amazon.cn/%E5%9B%BE%E4%B9%A6/dp/B00D2ID4PK/ref=sr_1_1?ie=UTF8&qid=1490516490&sr=8-1&keywords=%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3java%E8%99%9A%E6%8B%9F%E6%9C%BA
[4]: http://openjdk.java.net/jeps/122