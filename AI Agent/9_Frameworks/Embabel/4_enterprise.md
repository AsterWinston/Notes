# 1. Spring 集成

结论先行：Embabel 由 Spring 生态开发者创建，天然以 Spring 为一等公民。通过 `@EnableAgent` 开启自动配置，`AgentRunner` 作为 Bean 注入应用。集成成本低，但需明确 Bean 的作用域与生命周期。

## 1.1 依赖与自动配置

### 1.1.1 Maven 依赖

```xml
<dependency>
    <groupId>com.embabel</groupId>
    <artifactId>embabel-agent-spring-boot-starter</artifactId>
    <version>0.1.0</version>
</dependency>
```

版本以官方最新为准；生产环境应锁定版本，避免自动升级引入不兼容。

### 1.1.2 配置项

```yaml
embabel:
  agent:
    scan-packages:
      - com.example.agents
  planner:
    strategy: regression
    max-depth: 20
    timeout-ms: 2000
```

| 配置项 | 默认 | 说明 |
|--------|------|------|
| `agent.scan-packages` | 主包 | Agent 扫描路径 |
| `planner.strategy` | regression | 搜索策略 |
| `planner.max-depth` | 20 | 搜索深度上限 |
| `planner.timeout-ms` | 2000 | 规划超时 |

## 1.2 @EnableAgent 配置

### 1.2.1 完整配置类

```java
@Configuration
@EnableAgent
public class AgentConfig {

    @Bean
    public AgentRunner agentRunner(ChatClient.Builder chatClientBuilder) {
        return AgentRunner.builder()
                .chatClientBuilder(chatClientBuilder)
                .build();
    }
}
```

### 1.2.2 Bean 装配

| Bean | 作用域 | 说明 |
|------|--------|------|
| `AgentRunner` | Singleton | 无状态，可共享 |
| `AgentContext` | Prototype | 每次执行独立 |
| `ChatClient` | Singleton | 复用连接池 |

```java
@Bean
@Scope("prototype")
public AgentContext agentContext() {
    return AgentContext.builder().build();
}
```

反例：把 `AgentContext` 注册为 Singleton 会导致并发请求共享状态，产生串数据。上下文必须每次执行新建。

## 1.3 边界与反例

反例：在 `@Configuration` 中手动 `new` Agent 实例，绕过 Spring 扫描，导致动作未注册。应让 Agent 成为 Spring Bean。

# 2. 分层架构

结论先行：推荐"Controller → Service → AgentRunner → Agent"的分层，Agent 只关心规划与动作，不感知 Web 与持久化细节。

## 2.1 分层职责

```
Controller → Service → AgentRunner → Agent（GOAP）
                          │
                          └── ChatClient / Tools
```

### 2.1.1 Controller

负责协议转换、鉴权入口、参数校验，不包含业务规划逻辑。

### 2.1.2 Service

负责事务边界、领域编排、调用 AgentRunner 并组装结果。

### 2.1.3 AgentRunner

负责规划与执行，是 Agent 与 Spring 之间的适配层。

```java
@RestController
@RequestMapping("/agent")
public class TravelController {

    private final TravelService service;

    public TravelController(TravelService service) {
        this.service = service;
    }

    @PostMapping("/plan")
    public PlanResponse plan(@RequestBody PlanRequest req) {
        return service.plan(req.city());
    }
}
```

```java
@Service
public class TravelService {

    private final AgentRunner runner;

    public TravelService(AgentRunner runner) {
        this.runner = runner;
    }

    public PlanResponse plan(String city) {
        AgentContext ctx = AgentContext.builder()
                .withInput("city", city)
                .build();
        AgentResult result = runner.run(
                travelAgent(), Goal.of("planReady", true), ctx);
        return new PlanResponse((String) result.state().get("plan"));
    }
}
```

## 2.2 依赖方向

依赖只能自上而下：Controller 依赖 Service，Service 依赖 AgentRunner，AgentRunner 依赖 Agent。反向依赖会造成循环与测试困难。

| 层 | 可依赖 | 不可依赖 |
|----|--------|----------|
| Controller | Service | Agent |
| Service | AgentRunner | Controller |
| Agent | 领域服务/工具 | Web 层 |

## 2.3 边界与反例

反例：Agent 动作里直接读 `HttpServletRequest`。这会让 Agent 与 Web 容器耦合，无法在批处理或测试中复用。

# 3. 与 Spring AI 结合

结论先行：Embabel 负责"目标规划"，Spring AI 负责"模型调用与 RAG（Retrieval-Augmented Generation，检索增强生成）"。两者互补，动作内部调用 Spring AI 的 `ChatClient` 或向量库。

## 3.1 职责划分

```
Embabel 负责"目标规划"
Spring AI 负责"模型调用与 RAG"

两者组合：
  Embabel Agent 的动作内部调用 Spring AI 的 ChatClient
```

| 关注点 | Embabel | Spring AI |
|--------|---------|-----------|
| 目标与路径 | 负责 | 不涉及 |
| 模型调用 | 通过动作 | 负责 |
| RAG 检索 | 触发 | 负责 |
| 状态管理 | AgentContext | 会话记忆 |

### 3.1.1 Embabel 规划

把"先检索、再总结、后生成"的路径交给规划器，而非硬编码。

### 3.1.2 Spring AI 调模型与 RAG

把模型、向量库、提示模板等基础设施交给 Spring AI，避免重复造轮子。

## 3.2 集成示例

### 3.2.1 检索动作

```java
@Action(name = "检索知识", preconditions = {}, effects = {"retrieved"})
public void retrieve(AgentContext ctx, String question) {
    List<Document> docs = vectorStore.similaritySearch(
            SearchRequest.builder().query(question).topK(5).build());

    ctx.setState("context", joinText(docs));
    ctx.setState("retrieved", true);
}
```

### 3.2.2 ChatClient 配置

```java
@Bean
ChatClient chatClient(ChatClient.Builder builder, VectorStore vectorStore) {
    return builder
            .defaultSystem("你是企业内部助手，回答需引用知识库。")
            .defaultAdvisors(new QuestionAnswerAdvisor(vectorStore))
            .build();
}
```

### 3.2.3 函数调用（Function Calling）

除 RAG 外，Spring AI 的函数调用可把外部能力注册给模型，由动作触发：

```java
@Bean
FunctionCallback weatherFunction(WeatherService service) {
    return FunctionCallback.builder()
            .function("getWeather", service::get)
            .description("查询城市天气")
            .inputType(WeatherRequest.class)
            .build();
}
```

## 3.3 边界与反例

反例：在 Embabel 里重新实现一套模型调用层，与 Spring AI 重复，导致配置分散、维护成本翻倍。生产建议：明确分工，Embabel 不直接持有模型客户端。

# 4. 可观测

结论先行：可观测依赖事件与 Trace。Embabel 在执行动作时发布事件，监听器据此记录日志与指标。

## 4.1 事件与监听

### 4.1.1 ActionExecutedEvent

```java
@Component
public class AgentTraceListener {

    @EventListener
    public void onActionExecuted(ActionExecutedEvent event) {
        log.info("action={} duration={}ms state={}",
                event.actionName(), event.durationMs(),
                event.stateChanges());
    }
}
```

### 4.1.2 规划事件

除执行事件外，还应监听规划事件（选择了哪些动作、是否回溯），否则无法解释"为什么走了这条路径"。

| 事件 | 关注点 |
|------|--------|
| PlanningStarted | 目标、初始状态 |
| ActionSelected | 选中动作、依据 |
| ActionExecuted | 耗时、状态变化 |
| PlanCompleted | 最终路径 |

## 4.2 指标与追踪

| 指标 | 含义 |
|------|------|
| plan_duration | 规划耗时 |
| action_duration | 单动作耗时 |
| replan_count | 重规划次数 |
| step_count | 执行步数 |

结合 Micrometer / OpenTelemetry 输出，并让 traceId 贯穿规划与执行。

```java
Timer.Sample sample = Timer.start(meterRegistry);
try {
    Plan plan = planner.plan(goal, ctx);
    meterRegistry.counter("agent.plan.success").increment();
} finally {
    sample.stop(meterRegistry.timer("agent.plan.duration"));
}
```

## 4.3 边界与反例

反例：只记录成功日志，失败与回溯不记录，导致线上问题无法复盘。应记录所有规划决策。

# 5. 可靠性

结论先行：可靠性的核心是"限制 + 兜底"。限制指超时与步数上限，兜底指重试、降级与持久化恢复。

## 5.1 超时与重试

| 手段 | 说明 |
|------|------|
| 超时 | 每个动作设超时 |
| 重试 | 幂等动作 |
| 降级 | 备用路径 |
| 步数上限 | 防止规划过长 |
| 状态持久化 | 支持恢复 |

```java
RetryTemplate retry = RetryTemplate.builder()
        .maxAttempts(3)
        .fixedBackoff(200)
        .retryOn(TransientException.class)
        .build();
```

重试只对幂等动作安全；有副作用的动作必须先去重。

## 5.2 降级与熔断

当 LLM 或外部工具不可用时，规划器应能走"备用动作"。例如主检索失败则用缓存检索。熔断器（Circuit Breaker）可防止对故障依赖的持续冲击。

```java
CircuitBreaker breaker = CircuitBreaker.ofDefaults("weatherApi");
Supplier<String> guarded = CircuitBreaker.decorateSupplier(
        breaker, () -> weatherService.get(city));

try {
    return guarded.get();
} catch (CallNotPermittedException e) {
    return cache.get(city);   // 降级到缓存
}
```

## 5.3 状态持久化与恢复

长流程需要把 `AgentContext` 持久化，进程重启后从断点恢复。

| 存储 | 适用 |
|------|------|
| 内存 | 短流程、单机 |
| Redis | 中等流程、多实例 |
| 数据库 | 长流程、强一致 |

```java
@Bean
public ContextRepository contextRepository(RedisTemplate<String, Object> redis) {
    return new RedisContextRepository(redis, Duration.ofHours(2));
}
```

## 5.4 边界与反例

反例：对所有异常都重试，包括参数错误。参数错误重试不会成功，反而放大负载。应区分瞬时错误与永久错误。

# 6. 安全

结论先行：Agent 能调用工具与模型，安全边界必须显式定义。核心是动作级鉴权、数据隔离、注入防御与审计。

## 6.1 动作级鉴权

不同角色可见的动作集合不同。规划前先按调用者身份过滤动作集合。

```java
@PreAuthorize("hasAuthority('AGENT_TRAVEL_PLAN')")
public AgentResult runTravelPlan(String city) {
    // ...
}
```

```java
public List<Action> visibleActions(String role, List<Action> all) {
    return all.stream()
            .filter(a -> a.requiredAuthority().isEmpty()
                    || roleAuthorities(role).contains(a.requiredAuthority()))
            .toList();
}
```

## 6.2 数据隔离

| 层 | 措施 |
|----|------|
| 动作 | 动作级鉴权 |
| 数据 | 租户隔离 |
| LLM | 注入防御 |
| 审计 | 记录规划与执行 |

多租户场景下，向量库检索必须带租户过滤，否则会跨租户泄露。

```java
FilterExpression filter = FilterExpression.builder()
        .and("tenantId").isEqualTo(tenantId)
        .build();
SearchRequest request = SearchRequest.builder()
        .query(question)
        .filterExpression(filter)
        .topK(5)
        .build();
```

## 6.3 LLM 注入防御

外部输入进入提示词前需做转义与长度限制；系统提示与用户输入分区，避免指令覆盖。

## 6.4 审计

记录"谁、在什么目标下、执行了哪些动作、产生了什么状态变化"，满足合规要求。

# 7. 适用与不适用

结论先行：Embabel 适合"目标导向 + 动态环境"的任务，不适合"固定流程 + 延迟敏感"的场景。

## 7.1 适用场景

| 适用 | 说明 |
|------|------|
| 目标导向任务 | 给定目标，自动规划 |
| 动态环境 | 条件变化时重规划 |
| 需要自动规划 | 路径不固定 |
| Spring 生态 | 原生集成 |

## 7.2 不适用场景

| 不适用 | 原因 |
|--------|------|
| 固定流程 | 用 Graph 更简单可靠 |
| 简单问答 | 无需规划，直接调用即可 |
| 延迟敏感 | 搜索与重规划引入额外耗时 |
| 需高度可预测 | GOAP 路径不唯一 |

## 7.3 决策表

| 维度 | 适用 | 不适用 |
|------|------|--------|
| 目标导向任务 | 是 | 固定流程 |
| 动态环境 | 是 | 简单问答 |
| 需要自动规划 | 是 | 延迟敏感 |
| Spring 生态 | 是 | 需高度可预测 |

# 8. 生产注意事项

结论先行：Embabel 较新，生产引入需评估成熟度、约束规划、明确分工并保留迁移能力。

## 8.1 框架成熟度

框架较新，API 可能变动。建议锁定版本，跟踪发布说明，先在非核心链路试点。

## 8.2 规划不可预测

通过约束、代价函数与验证降低不可预测性；对关键路径加断言。

## 8.3 与 Spring AI 分工

明确 Embabel 负责规划、Spring AI 负责模型与 RAG，避免功能重复。

## 8.4 迁移能力

把动作实现为普通 Spring Bean，业务逻辑不依赖框架特有类型，必要时可迁移到其他编排框架。

| 风险 | 建议 |
|------|------|
| 框架较新 | 评估成熟度 |
| 规划不可预测 | 约束 + 验证 |
| 无终止 | 步数上限 |
| 与 Spring AI 重复 | 明确分工 |
| 生态小 | 保留迁移能力 |

# 9. 小结

## 9.1 核心要点

- Embabel 与 Spring 原生集成，`@EnableAgent` 开启自动配置。
- 推荐分层：Controller → Service → AgentRunner → Agent。
- 可与 Spring AI 组合：Embabel 规划，Spring AI 调模型与 RAG。
- 可观测、可靠性、安全与其他框架一致。

## 9.2 落地清单

- 适合目标导向、动态环境的任务。
- 不适合固定流程、延迟敏感的场景。
- 框架较新，需评估成熟度并保持迁移能力。
