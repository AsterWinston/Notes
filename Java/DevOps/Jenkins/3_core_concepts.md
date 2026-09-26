
# 1. Master（Controller）

Master 是 Jenkins 的核心控制节点，负责以下职能：

- **HTTP 服务**：提供 Web UI 和 REST API
- **配置存储**：管理所有 Job、Agent、凭据等配置
- **任务调度**：根据 Agent 的标签、负载、可用性将任务分发给合适的 Agent
- **构建队列管理**：维护待执行的构建任务队列
- **结果收集**：从 Agent 收集构建日志、测试结果、artifact 等
- **插件管理**：加载和管理插件

Master 本身也可以作为 Agent 执行构建任务，但生产环境建议将 Master 的执行者数量设为 0，将构建任务交给专门的 Agent 执行，以保证 Master 的稳定性。

# 2. Agent（Node）

Agent（旧称 Slave/Node）是 Jenkins 中实际执行构建任务的工作节点。

## 2.1 Agent 类型

| 类型 | 说明 | 连接方式 | 适用场景 |
|------|------|---------|---------|
| **Permanent Agent** | 永久运行的 Agent 节点 | SSH/JNLP | 物理机或虚拟机 |
| **Cloud Agent** | 动态创建的 Agent 节点 | JNLP | Docker/K8s 等云环境 |
| **Built-in Node** | Master 内置节点（默认） | 内置 | 小型测试环境 |

## 2.2 Agent 的连接方式

**a）SSH 方式（Linux/Unix）**

Master 通过 SSH 连接到 Agent，执行远程命令。

配置要求：
- Agent 机器上运行 SSH 服务
- Master 配置 SSH 凭据（用户名+密码 或 SSH Key）
- Agent 机器上安装 Java Runtime（JRE）

```
Agent 配置示例：
- Name: linux-agent-1
- Remote root directory: /home/jenkins/agent
- Labels: linux amd64 docker
- Usage: Only build jobs with label expressions matching
- Launch method: Launch agent via SSH
  - Host: 192.168.1.101
  - Port: 22
  - Credentials: jenkins-ssh-key
  - Host Key Verification: Known hosts file
- Availability: Keep this agent online as much as possible
```

**b）JNLP 方式（Java Web Start / Windows）**

Agent 主动连接 Master，适用于：
- Windows 环境（SSH 配置复杂时）
- Agent 在 NAT/防火墙后无法被 Master 直接连接

配置要求：
- Agent 机器上运行 Java Runtime
- Agent 启动时下载 agent.jar 并通过 JNLP 连接 Master
- Master 的 JNLP 端口（50000）可被 Agent 访问

```bash
# Agent 端执行
curl -sO http://jenkins-master:8080/jnlpJars/agent.jar
java -jar agent.jar \
  -url http://jenkins-master:8080/ \
  -secret <secret-key> \
  -name "windows-agent-1" \
  -webSocket \
  -workDir "C:\jenkins\agent"
```

**c）WebSocket 方式**

JNLP 的改进版本，通过 WebSocket 协议建立连接（需要 Jenkins 2.217+）：
- 无需打开额外的 TCP 端口（50000）
- 与 HTTP 反向代理兼容性更好
- 连接更稳定

```bash
# 启动时添加 -webSocket 参数
java -jar agent.jar -url http://jenkins-master:8080/ -secret xxx -name agent1 -webSocket
```

**d）Docker Agent**

基于 Docker 容器运行 Agent：
- 生命周期短，用完即销毁
- 环境隔离，每次构建使用干净的容器
- 通过 Docker 镜像预装构建工具

## 2.3 Agent 标签（Label）

标签是 Agent 的分类标识，用于 Pipeline 中指定执行节点。

```
Agent 的标签设计原则：
├── 架构标签：linux, windows, amd64, arm64
├── 工具标签：docker, maven, nodejs, python
├── 环境标签：dev, test, prod
└── 特性标签：ssd, gpu, high-memory
```

标签表达式语法：

| 表达式 | 含义 | 示例 |
|--------|------|------|
| `A` | 匹配标签包含 A 的节点 | `linux` |
| `A && B` | 同时包含 A 和 B | `linux && docker` |
| `A \|\| B` | 包含 A 或 B | `windows \|\| linux` |
| `!A` | 不包含 A | `!windows` |
| `(A \|\| B) && C` | 组合表达式 | `(linux \|\| windows) && docker` |

## 2.4 Agent 的可用性策略

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| Keep this agent online as much as possible | 一直在线 | 永久节点 |
| Bring this agent online when in demand, and take offline when idle | 按需上线 | 云实例（节省成本） |
| Take this agent online at specific times | 定时上线 | 限定时段 |

# 3. Executor（执行器）

Executor 是 Agent 上并行执行任务的线程。每个 Agent 可以配置多个 Executor。

配置要点：
- **不是越多越好**：Executor 数取决于 CPU 核数和任务类型
- **CPU 密集型任务**：Executor = CPU 核心数（或 CPU 核心数 - 1）
- **I/O 密集型任务**：Executor 可适当增加（为核心数的 1.5~2 倍）
- **Docker Agent**：通常设为 1（每个容器只执行一个任务）

```
Agent 配置中的 Executor：
# of executors: 2  # 表示该 Agent 同时执行 2 个构建任务
```

# 4. Job（任务/项目）

Job 是 Jenkins 中一个可执行的任务单元，定义"做什么"和"怎么做"。

## 4.1 Job 类型

| 类型 | 说明 | 适用场景 |
|------|------|---------|
| **Freestyle project** | 自由风格项目，通过 Web UI 配置构建步骤 | 简单的构建任务 |
| **Pipeline** | 流水线项目，通过 Jenkinsfile 定义流程 | 复杂构建流程 |
| **Multi-configuration project** | 多配置项目（矩阵项目），在多个维度组合构建 | 多环境/多平台测试 |
| **Folder** | 文件夹，用于组织和管理 Job | 大量 Job 分类 |
| **Multibranch Pipeline** | 多分支 Pipeline，自动发现分支创建对应 Pipeline | Git 分支策略 |
| **Organization Folder** | 组织文件夹，自动发现 GitHub/GitLab 组织中的仓库 | GitHub/GitLab 组织 |

## 4.2 Job 的触发方式

| 触发方式 | 配置方式 | 说明 |
|---------|---------|------|
| **SCM 轮询** | Poll SCM | 定时检查 SCM 变更 |
| **Webhook 触发** | GitHub/GitLab Webhook | SCM 推送事件触发 |
| **定时触发** | Build periodically | Cron 表达式定时触发 |
| **其他 Job 触发** | Build after other projects are built | 上游构建成功后触发 |
| **手动触发** | 用户点击 Build Now | 手动触发 |
| **REST API 触发** | curl POST /job/<name>/build | 外部系统调用触发 |
| **SCM 变更触发** | GitHub Branch Source | 多分支 Pipeline 自动检测 |

# 5. Build（构建）

Build 是 Job 的一次具体执行实例，包含完整的执行记录。

## 5.1 构建状态

| 状态 | 图标 | 含义 |
|------|------|------|
| **SUCCESS** | 蓝色 | 构建成功，所有步骤通过 |
| **UNSTABLE** | 黄色 | 构建完成，但有不稳定条件（如测试失败但未达到阈值） |
| **FAILURE** | 红色 | 构建失败（编译错误、测试失败等） |
| **ABORTED** | 灰色 | 构建被手动中止 |
| **NOT BUILT** | 白色 | 尚未构建过 |
| **IN PROGRESS** | 闪烁 | 正在构建中 |

## 5.2 构建编号与构建记录

每次构建都有一个唯一编号（从 1 开始递增），存储在 `JENKINS_HOME/jobs/<job>/builds/<build-number>/` 目录：

```
builds/<number>/
├── build.xml          # 构建元数据（参数、结果、时间等）
├── log                # 控制台输出日志（完整文本）
├── changelog.xml      # 变更日志（Git commits）
├── archive/           # 归档的构建产物
├── junitResult/       # JUnit 测试报告
├── workflow/          # Pipeline 执行数据
└── pipeline-graph/    # Pipeline 阶段图数据
```

## 5.3 构建队列

当所有 Executor 都被占用时，新的构建任务进入队列等待。

默认策略：**FIFO（先进先出）**
可配置策略：**优先级调度**（通过插件支持）

查看队列：`Jenkins > Build Queue > 侧边栏`

# 6. Workspace（工作空间）

Workspace 是 Agent 上用于检出代码和执行构建的目录。

- **默认路径**：`${JENKINS_HOME}/workspace/<job-name>`
- **自定义路径**：可在 Agent 配置中通过 `Remote root directory` 修改
- **清理策略**：
  - 每次构建前可使用 `Clean Workspace` 插件清空
  - Pipeline 中通过 `cleanWs()` 或 `deleteDir()` 清理

```groovy
// Pipeline 中管理工作空间
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                // 检出到子目录，避免污染工作空间
                checkout scm
            }
        }
        stage('Build') {
            steps {
                // 删除工作空间中除了 .git 外的所有内容
                deleteDir()
            }
        }
        stage('Test') {
            steps {
                // 清空工作空间（通过插件）
                cleanWs()
            }
        }
    }
}
```

# 7. View（视图）

View 用于组织和管理 Job 的展示方式。

## 7.1 视图类型

| 类型 | 说明 |
|------|------|
| **List View** | 列表视图，手动选择 Job 加入 |
| **My View** | 个人视图，每个用户独立配置 |
| **Pipeline View** | Pipeline 视图，展示 Pipeline 状态 |
| **Build Pipeline View** | 构建流水线视图，展示上下游依赖 |
| **Dashboard View** | 仪表盘视图，聚合展示多个信息 |
| **Sectioned View** | 分节视图，将 Job 分组显示 |

## 7.2 视图配置

```
List View 配置：
- Name: My-Services
- Description: 微服务构建视图
- Job Filters:
  - Status filter: All jobs (包含所有状态)
  - Recurse in subfolders: true
- Columns:
  - Status
  - Weather (构建健康度)
  - Name
  - Last Success
  - Last Failure
  - Last Duration
- Job Selection:
  - Use a regular expression to include jobs: ^service-.*
```

# 8. Credentials（凭据）

凭据是 Jenkins 中存储敏感信息（密码、Token、SSH Key 等）的安全机制。

## 8.1 凭据存储架构

```
┌────────────────────────────────────────────────┐
│                  Credentials                    │
├──────────────┬────────────────┬────────────────-┤
│   Global     │    System      │    Folder       │
│ (全局可用)    │ (仅 Jenkins)   │ (文件夹范围内)   │
├──────────────┴────────────────┴────────────────-┤
│ 加密方式：                                        │
│ master.key (主密钥, JENKINS_HOME/secrets/)       │
│   └── hudson.util.Secret (派生密钥)              │
│       └── 凭据密文 (存储在 config.xml 或其他文件) │
└──────────────────────────────────────────────────-┘
```

## 8.2 凭据类型详解

**Username with password**：

```groovy
// Pipeline 中使用
withCredentials([usernamePassword(
    credentialsId: 'gitlab-credentials',
    usernameVariable: 'GIT_USERNAME',
    passwordVariable: 'GIT_PASSWORD'
)]) {
    sh "git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@gitlab.company.com/repo.git"
}
```

**SSH Username with private key**：

```groovy
// Pipeline 中使用
sshagent(['gitlab-ssh-key']) {
    sh "git clone git@gitlab.company.com:group/repo.git"
}
```

**Secret text**：

```groovy
// Pipeline 中使用
withCredentials([string(
    credentialsId: 'sonar-token',
    variable: 'SONAR_TOKEN'
)]) {
    sh "sonar-scanner -Dsonar.login=$SONAR_TOKEN"
}
```

**Secret file**：

```groovy
// Pipeline 中使用
withCredentials([file(
    credentialsId: 'kubeconfig',
    variable: 'KUBECONFIG'
)]) {
    sh "kubectl --kubeconfig=$KUBECONFIG apply -f deployment.yaml"
}
```

## 8.3 凭据加密原理

```
1. Jenkins 启动时生成 master.key（AES 256 位密钥）
2. 使用 master.key 加密生成 hudson.util.Secret 的密钥
3. 所有凭据使用 Secret 加密后存储在 XML 中
4. 即使数据库泄露，没有 master.key 也无法解密
```

# 9. Pipeline（流水线）

Pipeline 是 Jenkins 2.0 的核心功能，用代码定义 CI/CD 流程。

## 9.1 Pipeline 的核心概念

| 概念 | 说明 |
|------|------|
| **Stage** | 阶段，逻辑分组，如 Build、Test、Deploy |
| **Step** | 步骤，最小执行单元，如 `sh`、`git`、`echo` |
| **Node** | 节点，在特定 Agent 上分配 Executor 执行 |
| **Agent** | 代理，指定在哪个/些节点上执行 |
| **Post** | 后置条件，根据构建结果执行清理或通知 |

## 9.2 两种语法风格

**Declarative Pipeline（声明式）**：

```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                echo 'Building...'
            }
        }
    }
}
```

**Scripted Pipeline（脚本式）**：

```groovy
node {
    stage('Build') {
        echo 'Building...'
    }
}
```

两者的选择：声明式语法更简洁、结构清晰，是推荐方式；脚本式更灵活，适合复杂逻辑。

# 10. Queue（队列）

Queue 是 Jenkins Master 内部的优先级队列，用于调度构建任务。

## 10.1 调度流程

```
1. Job 被触发 → 创建 Build 对象
2. Build 进入 Queue
3. Queue 查找匹配的 Agent（根据标签表达式）
4. 找到可用 Executor → 分配执行
5. 未找到 → 在队列中等待
6. Cloud Agent → 触发 Provisioning（动态创建）
```

## 10.2 队列等待的原因

| 原因 | 说明 | 排查方法 |
|------|------|---------|
| 无可用 Executor | 所有 Agent 繁忙 | 检查 Agent 负载，增加 Agent |
| 无匹配 Agent | 标签表达式无 Agent 满足 | 检查标签配置 |
| Agent 离线 | Agent 不可达 | 检查 Agent 连接状态 |
| 节点限制 | 节点被限制只执行特定 Job | 检查节点配置 |
| 队列阻塞 | 上游任务被阻塞 | 检查队列中的前序任务 |

# 11. Fingerprint（指纹）

指纹是 Jenkins 中跟踪文件来源的机制（通过 MD5 哈希）。

应用场景：
- 跟踪 JAR/WAR 文件的来源（来自哪个 Job 的哪个 Build）
- 检测文件在不同 Job 间的使用关系
- 版权审计和许可证管理

```groovy
// Pipeline 中归档文件并生成指纹
stage('Archive') {
    steps {
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
    }
}
```

# 12. Folder（文件夹）

Jenkins 的 Folder 类似于文件系统中的目录，用于组织和管理 Job。

特性：
- **嵌套层级**：文件夹可以包含其他文件夹
- **权限继承**：子项继承父文件夹的权限配置
- **凭据作用域**：文件夹内的凭据只在文件夹内可见
- **视图隔离**：不同文件夹下的 Job 视图独立

```
项目结构示例：
├── services/ (文件夹)
│   ├── user-service/ (Pipeline)
│   ├── order-service/ (Pipeline)
│   └── payment-service/ (Pipeline)
├── libraries/ (文件夹)
│   └── common-lib/ (Pipeline)
└── infrastructure/ (文件夹)
    ├── ci-pipeline/ (Pipeline)
    └── cd-pipeline/ (Multibranch Pipeline)
```

# 13. Build Trigger（构建触发器）

## 13.1 SCM Polling（SCM 轮询）

```
Poll SCM 配置：
Schedule: H/5 * * * *   # 每 5 分钟检查一次变更
```

Cron 语法说明：
```
MINUTE HOUR DOM MONTH DOW
H/5 * * * *   # H 表示哈希分散，避免所有 Job 同时轮询
```

**Cron 特殊参数**：
- `H`：哈希值，基于 Job 名称计算
  - `/5`：每 5 分钟，但分钟数为 H 的倍数
- `@daily`：每天零点 `H H * * *`
- `@weekly`：每周日零点 `H H * * 0`
- `@monthly`：每月 1 日零点 `H H 1 * *`

## 13.2 Webhook 触发

推荐使用 Webhook 替代 Polling，减少 Jenkins 负载，实现真实时触发。

**GitHub Webhook 配置**：

```
GitHub 侧：
1. Settings > Webhooks > Add webhook
2. Payload URL: http://jenkins:8080/github-webhook/
3. Content type: application/json
4. Events: Just the push event (或 Let me select individual events)
```

**GitLab Webhook 配置**：

```
GitLab 侧：
1. Settings > Webhooks > Add webhook
2. URL: http://jenkins:8080/project/<job-name>
3. Token: <jenkins-gitlab-token>
4. Trigger: Push events, Merge Request events
```

## 13.3 定时构建

```groovy
// Jenkinsfile 中使用 triggers 指令
pipeline {
    agent any
    triggers {
        // 工作日每天 10:00 和 16:00 构建
        cron('0 10,16 * * 1-5')
        // 或在特定分支上
        pollSCM('H/5 * * * *')
    }
    // ...
}
```

# 14. Parameter（参数）

Job 可以定义参数，在触发构建时传入。

## 14.1 参数类型

| 类型 | 说明 | 示例值 |
|------|------|--------|
| String Parameter | 字符串 | `feature_xxx` |
| Choice Parameter | 下拉选择 | `dev/test/prod` |
| Boolean Parameter | 布尔值 | `true/false` |
| Password Parameter | 密码（加密显示） | `****` |
| File Parameter | 文件上传 | 上传配置文件 |
| Extended Choice Parameter | 多选 | `[env, version]` |
| Active Choices Parameter | 动态选项（由 Groovy 生成） | 动态分支列表 |

```groovy
// Jenkinsfile 中的参数定义
pipeline {
    agent any
    parameters {
        string(name: 'BRANCH', defaultValue: 'main', description: 'Git 分支')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'test', 'prod'], description: '部署环境')
        booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: '是否跳过测试')
        password(name: 'DEPLOY_TOKEN', description: '部署 Token')
    }
    stages {
        stage('Deploy') {
            steps {
                echo "Deploying branch ${params.BRANCH} to ${params.ENVIRONMENT}"
                echo "Skip tests: ${params.SKIP_TESTS}"
            }
        }
    }
}
```
