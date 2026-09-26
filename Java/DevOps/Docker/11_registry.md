
# 1. Registry 在交付链路中的位置

Registry 是镜像的制品库：

1. **存储与分发**：push/pull 的目标端。
2. **版本与回滚**：tag/digest 作为版本标识。
3. **权限与审计**：谁能 push、谁能 pull、谁做了什么。

工程结论：

1. 不要把 Registry 只当“下载站”，它是交付链路核心。
2. 生产发布建议明确 tag 策略，并尽量使用 digest 锁定关键环境部署（见 `4_image.md`）。

# 2. Docker Hub（公共仓库）

```bash
docker login

docker tag my-app:1.0 username/my-app:1.0
docker push username/my-app:1.0

docker pull username/my-app:1.0
```

注意：

1. 公共仓库有速率限制与网络波动风险，生产更常用企业私有仓库。
2. 对外发布需要考虑镜像瘦身与敏感信息排除（`.dockerignore`、多阶段构建）。

# 3. 私有 Registry（registry:2）

## 3.1 快速启动（仅用于内网测试/学习）

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

## 3.2 认证（htpasswd）

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

## 3.3 TLS 与 insecure registry 的边界

生产私有仓库应使用 TLS（证书可信链/企业 CA）。

`insecure-registries`（允许非 TLS）只建议用于内网测试环境，并明确风险：

1. 传输可能被中间人攻击。
2. 凭证泄露风险更高。

# 4. Harbor（企业级仓库）

Harbor 常被用作企业标准镜像仓库，典型原因：

1. 项目/租户隔离与 RBAC 权限。
2. 漏洞扫描、镜像复制、审计。
3. 垃圾回收与保留策略（避免镜像无限增长）。

这一章不展开部署细节（偏平台工程），只给使用要点。

```bash
docker login harbor.example.com
docker tag my-app:1.0 harbor.example.com/project/my-app:1.0
docker push harbor.example.com/project/my-app:1.0
```

# 5. Tag 策略与“可回滚”

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

# 6. 常见错误与排查

## 6.1 `pull access denied`

常见原因：

1. 没登录或登录信息错误。
2. 仓库/项目无权限。
3. 镜像名或 namespace 写错。

## 6.2 `manifest unknown`

常见原因：

1. tag 不存在。
2. 推送未成功或被覆盖。

## 6.3 `x509: certificate signed by unknown authority`

常见原因：

1. 私有仓库使用自签证书，宿主机未信任该 CA。
2. TLS 链不完整。

解决方向：让宿主机信任企业 CA（不要用 insecure registry 作为长期方案）。

# 7. 面试要点

| 问题 | 答题关键词 |
|------|------------|
| Registry 的作用 | 镜像制品库、分发、版本、权限、审计 |
| tag vs digest | tag 可变、digest 不可变，生产建议用 digest 锁定 |
| 私有仓库常见故障 | 权限、tag 不存在、证书不信任、网络超时 |
| 为什么用 Harbor | RBAC、扫描、审计、复制、GC |
