---
title: SpringBoot
date: {{ date }}
categories:
- Spring
---

# Spring Boot

## Spring Boot 使用

### 静态资源访问

#### 1. 静态资源目录

只要项目的静态资源放在类路径下以下目录，就可以正常访问

```properties
/static 
/public 
/resources 
/META-INF/resources
```

#### 2. 静态资源访问

默认是当前项目的根路径 + 静态资源名

收到请求时，SpringBoot会先找Controller是否能处理，不能处理的所有请求再交给静态资源处理器

#### 3. 静态资源配置

配置访问前缀

```properties
spring.mvc.static-path-pattern=/static
```

配置访问路径：配置完访问路径后默认访问路径失效

```properties
spring.web.resources.static-locations=classpath:/my
```

## Spring Boot 热更新

1. 引入依赖 `spring-boot-devtools`

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <optional>true</optional>
</dependency>
```

2. 修改完之后按 `Ctrl + F9` 更新项目

## 配置绑定

> 只有在容器中的组件，才能使用 @ConfigurationProperties

使用配置绑定读取配置文件 `application.properties` 中的属性有以下2种方式

```properties
user.nickname=张三
user.age=18
```

**@ConfigurationProperties + @Component**

```java
@Data
@Component
@ConfigurationProperties(prefix = "user")
public class User {
    private String nickname;

    private Integer age;
}
```

**@ConfigurationProperties + @EnableConfigurationProperties(User.class)**

```java
@Data
@ConfigurationProperties(prefix = "user")
public class User {
    private String nickname;

    private Integer age;
}
```

```java
@Configuration
@EnableConfigurationProperties(User.class)
public class SpringConfig {

}
```

**自动配置提示**

引入 `spring-boot-configuration-processor`

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

在SpringBoot打包时排除该依赖

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <excludes>
                    <exclude>
                        <groupId>org.springframework.boot</groupId>
                        <artifactId>spring-boot-configuration-processor</artifactId>
                    </exclude>
                </excludes>
            </configuration>
        </plugin>
    </plugins>
</build>
```

重启项目 在配置文件中键入自定义配置类的配置 产生提示

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201228222953379.png)

## 指标监控

> Spring Boot Ac0tuator

### 监控规则

引入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

访问 `域名/actuator/health` 可以查看服务的健康信息

访问规则： `域名/actuator/*`

| `auditevents`      | Exposes audit events information for the current application. Requires an `AuditEventRepository` bean. |
| ------------------ | ------------------------------------------------------------ |
| `beans`            | Displays a complete list of all the Spring beans in your application. |
| `caches`           | Exposes available caches.                                    |
| `conditions`       | Shows the conditions that were evaluated on configuration and auto-configuration classes and the reasons why they did or did not match. |
| `configprops`      | Displays a collated list of all `@ConfigurationProperties`.  |
| `env`              | Exposes properties from Spring’s `ConfigurableEnvironment`.  |
| `flyway`           | Shows any Flyway database migrations that have been applied. Requires one or more `Flyway` beans. |
| `health`           | Shows application health information.                        |
| `httptrace`        | Displays HTTP trace information (by default, the last 100 HTTP request-response exchanges). Requires an `HttpTraceRepository` bean. |
| `info`             | Displays arbitrary application info.                         |
| `integrationgraph` | Shows the Spring Integration graph. Requires a dependency on `spring-integration-core`. |
| `loggers`          | Shows and modifies the configuration of loggers in the application. |
| `liquibase`        | Shows any Liquibase database migrations that have been applied. Requires one or more `Liquibase` beans. |
| `metrics`          | Shows ‘metrics’ information for the current application.     |
| `mappings`         | Displays a collated list of all `@RequestMapping` paths.     |
| `scheduledtasks`   | Displays the scheduled tasks in your application.            |
| `sessions`         | Allows retrieval and deletion of user sessions from a Spring Session-backed session store. Requires a Servlet-based web application using Spring Session. |
| `shutdown`         | Lets the application be gracefully shutdown. Disabled by default. |
| `startup`          | Shows the startup steps data collected by the `ApplicationStartup`. Requires the `SpringApplication` to be configured with a `BufferingApplicationStartup`. |
| `threaddump`       | Performs a thread dump.                                      |

### 开启Web端配置

SpringBoot默认不开启Web端的所有Endpoint，配置开启

```properties
management.endpoints.web.exposure.include=*
```

配置health端点展示详细数据

```properties
management.endpoint.health.show-details=always
```

配置info的信息

```properties
info.appName=my-project
info.version=1.0.0
info.mavenProjectName=@project.artifactId@
```

### 定制Endpoint

```java
@Component
public class MyHealthIndicator extends AbstractHealthIndicator {

    @Override
    protected void doHealthCheck(Health.Builder builder) throws Exception {
        Map<String, Object> map = new HashMap<>();
        map.put("count", 1);
        map.put("ms", 30);
        builder.status(Status.DOWN)
                .withDetail("code", 100)
                .withDetails(map);
    }
}
```

访问health后就有定制

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201231201651587.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

### Spring Boot Admin

#### Spring Boot Admin Server

新建一个SpringBoot项目 加入依赖

```xml
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-server</artifactId>
    <version>2.3.1</version>
</dependency>
```

在启动类注解上增加 `@EnableAdminServer`

```java
@EnableAdminServer
@SpringBootApplication
public class MyBootAdminServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyBootAdminServerApplication.class, args);
    }
}
```

修改端口为7000

```properties
server.port=7000
```

#### Spring Boot Admin Client

增加依赖

```xml
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-client</artifactId>
    <version>2.3.1</version>
</dependency>
```

修改配置文件使其指向 `Spring Boot Admin Server` 的地址

```properties
spring.application.name=my-demo-app
spring.boot.admin.client.url=http://127.0.0.1:7000
spring.boot.admin.client.instance.prefer-ip=true
```

#### 启动项目测试

输入地址 http://127.0.0.1:7000

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201231210122929.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

## Spring Boot 启动源码解析

### 总体概览

```java
//创建一个新的实例，这个应用程序的上下文将要从指定的来源加载Bean
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    // 资源初始化资源加载器，默认为null
	this.resourceLoader = resourceLoader;
    // 断言主要加载资源类不能为 null，否则报错
	Assert.notNull(primarySources, "PrimarySources must not be null");
    // 初始化主要加载资源类集合并去重
	this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    // 推断当前 WEB 应用类型，一共有三种：NONE,SERVLET,REACTIVE
	this.webApplicationType = WebApplicationType.deduceFromClasspath();
    // 设置应用上线文初始化器
    // 从"META-INF/spring.factories"读取ApplicationContextInitializer类的实例名称集合并去重，并使用set去重（一共7个）
	setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    // 设置监听器
    // 从"META-INF/spring.factories"读取ApplicationListener类的实例名称集合并去重，并使用set去重（一共11个）
	setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    // 推断主入口应用类，通过当前调用栈，获取Main方法所在类，并赋值给mainApplicationClass
	this.mainApplicationClass = deduceMainApplicationClass();
}
```

```java
public ConfigurableApplicationContext run(String... args) {
    // 创建并启动计时监控类
	StopWatch stopWatch = new StopWatch();
	stopWatch.start();
    // 初始化应用上下文和异常报告集合
	ConfigurableApplicationContext context = null;
	Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    // 设置系统属性“java.awt.headless”的值，默认为true，用于运行headless服务器，进行简单的图像处理
    // 多用于在缺少显示屏、键盘或者鼠标时的系统配置，很多监控工具如jconsole 需要将该值设置为true
	configureHeadlessProperty();
    // 1.创建所有spring运行监听器并发布应用启动事件
    // 获取SpringApplicationRunListener类型的实例EventPublishingRunListener
    // 并封装进SpringApplicationRunListeners，然后返回SpringApplicationRunListeners
    // 说的简单点，getRunListeners就是准备好了运行时监听器EventPublishingRunListener。
	SpringApplicationRunListeners listeners = getRunListeners(args);
	listeners.starting();
	try {
        // 初始化默认应用参数类
		ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        // 2.根据运行监听器和应用参数来准备spring环境
		ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
        // 将要忽略的bean的参数打开
		configureIgnoreBeanInfo(environment);
        // 创建banner打印类
		Banner printedBanner = printBanner(environment);
        // 3.创建应用上下文，可以理解为创建一个容器
		context = createApplicationContext();
        // 准备异常报告器，用来支持报告关于启动的错误
		exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
        // 4.准备应用上下文，将启动类注入容器，为后续开启自动化提供基础
		prepareContext(context, environment, listeners, applicationArguments, printedBanner);
        // 5.刷新应用上下文
		refreshContext(context);
        // 应用上下文刷新后置处理，做一些扩展功能
		afterRefresh(context, applicationArguments);
        // 停止计时监控类
		stopWatch.stop();
        // 输出日志记录执行主类名、时间信息
		if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
		}
        // 发布应用上下文启动监听事件
		listeners.started(context);
        // 执行所有的Runner运行器
		callRunners(context, applicationArguments);
	}catch (Throwable ex) {
		handleRunFailure(context, ex, exceptionReporters, listeners);
		throw new IllegalStateException(ex);
	}
	try {
        // 发布应用上下文就绪事件
		listeners.running(context);
	}catch (Throwable ex) {
		handleRunFailure(context, ex, exceptionReporters, null);
		throw new IllegalStateException(ex);
	}
    // 返回应用上下文
	return context;
}
```

### 1.创建所有spring运行监听器并发布应用启动事件

```java
SpringApplicationRunListeners listeners = getRunListeners(args);
listeners.starting();
```

```java
/* --- SpringApplicationRunListeners listeners = getRunListeners(args) --- */
// 创建 Spring 监听器
private SpringApplicationRunListeners getRunListeners(String[] args) {
	Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
	return new SpringApplicationRunListeners(logger,
				getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args));
}

// SpringApplicationRunListeners 构造方法，将日志和监听器们初始化
SpringApplicationRunListeners(Log log, Collection<? extends SpringApplicationRunListener> listeners) {
	this.log = log;
	this.listeners = new ArrayList<>(listeners);
}

/* --- listeners.starting() --- */
void starting() {
    //循环遍历获取监听器
	for (SpringApplicationRunListener listener : this.listeners) {
		listener.starting();
	}
}

// 此处的监听器可以看出是事件发布监听器，主要用来发布启动事件
@Override
public void starting() {
    //这里是创建application事件 'applicationStartingEvent'
	this.initialMulticaster.multicastEvent(new ApplicationStartingEvent(this.application, this.args));
}

// applicationStartingEvent是springboot框架最早执行的监听器，
// 在该监听器执行started方法时，会继续发布事件，主要是基于spring的事件机制
@Override
public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
    ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
    //获取线程池，如果为空则同步处理。这里线程池为空，还未初始化
    Executor executor = getTaskExecutor();
    for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
        if (executor != null) {
            // 异步发送事件
            executor.execute(() -> invokeListener(listener, event));
        }
        else {
            // 同步发送事件
            invokeListener(listener, event);
        }
    }
}
```

### 2.根据运行监听器和应用参数来准备spring环境

```java
ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
```

```java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
	ApplicationArguments applicationArguments) {
	// 获取或者创建应用环境
	ConfigurableEnvironment environment = getOrCreateEnvironment();
    // 配置应用环境，配置propertySource和activeProfiles
	configureEnvironment(environment, applicationArguments.getSourceArgs());
    // listeners环境准备，广播ApplicationEnvironmentPreparedEvent
	ConfigurationPropertySources.attach(environment);
	listeners.environmentPrepared(environment);
    // 将环境绑定给当前应用程序
	bindToSpringApplication(environment);
    // 对当前的环境类型进行判断，如果不一致进行转换
	if (!this.isCustomEnvironment) {
		environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
					deduceEnvironmentClass());
	}
    // 配置propertySource对它自己的递归依赖
	ConfigurationPropertySources.attach(environment);
	return environment;
}

// 获取或者创建应用环境，根据应用程序的类型可以分为servlet环境、标准环境(特殊的非web环境)和响应式环境
private ConfigurableEnvironment getOrCreateEnvironment() {
    //存在则直接返回
    if (this.environment != null) {
        return this.environment;
    }
    //根据webApplicationType创建对应的Environment
    switch (this.webApplicationType) {
        case SERVLET:
            return new StandardServletEnvironment();
        case REACTIVE:
            return new StandardReactiveWebEnvironment();
        default:
            return new StandardEnvironment();
    }
}

//配置应用环境
protected void configureEnvironment(ConfigurableEnvironment environment, String[] args) {
	if (this.addConversionService) {
		ConversionService conversionService = ApplicationConversionService.getSharedInstance();
		environment.setConversionService((ConfigurableConversionService) conversionService);
	}
    //配置property sources
	configurePropertySources(environment, args);
    //配置profiles
	configureProfiles(environment, args);
}
```

### 3.创建应用上下文，可以理解为创建一个容器

```java
context = createApplicationContext();
```

```java
protected ConfigurableApplicationContext createApplicationContext() {
    Class<?> contextClass = this.applicationContextClass;
    if (contextClass == null) {
        try {
            // 根据不同的应用类型初始化不同的上下文应用类
            switch (this.webApplicationType) {
                case SERVLET:
                    contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
                    break;
                case REACTIVE:
                    contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
                    break;
                default:
                    contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
            }
        }
        catch (ClassNotFoundException ex) {
            throw new IllegalStateException(
                "Unable create a default ApplicationContext, please specify an ApplicationContextClass", ex);
        }
    }
    return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```

### 4.准备应用上下文，将启动类注入容器，为后续开启自动化提供基础

```java
prepareContext(context, environment, listeners, applicationArguments, printedBanner);
```

```java
private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
			SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
    // 应用上下文的environment
    context.setEnvironment(environment);
    // 应用上下文后处理
    postProcessApplicationContext(context);
    // 为上下文应用所有初始化器，执行容器中的applicationContextInitializer(spring.factories的实例)
    // 将所有的初始化对象放置到context对象中
    applyInitializers(context);
    // 触发所有SpringApplicationRunListener监听器的ContextPrepared事件方法。添加所有的事件监听器
    listeners.contextPrepared(context);
    // 记录启动日志
    if (this.logStartupInfo) {
        logStartupInfo(context.getParent() == null);
        logStartupProfileInfo(context);
    }
    // 注册启动参数bean，将容器指定的参数封装成bean，注入容器
    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    // 设置banner
    if (printedBanner != null) {
        beanFactory.registerSingleton("springBootBanner", printedBanner);
    }
    if (beanFactory instanceof DefaultListableBeanFactory) {
        ((DefaultListableBeanFactory) beanFactory)
        .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
    }
    if (this.lazyInitialization) {
        context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
    }
    // 加载所有资源，指的是启动器指定的参数
    Set<Object> sources = getAllSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    // 将bean加载到上下文中
    load(context, sources.toArray(new Object[0]));
    // 触发所有springapplicationRunListener监听器的contextLoaded事件方法，
    listeners.contextLoaded(context);
}

// 这里没有做任何的处理过程，因为beanNameGenerator和resourceLoader默认为空，可以方便后续做扩展处理
protected void postProcessApplicationContext(ConfigurableApplicationContext context) {
    if (this.beanNameGenerator != null) {
        context.getBeanFactory().registerSingleton(AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR,
                                                   this.beanNameGenerator);
    }
    if (this.resourceLoader != null) {
        if (context instanceof GenericApplicationContext) {
            ((GenericApplicationContext) context).setResourceLoader(this.resourceLoader);
        }
        if (context instanceof DefaultResourceLoader) {
            ((DefaultResourceLoader) context).setClassLoader(this.resourceLoader.getClassLoader());
        }
    }
    if (this.addConversionService) {
        context.getBeanFactory().setConversionService(ApplicationConversionService.getSharedInstance());
    }
}

// 将启动器类加载到spring容器中，为后续的自动化配置奠定基础，之前看到的很多注解也与此相关
protected void load(ApplicationContext context, Object[] sources) {
    if (logger.isDebugEnabled()) {
        logger.debug("Loading source " + StringUtils.arrayToCommaDelimitedString(sources));
    }
    BeanDefinitionLoader loader = createBeanDefinitionLoader(getBeanDefinitionRegistry(context), sources);
    if (this.beanNameGenerator != null) {
        loader.setBeanNameGenerator(this.beanNameGenerator);
    }
    if (this.resourceLoader != null) {
        loader.setResourceLoader(this.resourceLoader);
    }
    if (this.environment != null) {
        loader.setEnvironment(this.environment);
    }
    loader.load();
}

// springboot会优先选择groovy加载方式，找不到在选择java方式
private int load(Class<?> source) {
    if (isGroovyPresent() && GroovyBeanDefinitionSource.class.isAssignableFrom(source)) {
        // Any GroovyLoaders added in beans{} DSL can contribute beans here
        GroovyBeanDefinitionSource loader = BeanUtils.instantiateClass(source, GroovyBeanDefinitionSource.class);
        load(loader);
    }
    if (isComponent(source)) {
        this.annotatedReader.register(source);
        return 1;
    }
    return 0;
}
```

### 5.刷新应用上下文

```java
refreshContext(context);
```

```java
private void refreshContext(ConfigurableApplicationContext context) {
    refresh(context);
    if (this.registerShutdownHook) {
        try {
            context.registerShutdownHook();
        }
        catch (AccessControlException ex) {
            // Not allowed in some environments.
        }
    }
}

public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // 刷新上下文环境，初始化上下文环境，对系统的环境变量或者系统属性进行准备和校验
        prepareRefresh();

        // 初始化beanfactory，解析xml，相当于之前的xmlBeanfactory操作
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // 为上下文准备beanfactory，对beanFactory的各种功能进行填充，
        // 如@autowired，设置spel表达式解析器，设置编辑注册器，添加applicationContextAwareProcessor处理器等等
        prepareBeanFactory(beanFactory);

        try {
            // 提供子类覆盖的额外处理，即子类处理自定义的beanfactorypostProcess
            postProcessBeanFactory(beanFactory);

            // 激活各种beanfactory处理器
            invokeBeanFactoryPostProcessors(beanFactory);

            // 注册拦截bean创建的bean处理器，即注册beanPostProcessor
            registerBeanPostProcessors(beanFactory);

            // 初始化上下文中的资源文件如国际化文件的处理
            initMessageSource();

            // 初始化上下文事件广播器
            initApplicationEventMulticaster();

            // 给子类扩展初始化其他bean
            onRefresh();

            // 在所有的bean中查找listener bean,然后 注册到广播器中
            registerListeners();

            // 初始化剩余的非懒惰的bean，即初始化非延迟加载的bean
            finishBeanFactoryInitialization(beanFactory);

            // 发完成刷新过程，通知声明周期处理器刷新过程，同时发出ContextRefreshEvent通知别人
            finishRefresh();
        }

        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                            "cancelling refresh attempt: " + ex);
            }

            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();

            // Reset 'active' flag.
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        }

        finally {
            // Reset common introspection caches in Spring's core, since we
            // might not ever need metadata for singleton beans anymore...
            resetCommonCaches();
        }
    }
}
```

## Spring Boot 自动配置原理

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020122821461787.png)

### 自动包配置原理

```java
@SpringBootApplication -> @EnableAutoConfiguration -> @AutoConfigurationPackage -> 
@Import(AutoConfigurationPackages.Registrar.class)
```

```java
// 获取到标注类上的包名，将包名下的所有组件注册进容器
register(registry, new PackageImports(metadata).getPackageNames().toArray(new String[0]));
```

### 自动配置原理

spring.factories 是帮助 SpringBoot 项目包以外的 Bean 注册到 SpringBoot 项目的 Spring 容器的。

由于 @ComponentScan 注解只能扫描 SpringBoot 项目包内的 Bean 并注册到 Spring 容器中，因此需要 @EnableAutoConfiguration 来注册项目包外的bean，而 spring.factories 则是用来记录项目包外需要注册的bean类名。

```java
@SpringBootApplication -> @EnableAutoConfiguration -> @Import(AutoConfigurationImportSelector.class)
```

```java
// 获取自动配置的入口
getAutoConfigurationEntry(annotationMetadata);
// 获取所有候选的配置    
List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
// META-INF/spring.factories 里面写了 SpringBoot 一启动就需加载的所有配置类
Enumeration<URL> urls = classLoader.getResources(FACTORIES_RESOURCE_LOCATION);
```

**按需开启配置项：以DispatcherServlet为例**

```java
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@Configuration(proxyBeanMethods = false)
// 在Servlet的Web模块才生效
@ConditionalOnWebApplication(type = Type.SERVLET)
// 容器中拥有DispatcherServlet这个类才生效
@ConditionalOnClass(DispatcherServlet.class)
// 在 ServletWebServerFactoryAutoConfiguration 后配置
@AutoConfigureAfter(ServletWebServerFactoryAutoConfiguration.class)
public class DispatcherServletAutoConfiguration{
	@Configuration(proxyBeanMethods = false)
    // 匹配自定义规则，只有满足这些条件才会将Bean注入进来
	@Conditional(DefaultDispatcherServletCondition.class)
	@ConditionalOnClass(ServletRegistration.class)
	@EnableConfigurationProperties(WebMvcProperties.class)
    protected static class DispatcherServletConfiguration {
        @Bean(name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
        public DispatcherServlet dispatcherServlet(WebMvcProperties webMvcProperties) {
            DispatcherServlet dispatcherServlet = new DispatcherServlet();
            dispatcherServlet.setDispatchOptionsRequest(webMvcProperties.isDispatchOptionsRequest());
            dispatcherServlet.setDispatchTraceRequest(webMvcProperties.isDispatchTraceRequest());
            dispatcherServlet.setThrowExceptionIfNoHandlerFound(webMvcProperties.isThrowExceptionIfNoHandlerFound());
            dispatcherServlet.setPublishEvents(webMvcProperties.isPublishRequestHandledEvents());
            dispatcherServlet.setEnableLoggingRequestDetails(webMvcProperties.isLogRequestDetails());
            return dispatcherServlet;
        }
    }
}
```

**自动配置启动流程**

```java
prepareContext -> load
```

```java
private int load(Object source) {
	Assert.notNull(source, "Source must not be null");
    // 如果是class类型，启用注解类型
	if (source instanceof Class<?>) {
		return load((Class<?>) source);
	}
    // 如果是resource类型，启动xml解析
	if (source instanceof Resource) {
		return load((Resource) source);
	}
    // 如果是package类型，启用扫描包，例如@ComponentScan
	if (source instanceof Package) {
		return load((Package) source);
	}
    // 如果是字符串类型，直接加载
	if (source instanceof CharSequence) {
		return load((CharSequence) source);
	}
	throw new IllegalArgumentException("Invalid source type " + source.getClass());
}
```
```java
private int load(Class<?> source) {
    // 判断是否使用groovy脚本
    if (isGroovyPresent() && GroovyBeanDefinitionSource.class.isAssignableFrom(source)) {
        GroovyBeanDefinitionSource loader = BeanUtils.instantiateClass(source, GroovyBeanDefinitionSource.class);
        load(loader);
    }
    // 如果数据来源合法 则注册Bean
    if (isEligible(source)) {
        this.annotatedReader.register(source);	
        return 1;
    }
    return 0;
}
```
进入 `this.annotatedReader.register(source)` 追溯

```java
/**
 * 从给定的bean class中注册一个bean对象，从注解中找到相关的元数据
 * 将启动类注册成为一个 BeanDefination
 */
private <T> void doRegisterBean(Class<T> beanClass, @Nullable String name,
      @Nullable Class<? extends Annotation>[] qualifiers, @Nullable Supplier<T> supplier,
      @Nullable BeanDefinitionCustomizer[] customizers) {

   AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(beanClass);
   if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
      return;
   }

   abd.setInstanceSupplier(supplier);
   ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
   abd.setScope(scopeMetadata.getScopeName());
   String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));

   AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
   if (qualifiers != null) {
      for (Class<? extends Annotation> qualifier : qualifiers) {
         if (Primary.class == qualifier) {
            abd.setPrimary(true);
         }
         else if (Lazy.class == qualifier) {
            abd.setLazyInit(true);
         }
         else {
            abd.addQualifier(new AutowireCandidateQualifier(qualifier));
         }
      }
   }
   if (customizers != null) {
      for (BeanDefinitionCustomizer customizer : customizers) {
         customizer.customize(abd);
      }
   }

   BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
   definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
   BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
}
```

刷新容器自动装配入口

```java
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // 此处是自动装配的入口
        invokeBeanFactoryPostProcessors(beanFactory);
    }
}
```

在invokeBeanFactoryPostProcessors方法中完成bean的实例化和执行

```java
public static void invokeBeanFactoryPostProcessors(
    ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {
    // 开始遍历三个内部类，如果属于BeanDefinitionRegistryPostProcessor子类，加入到bean注册的集合
    // 否则加入到regularPostProcessors
    for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
        if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
            BeanDefinitionRegistryPostProcessor registryProcessor =
                (BeanDefinitionRegistryPostProcessor) postProcessor;
            //* 进入此方法的实现 ConfigurationClassPostProcessor
            registryProcessor.postProcessBeanDefinitionRegistry(registry); 
            registryProcessors.add(registryProcessor);
        } else {
            regularPostProcessors.add(postProcessor);
        }
    }
}
```

```java
postProcessBeanDefinitionRegistry -> processConfigBeanDefinitions -> parser.parse
```

开始执行自动配置逻辑（启动类指定的配置，非默认配置），最终会在ConfigurationClassParser类中，此类是所有配置类的解析类，所有的解析逻辑在parser.parse(candidates)中

```java
public void parse(Set<BeanDefinitionHolder> configCandidates) {
    for (BeanDefinitionHolder holder : configCandidates) {
        BeanDefinition bd = holder.getBeanDefinition();
        try {
            if (bd instanceof AnnotatedBeanDefinition) {
                //* 解析逻辑
                parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
            }
            else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
                parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
            }
            else {
                parse(bd.getBeanClassName(), holder.getBeanName());
            }
        }
        catch (BeanDefinitionStoreException ex) {
            throw ex;
        }
        catch (Throwable ex) {
            throw new BeanDefinitionStoreException(
                "Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
        }
    }
	//* 执行配置类
    this.deferredImportSelectorHandler.process();
}
```

```java
parse -> processConfigurationClass -> doProcessConfigurationClass
```

跟进doProcessConfigurationClass方法，此方式是支持注解配置的核心逻辑

```java
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass) throws IOException {

    // 处理内部类逻辑，由于传来的参数是启动类，并不包含内部类，所以跳过
    if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
        // Recursively process any member (nested) classes first
        processMemberClasses(configClass, sourceClass);
    }

    // Process any @PropertySource annotations
    // 针对属性配置的解析
    for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
        sourceClass.getMetadata(), PropertySources.class,
        org.springframework.context.annotation.PropertySource.class)) {
        if (this.environment instanceof ConfigurableEnvironment) {
            processPropertySource(propertySource);
        }
        else {
            logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
                        "]. Reason: Environment must implement ConfigurableEnvironment");
        }
    }

    // Process any @ComponentScan annotations
    // 这里是根据启动类@ComponentScan注解来扫描项目中的bean
    Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
        sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
    if (!componentScans.isEmpty() &&
        !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {

        for (AnnotationAttributes componentScan : componentScans) {
            // The config class is annotated with @ComponentScan -> perform the scan immediately
            //遍历项目中的bean，如果是注解定义的bean，则进一步解析
            Set<BeanDefinitionHolder> scannedBeanDefinitions =
                this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
            // Check the set of scanned definitions for any further config classes and parse recursively if needed
            for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
                BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
                if (bdCand == null) {
                    bdCand = holder.getBeanDefinition();
                }
                if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
                    //递归解析，所有的bean,如果有注解，会进一步解析注解中包含的bean
                    parse(bdCand.getBeanClassName(), holder.getBeanName());
                }
            }
        }
    }

    // Process any @Import annotations
    // 递归解析，获取导入的配置类，很多情况下，导入的配置类中会同样包含导入类注解
    processImports(configClass, sourceClass, getImports(sourceClass), true);

    // Process any @ImportResource annotations
    //解析@ImportResource配置类
    AnnotationAttributes importResource =
        AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
    if (importResource != null) {
        String[] resources = importResource.getStringArray("locations");
        Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
        for (String resource : resources) {
            String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
            configClass.addImportedResource(resolvedResource, readerClass);
        }
    }

    // Process individual @Bean methods
    //处理@Bean注解修饰的类
    Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
    for (MethodMetadata methodMetadata : beanMethods) {
        configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
    }

    // Process default methods on interfaces
    // 处理接口中的默认方法
    processInterfaces(configClass, sourceClass);

    // Process superclass, if any
    //如果该类有父类，则继续返回，上层方法判断不为空，则继续递归执行
    if (sourceClass.getMetadata().hasSuperClass()) {
        String superclass = sourceClass.getMetadata().getSuperClassName();
        if (superclass != null && !superclass.startsWith("java") &&
            !this.knownSuperclasses.containsKey(superclass)) {
            this.knownSuperclasses.put(superclass, configClass);
            // Superclass found, return its annotation metadata and recurse
            return sourceClass.getSuperClass();
        }
    }

    // No superclass -> processing is complete
    return null;
}
```

查看获取配置类的逻辑

```java
processImports(configClass, sourceClass, getImports(sourceClass), true);

private Set<SourceClass> getImports(SourceClass sourceClass) throws IOException {
    Set<SourceClass> imports = new LinkedHashSet<>();
    Set<SourceClass> visited = new LinkedHashSet<>();
    collectImports(sourceClass, imports, visited);
    return imports;
}
// 所有的bean都以导入的方式被加载进去
private void collectImports(SourceClass sourceClass, Set<SourceClass> imports, Set<SourceClass> visited)
    throws IOException {

    if (visited.add(sourceClass)) {
        for (SourceClass annotation : sourceClass.getAnnotations()) {
            String annName = annotation.getMetadata().getClassName();
            if (!annName.equals(Import.class.getName())) {
                // 递归
                collectImports(annotation, imports, visited);
            }
        }
        imports.addAll(sourceClass.getAnnotationAttributes(Import.class.getName(), "value"));
    }
}
```

继续回到ConfigurationClassParser中的parse方法中的最后一行,继续跟进该方法

```java
this.deferredImportSelectorHandler.process()
```

```java
public void process() {
    List<DeferredImportSelectorHolder> deferredImports = this.deferredImportSelectors;
    this.deferredImportSelectors = null;
    try {
        if (deferredImports != null) {
            DeferredImportSelectorGroupingHandler handler = new DeferredImportSelectorGroupingHandler();
            deferredImports.sort(DEFERRED_IMPORT_COMPARATOR);
            deferredImports.forEach(handler::register);
            handler.processGroupImports();
        }
    }
    finally {
        this.deferredImportSelectors = new ArrayList<>();
    }
}
```

```java
public void processGroupImports() {
    for (DeferredImportSelectorGrouping grouping : this.groupings.values()) {
        grouping.getImports().forEach(entry -> {
            ConfigurationClass configurationClass = this.configurationClasses.get(
                entry.getMetadata());
            try {
                processImports(configurationClass, asSourceClass(configurationClass),
                               asSourceClasses(entry.getImportClassName()), false);
            }
            catch (BeanDefinitionStoreException ex) {
                throw ex;
            }
            catch (Throwable ex) {
                throw new BeanDefinitionStoreException(
                    "Failed to process import candidates for configuration class [" +
                    configurationClass.getMetadata().getClassName() + "]", ex);
            }
        });
    }
}
```

```java
public Iterable<Group.Entry> getImports() {
    for (DeferredImportSelectorHolder deferredImport : this.deferredImports) {
        this.group.process(deferredImport.getConfigurationClass().getMetadata(),
                           deferredImport.getImportSelector());
    }
    return this.group.selectImports();
}

public DeferredImportSelector getImportSelector() {
    return this.importSelector;
}

public void process(AnnotationMetadata annotationMetadata, DeferredImportSelector deferredImportSelector) {
    Assert.state(deferredImportSelector instanceof AutoConfigurationImportSelector,
                 () -> String.format("Only %s implementations are supported, got %s",
                                     AutoConfigurationImportSelector.class.getSimpleName(),
                                     deferredImportSelector.getClass().getName()));
    AutoConfigurationEntry autoConfigurationEntry = ((AutoConfigurationImportSelector) deferredImportSelector)
        .getAutoConfigurationEntry(getAutoConfigurationMetadata(), annotationMetadata);
    this.autoConfigurationEntries.add(autoConfigurationEntry);
    for (String importClassName : autoConfigurationEntry.getConfigurations()) {
        this.entries.putIfAbsent(importClassName, annotationMetadata);
    }
}
```

**定制化配置**

- 修改配置文件为自定义的值
- 使用@Bean去替换底层的组件
- 使用自定义器XxxCustomizer