# 1. GitLab CI 的定位与整体心智模型

GitLab CI 是内置在 GitLab 里的 CI/CD 引擎，配置以 `.gitlab-ci.yml` 的形式**和代码放在同一个仓库**。它和 Jenkins 最大的使用差异：

| 维度 | GitLab CI | Jenkins |
|------|-----------|---------|
| 配置载体 | `.gitlab-ci.yml`（与代码同版本） | Jenkinsfile / UI 配置 |
| 执行器 | Runner（注册制，官方维护） | Agent（插件式连接） |
| 触发 | Git 事件原生触发，零配置 | 需装 SCM 插件、配 webhook |
| 结果呈现 | MR/提交页内嵌状态 | 跳转到 Jenkins 界面 |
| 学习曲线 | 一套 yml 语法 | Groovy + 插件矩阵 |

心智模型一句话：**"在某个 Git 事件上，用 YAML 声明一组 Job，交给带对应标签的 Runner 在容器里执行"**。

# 2. 核心对象：Pipeline、Stage、Job

```text
Pipeline（一次流水线，对应一次 commit/MR）
 ├── Stage: build        ← 阶段间串行，按数组顺序执行
 │    ├── Job: compile   ← 阶段内 Job 默认并行
 │    └── Job: package
 ├── Stage: test
 │    ├── Job: unit-test
 │    └── Job: e2e-test
 └── Stage: deploy
      └── Job: deploy-staging
```

- **Pipeline**：一次完整执行，状态 = 所有 Job 状态的聚合。
- **Stage**：逻辑分组，定义顺序；Job 属于哪个 Stage 由 `stage:` 字段决定，未声明则进入同名默认阶段或直接失败（不在 `stages` 列表里的 stage 名会报错）。
- **Job**：最小执行单元，必须至少有 `script`。生命周期状态：`created → pending → running → success/failed/canceled/skipped/manual`。

# 3. 最小可用的 .gitlab-ci.yml

仓库根目录放这个文件，push 即触发：

```yaml
# .gitlab-ci.yml —— Java 项目最小流水线
stages:
  - build
  - test

compile:
  stage: build
  image: maven:3.9-eclipse-temurin-17   # 容器镜像，Runner 为 docker executor 时生效
  script:
    - mvn -B -DskipTests package
  artifacts:
    paths:
      - target/*.jar                    # 产出物传给下一个 Job

test:
  stage: test
  image: maven:3.9-eclipse-temurin-17
  script:
    - mvn -B test
  artifacts:
    when: always                        # 失败也要（报告有用）
    reports:
      junit: target/surefire-reports/TEST-*.xml   # MR 页面直接展示失败用例
```

**设计考量**：

- `image` 只对 docker executor 的 Runner 有效；shell executor 时依赖本机环境——团队标准做法是统一 docker Runner + 镜像化构建环境，保证"构建可复现"。
- `-B`（批处理）不是可选项：Maven 交互模式在 CI 里可能卡住。
- `junit` 报告让"测试失败原因"直接出现在 MR 页面，省一半沟通。

# 4. 触发机制：什么时候会产生 Pipeline

| 事件 | 默认行为 |
|------|---------|
| push 到任意分支 | 分支流水线 |
| 创建/更新 MR | MR 流水线（推荐主用，见第 7 篇 rules） |
| Tag | 标签流水线，常用于发布 |
| 定时（Schedule） | Cron 表达式 + 变量注入，夜构建/安全扫描 |
| Web/手动/API | 手动触发、trigger job、下游项目 |
| 上游流水线完成 | `needs: pipeline` 跨项目触发 |

一个 commit 同时触发"分支 + MR"两条流水线是新手最常见的浪费，用 `workflow:rules` 二选一（第 7 篇详解）。

# 5. 内置变量与上下文

Runner 执行 Job 时会注入几十个工作变量（`CI_*`），高频的记这几个：

| 变量 | 含义 | 典型用法 |
|------|------|---------|
| `CI_COMMIT_SHA` / `CI_COMMIT_SHORT_SHA` | 当前 commit | 镜像 tag、制品命名 |
| `CI_COMMIT_BRANCH` / `CI_COMMIT_TAG` | 分支名/标签 | 区分部署目标 |
| `CI_DEFAULT_BRANCH` | 项目默认分支 | "只在主干构建基线镜像" |
| `CI_PIPELINE_SOURCE` | 触发来源 | rules 判断 push/mr/schedule |
| `CI_PROJECT_DIR` | 源码检出目录 | 引用相对路径 |
| `CI_MERGE_REQUEST_IID` | 当前 MR 号 | 关联评论、制品命名 |
| `CI_REGISTRY` / `CI_REGISTRY_IMAGE` | 内置镜像仓库地址 | 构建推送镜像 |

自定义变量三个层级：**实例级 → 群组级 → 项目级**（Settings → CI/CD → Variables），同名就近覆盖；敏感值勾选 **Masked + Protected**（只对受保护分支可见、日志中打码）。

# 6. 日志、产物与缓存：Job 之间怎么"接力"

三个容易混淆的机制：

| 机制 | 生命周期 | 用途 | 配置字段 |
|------|---------|------|---------|
| **artifacts** | 跟随 Pipeline（有保留期） | 把构建产物/报告传给后续 Job、供下载 | `artifacts:paths/reports/expire_in` |
| **cache** | 跨 Pipeline，Runner 本地/对象存储 | 加速依赖下载（`.m2`、`node_modules`） | `cache:paths:key/policy` |
| **dependencies** | 控制本 Job 下载哪些上游 artifacts | 减少无谓下载 | `dependencies: [compile]` |

```yaml
# Maven 依赖缓存的正确姿势
variables:
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"
  MAVEN_CLI_OPTS: "--batch-mode"

cache:
  key: "$CI_COMMIT_REF_SLUG"          # 按分支隔离缓存
  paths:
    - .m2/repository                   # 必须落在项目目录内才能被缓存

compile:
  stage: build
  image: maven:3.9-eclipse-temurin-17
  script: mvn $MAVEN_CLI_OPTS -DskipTests package
```

> **注意**：cache 是**尽力而为**的加速，不保证存在（冷 Runner、被清理都会 miss），产物传递必须用 artifacts，不能拿 cache 当数据通道。

# 7. 手动任务与发布流程

```yaml
deploy-production:
  stage: deploy
  image: bitnami/kubectl:1.29
  script: kubectl apply -f k8s/ && kubectl rollout status deploy/app
  when: manual                        # 不自动跑，页面上点"播放"
  environment:
    name: production
```

`when: manual` 是 GitLab 里"持续交付（人工确认部署）"的标准表达：CI 阶段全自动，生产部署留一个按钮。

# 8. Auto DevOps：一条注释都不写的流水线

项目里**没有** `.gitlab-ci.yml` 时，GitLab 会套用内置的 Auto DevOps 模板（检测到语言后自动 build/test/code-scan/security-scan/review/deploy/staging/production）。

**使用场景对照**：

- **适合**：体验平台能力、极小项目、标准化部署（K8s）。
- **不适合**：绝大多数真实 Java 项目——它依赖内置 K8s 集成与自动探测，黑盒、不可控、跑不满你的场景。

> **行业实践**：生产团队几乎都会关掉 Auto DevOps（Settings → CI/CD → Auto DevOps 取消勾选），用显式 `.gitlab-ci.yml`。了解它的存在即可，把 auto-devops 当"模板灵感来源"而不是运行方案。

# 9. 调试流水线的实用手段

| 手段 | 做法 |
|------|------|
| CI Lint | Web: Build → Pipeline editor 右侧 lint；CLI: `glab ci lint`。改 yml 先过 lint |
| 干跑 | `.gitlab-ci.yml` + rules 模拟：编辑页选择分支/MR 上下文预览 job 是否创建 |
| 看排队原因 | Job 页 "Why is this job pending?" 悬停提示（无 Runner 匹配/配额） |
| 本地复现 | `gitlab-runner exec docker compile`（旧版）或干脆 docker 里手跑 script |
| 全量变量 | 加一个临时 job `script: env | sort` 看注入变量 |

# 10. 小结

| 要点 | 一句话 |
|------|--------|
| 对象模型 | Pipeline = Stage（串行）× Job（并行），配置即代码 |
| 环境一致性 | docker executor + `image:` 声明构建环境 |
| 接力机制 | artifacts 传产物、cache 加速、两者不可互替 |
| 变量体系 | CI_* 内置 + 三层自定义，敏感值 Masked/Protected |
| 发布控制 | when: manual 实现"自动化到预发，人工点生产" |

基础篇解决了"能跑"，下一篇解决"跑得巧"：rules 条件、needs DAG、矩阵、include 复用与企业级模板化。
