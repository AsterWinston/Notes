# 1. GitLab CI 的定位与整体心智模型

> **前置**：[第 5 篇 MR 与 Review](5_merge_requests_and_review.md)
> **本篇目标**：吃透 `.gitlab-ci.yml` 的骨架 —— 流水线由哪些对象组成、Job 的关键字有哪些、变量与缓存/构件怎么配、什么时候该用 `rules`
> **对应实际工作**：搭一条能跑起来的 CI 流水线

## 1.1 对象层级：严格分层、逐层实例化

很多 CI 问题不是 YAML 写错，而是对象关系没搞清楚：

```text
Pipeline（一次流水线实例，绑定 pipeline-source + ref + SHA）
  └─ Stage（顺序执行的阶段，由 stages: 定义顺序）
       └─ Job（最小执行单元，由 Runner 执行）
            └─ Script（一组 shell 命令，同一个容器/Shell 会话）
       └─ Bridge（占位 Job，指向另一条流水线，本身不执行脚本）
            └─ Child Pipeline（子流水线，与父流水线绑定，可级联取消）
  └─ DAG（由 needs 形成的跨阶段依赖图，可提前调度）
```

## 1.2 一次 push 到 Job 结束的完整链路

```text
git push → GitLab 收到 ref 更新
   ↓
按 trigger policy 创建 Pipeline（source = push）
   ↓
解析并合并 YAML（include / extends / !reference / 组件展开）→ YAML lint
   ↓
对每个 Job 求值 rules → 决定创建 / 不创建（needs 依赖在此裁剪）
   ↓
按 Stage 顺序 + needs 拓扑 → Job 进入 Pending
   ↓
Runner 轮询认领（匹配 tags / 是否允许无标签 Job / 是否受保护）
   ↓
Runner 建执行环境（拉镜像、启服务容器）→ restore cache → clone/fetch 代码
   ↓
before_script → script → after_script，产物按 artifacts 上传
   ↓
Job 状态回报（success / failed / canceled）→ 影响后续 Job 与 MR 合并门禁
```

## 1.3 由此推出的三条排查纪律

1. **Job 压根没出现** → 十有八九是 `rules` 求值为 false，或被 `needs` 裁剪。查第 2、3 步，不是查 Runner。
2. **Job 一直 Pending** → YAML 已经合法，问题在 Runner 匹配。查 tags、`run_untagged`、`access_level`、项目 Runner 配额（第 8 篇）。
3. **失败发生在 `Running with gitlab-runner ...` 之前** → 属于 YAML / 流水线创建阶段问题，用第 11 节的 lint 工具。

---

# 2. 最小骨架与保留关键字

`.gitlab-ci.yml` 默认放在**仓库根目录**。可改路径：项目 **Settings → CI/CD → General pipelines → CI/CD configuration file**（例如 `ci/build.yml`）；这个入口同时也是"被别的仓库 include 时该用哪个文件"的设置处。

## 2.1 一份结构完整的骨架

```yaml
# 1) 定义阶段顺序（不写则用内置 .pre/stage/.post；显式写全更不容易踩坑）
stages:
  - lint
  - build
  - test
  - package
  - deploy

# 2) 全局默认：所有 Job 继承，Job 里可覆盖
default:
  image: maven:3.9-eclipse-temurin-17
  before_script:
    - java -version
    - mvn -v
  cache:
    key:
      files:
        - pom.xml
    paths:
      - .m2/repository
  interruptible: true

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"

# 3) 具体 Job：顶层除保留关键字外，其它 key 一律视为 Job
lint:
  stage: lint
  script:
    - mvn -B checkstyle:check

build:
  stage: build
  script:
    - mvn -B -DskipTests package
  artifacts:
    paths:
      - target/*.jar
    expire_in: 1 week

test:
  stage: test
  needs: ["build"]
  script:
    - mvn -B test
  artifacts:
    when: always
    reports:
      junit: target/surefire-reports/TEST-*.xml
```

## 2.2 保留关键字与 `default` 的边界

顶层保留关键字（写成 Job 会立刻 lint 失败）：`stages`、`variables`、`default`、`include`、`workflow`、`pages`、`types`、`image`、`services`、`cache`、`before_script`、`after_script`、`after`、`before`。

`default:` 支持的 key 也是有限的：**`before_script`、`after_script`、`image`、`tags`、`cache`、`interruptible`、`retry`**。

```yaml
default:
  image: alpine:3.20
  tags: [docker]
  retry: { max: 1, when: runner_system_failure }
  # script: [...]     ← 报 unknown keys
  # rules: [...]      ← 报 unknown keys
  # variables: {...}  ← 报 unknown keys（变量请在顶层 variables 或 Job 内声明）
```

## 2.3 YAML 层的四个高频坑

| 坑 | 症状 | 处理 |
|----|------|------|
| Tab 缩进 | `found character '\t' that cannot start any token` | 全部换空格 |
| `on`/`off`/`yes`/`no` 被解析成布尔 | `rules: - if: $X == on` 行为诡异 | 加引号 `"on"` |
| 变量没引号导致 `:` 被当映射 | `could not find expected ':'` | `- "mvn deploy -Durl=$NEXUS"` |
| 从文档/聊天复制带全角空格、中文引号 | lint 报难以定位的 column 错误 | `grep -nP '[^\x00-\x7F]' .gitlab-ci.yml` |

---

# 3. stages 与 Job：执行顺序由谁决定

## 3.1 stage 的缺省落点（"为什么它跑在最前面"）

- **`stages:` 定义阶段顺序**，同阶段内 Job 并行。
- **Job 不写 `stage:` 时**：若仓库存在 `test` 阶段，落到 `test`；否则落到 `stages:` 列表的**第一个**。这就是"我只加了一个 job，为什么它跑在 lint 前面"的标准答案。
- **内置阶段**：`.pre` 永远最先、`.post` 永远最后，两者都能被普通阶段"依赖"（`needs: [".pre 的 job"]`）。
- **规范约定：每个 Job 显式写 `stage:`。**

## 3.2 needs 与 DAG（此处只讲与"是否运行"的边界）

- **`needs:` 建立 DAG**，让 Job 不等整个前序阶段。`needs: []` 是合法写法，表示"无依赖，流水线创建后立即调度"。
- **`needs` 只影响"什么时候跑"，不影响"跑不跑"** —— 控制是否运行是 `rules` 的职责。
- 被 `rules` 判定不创建的 Job，其 `needs` 项自动被忽略。但如果**所有**显式 `needs` 都被裁剪掉，该 Job 会**回退到按 stage 等待**，表现为"加了 needs 却还是等全阶段"。要让依赖可选存在，显式写 `needs: [{ job: security-scan, optional: true }]`。
- `needs` 里引用矩阵生成的 Job 要写成 `build: [os: linux]` 形式，或用 `needs: [{ job: build, parallel: { matrix: [{ os: linux }] } }]`。

详细编排见第 7 篇。

---

# 4. script 与钩子：退出码决定一切

## 4.1 script 的执行语义

```yaml
build:
  script:
    - set -e                  # 用 | 写多行 shell 时务必显式设置
    - set -o pipefail         # 管道中保留真实退出码
    - mkdir -p out
    - ./compile.sh 2>&1 | tee out/compile.log
```

**任意一条 `script` 命令退出码非 0，Job 立刻失败，后续命令不执行。** 每条命令在同一个 shell 会话里顺序执行，所以 `export`、`cd` 在同一 Job 内有效（跨 Job 无效，每个 Job 环境独立）。

**坑**：`cd /tmp && rm -rf "$BUILD_DIR"` 之后 `$BUILD_DIR` 变成相对路径；再叠加 `set -e`，行为与直觉完全不同。**所有路径都用 `$CI_PROJECT_DIR` 开头的绝对路径。**

## 4.2 before_script / after_script 的真实差异

```yaml
  before_script:
    - echo "克隆之后、script 之前：装依赖、配凭据"
  after_script:
    - echo "script 之后无条件执行"
```

| | before_script | after_script |
|---|---|---|
| 执行时机 | 代码克隆后、script 前 | script 结束后（**成功或失败都跑**） |
| 是否共享 shell 会话 | 与 script 同一会话，变量可传递 | **独立进程**，看不到 script 里 `export` 的变量 |
| 失败后果 | 失败则 script 不执行，Job 失败 | **退出码不影响 Job 状态** |
| 能否用 artifacts 传递信息 | — | 能：写文件 → `artifacts` 上传 |
| 典型用途 | 配 `~/.m2/settings.xml`、`docker login`、代理 | 清理、上报、抓诊断文件 |

**坑**：`when: on_failure` 的 Job 不会跑 `after_script`。想在失败时收集环境信息，用 `artifacts: when: on_failure` 而不是 `after_script`。

---

# 5. image 与 services：Job 的运行环境

## 5.1 服务容器用主机名访问，不是 localhost

```yaml
integration-test:
  image: maven:3.9-eclipse-temurin-17
  services:
    - name: mysql:8.0
      alias: db                       # 容器内用主机名 db 访问
      variables:
        MYSQL_ROOT_PASSWORD: root
        MYSQL_DATABASE: app_test
    - name: redis:7-alpine
  variables:
    SPRING_DATASOURCE_URL: "jdbc:mysql://db:3306/app_test?useSSL=false&allowPublicKeyRetrieval=true"
  script:
    - mvn -B verify
```

- **Job 容器与服务容器在同一网络**，服务用**镜像名（或 `alias`）作为主机名**访问。用 `localhost`/`127.0.0.1` 连数据库是初学者第一坑。
- **需要固定名字或多个服务互相发现**时用 `alias`；同名 alias 会冲突。
- **Docker executor 里 `--link` 已废弃**，GitLab 用自建 bridge 网络；`services:name` 带私有仓库全名时注意拉取凭据（第 8 篇）。

## 5.2 就绪等待与 entrypoint 覆盖

**服务启动是异步的**，必须自己写就绪等待，否则第一批用例随机失败：

```yaml
  before_script:
    - |
      for i in $(seq 1 30); do
        nc -z db 3306 && { echo "db ready after ${i}s"; break; }
        sleep 1
      done
      nc -z db 3306 || { echo "db not ready"; exit 1; }
```

有些镜像的 `entrypoint` 会吃掉 GitLab 注入的脚本（典型：`docker`、`nginx`、部分发行版镜像），标准解法是覆盖入口点：

```yaml
  image:
    name: docker:27
    entrypoint: [""]
  services:
    - name: docker:27-dind
      alias: docker
```

**坑**：`CI_REGISTRY_IMAGE` 指向项目镜像仓库（路径全小写）。`docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA` 里若 ref/tag 片段包含 `/`、`:`、大写会失败，构造 tag 前先归一化（第 15 篇有统一写法）。

---

# 6. variables：来源、优先级与三种特殊类型

## 6.1 优先级（高 → 低）

```text
触发流水线时传入的变量（Run pipeline / trigger API / 子流水线变量）
  >  Job 级 variables（同 Job 内按定义顺序）
  >  Project / Group / Instance 级 CI/CD 变量
  >  预定义变量（CI_*、GITLAB_*）
  >  Deployment variables（仅 environment scope 命中时）
  >  默认值（variable default value，GitLab 18.0+）
```

> 层级顺序在不同版本上确实调整过，且"多来源同名变量最终取哪个"随版本有差异。**动手前先读回现值**，不要凭记忆调试：
> ```bash
> glab api "projects/:id/variables" | jq -r '.[] | "\(.key)\tmasked=\(.masked)\tprotected=\(.protected)\ttype=\(.variable_type)"'
> ```

## 6.2 五种类型

| 类型 | 说明 | 典型用途 |
|------|------|----------|
| **Variable** | 纯文本 | 仓库地址、版本号 |
| **File** | 值写成临时文件，变量存的是**文件路径** | `.kube/config`、`~/.npmrc`、`settings.xml` |
| **Secret variable** | 掩码 + 受保护 | Token、密码 |
| **Masked** | job log 里替换为 `[MASKED]` | 不应出现在日志的任意串 |
| **Protected** | 只在受保护 ref 上可用（第 12 篇） | 生产密钥 |

**File 类型最容易写错**：

```yaml
# Settings → CI/CD → Variables：键 KUBE_CONFIG，Type = File，值 = kubeconfig 内容
deploy:
  script:
    - ls -l "$KUBE_CONFIG"        # 它是一个临时文件
    - kubectl --kubeconfig "$KUBE_CONFIG" -n app rollout status deploy/app
    # 错误写法：echo "$KUBE_CONFIG" | kubectl ...   → 传的是路径，不是内容
```

**Masked 的硬性约束**：值必须**单行**、**长度 ≥ 8**、**只含 Base64 安全字符集**（大小写字母、数字、`+ / = _ . -`）。不满足时 UI 直接拒绝并提示原因。JWT、带空格的多行私钥、含 `:` 的连接串都会踩这个约束 —— 改用 **Type: File**，或放宽掩码改在脚本里 `sed` 脱敏。

## 6.3 展开时机：`$VAR` 与 `$$VAR`

**变量展开发生在 shell 之前**（Runner 自己做一次插值）：

```yaml
  script:
    - echo "$TOKEN" > /dev/null     # Runner 先替换成实际值，日志里显示 [MASKED]
    - echo "$$TOKEN" > /dev/null    # 交给 shell 展开，Runner 不参与插值
```

- 想让 Runner **不要**插值（命令名、复杂脚本片段、避免密钥出现在命令行），用 `$$`。
- 想彻底禁止二次插值：`variables: { RAW: { value: "$NOT_EXPANDED", expand: false } }`。
- 变量名建议只用 `^[A-Za-z_][A-Za-z0-9_]*$`；`1VAR`、含 `-` 的名称会被拒绝（`invalid variable name`）。

**坑（真实事故）**：`masked` + `protected` 的变量在非保护 ref 上**根本不存在**，脚本里表现为命令拼出 `https://:@harbor.example.com` 或 `command not found`。定位法：

```yaml
  before_script:
    - test -n "$HARBOR_USER" || { echo "HARBOR_USER 未注入（当前 ref 是否受保护？）"; exit 1; }
```

---

# 7. rules 与 workflow：一个决定 Job，一个决定整条流水线

## 7.1 workflow：流水线级总闸

```yaml
workflow:
  rules:
    # MR 流水线优先：避免"push 分支 + 开 MR"时同一条 commit 跑两条重复流水线
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS
      when: never
    - if: $CI_COMMIT_BRANCH
    - if: $CI_COMMIT_TAG
```

**坑**：只写 `- if: $CI_PIPELINE_SOURCE == "push"` 会让**定时器、手动运行、下游触发全部跑不起来**（它们的 source 不是 `push`）。这是"流水线不创建"最常见的第二个原因。

**流水线级并发控制**（`auto_cancel`，无需自写 API 脚本）：

```yaml
workflow:
  rules:
    - if: $CI_COMMIT_BRANCH
      auto_cancel:
        duplicate_hide: true    # 隐藏同分支同 commit 的重复流水线
        mid_job_cancel: true    # 新流水线创建时取消进行中的旧 Job
```

## 7.2 rules：Job 级条件

```yaml
build:
  stage: build
  rules:
    - if: '$CI_COMMIT_TAG'
      variables: { RELEASE: "true" }
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      changes:
        paths:
          - "src/**/*"
          - "pom.xml"
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
    - when: manual
      allow_failure: true
```

**求值规则（背下来能省一小时）**：

- **从上到下匹配，命中第一条即生效**，后面的全部忽略 —— 所以"排除型" `- when: never` 必须放前面。
- `if` 是**字符串比较**：`$VAR == "x"` 右侧必须加引号；`$CI_JOB_ID > 100` 按字符串比，别用。
- 完全没写 `rules`/`only`/`except` 的 Job 等价于 `rules: [{ when: on_success }]`，即每次流水线都跑。
- 可用关键字族：`if`、`changes`、`exists`、`allow_failure`、`variables`、`needs`、`auto_cancel`。
- `changes` 只在**分支流水线（对比上一 commit）与 MR 流水线（对比目标分支）**中有意义；在 tag 流水线、定时/手动触发的流水线里结果可能为空或全部命中。需要"tag 时也按路径判定"时，用 `rules:changes:compare_to` 显式指定基准 ref（可用性与实例订阅级别有关，**以 lint 结果为准**）。
- GitLab 17.x 起把 `rules:changes:branches` / `:tags` 这类按 ref 类型限定的写法往 `rules:source`（`- source: merge_request_event`）迁移，旧写法会收到 deprecation 提示 —— **具体版本边界以你实例的 lint 警告为准**。

## 7.3 only / except：只做迁移，不再新增

GitLab 已停止为 `only`/`except` 增加功能，且**同一 Job 混用 `only` 与 `rules` 会直接 lint 失败**（`jobs:xxx has both only and rules`）。`include` 侧同理：`include:only/except` 已由 `include:rule`（18.1+）替代。迁移时**整段换，别局部混用**。

## 7.4 触发来源速查

| `CI_PIPELINE_SOURCE` | 场景 | 常见配套写法 |
|---|---|---|
| `push` | 分支/标签 push | 主干构建 |
| `merge_request_event` | MR 流水线 | 评审前反馈（第 5 篇） |
| `web` | UI 手动 Run pipeline | 一次性发布/回填 |
| `trigger` | Pipeline trigger token | 外部系统调用（第 11 篇） |
| `schedule` | 定时器 | 夜间全量、依赖扫描 |
| `api` | REST API 触发 | 平台脚本 |
| `pipeline` | 多项目下游触发 | 第 7 篇父子流水线 |
| `external` / `chat` / `webide` | 外部状态检查 / ChatOps / Web IDE | 一般不参与门禁 |

---

# 8. artifacts 与 cache：一条分界线

## 8.1 语义对照

| | **artifacts** | **cache** |
|---|---|---|
| 语义 | **构建产出**，属于流水线的一部分 | **加速用的依赖目录**，尽力而为 |
| 去向 | 对象存储（CI artifacts bucket），随流水线过期策略回收 | Runner 侧缓存后端（本地目录 / S3 / GCS） |
| 跨 Job 传递 | 下游自动下载（受 `dependencies`/`needs` 控制） | 只在同 key 的 Job 之间，**顺序无保证** |
| 能否当正确性依赖 | ✅ 能 | ❌ 不能（缓存被驱逐后行为会变） |
| 典型内容 | `target/*.jar`、测试报告、`build.env` | `node_modules`、`.m2/repository`、`.gradle/caches` |
| 大小纪律 | 单 artifact 有实例上限，大的推 registry/对象存储 | 压缩后约 1GB 上限，超限静默不上传 |
| 失败时是否保留 | `when: on_failure` / `always` | `when: always` |

## 8.2 关键参数与推荐写法

```yaml
test:
  script:
    - mvn -B test
  artifacts:
    when: always                     # 失败也要留下报告
    name: "$CI_JOB_NAME-$CI_COMMIT_REF_SLUG"
    expire_in: 7 days                # 默认 30 天，改小最省存储
    untracked: false                 # 谨慎：true 会按 git status 收集未跟踪文件
    paths: ["target/surefire-reports/"]
    exclude: ["**/*.bin"]
    expose_as: "测试报告"             # 在 Job 页面给出显眼入口
    public: false                    # 含敏感内容时必须 false
    reports:
      junit: target/surefire-reports/TEST-*.xml

package:
  needs:
    - job: test
      artifacts: true                # 只要 test 的 artifacts
  cache:
    key:
      files: ["pom.xml"]             # 依赖清单变了才换 key
    paths: [".m2/repository"]
    policy: pull-push                # 只读 Job 用 pull，避免互相覆盖
    fallback_keys:
      - maven-$CI_DEFAULT_BRANCH     # 新分支命中不到时退回主干缓存
  script:
    - mvn -B -o package              # -o 离线：缓存没命中会直接失败，主动暴露问题
```

- **`dependencies` 只影响"下载哪些上游 artifacts"**；一旦 Job 写了 `needs`，`dependencies` **被忽略**。统一用 `needs: [{job, artifacts: true|false}]` 表达。
- **`artifacts:reports:` 与 `artifacts:paths:` 分开发布**（把报告聚合到单独 Job）能分别设过期策略，详见第 7 篇。

## 8.3 真实报错对照

| 报错/现象 | 原因 | 处理 |
|----|----|----|
| `WARNING: target/*.jar: no matching files` | `paths` 没匹配到文件 | Job **仍然成功**（只告警），下游拿不到包 → 加 `test -f target/*.jar` 显式失败 |
| 失败 Job 页面没有产物 | 没写 `when: on_failure/always` | 显式指定 `when` |
| `upload artifacts ... 413 Request Entity Too Large` | 超过单 artifact 上限 | 压缩、拆分，或改推 registry |
| `restoring cache ... no url found for cache key` | 首次构建或 key 变了 | 正常现象，不是错误 |
| 缓存恢复 `403 Forbidden`（S3/GCS 后端） | 缓存桶凭据/区域不对 | 检查 Runner `config.toml` 的 `[runners.cache.s3]`（第 8 篇） |
| **下游 Job 拿不到上游 jar** | 用了 `needs` 但没 `artifacts: true`；或 `dependencies` 与 `needs` 混用 | 统一用 `needs: [{job, artifacts: true}]` |
| `expire_in` 改了没生效 | 上限受 Admin 配置约束 | 由实例管理员调整 |
| 日志明文出现 token | `public: true` 的 artifacts 任何人可下载 | 涉密产物必须 `public: false` |

---

# 9. 手动任务、发布与 Auto DevOps

## 9.1 when: manual 与 delayed

```yaml
# 发布类 Job：不阻塞流水线变绿，等人点
publish:
  stage: package
  rules:
    - if: '$CI_COMMIT_TAG'
      when: manual
      allow_failure: false     # 点之前流水线算成功；设为 true 则不点也成功
  script: [ ./ci/publish.sh ]

# 延迟部署：给主干保留 1 小时"刹车窗口"
canary:
  stage: deploy
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      when: delayed
      start_in: 1 hour
  script: [ ./ci/deploy.sh canary ]
```

**`when: manual` 的三个约束**：① 下游依赖它时，**不点则整个下游链不跑**；② 手动 Job **超时后会被取消**，别指望"过夜等人点"能保留上下文；③ 生产发布别只靠 `when: manual`，还要叠加保护 ref + 环境审批（第 12 篇）。

## 9.2 release 关键字：一条 Job 同时产出 Release 与产物

```yaml
release-app:
  stage: package
  release:
    tag_name: v$APP_VERSION
    description: |
      ## 变更
      - 见 $CI_COMMIT_REF_NAME
    ref: $CI_COMMIT_SHA
    milestones: [ "2.4" ]
    released_at: "2026-11-19T10:00:00Z"
    assets:
      links:
        - name: "安装包"
          url: "$CI_SERVER_URL/$CI_PROJECT_NAMESPACE/$CI_PROJECT_NAME/-/jobs/$CI_JOB_ID/artifacts/app.zip"
  script: [ ./ci/build-app.sh ]
```

> `release:` 也是**发布 CI/CD 组件的唯一途径**（Releases API 与 Web 创建的 Release 都不能发布组件，见第 7 篇）。

## 9.3 Auto DevOps：一条注释都不写的流水线

```yaml
# .gitlab-ci.yml 里这一行即可启用 Auto DevOps
include:
  - template: Auto-DevOps.gitlab-ci.yml
```

- Auto DevOps 会按语言/框架探测，自动生成 build/test/code quality/security review/DAST/deploy 等阶段，并依赖 **Auto Build** 从 Dockerfile/Procfile 推断镜像构建。
- **Java 后端项目不建议直接用**：它对 Maven 多模块、私服 `settings.xml`、内部基础镜像的处理不符合企业规范，跑起来后你仍要花更多时间关掉一半 Job。
- **合理的用法**：① 给新仓库当"占位 CI"，跑通后替换成团队模板；② 只想借用其中某个能力时，**单独 include 那个模板**（例如 `- template: Security/SAST.gitlab-ci.yml`），比整体启用 Auto DevOps 干净得多。
- 关闭/微调 Auto DevOps 的开关在项目 **Settings → CI/CD → Auto DevOps**，注意它是项目级配置，**模板项目（project template）不会自动带上这个开关**，靠 `include` 固化更可靠。

---

# 10. 与部署相关的关键字（此处只划边界）

- **`environment:`** 把 Job 关联到部署环境，`action: start/prepare/stop/verify/access` 决定它对环境的作用，`auto_stop_in` 控制过期环境回收。**注意 `auto_stop_in` 是时长字符串（`"48 hours"`），停止靠 `action: stop` 或 `on_stop`** —— 不存在 `auto_stop_if`，写它会得到 `unknown keys`。完整用法见第 9 篇。
- **`resource_group:`** 让同一资源的 Job 串行，是"同分支并发部署互相踩"的唯一正确解（第 9 篇）。
- **`timeout:`** Job 级超时（`- 30m` / `- 1h30m`），上限受实例与 Runner `--maximum-timeout` 约束。
- **`retry:`**、**`interruptible:`**、**`allow_failure:`** 三个"看起来像部署语义其实是调度语义"的关键字，详见第 7 篇。

---

# 11. 调试流水线的实用手段

## 11.1 从便宜到昂贵

```bash
# 1) 本地结构自检：成本最低、收益最高
test -f .gitlab-ci.yml && echo "file OK"
python3 -c 'import yaml; yaml.safe_load(open(".gitlab-ci.yml")); print("YAML OK")'
git ls-files --error-unmatch .gitlab-ci.yml      # 是否真的提交进仓库（注意大小写）

# 2) Web IDE 旁的 CI/CD 编辑器 / CI Lint 页面：能立刻指出行列号
#    API 版（把待检查内容放 ci.yml）：
curl -sS --form content=@ci.yml \
  -H "PRIVATE-TOKEN: $GITLAB_PAT" "$GITLAB_URL/api/v4/projects/$CI_PROJECT_ID/ci/lint" | jq .

# 3) 拿"展开后"的配置：验证 include / extends / 组件是否真的生效
#    （component 写法错误时，页面常见报错为 "Component '...' - content not found"）
curl -sS -H "PRIVATE-TOKEN: $GITLAB_PAT" \
  "$GITLAB_URL/api/v4/projects/$CI_PROJECT_ID/pipelines/$CI_PIPELINE_ID" | jq .

# 4) 单 Job 深挖：状态、失败原因、trace
curl -sS -H "PRIVATE-TOKEN: $GITLAB_PAT" \
  "$GITLAB_URL/api/v4/projects/$CI_PROJECT_ID/jobs/$JOB_ID" \
  | jq '{name,status,failure_reason,allow_failure,tag_list,duration}'
curl -sSL -H "PRIVATE-TOKEN: $GITLAB_PAT" \
  "$GITLAB_URL/api/v4/projects/$CI_PROJECT_ID/jobs/$JOB_ID/trace" | tail -80
```

在 YAML 里临时打开的诊断开关：

```yaml
variables:
  BUILDKIT_PROGRESS: plain       # 让镜像构建日志可读
  MAVEN_ARGS: "-B -ntp -X"       # 排查依赖解析时临时加 -X
  CI_DEBUG_TRACE: "true"         # Runner 输出更详细的执行细节（较新版本 Runner）
env-dump:
  stage: .pre
  script:
    - env | grep -E '^(CI_|GITLAB_)' | sort   # 预定义变量清单，比翻文档快
    - cat /etc/os-release; uname -a; df -h; id
```

**MR 页面上最快的一条路径**：流水线详情 → 具体 Job → **"Why did this job not run?"** 提示（Job 未创建时），或 Job 页的 **Actions → View deployment / View config** 直接看展开后的配置。

## 11.2 重试还是改 yaml：决策顺序

```text
Job 失败
  ├─ 脚本自身失败？      → 本地 docker run 同镜像复现，改代码，push
  ├─ Runner 问题？        → trace 含 "no runner matching" / "(system failure)" → 转第 8 篇
  ├─ 基础设施抖动？        → failure_reason ∈ {runner_system_failure, worker_timeout,
  │                          stuck_or_timeout_failure} → retry 自愈或手动 Retry（同 commit、同变量）
  └─ YAML 写错？           → CI Lint 改配置
```

> **Retry 不会重新解析 `.gitlab-ci.yml`**，它复用该流水线**已展开**的 Job 定义。改 YAML 必须触发新流水线；"改了 yaml 重跑还是老行为"就是误用了 Retry。

---

# 12. 一条可直接用的 Java 后端流水线（收敛本篇）

```yaml
stages: [.pre, build, test, package, deploy]

default:
  image: maven:3.9-eclipse-temurin-17
  interruptible: true
  cache:
    key:
      files: [pom.xml]
    paths: [.m2/repository]

variables:
  MAVEN_ARGS: "-B -ntp -Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"

workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS
      when: never
    - if: $CI_COMMIT_BRANCH
    - if: $CI_COMMIT_TAG

preflight:
  stage: .pre
  script:
    - python3 -c 'import yaml;yaml.safe_load(open(".gitlab-ci.yml"))'
    - test -f pom.xml || { echo "pom.xml 不存在"; exit 1; }

compile:
  stage: build
  script: [ mvn $MAVEN_ARGS -DskipTests package ]
  artifacts: { paths: [target/classes/], expire_in: 1 day }

unit-test:
  stage: test
  needs: [{ job: compile, artifacts: false }]
  script: [ mvn $MAVEN_ARGS test ]
  artifacts:
    when: always
    expire_in: 7 days
    reports: { junit: target/surefire-reports/TEST-*.xml }

coverage:
  stage: test
  needs: [{ job: unit-test, artifacts: false }]
  script:
    - mvn $MAVEN_ARGS jacoco:report
    - awk -F',' 'NR>1{m+=$4;c+=$5}END{printf "line coverage %.1f%%\n", 100*c/(c+m)}' target/site/jacoco/jacoco.csv
  coverage: '/line coverage (\d+\.\d+)%/'
  allow_failure: true            # 只观测；要卡门禁改用第 15 篇的阈值检查

deploy-staging:
  stage: deploy
  needs: [compile]
  resource_group: staging
  environment:
    name: staging
    url: https://staging.example.com
    deployment_tier: staging
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  script: [ ./ci/deploy.sh staging ]

# 生产：手动 + 只允许保护 ref；回滚方案见第 9 篇
deploy-prod:
  stage: deploy
  needs: [compile]
  resource_group: production
  environment: { name: production, url: https://example.com }
  rules:
    - if: $CI_COMMIT_TAG
      when: manual
  script: [ ./ci/deploy.sh production ]
```

---

# 13. 小结

| 要点 | 一句话 |
|------|--------|
| 模型 | Pipeline → Stage → Job/Bridge → Script；Job 没出现查 `rules`，Job 不动查 Runner，YAML 报错查 lint |
| 显式优于隐式 | 每个 Job 写 `stage:`；有 `needs` 时 `dependencies` 失效，统一用 `needs: [{job, artifacts}]` |
| 退出码 | `script` 由退出码决定成败；`after_script` 是新进程、看不到 `script` 的变量、退出码不影响 Job |
| 服务容器 | 用主机名（或 `alias`）访问，不是 `localhost`；必须自己写就绪等待 |
| variables | 多层优先级 + 两种展开时机：`$VAR` 由 Runner 先插值、`$$VAR` 交给 shell；`File` 类型变量存的是**路径**；`masked` 有字符集与长度约束，`protected` 在非保护 ref 上根本不存在 |
| 条件 | `workflow` 管建不建流水线，`rules` 管建不建 Job；`only/except` 与 `rules` 互斥 |
| 产物 | artifacts 讲正确性，cache 只讲速度；两者的过期与大小直接影响存储成本 |
| 排错 | 本地 YAML 自检 → CI Lint → 展开后配置 → Job trace；改 YAML 必须重新触发流水线，Retry 不复用新 yaml |

**自检清单**：

- [ ] 我能说出为什么同一次 push 会跑两条流水线，以及 `workflow` 怎么修掉它？
- [ ] 我能区分 `$$VAR` 与 `$VAR`，并解释生产密钥为什么在 MR 流水线里"消失"？
- [ ] 我能列出 artifacts 与 cache 各自的失败模式（`no matching files` 只是告警，`needs` 忘写 `artifacts: true` 才是真断链）？
- [ ] 我知道什么场景该 include 单个安全模板，而不是整体启用 Auto DevOps？
- [ ] 我改完 `.gitlab-ci.yml` 第一步跑的是本地 YAML 自检 + lint，而不是直接 push？

> 下一篇：[第 7 篇 GitLab CI 进阶](7_gitlab_ci_advanced.md) —— 模板复用体系（`extends` / `include` / CI/CD Components）、DAG 与多项目编排、矩阵并行、流水线报告。
