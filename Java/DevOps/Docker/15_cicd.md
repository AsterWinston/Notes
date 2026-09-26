
# 1. 目标：可重复构建、可追溯发布、可快速回滚

容器化交付的理想状态：

1. **同一份源码**在 CI 上构建出**同一份镜像结果**（尽量可重复）。
2. 每个镜像都能追溯到源码版本、构建参数、依赖来源。
3. 发布是“拉镜像并运行”，回滚是“切回旧版本镜像”。

关键点：镜像是制品，不是脚本执行结果。

# 2. 标准流水线骨架

典型阶段：

1. Test：单元测试/集成测试
2. Build Image：构建镜像
3. Scan：漏洞扫描/策略检查
4. Push：推送到 Registry
5. Deploy：部署到环境（Compose/K8s/其他）

# 3. 版本策略：tag 规则与不可变制品

推荐 tag 策略（至少两类）：

1. 可读版本：`1.2.3`
2. 可追溯版本：`1.2.3-<git-sha>` 或单独 `sha-<git-sha>`

示例：

```bash
APP=my-app
REG=my-registry.example.com/team
VERSION=1.2.3
GIT_SHA=$(git rev-parse --short HEAD)

docker build -t $REG/$APP:$VERSION -t $REG/$APP:$VERSION-$GIT_SHA .
docker push $REG/$APP:$VERSION
docker push $REG/$APP:$VERSION-$GIT_SHA
```

更严格的发布：部署用 digest 锁定（避免 tag 漂移）。

# 4. BuildKit/buildx：缓存与多架构

## 4.1 构建缓存的价值

缓存能显著降低 CI 构建时间，尤其是依赖下载（Maven/Gradle/npm）。

策略：

1. Dockerfile 层级设计要利于缓存（先 COPY 依赖描述文件，再下载依赖，再 COPY 源码）。
2. CI 侧要启用 BuildKit 并配置缓存导入/导出。

## 4.2 多架构镜像

如果你同时面向 amd64/arm64：

1. 用 `buildx` 构建 multi-arch manifest。
2. 注意基础镜像是否支持目标架构。

# 5. 不要在镜像里塞密钥

CI 常见反模式：

1. `ARG/ENV` 写入密钥并在构建层留下痕迹。
2. `COPY` 把证书/密钥文件塞进镜像。

原则：

1. 密钥在运行时注入（平台能力、挂载、secret 管理）。
2. 构建阶段需要私有依赖凭证时，优先使用 BuildKit secret 机制或受控的凭证注入手段，避免进入最终镜像层。

# 6. 发布与回滚：用镜像版本驱动

发布的核心操作：

1. 拉取目标镜像版本。
2. 更新运行参数（环境变量、配置挂载）。
3. 平滑切换或重启。

回滚的核心操作：

1. 选择上一个稳定镜像版本。
2. 重新部署并验证。

如果你的版本策略清晰，回滚会非常快。

# 7. 常见问题与防线

## 7.1 镜像不可重复构建

常见原因：

1. 构建中下载“浮动版本”（例如 `latest` 或未锁定依赖）。
2. Dockerfile 里引入时间戳、随机数等。

防线：

1. 基础镜像固定到具体版本（甚至 digest）。
2. 依赖版本锁定（Maven/Gradle lockfile 等）。

## 7.2 构建慢

常见原因：

1. Dockerfile 层级导致缓存无法命中。
2. CI 未复用缓存。

防线：

1. 调整 COPY 顺序。
2. 启用 BuildKit 缓存导入/导出。

# 8. 面试要点

| 问题 | 答题关键词 |
|------|------------|
| 为什么镜像是制品 | 可复现、可审计、可回滚 |
| tag 策略怎么设计 | SemVer + Git SHA；关键环境用 digest |
| 怎么加速 CI 构建 | BuildKit 缓存、Dockerfile 分层设计 |
| 密钥怎么处理 | 运行时注入，避免进入镜像层 |
