
# 1. Pipeline 设计原则

## 1.1 单一职责原则

```groovy
// ✅ 每个 Pipeline 只负责一个应用/服务
// ❌ 不要在一个 Pipeline 中构建多个无关项目

// ✅ 单一职责示例
pipeline {
    agent any
    stages {
        stage('Build') { /* 构建 */ }
        stage('Test') { /* 测试 */ }
        stage('Deploy to Dev') { /* 部署到开发环境 */ }
        // 没有部署到生产环境——那是 CD Pipeline 的职责
    }
}
```

## 1.2 Pipeline as Code

```yaml
# ✅ 必须做
- Jenkinsfile 存储在 Git 仓库根目录
- 所有配置通过 Pull Request 审查
- 分支使用不同 Pipeline（Multibranch Pipeline）

# ❌ 避免
- 在 Jenkins UI 中手动配置 Pipeline
- 在 Jenkinsfile 中硬编码环境特定值
- 缺少版本控制
```

## 1.3 幂等性

```groovy
// ✅ 幂等构建：多次执行结果一致
stage('Deploy') {
    steps {
        script {
            // 先检查是否已部署
            def deployed = sh(
                script: "kubectl get deployment myapp -n ${params.ENV} --ignore-not-found",
                returnStdout: true
            ).trim()

            if (!deployed) {
                sh "kubectl create deployment myapp --image=myapp:${BUILD_NUMBER}"
            } else {
                sh "kubectl set image deployment/myapp app=myapp:${BUILD_NUMBER}"
            }
        }
    }
}
```

# 2. Jenkinsfile 最佳实践

## 2.1 保持简洁

```groovy
// ✅ 简洁的 Jenkinsfile
@Library('shared-lib') _

pipeline {
    agent { label 'linux' }
    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }
    stages {
        stage('Build') { steps { buildJava() } }
        stage('Test') { steps { testJava() } }
        stage('Deploy') { steps { deploy() } }
    }
    post {
        failure { notifyFailure() }
        success { notifySuccess() }
    }
}
```

## 2.2 避免深层嵌套

```groovy
// ❌ 过深嵌套
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                script {
                    if (condition) {
                        try {
                            sh '...'
                        } catch (e) {
                            if (retry < 3) {
                                // ...
                            }
                        }
                    }
                }
            }
        }
    }
}

// ✅ 提取为共享库
// vars/buildWithRetry.groovy
def call(Closure body) {
    retry(3) {
        try {
            body()
        } catch (Exception e) {
            echo "Build failed: ${e.message}"
            throw e
        }
    }
}
```

## 2.3 环境变量管理

```groovy
// ✅ 使用 environment 块管理环境变量
pipeline {
    agent any
    environment {
        JAVA_HOME = tool name: 'JDK17', type: 'jdk'
        MAVEN_HOME = tool name: 'Maven3', type: 'maven'
        CI = 'true'
    }
    stages { /* ... */ }
}

// ✅ 使用配置文件（外部化配置）
// vars/config.groovy
def call(String env) {
    readYaml file: "config/${env}.yaml"
}
```

# 3. 安全性实践

## 3.1 凭据管理

```groovy
// ✅ 正确做法
withCredentials([usernamePassword(...)]) {
    sh '...'
}

// ✅ 隐蔽密钥输出
sh script: 'deploy.sh', label: 'Deploy application'

// ❌ 错误做法
// sh 'curl -u admin:password123 ...'         // 硬编码
// env.PASSWORD = 'secret'                     // 全局暴露
// echo "Password: ${env.PASSWORD}"            // 日志泄露
```

## 3.2 最小权限原则

```yaml
# ✅ 权限配置
授权策略: Role-Based Strategy
全局角色:
  admin: 完全访问
  developer: Job/Build, Job/Read
  viewer: Overall/Read, Job/Read
  anonymous: (无权限)

# ❌ 不要
# 任何人可做任何事
# 匿名用户有管理员权限
```

## 3.3 网络隔离

```yaml
# ✅ 生产安全配置
- Jenkins Master 不直接暴露在公网
- 使用反向代理（Nginx）提供 HTTPS
- Agent 通过 WebSocket 连接（不开放 50000 端口）
- 配置 IP 白名单
- 启用 CSRF 保护
```

# 4. 性能优化

## 4.1 构建加速

```groovy
// 1. 浅克隆
checkout([$class: 'GitSCM',
    extensions: [[$class: 'CloneOption', depth: 1, shallow: true]]
])

// 2. 依赖缓存
agent {
    docker {
        image 'maven:3.9-eclipse-temurin-17'
        args '-v maven-repo:/root/.m2'
    }
}

// 3. 并行构建
stage('Tests') {
    parallel {
        stage('Unit Test') { steps { sh 'mvn test' } }
        stage('Integration Test') { steps { sh 'mvn verify' } }
    }
}

// 4. 增量构建
stage('Build') {
    steps {
        script {
            if (isChanged('src/')) {
                sh 'mvn compile'
            } else {
                echo 'No source changes, skipping build'
            }
        }
    }
}
```

## 4.2 Master 优化

```yaml
# ✅ Master 配置建议
- 将 Master 的 Executor 数设为 0
- 使用外部 Agent 执行所有构建
- 启用构建丢弃策略
- 设置 GC 参数（使用 G1GC）
- 限制日志保留大小
- 定期清理工作空间
```

## 4.3 缓存策略

```groovy
// Pipeline 缓存策略
pipeline {
    agent any
    stages {
        stage('Restore Cache') {
            steps {
                script {
                    // 从缓存恢复 Maven 依赖
                    sh 'cp -r /cache/m2/repository ~/.m2/ || true'
                }
            }
        }
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Save Cache') {
            steps {
                script {
                    // 保存 Maven 依赖到缓存
                    sh 'cp -r ~/.m2/repository /cache/m2/ || true'
                }
            }
        }
    }
}
```

# 5. Pipeline 设计模式

## 5.1 模板模式

```groovy
// vars/javaLibraryPipeline.groovy
def call(Map config) {
    pipeline {
        agent { label config.agent ?: 'linux' }
        options {
            timeout(time: config.timeout ?: 30, unit: 'MINUTES')
            buildDiscarder(logRotator(numToKeepStr: '20'))
        }
        stages {
            stage('Checkout') { steps { checkout scm } }
            stage('Build') { steps { sh 'mvn clean compile' } }
            stage('Test') { steps {
                sh 'mvn test'
                junit 'target/surefire-reports/*.xml'
            }}
            stage('Package') { steps { sh 'mvn package -DskipTests' } }
        }
        post {
            failure { notifyFailure() }
            success { notifySuccess() }
        }
    }
}
```

## 5.2 分支策略模式

```groovy
pipeline {
    agent any
    stages {
        stage('Determine Strategy') {
            steps {
                script {
                    switch(env.BRANCH_NAME) {
                        case 'main':
                            env.DEPLOY_ENV = 'prod'
                            env.RUN_FULL_TESTS = 'true'
                            break
                        case 'develop':
                            env.DEPLOY_ENV = 'dev'
                            env.RUN_FULL_TESTS = 'false'
                            break
                        case ~/feature\/.*/:
                            env.DEPLOY_ENV = ''
                            env.RUN_FULL_TESTS = 'false'
                            break
                        case ~/release\/.*/:
                            env.DEPLOY_ENV = 'staging'
                            env.RUN_FULL_TESTS = 'true'
                            break
                    }
                }
            }
        }
    }
}
```

# 6. 日志和调试

## 6.1 日志规范

```groovy
// ✅ 有意义的日志
stage('Build') {
    steps {
        sh 'mvn clean package'
    }
    post {
        always {
            echo "[BUILD] Build completed with status: ${currentBuild.result}"
            echo "[BUILD] Duration: ${currentBuild.duration}ms"
            echo "[BUILD] Node: ${env.NODE_NAME}"
        }
    }
}
```

## 6.2 调试技术

```groovy
// 条件调试
def DEBUG = params.DEBUG ?: false

stage('Build') {
    steps {
        script {
            if (DEBUG) {
                echo "=== DEBUG ==="
                echo "BRANCH: ${env.BRANCH_NAME}"
                echo "WORKSPACE: ${env.WORKSPACE}"
                sh 'env | sort'
                sh 'ls -la'
            }
        }
        sh 'mvn clean package'
    }
}
```

# 7. 多环境管理

```yaml
# 环境配置示例（外部化）
environments:
  dev:
    url: https://dev.api.company.com
    replicas: 1
    resources: {cpu: "0.5", memory: "512Mi"}

  staging:
    url: https://staging.api.company.com
    replicas: 2
    resources: {cpu: "1", memory: "1Gi"}

  prod:
    url: https://api.company.com
    replicas: 5
    resources: {cpu: "2", memory: "2Gi"}
```

```groovy
// Pipeline 中使用环境配置
stage('Deploy') {
    steps {
        script {
            def config = readYaml file: 'environments.yaml'
            def envConfig = config.environments[params.ENV]

            sh """
                helm upgrade --install myapp ./chart \\
                    --namespace ${params.ENV} \\
                    --set replicaCount=${envConfig.replicas} \\
                    --set resources.requests.cpu=${envConfig.resources.cpu}
            """
        }
    }
}
```

# 8. 持续改进

## 8.1 Pipeline 审查清单

```yaml
# Pipeline 审查清单
审查项:
  - 安全性:
    - 凭据是否使用 withCredentials
    - 是否有硬编码密码
    - 是否启用 CSRF 保护
  - 性能:
    - 是否使用浅克隆
    - 是否使用缓存
    - 是否有并行 stage
    - 是否有构建丢弃策略
  - 可靠性:
    - 是否有超时设置
    - 是否有重试机制
    - 是否有幂等性检查
  - 可维护性:
    - Jenkinsfile 是否简洁
    - 是否使用共享库
    - 是否有充分的日志
    - 是否外部化配置
  - 可观测性:
    - 是否有构建通知
    - 是否有性能指标
    - 是否有健康检查
```

## 8.2 定期维护

```yaml
# Jenkins 定期维护任务
每日:
  - 检查失败的构建
  - 检查队列积压
  - 检查 Agent 状态

每周:
  - 检查插件更新
  - 检查磁盘空间
  - 审查构建日志
  - 备份 JENKINS_HOME

每月:
  - 审查用户权限
  - 清理旧的构建记录
  - 审查 Pipeline 性能
  - 更新过时的插件

每季度:
  - 安全审计
  - 灾难恢复演练
  - 升级 Jenkins 版本
  - 审查并更新共享库
```
