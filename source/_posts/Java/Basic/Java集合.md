---
title: Java集合
date: {{ date }}
categories:
- Java
---

# Java 集合

## Collection

### List

> ArrayList、LinkedList、Vector、Stack、CopyOnWriteArrayList

| 名称       | 特点                 | get(index) | add(E) | add(index, E) | remove(E) |
| ---------- | -------------------- | ---------- | ------ | ------------- | --------- |
| ArrayList  | 高效，线程不安全     | O(1)       | O(1)   | O(n)          | O(n)      |
| LinkedList | 删除更高效，查询低效 | O(n)       | O(1)   | O(n)          | O(1)      |
| Vector     | 低效，线程安全       | O(1)       | O(1)   | O(n)          | O(n)      |

#### ArrayList

1. 底层是数组
2. 默认装Object
3. 初始为10，(Jdk8之后默认添加数据的时候才开始给默认长度)。
4. 每次扩容是原长度的一半（取整）：第一次扩到15，第二次22
5. 扩容方式：Arrays.copyOf，默认把原数组复制到新数组
6. 不是线程安全的

#### LinkedList

1. 底层是双向链表
2. 链表删除和增加快，查询和修改慢
3. 实现了Queue接口，所以还提供了offer(), peek(), poll()等方法

#### CopyOnWriteArrayList

写时加锁，读时不加锁，复制一个新的数组，把新数组指向原来的数组

适用于读多写少的场景

### Set

> LinkedHashSet、TreeSet、EnumSet、CopyOnWriteArraySet、ConcurrentSkipListSet

| 名称          | 特点                         | add(E)  | remove(E) | contains(E) |
| ------------- | ---------------------------- | ------- | --------- | ----------- |
| HashSet       | 线程不安全，可存储null值     | O(1)    | O(1)      | O(1)        |
| LinkedHashSet | 查询时有序 (存储还是无序)    | O(logn) | O(logn)   | O(logn)     |
| TreeSet       | 可根据指定值排序(基于红黑树) | O(1)    | O(1)      | O(1)        |

#### HashSet

1. 底层是HashMap
2. 添加过程

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200209154216578.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

### Queue

在多线程的情况下，多考虑使用Queue

#### Deque 双端队列

> ArrayDeque、BlockingDeque、LinkedBlockingDeque

#### BlokingQueue

> ArrayBlockingQueue、ProrityBlockingQueue、LinkedBlockingQueue

获取数据时队列中无数据，阻塞。添加数据时队列已满，阻塞。

**添加元素**

add：添加元素的时候，若超出了度列的长度会直接抛出异常

offer：添加元素的时候，若超出了度列的长度会直接返回false

put：添加元素的时候，若超出了度列的长度会阻塞一直等待空间，以加入元素

**获取元素**

remove：获取元素，若队列为空，会抛出异常

poll：获取元素，队列为空时，返回null

take：获取元素，队列为空时，队列阻塞

element：查看队首元素，队列元素为空抛异常

peek：查看队首元素，队列元素为空返回 null

##### SynchronousQueue

容量为0的队列，使用put添加元素时阻塞，直到另一个线程取到数据

场景：两个线程交换数据

```java
static BlockingQueue<String> blockingQueue = new SynchronousQueue<>();
public static void main(String[] args) throws InterruptedException {
    new Thread(() -> {
        try {
            String value = blockingQueue.take();
            System.out.println("子线程取到主线程数据:" + value);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }).start();
    Thread.sleep(1000);
    blockingQueue.put("1");
}
```

##### TransferQueue LinkedTransferQueue

与 SynchronousQueue 的区别在于，使用 `transfer` 方法来添加数据，并且当这个数据不被取走，线程会一直守在原地，类似MQ的消息确认机制。

```java
 transferQueue.transfer("data");
```

#### ConcurrentLinkedQueue

> 底层使用CAS实现原子性操作

使用 ConcurrentLinkedQueue 实现卖票程序

```java
static Queue<String> tickets = new ConcurrentLinkedQueue<>();
static {
    for (int i = 0; i < 1000; i++)
        tickets.add("票 编号:" + i);
}
public static void main(String[] args) {
    for (int i = 0; i < 10; i++) {
        new Thread(() -> {
            while (true){
                String s = tickets.poll();
                if (s == null) break;
                System.out.println("销售了 - " + s);
            }
        }).start();
    }
}
```

#### PriorityQueue

有序的队列，内部使用二叉树实现

#### DelayQueue

按照内部到期的时间进行排序，等待时间短的会排到队列的前面。

使用场景：按时间进行任务调度

## Map

| 名称          | 特点                                  | get(key)      | put(key) |
| ------------- | ------------------------------------- | ------------- | -------- |
| HashMap       | 线程不安全，高效                      | O(1)~O(log n) | O(1)     |
| LinkedHashMap | 查询时有序 (存储还是无序)             | O(1)~O(log n) | O(1)     |
| TreeMap       | 可根据指定值排序(取决于Compare返回值) | O(log n)      | O(1)     |
| Hashtable     | 线程安全，低效                        | O(1)~O(log n) | O(1)     |

### HashMap

**JDK7和8的异同**

JDK7：

1. 数组 + 链表
2. 插入链表头部
3. 直接计算 key 的 HashCode 值
4. 扩容时会颠倒链表顺序
5. 只要大于阈值就直接扩容2倍

JDK8：

1. 数组 + 链表 + 红黑树
2. 插入链表尾部
3. 采用 Key 的 HashCode 异或上 Key 的 HashCode 进行无符号右移16位的结果 `(h = key.hashCode()) ^ (h >>> 16)`，避免了只靠低位数据来计算哈希时导致的冲突，计算结果由高低位结合决定，使元素分布更均匀
4. 扩容时保持原链表顺序
5. 当数组容量小于64时，直接扩容。大于64时，若链表长度大于8就转红黑树，否则就扩容。

**负载因子为什么是0.75**

负载因子过小会导致更快扩容，浪费空间。过大会导致哈希碰撞的几率变大。

**HashMap扩容复制**

不是简单的将原数组中的每一个元素取出进行重新hash映射，而是做移位检测。所谓移位检测的含义具体是针对HashMap做映射时的&运算所提出的，通过上文对&元算的分析可知，映射的本质即看hash值的某一位是0还是1，当扩容以后，会相比于原数组多出一位做比较，由多出来的这一位是0还是1来决定是否进行移位，而具体的移位距离，也是可知的。

**JDK8添加过程**

1. 底层：数组 + 链表 + 红黑树
2. 首次添加操作创建数组，长度16，存的是一维数组Entry[]
3. 扩容：超过临界值(Capacity * Load Factory)，则扩容为原来2倍，并将元数据复制过来
   ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200209154206861.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

### LinkedHashMap

有序基于链表实现的 HashMap

### TreeMap

基于红黑树实现的有序Map

```java
// 自定义排序规则
new TreeMap<String, String>(new Comparator<String>() {
  @Override
  public int compare(String o1, String o2) {
    return 0;
  }
});
```

### ConcurrentHashMap

1.7：Segment + HashEntry + Unsafe

1.8：移除Segment，使锁的粒度更小，Synchronized + CAS

### ConcurrentSkitListMap

> 同步容器，有序

**跳表**

算法在最稀疏的层次进行搜索，直至需要查找的元素在该层两个相邻的元素中间。这时，算法将跳转到下一个层次，重复刚才的搜索，直到找到需要查找的元素为止。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210115222639601.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

### WeakHashMap

Entry 是弱引用，如果没有被其他强引用，那么GC后就会被回收