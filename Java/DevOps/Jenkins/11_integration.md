
# 1. Git 集成

## 1.1 Git 插件配置

Git 插件是 Jenkins 最基础、最常用的插件，支持从 Git 仓库检出代码。

**插件安装**：
```
插件名称：Git Plugin
功能：
  - 支持 git://, https://, ssh:// 协议
  - 支持分支、标签、PR 检出
  - 支持子模块、浅克隆、稀疏检出
  - 提供变更日志
  - 支持 Git LFS
```

**Pipeline 中的 Git**：

```groovy
// 基础检出
stage('Checkout') {
    steps {
        checkout scm  // 多分支 Pipeline 自动使用 scm
    }
}

// 完整 Git 配置
stage('Custom Checkout') {
    steps {
        checkout([
            $class: 'GitSCM',
            branches: [[name: '*/main']],
            doGenerateSubmoduleConfigurations: false,
            extensions: [
                [$class: 'CloneOption', depth: 1, shallow: true, noTags: true],
                [$class: 'CleanBeforeCheckout'],
                [$class: 'SubmoduleOption', recursiveSubmodules: true],
                [$class: 'SparseCheckoutPaths', sparseCheckoutPaths: [[path: 'src/']]]
            ],
            submoduleCfg: [],
            userRemoteConfigs: [[
                url: 'https://github.com/myorg/repo.git',
                credentialsId: 'git-creds'
            ]]
        ])
    }
}

// 多个仓库检出
stage('Multi Repo') {
    steps {
        dir('app') {
            git branch: 'main',
                url: 'https://github.com/myorg/app.git',
                credentialsId: 'git-creds'
        }
        dir('common') {
            git branch: 'main',
                url: 'https://github.com/myorg/common.git',
                credentialsId: 'git-creds'
        }
    }
}
```

## 1.2 GitHub 集成

需要安装 `GitHub Integration Plugin` 或 `GitHub Branch Source Plugin`。

**配置 GitHub 认证**：

```yaml
# Jenkins > Manage Jenkins > Configure System > GitHub
GitHub:
  - GitHub Server:
    - Name: GitHub
    - API URL: https://api.github.com
    - Credentials: github-access-token
    - Manage hooks: true
    - Rate limiting: (默认即可)
```

**GitHub Webhook 自动配置**：

```
GitHub Plugin 支持自动配置 Webhook（需要 Token 有 admin:repo_hook 权限）：
1. 创建 Multibranch Pipeline
2. Branch Sources > Add source > GitHub
3. 选择组织/仓库
4. Jenkins 自动在仓库设置中添加 Webhook
5. 推送代码自动触发构建
```

**GitHub API 集成**：

```groovy
// 在 Pipeline 中与 GitHub API 交互
stage('GitHub Status') {
    steps {
        script {
            // 设置提交状态
            step([
                $class: 'GitHubCommitStatusSetter',
                reposSource: [$class: 'ManuallyEnteredRepositorySource', url: 'https://github.com/myorg/repo'],
                commitShaSource: [$class: 'ManuallyEnteredShaSource', sha: env.GIT_COMMIT],
                errorHandlers: [[$class: 'ShallowAnyErrorHandler']],
                statusResultSource: [
                    $class: 'ConditionalStatusResultSource',
                    results: [
                        [$class: 'AnyBuildResult', state: 'SUCCESS', message: 'Build passed'],
                        [$class: 'AnyBuildResult', state: 'FAILURE', message: 'Build failed']
                    ]
                ]
            ])
        }
    }
}

// 创建发布
stage('Create Release') {
    steps {
        withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
            sh '''
                curl -X POST https://api.github.com/repos/myorg/repo/releases \
                    -H "Authorization: token $GITHUB_TOKEN" \
                    -H "Content-Type: application/json" \
                    -d "{\\"tag_name\\": \\"v${BUILD_NUMBER}\\", \\"name\\": \\"Release v${BUILD_NUMBER}\\"}"
            '''
        }
    }
}
```

## 1.3 GitLab 集成

需要安装 `GitLab Plugin`。

**配置 GitLab 连接**：

```yaml
# Jenkins > Manage Jenkins > Configure System > GitLab
GitLab:
  - Server:
    - Name: GitLab
    - URL: https://gitlab.company.com
    - Credentials: gitlab-api-token
    - Ignore SSL errors: false
```

**GitLab Webhook 配置**：

```
GitLab > 项目 > Settings > Webhooks：
URL: http://jenkins:8080/project/<job-name>
Secret Token: <共享密钥>
Trigger:
  ✅ Push events
  ✅ Merge Request events
  ✅ Tag push events
```

**Pipeline 中的 GitLab 集成**：

```groovy
// Jenkinsfile 中的 GitLab 集成
pipeline {
    agent any
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
        stage('GitLab Status') {
            steps {
                // 更新 MR 中的状态
                updateGitlabCommitStatus name: 'build', state: 'pending'
            }
        }
    }
    post {
        success {
            updateGitlabCommitStatus name: 'build', state: 'success'
        }
        failure {
            updateGitlabCommitStatus name: 'build', state: 'failed'
        }
    }
}
```

# 2. Maven 集成

## 2.1 Maven 配置

```yaml
# Jenkins 全局工具配置
工具名称: Maven3
安装方式: Install automatically
版本: Apache Maven 3.9.6

# 或手动安装
工具名称: Maven3-Local
MAVEN_HOME: /opt/apache-maven-3.9.6
```

## 2.2 Maven Settings 配置

```groovy
// Pipeline 中使用自定义 Maven Settings
stage('Maven Build') {
    steps {
        withMaven(
            maven: 'Maven3',
            jdk: 'JDK17',
            mavenSettingsConfig: 'maven-settings-id',  // 配置文件 ID
            mavenLocalRepo: '.repository',
            options: [
                globals: [
                    artifactsPublisher: [disabled: true]
                ]
            ]
        ) {
            sh 'mvn clean package'
        }
    }
}
```

## 2.3 Maven 多模块构建

```groovy
// 多模块 Maven 项目构建
stage('Multi Module Build') {
    steps {
        // 构建所有模块
        sh 'mvn clean package -pl . -am'

        // 构建特定模块
        sh 'mvn clean package -pl module-a -am'

        // 跳过指定模块
        sh 'mvn clean package -pl "!module-skip"'
    }
}
```

## 2.4 Nexus/Artifactory 集成

```groovy
// Maven 部署到 Nexus
stage('Deploy to Nexus') {
    steps {
        withCredentials([usernamePassword(
            credentialsId: 'nexus-creds',
            usernameVariable: 'NEXUS_USER',
            passwordVariable: 'NEXUS_PASS'
        )]) {
            sh """
                mvn deploy \
                    -DaltDeploymentRepository=releases::default::https://nexus.company.com/repository/maven-releases/ \
                    -DrepositoryId=releases \
                    -Dmaven.wagon.http.ssl.insecure=true
            """
        }
    }
}
```

# 3. Docker 集成

## 3.1 Docker Pipeline 插件

需要安装 `Docker Pipeline Plugin`。

```groovy
// 在 Docker 容器中执行 Pipeline
pipeline {
    agent {
        docker {
            image 'maven:3.9-eclipse-temurin-17'
            args '-v /root/.m2:/root/.m2'
            label 'docker-agent'
            registryUrl 'https://registry.company.com'
            registryCredentialsId 'docker-registry-creds'
        }
    }
    stages {
        stage('Build in Docker') {
            steps {
                sh 'mvn clean package'
            }
        }
    }
}
```

## 3.2 Docker 构建与推送

```groovy
// Docker 镜像构建与推送
stage('Docker Build & Push') {
    steps {
        script {
            // 构建 Docker 镜像
            def myImage = docker.build(
                "registry.company.com/myapp:${BUILD_NUMBER}",
                "-f Dockerfile.prod ."
            )

            // 登录并推送
            docker.withRegistry('https://registry.company.com', 'docker-registry-creds') {
                myImage.push()
                myImage.push('latest')
            }
        }
    }
}
```

## 3.3 Docker 多容器构建

```groovy
// 多容器环境（如需要同时运行 MySQL 和 Redis）
pipeline {
    agent none
    stages {
        stage('Integration Test') {
            agent {
                docker {
                    image 'maven:3.9-eclipse-temurin-17'
                    label 'docker-agent'
                    args '--link mysql-container:mysql --link redis-container:redis'
                }
            }
            steps {
                sh 'mvn verify -Pintegration-test'
            }
        }
    }
}
```

# 4. SonarQube 集成

## 4.1 SonarQube 配置

```yaml
# Jenkins > Manage Jenkins > Configure System > SonarQube
SonarQube servers:
  - Name: SonarQube
    Server URL: https://sonar.company.com
    Server authentication token: sonar-token (凭据 ID)
    Webhook URL: (自动填写)
```

## 4.2 Pipeline 集成

```groovy
pipeline {
    agent any
    environment {
        SCANNER_HOME = tool name: 'sonar-scanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        ${SCANNER_HOME}/bin/sonar-scanner \
                            -Dsonar.projectKey=my-project \
                            -Dsonar.projectName=My Project \
                            -Dsonar.sources=. \
                            -Dsonar.java.binaries=target/classes \
                            -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                    '''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
    }
    post {
        success {
            // 更新 SonarQube 状态
            sh 'echo "Quality gate passed"'
        }
        failure {
            sh 'echo "Quality gate failed"'
        }
    }
}
```

## 4.3 Quality Gate

SonarQube Quality Gate 是代码质量的门禁：

```groovy
// Quality Gate 配置示例
stage('Quality Gate') {
    steps {
        script {
            // 等待 SonarQube 处理完成
            def qg = waitForQualityGate()
            if (qg.status != 'OK') {
                error "Quality Gate failed: ${qg.status}"
            }
            echo "Quality Gate passed: ${qg.status}"
        }
    }
}
```

**常见的 Quality Gate 条件**：
- 新增代码覆盖率 < 80%
- 新增 Bug > 0
- 新增漏洞 > 0
- 新增代码异味 > 一定阈值
- 重复率 > 3%

# 5. JUnit 测试报告

## 5.1 配置

```groovy
// JUnit 测试报告集成
stage('Test') {
    steps {
        sh 'mvn test'
    }
    post {
        always {
            // 发布 JUnit 测试报告
            junit testResults: 'target/surefire-reports/*.xml',
                allowEmptyResults: true,
                healthScaleFactor: 1.0,
                keepLongStdio: true,
                testDataPublishers: [
                    // 发布测试结果趋势
                    [$class: 'TestDataPublisher'],
                    // 发布测试结果变更分析
                    [$class: 'ClaimPublisher']
                ]
        }
    }
}
```

## 5.2 测试报告在 Jenkins 中的展示

JUnit 插件可以提供：
```
- 测试结果趋势图
- 失败测试列表
- 测试耗时分析
- 固定测试（从失败变成功）
- 测试结果历史
- 失败的测试详情（堆栈跟踪）
```

## 5.3 其他测试框架集成

```groovy
// pytest
stage('Python Test') {
    steps {
        sh 'pytest tests/ --junitxml=reports/junit.xml'
    }
    post {
        always {
            junit 'reports/junit.xml'
        }
    }
}

// Jest (JavaScript)
stage('Node Test') {
    steps {
        sh 'npx jest --ci --testResultsProcessor=jest-junit'
    }
    post {
        always {
            junit 'reports/junit.xml'
        }
    }
}

// Go test
stage('Go Test') {
    steps {
        sh 'go test -v -coverprofile=coverage.out -json > test-report.json'
    }
    post {
        always {
            junit 'test-report.json'
        }
    }
}
```

# 6. 通知集成

## 6.1 邮件通知（Email Extension）

```groovy
post {
    failure {
        emailext(
            to: 'dev-team@company.com',
            cc: 'tech-lead@company.com',
            bcc: 'build-master@company.com',
            replyTo: 'build-master@company.com',
            subject: "[BUILD FAILED] ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            body: '''
                <html>
                <body>
                    <h2>Build Failed</h2>
                    <p><b>Project:</b> ${env.JOB_NAME}</p>
                    <p><b>Build:</b> #${env.BUILD_NUMBER}</p>
                    <p><b>Branch:</b> ${env.BRANCH_NAME}</p>
                    <p><b>Duration:</b> ${currentBuild.durationString}</p>
                    <p><b>URL:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    <hr/>
                    <h3>Changes since last build:</h3>
                    ${CHANGES_SINCE_LAST_SUCCESS, reverse=true, format="%a: %m%n"}
                    <hr/>
                    <p><em>Auto-generated by Jenkins</em></p>
                </body>
                </html>
            ''',
            mimeType: 'text/html',
            attachmentsPattern: 'target/surefire-reports/*.xml',
            compressLog: true
        )
    }
}
```

## 6.2 Slack 通知

```groovy
// 需要安装 Slack Notification Plugin
post {
    success {
        slackSend(
            channel: '#ci-builds',
            color: 'good',
            message: "✅ *${env.JOB_NAME}* #${env.BUILD_NUMBER} - Success (<${env.BUILD_URL}|Open>)",
            teamDomain: 'company',
            tokenCredentialId: 'slack-token'
        )
    }
    failure {
        slackSend(
            channel: '#ci-builds',
            color: 'danger',
            message: "❌ *${env.JOB_NAME}* #${env.BUILD_NUMBER} - Failed (<${env.BUILD_URL}|Open>)",
            teamDomain: 'company',
            tokenCredentialId: 'slack-token'
        )
    }
    unstable {
        slackSend(
            channel: '#ci-builds',
            color: 'warning',
            message: "⚠️ *${env.JOB_NAME}* #${env.BUILD_NUMBER} - Unstable (<${env.BUILD_URL}|Open>)",
            teamDomain: 'company',
            tokenCredentialId: 'slack-token'
        )
    }
}
```

## 6.3 企业微信/钉钉通知（自定义）

```groovy
post {
    failure {
        script {
            def message = """
                **构建失败通知**
                >项目: ${env.JOB_NAME}
                >构建: #${env.BUILD_NUMBER}
                >分支: ${env.BRANCH_NAME}
                >提交者: ${env.CHANGE_AUTHOR}
                >[查看详情](${env.BUILD_URL})
            """.stripIndent().trim()

            sh """
                curl -X POST https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=${WEBHOOK_KEY} \
                    -H 'Content-Type: application/json' \
                    -d '{
                        "msgtype": "markdown",
                        "markdown": {
                            "content": ${message.toJson()}
                        }
                    }'
            """
        }
    }
}
```

## 6.4 飞书通知

```groovy
post {
    failure {
        script {
            def message = """
                {
                    "msg_type": "post",
                    "content": {
                        "post": {
                            "zh_cn": {
                                "title": "构建失败",
                                "content": [
                                    [{"tag": "text", "text": "项目: ${env.JOB_NAME}\\n"},
                                     {"tag": "text", "text": "构建: #${env.BUILD_NUMBER}\\n"},
                                     {"tag": "a", "text": "查看详情", "href": "${env.BUILD_URL}"}]
                                ]
                            }
                        }
                    }
                }
            """.stripIndent().trim()

            sh """
                curl -X POST https://open.feishu.cn/open-apis/bot/v2/hook/${WEBHOOK_KEY} \
                    -H 'Content-Type: application/json' \
                    -d '${message}'
            """
        }
    }
}
```

# 7. 数据存储集成

## 7.1 Nexus Repository

```groovy
// 从 Nexus 下载构建依赖
stage('Download from Nexus') {
    steps {
        withCredentials([usernamePassword(
            credentialsId: 'nexus-creds',
            usernameVariable: 'NEXUS_USER',
            passwordVariable: 'NEXUS_PASS'
        )]) {
            sh '''
                curl -u "$NEXUS_USER:$NEXUS_PASS" \
                    "https://nexus.company.com/repository/raw-hosted/myapp-${BUILD_NUMBER}.jar" \
                    -o myapp.jar
            '''
        }
    }
}

// 上传文件到 Nexus
stage('Upload to Nexus') {
    steps {
        withCredentials([usernamePassword(
            credentialsId: 'nexus-creds',
            usernameVariable: 'NEXUS_USER',
            passwordVariable: 'NEXUS_PASS'
        )]) {
            sh '''
                curl -u "$NEXUS_USER:$NEXUS_PASS" \
                    --upload-file target/myapp.jar \
                    "https://nexus.company.com/repository/raw-hosted/myapp-${BUILD_NUMBER}.jar"
            '''
        }
    }
}
```

## 7.2 Harbor/私有镜像仓库

```groovy
stage('Push to Harbor') {
    steps {
        withCredentials([usernamePassword(
            credentialsId: 'harbor-creds',
            usernameVariable: 'HARBOR_USER',
            passwordVariable: 'HARBOR_PASS'
        )]) {
            sh """
                docker login harbor.company.com -u $HARBOR_USER -p $HARBOR_PASS
                docker tag myapp:${BUILD_NUMBER} harbor.company.com/library/myapp:${BUILD_NUMBER}
                docker push harbor.company.com/library/myapp:${BUILD_NUMBER}
                docker tag myapp:${BUILD_NUMBER} harbor.company.com/library/myapp:latest
                docker push harbor.company.com/library/myapp:latest
            """
        }
    }
}
```

# 8. 部署工具集成

## 8.1 Ansible 集成

```groovy
stage('Ansible Deploy') {
    steps {
        withCredentials([file(
            credentialsId: 'ansible-vault-password',
            variable: 'ANSIBLE_VAULT_PASSWORD_FILE'
        )]) {
            sh """
                ansible-playbook -i inventories/${params.ENV}/hosts.ini \\
                    playbooks/deploy.yml \\
                    --vault-password-file ${ANSIBLE_VAULT_PASSWORD_FILE} \\
                    --extra-vars "version=${BUILD_NUMBER}"
            """
        }
    }
}
```

## 8.2 Shell 脚本部署

```groovy
stage('Shell Deploy') {
    steps {
        sshagent(['deploy-ssh-key']) {
            sh """
                ssh deploy@${DEPLOY_SERVER} "
                    cd /opt/app &&
                    git pull origin main &&
                    ./deploy.sh ${BUILD_NUMBER}
                "
            """
        }
    }
}
```

## 8.3 SSH Pipeline Steps 集成

需要安装 `SSH Pipeline Steps Plugin`：

```groovy
stage('SSH Deploy') {
    steps {
        script {
            sshRemove remote: remote, path: '/opt/app/old/'
            sshPut remote: remote, from: 'target/myapp.jar', into: '/opt/app/'
            sshCommand remote: remote, command: "systemctl restart myapp"
        }
    }
}
```

# 9. Kubernetes 集成

## 9.1 kubectl 操作

```groovy
stage('K8s Deploy') {
    steps {
        withCredentials([file(
            credentialsId: 'kubeconfig',
            variable: 'KUBECONFIG'
        )]) {
            sh """
                kubectl --kubeconfig=${KUBECONFIG} set image deployment/myapp \\
                    myapp-container=registry.company.com/myapp:${BUILD_NUMBER} \\
                    -n production
                kubectl --kubeconfig=${KUBECONFIG} rollout status deployment/myapp \\
                    -n production
            """
        }
    }
}
```

## 9.2 Helm 部署

```groovy
stage('Helm Deploy') {
    steps {
        withCredentials([file(
            credentialsId: 'kubeconfig',
            variable: 'KUBECONFIG'
        )]) {
            sh """
                helm upgrade --install myapp ./chart \\
                    --namespace ${params.ENV} \\
                    --set image.tag=${BUILD_NUMBER} \\
                    --set replicaCount=3 \\
                    --wait --timeout 5m0s
            """
        }
    }
}
```

# 10. 监控与告警集成

## 10.1 Prometheus 集成

```groovy
// 通过 Jenkins Metrics 插件暴露指标
// 访问：http://jenkins:8080/metrics/prometheus
// Prometheus 配置
scrape_configs:
  - job_name: 'jenkins'
    metrics_path: '/metrics/prometheus'
    static_configs:
      - targets: ['jenkins:8080']
```

## 10.2 Grafana 告警

```groovy
// Pipeline 中触发 Grafana 告警
stage('Performance Check') {
    steps {
        script {
            def response = sh(script: 'curl -s http://perf-server/api/check', returnStdout: true)
            def result = readJSON text: response

            if (result.p95_latency > 200) {
                // 触发告警
                sh """
                    curl -X POST http://grafana:3000/api/annotations \
                        -H "Authorization: Bearer ${GRAFANA_TOKEN}" \
                        -H "Content-Type: application/json" \
                        -d '{
                            "dashboardUID": "performance",
                            "panelId": 1,
                            "text": "P95 latency exceeded threshold: ${result.p95_latency}ms",
                            "tags": ["jenkins", "performance"]
                        }'
                """
            }
        }
    }
}
```
