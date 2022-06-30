---
title: Arthas
date: {{ date }}
categories:
- 工具
---

## Arthas 可以解决的问题

1. 这个类从哪个 jar 包加载的？为什么会报各种类相关的 Exception？
2. 我改的代码为什么没有执行到？难道是我没 commit？分支搞错了？
3. 遇到问题无法在线上 debug，难道只能通过加日志再重新发布吗？
4. 线上遇到某个用户的数据处理有问题，但线上同样无法 debug，线下无法重现！
5. 是否有一个全局视角来查看系统的运行状况？
6. 有什么办法可以监控到JVM的实时运行状态？
7. 怎么快速定位应用的热点，生成火焰图？
8. 怎样直接从JVM内查找某个类的实例？

## Arthas 安装

下载`arthas-boot.jar`，然后用`java -jar`的方式启动：

```sh
curl -O https://arthas.aliyun.com/arthas-boot.jar
java -jar arthas-boot.jar
```

## Arthas 使用

> 命令列表 https://arthas.gitee.io/commands.html

### Attach到进程

启动 Arthas 后，会让你选择一个应用

```sh
$ java -jar arthas-boot.jar
* [1]: 35542
  [2]: 71560 math-game.jar
```

`math-game` 进程是第2个，则输入`2`，按`回车/enter`。Arthas会attach到目标进程，并输出日志

```sh
[INFO] Try to attach process 71560
[INFO] Attach process 71560 success.
[INFO] arthas-client connect 127.0.0.1 3658
  ,---.  ,------. ,--------.,--.  ,--.  ,---.   ,---.
 /  O  \ |  .--. ''--.  .--'|  '--'  | /  O  \ '   .-'
|  .-.  ||  '--'.'   |  |   |  .--.  ||  .-.  |`.  `-.
|  | |  ||  |\  \    |  |   |  |  |  ||  | |  |.-'    |
`--' `--'`--' '--'   `--'   `--'  `--'`--' `--'`-----'
 
wiki: https://arthas.aliyun.com/doc
version: 3.0.5.20181127201536
pid: 71560
time: 2018-11-28 19:16:24
```

### Dashboard

输入 `dashboard` ，按`回车/enter`，会展示当前进程的信息，按`ctrl+c`可以中断执行。

```sh
$ dashboard
ID     NAME                   GROUP          PRIORI STATE  %CPU    TIME   INTERRU DAEMON
17     pool-2-thread-1        system         5      WAITIN 67      0:0    false   false
27     Timer-for-arthas-dashb system         10     RUNNAB 32      0:0    false   true
11     AsyncAppender-Worker-a system         9      WAITIN 0       0:0    false   true
9      Attach Listener        system         9      RUNNAB 0       0:0    false   true
3      Finalizer              system         8      WAITIN 0       0:0    false   true
2      Reference Handler      system         10     WAITIN 0       0:0    false   true
4      Signal Dispatcher      system         9      RUNNAB 0       0:0    false   true
26     as-command-execute-dae system         10     TIMED_ 0       0:0    false   true
13     job-timeout            system         9      TIMED_ 0       0:0    false   true
1      main                   main           5      TIMED_ 0       0:0    false   false
14     nioEventLoopGroup-2-1  system         10     RUNNAB 0       0:0    false   false
18     nioEventLoopGroup-2-2  system         10     RUNNAB 0       0:0    false   false
23     nioEventLoopGroup-2-3  system         10     RUNNAB 0       0:0    false   false
15     nioEventLoopGroup-3-1  system         10     RUNNAB 0       0:0    false   false
Memory             used   total max    usage GC
heap               32M    155M  1820M  1.77% gc.ps_scavenge.count  4
ps_eden_space      14M    65M   672M   2.21% gc.ps_scavenge.time(m 166
ps_survivor_space  4M     5M    5M           s)
ps_old_gen         12M    85M   1365M  0.91% gc.ps_marksweep.count 0
nonheap            20M    23M   -1           gc.ps_marksweep.time( 0
code_cache         3M     5M    240M   1.32% ms)
Runtime
os.name                Mac OS X
os.version             10.13.4
java.version           1.8.0_162
java.home              /Library/Java/JavaVir
                       tualMachines/jdk1.8.0
                       _162.jdk/Contents/Hom
                       e/jre
```

### Watch

> 让你能方便的观察到指定函数的调用情况。
>
> 能观察到的范围为：`返回值`、`抛出异常`、`入参`，通过编写 OGNL 表达式进行对应变量的查看。

```sh
# 观察该方法的 入参、目标、返回值(默认可不传)
watch com.demo.service.impl.ProductionServiceImpl queryProductions -x 2
# 观察该方法的 入参
watch com.demo.service.impl.ProductionServiceImpl queryProductions -x 2 {param}
# 观察该方法的 入参、异常
watch com.demo.service.impl.ProductionServiceImpl queryProductions -x 2 {param,throwExp}
```

更多用法 https://arthas.gitee.io/watch.html

### Thread

> 查看当前线程信息，查看线程的堆栈

```sh
# 展示第一页线程信息
thread
# 展示所有线程信息
thread -all
# 展示当前最忙的前N个线程并打印堆栈
thread -n 3
# 展示指定线程的运行堆栈
thread [id]
# 找出当前阻塞其他线程的线程
thread -b
```

更多用法 https://arthas.gitee.io/thread.html

### Stack

> 输出当前方法被调用的调用路径

```sh
stack com.demo.service.impl.ProductionServiceImpl queryProductions
```

更多用法 https://arthas.gitee.io/stack.html