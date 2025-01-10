# Day 64: Java容器化学习笔记 - Kubernetes部署与管理实战

## 引言

在上一篇文章中，我们学习了Kubernetes的基础概念。今天，让我们深入探讨如何在实际工作中部署和管理Java应用。我们会关注配置管理、资源调度、持久化存储等关键主题，这些都是在生产环境中必须掌握的核心技能。

## 应用配置管理

在生产环境中，应用的配置管理是一个重要话题。我们不能将配置硬编码在代码或镜像中，而是应该使用Kubernetes提供的配置管理机制。

### ConfigMap的使用

ConfigMap用于存储非敏感的配置信息。让我们看一个Spring Boot应用的配置示例：

```yaml
# application-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: spring-boot-config
data:
  application.properties: |
    server.port=8080
    spring.application.name=demo-service
    logging.level.root=INFO
    spring.datasource.url=jdbc:mysql://mysql-service:3306/demo
    spring.datasource.username=${MYSQL_USER}
```

这个ConfigMap可以被挂载到Pod中：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-app
spec:
  template:
    spec:
      containers:
      - name: spring-boot-app
        image: demo-app:1.0
        volumeMounts:
        - name: config-volume
          mountPath: /config
      volumes:
      - name: config-volume
        configMap:
          name: spring-boot-config
```

### Secret管理

对于敏感信息（如数据库密码、API密钥等），我们应该使用Secret：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-credentials
type: Opaque
data:
  username: dXNlcm5hbWU=  # base64编码的用户名
  password: cGFzc3dvcmQ=  # base64编码的密码
```

将Secret注入到Pod中：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-app
spec:
  template:
    spec:
      containers:
      - name: spring-boot-app
        env:
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: mysql-credentials
              key: username
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-credentials
              key: password
```

## 资源管理与调度

### 资源限制

合理的资源限制可以防止单个应用消耗过多资源：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-app
spec:
  template:
    spec:
      containers:
      - name: spring-boot-app
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
```

### 节点亲和性

有时我们需要将Pod调度到特定的节点上：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-app
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: disktype
                operator: In
                values:
                - ssd
```

## 持久化存储

### 创建PersistentVolume

对于需要持久化的数据，我们需要使用PersistentVolume：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/mysql
```

### 使用PersistentVolumeClaim

应用通过PVC来请求存储资源：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

将PVC挂载到Pod：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  template:
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
```

## 服务健康检查

### 配置Liveness探针

确保应用保持运行状态：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-app
spec:
  template:
    spec:
      containers:
      - name: spring-boot-app
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
```

### 配置Readiness探针

确保应用准备好接收流量：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-app
spec:
  template:
    spec:
      containers:
      - name: spring-boot-app
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 5
```

## 应用更新策略

### 滚动更新

配置应用的更新策略：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-app
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

执行更新：

```bash
# 更新镜像版本
kubectl set image deployment/spring-boot-app spring-boot-app=demo-app:2.0

# 查看更新状态
kubectl rollout status deployment/spring-boot-app
```

## 监控与日志

### 使用Prometheus监控

部署Prometheus和Grafana来监控应用：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: spring-boot-monitor
spec:
  selector:
    matchLabels:
      app: spring-boot-app
  endpoints:
  - port: metrics
```

### 日志收集

配置日志收集：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-app
spec:
  template:
    spec:
      containers:
      - name: spring-boot-app
        volumeMounts:
        - name: log-volume
          mountPath: /logs
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:7.9.3
        volumeMounts:
        - name: log-volume
          mountPath: /logs
        - name: filebeat-config
          mountPath: /usr/share/filebeat/filebeat.yml
          subPath: filebeat.yml
```

## 故障排查与调试

### 常用调试命令

```bash
# 查看Pod日志
kubectl logs <pod-name> -f

# 进入容器
kubectl exec -it <pod-name> -- /bin/bash

# 查看Pod详细信息
kubectl describe pod <pod-name>

# 端口转发
kubectl port-forward <pod-name> 8080:8080
```

### 常见问题解决方案

1. 镜像拉取失败
   - 检查镜像名称和标签
   - 验证镜像仓库访问权限
   - 配置imagePullSecrets

2. 应用启动失败
   - 检查资源限制
   - 查看应用日志
   - 验证配置正确性

3. 网络连接问题
   - 检查Service配置
   - 验证网络策略
   - 测试DNS解析

## 生产环境最佳实践

1. 高可用配置
   - 使用多副本部署
   - 配置Pod反亲和性
   - 使用PodDisruptionBudget

2. 安全性配置
   - 启用RBAC权限控制
   - 配置网络策略
   - 使用SecurityContext

3. 资源管理
   - 设置资源限制
   - 配置HPA自动扩缩容
   - 使用ResourceQuota

## 总结

本文我们深入探讨了Kubernetes在生产环境中的部署和管理实践。关键要点包括：

- 使用ConfigMap和Secret管理配置
- 合理分配和限制资源
- 配置持久化存储
- 实现健康检查和监控
- 掌握故障排查方法

记住，在实际操作中可能会遇到各种意外情况，保持耐心，善用kubectl describe和logs命令来定位问题。持续学习和实践是提升Kubernetes运维能力的关键。

## 参考资源

- [Kubernetes生产环境指南](https://kubernetes.io/docs/setup/production-environment/)
- [Spring Boot on Kubernetes](https://spring.io/guides/gs/spring-boot-kubernetes/)
- [Kubernetes最佳实践](https://kubernetes.io/docs/concepts/configuration/overview/)

---
作者：您的名字
日期：2025年1月3日
版本：1.0
