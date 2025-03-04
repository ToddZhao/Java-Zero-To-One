# Day 34: Java性能优化与调优

## 引言

性能优化是Java应用程序开发中的一个重要环节。通过合理的优化，我们可以提高应用程序的响应速度、降低资源消耗、提升用户体验。本文将详细介绍Java性能优化的各个方面，包括代码层面、JVM调优、数据库优化等。

## 1. 代码层面优化

### 1.1 基本原则

- 避免创建不必要的对象
- 使用合适的数据结构
- 减少方法调用层级
- 避免使用反射
- 合理使用字符串操作

### 1.2 字符串优化

```java
public class StringOptimization {
    public static String buildString() {
        // 不推荐
        String result = "";
        for (int i = 0; i < 1000; i++) {
            result += i;
        }

        // 推荐
        StringBuilder builder = new StringBuilder();
        for (int i = 0; i < 1000; i++) {
            builder.append(i);
        }
        return builder.toString();
    }
}
```

### 1.3 集合优化

```java
public class CollectionOptimization {
    // 初始化时指定容量
    List<String> list = new ArrayList<>(1000);
    
    // 使用批量操作
    public void batchProcess(List<String> items) {
        // 不推荐
        for (String item : items) {
            process(item);
        }
        
        // 推荐
        items.parallelStream()
             .forEach(this::process);
    }
}
```

## 2. JVM调优

### 2.1 内存分配

```bash
# 设置堆内存
-Xms2g -Xmx4g

# 设置新生代大小
-Xmn512m

# 设置元空间大小
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=512m
```

### 2.2 垃圾回收

```bash
# 使用G1收集器
-XX:+UseG1GC

# 设置停顿时间目标
-XX:MaxGCPauseMillis=200

# 设置并行线程数
-XX:ParallelGCThreads=4
```

## 3. 数据库优化

### 3.1 SQL优化

```java
public class DatabaseOptimization {
    // 使用批量操作
    public void batchInsert(List<User> users) {
        String sql = "INSERT INTO users (name, age) VALUES (?, ?)";
        try (PreparedStatement pstmt = conn.prepareStatement(sql)) {
            for (User user : users) {
                pstmt.setString(1, user.getName());
                pstmt.setInt(2, user.getAge());
                pstmt.addBatch();
            }
            pstmt.executeBatch();
        }
    }
    
    // 使用索引
    public User findUser(String name) {
        String sql = "SELECT * FROM users WHERE name = ?";
        // name字段应该建立索引
    }
}
```

### 3.2 连接池优化

```java
@Configuration
public class DataSourceConfig {
    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setMaximumPoolSize(10);
        config.setMinimumIdle(5);
        config.setConnectionTimeout(30000);
        return new HikariDataSource(config);
    }
}
```

## 4. 缓存优化

### 4.1 本地缓存

```java
public class CacheOptimization {
    private LoadingCache<String, User> cache;
    
    public CacheOptimization() {
        cache = CacheBuilder.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(10, TimeUnit.MINUTES)
            .build(new CacheLoader<String, User>() {
                @Override
                public User load(String key) {
                    return loadUserFromDB(key);
                }
            });
    }
    
    public User getUser(String id) {
        try {
            return cache.get(id);
        } catch (ExecutionException e) {
            throw new RuntimeException(e);
        }
    }
}
```

### 4.2 分布式缓存

```java
@Service
public class RedisCache {
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public void set(String key, Object value, long timeout) {
        redisTemplate.opsForValue().set(key, value, timeout, TimeUnit.SECONDS);
    }
    
    public Object get(String key) {
        return redisTemplate.opsForValue().get(key);
    }
}
```

## 5. 多线程优化

### 5.1 线程池配置

```java
public class ThreadPoolOptimization {
    private ThreadPoolExecutor executor;
    
    public ThreadPoolOptimization() {
        executor = new ThreadPoolExecutor(
            5,                      // 核心线程数
            10,                     // 最大线程数
            60L,                    // 空闲线程存活时间
            TimeUnit.SECONDS,       // 时间单位
            new LinkedBlockingQueue<>(100),  // 工作队列
            new ThreadPoolExecutor.CallerRunsPolicy()  // 拒绝策略
        );
    }
    
    public void processTask(Runnable task) {
        executor.execute(task);
    }
}
```

### 5.2 并发工具优化

```java
public class ConcurrentOptimization {
    // 使用ConcurrentHashMap代替HashMap
    private ConcurrentHashMap<String, Object> cache = new ConcurrentHashMap<>();
    
    // 使用原子类
    private AtomicInteger counter = new AtomicInteger(0);
    
    public void increment() {
        counter.incrementAndGet();
    }
}
```

## 6. 监控与分析

### 6.1 性能指标

- 响应时间
- 吞吐量
- CPU使用率
- 内存使用情况
- GC频率和时间

### 6.2 监控工具

1. JMX监控
```java
@ManagedResource
public class PerformanceMonitor {
    @ManagedAttribute
    public long getRequestCount() {
        return requestCount.get();
    }
    
    @ManagedOperation
    public void resetCounters() {
        requestCount.set(0);
    }
}
```

2. 日志监控
```java
public class PerformanceLogger {
    private static final Logger logger = LoggerFactory.getLogger(PerformanceLogger.class);
    
    public void logPerformance(String operation, long startTime) {
        long duration = System.currentTimeMillis() - startTime;
        logger.info("Operation: {}, Duration: {} ms", operation, duration);
    }
}
```

## 总结

本文介绍了Java性能优化的多个方面，包括：

1. 代码层面优化
2. JVM调优
3. 数据库优化
4. 缓存优化
5. 多线程优化
6. 监控与分析

通过合理应用这些优化技术，我们可以显著提升应用程序的性能。需要注意的是，性能优化应该建立在实际需求和监控数据的基础上，避免过度优化。

## 参考资源

1. Java性能优化指南：https://docs.oracle.com/javase/performance/
2. JVM调优文档：https://docs.oracle.com/javase/8/docs/technotes/guides/vm/
3. 数据库性能优化最佳实践
4. Java并发编程实践