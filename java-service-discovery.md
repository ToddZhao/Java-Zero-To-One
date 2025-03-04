# Day 41: Java微服务 - Service Discovery和Load Balancing

## 引言

服务发现和负载均衡是微服务架构中的关键组件，它们确保了服务的高可用性和可扩展性。本文将深入探讨Spring Cloud中的服务发现和负载均衡机制。

## 1. 服务发现基础

### 1.1 什么是服务发现

服务发现是一种自动检测服务实例的机制，主要功能包括：

- 服务注册
- 服务下线
- 服务健康检查
- 服务元数据管理

### 1.2 服务发现模式

- 客户端发现模式
- 服务端发现模式
- DNS服务发现
- 配置中心发现

## 2. Spring Cloud服务发现实现

### 2.1 Eureka服务端配置

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

配置文件application.yml：
```yaml
server:
  port: 8761
eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false
    fetchRegistry: false
  server:
    enableSelfPreservation: false
```

### 2.2 Eureka客户端配置

```java
@SpringBootApplication
@EnableDiscoveryClient
public class ServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServiceApplication.class, args);
    }
}
```

配置文件application.yml：
```yaml
spring:
  application:
    name: user-service
server:
  port: 8080
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```

## 3. 负载均衡实现

### 3.1 Ribbon配置

```java
@Configuration
public class RibbonConfig {
    @Bean
    public IRule ribbonRule() {
        // 使用轮询策略
        return new RoundRobinRule();
    }
    
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

### 3.2 负载均衡策略

```java
@Service
public class UserService {
    @Autowired
    private RestTemplate restTemplate;
    
    public User getUser(Long id) {
        // 使用服务名称调用
        return restTemplate.getForObject(
            "http://user-service/users/" + id,
            User.class
        );
    }
}
```

## 4. 高可用配置

### 4.1 Eureka集群配置

```yaml
# eureka-server-1
server:
  port: 8761
eureka:
  instance:
    hostname: eureka-server-1
  client:
    serviceUrl:
      defaultZone: http://eureka-server-2:8762/eureka/

# eureka-server-2
server:
  port: 8762
eureka:
  instance:
    hostname: eureka-server-2
  client:
    serviceUrl:
      defaultZone: http://eureka-server-1:8761/eureka/
```

### 4.2 服务实例配置

```yaml
eureka:
  instance:
    preferIpAddress: true
    lease-renewal-interval-in-seconds: 30
    lease-expiration-duration-in-seconds: 90
  client:
    serviceUrl:
      defaultZone: http://eureka-server-1:8761/eureka/,http://eureka-server-2:8762/eureka/
```

## 5. 实践案例

### 5.1 服务注册与发现

```java
@RestController
@RequestMapping("/users")
public class UserController {
    @Value("${server.port}")
    private String port;
    
    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return new User(id, "用户-" + port);
    }
}
```

### 5.2 自定义负载均衡策略

```java
public class CustomRule extends AbstractLoadBalancerRule {
    @Override
    public Server choose(Object key) {
        List<Server> servers = getLoadBalancer().getReachableServers();
        if (servers.isEmpty()) {
            return null;
        }
        // 实现自定义选择逻辑
        return servers.get(0);
    }
}
```

## 6. 最佳实践

1. 服务注册
   - 使用有意义的服务名称
   - 配置合适的健康检查间隔
   - 启用服务保护机制

2. 负载均衡
   - 选择合适的负载均衡策略
   - 配置重试机制
   - 监控服务调用情况

3. 高可用部署
   - 部署服务注册中心集群
   - 服务多实例部署
   - 跨区域部署

## 总结

通过本文的学习，我们掌握了：

1. 服务发现的基本概念和实现方式
2. Spring Cloud中的服务注册与发现配置
3. 负载均衡的实现和策略选择
4. 高可用部署方案
5. 实践中的注意事项

在实际项目中，需要根据业务规模和要求，选择合适的服务发现方案和负载均衡策略，确保系统的可用性和可扩展性。

## 参考资源

1. Spring Cloud Netflix文档：https://cloud.spring.io/spring-cloud-netflix/
2. Eureka Wiki：https://github.com/Netflix/eureka/wiki
3. Ribbon文档：https://github.com/Netflix/ribbon
4. Spring Cloud服务发现指南