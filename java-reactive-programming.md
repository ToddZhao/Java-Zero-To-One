# Day 115: Java高性能架构 - 响应式编程

## 1. 响应式编程概述

响应式编程是一种基于数据流和变化传播的编程范式。在响应式编程中，我们可以轻松地表达静态或动态数据流，而相关的计算模型会在数据流发生变化时自动传播这些变化。

## 2. 响应式编程的核心概念

### 2.1 响应式流规范

```java
public interface Publisher<T> {
    void subscribe(Subscriber<? super T> subscriber);
}

public interface Subscriber<T> {
    void onSubscribe(Subscription s);
    void onNext(T t);
    void onError(Throwable t);
    void onComplete();
}

public interface Subscription {
    void request(long n);
    void cancel();
}

public interface Processor<T, R> extends Subscriber<T>, Publisher<R> {
}
```

### 2.2 背压（Backpressure）机制

```java
public class BackpressureExample {
    public static void main(String[] args) {
        Flowable.range(1, 1000000)
            .onBackpressureBuffer(1000, () -> {}, BackpressureOverflowStrategy.DROP_LATEST)
            .observeOn(Schedulers.computation())
            .subscribe(new Subscriber<Integer>() {
                private Subscription subscription;
                
                @Override
                public void onSubscribe(Subscription s) {
                    this.subscription = s;
                    // 只请求处理5个元素
                    subscription.request(5);
                }
                
                @Override
                public void onNext(Integer t) {
                    System.out.println("处理元素: " + t);
                    // 处理完一个元素后，再请求一个新元素
                    subscription.request(1);
                }
                
                @Override
                public void onError(Throwable t) {
                    t.printStackTrace();
                }
                
                @Override
                public void onComplete() {
                    System.out.println("处理完成");
                }
            });
    }
}
```

## 3. 响应式编程框架

### 3.1 Project Reactor

```java
public class ReactorExample {
    
    public Flux<Product> getProducts() {
        return Flux.fromIterable(productRepository.findAll())
            .filter(product -> product.getStock() > 0)
            .map(this::enrichProduct)
            .doOnNext(product -> log.info("处理产品: {}", product.getId()))
            .subscribeOn(Schedulers.boundedElastic());
    }
    
    public Mono<Order> processOrder(OrderRequest request) {
        return Mono.fromCallable(() -> createOrder(request))
            .flatMap(order -> 
                Mono.zip(
                    processPayment(order),
                    updateInventory(order)
                ).thenReturn(order)
            )
            .timeout(Duration.ofSeconds(30))
            .onErrorResume(ex -> handleOrderError(request, ex));
    }
}
```

### 3.2 RxJava

```java
public class RxJavaExample {
    
    public Observable<String> searchItems(String query) {
        return Observable.fromCallable(() -> performSearch(query))
            .subscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread())
            .flatMap(results -> Observable.fromIterable(results))
            .filter(item -> item.isAvailable())
            .map(Item::getName)
            .doOnError(e -> log.error("搜索错误", e))
            .retry(3);
    }
    
    public Single<Response> makeHttpRequest(String url) {
        return httpClient.request(url)
            .timeout(10, TimeUnit.SECONDS)
            .subscribeOn(Schedulers.io())
            .observeOn(Schedulers.computation())
            .doOnSuccess(response -> log.info("请求成功: {}", url))
            .doOnError(e -> log.error("请求失败: {}", url, e));
    }
}
```

## 4. 响应式Web应用

### 4.1 Spring WebFlux

```java
@Configuration
@EnableWebFlux
public class WebFluxConfig {
    
    @Bean
    public RouterFunction<ServerResponse> routes(ProductHandler handler) {
        return RouterFunctions.route()
            .GET("/products", handler::getAllProducts)
            .GET("/products/{id}", handler::getProduct)
            .POST("/products", handler::createProduct)
            .PUT("/products/{id}", handler::updateProduct)
            .DELETE("/products/{id}", handler::deleteProduct)
            .build();
    }
}

@Component
public class ProductHandler {
    
    @Autowired
    private ProductService productService;
    
    public Mono<ServerResponse> getAllProducts(ServerRequest request) {
        return ServerResponse.ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(productService.findAll(), Product.class);
    }
    
    public Mono<ServerResponse> getProduct(ServerRequest request) {
        String id = request.pathVariable("id");
        return productService.findById(id)
            .flatMap(product -> 
                ServerResponse.ok()
                    .contentType(MediaType.APPLICATION_JSON)
                    .bodyValue(product)
            )
            .switchIfEmpty(ServerResponse.notFound().build());
    }
    
    public Mono<ServerResponse> createProduct(ServerRequest request) {
        return request.bodyToMono(Product.class)
            .flatMap(productService::save)
            .flatMap(product -> 
                ServerResponse.created(URI.create("/products/" + product.getId()))
                    .contentType(MediaType.APPLICATION_JSON)
                    .bodyValue(product)
            );
    }
}

### 4.2 响应式数据访问

```java
@Configuration
public class ReactiveMongoConfig {
    
    @Bean
    public ReactiveMongoTemplate reactiveMongoTemplate(ReactiveMongoClient mongoClient) {
        return new ReactiveMongoTemplate(mongoClient, "productDb");
    }
}

@Repository
public interface ProductRepository extends ReactiveMongoRepository<Product, String> {
    
    Flux<Product> findByCategory(String category);
    
    Flux<Product> findByPriceBetween(double minPrice, double maxPrice);
    
    Mono<Product> findByName(String name);
}

@Service
public class ProductService {
    
    @Autowired
    private ProductRepository productRepository;
    
    public Flux<Product> findAll() {
        return productRepository.findAll();
    }
    
    public Mono<Product> findById(String id) {
        return productRepository.findById(id);
    }
    
    public Flux<Product> findByCategory(String category) {
        return productRepository.findByCategory(category);
    }
    
    public Mono<Product> save(Product product) {
        return productRepository.save(product);
    }
    
    public Mono<Void> delete(String id) {
        return productRepository.deleteById(id);
    }
}
```

## 5. 响应式系统集成

### 5.1 响应式消息处理

```java
@Configuration
public class ReactiveMessagingConfig {
    
    @Bean
    public Flux<Message> messageFlux(MessageSource source) {
        return Flux.from(source)
            .publishOn(Schedulers.parallel())
            .doOnNext(this::logMessage)
            .share();
    }
    
    @Bean
    public Consumer<Flux<Message>> messageConsumer(MessageProcessor processor) {
        return flux -> flux
            .filter(Message::isValid)
            .flatMap(processor::process)
            .subscribe();
    }
}

@Service
public class ReactiveKafkaConsumer {
    
    private final KafkaReceiver<String, String> receiver;
    
    public ReactiveKafkaConsumer(ReceiverOptions<String, String> receiverOptions) {
        this.receiver = KafkaReceiver.create(receiverOptions);
    }
    
    public Flux<ConsumerRecord<String, String>> consumeMessages() {
        return receiver.receive()
            .doOnNext(record -> log.info("收到消息: {}", record.value()))
            .doOnError(e -> log.error("消费错误", e));
    }
}
```

### 5.2 响应式HTTP客户端

```java
@Service
public class ReactiveHttpClient {
    
    private final WebClient webClient;
    
    public ReactiveHttpClient(WebClient.Builder webClientBuilder) {
        this.webClient = webClientBuilder
            .baseUrl("https://api.example.com")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .filter(ExchangeFilterFunctions.basicAuthentication("username", "password"))
            .build();
    }
    
    public Mono<Product> getProduct(String id) {
        return webClient.get()
            .uri("/products/{id}", id)
            .retrieve()
            .onStatus(HttpStatus::is4xxClientError, response -> 
                Mono.error(new ProductNotFoundException("产品不存在: " + id))
            )
            .onStatus(HttpStatus::is5xxServerError, response -> 
                Mono.error(new ServiceUnavailableException("服务不可用"))
            )
            .bodyToMono(Product.class)
            .retryWhen(Retry.backoff(3, Duration.ofSeconds(1))
                .filter(ex -> ex instanceof ServiceUnavailableException)
            )
            .timeout(Duration.ofSeconds(5));
    }
    
    public Flux<Product> searchProducts(Map<String, String> queryParams) {
        return webClient.get()
            .uri(uriBuilder -> {
                queryParams.forEach(uriBuilder::queryParam);
                return uriBuilder.path("/products/search").build();
            })
            .retrieve()
            .bodyToFlux(Product.class);
    }
}
```

## 6. 最佳实践

### 6.1 错误处理

```java
public class ErrorHandlingExample {
    
    public Mono<Response> executeWithErrorHandling() {
        return callExternalService()
            .onErrorResume(ConnectionException.class, ex -> {
                log.warn("连接错误，尝试备用服务", ex);
                return callBackupService();
            })
            .onErrorResume(TimeoutException.class, ex -> {
                log.warn("服务超时，返回缓存数据", ex);
                return getCachedData();
            })
            .onErrorMap(ex -> !(ex instanceof BusinessException), 
                ex -> new SystemException("系统错误", ex))
            .doOnError(ex -> log.error("处理请求时发生错误", ex))
            .timeout(Duration.ofSeconds(30))
            .retryWhen(Retry.backoff(3, Duration.ofSeconds(2))
                .filter(ex -> isRetryable(ex))
                .onRetryExhaustedThrow((spec, ex) -> ex));
    }
}
```

### 6.2 测试响应式代码

```java
@ExtendWith(SpringExtension.class)
@WebFluxTest(ProductController.class)
public class ProductControllerTest {
    
    @Autowired
    private WebTestClient webTestClient;
    
    @MockBean
    private ProductService productService;
    
    @Test
    public void testGetProduct() {
        Product product = new Product("1", "测试产品", 100.0);
        
        when(productService.findById("1"))
            .thenReturn(Mono.just(product));
        
        webTestClient.get()
            .uri("/products/1")
            .exchange()
            .expectStatus().isOk()
            .expectBody(Product.class)
            .isEqualTo(product);
    }
    
    @Test
    public void testGetAllProducts() {
        List<Product> products = Arrays.asList(
            new Product("1", "产品1", 100.0),
            new Product("2", "产品2", 200.0)
        );
        
        when(productService.findAll())
            .thenReturn(Flux.fromIterable(products));
        
        webTestClient.get()
            .uri("/products")
            .exchange()
            .expectStatus().isOk()
            .expectBodyList(Product.class)
            .hasSize(2)
            .contains(products.toArray(new Product[0]));
    }
    
    @Test
    public void testCreateProduct() {
        Product product = new Product(null, "新产品", 150.0);
        Product savedProduct = new Product("3", "新产品", 150.0);
        
        when(productService.save(any(Product.class)))
            .thenReturn(Mono.just(savedProduct));
        
        webTestClient.post()
            .uri("/products")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue(product)
            .exchange()
            .expectStatus().isCreated()
            .expectBody(Product.class)
            .isEqualTo(savedProduct);
    }
}
```

## 7. 总结

响应式编程为构建高性能、可伸缩的应用提供了强大的编程模型。在实践中需要注意：

1. 理解并正确使用响应式流的核心概念
2. 合理处理背压机制，避免内存溢出
3. 选择合适的响应式框架和工具
4. 实现可靠的错误处理和重试机制
5. 注意响应式代码的测试和调试

## 参考资源

1. Project Reactor文档：https://projectreactor.io/docs
2. RxJava文档：https://github.com/ReactiveX/RxJava
3. Spring WebFlux文档：https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html
4. 响应式宣言：https://www.reactivemanifesto.org/