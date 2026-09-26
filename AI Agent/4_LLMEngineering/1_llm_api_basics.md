# 1. API 的基本形态

结论先行：大模型（LLM，Large Language Model，大语言模型）推理服务几乎都以“HTTP + JSON”的形式暴露。一次调用就是“提交 `messages` 与参数，拿回 `choices` 与 `usage`”。理解这个基本形态后，Spring AI、LangChain4j 等框架都只是它的封装。

## 1.1 请求与响应总览

### 1.1.1 请求报文

请求的核心是 `model`、`messages` 与采样参数：

```http
POST /v1/chat/completions
Content-Type: application/json
Authorization: Bearer sk-xxxxx
```

```json
{
  "model": "gpt-4o-mini",
  "messages": [
    {"role": "system", "content": "你是助手"},
    {"role": "user", "content": "你好"}
  ],
  "temperature": 0.7,
  "max_tokens": 1024
}
```

要点：

- `messages` 是**有顺序**的数组，顺序即对话历史。
- 参数既有“采样类”（temperature/top_p），也有“边界类”（max_tokens/stop）。
- 所有厂商的兼容接口都遵循这一骨架，差异集中在少数可选字段。

### 1.1.2 响应报文

```json
{
  "id": "chatcmpl-xxx",
  "object": "chat.completion",
  "created": 1710000000,
  "choices": [{
    "index": 0,
    "message": {"role": "assistant", "content": "你好！"},
    "finish_reason": "stop"
  }],
  "usage": {"prompt_tokens": 20, "completion_tokens": 5, "total_tokens": 25}
}
```

`finish_reason` 是判断“是否正常结束”的关键字段：

| finish_reason | 含义 | 处理建议 |
|---------------|------|----------|
| `stop` | 正常结束 | 直接使用 |
| `length` | 触达 max_tokens | 截断，可续写 |
| `tool_calls` | 请求调用工具 | 执行工具后回传 |
| `content_filter` | 被内容策略拦截 | 提示用户 |

### 1.1.3 字段速查

| 字段 | 层级 | 作用 |
|------|------|------|
| `id` | 顶层 | 请求追踪标识 |
| `choices[]` | 顶层 | 候选结果（通常 1 条） |
| `message.role` | 嵌套 | 固定为 `assistant` |
| `message.content` | 嵌套 | 文本回复 |
| `usage` | 顶层 | Token 计量 |

## 1.2 调用的生命周期

### 1.2.1 阶段拆解

一次调用可拆为五段，每段都有独立的超时与失败模式：

1. 建立连接（TCP/TLS）。
2. 发送请求体。
3. 排队与预填充（prefill，处理输入）。
4. 逐 Token 解码（decode，生成输出）。
5. 返回响应并关闭连接。

### 1.2.2 时序图

```text
客户端               网络                模型服务
  |                   |                     |
  |---- 连接(TLS) --->|-------------------->|
  |---- POST body --->|-------------------->|  预填充
  |                   |                     |  解码...
  |<--- 200 + JSON ---|<--------------------|
  |---- 关闭 -------->|                     |
```

### 1.2.3 可观测点

| 指标 | 说明 |
|------|------|
| 连接耗时 | 定位 DNS/TLS 问题 |
| 首字节延迟 | 反映排队与预填充 |
| 总耗时 | 反映解码长度 |
| 重试次数 | 反映稳定性 |

# 2. 消息角色

结论先行：`messages` 用角色（role）区分“谁说的话”。四种角色构成所有对话与工具调用的基础。

## 2.1 四种角色

### 2.1.1 system

系统指令，用于设定规则、人设与输出约束。它通常位于数组首位，优先级最高。

```json
{"role": "system", "content": "你是严谨的 Java 助教，只回答编程问题。"}
```

注意：部分模型对 `system` 的支持较弱，会把其视为普通消息；也有模型将其放在首位以外的位置会被忽略。

### 2.1.2 user 与 assistant

- `user`：用户输入。
- `assistant`：模型历史回复，可携带 `tool_calls`。

两者必须**交替出现**（多数厂商会校验），用于还原多轮对话。

### 2.1.3 tool

工具执行结果，必须携带 `tool_call_id`，与 assistant 的 `tool_calls` 一一对应。

```json
{"role": "tool", "tool_call_id": "call_1", "content": "{\"temp\": 25}"}
```

## 2.2 多轮对话组装

### 2.2.1 消息序列规则

```text
[system] -> [user] -> [assistant] -> [user] -> ...
带工具时：
[user] -> [assistant(tool_calls)] -> [tool] -> [assistant] -> ...
```

### 2.2.2 上下文裁剪

上下文有 Token 上限，超出需裁剪：

| 策略 | 说明 | 风险 |
|------|------|------|
| 滑动窗口 | 保留最近 N 轮 | 丢失早期约束 |
| 摘要压缩 | 用模型总结旧对话 | 引入信息损失 |
| 固定 system | 始终保留 system | 挤占历史空间 |

## 2.3 边界与反例

- 反例：连续两条 `user` 消息。多数厂商报错或行为不确定。
- 反例：`tool` 消息缺少 `tool_call_id`，导致 400。
- 边界：`system` 被用户内容“越权覆盖”，需在服务端做提示词隔离。

# 3. 核心参数

结论先行：参数分两类——控制“随机性”的采样参数，和约束“长度与边界”的结构参数。生产环境必须显式设置，避免依赖默认值。

## 3.1 采样参数

### 3.1.1 temperature

温度控制分布平滑程度：越低越确定，越高越发散。

| 取值 | 效果 | 场景 |
|------|------|------|
| 0~0.3 | 稳定、可复现 | 事实问答、抽取 |
| 0.4~0.7 | 平衡 | 通用对话 |
| 0.8~1.2 | 多样 | 创作、头脑风暴 |

### 3.1.2 top_p

核采样（nucleus sampling），仅从累计概率达到 `top_p` 的最小词集合中采样。`top_p=1` 表示不裁剪。

### 3.1.3 二者关系

官方建议：**二选一调整**，不要同时大改，否则效果难以归因。实践中固定一个、微调另一个。

## 3.2 长度与边界参数

### 3.2.1 max_tokens

输出 Token 上限，是成本控制的第一道闸门。`max_tokens` 越小，单次成本越可控，但可能触发 `finish_reason=length`。

### 3.2.2 stop

停止序列，命中后立即结束生成，可用于约束结构化输出边界。

```java
ChatRequest request = ChatRequest.builder()
        .model("gpt-4o-mini")
        .temperature(0.2)
        .maxTokens(1024)
        .stop(List.of("\n\n", "###"))
        .build();
```

## 3.3 其它参数与对比

| 参数 | 说明 | 建议 |
|------|------|------|
| `model` | 模型名 | 按任务分层 |
| `temperature` | 随机性 0~2 | 事实 0~0.3，创作 0.7~1.0 |
| `max_tokens` | 输出上限 | 必设，控成本 |
| `top_p` | 核采样 | 与 temperature 二选一 |
| `stop` | 停止序列 | 控制输出边界 |
| `stream` | 流式 | 对话类开启 |
| `tools` | 工具定义 | 需要时传 |
| `seed` | 随机种子 | 部分厂商支持 |

### 3.3.1 参数组合示例

不同任务对应不同参数组合：

```java
// 事实抽取：低温度 + 小输出
ChatOptions extract = ChatOptions.builder()
        .temperature(0.0)
        .maxTokens(256)
        .build();

// 创意写作：高温度 + 大输出
ChatOptions creative = ChatOptions.builder()
        .temperature(0.9)
        .maxTokens(2048)
        .build();
```

# 4. 鉴权

结论先行：调用方通过 HTTP 头携带密钥完成鉴权。密钥属于最高敏感信息，必须走环境变量或密钥管理服务，禁止硬编码。

## 4.1 密钥形态

### 4.1.1 Bearer Token

```http
Authorization: Bearer sk-xxxxx
```

### 4.1.2 组织/项目头

部分厂商额外需要组织与项目标识：

```http
OpenAI-Organization: org-xxxx
OpenAI-Project: proj-xxxx
```

## 4.2 密钥管理

### 4.2.1 环境变量与密钥管理服务

```java
@Component
public class ApiKeyProvider {

    // 从环境变量/密钥管理获取，禁止硬编码
    public String openAiKey() {
        return System.getenv("OPENAI_API_KEY");
    }
}
```

生产更推荐密钥管理服务（如 Vault、KMS），通过短期凭证下发。

### 4.2.2 轮换与最小权限

| 措施 | 说明 |
|------|------|
| 定期轮换 | 降低泄露影响面 |
| 按环境隔离 | dev/prod 使用不同密钥 |
| 按租户配额 | 单密钥限额，避免连带 |
| 审计 | 记录密钥使用来源 |

# 5. Java HTTP 调用

结论先行：JDK 17/21 提供了三种主流方式——`HttpClient`（原生）、`RestClient`（Spring 6.1+ 同步）、`WebClient`（响应式）。选择取决于并发模型与是否已在 Spring 生态内。

## 5.1 RestClient

### 5.1.1 构建与调用

```java
RestClient client = RestClient.builder()
        .baseUrl("https://api.openai.com")
        .defaultHeader("Authorization", "Bearer " + apiKey)
        .build();

ChatResponse response = client.post()
        .uri("/v1/chat/completions")
        .contentType(MediaType.APPLICATION_JSON)
        .body(Map.of(
                "model", "gpt-4o-mini",
                "messages", List.of(Map.of("role", "user", "content", "你好")),
                "max_tokens", 1024))
        .retrieve()
        .body(ChatResponse.class);
```

### 5.1.2 异常与超时

`RestClient` 默认把非 2xx 抛为 `RestClientResponseException`，可通过 `onStatus` 定制。

## 5.2 WebClient

### 5.2.1 响应式调用

```java
WebClient client = WebClient.builder()
        .baseUrl("https://api.openai.com")
        .defaultHeader("Authorization", "Bearer " + apiKey)
        .build();

Mono<ChatResponse> response = client.post()
        .uri("/v1/chat/completions")
        .bodyValue(request)
        .retrieve()
        .bodyToMono(ChatResponse.class);
```

### 5.2.2 背压与并发

响应式适合高并发与流式。注意：`WebClient` 需要 Reactor Netty 的连接池与超时配置，否则默认行为可能不符合预期。

## 5.3 框架封装

### 5.3.1 Spring AI

```java
ChatClient chatClient = ChatClient.builder(chatModel).build();

String answer = chatClient.prompt()
        .system("你是助手")
        .user("你好")
        .options(ChatOptions.builder()
                .temperature(0.7)
                .maxTokens(1024)
                .build())
        .call()
        .content();
```

### 5.3.2 LangChain4j

```java
ChatLanguageModel model = OpenAiChatModel.builder()
        .apiKey(System.getenv("OPENAI_API_KEY"))
        .modelName("gpt-4o-mini")
        .temperature(0.7)
        .timeout(Duration.ofSeconds(60))
        .build();

String answer = model.chat("你好");
```

框架与裸 HTTP 的取舍：

| 维度 | 裸 HTTP | Spring AI / LangChain4j |
|------|---------|-------------------------|
| 上手成本 | 低 | 中 |
| 多厂商适配 | 自己写 | 内置 |
| 结构化输出 | 自己解析 | 内置 |
| 流式 | 自己解析 SSE | 内置 |
| 可观测 | 自己埋点 | 部分内置 |

# 6. 超时配置

结论先行：LLM 生成耗时远高于普通接口，且波动大。**不设超时**是线上最常见的故障源。

## 6.1 超时类型与建议

### 6.1.1 连接/读取/总超时

- 连接超时：建立 TCP/TLS 的时间。
- 读取超时：等待响应的最长空闲时间。
- 总超时：整个请求的上限。

### 6.1.2 建议值

| 类型 | 建议 |
|------|------|
| 连接超时 | 1~3s |
| 读取超时 | 30~60s（生成慢） |
| 总超时 | 按业务 |

## 6.2 超时实现

```java
HttpClient httpClient = HttpClient.newBuilder()
        .connectTimeout(Duration.ofSeconds(3))          // 连接超时
        .build();

HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create(url))
        .timeout(Duration.ofSeconds(30))                // 请求超时
        .header("Content-Type", "application/json")
        .POST(BodyPublishers.ofString(body))
        .build();
```

# 7. 错误处理

结论先行：错误必须**分类**，不同类别对应不同动作（重试、降级、告警、直接失败）。统一抛出“调用失败”会掩盖根因。

## 7.1 错误分类

### 7.1.1 客户端错误 4xx

| 状态码 | 含义 | 动作 |
|--------|------|------|
| 400 | 请求非法 | 修请求，勿重试 |
| 401 | 鉴权失败 | 检查密钥 |
| 403 | 无权限 | 检查配额/模型权限 |
| 404 | 模型不存在 | 检查模型名 |
| 429 | 限流 | 退避重试 |

### 7.1.2 服务端错误 5xx

5xx 多为临时故障，可退避重试；持续失败应触发降级。

### 7.1.3 网络与超时

连接超时、读超时、连接重置等属于网络层，需与业务错误分开统计。

## 7.2 重试策略

### 7.2.1 幂等与退避

```java
public ChatResponse chat(ChatRequest request) {
    try {
        return doChat(request);
    } catch (HttpStatusCodeException e) {
        int code = e.getStatusCode().value();
        if (code == 401) throw new AuthException("API Key 无效");
        if (code == 429) throw new RateLimitException("触发限流");
        if (code >= 500) throw new ServiceUnavailableException("服务端错误");
        throw new ApiException("调用失败: " + code, e);
    } catch (ResourceAccessException e) {
        throw new TimeoutException("请求超时", e);
    }
}
```

重试必须满足：幂等 + 指数退避 + 抖动 + 次数上限。

### 7.2.2 反例

- 对 400/401 盲目重试，放大错误与成本。
- 无抖动退避，导致“惊群”加剧限流。

# 8. Token 与成本

结论先行：Token 是 LLM 的计量单位，也是成本与限流的基础。**必须采集 `usage`**，否则无法做成本归因。

## 8.1 Token 计量

### 8.1.1 估算

粗略经验：中文约 1 字 ≈ 1~2 Token，英文约 1 词 ≈ 1.3 Token。精确值应使用厂商 tokenizer。

### 8.1.2 Usage 采集

```java
Usage usage = response.getUsage();
log.info("inputTokens={} outputTokens={} totalTokens={}",
        usage.getPromptTokens(),
        usage.getCompletionTokens(),
        usage.getTotalTokens());
```

> **必须采集 Usage**，它是成本监控的基础。

## 8.2 成本控制

### 8.2.1 模型分层

| 层 | 模型 | 用途 |
|----|------|------|
| fast | 小模型 | 分类、改写 |
| standard | 中模型 | 通用对话 |
| strong | 大模型 | 复杂推理 |

### 8.2.2 预算与告警

按租户/接口维度累计 Token 成本，设置日/月预算与阈值告警。

# 9. 请求/响应日志

结论先行：日志是排障基础，但 LLM 请求体可能包含用户隐私与密钥。原则是“**默认少记，调试才全量，且必脱敏**”。

## 9.1 日志分级

### 9.1.1 DEBUG 全量

```java
if (log.isDebugEnabled()) {
    log.debug("请求: {}", sanitize(objectMapper.writeValueAsString(request)));
}
```

### 9.1.2 INFO 元数据

```java
// 生产环境：只记录元数据（模型、Token、耗时），不记录完整内容
log.info("model={} durationMs={} tokens={}",
        request.getModel(), duration, usage.getTotalTokens());
```

## 9.2 脱敏

| 字段 | 处理 |
|------|------|
| `Authorization` | 完全屏蔽 |
| 用户内容 | 采样/哈希 |
| 手机号、身份证 | 正则替换 |
| 邮箱 | 掩码 |

# 10. 生产注意事项

结论先行：把以下风险清单当作上线前的自检表，逐项确认。

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 无超时 | 必设连接/读取超时 |
| 密钥硬编码 | 环境变量/密钥管理 |
| 无重试 | 幂等调用可重试 |
| 无错误分类 | 分类处理 401/429/5xx |
| 无 Token 采集 | 采集 Usage |
| 日志泄露 | 脱敏 |
| 阻塞线程 | 高并发用响应式 |

## 10.2 上线检查表

- [ ] 连接/读取超时均已设置。
- [ ] 密钥来自密钥管理，未入库、未入日志。
- [ ] 4xx/5xx/网络异常分类处理。
- [ ] 采集 `usage` 并按租户归因。
- [ ] 日志默认仅记录元数据。
- [ ] 高并发路径使用响应式或连接池。

# 11. 小结

## 11.1 要点回顾

- API 基本形态：messages + 参数 → choices + usage。
- 角色：system / user / assistant / tool。
- 超时、鉴权、错误分类是工程基础。
- 必须采集 Usage 用于成本监控。
- 日志要脱敏，高并发用响应式。

## 11.2 落地建议

- 先搭最小可用调用，再逐项补齐超时、重试、成本与日志。
- 把参数与密钥抽到配置中心，避免散落在代码中。
- 用一层薄封装隔离厂商差异，为后续多模型接入留出空间。
