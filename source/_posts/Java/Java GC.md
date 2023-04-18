---
title: Java GC
date: {{ date }}
categories:
- Java
---

## 概述

GC是什么（分代收集算法）

- 频繁收集Young区
- 较少收集Old区
- 基本不动元空间

普通GC(Minor GC)：只针对新生代区域的GC，指发生在新生代的垃圾收集动作，因为大部分Java对象存活率不高，所以Minor GC非常频繁，一般回收速度也比较快。

全局GC(Major GC / Full GC)：指发生在老年代的垃圾收集动作，出现了Major GC，经常会伴随至少一次的Minor GC，Major GC的速度一般要比Minor GC慢10倍以上。

## 相关名词

### Card Table

由于新生代的垃圾收集通常很频繁，如果老年代对象引用了新生代的对象，那么，需要跟踪从老年代到新生代的所有引用，效率非常低，所以 JVM 设计了 Card Table，如果一个老年代的 Card Table 中有对象指向新生代，就将它标记为 Dirty，下次扫描时，只需要扫描 Ditry Card，大大提升效率。

在结构上，Card Table 用 Bit Map 实现。

### CSet（Collection Set）

一组可以被回收的集合，在CSet中存活的数据会在GC的过程中被移动到另一个可用分区，CSet中的分区可以来自Eden、Survivor、Old区，CSet会占用不到整个堆空间1%的大小。简单来说，G1中需要被回收的Card的集合。

### RSet（Remembered Set）

记录了其它 Region 中的对象到本 Region 的引用

使得垃圾回收器不需要扫描整个堆栈来找到谁引用了当前分区中的对象，只需要扫描 RSet 即可

由于RSet的存在，那么每次给对象赋值引用的时候，就得做一些额外的操作：在RSet中做一些额外的记录，在GC中被称为写屏障（这个写屏障 不等于内存屏障）

## 定位垃圾

### 引用计数法

没有被引用的内存空间就是垃圾，需要被收集

缺点：计数器本身有消耗，较难处理循环引用

### 根可达性分析算法

通过一系列的名为"GC Root"的对象作为起点，从这些节点向下搜索，搜索所走过的路径称为引用链(Reference Chain)，当一个对象到GC Root没有任何引用链相连时，则该对象不可达，该对象是不可使用的，垃圾收集器将回收其所占的内存。

Java 可以做GC Root的对象：局部变量表、静态变量引用的对象、常量池引用的对象、Native方法引用的对象。

## GC 算法

### 复制算法（Copying）

> 没有碎片，浪费空间

YGC用的是复制算法，复制算法的基本思想是将内存分为两块，每次只用其中一块，当一块内存用完，就将还活着的对象复制到另一块上面，复制算法不会产生内存碎片。

原理：从根集合（GC Root）开始，通过Tracing从From中找到存活对象，拷贝到To中。From和To交换身份，下次内存分配从To开始

缺点：浪费了一半内存

### 标记清除（Mark-Sweep）

> 位置不连续，产生碎片，效率偏低（两遍扫描）

老年代一般由标记清除和标记整理混合实现

原理：算法分成标记和清除两个阶段。在标记阶段，collector从根对象开始进行遍历，对从根对象可以访问到的对象都打上一个标识，将其记录为可达对象。在清除阶段，collector对堆内存从头到尾进行线性的遍历，如果发现某个对象没有标记为可达对象，则就将其回收。

解释：程序运行期间，可用内存将被耗尽的时候,GC线程就会被触发并将程序暂停，随后将要回收的对象标记一遍，最终统一回收这些对象。

缺点：两次扫描，耗时严重，会产生内存碎片（清理出来的内存是不连续的）

### 标记清除压缩（Mark-Compact）

> 没有碎片，效率偏低（两遍扫描，指针需要调整）

第一步：标记清除
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200117210457686.png)
第二步：压缩，再次扫描，并往一端滑动存活对象
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200117210521309.png)

## 垃圾回收器

### 概览

<img src="https://img-blog.csdnimg.cn/2021011714275194.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

### 分代模型

> 将内存分为年轻代和老年代

- Serial + Serial Old：串行回收，单线程，会暂停所有用户线程

- Parallel Scavenge + Parallel Old（1.8默认使用）：并行回收，多线程，会暂停所有的用户线程

- ParNew + CMS：并发标记清除：用户线程和垃圾收集线程同时执行（并行或交替）

  - CMS是老年代的垃圾回收器

  - CMS四个阶段

    1. 初始标记（STW）：标记根对象（GC Root）
    2. 并发标记
    3. 重新标记（STW）：标记在并发标记时产生的新垃圾
    4. 并发清理：清理过程中会产生新垃圾，称为浮动垃圾，只能等待下一次CMS将其清理

  - CMS的问题：① 会产生碎片 ② 有浮动垃圾，当老年代碎片过多，换Serial Old上场

  - CMS问题解决方案之一：降低触发CMS的阈值，如果频繁发生SerialOld卡顿，应该调小阈值

    ```sh
    -XX:CMSInitiatingOccupancyFraction 70% # 内存空间降低到70%再进行回收，默认是68%
    ```

### 分区模型

> 将内存分为一个一个的小区域

- G1：将堆内存分割成不同的区域并发的对其进行垃圾回收，只在逻辑上分年轻代老年代，物理不分代

  G1可以在大多数情况下实现指定的GC暂停时间，同时还能保持较高的吞吐量。

  G1可以动态地调整新老年代的比例，调整的依据是 YGC 的暂停时间。比如指定的暂停时间是20ms，此时10个 region 中有6个Y区，但回收时间是30ms，那么G1会将6个Y区减少至5个或4个Y区直到暂停时间小于20ms为止。

  G1在对象太多的时候也会产生Full GC，如果产生Full GC，我们应该做：

  ```
  1. 扩内存
  2. 提高 CPU 性能
  3. 降低 MixedGC 触发的阈值，让MixedGC提早发生（默认45%)
  MixedGC（类似CMS）：初始标记STW，并发标记，最终标记STW，筛选回收STW（并行）
  ```

- ZGC

- Shenandoah

## 垃圾回收器算法

### 各个垃圾回收器使用的算法

CMS：三色标记 + Incremental Update

G1：三色标记 + SATB（Snapshot at the begining）

ZGC：Colored Pointers（颜色指针）

### 三色标记

三色标记把对象在逻辑上分成三种颜色，黑色对象不再被扫描，灰色对象会被再次扫描

- 白：未被标记的对象
- 灰：自身被标记，成员变量未被标记
- 黑：自身和成员变量均已标记完成

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210214141554379.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

### Incremental Update

当一个白色对象被一个黑色对象引用，将黑色重启标记为灰色，让重新扫描。

但是CMS在使用Increment Update的时候有一个致命问题是 漏标问题。

**漏标问题**：在并发标记的过程中，业务逻辑线程可能会把黑色的属性重新指向白色，如果不对黑色重新扫描，则会把白色对象当做没有新引用指向从而回收掉。

**举例：**

```
对象A有两个属性：1、2
M1（垃圾回收线程）正在标记A，已经标完属性1，正在标记属性2
M2（业务逻辑线程）把属性1指向白色对象D
M3（垃圾回收线程）把A标位灰色
M1（垃圾回收线程）认为所有属性标完，把A设为黑色，结果D被漏标
```

所以CMS的标记阶段，必须从头扫描一遍。

### SATB (Snapshot at the begining)

在起始的时候做一个快照，当灰色->白色引用消失时，要把这个引用推到GC的堆栈，下次扫描时拿到这个引用，由于有RSet的存在，不需要扫描整个堆区查找指向白色的引用，效率比较高。

## 引用

- 强引用：OOM也不回收
- 软引用：内存不足时回收
- 弱引用：只要执行GC就被回收
- 虚引用：跟没引用一样，可以用来管理堆外内存（直接内存），当对象被回收时，通过Queue可以检测到，然后清理堆外内存。堆外内存如何回收 -- Unsafe.freeMemory(address)

## 常用参数

### GC 常用参数

```shell
# 年轻代 最小堆 最大堆 栈空间
-Xmn -Xms -Xmx -Xss
# 使用TLAB，默认打开
-XX:+UseTLAB
# 打印TLAB的使用情况
-XX:+PrintTLAB
# 设置TLAB大小
-XX:TLABSize
# 禁用 System.gc()，System.gc()是Full GC
-XX:+DisableExplictGC
# 打印GC
-XX:+PrintGC
-XX:+PrintGCDetails
-XX:+PrintGCTimeStamps
-XX:+PrintHeapAtGC
-XX:+PrintGCTimeStamps
# 打印应用程序时间
-XX:+PrintGCApplicationConcurrentTime
# 打印暂停时长
-XX:+PrintGCApplicationStoppedTime
# 记录回收了多少种不同引用类型的引用
-XX:+PrintReferenceGC
# 可在程序运行时，打印虚拟机接受到的命令行显示参数
-XX:+PrintVMOptions
# GC的升代年龄
-XX:MaxTenuringThreshold
# 锁自旋次数
-XX:PreBlockSpin
# 热点代码检测参数，执行多少次会变成热点代码进行本地化的编译
-XX:ComplieThreshold
```

### Parallel 常用参数

```shell
# Survivor的比例
-XX:SurvivorRatio
# 多大的大对象会被直接分配到Old区
-XX:PreTenureSizeThreshold
# 并行收集器的线程数，同样适用于CMS，一般设为和CPU核数相同
-XX:+ParallelGCThreads
# 自动选择各区大小比例
-XX:+UseAdaptiveSizePolicy
```

### CMS 常用参数

```shell
# 使用CMS
-XX:+UseConcMarkSweepGC
# CMS线程数量
-XX:ParallelCMSThreads
# 使用多少比例的老年代后开始CMS收集，默认是68%
-XX:CMSInitiatingOccupancyFraction
# 在FGC时进行压缩(标记整理)
-XX:+UseCMSCompactAtFullCollection
# 多少次FGC后进行压缩
-XX:CMSFullGCsBeforeCompaction
# 停顿时间
-XX:MaxGCPauseMillis
# 回收永久代
-XX:+CMSClassUnloadingEnabled
# 达到什么比例时进行Perm回收
-XX:CMSInitiatingPermOccupancyFraction
# 设置GC时间占用程序运行时间的百分比
GCTimeRatio
```

### G1 常用参数

```shell
# 使用G1
-XX:+UseG1GC
# 建议最大停顿时间，GC会尝试调整Young区的块数来达到这个值
-XX:MaxGCPauseMillis
# 分区大小，建议逐渐增大该值，1 2 4 8 16 32
# 随着size增加，垃圾存活的时间更长，GC间隔更长，但每次GC的时间也会更长
-XX:+G1HeapRegionSize
# 新生代最小比例，默认5%
G1NewSizePercent
# 新生代最大比例，默认60%
G1MaxNewSizePercent
# GC时间建议比例，G1会根据这个值调整空间
GCTimeRatio
# 线程数量
ConcGCThreads
# 启动G1的堆空间占用比例
InitiatingHeapOccupancyPercent
```
