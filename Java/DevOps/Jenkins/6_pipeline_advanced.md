
# 1. 并行执行

## 1.1 声明式并行

```groovy
pipeline {
    agent any
    stages {
        stage('Parallel Tests') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh './mvnw test'
                    }
                }
                stage('Integration Tests') {
                    steps {
                        sh './mvnw verify'
                    }
                }
                stage('Static Analysis') {
                    steps {
                        sh './mvnw checkstyle:checkstyle'
                    }
                }
            }
        }
    }
}
```

## 1.2 动态并行（脚本式）

使用 `parallel` 方法动态生成并行分支：

```groovy
def browsers = ['chrome', 'firefox', 'edge', 'safari']

def parallelStages = browsers.collectEntries { browser ->
    ["Testing on ${browser}": {
        node('selenium-agent') {
            stage("Test on ${browser}") {
                sh "./run-tests.sh --browser ${browser}"
            }
        }
    }]
}

parallel parallelStages
```

## 1.3 并行中的 failFast

```groovy
pipeline {
    agent any
    stages {
        stage('Tests') {
            // 任一并行分支失败，立即中止其他分支
            failFast true
            parallel {
                stage('Test A') {
                    steps { sh './test-a.sh' }
                }
                stage('Test B') {
                    steps { sh './test-b.sh' }
                }
            }
        }
    }
}
```

## 1.4 指定并行度

```groovy
// 限制同时运行的并行分支数量
def nodes = ['node1', 'node2', 'node3', 'node4']

def stages = [:]
nodes.eachWithIndex { node, index ->
    stages["Stage ${index}"] = {
        node(node) {
            stage("Stage on ${node}") {
                sh "echo Running on ${node}"
            }
        }
    }
}

// 使用 lock 或 throttle 限制并发
lock(resource: 'my-lock', variable: 'locked') {
    time = System.currentTimeMillis()
    parallel stages
}
```

# 2. 矩阵构建（Matrix）

声明式 Pipeline 的 `matrix` 指令支持多维度组合构建。

## 2.1 基本矩阵

```groovy
pipeline {
    agent any
    stages {
        stage('Matrix Build') {
            matrix {
                // 定义轴
                axes {
                    axis {
                        name 'PLATFORM'
                        values 'linux', 'windows', 'mac'
                    }
                    axis {
                        name 'BROWSER'
                        values 'chrome', 'firefox', 'edge'
                    }
                }

                // 排除不需要的组合
                excludes {
                    exclude {
                        axis {
                            name 'PLATFORM'
                            values 'linux'
                        }
                        axis {
                            name 'BROWSER'
                            values 'edge'
                        }
                    }
                }

                // 各组合的 stage
                stages {
                    stage('Test') {
                        steps {
                            echo "Testing on ${PLATFORM} with ${BROWSER}"
                            sh "./test.sh --platform ${PLATFORM} --browser ${BROWSER}"
                        }
                    }
                }
            }
        }
    }
}
```

## 2.2 矩阵高级用法

```groovy
pipeline {
    agent none
    stages {
        stage('Build and Test') {
            matrix {
                axes {
                    axis {
                        name 'OS'
                        values 'linux', 'windows'
                    }
                    axis {
                        name 'JDK'
                        values '11', '17', '21'
                    }
                }

                // 为每个组合指定不同的 agent
                agent {
                    label "${OS}-agent"
                }

                // 仅在特定组合运行特定 stage
                stages {
                    stage('Build') {
                        steps {
                            echo "Building with JDK ${JDK} on ${OS}"
                        }
                    }
                    stage('Test') {
                        when {
                            // 仅在 JDK17 和 linux 组合运行 Test
                            allOf {
                                expression { JDK == '17' }
                                expression { OS == 'linux' }
                            }
                        }
                        steps {
                            echo "Extended testing..."
                        }
                    }
                }
            }
        }
    }
}
```

# 3. 嵌套 Stage

## 3.1 声明式嵌套

```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            stages {
                stage('Compile') {
                    steps {
                        echo 'Compiling...'
                    }
                }
                stage('Package') {
                    steps {
                        echo 'Packaging...'
                    }
                }
            }
        }
        stage('Test') {
            stages {
                stage('Unit') {
                    steps {
                        echo 'Unit tests...'
                    }
                }
                stage('Integration') {
                    steps {
                        echo 'Integration tests...'
                    }
                }
            }
        }
    }
}
```

## 3.2 脚本式嵌套

```groovy
node {
    stage('Build') {
        stage('Compile') {
            sh 'mvn compile'
        }
        stage('Package') {
            sh 'mvn package'
        }
    }
    stage('Deploy') {
        stage('Deploy to Dev') {
            sh './deploy.sh dev'
        }
        stage('Deploy to Test') {
            sh './deploy.sh test'
        }
    }
}
```

# 4. 超时与重试

## 4.1 超时控制

```groovy
pipeline {
    agent any
    options {
        // Pipeline 级别超时
        timeout(time: 2, unit: 'HOURS')
    }
    stages {
        stage('Build') {
            options {
                // Stage 级别超时，覆盖 Pipeline 级别
                timeout(time: 30, unit: 'MINUTES')
            }
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Deploy') {
            steps {
                // 步骤级别超时
                timeout(time: 5, unit: 'MINUTES') {
                    sh './deploy.sh'
                }
            }
        }
        stage('Manual Approval') {
            steps {
                // input 步骤的超时
                timeout(time: 2, unit: 'HOURS') {
                    input message: 'Approve deployment?'
                }
            }
        }
    }
}
```

## 4.2 重试机制

```groovy
pipeline {
    agent any
    options {
        // Pipeline 级别重试（整个 Pipeline 重试）
        retry(3)
    }
    stages {
        stage('Flaky Test') {
            retry(5)  // Stage 级别重试
            steps {
                sh './flaky-test.sh'
            }
        }
        stage('Resilient Deploy') {
            steps {
                retry(3) {  // 步骤级别重试
                    sh './deploy.sh'
                }
            }
        }
    }
}
```

## 4.3 指数退避重试

```groovy
// 自定义指数退避重试逻辑
def retryWithBackoff(steps, maxRetries = 3, baseDelay = 5) {
    for (int i = 0; i < maxRetries; i++) {
        try {
            steps.call()
            return  // 成功则返回
        } catch (Exception e) {
            if (i == maxRetries - 1) {
                throw e  // 最后一次重试失败，抛出异常
            }
            def delay = baseDelay * Math.pow(2, i)
            echo "Attempt ${i + 1} failed, retrying in ${delay}s..."
            sleep delay
        }
    }
}

// 使用
stage('Resilient Deploy') {
    steps {
        script {
            retryWithBackoff({
                sh './deploy.sh'
            }, 5, 10)
        }
    }
}
```

# 5. Input 等待人工审批

## 5.1 基本 Input

```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                echo 'Building...'
            }
        }
        stage('Await Approval') {
            steps {
                input message: 'Deploy to production?',
                      ok: 'Deploy',
                      submitter: 'admin,release-manager',
                      submitterParameter: 'approvedBy'
            }
        }
        stage('Deploy') {
            steps {
                script {
                    echo "Approved by: ${approvedBy}"
                    sh './deploy.sh'
                }
            }
        }
    }
}
```

## 5.2 Input 带参数

```groovy
stage('Configurable Approval') {
    steps {
        script {
            def userInput = input(
                id: 'deployConfig',
                message: 'Configure deployment',
                parameters: [
                    string(name: 'VERSION', defaultValue: 'latest', description: 'Version to deploy'),
                    choice(name: 'ENV', choices: ['staging', 'prod'], description: 'Environment'),
                    text(name: 'RELEASE_NOTES', defaultValue: '', description: 'Release notes'),
                    booleanParam(name: 'ROLLBACK_ENABLED', defaultValue: false, description: 'Enable rollback')
                ]
            )
            // input 返回 Map
            env.VERSION = userInput.VERSION
            env.ENV = userInput.ENV
            env.RELEASE_NOTES = userInput.RELEASE_NOTES
            env.ROLLBACK_ENABLED = userInput.ROLLBACK_ENABLED
        }
    }
}
```

## 5.3 Input 超时处理

```groovy
stage('Timed Approval') {
    steps {
        script {
            try {
                timeout(time: 1, unit: 'HOURS') {
                    input message: 'Approve or reject deployment within 1 hour',
                          submitter: 'admin'
                }
            } catch (Exception err) {
                // 超时处理
                echo 'Approval timed out, skipping deployment'
                currentBuild.result = 'ABORTED'
                // 或使用 error 中止构建
            }
        }
    }
}
```

# 6. When 条件进阶

## 6.1 组合条件

```groovy
when {
    // 所有条件必须为真
    allOf {
        branch 'main'
        environment name: 'ENV', value: 'prod'
        expression { return currentBuild.previousBuild != null }
    }
}
```

```groovy
when {
    // 任一条件为真
    anyOf {
        branch 'release/*'
        tag 'v*'
        triggeredBy 'TimerTrigger'
    }
}
```

```groovy
when {
    // 条件取反
    not { branch 'main' }
}
```

## 6.2 构建原因判断

```groovy
when {
    // 由用户手动触发
    triggeredBy 'UserIdCause'

    // 由定时器触发
    triggeredBy 'TimerTrigger'

    // 由上游 Job 触发
    triggeredBy 'UpstreamCause'

    // 由 SCM 变更触发
    triggeredBy cause: 'SCMTriggerCause'

    // 详细判断触发原因
    expression {
        def causes = currentBuild.rawBuild.getCauses()
        return causes.any { cause ->
            cause.class.name.contains('GitHubWebhookCause')
        }
    }
}
```

## 6.3 变更文件检测

```groovy
when {
    // 有变更文件（需 git 插件）
    changeset "src/main/**"

    // 排除某些文件
    expression {
        return !changeset("*.md") && !changeset("*.txt")
    }
}

// 在 steps 中获取变更文件
steps {
    script {
        def changedFiles = []
        for (change in currentBuild.changeSets) {
            for (entry in change.getItems()) {
                changedFiles.add(entry.getPath())
            }
        }
        echo "Changed files: ${changedFiles}"
    }
}
```

# 7. Post 条件进阶

```groovy
post {
    // 构建失败
    failure {
        // 发送告警
        emailext(
            subject: "Build Failed: ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
            body: "See ${env.BUILD_URL}",
            to: 'dev-team@company.com'
        )
    }

    // 构建不稳定
    unstable {
        echo 'Build is unstable - check test results'
    }

    // 构建成功
    success {
        echo 'Build succeeded!'
    }

    // 构建状态在本次构建发生了变化
    //（例如上次失败这次成功，或上次成功这次失败）
    changed {
        echo 'Build status changed from previous build'
    }

    // 上一构建失败，本次成功
    fixed {
        echo 'Build is now stable again'
    }

    // 构建被中止
    aborted {
        echo 'Build was aborted'
    }

    // 回归：上次成功或不稳定，这次失败
    regression {
        echo 'Build regression detected'
    }

    // 始终执行（在所有其他 post 条件之后）
    cleanup {
        // 清理工作空间
        cleanWs()
        // 释放资源
        echo 'Cleanup completed'
    }
}

// 在 stage 级别使用 post
stage('Test') {
    steps {
        sh './run-tests.sh'
    }
    post {
        always {
            // 即使 stage 失败也归档测试报告
            junit 'target/surefire-reports/*.xml'
        }
    }
}
```

# 8. 工具与环境控制

## 8.1 withEnv 临时环境变量

```groovy
stage('Custom Env') {
    steps {
        withEnv([
            'MAVEN_OPTS=-Xmx2048m',
            "BRANCH=${params.BRANCH}",
            'ENV=production'
        ]) {
            sh 'mvn clean package'
        }
    }
}
```

## 8.2 withAnt、withMaven、withGradle

```groovy
// Maven（在声明式 tools 不满足时需要）
stage('Maven Build') {
    steps {
        withMaven(
            maven: 'Maven3',
            jdk: 'JDK17',
            mavenLocalRepo: '.repository',
            options: [
                globals: [
                    artifactsPublisher: [
                        disabled: true
                    ]
                ]
            ]
        ) {
            sh 'mvn clean package'
        }
    }
}

// Gradle（使用 tool 环境）
stage('Gradle Build') {
    steps {
        withGradle {
            sh './gradlew build'
        }
    }
}
```

## 8.3 withCredentials 绑定凭据

```groovy
stage('Docker Login') {
    steps {
        withCredentials([usernamePassword(
            credentialsId: 'docker-registry',
            usernameVariable: 'REGISTRY_USER',
            passwordVariable: 'REGISTRY_PASS'
        )]) {
            sh 'docker login -u $REGISTRY_USER -p $REGISTRY_PASS registry.company.com'
            sh 'docker push registry.company.com/myapp:latest'
        }
    }
}
```

# 9. 脚本式 Pipeline 模式

## 9.1 动态 Stage 生成

```groovy
def services = ['api-gateway', 'user-service', 'order-service', 'payment-service']

node {
    stage('Build All Services') {
        def builds = [:]
        services.each { service ->
            builds[service] = {
                node {
                    stage("Build ${service}") {
                        dir(service) {
                            checkout scm
                            sh './mvnw clean package'
                        }
                    }
                }
            }
        }
        parallel builds
    }
}
```

## 9.2 条件执行模式

```groovy
node {
    stage('Conditional Pipeline') {
        // 多分支条件
        def branchConfig = [
            'main': ['env': 'prod', 'deploy': true],
            'develop': ['env': 'dev', 'deploy': true],
            'release/*': ['env': 'staging', 'deploy': true],
        ]

        def currentBranch = env.BRANCH_NAME
        def config = branchConfig.find { k, v ->
            currentBranch ==~ /${k}/
        }?.value ?: ['env': 'feature', 'deploy': false]

        echo "Environment: ${config.env}"
        echo "Auto deploy: ${config.deploy}"
    }
}
```

## 9.3 状态机模式

```groovy
node {
    def pipelineState = 'INIT'

    stage('Code Checkout') {
        checkout scm
        pipelineState = 'CHECKOUT_DONE'
    }

    stage('Build') {
        sh './mvnw clean package'
        pipelineState = 'BUILD_DONE'
    }

    stage('Quality Gate') {
        // SonarQube 质量门禁
        sh './mvnw sonar:sonar'
        pipelineState = 'QUALITY_CHECKED'
    }

    stage('Deploy Decision') {
        if (pipelineState == 'QUALITY_CHECKED' && env.BRANCH_NAME == 'main') {
            pipelineState = 'READY_TO_DEPLOY'
        } else {
            echo "Not deploying: state=${pipelineState}, branch=${env.BRANCH_NAME}"
            pipelineState = 'SKIP_DEPLOY'
        }
    }
}
```

# 10. Pipeline 安全性

## 10.1 Groovy 沙箱

Jenkins Pipeline 默认在 Groovy 沙箱中执行，限制不安全的操作：

- 禁止使用 `new File()`、`System.exit()`、`Thread.sleep()` 等
- 禁止反射调用
- 禁止加载外部类

```groovy
// 沙箱中禁止的操作（会报错）
// File f = new File('/etc/passwd')
// System.exit(1)
// Class.forName('com.sun...')

// 需要管理员审批才能使用
// 位置：Manage Jenkins > In-process Script Approval
```

## 10.2 脚本审批

当 Pipeline 使用了签名的 Groovy 方法时，需要管理员审批：

```
审批路径：Manage Jenkins > In-process Script Approval
审批界面显示：
  Method: hudson.model.Run getRawBuild
  等待管理员审批或拒绝
```

**最佳实践**：
- 将复杂逻辑提取到共享库中，共享库代码不受沙箱限制
- 使用 `@NonCPS` 注解标记无需序列化的方法
- 减少对脚本审批的依赖

## 10.3 凭据安全

```groovy
// ✅ 正确：使用 withCredentials
withCredentials([usernamePassword(...)]) {
    sh '...'
}

// ❌ 错误：直接在代码中硬编码
// def password = 'my-password'

// ❌ 错误：使用 env 变量暴露凭据
// env.MY_PASSWORD = 'secret'

// ❌ 错误：在日志中打印凭据
// sh 'echo $PASSWORD'
```
