# 1. 部署形态总览

Agent 从开发到生产需要选择合适的部署形态，不同形态在运维成本、弹性与可控性上差异明显。

## 1.1 结论先行

- ADK 支持本地运行、容器部署、Cloud Run（Serverless）与 Vertex AI Agent Engine（托管运行时）四种主要形态。
- 选型的核心变量是：流量特征、会话持久化要求、合规约束与团队运维能力。
- 无论哪种形态，都必须解决会话持久化、密钥管理与可观测三件事。

## 1.2 机制拆解

### 1.2.1 部署方式对比

| 方式 | 说明 | 弹性 | 运维成本 | 适用 |
|------|------|------|----------|------|
| 本地运行 | 开发调试 | 无 | 极低 | 开发 |
| 容器部署 | Docker/K8s | 高 | 中高 | 私有化 |
| Cloud Run | Serverless | 自动 | 低 | 突发流量 |
| Vertex AI Agent Engine | 托管运行时 | 自动 | 极低 | 快速上线 |

### 1.2.2 选型决策

```
需要私有化/合规? --是--> 容器(K8s)
        | 否
        v
流量突发且团队小? --是--> Cloud Run
        | 否
        v
追求最低运维? --是--> Vertex AI Agent Engine
```

### 1.2.3 形态与团队规模

| 团队规模 | 推荐形态 |
|----------|----------|
| 1~3 人 | Cloud Run / 托管 |
| 有平台团队 | K8s |
| 强合规 | 私有 K8s |

### 1.2.4 成本维度

| 形态 | 固定成本 | 变动成本 |
|------|----------|----------|
| 本地 | 无 | 无 |
| K8s | 高（常驻） | 低 |
| Cloud Run | 低 | 按用量 |
| 托管 | 低 | 按调用 |

## 1.3 代码示例

```yaml
spring:
  profiles:
    active: ${APP_PROFILE:local}   # local | staging | prod
```

## 1.4 边界与坑

- Serverless 有冷启动，首 token 延迟可能明显升高。
- 多实例部署下，内存会话无法共享，必须外部化。
- 托管运行时可能限制自定义依赖与网络访问。

## 1.5 生产建议

- 开发用本地，预发用容器，生产按合规与流量选托管或 K8s。
- 把"环境差异"收敛到配置，代码保持一套。
- 上线前明确会话存储、密钥来源与监控接入点。

# 2. 本地运行与调试

本地运行是开发与调试的起点，目标是快速迭代指令、工具与流程。

## 2.1 结论先行

- 本地用内存会话服务即可，追求启动快、改完即跑。
- 调试的关键是能看到完整事件流（模型输出、工具调用、委托）。
- 本地配置应可通过环境变量覆盖，避免与生产分叉。

## 2.2 机制拆解

### 2.2.1 本地组件

| 组件 | 本地实现 |
|------|----------|
| SessionService | InMemorySessionService |
| ArtifactService | InMemoryArtifactService |
| 模型凭据 | 环境变量 / ADC |
| 日志 | 控制台 |

### 2.2.2 调试流程

```
改指令/工具 -> 重启 Runner -> 发测试输入 -> 看事件流 -> 定位问题
```

### 2.2.3 常用调试手段

| 手段 | 说明 |
|------|------|
| 事件打印 | 观察模型/工具/委托 |
| 固定用例 | 回归指令变更 |
| 断点 | 调试工具实现 |
| Mock 模型 | 离线跑流程 |

## 2.3 代码示例

```java
InMemorySessionService sessionService = new InMemorySessionService();

Runner runner = Runner.builder()
        .appName("local-dev")
        .agent(agent)
        .sessionService(sessionService)
        .build();

runner.runStreaming(session, "帮我查一下订单 A123")
        .doOnNext(e -> System.out.printf("[%s] %s%n", e.type(), e.content()))
        .blockingSubscribe();
```

## 2.4 边界与坑

- 内存会话在重启后清空，长会话调试需重复构造上下文。
- 本地密钥若写进代码，容易误提交。
- 本地并发低，无法暴露限流与竞争问题。

## 2.5 生产建议

- 用 `.env` 或系统环境变量注入密钥，仓库加 `.gitignore`。
- 准备一组固定测试用例，本地改完即回归。
- 本地日志级别可调，生产默认收敛。

# 3. 会话持久化

会话持久化决定 Agent 能否在多实例、重启后保持上下文，是生产化的分水岭。

## 3.1 结论先行

- 开发用 `InMemorySessionService`，生产必须用 `DatabaseSessionService` 或托管会话服务。
- Runner 通过 `sessionService` 加载与保存会话。
- 多实例部署时，会话必须存放在共享外部存储。

## 3.2 机制拆解

### 3.2.1 会话服务类型

| 会话服务 | 适用 | 特点 |
|----------|------|------|
| InMemory | 开发 | 快、重启即失 |
| Database | 生产 | 持久、可扩展 |
| Vertex AI | 托管 | 免运维 |

### 3.2.2 读写流程

```
请求 -> Runner 按 (app,user,session) 取会话 -> 追加事件 -> 保存 -> 返回
```

### 3.2.3 生产会话存储要点

| 要点 | 说明 |
|------|------|
| 主键 | app + user + session |
| 事件保序 | seq 递增 |
| TTL | 自动过期清理 |
| 加密 | 敏感字段落库加密 |

### 3.2.4 会话迁移

| 场景 | 做法 |
|------|------|
| 内存转数据库 | 一次性导出导入 |
| 换库 | 双写过渡 |
| 多地域 | 就近存储 + 同步 |

- 迁移期间需保证 sessionId 不变，避免上下文断裂。

### 3.2.5 会话安全

| 项 | 要求 |
|----|------|
| 越权读取 | 按 userId 强隔离 |
| 敏感字段 | 加密存储 |
| 传输 | TLS |
| 审计 | 记录访问 |

## 3.3 代码示例

```java
// 内存（开发）
InMemorySessionService sessionService = new InMemorySessionService();

// 数据库（生产）
DatabaseSessionService sessionService = DatabaseSessionService.builder()
        .dataSource(dataSource)
        .build();

Runner runner = Runner.builder()
        .agent(agent)
        .sessionService(sessionService)
        .build();
```

| 会话服务 | 适用 |
|----------|------|
| InMemory | 开发 |
| Database | 生产 |
| Vertex AI | 托管 |

## 3.4 边界与坑

- 会话表若不设 TTL 会无限增长，拖慢查询并抬高成本。
- 并发写同一会话需要乐观锁或串行化，否则事件错序。
- 大 State 落库会显著增加读写延迟。

## 3.5 生产建议

- 选择支持高并发的数据库并加索引（app+user+session）。
- 定期归档历史事件，冷热分离。
- 对会话数据做加密与访问审计。

# 4. 容器与 Serverless 部署

容器化是私有化与 K8s 部署的基础，Serverless 则进一步免去运维。

## 4.1 结论先行

- 用多阶段构建产出精简 JRE 镜像，非 root 运行。
- 通过环境变量注入配置与密钥，不打包进镜像。
- Serverless 需注意冷启动与实例内无状态。

## 4.2 机制拆解

### 4.2.1 镜像构建要点

| 要点 | 说明 |
|------|------|
| 基础镜像 | `eclipse-temurin:21-jre` |
| 多阶段 | 构建与运行分离 |
| 非 root | 最小权限 |
| 健康检查 | readiness/liveness |

### 4.2.2 部署拓扑

```
        +----------------+
Client -> LoadBalancer  |
        +-------+--------+
                |
     +----------+----------+
     |          |          |
  [pod]      [pod]      [pod]  <- 无状态，会话在数据库
```

### 4.2.3 弹性与扩缩容

| 指标 | 用途 |
|------|------|
| CPU/内存 | 基础扩缩容 |
| 并发请求数 | 更贴近 Agent 负载 |
| 队列长度 | 提前扩容 |
| 首 token 延迟 | 体验告警 |

- Agent 负载受模型延迟主导，按并发而非纯 CPU 扩缩更准确。

### 4.2.4 观测接入

| 组件 | 接入点 |
|------|--------|
| 日志 | stdout 结构化 |
| 指标 | /actuator/prometheus |
| Trace | OpenTelemetry SDK |

## 4.3 代码示例

```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY target/adk-app.jar app.jar
ENV GOOGLE_API_KEY=${GOOGLE_API_KEY}
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: adk-app
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: adk-app
          image: registry/adk-app:1.0.0
          envFrom:
            - secretRef:
                name: adk-secrets
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
```

## 4.4 边界与坑

- `ENV GOOGLE_API_KEY=${GOOGLE_API_KEY}` 只是声明，构建时不会自动注入，运行时仍需提供。
- Serverless 实例随时回收，内存态一律不可依赖。
- 冷启动叠加模型首 token，端到端延迟可能超标。

## 4.5 生产建议

- 镜像固定 digest，禁用 `latest`。
- 设置资源 requests/limits 与水平扩缩容（HPA）。
- 预热或保留最小实例缓解冷启动。

# 5. 配置与密钥管理

配置与密钥外置是安全底线，硬编码密钥是最常见的严重问题。

## 5.1 结论先行

- 配置分层：默认值 → 环境变量 → 密钥管理服务。
- 密钥绝不入库、不入镜像、不进日志。
- 通过统一配置对象注入，避免散落读取。

## 5.2 机制拆解

### 5.2.1 配置分层

| 层级 | 来源 | 优先级 |
|------|------|--------|
| 默认 | 代码内置 | 低 |
| 环境 | 环境变量 | 中 |
| 密钥 | Secret Manager/KMS | 高 |

### 5.2.2 密钥生命周期

```
生成 -> 存储(密钥管理) -> 注入(环境/挂载) -> 使用 -> 轮换 -> 吊销
```

### 5.2.3 多环境配置

| 环境 | 模型 | 会话 | 日志级别 |
|------|------|------|----------|
| dev | flash | 内存 | DEBUG |
| staging | flash | 数据库 | INFO |
| prod | pro/flash | 数据库 | INFO |

- 用 profile 切换，禁止在代码里写死环境判断。

## 5.3 代码示例

```yaml
google:
  adk:
    model: gemini-2.0-flash
    api-key: ${GOOGLE_API_KEY}
    project: ${GOOGLE_CLOUD_PROJECT}
    location: ${GOOGLE_CLOUD_LOCATION:us-central1}
```

```java
@ConfigurationProperties(prefix = "google.adk")
public record AdkProperties(
        String model,
        String apiKey,
        String project,
        String location) {
}
```

## 5.4 边界与坑

- 把密钥写进 `application.yml` 并提交仓库是高频事故。
- 日志中打印完整配置会泄露密钥，需脱敏。
- 密钥轮换若未做双密钥并行，切换期间会中断。

## 5.5 生产建议

- 使用云密钥管理服务，运行时按需拉取。
- 配置对象只暴露必要字段，打印时掩码。
- 建立密钥轮换流程与到期告警。

# 6. 可观测与可靠性

可观测让问题"看得见"，可靠性让系统"扛得住"。

## 6.1 结论先行

- 可观测三支柱：日志、指标、链路追踪（Trace）。
- Agent 场景必须额外记录：委托、工具调用、token 用量、耗时。
- 可靠性手段：超时、重试、降级、限流。

## 6.2 机制拆解

### 6.2.1 可观测三支柱

| 支柱 | 内容 | 工具 |
|------|------|------|
| 日志 | 事件、错误 | 结构化日志 |
| 指标 | QPS、延迟、token | Prometheus |
| 链路 | Trace 全链路 | OpenTelemetry |

### 6.2.2 可靠性手段

| 手段 | 说明 |
|------|------|
| 超时 | 模型与工具 |
| 重试 | 幂等调用 |
| 降级 | 备用模型 |
| 限流 | 按用户 |
| 熔断 | 依赖故障快速失败 |

### 6.2.3 SLO 与告警

| 指标 | 目标示例 |
|------|----------|
| 可用性 | 99.9% |
| 首 token 延迟 P95 | < 2s |
| 工具失败率 | < 1% |
| 单次成本 | 设上限 |

- 告警要可行动：明确阈值、责任人与处置手册。

## 6.3 代码示例

```java
// 记录 Agent 执行事件
runner.runStreaming(session, input)
        .doOnNext(event -> log.info("event={} content={}",
                event.type(), event.content()))
        .subscribe();
```

```java
// 带超时与降级的调用
runWithTimeout(session, input, Duration.ofSeconds(30))
        .onErrorResume(err -> runWithModel(session, input, fallbackModel));
```

| 记录项 | 说明 |
|--------|------|
| Agent 委托 | 哪个 Agent 处理 |
| 工具调用 | 工具名、参数、结果 |
| Token | 用量 |
| 耗时 | 各步耗时 |

## 6.4 边界与坑

- 只记日志不建指标，无法做容量规划与告警。
- Trace 若不含委托与工具节点，排障仍需人工拼凑。
- 无限重试会放大故障，必须配合退避与熔断。

## 6.5 生产建议

- 用 OpenTelemetry 统一埋点，Trace 贯穿 Agent 树。
- 为延迟、错误率、token 用量建看板与告警。
- 降级路径要定期演练，确保真的可用。

# 7. 生产注意事项

上线前需系统排查风险，尤其是会话、密钥、观测与版本。

## 7.1 结论先行

- 会话未持久化、密钥硬编码、无 Trace、无降级是四条高危项。
- Java SDK 相对较新，需评估成熟度并锁定版本。
- 成本与限流必须前置设计，而非事后补救。

## 7.2 风险与对策

| 风险 | 影响 | 建议 |
|------|------|------|
| 会话未持久化 | 重启丢上下文 | 用 Database |
| 密钥硬编码 | 泄露 | 密钥管理 |
| 无 Trace | 难排障 | 记录事件 |
| 无降级 | 模型故障全挂 | 备用模型 |
| Java SDK 较新 | 稳定性未知 | 评估成熟度 |
| 版本漂移 | 行为突变 | 锁定版本 |

## 7.3 成本与限流

| 手段 | 说明 |
|------|------|
| 按用户限流 | 防滥用 |
| 按租户配额 | 成本分摊 |
| 输出长度上限 | 控成本 |
| 缓存 | 复用结果 |

## 7.4 升级与版本

- 锁定 ADK 与模型版本，升级走灰度。
- 记录每次发布变更点，保留回滚包。
- 关注框架变更日志与破坏性变更。

## 7.5 安全与合规

| 项 | 要求 |
|----|------|
| 数据出境 | 遵守地域要求 |
| 日志脱敏 | 禁止记录敏感原文 |
| 访问控制 | 最小权限 |
| 审计留存 | 满足期限要求 |

- Agent 处理用户数据时，需明确数据用途与保留策略。

## 7.6 发布清单

| 项 | 检查 |
|----|------|
| 配置 | 环境变量齐全 |
| 密钥 | 来自密钥管理 |
| 会话 | 持久化已启用 |
| 观测 | 日志/指标/Trace 就绪 |
| 降级 | 备用模型可用 |
| 回滚 | 上一版可快速恢复 |

# 8. 小结

## 8.1 核心要点回顾

- ADK 支持本地、容器、Serverless（Cloud Run）与托管（Vertex AI Agent Engine）部署。
- 会话需持久化：开发用 InMemory，生产用 Database 或托管。
- 密钥外置，配置分层管理，禁止硬编码。
- 事件用于 Trace 与调试，需配套指标与告警。
- 可靠性依赖超时、重试、降级、限流与熔断。
- Java SDK 较新，生产需评估成熟度并锁定版本。
- 选型取决于合规、流量与团队运维能力。
- 无论何种形态，会话、密钥、观测三件事不可省。
- Serverless 注意冷启动，多实例注意无状态。
- 成本与限流应前置设计，而非事后补救。

## 8.2 生产检查清单

- [ ] 会话服务已切换为持久化实现并设置 TTL。
- [ ] 密钥来自密钥管理，未入库/入镜像/入日志。
- [ ] 日志、指标、Trace 三支柱齐备。
- [ ] 模型与工具均有超时，关键调用可重试。
- [ ] 具备备用模型与降级路径。
- [ ] 按用户/租户限流与配额已生效。
- [ ] 发布与回滚流程经过演练。
- [ ] 部署形态与团队运维能力匹配。
- [ ] Serverless 冷启动已评估或缓解。
- [ ] 数据出境与合规要求已确认。
