
# 1. 什么是共享库

共享库是 Jenkins Pipeline 中实现代码复用的一种机制，允许将通用的 Pipeline 逻辑提取到独立的 Git 仓库中，供多个 Jenkinsfile 引用。

核心价值：
- **消除重复**：避免在每个 Jenkinsfile 中重复相同的逻辑
- **统一标准**：团队统一使用相同的构建和部署逻辑
- **降低复杂度**：Jenkinsfile 变得简洁，复杂逻辑由共享库封装
- **独立演进**：共享库可以独立测试和版本化

# 2. 目录结构

共享库仓库的标准目录结构：

```
jenkins-shared-library/
├── vars/                    # 全局变量目录（最常用）
│   ├── buildJava.groovy    # 定义 buildJava 步骤
│   ├── deployApp.groovy    # 定义 deployApp 步骤
│   ├── gitCheckout.groovy  # 定义 gitCheckout 步骤
│   ├── log.groovy          # 定义 log 步骤
│   └── config.groovy       # 定义 config 步骤
├── src/                     # 源码目录（Groovy 类）
│   └── com/
│       └── company/
│           ├── PipelineUtils.groovy
│           ├── DockerUtils.groovy
│           └── Notifier.groovy
├── resources/               # 资源目录
│   ├── com/
│   │   └── company/
│   │       └── templates/
│   │           └── email.html
│   └── messages.properties
├── test/                    # 测试目录
│   └── groovy/
│       └── com/
│           └── company/
│               └── PipelineUtilsTest.groovy
└── Jenkinsfile              # 共享库本身的 CI
```

## 2.1 vars 目录详解

`vars/` 目录中的每个 `.groovy` 文件对应一个**全局变量**（即 Pipeline 中可直接调用的步骤）。

```groovy
// vars/buildJava.groovy
// 调用方式：buildJava(jdk: '17', mavenOpts: '-Xmx2048m')
def call(Map config = [:]) {
    def jdk = config.jdk ?: '11'
    def mavenOpts = config.mavenOpts ?: '-Xmx1024m'

    sh """
        export JAVA_HOME=/opt/jdk-${jdk}
        export MAVEN_OPTS="${mavenOpts}"
        mvn clean package
    """
}
```

函数命名规则：
- 文件名使用驼峰命名（如 `buildJava.groovy`)
- 文件中的 `call` 方法是默认入口
- 可直接在 Pipeline 中调用：`buildJava(...)`

## 2.2 src 目录详解

`src/` 目录存放标准的 Groovy 类文件，遵循 Java 的包结构。

```groovy
// src/com/company/PipelineUtils.groovy
package com.company

class PipelineUtils {
    static String getVersion(String branch, String buildNumber) {
        if (branch == 'main') {
            return "${buildNumber}"
        } else {
            return "${branch}-${buildNumber}"
        }
    }

    static Map parseConfig(String configFile) {
        // 读取并解析配置文件
        def config = readYaml file: configFile
        return config
    }

    String buildDockerImage(String appName, String version) {
        def imageName = "registry.company.com/${appName}:${version}"
        sh "docker build -t ${imageName} ."
        return imageName
    }
}
```

```groovy
// src/com/company/Notifier.groovy
package com.company

class Notifier implements Serializable {
    String channel
    String webhookUrl

    Notifier(String channel, String webhookUrl) {
        this.channel = channel
        this.webhookUrl = webhookUrl
    }

    void sendSuccess(String jobName, String buildNumber, String buildUrl) {
        send("✅ Build Success: ${jobName} #${buildNumber} - ${buildUrl}")
    }

    void sendFailure(String jobName, String buildNumber, String buildUrl) {
        send("❌ Build Failed: ${jobName} #${buildNumber} - ${buildUrl}")
    }

    private void send(String message) {
        sh """
            curl -X POST ${webhookUrl} \
                -H 'Content-Type: application/json' \
                -d '{"channel": "${channel}", "text": "${message}"}'
        """
    }
}
```

## 2.3 resources 目录详解

`resources/` 目录存放静态资源文件，可通过 `libraryResource` 步骤访问。

```groovy
// 在 vars 或 src 中使用 resources 中的文件
def loadTemplate() {
    def template = libraryResource 'com/company/templates/email.html'
    return template
}
```

# 3. 共享库的三种加载方式

## 3.1 全局共享库

在 Jenkins 全局配置中配置，对所有 Job 可用。

**配置路径**：Manage Jenkins > System > Global Pipeline Libraries

```
Name: my-shared-library
Default version: main
Load implicitly: true|false
Allow default version to be overridden: true
Retrieval method: Modern SCM
  - Git: https://github.com/myorg/jenkins-shared-library.git
  - Credentials: (optional)
```

```groovy
// 隐式加载（Load implicitly: true）
// 无需 @Library 注解，直接使用
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                buildJava(jdk: '17')
            }
        }
    }
}

// 显式加载（Load implicitly: false）
@Library('my-shared-library') _
pipeline { /* ... */ }
```

## 3.2 文件夹级共享库

在文件夹级别配置，仅对该文件夹下的 Job 可见。

**配置路径**：进入文件夹 > Configuration > Pipeline Libraries

```
Name: folder-library
Default version: main
Load implicitly: false
```

## 3.3 动态共享库

在 Jenkinsfile 中动态加载。

```groovy
// 方式一：仓库直接作为库
library 'my-shared-library@main'

// 方式二：从特定仓库加载
library identifier: 'custom-lib@v1.0',
        retriever: modernSCM(
            [$class: 'GitSCMSource',
             remote: 'https://github.com/myorg/custom-lib.git',
             credentialsId: 'git-creds']
        )

// 方式三：带版本范围
@Library('my-lib@>=1.0') _
```

# 4. 编写共享库函数

## 4.1 基本函数

```groovy
// vars/hello.groovy
def call(String name) {
    echo "Hello, ${name}!"
}

// 使用：
// hello('Jenkins')
```

## 4.2 带 Map 参数的函数

```groovy
// vars/buildApp.groovy
def call(Map config) {
    def lang = config.language ?: 'java'
    def task = config.task ?: 'build'
    def jdk   = config.jdk ?: '11'

    echo "Building ${lang} project with JDK ${jdk}, task: ${task}"

    switch(lang) {
        case 'java':
            sh "mvn ${task}"
            break
        case 'node':
            sh "npm run ${task}"
            break
        case 'python':
            sh "python setup.py ${task}"
            break
        default:
            error "Unsupported language: ${lang}"
    }
}

// 使用：
// buildApp(language: 'java', jdk: '17', task: 'package')
```

## 4.3 带 Closure 的函数

```groovy
// vars/withMavenEnv.groovy
def call(Closure body) {
    withEnv([
        "MAVEN_OPTS=-Xmx2048m",
        "MAVEN_HOME=${tool 'Maven3'}"
    ]) {
        sh 'mvn --version'
        body()
    }
}

// 使用：
// withMavenEnv {
//     sh 'mvn clean test'
// }
```

## 4.4 多步函数

```groovy
// vars/javaPipeline.groovy
def call(Map config) {
    pipeline {
        agent any
        options {
            timeout(time: config.timeout ?: 1, unit: 'HOURS')
        }
        stages {
            stage('Checkout') {
                steps {
                    checkout scm
                }
            }
            stage('Build') {
                steps {
                    sh 'mvn clean compile'
                }
            }
            stage('Test') {
                steps {
                    sh 'mvn test'
                }
                post {
                    always {
                        junit 'target/surefire-reports/*.xml'
                    }
                }
            }
            stage('Package') {
                steps {
                    sh 'mvn package -DskipTests=true'
                }
            }
        }
        post {
            always {
                cleanWs()
            }
        }
    }
}

// 使用：
// javaPipeline(timeout: 2)
```

# 5. 共享库中的序列化问题

Jenkins Pipeline 需要序列化所有变量（用于暂停/恢复）。共享库函数和类必须正确处理序列化。

## 5.1 实现 Serializable

```groovy
// ✅ 正确：实现 Serializable 接口
class MyUtils implements Serializable {
    def steps  // 必须存储 steps 引用

    MyUtils(steps) {
        this.steps = steps
    }
}

// ❌ 错误：非序列化字段
class BadUtils {
    // 闭包默认不可序列化
    def callback = { -> println "hello" }

    // 未标注 Serializable 的自定义类型
    def helper = new NonSerializableHelper()
}
```

## 5.2 使用 @NonCPS 注解

`@NonCPS`（Non-Continuable Passing Style）标记的方法不会被 Jenkins 序列化，适合处理不可序列化的对象。

```groovy
import com.cloudbees.groovy.cps.NonCPS

class StringUtils {
    @NonCPS
    static String reverse(String input) {
        return input.reverse()
    }

    @NonCPS
    static boolean matches(String str, String regex) {
        return str.matches(regex)
    }
}

// 使用 @NonCPS 处理集合操作
def processList(List items) {
    // 在 @NonCPS 方法中可以使用不可序列化的操作
    @NonCPS
    def transform(list) {
        list.findAll { it.startsWith('test') }
            .collect { it.toUpperCase() }
            .sort()
    }

    return transform(items)
}
```

**注意事项**：
- `@NonCPS` 方法中不能调用 Pipeline 步骤（如 `sh`、`echo`）
- `@NonCPS` 方法中不能使用 `sleep`
- `@NonCPS` 方法返回的结果必须可序列化

## 5.3 常见序列化陷阱

```groovy
// ❌ 陷阱 1：在类中直接使用 Pipeline 步骤
class Builder {
    def steps
    def build() {
        // 错误：在闭包中引用步骤
        def process = { ->
            steps.sh 'mvn clean'  // 闭包不可序列化
        }
        process()
    }
}

// ✅ 正确：传递 steps 引用并在方法级别使用
class Builder implements Serializable {
    def steps
    def build() {
        steps.sh 'mvn clean'
    }
}
```

```groovy
// ❌ 陷阱 2：使用不可序列化的对象
class ConfigHandler {
    // URL 类不可序列化
    def url = new URL('https://api.example.com')
}

// ✅ 正确处理：存储字符串，使用时再转换
class ConfigHandler implements Serializable {
    String urlStr = 'https://api.example.com'
    def getUrl() { new URL(urlStr) }
}
```

# 6. 共享库测试

## 6.1 使用 JenkinsPipelineUnit 测试

共享库可以使用 [JenkinsPipelineUnit](https://github.com/jenkinsci/JenkinsPipelineUnit) 框架进行单元测试。

```groovy
// 添加依赖（Gradle）
// build.gradle
dependencies {
    testImplementation 'com.github.stephenc:jenkins-pipeline-unit:1.23'
}

// 测试 vars/buildJava.groovy
// test/groovy/BuildJavaTest.groovy
import com.lesfurets.jenkins.unit.BasePipelineTest

class BuildJavaTest extends BasePipelineTest {

    def setUp() {
        baseSetUp()
        // 注册变量
        helper.registerAllowedMethod('sh', [String], { cmd ->
            println "Executing: ${cmd}"
            return ''
        })
    }

    def testBuildJava() {
        def script = loadScript('vars/buildJava.groovy')
        script.call(jdk: '17', mavenOpts: '-Xmx2048m')

        // 验证 sh 被调用
        assertThat(helper.callStack.find { it.methodName == 'sh' }, notNullValue())
    }
}
```

## 6.2 在测试 Pipeline 中验证

创建测试用的 Pipeline：

```groovy
// testJob/Jenkinsfile
@Library('my-shared-library@test') _

pipeline {
    agent any
    stages {
        stage('Test Library') {
            steps {
                buildJava(jdk: '17')
                deployApp(env: 'test', version: '1.0.0')
            }
        }
    }
}
```

## 6.3 集成测试

```groovy
// 共享库自身的 Jenkinsfile
@Library('pipeline-testing') _

pipeline {
    agent any
    stages {
        stage('Unit Tests') {
            steps {
                sh 'gradle test'
            }
        }
        stage('Integration Test') {
            steps {
                script {
                    // 创建一个测试 Job 并运行
                    jobDsl targets: 'testJob.groovy'
                    build job: 'test-job',
                          parameters: [string(name: 'BRANCH', value: 'develop')]
                }
            }
        }
    }
}
```

# 7. 共享库版本管理

## 7.1 版本策略

```yaml
# 推荐版本管理策略
分支策略:
  main: 稳定版本，用于生产
  develop: 开发版本
  标签: v1.0.0, v1.1.0 等语义化版本

使用示例:
  @Library('my-lib@main') _      # 使用 main 分支最新
  @Library('my-lib@develop') _   # 使用开发分支
  @Library('my-lib@v1.0.0') _   # 使用特定版本（推荐）
```

## 7.2 Semantic Versioning

```
v<major>.<minor>.<patch>

major: 不兼容的 API 修改
minor: 新增功能，向后兼容
patch: Bug 修复，向后兼容

示例：
v1.0.0 - 初始版本
v1.1.0 - 新增功能
v1.1.1 - Bug 修复
v2.0.0 - 不兼容变更
```

## 7.3 使用版本范围

```groovy
// 精确版本
@Library('my-lib@v1.2.3') _

// 版本范围（需插件支持）
@Library('my-lib@>=1.0 <2.0') _

// 动态版本
@Library('my-lib@develop') _
```

# 8. 共享库最佳实践

## 8.1 目录组织

```
# 推荐目录结构
jenkins-shared-library/
├── vars/
│   ├── dockerBuild.groovy      # 一个文件一个功能
│   ├── dockerPush.groovy
│   ├── gitCheckout.groovy
│   ├── mavenBuild.groovy
│   ├── npmBuild.groovy
│   ├── notifySlack.groovy
│   └── deploy.groovy
├── src/
│   └── com/company/
│       ├── utils/
│       │   ├── VersionUtil.groovy
│       │   └── ConfigUtil.groovy
│       ├── docker/
│       │   └── DockerClient.groovy
│       └── notifications/
│           ├── SlackNotifier.groovy
│           └── EmailNotifier.groovy
├── resources/
│   └── templates/
│       ├── email_success.html
│       └── email_failure.html
├── test/
│   └── groovy/
│       └── ...
└── README.md
```

## 8.2 错误处理

```groovy
// vars/safeDeploy.groovy
def call(Map config) {
    def maxRetries = config.retries ?: 3
    def env = config.env
    def version = config.version

    // 参数校验
    if (!env) {
        error 'env parameter is required'
    }
    if (!version) {
        error 'version parameter is required'
    }

    // 带重试的部署
    retry(maxRetries) {
        try {
            sh "./deploy.sh ${env} ${version}"
        } catch (Exception e) {
            echo "Deploy attempt failed: ${e.message}"
            // 执行回滚
            rollback(env: env)
            throw e  // 让 retry 捕获
        }
    }
}

// 使用
stage('Deploy') {
    steps {
        safeDeploy(env: 'prod', version: BUILD_NUMBER, retries: 3)
    }
}
```

## 8.3 配置管理

```groovy
// vars/withAppConfig.groovy
def call(String env, Closure body) {
    // 从 YAML 配置文件读取环境特定配置
    def config = readYaml file: "config/${env}/application.yml"

    withEnv([
        "APP_ENV=${env}",
        "DB_URL=${config.database.url}",
        "REDIS_URL=${config.redis.url}",
        "LOG_LEVEL=${config.logging.level}"
    ]) {
        body()
    }
}

// 使用
stage('Deploy') {
    steps {
        withAppConfig('prod') {
            sh './deploy.sh'
        }
    }
}
```

## 8.4 文档化

```groovy
// vars/notifySlack.groovy
/**
 * 发送 Slack 通知
 *
 * @param channel Slack 频道名称（必填）
 * @param message 通知消息（必填）
 * @param color 消息颜色: good/warning/danger（可选，默认 good）
 * @param notifyOnlyOnFailure 仅失败时通知（可选，默认 false）
 *
 * 用法:
 * notifySlack(
 *     channel: '#deployments',
 *     message: "Deploy complete: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
 *     color: 'good'
 * )
 */
def call(Map config) {
    // 实现代码
}
```

# 9. 共享库演练：完整示例

## 9.1 创建共享库

```groovy
// vars/javaBuild.groovy
def call(Map config) {
    def jdkVersion = config.jdk ?: '17'
    def skipTests = config.skipTests ?: false

    stage('Checkout') {
        checkout scm
    }

    stage('Build') {
        sh "mvn clean compile -Dmaven.test.skip=${skipTests}"
    }

    if (!skipTests) {
        stage('Test') {
            sh 'mvn test'
        }
        stage('Quality') {
            sh 'mvn checkstyle:checkstyle pmd:pmd'
        }
    }

    stage('Package') {
        sh 'mvn package -DskipTests=true'
    }
}
```

```groovy
// vars/dockerBuildAndPush.groovy
def call(Map config) {
    def dockerImage = config.image
    def dockerTag = config.tag ?: 'latest'
    def dockerRegistry = config.registry ?: 'registry.company.com'
    def dockerCredentials = config.credentialsId ?: 'docker-registry-creds'

    stage('Docker Build') {
        sh """
            docker build \
                -t ${dockerRegistry}/${dockerImage}:${dockerTag} \
                -t ${dockerRegistry}/${dockerImage}:latest \
                .
        """
    }

    stage('Docker Push') {
        withCredentials([usernamePassword(
            credentialsId: dockerCredentials,
            usernameVariable: 'DOCKER_USER',
            passwordVariable: 'DOCKER_PASS'
        )]) {
            sh """
                docker login -u ${DOCKER_USER} -p ${DOCKER_PASS} ${dockerRegistry}
                docker push ${dockerRegistry}/${dockerImage}:${dockerTag}
                docker push ${dockerRegistry}/${dockerImage}:latest
            """
        }
    }
}
```

```groovy
// vars/sendNotification.groovy
def call(String status) {
    def color = status == 'success' ? 'good' : 'danger'
    def message = status == 'success' ?
        "✅ Build Succeeded: ${env.JOB_NAME} #${env.BUILD_NUMBER}" :
        "❌ Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}"

    // Slack
    slackSend(color: color, message: message)

    // Email
    if (status == 'failure') {
        emailext(
            to: 'dev-team@company.com',
            subject: "[FAIL] ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            body: "Build failed: ${env.BUILD_URL}"
        )
    }
}
```

## 9.2 使用共享库的 Jenkinsfile

```groovy
@Library('my-shared-library@v1.0.0') _

pipeline {
    agent any
    options {
        timeout(time: 1, unit: 'HOURS')
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }
    stages {
        stage('CI Build') {
            steps {
                javaBuild(jdk: '17', skipTests: false)
            }
        }
        stage('Docker') {
            steps {
                dockerBuildAndPush(
                    image: 'my-app',
                    tag: BUILD_NUMBER,
                    registry: 'registry.company.com'
                )
            }
        }
        stage('Deploy') {
            steps {
                sh "./deploy.sh ${params.ENV} ${BUILD_NUMBER}"
            }
        }
    }
    post {
        success {
            sendNotification('success')
        }
        failure {
            sendNotification('failure')
        }
        always {
            cleanWs()
        }
    }
}
```
