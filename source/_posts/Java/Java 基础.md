---
title: Java 基础
date: {{ date }}
categories:
- Java
---

## 类加载顺序

1. 类初始化：静态方法 -> 静态代码块，先初始化父类再初始化子类
2. 实例初始化 ：顺序: 非静态实例变量、非静态代码块、构造器代码
3. 子类覆写了父类的方法，初始化时只会执行子类的方法，若父类方法没被覆写，则执行父类方法

## 内部类

| 静态内部类                                                   | 非静态内部类                                                 |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 可以有静态成员(方法，属性)                                   | 不能有静态成员(方法，属性)                                   |
| 实例化<br />OutClassTest.InnerStaticClass inner = new OutClassTest.InnerStaticClass(); | 实例化<br />OutClassTest oc1 = new OutClassTest();<br />OutClassTest.InnerClass inner = oc1.new InnerClass(); |
| 调用方法或静态变量，通过类名直接调用<br />OutClassTest.InnerStaticClass.staticValue<br />OutClassTest.InnerStaticClass.method() | 实例化出来之后正常调用<br />inner.method()                   |

## String

String中的intern()方法

如果字符串常量池中已经包含一个等于此String对象的字符串，则返回代表池中这个字符串的String对象的引用，否则会将此String对象包含的字符串添加到常量池中，并且返回此String对象的引用。

例题：

```java
String str1 = new StringBuilder("Hello").append("World").toString();
System.out.println(str1 == str1.intern());      // true
String str2 = new StringBuilder("ja").append("va").toString();
System.out.println(str2 == str2.intern());      // false
```

解释：有一个JDK自带的初始化的字符串"java"在加载sun.misc.Version这个类的时候进入了常量池

## 参数传递

- 基本数据类型：传递值
- 引用数据类型：传地址

String、包装类等属于引用数据类型，同时是**不可变对象**

> 《Effective Java》
> 不可变对象(Immutable Object)：对象一旦被创建后，对象所有的状态及属性在其生命周期内不会发生任何变化。
> 由于ImmutableObject不提供任何setter方法，并且成员变量value是基本数据类型，getter方法返回的是value的拷贝，所以一旦ImmutableObject实例被创建后，该实例的状态无法再进行更改，因此该类具备不可变性。

## 泛型

### 泛型使用方式

![在这里插入图片描述](https://img-blog.csdnimg.cn/8e5b70b8587b4c718adc753dac84e637.png)

### 泛型常用通配符

> 常用通配符为：T、E、K、V、?

- ? 表示不确定的 java 类型
- T（type）表示一个具体的 java 类型
- K V（key value）分别代表 java 键值中的 Key Value
- E（Element）代表 Element

### 泛型擦除

Java 的泛型是伪泛型，这是因为 Java 在编译期间，所有的类型信息都会被擦掉。也就是说，在运行的时候是没有泛型的。

例如

```java
LinkedList<Cat> cats = new LinkedList<Cat>();
LinkedList list = cats; // 注意我在这里把范型去掉了，但是list和cats是同一个链表！
list.add(new Dog()); 	// 完全没问题！
```

因为Java的范型只存在于源码里，编译的时候给你静态地检查一下范型类型是否正确，而到了运行时就不检查了。上面这段代码在JRE（Java运行环境）看来和下面这段没区别：

```java
LinkedList cats = new LinkedList(); // 注意：没有范型！
LinkedList list = cats;
list.add(new Dog());
```

为什么要类型擦除呢？

主要是为了向下兼容，因为JDK5之前是没有泛型的，为了让JVM保持向下兼容，就出了类型擦除这个策略。

## 序列化与反序列化

### 什么是序列化和反序列化

序列化：把 Java 对象转为二进制流，方便存储和传输

反序列化：把二进制流恢复成 Java 对象

### Serializable接口

这个接口只是一个标记，没有具体的作用，但是如果不实现这个接口，在有些序列化场景会报错，所以一般建议，创建的JavaBean类都实现 Serializable。

### serialVersionUID

用来验证序列化的对象和反序列化对应的对象ID 是否一致。

```java
// 这个ID其实不重要，无论是1L还是自动生成的，只要序列化的ID和反序列化ID一致就行
private static final long serialVersionUID = 1L;
```

所以如果你没有定义一个 serialVersionUID， 结果序列化一个对象之后，在反序列化之前把对象的类的结构改了，比如增加了一个成员变量，则此时的反序列化会失败。

因为类的结构变了，所以 serialVersionUID 就不一致。

### Java 序列化包含静态变量吗

序列化的时候是不包含静态变量的。

### 对于有些变量不想序列化，怎么办

对于不想进行序列化的变量，使用 transient 关键字修饰。

transient 关键字的作用是：阻止实例中那些用此关键字修饰的的变量序列化。

当对象被反序列化时，被 transient 修饰的变量值不会被持久化和恢复。

transient 只能修饰变量，不能修饰类和方法。

### 序列化方式

- Java对象流列化 ：Java原生序列化方法即通过Java原生流(InputStream和OutputStream之间的转化)的方式进行转化，一般是对象输出流ObjectOutputStream 和对象输入流 ObjectInputStream 。
- Json序列化：这个可能是我们最常用的序列化方式，Json序列化的选择很多，一般会使用jackson包，通过ObjectMapper类来进行一些操作，比如将对象转化为byte数组或者将json串转化为对象。
- ProtoBuff序列化：ProtocolBuffer是一种轻便高效的结构化数据存储格式。ProtoBuff序列化对象可以很大程度上将其压缩，可以大大减少数据传输大小，提高系统性能。

## 反射

我们想在时候动态地获取类信息、创建类实例、调用类方法这时候就要用到反射。

通过反射你可以获取任意一个类的所有属性和方法，你还可以调用这些方法和属性。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2579a6fbdbcd482f85a4f52e06ff48ac.png)

获取 Class 对象的方式

```java
// 获取 User 对象的Class
Class<User> clazz = User.class;
Class<User> clazz = user.getClass();
Class<?> clazz = Class.forName("com.demo.entity.User");
Class<?> clazz = classLoader.loadClass("com.demo.entity.User");
```

利用反射创建对象

```java
// User 需要有无参构造
User user = clazz.newInstance();
// 利用 User 的有参构造创建
User user = clazz.getConstructor(String.class).newInstance("name");
```

利用反射操作属性

```java
// 根据属性名获取属性值
public static Object getFieldValue(String fieldName, Object object) {
    try {
        Field field = object.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        return field.get(object);
    } catch (Exception e) {
        log.warn("反射获取数据异常", e);
    }
    return null;
}

// 根据属性名设置属性值
public static void setFieldValue(String fieldName, Object object, Object value) {
    try {
        Class<?> c = object.getClass();
        Field f = c.getDeclaredField(fieldName);
        f.setAccessible(true);
        f.set(object, value);
    } catch (Exception e) {
        log.warn("反射设置数据异常", e);
    }
}
```

**反射中Class.forName和ClassLoader.loadClass的区别**

1. class.forName除了将类的class文件加载到jvm中之外，还会对类进行解释，执行类中的static块，还会执行给静态变量赋值的静态方法

2. classLoader只干一件事情，就是将class文件加载到jvm中，不会执行static中的内容,只有在newInstance才会去执行static块。

## 注解

Java注解本质上是一个标记，可以标记在类上、方法上、属性上等，标记自身也可以设置一些值，有了标记之后，我们就可以在编译或者运行阶段去识别这些标记，去处理一些具体的业务。

注解的应用：AOP、Lombok

注解的生命周期

- RetentionPolicy.SOURCE：给编译器用的，不会写入 class 文件
- RetentionPolicy.CLASS：会写入 class 文件，在类加载阶段丢弃，也就是运行的时候就没这个信息了
- RetentionPolicy.RUNTIME：会写入 class 文件，永久保存，可以通过反射获取注解信息

## 申请堆外内存

```java
// 分配 10M 堆外内存
long address = unsafe.allocateMemory(10 * 1024 * 1024);
// 释放堆外内存
unsafe.freeMemory()
```

## 接口和抽象类

接口：自上向下，定义约束和规范

抽象类：自下向上，将具体子类公共实现抽象到一个父类之中

## IO模型

### Java BIO

`Java BIO`就是Java的传统IO模型，对应了操作系统IO模型里的阻塞IO。

`Java BIO`相关的实现都位于`java.io`包下，其通信原理是客户端、服务端之间通过`Socket`套接字建立管道连接，然后从管道中获取对应的输入/输出流，最后利用输入/输出流对象实现发送/接收信息。

那么我们就进入Java的下一种IO模型——`Java NIO`，它对应操作系统IO模型中的多路复用IO，底层采用了`epoll`实现。

### Java NIO

`Java-NIO`则是`JDK1.4`中新引入的`API`，它在`BIO`功能的基础上实现了非阻塞式的特性，其所有实现都位于`java.nio`包下。`NIO`是一种基于通道、面向缓冲区的`IO`操作，相较`BIO`而言，它能够更为高效的对数据进行读写操作，同时与原先的`BIO`使用方式也大有不同。

Java-NIO中有三个核心概念：**`Buffer`（缓冲区）、`Channel`（通道）、`Selector`（选择器）**。

- 每个客户端连连接本质上对应着一个`Channel`通道，每个通道都有自己的`Buffer`缓冲区来进行读写，这些`Channel`被`Selector`选择器管理调度
- `Selector`负责轮询所有已注册的`Channel`，监听到有事件发生，才提交给服务端线程处理，服务端线程不需要做任何阻塞等待，直接在`Buffer`里处理`Channel`事件的数据即可，处理完马上结束，或返回线程池供其他客户端事件继续使用。
- 通过`Selector`，服务端的一个`Thread`就可以处理多个客户端的请求
- Buffer（缓冲区）就是饭店用来存放食材的储藏室，当服务员点餐时，需要从储藏室中取出食材进行制作。
- Channel（通道）是用于传输数据的车道，就像饭店里的上菜窗口，可以快速把点好的菜品送到客人的桌上。
- Selector（选择器）就是大堂经理，负责协调服务员、厨师和客人的配合和沟通，以保证整个就餐过程的效率和顺畅。

### Java AIO

`Java-AIO`也被成为`NIO2`，它是在`NIO`的基础上，引入了新的异步通道的概念，并提供了异步文件通道和异步套接字的实现。

它们的主要区别就在于这个异步通道，见名知意：使用异步通道去进行`IO`操作时，所有操作都为异步非阻塞的，当调用`read()/write()/accept()/connect()`方法时，本质上都会交由操作系统去完成，比如要接收一个客户端的数据时，操作系统会先将通道中可读的数据先传入`read()`回调方法指定的缓冲区中，然后再主动通知Java程序去处理。

所有的操作都是异步进行，通过`completed`接收异步回调，通过`failed`接收错误回调。

而且我们发现，相较于之前的`NIO`而言，`AIO`其中少了`Selector`选择器这个核心组件，选择器在`NIO`中充当了协调者的角色。

但在`Java-AIO`中，类似的角色直接由操作系统担当，而且不是采用轮询的方式监听`IO`事件，而是采用一种类似于“订阅-通知”的模式。

## IO

### 流的分类

按操作数据单位不同分为：字节流(8 bit)，字符流(16 bit)
按数据流的流向不同分为：输入流，输出流
按流的角色的不同分为：节点流，处理流
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020020116265641.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)
IO流体系
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200201162728374.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

### 流的操作

#### 操作步骤

1. File类的实例化
2. 流的实例化
3. 读写的操作
4. 资源的关闭

#### 字符流操作文件(文本文件)

1. 读文件

```java
public void readFile(){
    // 实例化File类对象，指明要操作的文件
    File file = new File("src/main/resources/file/1.txt");
    FileReader fr = null;
    try {
        // 提供具体的流
        fr = new FileReader(file);
        // 每次读1024个字符
        char[] cbuf = new char[1024];
        int len;
        // read(char[] cbuf) 返回每次读入cbuf数组中字符的个数，如果到达文件末尾，返回-1
        while ((len = fr.read(cbuf)) != -1){
            for (int i = 0; i < len; i++) {
                System.out.print(cbuf[i]);
            }
        }
        fr.read(cbuf);
    } catch (IOException e) {
        e.printStackTrace();
    } finally {
        try {
            if (fr != null)
                fr.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

2. 写文件

```java
public void writeFile() {
    File file = new File("src/main/resources/file/3.txt");
    FileWriter fw = null;
    try {
        // 参数2表示是否对原有文件追加
        fw = new FileWriter(file, false);
        fw.write("Hello World 张三");
    } catch (IOException e) {
        e.printStackTrace();
    } finally {
        try {
            if (fw != null)
                fw.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

3. 先读后写 -- 复制文件

```java
public void copyFile() {
    File srcFile = new File("src/main/resources/file/3.txt");
    File dstFile = new File("src/main/resources/file/4.txt");
    FileReader fr = null;
    FileWriter fw = null;
    try {
        fr = new FileReader(srcFile);
        fw = new FileWriter(dstFile, false);
        char[] cbuf = new char[1024];
        int len;
        while ((len = fr.read(cbuf)) != -1){
            fw.write(cbuf, 0, len);
        }
    } catch (IOException e) {
        e.printStackTrace();
    } finally {
        try {
            if (fr != null)
                fr.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
        try {
            if (fw != null)
                fw.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

#### 字节流操作文件(图片、视频等)

复制图片

```java
public void copyFile(){
    File srcFile = new File("src/main/resources/img/1.jpg");
    File dstFile = new File("src/main/resources/img/2.jpg");
    FileInputStream fis = null;
    FileOutputStream fos = null;
    try {
        fis = new FileInputStream(srcFile);
        fos = new FileOutputStream(dstFile);
        byte[] buf = new byte[1024];
        int len;
        while ((len = fis.read(buf)) != -1){
            fos.write(buf, 0, len);
        }
    } catch (IOException e) {
        e.printStackTrace();
    } finally {
        try {
            if (fis != null)
                fis.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
        try {
            if (fos != null)
                fos.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

#### 缓冲流的使用

作用：提升流的读取、写入速度。

复制图片文件

```java
public void copyImgFile(){
    File srcFile = new File("src/main/resources/img/1.jpg");
    File dstFile = new File("src/main/resources/img/2.jpg");
    BufferedInputStream bis = null;
    BufferedOutputStream bos = null;
    try {
        // 创建节点流
        FileInputStream fis = new FileInputStream(srcFile);
        FileOutputStream fos = new FileOutputStream(dstFile);
        // 创建缓冲流
        bis = new BufferedInputStream(fis);
        bos = new BufferedOutputStream(fos);
        // 读取写入
        byte[] buf = new byte[1024];
        int len;
        while ((len = bis.read(buf)) != -1){
            bos.write(buf, 0, len);
        }
    } catch (IOException e) {
        e.printStackTrace();
    } finally {
        // 资源关闭，关闭外层流的同时，内层流也会自动关闭
        try {
            if (bis != null)
                bis.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
        try {
            if (bos != null)
                bos.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

复制文本文件

```java
public void copyTxtFile(){
    File srcFile = new File("src/main/resources/file/1.txt");
    File dstFile = new File("src/main/resources/file/5.txt");
    BufferedReader br = null;
    BufferedWriter bw = null;
    try {
        FileReader fr = new FileReader(srcFile);
        FileWriter fw = new FileWriter(dstFile);
        br = new BufferedReader(fr);
        bw = new BufferedWriter(fw);
        char[] cbuf = new char[1024];
        int len;
        while ((len = br.read(cbuf)) != -1){
            bw.write(cbuf, 0, len);
        }
    } catch (IOException e) {
        e.printStackTrace();
    } finally {
        try {
            if (br != null)
                br.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
        try {
            if (bw != null)
                bw.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

## Stream Api

Stream 是数据渠道，用于操作数据源所生成的元素序列。

1. Stream 不存储元素
2. Stream 不改变源对象，他们会返回一个持有结果的新的 Stream
3. Stream 操作是延迟执行的，这意味着他们会等到需要结果的时候执行

Stream 操作步骤：创建流 -> 中间操作 -> 终止操作

**Stream Create**

```java
// 1.集合类
Collection.stream();
// 2.数组
Arrays.stream(T[]);
// 3.of
Stream.of(T... values);
// 4.创建无限流
// 4.1.迭代，案例：获取前10个偶数
Stream.iterate(final T seed, final UnaryOperator<T> f);
Stream.iterate(0, x -> x + 2).limit(10).forEach(System.out::println);
// 4.2.生成，案例：获取前10个随机数
Stream.genrate(Supplier<T> s);
Stream.generate(Math::random).limit(10).forEach(System.out::println);
```

**Stream Middle Operation**

```java
/**
 * 筛选和切片
 * - filter     接收lambda，从流冲排除某些元素
 * - limit      截断流，使其元素不超过给定数量
 * - skip(n)    跳过元素，返回一个扔掉了前n个元素的流，若流中元素不足n个，则返回一个空流，与limit互补
 * - distinct   筛选，通过流所生成元素的hashCode()和equals()去除重复元素
 */
emps.stream().filter(e -> e.getSalary() >= 100).limit(3).forEach(System.out::println);
emps.stream().filter(e -> e.getSalary() >= 100).skip(1).forEach(System.out::println);
emps.stream().filter(e -> e.getSalary() >= 100).distinct().forEach(System.out::println);

/**
 * 映射
 * map      接收lambda，将元素转换成其它形式或提取信息。接收另一个函数作为参数，该函数会被应用到每个元素上，并将其映射成一个新元素。
 * flatMap  接收一个函数作为参数，将流中的每个值都换成另一个流，然后把所有流连接成一个流，类似于list.allAll()
 * mapToDouble
 * mapToInt
 * mapToLong
 */
list.stream().map(String::toUpperCase).forEach(System.out::println);
/**
 * 排序
 * sorted()                 自然排序(Comparable)
 * sorted(Comparator com)   定制排序(Comparator)
 */
list.stream().sorted().forEach(System.out::println);
```

**Stream Termination**

匹配和查找

```java
// 是否匹配所有元素 allMatch
boolean b = emps.stream().allMatch(e -> e.getStatus().equals(Employee.Status.BUSY));

// 至少匹配一个元素 anyMatch
boolean b1 = emps.stream().anyMatch(e -> e.getStatus().equals(Employee.Status.BUSY));

// 检查是否没有匹配所有元素 noneMatch
boolean b2 = emps.stream().noneMatch(e -> e.getStatus().equals(Employee.Status.BUSY));

// 倒排后返回第一个元素 findFirst
Optional<Employee> op = emps.stream().sorted(Comparator.comparingDouble(Employee::getSalary).reversed()).findFirst();
System.out.println(op.get());

// 返回任意元素的值 findAny
Optional<Employee> op1 = emps.stream().filter(e -> e.getStatus().equals(Employee.Status.FREE)).findAny();
System.out.println(op1.get());

// 返回元素总个数 count
long count = emps.stream().count();

// 返回流中最大值 max
Optional<Employee> max = emps.stream().max(Comparator.comparingDouble(Employee::getSalary));
System.out.println(max.get());

// 返回流中最小值 min
Optional<Double> min = emps.stream().map(Employee::getSalary).min(Double::compare);
System.out.println(min.get());
```

归约：可以将流中的元素反复结合起来，得到一个值

```java
reduce(T identity, BinaryOperator);
reduce(BinaryOperator);
// 案例：求和
List<Integer> list = Arrays.asList(1,2,3,4,5,6,7,8,9);
Integer sum = list.stream().reduce(0, Integer::sum);
```

收集：将流转换为其它形式，接收一个Collector接口的实现，用于给Stream中元素做汇总的方法

```java
// 收集为List
List<String> list = emps.stream().map(Employee::getName).collect(Collectors.toList());
// 分组
Map<Employee.Status, List<Employee>> map = emps.stream()
    .collect(Collectors.groupingBy(Employee::getStatus));
// 多级分组
Map<Employee.Status, Map<String, List<Employee>>> map = emps.stream()
    .collect(Collectors.groupingBy(Employee::getStatus, Collectors.groupingBy(e -> {
        if (e.getAge() <= 35) {
            return "青年";
        } else {
            return "老年";
        }
    })));
// 分区
Map<Boolean, List<Employee>> map = emps.stream()
    .collect(Collectors.partitioningBy(e -> e.getSalary() > 8000));
// 统计
DoubleSummaryStatistics summary = emps.stream()
    .collect(Collectors.summarizingDouble(Employee::getSalary));
System.out.println(summary.getSum());
System.out.println(summary.getAverage());
System.out.println(summary.getMax());
// 拼接
String str = emps.stream().map(Employee::getName).collect(Collectors.joining(","));
```

## Optional

Optional 是用于防范 NullPointerException 。

可以将 Optional 看做是包装对象（可能是 null , 也有可能非 null ）的容器。

当我们定义了 一个方法，这个方法返回的对象可能是空，也有可能非空的时候，我们就可以考虑用 Optional 来包装它，这也是在 Java 8 被推荐使用的做法。

```java
Optional<String> optional = Optional.of("bam");
optional.isPresent(); 			// true
optional.get(); 				// "bam"
optional.orElse("fallback"); 	// "bam"
optional.ifPresent((s) -> System.out.println(s.charAt(0)));
```

