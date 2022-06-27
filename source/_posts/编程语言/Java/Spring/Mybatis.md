---
title: Mybatis
date: {{ date }}
categories:
- 编程语言
- Java
---

# Mybatis

## SqlSession

每当我们使用MyBatis开启一次和数据库的会话，MyBatis会创建出一个SqlSession对象表示一次数据库会话。

在对数据库的一次会话中，我们有可能会反复地执行完全相同的查询语句，如果不采取一些措施的话，每一次查询都会查询一次数据库,而我们在极短的时间内做了完全相同的查询，那么它们的结果极有可能完全相同，由于查询一次数据库的代价很大，这有可能造成很大的资源浪费。

## Mybatis 缓存

### 一级缓存（默认开启）

本地缓存，MyBatis会在表示会话的SqlSession对象中建立一个简单的缓存，将每次查询到的结果结果缓存起来，当下次查询的时候，如果判断先前有个完全一样的查询，会直接从缓存中直接将结果取出返回给用户，不需要再进行一次数据库查询了，执行增删改后失效。

**一级缓存生命周期**

1. MyBatis在开启一个数据库会话时，会创建一个新的SqlSession对象，SqlSession对象中会有一个新的Executor对象，Executor对象中持有一个新的PerpetualCache对象；当会话结束时，SqlSession对象及其内部的Executor对象还有PerpetualCache对象也一并释放掉。
2. 如果SqlSession调用了close方法，会释放掉一级缓存PerpetualCache对象，一级缓存将不可用；
3. 如果SqlSession调用了clearCache，会清空PerpetualCache对象中的数据，但是该对象仍可使用；
4. SqlSession中执行了任何一个update操作(update、delete、insert) ，都会清空PerpetualCache对象的数据，但是该对象可以继续使用；

**一级缓存的两种级别**

1. session 级别的缓存，在同一个 sqlSession 内，对同样的查询将不再查询数据库，直接从缓存中。
2. statement 级别的缓存，避坑： 为了避免这个问题，可以将一级缓存的级别设为 statement 级别的，这样每次查询结束都会清掉一级缓存。

### 二级缓存

**开启二级缓存**

增加配置

```properties
mybatis.configuration.cache-enabled=true
```

在 Mapper.xml 中配置 cache 标签

```xml
<!-- 
各个参数解释
type: 指定自定义缓存的全类名(实现C)
size: 缓存存放多少个元素
eviction: 缓存回收策略：LRU、FIFO、SOFT、WEAK
	LRU(默认): 最近最少未回收
	FIFO: 先进先出，按照缓存进入的顺序来移除它们
	SOFT: 软引用，移除基于垃圾回收器状态和软引用规则的对象
	WEAK: 弱引用，更积极地移除基于垃圾回收器和弱引用规则的对象
flushInterval: 单位ms，缓存刷新间隔，缓存多少时间刷新一次，默认不清空
blocking: 若缓存中找不到对应的key，是否会一直blocking，直到有对应的数据进入缓存
-->

<cache type="org.apache.ibatis.cache.impl.PerpetualCache"
       size="1024"
       eviction="LRU"
       flushInterval="120000"
       readOnly="false"/>
```

**二级缓存原理**

二级缓存是用来解决一级缓存不能跨会话共享的问题的，范围是namespace 级别的，可以被多个SqlSession 共享（只要是同一个接口里面的相同方法，都可以共享），生命周期和应用同步。如果你的MyBatis使用了二级缓存，并且你的Mapper和select语句也配置使用了二级缓存，那么在执行select查询的时候，MyBatis会先从二级缓存中取，其次才是一级缓存，即MyBatis查询数据的顺序是：二级缓存  > 一级缓存 > 数据库。

MyBatis 用了一个装饰器的类来维护二级缓存，就是CachingExecutor。如果启用了二级缓存，MyBatis 在创建Executor 对象的时候会对Executor 进行装饰。CachingExecutor 对于查询请求，会判断二级缓存是否有缓存结果，如果有就直接返回，如果没有委派交给真正的查询器Executor 实现类，比如SimpleExecutor 来执行查询，再走到一级缓存的流程。最后会把结果缓存起来，并且返回给用户。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210208104315871.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

**不推荐开启二级缓存**

因为二级缓存是基于 namespace 的（即一个 Mapper.xml 文件），所有基于该 Mapper 的增删改操作都会刷新缓存，但是如果其他的 Mapper 中有对该 Mapper 中数据表的操作，就会导致两个 namesapce 中的数据不一致。推荐使用 redis 替代。

## Mybatis 批量删除

```java
int deleteByIds(@Param("ids") List<Long> ids);
```

```xml
<delete id="deleteByIds" parameterType="java.util.List">
    delete from DUB_ORIGIN_ATTACH
    <where>
        id in
        <foreach collection="ids" item="id" separator="," open="(" close=")">
            #{id}
        </foreach>
    </where>
</delete>
```

## Mybatis 多数据源配置

```properties
dub.datasource.oracle.driver-class-name=oracle.jdbc.driver.OracleDriver
dub.datasource.oracle.url=jdbc:oracle:thin:@127.0.0.1:1521:test
dub.datasource.oracle.username=root
dub.datasource.oracle.password=root

dub.datasource.mysql.driver-class-name=com.mysql.jdbc.Driver
dub.datasource.mysql.url=jdbc:mysql://127.0.0.1:3306/test?characterEncoding=utf-8&useSSL=false
dub.datasource.mysql.username=root
dub.datasource.mysql.password=root
```

```java
@Configuration
@MapperScan(basePackages ="com.smartebao.dub.export.declare.common.mapper.oracle", sqlSessionTemplateRef  = "oracleSqlSessionTemplate")
public class DubOracleDataSourceConfig {

    @Primary
    @Bean("oracleSqlSessionFactory")
    public SqlSessionFactory ds1SqlSessionFactory(@Qualifier("oracleDataSource") DataSource dataSource) throws Exception {
        MybatisSqlSessionFactoryBean sqlSessionFactory = new MybatisSqlSessionFactoryBean();
        sqlSessionFactory.setDataSource(dataSource);
        MybatisConfiguration configuration = new MybatisConfiguration();
        configuration.setDefaultScriptingLanguage(MybatisXMLLanguageDriver.class);
        configuration.setJdbcTypeForNull(JdbcType.NULL);
        sqlSessionFactory.setConfiguration(configuration);
        sqlSessionFactory.setMapperLocations(new PathMatchingResourcePatternResolver().
                getResources("classpath*:com/smartbao/dub/export/declare/common/mapper/oracle/**"));
        sqlSessionFactory.setGlobalConfig(new GlobalConfig().setBanner(false));
        return sqlSessionFactory.getObject();
    }

    @Primary
    @Bean(name = "oracleTransactionManager")
    public DataSourceTransactionManager ds1TransactionManager(@Qualifier("oracleDataSource") DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }

    @Primary
    @Bean(name = "oracleSqlSessionTemplate")
    public SqlSessionTemplate ds1SqlSessionTemplate(@Qualifier("oracleSqlSessionFactory") SqlSessionFactory sqlSessionFactory) {
        return new SqlSessionTemplate(sqlSessionFactory);
    }
}
```

```java
@Configuration
@MapperScan(basePackages ="com.smartebao.dub.export.declare.common.mapper.mysql", sqlSessionTemplateRef  = "mysqlSqlSessionTemplate")
public class DubMysqlDataSourceConfig {

    @Bean("mysqlSqlSessionFactory")
    public SqlSessionFactory ds1SqlSessionFactory(@Qualifier("mysqlDataSource") DataSource dataSource) throws Exception {
        MybatisSqlSessionFactoryBean sqlSessionFactory = new MybatisSqlSessionFactoryBean();
        sqlSessionFactory.setDataSource(dataSource);
        MybatisConfiguration configuration = new MybatisConfiguration();
        configuration.setDefaultScriptingLanguage(MybatisXMLLanguageDriver.class);
        configuration.setJdbcTypeForNull(JdbcType.NULL);
        sqlSessionFactory.setConfiguration(configuration);
        sqlSessionFactory.setMapperLocations(new PathMatchingResourcePatternResolver().
                getResources("classpath*:com/smartbao/dub/export/declare/common/mapper/mysql/**"));
        sqlSessionFactory.setGlobalConfig(new GlobalConfig().setBanner(false));
        return sqlSessionFactory.getObject();
    }

    @Bean(name = "mysqlTransactionManager")
    public DataSourceTransactionManager ds1TransactionManager(@Qualifier("mysqlDataSource") DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }

    @Bean(name = "mysqlSqlSessionTemplate")
    public SqlSessionTemplate ds1SqlSessionTemplate(@Qualifier("mysqlSqlSessionFactory") SqlSessionFactory sqlSessionFactory) {
        return new SqlSessionTemplate(sqlSessionFactory);
    }
}
```

```java
@Configuration
@MapperScan(basePackages ="com.smartebao.dub.export.declare.common.mapper.oracle", sqlSessionTemplateRef  = "oracleSqlSessionTemplate")
public class DubOracleDataSourceConfig {

    @Primary
    @Bean("oracleSqlSessionFactory")
    public SqlSessionFactory ds1SqlSessionFactory(@Qualifier("oracleDataSource") DataSource dataSource) throws Exception {
        MybatisSqlSessionFactoryBean sqlSessionFactory = new MybatisSqlSessionFactoryBean();
        sqlSessionFactory.setDataSource(dataSource);
        MybatisConfiguration configuration = new MybatisConfiguration();
        configuration.setDefaultScriptingLanguage(MybatisXMLLanguageDriver.class);
        configuration.setJdbcTypeForNull(JdbcType.NULL);
        sqlSessionFactory.setConfiguration(configuration);
        sqlSessionFactory.setMapperLocations(new PathMatchingResourcePatternResolver().
                getResources("classpath*:com/smartbao/dub/export/declare/common/mapper/oracle/**"));
        sqlSessionFactory.setGlobalConfig(new GlobalConfig().setBanner(false));
        return sqlSessionFactory.getObject();
    }

    @Primary
    @Bean(name = "oracleTransactionManager")
    public DataSourceTransactionManager ds1TransactionManager(@Qualifier("oracleDataSource") DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }

    @Primary
    @Bean(name = "oracleSqlSessionTemplate")
    public SqlSessionTemplate ds1SqlSessionTemplate(@Qualifier("oracleSqlSessionFactory") SqlSessionFactory sqlSessionFactory) {
        return new SqlSessionTemplate(sqlSessionFactory);
    }
}
```

