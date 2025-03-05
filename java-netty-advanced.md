# Day 116: Java高性能架构 - Netty网络编程

## 1. Netty简介

Netty是一个异步事件驱动的网络应用程序框架，用于快速开发可维护的高性能协议服务器和客户端。它极大地简化了网络编程，如TCP和UDP套接字服务器的开发。

## 2. Netty核心组件

### 2.1 Channel和ChannelPipeline

```java
public class NettyServer {
    public void start() throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) {
                        ChannelPipeline p = ch.pipeline();
                        p.addLast(new StringDecoder());
                        p.addLast(new StringEncoder());
                        p.addLast(new BusinessHandler());
                    }
                });
            
            ChannelFuture f = b.bind(8080).sync();
            f.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```

### 2.2 ByteBuf

```java
public class ByteBufExample {
    public void handleBuffer() {
        ByteBuf buf = Unpooled.buffer(256);
        
        // 写入数据
        buf.writeBytes("Hello Netty".getBytes());
        buf.writeInt(42);
        
        // 读取数据
        byte[] bytes = new byte[buf.readableBytes()];
        buf.readBytes(bytes);
        int number = buf.readInt();
        
        // 释放资源
        buf.release();
    }
}
```

## 3. 编解码器

### 3.1 自定义协议编解码

```java
public class CustomProtocolDecoder extends ByteToMessageDecoder {
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
        // 确保有足够的字节可读
        if (in.readableBytes() < 8) {
            return;
        }
        
        // 标记当前读取位置
        in.markReaderIndex();
        
        // 读取消息长度
        int length = in.readInt();
        
        // 检查是否有足够的字节
        if (in.readableBytes() < length) {
            in.resetReaderIndex();
            return;
        }
        
        // 读取消息内容
        byte[] content = new byte[length];
        in.readBytes(content);
        
        // 解析消息对象
        Message message = parseMessage(content);
        out.add(message);
    }
}

public class CustomProtocolEncoder extends MessageToByteEncoder<Message> {
    @Override
    protected void encode(ChannelHandlerContext ctx, Message msg, ByteBuf out) {
        // 序列化消息对象
        byte[] content = serializeMessage(msg);
        
        // 写入消息长度和内容
        out.writeInt(content.length);
        out.writeBytes(content);
    }
}
```

## 4. 心跳检测

```java
public class HeartbeatHandler extends ChannelInboundHandlerAdapter {
    private static final int READ_IDLE_TIME = 60; // 秒
    
    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        // 添加IdleStateHandler
        ctx.pipeline().addFirst(new IdleStateHandler(READ_IDLE_TIME, 0, 0));
    }
    
    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) {
        if (evt instanceof IdleStateEvent) {
            IdleStateEvent e = (IdleStateEvent) evt;
            if (e.state() == IdleState.READER_IDLE) {
                // 发送心跳请求
                ctx.writeAndFlush(new HeartbeatMessage())
                    .addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            }
        }
    }
}
```

## 5. 性能优化

### 5.1 内存优化

```java
public class MemoryOptimization {
    public void configureMemory() {
        // 使用内存池
        ByteBufAllocator allocator = PooledByteBufAllocator.DEFAULT;
        
        // 配置内存池参数
        ServerBootstrap bootstrap = new ServerBootstrap()
            .option(ChannelOption.ALLOCATOR, allocator)
            .childOption(ChannelOption.ALLOCATOR, allocator)
            .childOption(ChannelOption.RCVBUF_ALLOCATOR, 
                new AdaptiveRecvByteBufAllocator());
    }
}
```

### 5.2 线程模型优化

```java
public class ThreadModelOptimization {
    public void configureThreads() {
        // 计算线程数
        int processors = Runtime.getRuntime().availableProcessors();
        
        // 配置线程组
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup(processors * 2);
        
        // 配置线程参数
        ServerBootstrap bootstrap = new ServerBootstrap()
            .group(bossGroup, workerGroup)
            .option(ChannelOption.SO_BACKLOG, 1024)
            .childOption(ChannelOption.SO_KEEPALIVE, true)
            .childOption(ChannelOption.TCP_NODELAY, true);
    }
}
```

## 6. 实战案例

### 6.1 WebSocket服务器

```java
public class WebSocketServer {
    public void start() {
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) {
                        ChannelPipeline p = ch.pipeline();
                        p.addLast(new HttpServerCodec());
                        p.addLast(new HttpObjectAggregator(65536));
                        p.addLast(new WebSocketServerProtocolHandler("/websocket"));
                        p.addLast(new WebSocketFrameHandler());
                    }
                });
            
            Channel ch = b.bind(8080).sync().channel();
            ch.closeFuture().sync();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}

public class WebSocketFrameHandler extends SimpleChannelInboundHandler<WebSocketFrame> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, WebSocketFrame frame) {
        if (frame instanceof TextWebSocketFrame) {
            String request = ((TextWebSocketFrame) frame).text();
            ctx.channel().writeAndFlush(new TextWebSocketFrame("服务器收到消息: " + request));
        } else {
            String message = "不支持的消息类型: " + frame.getClass().getName();
            throw new UnsupportedOperationException(message);
        }
    }
}
```

### 6.2 高性能RPC框架

```java
public class RpcServer {
    private final Map<String, Object> serviceRegistry = new ConcurrentHashMap<>();
    
    public void registerService(String serviceName, Object serviceImpl) {
        serviceRegistry.put(serviceName, serviceImpl);
    }
    
    public void start(int port) {
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) {
                        ChannelPipeline p = ch.pipeline();
                        p.addLast(new RpcDecoder());
                        p.addLast(new RpcEncoder());
                        p.addLast(new RpcHandler(serviceRegistry));
                    }
                });
            
            ChannelFuture f = b.bind(port).sync();
            f.channel().closeFuture().sync();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}

public class RpcHandler extends SimpleChannelInboundHandler<RpcRequest> {
    private final Map<String, Object> serviceRegistry;
    
    public RpcHandler(Map<String, Object> serviceRegistry) {
        this.serviceRegistry = serviceRegistry;
    }
    
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, RpcRequest request) {
        RpcResponse response = new RpcResponse();
        response.setRequestId(request.getRequestId());
        
        try {
            Object result = handleRequest(request);
            response.setResult(result);
        } catch (Exception e) {
            response.setError(e);
        }
        
        ctx.writeAndFlush(response);
    }
    
    private Object handleRequest(RpcRequest request) throws Exception {
        // 获取服务实现
        Object service = serviceRegistry.get(request.getServiceName());
        if (service == null) {
            throw new RuntimeException("Service not found: " + request.getServiceName());
        }
        
        // 调用方法
        Method method = service.getClass().getMethod(request.getMethodName(), request.getParameterTypes());
        return method.invoke(service, request.getParameters());
    }
}
```

## 7. 最佳实践

### 7.1 异常处理

```java
public class ExceptionHandler extends ChannelDuplexHandler {
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        if (cause instanceof DecoderException) {
            // 处理解码异常
            log.error("解码异常", cause);
        } else if (cause instanceof IOException) {
            // 处理IO异常
            log.error("IO异常", cause);
        } else {
            // 处理其他异常
            log.error("未知异常", cause);
        }
        
        // 关闭连接
        ctx.close();
    }
}
```

### 7.2 优雅关闭

```java
public class GracefulShutdown {
    private final EventLoopGroup bossGroup;
    private final EventLoopGroup workerGroup;
    private final List<Channel> channels;
    
    public void shutdown() {
        // 停止接受新的连接
        channels.forEach(Channel::close);
        
        // 等待所有连接处理完成
        Future<?> bossFuture = bossGroup.shutdownGracefully();
        Future<?> workerFuture = workerGroup.shutdownGracefully();
        
        try {
            bossFuture.await(10, TimeUnit.SECONDS);
            workerFuture.await(10, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

## 8. 总结

Netty是一个强大的网络应用框架，在实践中需要注意：

1. 正确理解和使用Netty的核心组件
2. 实现可靠的协议编解码
3. 注意内存管理和性能优化
4. 实现完善的异常处理机制
5. 保证应用的优雅关闭

## 参考资源

1. Netty官方文档：https://netty.io/wiki/
2. Netty实战：https://book.douban.com/subject/27038538/
3. Netty源码分析：https://github.com/netty/netty
4. Java NIO编程：https://docs.oracle.com/javase/tutorial/essential/io/