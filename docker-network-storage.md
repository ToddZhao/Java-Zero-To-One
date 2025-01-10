# Docker 网络与存储详解

## 1. Docker 网络基础

### 1.1 网络类型概述

Docker 提供了多种网络驱动来满足不同的需求：

- bridge：默认网络驱动，适用于单机容器通信
- host：容器直接使用主机网络
- overlay：用于 Swarm 集群中的容器通信
- macvlan：容器可以直接使用物理网络
- none：禁用容器的所有网络功能

### 1.2 网络命令示例

```bash
# 列出所有网络
docker network ls

# 创建自定义网络
docker network create --driver bridge my-network

# 查看网络详情
docker network inspect my-network

# 删除网络
docker network rm my-network
```

## 2. Java 应用网络配置

### 2.1 Spring Boot 应用示例

```java
@SpringBootApplication
public class NetworkDemoApplication {
    @Value("${db.host:localhost}")
    private String dbHost;
    
    @Bean
    public DataSource dataSource() {
        return DataSourceBuilder.create()
            .url("jdbc:mysql://" + dbHost + ":3306/mydb")
            .username("user")
            .password("password")
            .build();
    }
}
```

### 2.2 Docker Compose 网络配置

```yaml
version: '3.8'
services:
  app:
    build: .
    networks:
      - backend
    environment:
      - DB_HOST=mysql
    depends_on:
      - mysql
  
  mysql:
    image: mysql:8.0
    networks:
      - backend
    environment:
      - MYSQL_ROOT_PASSWORD=root
      - MYSQL_DATABASE=mydb

networks:
  backend:
    driver: bridge
```

## 3. Docker 存储

### 3.1 存储类型

1. **数据卷（Volumes）**：
   - Docker 管理的持久化存储
   - 独立于容器生命周期
   - 可以在容器间共享

2. **绑定挂载（Bind Mounts）**：
   - 直接将主机目录挂载到容器
   - 适合开发环境
   - 对主机文件系统有依赖

3. **tmpfs 挂载**：
   - 临时文件系统
   - 数据存储在内存中
   - 容器停止后数据丢失

### 3.2 数据卷操作

```bash
# 创建数据卷
docker volume create my-data

# 列出数据卷
docker volume ls

# 查看数据卷详情
docker volume inspect my-data

# 删除数据卷
docker volume rm my-data

# 清理未使用的数据卷
docker volume prune
```

## 4. Java 应用存储配置

### 4.1 应用配置示例

```java
@Configuration
public class StorageConfig {
    @Value("${storage.path:/data}")
    private String storagePath;
    
    @Bean
    public FileStorage fileStorage() {
        return new FileStorage(storagePath);
    }
}

public class FileStorage {
    private final Path rootLocation;
    
    public FileStorage(String path) {
        this.rootLocation = Paths.get(path);
    }
    
    public void store(MultipartFile file) {
        try {
            Files.copy(file.getInputStream(), 
                      this.rootLocation.resolve(file.getOriginalFilename()),
                      StandardCopyOption.REPLACE_EXISTING);
        } catch (IOException e) {
            throw new StorageException("Failed to store file", e);
        }
    }
}
```

### 4.2 Docker 配置

```dockerfile
# Dockerfile
FROM openjdk:17-jdk-slim
VOLUME /data
WORKDIR /app
COPY target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 4.3 Docker Compose 存储配置

```yaml
version: '3.8'
services:
  app:
    build: .
    volumes:
      - app-data:/data
      - ./logs:/app/logs
    environment:
      - STORAGE_PATH=/data
  
  mysql:
    image: mysql:8.0
    volumes:
      - mysql-data:/var/lib/mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql

volumes:
  app-data:
  mysql-data:
```

## 5. 实践案例：文件上传服务

### 5.1 Java 实现

```java
@RestController
@RequestMapping("/api/files")
public class FileController {
    private final FileStorage fileStorage;
    
    @PostMapping("/upload")
    public ResponseEntity<String> uploadFile(@RequestParam("file") MultipartFile file) {
        fileStorage.store(file);
        return ResponseEntity.ok("File uploaded successfully");
    }
    
    @GetMapping("/{filename}")
    public ResponseEntity<Resource> getFile(@PathVariable String filename) {
        Resource file = fileStorage.loadAsResource(filename);
        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, 
                   "attachment; filename=\"" + file.getFilename() + "\"")
            .body(file);
    }
}
```

### 5.2 存储配置

```yaml
version: '3.8'
services:
  app:
    build: .
    volumes:
      - uploaded-files:/app/uploads
    environment:
      - STORAGE_PATH=/app/uploads
    ports:
      - "8080:8080"

volumes:
  uploaded-files:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /path/to/host/uploads
```

## 6. 网络和存储的最佳实践

### 6.1 网络最佳实践

1. **网络隔离**：
   - 使用自定义网络而不是默认桥接网络
   - 为不同的应用组创建独立网络
   - 只暴露必要的端口

```yaml
version: '3.8'
services:
  app:
    networks:
      backend:
      frontend:
  
  mysql:
    networks:
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true
```

2. **服务发现**：
   - 使用 Docker 内置的 DNS 服务
   - 通过服务名称而不是 IP 地址通信
   - 配置健康检查

### 6.2 存储最佳实践

1. **数据持久化**：
   - 使用命名卷而不是匿名卷
   - 定期备份重要数据
   - 使用卷驱动程序进行备份

```bash
# 备份数据卷
docker run --rm -v my-volume:/source -v $(pwd):/backup alpine \
    tar czf /backup/my-volume-backup.tar.gz -C /source .

# 恢复数据卷
docker run --rm -v my-volume:/target -v $(pwd):/backup alpine \
    tar xzf /backup/my-volume-backup.tar.gz -C /target
```

2. **性能优化**：
   - 使用 tmpfs 存储临时数据
   - 合理配置数据卷的驱动选项
   - 监控存储使用情况

## 7. 监控和故障排除

### 7.1 网络故障排除

```bash
# 检查容器网络连接
docker network inspect bridge

# 进入容器测试网络
docker exec -it container-name ping other-container

# 查看容器网络统计信息
docker stats container-name
```

### 7.2 存储故障排除

```bash
# 检查数据卷使用情况
docker system df -v

# 查看容器存储详情
docker inspect container-name

# 检查数据卷权限
docker run --rm -v my-volume:/vol alpine ls -l /vol
```

## 8. 总结

Docker 的网络和存储功能为 Java 应用提供了灵活的配置选项：

1. 网络配置允许：
   - 容器间安全通信
   - 服务隔离
   - 负载均衡
   - 服务发现

2. 存储配置支持：
   - 数据持久化
   - 容器间数据共享
   - 备份和恢复
   - 性能优化

在实际应用中，应根据具体需求选择合适的网络和存储配置，并注意遵循最佳实践。
