# Day 29: Java Web开发 - WebSocket

## 引言

WebSocket是HTML5提供的一种在单个TCP连接上进行全双工通讯的协议。WebSocket使得客户端和服务器之间的数据交换变得更加简单，允许服务端主动向客户端推送数据。在本文中，我们将深入探讨Java中的WebSocket编程。

## 1. WebSocket基础

### 1.1 什么是WebSocket

WebSocket是一种网络通信协议，具有以下特点：

- 建立在TCP协议之上
- 与HTTP协议有着良好的兼容性
- 数据格式比较轻量，性能开销小
- 可以发送文本，也可以发送二进制数据
- 没有同源限制，客户端可以与任意服务器通信

### 1.2 为什么需要WebSocket

- 实时性要求高的场景
- 需要服务器推送的场景
- 减少通信量

## 2. Java WebSocket API

### 2.1 服务端实现

```java
import javax.websocket.*;
import javax.websocket.server.ServerEndpoint;

@ServerEndpoint("/websocket")
public class WebSocketServer {

    @OnOpen
    public void onOpen(Session session) {
        System.out.println("WebSocket opened: " + session.getId());
    }

    @OnMessage
    public void onMessage(String message, Session session) {
        System.out.println("Message from " + session.getId() + ": " + message);
        try {
            session.getBasicRemote().sendText("Server received: " + message);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @OnClose
    public void onClose(Session session) {
        System.out.println("WebSocket closed: " + session.getId());
    }

    @OnError
    public void onError(Throwable error) {
        System.err.println("WebSocket error: " + error.getMessage());
    }
}
```

### 2.2 客户端实现

```java
import javax.websocket.*;

@ClientEndpoint
public class WebSocketClient {

    private Session session;

    @OnOpen
    public void onOpen(Session session) {
        this.session = session;
        System.out.println("Connected to server");
    }

    @OnMessage
    public void onMessage(String message) {
        System.out.println("Received message: " + message);
    }

    public void sendMessage(String message) {
        try {
            session.getBasicRemote().sendText(message);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

## 3. Spring WebSocket支持

### 3.1 配置WebSocket

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.config.annotation.*;

@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(new MyWebSocketHandler(), "/websocket")
               .setAllowedOrigins("*");
    }
}
```

### 3.2 实现WebSocket处理器

```java
import org.springframework.web.socket.*;
import org.springframework.web.socket.handler.TextWebSocketHandler;

public class MyWebSocketHandler extends TextWebSocketHandler {

    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        // 连接建立后调用
    }

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) {
        // 处理文本消息
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
        // 连接关闭后调用
    }
}
```

## 4. 实践案例：实时聊天室

### 4.1 服务端代码

```java
import java.util.concurrent.CopyOnWriteArraySet;

@ServerEndpoint("/chat")
public class ChatServer {
    private static final CopyOnWriteArraySet<Session> sessions = new CopyOnWriteArraySet<>();

    @OnOpen
    public void onOpen(Session session) {
        sessions.add(session);
        broadcast("User " + session.getId() + " joined");
    }

    @OnMessage
    public void onMessage(String message, Session session) {
        broadcast("User " + session.getId() + ": " + message);
    }

    @OnClose
    public void onClose(Session session) {
        sessions.remove(session);
        broadcast("User " + session.getId() + " left");
    }

    private void broadcast(String message) {
        sessions.forEach(session -> {
            try {
                session.getBasicRemote().sendText(message);
            } catch (IOException e) {
                e.printStackTrace();
            }
        });
    }
}
```

## 5. WebSocket最佳实践

1. 心跳机制
```java
public class HeartbeatHandler {
    private static final long HEARTBEAT_INTERVAL = 30000; // 30秒

    public void startHeartbeat(Session session) {
        new Thread(() -> {
            while (session.isOpen()) {
                try {
                    session.getBasicRemote().sendText("ping");
                    Thread.sleep(HEARTBEAT_INTERVAL);
                } catch (Exception e) {
                    break;
                }
            }
        }).start();
    }
}
```

2. 错误处理
3. 连接管理
4. 消息格式规范
5. 安全考虑

## 6. 性能优化

1. 使用消息队列
2. 连接池管理
3. 消息压缩
4. 适当的并发控制

## 总结

本文介绍了Java WebSocket编程的核心概念和实践应用，包括：

1. WebSocket基础知识
2. Java WebSocket API的使用
3. Spring WebSocket支持
4. 实时聊天室实现
5. 最佳实践和性能优化

通过WebSocket技术，我们可以实现更加实时和高效的双向通信，这在很多现代Web应用中都是非常重要的功能。

## 参考资源

1. Java WebSocket API文档：https://docs.oracle.com/javaee/7/api/javax/websocket/package-summary.html
2. Spring WebSocket文档：https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#websocket
3. WebSocket协议规范：https://tools.ietf.org/html/rfc6455
4. WebSocket最佳实践指南