# Day 24: Java Spring Boot测试和部署

## 引言

Spring Boot提供了全面的测试支持，并且可以轻松部署到各种环境中。本文将详细介绍Spring Boot应用的测试方法和部署策略。

## 1. 单元测试

### 1.1 基本测试配置

```java
@SpringBootTest
class UserServiceTest {
    @Autowired
    private UserService userService;
    
    @MockBean
    private UserRepository userRepository;
    
    @Test
    void findByIdShouldReturnUser() {
        // 准备测试数据
        User user = new User();
        user.setId(1L);
        user.setUsername("testuser");
        
        // 配置Mock行为
        when(userRepository.findById(1L)).thenReturn(Optional.of(user));
        
        // 执行测试
        User found = userService.findById(1L);
        
        // 验证结果
        assertNotNull(found);
        assertEquals("testuser", found.getUsername());
        verify(userRepository).findById(1L);
    }
}
```

### 1.2 Mock测试

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock
    private PaymentService paymentService;
    
    @InjectMocks
    private OrderService orderService;
    
    @Test
    void createOrderShouldProcessPayment() {
        // 准备测试数据
        Order order = new Order();
        order.setAmount(new BigDecimal("100.00"));
        
        // 配置Mock行为
        when(paymentService.processPayment(any(BigDecimal.class)))
            .thenReturn(true);
        
        // 执行测试
        boolean result = orderService.createOrder(order);
        
        // 验证结果
        assertTrue(result);
        verify(paymentService).processPayment(order.getAmount());
    }
}
```

## 2. 集成测试

### 2.1 Web层测试

```java
@SpringBootTest
@AutoConfigureMockMvc
class UserControllerTest {
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @Test
    void createUserShouldReturnCreated() throws Exception {
        // 准备测试数据
        User user = new User();
        user.setUsername("newuser");
        user.setEmail("newuser@example.com");
        
        // 执行测试
        mockMvc.perform(post("/api/users")
            .contentType(MediaType.APPLICATION_JSON)
            .content(objectMapper.writeValueAsString(user)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.username").value("newuser"))
            .andExpect(jsonPath("$.email").value("newuser@example.com"));
    }
}
```

### 2.2 数据层测试

```java
@DataJpaTest
class UserRepositoryTest {
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void findByUsernameShouldReturnUser() {
        // 准备测试数据
        User user = new User();
        user.setUsername("testuser");
        user.setEmail("test@example.com");
        userRepository.save(user);
        
        // 执行测试
        Optional<User> found = userRepository.findByUsername("testuser");
        
        // 验证结果
        assertTrue(found.isPresent());
        assertEquals("test@example.com", found.get().getEmail());
    }
}
```

## 3. 性能测试

### 3.1 负载测试

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class PerformanceTest {
    @LocalServerPort
    private int port;
    
    @Test
    void loadTest() {
        // 创建测试配置
        HttpClientConfig clientConfig = HttpClientConfig.custom()
            .setConnectTimeout(5000)
            .setMaxConnections(100)
            .build();
        
        // 执行负载测试
        LoadTestBuilder.withConfig(clientConfig)
            .withConcurrentUsers(50)
            .withDuration(Duration.ofMinutes(5))
            .withEndpoint(String.format("http://localhost:%d/api/users", port))
            .build()
            .execute();
    }
}
```

## 4. 部署策略

### 4.1 打包配置

```xml
<!-- pom.xml -->
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <executable>true</executable>
                <layers>
                    <enabled>true</enabled>
                </layers>
            </configuration>
        </plugin>
    </plugins>
</build>
```

### 4.2 Docker部署

```dockerfile
# Dockerfile
FROM openjdk:11-jre-slim
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

### 4.3 Kubernetes部署

```yaml
# deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:latest
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: prod
        - name: JAVA_OPTS
          value: "-Xmx512m -Xms256m"
```

## 5. 监控和日志

### 5.1 Actuator配置

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,info,prometheus
  endpoint:
    health:
      show-details: always
```

### 5.2 日志配置

```yaml
# application.yml
logging:
  level:
    root: INFO
    com.example: DEBUG
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
  file:
    name: logs/application.log
    max-size: 10MB
    max-history: 7
```

## 6. 实践案例

### 6.1 CI/CD配置

```yaml
# .github/workflows/ci.yml
name: CI/CD

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Set up JDK 11
      uses: actions/setup-java@v2
      with:
        java-version: '11'
        distribution: 'adopt'
        
    - name: Build with Maven
      run: mvn clean package
      
    - name: Run Tests
      run: mvn test
      
    - name: Build Docker image
      run: docker build -t myapp .
      
    - name: Deploy to Kubernetes
      run: |
        kubectl apply -f k8s/
```

### 6.2 性能监控

```java
@Configuration
public class MetricsConfig {
    @Bean
    public MeterRegistry meterRegistry() {
        return new SimpleMeterRegistry();
    }
}

@Service
public class MetricsService {
    private final MeterRegistry meterRegistry;
    
    public MetricsService(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }
    
    public void recordOperation(String name, long timeInMs) {
        meterRegistry.timer("app.operation", "name", name)
            .record(timeInMs, TimeUnit.MILLISECONDS);
    }
}
```

## 7. 最佳实践

1. 编写全面的测试用例
2. 使用合适的测试策略
3. 实现自动化部署
4. 配置适当的监控
5. 建立完整的日志体系

## 总结

本文介绍了Spring Boot测试和部署的核心概念和实践应用，包括：

1. 单元测试和集成测试
2. 性能测试方法
3. 部署策略和配置
4. 监控和日志管理
5. CI/CD实践

通过掌握这些知识，我们可以更好地保证Spring Boot应用的质量，并实现高效的部署和运维。

## 参考资源

1. Spring Boot测试指南
2. Docker官方文档
3. Kubernetes入门指南
4. CI/CD最佳实践