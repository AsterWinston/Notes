# 1. 从固定流程到自主决策

本章结论先行：传统 RAG 是一条**固定流水线**，而 Agentic RAG（智能体式检索增强生成）把"是否检索、检索什么、检索几次、用什么工具、结果够不够"这些决策交给 Agent（智能体）自主完成。理解两者的差异，是后续所有模式与取舍的基础。

## 1.1 传统 RAG 的固定流水线

结论：传统 RAG 的每一步都在代码里写死，稳定、可预测、便宜，但无法应对多跳与多意图问题。

### 1.1.1 固定流程的机制拆解

传统 RAG（Retrieval-Augmented Generation，检索增强生成）的典型链路固定为：

```
用户问题
   │
   ▼
[查询向量化] → [向量库 topK 检索] → [拼接上下文] → [LLM 生成] → 答案
```

每个方框都是一次函数调用，顺序与参数在编译期或配置期确定。其隐含假设是：

- 问题一定需要检索（检索决策恒为"是"）；
- 一次检索足以覆盖全部所需信息（迭代次数恒为 1）；
- 检索结果一定相关（无评估环节）；
- 生成只依赖检索到的文本（无外部工具）。

只要假设成立，固定流水线就是最优解：延迟低、成本低、行为可复现。

### 1.1.2 固定流程的边界与失败模式

当假设被打破时，固定流水线会以可预测的方式失败：

| 失败模式 | 触发条件 | 现象 |
|----------|----------|------|
| 无谓检索 | 闲聊、纯计算问题 | 多花一次向量化与检索 |
| 漏召回 | 多跳问题（A 的答案引出 B） | 上下文不完整，答错 |
| 噪声淹没 | 语义相近但无关 | 模型被干扰，幻觉 |
| 无法计算 | 需要算术或聚合 | 模型口算易错 |
| 数据过期 | 需要实时信息 | 只能答训练期知识 |

这些失败恰好是 Agentic RAG 要解决的场景。

## 1.2 Agentic RAG 的核心思想

结论：把固定流程中的"写死的决策"变成"模型驱动的决策"，用一次或多次 LLM 调用换取更强的适应性。

### 1.2.1 四个自主决策问题

Agentic RAG 让 Agent 自主回答以下问题：

```
要不要检索？检索什么？检索几次？用什么工具？结果够不够？
```

对应四种能力：

- **路由（Routing）**：判断是否需要检索、检索哪个知识源；
- **改写（Rewriting）**：把用户口语化问题改写为检索友好的查询；
- **评估（Evaluation）**：判断已收集的证据是否足够；
- **编排（Orchestration）**：在检索、计算、API、代码执行之间切换。

### 1.2.2 传统 RAG 与 Agentic RAG 对比

| 维度 | 传统 RAG | Agentic RAG |
|------|----------|-------------|
| 检索次数 | 固定 1 次 | 按需多次 |
| 检索策略 | 固定 | 自主选择 |
| 查询改写 | 固定流程 | 按需 |
| 结果评估 | 无 | 自主判断是否足够 |
| 工具 | 只有检索 | 检索 + 计算 + API + 代码 |
| 决策主体 | 开发者 | 模型（受约束） |
| 延迟 | 低且稳定 | 高且波动 |

## 1.3 术语与前提

结论：Agentic RAG 不是"更高级的 RAG"，而是"用 Agent 框架承载 RAG"，因此它继承了 Agent 的全部约束。

### 1.3.1 关键术语

| 英文术语 | 中文含义 | 说明 |
|----------|----------|------|
| Agent | 智能体 | 能调用工具并循环决策的 LLM 程序 |
| Tool | 工具 | Agent 可调用的函数，带名称与描述 |
| Routing | 路由 | 选择是否检索、检索哪个源 |
| Self-RAG | 自反思检索 | 模型对检索结果自我判断 |
| Multi-hop | 多跳 | 需要多轮推理才能回答 |
| Trace | 链路追踪 | 记录每轮决策与工具调用 |

### 1.3.2 适用前提

Agentic RAG 成立需要三个前提，缺一不可：

1. **模型具备函数调用（Function Calling）能力**，且能稳定输出结构化参数；
2. **工具契约清晰**，描述准确、参数可校验、失败可返回错误信息；
3. **有明确的终止条件**，否则循环不收敛、成本失控。

若第 3 点缺失，Agentic RAG 会比传统 RAG 更糟——既慢又贵还不一定对。

# 2. 核心模式

结论：Agentic RAG 的工程实现可拆成三种可独立落地的模式——检索决策、迭代检索、自适应检索。它们可以单独使用，也可以组合。

## 2.1 检索决策（Retrieve or Not）

结论：不是所有问题都需要检索。加一个轻量路由，能省去无谓检索，降低延迟与成本。

### 2.1.1 机制拆解

路由本质上是一个二分类或 N 分类问题：

```
问题 → [分类器/LLM] → NEED_RETRIEVAL（需要检索）
                    → DIRECT（直接回答）
                    → TOOL_ONLY（只调工具，不检索）
```

分类器可以是：

- **规则**：关键词匹配（"报销""政策"→ 检索）；
- **小模型**：本地 BERT 分类，延迟极低；
- **LLM**：最灵活，但增加一次调用。

实践中推荐"规则兜底 + LLM 判断"的混合方式。

### 2.1.2 代码示例（LangChain4j 风格）

```java
interface Router {

    @UserMessage("""
            判断下面的问题是否需要查询知识库。
            需要查询（NEED_RETRIEVAL）：涉及具体事实、政策、内部信息
            无需查询（DIRECT）：闲聊、通用常识、纯计算
            只输出 NEED_RETRIEVAL 或 DIRECT。
            问题：{{it}}
            """)
    String route(String question);
}

public String answer(String question) {
    if ("NEED_RETRIEVAL".equals(router.route(question))) {
        return ragAnswer(question);
    }
    return chatClient.prompt().user(question).call().content();
}
```

### 2.1.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 误判为 DIRECT | 漏检索导致幻觉 | 阈值偏保守，宁多检索 |
| 路由本身很贵 | 每题多一次大模型调用 | 用规则或小模型 |
| 输出不稳定 | 自由文本无法匹配 | 强制 JSON + 枚举 |
| 多语言/多意图 | 一句话含多种意图 | 先做意图拆解再路由 |

## 2.2 迭代检索（Iterative Retrieval）

结论：当一次检索无法覆盖答案时，用"检索 → 评估是否足够 → 不足则改写查询再检索"的循环补全证据。

### 2.2.1 机制拆解

```
query = 原问题
for round in 1..maxRounds:
    docs += retrieve(query)
    if 充分(docs): break
    query = 改写(原问题, docs, 缺失信息)
answer = 生成(原问题, docs)
```

关键组件有两个：**充分性判断器**（Sufficiency Check）与**查询改写器**（Query Rewriter）。前者决定何时停，后者决定下一轮搜什么。若缺少充分性判断，循环会一直跑到上限。

### 2.2.2 代码示例

```java
public String iterativeRag(String question, int maxRounds) {
    List<Document> collected = new ArrayList<>();
    String query = question;

    for (int i = 0; i < maxRounds; i++) {
        collected.addAll(retrieve(query, 10));

        SufficiencyCheck check = evaluator.checkSufficiency(question, collected);
        if (check.sufficient()) break;

        query = rewriter.refine(question, collected, check.missing());
    }
    return generate(question, collected);
}
```

充分性判断的提示词示例：

```java
interface SufficiencyEvaluator {

    @UserMessage("""
            判断已有资料是否足以回答该问题，只输出 JSON：
            {"sufficient":true|false,"missing":"还缺什么信息"}
            问题：{{q}}
            已有资料：{{ctx}}
            """)
    SufficiencyCheck checkSufficiency(@V("q") String question,
                                      @V("ctx") String context);
}
```

### 2.2.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 循环不收敛 | 改写器反复搜同类内容 | 设最大轮数 + 去重 |
| 上下文爆炸 | 每轮累加导致超长 | 每轮重新裁剪 topN |
| 重复文档 | 相同片段多次入库 | 按 hash/相似度去重 |
| 评估器太宽松 | 过早停止 | 用带 missing 字段的严格提示 |
| 评估器太严格 | 永远不足 | 设轮数上限兜底 |

## 2.3 自适应检索（Self-RAG）

结论：Self-RAG 让模型对每个检索结果做**相关性判断**，过滤无关内容，从而减少噪声引起的幻觉。

### 2.3.1 机制拆解

Self-RAG（Self-Reflective RAG，自反思检索增强生成）的核心是引入若干"反思标记"（reflection token）：

```
[Retrieve]  是否需要检索
[IsRel]     文档是否与问题相关
[IsSup]     生成内容是否被文档支持
[IsUse]     答案是否有用
```

工程上不必真的训练带标记的模型，用一次 LLM 调用近似即可：先判断相关性，再过滤，再生成。

### 2.3.2 代码示例

```java
interface RelevanceJudge {

    @UserMessage("""
            判断下面的文档是否与问题相关，只输出 RELEVANT 或 IRRELEVANT。
            问题：{{q}}
            文档：{{d}}
            """)
    String judge(@V("q") String question, @V("d") String doc);
}

public List<Document> filterRelevant(String question, List<Document> docs) {
    return docs.stream()
            .filter(d -> "RELEVANT".equals(
                    judge.judge(question, d.getText())))
            .toList();
}
```

批量判断可降低调用次数：让模型一次输入多段文档，返回相关编号列表。

### 2.3.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 逐条调用太慢 | 10 条文档 10 次调用 | 批量判断 |
| 判断过于严格 | 全部过滤导致无上下文 | 保底保留 top1 |
| 与 Rerank 重叠 | 都在做相关性排序 | 二选一或分工 |
| 判断本身有幻觉 | 误判相关 | 用更强的模型或交叉验证 |

# 3. 用 Agent 实现 RAG

结论：用 Agent 框架承载 RAG，可以把检索、计算、数据库查询、联网等能力统一为"工具"，由模型自主编排。

## 3.1 Agent 与工具抽象

### 3.1.1 机制拆解

Agent 的运行循环是"思考 → 调用工具 → 观察结果 → 再思考"，直到模型给出最终答案或达到上限。检索既可以是内置的 ContentRetriever（内容检索器），也可以是一个普通工具。

### 3.1.2 代码示例（LangChain4j 风格）

```java
interface KnowledgeAgent {
    String answer(String question);
}

KnowledgeAgent agent = AiServices.builder(KnowledgeAgent.class)
        .chatModel(model)
        .contentRetriever(retriever)          // 检索能力
        .tools(new CalculatorTools(),          // 计算
               new DatabaseTools(),            // 结构化查询
               new WebSearchTools())           // 联网兜底
        .build();

String result = agent.answer("上季度销售下滑多少？原因是什么？");
// Agent 自主：查数据库 → 检索文档 → 计算 → 综合
```

## 3.2 检索作为工具 vs 作为 ContentRetriever

结论：两种接入方式语义不同——ContentRetriever 是"每轮自动注入上下文"，工具是"模型显式调用"。选择取决于是否需要模型控制检索时机。

### 3.2.1 对比

| 维度 | ContentRetriever | 检索工具 |
|------|:----------------:|:--------:|
| 触发方式 | 框架自动 | 模型显式 |
| 时机控制 | 无 | 完全可控 |
| 查询改写 | 需插件 | 模型自拟参数 |
| 实现复杂度 | 低 | 中 |
| 适用 | 简单问答 | 复杂编排 |

### 3.2.2 代码示例（Spring AI 风格）

```java
@Component
public class RetrievalTools {

    private final VectorStore vectorStore;

    public RetrievalTools(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    @Tool(description = "在内部知识库中检索与查询最相关的文档片段")
    public String searchKnowledgeBase(
            @ToolParam(description = "检索查询语句") String query) {

        List<Document> docs = vectorStore.similaritySearch(
                SearchRequest.builder().query(query).topK(5).build());
        return docs.stream()
                .map(Document::getText)
                .collect(Collectors.joining("\n---\n"));
    }
}
```

工具返回值应保留来源信息（文件名、页码），否则生成阶段无法给出引用。

## 3.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 工具描述模糊 | 模型不知道何时调用 | 描述写清输入输出与适用场景 |
| 返回值过长 | 撑爆上下文 | 工具内部先裁剪 |
| 参数类型错误 | 调用失败 | 用强类型 + 校验 |
| 无限调用 | 循环不终止 | 设最大迭代次数 |
| 异常未处理 | Agent 卡死 | 捕获异常返回错误文本 |

# 4. 多工具协同

结论：真实问题往往需要多个数据源与算子协同，Agent 的价值在于把"数据源选择 + 计算 + 综合"编排成一条自适应路径。

## 4.1 工具编排机制

### 4.1.1 规划-执行（Plan-and-Execute）

先让模型产出计划，再逐步执行：

```
问题："我们的产品在竞品中表现如何？"
  │
  ├── 工具1：检索内部产品文档
  ├── 工具2：联网搜索竞品信息
  ├── 工具3：查询内部销售数据
  └── 综合生成对比分析
```

规划-执行相比 ReAct（推理-行动交替）的优势是：计划可见、便于审计、可并行；劣势是计划可能过时，需要重规划。

### 4.1.2 代码示例

```java
record PlanStep(int index, String tool, String input, String purpose) {}

interface Planner {
    @UserMessage("""
            为回答下面的问题，列出需要调用的工具与顺序，只输出 JSON 数组。
            可用工具：search_knowledge_base、web_search、query_sales_db、calculator
            问题：{{it}}
            """)
    List<PlanStep> plan(String question);
}
```

执行器逐条调用工具，若某步失败则把失败信息回灌给模型重新规划。

## 4.2 工具设计与契约

### 4.2.1 工具描述即提示词

工具的名称与描述会进入模型的上下文，因此应当像写提示词一样写工具描述：

| 要素 | 反例 | 正例 |
|------|------|------|
| 名称 | tool1 | search_knowledge_base |
| 用途 | 搜索 | 检索内部知识库，用于政策、流程类问题 |
| 参数 | x | query：自然语言查询语句 |
| 返回 | 结果 | 若干文档片段，含来源与页码 |

## 4.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 工具太多 | 模型选择困难 | 控制在 5~10 个，必要时分组 |
| 工具重叠 | 功能相近 | 合并或明确边界 |
| 并行误用 | 有依赖的步骤被并行 | 计划中声明依赖 |
| 结果不一致 | 多源冲突 | 提示模型标注冲突而非强行统一 |

# 5. 与传统 RAG 的取舍

结论：Agentic RAG 并非全面更优，它是在"效果"与"延迟/成本/可预测性"之间的一次交换。默认应走传统 RAG，仅在复杂问题上切换。

## 5.1 成本-延迟-效果三角

### 5.1.1 对比表

| 维度 | 传统 RAG | Agentic RAG |
|------|:--------:|:-----------:|
| 延迟 | 低（1 次检索） | 高（多次） |
| 成本 | 低 | 高 |
| 效果（简单问题） | 好 | 相当或更差 |
| 效果（复杂问题） | 差 | 好 |
| 可预测性 | 高 | 低 |
| 调试难度 | 低 | 高 |

## 5.2 混合路由策略

### 5.2.1 复杂度分类器

复杂度可以从三个维度评估：

- **意图数**：一句话包含几个独立诉求；
- **跳数**：答案是否依赖前一步结论；
- **外部性**：是否需要实时数据或计算。

### 5.2.2 代码示例

```java
public String smartAnswer(String question) {
    Complexity complexity = classifier.assess(question);

    return switch (complexity) {
        case SIMPLE -> traditionalRag(question);
        case COMPLEX -> agenticRag(question);
    };
}
```

分类器可以用规则先兜底（含"对比""原因""多少"等词 → COMPLEX），再逐步替换为模型。

## 5.3 边界与反例

| 反例 | 为何 Agentic 更差 |
|------|-------------------|
| 简单事实问答 | 多花路由与循环开销，无收益 |
| 高并发在线服务 | 延迟波动导致超时 |
| 强合规场景 | 行为不可预测，难审计 |
| 无评测体系 | 无法证明变好，只会变贵 |

> **建议**：默认走传统 RAG，只在检测到复杂问题（多意图、多跳）时切换到 Agentic 模式。

# 6. 生产注意事项

结论：Agentic RAG 的生产风险集中在"不收敛、成本失控、不可观测"三类，必须用工程手段而非提示词来兜底。

## 6.1 风险清单与对策

### 6.1.1 风险表

| 风险 | 建议 |
|------|------|
| 无限迭代 | 最大轮数 |
| 成本失控 | Token 预算 |
| 延迟过高 | 简单问题走传统 RAG |
| 工具越权 | 工具级鉴权 |
| 无 Trace | 记录每轮检索与决策 |
| 结果不收敛 | 设置充分性判断标准 |

### 6.1.2 代码：轮数与预算兜底

```java
public String guardedAgenticRag(String question) {
    Budget budget = new Budget(/* maxTokens */ 20000, /* maxRounds */ 5);
    List<Document> collected = new ArrayList<>();
    String query = question;

    while (budget.hasRemaining()) {
        List<Document> round = retrieve(query, 10);
        budget.consume(estimateTokens(round));
        collected.addAll(round);

        if (evaluator.sufficient(question, collected)) break;
        if (!budget.canAffordAnotherRound()) break;

        query = rewriter.refine(question, collected);
    }
    return generate(question, collected);
}
```

预算与轮数必须在框架外强制，不能依赖模型"自觉"停止。

## 6.2 可观测性与评测

### 6.2.1 Trace 字段

每轮至少记录：

| 字段 | 说明 |
|------|------|
| round | 轮次 |
| query | 本轮查询 |
| toolCalls | 调用的工具与参数 |
| docIds | 命中文档 ID |
| decision | 路由/充分性判断结果 |
| tokens | 本轮消耗 |
| latencyMs | 本轮耗时 |

## 6.3 落地建议

| 阶段 | 动作 |
|------|------|
| 起步 | 传统 RAG + 路由 |
| 进阶 | 加迭代检索与过滤 |
| 成熟 | 多工具编排 + 全链路 Trace |
| 治理 | 预算、鉴权、审计、灰度 |

# 7. 小结

## 7.1 核心结论

- Agentic RAG 让 Agent 自主决定"是否检索、检索几次、用什么工具"。
- 三种模式：检索决策、迭代检索、自适应过滤，可单独使用也可组合。
- 用 Agent 框架承载 RAG，检索可作为 ContentRetriever 或普通工具接入。
- 多工具协同靠"规划-执行"与清晰的工具契约，注意鉴权与幂等。
- 复杂问题效果显著，但延迟与成本更高，简单问题仍应走传统 RAG。

## 7.2 落地清单

- 必须有轮数上限、成本预算与全链路 Trace。
- 先用评测证明收益，再叠加模式，避免"只变贵不变好"。
- 路由与充分性判断优先用规则或小模型兜底，LLM 判断作为增强。
- 工具描述当作提示词来写，异常以"观察结果"形式回灌模型。
