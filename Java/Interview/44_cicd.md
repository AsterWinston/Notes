# 1. 知识点
- **CI(持续集成)**：开发人员频繁将代码合并到主线(trunk/main)，每次提交自动触发构建和测试，尽快发现集成问题
- **CD(持续交付)**：代码通过自动化测试后持续交付到类生产环境，最终部署到生产需人工审批
- **持续部署(Continuous Deployment)**：持续交付的延伸，通过全部自动化测试后直接全自动部署到生产环境，无需人工干预
- **Jenkins**：最广泛使用的开源 CI/CD 工具，插件生态丰富，支持自由风格和 Pipeline
- **自由风格Pipeline**：Jenkins 早期可视化配置任务类型，通过 UI 配置步骤，不便于版本控制和复用
- **Scripted Pipeline**：基于 Groovy 脚本的流水线，灵活但复杂，适合高级定制场景
- **Declarative Pipeline**：Jenkins 推荐的声明式流水线，结构化 `pipeline { stages { stage { steps {...} } } }`，简洁可读
- **Jenkinsfile**：定义声明式流水线的文件，存放在代码仓库根目录，实现 Pipeline as Code
- **常见Jenkins插件**：git（拉取代码）、maven/gradle（构建Java）、docker（构建推送镜像）、blue ocean（现代化可视化界面）
- **GitHub Actions**：GitHub 内置 CI/CD 平台，以 Workflow 为单元，YAML 文件在 `.github/workflows/` 目录下
- **workflow YAML**：事件驱动的自动化工作流文件，由 job 和 step 组成
- **on: push / pull_request / workflow_dispatch**：触发事件分别为 push 代码、PR 创建更新和手动触发
- **marketplace社区actions**：GitHub Actions Marketplace 提供大量社区 actions，如 checkout@v4、setup-java@v4、docker/build-push-action
- **GitLab CI**：GitLab 内置 CI/CD，通过 `.gitlab-ci.yml` 定义流水线
- **stages定义流水线**：GitLab CI 中通过 stages 定义阶段（如 build、test、deploy），每个 stage 包含多个 Job，stage 内 Job 并行执行，stage 间串行
- **GitLab Runner**：执行 CI Job 的代理程序，可部署在不同机器上，支持多种执行器（Shell、Docker、K8s）
- **内置Registry**：GitLab 内置容器镜像仓库 Container Registry，可直接推送和拉取镜像
- **Review App预览**：GitLab 的动态环境功能，每个 MR 可部署一个临时预览环境用于评审
- **代码质量**：将 SonarQube 集成到 CI 流水线，自动扫描代码 Bug、安全漏洞、代码异味、测试覆盖率
- **SonarQube质量门**：SonarQube 的 Quality Gate，设定代码质量达标条件（如覆盖率80%+、0 Bug），不达标则流水线失败
- **容器化CI/CD**：构建 Docker 镜像推送到 Harbor/Registry，在 CD 阶段通过 kubectl 部署到 K8s 集群
- **Harbor**：开源企业级容器镜像仓库，支持镜像漏洞扫描、签名、复制、RBAC
- **Rollback回滚策略**：部署失败后根据策略快速恢复到上一个稳定版本
- **蓝绿部署**：维护两套完全相同的生产环境(Blue/Green)，蓝环境运行当前版本，部署到绿环境并测试完毕后切换流量到绿环境，旧蓝环境保留以便回滚
- **金丝雀发布**：将少量流量（如 5%）切换到新版本验证，逐步增加流量占比，全部正常后全量切换，有问题立即回滚
- **滚动更新**：逐步替换实例，每次替换一部分 Pod/容器，直到全部更新完成，保证服务不间断
- **制品管理**：Nexus 管理 Maven/Java 私服仓库，Harbor 管理 Docker 镜像，Artifactory 通用制品仓库
- **Nexus**：Maven 私服仓库，也支持 Docker、npm、PyPI 等多种格式
- **Artifactory**：JFrog 的通用制品仓库，支持几乎所有包格式
- **典型CI流水线**：checkout 代码 → 单元测试 → Maven build → SonarQube 质量扫描 → Docker build → Docker push → 触发 CD
- **典型CD流水线**：拉取镜像 → 部署到 Dev 环境 → 自动化测试 → 部署到 Staging → 审批 → 部署到 Production（滚动更新 + 健康检查 + 失败自动回滚）
- **多环境部署**：dev → test → staging → production，流水线逐步推进，关键环境设审批门
- **Jenkinsfile示例**：`pipeline { agent any; stages { stage('Build') { steps { sh 'mvn package' } } } }`

- **项目发布方式**: (1) JAR部署 — 环境依赖，简单直接;(2) Docker部署 — 解决环境依赖但大规模管理困难;(3) K8s部署 — 流量监控/副本弹性扩缩/热部署/CI-CD集成/容器编排/YAML可配置调优参数

# 2. 面试题

## 2.1 什么是 CI/CD？CI 和 CD 的区别是什么？
- **CI（持续集成）**：开发人员频繁（每天多次）将代码推送到主干，每次 push 自动触发构建、单元测试，尽早发现集成错误。关键词：频繁提交、自动构建、自动测试
- **CD（持续交付/持续部署）**：
  - **持续交付(Continuous Delivery)**：在 CI 基础上，通过自动化测试后持续交付到类生产环境，但最终部署到生产需要人工手动审批。保证每次 build 都是可部署状态
  - **持续部署(Continuous Deployment)**：持续交付的延伸，代码通过所有自动化测试后全自动部署到生产，无需人工确认

核心价值：快速反馈、减少集成风险、提高软件质量和交付速度。

## 2.2 Jenkins 的 Scripted Pipeline 和 Declarative Pipeline 有什么区别？
| 维度 | Scripted Pipeline | Declarative Pipeline |
|------|------------------|---------------------|
| 语法 | 基于 Groovy 脚本，灵活 | 结构化 DSL，严格 |
| 学习曲线 | 较陡峭，需 Groovy 基础 | 平缓，清晰声明式语法 |
| 错误处理 | 需手动 try-catch | 内置 `post` 段处理（success/failure/always） |
| 适用场景 | 高度定制化需求 | 标准 CI/CD 流程（推荐） |
| 验证 | 运行时才发现错误 | 更易于静态检查，支持 Blue Ocean 可视化 |
| 代码结构 | `node { stage { ... } }` | `pipeline { agent; stages { stage { steps } } }` |

推荐使用 Declarative Pipeline，简洁可读且功能满足大部分需求。

## 2.3 Jenkinsfile 的基本结构是什么样的？
```groovy
pipeline {
    agent any                    // 在任何可用 agent 上执行

    environment {                // 环境变量
        DOCKER_IMAGE = 'myapp:latest'
    }

    options {                    // 全局选项
        timeout(time: 1, unit: 'HOURS')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm    // 拉取代码
            }
        }
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        stage('SonarQube') {
            steps {
                sh 'mvn sonar:sonar'
            }
        }
        stage('Docker Build & Push') {
            steps {
                sh 'docker build -t $DOCKER_IMAGE .'
                sh 'docker push $DOCKER_IMAGE'
            }
        }
    }

    post {                        // 流水线完成后操作
        always {
            junit '**/target/surefire-reports/*.xml'
            cleanWs()
        }
        success {
            echo 'Pipeline succeeded!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
```

## 2.4 GitHub Actions 的核心概念和常用语法是什么？
核心概念：
- **Workflow**：一个自动化流程，定义在 `.github/workflows/*.yml` 中
- **Job**：一个 Workflow 包含多个 Job，Job 之间的依赖有 defaults 并行
- **Step**：Job 中每一步可以执行一个 action 或 shell 命令
- **Action**：可复用的单元，社区 marketplace 有大量现成 actions
- **Runner**：执行 Workflow 的服务器，GitHub 提供托管 runner 或自托管 runner

```yaml
name: Java CI
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      - name: Build with Maven
        run: mvn -B package --file pom.xml
```

## 2.5 GitLab CI 的 .gitlab-ci.yml 如何定义？
```yaml
stages:
  - build
  - test
  - deploy

variables:
  MAVEN_CLI_OPTS: "-s .m2/settings.xml --batch-mode"
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"

cache:
  paths:
    - .m2/repository/

build:
  stage: build
  image: maven:3.9-eclipse-temurin-21
  script:
    - mvn $MAVEN_CLI_OPTS compile
  artifacts:
    paths:
      - target/

test:
  stage: test
  image: maven:3.9-eclipse-temurin-21
  script:
    - mvn $MAVEN_CLI_OPTS test

deploy:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl set image deployment/my-app my-app=registry.gitlab.com/my-app:$CI_COMMIT_SHA
  environment:
    name: production
  only:
    - main
```

## 2.6 常见的部署策略有哪些？各自优缺点是什么？
1. **滚动更新(Rolling Update)**：
   - 方式：逐个替换实例，新实例启动成功后再替换下一个
   - 优点：零停机、资源开销小
   - 缺点：回滚速度慢、短暂时间存在新旧版本并存

2. **蓝绿部署(Blue-Green)**：
   - 方式：维护两套环境，部署到闲置环境，验证后切换流量
   - 优点：即时回滚只需切流量、对用户零影响
   - 缺点：需要双倍资源、数据库兼容性难处理

3. **金丝雀发布(Canary)**：
   - 方式：小比例流量导到新版本，逐步增大比例
   - 优点：影响范围可控、真实流量验证、风险低
   - 缺点：实施较复杂、期间新旧版本并存

## 2.7 如何实现 CI/CD 中的自动回滚？
1. **Kubernetes 场景**：通过 `kubectl rollout undo` 回滚到上一个版本
```bash
kubectl rollout undo deployment/my-app
kubectl rollout status deployment/my-app
```
2. **蓝绿部署**：回滚仅需将流量切回蓝色环境
3. **金丝雀发布**：降低或清零新版本流量
4. **流水线策略**：CD 流水线部署后监控指标（错误率、响应时间），超过阈值自动触发回滚
5. Jenkins post 段：
```groovy
post {
    failure {
        sh 'kubectl rollout undo deployment/my-app'
    }
}
```

## 2.8 什么是制品管理？常用的工具有哪些？
制品(Artifact)是软件从源代码编译/打包/构建出来的可部署产物（如 jar 包、Docker 镜像、npm 包）。制品管理仓库统一存储版本化的制品，管理依赖和发布。

常用工具：
- **Nexus**：支持 Maven/Java、Docker、npm、PyPI、NuGet 等，常用作 Maven 私服
- **Artifactory (JFrog)**：通用制品仓库，企业功能强大，支持几乎所有包格式
- **Harbor**：专注容器镜像的仓库，开源、支持漏洞扫描、镜像复制、多项目隔离

优势：统一管理、版本可追溯、加速下载（局域网缓存）、减少对外部公网仓库的依赖。

## 2.9 如何将 SonarQube 集成到 CI 流水线？
1. 在 SonarQube 服务器创建项目，获取 token 和 project key
2. 在 Jenkins 中安装 SonarQube Scanner 插件，配置 SonarQube 服务器地址
3. 在 Jenkinsfile 或 settings.xml 中配置：
```groovy
stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv('SonarQube') {
            sh 'mvn sonar:sonar -Dsonar.projectKey=my-project -Dsonar.host.url=https://sonar.example.com'
        }
    }
}
stage('Quality Gate') {
    steps {
        timeout(time: 1, unit: 'HOURS') {
            waitForQualityGate abortPipeline: true
        }
    }
}
```
4. 也可在 Maven pom.xml 中配置 sonarqube plugin 属性或在命令行传参
5. SonarQube Quality Gate 设定门槛（如覆盖率≥80%、0 阻断 Bug），不达标则流水线失败，阻止部署

## 2.10 容器化 CI/CD 的典型流程是怎样的？
1. 开发者 push 代码到 Git
2. CI 触发：
   - checkout 代码
   - 单元测试
   - 编译构建（mvn package）
   - 代码质量扫描（SonarQube）
   - 构建 Docker 镜像（docker build）
   - 推送镜像到 Harbor/Registry（docker push）
   - 更新 K8s 清单文件中镜像 Tag
3. CD 触发（或手动审批后）：
   - `kubectl apply -f deployment.yaml`
   - 验证部署状态 `kubectl rollout status`
   - 健康检查（HTTP 探针、指标）
   - 如失败 → 自动回滚 `kubectl rollout undo`
   - 通知（钉钉、邮件、Slack）

## 2.11 Jenkins、GitHub Actions 和 GitLab CI 有什么优缺点？
| 工具 | 优点 | 缺点 |
|------|-----|------|
| Jenkins | 高度灵活、插件丰富（1800+）、自托管安全可控、社区庞大 | 配置复杂、UI 老旧、维护成本高、需自己管理服务器 |
| GitHub Actions | 与 GitHub 深度集成、Marketplace 生态好、托管执行免维护、YAML 简洁 | 受限于 GitHub 仓库、开源项目使用较多但企业定制较少 |
| GitLab CI | 与 GitLab 深度集成、Auto DevOps、内置 Registry、一体 DevSecOps | 需自托管 GitLab Runner、Community 版功能有限 |

## 2.12 什么是 Review App？GitLab 如何实现？
Review App 是 GitLab 的一项功能，每当创建 Merge Request 时，自动部署一个临时环境用于预览和评审该 MR 的代码变更。MR 关闭或合并后，临时环境自动销毁。

```yaml
review:
  stage: deploy
  script:
    - kubectl create ns review-$CI_COMMIT_REF_SLUG
    - kubectl apply -f k8s/app.yaml -n review-$CI_COMMIT_REF_SLUG
  environment:
    name: review/$CI_COMMIT_REF_NAME
    url: https://$CI_COMMIT_REF_SLUG.review.example.com
    on_stop: stop_review
  only:
    - merge_requests

stop_review:
  stage: deploy
  script:
    - kubectl delete ns review-$CI_COMMIT_REF_SLUG
  environment:
    name: review/$CI_COMMIT_REF_NAME
    action: stop
  when: manual

### 13. 项目的几种发布方式和各自特点？
| 发布方式 | 特点 | 适用场景 |
|---------|------|---------|
| JAR部署 | 直接运行JAR包，依赖宿主机JRE和基础环境（如配置、日志目录），部署简单直接，无额外抽象层 | 小型项目、开发测试环境、资源受限场景 |
| Docker部署 | 将应用及依赖打包为镜像，解决"在我机器上能跑"的环境一致性问题，镜像可移植性强，隔离性好。但大规模多容器时管理复杂（需搭配Compose等） | 中型项目、微服务起步、需要环境一致性 |
| K8s部署 | 容器编排平台，支持流量监控、副本弹性扩缩(HPA)、滚动更新/热部署、CI/CD集成、声明式配置管理(YAML可配置调优参数如JVM/连接池等)、服务自愈、多环境管理 | 生产级大规模微服务、需要高可用和弹性伸缩 |

三者演进关系：JAR → Docker → K8s，每层解决上一层的核心痛点（环境依赖 → 多容器管理 → 编排自治）。
```
