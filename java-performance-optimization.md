# Day 97: Java性能优化 - 从理论到实践

## 1. 引言

性能优化是Java企业级应用开发中的关键环节。本文将深入探讨Java性能优化的理论基础、关键技术和实践案例，帮助开发者构建高性能的Java应用。

## 2. 性能优化基础

### 2.1 性能指标

在进行性能优化前，需要明确以下关键指标：

- 响应时间：系统响应用户请求所需的时间
- 吞吐量：系统在单位时间内处理的请求数量
- 并发用户数：系统同时能够处理的用户数量
- 资源利用率：CPU、内存、磁盘、网络等资源的使用情况
- 延迟：请求从发出到接收响应的时间

### 2.2 性能优化方法论

性能优化应遵循以下方法论：

1. 建立性能基准
2. 识别性能瓶颈
3. 优化代码或系统架构
4. 验证优化效果
5. 重复上述过程

## 3. JVM性能优化

### 3.1 内存管理优化

```java
// JVM参数示例
// 设置堆内存大小
-Xms4g -Xmx4g

// 设置年轻代大小
-XX:NewSize=1g -XX:MaxNewSize=1g

// 设置元空间大小
-XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=256m

// 设置垃圾回收器
-XX:+UseG1GC
```

### 3.2 垃圾回收优化

```java
// G1垃圾回收器参数示例
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:InitiatingHeapOccupancyPercent=45
-XX:G1HeapRegionSize=8m
-XX:G1ReservePercent=10
-XX:G1HeapWastePercent=5
```

### 3.3 JIT编译器优化

```java
// JIT编译器参数示例
-XX:+TieredCompilation
-XX:CompileThreshold=10000
-XX:+UseCodeCacheFlushing
```

## 4. 代码级优化

### 4.1 数据结构与算法优化

```java
public class CollectionPerformance {
    // 不同集合类型的性能对比
    public void compareCollections() {
        // 对于频繁查找操作，使用HashMap
        Map<String, Object> hashMap = new HashMap<>();
        // 对于需要保持插入顺序的场景，使用LinkedHashMap
        Map<String, Object> linkedHashMap = new LinkedHashMap<>();
        // 对于需要排序的场景，使用TreeMap
        Map<String, Object> treeMap = new TreeMap<>();
        
        // 对于频繁随机访问的场景，使用ArrayList
        List<String> arrayList = new ArrayList<>();
        // 对于频繁插入删除的场景，使用LinkedList
        List<String> linkedList = new LinkedList<>();
        
        // 对于需要去重的场景，使用HashSet
        Set<String> hashSet = new HashSet<>();
        // 对于需要排序的场景，使用TreeSet
        Set<String> treeSet = new TreeSet<>();
    }
}
```

### 4.2 多线程优化

```java
public class ThreadPoolOptimization {
    public ThreadPoolExecutor createOptimizedThreadPool() {
        int corePoolSize = Runtime.getRuntime().availableProcessors();
        int maximumPoolSize = corePoolSize * 2;
        long keepAliveTime = 60L;
        TimeUnit unit = TimeUnit.SECONDS;
        BlockingQueue<Runnable> workQueue = new ArrayBlockingQueue<>(1000);
        ThreadFactory threadFactory = new ThreadFactoryBuilder()
                .setNameFormat("optimized-pool-%d")
                .setDaemon(true)
                .setPriority(Thread.NORM_PRIORITY)
                .build();
        RejectedExecutionHandler handler = new ThreadPoolExecutor.CallerRunsPolicy();
        
        return new ThreadPoolExecutor(
                corePoolSize,
                maximumPoolSize,
                keepAliveTime,
                unit,
                workQueue,
                threadFactory,
                handler);
    }
}
```

### 4.3 I/O优化

```java
public class IOOptimization {
    // 使用缓冲区提高I/O性能
    public void bufferedIO() throws IOException {
        // 不使用缓冲区的写入
        try (FileWriter writer = new FileWriter("output.txt")) {
            for (int i = 0; i < 100000; i++) {
                writer.write("data\n");
            }
        }
        
        // 使用缓冲区的写入，性能更好
        try (BufferedWriter writer = new BufferedWriter(new FileWriter("output.txt"))) {
            for (int i = 0; i < 100000; i++) {
                writer.write("data\n");
            }
        }
    }
    
    // 使用NIO提高I/O性能
    public void nioExample() throws IOException {
        Path path = Paths.get("output.txt");
        try (BufferedWriter writer = Files.newBufferedWriter(path, StandardCharsets.UTF_8)) {
            for (int i = 0; i < 100000; i++) {
                writer.write("data\n");
            }
        }
    }
}
```

## 5. 数据库性能优化

### 5.1 SQL优化

```java
@Repository
public class OptimizedUserRepository {
    @PersistenceContext
    private EntityManager entityManager;
    
    // 优化前：加载所有字段
    public List<User> findAllUsersUnoptimized() {
        return entityManager.createQuery("SELECT u FROM User u", User.class)
                .getResultList();
    }
    
    // 优化后：只加载需要的字段
    public List<UserDTO> findAllUsersOptimized() {
        return entityManager.createQuery(
                "SELECT new com.example.UserDTO(u.id, u.name, u.email) FROM User u",
                UserDTO.class)
                .getResultList();
    }
    
    // 优化前：不分页查询
    public List<User> findUsersByRoleUnoptimized(String role) {
        return entityManager.createQuery(
                "SELECT u FROM User u WHERE u.role = :role", User.class)
                .setParameter("role", role)
                .getResultList();
    }
    
    // 优化后：分页查询
    public List<User> findUsersByRoleOptimized(String role, int page, int size) {
        return entityManager.createQuery(
                "SELECT u FROM User u WHERE u.role = :role", User.class)
                .setParameter("role", role)
                .setFirstResult((page - 1) * size)
                .setMaxResults(size)
                .getResultList();
    }
}
```

### 5.2 索引优化

```sql
-- 创建适当的索引
-- 单列索引
CREATE INDEX idx_user_email ON users(email);

-- 复合索引（多列索引）
CREATE INDEX idx_user_name_email ON users(name, email);

-- 覆盖索引（包含所有查询所需字段的索引）
CREATE INDEX idx_user_role_name ON users(role, name);
```

### 5.3 连接池优化

```java
@Configuration
public class DataSourceConfig {
    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
        config.setUsername("user");
        config.setPassword("password");
        config.setDriverClassName("com.mysql.cj.jdbc.Driver");
        
        // 连接池优化配置
        config.setMaximumPoolSize(20); // 最大连接数
        config.setMinimumIdle(5);      // 最小空闲连接数
        config.setIdleTimeout(30000);  // 连接最大空闲时间
        config.setConnectionTimeout(30000); // 连接超时时间
        config.setMaxLifetime(1800000);     // 连接最大生命周期
        
        // 性能优化配置
        config.addDataSourceProperty("cachePrepStmts", "true");
        config.addDataSourceProperty("prepStmtCacheSize", "250");
        config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
        config.addDataSourceProperty("useServerPrepStmts", "true");
        
        return new HikariDataSource(config);
    }
}
```

## 6. Web应用性能优化

### 6.1 Spring Boot优化

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    // 配置静态资源缓存
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/static/**")
                .addResourceLocations("classpath:/static/")
                .setCachePeriod(3600) // 缓存1小时
                .resourceChain(true)
                .addResolver(new VersionResourceResolver().addContentVersionStrategy("/**"));
    }
    
    // 配置压缩响应
    @Bean
    public GzipRequestFilter gzipRequestFilter() {
        return new GzipRequestFilter();
    }
    
    // 配置异步处理
    @Override
    public void configureAsyncSupport(AsyncSupportConfigurer configurer) {
        configurer.setDefaultTimeout(30000); // 30秒超时
        configurer.setTaskExecutor(asyncTaskExecutor());
    }
    
    @Bean
    public AsyncTaskExecutor asyncTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        return executor;
    }
}
```

### 6.2 HTTP缓存优化

```java
@RestController
public class CacheOptimizedController {
    @GetMapping("/api/products/{id}")
    public ResponseEntity<Product> getProduct(@PathVariable Long id) {
        Product product = productService.findById(id);
        
        // 设置缓存控制头
        CacheControl cacheControl = CacheControl.maxAge(30, TimeUnit.MINUTES)
                .noTransform()
                .mustRevalidate();
        
        return ResponseEntity.ok()
                .cacheControl(cacheControl)
                .eTag(String.valueOf(product.getVersion()))
                .body(product);
    }
    
    @GetMapping("/api/categories")
    public ResponseEntity<List<Category>> getCategories() {
        List<Category> categories = categoryService.findAll();
        
        // 设置缓存控制头
        CacheControl cacheControl = CacheControl.maxAge(1, TimeUnit.HOURS)
                .noTransform();
        
        return ResponseEntity.ok()
                .cacheControl(cacheControl)
                .body(categories);
    }
}
```

## 7. 性能监控与分析

### 7.1 JVM监控

```java
@Configuration
@EnablePrometheusMetrics
public class MonitoringConfig {
    @Bean
    public MeterRegistryCustomizer<PrometheusMeterRegistry> configureMetrics() {
        return registry -> registry.config()
                .commonTags("application", "my-app")
                .meterFilter(new MeterFilter() {
                    @Override
                    public MeterFilterReply accept(Meter.Id id) {
                        if (id.getName().startsWith("jvm")) {
                            return MeterFilterReply.ACCEPT;
                        }
                        return MeterFilterReply.NEUTRAL;
                    }
                });
    }
}
```

### 7.2 应用性能监控

```java
@Aspect
@Component
public class PerformanceMonitoringAspect {
    private static final Logger logger = LoggerFactory.getLogger(PerformanceMonitoringAspect.class);
    
    @Around("@annotation(org.springframework.web.bind.annotation.RequestMapping) || "
            + "@annotation(org.springframework.web.bind.annotation.GetMapping) || "
            + "@annotation(org.springframework.web.bind.annotation.PostMapping)")
    public Object monitorControllerPerformance(ProceedingJoinPoint joinPoint) throws Throwable {
        long startTime = System.currentTimeMillis();
        Object result = joinPoint.proceed();
        long executionTime = System.currentTimeMillis() - startTime;
        
        String methodName = joinPoint.get