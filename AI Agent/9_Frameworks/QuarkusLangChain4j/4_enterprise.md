# 1. 原生镜像

## 1.1 构建与运行

### 1.1.1 结论先行

**Quarkus 原生镜像把应用在构建期编译为独立可执行文件，运行期无需 JVM，启动时间与内存占用大幅下降，是 Serverless 与资源受限场景的核心卖点。**

### 1.1.2 命令

```bash
# 构建原生镜像
./mvnw package -Pnative

# 运行
./target/app-runner
```

### 1.1.3 产物对比

| 产物 | 路径 | 运行方式 |
|------|------|----------|
| JVM | `target/quarkus-app/` | `java -jar` |
| 原生 | `target/*-runner` | 直接执行 |

## 1.2 收益与挑战

### 1.2.1 优势表

| 优势 | 说明 |
|------|------|
| 启动 | 毫秒级 |
| 内存 | 显著降低 |
| 镜像 | 更小 |
| 冷启动 | Serverless 友好 |

### 1.2.2 挑战表

| 挑战 | 说明 |
|------|------|
| 反射配置 | 需显式注册 |
| 构建慢 | 分钟级 |
| 兼容性 | 部分库不支持 |

## 1.3 反射与资源

### 1.3.1 注册方式

```java
@RegisterForReflection(targets = {Person.class})
public class ReflectionConfig {
}
```

也可通过 `reflect-config.json` 与 `resources-config.json` 声明。

### 1.3.2 边界与坑

- 未注册的反射调用在原生下抛 `ClassNotFoundException`，JVM 下却正常。
- 资源文件（模板、证书）需显式注册，否则运行期读不到。

### 1.3.3 构建优化

| 手段 | 效果 |
|------|------|
| 构建器镜像缓存 | 缩短重复构建 |
| 减少扩展 | 降低编译面 |
| 关闭无用解析器 | 降低反射面 |

## 1.4 生产建议

- 原生构建放入 CI 独立阶段，缓存构建器镜像加速。
- 维护一份"原生兼容性清单"，记录每个依赖的验证结论。

# 2. Serverless 部署

## 2.1 平台选择

| 平台 | 特点 |
|------|------|
| Knative | K8s 原生，缩容到零 |
| Cloud Run | 托管，冷启动计费 |
| AWS Lambda | 函数式，需适配层 |

## 2.2 Knative 清单

### 2.2.1 yaml

```yaml
# Knative / Cloud Run
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: ai-app
spec:
  template:
    spec:
      containers:
        - image: registry/ai-app-native:latest
          resources:
            limits:
              memory: 256Mi      # 原生镜像内存低
```

### 2.2.2 参数说明

| 参数 | 含义 | 建议 |
|------|------|------|
| memory | 内存上限 | 依压测设定 |
| image | 原生镜像 | 精简基础镜像 |
| concurrency | 单实例并发 | 依阻塞比例 |

> 原生镜像 + Serverless = 极低冷启动延迟。

### 2.2.3 扩缩容配置

```yaml
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/minScale: "1"
        autoscaling.knative.dev/maxScale: "20"
```

## 2.3 边界与坑

- 缩容到零后首个请求承担冷启动，需预热或保留最小实例。
- 模型调用是阻塞 IO，单实例并发不宜过高。

## 2.4 生产建议

- 对延迟敏感服务设置最小实例数，避免冷启动抖动。
- 把模型调用的超时纳入平台超时预算。

# 3. 可观测

## 3.1 指标

### 3.1.1 依赖

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-micrometer-registry-prometheus</artifactId>
</dependency>
```

### 3.1.2 配置

```properties
quarkus.micrometer.export.prometheus.enabled=true
quarkus.micrometer.binder.http-server.enabled=true
```

### 3.1.3 建议指标

| 指标 | 含义 |
|------|------|
| 请求延迟 | 端到端耗时 |
| 模型调用延迟 | LLM 往返 |
| Token 用量 | 成本 |
| 错误率 | 失败比例 |
| 工具调用次数 | 行为分布 |

## 3.2 追踪与日志

### 3.2.1 OpenTelemetry

通过 `quarkus-opentelemetry` 把模型调用与 HTTP 链路串联，便于定位慢点。

### 3.2.2 边界与坑

- 记录提示词会带来隐私与成本问题，需按需采样。
- 高基数标签（如用户 id）会拖垮指标系统。

## 3.3 生产建议

- 对成本指标设预算告警。
- 日志与追踪按合规要求脱敏。

## 3.4 告警与看板

### 3.4.1 告警项

| 告警 | 阈值示例 |
|------|----------|
| 错误率 | > 5% |
| P95 延迟 | > 5s |
| Token 成本 | > 日预算 |
| 熔断打开 | 立即 |

### 3.4.2 边界与坑

- 阈值过灵敏会造成告警疲劳。
- 无成本指标时，事故往往体现在账单而非监控。

## 3.5 生产建议

- 看板区分"系统指标"与"模型指标"。
- 告警分级，成本类告警走低优先级通道。

# 4. 健康检查

## 4.1 探针类型

### 4.1.1 Liveness / Readiness / Startup

| 探针 | 作用 | 失败后果 |
|------|------|----------|
| Liveness | 进程是否存活 | 重启 |
| Readiness | 是否可接流量 | 摘流 |
| Startup | 启动是否完成 | 延迟探活 |

### 4.1.2 代码

```java
@Liveness
@ApplicationScoped
public class LivenessCheck implements HealthCheck {

    @Override
    public HealthCheckResponse call() {
        return HealthCheckResponse.up("alive");
    }
}
```

## 4.2 边界与坑

- 把外部模型可用性放进 Liveness 会导致级联重启。
- Readiness 应反映本服务就绪，而非下游状态。

## 4.3 就绪探针示例

```java
@Readiness
@ApplicationScoped
public class ReadinessCheck implements HealthCheck {

    @Inject
    EmbeddingStore<TextSegment> store;

    @Override
    public HealthCheckResponse call() {
        return HealthCheckResponse.up("ready");
    }
}
```

## 4.4 生产建议

- Liveness 只做进程级检查，下游依赖用 Readiness 或独立指标。
- 为启动较慢的服务配置 Startup 探针。

# 5. 配置与密钥

## 5.1 配置分层

| 来源 | 优先级 | 用途 |
|------|--------|------|
| 默认配置 | 低 | 基线 |
| profile | 中 | 环境差异 |
| 环境变量 | 高 | 部署注入 |
| 系统属性 | 最高 | 调试覆盖 |

## 5.2 密钥注入

```properties
quarkus.langchain4j.openai.api-key=${OPENAI_API_KEY}
```

```bash
# 用 Secret 注入环境变量
kubectl create secret generic ai-secrets \
  --from-literal=openai-api-key=sk-xxx
```

### 5.2.1 边界与坑

- 密钥写入镜像层会被镜像仓库泄露。
- Secret 以环境变量注入时可能出现在进程环境或崩溃转储中。

### 5.2.2 生产建议

- 使用外部密钥管理（Vault / 云 KMS）并轮换。
- 不在日志、异常信息中打印密钥。

## 5.3 生产建议

- 配置与镜像分离，同一镜像多环境部署。
- 对密钥访问做审计。

## 5.4 多环境配置

```properties
%dev.quarkus.langchain4j.openai.chat-model.model-name=gpt-4o-mini
%prod.quarkus.langchain4j.openai.chat-model.model-name=gpt-4o
%prod.quarkus.langchain4j.openai.chat-model.log-requests=false
```

### 5.4.1 边界与坑

- profile 配置易被默认配置意外覆盖，需验证生效值。
- 环境变量命名冲突会导致注入错误。

### 5.4.2 生产建议

- 启动时打印生效配置（脱敏）以便核对。
- 用配置校验在启动期拦截缺失项。

# 6. 可靠性

## 6.1 手段总览

| 手段 | 实现 |
|------|------|
| 超时 | `chat-model.timeout` |
| 重试 | SmallRye Fault Tolerance |
| 熔断 | SmallRye Fault Tolerance |
| 限流 | 网关或应用层 |
| 降级 | 兜底回答 / 缓存 |

## 6.2 容错注解

### 6.2.1 代码

```java
@Retry(maxRetries = 3, delay = 500)
@Timeout(30000)
@CircuitBreaker(requestVolumeThreshold = 10, failureRatio = 0.5)
public String chat(String message) {
    return assistant.chat(message);
}
```

### 6.2.2 参数说明

| 注解 | 关键参数 | 含义 |
|------|----------|------|
| @Retry | maxRetries | 最大重试次数 |
| @Retry | delay | 重试间隔（毫秒） |
| @Timeout | value | 超时（毫秒） |
| @CircuitBreaker | failureRatio | 失败比例阈值 |

### 6.2.3 注解顺序

```
@Retry（最外）
  └── @CircuitBreaker
        └── @Timeout（最内）
              └── 实际调用
```

## 6.3 限流与降级

### 6.3.1 边界与坑

- 对非幂等工具的重试可能产生重复副作用。
- 超时过短会放大重试，形成雪崩。

### 6.3.2 生产建议

- 重试加指数退避与抖动。
- 熔断打开时提供降级回答或排队。

### 6.3.3 降级示例

```java
@Fallback(fallbackMethod = "fallbackChat")
public String chat(String message) {
    return assistant.chat(message);
}

public String fallbackChat(String message) {
    return "服务繁忙，请稍后再试。";
}
```

## 6.4 生产建议

- 为每个 AI 端点定义明确的降级语义。
- 记录降级触发次数，作为容量评估依据。

# 7. 分层架构

## 7.1 分层图

```
Resource（REST）→ Service → AI Service（LangChain4j）
                                │
                                └── 工具 / RAG / 记忆
```

## 7.2 职责边界

### 7.2.1 边界与坑

- 业务校验放在 Service，不要依赖提示词约束。
- AI Service 不直接对外暴露，避免提示词被探测。

### 7.2.2 生产建议

- 每层只依赖下一层，禁止跨层调用。
- 在 Service 层统一做超时、重试、审计。

## 7.3 代码骨架

```java
@Path("/api/chat")
public class ChatResource {

    @Inject
    ChatService service;

    @POST
    public String chat(String message) {
        return service.handle(message);
    }
}

@ApplicationScoped
public class ChatService {

    @Inject
    Assistant assistant;

    @Retry(maxRetries = 3, delay = 500)
    @Timeout(30000)
    public String handle(String message) {
        return assistant.chat(message);
    }
}
```

# 8. 适用与不适用

## 8.1 对比表

| 适用 | 不适用 |
|------|--------|
| Serverless | 需要庞大 Spring 生态 |
| 容器密集 | 团队不熟 Quarkus |
| 冷启动敏感 | 需要成熟第三方库 |
| 资源受限 | 复杂企业集成 |

## 8.2 决策建议

### 8.2.1 评估维度

| 维度 | 权重 |
|------|------|
| 冷启动 | 高 |
| 资源成本 | 高 |
| 生态覆盖 | 中 |
| 团队熟悉度 | 中 |

### 8.2.2 边界与坑

- 若核心依赖不兼容原生，收益会被抵消。
- 团队熟悉度低会拖慢交付，需计入成本。

## 8.3 生产建议

- 用试点服务验证，再决定推广范围。
- 保留 JVM 部署路径作为回退。

# 9. 生产注意事项

## 9.1 风险清单

| 风险 | 建议 |
|------|------|
| 原生镜像兼容性 | 逐个验证依赖 |
| 反射配置缺失 | 显式注册 |
| 无监控 | 集成 Micrometer |
| 无容错 | SmallRye FT |
| 生态较小 | 评估成熟度 |
| 密钥泄露 | 外部密钥管理 |

## 9.2 落地清单

- [ ] 原生构建纳入 CI
- [ ] 反射/资源配置完备
- [ ] 指标与告警接入
- [ ] 超时/重试/熔断配置
- [ ] 密钥走 Secret / KMS
- [ ] 健康探针合理

## 9.3 边界与坑

- 原生构建慢，需与日常开发流程分离。
- 探针与容错配置错误会引发级联故障。

## 9.4 生产建议

- 建立发布前检查清单，逐项确认。
- 对模型调用做成本与延迟双维度监控。

# 10. 小结

## 10.1 要点回顾

- Quarkus 原生镜像启动快、内存低，适合 Serverless。
- 可观测用 Micrometer，容错用 SmallRye Fault Tolerance。
- 密钥用环境变量/Secret。
- 适合冷启动敏感、资源受限场景。
- 原生镜像需验证依赖兼容性。
- 分层清晰、探针合理、降级到位是企业级落地的基础。

## 10.2 落地行动

- 建立发布前检查清单，逐项确认。
- 对模型调用做成本与延迟双维度监控。
