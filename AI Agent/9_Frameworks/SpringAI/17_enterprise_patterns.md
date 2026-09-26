# 1. 从 Demo 到生产的鸿沟

Demo 能跑通，不等于能上线。企业级 AI 应用的难点不在"调用模型"，而在围绕模型构建的一整套工程与治理能力。

## 1.1 差异清单

```
Demo：一个 ChatClient + 一个 prompt → 能跑
生产：多模型、多租户、可靠性、成本、安全、可观测、可运维
```

### 1.1.1 能力差异

| 维度 | Demo | 生产 |
|------|------|------|
| 模型 | 单一模型、写死 | 多模型、可路由、可切换 |
| 用户 | 单人 | 多租户、多角色 |
| 可靠性 | 出错就报错 | 超时/重试/熔断/降级 |
| 成本 | 不关心 | 预算、配额、监控 |
| 安全 | 无 | 注入防护、鉴权、脱敏 |
| 可观测 | 无 | Trace、指标、日志 |
| 质量 | 靠感觉 | 评测集 + 回归 |

### 1.1.2 工程差异

- Demo 允许"硬编码密钥、硬编码模型名"，生产必须外置；
- Demo 允许"无状态、单实例"，生产要考虑多实例与共享状态；
- Demo 允许"改代码上线"，生产要求配置化与灰度。

## 1.2 生产就绪的维度

生产就绪可以拆成六个可验收的维度：

| 维度 | 关键问题 |
|------|----------|
| 可靠性 | 依赖挂了会怎样？ |
| 可观测 | 出问题怎么定位？ |
| 安全 | 会不会越权/泄露？ |
| 成本 | 单次调用多少钱？ |
| 可运维 | 怎么发布与回滚？ |
| 质量 | 怎么证明没退化？ |

## 1.3 与其它章节的关系

本文件汇总 Spring AI 在企业落地中的关键模式，与 `14_ProductionEngineering`、`15_Security` 互为补充：本章讲**模式与选型**，它们讲**具体实现细节**。

# 2. 分层架构

分层的目的是**隔离变化**：模型会换、框架会升级、供应商会变，业务代码不应随之改动。

## 2.1 六层模型

```
┌───────────────────────────────────────────────┐
│  接入层   Controller / SSE / WebSocket         │
├───────────────────────────────────────────────┤
│  应用层   AiFacade / UseCase（编排、事务边界）  │
├───────────────────────────────────────────────┤
│  能力层   ChatClient / RAG / Tools / Memory    │
├───────────────────────────────────────────────┤
│  治理层   Advisor（限流/审计/脱敏/成本）        │
├───────────────────────────────────────────────┤
│  模型层   ChatModel / EmbeddingModel           │
├───────────────────────────────────────────────┤
│  基础设施 配置中心 / 密钥管理 / 向量库 / 缓存   │
└───────────────────────────────────────────────┘
```

## 2.2 依赖方向与原则

原则：**业务代码只依赖应用层接口**，模型/框架细节封装在能力层，便于替换与测试。

### 2.2.1 依赖倒置

```java
// 应用层只依赖接口
public interface Assistant {
    String ask(String question);
}

// 能力层实现接口，内部才出现 ChatClient
@Service
public class SpringAiAssistant implements Assistant {

    private final ChatClient chatClient;

    public SpringAiAssistant(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    @Override
    public String ask(String question) {
        return chatClient.prompt().user(question).call().content();
    }
}
```

### 2.2.2 包结构建议

```
com.example.ai
├── api          // 对外接口与 DTO
├── app          // 用例编排（Facade）
├── capability   // Chat / RAG / Tools / Memory
├── governance   // Advisor 与治理策略
├── model        // 领域模型与枚举
└── infra        // 配置、客户端、存储适配
```

## 2.3 边界与反例

- 反例：Controller 直接注入 `ChatClient`，业务与框架强耦合；
- 反例：把 RAG、工具、记忆塞进一个"上帝 Service"；
- 边界：分层不是越多越好，小项目三层足够，避免过度设计。

# 3. 多模型与路由

单一模型无法同时满足成本、能力与可用性，生产环境需要**多模型路由**。

## 3.1 网关抽象

### 3.1.1 接口设计

```java
public interface LlmGateway {
    String chat(ChatRequest request);
}
```

### 3.1.2 路由实现

```java
@Service
public class RoutingLlmGateway implements LlmGateway {

    private final Map<LlmTier, ChatClient> clients;
    private final TierSelector tierSelector;

    public RoutingLlmGateway(Map<LlmTier, ChatClient> clients,
                             TierSelector tierSelector) {
        this.clients = clients;
        this.tierSelector = tierSelector;
    }

    @Override
    public String chat(ChatRequest request) {
        LlmTier tier = tierSelector.select(request);   // SIMPLE / STANDARD / COMPLEX
        return clients.get(tier).prompt()
                .user(request.text())
                .call()
                .content();
    }
}
```

## 3.2 模型分层策略

| 分层 | 模型 | 用途 |
|------|------|------|
| SIMPLE | 小模型 | 分类、抽取、改写 |
| STANDARD | 中模型 | 常规问答 |
| COMPLEX | 大模型 | 复杂推理、代码 |

路由依据可以是：问题长度、任务类型、用户等级、成本预算、模型健康度。

> 模型分层可显著降本。详见 `4_LLMEngineering/9_model_routing.md`。

## 3.3 边界与坑

- 路由规则要**可配置、可观测**，否则难以调优；
- 不同模型能力差异大，切换前必须评测；
- 降级路由要避免"全量打到大模型"导致成本失控；
- 边界：路由本身增加延迟与复杂度，简单场景不必引入。

# 4. 配置与密钥治理

生产环境的第一条铁律：**密钥与模型名绝不硬编码**。

## 4.1 配置外置

### 4.1.1 配置中心

```yaml
spring:
  config:
    import: optional:configserver:http://config-server:8888
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}      # 来自配置中心/密钥管理
      base-url: ${OPENAI_BASE_URL:https://api.openai.com}
```

### 4.1.2 环境隔离

| 环境 | 密钥 | 配额 | 用途 |
|------|------|------|------|
| dev | 独立 | 低 | 开发调试 |
| test | 独立 | 中 | 集成测试 |
| prod | 独立 | 高 | 生产 |

## 4.2 密钥管理

| 要求 | 做法 |
|------|------|
| 密钥不落盘 | 密钥管理服务（Vault/KMS） |
| 分环境隔离 | dev/test/prod 独立密钥与配额 |
| 可轮换 | 支持热更新，无需重启 |
| 可审计 | 记录密钥使用与调用方 |

## 4.3 轮换与审计

- 密钥设置有效期，到期自动轮换；
- 轮换过程不中断服务（双密钥过渡）；
- 记录"哪个服务、何时、用了哪把密钥"。

> 边界：配置中心本身是单点，需高可用；本地缓存兜底，防止配置中心不可用时无法启动。

# 5. 可靠性

LLM 依赖外部服务，天然不可靠。可靠性是上线的门槛，不是可选项。

## 5.1 可靠性五件套

| 手段 | Spring 实现 | 说明 |
|------|-------------|------|
| 超时 | WebClient/RestClient timeout | 模型与工具都必须设 |
| 重试 | Spring Retry / Resilience4j Retry | 只重试幂等调用 |
| 熔断 | Resilience4j CircuitBreaker | 模型不可用时快速失败 |
| 限流 | Resilience4j RateLimiter / Bucket4j | 保护模型配额与成本 |
| 降级 | 备用模型 / 缓存 / 静态兜底 | 保证可用性 |
| 隔离 | 舱壁（线程池/信号量） | 防止一个工具拖垮全局 |

### 5.1.1 超时

- 连接超时与读取超时要分别设置；
- 流式请求需要"首字节超时"与"空闲超时"；
- 超时值应小于上游网关超时，避免级联。

### 5.1.2 重试与幂等

- 只对幂等调用重试（查询、读操作）；
- 写操作必须先有幂等键再重试；
- 指数退避 + 抖动，避免惊群。

### 5.1.3 熔断与限流

- 熔断保护自己与下游，半开探测恢复；
- 限流按租户/用户维度，保护配额与成本。

## 5.2 Spring 实现

```java
@Bean
public ChatClient resilientChatClient(ChatModel model) {
    return ChatClient.builder(model)
            .defaultAdvisors(
                    new ResilientAdvisor(circuitBreaker, rateLimiter, retry))
            .build();
}
```

> 说明：`ResilientAdvisor` 为示意组合 Advisor，实际可由多个 `CallAdvisor` 叠加实现。

## 5.3 降级与舱壁

- 降级顺序：缓存 → 备用模型 → 静态兜底；
- 舱壁：为不同工具分配独立线程池，隔离故障；
- 边界：降级结果要标注"降级中"，避免用户误以为正常。

> 详见 `14_ProductionEngineering/Reliability`。

# 6. 状态与会话持久化

有状态的东西不能只放内存，否则扩容与重启都会丢数据。

## 6.1 状态分类

| 状态 | 生命周期 | 示例 |
|------|----------|------|
| 会话记忆 | 会话级 | 对话历史 |
| 任务状态 | 任务级 | 长任务检查点 |
| 缓存 | 短期 | 语义缓存 |
| 配置 | 长期 | Prompt 版本 |

## 6.2 方案选型

| 场景 | 方案 |
|------|------|
| 对话记忆 | Redis / JDBC（见 `8_chat_memory.md`） |
| 长任务状态 | 状态机 + 数据库检查点 |
| 多实例部署 | 无状态服务 + 外部状态存储 |

## 6.3 多实例部署

```
        ┌──────────┐
请求 ──▶│ 实例 A   │──┐
        └──────────┘  │   ┌──────────────┐
        ┌──────────┐  ├──▶│ Redis / DB   │
请求 ──▶│ 实例 B   │──┘   └──────────────┘
        └──────────┘
```

服务本身无状态，状态外置到 Redis/数据库，才能水平扩展。

> 详见 `14_ProductionEngineering/Runtime`。

# 7. 多租户

SaaS 场景下，多租户隔离是安全与合规的底线。

## 7.1 隔离维度

隔离维度：**数据（向量/记忆）、模型与配额、密钥、权限、审计**。

## 7.2 实现示例

### 7.2.1 租户上下文

```java
public record TenantContext(
        String tenantId, String userId, String sessionId) {}
```

### 7.2.2 租户级记忆与 RAG

```java
public String chat(TenantContext ctx, String question) {
    // 1. 租户级模型/配额
    ChatClient client = tenantClientFactory.get(ctx.tenantId());

    // 2. 租户级记忆
    String conversationId = ctx.tenantId() + ":" + ctx.userId() + ":" + ctx.sessionId();

    // 3. 租户级 RAG 过滤
    // 见 10_rag.md 的元数据过滤

    return client.prompt()
            .user(question)
            .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, conversationId))
            .call()
            .content();
}
```

## 7.3 边界与坑

- `tenantId` 必须来自可信来源（网关/令牌），不能由前端传入；
- 向量库过滤条件**必须强制注入**，不能依赖调用方自觉；
- 缓存键要包含租户维度，防止跨租户命中；
- 边界：物理隔离（独立库/实例）成本高，按合规要求分级选择。

# 8. 成本治理

LLM 成本与用量正相关，不加治理会随用户增长线性上升。

## 8.1 成本构成

| 组成 | 说明 |
|------|------|
| 输入 Token | 提示、上下文、检索片段 |
| 输出 Token | 模型生成内容 |
| 工具调用 | 外部 API 费用 |
| 向量存储 | 存储与检索费用 |

## 8.2 降本手段

| 手段 | 说明 |
|------|------|
| 模型分层 | 简单任务用小模型 |
| 缓存 | Prompt 缓存、语义缓存、结果缓存 |
| Token 预算 | 限制 `maxTokens`、裁剪上下文 |
| 批处理 | 合并请求、离线任务错峰 |
| 监控告警 | 按租户/用户统计成本 |
| 配额 | 每用户/租户日限额 |

## 8.3 预算与告警

- 设置日/月预算，超阈值告警甚至熔断；
- 成本指标按租户、用户、功能维度下钻；
- 边界：缓存会带来一致性风险，敏感场景慎用结果缓存。

> 详见 `14_ProductionEngineering/Performance`。

# 9. 安全治理

AI 应用引入了新的攻击面：提示注入、越权工具调用、数据泄露。

## 9.1 分层防护

| 层 | 措施 |
|----|------|
| 输入 | 注入检测、长度限制、敏感词 |
| 提示 | 用户内容隔离，规则不可被覆盖 |
| 工具 | 工具级鉴权、最小权限、MCP 隔离 |
| 输出 | 内容审核、脱敏、引用校验 |
| 数据 | 租户隔离、加密、访问审计 |
| 运维 | 密钥轮换、依赖审计、出网管控 |

## 9.2 工具与 MCP 安全

- 每个工具单独鉴权，不依赖"整体登录"；
- 最小权限：只授予完成任务必需的权限；
- MCP（Model Context Protocol，模型上下文协议）服务器要隔离与限权。

## 9.3 合规审计

- 记录"谁在何时调用了什么工具、访问了什么数据"；
- 审计日志独立存储、不可篡改；
- 保留期限满足合规要求。

> 详见 `15_Security`。

# 10. 部署与交付

交付的核心是"可重复、可灰度、可回滚"。

## 10.1 交付流水线

```
代码仓库 ──CI（单测+评测）──▶ 镜像 ──CD──▶ K8s（蓝绿/金丝雀）
                                              │
                                    配置/Prompt 版本管理
```

## 10.2 Prompt 与配置版本管理

要点：

- **Prompt 与代码一起版本化**，可独立灰度；
- 模型切换要能通过配置完成，无需改代码；
- 发版前跑评测回归，指标不退化才放行。

## 10.3 灰度与回滚

- 灰度：按用户/租户/流量比例放量，观察指标；
- 回滚：保留镜像 + Prompt + 配置三层回滚能力；
- 边界：模型是外部依赖，回滚应用不等于回滚模型版本，需记录模型版本。

# 11. 典型企业架构示例

## 11.1 整体拓扑

```
                    ┌──────────────┐
   客户端 ─────────▶│  API 网关     │  鉴权 / 限流 / 路由
                    └──────┬───────┘
                           ▼
                 ┌───────────────────┐
                 │  AI 应用服务       │
                 │  ├ ChatFacade      │
                 │  ├ RagService      │
                 │  ├ ToolRegistry    │
                 │  └ MemoryService   │
                 └─────┬───────┬──────┘
                       │       │
        ┌──────────────┘       └───────────────┐
        ▼                                      ▼
 ┌────────────┐                        ┌──────────────┐
 │ 模型网关    │  多模型/配额/缓存       │ 向量库 + 记忆 │
 └─────┬──────┘                        └──────────────┘
       ▼
  OpenAI / 通义 / 本地 Ollama / vLLM
```

## 11.2 组件职责

| 组件 | 职责 |
|------|------|
| API 网关 | 鉴权、限流、路由、TraceId 生成 |
| AI 应用服务 | 编排用例，调用能力层 |
| 模型网关 | 多模型路由、配额、缓存、降级 |
| 向量库 + 记忆 | RAG 检索与会话状态 |

## 11.3 请求流转

```
请求 → 网关鉴权 → ChatFacade → 记忆读取 → RAG 检索
     → 模型网关选模型 → 模型调用 → 工具调用（如需）
     → 输出审核 → 记忆写入 → 响应
```

# 12. 落地路线建议

## 12.1 阶段划分

| 阶段 | 目标 |
|------|------|
| P0 原型 | 打通 Chat + Prompt，验证场景价值 |
| P1 可用 | 加 RAG/工具/记忆，形成闭环 |
| P2 可靠 | 加超时/重试/熔断/降级/可观测 |
| P3 治理 | 多租户、权限、成本、审计 |
| P4 规模化 | 多模型路由、评测回归、灰度发布 |

## 12.2 验收标准

- P0：能回答核心场景问题；
- P1：RAG 命中且工具可用；
- P2：依赖故障时可降级、可定位；
- P3：租户隔离与成本可观测；
- P4：评测回归通过、可灰度可回滚。

## 12.3 常见误区

- 跳过 P2 直接做 P4，导致规模越大越脆弱；
- 只做功能不做评测，质量随版本漂移；
- 过早追求多模型，增加复杂度却无收益。

# 13. 小结

## 13.1 核心要点

- 企业级 = 分层架构 + 可靠性 + 安全 + 成本 + 可观测。
- 业务与模型解耦，模型切换靠配置而非改代码。
- 多租户隔离要覆盖数据、模型、密钥、审计。

## 13.2 落地建议

- 可靠性五件套（超时/重试/幂等/限流/熔断）是上线门槛。
- 按 P0→P4 分阶段推进，每阶段有明确验收标准。
- 交付要可重复、可灰度、可回滚，Prompt 与代码一起版本化。
