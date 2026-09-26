# 1. 分层架构

分层（layered architecture）是 AI 应用最基础的架构模式：把系统按职责切成若干层，每层只依赖相邻下层，从而隔离变化。

## 1.1 六层结构

```
┌───────────────────────────────────────────────┐
│  接入层   REST / SSE / WebSocket / MQ          │  协议适配、鉴权、限流
├───────────────────────────────────────────────┤
│  应用层   Facade / UseCase                     │  编排、事务边界、幂等
├───────────────────────────────────────────────┤
│  能力层   Chat / RAG / Tools / Memory / Agent   │  业务能力
├───────────────────────────────────────────────┤
│  治理层   Advisor / Interceptor                │  限流、审计、脱敏、成本
├───────────────────────────────────────────────┤
│  模型层   ChatModel / EmbeddingModel            │  模型抽象
├───────────────────────────────────────────────┤
│  基础设施  配置 / 密钥 / 状态 / 向量库 / 缓存    │
└───────────────────────────────────────────────┘
```

### 1.1.1 各层职责

| 层 | 职责 | 典型组件 |
|----|------|----------|
| 接入层 | 协议适配、鉴权、限流 | Controller、Gateway |
| 应用层 | 用例编排、事务边界、幂等 | Facade、UseCase |
| 能力层 | 业务能力实现 | ChatService、RagService |
| 治理层 | 横切关注点 | Advisor、Interceptor |
| 模型层 | 模型调用抽象 | ChatModel、EmbeddingModel |
| 基础设施 | 外部资源 | 配置、Redis、向量库 |

### 1.1.2 依赖规则

依赖只能自上而下，且尽量**只依赖相邻层**。上层通过接口调用下层，下层不感知上层。

```
接入层 ──▶ 应用层 ──▶ 能力层 ──▶ 模型层
              │          │
              └──────────┴──▶ 治理层（切面横切）
```

## 1.2 分层原则

- **业务代码只依赖能力层接口**，不直接依赖模型；
- 治理能力用切面实现，不侵入业务；
- 模型可替换（配置化）。

## 1.3 反例：越层调用

```java
// 反例：Controller 直接调用模型，绕过能力层与治理层
@PostMapping("/chat")
public String chat(@RequestBody String q) {
    return chatModel.call(q);   // 无鉴权、无限流、无审计
}
```

```java
// 正例：Controller 只调应用层，治理在切面中完成
@PostMapping("/chat")
public ChatResponse chat(@RequestBody ChatRequest req) {
    return chatFacade.chat(req);   // 内部经能力层与治理层
}
```

# 2. 模型网关模式

模型网关（model gateway）是所有模型调用的统一入口，把多模型差异、鉴权、配额、降级等横切逻辑收拢到一处。

## 2.1 网关职责

```
应用服务 ──▶ 模型网关 ──▶ 多模型
                │
                ├── 统一鉴权
                ├── 多模型路由
                ├── 配额与限流
                ├── 缓存
                ├── 成本统计
                └── 故障降级
```

| 职责 | 说明 |
|------|------|
| 统一鉴权 | 密钥集中管理，业务不接触密钥 |
| 多模型路由 | 按能力/成本/可用性选择模型 |
| 配额与限流 | 按租户/模型维度限流 |
| 缓存 | 相同请求命中缓存，降低成本 |
| 成本统计 | 按调用方累计 Token 与费用 |
| 故障降级 | 主模型失败切换到备用模型 |

## 2.2 完整实现

### 2.2.1 接口与实现

```java
public interface LlmGateway {
    ChatResponse chat(ChatRequest request);
}

@Service
public class DefaultLlmGateway implements LlmGateway {

    private final Map<String, ChatModel> models;
    private final RateLimiter rateLimiter;
    private final CircuitBreaker circuitBreaker;
    private final CostTracker costTracker;

    public DefaultLlmGateway(Map<String, ChatModel> models,
                             RateLimiter rateLimiter,
                             CircuitBreaker circuitBreaker,
                             CostTracker costTracker) {
        this.models = models;
        this.rateLimiter = rateLimiter;
        this.circuitBreaker = circuitBreaker;
        this.costTracker = costTracker;
    }

    @Override
    public ChatResponse chat(ChatRequest request) {
        String modelKey = route(request);                  // 路由
        rateLimiter.acquire(modelKey);                     // 限流

        try {
            ChatResponse response = circuitBreaker.executeSupplier(
                    () -> models.get(modelKey).call(request));   // 熔断

            costTracker.record(modelKey, response);        // 成本
            return response;
        } catch (Exception e) {
            return fallback(modelKey, request);            // 降级
        }
    }
}
```

### 2.2.2 路由与降级细节

```java
// 路由：按租户策略或成本优先级选择模型
private String route(ChatRequest request) {
    if (request.isHighQuality()) {
        return "gpt-large";
    }
    return "gpt-small";
}

// 降级：主模型不可用时切换备用模型
private ChatResponse fallback(String failedKey, ChatRequest request) {
    String backup = backupOf(failedKey);
    return models.get(backup).call(request);
}
```

**优势**：应用无需关心多模型细节，统一治理。

## 2.3 边界与反例

| 反例 | 问题 |
|------|------|
| 业务各自直连模型 | 密钥分散、无法统一治理 |
| 网关内做业务编排 | 职责混淆，网关变胖 |
| 降级无兜底 | 备用模型也挂时直接失败 |
| 缓存不区分租户 | 数据串租户，安全风险 |

# 3. BFF 模式

BFF（Backend for Frontend，面向前端的后端）为不同前端定制接口，屏蔽后端差异。

## 3.1 结构与职责

```
前端 ──▶ BFF（Backend for Frontend）──▶ AI 服务
           │
           ├── 适配前端需求（流式、裁剪字段）
           ├── 聚合多个 AI 能力
           └── 会话与鉴权
```

适合：多端（Web/App/小程序）需求差异大。

| 前端 | 需求差异 | BFF 职责 |
|------|----------|----------|
| Web | 富交互、SSE 流式 | 组织流式片段 |
| App | 省流量、字段精简 | 裁剪响应字段 |
| 小程序 | 弱网、需兜底 | 聚合 + 降级 |

## 3.2 实现示例

```java
@RestController
@RequestMapping("/bff/app")
public class AppBffController {

    private final ChatFacade chatFacade;

    public AppBffController(ChatFacade chatFacade) {
        this.chatFacade = chatFacade;
    }

    @PostMapping("/chat")
    public AppChatView chat(@RequestBody AppChatRequest req) {
        ChatResponse resp = chatFacade.chat(req.toChatRequest());
        // 裁剪字段，只返回 App 需要的内容
        return AppChatView.of(resp.content(), resp.usage());
    }
}
```

## 3.3 适用与反例

- 适用：多端需求差异大、需要按端裁剪或聚合；
- 反例：只有单一前端时引入 BFF，只会增加一跳网络与维护成本。

# 4. 编排层模式

编排层（orchestration layer）负责把一个复杂任务拆解为多个步骤，调用能力层并维护状态。

## 4.1 编排职责

```
编排服务
├── 任务拆分
├── 调用能力层（Chat/RAG/Tools）
├── 状态管理
└── 结果聚合
```

## 4.2 状态机实现

### 4.2.1 状态定义

```java
public enum AgentStatus {
    PENDING,      // 待执行
    EXECUTING,    // 执行中
    COMPLETED,    // 完成
    FAILED        // 失败
}
```

### 4.2.2 执行循环

```java
@Service
public class AgentOrchestrator {

    private final StateStore stateStore;
    private final StepExecutor stepExecutor;

    public AgentOrchestrator(StateStore stateStore,
                             StepExecutor stepExecutor) {
        this.stateStore = stateStore;
        this.stepExecutor = stepExecutor;
    }

    public String handle(String task, UserContext user) {
        // 1. 状态初始化
        AgentState state = stateStore.create(task, user);

        // 2. 循环执行（带最大步数保护）
        while (state.status() == AgentStatus.EXECUTING) {
            state = stepExecutor.step(state);
            stateStore.save(state);

            if (state.steps() > MAX_STEPS) {
                state = state.fail("超过最大步数");
            }
        }
        return state.result();
    }
}
```

## 4.3 编排 vs 编排式智能体

| 维度 | 编排层 | 编排式智能体 |
|------|--------|--------------|
| 决策者 | 代码 / 状态机 | 模型自主决策 |
| 可控性 | 高 | 较低 |
| 灵活性 | 中 | 高 |
| 适用 | 流程稳定的业务 | 开放式任务 |

生产实践中常将两者结合：外层用编排层保证可控，内层允许模型在受限空间内自主决策。

# 5. 事件驱动模式

事件驱动（event-driven）用事件总线解耦任务提交与处理，适合高并发、异步、长任务场景。

## 5.1 事件流结构

```
任务提交 → 事件总线 → 多个消费者
              ├── 执行 Agent
              ├── 记录审计
              ├── 更新统计
              └── 发送通知
```

## 5.2 实现示例

### 5.2.1 事件发布

```java
@Service
public class TaskService {

    private final ApplicationEventPublisher publisher;

    public TaskService(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    public void submit(String taskId) {
        publisher.publishEvent(new TaskSubmittedEvent(taskId));
    }
}
```

### 5.2.2 事件消费

```java
@Component
public class TaskEventHandlers {

    @EventListener
    public void onTaskSubmitted(TaskSubmittedEvent event) {
        agentRunner.runAsync(event.taskId());
    }

    @EventListener
    public void onTaskCompleted(TaskCompletedEvent event) {
        auditService.record(event);
        metricsService.record(event);
        notificationService.notify(event.userId(), event.result());
    }
}
```

## 5.3 可靠性与幂等

| 风险 | 对策 |
|------|------|
| 事件丢失 | 持久化消息 + 重试 |
| 重复消费 | 幂等键去重 |
| 消费积压 | 水平扩容消费者 |
| 顺序错乱 | 按 Key 分区保序 |

# 6. 架构模式选择

## 6.1 选型对照表

| 场景 | 推荐架构 |
|------|----------|
| 单一 AI 功能 | 分层（接入 + 能力） |
| 多模型统一治理 | 模型网关 |
| 多端差异大 | BFF |
| 长任务、多步骤 | 编排层 + 状态机 |
| 高并发、异步 | 事件驱动 + 队列 |
| 企业统一 AI 平台 | 网关 + 编排 + 能力分层 |

## 6.2 组合使用

以上模式并不互斥，真实系统往往是组合：

```
客户端 ─▶ BFF ─▶ 接入层 ─▶ 应用/编排层 ─▶ 模型网关 ─▶ 模型
                                  │
                                  └─▶ 事件总线（异步任务）
```

## 6.3 选型误区

| 误区 | 后果 |
|------|------|
| 过早引入事件驱动 | 复杂度陡增，调试困难 |
| 单功能硬套微服务 | 运维成本过高 |
| 忽略网关 | 模型治理失控 |
| 编排层塞入业务逻辑 | 层职责混淆 |

# 7. 部署拓扑

## 7.1 拓扑图

```
                    ┌──────────────┐
  客户端 ──────────▶│  API 网关     │  鉴权/限流/WAF
                    └──────┬───────┘
                           ▼
                    ┌──────────────┐
                    │  AI 应用服务  │  无状态，可水平扩展
                    └──┬───┬───┬───┘
        ┌──────────────┘   │   └──────────────┐
        ▼                  ▼                  ▼
  ┌──────────┐      ┌────────────┐     ┌────────────┐
  │ 模型网关  │      │ 向量库      │     │ 状态存储    │
  └────┬─────┘      └────────────┘     └────────────┘
       ▼
  ┌──────────────────────────┐
  │ OpenAI / 通义 / 本地模型  │
  └──────────────────────────┘
```

## 7.2 关键组件说明

| 组件 | 职责 | 扩展方式 |
|------|------|----------|
| API 网关 | 鉴权、限流、WAF | 集群 |
| AI 应用服务 | 业务编排（无状态） | 水平扩展 |
| 模型网关 | 模型统一入口 | 集群 |
| 向量库 | 向量检索 | 分片 / 副本 |
| 状态存储 | 会话与任务状态 | 主从 / 集群 |

## 7.3 高可用要点

- 各组件均需多实例，避免单点；
- 模型侧配置多供应商，避免单供应商故障；
- 关键路径设置超时与熔断；
- 状态存储需持久化与备份。

# 8. 无状态设计

## 8.1 状态外置

```
应用服务无状态 → 可任意扩容、重启、故障转移
状态外置：
  - 会话/任务状态 → Redis / DB
  - 记忆 → Redis / DB
  - 向量 → 向量库
  - 文件 → 对象存储
```

> **无状态是水平扩展的前提**。

## 8.2 实现示例

```java
@Service
public class SessionService {

    private final StringRedisTemplate redis;

    public SessionService(StringRedisTemplate redis) {
        this.redis = redis;
    }

    // 状态写入外部存储，应用实例本身不持有
    public void save(String sessionId, SessionState state) {
        redis.opsForValue().set("session:" + sessionId, state.toJson());
    }

    public SessionState load(String sessionId) {
        String json = redis.opsForValue().get("session:" + sessionId);
        return json == null ? SessionState.empty() : SessionState.fromJson(json);
    }
}
```

## 8.3 边界与反例

| 反例 | 问题 |
|------|------|
| 会话存本地内存 | 扩容后请求落到别的实例即丢失 |
| 文件写本地磁盘 | 重启即丢，多实例不一致 |
| 状态与业务强耦合 | 难以外置，扩展受限 |
| 外置状态无 TTL | 存储无限膨胀 |

# 9. 多环境隔离

## 9.1 环境矩阵

| 环境 | 用途 | 模型/密钥 |
|------|------|-----------|
| dev | 开发 | 小模型、测试密钥 |
| test | 测试 | 小模型、独立配额 |
| staging | 预发 | 与生产同配置 |
| prod | 生产 | 正式模型与密钥 |

## 9.2 配置管理

```yaml
# application-prod.yml
llm:
  provider: openai
  model: gpt-large
  api-key: ${LLM_API_KEY}      # 由密钥管理服务注入
  quota:
    daily-tokens: 5000000
```

配置通过环境变量或密钥管理服务注入，严禁硬编码到代码或提交到仓库。

## 9.3 隔离要点

- 各环境网络与资源隔离，避免误连生产；
- 密钥分环境独立，禁止共用；
- 配额独立，防止测试流量挤占生产额度；
- staging 尽量与生产同配置，才能提前暴露问题。

# 10. 生产注意事项

## 10.1 风险与建议

| 风险 | 建议 |
|------|------|
| 业务耦合模型 | 通过网关/接口隔离 |
| 有状态服务 | 状态外置 |
| 无治理层 | 用切面统一治理 |
| 单点 | 各组件集群部署 |
| 环境混用 | 严格隔离 |
| 无降级设计 | 每层都要有降级 |

## 10.2 架构反模式

| 反模式 | 危害 |
|--------|------|
| 大泥球（Big Ball of Mud） | 无法替换模型与演进 |
| 分布式单体 | 拆分成本高却无独立性 |
| 密钥硬编码 | 泄露风险 |
| 无超时 / 无熔断 | 故障级联 |
| 状态内嵌于实例 | 无法水平扩展 |

# 11. 小结

- 分层架构：接入 → 应用 → 能力 → 治理 → 模型 → 基础设施。
- 模型网关统一多模型治理（路由、限流、缓存、降级）。
- BFF 适配多端差异，单端时不必引入。
- 长任务用编排层 + 状态机，高并发用事件驱动。
- 应用服务必须无状态，状态外置。
- 多环境严格隔离。
