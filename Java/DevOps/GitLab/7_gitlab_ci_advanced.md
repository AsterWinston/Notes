# 1. 进阶能力的出发点：控制何时跑、怎么编排、如何复用

> **前置**：[第 6 篇 CI 入门](6_gitlab_ci_basics.md)
> **本篇目标**：解决 CI 配置膨胀与多项目协同两大问题 —— `extends` / `!reference` / `include` / **CI/CD Components** 四级复用体系，DAG 与多项目编排，矩阵并行，失败策略，流水线报告
> **对应实际工作**：维护团队级 CI 模板仓库、拆分单体流水线、微服务全量构建

CI 配置失控的表现永远一样：`.gitlab-ci.yml` 长到 800 行、改一处要同步 20 个仓库、新人不敢动。本篇按"**先选对复用机制 → 再控制编排 → 最后管失败与观测**"三段展开。

## 1.1 现场表现 → 对应机制（按症状查目录）

| 现场表现 | 缺的是 | 本篇位置 |
|----------|--------|----------|
| 改一次模板要同步 20 个仓库，漏一个就红一条流水线 | **带版本的复用** | §2 选型、§5 include + `ref`、§6 Components + `@version` |
| yml 800 行、九成是复制粘贴，谁也不敢删 | **参数化** | §3 `extends`、§6.2 `spec:inputs` |
| 同一份脚本片段要在 5 个 job 里各追加几行 | **合并语义** | §3.2 深度合并、§4 `!reference` 追加 |
| MR 流水线 6 个 job 排队等全阶段，反馈 12 分钟 | **DAG** | §7 `needs`（含可选依赖、`needs: []`） |
| 微服务全量发布，下游不知道上游镜像好没好 | **跨项目编排** | §8 父子流水线、`strategy: depend`、`trigger` |
| 要跑 JDK 8/11/17 × MySQL/PG 的组合，写 6 个 job | **矩阵** | §9 `parallel:matrix` |
| 一个 flaky 测试拖垮整条 MR，团队开始习惯性无脑重跑 | **失败策略** | §10 `retry` / `allow_failure` / `on_failure` |
| 没人能说清"CI 一周花多少钱、哪个 job 最浪费" | **观测** | §11 报告、§12 缓存与 `inputs` 省资源 |

## 1.2 阅读与落地顺序（反过来必翻车）

```text
① 先定复用边界（§2）——哪些是平台能力、哪些是业务差异
② 再重构编排（§7/§8）——把串行拆成 DAG、把单体流水线拆成父子
③ 最后固化失败与成本（§10–§12）——retry 策略、报告、缓存纪律
```

最常见的翻车顺序是"**先参数化**"：结构还没理清就抽成 20 个模板/组件，结果每个业务项目都要覆盖一半参数，模板作者被 20 个项目的差异需求追着改 —— 这属于**边界画错**，不是机制用错。判断标准很简单：**如果一个参数被 80% 的项目改成同一个值，那它就不该是参数，应该直接进模板**。

## 1.3 本篇与前一篇的分界

第 6 篇解决"一条流水线怎么跑通"（关键字、阶段、变量、Artifact、环境）；本篇解决"**几十条流水线怎么一起维护**"。所以本篇的每个机制都以"仓库/版本/参数"为主语，而不是以"job"为主语。

---

# 2. 先选对复用机制：四种手段的边界

| 手段 | 解决什么 | 生效范围 | 版本可控 | 典型误用 |
|------|----------|----------|----------|----------|
| `extends` | 同文件内**减少重复字段** | 单文件（含 include 进来的文件） | ❌ 无版本概念 | 用它跨仓库同步（改不动别人） |
| `!reference` | **片段级复用**（一段 script/变量/rules） | 单文件 + 已 include 的文件 | ❌ | 用它替代 `include` 引整份配置 |
| YAML 锚点 `&/*` | 语法级复制粘贴 | **同一份 YAML 文档内** | ❌ | 跨 include 使用（拿不到）、当 `extends` 用（合并语义不同） |
| `include` | **跨文件/跨仓库/跨 URL 引入整块配置** | 项目、组、外部 URL | 靠 `ref:` | 不锁 ref，同名 key 被静默覆盖 |
| **CI/CD Components** | 跨仓库**带版本、带参数**的自包含能力包 | 组级/实例级 Catalog | ✅ `@1.2.0` | 在组件里写 `default:`、硬编码 `stage` |

**选择顺序**：先 `extends`（同文件）→ 再 `include`（跨仓库、量小）→ 需要"版本 + 参数 + 一处升级处处生效"时升级为 Component。

---

# 3. extends 与隐藏模板：合并语义

## 3.1 写法与多父合并

```yaml
# 隐藏模板：以 . 开头的顶层 key 不会被当成 Job
.maven-base:
  image: maven:3.9-eclipse-temurin-17
  variables:
    MAVEN_ARGS: "-B -ntp"
    LANG: "C.UTF-8"
  before_script:
    - java -version
  cache:
    key:
      files: [pom.xml]
    paths: [.m2/repository]
  artifacts:
    expire_in: 7 days
    reports:
      junit: target/surefire-reports/TEST-*.xml

.unit-test:
  extends: .maven-base          # 支持多级；也可写列表 [.base-a, .base-b]，后者覆盖前者
  stage: test
  script:
    - mvn $MAVEN_ARGS test

test:
  extends: .unit-test
  variables:
    MAVEN_ARGS: "-B -ntp -T 1C"   # 覆盖父级同名变量，其它变量继承
  before_script:
    - mvn -v                      # ⚠️ 数组字段是**整体替换**，父级的 java -version 不见了
```

## 3.2 三条必须记住的合并规则

1. **标量与哈希递归合并**：`variables`、`artifacts:reports`、`cache:key` 这类 map 逐字段合并，子级只覆盖写到的那个 key。
2. **数组整体替换**：`script`、`before_script`、`after_script`、`cache:paths`、`artifacts:paths`、`tags` 一律**不追加**。要"追加"就显式用 `!reference`（第 4 节）。
3. **不能循环引用**：`extends` 成环报 `jobs:xxx: 'extends' cycle definition`；多父时**列表靠后的赢**。

**坑**：`extends` 只合并字段，**不会让隐藏模板自己变成 Job**；同时继承来的 `stage` 可能不是你要的位置 —— 规范做法是**每个 Job 显式写 `stage:` 与 `needs:`**。

---

# 4. !reference：把"替换"变成"追加"

```yaml
.deploy-base:
  deploy_steps:
    - ./ci/precheck.sh
    - ./ci/apply.sh
    - ./ci/smoke.sh

deploy-staging:
  extends: .deploy-base
  script:
    - echo "start deploy"
    - !reference [.deploy-base, deploy_steps]     # 把父级数组内联进来
    - echo "done"

rules-only-prod:
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'

deploy-prod:
  extends: .deploy-base
  rules:
    - !reference [.rules-only-prod, rules]
    - if: $CI_PIPELINE_SOURCE == "schedule"
      when: never
```

- 路径是**任意 YAML 路径**，不要求以隐藏 Job 开头：`!reference [.job, artifacts, reports]` 合法。
- **可以引用 `include` 进来的文件里的片段**，但被引用文件必须已经通过 `include` 加载；**不能引用组件内部片段**。
- 跨文件形式：`!reference ["templates/build.yml", "base", "before_script"]`（需同时 include 该文件）。
- **写错路径是运行期才炸**（`jobs:xxx:script received null value` / `Invalid reference`），本地 lint 未必拦得住 —— 引用完必须跑一次流水线验证。

> **锚点 vs `!reference`**：YAML 锚点在 `include` 场景下不共享，也不参与 GitLab 的合并语义，官方推荐 `extends` / `!reference`。**已有锚点不必强行清理，新增一律用后两者。**

---

# 5. include：模板化与复用

## 5.1 五种来源

```yaml
include:
  # 1) 本仓库文件
  - local: '/ci/templates/backend.yml'

  # 2) 其它仓库的文件（最常用：团队模板库）
  - project: 'platform/ci-templates'
    ref: 'v2.4.1'                       # 必须锁定：branch / tag / commit SHA
    file: ['/templates/java/maven.yml', '/templates/security/dependency-scan.yml']

  # 3) GitLab 内置模板
  - template: Security/SAST.gitlab-ci.yml

  # 4) 远程 URL（自建环境谨慎使用，等于把流水线控制权交给外部）
  - remote: 'https://gitlab.example.com/ci/dast.gitlab-ci.yml'

  # 5) 组件（第 6 节）
  - component: $CI_SERVER_FQDN/platform/ci-templates/maven-build@1.2.0
    inputs:
      profile: prod

  # 条件包含：18.1 起用 rule，取代已冻结的 only/except
  - local: ci/nightly.yml
    rule: '$CI_PIPELINE_SOURCE == "schedule"'
```

## 5.2 展开顺序与冲突规则

1. **`include` 在 `workflow`/`stages` 之后、Job 求值之前展开**；同名 Job 冲突时**调用方赢**（Job 级 `include` 更是"包含内容被该 Job 覆盖"）。这是"我明明改了模板，为什么流水线没变"的第一答案 —— **你被本地同名 Job 覆盖了**。
2. **`ref` 不写就用模板库默认分支**，模板库推坏提交会让全公司流水线同时炸。**锁 tag，或至少锁 commit SHA，并用机器人提 MR 升级。**
3. **数量上限 150 个文件**（含递归），超限报 `Too many includes`。
4. **路径/ref 必须存在**：`include:local` 指向不存在的文件报 `Included file <path> not found in ...`；`include:project` 的 `ref` 分支被删则报 404 类错误。**新项目骨架最常留下的就是这个"引用了还没建的模板路径"。**
5. **业务仓库仍应保留自己的 `workflow` 与 `stages`**，模板只提供 Job；否则模板改阶段顺序会打乱所有业务仓库。

## 5.3 include 的 inputs：轻量参数化

```yaml
# 被包含文件 ci/templates/backend.yml
spec:
  inputs:
    jdk:
      default: "17"
---
build:
  stage: build
  image: maven:3.9-eclipse-temurin-${[[ inputs.jdk ]]}
```

```yaml
# 调用方
include:
  - local: ci/templates/backend.yml
    inputs: { jdk: "21" }
```

`inputs` 只解决"传参"，**不解决版本管理与可发现性**。需要版本时做组件（第 6 节），别继续给 include 加参数。

---

# 6. CI/CD Components：带版本、带参数的能力包

组件本质是"仓库 `templates/` 目录下一个带 `spec:` 段的 GitLab CI YAML 文件"。

## 6.1 目录结构（顶层必须叫 `templates`）

```text
platform/ci-templates
├── README.md                        # Catalog 页面会展示
├── .gitlab-ci.yml                   # 组件仓库自己的流水线（负责发布）
└── templates/
    ├── maven-build.yml              # 简单组件：templates/<name>.yml
    └── docker-build/
        └── template.yml             # 复杂组件：templates/<name>/template.yml（可带 README）
```

## 6.2 组件写法

```yaml
# templates/maven-build.yml
spec:
  description: "构建并测试 Maven 项目"
  model:
    inputs:
      version: {}                    # 由平台注入组件版本，便于自我标注
  inputs:
    profile:
      description: "Maven profile"
      default: "ci"
      options: ["ci", "prod"]        # options 需 GitLab 18.0+
    jdk:
      description: "JDK major version"
      regex: "^(11|17|21)$"
      default: "17"
---
maven-build:
  image: maven:3.9-eclipse-temurin-${[[ inputs.jdk ]]}
  variables:
    MAVEN_ARGS: "-B -ntp -P${[[ inputs.profile ]]}"
  script:
    - mvn $MAVEN_ARGS -DskipTests package
    - mvn $MAVEN_ARGS test
  artifacts:
    when: always
    expire_in: 7 days
    paths: [target/*.jar]
    reports:
      junit: target/surefire-reports/TEST-*.xml
```

**注意：组件里没有 `stage:`** —— 阶段归属交给引用方，这是组件设计的硬约束（见 6.4）。

## 6.3 引用语法

```yaml
---
include:
  - component: $CI_SERVER_FQDN/platform/ci-templates/maven-build@1.2.0
    inputs:
      jdk: "21"
      profile: prod
```

也可在 Job 内部做 Job 级 include（同 `include` 规则）。

## 6.4 六条必须记住的规则

1. **格式是 `$CI_SERVER_FQDN/<project path>/<component name>@<version>`**（用 `$CI_SERVER_FQDN` 让配置在测试/生产实例间可移植）。**不是**带协议或裸主机的 `gitlab.example.com/...` 写法。
2. **一个仓库最多 100 个组件**；一个组件文件可定义多个 Job。
3. **`spec:` 只允许 `description` / `inputs` / `component`（以及 `model.inputs`）**，**`spec:` 里不能写 `include`** —— 需要别的组件时只能在同一文件里再写一个 `--- include:` 块。`spec:` 完全为空会报 `Build component error: Spec must be a valid JSON schema`。
4. **不要在组件里写全局关键字**（`default`、`stages`、`workflow`）与**硬编码 `stage` / Job 名**：两个组件组合时后者会静默覆盖前者的 `default`，硬编码 stage 会让引用方的阶段模型被打乱。**需要可变就用 `inputs` 暴露。**（这正是第 15 篇"组件别硬编码 stage"的根因。）
5. **版本解析优先级：commit SHA > tag > branch**。`~latest` 与部分语义版本（`1.2`、`1`）**只对已发布到 Catalog 的组件生效**，且**不含 pre-release**。未发布的组件目录用 `~latest` 会得到 `Component '...@~latest' - content not found`。
6. **`$[[ inputs.x ]]` 在 YAML 解析阶段展开**，与 `$VAR`（shell 阶段）不同：**Job 名、`include` 目标、`rules` 的结构位置、正则**这些"解析期就要确定"的地方只能用 `$[[ ]]`；命令行参数用 `$VAR`。混用会得到 `unknown keys` 一类难以理解的错误。

## 6.5 发布到 Catalog（三步，顺序不能乱）

```text
1) Settings → General → CI/CD → 勾选 "CI/CD Catalog project"，
   并填好项目 Description 与 README（缺内容会导致发布流水线失败、Catalog 页无可读信息）
2) 打一个 **semver 格式**的 tag（v1.2.0 或 1.2.0）
3) 由流水线用 release: 关键字发布 —— Releases API 与 Web 创建 Release 都不能发布组件
```

```yaml
# platform/ci-templates/.gitlab-ci.yml
release-component:
  stage: release
  rules:
    - if: '$CI_COMMIT_TAG =~ /^v?\d+\.\d+\.\d+$/'
  release:
    tag_name: $CI_COMMIT_TAG
    description: |
      maven-build component $CI_COMMIT_TAG
      - 支持 JDK 11/17/21
```

发布后到 **Build → Pipelines → 组件仓库的流水线 → 组件名 → Releases** 确认版本列表，再在引用项目跑一次 lint。**引用方升级必须走 MR**（模板变更也要评审），描述里写清"影响面：N 个项目"。

## 6.6 自建实例使用 gitlab.com 上的公共组件

GitLab 提供 **GitLab Catalog Component Sync**：把外部组件同步到自建实例的 `groups/gitlab-catalog` 下。**前提较苛刻**：自建实例 GitLab 18.0+、启用内置容器 registry、同步项目的命名空间必须与 gitlab.com 上完全一致，且该能力为 Premium/Ultimate。条件不满足时，只能"镜像组件仓库 + 自己重新 release"，并**固定到 commit SHA**，避免每次流水线回源 gitlab.com。

---

# 7. needs 与 DAG：跳出 Stage 串行

```yaml
compile:
  stage: build
  script: [./ci/compile.sh]

# 形态 1：只等自己需要的 Job，不等整个阶段（最常见的 2-5 分钟收益来源）
unit-test:
  stage: test
  needs: [{ job: compile, artifacts: true }]
  parallel: 4

# 形态 2：可选依赖 —— 上游被 rules 裁掉时不要卡住
security-scan:
  stage: test
  needs: [{ job: compile, artifacts: true }]
  rules:
    - if: '$SAST_ENABLED == "true"'
pack:
  stage: package
  needs:
    - { job: unit-test, artifacts: true }
    - { job: security-scan, artifacts: true, optional: true }   # ← 关键

# 形态 3：无依赖，流水线创建即调度
lint:
  stage: test
  needs: []

# 形态 4：跨项目 needs（同一实例、当前令牌对上游有读权限）
ui-contract:
  stage: test
  needs:
    - { project: frontend/web, job: build-ui, ref: main, artifacts: true }
```

**边界与坑**：

- `needs` **不影响当前 Job 是否运行**，只影响它等多久 —— 文档明确写了这句，"让 Job 有条件运行"是 `rules` 的职责。
- **`needs` + `dependencies` 同时存在时 `dependencies` 被忽略**；要精确控制下载哪些产物，只用 `needs: [{job, artifacts: true|false}]`。
- **所有显式 `needs` 都被裁剪时，Job 回退到按 stage 等待**；给每项加 `optional: true`，或用 `rules:needs` 让依赖本身参与条件求值。
- `needs` 指向本流水线不存在的 Job 且未加 `optional` → 流水线创建失败，报错形如 `jobs:xxx:needs ... undefined needs` 并附带候选 job 名。
- **DAG 不等于更快**：`needs` 让 Job 早跑，但每个 `artifacts: true` 都是一次下载。测试只依赖编译产物就别下载 lint 报告（`artifacts: false`）。

---

# 8. 父子流水线与跨项目编排

## 8.1 根流水线只做编排

```yaml
# 父：只做编排与门禁
stages: [verify, downstream]

verify:
  stage: verify
  script: [./ci/verify.sh]

# 1) 手动桥接：MR 里"点一下再跑下游完整回归"
downstream-e2e:
  stage: downstream
  needs: ["verify"]
  trigger:
    project: qa/e2e-suite
    branch: main
    strategy: depend          # 下游失败 → 上游 Job 也失败（可阻塞 MR 合并）
    inherit:
      variables: false        # 默认 true：上游全部变量透传，子项目会静默取到"意外的值"
  variables:
    APP_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

# 2) 微服务全量：矩阵批量触发
trigger-all:
  stage: downstream
  parallel:
    matrix:
      - SERVICE: [svc-pay, svc-order, svc-user]
  trigger:
    project: "platform/apps/${SERVICE}"
    strategy: depend
    branch: production

# 3) 从文件读清单，规避 Job 数量上限
trigger-from-file:
  stage: downstream
  trigger:
    project: platform/apps
    list: |
      apps/svc-pay
      apps/svc-order@release/2.4
    inherit: { variables: false }
```

## 8.2 必须记住的语义

| 项 | 行为 |
|----|------|
| `strategy: depend` | 下游失败级联上游失败；**级联深度上限 5 层**，超出后不再传播 |
| `cascade_dependent_pipelines: true` | 取消上游时级联取消下游（GitLab 18.0+；17.x 用功能开关 `:ci_cascade_parent_pipelines_when_cancelled`） |
| `trigger: branch` | **子流水线只支持 branch**，写 `ref: <tag>` 会创建失败 |
| `parent_pipeline: null` | **故意解耦**，下游变为独立流水线（17.7+ 才支持这个显式写法） |
| `trigger: pipeline` | 触发已有、且 pipeline-source 为 `parent_pipeline` 的流水线 |
| 默认 ref 选择 | 上游优先用**同名 branch**；都没有时用**默认分支**（可能触发非预期流水线） |
| 可见性 | 上游用户看不到无权限的下游；MR 里"看起来少了 Job"通常是这个原因 |
| 重跑 | 子流水线与父流水线绑定，**不能从父流水线单独重跑子 Job**（重跑会带动非手动的子 Job） |
| `rules:policy`（18.0） | 按 **pipeline source** 控制"谁能触发这条流水线"，MR 里能看到被禁用的触发链；**不能与 `workflow:rules` 混用** |

## 8.3 跨项目触发的第一坑：同一个 SHA

`trigger: project` 复用**同一个 commit SHA**。如果下游仓库不包含这个 SHA，**下游流水线根本不会创建，上游 Job 却显示成功**。三种解法：

1. 显式 `trigger: branch`，让下游跑自己的分支（但拿不到上游代码，只适合"下游用自己的代码 + 上游产物"）。
2. **fork 流水线**：MR 流水线归属 fork 项目，fork 里含同一 commit（gitlab.com 常见，自建需 fork pipeline 设置）。
3. 保证下游含同一 SHA（同仓多分支同步 / 由同一次构建产物驱动），并用 `strategy: depend` 把"没创建下游"变成显式失败而不是静默通过。

> **`strategy: depend` 是治"静默成功"的开关**：任何多项目编排都应显式决定是否阻塞，否则跨项目流水线的失败会被隐藏。

---

# 9. parallel:matrix：一次声明跑整个矩阵

## 9.1 用法

```yaml
smoke:
  stage: test
  parallel:
    matrix:
      - OS: ["ubuntu:22.04", "rockylinux:9"]
        JDK: ["17", "21"]
        DB: ["mysql:8.0", "postgres:16"]     # 2×2×2 = 8 个 Job（笛卡尔积）
  image: maven:3.9-eclipse-temurin-$JDK
  services:
    - name: $DB
      alias: db
  script:
    - ./ci/smoke.sh "$OS" "$JDK" "$DB"
```

矩阵 Job 在 UI 中显示为 `smoke: [JDK: 17, DB: mysql:8.0]`，`needs` 引用时要写 `needs: [{ job: smoke, parallel: { matrix: [{ JDK: "17" }] } }]`。

## 9.2 组合爆炸与并发控制

- **矩阵值一律加引号**：`JDK: [17, 21]` 与字符串比较不一致；`DB: mysql:8.0` 不加引号时 YAML 把 `8.0` 解析成数字，镜像名变成 `mysql:8`。
- **矩阵 Job 共享 cache key 会互相覆盖** → 把矩阵变量放进 key：`cache: key: "maven-$JDK-$DB"`。
- 组合数与单流水线 Job 总数都有上限（`trigger: list` 也是 150），超限报 `exceeds the maximum allowed number of jobs`。控制手段按优先级：
  1. `parallel:matrix` + `exclude` 去掉无意义组合；
  2. `concurrency: { parallel: 5 }`（18.0，限制同时运行数量）；
  3. 拆到不同阶段用 `strategy: depend` 串起来；
  4. 把全量矩阵从"每次 push"降级为 `when: manual` 或夜间 `schedule`。

---

# 10. 失败处理：retry / allow_failure / on_failure

```yaml
flaky-prone:
  script: [./ci/e2e.sh]
  retry:
    max: 2
    when:
      - runner_system_failure      # Runner 挂了，重试安全
      - stuck_or_timeout_failure
      - worker_timeout
      - api_failure                # GitLab API 抖动
      - script_failure             # 用例失败也重试 = 掩盖不稳定，谨慎开
  # retry: never 可抵消 default.retry（集成测试明确不要重试时用）

lint-nonblocking:
  script: [./ci/lint.sh]
  allow_failure:
    exit_codes: [0, 1]             # 退出码 1 视为非阻断，其它失败仍算失败

rollback:
  stage: .post
  when: on_failure                 # 前序 Job 失败时，页面出现这个手动 Job
  resource_group: production
  environment: { name: production, action: access }
  rules:
    - if: '$CI_COMMIT_TAG'
      when: manual
      allow_failure: true
  variables:
    GIT_STRATEGY: none             # 回滚不需要代码
  script:
    - ./ci/rollback-to.sh "$CI_COMMIT_PREVIOUS_TAG"
```

**三条纪律**：

1. **基础设施类失败（`runner_system_failure` / `api_failure` / `stuck_or_timeout_failure`）重试无害，应作为团队默认。**
2. **`retry: when: script_failure` 会把真实失败变成"随机通过"** —— 只在已定位不稳定用例并计划修复时临时开，并**在 Job `description` 里写明到期时间**。
3. **不要用 `when: on_failure` 做自动回滚**：它只是让 Job **出现在页面上**（配合 `when: manual` 仍是"需要人点"）。自动回滚需要显式设计（`.post` 阶段 + `needs` + `resource_group`），详见第 9 篇。

---

# 11. 流水线报告：把数据从 Job 提取到 UI

```yaml
test:
  script:
    - mvn -B test
    - mvn -B jacoco:report
    - printf 'p95_latency_ms=%s\nbuild_seconds=%s\n' 128 74 > metrics.txt
    - printf 'IMAGE_TAG=%s\n' "$CI_COMMIT_SHORT_SHA" > build.env
  artifacts:
    when: always
    reports:
      junit: target/surefire-reports/TEST-*.xml      # MR 里直接看失败用例
      coverage_report:
        coverage_format: jacoco-xml                  # cobertura / clover / lcov 亦可
        path: target/site/jacoco/jacoco.xml
      codequality: gl-code-quality-report.json        # MR diff 上报新增问题
      metrics: metrics.txt                            # 趋势图（值必须是浮点）
      dotenv: build.env                               # 跨 Job / 跨项目传变量
      sbom: { cyclonedx: target/bom.json }
  coverage: '/Total.*?([0-9]{1,3})%/'                 # 正则必须含 1 个捕获组

quality-gate:
  needs: [{ job: test, artifacts: true }]
  script:
    - python3 ci/quality_gate.py --diff-coverage 80 --total-coverage 60 --fail-on-new-critical 0
```

要点：

- **报告与二进制分开发布**：把 `reports:` 聚合到单独 Job（`needs` + `artifacts`），可单独设过期策略，长期留存关键报告而丢弃大文件。
- **`junit` 是 diff coverage 的前提**：只上报覆盖率没有 junit，MR 里看不到"新增代码覆盖率"。
- **`metrics` 只接受浮点值**：写成 `128ms` 会让该指标被静默忽略 —— "趋势图空白"的第一原因。
- **`dotenv` 变量的传播边界**：只对**当前流水线内下游 Job**，以及**被 `trigger` 桥接的下游流水线**可用；**不会跨不相关流水线传播**。要跨流水线共享，用 API 查询或解析 artifact。
- **`codequality` 报告有体积上限**，超限静默不加载（UI 显示为空），别把整仓扫描结果塞进去。
- **覆盖率与用例已合并进 Quality report 面板**（旧 Coverage 看板停用）—— 找功能先看 Quality 页，别按老截图翻 UI。
- **`pages`**：`artifacts: paths: [public/]` + `publish: true`，产物上限 100MB、目录名必须是 `public/`，超限报 `Artifacts for pages jobs should be under 100 MB`。

---

# 12. 依赖图与资源优化的组合拳

```yaml
# Maven：依赖清单变了才失效；新分支从主干缓存兜底
cache:
  key:
    files: [pom.xml]
    prefix: maven-$CI_DEFAULT_BRANCH
  fallback_keys:
    - maven-$CI_DEFAULT_BRANCH
  paths: [.m2/repository]

# 只读型 Job：绝不回写，避免把脏缓存推给别人
lint:
  cache: { policy: pull, key: { files: [pom.xml] }, paths: [.m2/repository] }

# 跨流水线取件：部署直接用主干最近一次成功构建的产物，不重新编译
deploy:
  needs:
    - { job: package, ref: main, artifacts: true, pipeline: $CI_PIPELINE_ID }
```

- **`policy`**：`pull` 给消费者、`push` 给"夜间预热缓存"的生产者、`pull-push`（默认）给既要读又要写的构建 Job。**每个 Job 都回写缓存是成本浪费**。
- **`fallback_keys`** 是解决"新 feature branch 首次构建慢 10 倍"的标准手法。
- **`needs: [{project, job, ref, pipeline, artifacts}]`** 实现"上游产物触发下游使用"。**`pipeline` 只接受当前可解析的值**（如 `$CI_PIPELINE_ID`），写死数字会让流水线创建失败；指定别的分支时，该分支最新流水线必须含被指定的 Job。
- **不要用 `cache: untracked: true` / `artifacts: untracked: true` 图省事**：它们基于 `git status` 收集，极易把临时巨型目录一起打包。**明确列 `paths`。**

**性能优化顺序（收益从大到小）**：

1. **缓存**：`.m2/repository`、`node_modules` 纳入 `cache:key:files`（一次省 3-8 分钟）。
2. **DAG**：用 `needs` 把 test 从"等整个 build 阶段"改成"只等 compile"。
3. **`interruptible: true` + `workflow: auto_cancel`**：新提交不再排队跑旧提交，Runner 槽位及时回收。
4. **镜像**：固定 tag、`pull_policy: if-not-present`、依赖装进镜像层而不是 `before_script`。
5. **产物体积**：`expire_in` 30 天 → 7 天，`reports` 与 `paths` 分 Job。
6. **Runner 容量**（第 8 篇）。

```bash
# 10 秒定位最大瓶颈：各 Job 耗时与排队
curl -sS -H "PRIVATE-TOKEN: $GITLAB_PAT" \
  "$GITLAB_URL/api/v4/projects/$CI_PROJECT_ID/pipelines/$CI_PIPELINE_ID/jobs?per_page=100" \
| jq -r '.[] | select(.finished_at) | [(.duration // 0 | floor | tostring), .stage, .name] | @tsv' | sort -rn | head -20
```

> **区分"排队慢"与"执行慢"**：`started_at - created_at` 是排队时间，`duration` 是执行时间。前者大 → 容量/标签问题（转第 8 篇）；后者大 → 脚本问题（本篇）。

---

# 13. 身份与密钥：CI_JOB_TOKEN 不是万能令牌

| 用途 | 正确做法 | 为什么 |
|------|----------|--------|
| 克隆本仓库 | Runner 内置 git 凭据（默认可用）/ `CI_JOB_JWT_V2` | `CI_JOB_TOKEN` 已不用于克隆仓库 |
| 拉私有 Docker 镜像 / Maven 依赖 | `CI_JOB_TOKEN` + 目标项目 **Job token permissions** 入站授权 | 需显式授权，否则 401/403 |
| 部署到 K8s | Job `environment:` 绑定 Deployment variables，或 **ID Tokens（OIDC）** 换云厂商临时凭据 | 避免长期 `KUBE_TOKEN`；`id_tokens` 需在 **Settings → CI/CD → Token and secret access** 配置 provider，且 Job 绑定 `environment:` |
| 跨项目触发 / 读 API | **Project Access Token**（或 Group Access Token）+ 成员授权 | PAT 属于个人，离职即失效，权限范围通常远大于所需 |
| 下游变量控制 | `inherit: { variables: false }` | 否则下游静默继承上游变量导致行为变化 |

**两条与掩码/保护相关的硬事实**：

- **掩码约束**：单行、≥8 字符、仅 Base64 安全字符集。含空白的多行私钥、带 `:` 的 URL、含 `"` 的 JSON **不能**设 masked —— 改 Type: File，或读入后自行脱敏。
- **protected 变量只在受保护 ref 上存在**，而 **MR 流水线（`merge_request_event`）默认不是受保护 ref**。所以"生产密钥在 MR 流水线取不到"是设计如此。正确解法是**拆分流水线**（MR 只跑构建测试，部署走分支/tag 流水线），而不是取消 protected。

---

# 14. 常见报错原文与处理

| 报错/现象 | 真实原因 | 处理 |
|----|----|----|
| `This GitLab CI configuration is invalid: Component '<...>@~latest' - content not found` | 组件未发布到 Catalog 就用了 `~latest`/semver 简写；或 `templates/` 目录名不对；或组件名与文件路径不一致 | 打开 Catalog 开关 + `release:` job + semver tag；验证阶段先用 commit SHA 引用 |
| `Build component error: Spec must be a valid JSON schema` | `spec:` 为空段 | 写 `spec:`（哪怕只带 `description`） |
| `jobs:xxx has both only and rules` | 混用两套条件语法 | 只留 `rules` |
| `jobs:xxx: 'extends' cycle definition` | 模板互相继承成环 | 重排层次，公共部分下沉 |
| `Included file <path> not found` / 404 | include 路径不存在 / `ref` 分支已删 | 路径与 ref 一起检查；模板库用 tag 引用 |
| `jobs:xxx:needs ... undefined needs` | needs 指向被 `rules` 裁掉的 Job | 加 `optional: true`，或用 `rules:needs` |
| 下游流水线不创建、上游却 success | 下游仓库没有同一个 SHA | 见第 8.3 节三种解法 |
| `Parent Pipeline ID "0" does not refer to an existing pipeline` | 用了 `parent_pipeline` 但该 ref 上无上游流水线 | 加 `rules` 限定；要解耦就写 `parent_pipeline: null` |
| Job 日志被截断（`Log too long` / 尾部丢失） | `output_limit` 太小 | Runner 侧调大 `output_limit`（第 8 篇）或降低日志级别 |
| 矩阵里 `$DB` 变成 `mysql:8` 而非 `mysql:8.0` | YAML 把 `8.0` 解析成数字 | 矩阵值加引号 |
| 覆盖率不显示 / MR 无 diff coverage | `coverage` 正则缺捕获组；未上报 junit | 正则写 `...([0-9]{1,3})%`，同时上报 junit |
| 子流水线莫名变红，改的是父项目变量 | `inherit: variables` 默认 true | 下游显式 `inherit: { variables: false }` |
| 改了模板但行为没变 | 业务仓库有同名 Job 覆盖了模板 | 删本地同名 Job，或用不同名字 |

---

# 15. 小结

| 要点 | 一句话 |
|------|--------|
| 复用分层 | `extends` 减重复（数组整体替换）、`!reference` 实现追加、`include` 跨仓库（调用方覆盖被包含方、上限 150、`ref` 必须锁定）、Components 提供"版本 + 参数 + 可发现" |
| 组件规矩 | `templates/` 目录、≤100 组件/项目、`spec` 只允许 description/inputs/component、不写全局关键字与硬编码 stage、版本优先级 SHA>tag>branch、`$[[ ]]` 只在解析期生效 |
| 发布组件 | 只能靠流水线 `release:` 关键字 + semver tag，且要先打开 Catalog 项目开关 |
| 编排 | `needs` 控制何时运行（不管是否运行），`strategy: depend` 控制失败传播（**最深 5 级**），子流水线只认 branch，跨项目要求同一 SHA |
| 失败策略 | 基础设施类失败默认重试；用例失败重试要有退出计划；`allow_failure: exit_codes` 比 `allow_failure: true` 精确 |
| 报告 | junit / coverage_report / codequality / metrics / dotenv / sbom 各有约束，报告与二进制分开过期 |
| 身份 | `CI_JOB_TOKEN` 只用于依赖解析与跨项目 API 且有入站授权模型；部署凭据优先 Deployment variables 或 OIDC ID Tokens |

**自检清单**：

- [ ] 我能说清 `extends` 对数组和 map 的合并差异，并解释"父级 `before_script` 为什么消失了"？
- [ ] 我能写出组件的 `spec:inputs` + 引用方四行 include，并说出发布到 Catalog 的三步顺序？
- [ ] 我能说出 `strategy: depend` 的 5 层级联上限，以及"下游不跑但上游绿了"的根因与三种修法？
- [ ] 我知道 `inherit: variables` 默认 true 会带来什么静默行为变化？
- [ ] 我能在 10 分钟内用 API 拉出流水线耗时分布，并区分"排队慢"与"执行慢"？

> 下一篇：[第 8 篇 Runner 体系](8_runners.md) —— 三种注册方式（含 registration token 的弃用时间表）、执行器选型、config.toml 关键参数、令牌轮换与故障排查。
