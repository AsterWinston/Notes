# 1. 什么是护栏

**结论先行**：护栏（Guardrail，护栏）是在 AI 链路关键节点插入的校验与拦截，用确定性的规则约束不确定的模型输出，保证系统不越界。

## 1.1 定义与位置

护栏不是模型能力的一部分，而是模型外部的"检查站"。它在模型前、模型后以及工具调用处拦截不合规的输入、输出与行为。

```text
输入 --> [输入护栏] --> 模型 --> [输出护栏] --> 用户
              |                       |
          拦截注入                拦截有害内容
              |                       |
              +------ [行为护栏] ------+
```

### 1.1.1 护栏与模型对齐的区别

模型对齐（Alignment，对齐）在训练阶段让模型"倾向于安全"，护栏在推理阶段"强制安全"。对齐可被越狱（Jailbreak，越狱）绕过，护栏是确定性兜底，二者互补而非替代。

### 1.1.2 护栏的作用范围

护栏只保护"经过它的链路"。任何绕过护栏的旁路调用（如内部脚本直连模型、调试接口）都不受保护，因此必须收敛模型访问入口。

## 1.2 护栏的四个原则

| 原则 | 说明 |
|------|------|
| 确定性 | 护栏本身不能依赖模型判断，否则可被绕过 |
| 分层 | 输入、提示、工具、输出、行为多层设防 |
| 可观测 | 每次拦截都要可统计、可回溯 |
| 不侵入 | 用 Advisor/切面实现，业务代码无感 |

**边界与坑**：把护栏写成"再问一次模型是否安全"会导致双重幻觉，攻击者可用提示注入同时骗过模型和护栏。

**生产建议**：确定性规则（正则、白名单、预算）承担主拦截，模型审核仅作为补充信号，且其结论不可单独作为放行依据。

# 2. 护栏的类型

**结论先行**：护栏按所处链路位置分为五类，覆盖从输入到行为的完整路径。

## 2.1 五类护栏

| 类型 | 位置 | 作用 |
|------|------|------|
| 输入护栏 | 模型前 | 注入检测、长度限制、敏感词 |
| 提示护栏 | 提示中 | 规则声明、边界约束 |
| 工具护栏 | 工具调用 | 鉴权、参数校验、白名单 |
| 输出护栏 | 模型后 | 内容审核、格式校验、脱敏 |
| 行为护栏 | 全链路 | 步数、成本、频率限制 |

## 2.2 类型对比

| 类型 | 拦截时机 | 失败代价 | 典型实现 |
|------|----------|----------|----------|
| 输入护栏 | 调用前 | 低（省成本） | 前置校验器 |
| 提示护栏 | 组装时 | 低 | 模板约束 |
| 工具护栏 | 执行前 | 中 | 鉴权切面 |
| 输出护栏 | 返回前 | 高（已耗成本） | 后置校验器 |
| 行为护栏 | 全程 | 中 | 计数器/预算 |

**生产建议**：越靠前拦截成本越低，因此输入护栏要尽可能严格；输出护栏承担"最后一道防线"，即使输入已被过滤也必须保留。

# 3. 输入护栏

**结论先行**：输入护栏在成本最低的位置拦截恶意输入，重点是长度、注入与敏感词三个维度。

## 3.1 校验维度

| 维度 | 目的 | 手段 |
|------|------|------|
| 长度 | 防超长攻击与成本攻击 | 字符/Token 上限 |
| 空值 | 防无意义调用 | 非空校验 |
| 注入 | 防指令覆盖 | 模式匹配 |
| 敏感词 | 防违规内容 | 词库匹配 |

## 3.2 实现

```java
public class InputGuardrail {

    private static final int MAX_LENGTH = 4000;

    public GuardrailResult validate(String input) {
        // 长度限制（防超长攻击与成本攻击）
        if (input.length() > MAX_LENGTH) {
            return GuardrailResult.reject("输入过长，请精简后重试");
        }
        // 空输入
        if (input.isBlank()) {
            return GuardrailResult.reject("输入不能为空");
        }
        // 注入检测
        if (injectionDetector.isSuspicious(input)) {
            return GuardrailResult.reject("输入包含可疑内容");
        }
        // 敏感词
        if (sensitiveWordFilter.contains(input)) {
            return GuardrailResult.reject("输入包含敏感内容");
        }
        return GuardrailResult.pass();
    }
}
```

### 3.2.1 注入检测实现

```java
public class InjectionDetector {

    private static final List<Pattern> SUSPICIOUS = List.of(
            Pattern.compile("ignore\\s+(all\\s+)?(previous|above)", Pattern.CASE_INSENSITIVE),
            Pattern.compile("忽略(以上|之前|上述)(所有)?(指令|规则)"),
            Pattern.compile("reveal\\s+(your\\s+)?system\\s+prompt", Pattern.CASE_INSENSITIVE),
            Pattern.compile("</?system>", Pattern.CASE_INSENSITIVE)
    );

    public boolean isSuspicious(String input) {
        String normalized = normalize(input);      // 去零宽、统一全半角
        return SUSPICIOUS.stream()
                .anyMatch(p -> p.matcher(normalized).find());
    }

    private String normalize(String text) {
        return text.replaceAll("[\\u200B-\\u200D\\uFEFF]", "")
                   .toLowerCase();
    }
}
```

## 3.3 边界与坑

**边界与坑**：

- 只按字符数限制长度会漏掉 Token 膨胀（如大量罕见字），需同时限制 Token。
- 注入检测正则可被同音字、零宽字符、Base64 绕过，需要先做归一化（去零宽、统一全半角）。
- 敏感词命中即拒绝会误伤正常讨论（如安全研究），需配合上下文或分级处理。

**生产建议**：对输入做"归一化 → 模式匹配 → 分级处理"三步走；把命中样本落库，用于持续优化词库与规则。

# 4. 输出护栏

**结论先行**：输出护栏是最后一道防线，覆盖内容合规、提示泄露、引用校验、格式与脱敏五个维度。

## 4.1 校验维度

| 维度 | 检查内容 |
|------|----------|
| 内容审核 | 有害、违法、歧视 |
| 提示泄露 | 是否包含系统提示原文 |
| 引用校验 | RAG 引用是否真实存在 |
| 格式校验 | JSON/字段是否符合约定 |
| 脱敏 | 输出中的 PII 是否已处理 |

## 4.2 实现

```java
public class OutputGuardrail {

    public GuardrailResult validate(String output, String context) {
        // ① 内容审核
        ModerationResponse mod = moderationModel.call(
                new ModerationPrompt(output));
        if (mod.getResult().getOutput().isFlagged()) {
            return GuardrailResult.reject("生成内容不合规");
        }

        // ② 系统提示泄露检测
        if (containsSystemPrompt(output)) {
            return GuardrailResult.reject("检测到提示泄露");
        }

        // ③ 引用校验（RAG）
        if (hasCitations(output) && !citationsValid(output, context)) {
            return GuardrailResult.reject("引用来源不存在");
        }

        // ④ 格式校验
        if (!formatValid(output)) {
            return GuardrailResult.reject("输出格式异常");
        }

        // ⑤ 脱敏
        return GuardrailResult.pass(piiMasker.mask(output));
    }
}
```

### 4.2.1 系统提示泄露检测

检测输出是否包含系统提示的显著片段（n-gram 重合度），比逐字比对更能抓住改写后的泄露。

```java
public boolean containsSystemPrompt(String output) {
    for (String sentinel : SYSTEM_PROMPT_SENTINELS) {
        if (output.contains(sentinel)) {
            return true;
        }
    }
    // 兜底：计算与系统提示的 n-gram 重合率
    return ngramOverlap(output, SYSTEM_PROMPT) > 0.6;
}
```

## 4.3 边界与坑

**边界与坑**：输出护栏在模型已经消耗成本后才执行，无法挽回算力损失；审核模型本身也可能被绕过，不能作为唯一防线。

**生产建议**：输出护栏失败时不要直接抛异常给用户，应返回友好的兜底话术并记录 traceId，方便定位。

# 5. 行为护栏

**结论先行**：行为护栏约束 Agent 的"资源使用"，防止无限循环、成本失控与拒绝服务。

## 5.1 预算模型

| 预算 | 含义 | 超限处理 |
|------|------|----------|
| 步数 | 工具调用轮数 | 终止 |
| Token | 累计消耗 | 终止 |
| 时长 | 墙钟时间 | 终止 |
| 频率 | 单位时间请求数 | 限流 |

## 5.2 实现

```java
public class BehaviorGuardrail {

    private final int maxSteps;
    private final long maxTokens;
    private final Duration maxDuration;
    private final Instant start = Instant.now();

    public GuardrailResult check(int steps, long tokensUsed) {
        if (steps > maxSteps) {
            return GuardrailResult.reject("执行步数超限");
        }
        if (tokensUsed > maxTokens) {
            return GuardrailResult.reject("Token 预算耗尽");
        }
        if (Duration.between(start, Instant.now()).compareTo(maxDuration) > 0) {
            return GuardrailResult.reject("执行超时");
        }
        return GuardrailResult.pass();
    }
}
```

### 5.2.1 会话级预算上下文

预算状态应存放在单次会话上下文中，而非护栏对象字段。

```java
public record BudgetContext(int maxSteps, long maxTokens,
                            Duration maxDuration, Instant start) {

    public static BudgetContext of(int steps, long tokens, Duration duration) {
        return new BudgetContext(steps, tokens, duration, Instant.now());
    }
}

public GuardrailResult check(BudgetContext ctx, int steps, long tokens) {
    if (steps > ctx.maxSteps()) {
        return GuardrailResult.reject("执行步数超限");
    }
    if (tokens > ctx.maxTokens()) {
        return GuardrailResult.reject("Token 预算耗尽");
    }
    return GuardrailResult.pass();
}
```

**边界与坑**：`start` 是对象创建时间，若护栏对象被复用（单例），计时会从首次创建开始，导致误判超时。应把计时起点绑定到单次会话上下文。

**生产建议**：预算应可配置、可按租户区分；超限时保留已产生的部分结果并给出明确原因，而不是静默失败。

# 6. 工具护栏

**结论先行**：工具调用是 Agent 最危险的出口，必须在执行前完成鉴权与参数校验。

## 6.1 鉴权与白名单

```java
public class ToolGuardrail {

    private static final Set<String> ALLOWED_TOOLS =
            Set.of("search", "calculator", "queryOrder");

    public void beforeCall(String toolName, Map<String, Object> args,
                           UserContext user) {
        if (!ALLOWED_TOOLS.contains(toolName)) {
            throw new SecurityException("工具不在白名单: " + toolName);
        }
        if (!user.hasPermission(toolName)) {
            throw new SecurityException("无权限调用: " + toolName);
        }
    }
}
```

## 6.2 参数校验

**边界与坑**：模型生成的参数不可信，必须做类型、范围、格式校验；尤其要防止把用户输入直接拼进 SQL、命令或 URL。

**生产建议**：工具内部再做一次最小权限校验，不依赖上层护栏；对写操作要求二次确认或人工审批。

# 7. 在 Spring AI 中实现护栏 Advisor

**结论先行**：Spring AI 的 Advisor 机制允许在调用链上插入横切逻辑，是护栏的天然落点。

## 7.1 Advisor 原理

Advisor 通过 `adviseCall` 包裹真实调用，`chain.nextCall(request)` 前后分别对应"输入前"与"输出后"，因此一个 Advisor 即可同时实现输入与输出护栏。

## 7.2 完整实现

```java
public class GuardrailAdvisor implements CallAdvisor {

    private final InputGuardrail inputGuardrail;
    private final OutputGuardrail outputGuardrail;

    @Override
    public ChatClientResponse adviseCall(ChatClientRequest request,
                                         CallAdvisorChain chain) {
        // 输入护栏
        String userText = extractUserText(request);
        GuardrailResult inputCheck = inputGuardrail.validate(userText);
        if (!inputCheck.passed()) {
            return rejectedResponse(inputCheck.reason());
        }

        // 调用
        ChatClientResponse response = chain.nextCall(request);

        // 输出护栏
        String output = response.chatResponse().getResult().getOutput().getText();
        GuardrailResult outputCheck = outputGuardrail.validate(output, "");
        if (!outputCheck.passed()) {
            return rejectedResponse(outputCheck.reason());
        }
        return response;
    }

    @Override
    public String getName() { return "GuardrailAdvisor"; }

    @Override
    public int getOrder() { return 50; }     // 安全校验靠前
}
```

### 7.2.1 拆分为两个 Advisor

把输入与输出护栏拆开，可独立开关、独立观测；输入护栏 order 更小（更靠前），输出护栏 order 更大（更靠后）。

```java
@Component
@Order(10)
public class InputGuardrailAdvisor implements CallAdvisor { /* ... */ }

@Component
@Order(90)
public class OutputGuardrailAdvisor implements CallAdvisor { /* ... */ }
```

**边界与坑**：`getOrder()` 决定执行顺序，安全护栏必须排在日志、缓存等其他 Advisor 之前，否则敏感数据可能先被日志记录。

**生产建议**：把输入护栏与输出护栏拆成两个 Advisor，便于独立开关与观测；用 `@Order` 或 `getOrder()` 明确排序。

# 8. 在 LangChain4j 中实现

**结论先行**：LangChain4j 没有统一的 Advisor 抽象，护栏通常在服务层或自定义 `ChatLanguageModel` 装饰器中实现。

## 8.1 服务封装

```java
public class GuardrailService {

    public String chat(String input) {
        // 输入护栏
        GuardrailResult check = inputGuardrail.validate(input);
        if (!check.passed()) return check.reason();

        // 调用
        String output = assistant.chat(input);

        // 输出护栏
        GuardrailResult outCheck = outputGuardrail.validate(output, "");
        return outCheck.passed() ? output : outCheck.reason();
    }
}
```

## 8.2 与 Spring AI 的差异

| 维度 | Spring AI | LangChain4j |
|------|-----------|-------------|
| 横切机制 | Advisor | 装饰器/服务层 |
| 顺序控制 | getOrder | 手动编排 |
| 侵入性 | 低 | 中 |

**生产建议**：无论用哪种框架，都应把护栏逻辑封装在单一入口，避免在多个业务方法里重复编写校验。

# 9. 敏感词过滤

**结论先行**：敏感词过滤要用多模式匹配算法，逐词 `contains` 在大词库下会拖垮性能。

## 9.1 匹配算法

| 算法 | 复杂度 | 适用 |
|------|--------|------|
| 逐个 contains | O(n*m) | 小词库 |
| Trie（前缀树） | O(n) | 中等词库 |
| AC 自动机 | O(n) | 大词库、多模式 |

## 9.2 实现

```java
@Component
public class SensitiveWordFilter {

    private final Set<String> words;
    private final AhoCorasickTrie trie;      // 高效多模式匹配

    public boolean contains(String text) {
        return trie.search(text).isNotEmpty();
    }

    public String replace(String text) {
        return trie.replaceAll(text, "***");
    }
}
```

### 9.2.1 AC 自动机要点

AC 自动机（Aho-Corasick，多模式匹配）在 Trie 基础上增加失败指针，一次扫描即可匹配全部模式；构建成本 O(总词长)，查询成本 O(文本长 + 命中数)，适合大词库。

**边界与坑**：攻击者用谐音、拆字、插入符号绕过词库；词库陈旧会持续漏拦。

**生产建议**：词库要定期更新，并做归一化预处理；命中日志用于挖掘变体。

# 10. 护栏的取舍

**结论先行**：护栏没有"最优解"，只有与业务风险匹配的取舍。

## 10.1 取舍维度

| 取舍 | 说明 |
|------|------|
| 严格 vs 宽松 | 严格更安全但误拦多 |
| 同步 vs 异步 | 同步增加延迟，异步有窗口 |
| 本地 vs API | 本地快，API 准 |
| 拦截 vs 改写 | 拦截安全，改写体验好 |

## 10.2 分级处理

```java
// 分级处理：不是所有违规都直接拒绝
public GuardrailResult handle(String output) {
    if (isSevere(output)) {
        return GuardrailResult.reject("无法回答");       // 严重 → 拒绝
    }
    if (isMild(output)) {
        return GuardrailResult.pass(sensitiveFilter.replace(output)); // 轻微 → 改写
    }
    return GuardrailResult.pass(output);
}
```

### 10.2.1 阈值定义

| 等级 | 判定 | 处理 |
|------|------|------|
| 严重 | 命中高危类别或高风险实体 | 拒绝 |
| 轻微 | 命中低危词但语境正常 | 改写 |
| 正常 | 无命中 | 放行 |

**生产建议**：为"严重/轻微"定义明确阈值，并保留人工复核通道；改写后的内容也要再走一次输出护栏。

# 11. 护栏的可观测

**结论先行**：没有观测的护栏无法判断其是否有效，更无法优化误拦率。

## 11.1 指标

```java
registry.counter("guardrail.triggered", "type", type, "action", action)
        .increment();
log.warn("护栏触发 type={} action={} traceId={}",
        type, action, MDC.get("traceId"));
```

| 指标 | 用途 |
|------|------|
| 触发次数 | 按类型统计 |
| 误拦率 | 通过人工复核统计 |
| 拦截原因分布 | 指导词库优化 |

## 11.2 告警与调优

**边界与坑**：只统计触发次数会掩盖"该拦未拦"的漏拦问题，必须同时统计漏拦（通过红队样本回放）。

**生产建议**：把 traceId 与拦截原因关联，支持按会话回放；对误拦率设置告警阈值，超过则触发规则复审。

# 12. 生产注意事项

**结论先行**：护栏的常见故障是"缺失"和"误伤"，两者都要在工程上兜底。

## 12.1 风险与建议

| 风险 | 建议 |
|------|------|
| 只做输入护栏 | 输出也要护栏 |
| 无长度限制 | 必加 |
| 护栏抛异常 | 内部容错，降级放行/拒绝 |
| 误拦率高 | 分级处理 + 人工复核 |
| 无监控 | 监控触发与误拦 |
| 词库陈旧 | 定期更新 |

## 12.2 上线检查清单

- 五类护栏是否都已覆盖。
- 护栏自身异常是否有兜底（fail-open 还是 fail-close）。
- 安全 Advisor 是否排在日志 Advisor 之前。
- 是否具备红队样本回放能力。
- 误拦率是否可观测、可告警。

**边界与坑**：`fail-open`（出错放行）可用性高但有安全风险，`fail-close`（出错拒绝）安全但影响体验，必须按业务场景显式选择。

# 13. 小结

- 护栏覆盖输入、提示、工具、输出、行为五个层面。
- 输入护栏防注入与超长，输出护栏防有害与泄露。
- 用 Advisor/切面实现，不侵入业务。
- 分级处理：严重拒绝，轻微改写。
- 护栏必须可观测，并持续优化误拦率。
- 越靠前拦截成本越低，但最后一道防线不可省。
