# Day 114: Java高性能架构 - 异步编程模型

## 1. 异步编程概述

异步编程是一种并行处理的方式，它允许程序在执行长时间运行的任务时不会阻塞主线程，从而提高系统的响应性和性能。在Java中，我们有多种方式来实现异步编程。

## 2. Future和CompletableFuture

### 2.1 Future模式

```java
public class FutureExample {
    private ExecutorService executor = Executors.newFixedThreadPool(5);
    
    public Future<String> asyncOperation() {
        return executor.submit(() -> {
            // 模拟耗时操作
            Thread.sleep(1000);
            return "Operation completed";
        });
    }
    
    public void handleResult() {
        Future<String> future = asyncOperation();
        try {
            // 等待结果
            String result = future.get(2, TimeUnit.SECONDS);
            System.out.println(result);
        } catch (TimeoutException e) {
            future.cancel(true);
        }
    }
}
```

### 2.2 CompletableFuture

```java
public class CompletableFutureExample {
    
    public CompletableFuture<Order> processOrder(OrderRequest request) {
        return CompletableFuture.supplyAsync(() -> createOrder(request))
            .thenApplyAsync(order -> {
                // 处理支付
                processPayment(order);
                return order;
            })
            .thenApplyAsync(order -> {
                // 更新库存
                updateInventory(order);
                return order;
            })
            .exceptionally(ex -> {
                // 异常处理
                handleError(ex);
                return null;
            });
    }
    
    public void handleMultipleOrders(List<OrderRequest> requests) {
        List<CompletableFuture<Order>> futures = requests.stream()
            .map(this::processOrder)
            .collect(Collectors.toList());
            
        // 等待所有订单处理完成
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
            .thenRun(() -> System.out.println("All orders processed"));
    }
}
```

## 3. 响应式异步模型

### 3.1 Spring WebFlux

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    @Autowired
    private ProductService productService;
    
    @GetMapping
    public Flux<Product> getAllProducts() {
        return productService.findAll()
            .delayElements(Duration.ofMillis(100)) // 模拟延迟
            .log();
    }
    
    @GetMapping("/{id}")
    public Mono<Product> getProduct(@PathVariable String id) {
        return productService.findById(id)
            .switchIfEmpty(Mono.error(new NotFoundException()));
    }
    
    @PostMapping
    public Mono<Product> createProduct(@RequestBody Product product) {
        return productService.save(product);
    }
}

@Service
public class ProductService {
    
    @Autowired
    private ReactiveMongoRepository<Product, String> repository;
    
    public Flux<Product> findAll() {
        return repository.findAll();
    }
    
    public Mono<Product> findById(String id) {
        return repository.findById(id);
    }
    
    public Mono<Product> save(Product product) {
        return repository.save(product);
    }
}
```

## 4. 异步消息处理

### 4.1 Spring AMQP异步消息

```java
@Configuration
public class RabbitConfig {
    
    @Bean
    public Queue orderQueue() {
        return new Queue("order-queue", true);
    }
    
    @Bean
    public TopicExchange orderExchange() {
        return new TopicExchange("order-exchange");
    }
    
    @Bean
    public Binding binding(Queue orderQueue, TopicExchange orderExchange) {
        return BindingBuilder.bind(orderQueue).to(orderExchange).with("order.*");
    }
}

@Component
public class OrderProducer {
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public void sendOrder(Order order) {
        rabbitTemplate.convertAndSend("order-exchange", "order.created", order);
    }
}

@Component
public class OrderConsumer {
    
    @RabbitListener(queues = "order-queue")
    public void handleOrder(Order order) {
        // 异步处理订单
        processOrder(order);
    }
    
    private void processOrder(Order order) {
        // 订单处理逻辑
    }
}
```

## 5. 异步Web客户端

### 5.1 WebClient

```java
@Service
public class ProductClient {
    
    private final WebClient webClient;
    
    public ProductClient(WebClient.Builder webClientBuilder) {
        this.webClient = webClientBuilder
            .baseUrl("http://product-service")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .build();
    }
    
    public Mono<Product> getProduct(String id) {
        return webClient.get()
            .uri("/api/products/{id}", id)
            .retrieve()
            .bodyToMono(Product.class)
            .timeout(Duration.ofSeconds(5))
            .onErrorResume(ex -> Mono.empty());
    }
    
    public Flux<Product> getAllProducts() {
        return webClient.get()
            .uri("/api/products")
            .retrieve()
            .bodyToFlux(Product.class)
            .retryWhen(Retry.backoff(3, Duration.ofSeconds(1)));
    }
}
```

## 6. 最佳实践

### 6.1 异步任务执行器

```java
public class AsyncTaskExecutor {
    private final ExecutorService executor;
    private final Map<String, CompletableFuture<?>> taskMap;
    
    public AsyncTaskExecutor(int threadPoolSize) {
        this.executor = Executors.newFixedThreadPool(threadPoolSize);
        this.taskMap = new ConcurrentHashMap<>();
    }
    
    public <T> CompletableFuture<T> submitTask(String taskId, Supplier<T> task) {
        CompletableFuture<T> future = CompletableFuture.supplyAsync(task, executor)
            .whenComplete((result, ex) -> taskMap.remove(taskId));
            
        taskMap.put(taskId, future);
        return future;
    }
    
    public boolean cancelTask(String taskId) {
        CompletableFuture<?> future = taskMap.get(taskId);
        if (future != null) {
            return future.cancel(true);
        }
        return false;
    }
    
    public void shutdown() {
        executor.shutdown();
    }
}
```

### 6.2 异步结果缓存

```java
public class AsyncResultCache<K, V> {
    private final Cache<K, CompletableFuture<V>> cache;
    
    public AsyncResultCache(long maximumSize, Duration expiration) {
        this.cache = Caffeine.newBuilder()
            .maximumSize(maximumSize)
            .expireAfterWrite(expiration)
            .build();
    }
    
    public CompletableFuture<V> get(K key, Function<K, CompletableFuture<V>> loader) {
        return cache.get(key, k -> loader.apply(k)
            .whenComplete((v, ex) -> {
                if (ex != null) {
                    cache.invalidate(key);
                }
            }));
    }
}
```

## 7. 总结

异步编程是构建高性能系统的关键技术，在实践中需要注意：

1. 选择合适的异步编程模型
2. 正确处理异步操作的异常和超时
3. 合理使用线程池和资源管理
4. 实现可靠的异步结果处理机制
5. 注意异步操作的性能监控

## 参考资源

1. Java并发编程实战：https://book.douban.com/subject/10484692/
2. Spring WebFlux文档：https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html
3. CompletableFuture API：https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html
4. Spring AMQP文档：https://docs.spring.io/spring-amqp/docs/current/reference/html/