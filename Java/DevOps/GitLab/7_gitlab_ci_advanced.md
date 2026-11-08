# 1. 进阶能力的出发点：控制"何时跑、怎么编排、如何复用"

基础篇的流水线是"无条件全跑"。真实项目里要解决三类问题：

- **条件**：什么分支/什么文件变更/什么来源才跑？（`workflow` 与 `rules`）
- **编排**：阶段串行太慢，能否按依赖图并行？（`needs`、DAG）
- **复用**：几十个仓库不想各抄一份 yml？（`include` + 模板库）

# 2. workflow:rules：流水线级总闸

```yaml
# 只允许：MR 触发 + 主干 push + Tag 发布，其余来源不建流水线
workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_TAG
```

`workflow` 决定**整条流水线是否创建**，写在它之前的 job 级 rules 都白搭。最常见的收益：避免"push feature 分支同时跑分支流水线和 MR 流水线"的重复消耗。

# 3. rules：Job 级条件（取代 only/except）

`only/except` 是旧语法，表达能力弱且不可组合，新项目一律用 `rules`：

```yaml
unit-test:
  script: mvn test
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'

build-docs:
  script: make docs
  rules:
    - changes:                              # 只有文档目录变化才跑
        - docs/**/*
        - mkdocs.yml

nightly-e2e:
  script: ./e2e.sh
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule" && $NIGHTLY == "true"'
      variables: { PROFILE: "nightly" }    # 规则里还能附默认变量
```

高频条件字段：

| 字段 | 语义 |
|------|------|
| `if` | 变量表达式，支持 `=~ /regex/` |
| `changes` | 相对上一次流水线，指定 glob 有文件变更 |
| `exists` | 仓库里存在指定文件 |
| `when` | `on_success/on_failure/manual/delayed` |

**设计考量**：`rules` 自上而下**第一条命中即生效**；全都不命中则 job 不创建（注意与 `only` 默认行为差异——一个 yml 里只要出现 rules 且 job 无匹配规则，该 job 直接消失）。

# 4. needs 与 DAG：跳出 Stage 串行

默认按 stage 串行：test 全部完成才能进 deploy。`needs` 把流水线变成依赖图（DAG），**只等自己需要的上游**：

```yaml
stages: [build, test, deploy]

build-service-a:
  stage: build
  script: make build-a

build-service-b:
  stage: build
  script: make build-b

test-service-a:
  stage: test
  needs: [build-service-a]     # 不等 build-service-b
  script: make test-a

deploy-all:
  stage: deploy
  needs: [test-service-a, test-service-b]   # 精确依赖，替代"等整个 test 阶段"
  when: manual
```

| 对比 | stage | needs |
|------|-------|-------|
| 并行度 | 仅阶段内并行 | 打破阶段边界，无依赖即并行 |
| 下载 artifacts | 默认下载所有上游 | 只下载 needs 声明的上游 |
| 失败传播 | 上游阶段失败全停 | 只阻断依赖它的分支 |
| 跨项目 | `trigger` 下游 | `needs: [{project: "group/lib"}]` 等下游项目 job |

> **注意**：混用时 job 的 `needs` 必须显式列出跨阶段依赖，否则会出现"build 还没完 deploy 就开跑"的错觉；`needs` 指向的 job 若被 rules 裁掉会导致本 job 创建失败，用 `needs: [{job: "build-service-b", optional: true}]` 声明可选依赖。

# 5. parallel:matrix：一次声明跑整个矩阵

```yaml
integration-test:
  stage: test
  parallel:
    matrix:
      - JDK: [11, 17, 21]
        DB:  [mysql, postgres]
  image: maven:3.9-eclipse-temurin-$JDK
  services:
    - name: db:$DB                   # 数据库服务同样按矩阵取值
  script: mvn verify -P$DB
```

- 矩阵展开成 3×2=6 个并行 job，编号 `$CI_NODE_INDEX/$CI_NODE_TOTAL` 可用于分片测试。
- 排除组合：`rules: - if: '$DB == "postgres" && $JDK == "11"' when: never`。
- `parallel: N`（数字）用于**分片**：配合测试插件按 index 拆文件。

# 6. include：模板化与复用

多仓库共享 CI 的正确答案：把公共模板放进一个 **模板仓库**，业务仓库 `include` 它：

```yaml
# 业务仓库 .gitlab-ci.yml
include:
  - project: 'platform/ci-templates'        # 方式1：另一个仓库
    file: '/templates/java-service.yml'
    ref: 'v2.3.0'                           # 固定版本，避免上游模板变更击穿全公司
  - local: '.gitlab/ci-local.yml'           # 方式2：本仓库文件
  - remote: 'https://example.com/base.yml'  # 方式3：URL（谨慎，供应链）
  - template: Security/SAST.gitlab-ci.yml   # 方式4：GitLab 内置模板

# 冲突时业务仓库可覆盖模板里的同名变量/job
variables:
  MAVEN_CLI_OPTS: "--batch-mode -T 1C"
```

**设计考量**：

- include 文件按"后定义者覆盖"合并到同一命名空间，覆盖模板 job 时要保持 job 名完全一致。
- 模板仓库自己也要有 CI 与版本（semver tag），这是"内部基础设施即代码"的一部分。
- `trigger`/`child pipeline` 与 include 的边界：include 是**合并**（一次流水线内），trigger 是**另起一条流水线**（跨项目、可不同分支）。

# 7. 父子流水线（Parent-Child）

Monorepo 或"平台定义流程、业务只提供配置"的组织常用：

```yaml
# 根流水线（父）：只做编排
child-backend:
  trigger:
    include:
      - local: backend/.gitlab-ci.yml
    strategy: depend        # 父等子结果，失败传播
  rules:
    - changes: [backend/**/*]

child-frontend:
  trigger:
    include:
      - local: frontend/.gitlab-ci.yml
    strategy: depend
  rules:
    - changes: [frontend/**/*]
```

效果：根目录一个精简入口，各子项目维护自己的完整流水线；GitLab 页面聚合显示。对比 include 单流水线：子流水线有独立变量与状态，适合**按目录变更只触发相关子系统**的 Monorepo 场景。

# 8. 依赖图与资源优化组合拳

| 手段 | 效果 |
|------|------|
| `rules: changes` | 文档/测试/部署只跑相关变更，省 30-70% job |
| `cache` + `ARTIFACTS` 策略 | Maven 仓库缓存 + 小产物快传 |
| `interruptible: true` | 新 commit 到来自动取消旧流水线（MR 期间尤其省资源） |
| `resource_group` | 同一资源的 job 互斥（防止并发部署互踩） |
| `retry` | 基础设施抖动自动重试（`max: 2, when: runner_system_failure`） |
| `timeout` | 给 job 设上限，避免僵尸占坑 |

```yaml
test:
  interruptible: true
  resource_group: staging        # 部署串行化
  retry:
    max: 2
    when: runner_system_failure
  timeout: 30m
```

# 9. 跨项目流水线：制品上游触发

```yaml
downstream:
  trigger:
    project: platform/base-image-builder   # 另一个项目
    branch: main
```

上游（基础镜像、SDK）发布后自动触发下游（所有依赖它的服务重建）。配合 `needs: [{project: ..., job: ...}]` 还能直接引用上游项目的 job 状态。

# 10. 小结

| 要点 | 一句话 |
|------|--------|
| 两层条件 | workflow:rules 管"要不要建流水线"，rules 管"要不要建这个 job" |
| only/except | 旧语法，新代码一律 rules |
| needs | 把 stage 串行变成 DAG，精确依赖 + 精确下载 |
| matrix/parallel | 版本矩阵与分片并行 |
| include | 全公司 CI 模板化，模板仓库要版本化 |
| 父子流水线 | Monorepo/平台化的解耦编排 |
| 省资源 | changes、interruptible、resource_group、retry 是日常四件套 |

CI 语法侧到这就够用了；下一篇看"谁来跑"——Runner 的注册、executor 选型与伸缩。
