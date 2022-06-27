---
title: Java线程
date: {{ date }}
categories:
- 编程语言
- Java
---

## 线程的概念

操作系统是如何切换线程的：Cpu会从内存里取出线程，线程内部状态是由线程栈来维护的

一个程序的不同分支

```java
// 顺序执行
new T1().run();
// 并行执行
new T1().start();
// 睡眠500毫秒
Thead.sleep(500);
// 让出线程,使线程进入等待队列，但也有可能再次被Cpu拿出来执行
Thread.yield();
// t2运行中调用t1.join()即执行t1线程，保证t1结束以后t2才能继续运行
Thread t1 = new Thread(() -> {
    System.out.println("t1");
}, "t1");
new Thread(() -> {
    try { t1.join(); } catch (InterruptedException e) { e.printStackTrace(); }
}, "t2");
```

线程状态迁移图

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210107211056470.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

## 线程的创建方式

### 继承Thead类

```java
class MyThread extends Thread{
    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + "\t 运行了...");
    }
}
// main
MyThread t = new MyThread();
t.start();
```

### 实现Runnable接口

普通写法

```java
class MyThread implements Runnable{
    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + "\t 运行了...");
    }
}
// main
MyThread t = new MyThread();
new Thread(t).start();
```

lambda写法

```java
new Thread(() -> {
	System.out.println(Thread.currentThread().getName() + "\t 运行了...");
}).start();
```

### 实现Callable接口

> 可以抛出异常，支持泛型的返回值

- Future：可以获得线程的执行结果

```java
FutureTask<Integer> futureTask = new FutureTask<>(() -> 1);
new Thread(futureTask).start();
Integer result = futureTask.get();
System.out.println("结果:" + result);
```

- CompletableFuture

  使用`Future`获得异步执行结果时，要么调用阻塞方法`get()`，要么轮询看`isDone()`是否为`true`，这两种方法都不是很好，因为主线程也会被迫等待。

  从Java 8开始引入了`CompletableFuture`，它针对`Future`做了改进，可以传入回调对象，当异步任务完成或者发生异常时，自动调用回调对象的回调方法。

  `CompletableFuture`更强大的功能是，多个`CompletableFuture`可以串行执行

```java
// 创建异步执行任务
CompletableFuture<Integer> task1 = CompletableFuture.supplyAsync(() -> 1);
CompletableFuture<Integer> task2 = task1.thenApplyAsync(o -> o + 1);

// 如果执行成功
task2.thenAccept(res -> System.out.println("最终结果:" + res));
// 如果执行异常
task2.exceptionally(e -> {
  System.out.println("Failed: " + e.getMessage());
  return null;
});
// 主线程不要立刻结束，否则CompletableFuture默认使用的线程池会立刻关闭
Thread.sleep(200);
```

### 使用线程池

```java
ScheduledExecutorService pool = Executors.newScheduledThreadPool(5);
ScheduledFuture<Integer> result = pool.schedule(() -> {
    int num = new Random().nextInt(100);
    System.out.println(Thread.currentThread().getName() + "\t" + num);
    return num;
}, 100, TimeUnit.MILLISECONDS);
System.out.println(result.get());
pool.shutdown();
```

## 线程池详解

### 线程池种类

- newFixedThreadPool（固定大小的线程池）
- newSingleThreadExecutor（单线程线程池）
- newCachedThreadPool（可缓存线程的线程池）用于并发执行大量短期的小任务。
- newScheduledThreadPool：用于需要多个后台线程执行周期任务，同时需要限制线程数量的场景。

### 线程池参数

1. corePoolSize: 线程池中的常驻核心线程数，即使空闲也不归还。
2. maximumPoolSize: 线程池能够容纳同时执行的最大线程数，空闲了会归还给操作系统。
3. keepAliveTime: 多余的空闲线程存活时间。
4. unit: keepAliveTime的单位。
5. workQueue: 任务队列，被提交但尚未被执行的任务，一般使用阻塞队列。
6. threadFactory: 表示生成线程池中工作线程的线程工厂，用于创建线程，一般默认即可。
7. handler: 拒绝策略，表示当队列满了并且工作线程大于等于线程的最大线程数时如何来拒绝请求执行的runnable策略。

### 线程池底层工作原理

- 在创建了线程池后，等待提交过来的任务请求
- 当调用execute()方法添加一个请求任务时，线程池会做如下判断
  - 如果正在运行的线程数量小于corePoolSize，那么马上创建线程运行任务
  - 如果正在运行的线程大于等于corePoolSize，那么将这个任务放入队列
  - 如果这时队列满了且正在运行的线程数量小于maximumPoolSize，那么要创建非核心线程立刻运行这个任务
  - 如果队列满了且正在运行的线程数大于等于maximumPoolSize，那么线程池会启动拒绝策略

- 当一个线程完成任务时，他会从队列中取下一个任务来执行
- 当一个线程无事可做超过keepAliveTime时，线程会判断：
  - 如果当线程数大于corePoolSize，那么这个线程就被停掉
  - 线程池的所有任务完成后最终会收缩到corePoreSize

###  拒绝策略

定义：等待队列和max线程数都满了，那么就需要启用拒绝策略处理这个问题。

- AbortPolicy(默认)：直接抛出RejectedExecutionException异常
- CallerRunsPolicy：既不会抛弃任务，也不会抛出异常，而是把某些任务回退给调用者
- DiscardOldestPolicy：抛弃队列中等待最久的任务，然后把当前任务加入队列中尝试再次提交当前任务
- DiscardPolicy：直接丢弃任务，不予任何处理也不抛出异常
- 自定义Policy：实现 `RejectedExecutionHandler` 接口

### 自定义线程池

#### ThreadPoolExecutor

```java
public static void main(String[] args) {
  ExecutorService pool = new ThreadPoolExecutor(
    2, 5, 1L, TimeUnit.SECONDS, new LinkedBlockingQueue<Runnable>(3),
    new MyThreadFactory("myPool"), new ThreadPoolExecutor.CallerRunsPolicy());
  try {
    for (int i = 0; i < 10; i++) {
      final int fi = i;
      pool.execute(() -> {
        System.out.println(Thread.currentThread().getName() + "\t" + fi);
      });
    }
  } catch (Exception e) {
    e.printStackTrace();
  } finally {
    pool.shutdown();
  }
}

static class MyThreadFactory implements ThreadFactory {
  private static final AtomicInteger poolNumber = new AtomicInteger(1);
  private final ThreadGroup group;
  private final AtomicInteger threadNumber = new AtomicInteger(1);
  private final String namePrefix;

  MyThreadFactory(String prefix) {
    SecurityManager s = System.getSecurityManager();
    group = (s != null) ? s.getThreadGroup() :
    Thread.currentThread().getThreadGroup();
    namePrefix = prefix + "-" +
      poolNumber.getAndIncrement() +
      "-thread-";
  }

  @Override
  public Thread newThread(Runnable r) {
    Thread t = new Thread(group, r,
                          namePrefix + threadNumber.getAndIncrement(),
                          0);
    if (t.isDaemon())
      t.setDaemon(false);
    if (t.getPriority() != Thread.NORM_PRIORITY)
      t.setPriority(Thread.NORM_PRIORITY);
    return t;
  }
}
```

#### ThreadPoolTaskExecutor

> Spring 为我们提供的线程池类

```java
@Bean("taskExector")
public Executor taskExector() {
  ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
  int i = Runtime.getRuntime().availableProcessors();	//获取到服务器的cpu内核
  executor.setCorePoolSize(5);	//核心池大小
  executor.setMaxPoolSize(100);	//最大线程数
  executor.setQueueCapacity(1000);	//队列程度
  executor.setKeepAliveSeconds(1000);	//线程空闲时间
  executor.setThreadNamePrefix("task-asyn");/	/线程前缀名称
  executor.setRejectedExecutionHandler(new ThreadPoolExecutor.AbortPolicy());	//配置拒绝策略
  return executor;
}
```

```java
@Resource(name="taskExecutor")
ThreadPoolTaskExecutor taskExecutor;
```

### 如何合理配置线程池

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210116163921272.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

- Cpu密集型(Cpu一直运行)：Cpu核数+1个线程的线程池
- IO密集型(需要不断取数据)：
  - IO密集型并不是一直在执行任务，配置尽可能多的线程，如Cpu核数 * 2
  - Cpu核数 / (1 - 阻塞系数(0.8~0.9))	例如8核Cpu：8 / (1 - 0.9) = 80个线程数

