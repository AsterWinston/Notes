
# 1. 什么是 JCasC

Jenkins Configuration as Code（JCasC）是一种以 YAML 文件声明 Jenkins 配置的方法，使 Jenkins 的配置可以像代码一样进行版本管理、审查和自动化部署。

核心目标：**将 Jenkins 配置从点击 UI 转向声明式 YAML 文件管理**。

```
传统方式（UI 配置）：
├── 手动点击 Manage Jenkins > Configure System
├── 填写表单
├── 保存（存储在 config.xml 中）
└── ❌ 不可审查、不可回滚、不可自动化

JCasC 方式（代码配置）：
├── 编写 YAML 配置文件
├── 提交到 Git 仓库
├── Jenkins 启动时自动加载
└── ✅ 可审查、可版本管理、可自动化
```

# 2. 安装与启用

## 2.1 插件安装

```
必需插件：
├── Configuration as Code Plugin（核心）
├── Configuration as Code Support Plugin（辅助）
└── credentials-binding（凭据绑定）

推荐配套：
├── Job DSL Plugin（Job 定义）
├── Pipeline Plugin
├── Git Plugin
└── Role-based Authorization Strategy Plugin
```

## 2.2 配置加载

JCasC 默认从以下位置加载配置：

```yaml
# 加载顺序（优先级从高到低）
1. JENKINS_HOME/jenkins.yaml
2. JENKINS_HOME/jenkins.yml
3. JENKINS_ROOT/jenkins.yaml （WAR 包所在目录）
4. 环境变量 CASC_JENKINS_CONFIG 指定的路径
   - 可以是文件、目录或 URL
   - 多个配置用逗号分隔
```

**环境变量方式**：

```bash
# 指定配置文件路径
export CASC_JENKINS_CONFIG=/var/jenkins_home/casc_configs/jenkins.yaml

# 多个配置文件
export CASC_JENKINS_CONFIG=/config/jenkins.yaml,/config/plugins.yaml

# 从 URL 加载
export CASC_JENKINS_CONFIG=https://git.company.com/jenkins-config.yaml

# Docker 方式
docker run -d \
  -p 8080:8080 \
  -v jenkins_home:/var/jenkins_home \
  -v /path/to/casc_configs:/var/jenkins_home/casc_configs \
  -e CASC_JENKINS_CONFIG=/var/jenkins_home/casc_configs/jenkins.yaml \
  jenkins/jenkins:lts-jdk17
```

# 3. YCasC YAML 配置结构

## 3.1 完整配置结构

```yaml
# JCasC 根配置结构
jenkins:
  systemMessage: "Jenkins configured by JCasC"
  numExecutors: 0
  mode: NORMAL
  scmCheckoutRetryCount: 3
  views:
    - list:
        name: "All"
  primaryView:
    all:
      name: "All"
  globalNodeProperties:
    - envVars:
        env:
          - key: JAVA_HOME
            value: /usr/lib/jvm/java-17-openjdk
          - key: MAVEN_HOME
            value: /opt/apache-maven-3.9.6

tool:
  jdk:
    installations:
      - name: "JDK17"
        properties:
          - installSource:
              installers:
                - adoptium.net:
                    id: "jdk-17.0.9+9"
  maven:
    installations:
      - name: "Maven3"
        properties:
          - installSource:
              installers:
                - maven:
                    id: "3.9.6"
  git:
    installations:
      - name: "Default"
        home: "/usr/bin/git"

unclassified:
  location:
    url: "https://jenkins.company.com/"
    adminAddress: "admin@company.com"
  scmGit:
    globalConfigEmail: "jenkins@company.com"
    globalConfigName: "Jenkins CI"
  mailer:
    charset: "UTF-8"
    smtpHost: "smtp.company.com"
    smtpPort: 587
    useSsl: false
    useTls: true
    auth:
      username: "jenkins@company.com"
      password: "${SMTP_PASSWORD}"

credentials:
  system:
    domainCredentials:
      - credentials:
          - usernamePassword:
              scope: GLOBAL
              id: "github-credentials"
              username: "jenkins-ci"
              password: "${GITHUB_TOKEN}"
              description: "GitHub Authentication"
          - string:
              scope: GLOBAL
              id: "sonar-token"
              secret: "${SONAR_TOKEN}"
              description: "SonarQube Token"
          - file:
              scope: GLOBAL
              id: "kubeconfig-prod"
              fileName: "kubeconfig"
              secretBytes: "${KUBECONFIG_BASE64}"
              description: "Production K8s Config"

security:
  globalJobDslSecurityConfiguration:
    useScriptSecurity: false
  remotingCLI:
    enabled: false
  git:
    globalConfigEmail: "jenkins@company.com"

# 插件特定配置
sonar:
  servers:
    - name: "SonarQube"
      serverUrl: "https://sonar.company.com"
      token: "${SONAR_TOKEN}"
      serverAuthenticationToken: "sonar-token"

slack:
  configuredTeams:
    - name: "Company"
      teamDomain: "company"
      tokenCredentialId: "slack-token"
      botUser: true
```

## 3.2 分段配置

可以将 JCasC 配置拆分为多个文件（通过目录方式组织）：

```
casc_configs/
├── jenkins.yaml          # 主入口（引用其他片段）
├── credentials.yaml      # 凭据配置
├── security.yaml         # 安全配置
├── tools.yaml           # 工具配置
├── plugins.yaml         # 插件配置
├── jobs.yaml            # Job 配置
└── notifications.yaml   # 通知配置
```

**主配置引用方式**：

```yaml
# jenkins.yaml
jenkins:
  systemMessage: "JCasC Managed Jenkins"

# 在 Config 目录下，JCasC 会自动合并所有 YAML 文件
# 通过环境变量指定目录
export CASC_JENKINS_CONFIG=/var/jenkins_home/casc_configs
```

# 4. 凭据管理（Credentials）

```yaml
# credentials.yaml
credentials:
  system:
    domainCredentials:
      # 默认域
      - credentials:
          - usernamePassword:
              scope: GLOBAL
              id: "github-creds"
              username: "jenkins-bot"
              password: "${GITHUB_TOKEN}"

          - string:
              scope: GLOBAL
              id: "docker-registry-token"
              secret: "${DOCKER_REGISTRY_PASSWORD}"

          - file:
              scope: GLOBAL
              id: "ssh-key-deploy"
              fileName: "deploy-key"
              secretBytes: "${SSH_PRIVATE_KEY_BASE64}"

          - basicSSHUserPrivateKey:
              scope: GLOBAL
              id: "git-ssh-key"
              username: "git"
              privateKeySource:
                directEntry:
                  privateKey: "${SSH_PRIVATE_KEY}"
              passphrase: "${SSH_KEY_PASSPHRASE}"

      # 特定域下的凭据
      - domain:
          name: "gitlab.company.com"
          specifications:
            - hostnameSpecification:
                includes: "gitlab.company.com"
        credentials:
          - gitLabApiTokenImpl:
              id: "gitlab-token"
              apiToken: "${GITLAB_TOKEN}"
              description: "GitLab API Token"
```

# 5. 安全配置（Security）

```yaml
# security.yaml
jenkins:
  securityRealm:
    # LDAP 认证
    ldap:
      configurations:
        - server: "ldap.company.com"
          port: 389
          rootDN: "dc=company,dc=com"
          userSearchBase: "ou=people"
          userSearchFilter: "uid={0}"
          groupSearchBase: "ou=groups"
          managerDN: "cn=admin,dc=company,dc=com"
          managerPasswordSecret: "${LDAP_PASSWORD}"
          displayNameAttributeName: "displayName"
          mailAddressAttributeName: "mail"
      cache:
        size: 200
        ttl: 300
      userIdStrategy: "CaseInsensitive"
      groupIdStrategy: "CaseInsensitive"

  authorizationStrategy:
    # Role-Based 授权
    roleBased:
      roles:
        global:
          - name: "admin"
            permissions:
              - "Overall/Administer"
              - "Overall/Read"
              - "Job/Configure"
              - "Job/Create"
              - "Job/Delete"
              - "Job/Build"
              - "Job/Cancel"
              - "Credential/All"
              - "Agent/Configure"
              - "Agent/Delete"
              - "Agent/Create"
              - "Run/Delete"
            assignments:
              - "admin"
              - "jenkins-admin-group"
          - name: "developer"
            permissions:
              - "Overall/Read"
              - "Job/Read"
              - "Job/Build"
              - "Job/Cancel"
              - "Job/Workspace"
              - "View/Read"
            assignments:
              - "developers"
              - "authenticated"
          - name: "readonly"
            permissions:
              - "Overall/Read"
              - "Job/Read"
              - "View/Read"
            assignments:
              - "anonymous"
        folders:
          - name: "team-a"
            permissions:
              - "Job/Build"
              - "Job/Read"
            assignments:
              - "team-a-members"

  # 禁用 CLI over Remoting
  remotingCLI:
    enabled: false

  # CSRF 保护
  crumbIssuer:
    standard:
      excludeClientIPFromCrumb: true
```

# 6. 安全凭据（Sensitive Credentials）

JCasC 支持多种方式处理敏感信息：

## 6.1 环境变量引用

```yaml
credentials:
  system:
    domainCredentials:
      - credentials:
          - string:
              id: "api-token"
              secret: "${API_TOKEN}"  # 从环境变量读取
```

```bash
# 启动时设置环境变量
export API_TOKEN="my-secret-token"
```

## 6.2 文件引用

```yaml
credentials:
  system:
    domainCredentials:
      - credentials:
          - file:
              id: "kubeconfig"
              fileName: "kubeconfig"
              secretBytes: "${KUBECONFIG_BASE64}"  # Base64 编码的内容
```

```bash
# 从文件读取 Base64 内容
export KUBECONFIG_BASE64=$(base64 -w0 ~/.kube/config)
```

## 6.3 Kubernetes Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: jenkins-credentials
  namespace: jenkins
type: Opaque
stringData:
  GITHUB_TOKEN: "ghp_xxxxx"
  SONAR_TOKEN: "sqa_xxxxx"
  SMTP_PASSWORD: "email-password"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
spec:
  template:
    spec:
      containers:
      - name: jenkins
        envFrom:
        - secretRef:
            name: jenkins-credentials
```

## 6.4 HashiCorp Vault 集成

```yaml
# JCasC 配合 Vault 插件
jenkins:
  systemMessage: "Using Vault for credentials"
```

```groovy
// Pipeline 中使用 Vault
pipeline {
    agent any
    stages {
        stage('Get Secret from Vault') {
            steps {
                withVault(
                    configuration: [
                        vaultUrl: 'https://vault.company.com',
                        vaultCredentialId: 'vault-token',
                        engineVersion: 2
                    ],
                    vaultSecrets: [
                        [
                            path: 'secret/data/jenkins',
                            engineVersion: 2,
                            secretValues: [
                                [envVar: 'DEPLOY_TOKEN', vaultKey: 'deploy_token']
                            ]
                        ]
                    ]
                ) {
                    sh 'echo $DEPLOY_TOKEN'
                }
            }
        }
    }
}
```

# 7. Job DSL + JCasC

## 7.1 Pipeline Job 定义

```yaml
# jobs.yaml
jobs:
  - script: >
      pipelineJob('my-pipeline') {
        definition {
          cpsScm {
            scm {
              git {
                remote {
                  url('https://github.com/myorg/myrepo.git')
                  credentials('github-creds')
                }
                branch('*/main')
              }
            }
            scriptPath('Jenkinsfile')
            lightweight(true)
          }
        }
        triggers {
          cron('H/5 * * * *')
        }
        parameters {
          stringParam('ENV', 'dev', 'Environment')
        }
        properties {
          buildDiscarder {
            strategy {
              logRotator {
                numToKeepStr('20')
                artifactNumToKeepStr('5')
              }
            }
          }
        }
      }

  - script: >
      multibranchPipelineJob('my-multibranch-pipeline') {
        branchSources {
          git {
            remote('https://github.com/myorg/myrepo.git')
            credentialsId('github-creds')
            includes('*')
            excludes('release/*')
          }
        }
        orphanedItemStrategy {
          discardOldItems {
            numToKeepStr('10')
          }
        }
        triggers {
          periodicFolderTrigger {
            interval('1m')
          }
        }
      }
```

## 7.2 Folder 定义

```yaml
# folder 定义
jobs:
  - script: >
      folder('services') {
        description('Microservices CI Pipeline')
        displayName('Services')
      }

  - script: >
      multibranchPipelineJob('services/user-service') {
        branchSources {
          git {
            remote('https://github.com/myorg/user-service.git')
            credentialsId('github-creds')
          }
        }
      }

  - script: >
      multibranchPipelineJob('services/order-service') {
        branchSources {
          git {
            remote('https://github.com/myorg/order-service.git')
            credentialsId('github-creds')
          }
        }
      }
```

# 8. 插件管理

```yaml
# plugins.yaml
plugins:
  # 必需插件
  - artifactId: "configuration-as-code"
    source:
      version: "latest"
  - artifactId: "workflow-aggregator"
  - artifactId: "pipeline-model-definition"
  - artifactId: "git"
  - artifactId: "credentials"
  - artifactId: "role-strategy"

  # CI/CD 插件
  - artifactId: "docker-workflow"
  - artifactId: "kubernetes"
  - artifactId: "maven-plugin"
  - artifactId: "gradle-plugin"
  - artifactId: "sonar"

  # 通知插件
  - artifactId: "email-ext"
  - artifactId: "slack"

  # UI 插件
  - artifactId: "blueocean"

  # 指定版本
  - artifactId: "metrics"
    source:
      version: "4.2.0-27.ga66f0868610d"
```

# 9. 全局配置示例

## 9.1 完整配置示例

```yaml
# jenkins.yaml
jenkins:
  systemMessage: "Managed by JCasC - Do not edit via UI"
  numExecutors: 0
  mode: NORMAL
  scmCheckoutRetryCount: 3
  disabledAdministrativeMonitors:
    - "jenkins.security.UpdateSiteWarningsMonitor"

  clouds:
    - kubernetes:
        name: "kubernetes"
        serverUrl: "https://kubernetes.default.svc"
        skipTlsVerify: true
        namespace: "jenkins"
        jenkinsUrl: "http://jenkins:8080"
        webSocket: true
        maxRequestsPerHost: 100
        containerCapStr: "50"
        templates:
          - name: "default-agent"
            label: "k8s-agent"
            nodeUsageMode: "NORMAL"
            containers:
              - name: "jnlp"
                image: "jenkins/inbound-agent:latest-jdk17"
                workingDir: "/home/jenkins/agent"
                resourceRequestCpu: "500m"
                resourceRequestMemory: "512Mi"
                resourceLimitCpu: "2"
                resourceLimitMemory: "1Gi"
            podRetention: "never"
            idleMinutes: 5
            activeDeadlineSeconds: 1800

tool:
  jdk:
    installations:
      - name: "JDK17"
        home: "/usr/lib/jvm/java-17-openjdk"
  maven:
    installations:
      - name: "Maven3"
        home: "/opt/apache-maven-3.9.6"
  docker:
    installations:
      - name: "docker"
        dockerHost: "unix:///var/run/docker.sock"

unclassified:
  location:
    url: "https://jenkins.company.com/"
  mailer:
    charset: "UTF-8"
    smtpHost: "smtp.company.com"
    smtpPort: 587
    useTls: true
    auth:
      username: "jenkins@company.com"
      password: "${SMTP_PASSWORD}"
  shell:
    shell: "/bin/bash"
  scmGit:
    globalConfigEmail: "jenkins@company.com"
    globalConfigName: "Jenkins CI"
  sonarGlobalConfiguration:
    buildWrapperEnabled: false
    installations:
      - name: "SonarQube"
        serverUrl: "https://sonar.company.com"
        credentialsId: "sonar-token"

security:
  remotingCLI:
    enabled: false
  globalJobDslSecurityConfiguration:
    useScriptSecurity: false

credentials:
  system:
    domainCredentials:
      - credentials:
          - usernamePassword:
              scope: GLOBAL
              id: "github-creds"
              username: "jenkins-bot"
              password: "${GITHUB_TOKEN}"
          - string:
              scope: GLOBAL
              id: "sonar-token"
              secret: "${SONAR_TOKEN}"
          - usernamePassword:
              scope: GLOBAL
              id: "docker-registry"
              username: "jenkins"
              password: "${DOCKER_PASSWORD}"
          - file:
              scope: GLOBAL
              id: "kubeconfig-prod"
              fileName: "kubeconfig"
              secretBytes: "${KUBECONFIG_BASE64}"
```

## 9.2 验证配置

```bash
# 验证 YAML 配置语法
# 使用 JCasC 提供的验证 API
curl -X POST http://localhost:8080/configuration-as-code/checkConfig \
  -u admin:token \
  -H "Content-Type: application/yaml" \
  --data-binary @jenkins.yaml

# 查看当前配置
# http://jenkins:8080/configuration-as-code/view

# 导出当前配置
# http://jenkins:8080/configuration-as-code/export
```

# 10. JCasC 最佳实践

## 10.1 配置管理

```yaml
# 最佳实践清单
1. 版本控制
   ├── YAML 文件存储在 Git 仓库
   ├── 使用 Git Tag 标记版本
   └── 通过 PR/MR 审查配置变更

2. 敏感信息
   ├── 使用 ${ENV_VAR} 引用环境变量
   ├── 使用 Kubernetes Secret 注入
   ├── 或使用 Vault/其他密文管理工具
   └── 永远不要在 YAML 中硬编码密码

3. 配置分层
   ├── base.yaml: 基础配置（所有环境通用）
   ├── dev.yaml: 开发环境扩展
   ├── staging.yaml: 预发布环境扩展
   └── prod.yaml: 生产环境扩展

4. 变更管理
   ├── 修改配置 → 提交 → PR 审查 → 合并
   ├── Jenkins 自动重载配置
   └── 支持回滚到历史版本
```

## 10.2 分支环境配置

```yaml
# base.yaml（所有环境通用）
jenkins:
  numExecutors: 0
  mode: NORMAL

tool:
  jdk:
    installations:
      - name: "JDK17"
        home: "/usr/lib/jvm/java-17-openjdk"
  maven:
    installations:
      - name: "Maven3"
        home: "/opt/apache-maven-3.9.6"
```

```yaml
# prod.yaml（生产环境特定覆盖）
jenkins:
  systemMessage: "PRODUCTION - Do not edit via UI"

unclassified:
  location:
    url: "https://jenkins.prod.company.com/"

credentials:
  system:
    domainCredentials:
      - credentials:
          - string:
              id: "prod-token"
              secret: "${PROD_TOKEN}"
```

## 10.3 重载配置

JCasC 支持在不重启 Jenkins 的情况下重载配置：

```
方式一：通过 UI
Manage Jenkins > Reload Configuration as Code

方式二：通过 API
curl -X POST http://jenkins:8080/configuration-as-code/reload

方式三：通过命令行
java -jar jenkins-cli.jar -s http://localhost:8080 reload-jcasc-configuration
```

## 10.4 配置漂移检测

```bash
# 导出当前运行配置
curl -u admin:token http://jenkins:8080/configuration-as-code/export \
  > current-config.yaml

# 与期望配置对比
diff expected-config.yaml current-config.yaml

# 如果不一致，重新加载
curl -X POST http://jenkins:8080/configuration-as-code/reload \
  -u admin:token
```

## 10.5 持续集成 JCasC 自身

```yaml
# 为 JCasC 配置创建 CI Pipeline
jobs:
  - script: >
      pipelineJob('jcasc-pipeline') {
        definition {
          cpsScm {
            scm {
              git {
                remote {
                  url('https://github.com/myorg/jenkins-config.git')
                  credentials('github-creds')
                }
                branch('*/main')
              }
            }
            scriptPath('ci/Jenkinsfile')
          }
        }
        triggers {
          cron('H H * * *')
        }
      }
```

```groovy
// ci/Jenkinsfile - 验证 JCasC 配置
pipeline {
    agent any
    stages {
        stage('Validate YAML') {
            steps {
                script {
                    // 检查 YAML 语法
                    def files = findFiles(glob: '*.yaml')
                    files.each { file ->
                        sh "python -c \"import yaml; yaml.safe_load(open('${file.path}'))\""
                    }
                }
            }
        }
        stage('Check Breaking Changes') {
            steps {
                // 在测试环境部署并验证
                // ...
            }
        }
    }
}
```
