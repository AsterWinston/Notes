# 1. 失败定位三步法

```
① 复现：能否用相同输入稳定复现？
② 定位：失败在检索还是生成？
③ 归因：属于哪一类失败？
```

RAG 的失败往往不是"模型不行"这么简单。同样的错误答案，可能来自文档没入库、分块切断、召回不足，也可能来自提示未约束、上下文过长。**三步法的价值，是把模糊的"答错了"拆成可修复的具体环节。**

## 1.1 复现

### 1.1.1 稳定复现的意义

不能复现的失败，就无法验证修复是否有效。复现的第一目标是确认：给定完全相同的问题与配置，错误是否稳定出现。若时好时坏，问题多半在随机性（温度、采样）或数据漂移，而非链路本身。

### 1.1.2 复现记录要素

| 要素 | 说明 |
|------|------|
| 原始问题 | 用户原话，不要改写 |
| 完整配置 | 模型版本、Prompt 版本、topK、阈值 |
| 检索结果 | 命中的文档 ID、分数、片段 |
| 最终回答 | 模型原样输出 |
| 期望答案 | 正确结果或"应拒答" |

复现记录建议结构化保存，便于后续批量检索同类失败：

```java
public record FailureCase(
        String traceId,
        String question,
        String configVersion,
        List<String> hitDocIds,
        String answer,
        String expectedAnswer,
        String failureType
) {}
```

## 1.2 定位

### 1.2.1 检索还是生成

定位的唯一问题是：**正确答案有没有出现在喂给模型的上下文里？** 在，就是生成失败；不在，就是检索失败。这一步把搜索空间直接砍半。

### 1.2.2 定位工具

```
问题 ─► 检索 ─► [打印命中文档] ─► 生成 ─► [打印回答]
            │                          │
            └──── 答案在上下文？───────┘
                      │
              ┌───────┴───────┐
             否               是
              │                │
          A 类检索失败      B 类生成失败
```

## 1.3 归因

### 1.3.1 归因的层次

定位只到"检索/生成"这一层，归因要继续细到子类：检索失败是没入库、分块不当还是排序靠后？生成失败是幻觉、遗漏还是格式不符？子类决定了修复手段。

### 1.3.2 归因常见误区

- 把生成问题误判为检索问题，反复调分块却无效。
- 只看最终答案，不看中间检索结果。
- 修了个案就以为修了共性问题，未做批量分析。

# 2. 第一步：判断检索 or 生成

## 2.1 诊断代码

### 2.1.1 检索结果打印

```java
public void diagnose(String question, String expected) {
    List<Document> docs = retriever.retrieve(question);

    // 打印检索结果
    docs.forEach(d -> log.info("hit: score={} text={}",
            d.getScore(), truncate(d.getText())));

    // 答案是否在检索结果中
    boolean inContext = docs.stream()
            .anyMatch(d -> containsAnswer(d.getText(), expected));
    log.info("答案在上下文中: {}", inContext);

    String answer = generate(question, docs);
    log.info("生成: {}", answer);
}
```

### 2.1.2 判定辅助方法

```java
private boolean containsAnswer(String text, String expected) {
    if (expected == null || expected.isBlank()) {
        return false;
    }
    return text.contains(expected)
            || text.replaceAll("\\s+", "")
                   .contains(expected.replaceAll("\\s+", ""));
}
```

## 2.2 判定规则

### 2.2.1 三种结果

```
答案不在上下文 → 检索失败（A 类）
答案在上下文但答错 → 生成失败（B 类）
答案在上下文且答对 → 非失败，检查是否偶发
```

### 2.2.2 边界情况

| 情况 | 判定 | 处理 |
|------|------|------|
| 期望答案是"应拒答" | 上下文为空且模型拒答即正确 | 单独统计拒答率 |
| 答案跨多个文档 | 任一片段覆盖即可 | 检查多跳召回 |
| 期望答案表述多样 | 关键词命中即可 | 用 `mustContain` 校验 |

# 3. A 类：检索失败

## 3.1 A 类失败分类

### 3.1.1 分类总表

| 子类 | 表现 | 根因 | 对策 |
|------|------|------|------|
| A1 未入库 | 文档根本没进知识库 | 采集遗漏 | 补数据 |
| A2 解析失败 | 入库了但内容乱/空 | 解析器问题 | 换解析器/OCR |
| A3 分块不当 | 答案被切散 | 块太小/边界 | 调分块、父子分块 |
| A4 语义鸿沟 | 查询与文档表述差异大 | Embedding 弱 | 查询改写、HyDE |
| A5 专有名词 | 精确词匹配不到 | 稠密检索短板 | 加 BM25 混合 |
| A6 召回不足 | topK 太小 | 参数问题 | 提高 topK |
| A7 阈值过高 | 相关文档被过滤 | 阈值问题 | 校准阈值 |
| A8 过滤过严 | 权限/元数据过滤掉 | 过滤条件 | 检查过滤逻辑 |
| A9 排序靠后 | 在 10~50 名 | 缺重排 | 加 Rerank |

### 3.1.2 子类速记

- A1~A3 属于"数据进入检索库之前"的问题，修复要回到离线索引。
- A4~A5 属于"语义表示"问题，修复要动 Embedding 或混合检索。
- A6~A9 属于"检索参数与排序"问题，修复成本最低，优先排查。

## 3.2 检查手段

### 3.2.1 检查 A1/A2

```sql
-- 检查文档是否入库
SELECT COUNT(*) FROM vector_store WHERE metadata->>'source' = 'handbook.pdf';

-- 检查内容是否正常
SELECT content FROM vector_store WHERE metadata->>'source' = 'handbook.pdf' LIMIT 3;
```

若计数为 0，属于 A1 未入库；若计数正常但内容为乱码、空串或仅有页眉页脚，属于 A2 解析失败。

### 3.2.2 检查 A3（分块）

```java
// 找到答案在原文的位置，看分块后是否完整
String answerSentence = "费用发生后 30 天内";
documents.stream()
        .filter(d -> d.getText().contains(answerSentence))
        .findFirst()
        .ifPresentOrElse(
                d -> log.info("答案所在块完整: {}", d.getText()),
                () -> log.warn("答案被切断！需要调整分块"));
```

### 3.2.3 检查 A4~A9

| 子类 | 检查方法 |
|------|----------|
| A4 语义鸿沟 | 用同义改写重试，看是否命中 |
| A5 专有名词 | 用关键词做 BM25 检索对比 |
| A6 召回不足 | 把 topK 调到 50，看是否出现 |
| A7 阈值过高 | 把阈值降到 0，看是否出现 |
| A8 过滤过严 | 关闭元数据过滤重试 |
| A9 排序靠后 | 在完整候选列表里查排名 |

## 3.3 A 类修复优先级

### 3.3.1 优先级排序

| 优先级 | 子类 | 理由 |
|:------:|------|------|
| 1 | A6/A7/A8 | 改参数即可，成本最低 |
| 2 | A9 | 加重排，收益直接 |
| 3 | A4/A5 | 需引入混合检索或改写 |
| 4 | A3 | 需重做分块并重建索引 |
| 5 | A1/A2 | 需回到数据采集与解析 |

### 3.3.2 修复示例（提高 topK 与混合检索）

```java
public SearchRequest buildRequest(String query) {
    return SearchRequest.builder()
            .query(query)
            .topK(50)                       // A6：扩大召回
            .similarityThreshold(0.2)       // A7：放低阈值
            .hybrid(true)                   // A5：混合 BM25
            .rerankTopN(5)                  // A9：重排后取前 5
            .build();
}
```

# 4. B 类：生成失败

## 4.1 B 类失败分类

### 4.1.1 分类总表

| 子类 | 表现 | 根因 | 对策 |
|------|------|------|------|
| B1 幻觉 | 编造资料外内容 | 提示未约束 | 强化"只依据资料" |
| B2 答非所问 | 回答偏离问题 | 提示不清 | 明确任务与格式 |
| B3 遗漏 | 只用部分资料 | 上下文过长/顺序 | 精简上下文、关键前置 |
| B4 引用错误 | 来源编号对不上 | 编号管理混乱 | 结构化引用 |
| B5 过度推断 | 资料没说的也答 | 提示未禁止推断 | 明确"不足则说明" |
| B6 格式不符 | 输出格式错误 | 提示未约束 | 用结构化输出 |

### 4.1.2 子类速记

- B1、B5 是"越界"问题，核心是约束提示。
- B2、B6 是"对齐"问题，核心是任务描述与输出格式。
- B3、B4 是"组织"问题，核心是上下文与引用管理。

## 4.2 生成侧修复

### 4.2.1 强化约束提示

```java
@SystemMessage("""
        严格规则：
        1. 只依据【参考资料】回答；
        2. 不得使用你的外部知识；
        3. 资料不足时必须回答"资料中未提及"；
        4. 不得推断或编造；
        5. 每条结论标注来源编号 [n]。

        【参考资料】
        {{context}}
        """)
```

### 4.2.2 上下文治理

| 手段 | 针对子类 | 说明 |
|------|----------|------|
| 关键片段前置 | B3 遗漏 | 最相关内容放上下文开头 |
| 限制片段数量 | B3 遗漏 | 减少噪声，聚焦 |
| 结构化引用 | B4 引用 | 每段标注来源 ID |
| 输出 Schema 校验 | B6 格式 | 用 JSON Schema 强约束 |

## 4.3 生成失败的自动校验

### 4.3.1 确定性校验

除 LLM 评审外，可用确定性规则快速抓取典型生成失败：

```java
public List<String> lint(String answer, List<String> mustContain,
                         List<String> mustNotContain) {
    List<String> issues = new ArrayList<>();
    for (String s : mustContain) {
        if (!answer.contains(s)) {
            issues.add("B3 遗漏关键信息: " + s);
        }
    }
    for (String s : mustNotContain) {
        if (answer.contains(s)) {
            issues.add("B1 出现禁用表述: " + s);
        }
    }
    return issues;
}
```

### 4.3.2 引用校验

| 校验项 | 方法 | 对应子类 |
|--------|------|----------|
| 引用编号存在 | 解析 `[n]` 并核对来源列表 | B4 |
| 引用内容一致 | 比对引用片段与上下文 | B4 |
| 无外部知识 | 检索回答中的实体是否在上下文 | B1 |

# 5. 常见失败案例库

## 5.1 案例表

### 5.1.1 典型案例

| 案例 | 类型 | 修复 |
|------|------|------|
| 问"报销时限"答不出 | A3 分块 | 增大块或标题拼接 |
| 问"民法典 1024 条"答不出 | A5 专有名词 | 加 BM25 |
| 问同义表述答不出 | A4 语义鸿沟 | 查询改写 |
| 有资料却编造数字 | B1 幻觉 | 强化约束 |
| 只答了资料的一部分 | B3 遗漏 | 减少候选、关键前置 |
| 引用了不存在的来源 | B4 引用 | 校验引用 |

### 5.1.2 案例使用方式

案例库不是"收藏夹"，而是"诊断手册"。遇到新失败时，先按三步法定位，再到案例库里找同类，复用已验证的修复方案，避免重复试错。

## 5.2 案例扩展

### 5.2.1 新增案例

| 现象 | 可能类型 | 验证方式 |
|------|----------|----------|
| 数字答反 | B1/B5 | 检查原文是否有该数字 |
| 只答一半 | B3/A3 | 看上下文是否含另一半 |
| 多问题只答一个 | B2 | 检查是否多意图问题 |

### 5.2.2 案例生命周期

```
发现失败 ─► 定位归因 ─► 修复验证 ─► 归档案例 ─► 回流评测集
```

# 6. 失败案例回流

## 6.1 回流闭环

### 6.1.1 闭环流程

```
失败案例 → 分析归因 → 修复 → 加入评测集 → 防回归
```

### 6.1.2 Java 实现

```java
public void recordFailure(FailureCase c) {
    failureStore.save(c);

    // 自动转为评测用例
    EvalCase evalCase = new EvalCase(
            c.question(),
            c.expectedAnswer(),
            c.relevantDocIds(),
            c.mustContain());
    evalSet.add(evalCase);
}
```

## 6.2 回流治理

### 6.2.1 去重与标注

| 治理项 | 做法 |
|--------|------|
| 去重 | 按问题文本归一化后哈希 |
| 标注 | 记录类型（A/B）、子类、修复方式 |
| 优先级 | 高频失败优先回流 |

### 6.2.2 回流节奏

> **失败案例是最宝贵的评测资产**，必须回流。

- 每次线上事故后 24 小时内回流。
- 每周汇总一次高频失败，补充评测集。
- 回流用例与代码同版本化，纳入回归。

# 7. 批量失败分析

## 7.1 批量统计

### 7.1.1 Java 实现

```java
public FailureReport analyzeBatch(List<EvalCase> cases) {
    Map<String, Integer> failureTypes = new HashMap<>();

    for (EvalCase c : cases) {
        List<Document> docs = retriever.retrieve(c.question());

        if (!containsAnswer(docs, c.groundTruth())) {
            failureTypes.merge("RETRIEVAL", 1, Integer::sum);
        } else {
            String answer = generate(c.question(), docs);
            if (!isCorrect(answer, c.groundTruth())) {
                failureTypes.merge("GENERATION", 1, Integer::sum);
            }
        }
    }
    return new FailureReport(failureTypes);
}
```

### 7.1.2 输出解读

输出示例：

```
RETRIEVAL: 18
GENERATION: 7
→ 结论：检索是主要瓶颈，优先优化分块与混合检索
```

## 7.2 瓶颈排序

### 7.2.1 优先级判断

| 检索失败占比 | 结论 | 优先动作 |
|:------------:|------|----------|
| > 60% | 检索是瓶颈 | 分块、混合检索、重排 |
| 30%~60% | 两层并重 | 先修检索再调提示 |
| < 30% | 生成是瓶颈 | 提示约束、上下文治理 |

### 7.2.2 优化验证

任何优化后都要重跑同一批用例，用失败数下降验证效果，避免"感觉变好了"。

### 7.2.3 报告落库

```java
public void persist(FailureReport report, String runId) {
    Map<String, Object> row = new HashMap<>();
    row.put("run_id", runId);
    row.put("retrieval_failures", report.count("RETRIEVAL"));
    row.put("generation_failures", report.count("GENERATION"));
    row.put("created_at", Instant.now());
    reportStore.insert(row);
}
```

# 8. 生产注意事项

## 8.1 记录与观测

### 8.1.1 全链路记录

| 风险 | 建议 |
|------|------|
| 无失败记录 | 全量记录失败案例 |
| 只看结果 | 记录检索与生成全过程 |

### 8.1.2 观测字段

| 字段 | 用途 |
|------|------|
| traceId | 串联一次请求的全链路 |
| 检索命中列表 | 定位检索问题 |
| 上下文快照 | 定位生成问题 |
| 最终回答 | 复盘 |

## 8.2 分析与治理

### 8.2.1 批量分析

| 风险 | 建议 |
|------|------|
| 只修个案 | 批量分析找共性 |
| 不回流 | 失败案例进评测集 |

### 8.2.2 归因纪律

| 风险 | 建议 |
|------|------|
| 归因错误 | 先定位检索/生成，再细分 |

- 归因结论必须附原始证据（命中列表、上下文、回答）。
- 修复后必须回归验证，防止"按下葫芦浮起瓢"。

# 9. 小结

- 失败分析三步：复现 → 定位（检索/生成）→ 归因。
- 检索失败 9 类，生成失败 6 类，各有对策。
- 批量分析能找出主要瓶颈，指导优化优先级。
- 失败案例必须回流为评测用例。
- 记录检索与生成全过程，是归因的前提。
