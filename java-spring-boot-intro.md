# Day 21: Java Spring Boot入门

## 引言

Spring Boot是Spring框架的扩展，它消除了设置Spring应用程序所需的大量样板配置。本文将详细介绍Spring Boot的核心概念和快速开始指南。

## 1. Spring Boot基础

### 1.1 创建项目

```xml
<!-- pom.xml -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.7.0</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### 1.2 主应用类

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

## 2. Web应用开发

### 2.1 REST控制器

```java
@RestController
@RequestMapping("/api")
public class UserController {
    @Autowired
    private UserService userService;
    
    @GetMapping("/users")
    public List<User> getUsers() {
        return userService.findAll();
    }
    
    @GetMapping("/users/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        return userService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    @PostMapping("/users")
    public ResponseEntity<User> createUser(@RequestBody User user) {
        User savedUser = userService.save(user);
        return ResponseEntity
            .created(URI.create("/api/users/" + savedUser.getId()))
            .body(savedUser);
    }
}
```

### 2.2 服务层

```java
@Service
@Transactional
public class UserService {
    @Autowired
    private UserRepository userRepository;
    
    public List<User> findAll() {
        return userRepository.findAll();
    }
    
    public Optional<User> findById(Long id) {
        return userRepository.findById(id);
    }
    
    public User save(User user) {
        return userRepository.save(user);
    }
}
```

## 3. 数据访问

### 3.1 JPA配置

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb
    username: root
    password: password
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
```

### 3.2 实体和仓库

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String username;
    
    @Email
    private String email;
    
    // getters and setters
}

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username);
    List<User> findByEmailContaining(String email);
}
```

## 4. 配置管理

### 4.1 属性配置

```yaml
# application.yml
server:
  port: 8080
  servlet:
    context-path: /api

app:
  name: MyApp
  description: Spring Boot Demo Application
  security:
    jwt-secret: your-secret-key
    token-validity: 86400
```

### 4.2 配置类

```java
@Configuration
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private String name;
    private String description;
    private Security security;
    
    public static class Security {
        private String jwtSecret;
        private long tokenValidity;
        
        // getters and setters
    }
    
    // getters and setters
}
```

## 5. 异常处理

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleResourceNotFound(
            ResourceNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.NOT_FOUND.value(),
            ex.getMessage());
        return new ResponseEntity<>(error, HttpStatus.NOT_FOUND);
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGlobalException(
            Exception ex) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.INTERNAL_SERVER_ERROR.value(),
            "服务器内部错误");
        return new ResponseEntity<>(error, 
            HttpStatus.INTERNAL_SERVER_ERROR);
    }
}

public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}
```

## 6. 测试

```java
@SpringBootTest
@AutoConfigureMockMvc
public class UserControllerTest {
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    public void testGetUser() throws Exception {
        User user = new User();
        user.setUsername("testuser");
        user.setEmail("test@example.com");
        userRepository.save(user);
        
        mockMvc.perform(get("/api/users/" + user.getId()))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.username").value("testuser"))
            .andExpect(jsonPath("$.email").value("test@example.com"));
    }
    
    @Test
    public void testCreateUser() throws Exception {
        User user = new User();
        user.setUsername("newuser");
        user.setEmail("new@example.com");
        
        mockMvc.perform(post("/api/users")
            .contentType(MediaType.APPLICATION_JSON)
            .content(new ObjectMapper().writeValueAsString(user)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.username").value("newuser"));
    }
}
```

## 7. 实践案例

### 7.1 文件上传服务

```java
@Service
public class FileStorageService {
    private final Path fileStorageLocation;
    
    @Autowired
    public FileStorageService(@Value("${file.upload-dir}") String uploadDir) {
        this.fileStorageLocation = Paths.get(uploadDir)
            .toAbsolutePath().normalize();
        
        try {
            Files.createDirectories(this.fileStorageLocation);
        } catch (Exception ex) {
            throw new FileStorageException("无法创建文件上传目录", ex);
        }
    }
    
    public String storeFile(MultipartFile file) {
        String fileName = StringUtils.cleanPath(file.getOriginalFilename());
        
        try {
            Path targetLocation = this.fileStorageLocation.resolve(fileName);
            Files.copy(file.getInputStream(), targetLocation,
                StandardCopyOption.REPLACE_EXISTING);
            
            return fileName;
        } catch (IOException ex) {
            throw new FileStorageException("无法存储文件 " + fileName, ex);
        }
    }
}

@RestController
public class FileController {
    @Autowired
    private FileStorageService fileStorageService;
    
    @PostMapping("/upload")
    public ResponseEntity<String> uploadFile(
            @RequestParam("file") MultipartFile file) {
        String fileName = fileStorageService.storeFile(file);
        return ResponseEntity.ok("文件上传成功：" + fileName);
    }
}
```

## 8. 最佳实践

1. 使用适当的项目结构
2. 遵循RESTful API设计原则
3. 正确处理异常和错误
4. 编写完整的测试用例
5. 使用适当的日志级别

## 总结

本文介绍了Spring Boot的核心概念和实践应用，包括：

1. 项目创建和配置
2. Web应用开发
3. 数据访问集成
4. 配置管理
5. 异常处理
6. 测试方法
7. 实践案例

通过掌握这些知识，我们可以快速开发出高质量的Spring Boot应用程序。

## 参考资源

1. Spring Boot官方文档
2. Spring Boot实战（第2版）
3. Spring Boot测试指南
4. RESTful API设计最佳实践