# 1. 什么是 OpenAI 兼容

结论先行：OpenAI 兼容（OpenAI-Compatible）指厂商暴露与 OpenAI 相同的 HTTP 接口格式，客户端只改 `base_url` 与 `model` 即可切换。它是多模型统一接入成本最低的路径。

## 1.1 兼容的本质

### 1.1.1 协议即契约

“兼容”意味着请求与响应的**字段名、路径、语义**一致：

```text
POST {base_url}/chat/completions
body:  { model, messages, temperature, max_tokens, ... }
resp:  { choices[].message.content, usage, ... }
```

只要遵守这份契约，任何基于 OpenAI SDK 写的客户端都能直接复用。

### 1.1.2 base_url 与 model

切换厂商本质上只是替换两个变量：

```java
// 官方 OpenAI
baseUrl = "https://api.openai.com/v1"
model   = "gpt-4o-mini"

// 通义千问（兼容模式）
baseUrl = "https://dashscope.aliyuncs.com/compatible-mode/v1"
model   = "qwen-plus"

// DeepSeek
baseUrl = "https://api.deepseek.com/v1"
model   = "deepseek-chat"

// 自建网关
baseUrl = "http://llm-gateway.internal/v1"
model   = "internal-model"
```

> **这是接入成本最低的路径**，也是多模型统一接入的基础。

## 1.2 为什么成为事实标准

### 1.2.1 生态与工具链

大量框架、SDK、可观测工具默认按 OpenAI 格式对接，兼容它等于免费获得整个生态。

### 1.2.2 迁移成本

| 方案 | 迁移成本 | 说明 |
|------|----------|------|
| 兼容协议 | 极低 | 改两行配置 |
| 私有 SDK | 中 | 换客户端依赖 |
| 私有协议 | 高 | 重写调用层 |

迁移示例（从 OpenAI 切到兼容厂商）：

```java
// 之前
String baseUrl = "https://api.openai.com/v1";
String model = "gpt-4o-mini";

// 之后：仅这两行变化，其余调用代码不变
String baseUrl = "https://api.deepseek.com/v1";
String model = "deepseek-chat";
```

## 1.3 快速上手

### 1.3.1 配置示例

```yaml
llm:
  provider: dashscope
  base-url: https://dashscope.aliyuncs.com/compatible-mode/v1
  api-key: ${DASHSCOPE_API_KEY}
  model: qwen-plus
```

### 1.3.2 Java 示例

```java
OpenAiApi api = OpenAiApi.builder()
        .baseUrl(properties.getBaseUrl())
        .apiKey(properties.getApiKey())
        .build();

ChatResponse resp = api.chatCompletion(ChatCompletionRequest.builder()
        .model(properties.getModel())
        .messages(List.of(new UserMessage("你好")))
        .build());
```

# 2. 兼容的范围

结论先行：并非所有端点都同等兼容。`/chat/completions` 几乎全兼容，其余端点与高级能力兼容度递减。

## 2.1 端点覆盖

### 2.1.1 Chat 与 Embeddings

| 端点 | 兼容情况 |
|------|:--------:|
| `/chat/completions` | 几乎全部 |
| `/embeddings` | 多数 |
| `/models` | 多数 |
| `/images/generations` | 部分 |
| `/audio/*` | 部分 |

### 2.1.2 多模态端点

视觉输入通常通过 `content` 数组传入，格式为 `{"type":"image_url", ...}`，各厂商对 URL 与 base64 的支持不一。

```json
{
  "role": "user",
  "content": [
    {"type": "text", "text": "这张图里有什么？"},
    {"type": "image_url", "image_url": {"url": "https://example.com/a.png"}}
  ]
}
```

Embeddings 的兼容度也需逐项确认：

```java
EmbeddingResponse resp = api.embed(new EmbeddingRequest("text-embedding-3-small",
        List.of("第一段", "第二段")));
int dimension = resp.getEmbeddings().get(0).getEmbedding().length;
```

## 2.2 能力覆盖

### 2.2.1 流式与函数调用

| 能力 | 兼容情况 |
|------|:--------:|
| 流式（SSE） | 多数 |
| 函数调用 | 多数 |
| 并行工具调用 | 部分 |

### 2.2.2 JSON Mode 与结构化输出

JSON Mode（`response_format: {"type":"json_object"}`）多数支持，但 JSON Schema 严格模式（Structured Outputs）支持度较低。

## 2.3 兼容性分级表

| 等级 | 含义 | 使用建议 |
|------|------|----------|
| A | 完全兼容 | 可直接切换 |
| B | 兼容但有差异 | 需适配层 |
| C | 部分兼容 | 需契约测试 |
| D | 不兼容 | 单独接入 |

# 3. 厂商差异处理

结论先行：格式兼容不代表行为一致。差异集中在参数、格式与错误语义三处，必须用适配层显式处理。

## 3.1 差异类型

### 3.1.1 参数差异

| 差异点 | 说明 |
|--------|------|
| 模型名 | 各厂商不同 |
| 参数支持 | 部分参数不支持（如 `seed`、`logprobs`） |
| 参数范围 | 温度上限可能不同 |
| 默认值 | 默认行为可能不同 |

### 3.1.2 格式差异

工具调用（tool calling）的字段细节、`finish_reason` 取值、`usage` 是否返回缓存命中数等，都可能不同。

### 3.1.3 语义差异

| 差异点 | 说明 |
|--------|------|
| 错误码 | 语义可能不同 |
| 限流策略 | RPM/TPM 不同 |
| 上下文长度 | 上限不同 |
| 计费口径 | Token 统计口径不同 |

## 3.2 适配层实现

### 3.2.1 ProviderAdapter

```java
public class ProviderAdapter {

    public ChatRequest adapt(ChatRequest request, Provider provider) {
        // ① 过滤不支持的参数
        if (!provider.supportsSeed()) {
            request.setSeed(null);
        }
        // ② 模型名映射
        request.setModel(provider.resolveModel(request.getModel()));

        // ③ 参数范围裁剪
        request.setTemperature(clamp(
                request.getTemperature(), provider.minTemp(), provider.maxTemp()));

        return request;
    }
}
```

### 3.2.2 错误码映射

```java
public class ErrorMapper {

    public ApiError map(Provider provider, int httpCode, String body) {
        return switch (httpCode) {
            case 401, 403 -> new ApiError(ApiErrorType.AUTH, body);
            case 429 -> new ApiError(ApiErrorType.RATE_LIMIT, body);
            default -> httpCode >= 500
                    ? new ApiError(ApiErrorType.UPSTREAM, body)
                    : new ApiError(ApiErrorType.BAD_REQUEST, body);
        };
    }
}
```

# 4. 自建网关

结论先行：多厂商场景下，直接让应用对接各厂商会导致重复治理。统一网关屏蔽差异，是规模化接入的标准做法。

## 4.1 网关职责

### 4.1.1 路由

按租户、任务类型、成本预算选择模型。

```text
应用 ──▶ LLM 网关 ──▶ 各厂商
          │
          ├── 统一鉴权
          ├── 模型路由
          ├── 配额与限流
          ├── 缓存
          ├── 成本统计
          ├── 故障降级
          └── 审计日志
```

### 4.1.2 限流与配额

| 维度 | 说明 |
|------|------|
| RPM | 每分钟请求数 |
| TPM | 每分钟 Token 数 |
| 并发 | 同时在途请求数 |
| 预算 | 日/月成本上限 |

### 4.1.3 缓存与降级

- 缓存：相同请求命中缓存直接返回，降低成本与延迟。
- 降级：主模型失败时切备用模型，保证可用性。

```java
public ChatResponse chatWithFallback(ChatRequest request) {
    for (String modelKey : List.of("primary", "secondary", "cheap")) {
        try {
            return models.get(modelKey).call(request);
        } catch (ServiceUnavailableException | RateLimitException e) {
            log.warn("模型 {} 不可用，尝试降级", modelKey);
        }
    }
    throw new ApiException("所有模型均不可用");
}
```

缓存键需包含模型名、消息与关键参数，避免不同请求串味：

```java
String cacheKey = sha256(modelKey + "|" + objectMapper.writeValueAsString(messages)
        + "|" + temperature);
```

## 4.2 网关实现

### 4.2.1 Controller 骨架

```java
@RestController
@RequestMapping("/v1")
public class LlmGatewayController {

    private final Map<String, ChatModel> models;
    private final Router router;
    private final RateLimiter rateLimiter;
    private final CostTracker costTracker;

    @PostMapping("/chat/completions")
    public ChatResponse chat(@RequestBody ChatRequest request,
                             @RequestHeader("X-Tenant-Id") String tenantId) {
        // ① 路由
        String modelKey = router.route(request, tenantId);

        // ② 限流
        rateLimiter.acquire(tenantId);

        // ③ 调用
        ChatResponse response = models.get(modelKey).call(request);

        // ④ 成本
        costTracker.record(tenantId, modelKey, response.getUsage());

        return response;
    }
}
```

### 4.2.2 过滤器链

```java
// 伪代码：网关过滤器顺序
authFilter -> tenantFilter -> rateLimitFilter
    -> cacheFilter -> routeFilter -> retryFilter -> costFilter
```

## 4.3 网关价值表

| 网关价值 | 说明 |
|----------|------|
| 统一接入 | 应用只对接网关 |
| 集中治理 | 限流/缓存/成本/审计 |
| 无感切换 | 换模型只改网关配置 |
| 故障隔离 | 统一降级策略 |

# 5. 常用网关方案

结论先行：选型取决于团队技术栈与运维能力。开源聚合方案上手快，自建网关可控性高。

## 5.1 开源方案

| 方案 | 说明 |
|------|------|
| One API / New API | 开源，多厂商聚合 |
| LiteLLM | Python 生态，功能全 |
| 自建 Spring Boot 网关 | 可控，贴合业务 |
| 云厂商网关 | 托管，免运维 |

以 LiteLLM 为例，配置文件即可聚合多厂商：

```yaml
model_list:
  - model_name: fast
    litellm_params:
      model: openai/gpt-4o-mini
      api_key: os.environ/OPENAI_API_KEY
  - model_name: standard
    litellm_params:
      model: openai/qwen-plus
      api_base: https://dashscope.aliyuncs.com/compatible-mode/v1
      api_key: os.environ/DASHSCOPE_API_KEY
```

调用方仍使用 OpenAI 格式：

```bash
curl http://gateway:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-gateway" \
  -d '{"model":"standard","messages":[{"role":"user","content":"你好"}]}'
```

## 5.2 自建与托管对比

| 维度 | 自建 | 托管 |
|------|------|------|
| 可控性 | 高 | 中 |
| 运维成本 | 高 | 低 |
| 定制能力 | 强 | 弱 |
| 上线速度 | 慢 | 快 |

# 6. 模型名抽象

结论先行：应用不应硬编码真实模型名，而应使用逻辑名，由配置映射到具体厂商与模型，实现无感切换。

## 6.1 逻辑名

### 6.1.1 配置结构

```yaml
# 用逻辑名屏蔽真实模型名
llm:
  models:
    fast:
      provider: openai
      model: gpt-4o-mini
      base-url: https://api.openai.com/v1
    standard:
      provider: dashscope
      model: qwen-plus
      base-url: https://dashscope.aliyuncs.com/compatible-mode/v1
    strong:
      provider: deepseek
      model: deepseek-chat
      base-url: https://api.deepseek.com/v1
```

### 6.1.2 解析流程

```text
应用请求 "standard"
        │
        ▼
  ModelRegistry 查表
        │
        ▼
 provider=dashscope, model=qwen-plus
        │
        ▼
 ProviderAdapter 适配 -> 实际 HTTP 调用
```

```java
// 应用只用逻辑名
String answer = llmClient.chat("standard", prompt);
```

> 换模型时只改配置，**应用代码零改动**。

## 6.2 灰度与切换

| 策略 | 说明 |
|------|------|
| 按租户灰度 | 部分租户先切 |
| 按流量比例 | 百分比分流 |
| 按任务类型 | 不同任务不同模型 |
| 快速回滚 | 配置级回滚 |

```java
// 按流量比例灰度：10% 走新模型
String logicalName = ThreadLocalRandom.current().nextInt(100) < 10
        ? "standard-v2"
        : "standard";
String answer = llmClient.chat(logicalName, prompt);
```

> 灰度期间需分别统计各逻辑名的成功率与成本，避免“新模型拖垮整体”。

# 7. 兼容性测试

结论先行：切换厂商前必须用自动化测试验证关键能力。格式兼容是“看起来能跑”，契约测试才是“确认可用”。

## 7.1 测试维度

### 7.1.1 流式

```java
@Test
void provider_should_support_streaming() {
    ChatModel model = buildModel(provider);
    List<String> chunks = new ArrayList<>();

    model.stream("你好", chunk -> chunks.add(chunk));

    assertFalse(chunks.isEmpty());
}
```

### 7.1.2 工具调用

```java
@Test
void provider_should_support_tool_calling() {
    // 测试工具调用是否可用
    ChatResponse response = model.call(ChatRequest.builder()
            .messages(UserMessage.from("北京天气"))
            .toolSpecifications(...)
            .build());

    assertTrue(response.aiMessage().hasToolExecutionRequests());
}
```

### 7.1.3 结构化输出

验证 JSON Mode 是否稳定返回合法 JSON，并对 schema 校验。

## 7.2 契约测试

| 用例 | 断言 |
|------|------|
| 基础对话 | 返回非空 content |
| 流式 | 收到多个 chunk 且以结束标记收尾 |
| 工具调用 | 返回 tool_calls 且字段完整 |
| 超长输入 | 返回明确错误而非挂起 |
| 非法参数 | 返回可识别的 4xx |
| 错误码映射 | 401/429/5xx 映射到统一异常 |
| 长上下文 | 超限时返回明确错误 |

参数化测试可覆盖多厂商：

```java
@ParameterizedTest
@EnumSource(Provider.class)
void all_providers_should_satisfy_contract(Provider provider) {
    ChatModel model = buildModel(provider);

    // 基础对话
    assertFalse(model.call("你好").isBlank());
    // 流式
    assertFalse(streamAndJoin(model, "你好").isEmpty());
    // 错误映射
    assertThrows(AuthException.class, () -> buildModel(provider, "bad-key").call("hi"));
}
```

> 切换厂商前，用测试验证关键能力是否兼容。

# 8. 生产注意事项

结论先行：把“假设完全兼容”当作最大风险，逐项验证并做防御性适配。

## 8.1 风险清单

| 风险 | 建议 |
|------|------|
| 假设完全兼容 | 逐项测试差异 |
| 参数不兼容报错 | 按厂商裁剪参数 |
| 无网关 | 多厂商时建议加网关 |
| 模型名硬编码 | 用逻辑名 + 配置 |
| 错误码语义不同 | 统一映射 |
| 无限流 | 网关统一限流 |

## 8.2 上线检查表

- [ ] 关键能力（流式/工具/结构化）已通过契约测试。
- [ ] 参数按厂商裁剪，非法参数不会直传。
- [ ] 真实模型名已抽象为逻辑名。
- [ ] 错误码已统一映射。
- [ ] 网关已配置限流、缓存、降级。
- [ ] 成本已按租户归因。

# 9. 小结

## 9.1 要点回顾

- OpenAI 兼容协议是接入多模型的最快路径。
- 格式兼容但细节有差异，需适配。
- 自建网关可统一治理（路由/限流/缓存/成本/审计）。
- 用逻辑模型名屏蔽真实模型名，实现无感切换。
- 切换前必须做兼容性测试。

## 9.2 落地建议

- 从第一天就引入逻辑模型名与适配层，避免后续返工。
- 网关先做限流与成本统计，再逐步加缓存与降级。
- 把契约测试纳入 CI，切换厂商前自动回归。
