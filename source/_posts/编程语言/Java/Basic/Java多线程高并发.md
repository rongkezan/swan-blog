---
title: Java多线程高并发
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

## 锁

### 锁的分类

#### 按性质分类

**公平锁**：多个线程按照申请锁的顺序来获取锁。

**非公平锁**：多个线程获取锁的顺序并不是按照申请锁的顺序。

**乐观锁**：采用尝试更新，不断重新的方式更新数据。

**悲观锁**：对于同一个数据的并发操作，悲观锁采取加锁的形式。

**独享锁**：该锁一次只能被一个线程所持有。

**共享锁**：该锁可被多个线程所持有。

**互斥锁**：写锁。

**读写锁**：可以多人读，但只允许一人写。

**可重入锁**：在同一个线程的外层方法获取锁的时候，进入内层方法会自动获取锁。避免死锁。

**对象锁**：将sychronized放在普通同步方法中，sychronized同步监视器为普通对象

**全局锁**：将sychronized放在静态同步方法中，sychronized同步监视器为类对象

#### 按设计分类

**自旋锁**：采用循环的方式去尝试获取锁。

**自适应自旋锁**：循环多次发现等待时间过长，切换为阻塞状态。

**锁粗化**：如一个方法内加了多个锁，JVM认为没必要，于是将其合并为一个锁。

**锁消除**：JVM认为有些代码块无需加锁，于是删除了那个锁。

**偏向锁**：一段同步代码一直被一个线程访问，该线程会自动获得锁。

**轻量级锁**：当锁是偏向锁的时候，被另外线程访问，其它线程会通过自旋的形式尝试获取锁。

**重量级锁**：当锁是轻量级锁的时候，另一个线程自旋到一定次数未得到锁则进入阻塞。

**分段锁**：将数据分为多段，每次只给一段加锁。

### 锁概念

#### 锁升级

1. 无锁：程序不会有锁竞争
2. 偏向锁：经常只有一个线程加锁，markword 记录线程ID
3. 自旋锁：有线程来参与锁的竞争，但是获取锁的冲突时间很短
4. 重量级锁：自旋10次以后，升为重量级锁 - 去OS申请锁资源

什么时候用自旋什么时候用重量级锁？

执行时间长，线程多用重量级锁，否则用自旋。

#### 锁发生改变

1. 程序中如果出现异常，默认情况下锁会被释放

2. 如果锁对象发生改变，锁就会失效。

   解决方案： 锁对象加 `final`

```java
final Object obj = new Object()
```

### JUC和同步锁

#### Syncronized 实现细节

字节码层面：monitorenter monitorexit

JVM层面：C  C++ 调用了操作系统提供的同步机制

OS和硬件层面：x86是 `lock comxchg xxx`，lock是用来锁其它指令的

#### Sychronized and Lock

1. Sychronized：非公平，悲观，独享，互斥，可重入，重量级锁
2. Lock
   1. ReentrantLock：可公平，悲观，独享，互斥，可重入，重量级锁。
   2. ReentrantReadWriteLock：可公平，悲观，写独享，读共享，读写，可重入，重量级锁。

#### **Sychronized 和 ReentrantLock 的区别**

1. synchronized是关键字，Lock是Api
2. synchronized自动释放锁，Lock手动释放
3. synchronized不可以中断，ReentrantLock可中断(调用interrupt方法)
4. synchronized非公平锁，Lock两者皆可
5. synchronized只能随机或全部唤醒，Lock可以使用Condition精确唤醒

**Sychronized 和 ReentrantLock 的使用场景**

sychronized如果抢不到锁，就会一直等待

reentrantLock有tryLock机制，如果等待超时可以放弃等待

```java
if (lock.tryLock(3L, TimeUnit.SECONDS)){	// 3秒超时
    lock.lock();
    try{
        // 业务逻辑
    } finally {
        lock.unlock();
    }
} else {
    // 放弃等待后执行
}
```

#### ReadWriteLock - StampedLock

ReadLock：读锁，读的时候其它读线程依然可以进入

WriteLock，写锁，写的时候不允许其它线程进入

```java
static final ReadWriteLock readWriteLock = new ReentrantReadWriteLock();
static final Lock readLock = readWriteLock.readLock();
static final Lock writeLock = readWriteLock.writeLock();
```

#### CountDownLatch 

被减少到零之后才放行，否则阻塞等待。

定义一个CountDownLatch，有初始值，使用await阻塞线程，当减少到0时消除阻塞，类似于join但是更灵活

```java
CountDownLatch countDownLatch = new CountDownLatch(6);
for(int i = 1; i <= 6; i++){
    new Thread(() -> {
        countDownLatch.countDown();
    }, String.valueOf(i)).start();
}
countDownLatch.await();
System.out.println("解除门栓");
```

#### CyclicBarrier

先到的被阻塞，直到达到指定值时释放

await到指定个线程之后，放行

场景：某线程需等待其它线程执行完后才能执行

```java
CyclicBarrier cyclicBarrier = new CyclicBarrier(7, () -> System.out.println("释放通行"));
for(int i = 1; i <= 7; i++){
    final int tempInt = i;
    new Thread(() -> {
        System.out.println(Thread.currentThread().getName() + "\t 线程已到");
        try {
            cyclicBarrier.await();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }, String.valueOf(i)).start();
}
```

#### Phaser

CyclicBarrier升级版，使用arriveAndAwaitAdvance到达一个阶段的时候等待其它线程完成再向后执行

#### Semaphore

多共享资源的互斥使用，并发线程数的控制。

```java
// 在多个线程并发访问时，最多只有3个线程可以同时运行
// acquire:总量-1
// release:总量+1
Semaphore semaphore = new Semaphore(3);
new Thread(() -> {
    try {
        semaphore.acquire();
    } finally {
        semaphore.release();
    }
}).start();
```

#### Exchanger

两个线程间交换数据

```java
// 执行exchange()时阻塞线程
Exchanger<String> exchanger = new Exchanger<>();
new Thread(() -> {
    exchanger.exchange("1");
});
new Thread(() -> {
    exchanger.exchange("2");
});
```

## 线程等待和唤醒

### Object: wait, notify

1. 都需要在同步代码块中执行(synchronized)
2. 先wait再notify，等待中的线程才会被唤醒，否则无法唤醒
3. notify是随机唤醒一个线程
4. notify不释放锁，需要等待线程执行完或者线程中wait()才释放
5. notifyAll将所有线程唤醒，去争抢锁，但抢到锁的依旧只有一个线程

### Condition: await, signal

1. 都需要在同步代码块中执行
2. 先await再signal，等待中的线程才会被唤醒，否则无法唤醒
3. 可以精确的指定哪些线程被唤醒，即使用不同的condition加锁即可，condition的本质就是不同的等待队列

```java
Condition condition1 = lock.newCondition();
Condition condition2 = lock.newCondition();
```

### LockSupport: park, unpark

线程阻塞工具类，可以让线程在任意位置阻塞，阻塞后也有对应的唤醒方法，底层调用Unsafe的native方法

线程阻塞需要消耗Permit，Permit最多存在1个

当调用park方法时

- 如果有凭证，直接消耗掉这个凭证然后正常退出
- 如果无凭证，就阻塞等待凭证可用

当调用unpark方法时

- 增加一个凭证，但凭证最多有1个

## JMM (Java Memory Model)

是一组规范，可见性、原子性、有序性，定义了程序中各个变量的访问方式。

**解释**：线程创建时JVM会为其创建工作内存（线程私有），JMM规定所有变量存储在主内存（共享），但线程必须在工作内存中操作变量。具体流程：拷贝->操作->写回。各个工作内存存储主内存变量的复印件，不同线程无法互相访问，线程间通信必须通过主内存。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210121135743932.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

**JMM关于同步的规定**：

1. 线程解锁前，必须把共享变量的值刷新回主内存。
2. 线程加锁前，必须读取主内存的最新值到工作内存。
3. 加锁解锁是同一把锁。

## volatile

作用：保证可见性，禁止指令重排

### volatile指令重排实现

> JVM通过**内存屏障**（load和store指令组成）禁止特定类型的编译器重排序和指令重排序

为了实现 volatile 内存语义时，编译器在生成字节码时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。

对于编译器来说，发现一个最优布置来最小化插入屏障的总数几乎是不可能的，为此，JMM 采取了保守的策略。

- 在每个 volatile 写操作的前面插入一个 StoreStore 屏障。
- 在每个 volatile 写操作的后面插入一个 StoreLoad 屏障。
- 在每个 volatile 读操作的后面插入一个 LoadLoad 屏障。
- 在每个 volatile 读操作的后面插入一个 LoadStore 屏障。

| 内存屏障       | 说明                                                 |
| -------------- | ---------------------------------------------------- |
| StoreStore屏障 | 禁止上面的普通写和下面的volatile写重排序             |
| StoreLoad屏障  | 禁止上面的volatile写和下面可能有的volatile读写重排序 |
| LoadLoad屏障   | 禁止下面所有的普通读和上面的volatile读重排序         |
| LoadStore屏障  | 禁止下面所有的普通写操作和上面的volatile读重排序     |

### volatile可见性实现

> 缓存一致性协议、lock前缀指

#### lock前缀指令

lock 前缀会使处理器执行当前指令时产生一个 LOCK# 信号，会对总线进行锁定，其它 CPU 对内存的读写请求都会被阻塞，直到锁释放

通过 hsdis 和 jitwatch 工具可以得到编译后的汇编代码

lock前缀指令在多核处理器下会引发两件事情：

- 将当前处理器缓存行的数据写回到系统内存。
- 写回内存的操作会使在其他 CPU 里缓存了该内存地址的数据无效。

```java
//在 volatile 修饰的共享变量进行写操作的时候会多出 lock 前缀的指令
0x000000000295158c: lock cmpxchg %rdi,(%rdx)  
```

#### 缓存一致性协议

缓存是分段(line)的，一个段对应一块存储空间，称之为缓存行，它是 CPU 缓存中可分配的最小存储单元，大小 32 字节、64 字节、128 字节不等，这与 CPU 架构有关，通常来说是 64 字节。 LOCK# 因为锁总线效率太低，因此使用了多组缓存。 为了使其行为看起来如同一组缓存那样。因而设计了 缓存一致性协议。 缓存一致性协议有多种，但是日常处理的大多数计算机设备都属于 " 嗅探(snooping)" 协议。 所有内存的传输都发生在一条共享的总线上，而所有的处理器都能看到这条总线。 **缓存本身是独立的，但是内存是共享资源，所有的内存访问都要经过仲裁(同一个指令周期中，只有一个 CPU 缓存可以读写内存)。 CPU 缓存不仅仅在做内存传输的时候才与总线打交道，而是不停在嗅探总线上发生的数据交换，跟踪其他缓存在做什么。** 当一个缓存代表它所属的处理器去读写内存时，其它处理器都会得到通知，它们以此来使自己的缓存保持同步。 只要某个处理器写内存，其它处理器马上知道这块内存在它们的缓存段中已经失效。

### volatile有序性实现

> happends-before

happens-before 规则中有一条是 volatile 变量规则：对一个 volatile 域的写，happens-before 于任意后续对这个 volatile 域的读。

```java
//假设线程A执行writer方法，线程B执行reader方法
class VolatileExample {
    int a = 0;
    volatile boolean flag = false;
    
    public void writer() {
        a = 1;              // 1 线程A修改共享变量
        flag = true;        // 2 线程A写volatile变量
    } 
    
    public void reader() {
        if (flag) {         // 3 线程B读同一个volatile变量
        int i = a;          // 4 线程B读共享变量
        ……
        }
    }
}
```

根据 happens-before 规则，上面过程会建立 3 类 happens-before 关系。

- 根据程序次序规则：1 happens-before 2 且 3 happens-before 4。
- 根据 volatile 规则：2 happens-before 3。
- 根据 happens-before 的传递性规则：1 happens-before 4。

因为以上规则，当线程 A 将 volatile 变量 flag 更改为 true 后，线程 B 能够迅速感知。

### 为什么volatile不能实现原子性

没有原子性是因为底层代码一个++操作会被写成多行c++，这时候失去CPU分片就会值被改掉

### 单例模式中的DCL为什么要加volatile

由于指令重排，对象在半初始化状态的时候就赋值给这个变量了，即instance已经不再是null，第二个线程就直接拿来使用这个半初始化状态的对象。

### volatile引用对象

如果volatile修饰的是一个引用对象，那么引用对象内部的属性发生改变volatile是无法观察到的。

### 共享的long、double变量为什么要使用volatile

因为long和double两种数据类型的操作可分为高32位和低32位两部分，因此普通的long或double类型读/写可能不是原子的。因此，鼓励大家将共享的long和double变量设置为volatile类型，这样能保证任何情况下对long和double的单次读/写操作都具有原子性。

## CAS (Compare And Set)

**作用**：线程的期望值和物理内存真实值一样则修改，否则需要重新获得主物理内存的真实值，这个过程是原子的。

**原理**：Unsafe、自旋锁、乐观锁

- Unsafe：Java无法直接访问底层系统，可以基于Unsafe内部native方法可以像C的指针一样直接操作内存。

- 自旋锁：循环判断工作内存与主内存的值是否相等，如相等则返回。

**缺点**：循环时间长开销大、只能保证一个共享变量的原子操作、ABA问题。

### CAS是怎么保证原子性的

获取内存中的值，CAS比较不一致，则继续获取内存中的值，直到CAS成功为止

```java
// var1: 当前对象
// var2: 当前对象的内存偏移量地址
// var4: 增加的值
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        // 根据当前对象的内存偏移量获取当前对象的值
        var5 = this.getIntVolatile(var1, var2);
    // 如果CAS比较结果不一致，则继续循环，否则退出循环
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));
    return var5;
}
```

### 为什么CAS要比synchronized快

synchronized需要进行上下文切换，每一次线程进出Cpu就是一次上下文切换，而这一次切换大概需要3-5微秒，而Cpu执行一条执行大概只需要0.6纳秒，而CAS没有上下文切换的过程，那么效率就高。

### ABA问题

CAS只会判断最终的对象是否与期望的一致，但不会判断在这期间对象是否有改变，当这期间对象发生了改变，就会产生ABA问题，即虽然判断对象是同一个，但是其中的属性发生了改变。

解决方案：加版本号，如AtomicStampedReferece 

### 并发累加Long的三种方式

1. 加锁
2. AtomicLong：CAS
3. LongAdder：比AtomicXXX性能更高，内部维护了一个Cell数组，Cell 数组相当于一个分段的概念，把 AtomicXXX 中的一个值分成了多个值进行管理，当 CAS 更新失败时不再当前循环重试，而是尝试获取其他的资源锁，这样就降低了对于 AtomicXXX 中的单个资源的竞争，所以 LongAdder 的性能更高。代价是维护了 Cell 数组，也就意味着要占用更多的内存空间，以空间换时间，也是值得的。

### AtomicInteger

```java
AtomicInteger atomicInteger = new AtomicInteger(5);
System.out.println(atomicInteger.compareAndSet(5, 2019) + "\t current data:" + atomicInteger.get());
System.out.println(atomicInteger.compareAndSet(5, 1024) + "\t current data:" + atomicInteger.get());
atomicInteger.getAndIncrement();
```

### AtomicReference

```java
public class AtomicReferenceDemo {
    public static void main(String[] args) {
        User z3 = new User("z3", 22);
        User l4 = new User("l4", 25);

        AtomicReference<User> atomicReference = new AtomicReference<>();
        atomicReference.set(z3);
        System.out.println(atomicReference.compareAndSet(z3, l4) + "\t" + atomicReference.get().toString());
        System.out.println(atomicReference.compareAndSet(z3, l4) + "\t" + atomicReference.get().toString());
    }

    @Getter
    @ToString
    @AllArgsConstructor
    private static class User{
        String username;
        int age;
    }
}
```

## AQS (AbstractQueuedSynchronizer)

### 概念

是用来构建锁或者其它同步组件的抽象父类

- volatile state：用volatile修饰保证线程之间可见，state的值根据子类的具体实现来分配，如ReentrantLock加锁是1，不加锁是0；CountDownLatch设置为5，state就是5
- CAS：抢锁的时候使用CAS
- 双端队列：CLH变种的双端队列，Node中存放的是线程

### 抢锁流程

非公平锁上来直接抢锁，抢不到进入队列排队；公平锁判断队列是否有元素，没有的话得到锁，否则进入队列排队。

```java
/* 公平锁会通过 hasQueuedPredecessors 方法判断队列前是否有元素 有就排队 */
if (c == 0) {
    if (!hasQueuedPredecessors() && compareAndSetState(0, acquires)) {
        setExclusiveOwnerThread(current);
        return true;
    }
}
/* 非公平锁上来直接就抢锁 */
if (c == 0) {
    if (compareAndSetState(0, acquires)) {
        setExclusiveOwnerThread(current);
        return true;
    }
}
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201218203120948.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

### 源码说明

```java
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());	// 第一个线程抢到锁
    else
        acquire(1);	// 第二个线程及后续线程抢占
}
```

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&		// 尝试抢占，抢占成功返回true不进入队列
        // acquireQueued 尝试获取锁，若不成功则用LockSupport.park阻塞，直到被唤醒
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))	
        selfInterrupt();
}
```

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    // 第二次以后进入只需将Node加入队列末尾即可
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    // 第一次进入队列会先初始化，即生成一个空的头节点
    enq(node);
    return node;
}
```

```java
// 如果前置节点是头节点，那么第二个节点会尝试获得锁
if (p == head && tryAcquire(arg)){}
```

JDK9使用VarHandle：普通的原子操作，比反射快，直接操作二进制码

### AQS资源获取方式

AQS定义了两种资源获取的方式：独占、共享

独占：只有一个线程能访问运行，又根据顺序分为公平锁和非公平锁

共享：多个线程可同时访问运行，如Semaphore、CountDownLatch、CyclicBarrier

## ThreadLocal

> 线程独享的Map

**ThreadLocalMap中的Entry是弱引用**

1.  若是强引用，即使tl == null，但key的引用依然指向ThreadLocal对象，所以有内存泄露，而使用弱引用则不会。
2.  但还是有内存泄露的存在，ThreadLocalMap 是 Thread 的一个属性，生命周期跟 Thread 一致，当ThreadLocal被回收，key的值变成null，则导致整个value再也无法被访问到，因此依然存在内存泄露。所以ThreadLocal不用了需要调用 `remove()`回收

```java
ThreadLocal<M> tl = new ThreadLocal<>();
tl.set(new M());
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210114221056228.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

