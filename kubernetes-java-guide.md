# Day 63: Java容器化学习笔记 - Kubernetes入门实践

## 引言

在微服务架构盛行的今天，容器化技术已经成为现代应用部署的标配。作为Java开发者，掌握Kubernetes（K8s）不仅能帮助我们更好地管理容器化应用，还能提升在云原生领域的技术深度。让我们通过这篇文章，系统地了解K8s的基础知识，并动手实践部署一个Java应用。

## Kubernetes核心概念

### 什么是Kubernetes？

Kubernetes是一个开源的容器编排平台，它可以自动化地部署、扩展和管理容器化应用程序。简单来说，它就像一个大规模的"容器管理系统"，可以帮助我们高效地运行成百上千的容器。

### 核心组件介绍

#### Pod
Pod是Kubernetes中最基础的部署单元，它可以包含一个或多个容器。这些容器共享相同的网络空间和存储资源，可以通过localhost直接通信。在实际应用中，Pod通常只运行一个主容器，有时会搭配一些辅助容器（边车模式）。

#### Deployment
Deployment为Pod提供声明式更新能力。它负责维护Pod的期望状态，比如运行的副本数量、使用的镜像版本等。当Pod发生故障时，Deployment会自动创建新的Pod来保证服务的可用性。

#### Service
Service为一组功能相同的Pod提供统一的访问入口。它实现了负载均衡和服务发现的功能，使得前端应用无需关心后端Pod的具体部署位置。

### 架构简介

Kubernetes采用主从架构：
- Master节点：负责整个集群的管理和控制
  - API Server：集群的统一入口
  - Scheduler：负责资源调度
  - Controller Manager：维护集群状态
  - etcd：存储集群元数据
- Node节点：负责运行容器
  - kubelet：Pod管理
  - kube-proxy：网络代理
  - Container Runtime：容器运行时

## 实战演练：部署Java应用

现在让我们通过一个实际的案例来了解如何将Spring Boot应用部署到Kubernetes集群中。

### 第一步：准备Spring Boot应用

创建一个简单的Spring Boot应用：

```java
@SpringBootApplication
@RestController
public class DemoApplication {
    
    @GetMapping("/")
    public String hello() {
        // 返回一个简单的问候信息
        return "Hello from Kubernetes! 🚀";
    }
    
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

### 第二步：容器化应用

创建Dockerfile：

```dockerfile
# 使用官方的JDK基础镜像
FROM openjdk:11-jre-slim

# 设置工作目录
WORKDIR /app

# 复制编译好的jar包
COPY target/*.jar app.jar

# 暴露应用端口
EXPOSE 8080

# 启动命令
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 第三步：创建Kubernetes配置文件

deployment.yaml:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spring-boot-demo
  template:
    metadata:
      labels:
        app: spring-boot-demo
    spec:
      containers:
      - name: spring-boot-demo
        image: spring-boot-demo:1.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
```

service.yaml:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: spring-boot-service
spec:
  type: NodePort
  selector:
    app: spring-boot-demo
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30080
```

### 第四步：部署流程

1. 构建应用：
```bash
# 编译Java应用
mvn clean package

# 构建Docker镜像
docker build -t spring-boot-demo:1.0 .
```

2. 部署到Kubernetes：
```bash
# 创建deployment
kubectl apply -f deployment.yaml

# 创建service
kubectl apply -f service.yaml
```

3. 验证部署：
```bash
# 查看Pod状态
kubectl get pods

# 查看Service状态
kubectl get services
```

## 常用运维命令

### 查看应用状态
```bash
# 查看所有资源
kubectl get all

# 查看Pod详细信息
kubectl describe pod <pod-name>

# 查看应用日志
kubectl logs <pod-name>
```

### 排查问题
```bash
# 进入Pod内部
kubectl exec -it <pod-name> -- /bin/bash

# 查看Pod事件
kubectl get events

# 查看资源使用情况
kubectl top pods
```

### 伸缩应用
```bash
# 手动扩展副本数
kubectl scale deployment spring-boot-demo --replicas=5

# 自动伸缩配置
kubectl autoscale deployment spring-boot-demo --min=2 --max=5 --cpu-percent=80
```

## 最佳实践建议

1. 资源管理
   - 合理设置资源请求和限制
   - 使用命名空间隔离不同环境
   - 定期清理未使用的资源

2. 高可用配置
   - 配置健康检查和就绪探针
   - 设置合适的副本数量
   - 使用滚动更新策略

3. 监控告警
   - 部署监控组件（如Prometheus）
   - 配置关键指标告警
   - 收集应用日志

## 常见问题与解决方案

1. **Pod一直处于Pending状态**
   - 检查资源配额
   - 验证节点资源是否充足
   - 查看调度器日志

2. **容器启动失败**
   - 检查镜像是否正确
   - 查看容器日志
   - 验证配置文件正确性

3. **服务无法访问**
   - 确认Service配置正确
   - 检查标签选择器
   - 验证端口映射

## 总结

通过这篇文章，我们学习了Kubernetes的基础概念，并实践了如何将Spring Boot应用部署到Kubernetes集群中。这只是Kubernetes的入门内容，在实际工作中还需要深入学习：

- 配置管理（ConfigMap和Secret）
- 持久化存储
- 服务网格
- CI/CD集成
- 安全性配置

建议读者在本地环境（如Minikube或Docker Desktop的Kubernetes）上反复练习这些操作，以加深对Kubernetes的理解。

## 参考资源

- [Kubernetes官方文档](https://kubernetes.io/docs/home/)
- [Spring Boot官方文档](https://spring.io/projects/spring-boot)
- [Docker官方文档](https://docs.docker.com/)

记住："实践出真知"。只有在实际操作中遇到并解决问题，才能真正掌握Kubernetes的精髓。

---
作者：您的名字
日期：2025年1月3日
版本：1.0
