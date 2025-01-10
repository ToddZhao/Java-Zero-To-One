# Java 应用容器化 - Docker 基础指南

## 1. Docker 基础概念

Docker 是一个开源的容器化平台，它可以将应用程序和其依赖打包到一个可移植的容器中，实现一次构建、到处运行。主要概念包括：

- **镜像（Image）**：应用程序的模板，包含代码、运行时环境、依赖等
- **容器（Container）**：镜像的运行实例
- **Dockerfile**：用于构建镜像的脚本文件
- **仓库（Registry）**：存储和分发 Docker 镜像的服务

## 2. 安装与配置

### 2.1 安装 Docker

不同操作系统的安装方式：

```bash
# Ubuntu
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io

# CentOS
sudo yum install docker-ce docker-ce-cli containerd.io

# macOS 和 Windows
# 下载并安装 Docker Desktop
```

### 2.2 配置 Docker

启动 Docker 服务并设置开机自启：

```bash
# 启动 Docker
sudo systemctl start docker

# 设置开机自启
sudo systemctl enable docker

# 验证安装
docker --version
docker run hello-world
```

## 3. Java 应用容器化

### 3.1 准备示例应用

创建一个简单的 Spring Boot 应用：

```java
@SpringBootApplication
public class DockerDemoApplication {
    
    @RestController
    @RequestMapping("/api")
    public class HelloController {
        
        @GetMapping("/hello")
        public String hello() {
            return "Hello from Docker!";
        }
    }
    
    public static void main(String[] args) {
        SpringApplication.run(DockerDemoApplication.class, args);
    }
}
```

### 3.2 创建 Dockerfile

```dockerfile
# 使用官方的 Java 运行时作为基础镜像
FROM openjdk:17-jdk-slim

# 设置工作目录
WORKDIR /app

# 复制 jar 文件到容器中
COPY target/docker-demo.jar app.jar

# 暴露应用端口
EXPOSE 8080

# 启动命令
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 3.3 构建和运行容器

```bash
# 构建镜像
docker build -t docker-demo:1.0 .

# 运行容器
docker run -p 8080:8080 docker-demo:1.0
```

## 4. Docker 常用命令

### 4.1 镜像管理

```bash
# 列出本地镜像
docker images

# 拉取镜像
docker pull openjdk:17-jdk-slim

# 删除镜像
docker rmi image-id

# 构建镜像
docker build -t app-name:tag .
```

### 4.2 容器管理

```bash
# 列出运行中的容器
docker ps

# 列出所有容器（包括停止的）
docker ps -a

# 启动容器
docker start container-id

# 停止容器
docker stop container-id

# 删除容器
docker rm container-id
```

## 5. 高级配置

### 5.1 多阶段构建

使用多阶段构建可以显著减小最终镜像的大小：

```dockerfile
# 构建阶段
FROM maven:3.8.4-openjdk-17 AS build
WORKDIR /app
COPY pom.xml .
COPY src src
RUN mvn clean package -DskipTests

# 运行阶段
FROM openjdk:17-jdk-slim
WORKDIR /app
COPY --from=build /app/target/docker-demo.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 5.2 环境变量和配置

```dockerfile
# 设置环境变量
ENV JAVA_OPTS=""
ENV SPRING_PROFILES_ACTIVE="prod"

# 使用环境变量
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

运行时传递环境变量：

```bash
docker run -e SPRING_PROFILES_ACTIVE=dev -p 8080:8080 docker-demo:1.0
```

### 5.3 数据持久化

使用数据卷持久化数据：

```bash
# 创建数据卷
docker volume create myapp-data

# 使用数据卷运行容器
docker run -v myapp-data:/app/data -p 8080:8080 docker-demo:1.0
```

## 6. Docker Compose

### 6.1 创建 docker-compose.yml

```yaml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=prod
    depends_on:
      - db
  
  db:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=root
      - MYSQL_DATABASE=myapp
    volumes:
      - mysql-data:/var/lib/mysql

volumes:
  mysql-data:
```

### 6.2 运行多容器应用

```bash
# 启动所有服务
docker-compose up -d

# 查看服务状态
docker-compose ps

# 停止所有服务
docker-compose down
```

## 7. 最佳实践

1. **镜像优化**：
   - 使用多阶段构建
   - 选择合适的基础镜像
   - 合理使用缓存层

2. **安全性**：
   - 使用非 root 用户运行应用
   - 定期更新基础镜像
   - 扫描镜像漏洞

3. **资源管理**：
```bash
# 限制容器资源
docker run -m 512m --cpu-quota 50000 docker-demo:1.0
```

4. **日志管理**：
```bash
# 查看容器日志
docker logs container-id

# 限制日志大小
docker run --log-opt max-size=10m --log-opt max-file=3 docker-demo:1.0
```

## 8. 常见问题解决

1. **构建失败**：
   - 检查 Dockerfile 语法
   - 确保网络连接正常
   - 验证基础镜像是否可用

2. **容器启动失败**：
   - 检查端口冲突
   - 查看容器日志
   - 验证环境变量配置

3. **性能问题**：
   - 监控容器资源使用
   - 优化 JVM 配置
   - 使用性能分析工具

## 9. 总结

Docker 为 Java 应用提供了一个标准化的部署环境，主要优势包括：

1. 环境一致性
2. 快速部署
3. 资源隔离
4. 版本控制
5. 可移植性

在实际使用中，需要注意：

1. 合理规划镜像层次
2. 优化构建过程
3. 注意安全性配置
4. 做好监控和日志
5. 实施持续集成/持续部署
