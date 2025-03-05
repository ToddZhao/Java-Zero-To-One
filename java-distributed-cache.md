# Day 111: Java分布式架构 - 分布式缓存策略

## 1. 分布式缓存概述

分布式缓存是一种将数据缓存在分布式系统中多个节点上的技术，通过合理的缓存策略和数据分布方式，提高系统的性能和可扩展性。

## 2. 缓存架构设计

### 2.1 多级缓存架构

```java
public class MultiLevelCache {
    private Cache localCache;    // 本地缓存
    private Cache redisCache;    // 分布式缓存
    private DataSource dataSource; // 数据源

    public Object get(String key) {
        // 查询本地缓存
        Object value = localCache.get(key);
        if (value != null) {
            return value;
        }

        // 查询分布式缓存
        value = redisCache.get(key);
        if (value != null) {
            // 回填本地缓存
            localCache.put(key, value);
            return value;
        }

        // 查询数据源
        value = dataSource.get(key);
        if (value != null) {
            // 更新缓存
            redisCache.put(key, value);
            localCache.put(key, value);
        }
        return value;
    }
}
```

### 2.2 缓存一致性保证

```java
public class CacheConsistencyManager {
    private Cache cache;
    private DataSource dataSource;
    private MessageQueue messageQueue;

    @Transactional
    public void update(String key, Object value) {
        // 更新数据源
        dataSource.update(key, value);
        
        // 删除缓存
        cache.delete(key);
        
        // 发送消息通知其他节点
        CacheInvalidateMessage message = new CacheInvalidateMessage(key);
        messageQueue.send(message);
    }

    // 消息监听器
    @MessageListener
    public void handleCacheInvalidate(CacheInvalidateMessage message) {
        cache.delete(message.getKey());
    }
}
```

## 3. 缓存策略实现

### 3.1 缓存穿透处理

```java
public class CachePenetrationHandler {
    private Cache cache;
    private DataSource dataSource;
    private BloomFilter<String> bloomFilter;

    public Object get(String key) {
        // 布隆过滤器检查
        if (!bloomFilter.mightContain(key)) {
            return null;
        }

        // 查询缓存
        Object value = cache.get(key);
        if (value != null) {
            return value;
        }

        // 查询数据源
        value = dataSource.get(key);
        if (value != null) {
            cache.put(key, value);
        } else {
            // 缓存空值，防止缓存穿透
            cache.put(key, NullValue.INSTANCE, 5, TimeUnit.MINUTES);
        }
        return value;
    }
}
```

### 3.2 缓存击穿处理

```java
public class HotKeyHandler {
    private Cache cache;
    private DataSource dataSource;
    private Lock lock;

    public Object get(String key) {
        Object value = cache.get(key);
        if (value != null) {
            return value;
        }

        // 使用分布式锁防止并发重建缓存
        if (lock.tryLock(key)) {
            try {
                // 双重检查
                value = cache.get(key);
                if (value == null) {
                    value = dataSource.get(key);
                    cache.put(key, value);
                }
            } finally {
                lock.unlock(key);
            }
        } else {
            // 等待一段时间后重试
            Thread.sleep(50);
            return get(key);
        }
        return value;
    }
}
```

### 3.3 缓存雪崩处理

```java
public class CacheAvailabilityManager {
    private Cache cache;
    private DataSource dataSource;

    public void initCache() {
        // 添加随机过期时间，避免同时过期
        Random random = new Random();
        List<String> keys = dataSource.getAllKeys();
        
        for (String key : keys) {
            Object value = dataSource.get(key);
            // 过期时间加上随机值
            int expireTime = 3600 + random.nextInt(300);
            cache.put(key, value, expireTime, TimeUnit.SECONDS);
        }
    }

    public Object get(String key) {
        try {
            return cache.get(key);
        } catch (Exception e) {
            // 降级处理
            return getFromLocalCache(key);
        }
    }
}
```

## 4. 缓存框架应用

### 4.1 Spring Cache整合

```java
@Configuration
@EnableCaching
public class CacheConfig extends CachingConfigurerSupport {
    
    @Bean
    public CacheManager cacheManager() {
        RedisCacheManager.RedisCacheManagerBuilder builder = 
            RedisCacheManager.RedisCacheManagerBuilder.fromConnectionFactory(redisConnectionFactory());
        
        // 设置默认过期时间
        builder.cacheDefaults(RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10)));
            
        // 设置不同缓存空间的配置
        Map<String, RedisCacheConfiguration> configMap = new HashMap<>();
        configMap.put("userCache", RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(5)));
        builder.withInitialCacheConfigurations(configMap);
        
        return builder.build();
    }
}

@Service
public class UserService {
    
    @Cacheable(value = "userCache", key = "#id")
    public User getUser(Long id) {
        return userRepository.findById(id);
    }
    
    @CachePut(value = "userCache", key = "#user.id")
    public User updateUser(User user) {
        return userRepository.save(user);
    }
    
    @CacheEvict(value = "userCache", key = "#id")
    public void deleteUser(Long id) {
        userRepository.deleteById(id);
    }
}
```

## 5. 性能优化

### 5.1 缓存预热

```java
public class CacheWarmUpService {
    private Cache cache;
    private DataSource dataSource;

    @PostConstruct
    public void warmUp() {
        // 获取热点数据
        List<String> hotKeys = dataSource.getHotKeys();
        
        // 并行加载缓存
        CompletableFuture[] futures = hotKeys.stream()
            .map(key -> CompletableFuture.runAsync(() -> {
                Object value = dataSource.get(key);
                cache.put(key, value);
            }))
            .toArray(CompletableFuture[]::new);
        
        // 等待所有缓存加载完成
        CompletableFuture.allOf(futures).join();
    }
}
```

### 5.2 缓存更新策略

```java
public class CacheUpdateStrategy {
    private Cache cache;
    private DataSource dataSource;
    private ScheduledExecutorService scheduler;

    public void startUpdateTask() {
        scheduler.scheduleWithFixedDelay(() -> {
            try {
                // 获取需要更新的键
                List<String> keys = cache.getExpiredKeys();
                
                for (String key : keys) {
                    // 异步更新缓存
                    CompletableFuture.runAsync(() -> {
                        Object newValue = dataSource.get(key);
                        cache.put(key, newValue);
                    });
                }
            } catch (Exception e) {
                // 异常处理
                logger.error("Cache update failed", e);
            }
        }, 0, 5, TimeUnit.MINUTES);
    }
}
```

## 6. 总结

分布式缓存是提升系统性能的关键技术，在实践中需要注意：

1. 合理设计多级缓存架构
2. 注意解决缓存穿透、击穿、雪崩等问题
3. 保证缓存一致性
4. 实现合适的缓存更新策略
5. 进行必要的性能优化

## 参考资源

1. Redis官方文档：https://redis.io/documentation
2. Spring Cache文档：https://docs.spring.io/spring-framework/docs/current/reference/html/integration.html#cache
3. 缓存设计模式：https://martinfowler.com/bliki/TwoHardThings.html