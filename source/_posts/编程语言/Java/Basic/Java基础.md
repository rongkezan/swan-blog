---
title: Java基础
date: {{ date }}
categories:
- 编程语言
- Java
---

## 类加载顺序

1. 类初始化：静态方法 -> 静态代码块，先初始化父类再初始化子类
2. 实例初始化 ：顺序: 非静态实例变量、非静态代码块、构造器代码
3. 子类覆写了父类的方法，初始化时只会执行子类的方法，若父类方法没被覆写，则执行父类方法

```java
public class Parent {
    private int i = param();
    private int a = paramParent();
    private static int j = staticParam();

    static{
        System.out.println("父类静态代码块");
    }

    Parent(){
        System.out.println("父类构造方法");
    }

    {
        System.out.println("父类代码块");
    }

    public int param(){
        System.out.println("父类实例变量1");
        return 1;
    }

    public int paramParent(){
        System.out.println("父类实例变量2");
        return 1;
    }

    public static int staticParam(){
        System.out.println("父类静态实例变量");
        return 1;
    }
}
```

```java
public class Child extends Parent {
    private int i = param();
    private static int j = staticParam();

    static {
        System.out.println("子类静态代码块");
    }

    Child(){
        System.out.println("子类构造方法");
    }

    {
        System.out.println("子类代码块");
    }

    @Override
    public int param(){
        System.out.println("子类实例变量");
        return 1;
    }

    public static int staticParam(){
        System.out.println("子类静态实例变量");
        return 1;
    }
}
```

```java
public class Client {
    public static void main(String[] args) {
        Child c1 = new Child();
        System.out.println();
    }
}
```

执行结果：

父类静态实例变量

父类静态代码块

子类静态实例变量

子类静态代码块

子类实例变量

父类实例变量2

父类代码块

父类构造方法

子类实例变量

子类代码块

子类构造方法

## 静态变量

静态变量作用在类层面，所有实例都会共享

```java
public class StaticVariableDemo {
    static int num;

    {
        num++;
    }

    void plus(){
        num++;
    }

    public static void main(String[] args) {
        StaticVariableDemo d1 = new StaticVariableDemo();
        StaticVariableDemo d2 = new StaticVariableDemo();
        d1.plus();
        d2.plus();
        System.out.println(num);
    }
}
```

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

## 序列化与反序列化

对象的序列化主要有两种用途：

1. 把对象的字节序列永久地保存到硬盘上，通常存放在一个文件中
2. 在网络上传送对象的字节序列

serialVersionUID作用：序列化时为了保持版本的兼容性，即在版本升级时反序列化仍保持对象的唯一性。

有两种生成方式：

1. 默认的1L，比如：private static final long serialVersionUID = 1L
2. 根据类名、接口名、成员方法及属性等来生成一个64位的哈希字段，比如：

```java
private static final long serialVersionUID = xxxxL;
```

## 反射

> JAVA反射机制是在运行状态中，对于任意一个类，都能够知道这个类的所有属性和方法。对于任意一个对象，都能够调用它的任意方法和属性。

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

可以通过注解来获取相关属性，通过这些属性再配合反射实现相应业务。

案例：通过反射获取全类名，再通过全类名实例化对应的类

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@interface Channel{
    String value();
}
```

```java
/* 连接接口 */
public interface Connection {
    boolean build();
}

public class CloudConnection implements Connection {
    @Override
    public boolean build() {
        System.out.println("Connect to cloud");
        return true;
    }
}

public class DatabaseConnection implements Connection {
    @Override
    public boolean build() {
        System.out.println("Connect to database");
        return true;
    }
}
```

```java
@Channel("com.demo.basic.annotation.DatabaseConnection")
public class Message{

    private Connection conn;

    public Message(){
        Channel channel = this.getClass().getAnnotation(Channel.class);
        try {
            // 通过反射获取到Connection对象实例
            conn = (Connection) Class.forName(channel.value()).newInstance();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void send(String msg){
        // 执行build方法，实际上执行的是Connection接口的实现类
        if (conn.build()){
            System.out.println("消息发送:" + msg);
        }
    }
}
```

```java
public class Client {
    public static void main(String[] args) {
        Message msg = new Message();
        msg.send("Hello World");
    }
}
```

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

