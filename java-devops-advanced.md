# Day 93: Java DevOps实践 - 高级指南

## 1. 引言

DevOps已经成为现代软件开发中不可或缺的一部分。本文将深入探讨Java项目中的DevOps实践，包括持续集成/持续部署(CI/CD)、自动化测试、监控告警等关键环节，帮助开发团队构建高效的开发运维流程。

## 2. 现代CI/CD实践

### 2.1 GitHub Actions工作流配置

```yaml
name: Java CI/CD Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up JDK 17
      uses: actions/setup-java@v3
      with:
        java-version: '17'
        distribution: 'temurin'
        cache: 'maven'
    
    - name: Build with Maven
      run: mvn -B package --file pom.xml
    
    - name: Run Tests
      run: mvn test
      
    - name: Generate Test Report
      run: mvn surefire-report:report
      
    - name: Upload Test Results
      uses: actions/upload-artifact@v3
      with:
        name: test-results
        path: target/site/surefire-report.html

  security-scan:
    needs: build-and-test
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Run OWASP Dependency Check
      run: mvn org.owasp:dependency-check-maven:check
      
    - name: Run SonarQube Analysis
      env:
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      run: mvn sonar:sonar

  deploy:
    needs: security-scan
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
    - name: Deploy to Production
      run: |
        echo "Deploying to production..."
        # 实际部署步骤
```

### 2.2 Jenkins Pipeline配置

```groovy
pipeline {
    agent any
    
    environment {
        JAVA_HOME = '/usr/lib/jvm/java-17-openjdk'
        MAVEN_HOME = '/usr/share/maven'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
        
        stage('Unit Tests') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }
        
        stage('Integration Tests') {
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }
        
        stage('Code Quality') {
            parallel {
                stage('SonarQube Analysis') {
                    steps {
                        withSonarQubeEnv('SonarQube') {
                            sh 'mvn sonar:sonar'
                        }
                    }
                }
                stage('Security Scan') {
                    steps {
                        sh 'mvn org.owasp:dependency-check-maven:check'
                    }
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("myapp:${env.BUILD_NUMBER}")
                }
            }
        }
        
        stage('Deploy to Staging') {
            steps {
                script {
                    // 部署到测试环境
                    sh "kubectl apply -f k8s/staging/"
                }
            }
        }
        
        stage('Performance Tests') {
            steps {
                sh 'jmeter -n -t performance-tests/load-test.jmx -l results.jtl'
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                script {
                    // 生产环境部署
                    sh "kubectl apply -f k8s/production/"
                }
            }
        }
    }
    
    post {
        always {
            // 清理工作空间
            cleanWs()
        }
        success {
            // 发送成功通知
            emailext subject: "Pipeline Successful",
                     body: "Build completed successfully",
                     to: "team@example.com"
        }
        failure {
            // 发送失败通知
            emailext subject: "Pipeline Failed",
                     body: "Build failed. Please check the logs",
                     to: "team@example.com"
        }
    }
}
```

## 3. 自动化测试策略

### 3.1 测试金字塔实现

```java
// 单元测试示例
@SpringBootTest
class UserServiceTest {
    @MockBean
    private UserRepository userRepository;
    
    @Autowired
    private UserService userService;
    
    @Test
    void testCreateUser() {
        // 准备测试数据
        User user = new User("test", "test@example.com");
        when(userRepository.save(any(User.class))).thenReturn(user);
        
        // 执行测试
        User result = userService.createUser(user);
        
        // 验证结果
        assertThat(result).isNotNull();
        assertThat(result.getUsername()).isEqualTo("test");
        verify(userRepository).save(any(User.class));
    }
}

// 集成测试示例
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@TestContainers
class UserControllerIntegrationTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:14")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @LocalServerPort
    private int port;
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Test
    void testCreateUserEndpoint() {
        // 准备测试数据
        UserDTO userDTO = new UserDTO("test", "test@example.com");
        
        // 执行测试
        ResponseEntity<User> response = restTemplate.postForEntity(
            "http://localhost:" + port + "/api/users",
            userDTO,
            User.class
        );
        
        // 验证结果
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody()).isNotNull();
        assertThat(response.getBody().getUsername()).isEqualTo("test");
    }
}
```

### 3.2 性能测试配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jmeterTestPlan version="1.2" properties="5.0">
  <hashTree>
    <TestPlan guiclass="TestPlanGui" testclass="TestPlan" testname="Web API Load Test">
      <elementProp name="TestPlan.user_defined_variables" elementType="Arguments">
        <collectionProp name="Arguments.arguments"/>
      </elementProp>
      <stringProp name="TestPlan.comments"></stringProp>
      <boolProp name="TestPlan.functional_mode">false</boolProp>
      <boolProp name="TestPlan.serialize_threadgroups">false</boolProp>
      <stringProp name="TestPlan.user_define_classpath"></stringProp>
    </TestPlan>
    <hashTree>
      <ThreadGroup guiclass="ThreadGroupGui" testclass="ThreadGroup" testname="API Users">
        <elementProp name="ThreadGroup.main_controller" elementType="LoopController">
          <boolProp name="LoopController.continue_forever">false</boolProp>
          <intProp name="LoopController.loops">100</intProp>
        </elementProp>
        <stringProp name="ThreadGroup.num_threads">50</stringProp>
        <stringProp name="ThreadGroup.ramp_time">10</stringProp>
        <longProp name="ThreadGroup.start_time">1373789594000</longProp>
        <longProp name="ThreadGroup.end_time">1373789594000</longProp>
        <boolProp name="ThreadGroup.scheduler">true</boolProp>
        <stringProp name="ThreadGroup.duration">300</stringProp>
        <stringProp name="ThreadGroup.delay">0</stringProp>
      </ThreadGroup>
      <hashTree>
        <HTTPSamplerProxy guiclass="HttpTestSampleGui" testclass="HTTPSamplerProxy">
          <elementProp name="HTTPsampler.Arguments" elementType="Arguments">
            <collectionProp name="Arguments.arguments"/>
          </elementProp>
          <stringProp name="HTTPSampler.domain">api.example.com</stringProp>
          <stringProp name="HTTPSampler.port">443</stringProp>
          <stringProp name="HTTPSampler.protocol">https</stringProp>
          <stringProp name="HTTPSampler.path">/api/users</stringProp>
          <stringProp name="HTTPSampler.method">GET</stringProp>
          <boolProp name="HTTPSampler.follow_redirects">true</boolProp>
          <boolProp name="HTTPSampler.auto_redirects">false</boolProp>
          <boolProp name="HTTPSampler.use_keepalive">true</boolProp>
          <boolProp name="HTTPSampler.DO_MULTIPART_POST">false</boolProp>
        </HTTPSamplerProxy>
        <hashTree/>
      </hashTree>
    </hashTree>
  </hashTree>
</jmeterTestPlan>
```

## 4. 监控与可观测性

### 4.1 Prometheus监控配置

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'spring-boot-app'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['localhost:8080']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['localhost:9100']

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['localhost:9093']

rule_files:
  - 'alert_rules.yml'
```

### 4.2 Grafana仪表板配置

```json
{
  "dashboard": {
    "id": null,
    "title": "Java Application Dashboard",
    "tags": ["java", "spring-boot"],
    "timezone": "browser",
    "panels": [
      {
        "title": "JVM Memory Usage",
        "type": "graph",
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "jvm_memory_used_bytes{area=\"heap\"}",
            "legendFormat": "{{instance}} - Heap Used"
          },
          {
            "expr": "jvm_memory_max_bytes{area=\"heap\"}",
            "legendFormat": "{{instance}} - Heap Max"
          }
        ]
      },
      {
        "title": "HTTP Request Rate",
        "type": "graph",
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "rate(http_server_requests_seconds_count[5m])",
            "legendFormat": "{{method}} {{status}}"
          }
        ]
      }
    ]
  }
}
```

## 5. 日志管理与分析

### 5.1 ELK Stack配置

```yaml
# Logstash配置
input {
  beats {
    port => 5044
  }
}

filter {
  if [type] == "java-app-logs" {
    grok {
      match => { "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} %{GREEDYDATA:logger} - %{GREEDYDATA:message}" }
    }
    date {
      match => [ "timestamp", "ISO8601" ]
      target => "@timestamp"
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "java-app-logs-%{+YYYY.MM.dd}"
  }
}
```

### 5.2 应用日志配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
    </appender>
    
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/application.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.