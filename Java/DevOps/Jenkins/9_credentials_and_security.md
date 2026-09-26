
# 1. Jenkins 安全模型概述

Jenkins 的安全体系分为三个层次：

```
┌──────────────────────────────────────────────┐
│          认证（Authentication）              │
│  确认"你是谁"：用户名密码、LDAP、SSO、Token   │
├──────────────────────────────────────────────┤
│          授权（Authorization）               │
│  确认"你能做什么"：Matrix、Role-Based Strategy│
├──────────────────────────────────────────────┤
│          凭据管理（Credential Management）    │
│  安全存储敏感信息：密码、Token、SSH Key       │
├──────────────────────────────────────────────┤
│          审计日志（Audit Log）                │
│  记录"谁在什么时间做了什么操作"               │
└──────────────────────────────────────────────┘
```

# 2. 认证配置

## 2.1 Jenkins 内置用户数据库

默认安装时使用 Jenkins 内置用户数据库，适合小型团队。

**配置路径**：Manage Jenkins > Security > Security Realm

```
Security Realm:
  - Jenkins' own user database
    - Allow users to sign up: false（生产环境关闭注册）
    - Allow users to delete themselves: false
```

**用户管理**：

```bash
# 通过 UI 管理
1. Manage Jenkins > Users
2. 创建用户 / 修改密码 / 删除用户

# 通过脚本管理
jenkins.model.Jenkins.instance.securityRealm.createAccount('username', 'password')
```

## 2.2 LDAP 集成

企业环境中常通过 LDAP/AD 统一管理用户。

```yaml
Security Realm: LDAP
  Server: ldap.company.com
  Port: 389  # 或 636 (LDAPS)
  Root DN: dc=company,dc=com
  User search base: ou=people
  User search filter: uid={0}  # {0} 替换为输入的用户名
  Group search base: ou=groups
  Group search filter: member={0}
  Manager DN: cn=admin,dc=company,dc=com
  Manager Password: ****
  Display Name LDAP attribute: displayName
  Email Address LDAP attribute: mail
```

## 2.3 SSO 集成（SAML/CAS/OIDC）

通过插件支持单点登录：

```
支持的插件：
- SAML Plugin：使用 SAML 2.0 协议
- CAS Plugin：使用 CAS 协议  
- OpenID Connect Plugin：使用 OIDC 协议

典型配置（以 OIDC 为例）：
  Security Realm: OpenID Connect
  Client ID: jenkins-client
  Client Secret: ****
  Issuer URL: https://auth.company.com
  User name field: preferred_username
  Groups field: groups
```

## 2.4 API Token

API Token 用于外部系统（如 curl 脚本、其他 CI 系统）通过 REST API 认证 Jenkins。

```bash
# 创建 Token（UI）
1. 用户 > 用户设置 > API Token > Add new Token
2. 复制生成的 Token（关闭页面后不再显示）

# 使用 Token 调用 API
curl -u username:api_token http://jenkins:8080/job/myjob/build

# 或使用 Bearer Token
curl -H "Authorization: Bearer api_token" http://jenkins:8080/job/myjob/build
```

**Token 类型**：

| 类型 | 说明 | 有效期 |
|------|------|--------|
| 用户 Token | 用户手动生成 | 永不过期（建议手动轮换） |
| Machine Token | 机器间通信 | 可设置过期时间 |
| Pipeline Token | Pipeline 特定 Token | 按需使用 |

# 3. 授权配置

## 3.1 任何人可做任何事

```yaml
Authorization: Anyone can do anything
# 适用场景：个人开发环境、内网测试环境
# 安全性：无（不进行任何权限检查）
# ⚠️ 生产环境严禁使用
```

## 3.2 登录用户可做任何事

```yaml
Authorization: Logged-in users can do anything
# 适用场景：小型团队内部环境
# 安全性：低（仅认证，无授权粒度）
```

## 3.3 矩阵授权策略

Matrix-based security 提供精细化的权限控制。

```yaml
Authorization: Matrix-based security

# 用户/组权限矩阵
User/Group: admin
  整体: Administer(√), Read(√)
  Job: Create(√), Delete(√), Configure(√), Read(√), Build(√), Cancel(√), Move(√)
  Run: Delete(√), Update(√)
  View: Create(√), Delete(√), Configure(√), Read(√)
  SCM: Tag(√)
  Credentials: Create(√), Delete(√), Update(√), View(√)
  Agent: Configure(√), Delete(√), Create(√), Connect(√), Disconnect(√)

User/Group: developers
  整体: Read(√)
  Job: Read(√), Build(√), Cancel(√), Workspace(√)
  Run: (无)
  View: Read(√)
  SCM: (无)
  Credentials: (无)
  Agent: (无)

User/Group: anonymous
  整体: Read(√)  # 或空
  Job: Read(√), View(√)
  # 其他权限禁用
```

**权限说明**：

| 权限类别 | 权限项 | 说明 |
|---------|--------|------|
| 整体 | Administer | 系统管理权限（超级管理员） |
| 整体 | Read | 查看 Jenkins 整体状态 |
| Job | Create/Delete | 创建和删除 Job |
| Job | Configure | 修改 Job 配置 |
| Job | Read | 查看 Job 详情 |
| Job | Build | 触发构建 |
| Job | Cancel | 中止进行中的构建 |
| Job | Workspace | 查看工作空间 |
| Run | Delete/Update | 删除/更新构建记录 |
| View | Create/Delete/Configure/Read | 视图管理 |
| SCM | Tag | 创建标签 |
| Credentials | Create/Delete/Update/View | 凭据管理 |
| Agent | Configure/Delete/Create/Connect/Disconnect | Agent 管理 |

## 3.4 基于角色的授权（Role-Based Strategy）

推荐企业级使用，需要安装 `Role-based Authorization Strategy` 插件。

**角色类型**：

```
Global Roles（全局角色）：
  - admin: 完全访问
  - developer: 开发者权限
  - viewer: 只读权限
  - operator: 运维操作权限

Project Roles（项目角色）：
  - service-admin: 管理特定服务
  - service-developer: 开发特定服务
  - service-viewer: 查看特定服务

Slave Roles（Agent 角色）：
  - agent-admin: 管理所有 Agent
  - agent-user: 使用特定 Agent
```

**配置示例**：

```yaml
# 全局角色
Global roles:
  - name: admin
    permissions:
      - Overall/Administer
      - Overall/SystemRead
    SID: admin  # 关联用户或组

  - name: developer
    permissions:
      - Overall/Read
      - Job/Build
      - Job/Cancel
      - Job/Read
      - View/Read
    SID: developers  # 允许关联 LDAP 组

  - name: viewer
    permissions:
      - Overall/Read
      - Job/Read
      - View/Read
    SID: anonymous  # 关联匿名用户

# 项目角色（使用正则匹配 Job 名称）
Project roles:
  - name: team-a-developer
    pattern: team-a/.*  # 匹配 team-a 下的所有 Job
    permissions:
      - Job/Build
      - Job/Cancel
      - Job/Read
      - Job/Workspace
      - Run/Update
    SID: team-a-members  # 关联组

  - name: release-manager
    pattern: .*/release/.*  # 匹配发布 Pipeline
    permissions:
      - Job/Build
      - Job/Configure
      - Run/Update
    SID: release-managers
```

## 3.5 文件夹权限

文件夹可以覆盖全局权限，实现细粒度控制。

```yaml
# 文件夹权限配置（右键文件夹 > Configuration > Properties）
Properties:
  - Allow administration and item creation/cancellation in the folder
  - Allow deletion and relocation in the folder
  - Allow subfolder administration and item creation/cancellation in the folder
  - Allow subfolder deletion and relocation in the folder

# 文件夹级别角色
Folder roles:
  - name: folder-admin
    folder: my-project
    permissions:
      - Folder/Read
      - Folder/Configure
      - Job/Configure
      - Job/Build
```

# 4. 凭据管理

## 4.1 凭据加密机制

Jenkins 凭据加密基于对称加密，使用存储在 Master 文件系统中的密钥：

```
┌─────────────────────────────────────────────────┐
│                  加密流程                         │
│                                                   │
│  1. Master 启动时生成 master.key                  │
│     ├─ 位置：JENKINS_HOME/secrets/master.key      │
│     └─ 128位 AES 密钥                            │
│                                                   │
│  2. 使用 master.key 加密 hudson.util.Secret       │
│     ├─ 位置：JENKINS_HOME/secrets/hudson.util.Secret│
│     └─ 用于加密所有凭据                           │
│                                                   │
│  3. 凭据存储时用 Secret 加密                      │
│     ├─ 密文存储在 config.xml 中                   │
│     └─ 示例如下：                                │
│         <password>{AQAAABAAAA...}</password>      │
│                                                   │
│  4. 读取时自动解密                                │
│     └─ 仅在内存中使用，不会写入磁盘               │
└─────────────────────────────────────────────────┘
```

**安全注意事项**：
- `master.key` 和 `hudson.util.Secret` 必须安全备份
- 丢失这些文件将无法解密任何凭据
- 迁移 Jenkins 时必须同时迁移 `secrets/` 目录

## 4.2 凭据类型详解

**Username with password**：

```yaml
# 创建凭据（UI 或 API）
Kind: Username with password
Scope: GLOBAL
Username: deploy-user
Password: ****
ID: deploy-credentials
Description: 部署服务器凭据
```

```groovy
// Pipeline 中使用
withCredentials([usernamePassword(
    credentialsId: 'deploy-credentials',
    usernameVariable: 'DEPLOY_USER',
    passwordVariable: 'DEPLOY_PASS'
)]) {
    sh '''
        curl -u "$DEPLOY_USER:$DEPLOY_PASS" \
            https://deploy-server.company.com/api/deploy
    '''
}
```

**SSH Username with private key**：

```yaml
# 创建凭据
Kind: SSH Username with private key
Scope: GLOBAL
ID: gitlab-ssh-key
Username: git
Private Key:
  - Enter directly: (粘贴私钥内容)
  - From a file on Jenkins master: (私钥文件路径)
  - From the Jenkins master ~/.ssh: (自动使用 Master 的 SSH Key)
Passphrase: (如果私钥有密码)
```

```groovy
// Pipeline 中使用
sshagent(['gitlab-ssh-key']) {
    sh 'git clone git@gitlab.company.com:myorg/myrepo.git'
    sh 'git tag v1.0.0'
    sh 'git push origin v1.0.0'
}

// Pipeline 中使用特定 Key
sshagent(credentials: ['gitlab-ssh-key']) {
    sh 'git fetch origin'
}
```

**Secret text**：

```yaml
# 创建凭据
Kind: Secret text
Scope: GLOBAL
Secret: api-token-abcdef123456
ID: sonar-token
Description: SonarQube 认证 Token
```

```groovy
// Pipeline 中使用
withCredentials([string(
    credentialsId: 'sonar-token',
    variable: 'SONAR_TOKEN'
)]) {
    sh """
        sonar-scanner \
            -Dsonar.login=${SONAR_TOKEN} \
            -Dsonar.host.url=https://sonar.company.com
    """
}
```

**Secret file**：

```yaml
# 创建凭据
Kind: Secret file
Scope: GLOBAL
File: (上传文件，如 kubeconfig、证书、properties 文件)
ID: kubeconfig-prod
Description: 生产环境 K8s 配置
```

```groovy
// Pipeline 中使用
withCredentials([file(
    credentialsId: 'kubeconfig-prod',
    variable: 'KUBECONFIG_FILE'
)]) {
    sh """
        kubectl --kubeconfig=${KUBECONFIG_FILE} \
            apply -f deployment.yaml
    """
}
```

**Certificate**：

```yaml
# 创建凭据
Kind: Certificate
Scope: GLOBAL
Upload PKCS#12 certificate: (上传 .p12 文件)
Password: ****
ID: client-cert
```

## 4.3 凭据提供者

Jenkins 凭据不仅限于 Jenkins 自身，还可以集成外部凭据存储：

```
支持的凭据提供者（需插件）：
├── Jenkins 内置存储（默认）
├── HashiCorp Vault Plugin
│   └── 从 Vault 读取凭据，不在 Jenkins 中存储
├── CyberArk Conjur
├── Azure Key Vault
└── AWS Secrets Manager
```

**Vault 集成示例**：

```groovy
// 使用 HashiCorp Vault Plugin
pipeline {
    agent any
    stages {
        stage('Read from Vault') {
            steps {
                withVault(
                    configuration: [
                        vaultUrl: 'https://vault.company.com',
                        vaultCredentialId: 'vault-token',
                        engineVersion: 2
                    ],
                    vaultSecrets: [
                        [
                            path: 'secret/data/myapp',
                            engineVersion: 2,
                            secretValues: [
                                [envVar: 'DB_PASSWORD', vaultKey: 'db_password']
                            ]
                        ]
                    ]
                ) {
                    sh 'echo $DB_PASSWORD'
                }
            }
        }
    }
}
```

## 4.4 凭据管理最佳实践

```yaml
# 1. 命名规范
ID 命名规则：<用途>-<环境>-<类型>
  示例：
  - github-prod-token
  - gitlab-dev-ssh-key
  - docker-registry-creds
  - sonarqube-token
  - kubeconfig-staging

# 2. 作用域选择
GLOBAL: 所有 Job 可用（默认）
SYSTEM: 仅 Jenkins 内部使用（如邮件通知）
FOLDER: 仅特定文件夹及子项可用

# 3. 凭据轮换策略
- 定期轮换（推荐 90 天）
- 使用 Token 替代密码
- 启用密钥自动轮换（如 Vault）

# 4. 审计与监控
- 定期审查凭据列表
- 删除不再使用的凭据
- 使用 Credentials Plugin 的 Binding 功能限制作用域
```

# 5. CSRF 保护（跨站请求伪造）

Jenkins 默认启用 CSRF 保护，外部系统调用 API 时需要携带 CSRF Token。

## 5.1 获取 CSRF Token

```bash
# 方式一：使用用户名密码
curl -u username:api_token \
  http://jenkins:8080/crumbIssuer/api/json

# 响应示例
{"crumb": "abc123...", "crumbRequestField": "Jenkins-Crumb"}

# 方式二：使用 API Token
CRUMB=$(curl -s -u "username:api_token" \
  http://jenkins:8080/crumbIssuer/api/json | \
  jq -r '.crumb')

curl -X POST \
  -u "username:api_token" \
  -H "Jenkins-Crumb: $CRUMB" \
  http://jenkins:8080/job/myjob/build

# 方式三：禁用 CSRF（不推荐）
# Manage Jenkins > Security > CSRF Protection > 取消勾选
```

## 5.2 禁用 CSRF（仅限脚本调用）

```bash
# 通过 Jenkins CLI 禁用
java -jar jenkins-cli.jar -s http://localhost:8080 \
  groovy = << 'EOF'
import jenkins.model.Jenkins
def instance = Jenkins.instance
instance.setCrumbIssuer(null)
instance.save()
EOF
```

# 6. Agent/节点安全

## 6.1 Agent 访问控制

```yaml
# Agent 配置安全
节点配置:
  - Name: prod-agent
  - Remote root directory: /data/jenkins/agent
  - Labels: production critical
  - Usage:
    - Use this node as much as possible
    - Only build jobs with label expressions matching this node
  - Launch method: Launch agent via SSH
    - Host: 192.168.1.10
    - Credentials: jenkins-agent-key

# 高级安全设置
Security:
  - Disable deferred wipeout on this node: false
  - Disable workspace cache: false
  - Environment variables:
    - PATH + /usr/local/bin
```

## 6.2 JNLP Agent 安全

```bash
# JNLP Agent 连接安全
# 使用 Secret 进行身份验证
java -jar agent.jar \
  -url http://jenkins:8080/ \
  -secret <agent-secret> \
  -name "secure-agent" \
  -webSocket  # 使用 WebSocket 提高安全性

# 启用 Agent → Master 访问控制
# Manage Jenkins > Security > Agents
# Agent → Master Security: Disable 或 Strict
```

## 6.3 Agent 的隔离级别

| 隔离方式 | 安全性 | 资源消耗 | 适用场景 |
|---------|-------|---------|---------|
| 永久 Agent（SSH） | 低（共享系统） | 低 | 可信环境 |
| Docker Agent | 中（容器隔离） | 中 | 标准 CI |
| Kubernetes Pod | 高（Pod 隔离） | 中 | 云原生 |
| 一次性 VM | 高（完全隔离） | 高 | 不安全构建 |

# 7. Pipeline 中的安全实践

## 7.1 凭据使用安全

```groovy
// ✅ 安全使用凭据
stage('Secure Build') {
    steps {
        withCredentials([usernamePassword(
            credentialsId: 'deploy-creds',
            usernameVariable: 'USER',
            passwordVariable: 'PASS'
        )]) {
            sh '''
                # ✅ 环境变量方式注入
                curl -u "$USER:$PASS" https://api.company.com/deploy
            '''
        }
    }
}
```

```groovy
// ❌ 不安全操作
stage('Unsafe Build') {
    steps {
        script {
            // ❌ 硬编码密码
            def password = 'plaintext-password'

            // ❌ 将密码存入环境变量（全局暴露）
            env.SECRET = 'my-secret'

            // ❌ 日志中打印密码
            sh 'echo $PASSWORD'  // 密码会出现在构建日志中

            // ❌ 密码作为构建参数暴露
            // 参数化的 String 类型会显示在构建详情页
        }
    }
}
```

## 7.2 Groovy 沙箱安全

GitHub 安全实验室发现，Pipeline 沙箱并非完全隔离。安全增强措施：

```groovy
// 限制共享库中的危险操作
@Library('safe-lib') _

// ✅ 安全的 Pipeline
pipeline {
    agent any
    stages {
        stage('Safely read file') {
            steps {
                script {
                    // ✅ Jenkins 提供的安全 API
                    def content = readFile file: 'allowed/path.txt'
                    echo content
                }
            }
        }
    }
}

// ❌ 潜在的沙箱逃逸
// 以下操作可能被 Jenkins 安全设置阻止
// System.exit(1)
// File.createTempFile()
// Runtime.getRuntime().exec('rm -rf /')
```

## 7.3 Pipeline 审计日志

```groovy
// 审计日志最佳实践
pipeline {
    agent any
    stages {
        stage('Audited Action') {
            steps {
                script {
                    def user = currentBuild.rawBuild.getCause(
                        hudson.model.Cause.UserIdCause
                    )?.userId ?: 'unknown'

                    echo "[AUDIT] User ${user} triggered build #${BUILD_NUMBER}"

                    // 执行重要操作
                    sh './critical-operation.sh'

                    echo "[AUDIT] Operation completed by ${user} at ${new Date()}"
                }
            }
        }
    }
    post {
        always {
            // 将审计日志发送到外部系统
            sh """
                curl -X POST https://audit.company.com/log \
                    -H 'Content-Type: application/json' \
                    -d '{"job":"${JOB_NAME}","build":${BUILD_NUMBER},"status":"${currentBuild.result}"}'
            """
        }
    }
}
```

# 8. 安全加固清单

## 8.1 生产环境安全清单

```yaml
# 1. 认证
✅ 禁用匿名用户注册
✅ 使用 LDAP/SSO 统一认证
✅ 强制使用强密码
✅ 启用 API Token 替代密码调用

# 2. 授权
✅ 使用 Role-Based Strategy 插件
✅ 最小权限原则
✅ 定期审计用户权限
✅ 禁用匿名用户的 Configure 和 Administer 权限

# 3. 网络安全
✅ 启用 HTTPS（配置反向代理）
✅ 限制 Jenkins 监听地址（0.0.0.0 → 内网 IP）
✅ 配置防火墙规则限制访问来源
✅ 使用 WebSocket 连接 Agent

# 4. 凭据安全
✅ 定期轮换凭据
✅ 使用 Secret files 而非明文
✅ 限制凭据作用域
✅ 备份 master.key 和 hudson.util.Secret
✅ 使用外部凭据管理（Vault）

# 5. 插件安全
✅ 仅安装必要的插件
✅ 定期更新插件
✅ 关注安全公告（Security Advisory）
✅ 禁用未使用的插件

# 6. 数据安全
✅ 加密 JENKINS_HOME 磁盘
✅ 配置构建日志保留策略
✅ 归档 artifact 过期策略
✅ 备份关键配置

# 7. 审计与监控
✅ 启用审计日志
✅ 监控异常操作
✅ 配置关键事件告警
✅ 定期安全审查
```

## 8.2 安全维护命令

```bash
# 检查 Jenkins 安全设置
curl -u admin:token http://jenkins:8080/api/json?pretty=true \
  | jq '.useSecurity, .useCrumbs'

# 获取所有用户列表（需管理员权限）
curl -u admin:token http://jenkins:8080/asynchPeople/api/json

# 获取安全配置
java -jar jenkins-cli.jar -s http://localhost:8080 \
  groovy = << 'EOF'
import jenkins.model.*
def instance = Jenkins.instance
println "Use Security: ${instance.useSecurity}"
println "Use CRUMB: ${instance.crumbIssuer != null}"
EOF

# 安全模式（紧急情况）
# 在 Jenkins 启动时创建空文件
touch /var/lib/jenkins/jenkins.install.InstallUtil.lastExecVersion
# 重启后进入安全模式，仅管理员可登录
```
