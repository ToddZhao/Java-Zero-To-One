# Day 44: Java缓存技术 - Redis实战

## 引言

在高并发系统中，缓存是提升性能的关键技术。本文将介绍Redis缓存在Java应用中的实践，包括基本概念、Spring Boot集成以及常见的缓存策略。

## 1. Redis基础

### 1.1 核心特性

- 高性能：基于内存操作，读写性能极高
- 数据结构丰富：支持字符串、哈希、列表、集合、有序集合等
- 持久化：支持RDB和AOF两种持久化方式
- 主从复制：支持主从架构，提高可用性
- 分布式：支持集群模式，实现水平扩展

### 1.2 常用数据类型

- String：最基本的数据类型，可以存储字符串、整数或浮点数
- Hash：适合存储对象，类似Java中的HashMap
- List：有序列表，支持双向操作
- Set：无序集合，元素不重复
- Sorted Set：有序集合，每个元素关联一个分数

## 2. Spring Boot集成Redis

### 2.1 依赖配置

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

### 2.2 基本配置

```yaml
spring:
  redis:
    host: localhost
    port: 6379
    password: 
    database: 0
    timeout: 3000
    lettuce:
      pool:
        max-active: 8
        max-wait: -1ms
        max-idle: 8
        min-idle: 0
```

### 2.3 Redis配置类

```java
@Configuration
@EnableCaching
public class RedisConfig {
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        
        // 使用Jackson2JsonRedisSerializer序列化值
        Jackson2JsonRedisSerializer<Object> serializer = new Jackson2JsonRedisSerializer<>(Object.class);
        ObjectMapper mapper = new ObjectMapper();
        mapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        mapper.activateDefaultTyping(LaissezFaireSubTypeValidator.instance, 
                                    ObjectMapper.DefaultTyping.NON_FINAL);
        serializer.setObjectMapper(mapper);
        
        // 使用StringRedisSerializer序列化键
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(serializer);
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(serializer);
        template.afterPropertiesSet();
        
        return template;
    }
    
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))
            .serializeKeysWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()))
            .disableCachingNullValues();
            
        return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .build();
    }
}
```

## 3. Redis操作实践

### 3.1 基本操作

```java
@Service
public class ProductService {
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    private static final String PRODUCT_KEY_PREFIX = "product:";
    
    public void saveProduct(Product product) {
        String key = PRODUCT_KEY_PREFIX + product.getId();
        redisTemplate.opsForValue().set(key, product, 1, TimeUnit.HOURS);
    }
    
    public Product getProduct(Long id) {
        String key = PRODUCT_KEY_PREFIX + id;
        return (Product) redisTemplate.opsForValue().get(key);
    }
    
    public void deleteProduct(Long id) {
        String key = PRODUCT_KEY_PREFIX + id;
        redisTemplate.delete(key);
    }
}
```

### 3.2 使用注解缓存

```java
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;
    
    @Cacheable(value = "users", key = "#id")
    public User getUserById(Long id) {
        return userRepository.findById(id).orElse(null);
    }
    
    @CachePut(value = "users", key = "#user.id")
    public User updateUser(User user) {
        return userRepository.save(user);
    }
    
    @CacheEvict(value = "users", key = "#id")
    public void deleteUser(Long id) {
        userRepository.deleteById(id);
    }
    
    @CacheEvict(value = "users", allEntries = true)
    public void clearUserCache() {
        // 清空users缓存
    }
}
```

## 4. 高级特性

### 4.1 分布式锁实现

```java
@Service
public class DistributedLockService {
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    private static final String LOCK_PREFIX = "lock:";
    private static final long DEFAULT_EXPIRE = 30000; // 30秒
    
    public boolean acquireLock(String lockKey, String requestId, long expireTime) {
        String key = LOCK_PREFIX + lockKey;
        Boolean result = redisTemplate.opsForValue()
            .setIfAbsent(key, requestId, expireTime, TimeUnit.MILLISECONDS);
        return Boolean.TRUE.equals(result);
    }
    
    public boolean releaseLock(String lockKey, String requestId) {
        String key = LOCK_PREFIX + lockKey;
        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                        "return redis.call('del', KEYS[1]) else return 0 end";
        Long result = redisTemplate.execute(
            new DefaultRedisScript<>(script, Long.class),
            Collections.singletonList(key),
            requestId
        );
        return Long.valueOf(1).equals(result);
    }
    
    public void executeWithLock(String lockKey, Runnable task) {
        String requestId = UUID.randomUUID().toString();
        try {
            if (acquireLock(lockKey, requestId, DEFAULT_EXPIRE)) {
                task.run();
            } else {
                throw new RuntimeException("获取锁失败");
            }
        } finally {
            releaseLock(lockKey, requestId);
        }
    }
}
```

### 4.2 缓存穿透防护

```java
@Service
public class ProductServiceWithProtection {
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    @Autowired
    private ProductRepository productRepository;
    
    private static final String PRODUCT_KEY_PREFIX = "product:";
    private static final Object NULL_VALUE = new Object();
    
    public Product getProductWithProtection(Long id) {
        String key = PRODUCT_KEY_PREFIX + id;
        
        // 查询缓存
        Object cachedValue = redisTemplate.opsForValue().get(key);
        
        // 缓存命中
        if (cachedValue != null) {
            // 如果是空值标记，返回null
            if (cachedValue.equals(NULL_VALUE)) {
                return null;
            }
            return (Product) cachedValue;
        }
        
        // 缓存未命中，查询数据库
        Product product = productRepository.findById(id).orElse(null);
        
        // 数据库中不存在该商品，缓存空值防止缓存穿透
        if (product == null) {
            redisTemplate.opsForValue().set(key, NULL_VALUE, 5, TimeUnit.MINUTES);
            return null;
        }
        
        // 数据库中存在，放入缓存
        redisTemplate.opsForValue().set(key, product, 1, TimeUnit.HOURS);
        return product;
    }
}
```

## 5. 实践案例

### 5.1 热点数据缓存

```java
@Service
public class HotDataService {
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    @Autowired
    private HotDataRepository hotDataRepository;
    
    private static final String HOT_DATA_KEY = "hot:data";
    
    @Scheduled(fixedRate = 300000) // 5分钟更新一次
    public void refreshHotData() {
        List<HotData> hotDataList = hotDataRepository.findTop100ByOrderByViewCountDesc();
        redisTemplate.delete(HOT_DATA_KEY);
        redisTemplate.opsForList().rightPushAll(HOT_DATA_KEY, hotDataList.toArray());
        redisTemplate.expire(HOT_DATA_KEY, 10, TimeUnit.MINUTES);
    }
    
    public List<HotData> getHotData() {
        Long size = redisTemplate.opsForList().size(HOT_DATA_KEY);
        if (size == null || size == 0) {
            refreshHotData();
        }
        return redisTemplate.opsForList().range(HOT_DATA_KEY, 0, -1)
            .stream()
            .map(item -> (HotData) item)
            .collect(Collectors.toList());
    }
}
```

### 5.2 限流实现

```java
@Service
public class RateLimiterService {
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    public boolean isAllowed(String key, int limit, int period) {
        String luaScript = "local key = KEYS[1] " +
                          "local limit = tonumber(ARGV[1]) " +
                          "local period = tonumber(ARGV[2]) " +
                          "local current = tonumber(redis.call('get', key) or '0') " +
                          "if current >= limit then return 0 else " +
                          "redis.call('incrby', key, 1) " +
                          "redis.call('expire', key, period) " +
                          "return 1 end";
        
        Long result = redisTemplate.execute(
            new DefaultRedisScript<>(luaScript, Long.class),
            Collections.singletonList(key),
            String.valueOf(limit),
            String.valueOf(period)
        );
        
        return Long.valueOf(1).equals(result);
    }
    
    public boolean checkApiLimit(String apiKey, String apiPath) {
        String key = "rate:limit:" + apiKey + ":" + apiPath;
        // 限制每分钟最多60次请求
        return isAllowed(key, 60, 60);
    }
}
```

## 6. 最佳实践

1. 缓存策略
   - 合理设置过期时间
   - 使用缓存预热
   - 实现缓存更新策略

2. 缓存问题处理
   - 缓存穿透：缓存空值或使用布隆过滤器
   - 缓存击穿：使用互斥锁或热点数据永不过期
   - 缓存雪崩：过期时间添加随机值或使用多级缓存

3. 性能优化
   - 使用批量操作
   - 避免大key
   - 合理使用数据结构

## 总结

通过本文的学习，我们掌握了：

1. Redis的核心特性和数据类型
2. Spring Boot集成Redis的方法
3. 基本的Redis操作和注解缓存
4. 分布式锁和缓存穿透防护等高级特性
5. 热点数据缓存和限流等实践案例

在高并发系统中，合理使用Redis缓存可以显著提高系统性能，减轻数据库压力，是构建高性能系统的重要手段。

## 参考资源

1. Spring Data Redis文档：https://docs.spring.io/spring-data/redis/docs/current/reference/html/
2. Redis官方文档：https://redis.io/documentation
3. Redis设计与实现
4. 高性能缓存设计实践指南