
# 1. Registry 在交付链路中的位置

Registry 是镜像的制品库：

1. **存储与分发**：push/pull 的目标端。
2. **版本与回滚**：tag/digest 作为版本标识。
3. **权限与审计**：谁能 push、谁能 pull、谁做了什么。

从协议视角看，Registry 只是一个实现了 **Distribution Spec（OCI Distribution Specification）** 的 HTTP 服务：镜像清单（manifest）、层（blob）、配置对象都以内容寻址（content-addressable）方式存储——寻址键是内容的 SHA256 digest，这天然带来去重与防篡改两个属性。同一份基础层被一万个镜像引用，磁盘上只存一份；任何人拿到 digest 就能自行校验内容，无须信任传输通道。

工程结论：

1. 不要把 Registry 只当"下载站"，它是交付链路核心。
2. 生产发布建议明确 tag 策略，并尽量使用 digest 锁定关键环境部署（见 `4_image.md`）。

# 2. docker push / pull 的分层上传原理

`docker push` 并不上传"一个 tar 包"，而是按 manifest 逐层协商上传，这决定了内外网仓库性能表现的核心机制：

```text
docker push my-registry/my-app:1.0

1. PUT /v2/my-app/manifests/1.0 (尝试创建)
     └─ Registry 返回已存在的 blob 列表? 否 → 进入上传流程
2. POST /v2/my-app/blobs/uploads/            ← 为每个缺失层发起上传会话
3. PUT/PATCH ...?digest=sha256:<layer>       ← 流式上传层数据(可断点续传)
4. 相同 digest 的层直接跳过("Layer already exists")
5. PUT /v2/my-app/manifests/1.0              ← 最后写入 manifest(原子发布点)
```

三个工程推论：

1. **跨镜像去重**：两个服务共用同一基础镜像层时，第二个 push 几乎秒传（全部 `Layer already exists`）；pull 同理，节点扩容时大量层已缓存。
2. **manifest 最后写**：层上传完但 manifest 未写之前，镜像对拉取方不可见——不存在"拉到半截镜像"的问题，但 CI 中断可能留下无 manifest 引用的**孤儿 blob**，这就是需要 GC 的原因。
3. **层太大伤重试**：单层几百 MB 时弱网重试成本高，所以构建时要把大产物拆稳（见 `8_dockerfile_best.md` 的层设计）；registry:2 对单 blob 大小默认无硬限制，但代理层（Nginx `client_max_body_size`）常截断大层上传，这是自建仓库"push 413/504"的第一嫌疑人。

多架构镜像额外有 **manifest list（fat manifest）**：一次 `buildx build --platform linux/amd64,linux/arm64` 会推送各架构 manifest 再加一个索引 manifest，客户端按自己的 GOARCH 选择子 manifest。

# 3. Docker Hub（公共仓库）

```bash
docker login

docker tag my-app:1.0 username/my-app:1.0
docker push username/my-app:1.0

docker pull username/my-app:1.0
```

注意：

1. 公共仓库有速率限制与网络波动风险（匿名与付费账号的拉取限额不同，K8s 节点大规模匿名 pull 很容易触发 429），生产更常用企业私有仓库或云厂商镜像加速服务。
2. 对外发布需要考虑镜像瘦身与敏感信息排除（`.dockerignore`、多阶段构建）。
3. **docker.io 的镜像名有隐式规则**：`nginx` 实际是 `docker.io/library/nginx`，`username/app` 是 `docker.io/username/app`；而自建的 `localhost:5000/app`、`harbor.example.com/proj/app` 都带显式域名。写部署脚本时统一使用全名，避免"少写域名导致拉错仓库"。

# 4. 私有 Registry（registry:2）

官方 `registry:2` 是单二进制最小实现，适合内网测试、离线交付与学习 Distribution API。

## 4.1 快速启动

```bash
docker run -d -p 5000:5000 --name registry \
  --restart unless-stopped \
  --mount type=bind,source=/data/registry,target=/var/lib/registry \
  registry:2
```

推送与查看：

```bash
docker tag my-app:1.0 localhost:5000/my-app:1.0
docker push localhost:5000/my-app:1.0

curl http://localhost:5000/v2/_catalog
curl http://localhost:5000/v2/my-app/tags/list
```

`localhost:5000` 免 TLS 是特例（回环地址默认允许 HTTP）；换成局域网 IP 就必须配置 HTTPS 或把地址加入 `insecure-registries`，否则报 `http: server gave HTTP response to HTTPS client`。

## 4.2 认证（htpasswd）

```bash
# 生成 htpasswd
docker run --rm httpd:alpine htpasswd -Bbn admin password > htpasswd

docker run -d -p 5000:5000 --name registry \
  --restart unless-stopped \
  --mount type=bind,source=/data/registry,target=/var/lib/registry \
  --mount type=bind,source=$(pwd)/htpasswd,target=/auth/htpasswd,readonly \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  registry:2
```

登录：

```bash
docker login localhost:5000
```

htpasswd 只能做"仓库级读写都同一个口令"粒度的控制；要按项目/人员分权，要么前置 Nginx 按路径做认证转发，要么直接上 Harbor。

## 4.3 TLS 与 insecure registry 的边界

生产私有仓库应使用 TLS（证书可信链/企业 CA）。

`insecure-registries`（允许非 TLS）只建议用于内网测试环境，并明确风险：

1. 传输可能被中间人攻击。
2. 凭证泄露风险更高。

> **注意**：insecure 配置要同步到**所有会拉镜像的组件**——dockerd、containerd（`/etc/containerd/config.toml` 的 hosts 配置）、K8s 的 kubelet、以及 CI runner，只改一处会出现"本机推得上去、集群拉不下来"。长期方案仍是把企业 CA 装入各节点的 `/etc/docker/certs.d/<registry>/ca.crt`（或系统信任库）。

## 4.4 registry:2 的 GC 与配额

原生 registry 只有文件级存储，**删除 tag 不等于释放空间**：

```bash
# registry:2 的 GC 需要显式开启环境变量后，在容器内执行垃圾回收
docker exec registry /bin/registry garbage-collect \
  --delete-untagged /etc/docker/registry/config.yml
```

它扫描 blob 目录、清理没有 manifest 引用的层，运行期间要停写（read-only 模式）以免并发删除引发数据不一致。配额、保留策略（如"每个仓库只保留最近 30 个 tag"）、按项目统计存储占用，registry:2 一概没有，**这正是中小团队最终都会迁到 Harbor 的原因**。

# 5. Harbor（企业级仓库）

Harbor 常被用作企业标准镜像仓库，典型原因：

1. 项目/租户隔离与 RBAC 权限。
2. 漏洞扫描（内嵌 Trivy）、镜像复制、审计。
3. 垃圾回收与保留策略（避免镜像无限增长）。

## 5.1 架构组成

```text
┌──────────────────────────── Harbor ────────────────────────────┐
│  Portal(UI/API 网关 nginx-core)                                 │
│        │                                                        │
│  Harbor Core ──── 项目/用户/RBAC/复制策略/保留策略/GC 调度         │
│        │            │                                           │
│  JobService     Registryctl → distribution registry:2 后端存储    │
│  (扫描/复制/GC任务)  │           │                               │
│        Trivy    PostgreSQL   Redis + 对象存储(S3/MinIO/OSS)       │
└─────────────────────────────────────────────────────────────────┘
```

要点：Harbor 把 registry 只当**存储后端**，权限、元数据、策略全在 core + PostgreSQL 侧；存储层挂对象存储才能在多副本部署下共享数据。

## 5.2 复制（Replication）与代理缓存

- **复制**：配置推送/拉取模式，把 A 站点的某些项目定时或事件触发同步到 B 站点，用于多机房容灾与就近拉取；按命名空间+标签规则过滤，注意复制是异步的，发布脚本不能假设"push 完所有站点都有"。
- **代理项目（Proxy Cache）**：把 Harbor 配置成 Docker Hub/GCR 的透明缓存，集群统一以 Harbor 为上游拉公共镜像，解决外网抖动与限流，同时获得扫描与审计。

```bash
docker login harbor.example.com
docker tag my-app:1.0 harbor.example.com/project/my-app:1.0
docker push harbor.example.com/project/my-app:1.0
```

> **行业实践**：K8s 侧访问私有 Harbor 的标准做法不是给每个 namespace 手写 imagePullSecret，而是用 `docker-registry` 类型的 Secret + ServiceAccount 默认挂载，或云原生环境下直接用节点 RAM/IRSA 角色换取临时凭证（Harbor 也支持对接 OIDC）。

# 6. 镜像签名与供应链完整性

tag 与 digest 解决"内容可寻址"，但**不证明"谁发布的"**。供应链攻击（镜像投毒、构建被劫持）需要签名验证闭环，当前主流是 Sigstore 体系的 **cosign**：

```bash
# 生成密钥对(或使用 OIDC 免密钥方式)
cosign generate-key-pair

# 对镜像签名(私钥在 CI 手里)
cosign sign --key cosign.key \
  harbor.example.com/project/my-app:1.2.3-8f3c2a1

# 部署前/准入时验签
cosign verify --key cosign.pub \
  harbor.example.com/project/my-app@sha256:abcd...
```

**设计考量**：

- **验签发生在两处**：CI 部署前脚本验证一次；K8s 用 Kyverno/OPA 准入控制器再验证一次（拒绝未签名镜像），Harbor 的"镜像签名策略"可做仓库级强制。三层一致才算闭环。
- **签名对象应是 digest 而非 tag**：tag 可覆盖，签名跟着不可变 digest 才有意义。
- 更进一步可配合 **SBOM**（`syft` 生成、`cosign attest` 挂载）与 **SLSA provenance**（证明构建链路与来源），Harbor 界面均可查看。

# 7. Tag 策略与"可回滚"

建议至少具备两类 tag：

1. **可读版本**：`1.2.3` 或 `1.2.3-rc1`
2. **可追溯版本**：`1.2.3-<git-sha>` 或单独的 `<git-sha>`

示例：

```bash
VERSION=1.2.3
GIT_SHA=$(git rev-parse --short HEAD)

docker tag my-app my-registry/my-app:$VERSION
docker tag my-app my-registry/my-app:$VERSION-$GIT_SHA
docker push my-registry/my-app:$VERSION
docker push my-registry/my-app:$VERSION-$GIT_SHA
```

生产更严格的方式：部署用 digest 锁定（避免 tag 漂移）。

| 标识 | 可变性 | 适用 |
|------|--------|------|
| `latest` | 可变 | 仅限本地开发; 生产禁用 |
| `1.2.3` | 约定可变(建议不可变) | 人工阅读与回滚选择 |
| `1.2.3-<git-sha>` | 事实不可变 | 追溯源码提交 |
| `@sha256:...` digest | 完全不可变 | 关键环境部署与准入比对 |

> **注意**：`latest` 不是"最新版"语义，只是普通 tag。两个镜像服务共用 `latest` 时，一次误 push 会让所有引用者在下一次重建容器时被"无感升级"，这是生产事故里的常客。配合仓库侧开启 **immutable tag（不可变标签）** 策略（Harbor 项目设置支持），从机制上禁止覆盖已有 tag。

# 8. Java 项目镜像命名规范

多人多服务的团队需要统一的命名约定，常见推荐格式：

```text
<registry>/<项目或团队>/<应用>:<版本>-<git短sha>

例: harbor.corp.com/pay/order-service:1.2.3-8f3c2a1
    harbor.corp.com/pay/gateway:1.2.3-8f3c2a1
```

**设计考量**：

- **命名空间按团队/业务域划分**（而非按技术栈）：Harbor 项目 = 权限与配额边界，`pay` 团队的 CI 只有 `pay/*` 的 push 权限。
- **一个 Git 仓库对应一个应用**：镜像名与应用名一一对应，K8s Deployment、监控标签、日志字段用同一个名字，排查时全链路可 join。
- **版本与 sha 绑定**：Java 服务通常一次 Maven 出多个 jar 但一个 Dockerfile 一个镜像，tag 里带 git sha 能直接 `git checkout` 复现构建输入。
- **Maven/Gradle 产物名与镜像名对齐**：`artifactId` 与 `<应用>` 保持一致，避免"pom 里叫 order-api、镜像里叫 order-service"的双名混乱。

# 9. 常见错误与排查

## 9.1 `pull access denied` / `denied: requested access to the resource is forbidden`

常见原因：

1. 没登录或登录信息错误。
2. 仓库/项目无权限（Harbor 项目分公有/私有，节点用的机器人账号过期也会被拒）。
3. 镜像名或 namespace 写错。

## 9.2 `manifest unknown`

常见原因：

1. tag 不存在。
2. 推送未成功或被覆盖（immutable tag 开启后覆盖 push 会失败）。
3. 多架构 manifest list 存在但缺当前平台子 manifest（`buildx imagetools inspect <image>` 查看各平台）。

## 9.3 `x509: certificate signed by unknown authority`

常见原因：

1. 私有仓库使用自签证书，宿主机未信任该 CA。
2. TLS 链不完整。

解决方向：让宿主机信任企业 CA（不要用 insecure registry 作为长期方案）。

## 9.4 push 大镜像 504 / 413

检查 Nginx 代理的 `client_max_body_size`、`proxy_read_timeout`，registry 与对象存储之间的带宽；根治手段是层瘦身与镜像构建缓存复用（见 `15_cicd.md`）。

# 10. 面试要点

| 问题 | 答题关键词 |
|------|------------|
| Registry 的作用 | 镜像制品库、分发、版本、权限、审计 |
| docker push 原理 | 按 manifest 逐层协商, digest 去重, manifest 最后原子写入 |
| tag vs digest | tag 可变、digest 不可变，生产建议用 digest 锁定 |
| 私有仓库常见故障 | 权限、tag 不存在、证书不信任、网络超时 |
| 为什么用 Harbor | RBAC、扫描、审计、复制、代理缓存、GC、保留策略、不可变 tag |
| 镜像如何防篡改 | cosign 签名 + digest 引用 + 准入验签 + SBOM |

# 11. 小结

Registry 的本质是一个**内容寻址的 HTTP 制品库**：层以 digest 命名实现跨镜像去重与传输校验，manifest 最后写入保证发布的原子性——由此推导出全部治理规则：`latest` 危险（tag 可变）、GC 必要（孤儿 blob）、签名有价值（digest 可信但来源要证明）。选型阶梯清晰：Docker Hub 适合开源与起步，registry:2 适合内网测试与学习协议，Harbor 提供 RBAC、扫描、复制、代理缓存、保留策略与签名策略的企业闭环。Java 团队的命名规范落在"registry/域/应用:版本-sha"格式上，让镜像名成为贯穿源码、部署、监控、审计的连接键。下一篇进入资源限制（`12_resource_limits.md`），讨论容器拿到配额后 JVM 如何正确感知。
