---
title: Java锁
date: {{ date }}
sticky: 970
categories:
- Java
---

# Java 锁

## JMM (Java Memory Model)

是一组规范，可见性、原子性、有序性，定义了程序中各个变量的访问方式。

**解释**：线程创建时JVM会为其创建工作内存（线程私有），JMM规定所有变量存储在主内存（共享），但线程必须在工作内存中操作变量。具体流程：拷贝->操作->写回</。各个工作内存存储主内存变量的复印件，不同线程无法互相访问，线程间通信必须通过主内存。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210121135743932.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

**JMM关于同步的规定**：

1. 线程解锁前，必须把共享变量的值刷新回主内存。
2. 线程加锁前，必须读取主内存的最新值到工作内存。
3. 加锁解锁是同一把锁。

## volatile

### 作用

保证可见性，禁止指令重排

### 原理

缓存一致性协议。JMM模型里有8个指令完成数据的读写，通过其中load和store指令相互组成的4个内存屏障实现禁止指令重排序。

**字节码层面**：ACC_VOLATILE

**JVM层面**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210117102410887.png)

**JVM内存屏障**

LoadLoad：在Load2及后续读取操作的数据被访问前，保证Load1要读取的数据读取完毕。

StoreStore：在Store2及后续写入操作执行前，保证Store1的写入操作对其它处理器可见。

LoadStore：在Store2及后续写入操作刷出前，保证Load1要读取的数据被读取完毕。

StoreLoad：在Load2及后续所有读取操作执行前，保证Store1的写入对所有处理器可见。

**OS层面：硬件内存屏障、原子指令**

sfence：save | 在sfence指令前的写操作必须在sfence指令后的写操作前完成

lfence：load | 在lfence指令前的读操作必须在lfence指令后的读操作前完成

mfence：modify/mix | 在mfence指令前的读写操作必须在mfence指令后的读写操作前完成

原子指令：如x86上的lock前缀指令是一个Full Barrier，执行时会锁住内存子系统来确保执行顺序

**为什么volatile不能实现原子性**

没有原子性是因为底层代码一个++操作会被写成多行c++，这时候失去CPU分片就会值被改掉

**单例模式中的DCL为什么要加volatile**

由于指令重排，对象在半初始化状态的时候就赋值给这个变量了，即instance已经不再是null，第二个线程就直接拿来使用这个半初始化状态的对象。

**volatile引用对象**

如果volatile修饰的是一个引用对象，那么引用对象内部的属性发生改变volatile是无法观察到的。

## CAS (Compare And Set)

**作用**：线程的期望值和物理内存真实值一样则修改，否则需要重新获得主物理内存的真实值，这个过程是原子的。

**原理**：Unsafe、自旋锁、乐观锁

- Unsafe：Java无法直接访问底层系统，可以基于Unsafe内部native方法可以像C的指针一样直接操作内存。

- 自旋锁：循环判断工作内存与主内存的值是否相等，如相等则返回。

**缺点**：循环时间长开销大、只能保证一个共享变量的原子操作、ABA问题。

**CAS是怎么保证原子性的**

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

**为什么CAS要比synchronized快**

synchronized需要进行上下文切换，每一次线程进出Cpu就是一次上下文切换，而这一次切换大概需要3-5微秒，而Cpu执行一条执行大概只需要0.6纳秒，而CAS没有上下文切换的过程，那么效率就高。

**ABA问题**

CAS只会判断最终的对象是否与期望的一致，但不会判断在这期间对象是否有改变，当这期间对象发生了改变，就会产生ABA问题，即虽然判断对象是同一个，但是其中的属性发生了改变。

解决方案：加版本号。

**并发累加Long的三种方式**

1. 加锁
2. AtomicLong：CAS
3. LongAdder：分段锁，线程数量特别多的时候比Atomic更有优势

**AtomicInteger**

```java
AtomicInteger atomicInteger = new AtomicInteger(5);
System.out.println(atomicInteger.compareAndSet(5, 2019) + "\t current data:" + atomicInteger.get());
System.out.println(atomicInteger.compareAndSet(5, 1024) + "\t current data:" + atomicInteger.get());
atomicInteger.getAndIncrement();
```

**AtomicReference**

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

## 锁的分类

### 按性质分类

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

### 按照设计分类

**自旋锁**：采用循环的方式去尝试获取锁。

**自适应自旋锁**：循环多次发现等待时间过长，切换为阻塞状态。

**锁粗化**：如一个方法内加了多个锁，JVM认为没必要，于是将其合并为一个锁。

**锁消除**：JVM认为有些代码块无需加锁，于是删除了那个锁。

**偏向锁**：一段同步代码一直被一个线程访问，该线程会自动获得锁。

**轻量级锁**：当锁是偏向锁的时候，被另外线程访问，其它线程会通过自旋的形式尝试获取锁。

**重量级锁**：当锁是轻量级锁的时候，另一个线程自旋到一定次数未得到锁则进入阻塞。

**分段锁**：将数据分为多段，每次只给一段加锁。

## 锁概念

### 锁升级

1. 无锁：程序不会有锁竞争
2. 偏向锁：经常只有一个线程加锁，markword 记录线程ID
3. 自旋锁：有线程来参与锁的竞争，但是获取锁的冲突时间很短
4. 重量级锁：自旋10次以后，升为重量级锁 - 去OS申请锁资源

什么时候用自旋什么时候用重量级锁？

执行时间长，线程多用重量级锁，否则用自旋。

### 锁发生改变

1. 程序中如果出现异常，默认情况下锁会被释放

2. 如果锁对象发生改变，锁就会失效。

   解决方案： 锁对象加 `final`

```java
final Object obj = new Object()
```

## JUC和同步锁锁

### Syncronized 实现细节

字节码层面：monitorenter monitorexit

JVM层面：C  C++ 调用了操作系统提供的同步机制

OS和硬件层面：x86是 `lock comxchg xxx`，lock是用来锁其它指令的

### Sychronized and Lock

1. Sychronized：非公平，悲观，独享，互斥，可重入的重量级
2. Lock
   1. ReentrantLock：可公平，悲观，独享，互斥，可重入，重量级锁。
   2. ReentrantReadWriteLock：可公平，悲观，写独享，读共享，读写，可重入，重量级锁。

**Sychronized 和 ReentrantLock 的区别**

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

### ReadWriteLock - StampedLock

ReadLock：读锁，读的时候其它读线程依然可以进入

WriteLock，写锁，写的时候不允许其它线程进入

```java
static final ReadWriteLock readWriteLock = new ReentrantReadWriteLock();
static final Lock readLock = readWriteLock.readLock();
static final Lock writeLock = readWriteLock.writeLock();
```

### CountDownLatch 

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

### CyclicBarrier

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

### Phaser

CyclicBarrier升级版，使用arriveAndAwaitAdvance到达一个阶段的时候等待其它线程完成再向后执行

### Semaphore

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

### Exchanger

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

### LockSupport: pack, unpack

线程阻塞工具类，可以让线程在任意位置阻塞，阻塞后也有对应的唤醒方法，底层调用Unsafe的native方法

线程阻塞需要消耗Permit，Permit最多存在1个

当调用park方法时

- 如果有凭证，直接消耗掉这个凭证然后正常退出
- 如果无凭证，就阻塞等待凭证可用

当调用unpark方法时

- 增加一个凭证，但凭证最多有1个

## AQS (AbstractQueuedSynchronizer)

概念：是用来构建锁或者其它同步组件的抽象父类

**CAS + volatile state + 双端队列**

CAS：在往队列末端加线程的时候使用的是CAS

volatile：state变量用volatile修饰保证线程之间可见

state：根据子类的具体实现来分配，如ReentrantLock加锁,是1，不加锁是0；CountDownLatch设置为5，state就是5

双端队列：CLH变种的双端队列，Node中存放的是线程

**公平锁和非公平锁**

1. 非公平锁在调用 lock 后，首先就会调用 CAS 进行一次抢锁，如果这个时候恰巧锁没有被占用，那么直接就获取到锁返回了。
2. 非公平锁在 CAS 失败后，和公平锁一样都会进入到 tryAcquire 方法，在 tryAcquire 方法中，如果发现锁这个时候被释放了，非公平锁会直接 CAS 抢锁，但是公平锁会判断等待队列是否有线程处于等待状态，如果有则不去抢锁，乖乖排到后面。

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

源码说明

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

## ThreadLocal

> 线程独享的Map

**ThreadLocalMap中的Entry是弱引用**

1.  若是强引用，即使tl == null，但key的引用依然指向ThreadLocal对象，所以有内存泄露，而使用弱引用则不会。
2. 但还是有内存泄露的存在，ThreadLocalMap 是 Thread 的一个属性，生命周期跟 Thread 一致，当ThreadLocal被回收，key的值变成null，则导致整个value再也无法被访问到，因此依然存在内存泄露。所以ThreadLocal不用了需要调用 `remove()`回收

```java
ThreadLocal<M> tl = new ThreadLocal<>();
tl.set(new M());
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210114221056228.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)