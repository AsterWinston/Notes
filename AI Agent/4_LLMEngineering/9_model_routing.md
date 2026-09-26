# 1. 为什么需要路由

## 1.1 成本与能力错配

### 1.1.1 问题

结论先行：不同任务对模型能力的要求差异巨大，而大模型价格是小模型的数十倍，一律用大模型是最大的浪费。

机制：模型能力与成本近似同向增长，但任务难度分布极不均匀——大多数请求其实是简单分类、抽取、改写。

```
不同任务对模型的要求不同：
- 简单分类 → 小模型足够
- 复杂推理 → 需要大模型
- 一律用大模型 → 成本高、延迟高
```

### 1.1.2 收益

> 模型分层路由是**降本第一手段**（可省 30%~70%）。

结论：把简单请求下沉到小模型，既降成本又降延迟，且对用户几乎无感。

```
成本对比（示意，同一批 1000 次请求）
一律大模型：  1000 × 50 单位 = 50000 单位
分层路由：    700 × 1 + 250 × 8 + 50 × 50 = 5200 单位
降幅：        ≈ 90%（极端示意，实际 30%~70%）
```

## 1.2 路由的目标

### 1.2.1 降本

结论：路由的核心收益是成本，其次是延迟；两者都来自"用刚好够用的模型"。

| 手段 | 典型降本幅度 | 对质量影响 |
|------|:------------:|:----------:|
| 简单请求下沉小模型 | 30%~70% | 可控 |
| 升级路由 | 20%~50% | 极小 |
| 按租户分层 | 10%~30% | 与等级挂钩 |
| 缓存 + 路由 | 叠加 | 可控 |

### 1.2.2 保质量

结论：降本不能以牺牲质量为代价，路由必须用评测数据证明"小模型在对应任务上够用"。

生产建议：为每个路由层级建立质量基线，任何路由变更都跑一遍回归评测。

# 2. 分层模型

## 2.1 三层模型

### 2.1.1 定义

结论先行：把可用模型抽象为 FAST、STANDARD、STRONG 三层，是路由系统的最小可用模型。

```java
public enum ModelTier {
    FAST,       // 小模型：分类、抽取、改写
    STANDARD,   // 中模型：常规问答、RAG 生成
    STRONG      // 大模型：复杂推理、代码
}
```

### 2.1.2 成本对比

| 层级 | 任务 | 相对成本 |
|------|------|:--------:|
| FAST | 分类、抽取、改写、格式化 | 1x |
| STANDARD | 问答、RAG、总结 | 5~10x |
| STRONG | 复杂推理、代码、长文 | 20~50x |

结论：成本差可达数十倍，因此哪怕只有部分请求下沉，整体成本也会显著下降。

## 2.2 分层边界

### 2.2.1 能力边界

结论：每层都有明确的能力边界，越界即质量崩塌。

| 层级 | 擅长 | 不擅长 |
|------|------|--------|
| FAST | 短文本、结构化输出 | 多步推理、长上下文 |
| STANDARD | 常规问答、摘要 | 复杂数学、代码 |
| STRONG | 复杂推理、代码 | 成本敏感的大批量任务 |

### 2.2.2 选择原则

结论：优先用能满足质量要求的"最低层"，即"够用就好"。

生产建议：把边界写成可测试的规则，随模型迭代定期重估边界。

```java
// 边界写成断言，作为回归测试的一部分
@Test
void fastTierHandlesClassification() {
    ModelTier tier = router.route("这是投诉还是咨询？",
            TaskContext.of(TaskType.INTENT_CLASSIFY));
    assertEquals(ModelTier.FAST, tier);
}
```

# 3. 路由策略

## 3.1 规则路由

### 3.1.1 实现

结论先行：规则路由零成本、可解释、可测试，是路由的第一选择。

```java
public ModelTier routeByRule(String question, TaskContext ctx) {
    // ① 明确的任务类型
    if (ctx.taskType() == TaskType.CLASSIFY) return ModelTier.FAST;
    if (ctx.taskType() == TaskType.EXTRACT) return ModelTier.FAST;
    if (ctx.taskType() == TaskType.CODE) return ModelTier.STRONG;

    // ② 长度
    if (question.length() < 20) return ModelTier.FAST;
    if (question.length() > 500) return ModelTier.STRONG;

    // ③ 关键词
    if (containsComplexMarkers(question)) return ModelTier.STRONG;

    return ModelTier.STANDARD;
}

private boolean containsComplexMarkers(String q) {
    return Set.of("分析", "推理", "为什么", "对比", "设计", "代码")
            .stream().anyMatch(q::contains);
}
```

### 3.1.2 边界

结论：规则路由对模糊问题无能为力，且关键词表会随业务漂移。

> **优先用规则（零成本）**，规则判断不了的再用模型分类。

反例：用户问"帮我写一个排序"，长度短，规则可能误判为 FAST，但实际需要 STRONG。

生产建议：规则表要覆盖已知的"短文本高难度"模式，并保留人工反馈入口持续补规则。

| 信号 | 倾向层级 |
|------|:--------:|
| 显式任务类型 | 最可靠 |
| 关键词 | 较可靠 |
| 长度 | 弱信号 |
| 语言/格式 | 弱信号 |

## 3.2 模型分类路由

### 3.2.1 小模型分类

结论：用小模型给问题打复杂度标签，成本可忽略，能覆盖规则覆盖不了的模糊场景。

```java
interface ComplexityClassifier {
    @UserMessage("""
            判断问题复杂度，只输出 SIMPLE / MEDIUM / COMPLEX：
            {{it}}
            """)
    String classify(String question);
}

public ModelTier routeByModel(String question) {
    String complexity = classifier.classify(question);
    return switch (complexity) {
        case "SIMPLE" -> ModelTier.FAST;
        case "COMPLEX" -> ModelTier.STRONG;
        default -> ModelTier.STANDARD;
    };
}
```

> 用小模型做分类，成本可忽略。

### 3.2.2 成本与准确率

| 分类方式 | 单次成本 | 准确率 | 延迟 |
|----------|:--------:|:------:|:----:|
| 规则 | 0 | 中 | 纳秒 |
| 小模型分类 | 极低 | 较高 | 数十毫秒 |
| 混合 | 极低 | 高 | 数十毫秒 |

结论：分类器本身的成本必须远低于被路由请求的成本，否则得不偿失。

## 3.3 混合策略

### 3.3.1 规则优先

结论：先跑规则，规则能判定就直接返回，避免不必要的分类调用。

### 3.3.2 兜底分类

结论：规则无法判定时再用小模型分类，兼顾成本与准确率。

```java
public ModelTier route(String question, TaskContext ctx) {
    Optional<ModelTier> byRule = routeByRuleSafe(question, ctx);
    return byRule.orElseGet(() -> routeByModel(question));
}
```

# 4. 路由实现

## 4.1 路由器

### 4.1.1 多客户端注册

结论先行：路由器内部维护"层级到客户端"的映射，对上层屏蔽模型差异。

```java
@Service
public class ModelRouter {

    private final Map<ModelTier, ChatClient> clients;

    public ModelRouter(@Qualifier("fastModel") ChatModel fast,
                       @Qualifier("standardModel") ChatModel standard,
                       @Qualifier("strongModel") ChatModel strong) {
        clients = Map.of(
                ModelTier.FAST, ChatClient.builder(fast).build(),
                ModelTier.STANDARD, ChatClient.builder(standard).build(),
                ModelTier.STRONG, ChatClient.builder(strong).build());
    }

    public String chat(String question, TaskContext ctx) {
        ModelTier tier = routeByRule(question, ctx);
        return clients.get(tier).prompt().user(question).call().content();
    }
}
```

### 4.1.2 统一入口

结论：所有模型调用必须经过路由器，禁止业务代码直接持有具体模型客户端。

生产建议：把路由器作为唯一出口，便于统一打点、限流与降级。

```java
// 业务层只依赖路由器，不感知具体模型
@Service
public class QaService {
    private final ModelRouter router;

    public String answer(String question, TaskContext ctx) {
        return router.chat(question, ctx);
    }
}
```

## 4.2 配置与扩展

### 4.2.1 配置化

结论：层级与模型的对应关系应配置化，换模型无需改代码。

```yaml
ai:
  routing:
    tiers:
      FAST:     { model: gpt-4o-mini,    provider: openai }
      STANDARD: { model: gpt-4o,         provider: openai }
      STRONG:   { model: o3,             provider: openai }
    rules:
      keyword-strong: [分析, 推理, 对比, 设计, 代码]
      length-strong: 500
      length-fast: 20
```

### 4.2.2 动态调整

结论：规则与阈值应支持热更新，配合灰度发布验证效果。

生产建议：路由配置变更走配置中心，带版本号并支持一键回滚。

# 5. 升级路由

## 5.1 先便宜后升级

### 5.1.1 流程

结论先行：先用小模型尝试，结果不达标再升级到大模型，平均成本显著低于"一律大模型"。

```
先用小模型尝试
   │
   ▼
评估结果质量
   │ 不达标
   ▼
升级到大模型重试
```

### 5.1.2 实现

```java
public String chatWithEscalation(String question) {
    // ① 先试小模型
    String answer = fastClient.prompt().user(question).call().content();

    // ② 质量检查
    if (qualityChecker.isAcceptable(question, answer)) {
        return answer;                            // 小模型够用
    }
    // ③ 升级
    log.info("小模型结果不达标，升级到大模型");
    return strongClient.prompt().user(question).call().content();
}
```

> 平均成本显著低于"一律用大模型"，质量接近。

## 5.2 质量判定

### 5.2.1 判定方式

| 方式 | 说明 | 成本 |
|------|------|:----:|
| 规则校验 | 格式、长度、关键词 | 0 |
| 置信度阈值 | 分类器分数 | 低 |
| 小模型评审 | 让模型打分 | 低 |
| 自我一致性 | 多次采样比对 | 中 |

### 5.2.2 边界

反例：质量判定本身若依赖大模型，则升级路由的成本优势被抵消。

生产建议：优先用规则与轻量评审，且设置升级率上限，防止退化为"一律大模型"。

# 6. 按任务类型路由

## 6.1 任务映射

### 6.1.1 任务表

结论先行：显式任务类型是最可靠的路由信号，优先级高于长度与关键词。

| 任务类型 | 层级 |
|----------|:----:|
| INTENT_CLASSIFY | FAST |
| SLOT_EXTRACT | FAST |
| QUERY_REWRITE | FAST |
| RAG_ANSWER | STANDARD |
| SUMMARIZE | STANDARD |
| TRANSLATE | STANDARD |
| COMPLEX_REASONING | STRONG |
| CODE_GEN | STRONG |
| AGENT_PLAN | STRONG |

### 6.1.2 实现

```java
public ModelTier routeByTask(TaskType type) {
    return switch (type) {
        case INTENT_CLASSIFY, SLOT_EXTRACT, QUERY_REWRITE -> ModelTier.FAST;
        case RAG_ANSWER, SUMMARIZE, TRANSLATE -> ModelTier.STANDARD;
        case COMPLEX_REASONING, CODE_GEN, AGENT_PLAN -> ModelTier.STRONG;
    };
}
```

## 6.2 任务识别

### 6.2.1 显式标注

结论：能由调用方显式传入任务类型时，直接使用，零误差。

### 6.2.2 隐式识别

结论：无法显式传入时，用轻量分类器从请求中识别任务类型。

```java
interface TaskClassifier {
    @UserMessage("""
            判断任务类型，只输出以下之一：
            INTENT_CLASSIFY / SLOT_EXTRACT / QUERY_REWRITE /
            RAG_ANSWER / SUMMARIZE / TRANSLATE /
            COMPLEX_REASONING / CODE_GEN / AGENT_PLAN
            {{it}}
            """)
    TaskType classify(String request);
}
```

生产建议：把任务类型纳入请求上下文，随链路传递，避免重复识别。

# 7. 动态路由

## 7.1 按租户与预算

### 7.1.1 租户等级

结论先行：按租户等级差异化路由，既能控成本又能支撑商业化分层。

### 7.1.2 预算余量

```java
public ModelTier routeDynamic(String question, UserContext user) {
    // ① 租户等级
    if (user.tier() == TenantTier.FREE) {
        return ModelTier.FAST;                   // 免费用户用小模型
    }
    // ② 预算余量
    BigDecimal remaining = budgetService.remaining(user.tenantId());
    if (remaining.compareTo(LOW_THRESHOLD) < 0) {
        return ModelTier.FAST;                   // 预算紧张，降级
    }
    // ③ 常规路由
    return routeByRule(question, TaskContext.of(user));
}
```

结论：预算逼近上限时自动下沉，避免单租户透支整体预算。

## 7.2 按负载

### 7.2.1 过载降级

结论：系统过载时主动把部分请求下沉到小模型，保障可用性优先于质量。

### 7.2.2 时延预算

结论：对延迟敏感的场景，按时延预算选层——预算紧则用 FAST。

生产建议：动态路由的降级决策必须打点，便于事后评估质量波动。

# 8. 路由的可观测与评测

## 8.1 可观测

### 8.1.1 指标

结论先行：路由若不监控，就无法判断它是在降本还是在降质。

```java
registry.counter("model.routing",
        "tier", tier.name(),
        "task", taskType.name()).increment();
```

| 指标 | 用途 |
|------|------|
| 各层级调用占比 | 评估路由效果 |
| 各层级平均成本 | 降本幅度 |
| 升级率 | 小模型达标率 |
| 各层级质量分 | 验证降级不影响质量 |

### 8.1.2 分布分析

结论：按租户、任务、时段拆解占比，可发现路由倾斜与异常。

## 8.2 评测

### 8.2.1 分层评测

```java
public void evaluateRouting(List<EvalCase> cases) {
    for (ModelTier tier : ModelTier.values()) {
        double accuracy = evaluateWithTier(cases, tier);
        BigDecimal cost = estimateCost(cases, tier);
        log.info("tier={} accuracy={} cost={}", tier, accuracy, cost);
    }
}
```

> 用评测数据验证"小模型是否够用"。

### 8.2.2 升级率

结论：升级率过高说明路由过于激进，过低则说明下沉不足，应维持在目标区间。

生产建议：把升级率纳入 SLO，偏离区间即触发评审。

# 9. 边界与反模式

## 9.1 常见反模式

### 9.1.1 一律大模型

结论：不做路由是最大的成本浪费，尤其在批量任务中。

### 9.1.2 路由震荡

结论：同一请求在不同层之间反复跳变，会导致质量与成本都不可控。

反例：分类器对小模型输出不稳定，同一问题时而 FAST 时而 STRONG，用户体验波动。

生产建议：为分类阈值设置滞回区间（hysteresis），如进入 STRONG 用 0.7、退回 STANDARD 用 0.5，避免临界抖动。

## 9.2 质量风险

### 9.2.1 降级影响质量

结论：任何下沉都必须经过质量验证，不能只看成本。

### 9.2.2 评测缺失

结论：没有评测的路由等于盲飞，上线后无法回答"小模型到底行不行"。

生产建议：为路由建立回归评测集，每次调整模型或规则都重跑。

# 10. 生产注意事项

## 10.1 风险清单

### 10.1.1 风险表

| 风险 | 建议 |
|------|------|
| 一律用大模型 | 分层路由 |
| 路由判断错误 | 规则 + 模型分类 |
| 降级影响质量 | 评测验证 |
| 无升级机制 | 先便宜后升级 |
| 路由不可观测 | 监控各层占比 |
| 免费用户滥用 | 按租户等级路由 |
| 路由震荡 | 稳定分类器 + 阈值滞回 |

### 10.1.2 反模式

- 只看成本不看质量：下沉后投诉率上升。
- 硬编码模型名：换模型要改多处代码。
- 无灰度：规则一改全量生效，风险高。

## 10.2 生产建议

### 10.2.1 检查清单

- 层级与模型是否配置化？
- 规则与分类器是否有评测？
- 是否监控各层占比与升级率？
- 是否有升级上限与灰度？

### 10.2.2 落地顺序

结论：先规则路由，再小模型分类兜底，最后上动态路由与升级路由。

| 阶段 | 目标 | 风险 |
|------|------|:----:|
| 规则路由 | 覆盖明确任务 | 低 |
| 分类兜底 | 覆盖模糊请求 | 中 |
| 升级路由 | 质量兜底 | 中 |
| 动态路由 | 成本与负载自适应 | 高 |

# 11. 小结

## 11.1 核心要点

- 模型分层路由是降本第一手段。
- 路由策略：规则（零成本）、模型分类、动态路由。
- 升级路由：先小模型，不达标再升级。
- 按任务类型与租户等级差异化路由。
- 路由必须可观测与评测。

## 11.2 落地清单

- 抽象三层模型，配置化映射。
- 规则优先，分类器兜底。
- 建立分层评测与升级率 SLO。
- 动态路由决策全程打点。
