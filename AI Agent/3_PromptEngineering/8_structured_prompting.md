# 1. 为什么需要结构化输出

**结论先行**：LLM 的原生输出是自然语言，而程序的输入必须是有类型、可校验的数据；
结构化输出就是两者之间的"接口契约"。

## 1.1 一句话定义

```
自然语言输出 -> 难以可靠解析
结构化输出   -> 可直接被程序消费
```

## 1.2 自然语言为什么不可靠

### 1.2.1 解析不可靠的根因

| 根因 | 表现 |
|------|------|
| 表述自由 | 同一含义有多种写法 |
| 上下文依赖 | 省略主语、指代不清 |
| 格式漂移 | 这次是列表，下次是段落 |
| 幻觉字段 | 编造不存在的字段 |

### 1.2.2 程序消费的前提

| 前提 | 说明 |
|------|------|
| 类型确定 | 字符串 / 整数 / 布尔 / 数组 |
| 结构固定 | 字段名与层级稳定 |
| 可校验 | 能对值域与格式做约束 |
| 可失败 | 解析失败能检测并兜底 |

**边界与坑**：用正则去"抠"自然语言里的字段，看似可行，实则在模型换一版后立刻失效。

**生产建议**：把"能结构化"作为硬要求写进接口设计，而不是事后用字符串处理补救。

## 1.3 与 Agent 可靠性的关系

**结论先行**：Agent 的每一步（工具调用、状态更新、路由决策）都依赖可解析输出。

> 结构化输出是 Agent 可靠性的前提。详见 `4_LLMEngineering/4_structured_output.md`。

**生产建议**：把结构化输出视为内部 API 契约，一旦变更需版本化并做回归测试。

# 2. 三种约束方式

**结论先行**：可靠性由弱到强依次是提示约束、JSON Mode、JSON Schema。

## 2.1 提示约束

| 方式 | 可靠性 | 说明 |
|------|:------:|------|
| 提示约束 | 低 | 靠提示要求格式 |
| JSON Mode | 中 | 保证合法 JSON |
| JSON Schema | 高 | 强约束结构 |

## 2.2 JSON Mode

**结论先行**：JSON Mode 由服务端保证输出是合法 JSON，但不保证字段符合你的结构。

- 优点：不需要解析容错，不会返回 Markdown 包裹。
- 缺点：字段名、字段类型、必填项仍可能漂移。
- 适用：结构简单、字段少、能接受后校验的场景。

## 2.3 JSON Schema

**结论先行**：JSON Schema 在解码阶段就约束了字段、类型与必填项，是最强方式。

| 约束项 | JSON Mode | JSON Schema |
|--------|:---------:|:-----------:|
| 合法 JSON | 是 | 是 |
| 字段名固定 | 否 | 是 |
| 类型固定 | 否 | 是 |
| 必填项 | 否 | 是 |
| 枚举值 | 否 | 是 |
| 嵌套结构 | 否 | 是 |

## 2.4 选型建议

### 2.4.1 决策路径

```
是否需要程序解析？
  否 -> 用自然语言即可
  是 -> 字段是否多、是否有嵌套/枚举？
          否 -> JSON Mode
          是 -> JSON Schema（类型映射）
```

### 2.4.2 选型对比

| 场景 | 推荐方式 | 理由 |
|------|----------|------|
| 简单情感分类 | 枚举 + 类型映射 | 稳定且便宜 |
| 多字段抽取 | JSON Schema | 强约束 |
| 流式展示 | 提示约束 | 结构简单、需增量 |
| 工具调用参数 | JSON Schema | 必须精确 |

**生产建议**：默认用类型映射（框架生成 Schema），仅在框架不支持时退回提示约束。

# 3. 提示约束写法

**结论先行**：提示约束最灵活，但可靠性最低，需要大量技巧补强。

## 3.1 系统提示约束

```java
@SystemMessage("""
        严格按以下 JSON 格式输出，不要输出任何其他文字：

        {
          "summary": "一句话总结",
          "tags": ["标签1", "标签2"],
          "sentiment": "POSITIVE"
        }
        """)
```

**边界与坑**：提示约束易出现加解释、加代码块、字段漂移。

**生产建议**：在提示中明确"只输出 JSON，不要 Markdown 代码块，不要解释"。

## 3.2 字段约束技巧

### 3.2.1 类型与范围

```java
@UserMessage("""
        抽取信息，字段要求：
        - name：字符串，人名
        - age：整数，0~150
        - job：字符串，职业；未提及填 null
        - tags：字符串数组，最多 5 个

        输出 JSON：
        {"name":"","age":0,"job":null,"tags":[]}

        文本：{{it}}
        """)
```

### 3.2.2 约束技巧清单

| 技巧 | 说明 |
|------|------|
| 明确类型 | 字符串 / 整数 / 数组 |
| 明确范围 | 0~150 |
| 明确缺失值 | 未提及填 null |
| 明确数量 | 最多 5 个 |
| 给空模板 | 便于模型照填 |
| 给正例 | 一个填好的样例 |
| 禁用补充 | 不要额外字段 |

**生产建议**：把"空模板 + 一个正例"组合使用，比只描述字段更稳定。

## 3.3 枚举约束

**结论先行**：枚举把开放生成变成受限选择，显著提升分类任务的稳定性。

```java
public enum Sentiment { POSITIVE, NEUTRAL, NEGATIVE }

@UserMessage("""
        判断情感，只能从以下值中选择：
        POSITIVE / NEUTRAL / NEGATIVE

        文本：{{it}}
        情感：
        """)
Sentiment analyze(String text);
```

| 方式 | 稳定性 | 说明 |
|------|:------:|------|
| 自由文本 | 低 | 可能返回"偏正面" |
| 枚举提示 | 中 | 仍可能拼错 |
| 枚举类型映射 | 高 | 解码期约束 |

## 3.4 提示约束的坑

**结论先行**：提示约束无法在解码阶段兜底，任何格式异常都要靠后处理。

| 坑 | 示例 | 对策 |
|----|------|------|
| Markdown 包裹 | ```json ... ``` | 剥离围栏 |
| 前后解释 | "好的，结果是：" | 抽取首个 JSON |
| 尾随逗号 | `{"a":1,}` | 宽松解析 |
| 单引号 | `{'a':1}` | 归一化引号 |
| 中文引号 | `{"a"："x"}` | 替换全角冒号 |

**生产建议**：后处理容错可以缓解，但不要把它当主方案；能用 Schema 就用 Schema。

## 3.5 提示约束的容错后处理

**结论先行**：当无法使用 Schema 时，后处理是最后一道防线，但只能"减少"而非"消除"失败。

```java
public static String extractJson(String raw) {
    String text = raw.trim();
    // 剥离 Markdown 代码围栏
    if (text.startsWith("```")) {
        int start = text.indexOf('\n');
        int end = text.lastIndexOf("```");
        if (start > 0 && end > start) {
            text = text.substring(start + 1, end).trim();
        }
    }
    // 截取第一个完整 JSON 对象
    int begin = text.indexOf('{');
    int last = text.lastIndexOf('}');
    if (begin >= 0 && last > begin) {
        text = text.substring(begin, last + 1);
    }
    return text;
}
```

| 后处理步骤 | 目的 |
|------------|------|
| 剥离代码围栏 | 去掉 Markdown 包裹 |
| 截取首尾括号 | 去掉前后解释 |
| 全角转半角 | 修复中文标点 |
| 去尾随逗号 | 修复非法 JSON |
| 宽松解析 | 用容错解析器兜底 |

**边界与坑**：后处理规则会随模型版本变化而失效，维护成本高且难以测试。

**生产建议**：把后处理封装为纯函数并单测；一旦框架支持 Schema，立即替换掉它。

# 4. 类型映射

**结论先行**：类型映射由框架自动生成 Schema 并把输出反序列化为对象，是首选方案。

## 4.1 为什么推荐类型映射

| 维度 | 提示约束 | 类型映射 |
|------|:--------:|:--------:|
| Schema 生成 | 手写 | 自动 |
| 解析 | 手工 | 自动 |
| 类型安全 | 无 | 编译期 + 运行期 |
| 维护成本 | 高 | 低 |
| 可靠性 | 低 | 高 |

> 类型映射自动生成 Schema 并解析，比提示约束可靠得多。

## 4.2 Spring AI 风格

```java
public record Analysis(String summary, List<String> tags, Sentiment sentiment) {}
```

```java
Analysis result = chatClient.prompt()
        .user("分析：" + text)
        .call()
        .entity(Analysis.class);
```

**边界与坑**：`record` 的字段顺序会被用作提示线索，但不构成语义约束。

**生产建议**：为每个业务对象单独定义 record，避免用 `Map<String, Object>` 这类弱类型。

## 4.3 LangChain4j 风格

### 4.3.1 声明式接口

```java
interface Analyzer {
    Analysis analyze(String text);
}

// 装配
Analyzer analyzer = AiServices.create(Analyzer.class, chatModel);
```

### 4.3.2 手动 Schema

**结论先行**：当框架不支持自动 Schema 时，可手写 JSON Schema 交给模型。

```java
// 伪代码：部分模型提供 responseFormat 参数
ChatRequest request = ChatRequest.builder()
        .responseFormat(ResponseFormat.jsonSchema("""
                {
                  "type": "object",
                  "properties": {
                    "summary": { "type": "string" },
                    "tags": { "type": "array", "items": { "type": "string" } }
                  },
                  "required": ["summary", "tags"]
                }
                """))
        .build();
```

**生产建议**：Schema 与 Java 类型应保持单一来源，避免两处手写导致漂移。

# 5. 进阶模式

**结论先行**：推理分离、批量处理、嵌套结构是三个高频进阶需求。

## 5.1 推理与结果分离

**结论先行**：把"推理过程"与"最终结论"分成两个字段，兼顾可解释与可消费。

```java
public record Analysis(
        String reasoning,        // 推理过程
        String conclusion,       // 结论
        double confidence        // 置信度
) {}
```

```java
// 提示
"""
先推理，再给结论。输出 JSON：
{"reasoning":"...","conclusion":"...","confidence":0.0}
"""
```

> 推理与结论分离，便于程序只消费结论，同时保留可解释性。

**边界与坑**：`reasoning` 字段会显著增加输出 token，长文本场景要评估成本。

**生产建议**：对延迟敏感的场景，可只在调试环境开启 `reasoning`。

## 5.2 批量结构化输出

**结论先行**：一次调用处理多条，减少系统提示重复开销。

```java
public record BatchResult(List<Item> items) {
    public record Item(String id, String category, double score) {}
}

BatchResult result = chatClient.prompt()
        .user("""
                对以下条目分类，输出 JSON：
                {"items":[{"id":"","category":"","score":0.0}]}

                条目：
                %s
                """.formatted(formatItems(items)))
        .call()
        .entity(BatchResult.class);
```

**边界与坑**：批量过大时模型会漏项或错位，`id` 与内容对应关系容易错乱。

**生产建议**：每批控制在 20~50 条；在提示中要求回填 `id`；对返回数量做校验。

## 5.3 嵌套与可选字段

### 5.3.1 嵌套对象

```java
public record Order(
        String orderId,
        Customer customer,
        List<LineItem> items) {

    public record Customer(String name, String phone) {}

    public record LineItem(String sku, int qty, double price) {}
}
```

### 5.3.2 可选与联合类型

| 类型 | Java 表达 | 说明 |
|------|-----------|------|
| 可选 | `Optional<String>` 或可空 | 未提及填 null |
| 联合 | 独立字段 + 类型标记 | 避免 `Object` |
| 枚举 | `enum` | 受限取值 |
| 多态 | `type` + 子对象 | 便于反序列化 |

**生产建议**：尽量避免联合类型，改用"类型字段 + 多个可选子对象"，降低解析失败率。

### 5.3.3 数组与分页

**结论先行**：当结果条数不定时，应让模型同时返回"是否还有更多"。

```java
public record PagedResult(
        List<Item> items,
        boolean hasMore,
        String nextCursor) {

    public record Item(String id, String content) {}
}
```

**边界与坑**：让模型生成游标本身不可靠，游标应由服务端维护，模型只负责分页内内容。

**生产建议**：分页游标交给后端，模型输出只保留 `items` 与 `hasMore`。

# 6. 校验与兜底

**结论先行**：类型映射只保证"能解析"，不保证"值合法"，业务校验必须独立进行。

## 6.1 Bean Validation 校验

```java
public record UserProfile(
        @NotBlank String name,
        @Min(0) @Max(150) int age,
        @Email String email) {}

UserProfile profile = extractor.extract(text);

Set<ConstraintViolation<UserProfile>> violations = validator.validate(profile);
if (!violations.isEmpty()) {
    log.warn("校验失败: {}", violations);
    // 重试或兜底
}
```

## 6.2 解析失败重试

**结论先行**：解析失败往往是偶发的格式问题，带退避的重试即可显著降低失败率。

```java
public <T> T safeEntity(String prompt, Class<T> type, T fallback) {
    for (int i = 0; i < 3; i++) {
        try {
            return chatClient.prompt().user(prompt).call().entity(type);
        } catch (Exception e) {
            log.warn("解析失败，第 {} 次重试", i + 1, e);
        }
    }
    return fallback;
}
```

## 6.3 兜底与降级

| 策略 | 说明 | 适用 |
|------|------|------|
| 返回默认值 | 用 fallback 对象 | 非关键字段 |
| 降级为文本 | 只返回原文 | 展示类 |
| 转人工 | 进入人工队列 | 高价值 |
| 拒绝处理 | 返回错误码 | 强一致 |

**生产建议**：为每个结构化接口定义明确的兜底策略，并监控兜底触发率。

# 7. 边界与反例

**结论先行**：结构化输出失败多源于"约束过弱"或"约束与模型能力不匹配"。

## 7.1 常见失败模式

| 失败模式 | 表现 | 对策 |
|----------|------|------|
| 输出被截断 | JSON 不完整 | 提高 maxTokens |
| 字段缺失 | 少了必填字段 | 用 Schema 必填约束 |
| 类型错误 | 数字被写成字符串 | 强 Schema + 后校验 |
| 多余字段 | 自创字段 | Schema 禁止额外属性 |
| 中英混排 | 枚举值写中文 | 明确枚举字面量 |

## 7.2 反例

```java
// 反例：只用一句"输出 JSON"，无结构、无类型
String raw = model.chat("提取姓名和年龄，输出 JSON：" + text);

// 正例：类型化 + 校验
public record Person(@NotBlank String name, @Min(0) @Max(150) int age) {}
Person p = chatClient.prompt().user(text).call().entity(Person.class);
```

## 7.3 模型能力差异

**结论先行**：不同模型对 Schema 的遵循度差异很大，需按模型做回归。

| 模型能力 | 影响 | 建议 |
|----------|------|------|
| 强遵循 | 可直接用 Schema | 优先 |
| 中等 | 偶发漂移 | Schema + 校验 + 重试 |
| 弱 | 需提示补强 | 提示 + 后处理 |

**生产建议**：换模型或换版本后，必须重跑结构化输出的回归测试集。

## 7.4 与 Function Calling 的关系

**结论先行**：Function Calling（函数调用）的参数结构本质也是一种结构化输出。

| 维度 | 结构化输出 | Function Calling |
|------|------------|------------------|
| 目的 | 拿到数据 | 触发动作 |
| 约束 | Schema | 函数参数 Schema |
| 多轮 | 单次为主 | 常与工具循环配合 |
| 校验 | 业务校验 | 参数校验 + 权限校验 |

**生产建议**：工具参数同样要做校验与兜底，不能因为"框架保证 Schema"就跳过。

# 8. 生产注意事项

**结论先行**：结构化输出的生产重点是"约束强度、校验兜底、可观测"。

## 8.1 风险与对策

| 风险 | 建议 |
|------|------|
| 靠提示约束格式 | 用类型映射 / Schema |
| 字段无约束 | 明确类型与范围 |
| JSON 被截断 | 提高 maxTokens |
| 无校验 | 解析后校验 |

## 8.2 可观测性

**结论先行**：记录解析失败率与兜底率，才能发现模型漂移。

```java
public record ParseMetric(
        String schemaName,
        boolean success,
        int retryCount,
        boolean fallbackUsed,
        long latencyMs) {
}
```

**生产建议**：对失败样本采样保存原始输出，便于离线分析与提示优化。

## 8.3 评测

| 指标 | 说明 |
|------|------|
| 解析成功率 | 首次解析即成功比例 |
| 字段准确率 | 字段值正确的比例 |
| 重试率 | 需要重试的比例 |
| 兜底率 | 触发 fallback 的比例 |
| 延迟 P95 | 端到端耗时 |

**生产建议**：以"解析成功率 × 字段准确率"作为发布门槛，低于阈值阻断上线。

# 9. 小结

## 9.1 要点回顾

- 结构化输出是 Agent 可靠性的前提。
- 三种方式：提示约束、JSON Mode、Schema。
- 优先用类型映射（框架自动处理 Schema 与解析）。

## 9.2 落地清单

- 枚举、类型化字段、范围约束提升稳定性。
- 批量处理省 token，但要校验条数与 id 对应。
- 必须校验与兜底，并监控解析成功率与兜底率。
