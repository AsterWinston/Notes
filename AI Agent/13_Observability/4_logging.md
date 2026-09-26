# 1. LLM 应用日志的特殊性

## 1.1 与普通日志的差异

### 1.1.1 记录内容差异

结论先行：普通日志记录"代码做了什么"，LLM 日志还必须记录"模型看到了什么、生成了什么"。

普通后端日志围绕技术事实展开：入参、出参、异常栈、SQL、耗时，足以判断"程序逻辑是否正确"。而 LLM 的结果还取决于提示（Prompt）、模型版本、温度（temperature），甚至检索（Retrieval）召回的文档片段。因此仅凭技术日志无法复现一次"坏回答"。

机制拆解：一次 LLM 调用可拆为输入组装、模型调用、输出解析、后处理四个阶段。日志需在各阶段留痕，并通过关联 ID（traceId）串起来，才能还原完整链路。

### 1.1.2 敏感数据差异

结论先行：LLM 日志的敏感面远大于普通日志，因为提示与响应天然包含用户原话、企业文档与密钥。

用户提问可能含手机号、身份证号、邮箱；检索片段可能是企业合同；工具参数可能含访问令牌（API Key）。这些内容一旦进入日志系统，就会扩散到磁盘、备份与第三方平台。

边界与反例：只在"入口处"脱敏不够——模型输出可能复述用户敏感信息，检索片段、工具返回值同样需要脱敏。反例是"只对 prompt 脱敏，忘了 response"。

生产建议：把脱敏做成日志输出的统一拦截层，而不是散落在各调用点。

## 1.2 日志在可观测性中的定位

### 1.2.1 三大支柱关系

结论先行：日志（Logging）、指标（Metrics）、链路（Tracing）是三大支柱，日志负责"细节与证据"，不可被另外两者替代。

| 支柱 | 回答的问题 | 数据特征 | 成本 |
|------|-----------|----------|------|
| 日志 Logging | 到底发生了什么 | 离散事件、高维 | 高（存储） |
| 指标 Metrics | 整体是否健康 | 聚合数值、低基数 | 低 |
| 链路 Tracing | 请求经过了哪里 | 调用关系、耗时 | 中 |

```
指标发现异常  -->  链路定位到慢 Span  -->  日志查看该请求细节
   (宏观)               (中观)                   (微观)
```

LLM 场景还多两种角色：评估（Evaluation）需要从生产日志采样真实问答对做离线回归；审计（Audit）需要回答"某用户某时刻得到什么回答、花多少钱"。这决定了 LLM 日志不能随意丢弃或缩短保留期。生产建议：保留期按合规设定，评估数据走独立的脱敏采样通道。

# 2. 结构化日志

## 2.1 为什么需要结构化

### 2.1.1 非结构化的检索困境

结论先行：拼接字符串日志无法可靠检索与聚合。

```java
// 非结构化，难以检索
log.info("用户 " + userId + " 问 " + question + "，模型回答 " + answer);
```

反例：当 question 含空格或换行时，字段边界被破坏，命令行工具无法区分 userId 与内容；日志量大时，正则解析成本极高且易错。

### 2.1.2 结构化字段设计

结论先行：用固定字段名承载可检索维度，正文只放自由文本。

推荐字段：`timestamp`、`level`、`logger`、`traceId`、`tenantId`、`userId`、`sessionId`、`event`、`model`、`duration`、`inputTokens`、`outputTokens`、`cost`、`status`。

```java
// 结构化：参数占位符，避免字符串拼接
log.info("llm.request userId={} sessionId={} model={} messages={}",
        userId, sessionId, model, messageCount);
```

机制拆解：占位符 `{}` 在日志框架内部替换，只有级别开启时才做字符串拼接，兼顾性能与可读性。

## 2.2 JSON 日志落地

### 2.2.1 字段规范与 Java 实现

结论先行：JSON 结构化日志便于 ELK（Elasticsearch + Logstash + Kibana）直接索引。

```json
{
  "timestamp": "2026-09-21T10:00:00Z",
  "level": "INFO",
  "logger": "LlmService",
  "traceId": "abc123",
  "tenantId": "acme",
  "userId": "u-1",
  "event": "llm.request",
  "model": "gpt-4o-mini",
  "messageCount": 3,
  "promptTokens": 850
}
```

结论先行：优先用 MDC（Mapped Diagnostic Context，映射诊断上下文）承载公共字段，用结构化参数承载事件字段。

```java
public void logRequest(String model, int messageCount) {
    MDC.put("event", "llm.request");
    MDC.put("model", model);
    log.info("llm.request messageCount={}", messageCount);
}
```

边界与坑：MDC 值必须是字符串；复杂对象需先序列化；字段名不要与日志框架保留字冲突（如 `message`、`level`）。

## 2.3 边界与反例

| 反模式 | 后果 | 修正 |
|--------|------|------|
| 字符串拼接 | 无法检索 | 占位符 |
| 字段名随意 | 看板难建 | 统一 schema |
| 整段 JSON 塞进 message | 双重转义 | 拆字段 |
| 每行都带完整 Prompt | 存储爆炸 | 分字段 + 截断 |

生产建议：定义一份"日志字段字典"作为团队契约，并在代码评审中检查。

# 3. 日志分级

## 3.1 级别定义

### 3.1.1 五级模型

| 级别 | 内容 |
|------|------|
| ERROR | 调用失败、异常 |
| WARN | 降级、重试、护栏触发 |
| INFO | 请求/响应摘要、Token、成本 |
| DEBUG | 完整提示与响应（仅开发） |
| TRACE | 逐 Token（极少用） |

结论先行：生产环境默认 INFO，DEBUG 只在开发或临时排查时开启。

### 3.1.2 LLM 事件的级别映射

| 事件 | 级别 | 理由 |
|------|------|------|
| 模型调用失败 | ERROR | 影响用户 |
| 触发重试 | WARN | 潜在不稳定 |
| 护栏拦截 | WARN | 安全事件 |
| 正常请求摘要 | INFO | 常规审计 |
| 完整 Prompt | DEBUG | 含敏感内容 |
| 逐 Token 流 | TRACE | 数据量极大 |

## 3.2 级别使用代码

### 3.2.1 条件日志与延迟求值

```java
if (log.isDebugEnabled()) {
    log.debug("完整提示: {}", prompt);       // 仅开发开启
}
log.info("llm.response model={} inputTokens={} outputTokens={}",
        model, inputTokens, outputTokens);

// 延迟求值：只有开启时才拼接
log.debug("context={}", () -> heavySerialize(context));
```

机制拆解：`isDebugEnabled()` 与 `Supplier` 避免在级别关闭时仍构造昂贵参数。生产瓶颈在"参数构造"与"同步 IO"，不在级别判断；高频路径应用异步 Appender（Log4j2 AsyncLogger / Logback AsyncAppender），并设置有界队列与丢弃策略。

## 3.3 常见误用

| 误用 | 问题 |
|------|------|
| 用 INFO 打印完整 Prompt | 敏感信息泄漏 + 存储爆炸 |
| 用 ERROR 打印可恢复异常 | 告警噪声 |
| DEBUG 里做远程调用 | 关闭时行为不一致 |
| 级别配置写死 | 无法临时排障 |

边界与反例：临时把线上级别调到 DEBUG 排查，务必设置自动回滚时间，否则可能因日志量打满磁盘导致服务不可用。

# 4. 脱敏

## 4.1 脱敏目标与策略

### 4.1.1 字段与脱敏方式

结论先行：优先覆盖"可直接识别个人/企业身份"和"可直接造成损失"的字段。

| 类别 | 示例 | 处理 |
|------|------|------|
| 手机号 | 13812345678 | 掩码 |
| 身份证 | 110101199001011234 | 掩码 |
| 邮箱 | a@b.com | 掩码 |
| 密钥 | sk-xxxx | 全删 |
| 银行卡 | 6222... | 掩码 |
| 地址 | 具体门牌 | 泛化 |

| 方式 | 可读性 | 可逆 | 适用 |
|------|--------|------|------|
| 掩码 *** | 保留格式 | 否 | 日志展示 |
| 哈希 | 无 | 否 | 关联分析 |
| 截断 | 部分 | 否 | 长文本 |
| 删除 | 无 | 否 | 密钥 |

## 4.2 Java 实现

### 4.2.1 正则脱敏与哈希

```java
public class LogSanitizer {

    private static final Pattern PHONE = Pattern.compile("1[3-9]\\d{9}");
    private static final Pattern ID_CARD = Pattern.compile("\\d{17}[\\dXx]");
    private static final Pattern EMAIL = Pattern.compile("[\\w.-]+@[\\w.-]+\\.\\w+");
    private static final Pattern API_KEY = Pattern.compile("sk-[A-Za-z0-9]{20,}");

    public String sanitize(String text) {
        if (text == null) return null;
        String result = text;
        result = PHONE.matcher(result).replaceAll("***");
        result = ID_CARD.matcher(result).replaceAll("***");
        result = EMAIL.matcher(result).replaceAll("***@***");
        result = API_KEY.matcher(result).replaceAll("sk-***");
        return result;
    }

    public String truncate(String text, int maxLen) {
        if (text == null || text.length() <= maxLen) return text;
        return text.substring(0, maxLen) + "...[截断]";
    }
}
```

结论先行：脱敏后仍需截断，防止超长文本拖垮日志系统。需要"可关联但不可还原"时，用 SHA-256 取前 16 位十六进制作为哈希；若必须可逆，则改用加密而非哈希。

## 4.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 正则顺序 | 身份证可能被手机号规则误伤 | 先长后短 |
| 性能 | 大文本逐条正则慢 | 预编译 + 截断先行 |
| 遗漏字段 | 只处理 prompt | 统一拦截 |
| 不可逆 | 哈希后无法还原 | 需要时用加密而非哈希 |
| 编码 | 中文边界错位 | 统一 UTF-8 |

生产建议：脱敏器必须有单元测试，覆盖每种模式的正例与反例（如 11 位但非手机号的数字串）。

# 5. 日志埋点位置

## 5.1 埋点位置选择

### 5.1.1 入口/出口/异常

结论先行：一次调用至少在入口、出口、异常三处埋点，并用 Advisor（环绕拦截器）统一实现，避免业务代码重复。

```
请求 --> [入口日志] --> 处理 --> [出口日志] --> 响应
                         |
                         +--异常--> [异常日志]
```

机制拆解：Spring AI 的 `CallAdvisor` 在调用链上形成环绕（around）逻辑，可在 `chain.nextCall()` 前后插入日志，天然覆盖所有经过该链路的调用。

## 5.2 Advisor 实现

### 5.2.1 完整实现

结论先行：请求日志记 traceId、消息数；响应日志记耗时、Token；完整内容仅 DEBUG 且必须脱敏截断；异常必须记录堆栈。

```java
public class LlmLoggingAdvisor implements CallAdvisor {

    private final LogSanitizer sanitizer;

    @Override
    public ChatClientResponse adviseCall(ChatClientRequest request,
                                         CallAdvisorChain chain) {
        long start = System.currentTimeMillis();

        log.info("llm.request traceId={} messages={}",
                MDC.get("traceId"),
                request.prompt().getInstructions().size());

        if (log.isDebugEnabled()) {
            log.debug("llm.prompt traceId={} content={}",
                    MDC.get("traceId"),
                    sanitizer.truncate(sanitizer.sanitize(extractText(request)), 1000));
        }

        try {
            ChatClientResponse response = chain.nextCall(request);

            Usage usage = response.chatResponse().getMetadata().getUsage();
            log.info("llm.response traceId={} duration={}ms inputTokens={} outputTokens={}",
                    MDC.get("traceId"), System.currentTimeMillis() - start,
                    usage.getPromptTokens(), usage.getCompletionTokens());

            return response;

        } catch (Exception e) {
            log.error("llm.error traceId={} duration={}ms model={}",
                    MDC.get("traceId"), System.currentTimeMillis() - start,
                    extractModel(request), e);
            throw e;
        }
    }

    @Override
    public String getName() { return "LlmLoggingAdvisor"; }

    @Override
    public int getOrder() { return 0; }
}
```

## 5.3 边界与生产建议

| 边界 | 说明 |
|------|------|
| Advisor 顺序 | 日志应位于最外层 |
| 流式响应 | 需在聚合后记录，不能边流边记全文 |
| 多模态 | 图片/音频只记元数据（大小、类型） |
| 异常 | 用 try/catch 保证耗时仍被记录 |

生产建议：`getOrder()` 值越小越靠外；日志 Advisor 设为 0，指标 Advisor 设为 200，保证日志覆盖指标耗时。

# 6. MDC 上下文

## 6.1 MDC 机制

### 6.1.1 ThreadLocal 与字段

结论先行：MDC 基于 ThreadLocal，仅在当前线程内可见，跨线程不会自动传递。

```java
MDC.put("traceId", traceId);
MDC.put("tenantId", tenantId);
MDC.put("userId", userId);
MDC.put("sessionId", sessionId);

try {
    // 处理请求
} finally {
    MDC.clear();          // 必须清理
}
```

> MDC 用 ThreadLocal，**线程池复用时要清理**，异步场景需手动传递。

## 6.2 线程池与异步

### 6.2.1 线程池复用与异步传递

结论先行：线程池复用线程时，上一个请求的 MDC 会"串"到下一个请求。

反例：Tomcat 工作线程处理完请求 A 未清理，接着处理请求 B，B 的日志里出现 A 的 userId，导致排查时张冠李戴。

```java
Map<String, String> ctx = MDC.getCopyOfContextMap();

CompletableFuture.runAsync(() -> {
    if (ctx != null) MDC.setContextMap(ctx);
    try {
        // 业务
    } finally {
        MDC.clear();
    }
}, executor);
```

边界与坑：`@Async`、`CompletableFuture`、响应式（Reactor）都需显式传递；Reactor 场景可用 `Hooks.onEachOperator` 或 `contextWrite` 携带上下文。

## 6.3 边界与坑

| 坑 | 后果 | 对策 |
|----|------|------|
| 未 clear | 上下文串号 | finally clear |
| 异步丢失 | 日志无 traceId | 显式拷贝 |
| 值过大 | 内存占用高 | 只放 ID |
| 线程池包装 | 遗漏传递 | 用 TaskDecorator |

生产建议：用 Spring 的 `TaskDecorator` 统一包装线程池，自动完成 MDC 的拷贝与清理。

# 7. 日志配置

## 7.1 Spring Boot 配置

### 7.1.1 级别格式与环境差异化

```yaml
logging:
  level:
    root: INFO
    com.example.ai: INFO
    # 开发环境开启详细日志
    com.example.ai.llm: DEBUG
  pattern:
    console: "%d{ISO8601} [%thread] %-5level [%X{traceId}] %logger{36} - %msg%n"
```

| 环境 | 级别 | 输出 |
|------|------|------|
| 开发 | DEBUG | 控制台可读 |
| 测试 | INFO | JSON |
| 生产 | INFO | JSON + 异步 |

结论先行：通过 `application-{profile}.yml` 做环境差异化，避免生产误开 DEBUG。

## 7.2 Logback JSON 配置

### 7.2.1 编码器与 MDC 字段

结论先行：只把低基数、需检索的 MDC 字段写入 JSON，避免每行日志臃肿。

```xml
<encoder class="net.logstash.logback.encoder.LogstashEncoder">
    <includeMdcKeyName>traceId</includeMdcKeyName>
    <includeMdcKeyName>tenantId</includeMdcKeyName>
    <includeMdcKeyName>userId</includeMdcKeyName>
</encoder>
```

## 7.3 生产建议

| 项 | 建议 |
|----|------|
| 异步 | 有界队列 + 丢弃策略 |
| 滚动 | 按大小 + 时间双策略 |
| 保留 | 按合规要求 |
| 采样 | 高频路径采样 |

# 8. 日志采样

## 8.1 采样必要性

### 8.1.1 成本与维度

结论先行：高频请求全量记录成本高，需按比例采样；但错误与慢请求必须全量。

| 维度 | 说明 |
|------|------|
| 概率 | 1% 随机 |
| 状态 | 错误全采 |
| 耗时 | 慢请求全采 |
| 租户 | 大客户全采 |

## 8.2 采样实现

### 8.2.1 概率采样与错误全采

```java
private final ThreadLocalRandom random = ThreadLocalRandom.current();

boolean shouldSample() {
    return random.nextDouble() < 0.01;   // 1%
}

// 正常请求采样，错误全记
if (log.isInfoEnabled() && (isError || shouldSample())) {
    log.info(...);
}
```

## 8.3 边界与坑

结论先行：采样必须"整条链路一致"，否则同一请求在不同服务有的记、有的不记。

生产建议：用 traceId 哈希做采样决策，保证同一 trace 在所有服务的采样结果一致。

# 9. 日志与 Trace 关联

## 9.1 关联机制

### 9.1.1 traceId 贯穿与互跳

结论先行：日志平台与链路平台通过 traceId 建立双向跳转，缩短排查路径。

```java
// 日志中带 traceId，可在日志平台与 Trace 互跳
log.info("llm.request traceId={} ...", MDC.get("traceId"));
```

```
日志平台（ELK） --traceId--> Trace 平台（Jaeger/Langfuse）
```

## 9.2 与指标关联

### 9.2.1 exemplar 与联合排查

结论先行：指标的 exemplar（样本）可携带 traceId，从异常指标点直接跳到具体请求链路。

| 步骤 | 工具 | 目的 |
|------|------|------|
| 1 | 指标 | 发现异常 |
| 2 | 链路 | 定位慢 Span |
| 3 | 日志 | 查看细节 |

## 9.3 边界

结论先行：日志与 Trace 的 ID 必须在同一处生成（请求入口），否则无法关联。

# 10. 生产注意事项

## 10.1 风险清单

### 10.1.1 安全与性能风险

| 风险 | 建议 |
|------|------|
| 记录完整 Prompt | 脱敏 + 截断 |
| 记录 API Key | 正则过滤 |
| MDC 未清理 | finally 清理 |

| 风险 | 建议 |
|------|------|
| 日志级别过高 | 生产 INFO |
| 全量记录 | 采样 |
| 同步 IO | 异步 Appender |

## 10.2 落地建议

### 10.2.1 规范与审查

结论先行：把字段字典、脱敏规则、级别约定写成文档并代码化；在 CI 中扫描硬编码敏感信息与直接打印日志的写法。

## 10.3 检查表

| 检查项 | 通过标准 |
|--------|----------|
| 结构化 | JSON 字段 |
| traceId | MDC 关联 |
| 脱敏 | 覆盖四类 |
| 采样 | 有策略 |
| 级别 | 生产 INFO |

# 11. 小结

## 11.1 核心要点

- LLM 日志需记录"模型看到/生成了什么"，但必须脱敏。
- 结构化日志 + MDC traceId 便于检索与关联。
- 分级：生产 INFO，完整内容仅 DEBUG。
- 脱敏覆盖手机号、身份证、邮箱、密钥。
- 日志与 Trace 通过 traceId 关联。

## 11.2 落地检查清单

| 项 | 状态 |
|----|------|
| 字段字典 | 已定义 |
| 脱敏器 | 有测试 |
| MDC 清理 | finally |
| 采样策略 | 已配置 |
| 告警联动 | 已接入 |
