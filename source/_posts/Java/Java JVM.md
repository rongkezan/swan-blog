---
title: Java JVM
date: {{ date }}
categories:
- Java
---

## 对象和Class

### 对象的创建过程

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210111220751109.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

1. class loading

2. class linking (verification preparation resolution)

3. class initializing：静态变量赋值，执行静态语句块

4. 申请内存对象

5. 成员变量赋值

6. 调用构造方法
   1. 成员变量顺序赋初始值
   2. 执行构造方法语句

### 对象在内存中的存储布局

一个Object对象占16个字节 = markword + class pointer + padding = 8 + 4 + 4 = 16

#### 普通对象

- 对象头
  - markword：锁信息、HashCode、GC信息，8 bytes（64位JDK）
  - class pointer：指向Class对象，4 bytes（压缩），8 bytes（不压缩）
- instance data：对象的属性（大小根据属性计算）
- padding：8的倍数

#### 数组对象

- 对象头
  - markword：锁信息、HashCode、GC信息，8 bytes（64位JDK）
  - class pointer：指向Class对象，4 bytes（压缩），8 bytes（不压缩）

- 数组长度：4字节
- 数组数据
- Padding：8的倍数

```sh
-XX:+UseCompressedClassPoiners # class pointer压缩，默认开启
```

### 对象头信息

对象头信息包括：对象的HashCode，锁标志位、GC标记（分代的年龄）等

markword 64位

![在这里插入图片描述](https://img-blog.csdnimg.cn/2021011712102443.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

### 对象定位

1. 句柄池：间接指针，一个指向对象，另一个指向了.class
2. 直接指针（HotSpot）：直接指向对象，对象再指向 .class

### 对象分配

首先new一个对象的时候尝试往栈上分配，如可以分配下，就分配到栈上，栈一弹出对象就没了。

如果对象过大，栈分配不下，直接分配到堆内存（老年代）。

如果对象不大，先进行线程本地分配，分配不下找伊甸区，然后进行GC的过程，年龄到了进入老年代。

### 对象生命周期

![在这里插入图片描述](https://img-blog.csdnimg.cn/de1cabf13ea549b991e3e3522f5d229e.png)

### Java从编码到执行

class被加载到内存之后，class的二进制文件加载到内存里，与此同时生成了class类的对象，该对象指向了二进制文件。class对象存在metaspace

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210111203500781.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

### ClassLoader

**类加载流程图：**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200131191909464.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)
**类加载器示意图：**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210111220915818.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

**ClassLoader**：负责加载class文件（class文件在文件开头有特定文件标识）

**各个类加载器的作用**

BootStrapClassLoader 引导类加载器：加载JVM自身需要的类，使用C++实现，负责加载`%JAVA_HOME%/jre/lib.jar`核心类库。

ExtensionClassLoader 扩展类加载器：负责加载%JAVA_HOME%/lib/ext目录下的类。

AppClassLoader 系统类加载器：负责加载系统类路径`java -classpath`或`-D java.class.path` 指定路径下的类库。

CustomClassLoader 自定义类加载器：继承ClassLoader重写findClass方法

**双亲委派**：JVM收到类加载请求，他会自底向上地去缓存中找这个类，找到了返回，没找到就把这个请求委派给父加载器（不是继承）去寻找，直到BootstrapClassLoader也没找到时，会自顶向下加载这个class，如果到最后还没加载成功，则会抛出异常 `ClassNotFoundException`

作用：沙箱安全，不让自己定义的类去勿扰JDK出厂自带的类

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210111223139570.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

## JVM 内存模型

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200131192106991.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

1. **程序计数器**：存放指令位置，虚拟机的运行就是循环取PC中的指令
2. **栈**：每个JVM都有自己私有的JVM栈，JVM栈用来存储栈帧
3. **本地方法栈**：存放native方法的地方。
4. **堆**：所有线程共享，存放对象实例，几乎所有的对象实例以及数组都在这里分配内存。
5. **方法区**：存储class二进制文件、类信息、常量、静态变量、运行时常量池
6. **直接内存**：JVM可以直接访问的内核空间的内存。

## 栈

栈：每个JVM都有自己私有的JVM栈，JVM栈用来存储Frame

Frame：每个方法对应一个 Frame

Frame 存放：Local Variable Table, Operated Stack, Dynamic Linking, Return Address

- Local Variable Table：byte、short、int、long、float、double、boolean、char、reference
- Dynamic Linking：A方法调用B方法，这个过程就叫动态链接
- Return Address：A方法调用B方法，B方法返回值的存放地址

案例：输出结果为8

```java
// 将8压入操作数栈，再将8拿出来赋值给i
int i = 8;
// 将8压入操作数栈，i加1，从操作数栈中弹出8赋值给i
i = i++;
// 输出最终结果 8
System.out.println(i);
```

**栈上分配** 

逃逸分析：逃逸分析的目的是判断对象的作用域是否有可能逃逸出函数体。 

标量替换：允许将对象打散分配在栈上，比如若一个对象拥有两个字段，会将这两个字段视作局部变量进行分配。 

## 堆

### 堆的基本概念

Java 中的堆是用来存储对象本身的以及数组（当然，数组引用是存放在 Java 栈中的）， 堆是被所有线程共享的，在 JVM 中只有一个堆。所有对象实例以及数组都要在堆上分配内存，但随着 JIT 发展，栈上分配，标量替换优化技术，在堆上分配变得不那么到绝对，只能在 server 模式下才能启用逃逸分析。

```java
// 左边存放在栈中，右边存放在堆中
Person person = new Person("张三", 22);
```

### JVM 内存分代模型

> 除了 Epsilon ZGC Shenandoah 之外的GC都是使用逻辑分代模型
>
> G1是逻辑分代，物理不分代
>
> 除上述 GC 模型之外不仅是逻辑分代，而且是物理分代

新生代 = Eden区 + 2 个 Suvivor区

1. YGC 回收之后，大多数对象被回收，活着的进入S0
2. 再次 YGC ，活着的对象 Eden + S0 -> S1
3. 再次 YGC， Eden + S1 -> S0
4. 年龄足够进入老年代
5. 分配担保：Suvivor区装不下直接进入老年代 

老年代：

1. 老年代满了就Full GC

永久代（1.7）/ 元空间（1.8）

1. 永久代 元空间 - Class
2. 永久代必须指定大小限制，元空间可以设置，也可以不设置，上限取决于物理内存
3. 字符串常量 1.7 - 永久代，1.8 - 堆
4. 永久代和元空间都是方法区的实现

图示

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200131193503949.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

### 实例化对象分配

1. 栈上分配

   线程私有小对象、无逃逸、支持标量替换

2. 线程本地分配 TLAB （Thread Local Alllocation Buffer）

   默认占用Eden的1%、多线程的时候不用竞争Eden就可以申请空间，提升效率、小对象

3. 老年代：大对象

## JVM调优

### 基本概念

吞吐量：用户代码时间 / ( 用户代码执行时间 + 垃圾回收时间 )

响应时间：STW（Stop The World）越短，响应时间越好

### JVM调优指令

```shell
# 查看所有指令
java -X
java -XX:+PrintFlagsFinal -version
# 模糊查询指令
java -XX:+PrintFlagsFinal -version | grep <command>
```

常用指令

```shell
-Xms<size>        					# 初始堆内存
-Xmx<size>        					# 最大堆内存
-Xss<size>        					# 每个线程的栈大小
-XX:MetaspaceSize=128m				# 初始元空间大小
-XX:MaxMetaspaceSize=256m			# 最大元空间大小，默认没有限制
-XX:MaxTenuringThreshold=15			# 老年代的大小
```
```shell
jinfo <pid>		# 打印虚拟机详细信息
jstat -gc <pid> <time>	# 打印gc信息，每<time>毫秒打印一次
jconsole	# java控制面板
```

### JVM调优场景

#### 系统CPU经常100%，如何调优

CPU 100% 一定有线程在占用系统资源

1. 找出哪个进程的 CPU 高（top）
2. 该进程的哪个线程 CPU 高（top - Hp [pid]）
3. 导出该线程的堆栈（jstack）
4. 查找哪个方法（栈帧）消耗时间 （jstack）
5. 工作线程占比高 | 垃圾回收线程占比高

```shell
# 查看Linux中哪个进程占资源
top
# 只列出java的进程
jps
# 查看这个<pid>的进程中哪个线程占资源
top -Hp <pid>
# 查看这个<pid>的线程堆栈
jstack <pid>
# 导出堆内存
jmap -heap <pid>
# 观察前20个占用内存比较大的类
jmap -histo <pid> | head -20
```

#### 如何监控JVM

```shell
# 格式模板
jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]
# 类加载统计
jstat -class <pid>
# 编译统计
jstat -compiler <pid>
```

#### 执行GC之后内存占用依然很高

进入 arthas 控制台dump一份内存快照

```sh
heapdump /tmp/dump.hprof
```

使用 `%JAVA_HOME%/bin/jvisualvm.exe` 装入 `dump.hprof` 观察哪些对象占用了大量的内存，也可以比较两次dump的区别。

但是对于内存特别大的系统，jmap执行期间会对进程产生很大影响，甚至卡顿。

解决方案：设定以下参数，OOM的时候会自动生成堆转储文件

```sh
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/usr/local/base
```

### GC 日志分析

执行命令

```shell
java -Xms20M -Xmx20M -XX:+PrintGCDetails -XX:+UseConcMarkSweepGC GCDemo
```

日志说明

```shell
[GC (Allocation Failure) [ParNew: 4544K->260K(6144K), 0.0012072 secs] 4544K->261K(19840K), 0.0012674 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 

ParNew：年轻代收集器
4544k->260k: 收集前后对比
(6144k): 整个年轻代容量
4544K->261K: 整个堆的情况
(19840K)：整个堆的大小
```

### G1 日志

```shell
[GC pause (G1 Evacuation pause)(young)(initial-mark), 0.0015790 secs]

G1 Evacuation pause: 年轻代复制存活对象
initial-mark: 混合回收阶段，这里是YGC混合老年代回收
```
