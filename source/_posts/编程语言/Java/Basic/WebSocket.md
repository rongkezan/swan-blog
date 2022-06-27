---
title: Websocket
date: {{ date }}
categories:
- 编程语言
- Java
---

## WebSocket Server

> 基于 Spring Boot 的 WebSocket Server

引入依赖

```xml
<!-- Spring Boot Web -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- Spring Boot Web Socket -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

启动类

```java
@SpringBootApplication
public class ServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServerApplication.class, args);
    }
}
```

WebSocket 配置类

```java
@Configuration
public class WebSocketConfig {

    @Bean
    public ServerEndpointExporter serverEndpointExporter(){
        return new ServerEndpointExporter();
    }
}
```

WebSocket 服务

```java
/**
 * 广播服务
 */
@Slf4j
@Component
@ServerEndpoint("/multicast")
public class WebSocketMulticastServer {

    /** 记录当前在线连接数 */
    private static AtomicInteger onlineCount = new AtomicInteger(0);

    /** 存放所有在线的客户端 */
    private static Map<String, Session> clients = new ConcurrentHashMap<>();

    /**
     * 连接建立成功调用的方法
     */
    @OnOpen
    public void onOpen(Session session) {
        onlineCount.incrementAndGet(); // 在线数加1
        clients.put(session.getId(), session);
        log.info("有新连接加入：{}，当前在线人数为：{}", session.getId(), onlineCount.get());
    }

    /**
     * 连接关闭调用的方法
     */
    @OnClose
    public void onClose(Session session) {
        onlineCount.decrementAndGet(); // 在线数减1
        clients.remove(session.getId());
        log.info("有一连接关闭：{}，当前在线人数为：{}", session.getId(), onlineCount.get());
    }

    /**
     * 收到客户端消息后调用的方法
     *
     * @param message
     *            客户端发送过来的消息
     */
    @OnMessage
    public void onMessage(String message, Session session) {
        log.info("服务端收到客户端[{}]的消息:{}", session.getId(), message);
        this.sendMessage(message, session);
    }

    @OnError
    public void onError(Session session, Throwable error) {
        log.error("发生错误", error);
    }

    /**
     * 广播消息
     *
     * @param message 消息内容
     */
    private void sendMessage(String message, Session fromSession) {
        for (Map.Entry<String, Session> sessionEntry : clients.entrySet()) {
            Session toSession = sessionEntry.getValue();
            // 排除掉自己
            if (!fromSession.getId().equals(toSession.getId())) {
                log.info("服务端给客户端[{}]发送消息{}", toSession.getId(), message);
                toSession.getAsyncRemote().sendText(message);
            }
        }
    }
}
```

## WebSocket Client (基于 Js)

```html
<!DOCTYPE HTML>
<html>
<head>
    <title>My WebSocket</title>
</head>

<body>
<input id="text" type="text" />
<button onclick="send()">Send</button>
<button onclick="closeWebSocket()">Close</button>
<div id="message"></div>
</body>

<script type="text/javascript">
    var websocket = null;

    //判断当前浏览器是否支持WebSocket, 主要此处要更换为自己的地址
    if ('WebSocket' in window) {
        websocket = new WebSocket("ws://localhost:18092/multicast");
    } else {
        alert('Not support websocket')
    }

    //连接发生错误的回调方法
    websocket.onerror = function() {
        setMessageInnerHTML("error");
    };

    //连接成功建立的回调方法
    websocket.onopen = function(event) {
        //setMessageInnerHTML("open");
    }

    //接收到消息的回调方法
    websocket.onmessage = function(event) {
        console.log(1)
        console.log(event.data)
        setMessageInnerHTML(event.data);
    }

    //连接关闭的回调方法
    websocket.onclose = function() {
        setMessageInnerHTML("close");
    }

    //监听窗口关闭事件，当窗口关闭时，主动去关闭websocket连接，防止连接还没断开就关闭窗口，server端会抛异常。
    window.onbeforeunload = function() {
        websocket.close();
    }

    //将消息显示在网页上
    function setMessageInnerHTML(innerHTML) {
        document.getElementById('message').innerHTML += innerHTML + '<br/>';
    }

    //关闭连接
    function closeWebSocket() {
        websocket.close();
    }

    //发送消息
    function send() {
        var message = document.getElementById('text').value;
        websocket.send(message);
    }
</script>
</html>
```

## WebSocket Client (基于 Java)

引入依赖

```xml
<!-- Web Socket Client -->
<dependency>
    <groupId>org.java-websocket</groupId>
    <artifactId>Java-WebSocket</artifactId>
    <version>1.3.8</version>
</dependency>
```

启动类

```java
@Slf4j
public class WebSocketClientTest {

    private static final String URL = "ws://localhost:18092/multicast";

    public static void main(String[] args) throws Exception {
        WsClient wsClient = new WsClient(new URI(URL));
        wsClient.connect();
        // 判断是否连接成功，未成功后面发送消息时会报错
        while (!wsClient.getReadyState().equals(WebSocket.READYSTATE.OPEN)) {
            System.out.println("连接中···请稍后");
            Thread.sleep(1000);
        }
        wsClient.send("Java WebSocket Client");
        log.info("消息发送成功");
    }

    @Slf4j
    private static class WsClient extends WebSocketClient {

        public WsClient(URI serverUri) {
            super(serverUri);
        }

        @Override
        public void onOpen(ServerHandshake serverHandshake) {
            log.info("握手成功");
        }

        @Override
        public void onMessage(String s) {
            log.info("收到消息：{}", s);
        }

        @Override
        public void onClose(int i, String s, boolean b) {
            log.info("连接关闭");
        }

        @Override
        public void onError(Exception e) {
            log.info("发生错误", e);
        }
    }
}
```

## 启动测试

当一个客户端发消息时，其它所有客户端都可以接收消息

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210411182945969.png)