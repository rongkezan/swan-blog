---
title: Spring MVC
date: {{ date }}
categories:
- Spring
---

## Spring MVC 基本配置

web.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns="http://java.sun.com/xml/ns/javaee"
    xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
    id="WebApp_ID" version="3.0">

    <servlet>
        <!--名称 -->
        <servlet-name>springmvc</servlet-name>
        <!-- Servlet类 -->
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <!-- 启动顺序，数字越小，启动越早 -->
        <load-on-startup>1</load-on-startup>
        <init-param>
            <!--SpringMVC配置参数文件的位置 -->
            <param-name>contextConfigLocation</param-name>
            <!--默认名称为ServletName-servlet.xml -->
            <param-value>classpath*:springmvc-servlet.xml</param-value>
        </init-param>
    </servlet>

    <!--所有请求都会被springmvc拦截 -->
    <servlet-mapping>
        <servlet-name>springmvc</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
</web-app>
```

SpringMvc配置文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
    xmlns:context="http://www.springframework.org/schema/context"
    xmlns:mvc="http://www.springframework.org/schema/mvc"
    xsi:schemaLocation="http://www.springframework.org/schema/beans 
         http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context 
        http://www.springframework.org/schema/context/spring-context-4.3.xsd
        http://www.springframework.org/schema/mvc 
        http://www.springframework.org/schema/mvc/spring-mvc-4.3.xsd">

    <!-- 自动扫描包，实现支持注解的IOC -->
    <context:component-scan base-package="com.zhangguo.springmvc01" />

    <!-- Spring MVC不处理静态资源 -->
    <mvc:default-servlet-handler />

    <!-- 支持mvc注解驱动 -->
    <mvc:annotation-driven />

    <!-- 视图解析器 -->
    <bean
        class="org.springframework.web.servlet.view.InternalResourceViewResolver"
        id="internalResourceViewResolver">
        <!-- 前缀 -->
        <property name="prefix" value="/WEB-INF/view/" />
        <!-- 后缀 -->
        <property name="suffix" value=".jsp" />
    </bean>
</beans>
```

Controller

```java
@Controller
@RequestMapping("/hello")
public class HelloWorld {
    @RequestMapping("/sayhi")
    public String SayHi(Model model) {
        model.addAttribute("message", "Hello Spring MVC!");
        return "sayhi";
    }
}
```

Html

```html
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>
<!DOCTYPE html>
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
<title>Hello Spring MVC!</title>
</head>
<body>
    <h2>${message}</h2>
</body>
</html>
```

## Spring MVC 支持参数

1. 注解

```
@PathVariable		路径变量
@RequestHeader		获取请求头
@RequestParam		获取请求参数
@CookieValue		获取Cookie值
@RequestAttribute	获取request域属性
@RequestBody		获取请求体
@MatrixVariable		矩阵变量
```

## Spring MVC 九大组件

```java
/* 文件上传解析器 */
MultipartResolver multipartResolver;
/* 区域信息解析器 */
LocaleResolver localeResolver;
/* 主题解析器 */
ThemeResolver themeResolver;
/* Handler映射信息 */
List<HandlerMapping> handlerMappings;
/* Handler适配器 */
List<HandlerAdapter> handlerAdapters;
/* 异常解析器 */
List<HandlerExceptionResolver> handlerExceptionResolvers;
/* 请求到视图转换器 */
RequestToViewNameTranslator viewNameTranslator;
/* SpringMVC中运行重定向携带数据的功能 */
FlashMapManager flashMapManager;
/* 视图解析器 */
List<ViewResolver> viewResolvers;
```

## Spring MVC 流程

### 流程说明

1. 客户端请求提交到 DispatcherServlet
2. DispatcherServlet 收到请求后，遍历 HandlerMapping 集合得到 HandlerExecutionChain ，HandlerExecutionChain 中包含Handler 和 Intercepetor (处理器和拦截器)
3. HandlerMapping 根据 Url 找到 HandlerAdapter，由 HandlerAdapter 调用具体的Handler
   1. 先执行前置拦截器applyPreHandle
   2. 再执行处理器中目标方法，返回ModelAndView
   3. 最后执行后置拦截器 applyPostHandle
   4. 执行完成后返回ModelAndView给DispatcherServlet
4. DispatcherServlet 将ModelAndView传给ViewResolver解析后返回View
5. 将Model中的数据填充至View中，渲染视图返回给客户端

### 各个组件作用

DispatcherServlet：接收请求，请求转发，处理响应

HandlerMapping：根据URL找到对应的HandlerAdapter

HandlerAdapter：调用Handler

Handler：又名Controller，处理业务请求

ViewResolver：把逻辑视图解析成物理视图

### 图解

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201227173331827.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

### 关键源码

入口方法 `org.springframework.web.servlet.DispatcherServlet#doDispatch`

1. 获取处理器执行链 HandlerExecutionChain

```java
mappedHandler = getHandler(processedRequest);
// 遍历HandlerMapping获取handler
for (HandlerMapping mapping : this.handlerMappings) {
    HandlerExecutionChain handler = mapping.getHandler(request);
    if (handler != null) {
        return handler;
    }
}
```

2. 根据handler来获取合适的适配器

```java
HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
```

3. 执行Handler的业务逻辑

```java
// 执行前置拦截器
if (!mappedHandler.applyPreHandle(processedRequest, response))
   return;
// 执行业务逻辑返回 ModelAndView
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
// 执行后置拦截器
mappedHandler.applyPostHandle(processedRequest, response, mv);
```

4. 处理转发的信息渲染为视图

```java
processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
// 渲染视图
render(mv, request, response);
view = mv.getView();
view.render(mv.getModelInternal(), request, response);
```

## Spring MVC 定时器

```xml
<!-- 配置任务扫描 -->
<task:annotation-driven />

<!-- 扫描任务 -->
<context:component-scan base-package="com.demo.springTask" />
```

```java
@Component
public class MyTask{
    @Scheduled(cron = "0/5 * * * * ? ") // 间隔5秒执行
    public void taskCycle() {
        LocalDateTime date = LocalDateTime.now();
        String dateStr = date.format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
        System.out.println("定时任务 : " + dateStr);
    }
}
```

