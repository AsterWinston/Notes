
# 1. Docker 在 Jenkins 中的角色

Docker 在 Jenkins CI/CD 流程中扮演三个关键角色：

```
┌────────────────────────────────────────────────────────────┐
│  Docker 在 Jenkins 中的三种使用场景                        │
├────────────────────────────────────────────────────────────┤
│  1. Agent 运行环境：在 Docker 容器中执行 Pipeline          │
│     ┌──────────────────────────────────────────────┐      │
│     │ Pipeline 在 maven:3.9 容器中运行 Maven 构建   │      │
│     └──────────────────────────────────────────────┘      │
│                                                            │
│  2. 构建产物：使用 Docker 构建应用镜像                     │
│     ┌──────────────────────────────────────────────┐      │
│     │ Dockerfile → docker build → docker push       │      │
│     └──────────────────────────────────────────────┘      │
│                                                            │
│  3. 测试环境：运行服务容器进行集成测试                      │
│     ┌──────────────────────────────────────────────┐      │
│     │ MySQL + Redis + App 容器 → 集成测试          │      │
│     └──────────────────────────────────────────────┘      │
└────────────────────────────────────────────────────────────┘
```

# 2. Docker Pipeline 插件

## 2.1 插件安装

```
必需插件：
├── Docker Pipeline Plugin（核心）
├── Docker Plugin（管理 Docker Agent）
└── docker-build-step（Docker 构建步骤）

依赖插件：
├── credentials-binding
├── pipeline-model-definition
├── workflow-basic-steps
└── docker-commons
```

## 2.2 全局配置

```yaml
# Jenkins > Manage Jenkins > Configure System > Docker
Docker:
  - Docker URL: unix:///var/run/docker.sock
  - Registry: https://registry.company.com
    - Credentials: docker-registry-creds

# Jenkins > Manage Jenkins > Tools > Docker
Docker:
  Name: docker
  Docker Host URI: unix:///var/run/docker.sock
  Credentials: (可选，用于远程 Docker)

Docker Builder:
  Name: docker-build
  Docker Host URI: unix:///var/run/docker.sock
```

# 3. 使用 Docker Agent

## 3.1 基础用法

```groovy
// 在 Docker 容器中运行整个 Pipeline
pipeline {
    agent {
        docker {
            image 'maven:3.9-eclipse-temurin-17'
            label 'docker-agent'  // 在具有该标签的节点上启动容器
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
    }
}

// 使用 Docker 的 args 自定义容器
pipeline {
    agent {
        docker {
            image 'node:20-alpine'
            args '-v /cache:/cache --network host'
            reuseNode true  // 复用同一个节点，避免额外资源
        }
    }
    stages {
        stage('Install') {
            steps {
                sh 'npm ci'
            }
        }
        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }
    }
}
```

## 3.2 Stage 级别的 Docker Agent

```groovy
// 不同阶段使用不同的容器
pipeline {
    agent none
    stages {
        stage('Backend Build') {
            agent {
                docker { image 'maven:3.9-eclipse-temurin-17' }
            }
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Frontend Build') {
            agent {
                docker { image 'node:20-alpine' }
            }
            steps {
                sh 'npm ci && npm run build'
            }
        }
        stage('Docker Image') {
            agent {
                docker { image 'docker:24-cli' }
            }
            steps {
                sh 'docker build -t myapp:latest .'
            }
        }
    }
}
```

## 3.3 Docker with Registry

```groovy
// 从私有仓库拉取镜像
pipeline {
    agent {
        docker {
            image 'registry.company.com/internal/maven:3.9'
            registryUrl 'https://registry.company.com'
            registryCredentialsId 'docker-registry-creds'
            args '-v /root/.m2:/root/.m2'
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
    }
}
```

## 3.4 Docker Pipeline Agent 参数

| 参数 | 说明 | 示例 |
|------|------|------|
| `image` | Docker 镜像名 | `'maven:3.9'` |
| `args` | Docker run 参数 | `'-v /host:/container'` |
| `label` | 在指定标签的节点上运行 | `'docker-agent'` |
| `registryUrl` | 私有仓库地址 | `'https://registry.company.com'` |
| `registryCredentialsId` | 仓库认证凭据 | `'docker-creds'` |
| `reuseNode` | 复用父级节点 | `true` |
| `alwaysPull` | 每次都拉取最新镜像 | `true` |

# 4. Docker 构建与发布

## 4.1 构建 Docker 镜像

```groovy
// 基础构建
stage('Docker Build') {
    steps {
        script {
            def myImage = docker.build("myapp:${BUILD_NUMBER}")
        }
    }
}

// 使用自定义 Dockerfile 和参数
stage('Docker Build Advanced') {
    steps {
        script {
            def myImage = docker.build(
                "registry.company.com/myapp:${BUILD_NUMBER}",
                "--build-arg JAR_FILE=target/myapp.jar -f Dockerfile.prod ."
            )
        }
    }
}

// 多标签构建
stage('Docker Build Multi Tag') {
    steps {
        script {
            def myImage = docker.build(
                "registry.company.com/myapp:${BUILD_NUMBER}"
            )
            // 添加额外标签
            myImage.push('latest')
            if (params.ENV == 'prod') {
                myImage.push('stable')
            }
        }
    }
}
```

## 4.2 Docker 登录与推送

```groovy
// 登录私有仓库并推送
stage('Docker Push') {
    steps {
        script {
            docker.withRegistry('https://registry.company.com', 'docker-registry-creds') {
                // 登录 → 推送 → 登出（自动处理）
                docker.image("registry.company.com/myapp:${BUILD_NUMBER}").push()
                docker.image("registry.company.com/myapp:${BUILD_NUMBER}").push('latest')
            }
        }
    }
}

// 多个仓库推送
stage('Multi Registry Push') {
    steps {
        script {
            // 推送到第一个仓库
            docker.withRegistry('https://registry-1.company.com', 'creds-1') {
                def img = docker.image("myapp:${BUILD_NUMBER}")
                img.push()
            }
            // 推送到第二个仓库
            docker.withRegistry('https://registry-2.company.com', 'creds-2') {
                def img = docker.image("myapp:${BUILD_NUMBER}")
                img.push()
            }
        }
    }
}
```

## 4.3 Dockerfile 最佳实践

```dockerfile
# 多阶段构建
# 第一阶段：编译
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src/ src/
RUN mvn clean package -DskipTests

# 第二阶段：运行
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=builder /build/target/*.jar app.jar
EXPOSE 8080
USER appuser
ENTRYPOINT ["java", "-jar", "app.jar"]

# 优化建议：
# 1. 使用 Alpine 基础镜像减小体积
# 2. 多阶段构建分离编译和运行环境
# 3. 使用非 root 用户运行
# 4. 分离依赖层和应用层以利用缓存
```

# 5. Docker Compose 集成

## 5.1 集成测试场景

```groovy
// 使用 Docker Compose 启动集成测试环境
stage('Integration Test') {
    steps {
        script {
            // 启动依赖服务
            sh 'docker-compose -f docker-compose.test.yml up -d db redis'

            // 等待服务就绪
            sh '''
                until curl -s http://localhost:3306 > /dev/null; do
                    echo "Waiting for MySQL..."
                    sleep 2
                done
            '''

            try {
                // 运行测试
                sh './mvnw verify -Pintegration-test'
            } finally {
                // 清理：关闭所有容器
                sh 'docker-compose -f docker-compose.test.yml down -v'
            }
        }
    }
}
```

## 5.2 Docker Compose 文件

```yaml
# docker-compose.test.yml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=test
      - DB_URL=jdbc:mysql://db:3306/testdb
      - REDIS_HOST=redis
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

  db:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=test
      - MYSQL_DATABASE=testdb
      - MYSQL_USER=test
      - MYSQL_PASSWORD=test
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 5s
      timeout: 3s
      retries: 10

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
```

# 6. Docker Agent 动态扩展

## 6.1 Docker Agent 配置

通过 Docker Plugin，Jenkins 可以动态创建 Agent 容器。

**配置路径**：Manage Jenkins > Nodes > Configure Clouds > Add a new cloud > Docker

```yaml
# Docker Cloud 配置
Docker Cloud:
  Name: docker-cloud
  Docker Host URI: unix:///var/run/docker.sock
  Enabled: true

  # Docker Agent 模板
  Docker Agent templates:
    - Name: maven-agent
      Labels: maven docker
      Enabled: true
      Image: jenkins/inbound-agent:latest-jdk17
      Remote File System Root: /home/jenkins/agent
      Usage: Use this node as much as possible
      Mode: Normal
      # 容器设置
      Volumes:
        - /var/run/docker.sock:/var/run/docker.sock  # Docker in Docker
        - /root/.m2:/root/.m2  # Maven 缓存共享
      Host networking: false
      Environment:
        - JAVA_OPTS=-Xmx1024m
      Pull strategy: Pull always
      Instance Capacity: 5  # 最大同时运行数
      Timeout (seconds): 300  # 容器创建超时
      Retention Strategy: Idle timeout (minutes: 10)
```

## 6.2 Pipeline 中使用 Docker Agent

```groovy
// Pipeline 使用 Docker Cloud 中的 Agent
pipeline {
    agent {
        label 'maven && docker'
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
    }
}
```

## 6.3 Docker Agent 生命周期

```
1. Job 触发，需要 maven 标签的 Agent
2. Docker Cloud 检查是否有空闲容器
3. 无 → 根据模板创建新容器
4. 容器启动 → 自动注册为 Agent
5. Jenkins 分配 Build 到该 Agent
6. 构建完成 → Agent 断开连接
7. 根据保留策略（空闲超时）删除容器
```

# 7. Docker in Docker（DinD）

## 7.1 场景说明

当 Pipeline 在 Docker 容器中执行时，需要在容器内再执行 Docker 命令。

```groovy
// 方式一：挂载 Docker Socket
pipeline {
    agent {
        docker {
            image 'docker:24-cli'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    stages {
        stage('Docker Ops') {
            steps {
                sh 'docker ps'  // 使用宿主机 Docker
                sh 'docker build -t myapp:latest .'
                sh 'docker push registry.company.com/myapp:latest'
            }
        }
    }
}

// 方式二：Docker in Docker（需特权模式）
pipeline {
    agent {
        docker {
            image 'docker:24-dind'
            args '--privileged -v /var/lib/docker:/var/lib/docker'
        }
    }
    stages {
        stage('Build in DinD') {
            steps {
                sh 'dockerd &'  // 启动内部 Docker 守护进程
                sh 'docker build -t myapp:latest .'
            }
        }
    }
}
```

## 7.2 Docker Socket 挂载 vs DinD

| 方式 | 安全性 | 性能 | 适用场景 |
|------|-------|------|---------|
| Docker Socket 挂载 | 低（容器可控制宿主机 Docker） | 高 | 信任的 Pipeline |
| Docker in Docker | 高（隔离的 Docker 环境） | 低 | 需要完全隔离的环境 |

# 8. Docker 缓存策略

## 8.1 层缓存优化

```dockerfile
# 优化层缓存：将变化频率低的层放在前面
FROM eclipse-temurin:17-jre-alpine

# 1. 系统依赖（几乎不变）
RUN apk add --no-cache tzdata curl

# 2. 应用层（经常变化）
COPY target/*.jar app.jar

# 3. 启动命令
ENTRYPOINT ["java", "-jar", "app.jar"]
```

## 8.2 Docker BuildKit 缓存

```groovy
stage('Build with Cache') {
    steps {
        script {
            // 使用 BuildKit 构建缓存
            sh """
                DOCKER_BUILDKIT=1 docker build \
                    --cache-from registry.company.com/myapp:latest \
                    --build-arg BUILDKIT_INLINE_CACHE=1 \
                    -t registry.company.com/myapp:${BUILD_NUMBER} \
                    -t registry.company.com/myapp:latest \
                    .
            """
        }
    }
}
```

## 8.3 挂载缓存目录

```groovy
// 挂载 Maven/NPM 缓存提升构建速度
pipeline {
    agent {
        docker {
            image 'maven:3.9-eclipse-temurin-17'
            args '''
                -v maven-repo:/root/.m2
                -v npm-cache:/root/.npm
            '''
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
    }
}
```

# 9. Docker 安全最佳实践

## 9.1 镜像安全扫描

```groovy
stage('Security Scan') {
    steps {
        script {
            // 使用 Trivy 扫描镜像漏洞
            sh """
                docker run --rm \
                    -v /var/run/docker.sock:/var/run/docker.sock \
                    aquasec/trivy:latest \
                    image --severity CRITICAL,HIGH \
                    registry.company.com/myapp:${BUILD_NUMBER}
            """
        }
    }
}
```

## 9.2 最小权限原则

```groovy
// 不要在 Pipeline 中使用特权模式除非必要
❌ agent {
    docker {
        image 'myimage'
        args '--privileged'  // 不推荐
    }
}

// ✅ 指定需要的权限
✅ agent {
    docker {
        image 'myimage'
        args '--cap-add=SYS_PTRACE'  // 仅添加需要的 capability
    }
}

// ✅ 使用非 root 用户运行
✅ agent {
    docker {
        image 'myimage'
        args '-u 1000:1000'  // 指定用户 ID
    }
}
```

## 9.3 镜像清理策略

```groovy
// 定期清理未使用的镜像和容器
stage('Docker Cleanup') {
    steps {
        sh '''
            # 删除所有已停止的容器
            docker container prune -f

            # 删除未使用的镜像
            docker image prune -af

            # 删除未使用的卷
            docker volume prune -f

            # 删除构建缓存
            docker builder prune -af
        '''
    }
}
```

# 10. 实际案例：完整的 Docker CI/CD Pipeline

```groovy
@Library('jenkins-shared-library') _

pipeline {
    agent any
    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
    }
    environment {
        REGISTRY = 'registry.company.com'
        APP_NAME = 'my-application'
        IMAGE_TAG = "${BUILD_NUMBER}-${env.BRANCH_NAME?.replaceAll('/', '-')}"
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Unit Tests') {
            agent {
                docker {
                    image 'maven:3.9-eclipse-temurin-17'
                    args '-v maven-cache:/root/.m2'
                }
            }
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Build Artifact') {
            agent {
                docker {
                    image 'maven:3.9-eclipse-temurin-17'
                    args '-v maven-cache:/root/.m2'
                }
            }
            steps {
                sh 'mvn clean package -DskipTests=true'
            }
            post {
                success {
                    archiveArtifacts 'target/*.jar'
                }
            }
        }

        stage('Docker Build') {
            agent {
                docker {
                    image 'docker:24-cli'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                script {
                    sh """
                        docker build \
                            --cache-from ${REGISTRY}/${APP_NAME}:latest \
                            -t ${REGISTRY}/${APP_NAME}:${IMAGE_TAG} \
                            -t ${REGISTRY}/${APP_NAME}:latest \
                            .
                    """
                }
            }
        }

        stage('Security Scan') {
            agent {
                docker {
                    image 'aquasec/trivy:latest'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                sh """
                    trivy image --severity CRITICAL \
                        --exit-code 1 \
                        ${REGISTRY}/${APP_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage('Push to Registry') {
            agent {
                docker {
                    image 'docker:24-cli'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                script {
                    docker.withRegistry("https://${REGISTRY}", 'docker-registry-creds') {
                        docker.image("${REGISTRY}/${APP_NAME}:${IMAGE_TAG}").push()
                        docker.image("${REGISTRY}/${APP_NAME}:${IMAGE_TAG}").push('latest')
                    }
                }
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                script {
                    sh "kubectl set image deployment/${APP_NAME} \
                        app=${REGISTRY}/${APP_NAME}:${IMAGE_TAG} -n production"
                    sh "kubectl rollout status deployment/${APP_NAME} -n production"
                }
            }
        }
    }
    post {
        always {
            sh 'docker system prune -f'
            cleanWs()
        }
        failure {
            slackSend(
                color: 'danger',
                message: "Docker build failed: ${APP_NAME} #${BUILD_NUMBER}"
            )
        }
    }
}
```
