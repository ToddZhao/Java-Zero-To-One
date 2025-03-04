# Day 88: Java持续集成与持续部署实践指南

## 1. 引言

持续集成（CI）和持续部署（CD）是现代软件开发中不可或缺的实践。本文将介绍如何在Java项目中实施CI/CD，提高开发效率和代码质量。

## 2. Jenkins配置与使用

### 2.1 Jenkins安装与基础配置

```bash
# 下载并运行Jenkins
docker run -d -p 8080:8080 -p 50000:50000 jenkins/jenkins:lts

# 获取初始管理员密码
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

### 2.2 创建Jenkins Pipeline

```groovy
pipeline {
    agent any
    
    tools {
        maven 'Maven 3.8.4'
        jdk 'JDK 11'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/your-repo/your-project.git'
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        
        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }
        
        stage('Deploy') {
            steps {
                sh './deploy.sh'
            }
        }
    }
}
```

## 3. Git工作流实践

### 3.1 Gitflow工作流

```bash
# 创建特性分支
git checkout -b feature/new-feature develop

# 完成特性开发
git add .
git commit -m "feat: 添加新特性"

# 合并到develop分支
git checkout develop
git merge --no-ff feature/new-feature
```

### 3.2 提交规范

```plaintext
# 提交信息格式
<type>(<scope>): <subject>

<body>

<footer>

# 示例
feat(user): 添加用户注册功能

- 实现用户注册表单
- 添加邮箱验证
- 集成短信验证码

Closes #123
```

## 4. 自动化测试策略

### 4.1 单元测试

```java
@SpringBootTest
public class UserServiceTest {
    @Autowired
    private UserService userService;
    
    @Test
    public void testCreateUser() {
        User user = new User("test@example.com", "password123");
        User createdUser = userService.createUser(user);
        
        assertNotNull(createdUser.getId());
        assertEquals(user.getEmail(), createdUser.getEmail());
    }
}
```

### 4.2 集成测试

```java
@SpringBootTest
@AutoConfigureMockMvc
public class UserControllerTest {
    @Autowired
    private MockMvc mockMvc;
    
    @Test
    public void testRegisterUser() throws Exception {
        String userJson = "{\"email\":\"test@example.com\",\"password\":\"password123\"}";
        
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(userJson))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.email").value("test@example.com"));
    }
}
```

## 5. 代码质量控制

### 5.1 SonarQube集成

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.sonarsource.scanner.maven</groupId>
    <artifactId>sonar-maven-plugin</artifactId>
    <version>3.9.1.2184</version>
</plugin>
```

```groovy
// Jenkinsfile中添加SonarQube分析
stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv('SonarQube') {
            sh 'mvn sonar:sonar'
        }
    }
}
```

### 5.2 代码覆盖率检查

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.7</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

## 6. 部署策略

### 6.1 蓝绿部署

```yaml
# Kubernetes部署配置
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: myapp
        image: myapp:1.0.0
        ports:
        - containerPort: 8080
```

### 6.2 金丝雀发布

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp.example.com
  http:
  - route:
    - destination:
        host: myapp-v1
      weight: 90
    - destination:
        host: myapp-v2
      weight: 10
```

## 7. 监控与告警

### 7.1 Prometheus集成

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: prometheus
  metrics:
    tags:
      application: ${spring.application.name}
```

### 7.2 Grafana仪表板

```json
{
  "dashboard": {
    "id": null,
    "title": "Java应用监控",
    "panels": [
      {
        "title": "JVM堆内存使用",
        "type": "graph",
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "jvm_memory_used_bytes{area=\"heap\"}"
          }
        ]
      }
    ]
  }
}
```

## 8. 最佳实践

1. **版本控制**
   - 使用语义化版本
   - 保持提交信息清晰明确
   - 定期合并主分支变更

2. **自动化测试**
   - 维持高测试覆盖率
   - 包含单元测试和集成测试
   - 实现端到端测试

3. **部署流程**
   - 实现自动化部署
   - 使用环境配置管理
   - 实施回滚机制

4. **监控告警**
   - 设置合理的告警阈值
   - 实现多级告警策略
   - 建立事件响应机制

## 9. 总结

本文介绍了Java项目中实施CI/CD的主要方面：
- Jenkins配置与Pipeline创建
- Git工作流程规范
- 自动化测试策略
- 代码质量控制
- 部署策略
- 监控与告警

通过这些实践，可以显著提高团队的开发效率和代码质量，实现快速、可靠的软件交付。

## 10. 练习建议

1. 搭建本地Jenkins环境
2. 编写完整的Pipeline脚本
3. 实践Gitflow工作流
4. 配置SonarQube进行代码分析
5. 实现自动化测试和部署

持续集成和持续部署是一个需要不断优化的过程，建议在实践中逐步完善和改进。