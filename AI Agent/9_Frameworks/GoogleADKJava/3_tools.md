# 1. 工具概述与分类

工具（Tool）是 Agent 与外部世界交互的接口。模型本身只能生成文本，是工具让 Agent 能够查库、下单、搜索与执行代码。

## 1.1 结论先行

- 工具 = 名称 + 描述 + 参数 schema + 实现；模型依据描述与 schema 决定是否调用。
- ADK 支持三类工具：注解式工具（`@Tool`）、函数工具（`FunctionTool`）、内置工具（搜索、代码执行等）。
- 工具描述质量直接决定调用准确率，与指令同等重要。
- 工具在"模型—执行—回填"循环中被触发，属于 Agent 的"手和脚"。

## 1.2 机制拆解

### 1.2.1 工具类型对比

| 类型 | 定义方式 | 优点 | 缺点 |
|------|----------|------|------|
| 注解式 | `@Tool` / `@ToolParam` | 声明简洁、可读性好 | 依赖反射 |
| 函数工具 | `FunctionTool.create(...)` | 动态、可编程 | 手写 schema |
| 内置工具 | 框架提供 | 开箱即用 | 可定制性弱 |

### 1.2.2 工具在 Agent 循环中的位置

```
模型输出
   |
   v
是否含 function call? --否--> 最终文本
   | 是
   v
参数校验(schema) --> 鉴权 --> 执行 --> 结果序列化 --> 回填模型
```

- schema 校验失败会返回错误给模型，模型可据此修正参数。
- 工具结果会被序列化进上下文，体积越大越费 token。

### 1.2.3 工具与函数调用的关系

| 概念 | 说明 |
|------|------|
| 工具 | 面向开发者的封装单元 |
| 函数调用 | 面向模型的对齐格式 |
| 工具调用 | ADK 侧执行并回填的过程 |

- 模型看到的是工具名、描述与参数 schema，看不到 Java 实现。
- 因此实现可以任意重构，只要保持"契约"不变。

## 1.3 代码示例

### 1.3.1 一个工具类

```java
public class OrderTools {

    @Tool(description = "按订单号查询订单详情")
    public OrderResult getOrder(
            @ToolParam(description = "订单号，如 A123") String orderId) {
        return orderService.get(orderId);
    }
}
```

## 1.4 边界与坑

- 工具粒度过细会导致模型频繁多次调用，粒度过粗则参数复杂难填。
- 工具名与描述含糊（如 `doThing`）会让模型误触发。
- 工具返回自然语言大段文本会污染上下文，应返回结构化精简数据。

## 1.5 生产建议

- 按"业务动作"而非"底层 API"设计工具粒度。
- 工具名用动词短语（`getOrder`、`cancelOrder`），描述写清触发条件。
- 为每个工具定义稳定的返回 DTO，避免返回原始数据库行。

# 2. 注解式工具（@Tool）

注解式工具是最常用的定义方式：在普通 Java 方法上加 `@Tool` 与 `@ToolParam`，框架自动生成 schema 并注册。

## 2.1 结论先行

- `@Tool(description=...)` 标注方法，`@ToolParam(description=...)` 标注参数。
- 方法参数与返回值会被框架反射解析为 JSON schema 与结果。
- 一个类可包含多个工具方法，实例化后整体传给 `tools(...)`。

## 2.2 机制拆解

### 2.2.1 注解字段

| 注解 | 位置 | 字段 | 说明 |
|------|------|------|------|
| `@Tool` | 方法 | `description` | 工具功能描述，必填 |
| `@Tool` | 方法 | `name` | 工具名，缺省用方法名 |
| `@ToolParam` | 参数 | `description` | 参数说明，必填 |
| `@ToolParam` | 参数 | `required` | 是否必填，默认 true |

### 2.2.2 参数与返回值

| 类型 | 支持 | 说明 |
|------|------|------|
| String/int/boolean | 是 | 基础类型 |
| 枚举 | 是 | 映射为枚举值 |
| POJO | 是 | 映射为对象 schema |
| List/Map | 是 | 容器类型 |
| void | 否 | 需有返回值 |
| 异常 | - | 应转结构化错误 |

## 2.3 代码示例

### 2.3.1 完整工具类

```java
public class OrderTools {

    @Tool(description = "按订单号查询订单详情")
    public OrderResult getOrder(
            @ToolParam(description = "订单号，如 A123") String orderId) {
        return orderService.get(orderId);
    }

    @Tool(description = "取消订单")
    public ToolResult cancelOrder(
            @ToolParam(description = "订单号") String orderId,
            @ToolParam(description = "取消原因") String reason) {
        return orderService.cancel(orderId, reason);
    }
}
```

### 2.3.2 返回 DTO

```java
public record OrderResult(
        String orderId,
        String status,
        BigDecimal amount) {
}
```

## 2.4 边界与坑

- 方法抛异常时，需确认框架是否转成模型可读错误；否则会中断整轮。
- 参数为复杂对象时，模型填写准确率下降，应拆成扁平参数。
- 描述若与指令重复或矛盾，模型行为不稳定。

## 2.5 生产建议

- 工具方法保持"薄"，只做参数适配与调用服务层。
- 在工具内部记录调用日志与耗时。
- 对返回值做裁剪，只保留模型决策需要的信息。

# 3. 工具注册与作用域

工具必须注册到 Agent 才可用，注册范围决定模型能看到哪些工具。

## 3.1 结论先行

- 通过 `LlmAgent.builder().tools(...)` 注册，可传对象、方法或内置工具。
- 工具集可按用户、场景动态裁剪，实现最小权限。
- 注册的工具越多，schema 占用上下文越大，越容易误调用。

## 3.2 机制拆解

### 3.2.1 注册方式

| 方式 | 示例 | 适用 |
|------|------|------|
| 实例 | `.tools(new OrderTools())` | 常用 |
| 多实例 | `.tools(a, b)` | 分组 |
| 函数工具 | `.tools(weatherTool)` | 动态 |
| 内置工具 | `.tools(GoogleSearchTool.create())` | 搜索/执行 |

### 3.2.2 作用域

| 作用域 | 说明 |
|--------|------|
| Agent 级 | 仅该 Agent 可见 |
| 子 Agent 级 | 各自持有独立工具集 |
| 运行时动态 | 每次请求按上下文裁剪 |

### 3.2.3 工具冲突与优先级

| 冲突 | 现象 | 处理 |
|------|------|------|
| 同名工具 | 模型不知选哪个 | 改名或去重 |
| 功能重叠 | 随机触发 | 合并或明确描述 |
| 描述矛盾 | 行为漂移 | 统一口径 |

- 相似工具（如 `searchDocs` 与 `searchWeb`）必须在描述里写清差异。

## 3.3 代码示例

```java
LlmAgent agent = LlmAgent.builder()
        .name("order-agent")
        .model("gemini-2.0-flash")
        .instruction("你是订单助手。")
        .tools(new OrderTools())
        .build();
```

```java
// 动态裁剪工具集
List<Object> tools = toolsFor(user);
LlmAgent agent = LlmAgent.builder()
        .name("order-agent")
        .model("gemini-2.0-flash")
        .tools(tools.toArray())
        .build();
```

## 3.4 边界与坑

- 同一工具重复注册可能产生歧义，需去重。
- 运行时裁剪需要每次重建 Agent，注意构建成本与缓存。
- 子 Agent 看不到父 Agent 的工具，需要显式共享。

## 3.5 生产建议

- 按业务域拆分工具类，按场景组合注册。
- 用配置或权限服务驱动工具集，而非硬编码。
- 监控每个工具的调用频次，发现异常触发。

# 4. 内置工具

ADK 提供若干开箱即用的内置工具，覆盖联网搜索、代码执行与企业检索。

## 4.1 结论先行

- 常见内置工具：Google Search、Code Executor、Vertex AI Search。
- 内置工具适合快速搭建，但可定制性弱、成本与合规需评估。
- 使用前确认模型与工具的区域可用性与配额。

## 4.2 机制拆解

| 工具 | 能力 | 典型场景 |
|------|------|----------|
| Google Search | 联网搜索 | 时效性问题 |
| Code Executor | 执行代码 | 计算、数据处理 |
| Vertex AI Search | 企业检索 | 私有知识库 |

### 4.2.1 内置工具与自建工具对比

| 维度 | 内置 | 自建 |
|------|------|------|
| 上线速度 | 快 | 慢 |
| 可控性 | 低 | 高 |
| 合规 | 依赖供应商 | 自主 |
| 成本 | 按调用计费 | 自建成本 |

### 4.2.2 内置工具配置

```yaml
google:
  adk:
    tools:
      google-search:
        enabled: true
        max-results: 5
      code-executor:
        enabled: true
        timeout-seconds: 10
      vertex-search:
        enabled: true
        datastore: projects/p/locations/l/collections/default_collection
```

## 4.3 代码示例

```java
LlmAgent agent = LlmAgent.builder()
        .name("search-agent")
        .model("gemini-2.0-flash")
        .instruction("需要最新信息时使用搜索工具。")
        .tools(GoogleSearchTool.create())
        .build();
```

```java
LlmAgent coder = LlmAgent.builder()
        .name("coder")
        .model("gemini-2.0-flash")
        .tools(CodeExecutorTool.create())
        .build();
```

## 4.4 边界与坑

- Code Executor 执行不可信代码有安全风险，须沙箱隔离。
- 联网搜索可能返回未经验证的信息，需在指令中要求标注来源。
- 企业检索需要先完成数据接入与权限配置。

## 4.5 生产建议

- 对内置工具做统一封装，便于替换与降级。
- 为搜索类工具设置超时与结果条数上限。
- 记录外部调用以满足合规审计。

# 5. 函数工具（FunctionTool）

函数工具允许用代码动态构造工具，适合参数或行为随运行时而定的场景。

## 5.1 结论先行

- `FunctionTool.create(name, description, schemaFn, execFn)` 用函数定义工具。
- 适合把已有 Java 方法快速包装成工具，或按配置动态生成。
- 参数 schema 需显式给出，否则模型无法正确填参。

## 5.2 机制拆解

### 5.2.1 四个组成部分

| 部分 | 作用 |
|------|------|
| name | 工具名 |
| description | 功能描述 |
| schemaFn | 参数校验/归一化 |
| execFn | 实际执行逻辑 |

### 5.2.2 执行流程

```
模型填参 -> schemaFn 校验/归一化 -> execFn 执行 -> 结果回填
```

### 5.2.3 与注解式工具的取舍

| 维度 | 注解式 | FunctionTool |
|------|--------|--------------|
| 定义成本 | 低 | 中 |
| 动态性 | 弱 | 强 |
| 类型安全 | 强 | 弱 |
| 适用 | 固定工具 | 运行时生成 |

- 固定能力优先用注解式；需要按配置/租户动态生成的用 FunctionTool。

## 5.3 代码示例

```java
FunctionTool weatherTool = FunctionTool.create(
        "getWeather",
        "查询城市天气",
        params -> Map.of("city", params.get("city")),
        params -> weatherService.get((String) params.get("city")));

LlmAgent agent = LlmAgent.builder()
        .name("weather-agent")
        .model("gemini-2.0-flash")
        .tools(weatherTool)
        .build();
```

```java
// 按配置动态生成工具
FunctionTool tool = FunctionTool.create(
        "lookup",
        "按关键字检索 " + dataset,
        p -> Map.of("keyword", p.get("keyword")),
        p -> searchClient.search(dataset, (String) p.get("keyword")));
```

## 5.4 边界与坑

- schemaFn 里做重活会拖慢每次调用，应只做轻量校验。
- execFn 抛异常需转结构化错误。
- 动态生成的工具数量不受控会导致上下文膨胀。

## 5.5 生产建议

- 把 schemaFn 用作参数白名单校验的落点。
- 对动态工具设数量上限。
- 为每个函数工具补充单元测试，覆盖参数边界。

# 6. 工具权限与安全

工具是权限风险的集中点：模型可能被诱导调用越权工具，因此必须在工具侧二次校验。

## 6.1 结论先行

- 权限控制分两层：注册时裁剪工具集 + 执行时二次鉴权。
- 不信任模型传来的任何参数，尤其是 ID、金额、用户标识。
- 高风险操作（退款、删除）应加人工确认或双人复核。

## 6.2 机制拆解

### 6.2.1 权限模型

| 层级 | 手段 | 时机 |
|------|------|------|
| 工具可见性 | 按用户裁剪工具集 | 注册时 |
| 参数授权 | 校验资源归属 | 调用时 |
| 操作审批 | 人工确认 | 高风险时 |
| 审计 | 记录调用 | 全程 |

### 6.2.2 最小权限原则

```
用户 -> 角色 -> 允许的工具集合 -> 注册给 Agent -> 模型只能调用这些工具
```

### 6.2.3 审批流

```
高风险工具被调用
   |
   v
生成待审批单(含参数与上下文)
   |
   v
人工确认 --通过--> 执行 --拒绝--> 返回"已拒绝"给模型
```

- 审批期间会话可挂起，恢复后继续执行，避免阻塞线程。

## 6.3 代码示例

### 6.3.1 按用户裁剪工具

```java
public List<Object> toolsFor(UserContext user) {
    Set<String> allowed = permissionService.allowedTools(user);

    return allTools.stream()
            .filter(t -> allowed.contains(nameOf(t)))
            .toList();
}

LlmAgent agent = LlmAgent.builder()
        .name("guarded-agent")
        .model("gemini-2.0-flash")
        .tools(toolsFor(user).toArray())
        .build();
```

### 6.3.2 工具内二次鉴权

```java
@Tool(description = "取消订单")
public ToolResult cancelOrder(
        @ToolParam(description = "订单号") String orderId,
        @ToolParam(description = "取消原因") String reason) {
    Order order = orderService.get(orderId);
    if (!order.belongsTo(currentUser())) {
        return ToolResult.error("无权操作该订单");
    }
    return orderService.cancel(orderId, reason);
}
```

## 6.4 边界与坑

- 只靠"工具不可见"不等于安全：模型可能通过其他工具间接越权。
- 依赖模型传入的 userId 做鉴权等于没有鉴权，应从会话上下文取。
- 错误信息泄露内部细节，需脱敏。

## 6.5 生产建议

- 鉴权统一走 `currentUser()` 之类的上下文，而非模型参数。
- 高风险工具加幂等键与审批流。
- 所有工具调用写入审计日志，含参数与结果摘要。

# 7. 生产注意事项

工具是 Agent 生产化的主要风险面与成本来源，需要系统化治理。

## 7.1 结论先行

- 工具治理围绕四点：鉴权、裁剪、超时、审计。
- 工具结果是上下文的主要"膨胀源"，必须精简。
- 工具错误要可被模型理解，支持重试或换路。

## 7.2 风险与对策

| 风险 | 影响 | 建议 |
|------|------|------|
| 工具无鉴权 | 越权操作 | 工具内校验 |
| 工具过多 | 误调用、费 token | 按场景裁剪 |
| 结果过大 | 上下文爆炸 | 精简返回 |
| 无超时 | 线程阻塞 | 工具内设超时 |
| 无审计 | 无法追责 | 记录调用 |

## 7.3 超时、重试与幂等

```java
@Tool(description = "查询物流")
public ShipmentResult track(
        @ToolParam(description = "运单号") String no) {
    return withTimeout(() -> logistics.query(no), Duration.ofSeconds(3));
}
```

| 手段 | 说明 |
|------|------|
| 超时 | 单次调用上限 |
| 重试 | 仅幂等读操作 |
| 幂等键 | 写操作防重复 |
| 熔断 | 依赖故障时快速失败 |

## 7.4 审计

| 记录项 | 说明 |
|--------|------|
| 工具名 | 调用了哪个 |
| 参数 | 入参摘要（脱敏） |
| 结果 | 状态与耗时 |
| 调用者 | 用户/会话 |
| 链路 | 所属 Trace |

## 7.5 工具描述质量

- 描述应包含"何时用、做什么、返回什么"。
- 多个相似工具要在描述中说明区分点。
- 描述与指令、参数 schema 保持一致。

## 7.6 工具评测

| 指标 | 说明 |
|------|------|
| 调用准确率 | 该调用时是否调用 |
| 参数正确率 | 参数是否填对 |
| 误调用率 | 不该调用却调用 |
| 失败率 | 执行异常占比 |
| 平均耗时 | 单次调用延迟 |

- 用固定用例集回归，工具描述或模型变更后必须重跑。

# 8. 小结

## 8.1 核心要点回顾

- 用 `@Tool` 注解或 `FunctionTool` 定义工具，支持内置工具（搜索、代码执行、企业检索）。
- 工具通过 `tools(...)` 注册，可按用户权限动态裁剪。
- 工具在"模型—执行—回填"循环中触发，schema 校验与结果精简同样重要。
- 工具内必须鉴权、超时、精简返回，且需审计。
- 高风险操作需审批与幂等保护。
- 工具描述质量与指令同等重要，需版本化管理。
- 工具集膨胀会同时抬高成本与误调用率，应持续裁剪。
- 工具与模型一样需要版本化与灰度发布。
- 工具契约（名称、参数、返回）变更属于破坏性变更。

## 8.2 生产检查清单

- [ ] 每个工具有清晰描述与稳定返回 DTO。
- [ ] 工具集按用户/场景最小化。
- [ ] 工具内有二次鉴权与超时。
- [ ] 写操作具备幂等键。
- [ ] 工具调用有审计与指标。
- [ ] 内置工具已评估合规与成本。
- [ ] 工具契约变更已评估兼容性。
