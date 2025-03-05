# Day 119: Java微服务高级 - API网关设计

## 1. API网关概述

API网关是微服务架构中的重要组件，它作为系统的统一入口，负责请求路由、负载均衡、认证授权、限流熔断等功能。API网关可以帮助我们解决微服务架构中的跨切面关注点，提供统一的API管理和安全控制。

## 2. Spring Cloud Gateway实现

### 2.1 基本配置

```java
@Configuration
public class GatewayConfig {
    
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("product_route", r -> r
                .path("/api/products/**")
                .filters(f -> f
                    .stripPrefix(1)
                    .addRequestHeader("X-Request-Source", "API-Gateway")
                    .requestRateLimiter(c -> c
                        .setRateLimiter(redisRateLimiter())
                        .setKeyResolver(userKeyResolver())))
                .uri("lb://product-service"))
            .route("order_route", r -> r
                .path("/api/orders/**")
                .filters(f -> f
                    .circuitBreaker(c -> c
                        .setName("orderCircuitBreaker")
                        .setFallbackUri("forward:/fallback/orders")))
                .uri("lb://order-service"))
            .build();
    }
    
    @Bean
    public RedisRateLimiter redisRateLimiter() {
        return new RedisRateLimiter(10, 20); // 令牌桶容量和补充速率
    }
    
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> Mono.just(exchange.getRequest()
            .getHeaders()
            .getFirst("X-User-Id"));
    }
}
```

### 2.2 自定义过滤器

```java
@Component
public class AuthenticationFilter extends AbstractGatewayFilterFactory<AuthenticationFilter.Config> {
    
    @Autowired
    private JwtTokenProvider tokenProvider;
    
    public AuthenticationFilter() {
        super(Config.class);
    }
    
    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            ServerHttpRequest request = exchange.getRequest();
            
            // 检查JWT token
            String token = extractToken(request);
            if (!tokenProvider.validateToken(token)) {
                return onError(exchange, "Invalid token", HttpStatus.UNAUTHORIZED);
            }
            
            // 添加用户信息到请求头
            String userId = tokenProvider.getUserIdFromToken(token);
            ServerHttpRequest modifiedRequest = request.mutate()
                .header("X-User-Id", userId)
                .build();
            
            return chain.filter(exchange.mutate().request(modifiedRequest).build());
        };
    }
    
    private String extractToken(ServerHttpRequest request) {
        List<String> headers = request.getHeaders().get("Authorization");
        if (headers != null && !headers.isEmpty()) {
            String header = headers.get(0);
            if (header.startsWith("Bearer ")) {
                return header.substring(7);
            }
        }
        return null;
    }
    
    private Mono<Void> onError(ServerWebExchange exchange, String message, HttpStatus status) {
        ServerHttpResponse response = exchange.getResponse();
        response.setStatusCode(status);
        return response.writeWith(Mono.just(response.bufferFactory()
            .wrap(message.getBytes())));
    }
    
    public static class Config {
        // 配置属性
    }
}
```

## 3. 高级功能实现

### 3.1 请求聚合

```java
@Component
public class RequestAggregator {
    
    @Autowired
    private WebClient.Builder webClientBuilder;
    
    public Mono<ProductDetails> getProductDetails(String productId) {
        return Mono.zip(
            getProduct(productId),
            getInventory(productId),
            getReviews(productId)
        ).map(tuple -> {
            ProductDetails details = new ProductDetails();
            details.setProduct(tuple.getT1());
            details.setInventory(tuple.getT2());
            details.setReviews(tuple.getT3());
            return details;
        });
    }
    
    private Mono<Product> getProduct(String productId) {
        return webClientBuilder.build()
            .get()
            .uri("http://product-service/products/" + productId)
            .retrieve()
            .bodyToMono(Product.class);
    }
    
    private Mono<Inventory> getInventory(String productId) {
        return webClientBuilder.build()
            .get()
            .uri("http://inventory-service/inventory/" + productId)
            .retrieve()
            .bodyToMono(Inventory.class);
    }
    
    private Mono<List<Review>> getReviews(String productId) {
        return webClientBuilder.build()
            .get()
            .uri("http://review-service/reviews/product/" + productId)
            .retrieve()
            .bodyToFlux(Review.class)
            .collectList();
    }
}
```

### 3.2 响应转换

```java
@Component
public class ResponseTransformer extends AbstractGatewayFilterFactory<ResponseTransformer.Config> {
    
    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> chain.filter(exchange)
            .then(Mono.defer(() -> {
                ServerHttpResponse response = exchange.getResponse();
                
                // 获取原始响应数据
                DataBufferFactory dataBufferFactory = response.bufferFactory();
                return response.writeWith(response.getBody()
                    .map(dataBuffer -> {
                        byte[] content = new byte[dataBuffer.readableByteCount()];
                        dataBuffer.read(content);
                        DataBufferUtils.release(dataBuffer);
                        
                        // 转换响应
                        String transformedContent = transformResponse(
                            new String(content, StandardCharsets.UTF_8));
                        
                        return dataBufferFactory.wrap(
                            transformedContent.getBytes(StandardCharsets.UTF_8));
                    }));
            }));
    }
    
    private String transformResponse(String originalContent) {
        // 实现响应转换逻辑
        return originalContent;
    }
    
    public static class Config {
        // 配置属性
    }
}
```

## 4. 安全性实现

### 4.1 OAuth2集成

```java
@Configuration
@EnableWebFluxSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityWebFilterChain springSecurityFilterChain(ServerHttpSecurity http) {
        return http
            .oauth2ResourceServer()
                .jwt()
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
            .and()
            .authorizeExchange()
                .pathMatchers("/public/**").permitAll()
                .pathMatchers("/api/**").authenticated()
            .and()
            .build();
    }
    
    @Bean
    public ReactiveJwtAuthenticationConverter jwtAuthenticationConverter() {
        return new ReactiveJwtAuthenticationConverter();
    }
}
```

### 4.2 CORS配置

```java
@Configuration
public class CorsConfig {
    
    @Bean
    public CorsWebFilter corsWebFilter() {
        CorsConfiguration config = new CorsConfiguration();
        
        // 允许的来源
        config.addAllowedOrigin("*");
        
        // 允许的HTTP方法
        config.addAllowedMethod("*");
        
        // 允许的请求头
        config.addAllowedHeader("*");
        
        // 是否允许携带认证信息
        config.setAllowCredentials(true);
        
        // 预检请求的有效期
        config.setMaxAge(3600L);
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        
        return new CorsWebFilter(source);
    }
}
```

## 5. 性能优化

### 5.1 缓存优化

```java
@Component
public class CachingFilter extends AbstractGatewayFilterFactory<CachingFilter.Config> {
    
    @Autowired
    private ReactiveRedisTemplate<String, String> redisTemplate;
    
    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            String cacheKey = generateCacheKey(exchange.getRequest());
            
            return redisTemplate.opsForValue().get(cacheKey)
                .switchIfEmpty(Mono.defer(() -> chain.filter(exchange)
                    .then(Mono.defer(() -> {
                        ServerHttpResponse response = exchange.getResponse();
                        return response.writeWith(response.getBody()
                            .map(dataBuffer -> {
                                byte[] content = new byte[dataBuffer.readableByteCount()];
                                dataBuffer.read(content);
                                
                                // 缓存响应
                                String responseBody = new String(content, StandardCharsets.UTF_8);
                                redisTemplate.opsForValue().set(cacheKey, responseBody, 
                                    Duration.ofMinutes(10));
                                
                                return exchange.getResponse().bufferFactory()
                                    .wrap(content);
                            }));
                    })));
        };
    }
    
    private String generateCacheKey(ServerHttpRequest request) {
        return "cache:" + request.getURI().getPath() + ":" + 
               request.getQueryParams().toString();
    }
    
    public static class Config {
        // 配置属性
    }
}
```

### 5.2 负载均衡优化

```java
@Configuration
public class LoadBalancerConfig {
    
    @Bean
    public ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(
            LoadBalancerClientFactory factory, Environment environment) {
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new WeightedResponseTimeLoadBalancer(factory
            .getLazyProvider(name, ServiceInstanceListSupplier.class),
            name);
    }
}

public class WeightedResponseTimeLoadBalancer 
        implements ReactorServiceInstanceLoadBalancer {
    
    private final ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider;
    private final String serviceId;
    private final Map<String, Double> responseTimeWeights = new ConcurrentHashMap<>();
    
    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        return serviceInstanceListSupplierProvider.getIfAvailable()
            .get(request)
            .map(instances -> {
                // 根据响应时间权重选择实例
                return chooseInstance(instances);
            });
    }
    
    private Response<ServiceInstance> chooseInstance(List<ServiceInstance> instances) {
        // 实现加权轮询算法
        return null;
    }
}
```

## 6. 监控与运维

### 6.1 指标收集

```java
@Configuration
public class MetricsConfig {
    
    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCommonTags() {
        return registry -> registry.config().commonTags(
            "application", "api-gateway");
    }
    
    @Bean
    public TimedAspect timedAspect(MeterRegistry registry) {
        return new TimedAspect(registry);
    }
}

@Component
public class MetricsFilter extends AbstractGatewayFilterFactory<MetricsFilter.Config> {
    
    private final MeterRegistry meterRegistry;
    
    public MetricsFilter(MeterRegistry meterRegistry) {
        super(Config.class);
        this.meterRegistry = meterRegistry;
    }
    
    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            long startTime = System.currentTimeMillis();
            
            return chain.filter(exchange)
                .doFinally(signalType -> {
                    long duration = System.currentTimeMillis() - startTime;
                    
                    // 记录请求指标
                    Timer.builder("gateway.request.duration")
                        .tag("path", exchange.getRequest().getPath().value())
                        .tag("method", exchange.getRequest().getMethod().name())
                        .tag("status", String.valueOf(
                            exchange.getResponse().getStatusCode().value()))
                        .register(meterRegistry)
                        .record(duration, TimeUnit.MILLISECONDS);
                });
        };
    }
    
    public static class Config {
        // 配置属性
    }
}
```

### 6.2 日志追踪

```java
@Component
public class LoggingFilter extends AbstractGatewayFilterFactory<LoggingFilter.Config> {
    
    private static final Logger log = LoggerFactory.getLogger(LoggingFilter.class);
    
    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            // 生成请求ID
            String requestId = UUID.randomUUID().toString();