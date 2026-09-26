# 1. 评测集的价值

评测集（Evaluation Set / Benchmark Set）是一组带有输入、参考答案与判定条件的用例集合，相当于 LLM 应用的"单元测试 + 集成测试"。

## 1.1 为什么评测集是资产

结论先行：评测集决定了团队优化的上限与速度，是最应长期投入的资产。

### 1.1.1 评测集决定优化上限

优化只能沿着评测集能衡量的方向前进。评测集不覆盖的能力，团队既不知道它退化，也不会去优化它。

### 1.1.2 评测集是团队共识载体

"什么算好答案"往往是模糊的。把标准写成评测用例，团队对质量的定义就从口头共识变成可执行规范。

## 1.2 评测集的成本收益

### 1.2.1 构建成本

| 阶段 | 主要投入 | 量级 |
|------|----------|------|
| 首版 | 选真实问题、写答案 | 1~3 人日 |
| 扩充 | 补边界、对抗 | 持续，低 |
| 维护 | 回流、审查 | 每周，低 |

### 1.2.2 复利效应

评测集一旦建立，可复用于：选型对比、发版回归、Prompt 调优、线上问题复现。一次投入，长期复用。

## 1.3 边界与反例

- 评测集不是越大越好，覆盖质量比数量更重要。
- 反例：把评测集当"通过率秀场"，只加容易通过的用例，会掩盖真实风险。
- 评测集若长期不更新，会与真实分布脱节。

# 2. 数据来源

结论先行：最好的评测集来自"真实问题 + 专家标注 + LLM 扩充 + 失败回流"的组合。

## 2.1 四类来源

### 2.1.1 真实用户问题

直接来自日志、客服工单、搜索词。质量最高、最贴近分布，但需脱敏（去除 PII，即个人隐私信息）。

### 2.1.2 业务专家编写

由业务专家针对关键流程编写，覆盖深、答案准，但成本高、规模小。

### 2.1.3 LLM 生成

用大模型根据文档批量生成问题与答案，成本低、规模大，但需人工审核。

### 2.1.4 失败案例回流

把线上或评测中失败的案例固化为新用例，针对性强、价值最高。

## 2.2 最佳组合策略

| 来源 | 质量 | 成本 | 规模 |
|------|:----:|:----:|:----:|
| 真实用户问题 | 最高 | 低（需脱敏） | 大 |
| 业务专家编写 | 高 | 高 | 小 |
| LLM 生成 | 中 | 低 | 大 |
| 失败案例回流 | 最高（针对性强） | 低 | 小 |

> 组合方式：真实问题打底（分布），专家标注提质量，LLM 扩充补规模，失败回流补短板。

## 2.3 从日志抽取真实问题

从日志中抽取时需按"解析用户查询 → 去噪 → 脱敏（手机号、邮箱、身份证）→ 去重"的流程处理。抽取后仍需人工确认"这是不是一个值得评测的问题"，日志里大量是无意义查询。

## 2.4 边界与坑

- 真实问题含噪声与无效查询，需清洗。
- LLM 生成的问题常"过于标准"，与真实口语表达有差距。
- 专家编写容易集中在自己熟悉的场景，覆盖不均。
- 日志抽取若不去重，会出现大量"同一个问题的重复问法"，虚增规模。

# 3. 数据集结构

结论先行：一条评测用例至少要包含"输入、判定依据、分类"三类信息，缺一不可。

## 3.1 EvalCase 定义

### 3.1.1 字段说明

```java
public record EvalCase(
        String id,
        String question,
        String groundTruth,             // 标准答案
        List<String> relevantDocIds,    // 相关文档（检索评测）
        List<String> mustContain,       // 关键信息
        List<String> mustNotContain,    // 禁止内容
        String category,                // 分类（便于分组统计）
        String difficulty               // 难度
) {}
```

| 字段 | 用途 | 必填 |
|------|------|:----:|
| id | 唯一标识、归因 | 是 |
| question | 输入 | 是 |
| groundTruth | 标准答案 | 否 |
| relevantDocIds | 检索评测标注 | 否 |
| mustContain | 关键词断言 | 否 |
| mustNotContain | 反向断言 | 否 |
| category | 分组统计 | 建议 |
| difficulty | 分组统计 | 建议 |

### 3.1.2 JSON 示例

```json
{
  "id": "q-001",
  "question": "差旅费报销的时限是多久？",
  "groundTruth": "费用发生后 30 天内提交报销申请",
  "relevantDocIds": ["handbook-3-2"],
  "mustContain": ["30 天"],
  "mustNotContain": ["我不知道", "无法确定"],
  "category": "policy",
  "difficulty": "easy"
}
```

## 3.2 扩展字段

### 3.2.1 分组字段

`category`、`difficulty`、`language` 等用于分组统计，定位"哪类问题退化"。

### 3.2.2 元数据字段

用 `Map<String, Object>` 承载来源、标注人、创建时间、是否黄金集等。

```java
public record EvalCaseV2(
        String id,
        String question,
        String groundTruth,
        String category,
        String difficulty,
        boolean golden,
        Map<String, Object> metadata
) {}
```

## 3.3 边界与坑

- `groundTruth` 不是所有场景都有（如开放式创作），此时靠 `mustContain` 与 Judge。
- `mustContain` 过严会误判（同义表达），过松则形同虚设。
- 分类字段的取值域要事先约定，否则统计时无法聚合。

# 4. LLM 生成评测集

结论先行：LLM 生成能快速扩充规模，但必须经过人工审核，绝不能直接入库。

## 4.1 生成流程

### 4.1.1 从文档生成

把知识库文档片段喂给 LLM，让它生成"用户可能问的问题 + 依据文档的答案"。

### 4.1.2 从种子扩写

以少量真实问题为种子，让 LLM 改写、扩展为更多表达变体（口语化、省略、错别字）。

```
文档片段 ──→ LLM 生成 ──→ 候选用例 ──→ 人工审核 ──→ 入库
真实问题 ──→ LLM 改写 ──→ 表达变体 ──→ 人工审核 ──→ 入库
```

## 4.2 生成代码

```java
interface EvalCaseGenerator {

    @UserMessage("""
            根据下面的文档片段，生成 {{count}} 个用户可能提出的问题，
            并为每个问题给出标准答案。

            要求：
            - 问题贴近真实用户表达（口语化、多样化）
            - 答案必须能在文档中找到依据
            - 输出 JSON 数组

            文档：{{doc}}

            输出格式：
            [{"question":"","answer":""}]
            """)
    List<GeneratedCase> generate(@V("doc") String doc,
                                  @V("count") int count);
}
```

```java
public List<EvalCase> generateAndDraft(String doc, int count) {
    return generator.generate(doc, count).stream()
            .map(d -> new EvalCase(
                    UUID.randomUUID().toString(),
                    d.question(), d.answer(),
                    List.of(), List.of(), List.of(),
                    "generated", "medium"))
            .toList();
}
```

## 4.3 生成后必须人工审核

### 4.3.1 常见生成缺陷

```
LLM 生成的问题可能：
- 与文档无关
- 答案有误
- 表达不自然
- 覆盖不均
```

### 4.3.2 审核流程

```
① 自动预筛：与文档相似度过低者丢弃
② 人工抽检：核对答案是否在文档中有依据
③ 覆盖检查：统计分类分布，补齐稀疏类别
④ 标记来源：generated + reviewed
```

> **LLM 生成 + 人工审核**，不能直接用。

## 4.4 边界与坑

- 生成器可能"自问自答"，问题与答案同源，导致评测过于简单。
- 审核人力不足时，宁少勿滥，低质量用例会污染指标。
- 生成的问题可能泄露文档结构，使检索评测失真。

# 5. 数据集规模

结论先行：起步 50 条即可，先跑起来，再按边际收益逐步扩充。

## 5.1 规模建议

| 用途 | 建议规模 |
|------|----------|
| 快速回归 | 20~50 |
| 常规评测 | 50~200 |
| 全面评测 | 200~500 |
| 选型对比 | 100+ |

## 5.2 规模与统计显著性

### 5.2.1 样本量估算

若想以 95% 置信度区分两个版本，样本量大致需满足：

```
n ≈ 16 * p * (1 - p) / Δ²
```

其中 `p` 为通过率，`Δ` 为希望检出的最小差异。例如 `p = 0.8`、`Δ = 0.05` 时，`n ≈ 1024`。这意味着微小提升很难在小集上验证。

### 5.2.2 边际收益递减

从 20 条扩到 100 条，指标稳定性提升显著；从 500 条扩到 1000 条，收益递减，除非要做精细的选型对比。

## 5.3 边界与坑

- 用例太少时，一条用例的翻转就能让通过率跳动 5%。
- 用例数量不等于覆盖度，100 条同质用例不如 30 条多样用例。
- 评测集越大，单次评测成本越高，需权衡运行频率。

# 6. 数据集覆盖

结论先行：覆盖度要从问题类型、难度、长度、边界、对抗、语言六个维度共同检查。

## 6.1 覆盖维度

### 6.1.1 问题类型与难度

```
维度：
├── 问题类型：事实、流程、比较、推理
├── 难度：简单、中等、困难
├── 长度：短查询、长查询
├── 边界：无答案、多意图、模糊
├── 对抗：注入、越狱、诱导
└── 语言：中文、英文、混合
```

### 6.1.2 长度与语言

长查询考察上下文处理，短查询考察意图补全；中英混合考察多语言鲁棒性。

### 6.1.3 边界与对抗

无答案问题考察"是否会拒答"；对抗问题考察"是否会被注入攻破"。

## 6.2 覆盖度检查代码

```java
public record CoverageReport(
        Map<String, Long> byCategory,
        Map<String, Long> byDifficulty,
        long noAnswerCount,
        long adversarialCount
) {}
```

```java
public CoverageReport analyze(List<EvalCase> cases) {
    Map<String, Long> byCategory = cases.stream()
            .collect(Collectors.groupingBy(
                    EvalCase::category, Collectors.counting()));
    Map<String, Long> byDifficulty = cases.stream()
            .collect(Collectors.groupingBy(
                    EvalCase::difficulty, Collectors.counting()));

    long noAnswer = cases.stream()
            .filter(c -> c.groundTruth() == null)
            .count();
    long adversarial = byCategory.getOrDefault("adversarial", 0L);

    return new CoverageReport(byCategory, byDifficulty, noAnswer, adversarial);
}
```

## 6.3 边界与坑

- 只看总数不看分布，会漏掉"某一类为零"的盲区。
- 对抗用例占比过高会拉低整体通过率，需分组看。
- 覆盖度检查应作为入库门禁的一部分。

# 7. 黄金集（Golden Set）

结论先行：黄金集是从不用于调优的高质量子集，用于最终判定"是否真提升"。

## 7.1 黄金集定义

```
黄金集 = 经过人工严格审核的高质量评测子集
特点：答案唯一、标注准确、长期稳定
用途：核心回归，不参与调优
```

## 7.2 黄金集的使用纪律

### 7.2.1 不参与调优

黄金集的题目与答案不得出现在 Prompt、Few-shot 示例或微调数据中，否则就失去了独立检验的意义。

### 7.2.2 冻结与版本

黄金集一旦冻结，只增不改；如确需修改，必须记录原因并重置历史基线。

| 属性 | 要求 |
|------|------|
| 标注 | 双人独立标注 + 仲裁 |
| 答案 | 唯一且无歧义 |
| 变更 | 冻结，变更需审批 |
| 可见性 | 调优者不可见 |

> 调优只看非黄金集，验收只看黄金集，两者物理隔离。

## 7.3 边界与坑

- 黄金集过小（< 30 条）时结论不稳定。
- 黄金集若泄露给调优团队，等价于普通评测集。
- 黄金集也需定期补充新场景，但要单独成"版本"。

# 8. 数据集维护

结论先行：评测集要像代码一样版本化、审计化、回流化，才能持续保值。

## 8.1 版本化管理

### 8.1.1 版本模型

```java
public record DatasetVersion(
        String name,
        int version,
        List<EvalCase> cases,
        Instant createdAt
) {}
```

### 8.1.2 变更审计

每次增删改都记录：谁改的、改了什么、为什么。评测结果必须绑定数据集版本，否则无法复现。

## 8.2 失败案例回流

### 8.2.1 回流流程

```
线上/评测失败 → 脱敏 → 标注期望 → 入库 → 标记 regression
```

### 8.2.2 回流代码

```java
public class EvalDatasetManager {

    // 版本化
    public void version(String name, List<EvalCase> cases) {
        store.save(new DatasetVersion(
                name, nextVersion(name), cases, Instant.now()));
    }

    // 失败案例回流
    public void addFromFailure(FailureCase failure) {
        EvalCase c = new EvalCase(
                "fail-" + failure.id(),
                failure.question(),
                failure.expected(),
                failure.relevantDocIds(),
                failure.mustContain(),
                List.of(),
                failure.category(),
                "regression");
        store.append(c);
    }
}
```

## 8.3 定期审查

```java
@Scheduled(cron = "0 0 1 * * MON")
public void reviewStale() {
    List<EvalCase> stale = store.findNotUsedIn(Duration.ofDays(90));
    log.info("长期未使用的评测用例: {}", stale.size());
}
```

## 8.4 边界与坑

- 回流用例若未经确认，可能把"错误的期望"固化进评测集。
- 长期不用的用例可能已过时（业务规则变了），需审查或淘汰。
- 版本膨胀会导致评测时间越来越长，可分层运行。

# 9. 数据集质量检查

结论先行：入库前必须做字段校验与重复检测，防止脏数据污染指标。

## 9.1 校验规则

| 检查项 | 规则 |
|--------|------|
| 非空 | 问题不能为空 |
| 长度 | 问题不宜过长 |
| 有判定依据 | 必须有答案或关键信息 |
| 重复 | 与现有用例语义重复需提示 |
| 分类合法 | 取值在约定域内 |

## 9.2 校验代码

### 9.2.1 字段校验

```java
public void validate(EvalCase c) {
    Assert.hasText(c.question(), "问题不能为空");
    Assert.isTrue(c.question().length() < 500, "问题过长");
    Assert.isTrue(c.groundTruth() != null
                    || (c.mustContain() != null && !c.mustContain().isEmpty()),
            "必须有答案或关键信息");

    if (isDuplicate(c, existing)) {
        log.warn("疑似重复用例: {}", c.id());
    }
}
```

### 9.2.2 重复检测

字符级可用 Jaccard 相似度，语义级可用向量余弦：

```java
// 伪代码：基于词集合的 Jaccard 相似度
double jaccard(String a, String b) {
    Set<String> sa = tokenize(a);   // 分词后的集合
    Set<String> sb = tokenize(b);
    Set<String> inter = new HashSet<>(sa);
    inter.retainAll(sb);
    Set<String> union = new HashSet<>(sa);
    union.addAll(sb);
    return inter.size() / (double) union.size();
}
```

> 阈值建议 0.9 以上判为重复；语义重复用向量相似度阈值 0.95 以上。

## 9.3 边界与坑

- 改写型用例（同一问题的不同问法）不应判为重复，否则丢失覆盖。
- Jaccard 对同义词无效，需结合语义相似度。
- 校验脚本要纳入 CI，防止手工入库绕过检查。

# 10. 生产注意事项

结论先行：评测集是长期资产，重点是版本化、脱敏、回流与防过拟合。

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 无评测集 | 先建 50 条 |
| 只用 LLM 生成 | 人工审核 |
| 覆盖不均 | 按维度检查 |
| 过拟合 | 保留黄金集 |
| 不更新 | 失败案例回流 |
| 无版本 | 版本化管理 |
| 含 PII | 脱敏 |

## 10.2 落地清单

```
① 从真实问题选 50 条建首版
② 补齐边界、无答案、对抗用例
③ 划分黄金集并冻结
④ 建立版本与变更审计
⑤ 接入 CI 做字段与重复校验
⑥ 失败案例自动回流
```

## 10.3 常见坑

- 评测集与调优数据混用，导致指标虚高。
- 只维护一份集，既调优又验收，无法自证。
- 标注无规范，不同人标注口径不一致。

# 11. 小结

## 11.1 核心结论

- 评测集是团队最宝贵的资产，需版本化管理。
- 来源：真实问题 + 专家标注 + LLM 扩充 + 失败回流。
- LLM 生成需人工审核。
- 覆盖多维度：类型、难度、边界、对抗、语言。
- 保留一份从不用于调优的黄金集。

## 11.2 落地清单

建集 → 审核 → 划黄金集 → 版本化 → 接 CI → 失败回流。
