---
title: Logback日志日期输出有误
date: {{ date }}
categories:
- 线上问题
---

## 现象

不定时出现（很难复现）今天的日志输出到了昨天的日志文件中，比如 今天是 2018-11-27 日，默认的日志输出文件是 default.log， 会出现日志打印到了 default.log.2018-11-26 然而 default.log 却只有当天进行 diamond 推送的日志

```
	由于应用和diamond进行了对接并和spring进行了集成，所以会一个异步监听diamond推送并对应用中
	使用@Value注解的地方进行动态修改， 每次推送事件发生时都会打印日志
```

- @Slf4j 编译后的代码

Logger log = org.slf4j.LoggerFactory.getLogger(Foo.class);

## slf4j 和 logback 的绑定和初始化

- slf4j 约定了绑定规范，需要在 org.slf4j.impl 包下有一个 StaticLoggerBinder implements LoggerFactoryBinder 的类，这个可以是任何日志框架的类，当然我们也可以自己搞一个

  - LoggerFactoryBinder 的接口定义

  ```java
  	public interface LoggerFactoryBinder {
  
  /**
   * Return the instance of {@link ILoggerFactory} that 
   * {@link org.slf4j.LoggerFactory} class should bind to.
   * 
   * @return the instance of {@link ILoggerFactory} that 
   * {@link org.slf4j.LoggerFactory} class should bind to.
   */
  public ILoggerFactory getLoggerFactory();
  
  /**
   * The String form of the {@link ILoggerFactory} object that this 
   * <code>LoggerFactoryBinder</code> instance is <em>intended</em> to return. 
   * 
   * <p>This method allows the developer to interrogate this binder's intention
   * which may be different from the {@link ILoggerFactory} instance it is able to 
   * yield in practice. The discrepancy should only occur in case of errors.
   * 
   * @return the class name of the intended {@link ILoggerFactory} instance
   */
  public String getLoggerFactoryClassStr();
  ```

}

```
​```
```

- 当调用 org.slf4j.LoggerFactory.getLogger (Foo.class); 时，会通过 StaticLoggerBinder 的 getLoggerFactory 方法获取日志框架的 ILoggerFactory 实现，进而获取日志输出对象

  - StaticLoggerBinder 在类加载的时候会进行日志初始化，包括加载日志配置比如默认的 lomback.xml

  ```java
  	public class StaticLoggerBinder implements LoggerFactoryBinder {
  
  /**
   * Declare the version of the SLF4J API this implementation is compiled
   * against. The value of this field is usually modified with each release.
   */
  // to avoid constant folding by the compiler, this field must *not* be final
  public static String REQUESTED_API_VERSION = "1.7.16"; // !final
  
  final static String NULL_CS_URL = CoreConstants.CODES_URL + "#null_CS";
  
  /**
   * The unique instance of this class.
   */
  	// 这里是单例模式这个是slf4j的约定要求单例模式
  private static StaticLoggerBinder SINGLETON = new StaticLoggerBinder();
  
  private static Object KEY = new Object();
  
  static {
  // 这个地方进行初始化，类加载的时候就会触发
      SINGLETON.init();
  }
  	...
  	}
  ```

- 应用通过 org.slf4j.LoggerFactory.getLogger (Foo.class); 获得了日志框架的 Logger 实现进行日志打印

## 问题排查

1. 断点 logback 初始化的地方，StaticLoggerBinder 静态语句块

```java
 	static {
    // 这个地方进行初始化，类加载的时候就会触发
        SINGLETON.init();
    }
```

1. 发现断点被执行了多次

   - 静态语句块只有类加载的时候仅且执行一次，这里被执行了多次，怀疑是被加载了多次
   - 通过 IDEA 的 alt + F8，执行 this.getClass ().getClassLoader ().getClass ().getName () 发现每次都不一样
   - 发现应用连接 diamond 的方法被执行了两次
     - 在 diamond 推送时，同样也会执行两次
     - **跟踪发现一个是 Pandora 调用的，一个是系统启动第一次调用的**
     - 推送一次，因为有两个链接顾推送事件的日志会被打印两次，查看日志对象 Logger 发现一个是 springboot 加载的一个是 Pandora 加载的
   - 在应用启动完毕之后，发现 Pandora 加载的 bean 是用的 com.taobao.pandora.boot.loader.ReLaunchURLClassLoader
   - springboot 加载用的是 org.springframework.boot.loader.LaunchedURLClassLoader

2. 此时已经确定系统有两套独立的日志输出入口，并打印到同一个日志文件

   ## logback 源码跟踪

   **应用 appender 配置**

```xml
    <appender name="defaultAppender" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/txp-default.log</file>
        <encoder>
            <pattern>${FILE_LOG_PATTERN}</pattern>
            <charset>utf8</charset>
        </encoder>
        <append>true</append>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${LOG_PATH}/default.log.%d{yyyy-MM-dd}</fileNamePattern>
            <maxHistory>15</maxHistory>
        </rollingPolicy>
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>INFO</level>
        </filter>
    </appender>
```

**源码追踪到日志跨天翻滚的地方**

```java
public class RollingFileAppender<E> extends FileAppender<E> {

...

 @Override
    protected void subAppend(E event) {
        // The roll-over check must precede actual writing. This is the
        // only correct behavior for time driven triggers.

        // We need to synchronize on triggeringPolicy so that only one rollover
        // occurs at a time
        synchronized (triggeringPolicy) {
        // 这个地方判断是否满足日志翻滚条件
        // 我们的应用配置是按天输出日志，所以这里会进行跨天判断
            if (triggeringPolicy.isTriggeringEvent(currentlyActiveFile, event)) {
                rollover(); // 这个地方进行日志翻转
            }
        }

        super.subAppend(event);
    }
...
}
```

**跨天判断的代码片**

```java
@NoAutoStart
public class DefaultTimeBasedFileNamingAndTriggeringPolicy<E> extends TimeBasedFileNamingAndTriggeringPolicyBase<E> {

    public boolean isTriggeringEvent(File activeFile, final E event) {
    // 获得当前时间
        long time = getCurrentTime();
        // nextCheck因为我们的配置是按天那么nextCheck的值就是明天凌晨
        if (time >= nextCheck) {
            Date dateOfElapsedPeriod = dateInCurrentPeriod;
            addInfo("Elapsed period: " + dateOfElapsedPeriod);
            elapsedPeriodsFileName = tbrp.fileNamePatternWithoutCompSuffix.convert(dateOfElapsedPeriod);
            setDateInCurrentPeriod(time);
            computeNextCheck();
            return true;
        } else {
            return false;
        }
    }
}
```

日志翻转代码片段

```java
public class RollingFileAppender<E> extends FileAppender<E> {
    public void rollover() {
        lock.lock();
        try {
            // Note: This method needs to be synchronized because it needs exclusive
            // access while it closes and then re-opens the target file.
            //
            // make sure to close the hereto active log file! Renaming under windows
            // does not work for open files.
            // 这里关闭当前文件输出流
            this.closeOutputStream();
            // 翻转日志文件，根据应用的配置进行反转，本应用其实就是把文件重命名
            // （用的类是TimeBasedRollingPolicy），比如重命名为default.log.2018-11-26
            attemptRollover();
            // 这个地方打开新文件default.log进行日志输出
            attemptOpenFile();
        } finally {
            lock.unlock();
        }
    }
    
    // 这个是打开新日志文件的逻辑
     private void attemptOpenFile() {
        try {
            // update the currentlyActiveFile LOGBACK-64
            currentlyActiveFile = new File(rollingPolicy.getActiveFileName());

            // This will also close the file. This is OK since multiple close operations are safe.
            this.openFile(rollingPolicy.getActiveFileName());
        } catch (IOException e) {
            addError("setFile(" + fileName + ", false) call failed.", e);
        }
    }
 }
```

对日志文件进行重命名的代码

```java
public class TimeBasedRollingPolicy<E> extends RollingPolicyBase implements TriggeringPolicy<E> {
	public void rollover() throws RolloverFailure {

        // when rollover is called the elapsed period's file has
        // been already closed. This is a working assumption of this method.

        String elapsedPeriodsFileName = timeBasedFileNamingAndTriggeringPolicy.getElapsedPeriodsFileName();

        String elapsedPeriodStem = FileFilterUtil.afterLastSlash(elapsedPeriodsFileName);

        if (compressionMode == CompressionMode.NONE) {
            if (getParentsRawFileProperty() != null) {
                // 这个地方对文件进行了重命名
   				renameUtil.rename(getParentsRawFileProperty(), elapsedPeriodsFileName);
            } // else { nothing to do if CompressionMode == NONE and parentsRawFileProperty == null }
        } else {
            if (getParentsRawFileProperty() == null) {
                compressionFuture = compressor.asyncCompress(elapsedPeriodsFileName, elapsedPeriodsFileName, elapsedPeriodStem);
            } else {
                compressionFuture = renameRawAndAsyncCompress(elapsedPeriodsFileName, elapsedPeriodStem);
            }
        }

        if (archiveRemover != null) {
            Date now = new Date(timeBasedFileNamingAndTriggeringPolicy.getCurrentTime());
            this.cleanUpFuture = archiveRemover.cleanAsynchronously(now);
        }
    }
}
```

## 问题逻辑梳理

1. 系统引入了 Pandora，导致 diamond 对接地方被执行两次，分别是 Pandora 加载和默认加载

   其实类似 tomcat 一个 JVM 有多个 webapp，他们之间可能用了很多相同的类和框架等，但是又不能互相冲突，肯定需要一个 webapp 一个 classloader, 而一些容器公共的东西就用公共父亲 classloader 加载

2. 系统出现两套 log， 这里叫 pandora 加载的为 log1， 默认的加 log2

3. 跨天业务日志持续输出，log1 触发跨天事件，关闭 default.log 输出流，重命名 default.log 为 default.log.2018-11-26, 创建新文件 default.log

4. 这个时候发生 diamond 推送，那么就会触发日志打印表示有推送，由于步骤 1， log1 和 log2 都会打印

5. log2 触发跨天事件，重命名步骤 3 产生的 default.log 为 default.log.2018-11-26，创建新文件 default.log

6. log1 持续输出日志到了 default.log.2018-11-26， log2 输出日志到 default.log

![img](https://oscimg.oschina.net/oscnet/d4876e3c306c5a25a53baffa1a7faa7e397.jpg)

## 问题复现

1. 自定义一个 classloader

   ```java
   	package com.alibaba.log.test;
   
   	import java.net.URL;
   	import java.net.URLClassLoader;
   	import java.util.HashMap;
   
   	public class MyURLClassLoader extends URLClassLoader {
   	    public MyURLClassLoader(URL[] urls, ClassLoader parent) {
       super(urls, parent);
   }
   
   public MyURLClassLoader(URL[] urls) {super(urls);}
   
   HashMap<String, Class<?>> hashMap = new HashMap<>();
   
   // 重写父类的类加载，实现自定义加载
   @Override
   public Class<?> loadClass(String name)
       throws ClassNotFoundException
   {
       synchronized (getClassLoadingLock(name)) {
           // First, check if the class has already been loaded
           Class<?> c = null;
           if (!name.startsWith("org.slf4j.") &&
               !name.startsWith("com.alibaba") &&
               !name.startsWith("ch.qos.logback.") ) {
               // 默认加载
               c = findLoadedClass(name);
           } else if (name.contains("MyURLClassLoader")) {
               // 使用默认加载
               c = findLoadedClass(name);
           } else {
               if (hashMap.containsKey(name)) {
                   // 加载过直接返回避免重复加载
                   return hashMap.get(name);
               }
               // 自定义加载
               c = findClass(name);
   
               hashMap.put(name, c);
           }
   
           if (c == null) {
               long t0 = System.nanoTime();
               try {
                   c = findSystemClass(name);
               } catch (ClassNotFoundException e) {
               }
   
               if (c == null) {
                   // If still not found, then invoke findClass in order
                   // to find the class.
                   long t1 = System.nanoTime();
                   c = findClass(name);
   
                   // this is the defining class loader; record the stats
                   sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                   sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                   sun.misc.PerfCounter.getFindClasses().increment();
               }
           }
           if (false) {
               resolveClass(c);
           }
           return c;
       }
   }
   
   	}
   ```

2. 日志输出类（此类有第一步的类加载加载）

```java
	package com.alibaba.log.test;

	import org.slf4j.Logger;

	public class WriteLog {

	    private static Logger log1 = org.slf4j.LoggerFactory.getLogger(WriteLog.class);
	
	    {
	        new Thread() {
	            @Override
	            public void run() {
	                for (int i = 0; i < 999999; i++) {
	                    log1.info("log1-> " + log1.getClass().getClassLoader().getClass().getName() + " " + i);
	                    try {
	                        Thread.sleep(3000);
	                    } catch (InterruptedException e) {
	                        e.printStackTrace();
	                    }
	                }
	            }
	        }.start();
	    }
}
```

\3. 程序入口类，此类也有一个日志打印 Log 不过是用默认类加载器加载

```java
package com.alibaba.log.test;

import org.slf4j.Logger;

import java.io.File;
import java.net.URL;
import java.util.Scanner;

public class Main {
    private static Logger log2 = org.slf4j.LoggerFactory.getLogger(WriteLog.class);

    public static void main(String[] args) throws Exception {
        test();
        Scanner scanner = new Scanner(System.in);
        System.out.println("输入任意字符触发log2打印日志(记得修改系统时间跨天)");
        String line = scanner.nextLine();

        log2.info("log2-> " + log2.getClass().getClassLoader().getClass().getName() + " " + line);
    }

    public static void test() throws Exception {
        String[] paths = new String[]{
            "/Users/shushangjin/.m2/txp/repository/org/slf4j/slf4j-api/1.7.25/slf4j-api-1.7.25.jar",
            "/Users/shushangjin/.m2/txp/repository/ch/qos/logback/logback-classic/1.2.3/logback-classic-1.2.3.jar",
            "/Users/shushangjin/.m2/txp/repository/ch/qos/logback/logback-core/1.2.3/logback-core-1.2.3.jar",
            "/Users/shushangjin/IdeaProjects/code1/testlog/target/classes/"
        };
        URL[] urls = new URL[paths.length];
        for (int i = 0; i < paths.length; i++) {
            urls[i] = new File(paths[i]).toURI().toURL();
        }

        MyURLClassLoader loader = new MyURLClassLoader (urls);
        Class cl = Class.forName ("com.alibaba.log.test.WriteLog", true, loader);
        System.out.println(cl.getClassLoader().getClass().getName());
        cl.newInstance();
    }

}
```

\4. 当程序跑起来之后，修改系统时间为明天，我这里修改为 2018-11-28 5. 在程序终端输入任意字符串 6. 见证奇迹的时刻

## 解决方案

1， 排除程序中有多个 load 的实例进行日志输出，当然也包括多进程 2， 从 logback 源码层面解决此问题，比如跨天判断的时候，触发了翻转，昨天的日志已经生成，那么不做任何动作