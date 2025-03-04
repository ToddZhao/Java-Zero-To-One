# Day 17: Java Spring框架 - IoC和DI

## 引言

Spring框架的核心功能是IoC（控制反转）容器和DI（依赖注入）机制。这些特性使得应用程序更加模块化、可测试和可维护。本文将详细介绍Spring IoC和DI的核心概念及其实践应用。

## 1. IoC容器基础

### 1.1 什么是IoC

IoC（Inversion of Control）是一种设计原则，它将对象的创建和依赖关系的管理从代码中移出，交给容器来处理：

```java
// 传统方式
public class UserService {
    private UserDao userDao = new UserDaoImpl(); // 硬编码依赖
}

// IoC方式
public class UserService {
    private UserDao userDao; // 由容器注入依赖
    
    public void setUserDao(UserDao userDao) {
        this.userDao = userDao;
    }
}
```

### 1.2 Spring IoC容器

```java
// 配置类
@Configuration
public class AppConfig {
    @Bean
    public UserDao userDao() {
        return new UserDaoImpl();
    }
    
    @Bean
    public UserService userService() {
        UserService service = new UserService();
        service.setUserDao(userDao());
        return service;
    }
}

// 使用容器
public class Application {
    public static void main(String[] args) {
        ApplicationContext context = 
            new AnnotationConfigApplicationContext(AppConfig.class);
        UserService userService = context.getBean(UserService.class);
    }
}
```

## 2. 依赖注入

### 2.1 构造器注入

```java
@Service
public class UserServiceImpl implements UserService {
    private final UserDao userDao;
    private final EmailService emailService;
    
    @Autowired
    public UserServiceImpl(UserDao userDao, EmailService emailService) {
        this.userDao = userDao;
        this.emailService = emailService;
    }
}
```

### 2.2 Setter注入

```java
@Service
public class OrderService {
    private PaymentService paymentService;
    
    @Autowired
    public void setPaymentService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

### 2.3 字段注入

```java
@Service
public class ProductService {
    @Autowired
    private ProductDao productDao;
    
    @Autowired
    private PriceService priceService;
}
```

## 3. Bean的配置

### 3.1 Java配置

```java
@Configuration
@ComponentScan("com.example")
public class AppConfig {
    @Bean
    @Scope("singleton")
    public DataSource dataSource() {
        DriverManagerDataSource dataSource = new DriverManagerDataSource();
        dataSource.setDriverClassName("com.mysql.cj.jdbc.Driver");
        dataSource.setUrl("jdbc:mysql://localhost:3306/mydb");
        dataSource.setUsername("root");
        dataSource.setPassword("password");
        return dataSource;
    }
    
    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}
```

### 3.2 XML配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                           http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="userDao" class="com.example.UserDaoImpl"/>
    
    <bean id="userService" class="com.example.UserServiceImpl">
        <constructor-arg ref="userDao"/>
    </bean>
</beans>
```

## 4. 注解配置

### 4.1 组件扫描

```java
@Component
public class SimpleBean {
    // ...
}

@Repository
public class UserRepository {
    // ...
}

@Service
public class UserService {
    // ...
}

@Controller
public class UserController {
    // ...
}
```

### 4.2 自动装配

```java
@Configuration
@ComponentScan(basePackages = "com.example")
public class AppConfig {
    @Bean
    @Qualifier("primary")
    public DataSource primaryDataSource() {
        // 配置主数据源
        return new DriverManagerDataSource();
    }
    
    @Bean
    @Qualifier("secondary")
    public DataSource secondaryDataSource() {
        // 配置从数据源
        return new DriverManagerDataSource();
    }
}

@Service
public class DatabaseService {
    private final DataSource dataSource;
    
    @Autowired
    public DatabaseService(@Qualifier("primary") DataSource dataSource) {
        this.dataSource = dataSource;
    }
}
```

## 5. 实践案例

### 5.1 配置管理

```java
@Configuration
@PropertySource("classpath:application.properties")
public class AppProperties {
    @Value("${app.name}")
    private String appName;
    
    @Value("${app.description}")
    private String appDescription;
    
    // getters
}

// application.properties
app.name=MyApp
app.description=Spring IoC Demo Application
```

### 5.2 多环境配置

```java
@Configuration
@Profile("development")
public class DevConfig {
    @Bean
    public DataSource dataSource() {
        // 开发环境数据源配置
    }
}

@Configuration
@Profile("production")
public class ProdConfig {
    @Bean
    public DataSource dataSource() {
        // 生产环境数据源配置
    }
}
```

## 6. 最佳实践

1. 优先使用构造器注入
2. 避免使用字段注入
3. 使用Java配置代替XML配置
4. 合理使用组件扫描
5. 正确处理循环依赖

## 总结

本文介绍了Spring框架中IoC和DI的核心概念及实践应用，包括：

1. IoC容器的基本概念
2. 依赖注入的方式
3. Bean的配置方法
4. 注解的使用
5. 实践案例和最佳实践

通过掌握这些知识，我们可以更好地使用Spring框架开发可维护的应用程序。

## 参考资源

1. Spring官方文档：https://docs.spring.io/spring-framework/docs/current/reference/html/core.html
2. Spring实战（第5版）
3. Spring Boot实践指南
4. Spring依赖注入最佳实践