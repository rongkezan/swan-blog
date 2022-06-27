---
title: Tomcat
date: {{ date }}
categories:
- 基础知识
---

# Tomcat

## 总体架构
Tomcat为什么慢，因为它在应用层，是Java开发跑在JVM上的，相当于在内核上又虚拟的一块内存出来，在CPU调内核的时候又切换成虚拟机的状态，所以性能低。

![在这里插入图片描述](https://img-blog.csdnimg.cn/202101161546587.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

- 面向组件架构
- 基于JMX
- 事件侦听

## 面向组件架构

tomcat代码看似很庞大，但从结构上看却很清晰和简单，它主要由一堆组件组成，如Server、Service、Connector等，并基于JMX管理这些组件，另外实现以上接口的组件也实现了代表生存期的接口Lifecycle，使其组件履行固定的生存期，在其整个生存期的过程中通过事件侦听LifecycleEvent实现扩展。Tomcat的核心类图如下所示：

![img](https://p-blog.csdn.net/images/p_blog_csdn_net/cutesource/EntryImages/20091214/coreClass.jpg)

Catalina：与开始/关闭shell脚本交互的主类，因此如果要研究启动和关闭的过程，就从这个类开始看起。

Server：是整个Tomcat组件的容器，包含一个或多个Service。

Service：Service是包含Connector和Container的集合，Service用适当的Connector接收用户的请求，再发给相应的Container来处理。

Connector：实现某一协议的连接器，如默认的有实现HTTP、HTTPS、AJP协议的。

Container：可以理解为处理某类型请求的容器，处理的方式一般为把处理请求的处理器包装为Valve对象，并按一定顺序放入类型为Pipeline的管道里。Container有多种子类型：Engine、Host、Context和Wrapper，这几种子类型Container依次包含，处理不同粒度的请求。另外Container里包含一些基础服务，如Loader、Manager和Realm。

Engine：Engine包含Host和Context，接到请求后仍给相应的Host在相应的Context里处理。

Host：就是我们所理解的虚拟主机。

Context：就是我们所部属的具体Web应用的上下文，每个请求都在是相应的上下文里处理的。

Wrapper：Wrapper是针对每个Servlet的Container，每个Servlet都有相应的Wrapper来管理。

可以看出Server、Service、Connector、Container、Engine、Host、Context和Wrapper这些核心组件的作用范围是逐层递减，并逐层包含。

**下面就是些被Container所用的基础组件**

Loader：是被Container用来载入各种所需的Class。

Manager：是被Container用来管理Session池。

Realm：是用来处理安全里授权与认证。

## 基于JMX

Tomcat会为每个组件进行注册过程，通过Registry管理起来，而Registry是基于JMX来实现的，因此在看组件的init和start过程实际上就是初始化MBean和触发MBean的start方法，会大量看到形如下面这样的代码，这实际上就是通过JMX管理各种组件的行为和生命期。

```java
Registry.getRegistry(null, null).invoke(mbeans, "init", false);
Registry.getRegistry(null, null).invoke(mbeans, "start", false);
```

## 事件侦听

各个组件在其生命期中会有各种各样行为，而这些行为都有触发相应的事件，Tomcat就是通过侦听这些时间达到对这些行为进行扩展的目的。在看组件的init和start过程中会看到大量如：

lifecycle.fireLifecycleEvent(AFTER_START_EVENT, null);这样的代码，这就是对某一类型事件的触发，如果你想在其中加入自己的行为，就只用注册相应类型的事件即可。