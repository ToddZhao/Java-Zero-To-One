# Day 90: Java消息队列实践指南

## 1. 引言

消息队列是分布式系统中不可或缺的组件，用于解耦服务、异步处理和流量削峰。本文将介绍Java中常用的消息队列实践，帮助开发者构建可靠的消息处理系统。

## 2. 消息队列选型

### 2.1 主流消息队列对比

- RabbitMQ：AMQP协议实现，支持多种消息模式
- Kafka：高吞吐量，适合日志收集和流处理
- RocketMQ：阿里开源，金融级可靠性
- ActiveMQ：成熟的消息中间件，JMS规范实现

### 2.2 RabbitMQ实践

```java
@Configuration
public class RabbitConfig {
    @Bean
    public Queue orderQueue() {
        return new Queue("order.queue", true);
    }

    @Bean
    public TopicExchange orderExchange() {
        return new TopicExchange("order.exchange");
    }

    @Bean
    public Binding bindingOrder(Queue orderQueue, TopicExchange orderExchange) {
        return BindingBuilder.bind(orderQueue).to(orderExchange).with("order.#");
    }
}
```

## 3. 消息发送

### 3.1 生产者实现

```java
@Service
public class OrderMessageProducer {
    @Autowired
    private RabbitTemplate rabbitTemplate;

    public void sendOrderMessage(Order order) {
        try {
            rabbitTemplate.convertAndSend("order.exchange", "order.created", order);
            log.info("订单消息发送成功: {}", order.getId());
        } catch (Exception e) {
            log.error("订单消息发送失败", e);
            // 消息发送失败处理
            handleMessageSendFailure(order);
        }
    }

    private void handleMessageSendFailure(Order order) {
        // 实现消息发送失败的重试逻辑
    }
}
```

### 3.2 消息序列化

```java
@Configuration
public class MessageConverter {
    @Bean
    public Jackson2JsonMessageConverter jsonMessageConverter() {
        ObjectMapper objectMapper = new ObjectMapper()
            .registerModule(new JavaTimeModule())
            .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
        return new Jackson2JsonMessageConverter(objectMapper);
    }
}
```

## 4. 消息消费

### 4.1 消费者实现

```java
@Component
@RabbitListener(queues = "order.queue")
public class OrderMessageConsumer {

    @Autowired
    private OrderService orderService;

    @RabbitHandler
    public void handleOrderMessage(Order order, Channel channel, 
            @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws IOException {
        try {
            // 处理订单消息
            orderService.processOrder(order);
            // 确认消息处理成功
            channel.basicAck(tag, false);
        } catch (Exception e) {
            // 处理失败，进行重试或死信处理
            channel.basicNack(tag, false, true);
        }
    }
}
```

### 4.2 消费者配置

```java
@Configuration
public class ConsumerConfig {
    @Bean
    public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(
            ConnectionFactory connectionFactory,
            Jackson2JsonMessageConverter jsonMessageConverter) {
        SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);
        factory.setMessageConverter(jsonMessageConverter);
        factory.setPrefetchCount(1);
        factory.setAcknowledgeMode(AcknowledgeMode.MANUAL);
        return factory;
    }
}
```

## 5. 可靠性保证

### 5.1 生产者确认

```java
@Configuration
public class RabbitConfirmConfig {
    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
        rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
            if (!ack) {
                log.error("消息发送到交换机失败: {}", cause);
                // 处理消息发送失败的情况
            }
        });
        rabbitTemplate.setReturnCallback((message, replyCode, replyText, exchange, routingKey) -> {
            log.error("消息路由到队列失败: exchange: {}, route: {}, replyCode: {}, replyText: {}, message: {}",
                    exchange, routingKey, replyCode, replyText, message);
            // 处理消息路由失败的情况
        });
        return rabbitTemplate;
    }
}
```

### 5.2 死信队列处理

```java
@Configuration
public class DeadLetterConfig {
    @Bean
    public Queue deadLetterQueue() {
        return QueueBuilder.durable("order.dead.letter.queue")
                .build();
    }

    @Bean
    public Queue orderQueue() {
        return QueueBuilder.durable("order.queue")
                .withArgument("x-dead-letter-exchange", "")
                .withArgument("x-dead-letter-routing-key", "order.dead.letter.queue")
                .withArgument("x-message-ttl", 60000) // 1分钟TTL
                .build();
    }
}
```

## 6. 性能优化

### 6.1 批量处理

```java
@Component
@RabbitListener(queues = "order.queue")
public class BatchOrderConsumer {
    @RabbitHandler
    public void handleBatchOrders(@Payload List<Order> orders,
                                 Channel channel,
                                 @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws IOException {
        try {
            // 批量处理订单
            orderService.processBatchOrders(orders);
            channel.basicAck(tag, false);
        } catch (Exception e) {
            channel.basicNack(tag, false, true);
        }
    }
}
```

### 6.2 并发消费

```java
@Configuration
public class ConcurrentConsumerConfig {
    @Bean
    public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(
            ConnectionFactory connectionFactory) {
        SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);
        factory.setConcurrentConsumers(3);
        factory.setMaxConcurrentConsumers(10);
        return factory;
    }
}
```

## 7. 监控与运维

### 7.1 消息追踪

```java
@Aspect
@Component
public class MessageTraceAspect {
    @Around("@annotation(org.springframework.amqp.rabbit.annotation.RabbitHandler)")
    public Object traceMessage(ProceedingJoinPoint joinPoint) throws Throwable {
        String messageId = MDC.get("messageId");
        log.info("开始处理消息: {}", messageId);
        try {
            Object result = joinPoint.proceed();
            log.info("消息处理成功: {}", messageId);
            return result;
        } catch (Exception e) {
            log.error("消息处理失败: {}", messageId, e);
            throw e;
        }
    }
}
```

### 7.2 监控指标

```java
@Configuration
public class MetricsConfig {
    @Bean
    public MeterRegistry meterRegistry() {
        return new SimpleMeterRegistry();
    }

    @Bean
    public RabbitMetricsCollector rabbitMetricsCollector(MeterRegistry meterRegistry) {
        return new RabbitMetricsCollector(meterRegistry);
    }
}
```

## 8. 最佳实践

1. **消息可靠性**
   - 实现消息确认机制
   - 使用死信队列处理失败消息
   - 实现消息幂等性处理

2. **性能优化**
   - 合理设置预取数量
   - 实现消息批量处理
   - 配置适当的并发消费者

3. **监控运维**
   - 实现消息追踪
   - 监控队列积压情况
   - 设置告警阈值

4. **异常处理**
   - 实现消息重试机制
   - 记录详细错误日志
   - 设置告警通知

## 9. 总结

本文介绍了Java消息队列开发的核心概念和实践：
- 消息队列选型考虑
- 消息发送与消费实现
- 可靠性保证机制
- 性能优化策略
- 监控与运维方案

通过这些实践，可以构建出高可用、可靠的消息处理系统。

## 10. 练习建议

1. 搭建RabbitMQ环境
2. 实现基本的消息发送和消费
3. 实现死信队列处理
4. 配置消息持久化
5. 实现消息追踪

消息队列的使用需要考虑多个方面，建议在实践中逐步完善各项机制。