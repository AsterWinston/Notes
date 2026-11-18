# 1. 内置包仓库：交付物的"最后一公里"

> **前置**：[第 9 篇 Environments 与部署](9_environments_and_deployment.md)
> **本篇目标**：把镜像与制品"存到哪、谁能推拉、怎么防膨胀、怎么防被覆盖"讲清楚：Container Registry 命名与权限、清理策略、tag/包保护、Maven/npm/Generic 端点与令牌
> **对应实际工作**：CI 产物落地、内部 SDK 复用、镜像仓库不撑爆磁盘

## 1.1 GitLab 制品体系全景

GitLab 除 Container Registry 外，还内置一整套 **Package Registry**：Maven、npm、PyPI、NuGet、Go Proxy、Conan 1/2、Debian、RPM、Cargo、Composer、RubyGems、Helm、Terraform module、Generic 等，与项目/群组原生打通 —— **CI 产出的 jar 和镜像可以直接 push 回同一个平台**，权限与溯源天然继承代码库（或作为 Nexus/Harbor 的前置/补充）。

| 仓库类型 | 端点规则 | 典型用途 |
|----------|---------|---------|
| Container Registry | `<registry>/<group>/<project>[:tag]` | 应用镜像、基镜像、Helm chart（OCI） |
| Maven（项目级） | `https://<host>/api/v4/projects/<ID>/packages/maven` | 内部 SDK/BOM 发布与消费 |
| Maven（群组级） | `https://<host>/api/v4/groups/<GROUP_ID>/-/packages/maven` | 整组共享依赖解析 |
| Maven（实例级） | `https://<host>/api/v4/packages/maven` | 包坐标必须与项目路径一致（见 7.3） |
| npm | `.../api/v4/projects/<ID>/packages/npm/` | 前端私有组件 |
| Generic | `.../packages/generic/<name>/<version>/<file>` | 任意文件（SQL 包、配置包、离线补丁） |
| Dependency proxy | `https://<host>/dependencies/docker`（容器）等 | 代理外网上游，减少重复外网拉取 |

## 1.2 三类端点与命名规则的选择

同一个制品可以有 **项目 / 群组 / 实例** 三种访问入口，差别不在"能不能访问"，而在**命名约束与重名风险**：

- **项目级**：最常用，URL 里带 `project_id`，包名随意，权限就是项目权限。
- **群组级**：一个 URL 覆盖组内所有项目的包，适合"全公司统一一个 Maven 地址"。**但 GitLab 不保证组内包名唯一**（见 7.3 的坑）。
- **实例级**：URL 最短，但**只有包路径与项目路径一致的包才会暴露**：`groupId = group-slug[.subgroup-slug]`、`artifactId = project-slug`。不满足这个约定的项目，用实例级端点会 404。

## 1.3 什么时候用 GitLab，什么时候留给 Nexus/Harbor

| 需求 | 建议 |
|------|------|
| 代码、CI、制品、权限、溯源一体化，团队规模中小 | 全用 GitLab，运维成本最低 |
| 需要**强力的外网代理 + 异构仓库聚合 + 精细的仓库治理** | 外网依赖代理仍可留给 Nexus；GitLab 侧承担"发布产物仓 + 内源消费" |
| 需要企业级镜像扫描/签名策略/多集群复制 | Harbor（GitLab 提供 Harbor 兼容层与 Harbor 虚拟仓库集成） |
| 只是想要一个"能存的货架"（安装包、脚本包、模型文件） | **Generic packages**，别为它单独搭服务 |

> **纪律**：无论用哪套，**"谁有权覆盖生产 tag"必须显式配置**（第 5 节），这是制品侧唯一的不可逆风险点。

---

# 2. Container Registry 使用

## 2.1 镜像命名与"仓库路径绑定项目路径"

```text
<registry-host>/<group>/<project>[/<子路径，最多两层>][:tag|@sha256:...]
例：registry.example.com/pay/checkout-web:1.4.2
    registry.example.com/pay/checkout-web/tmp/nginx-sidecar:1.0
```

- 入口：项目左侧 **Deploy → Container Registry**。
- **容器仓库的路径与项目路径绑定**：不能单独给某个容器仓库改名或迁移。**改项目路径会连带改镜像路径**（较新版本支持随项目改名迁移容器仓库，但**项目内容器仓库数量上限约 1000 时不允许改名**）。
- 推论：**镜像地址写进配置/部署清单前要把它当"API 契约"**。要移动项目，先规划 tag 双推过渡期，别直接改路径。

## 2.2 可见性与禁用行为

Settings → Packages and registries → **Container registry visibility**：

| 选项 | 含义 |
|------|------|
| **Everyone With Access** | 所有对该项目有权限的用户可见/可用 |
| **Only Project Members** | 仅项目成员 |
| 关闭 Container registry | **全部操作不可用**（推、拉、UI 列表都停） |

## 2.3 OCI 支持与"额外内容"

- GitLab 容器仓库**同时支持 Docker v2 manifest 与 OCI manifest**，因此可以把 **Helm 3+ chart**、SBOM、签名等"OCI artifact"放进同一个仓库。
- **多架构徽章（Supported platforms）与签名展示都依赖 registry metadata database**：GitLab.com 默认启用；**自建实例需要单独启用**（`registry_metadata_database`），没启用时这些 UI 元素不会出现 —— 不是 bug。

---

# 3. CI 里推拉镜像：正确姿势

## 3.1 三个内置变量与 CI_JOB_TOKEN

```yaml
build-image:
  stage: build
  image: docker:27
  services:
    - docker:27-dind          # 生产更推荐 kaniko/buildah，见第 8 篇 Runner 安全
  before_script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
  script:
    - docker build -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA" .
    - docker push "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA"
```

| 变量 | 值 | 说明 |
|------|----|------|
| `CI_REGISTRY` | `registry.example.com` | 仅主机名，**不要**自己拼 `https://` |
| `CI_REGISTRY_IMAGE` | `registry.example.com/group/project`（**全小写**） | 项目镜像前缀 |
| `CI_REGISTRY_USER` | `gitlab-ci-token` | 临时凭据用户名 |
| `CI_REGISTRY_PASSWORD` | = `CI_JOB_TOKEN` | **Job 结束即失效**，只对本项目仓库有效 |

**要点**：`--password-stdin` 而不是 `-p $TOKEN`，避免令牌出现在进程列表/日志里。

## 3.2 不可变 tag 与多架构

```yaml
# 一次构建产出 amd64 + arm64，同一 tag 指向 manifest list
multiarch:
  stage: build
  image: docker:27
  services: [{ name: docker:27-dind }]
  before_script:
    - docker buildx create --use --driver docker-container --name bx
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
  script:
    - docker buildx inspect --bootstrap
    - >
      docker buildx build --platform linux/amd64,linux/arm64
      --tag "$CI_REGISTRY_IMAGE:${CI_COMMIT_TAG:-$CI_COMMIT_SHORT_SHA}"
      --push .
```

**tag 设计纪律**（决定你后面能不能干净回滚）：

| 用途 | 写法 | 是否可覆盖 |
|------|------|-----------|
| 唯一不可变标识 | `$CI_COMMIT_SHA`（或 digest） | **永不覆盖** |
| 版本 tag | `v1.2.3`（来自 tag 流水线） | 配 immutable 规则禁止覆盖 |
| 滚动 tag | `latest` / `main` / `stable` | 可覆盖，但**不要**用它做回滚依据 |
| 环境 tag | `prod-<sha>` | 加入清理豁免（第 4 节） |

> **不要指望 `latest` 能回滚**：清理策略与并发构建都会让它指向非预期镜像。回滚用 digest 或不可变版本 tag（第 9 篇 4.4）。

## 3.3 跨项目/群组推送与 Deploy Token

```bash
# 群组基镜像：推一次，多个项目复用
docker login -u "$GROUP_PUSH_USER" -p "$GROUP_PUSH_TOKEN" registry.example.com
docker tag base:1.0 registry.example.com/platform/base-images/jdk:17.0.11
docker push registry.example.com/platform/base-images/jdk:17.0.11
```

- **CI Job Token 默认只能推本项目仓库**；跨项目推拉需要在被访问项目 **Settings → CI/CD → Job token permissions** 显式授权（见 10.3）。
- 人手动推拉镜像用 **Deploy Token**（`read_registry`/`write_registry`）或带 `read_registry`/`write_registry` scope 的 PAT；**"人的账号尽量不直接 push"**（见 10.4）。
- 群组级 Registry（`<registry>/<group>/<name>`）适合基镜像/跨项目共享镜像。

## 3.4 镜像签名与构建溯源

```yaml
sign-image:
  stage: package
  image: alpine:3.20
  variables:
    COSIGN_EXPERIMENTAL: "1"          # 需要 OCI 1.1 referrers 行为
  before_script:
    - wget -qO /usr/local/bin/cosign https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64 && chmod +x /usr/local/bin/cosign
  script:
    - cosign sign --yes --registry-referrers-mode oci-1-1
        "$CI_REGISTRY_IMAGE@$IMAGE_DIGEST"
```

- 签名用 **Cosign v2.0+**，并**必须加 `--registry-referrers-mode oci-1-1`**：GitLab 未完整实现 OCI 1.1 Referrers API 时，这个模式让签名以 referrer 形式落到同一仓库并可被 UI 识别。
- 签名展示依赖 metadata database；自建未启用时**签名能生成但 UI 不显示**（`cosign verify` 仍可用，这是验证入口）。
- 推荐做法：**记录 digest 而非 tag**（`IMAGE_DIGEST` 从 `docker buildx build --metadata-file` 或 `docker inspect` 取），签名与部署都锁 digest。

---

# 4. 镜像清理策略（保留策略）

Registry 会无限膨胀 —— 每次 CI 都推新 tag，几百 GB 很快耗尽磁盘。GitLab 提供 **Cleanup Policies**，但它有几个必须知道的陷阱。

## 4.1 两个策略字段

| 字段（API 名） | UI 位置 | 作用 |
|----------------|---------|------|
| `default_container_expiration_policy` | 项目 **Settings → Packages and registries → Cleanup policies for image tags**；群组同名页 | 按 `older_than` + `keep_n` + `name_regex` + `name_regex_keep` 删 tag |
| `default_cleanup_immediate_tags_policy` | 群组同名页（History 较新） | **策略刚保存后立即执行一次清理**，否则要等到下个周期 |

参数取值范围：

| 参数 | 可选值 | 说明 |
|------|--------|------|
| `older_than` | `default`/`1d`/`7d`/`14d`/`30d` | 早于此时间且**不在保留名额内**的 tag 被删 |
| `keep_n` | `1,2,3,4,5,10,15,25,50,100` | 每个 tag 模式保留最近 N 个 |
| `name_regex` | 默认 `.*` | **哪些 tag 参与清理**（RE2，勿用反向引用） |
| `name_regex_keep` | 例 `(latest\|stable\|release\|prod.*)` | **永不删除**（必须 RE2 兼容，否则报 `error parsing regexp`） |

## 4.2 三个必踩的坑

1. **实例级默认策略看不到也不报错**：如果实例配了 `default_container_expiration_policy`，**新项目会自动继承，即使项目页策略列表里看不到它**。用 UI 删除项目策略 = 把项目退回继承实例策略（可能更激进）而不是"关闭清理"。
2. **群组策略会盖过实例默认**：想在群组内用更宽松的策略，必须在群组上显式设置并 **Apply to all projects**。
3. **清理在每小时第 15 分钟跑一次**（GitLab.com 不保证精确时间）；**项目删除后其清理任务被取消**。想"配了立刻生效"要用 `default_cleanup_immediate_tags_policy`。

```bash
# 用 API 读回"实际生效"的策略，而不是只看 UI 列表（字段名以现值为准）
glab api "projects/:id" | jq .container_expiration_policy
glab api "groups/:id"   | jq .default_container_expiration_policy
```

## 4.3 清理到底删了什么

- 策略**删的是 tag（引用）**，不是"磁盘上的 blob"。
- 未被任何 tag 引用的 manifest/layer 需要 **registry garbage collection** 才真正释放空间。Omnibus 下需要启用 GC worker（`registry_gc['enable'] = true` + `gitlab-ctl reconfigure`）并在低峰执行；**GC 具体步骤必须按你版本的官方文档**（它会移动/删除文件，做错会导致 404）。
- metadata database 里的记录由后台 worker 同步（较新版本用 `migrate_container_repository_metadata_worker` 批量迁移），大批量迁移时要限流。

**所以正确的空间预期**：

```
配置清理策略 ──► tag 数量下降（分钟级可见）
              └─► blob 释放：需 GC（可能延迟数天，需管理员安排窗口）
```

## 4.4 推荐配置（Java 团队）

```yaml
# 群组级"应用项目"默认策略（通过 API 下发到群组）
{
  "default_cleanup_immediate_tags_policy": true,
  "default_container_expiration_policy": {
    "cadence": "1h",
    "enabled": true,
    "keep_n": 10,
    "name_regex": ".*",
    "name_regex_keep": "^(latest|stable|prod-.*|v\\d+\\.\\d+\\.\\d+)$",
    "older_than": "14d"
  }
}
```

> **必须配 `name_regex_keep`**：否则**回滚用的镜像会被清理掉**。更稳的做法是给这些 tag 上保护规则（第 5 节）—— **被 protected / immutable 规则匹配的 tag 不会被清理策略删除**，这才是"制度级"保障。

---

# 5. tag 与仓库保护：protected / immutable / package protection

## 5.1 Protected container tags（Free 起可用）

**控制"谁能推/删某类 tag"**，最多 **5 条规则/项目**，作用于项目内所有容器仓库。

```text
Settings → Packages and registries → Container registry → Protected container tags → Add protection rule
  Protect container tags matching:  ^v\d+\.\d+\.\d+$        # RE2，≤100 字符
  Minimum role allowed to push:     Maintainer
  Minimum role allowed to delete:   Owner
```

**四条关键语义**：

1. **只要有 ≥1 条规则匹配，该 tag 即受保护；多条匹配时取最严格的那条。**
2. **被保护的 tag 不会被清理策略删除**（这是比 `name_regex_keep` 更强的保障）。
3. **规则变更有传播延迟**：依赖 JWT token 过期，**默认 5 分钟，GitLab.com 15 分钟**。所以"我刚加了保护，为什么 Developer 还能推"通常是延迟，不是配置无效。
4. **需要新版容器仓库**：GitLab.com 默认启用；**自建必须先启用 registry metadata database**。

**两个连带影响（容易误判为 bug）**：

- GitLab UI/API 不支持直接删 manifest；**直接调 registry API 删 manifest 会影响该 manifest 的所有 tag**，且只有"保护未开启"或"你有权删受保护 tag"时才允许。
- **删除容器镜像**的判定是：镜像有 tag + 项目存在任意 tag 保护规则 + 你的权限低于任一规则的 `minimum_access_delete_level` → **禁止删除**，**与规则模式是否匹配该 tag 无关**。也就是说"加了一条只保护 `latest` 的规则"会让普通 Developer 删不掉任何带 tag 的镜像。

## 5.2 Immutable container tags（Ultimate）

| | Protected tags | Immutable tags |
|---|---|---|
| 目的 | 限制**谁能**创建/更新/删除 | **创建后任何人都不能更新/删除** |
| 订阅 | Free/Premium/Ultimate | **Ultimate** |
| 创建所需角色 | Maintainer/Owner | **Owner** |
| 规则本身 | 可增可删 | **只能创建，不能更新**（可删除） |
| 与清理策略 | 匹配的 tag 不被清理 | 同 |

推送时的判定顺序（官方流程图口径）：

```text
推 tag ──► ① 受保护规则：角色够吗？不够 → 拒绝（Protected tag - insufficient permissions）
        └─► ② tag 已存在吗？不存在 → 创建成功
                          已存在 → ③ 是否命中 immutable 规则？是 → 拒绝（Tag is immutable）
                                                                  否 → 覆盖成功
```

**实践建议**：`v\d+\.\d+\.\d+`（正式版本）上 immutable；`latest`/分支滚动 tag 用 protected 限角色。**先确认自建实例已启用 metadata database**，否则规则不生效。

## 5.3 包保护规则（Package protection rules）

Package Registry 侧的对应能力：**Settings → Packages and registries → Package registry → Protected packages → Add protection rule**。

- 字段：**Name pattern**（支持 `*` 通配）、**Package type**、**Minimum access level for push**、**Minimum access level for delete**。
- 支持格式：Cargo、Conan、Generic、Helm、Maven、npm、NuGet、PyPI、RubyGems、Terraform module。
- 至少一条规则匹配即受保护；多条匹配取最严格。
- **受保护的包不能用 Deploy Token 推送**（规则匹配就拒绝）—— 这是最常见的"CI 突然 403"原因之一。
- **先保护后发布 → 发布会被拒**（`403 Forbidden` / `Authorization failed`）。要保护"正式发布版本"，正确做法是保护 `*-RELEASE`/精确名字模式，别把 SNAPSHOT 也纳入。

---

# 6. 上游代理与多仓聚合

| 能力 | 解决什么 | 关键前提 |
|------|----------|----------|
| **Dependency proxy（容器）** `<host>/dependencies/docker` | 把 Docker Hub 等上游镜像缓存在实例内，避免每个 Runner 各自出网拉、避免被上游限流 | 自建需在 `gitlab.rb` 启用 `registry['dependency_proxy']['enabled'] = true` 并 `gitlab-ctl reconfigure`；实例需可出网 |
| **Dependency proxy（包）** | npm/PyPI/Maven/Go 等外网包代理缓存 | 按格式分别开关，见"Dependency proxy for packages" |
| **Virtual registry** | 在群组下用一个前缀聚合多个上游仓库（GitLab 群组 / Harbor / Quay 等），客户端只需配一个地址 | 需要新版容器仓库/metadata database；上游类型与订阅有关，**落地前先读该页 Prerequisites** |
| **Harbor registry 集成** | 已有 Harbor 时统一入口 | 见 Harbor 专页 |

**用法示意（容器 dependency proxy）**：

```bash
# 登录实例后按上游路径拉取，实例负责缓存与回源
docker login <your-gitlab-host>
docker pull <your-gitlab-host>/dependencies/docker/library/alpine:3.20
docker tag  <your-gitlab-host>/dependencies/docker/library/alpine:3.20 registry.example.com/base/alpine:3.20
docker push registry.example.com/base/alpine:3.20
```

> 具体登录所需的凭据类型（PAT / Deploy Token / CI Job Token）随版本与实例配置不同，**以 dependency proxy 页面为准**，不要照抄旧博客。

---

# 7. Maven 包仓库：Java 团队的主战场

## 7.1 认证：header 名必须与令牌类型匹配

GitLab Maven 仓库支持**自定义 HTTP header** 或 **Basic Auth**，但 **header 名必须与令牌类型严格对应**：

| 令牌类型 | header 名（必须） | 值 |
|----------|-------------------|----|
| Personal access token | `Private-Token` | PAT 原文 |
| Deploy token | `Deploy-Token` | token 原文 |
| CI Job token | `Job-Token` | `${CI_JOB_TOKEN}` |
| OAuth token | `Authorization` | `Bearer <oauth_token>` |

```xml
<!-- ci_settings.xml：CI 用 Job-Token，天然短时效 -->
<settings>
  <servers>
    <server>
      <id>gitlab-maven</id>
      <configuration>
        <httpHeaders>
          <property>
            <name>Job-Token</name>
            <value>${CI_JOB_TOKEN}</value>
          </property>
        </httpHeaders>
      </configuration>
    </server>
  </servers>
</settings>
```

Basic Auth 等价写法（CI Job token 用户名固定为 `gitlab-ci-token`）：

```xml
<server>
  <id>gitlab-maven</id>
  <username>gitlab-ci-token</username>
  <password>${env.CI_JOB_TOKEN}</password>
</server>
```

> **文档明确警告**：不要使用文档之外的认证方式，未记录的认证方式可能被随时移除。

## 7.2 发布与消费

```xml
<!-- pom.xml：发布必须指向具体项目；仓库可用项目/群组/实例端点 -->
<repositories>
  <repository>
    <id>gitlab-maven</id>
    <url>${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/maven</url>
  </repository>
</repositories>
<distributionManagement>
  <repository>
    <id>gitlab-maven</id>
    <url>${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/maven</url>
  </repository>
  <snapshotRepository>
    <id>gitlab-maven</id>
    <url>${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/maven</url>
  </snapshotRepository>
</distributionManagement>
```

```yaml
publish-maven:
  stage: publish
  script:
    - mvn -B -ntp -s ci_settings.xml deploy          # 少了 -s 就会 401
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_TAG
```

**Gradle 等价（关键片段）**：

```groovy
publishing {
  publications { library(MavenPublication) { from components.java } }
  repositories {
    maven {
      url "https://gitlab.example.com/api/v4/projects/${System.getenv("CI_PROJECT_ID")}/packages/maven"
      credentials(HttpHeaderCredentials) { name = "Job-Token"; value = System.getenv("CI_JOB_TOKEN") }
      authentication { header(HttpHeaderAuthentication) }
    }
  }
}
```

## 7.3 端点选择与两个经典坑

| 端点 | URL | 坑 |
|------|-----|----|
| Project | `/api/v4/projects/<id>/packages/maven` | 多项目消费时要拼很多 URL → 交给群组端点 |
| Group | `/api/v4/groups/<group_id>/-/packages/maven` | **不保证组内包名唯一**：同名同版本存在时**返回最近更新的那个** —— 静默拿到别人的包 |
| Instance | `/api/v4/packages/maven` | 只有 `groupId=group-slug[.subgroup]` + `artifactId=project-slug` 的包才暴露 |

**坑 1（供应链向）**：群组端点的重名"取最新"意味着**任何组内项目都能用一个同名同版本包影响解析结果**。治理方式：包名加项目前缀、把发布权收到受保护的发布项目、并启用 **Duplicate packages** 关闭重复（7.4）。

**坑 2（Maven Central 转发）**：GitLab 提供"包找不到时转发到 Maven Central"的能力（受 feature flag 控制，管理员可在 CI 设置里关），**且只对 project/group 端点生效**，instance 端点因命名限制不提供。想让 Maven 走 GitLab 代理，需覆盖 `central`：

```xml
<mirrors>
  <mirror>
    <id>central-proxy</id>
    <name>GitLab proxy of central repo</name>
    <url>https://gitlab.example.com/api/v4/projects/<id>/packages/maven</url>
    <mirrorOf>central</mirrorOf>
  </mirror>
</mirrors>
```

## 7.4 重复包、版本校验与 SNAPSHOT

- **默认允许重复**：同名同版本再次发布时，**文件被追加到既有包**（旧 asset 仍可访问）。要禁止：**群组 Settings → Packages and registries → Duplicate packages → 关闭 Maven 的 Allow duplicates**，可用 **Exceptions** 正则放行例外。
- 用 `DeployAtEnd`（Gradle/插件在结束时统一上传）可能报 `400 bad request {"message":"Validation failed: Name has already been taken"}`。
- **版本字符串有校验规则**：`\A(?!.*\.\.)[\w+.-]+\z`；不合法会报 `400 Bad Request ... Validation failed: Version is invalid`（例如 `1..0`、`1.0.`）。**用 CI 变量拼版本号时务必先归一化**。
- **SNAPSHOT** 可以覆盖，但每次 deploy 会生成带时间戳的新文件名（`app-1.0-20240102.090000-11.jar`），**这是包清理策略的主要负担来源**（见 12 节）。

---

# 8. Generic 仓库：被低估的万能货架

```bash
# 发布：数据库脚本包、离线补丁、安装包、模型文件……
curl --fail --silent --show-error --request PUT \
     --header "JOB-TOKEN: $CI_JOB_TOKEN" \
     --upload-file bundle.tar.gz \
     "https://gitlab.example.com/api/v4/projects/$CI_PROJECT_ID/packages/generic/release-assets/1.2.3/bundle.tar.gz"

# 消费（同一项目 CI 内）
curl --fail --location --header "JOB-TOKEN: $CI_JOB_TOKEN" \
     -o bundle.tar.gz \
     "https://gitlab.example.com/api/v4/projects/$CI_PROJECT_ID/packages/generic/release-assets/1.2.3/bundle.tar.gz"
```

**为什么值得优先用 Generic**：没有对应格式的语言/资产（内部 DSL 包、签名前的 APK、Ansible collection 源码包）都能放进去，**版本化 + 权限 + UI 展示 + API 删除全部白送**，不需要额外服务。

```yaml
# 实用组合：把"发布 → 拿 URL → 写进 Release"串起来
publish-generic:
  stage: package
  script:
    - PKG_URL="https://$CI_SERVER_HOST/api/v4/projects/$CI_PROJECT_ID/packages/generic/app/$APP_VERSION/app-$CI_COMMIT_SHORT_SHA.tar.gz"
    - 'curl --fail -PUT --header "JOB-TOKEN: $CI_JOB_TOKEN" --upload-file app.tar.gz "$PKG_URL"'
    - printf 'ASSET_URL=%s\n' "$PKG_URL" > asset.env
  artifacts:
    reports:
      dotenv: asset.env

link-release:
  stage: release
  needs: [{ job: publish-generic, artifacts: true }]
  release:
    tag_name: v$APP_VERSION
    assets:
      links:
        - name: "离线安装包"
          url: "$ASSET_URL"
```

**注意**：Generic 包 URL 里**必须带文件名**（`.../1.2.3/bundle.tar.gz`），只写到版本号会 404。

---

# 9. Go Module 与 npm

- **Go**：私有 module 用 `GOPRIVATE=gitlab.example.com` + `git config --global url."git@gitlab.example.com:".insteadOf "https://gitlab.example.com/"`（SSH），或走 **Go Proxy 端点** + token（`GONOSUMDB`/`GONOSUMCHECK` 视代理配置）。CI 内推荐 HTTPS + `:_param_`/PAT 或 CI Job Token 形式，**具体端点以 Go Proxy 页面为准**。
- **npm**：

```ini
# .npmrc（作用域挂到项目或群组仓库）
@pay:registry=https://gitlab.example.com/api/v4/projects/123/packages/npm/
//gitlab.example.com/api/v4/projects/123/packages/npm/:_authToken=${CI_JOB_TOKEN}
```

```bash
npm publish --registry https://gitlab.example.com/api/v4/projects/123/packages/npm/
```

> **npm 的两个已知限制**：**不支持 npm dist-tag**，且**下载 tarball 的 URL 带 token（以签名 URL 形式存在）** —— 这意味着 npm 场景下**权限控制更粗**，敏感私有包要评估是否放 npm 端点（第 15 篇"发布通道"讨论过，建议改 Generic）。

- **Helm**：可用传统 Helm 仓库端点，也可用 OCI 方式把 chart 推到容器仓库（`helm push ... oci://<registry>/<group>/<project>/charts`，需 Helm 3.8+）。

---

# 10. 权限模型与令牌选择

## 10.1 角色矩阵

| 操作 | 所需角色/令牌 |
|------|---------------|
| 查看/拉取（私有项目） | Reporter 起（公开项目更低） |
| 发布包 / 推镜像 | **Developer / Maintainer / Owner** |
| 删除包、删除资产 | **Maintainer / Owner** |
| 创建 tag/包保护规则 | Maintainer/Owner（immutable 规则需 **Owner**） |
| 修改清理策略 | Maintainer/Owner（群组/实例级可覆盖项目） |

## 10.2 令牌对照

| 令牌 | 生命周期 | 适用 | 注意 |
|------|----------|------|------|
| `CI_JOB_TOKEN` | **单 Job** | CI 推拉本项目制品 | 跨项目需入站授权；不用于克隆仓库 |
| **Deploy Token** | 长期（可设过期） | 外部系统只读拉包 | **受包保护规则匹配的包不能用 deploy token 推送** |
| **Project/Group Access Token** | 长期 | 机器人发版、批量管理 | 需授予成员身份 + `read_api`/`write_repository` 等 scope |
| **Personal Access Token** | 跟人走 | 本地调试 | **不要进 CI**（离职即断链、权限过大） |
| `CI_REGISTRY_USER/PASSWORD` | 单 Job | 容器仓库 | 只覆盖本项目仓库 |

## 10.3 跨项目消费：Job token permissions

```text
被访问的项目（提供 SDK/基镜像的那个）
  → Settings → CI/CD → Job token permissions
  → 把消费方项目加入 Allowlist（或加入同群组以简化）
```

**症状对照**：下游流水线里 `mvn` 报 `401 Unauthorized`、`docker pull` 报 `unauthorized to access repository` —— 90% 是这个入站白名单没配，而不是令牌本身有问题。

## 10.4 行业实践纪律

> **人的账号尽量不直接 push 镜像/包**（本地 `mvn deploy` 绕过构建审计）；只允许 CI 推，发布 Job 绑定保护分支/Tag（第 12 篇），凭据用短时效 `CI_JOB_TOKEN` 或专用 Deploy Token。

---

# 11. 包与 Release / MR 的关联

制品的"最后一公里"不是 push 成功，而是**别人能在正确的位置找到它、并且知道它是哪次流水线产出的**。

## 11.1 包页与 API：先看清"谁发的、哪个流水线"

- 入口：**Deploy → Package registry**（旧路径 Project → Packages），可按 `package_type`、名称、版本过滤；**Deploy → Container Registry** 显示 tag、大小、平台徽章与签名。
- 包列表 API 是最省事的对账工具（发版后核对"应该有的包都在吗"）：

```bash
glab api "projects/$CI_PROJECT_ID/packages?package_type=maven&per_page=50" \
  | jq -r '.[] | "\(.id)\t\(.name):\(.version)\t\(.status)\t\(.created_at)"'
# 单个包的资产清单（文件名 + 大小，排查"少上传了一个文件"）
glab api "projects/$CI_PROJECT_ID/packages/$PKG_ID/package_files" \
  | jq -r '.[] | "\(.file_name)\t\(.size)"'
```

## 11.2 挂到 Release 上（对外交付的唯一入口）

发布 Job 用 `release:assets:links` 把**镜像地址、包坐标、通用文件下载地址**一起写进版本页，使用方只看 Release：

```yaml
release-notes:
  stage: release
  needs:
    - { job: publish-maven, artifacts: true }
    - { job: package-image, artifacts: true }
  release:
    tag_name: "v$APP_VERSION"
    description: "见 CHANGELOG.md"
    assets:
      links:
        - name: "镜像（digest 固定）"
          url: "$CI_REGISTRY_IMAGE@$IMAGE_DIGEST"
          link_type: "package"          # 合法值：runbook|image|package|documentation|other
        - name: "Maven 坐标"
          url: "$CI_PROJECT_URL/packages/maven"
          filepath: "com.acme:order-api:$APP_VERSION"
```

**为什么强调 digest 而不是 tag**：tag 可被覆盖（除非上了 immutable 规则，第 5.2 节），`@sha256:` 才是可审计的交付物标识；把 digest 写进 Release，事故复盘时才能确定"当时上线的到底是哪个字节序列"。

## 11.3 删除语义：异步、且会留痕

```bash
# 删包（Maintainer/Owner）：立即返回，后台真删，期间 status=deleting
glab api --method DELETE "projects/$CI_PROJECT_ID/packages/$PKG_ID"
# 只删某个资产文件（例：误传的大文件）
glab api --method DELETE "projects/$CI_PROJECT_ID/packages/$PKG_ID/package_files/$FILE_ID"
# 删镜像 tag（受保护 tag 会被拒；见 5.1 的"删不掉"判定顺序）
glab api --method DELETE "projects/$CI_PROJECT_ID/registry/repositories/$REPO_ID/tags/$TAG"
```

| 认知点 | 实际行为 |
|--------|----------|
| "删了就应该消失" | 包删除是**异步**的，UI 里先变 `deleting`；大对象存储桶下更慢 |
| "删包 = 释放空间" | 空间统计与配额是**周期性重算**的，别指望立刻掉；镜像 blob 还要 GC（4.3） |
| "删包没风险" | 开启 request forwarding 时，删掉的 Maven/ npm 包可能被**公共仓库同名包**顶上 → dependency confusion（12 节） |
| "用 API 删更安全" | API 删除同样不可恢复，且**不会**因为"这个版本还有人依赖"而拦住你。删除前列一遍依赖方 |

## 11.4 包页变慢 / 包数量爆炸时的治理顺序

包多到"页面卡、API 慢"时，**先归因再动手**（实例设置项名称随版本变化，别照抄网上片段）：

1. **统计**：按类型与项目分组数包，找出真正的贡献者（通常是 SNAPSHOT 高频发布与 Generic 大文件）：

```bash
glab api "projects/$P/packages?per_page=100&order_by=created_at" \
  | jq -r '.[].package_type' | sort | uniq -c | sort -nr
```

2. **掐源头**：把发布 Job 限制到 tag/默认分支（`rules`），**关闭重复包**（群组 Settings → Packages and registries → Duplicate packages，7.4），Generic/CI 临时产物设短 `expire_in`。
3. **清存量**：配置**包清理策略**（按"重复资产数量"语义，12 节）与**容器 tag 清理策略**（4 节），注意两者都有小时级延迟。
4. **兜底**：仍慢就查后台任务与索引（管理员侧：包清理 worker 容量、`Package::Pipeline` 类历史数据迁移），而不是继续加机器。

> **注意**：GitLab 的包页没有"按依赖方反查"的能力 —— 删包前想知道"谁在用这个版本"，只能靠你自己的构建日志/锁文件盘点（第 15 篇的治理脚本）。

---

# 12. 体积治理与成本

| 对象 | 控制手段 | 生效位置 |
|------|----------|----------|
| 镜像 tag | 容器清理策略（`older_than`/`keep_n`/豁免正则） | 项目/群组/实例 |
| 镜像 blob | **registry garbage collection** | 实例（管理员） |
| CI artifacts | `artifacts:expire_in`（第 6 篇） | Job 级 |
| 包资产 | **Package cleanup policy**（重复资产数量） | 项目 |
| 未使用的 Generic 包 | `deleteGenericPackagesNotDownloadedInDays`（GraphQL，试验性） | 项目 |

**包清理策略的现行语义（与旧文档不同，容易误配）**：

- UI 位置：**Settings → Packages and registries → Package registry → Manage storage used by package assets**。
- 判定"重复资产"的**两个条件**：同一包（多数格式 = 同名同版本）+ **文件名完全相同（大小写敏感，不支持通配）**。默认 **Number of duplicate assets to keep = All（即不删）**。
- **Maven SNAPSHOT 特殊处理（Beta）**：策略会忽略文件名里的时间戳/build 号来比对同一文件，按 build 顺序保留最新 N 份。**风险**：策略若在 deploy 上传中途或失败后运行，可能保留**不完整 build** 并删掉 `maven-metadata.xml` 指向的文件 → 请求返回 **404，直到下一次成功 deploy**。**且删除不可恢复。**
- **执行频率**：约每 **12 小时**一次，**首次运行在保存设置约 12 小时后** —— "配了没效果"多半只是还没跑。
- 后台并发由实例设置 `package_registry_cleanup_policies_worker_capacity`（默认 2）控制，设 `0` = 不执行清理。

> **删包前必须知道**：**在开启 request forwarding（转发到 Maven Central 等）的情况下删除包，会引入 dependency confusion 风险** —— 外网同名的公开包会被解析到。删包前先确认该坐标不在公共仓库存在，或关闭转发。

---

# 13. 常见报错原文与处理

| 报错 | 原因 | 处理 |
|------|------|------|
| `denied: requested access to the resource is denied`（docker push） | 未登录 / 路径拼错（大小写、层级 >2） / 项目未开 Container registry | 检查 `CI_REGISTRY_IMAGE`、`visibility`、登录域是否为 `CI_REGISTRY` |
| `unauthorized to access repository`（跨项目 pull） | Job token 未入站授权 | 被访问项目 **Job token permissions** 加白名单 |
| `401 Unauthorized`（mvn deploy） | **漏了 `-s settings.xml`**（第一高频）、令牌过期、scope 不足 | `mvn help:effective-settings` 验证；always `-s` |
| `403 Forbidden` + `Authorization failed`（发布包） | 角色不足（需 Developer+）、**包被 push 保护规则匹配**、deploy token 推受保护包 | 检查 Package protection rules；确认 CI 用的是 Job-Token |
| `400 Bad Request ... Validation failed: Version is invalid` | 版本字符串不合规（`1..0`、`1.0.`、含非法字符） | 按 `\A(?!.*\.\.)[\w+.-]+\z` 归一化版本号 |
| `400 bad request {"message":"Validation failed: Name has already been taken"}` | 重复发布 + 关闭了 Allow duplicates（或 DeployAtEnd 行为） | 升版本号，或加 Exceptions 放行 |
| `error pushing image: failed to copy: ... 500 Internal Server Error` | Registry 存储/网关异常（大层上传、代理超时） | 查 `gitlab-ctl tail registry`、`proxy_read_timeout`/`registry` 相关设置 |
| `404` 下载某个 SNAPSHOT 文件 | 清理策略删掉了 `maven-metadata.xml` 指向的 build 文件 | 重新 deploy 一次成功构建；调大 `duplicate assets to keep` |
| `Adding tag to ... : received unexpected HTTP status: 500` 或推 tag 被拒 | 命中 **protected/immutable** 规则；或保护规则 JWT 尚未过期导致行为不一致 | 用 Owner 操作，或调整规则；等待 token 过期（默认 5 分钟） |
| UI 里"镜像不存在"但 `docker pull` 成功 | metadata database 未启用或迁移 worker 未完成 | 启用 `registry_metadata_database`（自建） |
| 清理策略保存后空间没降 | 只删 tag 不删 blob；GC 未启用 | 安排 registry GC（第 4.3 节） |
| `curl` 上传 Generic 包返回 `404` | URL 少文件名（只到版本号） | 补 `/1.2.3/bundle.tar.gz` |

**自助验证三连**（把制品侧问题快速分层）：

```bash
IMG=registry.example.com/pay/checkout:1.4.2
docker manifest inspect -v "$IMG" | jq '.[].Descriptor.digest'      # 仓库里到底有什么
curl -s --header "JOB-TOKEN: $CI_JOB_TOKEN" \
  "$CI_API_V4_URL/projects/$CI_PROJECT_ID/registry/repositories" | jq '.[].location'
curl -s --header "JOB-TOKEN: $CI_JOB_TOKEN" \
  "$CI_API_V4_URL/projects/$CI_PROJECT_ID/packages?order_by=created_at&per_page=20" | jq -r '.[] | [.name,.version,.package_type] | @tsv'
```

---

# 14. 小结

| 要点 | 一句话 |
|------|--------|
| 定位 | 制品与代码、CI 同平台，权限与溯源天然打通；外网代理重活可仍留给 Nexus/Harbor |
| 镜像路径 | `<registry>/<group>/<project>[/子路径 ≤2 层]`，**与项目路径绑定，改项目路径等于改镜像地址** |
| tag 纪律 | 唯一标识用 commit SHA/digest；`latest` 只做便捷入口；回滚永远基于不可变 tag 或 digest |
| 清理策略 | 项目/群组/实例三层合并，**实例默认策略看不到但会继承**；必须配豁免；**被保护/immutable 的 tag 不会被清理**；**删 tag ≠ 释放磁盘，GC 才释放** |
| 保护 | protected tags（限角色，Free 起）+ immutable tags（**Ultimate**，Owner 创建、只能创建不能更新）+ package protection rules（**受保护包不能用 deploy token 推**）；规则生效有 **5/15 分钟 JWT 延迟** |
| Maven | 三种端点各有约束（群组端点**不保证包名唯一**、实例端点要求坐标=项目路径）；认证 header 名必须匹配令牌类型；**永远带 `-s`**；默认允许重复资产 |
| Generic | 任意资产的版本化货架，URL 必须带文件名 |
| 空间 | 包清理策略按"重复资产数量"语义、每 12 小时跑、首次要等 12 小时；**删包 + request forwarding = dependency confusion 风险** |
| 原则 | **人拉 CI 推，凭证短时效**；跨项目消费先配 Job token permissions |

**自检清单**：

- [ ] 我能说出实例/群组/项目三层清理策略的合并与"隐藏继承"行为，并解释为什么删掉项目策略反而更危险？
- [ ] 我知道加了 tag 保护规则后，普通 Developer 会连"不匹配规则的镜像"都删不掉吗？
- [ ] 我能写出 CI 里 Maven 发布所需的 `settings.xml`（header 名 = `Job-Token`）并解释 `401` 的第一排查项为什么是漏 `-s`？
- [ ] 我能区分 protected container tags（Free/Premium/Ultimate）与 immutable container tags（Ultimate）的能力边界？
- [ ] 我知道清理策略只删 tag、要 GC 才释放磁盘，并且包清理首次运行要等约 12 小时？

> 制品就位。下一篇回到日常协作：[第 11 篇 Issue、看板与 Wiki](11_issues_boards_wiki.md) —— 需求侧如何与交付流咬合。
