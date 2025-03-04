# Day 43: Java消息队列 - RabbitMQ实战

## 引言

消息队列是分布式系统中的重要组件，它可以实现系统间的解耦、异步通信和流量削峰。本文将介绍RabbitMQ的核心概念和在Java中的实践应用。

## 1. RabbitMQ基础

### 1.1 核心概念

- Producer：消息生产者
- Consumer：消息消费者
- Exchange：交换机，负责消息路由
- Queue：消息队列，存储消息
- Binding：绑定，连接Exchange和Queue

### 1.2 Exchange类型

- Direct：直接匹配路由键
- Topic：模式匹配路由键
- Fanout：广播模式
- Headers：基于消息属性匹配

## 2. Spring AMQP配置

### 2.1 依赖配置

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

### 2.2 基本配置

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    virtual-host: /
```

### 2.3 Exchange和Queue配置

```java
@Configuration
public class RabbitConfig {
    @Bean
    public DirectExchange orderExchange() {
        return new DirectExchange("order.exchange");
    }
    
    @Bean
    public Queue orderQueue() {
        return new Queue("order.queue");
    }
    
    @Bean
    public Binding orderBinding() {
        return BindingBuilder.bind(orderQueue())
            .to(orderExchange())
            .with("order.routing.key");
    }
}
```

## 3. 消息发送和接收

### 3.1 消息发送

```java
@Service
public class OrderService {
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public void createOrder(Order order) {
        // 业务处理
        rabbitTemplate.convertAndSend("order.exchange", "order.routing.key", order);
    }
}
```

### 3.2 消息接收

```java
@Component
public class OrderConsumer {
    @RabbitListener(queues = "order.queue")
    public void handleOrder(Order order) {
        try {
            // 处理订单消息
            processOrder(order);
        } catch (Exception e) {
            // 异常处理
            handleError(order, e);
        }
    }
}
```

## 4. 高级特性

### 4.1 消息确认机制

```yaml
spring:
  rabbitmq:
    publisher-confirm-type: correlated
    publisher-returns: true
```

```java
@Service
public class MessageService {
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    @PostConstruct
    public void init() {
        rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
            if (!ack) {
                log.error("消息发送失败：{}", cause);
                // 重试逻辑
            }
        });
        
        rabbitTemplate.setReturnCallback((message, replyCode, replyText, exchange, routingKey) -> {
            log.error("消息路由失败：{}", replyText);
            // 处理路由失败
        });
    }
}
```

### 4.2 死信队列

```java
@Configuration
public class DeadLetterConfig {
    @Bean
    public Queue orderQueue() {
        Map<String, Object> args = new HashMap<>();
        args.put("x-dead-letter-exchange", "order.dlx.exchange");
        args.put("x-dead-letter-routing-key", "order.dlx.routing.key");
        return new Queue("order.queue", true, false, false, args);
    }
    
    @Bean
    public DirectExchange deadLetterExchange() {
        return new DirectExchange("order.dlx.exchange");
    }
    
    @Bean
    public Queue deadLetterQueue() {
        return new Queue("order.dlx.queue");
    }
}
```

## 5. 实践案例

### 5.1 延迟消息实现

```java
@Service
public class DelayMessageService {
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public void sendDelayMessage(Message message, long delayTime) {
        rabbitTemplate.convertAndSend("delay.exchange", "delay.routing.key", message,
            msg -> {
                msg.getMessageProperties().setDelay((int) delayTime);
                return msg;
            });
    }
}
```

### 5.2 消息重试机制

```java
@Component
public class RetryConsumer {
    @RabbitListener(queues = "retry.queue",
        containerFactory = "retryListenerContainerFactory")
    public void handleMessage(Message message) {
        try {
            processMessage(message);
        } catch (Exception e) {
            throw new AmqpRejectAndDontRequeueException("处理失败，进入重试队列");
        }
    }
}

@Configuration
public class RetryConfig {
    @Bean
    public SimpleRabbitListenerContainerFactory retryListenerContainerFactory(
            ConnectionFactory connectionFactory) {
        SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);
        
        RetryInterceptorBuilder<?> builder = RetryInterceptorBuilder.stateless()
            .maxAttempts(3)
            .backOffOptions(1000, 2.0, 10000);
        factory.setAdviceChain(builder.build());
        
        return factory;
    }
}
```

## 6. 最佳实践

1. 消息可靠性
   - 使用消息确认机制
   - 实现消息持久化
   - 配置死信队列处理失败消息

2. 性能优化
   - 合理设置预取数量
   - 使用消息批量处理
   - 控制消费者并发数

3. 监控告警
   - 监控队列长度
   - 跟踪消息处理时间
   - 设置告警阈值

## 总结

通过本文的学习，我们掌握了：

1. RabbitMQ的核心概念和工作原理
2. Spring AMQP的基本配置和使用方法
3. 消息的发送和接收处理
4. 高级特性的实现
5. 实际应用案例

在分布式系统中，合理使用消息队列可以提高系统的可扩展性和可靠性，是构建高性能系统的重要手段。

## 参考资源

1. Spring AMQP文档：https://docs.spring.io/spring-amqp/docs/current/reference/html/
2. RabbitMQ官方文档：https://www.rabbitmq.com/documentation.html
3. 消息中间件设计与实现
4. 分布式消息服务实践指南