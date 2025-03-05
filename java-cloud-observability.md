# Day 127: Java云原生高级 - 云原生可观测性

## 1. 云原生可观测性概述

云原生可观测性是指在云原生环境中，通过收集和分析应用程序的各种运行时数据，来了解和诊断系统的运行状态。它主要包括三个核心支柱：监控（Metrics）、日志（Logging）和追踪（Tracing）。

## 2. 监控（Metrics）

### 2.1 Prometheus集成

```java
@Configuration
public class PrometheusConfig {
    
    @Bean
    MeterRegistryCustomizer<MeterRegistry> metricsCommonTags() {
        return registry -> registry.config().commonTags(
            "application", "order-service",
            "environment", System.getenv().getOrDefault("ENVIRONMENT", "prod")
        );
    }
    
    @Bean
    public PrometheusMeterRegistry prometheusMeterRegistry() {
        return new PrometheusMeterRegistry(PrometheusConfig.DEFAULT);
    }
}

@Service
public class OrderService {
    private final Counter orderCounter;
    private final Timer orderProcessingTimer;
    private final DistributionSummary orderAmountSummary;
    
    public OrderService(MeterRegistry registry) {
        this.orderCounter = registry.counter("orders.created.total");
        this.orderProcessingTimer = registry.timer("orders.processing.time");
        this.orderAmountSummary = registry.summary("orders.amount");
    }
    
    @Timed(value = "orders.create", description = "Time taken to create an order")
    public Order createOrder(OrderRequest request) {
        return orderProcessingTimer.record(() -> {
            Order order = processOrder(request);
            orderCounter.increment();
            orderAmountSummary.record(order.getAmount().doubleValue());
            return order;
        });
    }
}
```

### 2.2 自定义指标

```java
@Component
public class CustomMetricsCollector {
    private final MeterRegistry registry;
    private final Gauge cacheSize;
    private final Map<String, Counter> errorCounters = new ConcurrentHashMap<>();
    
    public CustomMetricsCollector(MeterRegistry registry, CacheManager cacheManager) {
        this.registry = registry;
        this.cacheSize = Gauge.builder("cache.size", cacheManager, this::getCacheSize)
            .description("Number of entries in the cache")
            .register(registry);
    }
    
    private long getCacheSize(CacheManager cacheManager) {
        return cacheManager.getCacheNames().stream()
            .mapToLong(name -> cacheManager.getCache(name).getNativeCache().size())
            .sum();
    }
    
    public void recordError(String errorType) {
        errorCounters.computeIfAbsent(errorType, type ->
            Counter.builder("application.errors")
                .tag("type", type)
                .description("Number of errors by type")
                .register(registry)
        ).increment();
    }
}
```

## 3. 日志（Logging）

### 3.1 结构化日志

```java
@Slf4j
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @PostMapping
    public ResponseEntity<Order> createOrder(@RequestBody OrderRequest request) {
        MDC.put("customerId", request.getCustomerId());
        MDC.put("orderType", request.getOrderType());
        
        try {
            log.info("Received order creation request", 
                kv("amount", request.getAmount()),
                kv("items", request.getItems().size()));
            
            Order order = orderService.createOrder(request);
            
            log.info("Order created successfully",
                kv("orderId", order.getId()),
                kv("status", order.getStatus()));
            
            return ResponseEntity.ok(order);
        } catch (Exception e) {
            log.error("Failed to create order",
                kv("error", e.getMessage()),
                kv("errorType", e.getClass().getSimpleName()));
            throw e;
        } finally {
            MDC.clear();
        }
    }
    
    private KeyValue kv(String key, Object value) {
        return KeyValue.of(key, String.valueOf(value));
    }
}
```

### 3.2 ELK集成

```java
@Configuration
public class LoggingConfig {
    
    @Bean
    public LogstashTcpSocketAppender logstashAppender() {
        LogstashTcpSocketAppender appender = new LogstashTcpSocketAppender();
        appender.addDestination("logstash:5000");
        
        LogstashEncoder encoder = new LogstashEncoder();
        encoder.setCustomFields("{\"app_name\":\"order-service\",\"environment\":\"prod\"}");
        appender.setEncoder(encoder);
        
        appender.start();
        return appender;
    }
}

// logback-spring.xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/base.xml"/>
    
    <appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
        <destination>logstash:5000</destination>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"app_name":"order-service","environment":"prod"}</customFields>
        </encoder>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="LOGSTASH"/>
    </root>
</configuration>
```

## 4. 追踪（Tracing）

### 4.1 OpenTelemetry集成

```java
@Configuration
public class TracingConfig {
    
    @Bean
    public OpenTelemetry openTelemetry() {
        Resource resource = Resource.getDefault()
            .merge(Resource.create(Attributes.of(
                ResourceAttributes.SERVICE_NAME, "order-service",
                ResourceAttributes.DEPLOYMENT_ENVIRONMENT, "prod"
            )));
        
        SdkTracerProvider sdkTracerProvider = SdkTracerProvider.builder()
            .addSpanProcessor(BatchSpanProcessor.builder(
                OtlpGrpcSpanExporter.builder()
                    .setEndpoint("http://collector:4317")
                    .build())
                .build())
            .setResource(resource)
            .build();
        
        return OpenTelemetrySdk.builder()
            .setTracerProvider(sdkTracerProvider)
            .setPropagators(ContextPropagators.create(
                TextMapPropagator.composite(
                    W3CTraceContextPropagator.getInstance(),
                    B3Propagator.injectingSingleHeader())))
            .build();
    }
    
    @Bean
    public TracerProvider tracerProvider(OpenTelemetry openTelemetry) {
        return openTelemetry.getTracerProvider();
    }
}

@Service
public class OrderService {
    private final Tracer tracer;
    
    public OrderService(TracerProvider tracerProvider) {
        this.tracer = tracerProvider.get("order-service");
    }
    
    public Order processOrder(OrderRequest request) {
        Span span = tracer.spanBuilder("process-order")
            .setAttribute("customerId", request.getCustomerId())
            .setAttribute("orderType", request.getOrderType())
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            // 处理订单逻辑
            validateOrder(request);
            calculateTotal(request);
            saveOrder(request);
            
            return order;
        } catch (Exception e) {
            span.setStatus(StatusCode.ERROR);
            span.recordException(e);
            throw e;
        } finally {
            span.end();
        }
    }
}
```

### 4.2 分布式追踪

```java
@Configuration
public class WebClientConfig {
    
    @Bean
    public WebClient.Builder webClientBuilder(TracerProvider tracerProvider) {
        return WebClient.builder()
            .filter(new TracingExchangeFilter(tracerProvider));
    }
}

public class TracingExchangeFilter implements ExchangeFilterFunction {
    private final Tracer tracer;
    
    public TracingExchangeFilter(TracerProvider tracerProvider) {
        this.tracer = tracerProvider.get("web-client");
    }
    
    @Override
    public Mono<ClientResponse> filter(ClientRequest request, ExchangeFunction next) {
        Span span = tracer.spanBuilder("http-request")
            .setAttribute("http.method", request.method().name())
            .setAttribute("http.url", request.url().toString())
            .startSpan();
        
        ClientRequest.Builder builder = ClientRequest.from(request);
        span.getSpanContext().forEach((key, value) ->
            builder.header(key, value));
        
        return next.exchange(builder.build())
            .doFinally(signalType -> span.end());
    }
}
```

## 5. 可观测性最佳实践

### 5.1 健康检查

```java
@Component
public class CustomHealthIndicator implements HealthIndicator {
    private final DataSource dataSource;
    private final RedisConnectionFactory redisConnectionFactory;
    
    @Override
    public Health health() {
        Health.Builder builder = new Health.Builder();
        
        try {
            checkDatabase(builder);
            checkRedis(builder);
            checkExternalServices(builder);
            
            return builder.up().build();
        } catch (Exception e) {
            return builder.down(e).build();
        }
    }
    
    private void checkDatabase(Health.Builder builder) {
        try (Connection conn = dataSource.getConnection()) {
            builder.withDetail("database", "UP");
        } catch (Exception e) {
            builder.withDetail("database", "DOWN");
            throw e;
        }
    }
    
    private void checkRedis(Health.Builder builder) {
        try {
            RedisConnection conn = redisConnectionFactory.getConnection();
            conn.close();
            builder.withDetail("redis", "UP");
        } catch (Exception e) {
            builder.withDetail("redis", "DOWN");
            throw e;
        }
    }
}
```

### 5.2 告警配置

```yaml
# Prometheus告警规则
groups:
- name: application_alerts
  rules:
  - alert: HighErrorRate
    expr: sum(rate(application_errors_total[5m])) by (service) > 0.1
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: High error rate detected
      description: Service {{ $labels.service }} is experiencing high error rate

  - alert: SlowResponses
    expr: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)) > 1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Slow response times detected
      description: Service {{ $labels.service }} is experiencing slow response times

  - alert: HighCPUUsage
    expr: process_cpu_usage > 0.8
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: High CPU usage detected
      description: Instance {{ $labels.instance }} has high CPU usage
```

## 参考资源

1. OpenTelemetry文档：https://opentelemetry.io/docs/
2. Prometheus最佳实践：https://prometheus.io/docs/practices/
3. ELK Stack文档：https://www.elastic.co/guide/
4. Spring Boot Actuator文档：https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html