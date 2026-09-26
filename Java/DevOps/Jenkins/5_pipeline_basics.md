
# 1. 什么是 Jenkins Pipeline

Jenkins Pipeline 是一套用代码定义 CI/CD 流程的框架，是 Jenkins 2.x 的核心特性。Pipeline 支持两种语法：声明式（Declarative）和脚本式（Scripted），两者都基于 Groovy 语言。

核心设计目标：
- **持久性**：Pipeline 可以安全地重启，即使 Jenkins Master 宕机重启，正在执行的 Pipeline 不会丢失状态
- **可暂停**：Pipeline 可以暂停等待人工审批或外部输入
- **可编程**：支持条件、循环、并行、异常处理等编程结构
- **可视化**：Pipeline 的执行进度在 UI 上实时展示

# 2. 声明式 Pipeline（Declarative）

## 2.1 基本结构

```groovy
pipeline {
    agent any  // 在任何可用 Agent 上执行

    // 所有阶段定义
    stages {
        stage('Build') {
            steps {
                echo 'Building...'
            }
        }
        stage('Test') {
            steps {
                echo 'Testing...'
            }
        }
        stage('Deploy') {
            steps {
                echo 'Deploying...'
            }
        }
    }

    // 后置处理，无论构建结果如何
    post {
        always {
            echo 'This will always run'
        }
        success {
            echo 'This will run only on success'
        }
        failure {
            echo 'This will run only on failure'
        }
        unstable {
            echo 'This will run only on unstable'
        }
        changed {
            echo 'This will run when the build status changes'
        }
    }
}
```

## 2.2 指令详解

**agent 指令**

指定 Pipeline 在哪个 Agent 上执行。

```groovy
pipeline {
    // 在任何可用 Agent 上执行
    agent any

    // 在特定标签的 Agent 上执行
    agent { label 'linux && docker' }

    // 在特定 Agent 上执行（通过名称）
    agent { node { label 'master' } }

    // 不使用 Agent（用于发送通知等不需执行的步骤）
    agent none
}

// stage 级别 agent：不同阶段使用不同 Agent
pipeline {
    agent none
    stages {
        stage('Build on Linux') {
            agent { label 'linux' }
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Test on Windows') {
            agent { label 'windows' }
            steps {
                bat 'mvn test'
            }
        }
    }
}
```

**stages 指令**

包含一个或多个 stage，每个 stage 包含一组步骤。

```groovy
pipeline {
    agent any
    stages {
        stage('Parallel Stages') {
            parallel {
                stage('Unit Test') {
                    steps {
                        sh 'mvn test'
                    }
                }
                stage('Integration Test') {
                    steps {
                        sh 'mvn verify'
                    }
                }
            }
        }
    }
}
```

**steps 指令**

每个 stage 中的具体步骤。

```groovy
stage('Steps demo') {
    steps {
        // Shell 命令
        sh 'echo hello'

        // 并行文件操作
        dir('subdir') {
            sh 'ls -la'
        }

        // 条件执行（需要 script 块包裹）
        script {
            if (env.BRANCH_NAME == 'main') {
                echo 'Deploying to production'
            }
        }
    }
}
```

**post 指令**

按构建结果执行后置操作。

| 条件 | 说明 |
|------|------|
| `always` | 总是执行 |
| `success` | 构建成功时 |
| `failure` | 构建失败时 |
| `unstable` | 构建不稳定时（如测试部分失败） |
| `changed` | 构建状态改变时（上次成功这次失败，或反之） |
| `fixed` | 上次失败这次成功 |
| `regression` | 上次成功或不稳定，这次失败 |
| `aborted` | 构建被中止 |
| `cleanup` | 在其他所有 post 条件之后执行 |

**options 指令**

Pipeline 的可选配置。

```groovy
pipeline {
    agent any
    options {
        // 构建超时：60 分钟后中止
        timeout(time: 60, unit: 'MINUTES')

        // 构建重试：失败后重试 2 次
        retry(2)

        // 保留最近 10 次构建
        buildDiscarder(logRotator(numToKeepStr: '10'))

        // 禁止并发执行
        disableConcurrentBuilds()

        // 跳过 stage 的默认 checkout
        skipDefaultCheckout()

        // 跳过构建状态检查点
        skipStagesAfterUnstable()

        // 设置 Pipeline 计时器
        timestamps()

        // 设置构建名称
        buildName('${JOB_NAME}-${BUILD_NUMBER}')

        // 设置 Git 连接超时
        checkoutToSubdirectory('src')

        // 并行阶段数量上限
        parallelsAlwaysFailFast()
    }
    // ...
}
```

**parameters 指令**

```groovy
pipeline {
    agent any
    parameters {
        string(name: 'BRANCH', defaultValue: 'develop', description: 'Git branch to build')
        choice(name: 'ENV', choices: ['dev', 'test', 'staging', 'prod'], description: 'Target environment')
        booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip unit tests')
        text(name: 'DEPLOY_NOTES', defaultValue: '', description: 'Deployment notes')
        password(name: 'API_TOKEN', description: 'API token for deployment')
    }
    stages {
        stage('Build') {
            steps {
                echo "Building branch: ${params.BRANCH}"
                echo "Target env: ${params.ENV}"
                echo "Skip tests: ${params.SKIP_TESTS}"
            }
        }
    }
}
```

**triggers 指令**

```groovy
pipeline {
    agent any
    triggers {
        // 定时触发（Cron 语法）
        cron('H */4 * * *')

        // SCM 轮询
        pollSCM('H/5 * * * *')

        // 上游构建触发
        upstream(
            upstreamProjects: 'some-other-job',
            threshold: hudson.model.Result.SUCCESS
        )
    }
    // ...
}
```

**environment 指令**

定义 Pipeline 或 stage 级别的环境变量。

```groovy
pipeline {
    agent any
    environment {
        // 全局环境变量
        MAVEN_OPTS = '-Xmx2048m'
        JAVA_HOME = '/usr/lib/jvm/java-17-openjdk'
    }
    stages {
        stage('Build') {
            environment {
                // Stage 级别环境变量（覆盖全局）
                MAVEN_OPTS = '-Xmx1024m'
                SONAR_TOKEN = credentials('sonar-token')
            }
            steps {
                sh 'echo ${MAVEN_OPTS}'
                sh 'echo ${SONAR_TOKEN}'
            }
        }
    }
}
```

**tools 指令**

指定 Pipeline 需要的工具（需先在全局工具中配置）。

```groovy
pipeline {
    agent any
    tools {
        maven 'Maven3'
        jdk 'JDK17'
        gradle 'Gradle8'
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn --version'
                sh 'java -version'
                sh 'gradle --version'
            }
        }
    }
}
```

**when 指令**

条件执行 stage。

```groovy
pipeline {
    agent any
    stages {
        stage('Deploy to Prod') {
            when {
                // 仅在 main 分支执行
                branch 'main'

                // 表达式条件
                expression {
                    return env.BUILD_NUMBER > '100'
                }

                // 特定构建参数值
                environment name: 'ENV', value: 'prod'

                // 所有条件都满足
                allOf {
                    branch 'main'
                    expression { return true }
                }

                // 任一条件满足
                anyOf {
                    branch 'main'
                    branch 'release/*'
                }

                // 条件取反
                not { branch 'main' }

                // 构建原因（由某个 Job 触发）
                triggeredBy 'TimerTrigger'
                triggeredBy cause: 'UserIdCause', detail: 'admin'
                triggeredBy 'UpstreamCause'
            }
            steps {
                echo 'Deploying to production...'
            }
        }
    }
}
```

# 3. 脚本式 Pipeline（Scripted）

## 3.1 基本结构

```groovy
node {
    stage('Checkout') {
        checkout scm
    }

    stage('Build') {
        // 条件逻辑
        if (env.BRANCH_NAME == 'main') {
            sh './gradlew build -Pprod'
        } else {
            sh './gradlew build'
        }
    }

    stage('Test') {
        sh './gradlew test'
    }

    stage('Deploy') {
        // 异常捕获
        try {
            sh './deploy.sh'
        } catch (Exception e) {
            echo "Deploy failed: ${e.message}"
            currentBuild.result = 'FAILURE'
        } finally {
            echo 'Cleanup...'
        }
    }

    stage('Parallel') {
        // 并行执行
        parallel(
            unitTest: {
                sh './gradlew test'
            },
            integrationTest: {
                sh './gradlew integrationTest'
            }
        )
    }
}
```

## 3.2 Groovy 控制结构

脚本式 Pipeline 直接使用 Groovy 语法，支持完整的编程控制。

**条件语句**：

```groovy
node {
    stage('Conditional') {
        if (env.BRANCH_NAME ==~ /release\/.*/) {
            echo 'This is a release branch'
        } else if (env.BRANCH_NAME == 'main') {
            echo 'Main branch'
        } else {
            echo 'Feature branch'
        }

        // switch 语句
        switch (env.BRANCH_NAME) {
            case 'main':
                echo 'Production'
                break
            case 'develop':
                echo 'Development'
                break
            default:
                echo 'Feature'
        }
    }
}
```

**循环语句**：

```groovy
node {
    stage('Loop') {
        // for 循环
        def projects = ['service-a', 'service-b', 'service-c']
        for (project in projects) {
            echo "Building ${project}..."
        }

        // each 循环
        projects.each { project ->
            echo "Testing ${project}..."
        }

        // while 循环
        def retries = 0
        while (retries < 3) {
            try {
                sh 'curl -f http://health-endpoint'
                break
            } catch (Exception e) {
                retries++
                echo "Retry ${retries}/3"
                sleep 5
            }
        }
    }
}
```

**异常处理**：

```groovy
node {
    stage('Error Handling') {
        try {
            sh './build.sh'
        } catch (Exception err) {
            echo "Caught exception: ${err}"
            currentBuild.result = 'FAILURE'
            // 重新抛出会标记构建失败但不会执行后续 post
            // throw err
        } finally {
            echo 'Always runs - cleanup'
        }

        // 超时
        timeout(time: 30, unit: 'MINUTES') {
            sh './long-running-task.sh'
        }

        // 重试
        retry(3) {
            sh './unstable-task.sh'
        }
    }
}
```

# 4. 声明式 vs 脚本式

| 维度 | 声明式 Pipeline | 脚本式 Pipeline |
|------|----------------|----------------|
| 语法复杂度 | 低（结构化 DSL） | 高（完整 Groovy） |
| 可读性 | 高（声明式清晰） | 中等 |
| 灵活性 | 受限（需 script {} 突破） | 极高（完整编程能力） |
| 代码审查 | 友好 | 较复杂 |
| 错误提示 | 好（语法检查） | 一般（运行时错误） |
| 并行支持 | 原生（parallel 指令） | 原生 |
| 条件执行 | when 指令 | if/else |
| 循环 | 受限（需 script {}） | 原生 |
| 学习曲线 | 平缓 | 陡峭 |
| 推荐程度 | **首选** | 复杂场景使用 |

**选择指南**：
- 新项目：优先使用声明式 Pipeline
- 需要复杂逻辑（循环、动态 stage、深度条件分支）：用脚本式
- 混合使用：声明式中用 `script {}` 嵌入 Groovy 逻辑

```groovy
// 推荐：声明式中嵌入必要脚本
pipeline {
    agent any
    stages {
        stage('Dynamic Build') {
            steps {
                script {
                    // 动态决定构建参数
                    def envMap = ['dev': 'dev', 'test': 'test', 'prod': 'prod']
                    envMap.each { key, value ->
                        echo "Deploying to ${key}..."
                    }
                }
            }
        }
    }
}
```

# 5. 常用内置步骤

## 5.1 文件操作

```groovy
// 写文件
writeFile file: 'output.txt', text: 'Hello from Jenkins'

// 读文件
def content = readFile file: 'output.txt', encoding: 'UTF-8'

// 读 JSON
def data = readJSON file: 'data.json'

// 写 JSON
writeJSON file: 'output.json', json: [name: 'test', version: 1.0]

// 读 YAML（需 YamlAxis 插件）
def config = readYaml file: 'application.yml'

// 删除目录
deleteDir()

// 创建目录
dir('build') {
    sh 'touch output.jar'
}

// 文件是否存在
fileExists 'pom.xml'

// 文件查找
def jars = findFiles(glob: 'target/*.jar')
```

## 5.2 SCM 操作

```groovy
// 检出代码
checkout([
    $class: 'GitSCM',
    branches: [[name: '*/main']],
    doGenerateSubmoduleConfigurations: false,
    extensions: [[$class: 'CloneOption',
        depth: 1,
        shallow: true,
        noTags: true,
        timeout: 10]],
    submoduleCfg: [],
    userRemoteConfigs: [[
        url: 'https://github.com/myorg/repo.git',
        credentialsId: 'github-creds'
    ]]
])

// 简化调用（多分支 Pipeline 默认提供 scm 变量）
checkout scm

// Git 操作（需 Git 插件）
git branch: 'main',
    url: 'https://github.com/myorg/repo.git',
    credentialsId: 'github-creds',
    changelog: true,
    poll: true
```

## 5.3 构建工具步骤

```groovy
// Maven
sh 'mvn clean package -DskipTests=false'

// 或使用 Maven 步骤（需 Maven 插件）
// 注意：建议直接使用 sh 替代

// Gradle
sh './gradlew clean build'

// NPM
sh 'npm ci && npm run build'

// Docker
sh '''
    docker build -t myapp:latest .
    docker push registry.company.com/myapp:latest
'''
```

## 5.4 构建状态和结果

```groovy
// 设置构建结果（不影响后续 stage 执行）
currentBuild.result = 'UNSTABLE'

// 中止构建
error('Something went wrong, aborting')

// 设置构建显示名称
currentBuild.displayName = "#${BUILD_NUMBER}-${BRANCH}"

// 设置构建描述
currentBuild.description = "Deploy to ${ENV}"

// 获取构建变量
echo "Build ID: ${BUILD_ID}"
echo "Build Number: ${BUILD_NUMBER}"
echo "Job Name: ${JOB_NAME}"
echo "Workspace: ${WORKSPACE}"
echo "Jenkins Home: ${JENKINS_HOME}"
echo "Build URL: ${BUILD_URL}"
```

## 5.5 输入和审批

```groovy
// 等待用户输入
stage('Approval') {
    steps {
        input message: 'Deploy to production?',
              ok: 'Deploy',
              parameters: [
                  string(defaultValue: 'main', description: 'Branch', name: 'BRANCH'),
                  choice(choices: ['prod', 'staging'], description: 'Env', name: 'ENV')
              ],
              submitter: 'admin,release-manager',
              submitterParameter: 'APPROVER'
    }
}
```

## 5.6 存根/时间戳

```groovy
// 暂停
sleep time: 30, unit: 'SECONDS'

// 带时间戳的输出
timestamps {
    sh 'sleep 10'
    echo 'This output has timestamp'
}

// 超时
timeout(time: 5, unit: 'MINUTES') {
    sh './maybe-hanging-task.sh'
}
```

## 5.7 凭据使用

```groovy
// 用户名密码
withCredentials([usernamePassword(
    credentialsId: 'git-creds',
    usernameVariable: 'GIT_USER',
    passwordVariable: 'GIT_PASS'
)]) {
    sh 'git clone https://$GIT_USER:$GIT_PASS@github.com/myorg/repo.git'
}

// SSH Key
sshagent(['git-ssh-key']) {
    sh 'git clone git@github.com:myorg/repo.git'
}

// Secret 文本
withCredentials([string(
    credentialsId: 'slack-token',
    variable: 'SLACK_TOKEN'
)]) {
    sh 'curl -H "Authorization: Bearer $SLACK_TOKEN" ...'
}

// Secret 文件
withCredentials([file(
    credentialsId: 'kube-config',
    variable: 'KUBECONFIG'
)]) {
    sh 'kubectl --kubeconfig=$KUBECONFIG apply -f deploy.yaml'
}
```

# 6. Pipeline 语法生成器

Jenkins 内置 **Snippet Generator**（片段生成器），帮助生成步骤代码：

```
访问路径：Jenkins URL > <任意 Pipeline Job> > Pipeline Syntax
功能：
1. Sample Step：选择步骤 → 填写参数 → 生成 Groovy 代码
2. Declarative Directive Generator：生成声明式指令
3. Global Variable Reference：查看全局变量文档
4. Online Pipeline Syntax：在线参考文档
```

# 7. Pipeline 执行模型

## 7.1 执行流程

```
1. Jenkins 解析 Jenkinsfile → 构建 Pipeline 模型
2. 分配 Agent → 创建工作空间
3. 顺序执行 stages
4. 每个 stage 中顺序执行 steps
5. 完成后执行 post 块
6. 收集结果、归档、通知
```

## 7.2 流水线暂停与恢复

Jenkins Pipeline 使用"飞行中"（flyweight）持久化机制：

- Pipeline 状态序列化到磁盘（`JENKINS_HOME/jobs/<job>/builds/<build>/workflow/`）
- 当 Master 重启后，Pipeline 从上次保存的检查点恢复
- `input` 步骤是常见的暂停点

# 8. 流水线可视化

## 8.1 Stage View

传统 UI 中的阶段视图：

```
┌─────────────────────────────────────────────────┐
│ Pipeline Job                                    │
├─────────────────────────────────────────────────┤
│ Stage View                                      │
│ ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐        │
│ │ Check │ │ Build │ │ Test  │ │ Deploy│        │
│ │ Out   │ │       │ │       │ │       │        │
│ ├───────┤ ├───────┤ ├───────┤ ├───────┤        │
│ │  ✓    │ │  ✓    │ │  ✓    │ │  ⏳   │        │
│ │ 12s   │ │ 45s   │ │ 30s   │ │ 5s   │        │
│ └───────┘ └───────┘ └───────┘ └───────┘        │
│ 所有构建历史列表演示每次构建各阶段的耗时和状态      │
└─────────────────────────────────────────────────┘
```

## 8.2 Blue Ocean

Blue Ocean 提供现代化的 Pipeline 可视化界面：

```
┌─────────────────────────────────────────────┐
│ Blue Ocean Dashboard                        │
│                                             │
│ ┌──────────────────────────────────────┐    │
│ │ Pipeline Run #128                     │    │
│ │                                      │    │
│ │ [Checkout] → [Build] → [Test] → [..]│    │
│ │   ● Commit abc123   ● PASS          │    │
│ │   ● Branch: main    ● 45s           │    │
│ │                                      │    │
│ │           [Log] [Replay]             │    │
│ └──────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
```
