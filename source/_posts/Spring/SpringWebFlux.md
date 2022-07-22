---
title: SpringWebFlux
date: {{ date }}
categories:
- Spring
---

# Spring WebFlux

开启线程会有1M栈内存的开销，栈内存使用超过1M会导致栈溢出

Tomcat有两个线程池：连接线程池、业务线程池 

长轮询：客户端和服务器建立连接后不断开，等待服务端数据返回，默认90s，超出则续租。

响应式编程 (Reactor实现)

1. 响应式编程操作中，Reactor是满足Reactive规范框架
2. Reactor有两个核心类，Mono和Flux，这两个类实现接口Publisher，提供丰富操作符
   1. Flux返回N个元素
   2. Mono返回0或1个元素
3. Flux和Mono都是数据流的发布者，使用Flux和Mono都可以发出3种数据信号：元素值，错误信号，完成信号。错误信号和完成信号都代表终止信号，终止信号用于告诉订阅者数据流结束了，错误信号终止数据流同时把错误信息传递给订阅者。

**具体实现**

```xml
<dependency>
    <groupId>io.projectreactor</groupId>
    <artifactId>reactor-core</artifactId>
    <version>3.1.5.RELEASE</version>
</dependency>
```

```java
@RestController
@RequestMapping("user")
public class UserController {

    @Resource
    UserService userService;

    @GetMapping("{id}")
    public Mono<User> getUser(@PathVariable Integer id){
        return userService.selectOne(id);
    }

    @GetMapping("list")
    public Flux<User> getUsers(){
        return userService.selectAll();
    }

    @PostMapping("save")
    public Mono<Void> saveUser(@RequestBody User user){
        Mono<User> userMono = Mono.just(user);
        return userService.save(userMono);
    }
}
```

```java
@Service
public class UserServiceImpl implements UserService {

    private final Map<Integer, User> userMap = new HashMap<>();

    public UserServiceImpl(){
        this.userMap.put(1, new User(1, "Tom"));
        this.userMap.put(2, new User(2, "Jerry"));
    }

    @Override
    public Mono<User> selectOne(Integer id) {
        return Mono.justOrEmpty(this.userMap.get(id));
    }

    @Override
    public Flux<User> selectAll() {
        return Flux.fromIterable(this.userMap.values());
    }

    @Override
    public Mono<Void> save(Mono<User> user) {
        return user.doOnNext(person -> {
           int id = userMap.size() + 1;
           userMap.put(id, person);
        }).thenEmpty(Mono.empty());
    }
}
```