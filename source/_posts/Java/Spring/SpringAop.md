---
title: Spring AOP
date: {{ date }}
categories:
- Spring
---

# Spring AOP 

开启Aop 使用 `@Aspect` 标注其为切面类，并把该类加入容器

```java
@Aspect
@Component
public class SpringAspect {
    
}
```

在配置类中开启Aop模式 `@EnableAspectJAutoProxy`

```java
@Configuration
@EnableAspectJAutoProxy
public class SpringConfig {

}
```

## Spring AOP常用注解

@Before：前置通知，目标方法之前执行

@After：后置通知，目标方法之后执行（必然执行）

@AfterReturning：返回后通知，执行方法结束前执行（异常不执行）

@AfterThrowing：异常通知，出现异常时执行

@Around：环绕通知，环绕目标方法执行

## Spring Aop执行顺序

**Spring4**

正常：@Around @Before @Around @After @AfterReturning

异常：@Around @Before @After @AfterThrowing

**Spring5**

正常：@Around @Before @AfterReturning @After @Around

异常：@Around @Before @AfterThrowing @After

## Spring Aop 原理

> 使用动态代理执行目标方法

### 执行流程

```
1. @EnableAspectJAutoProxy 为容器中增加一个 AspectJAutoProxyRegistrar 类
2. 容器创建
	2.1 registerBeanPostProcessors() 注册后置处理器
	2.2 finishBeanFactoryInitialization() 初始化剩下的单实例Bean
		2.2.1 创建业务逻辑和切面组件
		2.2.2 AnnotationAwareAspectJAutoProxyCreator拦截组件创建过程
		2.2.3 组件创建完之后，判断组件若需要增强，切面通知方法包装成Advisor，给目标对象创建一个代理对象（默认使用cglib创建）
3. 代理对象执行目标方法 CglibAopProxy.intercept()
	3.1 得到目标方法的拦截器链，包装成拦截器MethodInterceptor
	3.2 利用拦截器的链式机制依次进入每一个拦截器进行执行
```

### 动态代理

Spring 提供了两种方式来生成代理对象: JDKProxy 和 Cglib，具体使用哪种方式生成由AopProxyFactory 根据 AdvisedSupport 对象的配置来决定。默认的策略是如果目标类是接口，则使用 JDK 动态代理技术，否则使用 Cglib 来生成代理。

**JDK动态接口代理**

> 需要有接口

JDK 动态代理主要涉及到 java.lang.reflect 包中的两个类：Proxy 和 InvocationHandler。InvocationHandler是一个接口，通过实现该接口定义横切逻辑，并通过反射机制调用目标类的代码，动态将横切逻辑和业务逻辑编制在一起。Proxy 利用 InvocationHandler 动态创建一个符合某一接口的实例，生成目标类的代理对象。

**CGLib 动态代理**

> 不需要有接口

CGLib 全称为 Code Generation Library，是一个强大的高性能，高质量的代码生成类库，可以在运行期扩展 Java 类与实现 Java 接口，CGLib 封装了 asm，可以再运行期动态生成新的 class。和 JDK 动态代理相比较：JDK 创建代理有一个限制，就是只能为接口创建代理实例，而对于没有通过接口定义业务方法的类，则可以通过 CGLib 创建动态代理。

### Spring 创建代理对象

如果该Bean有Advice则返回代理对象，否则返回普通对象

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
        return bean;
    }
    if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
        return bean;
    }
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }

    // Create proxy if we have advice.
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        Object proxy = createProxy(
            bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}
```

如果是接口，创建JDK代理对象，否则创建Cglib代理对象

```java
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
        if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
            Class<?> targetClass = config.getTargetClass();
            if (targetClass == null) {
                throw new AopConfigException("TargetSource cannot determine target class: " +
                        "Either an interface or a target is required for proxy creation.");
            }
            if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
                return new JdkDynamicAopProxy(config);
            }
            return new ObjenesisCglibAopProxy(config);
        }
        else {
            return new JdkDynamicAopProxy(config);
        }
    }
```

## Spring 事务

### 配置事务步骤

1. 配置数据源

2. 配置事务管理器 `PlatformTransactionManager`

3. 开启事务 `@EnableTransactionManagement`

```java
@Configuration
@EnableTransactionManagement
public class JdbcConfig {

    @Bean
    public DataSource dataSource() {
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setUsername("root");
        dataSource.setPassword("root");
        dataSource.setDriverClassName("com.mysql.jdbc.Driver");
        dataSource.setJdbcUrl("jdbc:mysql//localhost:3306/test");
        return dataSource;
    }

    /**
     * 配置事务管理器
     */
    @Bean
    public PlatformTransactionManager transactionManager(){
        return new DataSourceTransactionManager(dataSource());
    }
}
```

### 事务管理器

Spring只是个容器，因此它并不做任何事务的具体实现。他只是提供了事务管理的接口PlatformTransactionManager，具体内容由就由各个事务管理器来实现。

原理：通过 `TransactionAwareDataSourceProxy` 包装 `DataSource` 

而在 `PlatformTransactionManager` 的实现类中可以操作 `DataSource` ，在 Spring 中实现 `commit` 和 `rollback`

### Transactional 注解参数

```java
/**
 * 事务管理器
 */
String transactionManager() default "";

/**
 * 事务传播行为
 */
Propagation propagation() default Propagation.REQUIRED;

/**
 * 事务隔离级别
 */
Isolation isolation() default Isolation.DEFAULT;

/**
 * 事务的超时时间
 */
int timeout() default TransactionDefinition.TIMEOUT_DEFAULT;

/**
 * 该事务是否为只读
 */
boolean readOnly() default false;

/**
 * 哪种异常需要回滚
 */
Class<? extends Throwable>[] rollbackFor() default {};

/**
 * 哪种异常不需要回滚
 */
Class<? extends Throwable>[] noRollbackFor() default {};
```

### Spring 事务传播行为

```java
/**
 * 若当前事务存在，则在当前事务中运行，否则开启一个新事务
 * Support a current transaction, create a new one if none exists.
 */
REQUIRED(TransactionDefinition.PROPAGATION_REQUIRED),

/**
 * 若当前事务存在，则在当前事务中运行，否则不开启事务
 * Support a current transaction, execute non-transactionally if none exists.
 */
SUPPORTS(TransactionDefinition.PROPAGATION_SUPPORTS),

/**
 * 若当前事务不存在则抛异常
 * Support a current transaction, throw an exception if none exists.
 */
MANDATORY(TransactionDefinition.PROPAGATION_MANDATORY),

/**
 * 开启一个事务并挂起当前事务
 * Create a new transaction, and suspend the current transaction if one exists.
 */
REQUIRES_NEW(TransactionDefinition.PROPAGATION_REQUIRES_NEW),

/**
 * 运行在事务中被挂起
 * Execute non-transactionally, suspend the current transaction if one exists.
 */
NOT_SUPPORTED(TransactionDefinition.PROPAGATION_NOT_SUPPORTED),

/**
 * 在事务中运行将会抛异常
 * Execute non-transactionally, throw an exception if a transaction exists.
 */
NEVER(TransactionDefinition.PROPAGATION_NEVER),

/**
 * 嵌套在当前事务中运行
 * Execute within a nested transaction if a current transaction exists,
 * behave like PROPAGATION_REQUIRED else. There is no analogous feature in EJB.
 */
NESTED(TransactionDefinition.PROPAGATION_NESTED);
```

### Spring 事务失效场景

1. 数据库引擎不支持事务：InnoDB支持事务，MyISAM不支持事务
2. 数据源没配置事务管理器
3. 没有抛出异常，或异常类型错误
4. 方法没有被切面管理，即不是代理对象

```java
@Service
public class OrderServiceImpl implements OrderService {

    @Transactional
    public void update(Order order) {
        updateOrder(order);
    }
    
    // 新开的事务不管用，因为没有被切面管理
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void updateOrder(Order order) {
        // update order
    }
}

/**
 * 解决上述问题可以将新开的事务写在不同类中
 */
@Service
public class ServiceA {
    @Autowired
    private ServiceB serviceB;

    @Transactional
    public void doSomething(){
        serviceB.insert();
    }
}

@Service
public class ServiceB {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void insert(){
        // 向数据库中添加数据
    }
}

/**
 * 解决方案2: 配置文件中加 <aop:config expose-proxy="true">
 * 并且使用如下方式调用:
 * ((ServiceA) AopContext.currentProxy()).insert();
 */
```

### 附录

**声明式事务和编程式事务**

编程式事务：通过硬编码的形式手动控制事务的提交和回滚。

声明式事务：只需告诉Spring哪个方法是事务方法即可。

**Spring事务异常**

运行时异常：可以不用处理，默认都回滚。

编译时异常：要么try-catch，要么thows，默认不回滚。