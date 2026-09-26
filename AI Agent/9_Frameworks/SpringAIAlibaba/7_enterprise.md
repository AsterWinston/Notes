# 1. 分层架构

分层架构的核心价值是**隔离变化**：把频繁变动的模型、提示词、检索策略与稳定的接入、治理逻辑解耦，使每一层可以独立演进、独立测试、独立扩容。企业级 AI 应用建议至少划分为六层，从外到内依次为接入层、编排层、能力层、治理层、模型层与基础设施层。

## 1.1 分层总览

### 1.1.1 六层职责

下表给出各层的核心职责与典型实现，实际项目可按规模裁剪，但**模型层与治理层不应省略**。

| 层 | 职责 | 典型实现 |
|----|------|----------|
| 接入层 | 协议适配、鉴权、限流入口 | Controller / SSE / WebFlux |
| 编排层 | 对话流程、图编排、会话管理 | ChatClient / Graph |
| 能力层 | 检索、工具、记忆等可复用能力 | RAG / Tools / Memory |
| 治理层 | 横切关注点，对调用链无侵入 | Advisor / 限流 / 审计 |
| 模型层 | 屏蔽具体模型差异 | ChatModel 抽象 |
| 基础设施 | 配置、密钥、向量库、状态存储 | Nacos / Redis / DB |

整体结构可用如下 ASCII 图表示：

```
┌──────────────────────────────────────┐
│  接入层   Controller / SSE            │
├──────────────────────────────────────┤
│  编排层   Graph / ChatClient          │
├──────────────────────────────────────┤
│  能力层   RAG / Tools / Memory        │
├──────────────────────────────────────┤
│  治理层   Advisor / 限流 / 审计        │
├──────────────────────────────────────┤
│  模型层   DashScope / 兼容模型         │
├──────────────────────────────────────┤
│  基础设施  配置 / 密钥 / 向量库        │
└──────────────────────────────────────┘
```

### 1.1.2 依赖方向与调用链

依赖方向必须**自上而下单向**：上层可依赖下层，下层不得反向依赖上层。跨层调用通过接口而非具体实现，便于替换与 Mock。

```
请求 ─▶ 接入层 ─▶ 编排层 ─▶ 治理层 ─▶ 能力层 ─▶ 模型层
                                  │
                                  └──▶ 基础设施（配置/存储）
```

调用链上一个常见误区是把 RAG 检索写在 Controller 里，导致接入层直接依赖向量库，破坏了分层。正确做法是把检索下沉到能力层，接入层只负责协议转换。

## 1.2 各层落地要点

### 1.2.1 接入层与编排层

接入层推荐使用 WebFlux 提供 SSE 流式响应，避免阻塞线程池。

```java
@RestController
@RequestMapping("/api/chat")
public class ChatController {

    private final ChatClient chatClient;

    public ChatController(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    @PostMapping(produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> chat(@RequestBody ChatRequest request) {
        return chatClient.prompt()
                .user(request.question())
                .stream()
                .content();
    }
}
```

编排层通过 `ChatClient` 组合默认系统提示、记忆与检索 Advisor。

```java
@Bean
ChatClient chatClient(ChatModel chatModel, VectorStore vectorStore, ChatMemory memory) {
    return ChatClient.builder(chatModel)
            .defaultSystem("你是企业知识助手，回答需引用资料。")
            .defaultAdvisors(
                    MessageChatMemoryAdvisor.builder(memory).build(),
                    QuestionAnswerAdvisor.builder(vectorStore).build())
            .build();
}
```

### 1.2.2 能力层与治理层

能力层以接口暴露能力，内部实现可替换，例如检索能力只暴露 `retrieve` 方法，调用方不感知向量库类型。治理层通过 Advisor 实现审计，对业务代码零侵入。

```java
public class AuditAdvisor implements CallAdvisor {

    private final AuditRepository repository;

    public AuditAdvisor(AuditRepository repository) {
        this.repository = repository;
    }

    @Override
    public ChatClientResponse adviseCall(
            ChatClientRequest request, CallAdvisorChain chain) {
        long start = System.nanoTime();
        ChatClientResponse response = chain.nextCall(request);
        repository.save(new AuditLog(
                request.prompt().getContents(),
                response.chatResponse().getResult().getOutput().getText(),
                System.nanoTime() - start));
        return response;
    }
}
```

### 1.2.3 模型层与基础设施层

模型层统一由 `ChatModel` 抽象，业务代码只依赖该接口。

```java
@Bean
ChatModel dashscopeModel(@Value("${spring.ai.dashscope.api-key}") String apiKey) {
    return DashScopeChatModel.builder()
            .apiKey(apiKey)
            .modelName("qwen-max")
            .build();
}
```

基础设施通过配置中心与外部存储承载，密钥一律来自环境变量而非代码。

```yaml
spring:
  ai:
    dashscope:
      api-key: ${DASHSCOPE_API_KEY}
  data:
    redis:
      host: redis
      port: 6379
```

## 1.3 边界与生产建议

分层的代价是增加抽象与文件数量。小团队在早期可合并编排层与能力层，但**模型层抽象与治理层 Advisor 机制建议从第一天就保留**，否则后期替换模型或补审计会付出高昂重构成本。

# 2. 多模型与降级

生产环境不应绑定单一模型。模型可能因配额、限流、区域故障而不可用，多模型与自动降级是可用性的基本保障。降级的目标是**在主模型不可用时仍能返回可接受的结果**，而非追求结果完全一致。

## 2.1 模型分层策略

### 2.1.1 主备兜底三级模型

按下表配置三级模型，能力与成本递减，可用性递增。

| 层级 | 模型 | 定位 | 触发条件 |
|------|------|------|----------|
| 主 | qwen-max | 高质量 | 默认 |
| 备 | qwen-plus | 均衡 | 主模型失败或超时 |
| 兜底 | qwen-turbo | 低延迟 | 主备均失败 |

### 2.1.2 路由决策

除故障降级外，也可按业务维度主动路由，把简单问题交给便宜模型。

```java
public enum ModelTier { FAST, BALANCED, STRONG }

@Component
public class ModelRouter {

    private final Map<ModelTier, ChatClient> clients;

    public ModelRouter(Map<ModelTier, ChatClient> clients) {
        this.clients = clients;
    }

    public ChatClient route(String question, UserContext user) {
        if (user.vip() || question.length() > 500) {
            return clients.get(ModelTier.STRONG);
        }
        if (question.length() < 30) {
            return clients.get(ModelTier.FAST);
        }
        return clients.get(ModelTier.BALANCED);
    }
}
```

## 2.2 降级实现

### 2.2.1 FallbackAdvisor

在 `ChatClient` 上挂载 `FallbackAdvisor`，主模型抛出异常时自动切换备用模型。

```java
@Bean
ChatClient resilientChatClient(
        @Qualifier("dashscopeModel") ChatModel primary,
        @Qualifier("backupModel") ChatModel backup) {

    return ChatClient.builder(primary)
            .defaultAdvisors(new FallbackAdvisor(backup))
            .build();
}
```

### 2.2.2 降级边界与幂等

降级并非万能，需明确边界，避免把不可重试的错误也降级重放。

| 错误类型 | 是否降级 | 说明 |
|----------|----------|------|
| 超时 | 是 | 换模型重试 |
| 限流 429 | 是 | 换模型或排队 |
| 参数错误 400 | 否 | 换模型同样失败 |
| 内容审核拒绝 | 否 | 业务规则问题 |
| 工具副作用已执行 | 谨慎 | 需幂等键去重 |

对带有副作用的工具调用，降级重放必须携带幂等键，否则可能重复下单、重复发消息。

## 2.3 生产建议

降级链路本身要有监控与告警：记录主备切换次数与原因，若切换率持续升高，说明主模型容量不足而非偶发故障。同时为每个模型设置独立配额，防止降级时把备用模型打爆。

# 3. 多租户

多租户的关键是**数据与资源双重隔离**。仅隔离数据而共享模型配额，仍会因单个租户耗尽配额而影响全局；仅隔离配额而不隔离数据，则存在越权检索风险。

## 3.1 隔离维度

### 3.1.1 隔离模型

下表列出需要隔离的维度及落地方式。

| 维度 | 隔离方式 | 风险 |
|------|----------|------|
| 模型配额 | 租户级限流与预算 | 单租户耗尽全局配额 |
| 知识数据 | 向量库元数据过滤 | 越权检索他人文档 |
| 对话记忆 | 会话 ID 带租户前缀 | 记忆串号 |
| 密钥 | 租户自带密钥或代理 | 密钥泄露 |
| 审计 | 日志带租户标识 | 无法追责 |

## 3.2 多租户实现

### 3.2.1 租户上下文

以不可变记录承载租户信息，通过 `ThreadLocal` 或请求作用域在链路上传递。

```java
public record UserContext(String tenantId, String userId, boolean vip) {}
```

### 3.2.2 元数据过滤

检索时强制注入租户过滤条件，记忆会话 ID 拼接租户前缀。

```java
public String chat(String question, UserContext user) {
    // 租户级模型
    ChatClient client = tenantClientFactory.get(user.tenantId());

    // 租户级 RAG 过滤
    List<Document> docs = vectorStore.similaritySearch(
            SearchRequest.builder()
                    .query(question)
                    .filterExpression("tenantId == '" + user.tenantId() + "'")
                    .build());

    // 租户级记忆
    String conversationId = user.tenantId() + ":" + user.userId();

    return client.prompt()
            .user(buildPrompt(question, docs))
            .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, conversationId))
            .call().content();
}
```

### 3.2.3 配额与审计

在调用入口做租户级配额校验，超限直接拒绝，避免把压力传导到模型层。配额计数可基于 Redis 实现，以支持多实例共享与跨重启累计。

## 3.3 边界与生产建议

过滤表达式若直接拼接用户输入，存在注入风险，应使用参数化或白名单校验租户 ID。另需注意向量库的过滤是**后过滤还是前过滤**：后过滤会先召回再丢弃，可能造成有效结果不足，需相应放大 `topK`。

# 4. 可靠性

可靠性的目标是在依赖不稳定时保持服务可用。对 LLM 应用而言，模型调用是最大的不稳定源，因此超时、重试、熔断、限流、降级五项缺一不可。

## 4.1 可靠性手段总览

| 手段 | 实现 | 作用 |
|------|------|------|
| 超时 | 模型配置 timeout | 防止线程长期占用 |
| 重试 | Resilience4j | 应对瞬时抖动 |
| 熔断 | Resilience4j | 快速失败，避免雪崩 |
| 限流 | 按租户 | 保护下游容量 |
| 降级 | 备用模型 | 保底可用 |

## 4.2 超时与重试

### 4.2.1 模型超时配置

超时应覆盖连接与读取两个阶段，且应小于上游网关超时。

```java
ChatModel model = DashScopeChatModel.builder()
        .apiKey(apiKey)
        .modelName("qwen-plus")
        .timeout(Duration.ofSeconds(30))
        .maxRetries(2)
        .build();
```

### 4.2.2 重试策略

重试必须限制次数并加退避，且只重试可恢复错误。

```java
@Bean
RetryRegistry retryRegistry() {
    RetryConfig config = RetryConfig.custom()
            .maxAttempts(3)
            .waitDuration(Duration.ofMillis(500))
            .retryExceptions(ResourceAccessException.class, TimeoutException.class)
            .build();
    return RetryRegistry.of(config);
}
```

## 4.3 熔断与限流

### 4.3.1 Resilience4j 熔断

熔断器在失败率超阈值后打开，直接快速失败，避免持续拖垮下游。

```java
@Bean
CircuitBreakerRegistry circuitBreakerRegistry() {
    CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)
            .slidingWindowSize(20)
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .build();
    return CircuitBreakerRegistry.of(config);
}
```

### 4.3.2 按租户限流

限流以租户为维度，防止单租户独占配额。可基于 Resilience4j 的 RateLimiter 或 Redis 令牌桶实现，阈值放配置中心便于动态调整。

## 4.4 降级与兜底

当熔断打开且备用模型也不可用时，应返回预置兜底话术而非抛错给用户，同时记录降级事件用于告警。降级链路要定期演练，避免真正故障时才发现备用模型未授权或配额为零。

## 4.5 生产建议

为每个环节设置独立的超时预算并自上而下递减：网关 35s、模型 30s、单次重试 8s。重试会放大流量，务必配合限流与熔断，否则重试风暴会进一步压垮下游。

# 5. 成本治理

LLM 成本与 token 量线性相关，且随上下文增长而放大。成本治理的本质是**在质量与花费之间寻找帕累托最优**，而非一味削减。

## 5.1 成本来源

| 来源 | 说明 | 可控性 |
|------|------|--------|
| 输入 token | 系统提示、检索片段、历史 | 高 |
| 输出 token | 生成内容长度 | 中 |
| 模型单价 | 不同模型差异大 | 高 |
| 调用次数 | 重试、规划循环 | 中 |

## 5.2 治理手段

### 5.2.1 模型分层

按任务复杂度选择模型：分类、抽取用 turbo，生成用 plus，复杂推理才用 max。分层可直接降低 50% 以上成本。

### 5.2.2 缓存

缓存分三级：精确结果缓存、语义缓存、Prompt 前缀缓存。语义缓存把问题向量化后检索历史问答，命中需相似度阈值兜底，阈值过低会命中不相关内容。

### 5.2.3 上下文裁剪

检索片段数量并非越多越好。片段过多既增加成本又稀释关键信息，建议 topK 控制在 3 到 5，并对片段做去重与截断。

## 5.3 预算与监控

按租户设置日/月预算，超预算时降级到便宜模型或拒绝新请求。成本指标需按租户与模型维度上报，并结合预算告警与账单核对。

## 5.4 边界与生产建议

缓存存在失效与一致性风险：知识库更新后必须使相关缓存失效，否则会返回过期答案。语义缓存阈值过低会命中不相关内容，建议在测试集上标定阈值。

# 6. 部署

AI 应用部署的核心原则是**应用无状态、状态外置**，从而支持水平扩展与滚动发布。

## 6.1 部署形态

### 6.1.1 无状态应用

应用进程不保存会话与向量数据，任何实例都可处理任意请求。

```
应用（Spring Boot）
  ├── 无状态，可水平扩展
  ├── 状态外置（Redis/DB）
  ├── 向量库（独立服务）
  └── 配置中心（Nacos）
```

### 6.1.2 状态外置

对话记忆放 Redis，向量数据放独立向量库，二者均需独立扩容与备份策略。

## 6.2 配置中心

### 6.2.1 Nacos 配置

模型名、超时、限额等易变参数放配置中心，支持热更新。

```yaml
spring:
  cloud:
    nacos:
      config:
        server-addr: nacos:8848
  ai:
    dashscope:
      api-key: ${DASHSCOPE_API_KEY}
```

### 6.2.2 密钥管理

密钥不得硬编码或提交到仓库，应通过环境变量、密钥管理服务注入，并支持轮换。配置中心账号同样以环境变量提供，避免明文落盘。

## 6.3 生产建议

发布采用灰度：先放 5% 流量，观察错误率与成本，再逐步扩大。向量库与应用应解耦部署，避免模型侧扩容影响检索侧。

# 7. 典型企业架构

## 7.1 整体架构图

```
              ┌──────────────┐
  客户端 ────▶│  API 网关     │
              └──────┬───────┘
                     ▼
              ┌──────────────┐
              │  AI 应用服务  │
              │  ├ Graph 编排 │
              │  ├ RAG        │
              │  └ Tools      │
              └──┬────┬────┬──┘
                 │    │    │
        ┌────────┘    │    └────────┐
        ▼             ▼             ▼
   通义/百炼      向量库        状态存储
```

## 7.2 关键链路

### 7.2.1 请求链路

请求经网关鉴权与限流后进入应用，编排层组织检索与工具调用，最终调用模型生成。

### 7.2.2 数据链路

知识文档经切分、嵌入后写入向量库；对话记忆写入 Redis；审计与成本写入监控系统。

## 7.3 生产建议

架构图中每个外部依赖都应有超时与降级策略；向量库与状态存储应配置主从或集群，避免单点。

# 8. 落地路线

## 8.1 分阶段目标

### 8.1.1 P0-P2

| 阶段 | 目标 |
|------|------|
| P0 | 打通 Chat + 通义模型 |
| P1 | 加 RAG + 工具 |
| P2 | 加 Graph 编排 |

### 8.1.2 P3-P4

| 阶段 | 目标 |
|------|------|
| P3 | 加可靠性 + 可观测 |
| P4 | 多租户 + 成本治理 |

## 8.2 验收标准

每个阶段需有可量化验收：P0 端到端可用，P1 检索命中率达标，P2 复杂流程可编排，P3 故障可降级，P4 成本可归因。

## 8.3 生产建议

不要跳过 P3 直接做 P4。缺乏可观测时，成本归因与故障定位都无从谈起，治理会变成盲调。

# 9. 生产注意事项

## 9.1 风险清单

| 风险 | 建议 |
|------|------|
| 绑定单一云 | 保持模型层抽象 |
| 密钥硬编码 | 密钥管理 |
| 无降级 | 备用模型 |
| 租户越权 | 元数据过滤 |
| 成本失控 | 分层 + 预算 |
| 版本不兼容 | 锁定版本 |

## 9.2 上线检查清单

### 9.2.1 检查项

上线前逐项确认：超时是否配置、降级是否演练、密钥是否外置、租户过滤是否生效、成本指标是否上报、依赖版本是否锁定、回滚方案是否就绪。定期演练模型不可用、向量库不可用、配置中心不可用三类故障，验证降级与回滚路径。

## 9.3 生产建议

把检查清单固化为发布门禁，任何一项未通过则不允许上线，避免依赖人工记忆。

# 10. 小结

## 10.1 要点回顾

- 分层架构 + 多模型降级 + 多租户隔离。
- 可靠性：超时、重试、熔断、限流、降级。
- 成本治理：模型分层、缓存、预算。
- 应用无状态，状态外置，配置中心统一管理。
- 落地从 Chat 到 Graph 到治理逐步推进。

## 10.2 演进建议

后续可引入更细粒度的路由策略、自动化的成本优化（如动态上下文压缩）以及跨区域的模型容灾，使系统在质量、成本、可用性三个维度持续逼近最优。
