# Day 89: Java微服务架构实践指南

## 1. 引言

微服务架构已成为现代分布式系统的主流架构模式。本文将介绍Java微服务开发的核心概念和最佳实践，帮助开发者构建可扩展、高可用的微服务系统。

## 2. 微服务设计原则

### 2.1 单一职责

每个微服务应该专注于单一业务功能：

```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentClient paymentClient;

    public OrderService(OrderRepository orderRepository, PaymentClient paymentClient) {
        this.orderRepository = orderRepository;
        this.paymentClient = paymentClient;
    }

    public Order createOrder(OrderRequest request) {
        // 处理订单创建逻辑
        Order order = new Order(request);
        return orderRepository.save(order);
    }
}
```

### 2.2 API设计

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {
    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @PostMapping
    public ResponseEntity<Order> createOrder(@RequestBody @Valid OrderRequest request) {
        Order order = orderService.createOrder(request);
        return ResponseEntity.created(URI.create("/api/v1/orders/" + order.getId()))
                .body(order);
    }
}
```

## 3. 服务注册与发现

### 3.1 Eureka服务注册

```yaml
# application.yml
spring:
  application:
    name: order-service

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
  instance:
    preferIpAddress: true
```

### 3.2 服务发现

```java
@FeignClient(name = "payment-service")
public interface PaymentClient {
    @PostMapping("/api/v1/payments")
    PaymentResponse processPayment(@RequestBody PaymentRequest request);
}
```

## 4. 负载均衡

### 4.1 Ribbon配置

```yaml
payment-service:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RoundRobinRule
    ServerListRefreshInterval: 1000
```

### 4.2 自定义负载均衡策略

```java
public class CustomLoadBalancerRule extends AbstractLoadBalancerRule {
    @Override
    public Server choose(Object key) {
        List<Server> servers = getLoadBalancer().getReachableServers();
        if (servers.isEmpty()) {
            return null;
        }
        // 实现自定义负载均衡逻辑
        return servers.get(0);
    }
}
```

## 5. 断路器模式

### 5.1 Hystrix配置

```java
@HystrixCommand(fallbackMethod = "processPaymentFallback",
    commandProperties = {
        @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "20"),
        @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "50"),
        @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "5000")
    })
public PaymentResponse processPayment(PaymentRequest request) {
    return paymentClient.processPayment(request);
}

public PaymentResponse processPaymentFallback(PaymentRequest request) {
    return new PaymentResponse(PaymentStatus.FAILED, "Service unavailable");
}
```

## 6. 配置管理

### 6.1 Spring Cloud Config

```yaml
# bootstrap.yml
spring:
  cloud:
    config:
      uri: http://config-server:8888
      fail-fast: true
```

### 6.2 动态配置刷新

```java
@RefreshScope
@RestController
public class ConfigurationController {
    @Value("${app.feature.enabled}")
    private boolean featureEnabled;

    @GetMapping("/feature")
    public boolean isFeatureEnabled() {
        return featureEnabled;
    }
}
```

## 7. 服务网关

### 7.1 Spring Cloud Gateway配置

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: order_service
        uri: lb://order-service
        predicates:
        - Path=/api/v1/orders/**
        filters:
        - name: CircuitBreaker
          args:
            name: orderCircuitBreaker
            fallbackUri: forward:/fallback
```

### 7.2 自定义过滤器

```java
@Component
public class AuthenticationFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = exchange.getRequest().getHeaders().getFirst("Authorization");
        if (token == null || !token.startsWith("Bearer ")) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        return chain.filter(exchange);
    }
}
```

## 8. 分布式追踪

### 8.1 Sleuth配置

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>
```

### 8.2 追踪实现

```java
@Slf4j
@Service
public class OrderService {
    @Autowired
    private Tracer tracer;

    public Order processOrder(OrderRequest request) {
        Span span = tracer.nextSpan().name("process-order");
        try (Tracer.SpanInScope ws = tracer.withSpanInScope(span.start())) {
            log.info("Processing order: {}", request.getOrderId());
            // 处理订单逻辑
            return orderRepository.save(new Order(request));
        } finally {
            span.finish();
        }
    }
}
```

## 9. 最佳实践

1. **服务拆分**
   - 基于业务边界进行服务拆分
   - 保持服务的独立性和自治性
   - 避免服务之间的强耦合

2. **数据管理**
   - 每个服务使用独立的数据库
   - 实现数据一致性策略
   - 使用事件驱动架构处理跨服务数据

3. **安全性**
   - 实现统一的认证授权
   - 服务间通信加密
   - 实施细粒度的访问控制

4. **可观测性**
   - 实现全面的监控指标
   - 集中式日志管理
   - 分布式追踪

## 10. 总结

本文介绍了Java微服务开发的核心概念和实践：
- 微服务设计原则
- 服务注册与发现
- 负载均衡
- 断路器模式
- 配置管理
- 服务网关
- 分布式追踪

通过这些实践，可以构建出高可用、可扩展的微服务系统。

## 11. 练习建议

1. 搭建基础微服务框架
2. 实现服务注册与发现
3. 配置负载均衡和断路器
4. 实现分布式追踪
5. 部署微服务到容器环境

微服务架构的实践需要不断积累经验，建议从小规模项目开始，逐步扩展和优化。