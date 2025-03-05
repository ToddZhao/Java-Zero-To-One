# Day 118: Java微服务高级 - 分布式追踪

## 1. 分布式追踪概述

分布式追踪是一种用于监控和诊断微服务架构中请求流程的技术。它通过收集和分析请求在不同服务间传播的数据，帮助开发人员理解系统行为、定位性能瓶颈和排查故障。

## 2. 分布式追踪的核心概念

### 2.1 Trace和Span

```java
public class TracingConcepts {
    
    public void explainTracingConcepts() {
        // Trace：表示一个完整的请求流程，由多个Span组成
        // Span：表示一个操作单元，如服务调用、数据库查询等
        // SpanContext：包含Trace ID、Span ID等信息，用于在服务间传递
        // Baggage Items：随Trace传播的键值对，用于传递业务上下文
    }
}
```

### 2.2 采样策略

```java
@Configuration
public class TracingSamplerConfig {
    
    @Bean
    public Sampler defaultSampler() {
        // 常量采样器：始终采样或从不采样
        // return Sampler.ALWAYS_SAMPLE; // 100%采样
        // return Sampler.NEVER_SAMPLE;  // 0%采样
        
        // 概率采样器：按比例采样
        return ProbabilityBasedSampler.create(0.1f); // 10%采样率
        
        // 速率限制采样器：限制每秒采样数量
        // return RateLimitingSampler.create(10); // 每秒最多10个追踪
    }
}
```

## 3. 分布式追踪框架

### 3.1 Spring Cloud Sleuth

```java
@SpringBootApplication
@EnableDiscoveryClient
public class OrderServiceApplication {
    
    private static final Logger log = LoggerFactory.getLogger(OrderServiceApplication.class);
    
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
    
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
    
    @Service
    public class OrderService {
        
        @Autowired
        private RestTemplate restTemplate;
        
        public Order createOrder(OrderRequest request) {
            log.info("Creating order for user: {}", request.getUserId());
            
            // 调用库存服务，Sleuth自动传播追踪上下文
            InventoryResponse inventory = restTemplate.getForObject(
                "http://inventory-service/check/{productId}", 
                InventoryResponse.class, 
                request.getProductId());
            
            // 调用支付服务，Sleuth自动传播追踪上下文
            PaymentResponse payment = restTemplate.postForObject(
                "http://payment-service/process", 
                new PaymentRequest(request.getUserId(), request.getAmount()),
                PaymentResponse.class);
            
            log.info("Order created successfully");
            return new Order(UUID.randomUUID().toString(), request.getUserId(), request.getProductId());
        }
    }
}
```

### 3.2 OpenTelemetry

```java
@Configuration
public class OpenTelemetryConfig {
    
    @Bean
    public OpenTelemetry openTelemetry() {
        // 配置导出器
        JaegerGrpcSpanExporter jaegerExporter = JaegerGrpcSpanExporter.builder()
            .setEndpoint("http://jaeger:14250")
            .build();
        
        // 配置批处理
        BatchSpanProcessor spanProcessor = BatchSpanProcessor.builder(jaegerExporter)
            .setScheduleDelay(100, TimeUnit.MILLISECONDS)
            .setMaxQueueSize(10000)
            .setMaxExportBatchSize(1000)
            .build();
        
        // 创建SDK
        SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
            .addSpanProcessor(spanProcessor)
            .setSampler(Sampler.alwaysOn())
            .build();
        
        // 创建OpenTelemetry实例
        return OpenTelemetrySdk.builder()
            .setTracerProvider(tracerProvider)
            .setPropagators(ContextPropagators.create(
                TextMapPropagator.composite(
                    W3CTraceContextPropagator.getInstance(),
                    B3Propagator.injectingMultiHeaders())))
            .build();
    }
    
    @Bean
    public Tracer tracer(OpenTelemetry openTelemetry) {
        return openTelemetry.getTracer("app-tracer");
    }
}

@Service
public class ProductService {
    
    private final Tracer tracer;
    private final WebClient webClient;
    
    public ProductService(Tracer tracer, WebClient.Builder webClientBuilder) {
        this.tracer = tracer;
        this.webClient = webClientBuilder.build();
    }
    
    public Product getProductDetails(String productId) {
        // 创建Span
        Span span = tracer.spanBuilder("getProductDetails").startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            // 添加属性
            span.setAttribute("productId", productId);
            
            // 创建子Span
            Span dbSpan = tracer.spanBuilder("queryDatabase").startSpan();
            try (Scope dbScope = dbSpan.makeCurrent()) {
                dbSpan.setAttribute("db.operation", "query");
                Product product = queryDatabase(productId);
                return product;
            } finally {
                dbSpan.end();
            }
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }
    
    private Product queryDatabase(String productId) {
        // 数据库查询逻辑
        return new Product(productId, "Sample Product", 100.0);
    }
}
```

## 4. 追踪数据收集与存储

### 4.1 Zipkin集成

```java
@Configuration
public class ZipkinConfig {
    
    @Bean
    public Reporter<Span> zipkinReporter() {
        return AsyncReporter.create(
            URLConnectionSender.create("http://zipkin:9411/api/v2/spans"));
    }
    
    @Bean
    public Tracing tracing(@Value("${spring.application.name}") String serviceName, Reporter<Span> reporter) {
        return Tracing.newBuilder()
            .localServiceName(serviceName)
            .sampler(Sampler.ALWAYS_SAMPLE)
            .spanReporter(reporter)
            .build();
    }
    
    @Bean
    public HttpTracing httpTracing(Tracing tracing) {
        return HttpTracing.create(tracing);
    }
}
```

### 4.2 Jaeger集成

```java
@Configuration
public class JaegerConfig {
    
    @Bean
    public io.jaegertracing.Configuration jaegerConfiguration(
            @Value("${spring.application.name}") String serviceName) {
        return new io.jaegertracing.Configuration(serviceName)
            .withSampler(new io.jaegertracing.Configuration.SamplerConfiguration()
                .withType("const")
                .withParam(1))
            .withReporter(new io.jaegertracing.Configuration.ReporterConfiguration()
                .withLogSpans(true)
                .withSender(new io.jaegertracing.Configuration.SenderConfiguration()
                    .withEndpoint("http://jaeger-collector:14268/api/traces")));
    }
    
    @Bean
    public Tracer jaegerTracer(io.jaegertracing.Configuration configuration) {
        return configuration.getTracer();
    }
}
```

## 5. 跨服务追踪实现

### 5.1 HTTP请求追踪

```java
@Configuration
public class WebClientTracingConfig {
    
    @Bean
    public WebClient webClient(Tracer tracer) {
        return WebClient.builder()
            .filter((request, next) -> {
                Span span = tracer.spanBuilder("HTTP GET: " + request.url().toString())
                    .setSpanKind(SpanKind.CLIENT)
                    .startSpan();
                
                try (Scope scope = span.makeCurrent()) {
                    // 将当前追踪上下文注入HTTP头
                    ClientRequest tracedRequest = ClientRequest.from(request)
                        .headers(headers -> {
                            TextMapSetter<HttpHeaders> setter = HttpHeaders::add;
                            GlobalOpenTelemetry.getPropagators().getTextMapPropagator()
                                .inject(Context.current(), headers, setter);
                        })
                        .build();
                    
                    return next.exchange(tracedRequest)
                        .doFinally(signalType -> span.end());
                }
            })
            .build();
    }
}
```

### 5.2 消息队列追踪

```java
@Configuration
public class KafkaTracingConfig {
    
    @Bean
    public ProducerFactory<String, String> producerFactory(Tracer tracer) {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        
        DefaultKafkaProducerFactory<String, String> factory = new DefaultKafkaProducerFactory<>(props);
        factory.addListener(new MicrometerProducerListener<>(meterRegistry));
        
        return factory;
    }
    
    @Bean
    public KafkaTemplate<String, String> kafkaTemplate(ProducerFactory<String, String> producerFactory, Tracer tracer) {
        KafkaTemplate<String, String> template = new KafkaTemplate<>(producerFactory);
        
        template.setProducerInterceptor(new ProducerInterceptor<String, String>() {
            @Override
            public ProducerRecord<String, String> onSend(ProducerRecord<String, String> record) {
                Span span = tracer.spanBuilder("send-to-" + record.topic())
                    .setSpanKind(SpanKind.PRODUCER)
                    .startSpan();
                
                try (Scope scope = span.makeCurrent()) {
                    // 将追踪上下文注入消息头
                    ProducerRecord<String, String> tracedRecord = new ProducerRecord<>(
                        record.topic(), record.partition(), record.key(), record.value());
                    
                    TextMapSetter<ProducerRecord<String, String>> setter = 
                        (carrier, key, value) -> carrier.headers().add(key, value.getBytes());
                    
                    GlobalOpenTelemetry.getPropagators().getTextMapPropagator()
                        .inject(Context.current(), tracedRecord, setter);
                    
                    return tracedRecord;
                } finally {
                    span.end();
                }
            }
            
            @Override
            public void onAcknowledgement(RecordMetadata metadata, Exception exception) {
                // 处理确认
            }
            
            @Override
            public void close() {
                // 关闭资源
            }
            
            @Override
            public void configure(Map<String, ?> configs) {
                // 配置拦截器
            }
        });
        
        return template;
    }
}
```

## 6. 最佳实践

### 6.1 自定义Span丰富追踪信息

```java
@Component
public class OrderProcessor {
    
    private final Tracer tracer;
    
    public OrderProcessor(Tracer tracer) {
        this.tracer = tracer;
    }
    
    public void processOrder(Order order) {
        // 创建自定义Span
        Span span = tracer.spanBuilder("process-order")
            .setAttribute("orderId", order.getId())
            .setAttribute("userId", order.getUserId())
            .setAttribute("amount", order.getAmount())
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            // 记录事件
            span.addEvent("Order validation started");
            validateOrder(order);
            span.addEvent("Order validation completed");
            
            // 记录处理阶段
            span.addEvent("Payment processing started");
            processPayment(order);
            span.addEvent("Payment processing completed");
            
            // 记录业务指标
            span.setAttribute("order.items.count", order.getItems().size());
            span.setAttribute("order.processing.time", System.currentTimeMillis() - order.getCreatedAt());
            
        } catch (Exception e) {
            // 记录异常
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }
    
    private void validateOrder(Order order) {
        // 订单验证逻辑