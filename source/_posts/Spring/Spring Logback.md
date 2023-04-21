---
title: Spring Logback
date: {{ date }}
categories:
- Spring
---

### Logback 日志配置文件

```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE configuration>
<configuration scan="true" scanPeriod="60 seconds" debug="false">

    <!-- Colorful log class -->
    <conversionRule conversionWord="clr" converterClass="org.springframework.boot.logging.logback.ColorConverter"/>
    <conversionRule conversionWord="wex" converterClass="org.springframework.boot.logging.logback.WhitespaceThrowableProxyConverter"/>
    <conversionRule conversionWord="wEx" converterClass="org.springframework.boot.logging.logback.ExtendedWhitespaceThrowableProxyConverter"/>

    <!-- Project name -->
    <property name="PROJECT_NAME" value="swan-server"/>

    <!-- Colorful log pattern -->
    <property name="CONSOLE_LOG_PATTERN" value="${CONSOLE_LOG_PATTERN:-%clr(%d{yyyy-MM-dd HH:mm:ss.SSS}){faint} %clr(${LOG_LEVEL_PATTERN:-%5p}) %clr(${PID:- }){magenta} %clr(---){faint} [%X{trace_id}] %clr([%15.15t]){faint} %clr(%-40.40logger{39}){cyan} %clr(:){faint} %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}}"/>
    <property name="FILE_LOG_PATTERN_EXT" value="%d{yyyy-MM-dd HH:mm:ss.SSS}[${PROJECT_NAME}][%level][%X{trace_id}][%thread] %logger{50} - %msg%n"/>

    <property name="LOG_CHARSET" value="UTF-8"/>
    <property name="LOG_PATH" value="/opt/logs"/>

    <!-- Console output log -->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <layout class="ch.qos.logback.classic.PatternLayout">
            <pattern>${CONSOLE_LOG_PATTERN}</pattern>
        </layout>
    </appender>

    <!-- ALL -->
    <appender name="ALL" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <File>${LOG_PATH}/web_info.log</File>
        <encoder>
            <pattern>${FILE_LOG_PATTERN_EXT}</pattern>
            <charset>${LOG_CHARSET}</charset>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <!-- Filename rolling daily -->
            <FileNamePattern>${LOG_PATH}/web_info-%d{yyyy-MM-dd}-%i.log</FileNamePattern>
            <!-- Keep 30 days worth of history -->
            <MaxHistory>30</MaxHistory>
            <!-- Each file should be must at 100m -->
            <maxFileSize>100MB</maxFileSize>
        </rollingPolicy>
    </appender>

    <!-- Default log -->
    <root>
        <appender-ref ref="STDOUT"/>
        <appender-ref ref="ALL"/>
    </root>
</configuration>
```

### Logback 日志丢失问题

在将Connectivity工程成功部署到阿里云Swarm集群上后。PVC组对集群进行了[压力测试](https://so.csdn.net/so/search?q=压力测试&spm=1001.2101.3001.7020)。并回收了集群的log日志进行分析。发现在发送了100万条数据后丢失了498条数据。前后log都有唯独中间log丢失。于是就着手开始调查日志丢失原因。

#### 调查过程

1、因为Connectivity运行在高并发环境下，单机需要承受3000 QPS ，首先需要判断日志是否是在高并发中线程切换导致log无法及时写入到文件因而丢失。
在本地通过1000个线程每个线程写入100次，以及500个线程每个线程写入200次等多次测试，发现丢失日志问题可以反复重现。排除是Connectivity项目代码问题。

2、通过对丢失log的时间和Java虚拟机垃圾回收的时间进行对比。在垃圾回收的时候并没有发现日志丢失问题， 判断不是垃圾回收对log写入造成了影响。

3、对日志进行按照线程写入情况统计，发现多个线程是在同一时刻进行了日志丢失。即写入843748条日志后丢失日志（400-500条不等）。并反复测试可以重现。通过计算每条日志大小为124K。124*843748 = 100MB。这个时候基本可以判断是文件大小以及写入策略的影响了。

4、对Logback配置文件进行分析。发现同时使用了TimeBasedRollingPolicy和SizeBasedTriggeringPolicy。并且SizeBasedTriggerPolicy设置文件最大为100MB。通过搜索相关问题并对源代码进行debug和分析。发现在同时使用这两种策略会导致Logback在执行的过程中由于自身代码问题抛出异常。我已经将issue提在了Logback的JIRA上。

5、如果需要按照时间分文件而且需要限制每个文件的大小应该使用SizeAndTimeBasedRollingPolicy，官方文档说明如下：

   Sometimes you may wish to archive files essentially by date but at the same time limit the size of each log file, in particular if post-processing tools impose size limits on the log files. In order to address this requirement, logback ships with SizeAndTimeBasedRollingPolicy.

​    Note that TimeBasedRollingPolicy already allows limiting the combined size of archived log files. If you only wish to limit the combined size of log archives, then TimeBasedRollingPolicy described above and setting the totalSizeCap property should be amply sufficent.

#### 错误示例

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration debug="false">
    <property name="LOG_HOME" value="log" />
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
        </encoder>
    </appender>
    <appender name="FILE"  class="ch.qos.logback.core.rolling.RollingFileAppender">
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <FileNamePattern>${LOG_HOME}/connectivi.log.%d{yyyy-MM-dd}.log</FileNamePattern>
            <MaxHistory>6</MaxHistory>
        </rollingPolicy>
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
            <immediateFlush>true</immediateFlush>
        </encoder>
        <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
            <MaxFileSize>100MB</MaxFileSize>
        </triggeringPolicy>
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>TRACE</level>
        </filter>

    </appender>
    <logger name="com.mindsphere.china.poc.connectivity" level="DEBUG" additivity="false">
        <appender-ref ref="STDOUT" />
        <appender-ref ref="FILE" />
    </logger>

    <root level="ERROR">
        <appender-ref ref="STDOUT" />
        <appender-ref ref="FILE" />
    </root>
</configuration>
```

#### 正确示例

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration debug="false">
    <property name="LOG_HOME" value="log" />
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
        </encoder>
    </appender>

    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <!-- rollover daily -->
            <fileNamePattern>${LOG_HOME}/connectivity.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <!-- each file should be at most 100MB, keep 30 days worth of history, but at most 20GB -->
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
        </encoder>
    </appender>

    <logger name="com.mindsphere.china.poc.connectivity" level="DEBUG" additivity="false">
        <appender-ref ref="STDOUT" />
        <appender-ref ref="FILE" />
    </logger>

    <root level="ERROR">
        <appender-ref ref="STDOUT" />
        <appender-ref ref="FILE" />
    </root>
</configuration>
```

