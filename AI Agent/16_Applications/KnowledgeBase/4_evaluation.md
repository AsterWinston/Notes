# 1. 评测体系总览

**结论先行**：知识库问答（RAG, Retrieval-Augmented Generation，检索增强生成）的质量不能只看最终答案，必须分层评测：检索层回答"找没找到"，生成层回答"答得忠不忠实"，端到端层回答"用户是否被正确解决"，效率层回答"是否可负担"。

## 1.1 评测分层

| 层次 | 关注问题 | 代表指标 | 数据来源 |
|------|----------|----------|----------|
| 检索 | 相关文档是否召回 | Recall@5、MRR、nDCG | 标注的 relevantDocIds |
| 生成 | 答案是否忠于上下文 | 忠实度、相关性、引用准确率 | LLM 评审 + 规则 |
| 端到端 | 用户问题是否被正确解决 | 答案正确率、拒答准确率、越权泄露率 | 人工 + 规则 |
| 效率 | 是否可用可负担 | 延迟 P95/P99、单次成本 | 埋点 |

### 1.1.1 检索层

检索层是 RAG 的地基。检索失败时，再强的生成模型也无法补救。检索指标依赖标注数据 `relevantDocIds`，因此评测集必须包含文档 ID 标注。

### 1.1.2 生成层

生成层评测答案与上下文的"接地"（grounding）程度。常用 LLM-as-Judge（大模型评审）打分，辅以引用校验等确定性规则。

### 1.1.3 端到端层

端到端层最贴近用户价值，但成本最高。通常采样评测，覆盖事实问答、流程问答、无答案、权限、边界等类别。

## 1.2 指标定义与公式

| 指标 | 公式 | 说明 |
|------|------|------|
| Recall@K | 命中相关文档数 / 全部相关文档数 | 只看前 K 条 |
| MRR | 1 / 首个相关文档排名 | 反映首个命中的位置 |
| nDCG@K | DCG@K / IDCG@K | 考虑排序位置与相关性等级 |
| 忠实度 | 被上下文支持的句子数 / 答案句子数 | 越高越"接地" |
| 引用准确率 | 正确引用数 / 全部引用数 | 引用是否指向真实来源 |
| 拒答准确率 | 正确拒答数 / 无答案问题数 | 防止幻觉 |
| 越权泄露率 | 越权返回次数 / 权限测试数 | 必须为 0 |

## 1.3 评测驱动开发流程

```
① 定义指标与阈值
② 构建评测集
③ 实现自动化评测
④ 每次改动跑回归
⑤ 达标 → 灰度 → 上线
```

**边界与坑**：指标之间会互相冲突，例如提高召回会拉低精度、增强拒答会拉低答案覆盖率，必须成对观测。

**生产建议**：把评测做成 CI（Continuous Integration，持续集成）的一环，任何提示词、分块、模型变更都触发回归，未达标禁止合并。

# 2. 检索指标

**结论先行**：检索指标回答"该找到的文档有没有找到、排得够不够前"。它们是最早能发现问题的信号。

## 2.1 Recall@K 与 MRR

### 2.1.1 定义

`Recall@5` 表示前 5 条结果中覆盖了多少比例的相关文档。`MRR`（Mean Reciprocal Rank，平均倒数排名）只看第一个相关文档的位置，适合"只要有一条对就够"的场景。

### 2.1.2 Java 实现

```java
public double recallAt(List<String> retrieved, List<String> relevant) {
    if (relevant == null || relevant.isEmpty()) return 0.0;
    Set<String> rel = new HashSet<>(relevant);
    long hit = retrieved.stream().limit(5).filter(rel::contains).count();
    return (double) hit / relevant.size();
}

public double mrr(List<String> retrieved, List<String> relevant) {
    Set<String> rel = new HashSet<>(relevant);
    for (int i = 0; i < retrieved.size(); i++) {
        if (rel.contains(retrieved.get(i))) {
            return 1.0 / (i + 1);
        }
    }
    return 0.0;
}
```

## 2.2 nDCG

`nDCG@K`（Normalized Discounted Cumulative Gain，归一化折损累计增益）支持"相关性分级"，比二值 Recall 更细腻。

```java
public double ndcgAt(List<String> retrieved, Map<String, Integer> gains, int k) {
    double dcg = 0.0;
    for (int i = 0; i < Math.min(k, retrieved.size()); i++) {
        int g = gains.getOrDefault(retrieved.get(i), 0);
        dcg += (Math.pow(2, g) - 1) / (Math.log(i + 2) / Math.log(2));
    }
    double idcg = idealDcg(gains, k);
    return idcg == 0 ? 0.0 : dcg / idcg;
}
```

## 2.3 指标选择建议

| 场景 | 推荐指标 |
|------|----------|
| 单跳事实问答 | Recall@5 + MRR |
| 多跳/综合 | nDCG@10 |
| 首条即答 | MRR |

**边界与坑**：若一个问题的答案分散在多篇文档，Recall@5 可能被"文档粒度"稀释，需按"是否覆盖所有必需文档"单独统计。

**生产建议**：检索指标要按类别（事实/流程/权限）分别统计，避免平均值掩盖短板。

# 3. 生成指标

**结论先行**：生成指标回答"答案是否忠于检索到的证据、是否切题、引用是否真实"。它是防幻觉的核心防线。

## 3.1 忠实度

忠实度（Faithfulness）衡量答案中每个论断是否被上下文支持。做法是把答案拆句，逐句让评审模型判断"能否由上下文推出"。

```java
public double faithfulness(String answer, String context) {
    List<String> claims = splitClaims(answer);
    int supported = 0;
    for (String claim : claims) {
        JudgeVerdict v = judge.judge(
                "上下文能否支持该论断？只回 YES/NO。\n上下文：" + context
                        + "\n论断：" + claim);
        if (v.isYes()) supported++;
    }
    return claims.isEmpty() ? 1.0 : (double) supported / claims.size();
}
```

## 3.2 相关性

相关性（Relevance）衡量答案是否正面回应用户问题，避免"答非所问但句句有据"。

## 3.3 引用准确率

引用准确率检查答案中的引用标记 `[1]` 是否指向真实且相关的来源文档。

| 检查项 | 说明 |
|--------|------|
| 引用存在 | 每个事实句都有引用 |
| 引用有效 | 引用 ID 存在于来源列表 |
| 引用相关 | 被引文档确实支持该句 |

## 3.4 LLM-as-Judge

用强模型做评审，需要固定评审提示、温度设为 0，并对评审模型本身做一致性校验（如同一答案多次评分方差）。

**边界与坑**：评审模型有位置偏好和长度偏好，需打乱候选顺序、控制答案长度差异。

**生产建议**：确定性规则（引用校验、关键词 mustContain）与 LLM 评审结合，前者做门禁，后者做趋势。

# 4. 端到端指标

**结论先行**：端到端指标最接近用户价值，也最难自动化，通常以采样加规则的方式落地。

## 4.1 答案正确率

以 `groundTruth` 为准，判断答案是否包含关键事实。可用 `mustContain` 关键词做快速判定，人工复核疑难样例。

## 4.2 拒答准确率

无答案问题（如"火星办事处地址"）必须拒答而非编造。拒答准确率 = 正确拒答数 / 无答案问题数。

## 4.3 越权泄露率

权限类问题必须以当前用户权限为准，越权返回次数必须为 0，这是硬性红线。

| 指标 | 目标 | 是否门禁 |
|------|------|:--------:|
| 答案正确率 | ≥ 90% | 是 |
| 拒答准确率 | ≥ 95% | 是 |
| 越权泄露率 | 0 | 是 |

**边界与坑**：拒答率与答案正确率此消彼长，过度拒答会伤害体验，需同时看"漏答"与"错答"。

**生产建议**：越权测试要覆盖"同部门可见""跨部门不可见""管理员可见"三类账号。

# 5. 评测集构建

**结论先行**：评测集的质量决定评测的价值。它必须覆盖正常、无答案、权限、边界四类，并长期维护。

## 5.1 数据模型

```json
[
  {
    "id": "k1",
    "question": "差旅费报销时限是多久？",
    "groundTruth": "费用发生后 30 天内",
    "relevantDocIds": ["handbook-3-2"],
    "mustContain": ["30 天"],
    "category": "policy"
  },
  {
    "id": "k2",
    "question": "公司的火星办事处地址",
    "groundTruth": null,
    "category": "no-answer"
  },
  {
    "id": "k3",
    "question": "财务部的报销审批流程",
    "relevantDocIds": ["finance-1-2"],
    "category": "permission",
    "user": {"department": "HR"}
  }
]
```

## 5.2 类别分布

| 类别 | 占比 | 目的 |
|------|:----:|------|
| 事实问答 | 40% | 基础能力 |
| 流程问答 | 20% | 多步推理 |
| 无答案 | 15% | 防幻觉 |
| 权限 | 15% | 防越权 |
| 边界 | 10% | 鲁棒性 |

## 5.3 数据来源与标注

| 来源 | 说明 |
|------|------|
| 真实工单 | 从客服/IT 工单提炼 |
| 专家编写 | 覆盖政策、流程 |
| 失败案例 | 线上 badcase 回填 |
| 合成 | 模型生成后人工校验 |

**边界与坑**：评测集会随知识库更新而过期，`relevantDocIds` 指向的文档可能被删除或改写，需定期校验有效性。

**生产建议**：为评测集打版本号，与知识库快照绑定，保证评测可复现。

# 6. 自动化评测

**结论先行**：自动化评测把"人工抽查"变成"每次改动必跑"的回归，是质量可持续的保障。

## 6.1 评测执行器

```java
public KbEvalReport evaluate(List<KbEvalCase> cases) {
    List<Double> recall = new ArrayList<>();
    List<Double> faithfulness = new ArrayList<>();
    int correctRefusal = 0, noAnswerTotal = 0, permissionLeak = 0;

    for (KbEvalCase c : cases) {
        QaResult result = qaService.ask(c.question(), c.user());

        if (c.relevantDocIds() != null) {
            recall.add(recallAt(result.retrievedIds(), c.relevantDocIds()));
        }
        if ("no-answer".equals(c.category())) {
            noAnswerTotal++;
            if (result.isRefusal()) correctRefusal++;
        }
        if ("permission".equals(c.category())
                && result.sourcesContain(c.forbiddenDocIds())) {
            permissionLeak++;
        }
        faithfulness.add(judge.faithfulness(result.answer(), result.context()));
    }

    return new KbEvalReport(
            avg(recall), avg(faithfulness),
            correctRefusal, noAnswerTotal, permissionLeak);
}
```

## 6.2 报告与阈值

```java
public record KbEvalReport(
        double recallAt5,
        double faithfulness,
        int correctRefusal,
        int noAnswerTotal,
        int permissionLeak) {

    public boolean pass() {
        return recallAt5 >= 0.85
                && faithfulness >= 0.90
                && permissionLeak == 0
                && refusalRate() >= 0.95;
    }

    public double refusalRate() {
        return noAnswerTotal == 0 ? 1.0
                : (double) correctRefusal / noAnswerTotal;
    }
}
```

## 6.3 回归测试

| 触发条件 | 动作 |
|----------|------|
| 提示词变更 | 跑全量评测 |
| 分块/Embedding 变更 | 跑全量 + 重建索引 |
| 模型升级 | 跑全量 + 对比基线 |
| 知识库更新 | 跑受影响类别 |

**边界与坑**：LLM 评审有随机性，回归对比要用相同评审提示与温度，并关注多次运行的方差。

**生产建议**：把 `pass()` 作为 CI 门禁，报告存档，支持历史趋势对比。

# 7. 分块策略对比

**结论先行**：分块（Chunking）直接决定检索上限。不同策略没有绝对优劣，必须用对比实验选择。

## 7.1 策略定义

| 策略 | 参数 | 特点 |
|------|------|------|
| 固定长度 | 500/50 | 简单，易切断语义 |
| 固定长度 | 1000/100 | 上下文更全，噪声更多 |
| 递归分割 | 800/100 | 按标题/段落递归切 |
| 递归+标题 | 800/100 | 每块带标题路径 |

## 7.2 对比实验

```java
@Test
void compare_chunking_strategies() {
    List<ChunkingStrategy> strategies = List.of(
            fixed(500, 50),
            fixed(1000, 100),
            recursive(800, 100),
            recursiveWithTitle(800, 100));

    for (ChunkingStrategy s : strategies) {
        double recall = evaluateRecall(cases, s);
        log.info("策略 {}: Recall@5 = {}", s.name(), recall);
    }
}
```

## 7.3 结论

经验上"递归+标题"通常最优，因为标题路径提供了额外语义锚点。但需在自己的语料上验证。

**边界与坑**：分块大小与 Embedding 模型的最大输入长度、重排模型的输入长度都相关，改动需联动。

**生产建议**：把分块参数纳入配置，支持 A/B 对比与快速回滚。

# 8. 上线检查清单

**结论先行**：上线前必须逐项确认质量、安全、性能三类门禁全部通过。

## 8.1 质量门禁

- [ ] 解析正确率 > 95%
- [ ] Recall@5 ≥ 85%
- [ ] 忠实度 ≥ 90%
- [ ] 无答案正确拒答
- [ ] 引用可溯源

## 8.2 安全与权限

- [ ] 越权测试通过（零泄露）
- [ ] 敏感信息脱敏

## 8.3 性能与成本

- [ ] 增量更新生效
- [ ] 延迟 < 3s
- [ ] 成本在预算内

**边界与坑**：检查清单容易被当成一次性动作，实际上每次重大变更都应重新过一遍。

**生产建议**：把清单固化为发布流水线的检查项，未勾选不允许发布。

# 9. 灰度上线

**结论先行**：灰度（Canary Release，金丝雀发布）用小流量验证线上真实表现，是降低风险的必备环节。

## 9.1 阶段划分

```
阶段1：内部小范围（HR + IT）
阶段2：5% 员工
阶段3：20%
阶段4：全员
```

## 9.2 观测指标

观察：召回率、忠实度、用户反馈、越权。

| 阶段 | 停留条件 | 晋级条件 |
|------|----------|----------|
| 阶段1 | ≥ 3 天 | 无严重问题 |
| 阶段2 | ≥ 3 天 | 指标达标 |
| 阶段3 | ≥ 5 天 | 指标稳定 |
| 阶段4 | 持续 | — |

## 9.3 回滚策略

保留上一版本索引与配置，出现越权或指标骤降时一键回滚。

**边界与坑**：灰度用户可能与全量用户分布不同，指标不可直接外推。

**生产建议**：灰度期间安排人工值班，快速响应反馈。

# 10. 监控告警

**结论先行**：上线后靠监控发现退化，越权与检索退化是最关键的信号。

## 10.1 指标与阈值

| 指标 | 告警 |
|------|------|
| 检索命中率 | 突降 |
| 拒答率 | 突增（可能索引问题） |
| 越权访问 | 任意（严重） |
| 延迟 P99 | > 5s |
| 成本 | 超预算 |

## 10.2 告警分级

| 级别 | 场景 | 响应 |
|------|------|------|
| P0 | 越权泄露 | 立即回滚 |
| P1 | 检索命中率骤降 | 30 分钟内排查 |
| P2 | 延迟升高 | 当日处理 |
| P3 | 成本上涨 | 周内优化 |

## 10.3 根因定位

```java
public void diagnose(String traceId) {
    QaTrace t = traceStore.get(traceId);
    log.info("query={} rewrite={}", t.query(), t.rewrittenQuery());
    log.info("retrieved={} scores={}", t.retrievedIds(), t.scores());
    log.info("reranked={} threshold={}", t.rerankedIds(), t.threshold());
    log.info("answer={} citations={}", t.answer(), t.citations());
}
```

**边界与坑**：只监控端到端指标会掩盖分层问题，必须保留检索、重排、生成的中间信号。

**生产建议**：为每个请求保留可回放的 Trace，支持按 traceId 复现。

# 11. 持续优化

**结论先行**：优化不是拍脑袋调参，而是"收集失败案例 → 归因 → 优化 → 评测 → 灰度"的闭环。

## 11.1 失败案例归因

```
① 收集失败案例（答不出/答错）
② 归因：检索失败 or 生成失败
③ 优化：分块 / Embedding / 重排 / 提示
④ 评测验证
⑤ 灰度上线
```

## 11.2 优化手段矩阵

| 优化方向 | 手段 |
|----------|------|
| 召回 | 分块、查询改写、混合检索 |
| 精度 | 重排、阈值 |
| 忠实度 | 提示约束、引用校验 |
| 延迟 | 缓存、并行、索引调优 |

## 11.3 迭代节奏

| 周期 | 动作 |
|------|------|
| 每日 | 收集 badcase |
| 每周 | 归因 + 小优化 |
| 每月 | 评测集扩充 |
| 每季 | 模型/索引升级 |

**边界与坑**：一次改多个变量会导致无法归因，应遵循"单变量"原则。

**生产建议**：维护优化日志，记录每次变更与指标变化。

# 12. 运维

**结论先行**：RAG 系统的运维重点是索引与 Embedding 的生命周期管理。

## 12.1 知识更新

| 项 | 说明 |
|----|------|
| 知识更新 | 定时增量 + 事件触发 |
| 全量重建 | 新索引 + 别名切换 |

## 12.2 索引重建与别名切换

```java
public void rebuild(String newIndex) {
    indexer.build(newIndex);
    evaluator.run(cases, newIndex);
    if (report.pass()) {
        alias.switchTo(newIndex);
    } else {
        log.warn("新索引未达标，保持旧索引");
    }
}
```

## 12.3 容量与备份

| 项 | 说明 |
|----|------|
| Embedding 升级 | 需全量重建 |
| 容量规划 | 按文档量与 QPS（Queries Per Second，每秒查询数） |
| 备份 | 向量数据也要备份 |

**边界与坑**：别名切换要保证读写一致性，切换瞬间可能出现短暂双写。

**生产建议**：保留最近两个索引版本，便于快速回滚。

# 13. 小结

- 评测分层：检索、生成、端到端、效率。
- 分块策略用对比实验选择。
- 越权测试必须零泄露。
- 灰度上线并监控检索命中率与拒答率。
- 持续优化：归因 → 优化 → 评测 → 灰度。
