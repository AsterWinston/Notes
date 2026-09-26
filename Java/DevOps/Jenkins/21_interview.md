
# 1. 基础概念

## 1.1 CI/CD 基础

**Q: 什么是 CI/CD？CI 和 CD 的区别是什么？**

A: CI（持续集成）是开发人员频繁地将代码合并到主干，每次合并都通过自动化构建来验证。CD（持续交付/持续部署）是在 CI 基础上，确保软件可以随时可靠地发布。区别在于：CI 关注构建和测试自动化，CD 关注部署自动化；持续交付需要人工确认才能部署到生产，持续部署则全程自动化。

**Q: Jenkins 是什么？有什么特点？**

A: Jenkins 是一个开源的 CI/CD 自动化服务器，基于 Java 开发。特点：Master/Agent 分布式架构、1000+ 插件生态、支持 Pipeline as Code、丰富的 REST API、支持多种 SCM 和构建工具集成。

## 1.2 核心概念

**Q: 解释 Jenkins 的 Master/Agent 架构**

A: Master 负责任务调度、配置管理、UI 展示；Agent 实际执行构建任务。Master 将任务加入队列，根据 Agent 标签和可用性分配任务。Agent 可以通过 SSH/JNLP/WebSocket/K8s 等方式连接。

**Q: 什么是 Pipeline？声明式和脚本式的区别？**

A: Pipeline 是用代码定义 CI/CD 流程的框架。声明式（Declarative）使用结构化 DSL，语法简洁，适合大多数场景；脚本式（Scripted）使用完整 Groovy 语法，更灵活。推荐优先使用声明式。

**Q: 什么是 Jenkinsfile？为什么要使用它？**

A: Jenkinsfile 是定义 Pipeline 的文本文件，存放在仓库根目录。优势：版本控制、代码审查、可追溯、可复用、分支感知。

# 2. 实战经验

## 2.1 Pipeline 相关

**Q: 如何实现 Pipeline 的并行构建？**

```groovy
// 声明式并行
stage('Tests') {
    parallel {
        stage('Unit') { steps { sh 'mvn test' } }
        stage('Integration') { steps { sh 'mvn verify' } }
    }
}

// 动态并行
def branches = [:]
['linux', 'windows', 'mac'].each { os ->
    branches[os] = {
        node(os) { sh "./test.sh --os ${os}" }
    }
}
parallel branches
```

**Q: 如何实现条件执行？**

```groovy
stage('Deploy') {
    when {
        branch 'main'
        expression { return params.ENV == 'prod' }
    }
    steps {
        sh './deploy.sh'
    }
}
```

**Q: 如何实现人工审批？**

```groovy
stage('Approval') {
    steps {
        input message: 'Deploy to production?',
              ok: 'Deploy',
              submitter: 'admin,release-manager',
              parameters: [string(name: 'VERSION', defaultValue: params.VERSION)]
    }
}
```

## 2.2 集成实践

**Q: Jenkins 如何与 Docker 集成？**

A: 三种集成方式：1）在 Docker 容器中运行 Pipeline（agent docker）；2）构建和推送 Docker 镜像（docker.build/push）；3）使用 Docker 作为 Agent 提供者（Docker Cloud）。

**Q: Jenkins 如何与 Kubernetes 集成？**

A: 通过 Kubernetes Plugin，Jenkins 在 K8s 上动态创建 Pod 作为 Agent。Pipeline 中可以定义 Pod Template、使用边车容器、执行 kubectl/helm 命令进行部署。

**Q: 如何实现配置即代码（JCasC）？**

A: 使用 Configuration as Code Plugin，用 YAML 声明 Jenkins 的全局配置、凭据、安全设置等。配置文件可以纳入 Git 管理，实现 Jenkins 配置的版本化和自动化部署。

## 2.3 安全相关

**Q: Jenkins 如何安全管理凭据？**

A: 使用 Credentials Plugin，凭据加密存储在 JENKINS_HOME 中（使用 master.key + AES 加密）。支持 UsernamePassword、SSH Key、Secret text、Secret file、Certificate 等类型。Pipeline 中使用 withCredentials 安全访问。

**Q: 如何实现 Jenkins 的权限控制？**

A: 推荐使用 Role-Based Strategy Plugin。支持全局角色（admin/developer/viewer）、项目角色（按 Job 正则匹配）、Agent 角色。可与 LDAP/SSO 集成实现统一认证。

# 3. 场景题

## 3.1 微服务 CI/CD

**Q: 如何设计一个微服务 CI/CD Pipeline？**

```groovy
@Library('shared-lib') _

def services = ['service-a', 'service-b', 'service-c']

pipeline {
    agent none
    stages {
        stage('Build All') {
            parallel {
                script {
                    services.each { service ->
                        stage("Build ${service}") {
                            agent { label 'maven' }
                            steps {
                                dir(service) {
                                    checkout scm
                                    sh 'mvn clean package'
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('Integration Test') {
            agent { label 'k8s-agent' }
            steps { sh './integration-test.sh' }
        }
        stage('Deploy') {
            when { branch 'main' }
            agent { label 'k8s-agent' }
            steps {
                services.each { service ->
                    sh "kubectl set image deployment/${service} app=${service}:${BUILD_NUMBER}"
                }
            }
        }
    }
}
```

## 3.2 多环境部署

**Q: 如何管理开发、测试、生产环境？**

```groovy
pipeline {
    agent any
    parameters {
        choice(name: 'ENV', choices: ['dev', 'staging', 'prod'])
        string(name: 'VERSION', defaultValue: 'latest')
    }
    stages {
        stage('Environment Validation') {
            steps {
                script {
                    // 环境特定的安全检查
                    if (params.ENV == 'prod' && env.BRANCH_NAME != 'main') {
                        error 'Production deployment only allowed from main branch'
                    }
                }
            }
        }
        stage('Deploy') {
            steps {
                script {
                    // 使用环境特定配置
                    def config = readYaml file: "config/${params.ENV}.yaml"
                    sh """
                        helm upgrade --install myapp ./chart \
                            --namespace ${params.ENV} \
                            --set image.tag=${params.VERSION} \
                            --set replicaCount=${config.replicas}
                    """
                }
            }
        }
    }
}
```

## 3.3 紧急修复流程

**Q: 紧急修复（hotfix）的 Pipeline 如何设计？**

```groovy
pipeline {
    agent any
    triggers {
        // hotfix 分支触发
        pollSCM('H/1 * * * *')
    }
    stages {
        stage('Validate Branch') {
            when { branch 'hotfix/*' }
            steps {
                script {
                    // 自动创建 PR 到 main
                    sh """
                        gh pr create --base main --head ${env.BRANCH_NAME} \
                            --title "Hotfix: ${env.BRANCH_NAME}" \
                            --body "Emergency fix"
                    """
                }
            }
        }
        stage('Quick Build') {
            steps { sh 'mvn clean package -DskipTests' }
        }
        stage('Critical Tests') {
            steps { sh 'mvn test -Pcritical' }
        }
        stage('Deploy') {
            steps {
                input message: 'Deploy hotfix to production?'
                sh './deploy.sh prod'
            }
        }
    }
}
```

## 3.4 大规模构建

**Q: 如何优化大规模 Monorepo 的构建速度？**

A: 1）增量构建：只构建变更的模块；2）并行构建：按模块并行；3）依赖缓存：共享 Maven/Gradle 缓存；4）浅克隆：减少 Git 数据传输；5）构建分发：多个 Agent 分担负载；6）流水线优化：减少不必要的步骤。

```groovy
// 增量构建示例
def changedModules = getChangedModules()  // 从 git diff 获取

stage('Build') {
    parallel {
        script {
            changedModules.each { module ->
                stage("Build ${module}") {
                    steps {
                        dir(module) {
                            sh 'mvn clean package'
                        }
                    }
                }
            }
        }
    }
}
```

# 4. 原理题

## 4.1 调度原理

**Q: Jenkins 的任务调度流程是什么？**

A: 1）Job 触发（Webhook/定时/手动）→ 创建 Build 对象；2）Build 进入队列 Queue；3）Queue 根据标签表达式查找匹配 Agent；4）找到可用 Executor → 分配执行；5）Agent 检出代码、执行 Pipeline、收集结果；6）结果归档、通知。

**Q: Pipeline 为什么可以暂停和恢复？**

A: Jenkins Pipeline 使用"飞行中"（flyweight）持久化机制。Pipeline 执行到暂停点（如 input 步骤、等待 Agent）时，将当前状态序列化到磁盘。Master 重启后，从保存的检查点反序列化恢复执行。

## 4.2 插件机制

**Q: Jenkins 的插件机制是如何工作的？**

A: Jenkins 基于 Java 的 Extension 机制（`@Extension` 注解）。插件实现特定的扩展点（如 Builder、Publisher、SCM、Step 等），Jenkins 启动时扫描并加载所有插件。插件使用 Descriptor 模式管理配置界面。

## 4.3 凭据加密

**Q: Jenkins 是如何加密存储凭据的？**

A: 使用对称加密。流程：1）Master 启动时生成 master.key（AES 128 位）；2）使用 master.key 加密 hudson.util.Secret 的密钥；3）凭据使用 Secret 加密后存储在 XML 中。即使数据库泄露，没有 master.key 也无法解密。

# 5. 架构设计题

## 5.1 企业级 Jenkins 架构

**Q: 如何设计一个支持 500+ 开发人员的企业级 Jenkins 平台？**

```yaml
# 架构要点
1. 高可用
   - Active/Passive 模式
   - 共享存储（NFS/EFS）
   - 负载均衡（HAProxy/Nginx）

2. 弹性伸缩
   - Kubernetes 动态 Agent
   - 按需创建，用完即销毁
   - Cluster Autoscaler 自动扩展节点

3. 安全
   - LDAP/SSO 统一认证
   - Role-Based 权限控制
   - 项目级别的文件夹隔离
   - 凭据加密存储

4. 性能
   - Master 不执行构建
   - 构建历史定期清理
   - Pipeline 缓存优化
   - 监控告警（Prometheus + Grafana）

5. 标准化
   - JCasC 配置管理
   - 共享库统一 CI/CD 逻辑
   - Jenkinsfile 模板化
```

## 5.2 多数据中心

**Q: 如何实现 Jenkins 的多数据中心部署？**

```yaml
# 多数据中心方案
主数据中心:
  - Jenkins Master Active
  - 所有 Job 定义和管理
  - 数据定期同步到备份中心

备份数据中心:
  - Jenkins Master Standby
  - 从主数据中心同步数据
  - 主中心宕机时接管

Agent 部署:
  - 每个数据中心部署独立 Agent
  - Agent 连接到所属数据中心的 Master
  - Pipeline 中指定数据中心标签

网络要求:
  - 数据中心间低延迟连接（< 50ms）
  - 共享存储跨数据中心复制
  - DNS/负载均衡全局调度

切换流程:
  主中心宕机 → 健康检查失败 → DNS 切换到备中心
  → 备中心 Master 激活 → Agent 重新连接 → 恢复构建
```

# 6. DevOps 方法论

**Q: Jenkins 在 DevOps 实践中扮演什么角色？**

A: Jenkins 是 DevOps 工具链的核心枢纽，负责：
- CI：代码提交自动触发构建和测试
- CD：自动化部署到各环境
- 质量门禁：集成 SonarQube 等代码质量工具
- 自动化：替代重复的手动操作
- 反馈：构建结果及时通知团队
- 度量：构建频率、成功率、耗时等 DevOps 指标

**Q: GitOps 流程中 Jenkins 的位置？**

```yaml
GitOps with Jenkins:
  开发人员 → Push 代码 → Git Webhook → Jenkins Pipeline
    → CI: Build → Test → Scan
    → CD: Build Image → Push Registry
    → GitOps Operator: Update Manifest → PR to Git
    → ArgoCD/Flux: Sync → Deploy to K8s

# Jenkins 在这个流程中主要负责 CI 部分
# GitOps Operator（ArgoCD/Flux）负责 CD 部分
```
