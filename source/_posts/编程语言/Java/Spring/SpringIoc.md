---
title: SpringIOC
date: {{ date }}
categories:
- 编程语言
- Java
---

## Spring 容器

### Spring 容器 概述

IoC也称为依赖注入（DI）。 在此过程中，对象仅通过构造函数参数，工厂方法的参数或在构造或从工厂方法返回后在对象实例上设置的属性来定义其依赖项 。 然后，容器在创建bean时注入那些依赖项。 此过程从根本上讲是通过使用类的直接构造或诸如服务定位器模式之类的控件来控制其依赖项的实例化或位置的bean本身的逆过程（因此称为Control的倒置）。

核心依赖包：`org.springframework.beans` `org.springframework.context`

BeanFacotory接口提供了能管理任何类型对象的配置机制，ApplicationContext是它的一个子接口，增加了如下机制：

- Spring AOP 特性的简单集成
- 消息资源处理
- 事件发布
- 应用层面类似 `WebApplicationContext` 的上下文用于Web应用

ApplicationContext接口代表Spring IoC容器，并负责实例化，配置和组装Bean

容器通过读取配置元数据获取有关要实例化，配置和组装哪些对象的指令。

配置元数据：XML，Java注解，Java代码

ApplicationContext 常用的两种实现：ClassPathXmlApplicationContext, FileSystemXmlApplicationContext

### Spring 容器整体视图

![在这里插入图片描述](https://img-blog.csdnimg.cn/a74433be6b774ecd8313f7caea5802ef.png)

### Spring 接口

#### Spring顶层接口图

![在这里插入图片描述](https://img-blog.csdnimg.cn/bc2074457f8f4aabad6e23615f2cc6b7.png)

#### BeanRegistry

Spring 配置文件中每一个`<bean>`节点元素在 Spring 容器里都通过一个 BeanDefinition 对象表示，它描述了 Bean 的配置信息。而 BeanDefinitionRegistry 接口提供了向容器手工注册 BeanDefinition 对象的方法。

#### ApplicationContext

![在这里插入图片描述](https://img-blog.csdnimg.cn/ba357bb3e1bd496db88567d133da975d.png)

- **HierarchicalBeanFactory 和 ListableBeanFactory**： ApplicationContext 继承了 HierarchicalBeanFactory 和 ListableBeanFactory 接口，在此基础上，还通过多个其他的接口扩展了 BeanFactory 的功能。
- **ApplicationEventPublisher**：让容器拥有发布应用上下文事件的功能，包括容器启动事件、关闭事件等。实现了 ApplicationListener 事件监听接口的 Bean 可以接收到容器事件 ， 并对事件进行响应处理 。 在 ApplicationContext 抽象实现类AbstractApplicationContext 中，我们可以发现存在一个 ApplicationEventMulticaster，它负责保存所有监听器，以便在容器产生上下文事件时通知这些事件监听者。
- **MessageSource**：为应用提供 i18n 国际化消息访问的功能；
- **ResourcePatternResolver** ： 所有 ApplicationContext 实现类都实现了类似于PathMatchingResourcePatternResolver 的功能，可以通过带前缀的 Ant 风格的资源文件路径装载 Spring 的配置文件。

### Spring 容器 实例化

下面代码可以实例化一个Spring容器

```java
// create and configure beans
ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");

// retrieve configured instance
PetStoreService service = context.getBean("petStore", PetStoreService.class);

// use configured instance
List<String> userList = service.getUsernameList();
```

更灵活的变体是使用 GenericApplicationContext 结合读取委托器使用

```java
GenericApplicationContext context = new GenericApplicationContext();
new XmlBeanDefinitionReader(context).loadBeanDefinitions("applicationContext.xml");
context.refresh();
```

实现ApplicationContextAware接口得到ApplicationContext

```java
public class MyApplicationContext implements ApplicationContextAware {

    private ApplicationContext applicationContext;
    
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;    
    }
}
```

applicationContext.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <!-- services -->
    <bean id="petStore" class="org.springframework.samples.jpetstore.services.PetStoreServiceImpl">
        <property name="accountDao" ref="accountDao"/>
        <property name="itemDao" ref="itemDao"/>
        <!-- additional collaborators and configuration for this bean go here -->
    </bean>

    <!-- more bean definitions for services go here -->
</beans>
```

### Spring 容器 核心流程

```java
// There has a method named "loadBeanDefinitions" attampt to resolve resources like xml, annotation, groovy.
// These sources will be resolved to "BeanDefinition" through "BeanDefinitionReader". 
ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

// Invoke factory processors registered as beans in the context.
// First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
// Next, invoke the BeanFactoryPostProcessors that implement Ordered.
// Finally, invoke all other BeanFactoryPostProcessors.
invokeBeanFactoryPostProcessors(beanFactory);

// Register bean processors that intercept bean creation.
// add "BeanPostProcessor" to a CopyOnWriteArrayList named "beanPostProcessors"
registerBeanPostProcessors(beanFactory);

// Add beans that implement ApplicationListener as listeners.
registerListeners();

// Instantiate all remaining (non-lazy-init) singletons.
// First, it will merge beans with same beanName, there has a "applyMergedBeanDefinitionPostProcessors" to handle these merged beans.
// Next, it will find from cache whether bean exists, if the bean is not exists, it will create by "ObjectFactory".
// Finally, put this bean into cache and return this bean.
finishBeanFactoryInitialization(beanFactory);

// Last step: publish corresponding event, Observer pattern.
finishRefresh();
```

### Spring 父子容器

寻找Bean的时候，先从子容器里拿，拿不到再从父容器中拿

父容器不能访问子容器，子容器可以访问父容器，原因是父容器没有子容器的引用。

HierarchicalBeanFactory 中只有得到父容器的方法 getParentBeanFactory

如果在父容器中对Bean进行了增强，而这个Bean定义在了子容器中，那就不会把子容器中的Bean进行增强

## Spring Bean

Spring IoC容器管理一个或多个Bean。这些Bean是使用您提供给容器的配置元数据创建的（例如，以XML`<bean/>`定义的形式 ）。

在容器本身内，这些bean定义表示为`BeanDefinition` 对象，其中包含以下元数据：

- 包限定的类名：通常，定义了Bean的实际实现类。
- Bean行为配置元素，用于声明Bean在容器中的行为（作用域，生命周期回调等）。
- 引用该bean完成其工作所需的其他bean。这些引用也称为协作者或依赖项。
- 要在新创建的对象中设置的其他配置设置。例如，池的大小限制。

依赖注入的三种方式：构造函数注入、setter注入、接口注入

### Spring Bean 创建

**Bean创建流程**

BeanDefinitionReader 通过 xml/annotation 获得 Bean 的源信息，Bean 被实例化之后通过一系列的 Processor 最终完成Bean的创建

#### 1. 包扫描注解  +  组件注解

@ComponentScan + @Controller @Service @Component ...

```java
package com.demo.controller;
@Controller
public class TestController {

}
```

```java
@Configuration
@ComponentScan(value = "com.demo")
public class SpringConfig {

}
```

ComponentScan 使用 Filter 排除某一种类型的 Bean

```java
// 例: 排除了Contrller注解的Bean
@ComponentScan(value = "com.demo",
        excludeFilters = {@ComponentScan.Filter(type = FilterType.ANNOTATION, classes = Controller.class)})
```

FilterType种类

```
ANNOTATION: 注解
ASSIGNABLE_TYPE: 类型
ASPECTJ: 使用ASPECTJ表达式
REGEX: 使用正则表达式
CUSTOM: 自定义规则(TypeFilter的实现类)
```

#### 2. @Bean

通常用于导入第三方包里面的组件

```java
@Configuration
public class SpringConfig {
    @Bean
    public User user(){
        return new User();
    }
}
```

#### 3. @Import

@Import直接注入

```
@Configuration
@Import(User.class)
public class SpringConfig {

}
```

ImportSelector：使用Import给容器中导入多个组件

```java
@Configuration
@Import(MyImportSelector.class)
public class SpringConfig {

}
```

```java
public class MyImportSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        return new String[]{User.class.getName()};
    }
}
```

ImportBeanDefinitionRegistrar：手动注册Bean到容器中

```java
@Configuration
@Import(MyImportBeanDefinitionRegistrar.class)
public class SpringConfig {

}
```

```java
public class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        BeanDefinition beanDefinition = new RootBeanDefinition(User.class);
        registry.registerBeanDefinition("user", beanDefinition);
    }
}
```

#### 4. FactoryBean

可用于自定义实例化逻辑

```java
public class UserFactoryBean implements FactoryBean<User> {

    @Override
    public User getObject() throws Exception {
        return new User();
    }

    @Override
    public Class<?> getObjectType() {
        return User.class;
    }

    @Override
    public boolean isSingleton() {
        return true;
    }
}
```

```java
@Configuration
public class SpringConfig {
    @Bean
    public UserFactoryBean userFactoryBean(){
        return new UserFactoryBean();
    }
}
```

```java
// 返回的是User对象
User user = (User) applicationContext.getBean("userFactoryBean");
```

### Spring Bean Scope

1. singleton：每个IOC容器仅有一个单实例，容器创建时创建Bean

   使用 `@Lazy` 注解懒加载Bean，即获取时加载

2. prototype：每次请求产生一个新实例，请求时创建Bean

3. request：每次Http请求产生一个新实例

4. session：每次Http请求产生一个新的Bean，仅在当前Http Session内有效

5. application：类似标准HttpSession作用域

### Spring Bean 创建流程源码

```
refresh();
finishBeanFactoryInitialization(beanFactory);
beanFactory.preInstantiateSingletons();
getBean(beanName);
doGetBean(name, null, null, false);
```

```java
// 从缓存中检查是否有这个Bean
Object sharedInstance = getSingleton(beanName);
// 如果没有，就实例化一个Bean
if (sharedInstance != null && args == null) {
    bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
}
else {
    final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
    // 取得依赖的Bean，即创建当前Bean之前需要提前创建的Bean
    String[] dependsOn = mbd.getDependsOn();
    if (dependsOn != null) {
        for (String dep : dependsOn) {
            registerDependentBean(dep, beanName);
            getBean(dep);
        }
    }
    // 创建Bean实例
    if (mbd.isSingleton()) {
        sharedInstance = getSingleton(beanName, () -> createBean(beanName, mbd, args));
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
    }
}
return (T) bean;
```

```java
getSingleton(beanName, () -> createBean(beanName, mbd, args));
```

```java
// 先从Map中拿
Object singletonObject = this.singletonObjects.get(beanName);
// 没有的话再创建
singletonObject = singletonFactory.getObject();
// 加到Map中
addSingleton(beanName, singletonObject);
```

 IOC容器之一：保存单实例Bean

```java
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
```

BeanFactory：负责创建bean实例，容器里保存的所有单例Bean其实是一个map

ApplicationContext：BeanFactory的子接口，基于BeanFactory创建的对象之上完成容器的功能实现

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020121916571615.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

### Spring Bean 生命周期回调

> 只有单实例Bean才会被容器管理，多实例Bean不会被容器管理

调用顺序

- 初始化：对象创建完成，并赋值好，调用初始化方法
- 销毁：容器关闭时

控制 Bean 生命周期行为的三个选项：

- @PostConstruct 和@PreDestroy 注解（推荐）

```java
public class DemoBean {

    @PostConstruct
    public void init() {
        // init method
    }

    @PreDestroy
    public void destory() {
        // destory method
    }
}
```

- 自定义 init() 和 destroy() 方法

```java
@Bean(initMethod = "init", destroyMethod = "destroy")
public User user(){
	return new User();
}
```

- 实现InitializingBean和DisposableBean接口

```java
public class DemoBean implements InitializingBean, DisposableBean {

    @Override
    public void afterPropertiesSet() throws Exception {
        // init method
    }

    @Override
    public void destroy() throws Exception {
        // destroy method
    }
}
```

**BeanPostProcessor：Bean的后置处理器**

Spring底层对BeanPostProcessor的使用：Bean赋值，组件的注入，生命周期注解

- postProcessBeforeInitialization：在初始化（例如 @PostConstruct）之前工作
- postProcessAfterInitialization：在初始化之后工作

```java
public class MyBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("postProcessBeforeInitialization");
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("postProcessAfterInitialization");
        return bean;
    }
}
```

### Spring Bean 属性赋值

@Value

1. 基本数值
2. SPEL表达式：`#{}`
3. 取出配置文件的值：`${} `

配合 `@PropertySource` 使用

```java
@Data
public class User {
    @Value("${user.nick.name}")
    private String nickname;
    private Integer age;
}
```

```java
@Configuration
@PropertySource(value = "classpath:user.properties")
public class SpringConfig {
    @Bean
    public User user(){
        return new User();
    }
}
```

user.properties

```properties
user.nick.name=keith
```

### Spring Bean 自动装配

1. @Autowired 根据类型注入，找不到再根据名称注入

2. @Qualifier 根据名称注入，@Primary 根据类型优先注入当前Bean

3. @Resource 根据名称注入，找不到再根据类型注入

4. 构造方法注入：默认加载IOC容器中的组件，容器启动会调用无参构造器创建对象，再进行初始化赋值等操作

后置处理器 `AutowiredAnnotationBeanPostProcessor` 用于解析自动装配

### Spring Bean Post Processor

1. BeanPostPorcessor：Bean后置处理器，Bean创建对象初始化前后进行拦截工作

2. BeanFactoryPostProcessor：BeanFactory后置处理器，

   BeanFactory标准初始化所有Bean定义已经保存到加载到BeanFactory，但是Bean实例还未创建

   原理：IOC容器加载时调用 refresh() -> invokeBeanFactoryPostProcessors()调用所有的Processor

   实现 `BeanFactoryPostProcessor` 接口即可拿到容器的BeanFactory

```java
@Component
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        System.out.println("My Bean Factory");
    }
}
```

3. BeanDefinitionRegistry：在所有Bean将要被加载，而Bean实例还未创建

```java
@Component
public class MyBeanDefinitionRegistryPostProcessor implements BeanDefinitionRegistryPostProcessor {

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        // 该方法后执行
        System.out.println("My Registry");
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        // 该方法先执行
        System.out.println("My Factory");
    }
}
```

### BeanFactory和FactoryBean

- BeanFactory：必须遵循完整的Bean的生命周期去创建对象
- FactoryBean：创建对象，没有标准的流程，更像私人定制
  - isSingleton：判断是否是单例
  - getObjectType：返回对象的类型
  - getObject：返回对象

## Spring Profile

指定组件在哪个环境的情况下才能注册到容器中，不指定任何环境都能注册

```java
@Configuration
@PropertySource(value = "classpath:user.properties")
public class SpringConfig {

    @Bean
    @Profile("test")
    public User userTest(){
        System.out.println("This is test environment");
        return new User();
    }

    @Bean
    @Profile("prod")
    public User userProd(){
        System.out.println("This is prod environment");
        return new User();
    }
}
```

在 properties 配置文件中可以根据文件名配置不同的环境

```
application.properties
application-dev.properties
application-test.properties
```

在 yml 配置文件中可以使用 `---` 分隔配置不同的环境

```yaml
spring:
  profiles:
    active: dev
---
server:
  port: 8001
spring:
  profiles: dev
---
server:
  port: 8002
spring:
  profiles: test
```

在 VM options 中加入以下参数表示目前的环境为 test

```properties
-Dspring.profiles.active=test
## 使用java -jar启动
java -jar -Dspring.profiles.active=test app.jar 
```

在 Environment variables 中加入以下参数表示目前环境为 test

```properties
--spring.profiles.active=test
## 使用java -jar启动
java -jar app.jar --spring.profiles.active=test
```

加载配置文件关键源码

```java
// Spring 启动时 run 方法中准备环境
ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
// 进入之后调用了ConfigFileApplicationListener.Loader#load
protected void addPropertySources(ConfigurableEnvironment environment, ResourceLoader resourceLoader) {
    RandomValuePropertySource.addToEnvironment(environment);
    new Loader(environment, resourceLoader).load();
}
// 追溯到 ConfigFileApplicationListener#loadForFileExtension 根据文件后缀遍历加载
for (PropertySourceLoader loader : this.propertySourceLoaders) {
    for (String fileExtension : loader.getFileExtensions()) {
        if (processed.add(fileExtension)) {
            loadForFileExtension(loader, location + name, "." + fileExtension, profile, filterFactory,
                                 consumer);
        }
    }
}
```

对于 yml 和 properties 文件的解释

https://blog.csdn.net/weixin_42103026/article/details/112846171

## Spring 事件监听器

### 基本使用

ApplicationListener：监听容器中的事件，事件驱动模型开发。

1. 写一个监听器（实现 `ApplicationListener`）来监听某个事件（ `ApplicationEvent` 及其子类）

2. 将监听器加入容器中

3. 只要容器中有相关事件发布，我们就能监听到这个事件

   例：ContextRefreshedEvent：容器刷新完成（所有Bean都完全创建）会发布这个事件

4. 发布一个事件 applicationContext.publishEvent()

**实现 `ApplicationListener` 接口实现**

```java
/* 事件对象 */
public class MyApplicationEvent extends ApplicationEvent {

    public MyApplicationEvent(Object source) {
        super(source);
        System.out.println("发送事件:" + super.getTimestamp());
    }
}

/* 事件监听器 */
@Component
public class MyApplicationListener implements ApplicationListener<MyApplicationEvent> {

    /**
     * 当容器中发布事件 MyApplicationEvent 以后，方法触发
     */
    @Override
    public void onApplicationEvent(MyApplicationEvent event) {
        System.out.println("接收事件:" + event.getTimestamp());
    }
}

/* 发布事件 */
@SpringBootTest
public class SpringDemoApplicationTests implements ApplicationContextAware {

    private static ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    @Test
    public void contextLoads() {
        applicationContext.publishEvent(new MyApplicationEvent(this));
    }
}
```

**使用注解 `@EventListener` 实现**

可以使用该注解实现与上面 `MyApplicationListener` 一样的效果

```java
@Service
public class MyService {

    @EventListener
    public void listen(MyApplicationEvent event){
        System.out.println("接收事件:" + event.getTimestamp());
    }
}
```

### 源码解析

> Observer模式，优势在于发布一个事件后可以有多个监听器对其作出反应，对其进行处理

```java
// 1.发布事件
applicationContext.publishEvent(new MyApplicationEvent(this));
// 2.广播这个事件
getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
// 3.获取线程池，后续如果有线程池则异步执行，如果没有则同步执行
Executor executor = getTaskExecutor();
// 4.根据事件和事件类型获取Listeners
getApplicationListeners(event, type);
// 4.1.cacheKey = 事件类型+事件源类型
ListenerCacheKey cacheKey = new ListenerCacheKey(eventType, sourceType);
// 4.2.通过cacheKey查retrieverCache
CachedListenerRetriever existingRetriever = this.retrieverCache.get(cacheKey);
// 4.2.1.如果查不到CachedListenerRetriever则就新建并填充，然后put到retrieverCache
CachedListenerRetriever newRetriever = new CachedListenerRetriever();
retriever.applicationListeners = filteredListeners;
retriever.applicationListenerBeans = filteredListenerBeans;
// 4.3.最终返回符合条件的Listeners
return retrieveApplicationListeners(eventType, sourceType, newRetriever);
// 5.遍历这些Listeners，调用每个Listener
for (ApplicationListener<?> listener : getApplicationListeners(event, type))
```

## Spring 循环依赖

循环依赖异常：BeanCurrentlyInCreationException

循环依赖指的是 默认的单例Bean中，属性互相引用的场景。在Spring中如果使用构造方法注入，或是实例化Bean的时候指定Scope为prototype等情况，就会可能出现循环依赖的问题。

Spring容器内部是通过3级缓存来解决循环依赖 -- `DefaultSingletonBeanRegistry`

一级缓存（singletonObjects）：存放已经经历了完整生命周期的Bean对象

二级缓存（earlySingtonObjects）：存放早期暴露出来的Bean对象（Bean的属性还未赋值）

三级缓存（singletonFacoties）：存放可以生成Bean的工厂

只有单例的Bean会通过三级缓存提前暴露来解决循环依赖的问题，而非单例的Bean，每次从容器中获取的都是一个新的对象，都会重新创建，所有非单例的Bean是没有缓存的，不会将其放到三级缓存中。

过程：

1. A创建的过程中需要B，于是A将自己放到三级缓存里面，去实例化B	
2. B实例化的时候发现需要A，于是B先查一级缓存，没有再查二级缓存，还是没有再查三级缓存，找到A然后把三级缓存里面的A放到二级缓存里面，并删除三级缓存里的A
3. B顺利初始化完毕，将自己放到一级缓存里面（此时B里面的A依然是创建中状态），然后回来接着创建A，此时B已经创建结束，直接从一级缓存里面拿到B，然后完成创建，并将A自己放到一级缓存里面。

总结：Spring解决循环依赖依靠的是Bean的"中间态"的概念，"中间态"指的是已经实例化但还没初始化的状态。

**源码说明**

```java
protected void addSingleton(String beanName, Object singletonObject) {
    synchronized (this.singletonObjects) {
        // 加入到单例缓存池中
        this.singletonObjects.put(beanName, singletonObject);
        // 从三级缓存中移除（针对不处理循环依赖的Bean）
        this.singletonFactories.remove(beanName);
        // 从二级缓存中移除（针对循环依赖的Bean）
        this.earlySingletonObjects.remove(beanName);
        // 用来记录已经处理的Bean
        this.registeredSingletons.add(beanName);
    }
}
```

## Spring IOC 附录

**@Conditional：根据条件加载Bean**

```java
@Bean
@Conditional(MyCondition.class)
public User user(){
    return new User();
}
```

```java
public class MyCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        // 做一些判断逻辑，true表示加载bean，false表示不加载bean
        return true;
    }
}
```

**@DependsOn**

```java
// 创建B时会先去创建A
@Bean
@DependsOn("a")
public B b(){
    return new B();
}
```

**Filter和Interceptor的区别**

- Filter是基于函数回调的，而Interceptor则是基于Java反射的。
- Filter依赖于Servlet容器，而Interceptor不依赖于Servlet容器。
- Filter对几乎所有的请求起作用，而Interceptor只能对action请求起作用。
- Interceptor可以访问Action的上下文，值栈里的对象，而Filter不能。
- 在action的生命周期里，Interceptor可以被多次调用，而Filter只能在容器初始化时调用一次，

**Filter生命周期方法**

1. init : 服务器启动后创建Filter对象，然后调用init方法，只执行一次
2. doFilter : 每一次请求被拦截资源时，会执行，执行多次
3. destroy : 在服务器关闭后，Filter对象被销毁，若服务器正常关闭会执行destroy方法用于释放资源

配置拦截路径 `@WebFilter("/*")`

