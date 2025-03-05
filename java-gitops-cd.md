# Day 128: Java云原生高级 - GitOps与持续部署

## 1. GitOps概述

GitOps是一种进行Kubernetes集群管理和应用程序交付的实践方式，它使用Git作为声明式基础设施和应用程序的单一事实来源。GitOps的核心理念是将Git仓库作为系统期望状态的定义源，并通过自动化工具确保实际系统状态与Git中定义的期望状态保持一致。

## 2. GitOps核心概念

### 2.1 声明式基础设施

```yaml
# Kubernetes部署清单示例
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: my-registry/order-service:1.2.3
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: order-service-config
              key: db.host
        resources:
          limits:
            cpu: "1"
            memory: "1Gi"
          requests:
            cpu: "500m"
            memory: "512Mi"
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 15
```

### 2.2 GitOps工具链

```java
// Java应用程序配置，支持GitOps流程
@Configuration
public class ApplicationConfig {
    
    @Bean
    @ConfigurationProperties(prefix = "app")
    public ApplicationProperties applicationProperties() {
        return new ApplicationProperties();
    }
    
    @Bean
    public VersionInfoContributor versionInfoContributor() {
        return new VersionInfoContributor();
    }
}

@Component
public class VersionInfoContributor implements InfoContributor {
    
    @Value("${git.commit.id:unknown}")
    private String commitId;
    
    @Value("${git.commit.time:unknown}")
    private String commitTime;
    
    @Value("${git.branch:unknown}")
    private String branch;
    
    @Override
    public void contribute(Info.Builder builder) {
        Map<String, Object> gitInfo = new HashMap<>();
        gitInfo.put("commit", commitId);
        gitInfo.put("commitTime", commitTime);
        gitInfo.put("branch", branch);
        
        builder.withDetail("git", gitInfo);
    }
}
```

## 3. GitOps实现

### 3.1 ArgoCD配置

```yaml
# ArgoCD应用定义
# application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/gitops-manifests.git
    targetRevision: HEAD
    path: apps/order-service
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

### 3.2 Helm Chart集成

```yaml
# values.yaml
replicaCount: 3

image:
  repository: my-registry/order-service
  tag: 1.2.3
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

resources:
  limits:
    cpu: 1
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi

config:
  activeProfiles: prod
  database:
    host: order-db.production.svc.cluster.local
    port: 5432
    name: orders
```

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "order-service.fullname" . }}
  labels:
    {{- include "order-service.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "order-service.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "order-service.selectorLabels" . | nindent 8 }}
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: {{ .Values.config.activeProfiles }}
            - name: DB_HOST
              value: {{ .Values.config.database.host }}
            - name: DB_PORT
              value: "{{ .Values.config.database.port }}"
            - name: DB_NAME
              value: {{ .Values.config.database.name }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

## 4. 持续集成与持续部署

### 4.1 Jenkins Pipeline

```groovy
// Jenkinsfile
pipeline {
    agent {
        kubernetes {
            yaml """
            apiVersion: v1
            kind: Pod
            spec:
              containers:
              - name: maven
                image: maven:3.8-openjdk-11
                command:
                - cat
                tty: true
              - name: docker
                image: docker:20.10
                command:
                - cat
                tty: true
                volumeMounts:
                - mountPath: /var/run/docker.sock
                  name: docker-sock
              volumes:
              - name: docker-sock
                hostPath:
                  path: /var/run/docker.sock
            """
        }
    }
    
    environment {
        DOCKER_REGISTRY = "my-registry"
        IMAGE_NAME = "order-service"
        GITOPS_REPO = "git@github.com:myorg/gitops-manifests.git"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build & Test') {
            steps {
                container('maven') {
                    sh 'mvn clean verify'
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                container('docker') {
                    sh "docker build -t ${DOCKER_REGISTRY}/${IMAGE_NAME}:${env.BUILD_NUMBER} ."
                    sh "docker tag ${DOCKER_REGISTRY}/${IMAGE_NAME}:${env.BUILD_NUMBER} ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest"
                }
            }
        }
        
        stage('Push Docker Image') {
            steps {
                container('docker') {
                    withCredentials([string(credentialsId: 'docker-registry-token', variable: 'DOCKER_TOKEN')]) {
                        sh "echo ${DOCKER_TOKEN} | docker login ${DOCKER_REGISTRY} -u jenkins --password-stdin"
                        sh "docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:${env.BUILD_NUMBER}"
                        sh "docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest"
                    }
                }
            }
        }
        
        stage('Update GitOps Repository') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'gitops-ssh-key', keyFileVariable: 'SSH_KEY')]) {
                    sh """
                        git config --global user.email "jenkins@example.com"
                        git config --global user.name "Jenkins"
                        
                        mkdir -p ~/.ssh
                        ssh-keyscan github.com >> ~/.ssh/known_hosts
                        cp ${SSH_KEY} ~/.ssh/id_rsa
                        chmod 600 ~/.ssh/id_rsa
                        
                        git clone ${GITOPS_REPO} gitops
                        cd gitops/apps/order-service
                        
                        # Update image tag in values.yaml
                        sed -i 's|tag: .*|tag: ${env.BUILD_NUMBER}|' values.yaml
                        
                        git add values.yaml
                        git commit -m "Update order-service image to ${env.BUILD_NUMBER}"
                        git push origin main
                    """
                }
            }
        }
    }
    
    post {
        always {
            junit '**/target/surefire-reports/*.xml'
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
```

### 4.2 GitHub Actions

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v2
      with:
        fetch-depth: 0
    
    - name: Set up JDK 11
      uses: actions/setup-java@v2
      with:
        java-version: '11'
        distribution: 'adopt'
        cache: maven
    
    - name: Build with Maven
      run: mvn -B package --file pom.xml
    
    - name: Run tests
      run: mvn test
    
    - name: Build and push Docker image
      uses: docker/build-push-action@v2
      with:
        context: .
        push: ${{ github.event_name != 'pull_request' }}
        tags: |
          my-registry/order-service:${{ github.sha }}
          my-registry/order-service:latest
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}
    
    - name: Update GitOps repository
      if: github.event_name != 'pull_request'
      uses: actions/checkout@v2
      with:
        repository: myorg/gitops-manifests
        token: ${{ secrets.GITOPS_PAT }}
        path: gitops
    
    - name: Update image tag in GitOps repo
      if: github.event_name != 'pull_request'
      run: |
        cd gitops/apps/order-service
        sed -i 's|tag: .*|tag: ${{ github.sha }}|' values.yaml
        git config --global user.email "github-actions@github.com"
        git config --global user.name "GitHub Actions"
        git add values.yaml
        git commit -m "Update order-service image to ${{ github.sha }}"
        git push
```

## 5. GitOps最佳实践

### 5.1 环境分离

```yaml
# 多环境配置示例
# environments/dev/values.yaml
environment: dev
replicaCount: 1
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 200m
    memory: 256Mi
config:
  activeProfiles: dev
  database:
    host: order-db-dev.svc.cluster.local

# environments/staging/values.yaml
environment: staging
replicaCount: 2
resources:
  limits:
    cpu: 800m
    memory: 768Mi
  requests:
    cpu: 400m
    memory: 384Mi
config:
  activeProfiles: staging
  database:
    host: order-db-staging.svc.cluster.local

# environments/production/values.yaml
environment: production
replicaCount: 3
resources:
  limits:
    cpu: 1
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi
config:
  activeProfiles: prod
  database:
    host: order-db-production.svc.cluster.local
```

### 5.2 密钥管理

```yaml
# 使用Sealed Secrets加密敏感信息
# sealed-secret.yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: order-service-secrets
  namespace: production
spec:
  encryptedData:
    DB_PASSWORD: AgBy8hCL8rQGZ4Wqs4YbVTKu9XnwH7CoLIJnZzMEtpcD7EKj...
    API_KEY: AgCtr9GM1F8yqL9tGPKT9Fq5ce6Dv+...
  template:
    metadata:
      name: order-service-secrets
      namespace: production
    type: Opaque
```

```java
@Configuration
p