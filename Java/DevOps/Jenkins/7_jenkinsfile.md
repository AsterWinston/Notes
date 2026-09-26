
# 1. 什么是 Jenkinsfile

Jenkinsfile 是一个文本文件，使用 Groovy DSL 定义 Jenkins Pipeline 的完整流程。它通常存放在仓库根目录，纳入版本控制，实现 **Pipeline as Code**。

核心优势：
- **版本控制**：Pipeline 变更随代码一起审查
- **代码审查**：通过 PR/MR 审查 Pipeline 变更
- **可追溯**：可以回溯每个版本的 Pipeline 定义
- **可复用**：可在不同分支上使用不同的 Pipeline
- **标准化**：团队统一使用 Jenkinsfile 定义 CI/CD

# 2. 多分支 Pipeline（Multibranch Pipeline）

多分支 Pipeline 自动发现 SCM 中的分支，为每个包含 Jenkinsfile 的分支创建对应的 Pipeline。

## 2.1 创建多分支 Pipeline

```
1. Jenkins > New Item > Multibranch Pipeline
2. Branch Sources:
   - Git: https://github.com/myorg/myrepo.git
   - Credentials: github-creds
3. Build Configuration:
   - Mode: by Jenkinsfile
   - Script Path: Jenkinsfile  # 默认为 Jenkinsfile
4. Scan Multibranch Pipeline Triggers:
   - Periodically if not otherwise run: 1 minute
5. Orphaned Item Strategy:
   - Discard old items: 30 days / 100 items
```

## 2.2 分支发现策略

```yaml
# 分支发现配置
Branch Sources:
  - Git:
    - Repository URL: https://github.com/myorg/myrepo.git
    - Discover branches:
      - Strategy: All branches
      - Strategy: Only branches that are also filed as PRs
    - Discover pull requests from origin:
      - Strategy: Merging the PR with the current target branch
    - Discover tags:
      - Strategy: Advanced, check "Discover tags"
```

## 2.3 Jenkinsfile 中的分支感知

```groovy
pipeline {
    agent any
    triggers {
        // 多分支自动提供 BRANCH_NAME 环境变量
        pollSCM('H/5 * * * *')
    }
    stages {
        stage('Build') {
            steps {
                echo "Building branch: ${env.BRANCH_NAME}"
                script {
                    switch(env.BRANCH_NAME) {
                        case 'main':
                            env.DEPLOY_ENV = 'prod'
                            env.SKIP_TESTS = false
                            break
                        case 'develop':
                            env.DEPLOY_ENV = 'dev'
                            env.SKIP_TESTS = false
                            break
                        case ~/release\/.*/:
                            env.DEPLOY_ENV = 'staging'
                            env.SKIP_TESTS = false
                            break
                        default:
                            env.DEPLOY_ENV = ''
                            env.SKIP_TESTS = true  // 特性分支可跳过部分测试
                    }
                }
            }
        }
        stage('Test') {
            when {
                expression { env.BRANCH_NAME !=~ /feature\/.*/ }
            }
            steps {
                sh './mvnw test'
            }
        }
        stage('Deploy') {
            when {
                expression { env.DEPLOY_ENV != '' }
            }
            steps {
                sh "./deploy.sh ${env.DEPLOY_ENV}"
            }
        }
    }
}
```

# 3. Jenkinsfile 通用模板

## 3.1 Java Maven 项目模板

```groovy
pipeline {
    agent {
        label 'linux && docker'
    }
    options {
        timeout(time: 1, unit: 'HOURS')
        buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '5'))
        timestamps()
        skipDefaultCheckout()
        disableConcurrentBuilds()
    }
    environment {
        MAVEN_OPTS = '-Xmx2048m -XX:MaxMetaspaceSize=512m'
        MAVEN_CLI_OPTS = '--batch-mode --errors --fail-at-end --show-version'
        MAVEN_HOME = tool name: 'Maven3', type: 'maven'
        JAVA_HOME = tool name: 'JDK17', type: 'jdk'
    }
    parameters {
        string(name: 'BRANCH', defaultValue: 'develop', description: 'Git branch')
        choice(name: 'ENV', choices: ['dev', 'test', 'staging', 'prod'], description: 'Deploy environment')
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Build') {
            steps {
                sh '''
                    ${MAVEN_HOME}/bin/mvn ${MAVEN_CLI_OPTS} \
                        clean compile \
                        -Dmaven.test.skip=true
                '''
            }
        }
        stage('Unit Test') {
            steps {
                sh '''
                    ${MAVEN_HOME}/bin/mvn ${MAVEN_CLI_OPTS} test
                '''
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }
        stage('Package') {
            steps {
                sh '''
                    ${MAVEN_HOME}/bin/mvn ${MAVEN_CLI_OPTS} package \
                        -DskipTests=true
                '''
            }
            post {
                success {
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                }
            }
        }
        stage('Docker Build & Push') {
            when {
                expression { params.ENV != 'dev' }
            }
            steps {
                script {
                    docker.build("registry.company.com/myapp:${BUILD_NUMBER}")
                    docker.withRegistry('https://registry.company.com', 'docker-registry-creds') {
                        docker.image("registry.company.com/myapp:${BUILD_NUMBER}").push()
                        if (params.ENV == 'prod') {
                            docker.image("registry.company.com/myapp:${BUILD_NUMBER}").push('latest')
                        }
                    }
                }
            }
        }
        stage('Deploy') {
            when {
                expression { params.ENV != 'dev' }
            }
            steps {
                script {
                    sh "./deploy.sh ${params.ENV} ${BUILD_NUMBER}"
                }
            }
        }
    }
    post {
        always {
            cleanWs()
        }
        failure {
            emailext(
                subject: "[FAIL] ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Build failed: ${env.BUILD_URL}",
                to: 'dev-team@company.com'
            )
        }
        changed {
            emailext(
                subject: "[${env.BUILD_STATUS}] ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Build status changed: ${env.BUILD_URL}",
                to: 'dev-team@company.com'
            )
        }
    }
}
```

## 3.2 Node.js 前端项目模板

```groovy
pipeline {
    agent {
        label 'linux && nodejs'
    }
    options {
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '30'))
    }
    environment {
        CI = 'true'
        NODE_OPTIONS = '--max-old-space-size=4096'
    }
    stages {
        stage('Install') {
            steps {
                sh 'npm ci'
            }
        }
        stage('Lint') {
            steps {
                sh 'npm run lint'
            }
        }
        stage('Test') {
            steps {
                sh 'npm run test -- --coverage --reporters=jest-junit'
            }
            post {
                always {
                    junit 'reports/junit.xml'
                }
            }
        }
        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }
        stage('Archive') {
            steps {
                zip archive: true, dir: 'dist/', glob: ''
                archiveArtifacts artifacts: 'dist.tar.gz'
            }
        }
    }
    post {
        always {
            cleanWs()
        }
    }
}
```

## 3.3 Python 项目模板

```groovy
pipeline {
    agent {
        label 'linux && python'
    }
    options {
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
    }
    stages {
        stage('Setup') {
            steps {
                sh '''
                    python3 -m venv venv
                    source venv/bin/activate
                    pip install -r requirements.txt -r requirements-dev.txt
                '''
            }
        }
        stage('Lint') {
            steps {
                sh '''
                    source venv/bin/activate
                    flake8 src/ tests/
                    pylint src/
                '''
            }
        }
        stage('Test') {
            steps {
                sh '''
                    source venv/bin/activate
                    pytest tests/ \
                        --junitxml=reports/junit.xml \
                        --cov=src/ \
                        --cov-report=xml \
                        --cov-report=html
                '''
            }
            post {
                always {
                    junit 'reports/junit.xml'
                }
            }
        }
        stage('Build') {
            steps {
                sh '''
                    source venv/bin/activate
                    python setup.py sdist bdist_wheel
                '''
            }
        }
    }
}
```

# 4. 共享库集成

## 4.1 在 Jenkinsfile 中加载共享库

```groovy
// 全局共享库（在 Jenkins 中配置）
@Library('my-shared-library') _

// 特定版本
@Library('my-shared-library@v1.2.3') _

// 多个库
@Library(['lib-one', 'lib-two']) _

// 带类名导入
@Library('my-shared-library')
import com.company.pipeline.Utils

pipeline {
    agent any
    stages {
        stage('Use Shared Lib') {
            steps {
                // 使用共享库中的全局变量
                hello 'Jenkins'

                // 使用共享库中的类
                script {
                    def utils = new Utils()
                    utils.someMethod()
                }
            }
        }
    }
}
```

## 4.2 Jenkinsfile 中的共享库配置

在 Jenkins 中配置共享库：**Manage Jenkins > System > Global Pipeline Libraries**

```yaml
# 全局共享库配置
Name: my-shared-library
Default version: main
Load implicitly: false  # 需在 Jenkinsfile 中显式 @Library
Allow default version to be overridden: true
Include in changeset: false
Retrieval method: Modern SCM
  Source Code Management: Git
    Project Repository: https://github.com/myorg/jenkins-shared-library.git
    Credentials: github-creds
    Behaviours: (默认即可)
```

# 5. Jenkinsfile 中的安全实践

## 5.1 凭据管理

```groovy
// ✅ 使用凭据绑定，不在日志中暴露
withCredentials([usernamePassword(
    credentialsId: 'deploy-creds',
    usernameVariable: 'DEPLOY_USER',
    passwordVariable: 'DEPLOY_PASS'
)]) {
    sh '''
        curl -u "$DEPLOY_USER:$DEPLOY_PASS" \
            -X POST https://api.company.com/deploy
    '''
}

// ✅ 使用 mask 密码
sh script: '''
    echo "Password: $SECRET"
''', label: 'Execute with credentials'

// ❌ 禁止：硬编码
// sh 'curl -u admin:password123 https://api.company.com/deploy'

// ❌ 禁止：直接打印凭据
// sh 'echo $DEPLOY_PASS'
```

## 5.2 Script Approval 最小化

```groovy
// ✅ 将复杂逻辑放入共享库
@Library('my-lib') _
pipeline {
    agent any
    stages {
        stage('Deploy') {
            steps {
                // 共享库中的方法不受沙箱限制
                safeDeploy(
                    env: params.ENV,
                    version: BUILD_NUMBER
                )
            }
        }
    }
}

// ❌ 避免在 Pipeline 中使用需要审批的方法
// def rawBuild = currentBuild.rawBuild  // 需要审批
```

## 5.3 Git 安全

```groovy
// ✅ 使用凭据 ID
checkout([
    $class: 'GitSCM',
    userRemoteConfigs: [[
        url: 'https://github.com/myorg/repo.git',
        credentialsId: 'my-git-creds'
    ]],
    branches: [[name: '*/main']]
])

// ❌ 禁止：URL 中嵌入凭据
// checkout url: 'https://user:pass@github.com/myorg/repo.git'
```

# 6. Jenkinsfile 调试技巧

## 6.1 本地验证语法

```bash
# 通过 Jenkins CLI 验证 Jenkinsfile
curl -X POST -u admin:token \
  http://jenkins:8080/pipeline-model-converter/validate \
  --data-urlencode "jenkinsfile@Jenkinsfile"

# 使用 Replay 功能
# 在构建历史中点击 Replay，可修改 Jenkinsfile 重新执行
```

## 6.2 日志输出

```groovy
// 条件输出日志
def DEBUG = true

stage('Debug Info') {
    steps {
        script {
            if (DEBUG) {
                echo "Branch: ${env.BRANCH_NAME}"
                echo "Build Number: ${env.BUILD_NUMBER}"
                echo "Workspace: ${env.WORKSPACE}"
                echo "Node Name: ${env.NODE_NAME}"

                // 查看当前环境变量
                sh 'env | sort'

                // 查看工作空间内容
                sh 'ls -la'
            }
        }
    }
}
```

## 6.3 Pipeline 重放

```
在构建历史页面，点击任意构建：
1. 进入构建详情页
2. 点击左侧 "Replay" 按钮
3. 修改 Jenkinsfile 内容
4. 点击 Run
用途：在不提交代码的情况下快速调试 Pipeline
```

## 6.4 使用 spy 调试

```groovy
// 编写调试用的共享库函数
// vars/debug.groovy
def call(Map config = [:]) {
    echo "=== DEBUG ==="
    config.each { key, value ->
        echo "${key}: ${value}"
    }
    echo "=============="
}

// Jenkinsfile 中使用
stage('Debug') {
    steps {
        debug(
            branch: env.BRANCH_NAME,
            buildNumber: env.BUILD_NUMBER,
            env: env.BUILD_ENV
        )
    }
}
```

# 7. Jenkinsfile 中的工具链

## 7.1 多 JDK 构建

```groovy
pipeline {
    agent any
    stages {
        stage('Build with JDK 11') {
            tools {
                jdk 'JDK11'
            }
            steps {
                sh 'java -version'
                sh 'mvn clean compile'
            }
        }
        stage('Test with JDK 17') {
            tools {
                jdk 'JDK17'
            }
            steps {
                sh 'java -version'
                sh 'mvn test'
            }
        }
        // JDK 矩阵构建
        stage('Matrix JDK') {
            matrix {
                axes {
                    axis {
                        name 'JDK_VERSION'
                        values '11', '17', '21'
                    }
                }
                stages {
                    stage('Build') {
                        tools {
                            jdk "JDK${JDK_VERSION}"
                        }
                        steps {
                            sh "echo Building with JDK ${JDK_VERSION}"
                            sh 'mvn clean compile'
                        }
                    }
                }
            }
        }
    }
}
```

## 7.2 Docker 构建工具链

```groovy
pipeline {
    agent any
    stages {
        stage('Build in Docker') {
            agent {
                docker {
                    image 'maven:3.9-eclipse-temurin-17'
                    args '-v /root/.m2:/root/.m2'
                }
            }
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Test in Docker') {
            agent {
                docker {
                    image 'node:20-alpine'
                }
            }
            steps {
                sh 'node --version'
                sh 'npm --version'
            }
        }
    }
}
```

# 8. Jenkinsfile 的版本管理

## 8.1 多版本 Jenkinsfile

```
# 在仓库中的位置
repo/
├── Jenkinsfile              # 标准 Pipeline
├── Jenkinsfile.monolith     # 单体应用 Pipeline
├── Jenkinsfile.microservice # 微服务 Pipeline
├── Jenkinsfile.library      # 库项目 Pipeline
└── src/
```

多分支 Pipeline 配置中指定 `Script Path` 为对应文件名。

## 8.2 Jenkinsfile 模板化

使用共享库定义 Pipeline 模板：

```groovy
// shared-library/vars/javaPipeline.groovy
def call(Map config) {
    pipeline {
        agent { label config.agentLabel ?: 'linux' }
        options {
            timeout(time: config.timeout ?: 1, unit: 'HOURS')
            buildDiscarder(logRotator(numToKeepStr: '20'))
        }
        environment {
            MAVEN_OPTS = config.mavenOpts ?: '-Xmx2048m'
        }
        stages {
            stage('Build') {
                steps {
                    sh './mvnw clean package'
                }
            }
            stage('Test') {
                steps {
                    sh './mvnw test'
                }
            }
            if (config.deployEnabled) {
                stage('Deploy') {
                    steps {
                        sh "./deploy.sh ${config.env}"
                    }
                }
            }
        }
    }
}

// 使用模板
// Jenkinsfile
@Library('my-lib') _

javaPipeline(
    agentLabel: 'linux && docker',
    timeout: 2,
    deployEnabled: true,
    env: params.ENV
)
```

# 9. Jenkinsfile 中的通知集成

## 9.1 邮件通知

```groovy
post {
    failure {
        emailext(
            to: 'dev-team@company.com',
            subject: "[FAILED] ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            body: """
                <h2>构建失败</h2>
                <p>项目: ${env.JOB_NAME}</p>
                <p>构建: #${env.BUILD_NUMBER}</p>
                <p>分支: ${env.BRANCH_NAME}</p>
                <p>持续时间: ${currentBuild.durationString}</p>
                <p>构建 URL: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                <p>提交者: ${env.CHANGE_AUTHOR}</p>
                <h3>变更内容:</h3>
                <pre>${env.CHANGE_LOG}</pre>
            """,
            mimeType: 'text/html',
            attachmentsPattern: 'target/surefire-reports/*.xml'
        )
    }
}
```

## 9.2 Slack 通知

```groovy
post {
    success {
        slackSend(
            channel: '#ci-builds',
            color: 'good',
            message: "✅ Build Success: ${env.JOB_NAME} #${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)"
        )
    }
    failure {
        slackSend(
            channel: '#ci-builds',
            color: 'danger',
            message: "❌ Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)"
        )
    }
    unstable {
        slackSend(
            channel: '#ci-builds',
            color: 'warning',
            message: "⚠️ Build Unstable: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        )
    }
}
```

## 9.3 企业微信/钉钉通知

```groovy
post {
    failure {
        sh """
            curl -X POST https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx \
                -H 'Content-Type: application/json' \
                -d '{
                    "msgtype": "markdown",
                    "markdown": {
                        "content": "## 构建失败\\n项目: ${env.JOB_NAME}\\n构建: #${env.BUILD_NUMBER}\\n[查看详情](${env.BUILD_URL})"
                    }
                }'
        """
    }
}
```

# 10. 大型项目 Jenkinsfile 结构建议

## 10.1 微服务多模块

```groovy
// 使用 shared library 来管理通用逻辑
@Library(['pipeline-utils', 'deployment-lib']) _

def modules = ['common', 'service-a', 'service-b', 'service-c']

pipeline {
    agent none
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
    }
    stages {
        stage('Build All') {
            parallel {
                script {
                    modules.each { module ->
                        def moduleName = module
                        stage("Build ${moduleName}") {
                            agent {
                                docker {
                                    image 'maven:3.9-eclipse-temurin-17'
                                    args '-v /root/.m2:/root/.m2'
                                }
                            }
                            steps {
                                dir(moduleName) {
                                    checkout scm
                                    sh "mvn clean package -pl ${moduleName} -am"
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('Integration Test') {
            agent {
                label 'integration-agent'
            }
            steps {
                sh './run-integration-tests.sh'
            }
        }
    }
}
```

## 10.2 性能优化建议

```groovy
// 1. 浅克隆：减少 Git 数据传输
checkout([
    $class: 'GitSCM',
    branches: [[name: '*/main']],
    extensions: [[$class: 'CloneOption', depth: 1, shallow: true, noTags: true]],
    userRemoteConfigs: [[url: 'https://github.com/myorg/repo.git']]
])

// 2. 合理利用 Agent 标签，避免不必要的 Agent 切换
agent { label 'linux && (docker || high-memory)' }

// 3. 使用 Artifact 缓存
// 在构建间共享依赖缓存
agent {
    docker {
        image 'maven:3.9-eclipse-temurin-17'
        args '-v maven-repo-cache:/root/.m2'
    }
}

// 4. 避免在 Pipeline 中使用大量环境变量
// ❌ 效率低
// env.A = '1'
// env.B = '2'
// env.C = '3'

// ✅ 使用局部变量
// def a = '1', b = '2', c = '3'
```
