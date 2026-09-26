# 1. 架构与分层

好的架构让模型可替换、业务可测试、故障可隔离。以下实践适用于绝大多数 Spring AI 应用。

## 1.1 解耦与面向接口

| 实践 | 反例 | 建议 |
|------|------|------|
| 业务与框架解耦 | Controller 直接注入 `ChatClient` | 通过 Service/Facade 封装 |
| 面向接口 | 到处 `new OpenAiChatModel()` | 依赖 `ChatModel`/`ChatClient` 抽象 |
| 单一职责 | 一个类既做 RAG 又做工具又做记忆 | 按能力拆分为独立组件 |
| 配置外置 | 模型名/密钥写死在代码 | 配置中心 + 环境变量 |

## 1.2 单一职责与配置外置

```java
public interface Assistant {
    String ask(String question);
}

@Service
public class SpringAiAssistant implements Assistant {

    private final ChatClient chatClient;

    public SpringAiAssistant(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    @Override
    public String ask(String question) {
        return chatClient.prompt().user(question).call().content();
    }
}
```

配置外置示例：

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: ${AI_MODEL:gpt-4o-mini}
```

## 1.3 反例速查

- 反例：把 `ChatClient` 暴露给 Controller，接口与框架细节耦合；
- 反例：一个 Service 同时管理记忆、检索、工具与业务规则；
- 边界：分层要适度，小项目不必强套六层。

# 2. 提示工程

提示（Prompt）是 AI 应用的核心资产，应当像代码一样被管理。

## 2.1 模板管理

- **模板外置**到资源文件，命名 + 版本化，支持灰度回滚。
- 禁止在代码中拼接大段提示字符串。
- 模板变更需走评测回归，而非直接上线。

```
src/main/resources/prompts/
├── system/assistant-v1.txt
├── system/assistant-v2.txt
└── rag/qa-with-citation-v1.txt
```

## 2.2 结构与顺序

### 2.2.1 稳定内容在前

**稳定内容在前、易变内容在后**，提高 Prompt 缓存（prompt caching）命中率：系统指令、规则、少样本示例放前面，用户问题与检索片段放后面。

### 2.2.2 角色与边界

- **用户输入永远放在 user 角色**，绝不拼进 system；
- system 中的规则要声明"不可被用户覆盖"；
- 工具返回的内容也视为不可信输入，与用户输入同等对待。

## 2.3 上下文预算

- **上下文要有预算意识**，检索片段不是越多越好；
- 为输入设置 Token 上限，超限时裁剪或摘要；
- 边界：裁掉的内容可能包含关键信息，需评测验证。

# 3. 结构化输出

结构化输出把自由文本变成可编程对象，是可靠性的基础。

## 3.1 结构设计

### 3.1.1 扁平稳定

- 结构要**扁平、稳定、可校验**，避免深层嵌套；
- 字段名一旦被消费即为契约，变更需版本化；
- 可选字段与必填字段要区分清楚。

### 3.1.2 枚举与校验

- 关键字段用**枚举**而非自由文本；
- 加 **Bean Validation**，解析后立即校验。

```java
public record Ticket(
        @NotBlank String title,
        @NotNull Priority priority,
        @Positive int estimateHours) {}

Ticket ticket = chatClient.prompt()
        .user("把这条需求整理成工单：" + raw)
        .call()
        .entity(Ticket.class);
```

## 3.2 解析失败兜底

- **不要用提示约束格式**，用结构化输出（`entity()`）；
- 永远准备**解析失败兜底**（重试 / 默认值 / 降级）。

```java
Ticket safeParse(String raw) {
    try {
        return chatClient.prompt().user(raw).call().entity(Ticket.class);
    } catch (Exception e) {
        return retryOnceOrFallback(raw, e);
    }
}
```

## 3.3 边界

- 复杂嵌套结构解析失败率显著上升；
- 模型可能返回合法 JSON 但语义错误，需业务校验；
- 边界：结构化输出会略微增加 Token 与延迟。

# 4. 工具调用

工具让模型从"会说"走向"会做"，但风险也随之上线。

## 4.1 描述与返回

### 4.1.1 描述质量

| 原则 | 说明 |
|------|------|
| 描述清晰 | 说清"何时用、参数含义、返回什么" |
| 结果结构化 | 成功/失败都返回结构化数据 |
| 不抛异常 | 让模型看到错误并自我纠错 |

### 4.1.2 描述反例

- 反例："查询数据"这类模糊描述会让模型误用；
- 反例：把多个动作塞进一个工具，参数含义不清。

## 4.2 幂等与鉴权

| 原则 | 说明 |
|------|------|
| 写操作幂等 | 幂等键 + 去重 |
| 必做鉴权 | 工具级权限校验 |

```java
@Tool(description = "创建工单，返回是否成功")
CreateResult createTicket(
        @ToolParam(description = "幂等键") String idempotencyKey,
        @ToolParam(description = "问题描述") String description) {
    SecurityUtils.requireRole("AGENT");
    return ticketService.createIdempotent(idempotencyKey, description);
}
```

## 4.3 数量与超时

- **必设超时**：防止单个工具拖垮全局；
- **控制数量**：单次挂载工具不宜过多（建议不超过 10~15 个）；
- 工具过多会提示膨胀、选择困难、误用率上升；
- 边界：工具是攻击面，新增工具必须评审权限。

# 5. 记忆

记忆决定多轮体验，也最容易引入越权与成本问题。

## 5.1 持久化

- 生产**禁用 `InMemoryChatMemory`**（重启丢失、无法扩展）；
- 使用 Redis / JDBC 等外部存储。

## 5.2 隔离与增长控制

- `conversationId` 必须校验归属，防水平越权；
- 设 `maxMessages` + TTL，防无限增长；
- 长对话配合摘要 / 向量记忆，而非无限堆历史。

## 5.3 隐私

- 敏感信息落库前脱敏/加密；
- 记忆数据受租户隔离与访问审计约束；
- 边界：摘要会丢失细节，关键信息应显式保留。

# 6. RAG

RAG（Retrieval-Augmented Generation，检索增强生成）的质量取决于检索与生成两端。

## 6.1 分块与元数据

- 分块策略**没有万能参数**，必须评测驱动；
- 元数据**建库时就设计好**，事后补代价极高；
- Embedding 模型**一旦选定不要轻易更换**，否则全量重建。

## 6.2 检索安全

- 检索**必须带租户/权限过滤**；
- 过滤条件由后端强制注入，不能信任前端参数；
- 边界：过滤会降低召回，需在安全与效果间权衡。

## 6.3 生成与诊断

- 回答要求**引用来源**，无法引用则拒答；
- 先诊断"检索失败"还是"生成失败"再优化；
- 边界：拒答率过高会伤害体验，需平衡。

# 7. Advisor

Advisor 是 Spring AI 的切面机制，治理逻辑应集中在此。

## 7.1 顺序与测试

- 顺序（`order`）**显式声明并写测试**；
- 顺序影响观测、脱敏、限流的执行次序；
- 建议顺序：Trace → 日志/脱敏 → 限流 → 业务。

## 7.2 容错与脱敏

- Advisor 内部 **try-catch**，避免拖垮主流程；
- 日志/Trace **必须脱敏**。

```java
public class SafeLoggingAdvisor implements CallAdvisor {

    @Override
    public ChatClientResponse adviseCall(ChatClientRequest req,
                                         CallAdvisorChain chain) {
        try {
            ChatClientResponse resp = chain.nextCall(req);
            log.info("llm done, tokens={}", resp.chatResponse()
                    .getMetadata().getUsage().getTotalTokens());
            return resp;
        } catch (Exception e) {
            log.warn("llm failed", e);
            throw e;
        }
    }

    @Override
    public String getName() { return "SafeLoggingAdvisor"; }

    @Override
    public int getOrder() { return 0; }
}
```

## 7.3 流式

- 流式场景要同时实现 `StreamAdvisor`；
- 流式下超时与取消语义更复杂，需专门测试。

# 8. 可靠性与性能

## 8.1 五件套

```
超时 → 重试 → 幂等 → 限流 → 熔断 → 降级
```

| 手段 | 关键点 |
|------|--------|
| 超时 | 模型、工具、检索都要设 |
| 重试 | 只重试幂等操作，指数退避 |
| 幂等 | 写操作必须有幂等键 |
| 限流 | 保护配额与成本，按租户维度 |
| 熔断 | 依赖不可用时快速失败 |
| 降级 | 备用模型 / 缓存 / 静态兜底 |

## 8.2 实现要点

### 8.2.1 组合 Advisor

```java
@Bean
public ChatClient chatClient(ChatModel model,
                             CircuitBreaker cb, RateLimiter rl) {
    return ChatClient.builder(model)
            .defaultAdvisors(new ResilienceAdvisor(cb, rl))
            .build();
}
```

### 8.2.2 参数取值

- 超时值应小于上游网关超时，避免级联失败；
- 熔断阈值与限流速率要按系统容量设定，而非拍脑袋。

## 8.3 边界

- 重试放大下游压力，必须配合退避与熔断；
- 降级结果要标注状态，避免误导用户；
- 边界：超时值需小于上游超时，否则级联失败。

# 9. 可观测与成本

## 9.1 指标与 Trace

### 9.1.1 关键指标

- **Token 与成本是一等公民**，必须监控；
- Trace 要能"回放"完整链路（含工具与检索）。

```
Trace: 用户提问
├── Span: 记忆读取
├── Span: 模型调用
├── Span: 工具调用 getOrder("A123")
└── Span: 记忆写入
```

## 9.2 成本治理

- 按租户/用户维度统计成本，设置预算告警；
- 模型分层、缓存、Token 预算三管齐下。

## 9.3 高基数陷阱

- 指标避免高基数标签（别把 userId 当 label）；
- 生产采样上报，错误全采样；
- 边界：采样会漏掉低频问题，错误必须全量。

# 10. 安全

## 10.1 分层防护

| 层 | 必做 |
|----|------|
| 输入 | 注入检测、长度限制 |
| 工具 | 最小权限、工具级鉴权、MCP 隔离 |
| 数据 | 租户隔离、加密、脱敏 |
| 输出 | 内容审核、引用校验 |

## 10.2 数据与审计

| 层 | 必做 |
|----|------|
| 审计 | 记录谁在何时调用了什么 |

- 审计日志独立存储、保留合规期限；
- 敏感数据出境需评估合规。

## 10.3 运维安全

| 层 | 必做 |
|----|------|
| 运维 | 密钥轮换、依赖审计、出网管控 |

- 依赖审计：定期扫描 AI SDK 与传递依赖漏洞；
- 出网管控：限制服务可访问的外部地址。

# 11. 测试与评测

## 11.1 分层测试

- 单元测试 **Mock 模型**，不真调；
- 集成测试用**小样例 + 便宜模型**。

## 11.2 接入 CI

- 评测集与 Prompt **一起版本化**；
- 评测接入 **CI**，不达标不放行。

## 11.3 防过拟合

- 保留一份**未参与调优**的测试集防过拟合；
- 评测不稳定时多次采样取均值；
- 边界：评测集会被"针对优化"，需定期轮换。

# 12. 版本与升级

## 12.1 版本锁定

| 事项 | 建议 |
|------|------|
| 版本锁定 | BOM 统一管理，锁死版本 |

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.ai</groupId>
      <artifactId>spring-ai-bom</artifactId>
      <version>${spring-ai.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

## 12.2 升级策略

| 事项 | 建议 |
|------|------|
| 升级评估 | 关注 upgrade-notes 与破坏性变更 |
| 模型升级 | 先评测再切换，保留回滚 |
| Prompt 变更 | 灰度发布，监控指标 |
| Embedding 升级 | 需整体重建向量库 |

## 12.3 回滚

- 保留镜像、Prompt、配置三层回滚能力；
- 模型版本也要记录，否则无法解释行为变化；
- 边界：供应商下线旧模型时，回滚可能不可用。

# 13. 常见坑速查

## 13.1 启动与配置类

| 坑 | 现象 | 解法 |
|----|------|------|
| 用错 starter 坐标 | 启动报缺 Bean | 用 `spring-ai-starter-model-*` |
| 未设超时 | 请求长时间挂起 | 配置 HTTP 超时 |
| 密钥进 Git | 安全事故 | 环境变量 + 密钥管理 |

## 13.2 运行时类

| 坑 | 现象 | 解法 |
|----|------|------|
| 重试非幂等写操作 | 重复下单/重复扣款 | 幂等键 |
| 记忆无限增长 | 成本飙升、上下文超限 | 窗口 + TTL + 摘要 |
| RAG 检索越权 | 看到他人数据 | 元数据强制过滤 |
| 结构化输出解析失败 | 偶发异常 | 重试 + 兜底 + 原生 Schema |
| 工具调用死循环 | 请求不返回 | 最大轮数 + 终止条件 |
| 无评测就上线 | 质量不稳定 | 建评测集 + CI |

## 13.3 安全与合规类

| 坑 | 现象 | 解法 |
|----|------|------|
| 日志泄露隐私 | 合规风险 | 脱敏 + 分级 |
| 工具权限过大 | 越权操作 | 最小权限 + 工具级鉴权 |

# 14. 上线检查清单

## 14.1 安全与配置

- [ ] 密钥外置，未提交到仓库
- [ ] 工具权限已收敛到最小集
- [ ] RAG 已做租户/权限过滤
- [ ] 记忆会话隔离已校验

## 14.2 可靠性与数据

- [ ] 模型/工具/检索均设置超时
- [ ] 写操作幂等
- [ ] 限流与熔断已配置
- [ ] 记忆已持久化
- [ ] 结构化输出有兜底
- [ ] 有降级与回滚方案

## 14.3 可观测与交付

- [ ] Token/成本指标已接入监控
- [ ] Trace 可回放，日志已脱敏
- [ ] 评测集已建立并接入 CI
- [ ] Prompt 与代码一起版本化

# 15. 小结

## 15.1 核心要点

- 好的 Spring AI 应用 = **清晰分层 + 可靠调用 + 安全隔离 + 可观测 + 可评测**。
- 框架用法简单，生产治理才是难点。

## 15.2 落地建议

- 把这份清单当作每个 AI 功能上线前的自检表。
- 先建立评测基线与可靠性五件套，再谈规模化。
- 模板、模型、评测集三者一起版本化，变更可灰度可回滚。
