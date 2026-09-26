# 1. 配置治理的价值

## 1.1 结论先行

LLM 应用的配置变化频繁，**如果每次都要改代码 + 发版，迭代速度会被拖死**。配置治理的目标是让模型、Prompt、阈值、开关等在不发版的前提下可调、可灰度、可回滚。

## 1.2 为什么变化频繁

```
模型切换、Prompt 调整、阈值调优、灰度比例、限流阈值...
```

| 变化类型 | 频率 | 影响面 |
|----------|------|--------|
| 模型切换 | 低 | 效果、成本 |
| Prompt 调整 | 高 | 效果 |
| 阈值调优 | 中 | 召回/精度 |
| 灰度比例 | 高 | 可用性 |
| 限流阈值 | 中 | 可用性 |

## 1.3 边界与反例

- 把所有东西都塞进配置中心，会让配置变得难以维护，反而增加故障面。
- 密钥类配置若进入配置仓库，等同于泄漏。

## 1.4 生产建议

- 只把"需要动态调整"的内容配置化，代码逻辑仍以代码为准。
- 为配置变更建立与代码同等的评审流程。

# 2. 需要配置化的内容

## 2.1 结论先行

需要配置化的内容可分为八类，覆盖模型、Prompt、RAG、可靠性、限流、成本、特性、密钥。

## 2.2 配置分类清单

| 类别 | 示例 |
|------|------|
| 模型 | 模型名、temperature、maxTokens |
| Prompt | 模板版本 |
| RAG | topK、阈值、分块参数 |
| 可靠性 | 超时、重试次数、熔断阈值 |
| 限流 | 各维度阈值 |
| 成本 | 预算、配额 |
| 特性 | 开关、灰度比例 |
| 密钥 | API Key（从密钥管理） |

## 2.3 边界与坑

- 配置项粒度太细会导致组合爆炸，应保持合理的抽象层次。
- 未设默认值的配置项在缺失时会导致启动失败，应提供兜底。

## 2.4 生产建议

- 为每类配置定义明确的 owner 与变更流程。
- 敏感配置与非敏感配置分开存储。

# 3. 配置分层

## 3.1 结论先行

配置应按优先级分层：**环境变量 > 配置中心 > 配置文件 > 代码默认**，同时敏感信息走密钥管理、租户级配置走数据库。

## 3.2 分层模型

```
① 代码默认值（兜底）
② 配置文件（application.yml）
③ 配置中心（Nacos/Apollo/Consul）  ← 可热更新
④ 环境变量/密钥管理（敏感信息）
⑤ 数据库（租户级配置）
```

| 层级 | 存储 | 特点 |
|------|------|------|
| 代码默认 | 源码 | 兜底，最稳定 |
| 配置文件 | 仓库 | 随版本发布 |
| 配置中心 | Nacos 等 | 可热更新 |
| 环境变量/密钥 | 运行时/Vault | 敏感、环境相关 |
| 数据库 | DB | 租户级、动态 |

## 3.3 优先级解析

### 3.3.1 覆盖规则

高优先级覆盖低优先级。Spring 的 `Environment` 会按属性源顺序解析，环境变量通常优先级最高。

### 3.3.2 示例

```yaml
app:
  ai:
    model:
      name: gpt-4o-mini   # 可被配置中心或环境变量覆盖
```

## 3.4 边界与坑

- 环境变量命名需遵循规范（大写、下划线），否则无法正确映射。
- 多层覆盖可能导致"改了没生效"，需提供配置来源追踪能力。

## 3.5 生产建议

- 提供接口查询某配置的最终生效值与来源。
- 启动时打印关键配置（脱敏）便于排查。

# 4. 类型安全配置

## 4.1 结论先行

使用 `@ConfigurationProperties` + `@Validated` 实现类型安全与启动期校验，把配置错误挡在启动阶段，而不是运行时。

## 4.2 代码示例

```java
@ConfigurationProperties(prefix = "app.ai")
@Validated
public record AiProperties(
        @NotNull ModelConfig model,
        @NotNull RagConfig rag,
        @NotNull ReliabilityConfig reliability
) {
    public record ModelConfig(
            @NotBlank String name,
            @DecimalMin("0.0") @DecimalMax("2.0") double temperature,
            @Min(1) int maxTokens,
            @NotNull Duration timeout
    ) {}

    public record RagConfig(
            @Min(1) int topK,
            @DecimalMin("0.0") @DecimalMax("1.0") double minScore,
            @Min(1) int rerankTopN
    ) {}

    public record ReliabilityConfig(
            @Min(1) int maxRetries,
            @NotNull Duration toolTimeout,
            @DecimalMin("0.0") @DecimalMax("1.0") double failureRateThreshold
    ) {}
}
```

## 4.3 配置示例

```yaml
app:
  ai:
    model:
      name: gpt-4o-mini
      temperature: 0.3
      max-tokens: 2048
      timeout: 30s
    rag:
      top-k: 20
      min-score: 0.7
      rerank-top-n: 3
    reliability:
      max-retries: 2
      tool-timeout: 5s
      failure-rate-threshold: 0.5
```

## 4.4 边界与坑

- `record` 与 `@ConfigurationProperties` 需配合构造函数绑定（`@ConstructorBinding` 或 Spring Boot 3 自动识别）。
- 校验失败默认导致启动失败，热更新场景需捕获异常并保留旧值。

## 4.5 生产建议

- 所有必填项加 `@NotNull`，所有数值加范围校验。
- 为配置类编写绑定测试，防止重构破坏绑定。

# 5. 特性开关

## 5.1 结论先行

特性开关（Feature Flag）支持灰度放量、紧急降级、A/B 测试与租户定制，是配置治理中价值最高的能力之一。

## 5.2 实现机制

### 5.2.1 灰度算法

按 `userId + feature` 做稳定哈希，取模 100 与灰度比例比较，保证同一用户结果稳定。

### 5.2.2 代码示例

```java
@Component
public class FeatureFlags {

    private final Map<String, Boolean> flags = new ConcurrentHashMap<>();

    public boolean isEnabled(String feature, UserContext user) {
        FeatureConfig config = configStore.get(feature);
        if (config == null || !config.enabled()) return false;

        // 灰度：按用户哈希
        if (config.rolloutPercentage() < 100) {
            int hash = Math.abs((user.userId() + feature).hashCode());
            return (hash % 100) < config.rolloutPercentage();
        }
        return true;
    }
}
```

```java
public String answer(String question, UserContext user) {
    if (featureFlags.isEnabled("agentic-rag", user)) {
        return agenticRag(question);          // 新特性
    }
    return traditionalRag(question);           // 旧逻辑
}
```

## 5.3 开关用途

| 开关用途 | 说明 |
|----------|------|
| 新功能灰度 | 按比例放量 |
| 紧急降级 | 一键关闭有问题的功能 |
| A/B 测试 | 对比效果 |
| 租户定制 | 按租户开启 |

## 5.4 边界与坑

- `String.hashCode` 分布不均，海量用户下灰度比例可能偏差，建议用一致性哈希或分桶。
- 开关数量膨胀会形成"配置地狱"，需定期清理已全量的开关。
- 开关判断应集中，避免散落各处难以审计。

## 5.5 生产建议

- 开关命名带业务前缀，明确含义与负责人。
- 紧急降级开关要能在秒级生效，并经过演练。

# 6. 动态配置热更新

## 6.1 结论先行

热更新让配置改动无需重启即可生效，但需要处理"重建昂贵资源"（如模型客户端）与"旧值兜底"两个问题。

## 6.2 实现方式

### 6.2.1 Spring Cloud RefreshScope

```java
@Component
@RefreshScope                                    // Spring Cloud
public class DynamicModelConfig {

    @Value("${app.ai.model.name}")
    private String modelName;

    @Bean
    @RefreshScope
    public ChatModel chatModel(AiProperties props) {
        return OpenAiChatModel.builder()
                .apiKey(apiKeyProvider.get())
                .modelName(props.model().name())
                .temperature(props.model().temperature())
                .build();
    }
}
```

### 6.2.2 Nacos 配置监听

```java
@NacosConfigListener(dataId = "ai-config.yaml")
public void onConfigChanged(String config) {
    AiProperties props = parse(config);
    modelRegistry.refresh(props);                // 重建模型实例
    log.info("配置已热更新");
}
```

## 6.3 热更新流程

```
配置中心变更
      │
      ▼
监听器收到新配置
      │
      ▼
校验（类型 + 范围）
      │
      ├─ 失败 → 记录告警，保留旧值
      └─ 成功 → 原子替换引用
                  │
                  ▼
            重建模型/Prompt 实例
```

## 6.4 边界与坑

- 重建模型实例有成本，频繁变更会抖动，应加防抖或限频。
- 热更新期间并发请求可能读到新旧混合状态，应保证替换是原子引用切换。

## 6.5 生产建议

- 变更先校验后应用，失败不影响线上。
- 记录每次热更新前后的值，便于回滚。

# 7. Prompt 版本管理

## 7.1 结论先行

Prompt 应与代码分离并版本化，从而可独立灰度与回滚。

## 7.2 配置示例

```yaml
app:
  ai:
    prompt:
      version: v2                              # 切换版本
      rollout:
        v2: 30                                 # v2 灰度 30%
```

## 7.3 代码示例

```java
public String resolvePrompt(String name, UserContext user) {
    String version = promptVersionResolver.resolve(name, user);
    return promptRegistry.get(name, version);
}
```

## 7.4 边界与坑

- 版本回滚要保证旧版本模板仍可获取，不能删旧版本。
- Prompt 变更往往需要配套评测集验证效果。

## 7.5 生产建议

- 每个 Prompt 版本关联一次评测记录。
- 灰度期间对比新旧版本的关键指标（满意度、错误率）。

# 8. 配置安全

## 8.1 结论先行

敏感配置必须走密钥管理（Vault/KMS），绝不能以明文进入配置仓库。

## 8.2 处理策略

| 配置 | 处理 |
|------|------|
| API Key | 密钥管理（Vault/KMS） |
| 数据库密码 | 密钥管理 |
| 业务阈值 | 配置中心 |
| 租户配置 | 数据库（加密敏感字段） |

## 8.3 正反例

```java
// 禁止：密钥写死在配置文件中
// app.ai.openai.api-key: sk-xxxxx    ← 错误

// 正确：引用环境变量/密钥管理
// app.ai.openai.api-key: ${OPENAI_API_KEY}
```

## 8.4 边界与坑

- 环境变量在进程信息中可能可见，高敏场景应使用密钥管理动态拉取。
- 日志、异常堆栈、监控上报都可能泄漏密钥，需统一脱敏。

## 8.5 生产建议

- 引入密钥扫描工具，阻止密钥提交到仓库。
- 密钥定期轮换，并验证轮换后服务可用。

# 9. 配置变更的审计

## 9.1 结论先行

配置变更也是变更，必须可追溯：记录谁、何时、把哪个键从什么值改成了什么值。

## 9.2 代码示例

```java
@EventListener
public void onConfigChanged(ConfigChangedEvent event) {
    auditLog.record(new AuditEntry(
            event.user(),
            event.key(),
            event.oldValue(),          // 脱敏后
            event.newValue(),
            Instant.now()));
}
```

## 9.3 审计字段

| 字段 | 说明 |
|------|------|
| 操作人 | 变更发起者 |
| 配置键 | 变更的配置项 |
| 旧值/新值 | 脱敏后的内容 |
| 时间 | 变更时间 |
| 来源 | 手动/自动化 |

## 9.4 边界与坑

- 审计日志本身可能包含敏感值，需脱敏后再存储。
- 审计日志不可被业务代码修改，应只追加。

## 9.5 生产建议

- 审计日志接入告警，异常时段的高频变更触发人工确认。
- 支持按配置键查询完整变更历史。

# 10. 配置变更的影响评估

## 10.1 结论先行

不同配置变更的影响面差异巨大，应先评估再执行：换模型影响最大，需评测 + 灰度。

## 10.2 影响评估表

| 变更 | 影响 | 建议 |
|------|------|------|
| 换模型 | 效果/成本变化大 | 先评测 + 灰度 |
| 调 Prompt | 效果变化 | 灰度 + 对比指标 |
| 调阈值 | 召回/精度变化 | 评测 |
| 调限流 | 可用性 | 观察负载 |
| 关开关 | 功能不可用 | 确认影响范围 |

## 10.3 灰度与回滚

```
变更提交 → 灰度 1% → 观察指标 → 放量 10% → 观察 → 全量
                     │
                     └─ 指标异常 → 立即回滚
```

## 10.4 边界与坑

- 部分变更（如切换向量模型）不可逆，历史向量可能需重建。
- 回滚不总是即时生效，注意缓存与已建立连接。

## 10.5 生产建议

- 所有变更先在小流量灰度，指标正常再放量。
- 保留历史配置版本，确保能一键回滚。

# 11. 生产注意事项

## 11.1 风险清单

| 风险 | 建议 |
|------|------|
| 密钥进配置仓库 | 密钥管理 |
| 无校验 | `@Validated` |
| 变更无审计 | 记录变更 |
| 无灰度 | 特性开关 |
| 配置不一致 | 配置中心统一 |
| 无回滚 | 保留历史配置 |
| 开关膨胀 | 定期清理 |

## 11.2 监控指标

| 指标 | 含义 |
|------|------|
| 配置变更次数 | 变更活跃度 |
| 热更新失败率 | 配置质量 |
| 开关生效延迟 | 降级能力 |
| 配置校验失败 | 误配置风险 |

## 11.3 上线检查清单

- 敏感项是否全部走密钥管理。
- 是否有启动期校验。
- 是否可热更新、可回滚。
- 是否有变更审计。
- 是否有灰度能力。

# 12. 小结

- 配置治理让迭代无需发版。
- 分层：默认值 → 配置文件 → 配置中心 → 环境变量/密钥 → 租户配置。
- 类型安全 + 校验，避免配置错误。
- 特性开关支持灰度与紧急降级。
- 配置变更必须审计与可回滚。
- 敏感配置一律走密钥管理。
