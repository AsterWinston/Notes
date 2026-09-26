# 1. 整体流程与设计目标

## 1.1 端到端流程

结论先行：研究 Agent（深度研究智能体）的本质是一条"分解—检索—抓取—评估—整合—生成—校验"的流水线，任何环节缺失都会导致报告质量或可信度塌陷。机制：主题先拆成若干可独立检索的子问题，逐个执行检索与抓取；信息不足则回到检索补充，直到充分或触发上限；随后整合去重、生成报告并校验引用。

```
研究任务
   │
   ▼
① 任务分解 → 子问题列表
   │
   ▼
② 对每个子问题：
   ├── 检索（搜索）
   ├── 抓取（网页）
   ├── 清洗（去噪、防注入）
   └── 记录来源
   │
   ▼
③ 充分性判断 → 不足则补充检索
   │
   ▼
④ 整合去重 → 结构化信息
   │
   ▼
⑤ 生成报告（带引用）
   │
   ▼
⑥ 引用校验
```

边界与坑：图中的"循环"最易失控，若无轮数上限，弱模型会在"检索—评估"间反复横跳；若无成本上限，单任务可能烧掉数十元。生产建议：把每一步做成可观测的独立节点，记录耗时、Token（词元）消耗与产物摘要。

## 1.2 设计目标与非目标

结论先行：研究 Agent 优先保证"可信"，其次才是"全面"和"快"。机制：目标分功能与非功能两类，非目标用于划定边界、防止范围蔓延。

| 类别 | 目标 | 说明 |
|------|------|------|
| 功能 | 任务分解 | 自动生成 3~5 个可检索子问题 |
| 功能 | 多轮联网检索 | 每个子问题可多轮补充 |
| 功能 | 抓取与清洗 | 去噪、去脚本、防注入 |
| 功能 | 结构化报告 | 背景、发现、分析、结论 |
| 功能 | 引用可溯源 | 每条关键结论带来源编号 |
| 非功能 | 时长 | 单任务小于 10 分钟 |
| 非功能 | 成本 | 单报告小于 5 元 |
| 非功能 | 安全 | 无 SSRF（服务端请求伪造）、无注入 |
| 非目标 | 实时数据 | 股价、汇率等不做保证 |
| 非目标 | 付费数据库 | 不接入需授权的闭源库 |

边界与坑：把"实时数据"列为非目标后，报告中的时效性结论必须标注"待核实"，否则会误导用户。生产建议：把非目标写进系统提示与验收清单，避免无意识扩张。

## 1.3 设计原则

结论先行：四条原则贯穿全流程——不可信输入、可溯源、有界循环、渐进降级。机制：所有网页内容视为数据而非指令；任何结论都能回指来源编号；轮数、步数、预算三道上限；模型不可用时降级到检索摘要而非直接失败。

边界与坑：原则只写在文档里等于没有，"有界循环"必须落地为 `MAX_ROUNDS` 常量与预算检查点。生产建议：把每条原则映射到一个可测试断言（如"注入用例全部被过滤"），纳入 CI（持续集成）。

# 2. 系统架构

## 2.1 分层架构

结论先行：采用"接入层—编排层—能力层—数据层"四层架构，编排层是唯一的流程大脑。机制：接入层负责鉴权与限流；编排层负责图编排与状态流转；能力层提供搜索、抓取、清洗、模型等原子能力；数据层负责持久化与缓存。

```
+--------------------------------------------------+
| 接入层  REST / SSE / 鉴权 / 限流                 |
+--------------------------------------------------+
| 编排层  StateGraph / 路由 / 检查点               |
+--------------------------------------------------+
| 能力层  Search Fetch Clean Assess LLM            |
+--------------------------------------------------+
| 数据层  PostgreSQL(状态) Redis(缓存) 对象存储    |
+--------------------------------------------------+
```

边界与坑：能力层不应互相调用（如清洗器直接调模型），否则依赖成环、难以替换。生产建议：能力层接口保持窄而稳定，便于单独 Mock（模拟）测试。

## 2.2 组件职责与依赖

结论先行：每个组件单一职责，依赖通过构造注入，便于替换与测试。机制：核心组件及职责如下表。

| 组件 | 职责 | 关键依赖 |
|------|------|----------|
| Decomposer | 主题拆解为子问题 | LLM |
| SearchClient | 关键词检索 | 搜索 API |
| UrlValidator | URL 白名单与内网屏蔽 | 配置 |
| Fetcher | 抓取网页原文 | HTTP 客户端 |
| ContentCleaner | 去噪、提取正文 | Jsoup |
| InjectionSanitizer | 过滤注入模式 | 正则 |
| Assessor | 充分性判断 | LLM |
| Synthesizer | 整合去重 | LLM |
| ReportGenerator | 生成带引用报告 | LLM |
| CitationValidator | 校验引用编号 | 规则 |

边界与坑：Decomposer 与 Assessor 都依赖 LLM，共用同一提示模板容易串味，应分模板管理。生产建议：LLM 调用统一走带重试与计费的网关。

## 2.3 部署拓扑

结论先行：单实例可跑通，生产需无状态编排加外部状态存储才能水平扩展。机制：编排器无状态，状态写 PostgreSQL；长任务经消息队列解耦，SSE（服务器推送事件）从 Redis 订阅进度。

```
Client ──> API Gateway ──> Research Service (无状态, 多副本)
                                │
                ┌───────────────┼───────────────┐
                ▼               ▼               ▼
           PostgreSQL        Redis           MQ/Worker
           (状态/任务)     (进度/缓存)     (长任务执行)
```

边界与坑：状态放在 JVM 内存时，副本扩容后任务恢复会失败。生产建议：状态表加乐观锁版本号，避免并发写覆盖。

# 3. 状态与数据模型

## 3.1 状态设计

结论先行：状态对象是流程的唯一事实来源，字段覆盖输入、中间产物、输出与成本四类。机制：图编排中节点返回增量补丁而非直接改原状态，才能安全重放与恢复。

### 3.1.1 字段定义

```java
public class ResearchState {

    private String topic;                          // 研究主题
    private List<SubQuestion> subQuestions;        // 子问题
    private Map<String, List<Finding>> findings;   // 各子问题的发现
    private List<Source> sources;                  // 来源
    private String report;                         // 最终报告
    private int round;                             // 轮次
    private BigDecimal cost;                       // 成本
}
```

边界与坑：`findings` 用 `Map` 时键必须稳定（子问题原文或 ID），否则整合阶段丢数据。生产建议：增加 `traceId` 与 `budgetLeft` 字段，便于追踪与预算控制。

### 3.1.2 状态不可变性

```java
StateGraph graph = new StateGraph(ResearchState.class)
        .addNode("search", this::search,
                state -> Map.of("findings", mergeFindings(state)));
```

边界与坑：节点直接修改共享集合会在重试时重复追加。生产建议：集合字段统一用"追加加去重"的 reducer（归约器）合并。

## 3.2 核心实体模型

结论先行：实体保持扁平，避免深层嵌套，便于序列化与检索。机制：四个核心实体如下。

| 实体 | 关键字段 | 说明 |
|------|----------|------|
| SubQuestion | question, rationale | 子问题与理由 |
| Finding | question, content, sourceId | 发现及其来源编号 |
| Source | id, url, title, fetchedAt | 来源元数据 |
| Assessment | sufficient, missing | 充分性与缺口 |

```java
public record Source(int id, String url, String title, Instant fetchedAt) {}

public record Finding(String question, String content, int sourceId) {}
```

边界与坑：`Source.id` 与报告中的 `[n]` 必须严格一致，编号错位是最常见的引用失败原因。生产建议：来源编号在写入时即确定，后续只读不重排。

## 3.3 持久化表结构

结论先行：长任务必须持久化状态快照，支持中断恢复。机制：一张任务表存状态 JSON 与状态机字段，一张来源表存去重后的来源。

```sql
CREATE TABLE research_task (
    id          VARCHAR(64) PRIMARY KEY,
    topic       VARCHAR(512) NOT NULL,
    status      VARCHAR(16) NOT NULL,
    state_json  JSONB NOT NULL,
    version     INT NOT NULL DEFAULT 0,
    created_at  TIMESTAMP NOT NULL,
    updated_at  TIMESTAMP NOT NULL
);

CREATE TABLE research_source (
    task_id     VARCHAR(64) NOT NULL,
    source_id   INT NOT NULL,
    url         VARCHAR(1024) NOT NULL,
    title       VARCHAR(512),
    PRIMARY KEY (task_id, source_id)
);
```

边界与坑：`JSONB` 更新整列，状态大时写放大明显，应限制快照频率。生产建议：状态与来源分离存储，报告查询只读任务表即可。

# 4. 图编排

## 4.1 节点与边

结论先行：用状态图表达流程，节点即能力，边即流转，条件边即循环。机制：定义节点、串联固定边、用条件边处理补充检索。

```
START → decompose → [search → fetch → clean] × N → assess
                        ▲                              │
                        └────── 不足则补充 ─────────────┘
                                                       │
                                                       ▼
                                                    synthesize → validate → END
```

```java
StateGraph graph = new StateGraph(ResearchState.class)
        .addNode("decompose", this::decompose)
        .addNode("search", this::search)
        .addNode("fetch", this::fetch)
        .addNode("clean", this::clean)
        .addNode("assess", this::assess)
        .addNode("synthesize", this::synthesize)
        .addNode("validate", this::validate)
        .addEdge(START, "decompose")
        .addEdge("decompose", "search")
        .addEdge("search", "fetch")
        .addEdge("fetch", "clean")
        .addEdge("clean", "assess")
        .addConditionalEdges("assess", this::routeAfterAssess,
                Map.of("continue", "search", "done", "synthesize"))
        .addEdge("synthesize", "validate")
        .addEdge("validate", END)
        .compile();
```

边界与坑：把 fetch 与 clean 合成一个节点虽简单，但失败重试粒度变粗。生产建议：抓取与清洗拆开，各自可独立重试。

## 4.2 条件路由与循环

结论先行：循环必须有退出条件，否则任务永不结束。机制：路由函数综合"充分性、轮数、预算"三个条件。

```java
private String routeAfterAssess(ResearchState state) {
    boolean sufficient = state.assessment().sufficient();
    boolean budgetOk = state.round() < MAX_ROUNDS
            && state.cost().compareTo(BUDGET) < 0;
    return sufficient || !budgetOk ? "done" : "continue";
}
```

| 条件 | 取值 | 去向 |
|------|------|------|
| 充分且预算内 | true | done |
| 不充分但达上限 | false | done（降级） |
| 不充分且预算内 | false | continue |

边界与坑：达上限时若直接失败，用户体验差，应降级为"信息有限"的报告。生产建议：路由函数保持纯函数，便于单元测试覆盖所有分支。

## 4.3 检查点与恢复

结论先行：检查点让长任务可恢复，是生产可用性的关键。机制：每次节点完成后持久化状态快照，恢复时从最近检查点继续。

```java
CompiledGraph<ResearchState> compiled = graph.compile(
        CompileConfig.builder()
                .checkpointSaver(new PostgresSaver(dataSource))
                .build());
```

边界与坑：检查点过于频繁拖慢流程，过于稀疏则恢复代价高。生产建议：在"轮次边界"和"阶段边界"落检查点，兼顾性能与恢复粒度。

# 5. 任务分解

## 5.1 分解策略

结论先行：好的分解让子问题彼此独立、可检索、可验证。机制：常用三种策略。

| 策略 | 说明 | 适用 |
|------|------|------|
| 维度分解 | 按方面（技术/市场/风险） | 行业研究 |
| 对比分解 | 按对象对比 | 竞品分析 |
| 时序分解 | 按阶段（现状/趋势） | 趋势研究 |

边界与坑：子问题过多放大成本，过少覆盖不足，通常 3~5 个最佳。生产建议：让模型同时输出每个子问题的 rationale（理由），便于人工审核。

## 5.2 接口与质量校验

结论先行：分解接口输入主题、输出结构化子问题列表，且结果必须校验，防止空问题、重复问题、超量问题。机制：用声明式接口约束输出 JSON，再用非空、去重、数量区间、长度上限做校验。

```java
interface Decomposer {
    @UserMessage("""
            把研究主题拆解为 3~5 个可独立检索的子问题。
            主题：{{it}}
            输出 JSON：[{"question":"","rationale":""}]
            """)
    List<SubQuestion> decompose(String topic);
}
```

```java
List<SubQuestion> checked = subQuestions.stream()
        .filter(sq -> !sq.question().isBlank())
        .filter(sq -> sq.question().length() <= 200)
        .distinct()
        .limit(MAX_SUB_QUESTIONS)
        .toList();
```

边界与坑：模型偶尔返回非 JSON；`distinct()` 对 record 依赖 `equals`，字段含列表时需自定义比较。生产建议：对输出做 Schema（模式）校验，失败时回退到"按维度模板"的规则分解；把校验失败率作为监控指标。

# 6. 检索、抓取与安全

## 6.1 搜索与相关性筛选

结论先行：搜索求召回，筛选求精度，两步缺一不可。机制：先取 Top-N 结果，再由模型或规则筛选出最相关的少数 URL。

```java
List<SearchResult> results = searchClient.search(subQuestion, 10);
List<String> selected = selector.select(subQuestion, results, 3);
```

边界与坑：只按关键词排序会漏掉高价值但措辞不同的页面。生产建议：对搜索时间范围加过滤，缓解信息过时。

## 6.2 抓取与清洗

结论先行：抓取要稳，清洗要净，二者决定后续所有环节的质量上限。机制：抓取 HTML，清洗去除脚本、样式、导航、广告，提取正文。

```java
for (String url : selected) {
    if (!urlValidator.isAllowed(url)) continue;      // SSRF 防护
    String content = fetcher.fetch(url);
    String cleaned = cleaner.clean(content);          // 去噪
    String sanitized = sanitizer.sanitize(cleaned);   // 防注入
    findings.add(new Finding(subQuestion, sanitized, url));
}
```

边界与坑：动态渲染页面用 Jsoup 抓不到正文，需 Playwright 兜底。生产建议：抓取设置超时、大小上限与编码探测，防止慢页面拖垮流程。

## 6.3 SSRF 防护

结论先行：URL 必须经过白名单与内网屏蔽，否则可被诱导访问内网。机制：解析目标 IP，拒绝私有网段与云元数据地址。

```java
public boolean isAllowed(String url) {
    URI uri = URI.create(url);
    if (!"https".equals(uri.getScheme())) return false;
    InetAddress addr = InetAddress.getByName(uri.getHost());
    return !addr.isSiteLocalAddress()
            && !addr.isLoopbackAddress()
            && !addr.isLinkLocalAddress();
}
```

| 拦截目标 | 示例 |
|----------|------|
| 回环地址 | 127.0.0.1 |
| 私有网段 | 10.0.0.0/8 |
| 链路本地 | 169.254.169.254 |

边界与坑：DNS 重绑定（DNS Rebinding）可绕过首次解析，需在连接时再次校验。生产建议：使用带出口代理的抓取服务，从网络层再兜底一层。

# 7. 内容安全与注入防护

## 7.1 威胁模型

结论先行：网页是不可信输入，可能携带提示注入与恶意指令。机制：主要威胁与影响如下。

| 威胁 | 表现 | 影响 |
|------|------|------|
| 指令覆盖 | "忽略之前指令" | 输出被劫持 |
| 伪系统消息 | "system:" | 角色混淆 |
| 隐藏注入 | HTML 注释、白字 | 绕过肉眼检查 |

边界与坑：仅靠正则无法穷尽所有变体。生产建议：采用"清洗 + 提示隔离 + 输出校验"三重防御。

## 7.2 清洗与提示隔离

结论先行：消毒是纵深防御的一层，提示隔离是低成本高收益的补充。机制：去除 HTML 注释、过滤典型注入短语，并在提示中声明网页内容非指令。

```java
public String sanitize(String content) {
    return content
            .replaceAll("(?i)ignore (all )?previous instructions", "[已过滤]")
            .replaceAll("(?i)system\\s*:", "[已过滤]")
            .replaceAll("(?i)忽略(上面|之前).{0,10}(指令|规则)", "[已过滤]")
            .replaceAll("<!--[\\s\\S]*?-->", "");
}
```

边界与坑：过度过滤会误伤正常内容（产品名含 "system"）；强诱导内容仍可能影响模型。生产建议：正则列表可配置化并记录命中率；对模型输出做二次校验，拒绝包含系统提示片段的结果。

# 8. 充分性判断与整合

## 8.1 充分性评估

结论先行：充分性判断决定是否继续检索，是控制成本与质量的阀门。机制：模型读取已有发现，输出是否充分及缺口。

```java
interface Assessor {
    @UserMessage("""
            判断当前信息是否足以回答子问题 {{question}}。
            已有信息：{{findings}}
            输出 JSON：{"sufficient":true,"missing":["..."]}
            """)
    Assessment assess(String question, String findings);
}
```

边界与坑：模型常"过度自信"判定充分，需在提示中强调严格标准。生产建议：把 `missing` 缺口回灌到下一轮检索关键词，提升补充效率。

## 8.2 去重与整合

结论先行：多来源必然重复，整合要去重并消解冲突；补充检索只针对缺口，不重复全量搜索。机制：按语义聚类去重，用 `missing` 生成新关键词仅检索缺口。

| 情况 | 处理 |
|------|------|
| 完全重复 | 合并保留最早来源 |
| 部分重叠 | 合并内容，保留多来源 |
| 相互冲突 | 两者都留，标注分歧 |

边界与坑：去重过狠会丢失少数派证据；缺口描述模糊时新查询质量差，可让模型输出具体检索词。生产建议：保留来源计数供"多来源印证"使用；记录每轮新增发现数，新增为 0 时提前终止循环。

# 9. 报告生成与引用校验

## 9.1 报告结构与提示

结论先行：固定报告结构能显著提升可读性与评测可比性。机制：报告分为背景、核心发现、分析、结论四部分，每条结论带来源编号。

```java
String report = chatClient.prompt()
        .system("""
                你是研究分析师。基于提供的发现撰写报告。
                要求：
                1. 结构化（背景、发现、分析、结论）；
                2. 每条关键结论标注来源 [n]；
                3. 不得编造，无法确认的标注"待核实"。
                """)
        .user("主题：%s\n\n发现：\n%s".formatted(topic, formatFindings(findings)))
        .call().content();
```

边界与坑：模型可能引用不存在的编号，必须校验。生产建议：在提示中提供完整来源列表，并要求只能引用列表内编号。

## 9.2 引用映射与校验

结论先行：来源编号与 URL 的映射必须唯一且稳定，引用校验是可信度的最后闸门。机制：来源入库时分配自增编号，报告中仅用该编号；校验编号是否落在来源范围内。

| 报告编号 | 来源 URL |
|----------|----------|
| [1] | https://example.com/a |
| [2] | https://example.com/b |

```java
public boolean citationsValid(String report, List<Source> sources) {
    Set<Integer> cited = extractCitations(report);      // [1] [2] ...
    return cited.stream().allMatch(i -> i >= 1 && i <= sources.size());
}
```

边界与坑：编号合法不代表内容真实对应，需抽样人工核对；编号跨轮次复用会导致映射错乱。生产建议：来源列表随任务持久化；校验失败时重生成一次，仍失败则降级并告警。

# 10. 接口与事件协议

## 10.1 接口清单

结论先行：同步接口适合短任务，流式接口适合长任务。机制：三类核心接口如下。

| 接口 | 方法 | 用途 |
|------|------|------|
| /api/research | POST | 提交研究任务 |
| /api/research/{id} | GET | 查询任务与报告 |
| /api/research/stream | GET(SSE) | 流式进度 |

```java
@PostMapping("/api/research")
public ResearchAccepted submit(@RequestBody ResearchRequest req) {
    String taskId = taskService.create(req.topic());
    return new ResearchAccepted(taskId);
}
```

边界与坑：同步等待长任务会触发网关超时。生产建议：提交即返回任务 ID，结果通过查询或流式获取。

## 10.2 流式事件与错误处理

结论先行：事件协议要稳定可扩展，错误要分类处理。机制：定义 progress、finding、done、error 四类事件；可重试的自动重试，不可重试的明确告知。

```
event: progress  data: {"stage":"检索子问题1/4"}
event: finding   data: {"summary":"..."}
event: progress  data: {"stage":"生成报告"}
event: done      data: {"reportId":"..."}
```

| 错误 | 类型 | 处理 |
|------|------|------|
| 搜索超时 | 可重试 | 指数退避重试 |
| 抓取 403 | 可跳过 | 换来源 |
| 预算耗尽 | 终止 | 返回部分报告 |
| 校验失败 | 可重试 | 重生成一次 |

边界与坑：事件体变更会破坏前端解析，需版本化；无限重试会放大成本。生产建议：为事件增加序号与时间戳便于断线重连；错误分类与告警联动。

# 11. 关键权衡与演进

## 11.1 质量与成本

结论先行：质量与成本正相关，需用预算约束找到平衡点。机制：更强模型、更多来源、更多轮次都能提升质量，但成本上升。

| 手段 | 质量影响 | 成本影响 |
|------|----------|----------|
| 换强模型 | 高 | 高 |
| 增加来源 | 中 | 中 |
| 增加轮次 | 中 | 高 |

边界与坑：盲目堆轮次收益递减。生产建议：对子问题分级，重要子问题用强模型，其余用轻模型。

## 11.2 风险与对策

结论先行：主要风险集中在可信度、成本与安全三类；深度检索还会增加时延，需并行化抵消。机制：子问题之间可并行检索缩短总时长；风险与对策如下。

```
串行：sq1 ──> sq2 ──> sq3 ──> 生成
并行：sq1 ──┐
      sq2 ──┼──> 汇总 ──> 生成
      sq3 ──┘
```

| 风险 | 应对 |
|------|------|
| 信息过时 | 时效过滤 |
| 编造引用 | 引用校验 |
| 成本失控 | 步数与预算上限 |
| 循环检索 | 充分性判断 + 轮数上限 |
| 内容注入 | 网页内容视为不可信 |
| SSRF | URL 白名单 + 内网屏蔽 |

边界与坑：并行会提高瞬时并发与限流压力；风险对策需定期演练，否则形同虚设。生产建议：并行度可配置并加信号量限流；把对策转化为自动化测试与告警规则。

# 12. 小结

- 流程：分解 → 检索 → 抓取 → 清洗 → 评估 → 整合 → 报告 → 校验。
- 用图编排，条件边控制补充检索，检查点支持恢复。
- 网页内容必须清洗并视为不可信数据。
- 引用校验防止编造来源，编号映射必须稳定。
- 必须限制轮数与成本，长任务需持久化与流式进度。
