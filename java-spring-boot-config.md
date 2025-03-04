# Day 22: Java Spring Boot配置和自动化

## 引言

Spring Boot的一大特点是约定优于配置，它提供了丰富的自动配置功能和灵活的配置管理机制。本文将详细介绍Spring Boot的配置管理和自动化特性。

## 1. 配置管理

### 1.1 配置文件类型

```yaml
# application.yml
spring:
  application:
    name: my-application
  profiles:
    active: dev

---
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:mysql://localhost:3306/devdb
    username: dev
    password: dev123

---
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: jdbc:mysql://prod-server:3306/proddb
    username: prod
    password: prod123
```

### 1.2 配置属性绑定

```java
@Configuration
@ConfigurationProperties(prefix = "app")
@Validated
public class AppProperties {
    @NotEmpty
    private String name;
    
    @Min(1)
    @Max(100)
    private int maxConnections;
    
    private Map<String, String> settings;
    
    private Security security = new Security();
    
    public static class Security {
        private String apiKey;
        private List<String> allowedOrigins;
        
        // getters and setters
    }
    
    // getters and setters
}

// 使用示例
@Service
public class AppService {
    @Autowired
    private AppProperties appProperties;
    
    public void doSomething() {
        String appName = appProperties.getName();
        int maxConn = appProperties.getMaxConnections();
        // 使用配置属性
    }
}
```

## 2. 自动配置

### 2.1 条件注解

```java
@Configuration
@ConditionalOnClass(DataSource.class)
public class DatabaseAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnProperty(prefix = "app.datasource", name = "enabled", havingValue = "true")
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build();
    }
}

@Configuration
@ConditionalOnWebApplication
public class WebAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public WebMvcConfigurer corsConfigurer() {
        return new WebMvcConfigurer() {
            @Override
            public void addCorsMappings(CorsRegistry registry) {
                registry.addMapping("/api/**")
                    .allowedOrigins("*")
                    .allowedMethods("GET", "POST", "PUT", "DELETE");
            }
        };
    }
}
```

### 2.2 自定义启动器

```java
// 自动配置类
@Configuration
@ConditionalOnClass(MyService.class)
@EnableConfigurationProperties(MyProperties.class)
public class MyAutoConfiguration {
    
    @Autowired
    private MyProperties properties;
    
    @Bean
    @ConditionalOnMissingBean
    public MyService myService() {
        return new MyService(properties.getConfig());
    }
}

// 配置属性类
@ConfigurationProperties(prefix = "myapp")
public class MyProperties {
    private String config;
    // getters and setters
}

// 在 META-INF/spring.factories 中注册
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.MyAutoConfiguration
```

## 3. 外部化配置

### 3.1 命令行参数

```bash
java -jar myapp.jar --server.port=8081 --spring.profiles.active=prod
```

### 3.2 环境变量

```java
@Configuration
public class EnvConfig {
    @Value("${DATABASE_URL:jdbc:mysql://localhost:3306/mydb}")
    private String databaseUrl;
    
    @Value("${API_KEY:#{null}}")
    private String apiKey;
    
    @Bean
    public DataSource dataSource() {
        return DataSourceBuilder
            .create()
            .url(databaseUrl)
            .build();
    }
}
```

## 4. 配置加密

### 4.1 使用Jasypt加密

```java
@Configuration
public class JasyptConfig {
    @Bean
    public StringEncryptor stringEncryptor() {
        PooledPBEStringEncryptor encryptor = new PooledPBEStringEncryptor();
        SimpleStringPBEConfig config = new SimpleStringPBEConfig();
        config.setPassword("your-secret-key");
        config.setAlgorithm("PBEWithMD5AndDES");
        config.setKeyObtentionIterations(1000);
        config.setPoolSize(1);
        encryptor.setConfig(config);
        return encryptor;
    }
}

# application.yml
spring:
  datasource:
    password: ENC(encrypted-password-here)
```

## 5. 实践案例

### 5.1 多环境配置

```java
@Configuration
public class MultiEnvConfig {
    
    @Bean
    @Profile("dev")
    public DataSource devDataSource() {
        return DataSourceBuilder
            .create()
            .url("jdbc:h2:mem:devdb")
            .username("sa")
            .build();
    }
    
    @Bean
    @Profile("prod")
    public DataSource prodDataSource() {
        return DataSourceBuilder
            .create()
            .url("jdbc:mysql://prodhost:3306/proddb")
            .username("produser")
            .password("prodpass")
            .build();
    }
}
```

### 5.2 动态配置刷新

```java
@RefreshScope
@RestController
public class ConfigController {
    
    @Value("${app.message:Hello}")
    private String message;
    
    @GetMapping("/message")
    public String getMessage() {
        return message;
    }
}

@Configuration
@EnableScheduling
public class ConfigRefreshScheduler {
    
    @Autowired
    private ContextRefresher contextRefresher;
    
    @Scheduled(fixedRate = 30000)
    public void refreshConfig() {
        contextRefresher.refresh();
    }
}
```

## 6. 最佳实践

1. 使用类型安全的配置属性
2. 合理组织配置文件
3. 使用配置文件加密
4. 实现优雅的配置刷新
5. 遵循配置分离原则

## 总结

本文介绍了Spring Boot配置和自动化的核心概念和实践应用，包括：

1. 配置文件管理
2. 自动配置机制
3. 外部化配置
4. 配置加密
5. 实践案例

通过掌握这些知识，我们可以更好地管理Spring Boot应用的配置，提高开发效率和系统的可维护性。

## 参考资源

1. Spring Boot官方文档
2. Spring Cloud Config文档
3. Jasypt加密指南
4. Spring Boot配置最佳实践