# Day 113: Java高性能架构 - 高并发系统设计

## 1. 高并发系统概述

高并发系统是指能够同时处理大量请求的系统。在互联网应用中，高并发是一个常见的挑战，尤其是在电商、社交媒体、金融交易等领域。设计高并发系统需要综合考虑性能、可用性、可扩展性和一致性等多个方面。

## 2. 高并发系统设计原则

### 2.1 无状态设计

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    @Autowired
    private ProductService productService;
    
    @GetMapping("/{id}")
    public ResponseEntity<Product> getProduct(@PathVariable Long id) {
        // 无状态设计，不依赖服务器上的会话状态
        Product product = productService.getProduct(id);
        return ResponseEntity.ok(product);
    }
}
```

### 2.2 水平扩展

```java
@Configuration
public class LoadBalancerConfig {
    
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
    
    @Service
    public class ProductServiceClient {
        
        @Autowired
        private RestTemplate restTemplate;
        
        public Product getProduct(Long id) {
            // 通过服务名调用，支持水平扩展
            return restTemplate.getForObject("http://product-service/api/products/" + id, Product.class);
        }
    }
}
```

### 2.3 数据分片

```java
public class ShardingDataSource {
    
    private List<DataSource> shards;
    private ShardingStrategy strategy;
    
    public Connection getConnection(Object shardKey) {
        // 根据分片键选择数据源
        int shardIndex = strategy.determineShardIndex(shardKey);
        return shards.get(shardIndex).getConnection();
    }
}

public class UserRepository {
    
    @Autowired
    private ShardingDataSource dataSource;
    
    public User findById(Long userId) {
        // 使用用户ID作为分片键
        try (Connection conn = dataSource.getConnection(userId)) {
            // 执行查询
            return executeQuery(conn, userId);
        }
    }
}
```

## 3. 高并发架构模式

### 3.1 读写分离

```java
@Configuration
public class DataSourceConfig {
    
    @Bean
    @Primary
    public DataSource masterDataSource() {
        // 配置主数据源，用于写操作
        return createDataSource("master");
    }
    
    @Bean
    public DataSource slaveDataSource() {
        // 配置从数据源，用于读操作
        return createDataSource("slave");
    }
    
    @Bean
    public RoutingDataSource routingDataSource() {
        RoutingDataSource routingDataSource = new RoutingDataSource();
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put("master", masterDataSource());
        targetDataSources.put("slave", slaveDataSource());
        routingDataSource.setTargetDataSources(targetDataSources);
        routingDataSource.setDefaultTargetDataSource(masterDataSource());
        return routingDataSource;
    }
}

public class RoutingDataSource extends AbstractRoutingDataSource {
    
    @Override
    protected Object determineCurrentLookupKey() {
        return TransactionSynchronizationManager.isCurrentTransactionReadOnly() ? "slave" : "master";
    }
}
```

### 3.2 缓存架构

```java
@Service
public class ProductService {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private CacheManager cacheManager;
    
    public Product getProduct(Long id) {
        // 多级缓存查询
        Product product = getFromLocalCache(id);
        if (product != null) {
            return product;
        }
        
        product = getFromDistributedCache(id);
        if (product != null) {
            addToLocalCache(id, product);
            return product;
        }
        
        product = productRepository.findById(id).orElse(null);
        if (product != null) {
            addToDistributedCache(id, product);
            addToLocalCache(id, product);
        }
        
        return product;
    }
}
```

### 3.3 异步处理

```java
@Service
public class OrderService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        // 同步处理：保存订单基本信息
        Order order = new Order();
        order.setUserId(request.getUserId());
        order.setAmount(request.getAmount());
        order.setStatus(OrderStatus.PENDING);
        orderRepository.save(order);
        
        // 异步处理：发送消息进行后续处理
        OrderEvent event = new OrderEvent(order.getId(), request);
        kafkaTemplate.send("order-processing", event);
        
        return order;
    }
}

@Component
public class OrderProcessor {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private PaymentService paymentService;
    
    @KafkaListener(topics = "order-processing")
    public void processOrder(OrderEvent event) {
        try {
            // 异步处理订单逻辑
            boolean inventoryResult = inventoryService.reserve(event.getOrderId(), event.getItems());
            boolean paymentResult = paymentService.process(event.getOrderId(), event.getAmount());
            
            OrderStatus status = (inventoryResult && paymentResult) ? 
                OrderStatus.CONFIRMED : OrderStatus.FAILED;
            
            // 更新订单状态
            Order order = orderRepository.findById(event.getOrderId()).orElse(null);
            if (order != null) {
                order.setStatus(status);
                orderRepository.save(order);
            }
        } catch (Exception e) {
            // 异常处理
            handleOrderProcessingFailure(event.getOrderId(), e);
        }
    }
}
```

## 4. 并发控制

### 4.1 限流策略

```java
public class RateLimiter {
    private final long ratePerSecond;
    private long lastRequestTime = System.currentTimeMillis();
    private long allowance = 0;
    
    public RateLimiter(long ratePerSecond) {
        this.ratePerSecond = ratePerSecond;
    }
    
    public synchronized boolean allowRequest() {
        long current = System.currentTimeMillis();
        long timePassed = current - lastRequestTime;
        lastRequestTime = current;
        
        // 计算令牌桶中新增的令牌数
        allowance += timePassed * (ratePerSecond / 1000.0);
        if (allowance > ratePerSecond) {
            allowance = ratePerSecond;
        }
        
        if (allowance < 1.0) {
            return false;
        } else {
            allowance -= 1.0;
            return true;
        }
    }
}

@Component
public class RateLimitingFilter extends OncePerRequestFilter {
    
    private final Map<String, RateLimiter> limiters = new ConcurrentHashMap<>();
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, 
                                    FilterChain filterChain) throws ServletException, IOException {
        String clientId = getClientId(request);
        RateLimiter limiter = limiters.computeIfAbsent(clientId, k -> new RateLimiter(10)); // 每秒10个请求
        
        if (limiter.allowRequest()) {
            filterChain.doFilter(request, response);
        } else {
            response.setStatus(HttpServletResponse.SC_TOO_MANY_REQUESTS);
            response.getWriter().write("Rate limit exceeded");
        }
    }
    
    private String getClientId(HttpServletRequest request) {
        // 可以使用IP地址、用户ID等作为客户端标识
        return request.getRemoteAddr();
    }
}
```

### 4.2 熔断降级

```java
@Service
public class RecommendationService {
    
    @Autowired
    private RestTemplate restTemplate;
    
    @HystrixCommand(fallbackMethod = "getDefaultRecommendations",
                   commandProperties = {
                       @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "20"),
                       @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "50"),
                       @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "5000")
                   })
    public List<Product> getRecommendations(Long userId) {
        // 调用推荐服务
        return restTemplate.getForObject("http://recommendation-service/api/recommendations?userId=" + userId,
                                         List.class);
    }
    
    public List<Product> getDefaultRecommendations(Long userId) {
        // 降级逻辑：返回热门商品列表
        return getHotProducts();
    }
}
```

## 5. 性能优化

### 5.1 连接池优化

```java
@Configuration
public class DataSourceConfig {
    
    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
        config.setUsername("user");
        config.setPassword("password");
        
        // 连接池优化配置
        config.setMaximumPoolSize(100); // 最大连接数
        config.setMinimumIdle(10);      // 最小空闲连接
        config.setIdleTimeout(30000);    // 空闲连接超时
        config.setConnectionTimeout(10000); // 连接超时
        config.setMaxLifetime(1800000);  // 连接最大生命周期
        
        return new HikariDataSource(config);
    }
}
```

### 5.2 线程池优化

```java
@Configuration
public class ThreadPoolConfig {
    
    @Bean
    public ThreadPoolExecutor serviceExecutor() {
        int corePoolSize = Runtime.getRuntime().availableProcessors();
        int maxPoolSize = corePoolSize * 2;
        long keepAliveTime = 60L;
        
        return new ThreadPoolExecutor(
            corePoolSize,
            maxPoolSize,
            keepAliveTime,
            TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(1000),
            new ThreadFactoryBuilder().setNameFormat("service-pool-%d").build(),
            new ThreadPoolExecutor.CallerRunsPolicy() // 拒绝策略
        );
    }
    
    @Bean
    public ThreadPoolExecutor ioExecutor() {
        // IO密集型任务，线程数可以更多
        int corePoolSize = Runtime.getRuntime().availableProcessors() * 2;
        int maxPoolSize = corePoolSize * 4;
        
        return new ThreadPoolExecutor(
            corePoolSize,
            maxPoolSize,
            60L,
            TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(5000),
            new ThreadFactoryBuilder().setNameFormat("io-pool-%d").build(),
            new ThreadPoolExecutor.CallerRunsPolicy()
        );
    }
}
```

## 6. 监控与预警

```java
@Configuration
@EnablePrometheusMetrics
public class MetricsConfig {
    
    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCommonTags() {
        return registry -> registry.config().commonTags("application", "high-concurrency-app");
    }
}

@RestController
public class ProductController {
    
    private final Counter requestCounter;
    private final Timer responseTimer;
    
    public ProductController(MeterRegistry registry) {
        this.requestCounter = registry.counter("http.requests", "endpoint", "products");
        this.responseTimer = registry.timer("http.response.time", "endpoint", "products");
    }
    
    @GetMapping("/api/products/{id}")
    public ResponseEntity<Product> getProduct(@PathVariable Long id) {
        requestCounter.increment();
        
        return responseTimer.record(() -> {
            Product product = productService.getProduct(id);
            return ResponseEntity.ok(product);
        });
    }
}
```

## 7. 总结

高并发系统设计是一个复杂的工程，需要从多个维度进行考虑：

1. 遵循无状态设计、水平扩展、数据分片等基本原则
2. 采用读写分离、缓存架构、异步处理等架构模式
3. 实施限流、熔断、降级等并发控制策略
4. 优化连接池、线程池等系统资源
5. 建立完善的监控与预警机制

## 参考资源

1. Java并发编程实战：https://book.douban.com/subject/10484692/
2. Spring Cloud官方文档：https://spring.io/projects/spring-cloud
3. Redis官方文档：https://redis.io/documentation
4. 高性能MySQL：https://book.douban.com/subject/23008813/