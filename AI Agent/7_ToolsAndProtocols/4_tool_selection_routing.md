# 1. 工具过多的代价

结论先行：挂载的工具越多，提示（Prompt）越长、选择越难、误用越多、延迟越高，因此单次挂载的工具数量必须受控。

## 1.1 代价拆解

### 1.1.1 四类代价

工具数量增长会同时放大四类成本：

```
工具数量 ↑
  ├── Schema 占用 Token ↑（每个工具的描述都进提示）
  ├── 选择难度 ↑（相似工具易混淆）
  ├── 误用概率 ↑
  └── 延迟 ↑（提示更长）
```

### 1.1.2 量化示例

| 工具数 | 单个 Schema 约 | 提示增量 | 典型观察 |
|--------|----------------|----------|----------|
| 5 | 120 Token | ~600 Token | 选择准确率高 |
| 15 | 120 Token | ~1800 Token | 偶有混淆 |
| 40 | 120 Token | ~4800 Token | 误用明显上升 |
| 100 | 120 Token | ~12000 Token | 提示成本与误用双高 |

> 表中为经验量级，实际值取决于描述长度与所用模型。

### 1.1.3 成本曲线

```
准确率
  │  ▄▄▄▄▄▄▄▄
  │ █        ██▄
  │█            ███▄
  │                  ████▄▄
  └──────────────────────────▶ 工具数量
     5   15   30   50   80
        （甜点区）    （急剧下降）
```

成本随工具数近似线性增长，而准确率在越过甜点区后急剧下降。

## 1.2 挂载数量经验值

### 1.2.1 阈值建议

经验：**单次挂载建议不超过 10~15 个工具**。超过后优先考虑分组、裁剪、检索或合并。

### 1.2.2 边界与坑

- 阈值不是硬约束：描述极短且差异极大时 20 个也可用；反之 8 个相似工具也会混淆。
- 只数工具个数、不看描述长度：真正占用 Token 的是描述文本。
- 忽略下游延迟：提示变长会抬高首字延迟（TTFT）。

# 2. 减少工具的四种策略

结论先行：减少工具的手段有四种——按场景分组、按权限裁剪、工具检索、工具合并，它们可组合使用。

## 2.1 策略总览

### 2.1.1 策略表

| 策略 | 说明 | 触发条件 |
|------|------|----------|
| 按场景分组 | 不同场景挂载不同工具集 | 场景边界清晰 |
| 按权限裁剪 | 只挂用户有权限的 | 有用户体系 |
| 工具检索 | 先筛选相关工具再挂载 | 工具数量很多 |
| 工具合并 | 相似工具合并为一个 | 存在同族工具 |

### 2.1.2 选择决策树

```
工具总数 > 15?
  ├─ 否 → 直接挂载（最多加权限裁剪）
  └─ 是 → 场景是否可枚举?
            ├─ 是 → 按场景分组
            └─ 否 → 工具检索
```

## 2.2 组合使用

### 2.2.1 分层组合

生产上通常是分层组合：先按权限裁剪，再按场景分组，最后对候选做检索。

### 2.2.2 边界与坑

- 只做分组不做权限：越权工具仍会进入提示。
- 只做检索不做分组：每次请求都多一次模型调用。
- 合并过度：单个工具参数爆炸，模型填参错误上升。

### 2.2.3 收益优先级

| 顺序 | 策略 | 成本 | 收益 |
|:----:|------|:----:|:----:|
| 1 | 权限裁剪 | 低 | 高 |
| 2 | 场景分组 | 中 | 高 |
| 3 | 工具合并 | 中 | 中 |
| 4 | 工具检索 | 高 | 中高 |

> 先做低成本高收益的裁剪与分组，最后再考虑检索。

# 3. 按场景分组

结论先行：按业务场景把工具切分成互不重叠的集合，请求时只挂当前场景的集合。

## 3.1 场景路由实现

### 3.1.1 场景枚举与映射

用枚举声明场景，用映射表声明每个场景可用的工具名。

```java
public enum Scenario { ORDER, REFUND, PRODUCT, SUPPORT }
```

### 3.1.2 完整 Java 示例

```java
public class ScenarioToolRouter {

    private static final Map<Scenario, Set<String>> SCENARIO_TOOLS = Map.of(
            Scenario.ORDER,   Set.of("queryOrder", "cancelOrder", "urgeOrder"),
            Scenario.REFUND,  Set.of("queryOrder", "applyRefund", "queryRefund"),
            Scenario.PRODUCT, Set.of("searchProduct", "queryStock", "queryPrice"),
            Scenario.SUPPORT, Set.of("createTicket", "queryTicket", "queryPolicy")
    );

    private final List<ToolCallback> allTools;
    private final PermissionService permissionService;

    public ScenarioToolRouter(List<ToolCallback> allTools,
                              PermissionService permissionService) {
        this.allTools = allTools;
        this.permissionService = permissionService;
    }

    public List<ToolCallback> route(Scenario scenario, UserContext user) {
        Set<String> allowed = SCENARIO_TOOLS.get(scenario);
        return allTools.stream()
                .filter(t -> allowed.contains(t.name()))
                .filter(t -> permissionService.canUse(user, t.name()))
                .toList();
    }
}
```

## 3.2 场景识别的来源

### 3.2.1 显式与隐式

| 来源 | 说明 | 可靠性 |
|------|------|--------|
| 入口显式 | 用户从"退款"入口进入 | 高 |
| 会话上下文 | 上文已确定场景 | 高 |
| 模型分类 | 由意图分类得到 | 中 |
| 关键词规则 | 命中关键词 | 低到中 |

### 3.2.2 边界与坑

- 场景之间工具重叠：同一工具出现在多个场景是正常的，但要注意去重。
- 场景遗漏：出现新场景时若无兜底，会挂载空工具集。
- 场景过细：场景过多等于没有分组。

# 4. 意图路由

结论先行：先判断意图类别，再挂载该类别对应的工具集；粗分类用规则，复杂情况用小模型。

## 4.1 两级路由

### 4.1.1 规则粗分类

关键词或正则零成本，先过滤明显意图（如命中"退款"直接判定为退款场景）。

```java
public Scenario roughClassify(String question) {
    if (question.contains("退款") || question.contains("退货")) {
        return Scenario.REFUND;
    }
    if (question.contains("物流") || question.contains("快递")) {
        return Scenario.ORDER;
    }
    return null;   // 规则无法判定，交给小模型
}
```

### 4.1.2 小模型细分类

规则无法判定时调用小模型，输出枚举之一，成本低、延迟小。

## 4.2 完整 Java 示例

### 4.2.1 LangChain4j 风格

```java
interface IntentClassifier {
    @UserMessage("""
            判断用户意图类别，只输出其中一个：
            ORDER / REFUND / PRODUCT / SUPPORT / CHITCHAT
            用户：{{it}}
            """)
    String classify(String question);
}

public String handle(String question, UserContext user) {
    String intent = classifier.classify(question);

    if ("CHITCHAT".equals(intent)) {
        return chatClient.prompt().user(question).call().content();   // 不挂工具
    }

    Scenario scenario = Scenario.valueOf(intent);
    List<ToolCallback> tools = router.route(scenario, user);

    return chatClient.prompt()
            .user(question)
            .toolCallbacks(tools)
            .call()
            .content();
}
```

### 4.2.2 边界与坑

- 分类结果非法：模型可能输出枚举外的词，需兜底到默认场景。
- 分类延迟：每次请求多一次模型调用，可缓存同类问句。
- 多意图：一句话可能同时含"查订单"和"退款"，单一枚举会丢信息，需支持多标签。

# 5. 工具检索（Tool Search）

结论先行：工具很多时，先用一次调用筛出与问题最相关的若干工具，再挂载给主模型。

## 5.1 检索机制

### 5.1.1 语义检索与 LLM 筛选

| 方式 | 原理 | 成本 | 适用 |
|------|------|------|------|
| 向量检索 | 工具描述向量化后相似度匹配 | 低 | 工具上千 |
| LLM 筛选 | 让模型从列表里挑 | 中 | 工具几十到几百 |
| 关键词 | 名称/描述匹配 | 极低 | 命名规范时 |

### 5.1.2 完整 Java 示例

```java
interface ToolSelector {
    @UserMessage("""
            从下面的工具列表中选出与用户问题最相关的工具（最多 5 个）。
            只输出工具名，逗号分隔。

            可用工具：
            {{tools}}

            用户问题：{{question}}
            """)
    String selectTools(@V("tools") String tools, @V("question") String question);
}

public List<ToolCallback> selectRelevant(String question, UserContext user) {
    List<ToolCallback> candidates = permissionService.allowedTools(user);

    String toolList = candidates.stream()
            .map(t -> t.name() + ": " + t.description())
            .collect(Collectors.joining("\n"));

    String selected = selector.selectTools(toolList, question);
    Set<String> names = Set.of(selected.split(",\\s*"));

    return candidates.stream()
            .filter(t -> names.contains(t.name()))
            .toList();
}
```

## 5.2 成本与召回

### 5.2.1 成本表

| 项 | 说明 |
|----|------|
| 额外调用 | 每次请求多一次 LLM 调用 |
| 召回风险 | 筛错会直接丢掉正确工具 |
| 缓存机会 | 相同问题可缓存筛选结果 |

### 5.2.2 边界与坑

- 只挂筛选结果：若筛选失败会无工具可用，需保留核心工具的兜底。
- 筛选数量过多：选出 20 个等于没筛。
- 不记录筛选日志：无法评估筛选准确率。

### 5.2.3 兜底策略

```java
public List<ToolCallback> selectWithFallback(String question, UserContext user) {
    List<ToolCallback> selected = selectRelevant(question, user);
    if (selected.isEmpty()) {
        return coreTools;      // 筛选为空时退回核心工具集
    }
    return selected;
}
```

# 6. 工具合并

结论先行：把同一实体的多个查询或相似操作合并为一个工具，用参数区分，减少工具数量。

## 6.1 合并模式

### 6.1.1 合并场景表

| 合并场景 | 说明 | 示例 |
|----------|------|------|
| 同一实体不同字段 | 合并为"查询详情" | 订单状态/金额/商品 |
| 相似操作 | 用枚举参数区分 | 上架/下架 |
| 分页/排序变体 | 合并为可选参数 | 列表查询 |

### 6.1.2 Java 示例

```java
// 反例：多个相似工具
// @Tool("查询订单状态") ...
// @Tool("查询订单金额") ...
// @Tool("查询订单商品") ...

// 正例：合并为一个
@Tool("查询订单详情（含状态、金额、商品、时间）")
OrderDetail getOrderDetail(@P("订单号") String orderId);
```

## 6.2 合并的代价

### 6.2.1 参数膨胀

合并后参数变多，模型填参难度上升，需用参数描述与默认值降低负担。

```java
@Tool("管理商品上下架")
void manageProduct(
        @P("商品ID") String productId,
        @P("动作：ON=上架, OFF=下架") String action);
```

### 6.2.2 边界与坑

- 把不相关工具硬合并：参数语义混乱，模型更易填错。
- 合并后权限无法细分：一个大工具含多动作，权限只能整体授予。
- 合并后返回体过大：所有字段一次返回会浪费 Token，需支持字段筛选。

### 6.2.3 合并前后对比

| 维度 | 合并前 | 合并后 |
|------|--------|--------|
| 工具数 | 3 | 1 |
| 参数数 | 各 1 个 | 2 个 |
| 提示 Token | 高 | 低 |
| 填参难度 | 低 | 中 |
| 权限粒度 | 细 | 粗 |

# 7. 描述消歧

结论先行：相似工具必须通过描述互相说明"不适用"的场景，这是最有效的消歧手段。

## 7.1 描述三要素

### 7.1.1 适用/不适用/示例

| 要素 | 作用 |
|------|------|
| 适用 | 明确何时用 |
| 不适用 | 明确何时别用、该用哪个 |
| 示例 | 给出典型问法 |

### 7.1.2 Java 示例

```java
@Tool("查询订单详情。适用：问订单状态/金额/商品。不适用：查物流（用 queryLogistics）")
Order getOrder(String orderId);

@Tool("查询物流轨迹。适用：问包裹到哪了/物流进度。不适用：查订单（用 getOrder）")
Logistics queryLogistics(String waybill);
```

## 7.2 命名与描述

### 7.2.1 命名规范

工具名用动词加名词、避免缩写、避免同义近义名。

| 差 | 好 |
|----|----|
| query1 | queryOrder |
| getInfo | getOrderDetail |
| doThing | cancelOrder |

### 7.2.2 边界与坑

- 描述只写"查询订单"：两个查询类工具无法区分。
- 描述过长：挤占 Token，应精炼。
- 中英混杂：模型理解尚可，但团队维护困难，建议统一。

# 8. 工具选择的评测

结论先行：工具路由必须可评测，核心指标是"期望工具是否出现在挂载列表中"（召回率）与"是否挂载了不该挂的工具"（精确率）。

## 8.1 评测指标

### 8.1.1 指标表

| 指标 | 定义 |
|------|------|
| 召回率 | 期望工具在挂载集合中的比例 |
| 精确率 | 挂载集合中真正需要的比例 |
| 误选率 | 挂载了不该挂的高风险工具 |

### 8.1.2 完整 Java 示例

```java
public void evaluateToolRouting(List<EvalCase> cases) {
    int correct = 0;

    for (EvalCase c : cases) {
        List<ToolCallback> tools = router.route(c.scenario(), c.user());
        Set<String> names = tools.stream().map(ToolCallback::name)
                .collect(Collectors.toSet());

        // 期望工具是否都在挂载列表中
        if (names.containsAll(c.expectedActions())) {
            correct++;
        }
    }
    System.out.printf("工具路由召回率: %.2f%n", correct / (double) cases.size());
}
```

## 8.2 评测集构建

### 8.2.1 用例设计

用例应覆盖：典型问法、歧义问法、越权问法、跨场景问法。

```java
public record EvalCase(
        String question,
        Scenario scenario,
        UserContext user,
        Set<String> expectedActions
) {}
```

### 8.2.2 边界与坑

- 只测"能否回答"：回答对但用了错工具无法发现。
- 用例太少：几十条无法覆盖歧义。
- 不回归：新增工具后不重跑评测，路由会悄悄退化。

### 8.2.3 端到端评测

路由召回只是第一层，还需评测"最终是否调用了正确工具"，可用轨迹（Trace）比对实现。

```java
public double toolPrecision(List<Trace> traces) {
    long hit = traces.stream()
            .filter(t -> t.actualTools().containsAll(t.expectedTools()))
            .count();
    return hit / (double) traces.size();
}
```

# 9. 动态工具集

结论先行：工具集应随会话上下文动态变化——已完成的任务去掉对应工具，未满足前置条件的工具不挂。

## 9.1 上下文驱动裁剪

### 9.1.1 状态驱动

| 状态 | 裁剪 |
|------|------|
| 已退款成功 | 去掉退款工具 |
| 订单未确认 | 去掉发货工具 |
| 无权限 | 去掉对应工具 |
| 流程结束 | 去掉流程内工具 |

### 9.1.2 完整 Java 示例

```java
public List<ToolCallback> dynamicTools(AgentContext ctx, UserContext user) {
    Set<String> allowed = permissionService.allowedTools(user);

    // 已退款成功 → 移除退款工具
    if (ctx.hasCompleted("applyRefund")) {
        allowed.remove("applyRefund");
    }
    // 订单未确认 → 移除发货工具
    if (!ctx.isOrderConfirmed()) {
        allowed.remove("shipOrder");
    }
    return filterTools(allowed);
}
```

## 9.2 边界与坑

### 9.2.1 缓存与一致性

动态工具集会使提示前缀变化，影响提示缓存命中，需权衡变化频率与缓存收益。

### 9.2.2 边界与坑

- 动态裁剪后不可逆：误删工具会导致模型无法完成任务，需保留兜底。
- 状态读取延迟：上下文与真实状态不一致时会挂错工具。
- 缓存前缀失效：频繁变化的工具集会降低缓存收益。

### 9.2.3 与权限裁剪的区别

| 维度 | 权限裁剪 | 动态裁剪 |
|------|----------|----------|
| 依据 | 用户身份 | 会话状态 |
| 变化频率 | 低 | 高 |
| 安全性质 | 强制 | 优化 |
| 缓存影响 | 小 | 大 |

# 10. 生产注意事项

结论先行：上线前核对工具数量、分组、消歧、评测与裁剪五件事。

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 工具过多 | 分组/检索/合并 |
| 相似工具混淆 | 描述消歧 |
| 全量挂载 | 按权限与场景裁剪 |
| 意图误判 | 规则 + 小模型 |
| 工具检索成本 | 仅工具多时使用 |
| 无评测 | 测路由召回率 |

## 10.2 部署检查表

- 单次挂载工具数在 10~15 以内。
- 高风险工具默认不挂，需显式场景才挂。
- 相似工具描述包含"不适用"说明。
- 路由评测纳入回归。
- 动态裁剪有兜底工具集。

# 11. 小结

## 11.1 核心要点

- 工具过多会膨胀提示、增加误用。
- 四种策略：场景分组、权限裁剪、工具检索、工具合并。
- 意图路由先分类再挂载工具。
- 相似工具靠描述互相消歧。
- 工具路由需要评测。

## 11.2 落地建议

1. 先从权限裁剪做起，成本最低、收益明确。
2. 场景边界清晰时优先按场景分组。
3. 工具超过几十个再引入检索。
4. 每条路由规则都要有用例，纳入回归。
