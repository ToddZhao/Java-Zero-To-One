# Day 96: Java高可用架构设计 - 原理与实践

## 1. 引言

高可用性是现代企业级应用的核心需求之一。本文将深入探讨Java高可用架构设计的原理、关键技术和最佳实践，帮助开发者构建稳定可靠的系统。

## 2. 高可用架构基本概念

### 2.1 什么是高可用

高可用（High Availability，HA）是指系统无中断地执行其功能的能力，通常用系统可用性百分比来衡量：

```
可用性 = 正常运行时间 / (正常运行时间 + 故障时间)
```

常见的可用性等级：
- 两个9（99%）：每年停机时间约87.6小时
- 三个9（99.9%）：每年停机时间约8.76小时
- 四个9（99.99%）：每年停机时间约52.56分钟
- 五个9（99.999%）：每年停机时间约5.26分钟

### 2.2 高可用架构的特点

- 冗余设计：关键组件有备份
- 故障检测：能够快速发现故障
- 故障恢复：能够自动从故障中恢复
- 无单点故障：任何单个组件故障不会导致整个系统不可用
- 可伸缩性：能够根据负载动态调整资源

## 3. 高可用架构设计原则

### 3.1 故障隔离原则

```java
// 使用舱壁模式隔离故障
public class BulkheadExample {
    private final Bulkhead bulkhead;
    
    public BulkheadExample() {
        BulkheadConfig config = BulkheadConfig.custom()
            .maxConcurrentCalls(10)
            .maxWaitDuration(Duration.ofMillis(500))
            .build();
        bulkhead = Bulkhead.of("serviceA", config);
    }
    
    public String executeService() {
        return bulkhead.executeSupplier(() -> {
            // 调用可能失败的服务
            return callExternalService();
        });
    }
    
    private String callExternalService() {
        // 实际服务调用
        return "service result";
    }
}
```

### 3.2 超时控制原则

```java
@Service
public class TimeoutService {
    @Autowired
    private RestTemplate restTemplate;
    
    public ResponseEntity<String> callWithTimeout() {
        SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
        factory.setConnectTimeout(1000); // 连接超时1秒
        factory.setReadTimeout(2000);    // 读取超时2秒
        
        RestTemplate template = new RestTemplate(factory);
        return template.getForEntity("http://example.com/api", String.class);
    }
}
```

### 3.3 重试机制原则

```java
@Service
public class RetryService {
    @Retryable(value = {ServiceException.class}, 
              maxAttempts = 3, 
              backoff = @Backoff(delay = 1000, multiplier = 2))
    public String serviceWithRetry() {
        // 可能失败的服务调用
        return callUnstableService();
    }
    
    @Recover
    public String recover(ServiceException e) {
        // 重试失败后的恢复逻辑
        return "fallback response";
    }
}
```

## 4. 高可用架构关键技术

### 4.1 负载均衡

```java
@Configuration
public class LoadBalancerConfig {
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
    
    @Bean
    public IRule ribbonRule() {
        // 使用可用性筛选策略
        return new AvailabilityFilteringRule();
    }
}
```

### 4.2 服务降级

```java
@Service
public class ProductService {
    @HystrixCommand(fallbackMethod = "getProductFallback",
                   commandProperties = {
                       @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "1000")
                   })
    public Product getProduct(Long productId) {
        // 调用可能超时的服务
        return productRepository.findById(productId)
                .orElseThrow(() -> new ProductNotFoundException(productId));
    }
    
    public Product getProductFallback(Long productId) {
        // 返回默认商品信息
        return new Product(productId, "默认商品", "商品信息暂时不可用", 0.0);
    }
}
```

### 4.3 服务熔断

```java
@Service
public class CircuitBreakerService {
    private final CircuitBreaker circuitBreaker;
    
    public CircuitBreakerService() {
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)  // 50%的失败率将触发熔断
            .waitDurationInOpenState(Duration.ofMillis(10000)) // 熔断后等待10秒
            .ringBufferSizeInHalfOpenState(5) // 半开状态时尝试5次请求
            .ringBufferSizeInClosedState(10) // 关闭状态时记录10次请求
            .build();
            
        circuitBreaker = CircuitBreaker.of("serviceB", config);
    }
    
    public String execute() {
        return circuitBreaker.executeSupplier(() -> {
            // 可能失败的服务调用
            return callService();
        });
    }
}
```

## 5. 高可用数据库设计

### 5.1 主从复制

```java
@Configuration
public class DataSourceConfig {
    @Bean
    @Primary
    @ConfigurationProperties("spring.datasource.master")
    public DataSource masterDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    @ConfigurationProperties("spring.datasource.slave")
    public DataSource slaveDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    public DataSource routingDataSource() {
        ReadWriteRoutingDataSource routingDataSource = new ReadWriteRoutingDataSource();
        
        Map<Object, Object> dataSourceMap = new HashMap<>();
        dataSourceMap.put("master", masterDataSource());
        dataSourceMap.put("slave", slaveDataSource());
        
        routingDataSource.setTargetDataSources(dataSourceMap);
        routingDataSource.setDefaultTargetDataSource(masterDataSource());
        
        return routingDataSource;
    }
}
```

### 5.2 分库分表

```java
@Configuration
@EnableShardingSphereDataSource
public class ShardingConfig {
    @Bean
    public DataSource shardingDataSource() {
        // 配置真实数据源
        Map<String, DataSource> dataSourceMap = createDataSourceMap();
        
        // 配置分片规则
        ShardingRuleConfiguration shardingRuleConfig = new ShardingRuleConfiguration();
        
        // 配置表分片策略
        TableRuleConfiguration orderTableRuleConfig = new TableRuleConfiguration("t_order", "ds${0..1}.t_order${0..1}");
        orderTableRuleConfig.setTableShardingStrategyConfig(
            new StandardShardingStrategyConfiguration("order_id", "orderTableShardingAlgorithm"));
        orderTableRuleConfig.setDatabaseShardingStrategyConfig(
            new StandardShardingStrategyConfiguration("user_id", "databaseShardingAlgorithm"));
        shardingRuleConfig.getTableRuleConfigs().add(orderTableRuleConfig);
        
        // 创建数据源
        return ShardingDataSourceFactory.createDataSource(dataSourceMap, shardingRuleConfig, new Properties());
    }
    
    private Map<String, DataSource> createDataSourceMap() {
        // 创建数据源映射
        Map<String, DataSource> result = new HashMap<>();
        result.put("ds0", createDataSource("jdbc:mysql://localhost:3306/ds0"));
        result.put("ds1", createDataSource("jdbc:mysql://localhost:3306/ds1"));
        return result;
    }
    
    private DataSource createDataSource(String url) {
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setDriverClassName("com.mysql.cj.jdbc.Driver");
        dataSource.setJdbcUrl(url);
        dataSource.setUsername("root");
        dataSource.setPassword("password");
        return dataSource;
    }
}
```

## 6. 高可用缓存设计

### 6.1 Redis集群

```java
@Configuration
public class RedisClusterConfig {
    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        RedisClusterConfiguration clusterConfig = new RedisClusterConfiguration();
        clusterConfig.setClusterNodes(Arrays.asList(
            new RedisNode("redis1", 6379),
            new RedisNode("redis2", 6379),
            new RedisNode("redis3", 6379)
        ));
        clusterConfig.setMaxRedirects(3);
        
        return new JedisConnectionFactory(clusterConfig);
    }
    
    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory());
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
}
```

### 6.2 缓存穿透防护

```java
@Service
public class CacheProtectionService {
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    @Autowired
    private ProductRepository productRepository;
    
    // 使用布隆过滤器防止缓存穿透
    private BloomFilter<Long> bloomFilter = BloomFilter.create(
        Funnels.longFunnel(),
        10000000,  // 预计元素数量
        0.01       // 误判率
    );
    
    @PostConstruct
    public void initBloomFilter() {
        // 初始化布隆过滤器，加载所有商品ID
        List<Long> allProductIds = productRepository.findAllIds();
        for (Long id : allProductIds) {
            bloomFilter.put(id);
        }
    }
    
    public Product getProduct(Long productId) {
        // 先判断ID是否可能存在
        if (!bloomFilter.mightContain(productId)) {
            return null; // ID一定不存在
        }
        
        // 查询缓存
        String cacheKey = "product:" + productId;
        Product product = (Product) redisTemplate.opsForValue().get(cacheKey);
        
        if (product != null) {
            return product;
        }
        
        // 查询数据库
        product = productRepository.findById(productId).orElse(null);
        
        // 即使为null也缓存，但设置较短的过期时间，防止缓存穿透
        redisTemplate.opsForValue().set(
            cacheKey,
            product != null ? product : new NullValueObject(),
            product != null ? 30 : 5,
            TimeUnit.MINUTES
        );
        
        return product;
    }
}
```

## 7. 高可用消息队列设计

### 7.1 Kafka集群

```java
@Configuration
public class KafkaConfig {
    @Bean
    public ProducerFactory<String, String> producerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka1:9092,kafka2:9092,kafka3:9092");
        configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        // 提高可靠性配置
        configProps.put(ProducerConfig.ACKS_CONFIG, "all");
        configProps.put(ProducerConfig.RETRIES_CONFIG, 3);
        configProps.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        
        return new DefaultKafkaProducerFactory<>(configProps);
    }
    
    @Bean
    public ConsumerFactory<String, String> consumerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka1:9092,kafka2:9092,kafka3:9092");
        configProps.put(ConsumerConfig.GROUP_ID_CONFIG, "my-