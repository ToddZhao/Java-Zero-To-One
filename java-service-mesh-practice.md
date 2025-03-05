# Day 126: Java云原生高级 - Service Mesh实践

## 1. Service Mesh概述

Service Mesh（服务网格）是一个基础设施层，用于处理服务间通信，负责在现代云原生应用的复杂服务拓扑中可靠地传递请求。与传统的服务间通信方式不同，Service Mesh不需要修改应用代码，而是通过部署轻量级网络代理（通常称为Sidecar）来实现服务间通信的控制和管理。

## 2. Service Mesh核心概念

### 2.1 数据平面与控制平面

```java
// 数据平面示例：使用Envoy作为Sidecar代理
// 在Java应用中，我们不需要直接编写Envoy配置，而是通过控制平面来管理
// 以下是一个简单的Spring Boot应用，它将与Envoy Sidecar一起部署

@SpringBootApplication
public class ServiceMeshApplication {

    public static void main(String[] args) {
        SpringApplication.run(ServiceMeshApplication.class, args);
    }
}

@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    private final ProductService productService;
    
    public ProductController(ProductService productService) {
        this.productService = productService;
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<Product> getProduct(@PathVariable String id) {
        return productService.getProduct(id)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }
    
    @PostMapping
    public ResponseEntity<Product> createProduct(@RequestBody Product product) {
        Product created = productService.createProduct(product);
        return ResponseEntity.created(URI.create("/api/products/" + created.getId()))
                .body(created);
    }
}
```

### 2.2 Istio架构

```java
// 在Istio环境中运行的服务，通常不需要特殊的代码修改
// 但我们可以添加一些与Istio集成的功能，如分布式追踪

@Configuration
public class IstioConfig {
    
    @Bean
    public Filter traceIdPropagationFilter() {
        return new Filter() {
            @Override
            public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
                    throws IOException, ServletException {
                HttpServletRequest httpRequest = (HttpServletRequest) request;
                String traceId = httpRequest.getHeader("x-b3-traceid");
                String spanId = httpRequest.getHeader("x-b3-spanid");
                
                if (traceId != null) {
                    MDC.put("traceId", traceId);
                }
                if (spanId != null) {
                    MDC.put("spanId", spanId);
                }
                
                try {
                    chain.doFilter(request, response);
                } finally {
                    MDC.remove("traceId");
                    MDC.remove("spanId");
                }
            }
        };
    }
    
    @Bean
    public RestTemplate restTemplate() {
        RestTemplate restTemplate = new RestTemplate();
        restTemplate.getInterceptors().add((request, body, execution) -> {
            String traceId = MDC.get("traceId");
            String spanId = MDC.get("spanId");
            
            if (traceId != null) {
                request.getHeaders().add("x-b3-traceid", traceId);
            }
            if (spanId != null) {
                request.getHeaders().add("x-b3-spanid", spanId);
            }
            
            return execution.execute(request, body);
        });
        return restTemplate;
    }
}
```

## 3. Service Mesh实现

### 3.1 Istio部署与配置

```yaml
# Kubernetes部署清单示例
# product-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
  labels:
    app: product-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: product-service
  template:
    metadata:
      labels:
        app: product-service
    spec:
      containers:
      - name: product-service
        image: my-registry/product-service:1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"
        resources:
          limits:
            cpu: "1"
            memory: "512Mi"
          requests:
            cpu: "0.5"
            memory: "256Mi"
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: product-service
spec:
  selector:
    app: product-service
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
```

### 3.2 流量管理

```yaml
# Istio流量管理配置示例
# virtual-service.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: product-service
spec:
  hosts:
  - product-service
  http:
  - match:
    - headers:
        end-user:
          exact: beta-tester
    route:
    - destination:
        host: product-service
        subset: v2
  - route:
    - destination:
        host: product-service
        subset: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: product-service
spec:
  host: product-service
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

## 4. Service Mesh高级功能

### 4.1 熔断与限流

```yaml
# 熔断配置示例
# circuit-breaker.yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: product-service
spec:
  host: product-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 10
        maxRequestsPerConnection: 10
    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 60s
      maxEjectionPercent: 50
```

### 4.2 安全通信

```yaml
# mTLS配置示例
# mtls-policy.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: default
spec:
  mtls:
    mode: STRICT
```

```java
// 在应用中，我们不需要处理TLS，因为这是由Istio处理的
// 但我们可以添加一些安全相关的配置

@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests()
                .antMatchers("/actuator/health").permitAll()
                .anyRequest().authenticated()
            .and()
            .oauth2ResourceServer()
                .jwt();
    }
    
    @Bean
    public JwtDecoder jwtDecoder() {
        return JwtDecoders.fromOidcIssuerLocation("https://auth-server/.well-known/openid-configuration");
    }
}
```

## 5. 可观测性与监控

### 5.1 分布式追踪

```java
@Configuration
public class TracingConfig {
    
    @Bean
    public Tracer jaegerTracer() {
        return io.jaegertracing.Configuration.fromEnv("product-service")
                .getTracerBuilder()
                .build();
    }
    
    @Bean
    public Filter tracingFilter(Tracer tracer) {
        return new TracingFilter(tracer);
    }
    
    public class TracingFilter implements Filter {
        private final Tracer tracer;
        
        public TracingFilter(Tracer tracer) {
            this.tracer = tracer;
        }
        
        @Override
        public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
                throws IOException, ServletException {
            HttpServletRequest httpRequest = (HttpServletRequest) request;
            HttpServletResponse httpResponse = (HttpServletResponse) response;
            
            Span span = tracer.buildSpan(httpRequest.getMethod() + " " + httpRequest.getRequestURI())
                    .withTag(Tags.SPAN_KIND.getKey(), Tags.SPAN_KIND_SERVER)
                    .withTag(Tags.HTTP_METHOD.getKey(), httpRequest.getMethod())
                    .withTag(Tags.HTTP_URL.getKey(), httpRequest.getRequestURL().toString())
                    .start();
            
            try (Scope scope = tracer.scopeManager().activate(span)) {
                chain.doFilter(request, response);
                span.setTag(Tags.HTTP_STATUS.getKey(), httpResponse.getStatus());
            } catch (Exception e) {
                Tags.ERROR.set(span, true);
                span.log(Map.of("event", "error", "error.object", e));
                throw e;
            } finally {
                span.finish();
            }
        }
    }
}
```

### 5.2 指标收集

```java
@Configuration
public class MetricsConfig {
    
    @Bean
    MeterRegistryCustomizer<MeterRegistry> metricsCommonTags() {
        return registry -> registry.config().commonTags(
            "application", "product-service",
            "environment", System.getenv().getOrDefault("ENVIRONMENT", "dev")
        );
    }
    
    @Bean
    public TimedAspect timedAspect(MeterRegistry registry) {
        return new TimedAspect(registry);
    }
}

@Service
public class ProductService {
    
    private final ProductRepository repository;
    private final Counter productCreationCounter;
    private final Timer productRetrievalTimer;
    
    public ProductService(ProductRepository repository, MeterRegistry registry) {
        this.repository = repository;
        this.productCreationCounter = registry.counter("product.creation");
        this.productRetrievalTimer = registry.timer("product.retrieval");
    }
    
    @Timed(value = "product.create", description = "Time taken to create a product")
    public Product createProduct(Product product) {
        Product created = repository.save(product);
        productCreationCounter.increment();
        return created;
    }
    
    public Optional<Product> getProduct(String id) {
        return productRetrievalTimer.record(() -> repository.findById(id));
    }
}
```

## 6. 最佳实践

### 6.1 渐进式迁移

```java
// 在迁移到Service Mesh的过程中，可以采用混合模式
// 例如，使用Spring Cloud与Istio结合的方式

@SpringBootApplication
@EnableDiscoveryClient
public class HybridServiceMeshApplication {

    public static void main(String[] args) {
        SpringApplication.run(HybridServiceMeshApplication.class, args);
    }
    
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
    
    @Bean
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder();
    }
}

@Service
public class HybridProductService {
    
    private final RestTemplate restTemplate;
    private final WebClient.Builder webClientBuilder;
    
    public HybridProductService(RestTemplate restTemplate, WebClient.Builder webClientBuilder) {
        this.restTemplate = restTemplate;
        this.webClientBuilder = webClientBuilder;
    }
    
    // 使用Spring Cloud方式调用服务
    public Product getProductViaRibbon(String id) {
        return restTemplate.getForObject("http://inventory-service/api/inventory/{id}", Product.class, id);
    }
    
    // 使用Service Mesh方式调用服务
    public Mono<Product> getProductViaServiceMesh(String id) {
        return webClientBuilder.build()
                .get()
                .uri("http://inventory-service/api/inventory/{id}", id)
                .retrieve()
                .bodyToMono(Product.class);
    }
}
```

### 6.2 故障注入测试

```yaml
# 故障注入配置示例
# fault-injection.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: product-service
spec:
  hosts:
  - product-service
  http:
  - fault:
      delay:
        percentage:
          value: 10
        fixedDelay: 5s
      abort:
        percentage:
          value: 5
        httpStatus: 500
    route: