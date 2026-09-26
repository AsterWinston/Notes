# 1. 分层架构

## 1.1 六层职责

**结论先行**：LLM 应用采用六层架构，每层职责单一，上层依赖下层抽象。

```
┌───────────────────────────────────────────────┐
│  接入层   Controller / SSE / WebSocket         │
├───────────────────────────────────────────────┤
│  应用层   Facade / UseCase（编排、事务边界）    │
├───────────────────────────────────────────────┤
│  能力层   Chat / RAG / Tools / Memory           │
├───────────────────────────────────────────────┤
│  治理层   Advisor（限流/审计/脱敏/成本）        │
├───────────────────────────────────────────────┤
│  模型层   ChatModel / EmbeddingModel            │
├───────────────────────────────────────────────┤
│  基础设施  配置 / 密钥 / 状态 / 向量库 / 缓存    │
└───────────────────────────────────────────────┘
```

### 1.1.1 接入层与应用层

- **接入层**：只负责协议适配（HTTP、SSE 流式、WebSocket），不做业务。
- **应用层**：负责用例编排、事务边界、权限校验，是业务的入口。

### 1.1.2 能力层与治理层

- **能力层**：封装 Chat、RAG、Tools、Memory 等原子能力。
- **治理层**：以 Advisor/切面形式横切限流、审计、脱敏、成本。

### 1.1.3 模型层与基础设施

- **模型层**：屏蔽具体厂商（OpenAI、通义、本地模型）。
- **基础设施**：配置、密钥、状态、向量库、缓存等支撑组件。

## 1.2 分层原则

**结论先行**：业务代码只依赖能力层接口，模型细节封装在能力层。

- 上层不得直接依赖模型层具体实现。
- 每层只允许依赖其直接下层或抽象接口。
- 跨层调用通过事件或抽象接口，避免硬编码。

## 1.3 边界与坑

**结论先行**：分层最容易被"图快"破坏，最终导致模型细节渗透全栈。

- Controller 直接注入 `ChatModel` 会让换模型波及全项目。
- 治理逻辑散落各处，无法统一灰度。
- 层数过多会引入无谓样板代码。

**生产建议**：用 ArchUnit 等工具在 CI 中强制分层依赖规则。

# 2. 依赖方向

## 2.1 单向依赖

**结论先行**：依赖必须单向流动，禁止反向依赖与循环依赖。

```
Controller → Facade → Service → (ChatClient / VectorStore / Tools)
                         │
                         └── 只依赖抽象，不依赖具体模型
```

## 2.2 依赖倒置

**结论先行**：通过接口抽象，让业务层不感知模型实现。

```java
// ✅ 依赖抽象
@Service
@RequiredArgsConstructor
public class QaService {
    private final ChatClient chatClient;      // 抽象
    private final ContentRetriever retriever; // 抽象
}

// ❌ 依赖具体实现
@Service
public class BadService {
    private final OpenAiChatModel model;      // 具体厂商
}
```

## 2.3 边界与坑

**结论先行**：抽象层次不当同样有害。

- 抽象过细：为每个方法建接口，样板爆炸。
- 抽象过粗：一个接口塞进所有能力，形同虚设。
- 泄漏抽象：接口返回厂商专属类型，等于没抽象。

**生产建议**：抽象以"是否可能替换实现"为准，不为抽象而抽象。

# 3. 配置与密钥

## 3.1 配置模型

**结论先行**：所有可变参数外置到配置，代码中不出现魔法值。

```yaml
app:
  ai:
    model:
      name: gpt-4o-mini
      temperature: 0.3
      timeout: 30s
    rag:
      top-k: 20
      min-score: 0.7
```

## 3.2 类型安全绑定

**结论先行**：用 `@ConfigurationProperties` 把配置绑定到强类型对象。

```java
@ConfigurationProperties(prefix = "app.ai")
@Validated
public record AiProperties(
        @NotNull ModelConfig model,
        @NotNull RagConfig rag
) {
    public record ModelConfig(
            @NotBlank String name,
            @DecimalMin("0.0") @DecimalMax("2.0") double temperature,
            @NotNull Duration timeout) {}

    public record RagConfig(
            @Min(1) int topK,
            @DecimalMin("0.0") @DecimalMax("1.0") double minScore) {}
}
```

## 3.3 密钥管理

**结论先行**：密钥绝不硬编码，用环境变量或密钥管理服务。

### 3.3.1 注入配置

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}          # 环境变量/密钥管理
```

### 3.3.2 来源优先级

| 方式 | 安全性 | 适用 |
|------|--------|------|
| 环境变量 | 中 | 常规部署 |
| 密钥管理服务 | 高 | 生产 |
| 配置文件明文 | 低 | 禁止 |

## 3.4 边界与坑

**结论先行**：配置错误往往在上线后才暴露。

- 配置项无校验，错误值直到运行才报错。
- 多环境配置漂移，dev 能跑 prod 不能。
- 密钥被日志打印，造成泄露。

**生产建议**：启动时校验配置，密钥只在内存中解密，禁止落盘与打印。

# 4. 治理层（Advisor 链）

## 4.1 链式拦截机制

**结论先行**：治理能力通过 Advisor 链统一实现，不侵入业务代码。

```
Request  → [Log] → [Guardrail] → [Memory] → [RAG] → Model
Response ← [Log] ← [Guardrail] ← [Memory] ← [RAG] ←
```

## 4.2 完整装配

**结论先行**：在 `ChatClient` 构建时注册默认 Advisor 链。

### 4.2.1 装配代码

```java
@Configuration
public class AdvisorConfig {

    @Bean
    public ChatClient chatClient(ChatModel model,
                                 ChatMemory memory,
                                 MeterRegistry registry) {
        return ChatClient.builder(model)
                .defaultAdvisors(
                        new LoggingAdvisor(),                    // order 0
                        new GuardrailAdvisor(),                  // order 50
                        MessageChatMemoryAdvisor.builder(memory).build(),  // 100
                        new MetricsAdvisor(registry),            // order 200
                        new CostTrackingAdvisor()                // order 210
                )
                .build();
    }
}
```

## 4.3 顺序设计

**结论先行**：Advisor 顺序决定了数据流向与安全边界。

| 顺序 | Advisor | 作用 |
|------|---------|------|
| 0 | 日志 | 记录原始请求 |
| 50 | 护栏 | 安全校验 |
| 100 | 记忆 | 注入历史 |
| 150 | RAG | 注入检索 |
| 200 | 指标 | 统计 |
| 210 | 成本 | 计费 |

## 4.4 边界与坑

**结论先行**：顺序错误会导致脏数据进入上下文或被计费绕过。

- 护栏晚于记忆，可能把注入内容写入历史。
- 成本 Advisor 早于护栏，会把被拒请求也计费。
- Advisor 中做阻塞 I/O，拖慢首字延迟。

**生产建议**：为 Advisor 的 `order` 定义常量化规范，避免随意取值。

# 5. 业务隔离

## 5.1 Controller 与 Facade

**结论先行**：Controller 只依赖 Facade，不直接依赖模型或 ChatClient。

```java
@RestController
@RequiredArgsConstructor
public class ChatController {

    private final ChatFacade chatFacade;      // 只依赖 Facade

    @PostMapping("/chat")
    public String chat(@RequestBody ChatRequest req, UserContext user) {
        return chatFacade.chat(req.message(), user);
    }
}
```

## 5.2 Service 能力封装

**结论先行**：Facade 编排用例，能力细节封装在 Service。

```java
@Service
@RequiredArgsConstructor
public class ChatFacade {

    private final ChatClient chatClient;

    public String chat(String message, UserContext user) {
        // 业务逻辑：鉴权、上下文装配、调用、后处理
        return chatClient.prompt()
                .user(message)
                .call()
                .content();
    }
}
```

## 5.3 边界与坑

**结论先行**：Facade 过厚会变成"上帝类"，过薄则失去隔离意义。

- 把 RAG、Tools 细节写进 Facade，职责混乱。
- Controller 里写 Prompt 模板，无法复用。
- 缺少用户上下文传递，越权风险。

**生产建议**：Facade 只做编排，具体能力下沉到 Service，上下文显式传递。

# 6. 事务边界

## 6.1 为什么不能在事务内调 LLM

**结论先行**：LLM 调用耗时数秒到数十秒，放在事务内会长时间占用数据库连接。

- 事务持有连接与锁，导致连接池耗尽。
- 网络抖动会触发事务回滚，产生不一致。
- 事务超时后调用仍在进行，资源浪费。

## 6.2 正确姿势

**结论先行**：LLM 调用在事务外，只有落库才开事务。

### 6.2.1 反例与正例

```java
@Service
@RequiredArgsConstructor
public class OrderAiService {

    private final ChatClient chatClient;

    // ❌ 不要在事务内调用 LLM（耗时长，占用连接）
    @Transactional
    public String badAsk(String question) {
        return chatClient.prompt().user(question).call().content();
    }

    // ✅ LLM 调用在事务外
    public String goodAsk(String question) {
        String answer = chatClient.prompt().user(question).call().content();

        saveResultInTransaction(answer);          // 落库才用事务
        return answer;
    }

    @Transactional
    protected void saveResultInTransaction(String answer) {
        // 仅持久化操作
    }
}
```

## 6.3 边界与坑

**结论先行**：拆分事务要注意自调用失效与最终一致性。

- 同类内部调用 `@Transactional` 方法会绕过代理，事务不生效。
- 调用与落库之间失败，需补偿或幂等。
- 长事务拆分后需明确一致性级别。

**生产建议**：用 `TransactionTemplate` 显式控制事务范围，避免注解自调用陷阱。

# 7. 异常处理

## 7.1 异常分类

**结论先行**：把模型相关异常映射为业务可理解的错误码。

| 异常 | 含义 | HTTP 状态 |
|------|------|-----------|
| BudgetExceededException | 配额用尽 | 429 |
| RateLimitException | 触发限流 | 429 |
| ModelUnavailableException | 模型不可用 | 503 |
| GuardrailException | 护栏拦截 | 400 |

## 7.2 统一映射

**结论先行**：用 `@RestControllerAdvice` 统一转换异常为响应。

### 7.2.1 映射代码

```java
@RestControllerAdvice
public class AiExceptionHandler {

    @ExceptionHandler(BudgetExceededException.class)
    public ResponseEntity<ErrorResponse> handleBudget(BudgetExceededException e) {
        return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS)
                .body(new ErrorResponse("BUDGET_EXCEEDED", "配额已用尽"));
    }

    @ExceptionHandler(ModelUnavailableException.class)
    public ResponseEntity<ErrorResponse> handleUnavailable(ModelUnavailableException e) {
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
                .body(new ErrorResponse("MODEL_UNAVAILABLE", "服务暂时不可用"));
    }

    @ExceptionHandler(RateLimitException.class)
    public ResponseEntity<ErrorResponse> handleRateLimit(RateLimitException e) {
        return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS)
                .header("Retry-After", "10")
                .body(new ErrorResponse("RATE_LIMITED", "请求过于频繁"));
    }
}
```

## 7.3 边界与坑

**结论先行**：异常处理要防止泄露内部细节与丢失可观测信息。

- 直接把厂商异常堆栈返回给用户，泄露实现。
- 吞掉异常导致问题难以定位。
- 缺少 traceId，无法关联链路。

**生产建议**：对外返回稳定错误码，对内保留原始异常与 traceId。

# 8. 多环境隔离

## 8.1 配置分环境

**结论先行**：不同环境使用独立配置，模型与配额严格隔离。

```yaml
# application-dev.yaml
app:
  ai:
    model:
      name: gpt-4o-mini          # 开发用小模型
```

```yaml
# application-prod.yaml
app:
  ai:
    model:
      name: gpt-4o               # 生产用大模型
```

## 8.2 模型与密钥策略

**结论先行**：环境间模型、密钥、配额三者都要隔离。

### 8.2.1 隔离矩阵

| 环境 | 模型 | 密钥 |
|------|------|------|
| dev | 小模型/Mock | 测试密钥 |
| test | 小模型 | 独立配额 |
| prod | 正式模型 | 正式密钥 |

## 8.3 边界与坑

**结论先行**：环境混用最危险的是生产数据被测试密钥调用。

- 测试密钥误配到生产，导致配额被占。
- Mock 逻辑漏进生产，返回假数据。
- 配置覆盖顺序不清，profile 互相污染。

**生产建议**：启动时打印生效的模型名与密钥指纹（非密钥本身），便于核对。

# 9. 项目结构建议

## 9.1 目录划分

**结论先行**：按架构分层组织包结构，让依赖关系一目了然。

### 9.1.1 目录树

```
src/main/java/com/example/ai
├── config/             配置类
│   ├── AiProperties.java
│   ├── AdvisorConfig.java
│   └── ModelConfig.java
├── controller/         接入层
│   └── ChatController.java
├── facade/             应用层
│   └── ChatFacade.java
├── service/            能力层
│   ├── ChatService.java
│   ├── RagService.java
│   └── ToolService.java
├── advisor/            治理层
│   ├── GuardrailAdvisor.java
│   └── MetricsAdvisor.java
└── domain/             领域模型
    └── ChatRequest.java
```

## 9.2 模块边界

**结论先行**：包结构应映射架构分层，禁止跨层 import。

- `controller` 只能依赖 `facade`。
- `facade` 只能依赖 `service` 与 `domain`。
- `service` 依赖模型抽象，不依赖 `controller`。

## 9.3 边界与坑

**结论先行**：结构易随迭代腐化，需要自动化约束。

- 图省事把工具类塞进 `controller`，破坏分层。
- `domain` 中混入框架注解，污染领域模型。
- 包按技术分层而非业务，导致跨包调用频繁。

**生产建议**：用包可见性与 ArchUnit 规则守住边界。

# 10. 生产注意事项

## 10.1 风险清单

**结论先行**：架构类问题一旦上线，修复成本远高于编码阶段。

| 风险 | 建议 |
|------|------|
| Controller 依赖模型 | 通过 Facade/Service 隔离 |
| 密钥硬编码 | 环境变量/密钥管理 |
| 事务内调 LLM | 移出事务 |
| 无异常映射 | 统一异常处理 |
| 环境混用 | 严格隔离 |
| 治理散落 | 用 Advisor 统一 |

## 10.2 落地检查表

**结论先行**：上线前逐项核对架构约束。

- 业务层是否只依赖抽象
- 密钥是否外置且未打印
- LLM 调用是否在事务外
- 异常是否统一映射
- 治理是否集中在 Advisor
- 环境配置是否隔离

**生产建议**：把检查表纳入代码评审模板。

# 11. 小结

## 11.1 要点回顾

- 分层架构：接入 → 应用 → 能力 → 治理 → 模型 → 基础设施。
- 业务只依赖能力层抽象，不依赖具体模型。
- 治理能力用 Advisor 统一实现。
- LLM 调用必须移出数据库事务。
- 密钥外置、异常统一映射、环境严格隔离。

## 11.2 演进方向

**结论先行**：从单体分层走向模块化与多模型路由。

- 按业务域拆分模块，独立演进。
- 引入模型网关做多模型路由与降级。
- 用契约测试保障层间接口稳定。
