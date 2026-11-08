# 1. 内置包仓库：交付物的"最后一公里"

GitLab 除了 Container Registry，还内置一整套 **Package Registry**：Maven、npm、PyPI、NuGet、Go Module、Conan、Rpm/Debian 等，与项目/群组原生打通——**CI 产出的 jar 和镜像可以直接 push 回同一个平台**，不需要再搭 Nexus/Harbor（或作为其前置）。

| 仓库类型 | 路径规则 | 典型用途 |
|----------|---------|---------|
| Container Registry | `registry.example.com/<group>/<project>[:tag]` | CI 构建应用镜像、基镜像 |
| Maven | `https://gitlab.example.com/api/v4/projects/<ID>/packages/maven` | 内部 SDK/BOM 发布与消费 |
| npm | `.../packages/npm/` | 前端私有组件 |
| Generic | `.../packages/generic/<name>/<version>` | 任意文件（SQL 包、配置包、离线补丁） |

# 2. Container Registry 使用

每个项目默认自带命名空间 `registry.example.com/group/project`，直接在 CI 里用预定义变量构建推送：

```yaml
build-image:
  stage: build
  image: docker:27
  services:
    - docker:27-dind          # 生产更推荐 kaniko/buildah，见第 8 篇 Runner 安全
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    # ^ 这两个变量是 CI 自动注入的"临时部署令牌"，只对本项目 Registry 有效，job 结束即失效
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
    - docker tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA $CI_REGISTRY_IMAGE:stable
    - docker push $CI_REGISTRY_IMAGE:stable
```

要点：

- **群组级 Registry**（Group Container Registry）：镜像归属群组而非单项目，适合基镜像/跨项目共享镜像（`registry.example.com/group/base`）。
- 拉取权限 = 项目 Reporter 角色；推送 = Developer 且通常只开放给 CI 令牌（人的 token 一般不给推镜像）。
- 支持多架构：buildx + manifest list 推 `linux/amd64,linux/arm64` 到同一 tag。

# 3. 镜像清理策略（保留策略）

Registry 会无限膨胀——每次 CI 都推新 tag，几百 GB 很快耗尽磁盘。GitLab 提供 **Cleanup Policies**（Admin/项目 → Settings → Packages and registries → Container Registry cleanup）：

| 策略 | 作用 | 建议 |
|------|------|------|
| 保留最新 N 个 tag | 每分支留最近 n 个（默认 10） | 按需 |
| 旧镜像清理（older_than） | 超过 N 天且非最新的一律删 | 生产：`14d` |
| 豁免正则（keep_n/tags_to_keep） | `stable|prod.*` 永不删 | **必须配**，否则回滚 tag 被清掉 |
| 拉取限制（pull rate limit） | 防外部刷流量 | 公网暴露时开 |

> **注意**：清理只删"未被任何 tag 引用的 manifest"。**给回滚用的镜像一定打不可变 tag（如 `prod-v1.2.3`）并加入豁免**，别依赖 `latest`。

# 4. Maven 包仓库：Java 团队的主战场

发布到 GitLab Maven 仓库：

```xml
<!-- settings.xml：用 CI 变量注入凭证 -->
<server>
  <id>gitlab-maven</id>
  <configuration>
    <username>${env.CI_DEPLOY_TOKEN_USERNAME}</username>
    <password>${env.CI_JOB_TOKEN}</password>
  </configuration>
</server>
```

```xml
<!-- 项目 pom.xml -->
<distributionManagement>
  <repository>
    <id>gitlab-maven</id>
    <url>https://gitlab.example.com/api/v4/projects/${project.registry_id}/packages/maven</url>
  </repository>
</distributionManagement>
```

```bash
# CI 中发布
mvn -B deploy -DaltDeploymentRepository=gitlab-maven::default::\
https://gitlab.example.com/api/v4/projects/$CI_PROJECT_ID/packages/maven
```

消费端（依赖内部 SDK）：`<repositories>` 同样指向 GitLab，或走 CI 模板统一注入 `settings.xml`。

**设计考量**：

- GitLab Maven 仓库**只读优先**：多数团队把它当"发布产物仓"，代理外网中央仓库的能力仍弱于 Nexus——**外网依赖代理 + 内部发布**建议仍留给 Nexus/Harbor，或 Ultimate 的群组 Maven 虚拟仓库。
- `CI_JOB_TOKEN` 可跨项目拉包（上游 SDK 项目授权给下游项目即可），这是内源复用的关键机制。

# 5. Generic 仓库：被低估的万能货架

```bash
# 发布：数据库脚本包、离线补丁、合同模板……
curl --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
     --upload-file bundle.tar.gz \
     "https://gitlab.example.com/api/v4/projects/$CI_PROJECT_ID/packages/generic/release-assets/1.2.3/bundle.tar.gz"

# 消费
curl --header "PRIVATE-TOKEN: $GITLAB_TOKEN" -L -o bundle.tar.gz \
     "https://gitlab.example.com/api/v4/projects/$CI_PROJECT_ID/packages/generic/release-assets/1.2.3/bundle.tar.gz"
```

没有对应格式的语言/资产（比如内部 DSL 包、模型文件、签名前的 APK）都走 Generic，版本化 + 权限控制全白送。

# 6. Go Module 与 npm

- **Go**：GitLab 原生支持私有 module 代理——`GOPRIVATE=gitlab.example.com` + `git config insteadOf` 走 SSH，或配置群组级 module 代理令牌；`go get` 无需额外服务。
- **npm**：项目 `.npmrc` 写 `@scope:registry=https://gitlab.example.com/api/v4/projects/<ID>/packages/npm/`，发布 `npm publish --registry ...`，作用域包建议挂群组级仓库供全部门共享。

# 7. 包与 Release / MR 的关联

- 发布包后，在 **Project → Packages** 可见每个包的"来源 MR/Tag/Pipeline"，供应链可追溯。
- **Release 资产**：Release Notes 里可挂指向 Registry 镜像/包的链接，或上传通用文件作为 release assets——对外交付时把"下载入口"集中到版本页。

# 8. 权限模型

| 操作 | 所需角色 |
|------|---------|
| 拉取包/镜像 | Guest（公开项目）/ Reporter（私有）起 |
| 上传（CI 推送） | CI Job Token 即可（默认） |
| 删除包 | Maintainer |
| 群组包仓库授权 | Owner/Maintainer 配置 access token 作用范围 |

> **行业实践**：**人的账号尽量不直接 push 镜像/包**（本地 mvn deploy 会绕过构建审计）；只允许 CI 推，且发布 job 绑定受保护分支/Tag，凭证用短时效的 `CI_JOB_TOKEN` 或专用 Deploy Token。

# 9. 小结

| 要点 | 一句话 |
|------|--------|
| 定位 | 制品与代码、CI 同平台，权限与溯源天然打通 |
| 镜像 | CI_PROJECT_ID + 注入令牌即可推拉；群组 Registry 放基镜像 |
| 清理 | 保留策略必须配，回滚 tag 要豁免 |
| Maven | 发布/内部消费够用，外网代理仍需 Nexus |
| Generic | 任意资产的版本化货架 |
| 原则 | 人拉 CI 推，凭证短时效 |

制品就位。下一篇回到日常协作：Issue、标签、看板与路线图——需求侧如何与交付流咬合。
