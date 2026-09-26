# 1. 多租户的隔离维度

## 1.1 结论先行

多租户（Multi-Tenancy）的核心是**隔离**。少隔离任何一项，都可能造成数据泄露或资源争抢，因此必须同时隔离六样东西。

## 1.2 六维隔离模型

```
多租户必须隔离的六样东西：
① 数据（向量、记忆、会话）
② 模型与配额
③ 密钥与凭证
④ 权限与工具
⑤ 成本与计费
⑥ 审计日志
```

| 维度 | 隔离对象 | 泄露后果 |
|------|----------|----------|
| 数据 | 向量、记忆、会话 | 跨租户读到他人数据 |
| 模型与配额 | 可用模型、调用量 | 大租户挤占资源 |
| 密钥 | API Key、凭证 | 费用转嫁、密钥泄漏 |
| 权限与工具 | 可调用工具 | 越权操作 |
| 成本与计费 | 用量与账单 | 无法计费、预算失控 |
| 审计日志 | 操作记录 | 无法追责 |

## 1.3 边界与反例

- 只做数据隔离而忽略密钥隔离，会导致 A 租户的调用消耗 B 租户的额度。
- 只做认证不做授权，登录成功即可访问任意租户资源。

## 1.4 生产建议

- 把六个维度做成上线检查清单，逐项验证。
- 每个维度都要有对应的自动化越权测试。

# 2. 数据隔离方案

## 2.1 结论先行

数据隔离有"共享存储 + 元数据过滤""共享实例 + 独立 schema""独立实例"三档，应按租户等级与合规要求分层选型，多数 SaaS 从元数据过滤起步。

## 2.2 三种方案对比

| 方案 | 隔离强度 | 成本 | 适用 |
|------|:--------:|:----:|------|
| 共享存储 + 元数据过滤 | 中 | 低 | 多数 SaaS |
| 共享实例 + 独立 schema | 高 | 中 | 中型租户 |
| 独立实例 | 最高 | 高 | 大客户、强合规 |

### 2.2.1 隔离强度光谱

```
弱 ─────────────────────────────────────────▶ 强
共享存储           独立 schema            独立实例
(元数据过滤)       (同库不同表/集合)      (独立库/集群)
成本低                                    成本高
运维简单                                  运维复杂
租户多时易扩展                            租户多时难扩展
```

### 2.2.2 选型判断

| 判断条件 | 建议 |
|----------|------|
| 租户数多、单租户数据小 | 元数据过滤 |
| 需要更强隔离但可共享库 | 独立 schema |
| 强合规、大客户 | 独立实例 |
| 混合场景 | 分层：小租户共享，大客户独立 |

> 选型不是一次性的，应支持租户在方案之间平滑迁移。

## 2.3 元数据过滤（推荐起点）

### 2.3.1 机制

写入时给每条向量打上 `tenantId` 标签，检索时由服务端强制注入过滤条件，只返回当前租户的数据。

### 2.3.2 代码示例

```java
// 写入时打标签
TextSegment segment = TextSegment.from(text,
        Metadata.from(Map.of("tenantId", tenantId)));

// 检索时强制过滤
EmbeddingStoreContentRetriever.builder()
        .dynamicFilter(q -> metadataKey("tenantId")
                .isEqualTo(TenantContext.currentTenant()))
        .build();
```

```java
// Spring AI
SearchRequest.builder()
        .query(q)
        .filterExpression("tenantId == '" + tenant + "'")
        .build();
```

### 2.3.3 边界与坑

- **过滤条件必须由服务端注入，绝不能来自客户端参数。**
- 字符串拼接过滤表达式存在注入风险，应使用参数化或白名单校验。

## 2.4 独立集合/表

```java
// 按租户分表
String table = "embeddings_" + tenantId;
// 或分集合
String collection = "kb_" + tenantId;
```

优点：隔离强、单租户查询快；缺点：租户多时管理复杂，DDL 与迁移成本高。

## 2.5 生产建议

- 免费/标准租户用元数据过滤，企业租户可独立 schema 或独立实例。
- 为过滤字段建立索引，避免全表扫描。

# 3. 会话与记忆隔离

## 3.1 结论先行

会话 ID 与记忆存储必须绑定租户，否则不同租户的对话历史会相互污染。

## 3.2 机制拆解

会话 ID 采用 `tenantId:userId:uuid` 复合结构，既保证唯一性又便于按租户清理。记忆存储按租户路由到独立的 store 或分区。

## 3.3 代码示例

```java
// 会话 ID 含租户
String sessionId = tenantId + ":" + userId + ":" + UUID.randomUUID();

// 记忆按租户隔离
ChatMemoryProvider provider = memoryId ->
        MessageWindowChatMemory.builder()
                .id(memoryId)
                .chatMemoryStore(tenantStore(TenantContext.currentTenant()))
                .build();
```

## 3.4 边界与坑

- 若从请求参数直接取 `sessionId`，攻击者可能伪造他租户会话 ID 读取记忆。
- 记忆 TTL 需按租户配置，避免长期占用存储。

## 3.5 生产建议

- 会话 ID 由服务端生成并签名，客户端不可篡改。
- 记忆读取前再次校验会话归属租户。

# 4. 模型与配额隔离

## 4.1 结论先行

不同租户等级应映射到不同模型层级与配额上限，防止免费租户耗尽高成本模型额度。

## 4.2 配额模型

| 租户等级 | 模型 | 配额 | 并发 |
|----------|------|------|------|
| 免费 | 小模型 | 低 | 低 |
| 标准 | 中模型 | 中 | 中 |
| 企业 | 大模型 | 高 | 高 |

### 4.2.1 配额维度

| 维度 | 说明 | 超限处理 |
|------|------|----------|
| 日 token 数 | 每日累计消耗 | 拒绝或降级小模型 |
| 日成本 | 每日金额上限 | 告警 + 拒绝 |
| 并发数 | 同时在途请求 | 排队或拒绝 |
| QPS | 每秒请求数 | 限流 |

### 4.2.2 原子扣减

```java
// 使用 Redis 原子计数，避免"检查-扣减"竞态
public boolean tryAcquire(String tenantId, int tokens, int limit) {
    Long used = redis.opsForValue()
            .increment("quota:" + tenantId, tokens);
    if (used > limit) {
        redis.opsForValue().decrement("quota:" + tenantId, tokens);
        return false;
    }
    return true;
}
```

> 检查与扣减必须原子完成，否则高并发下会超发。

## 4.3 代码示例

```java
public record TenantConfig(
        String tenantId,
        String modelTier,               // 可用模型层级
        long dailyTokens,
        BigDecimal dailyCost,
        int maxConcurrency
) {}

public void checkQuota(TenantConfig config, int estimatedTokens) {
    if (quotaService.usedToday(config.tenantId()) + estimatedTokens
            > config.dailyTokens()) {
        throw new QuotaExceededException("租户配额已用尽");
    }
}
```

## 4.4 边界与坑

- 配额检查与实际扣减之间存在竞态，高并发下可能超发，应使用原子计数器或分布式限流。
- 按 token 预估可能偏差大，需在流结束后按实际用量回补。

## 4.5 生产建议

- 配额接近上限时提前告警，避免硬性拒绝影响体验。
- 并发上限按租户舱壁（Bulkhead）隔离，防止单租户拖垮全局。

# 5. 密钥隔离

## 5.1 结论先行

大客户常要求 **BYOK（Bring Your Own Key，自带密钥）**，因此密钥解析必须支持租户级粒度。

## 5.2 机制拆解

通过统一的 `TenantSecretProvider` 接口按 `tenantId + provider` 解析密钥，底层对接密钥管理服务（Vault/KMS），避免密钥落到应用配置或数据库明文。

## 5.3 代码示例

```java
public interface TenantSecretProvider {
    String apiKey(String tenantId, String provider);
}

// 租户自带密钥（BYOK）
public String resolveApiKey(String tenantId) {
    return secretManager.get("tenant/%s/openai-key".formatted(tenantId));
}
```

## 5.4 边界与坑

- 密钥解析结果不可缓存过久，轮换后要能及时生效。
- 日志中绝不能打印密钥，注意脱敏。

## 5.5 生产建议

- 平台级密钥与租户级密钥分路径存储，便于审计。
- 记录密钥使用记录，支持按租户对账。

# 6. 工具与权限隔离

## 6.1 结论先行

Agent 可调用的工具必须按"租户配置 + 用户角色"裁剪，只挂载有权限的工具，而不是运行时再判断。

## 6.2 机制拆解

先计算允许的工具集合，再在构建 Assistant 时只挂载该集合。未挂载的工具模型根本看不到，从源头消除越权可能。

## 6.3 代码示例

```java
public List<ToolCallback> toolsFor(UserContext user) {
    // 按租户配置 + 用户角色决定可用工具
    Set<String> allowed = permissionService.allowedTools(
            user.tenantId(), user.roles());

    return allTools.stream()
            .filter(t -> allowed.contains(t.name()))
            .toList();
}
```

```java
Assistant assistant = AiServices.builder(Assistant.class)
        .chatModel(model)
        .tools(toolsFor(user))            // 只挂载有权限的工具
        .build();
```

## 6.4 边界与坑

- 仅在 Prompt 中声明"不要调用某工具"不可靠，模型可能被诱导。
- 工具内部仍需二次鉴权，防止参数被构造绕过。

## 6.5 生产建议

- 敏感工具（如数据库写入、发邮件）要求额外确认或人工审批。
- 记录每次工具调用，纳入审计。

# 7. 成本与计费隔离

## 7.1 结论先行

成本必须按租户实时归集，既能计费，也能在超预算时及时告警。

## 7.2 代码示例

```java
public void recordCost(String tenantId, String model, Usage usage) {
    BigDecimal cost = price(model).calculate(usage);

    costStore.add(tenantId, cost);
    meterRegistry.counter("ai.cost", "tenant", tenantId).increment(cost.doubleValue());

    if (costStore.daily(tenantId).compareTo(budget(tenantId)) > 0) {
        alertService.warn("租户成本超预算: " + tenantId);
    }
}
```

### 7.2.1 价格表配置化

```yaml
app:
  pricing:
    gpt-4o-mini:
      input-per-1k: 0.00015
      output-per-1k: 0.0006
    gpt-4o:
      input-per-1k: 0.005
      output-per-1k: 0.015
```

### 7.2.2 异常路径记账

```java
return chatClient.prompt().user(q).stream().content()
        .doOnComplete(() -> recordCost(
                tenantId, model, usageTracker.snapshot()))
        .doOnError(e -> recordCost(
                tenantId, model, usageTracker.snapshot()));
```

> 成功与失败都要记账，否则异常重试会漏计成本。

## 7.3 边界与坑

- 模型价格频繁变动，价格表应配置化并带生效时间。
- 流式场景在结束时才拿到完整用量，需保证异常路径也能记账。

## 7.4 生产建议

- 成本指标接入监控大盘，按租户维度展示。
- 计费数据不可变，写入后只追加不回改。

# 8. 租户上下文传递

## 8.1 结论先行

租户上下文需要在一次请求的整个链路中传递，WebFlux 场景下 `ThreadLocal` 会因线程切换失效，应优先使用 Reactor Context。

## 8.2 ThreadLocal 方案

```java
public class TenantContext {

    private static final ThreadLocal<String> CURRENT = new ThreadLocal<>();

    public static void set(String tenantId) { CURRENT.set(tenantId); }
    public static String current() { return CURRENT.get(); }
    public static void clear() { CURRENT.remove(); }
}

// WebFilter 中设置
@Component
public class TenantFilter implements WebFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String tenantId = resolveTenant(exchange.getRequest());
        TenantContext.set(tenantId);
        return chain.filter(exchange)
                .doFinally(s -> TenantContext.clear());   // 必须清理
    }
}
```

## 8.3 Reactor Context 方案

### 8.3.1 机制

Reactor Context 随 `Flux`/`Mono` 一起流动，跨线程切换时依然可读，是响应式场景的正确选择。

### 8.3.2 代码示例

```java
public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
    String tenantId = resolveTenant(exchange.getRequest());
    return chain.filter(exchange)
            .contextWrite(ctx -> ctx.put("tenantId", tenantId));
}

// 读取
String tenant = Mono.deferContextual(ctx -> Mono.just(ctx.get("tenantId")))
        .block();
```

## 8.4 边界与坑

- **ThreadLocal 必须清理**，否则线程复用时串租户。
- Reactor Context 是只读的，只能通过 `contextWrite` 向上游写入。

## 8.5 生产建议

- 统一封装上下文读写工具类，禁止业务代码直接操作 ThreadLocal。
- 为上下文缺失设置快速失败，避免默认租户导致数据错乱。

# 9. 越权测试（必做）

## 9.1 结论先行

每个 RAG、记忆、工具接口都必须有跨租户越权测试，这是多租户系统最重要的安全防线。

## 9.2 代码示例

```java
@Test
void tenant_A_cannot_access_tenant_B_data() {
    // 以租户 B 身份写入
    ingestDocument("secret-of-B", "tenant-B");

    // 以租户 A 身份检索
    List<Document> results = retrieveAs("tenant-A", "secret");

    // 必须检索不到
    assertTrue(results.stream()
            .noneMatch(d -> d.getText().contains("secret-of-B")));
}
```

### 9.2.1 测试用例设计

| 场景 | 构造方式 | 期望 |
|------|----------|------|
| 伪造 tenantId | 请求头注入他人租户 | 服务端忽略，使用认证租户 |
| 篡改会话 ID | 传入他人 sessionId | 拒绝访问 |
| 缓存穿透 | 复用他人缓存 key | 不命中 |
| 工具越权 | 调用未授权工具 | 工具不可见 |

## 9.3 测试矩阵

| 接口类型 | 越权用例 |
|----------|----------|
| 向量检索 | A 查不到 B 的文档 |
| 会话记忆 | A 读不到 B 的会话 |
| 工具调用 | A 调不了 B 专属工具 |
| 配置读取 | A 改不了 B 的配置 |

## 9.4 边界与坑

- 只测正常路径不够，要构造伪造 `tenantId`、篡改会话 ID 等攻击用例。
- 缓存层也要纳入测试，避免缓存 key 未含租户导致串数据。

## 9.5 生产建议

- 把越权测试纳入 CI 必过门禁。
- 定期做渗透测试，验证隔离边界。

# 10. 生产注意事项

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 过滤条件来自客户端 | 服务端强制注入 |
| ThreadLocal 未清理 | finally 清理 / Reactor Context |
| 大租户挤占资源 | 按租户舱壁隔离 |
| 密钥混用 | 租户级密钥 |
| 无越权测试 | 每个接口都测 |
| 成本未按租户统计 | 分租户计费 |
| 缓存 key 未含租户 | 租户维度参与 key |

## 10.2 容量与限流

- 按租户设置 QPS 与并发上限，使用令牌桶或漏桶。
- 单租户异常流量触发熔断，避免影响其他租户。

### 10.2.1 并发舱壁隔离

```java
// 按租户并发舱壁隔离
private final Map<String, Semaphore> bulkheads = new ConcurrentHashMap<>();

public <T> Mono<T> withBulkhead(String tenantId, int maxConcurrent, Mono<T> task) {
    Semaphore sem = bulkheads.computeIfAbsent(tenantId,
            id -> new Semaphore(maxConcurrent));
    if (!sem.tryAcquire()) {
        return Mono.error(new TooManyRequestsException("租户并发已满"));
    }
    return task.doFinally(s -> sem.release());
}
```

## 10.3 审计与合规

- 所有跨租户访问尝试都要记录审计日志。
- 支持按租户导出操作记录，满足合规要求。

# 11. 小结

- 多租户需隔离六样：数据、模型配额、密钥、权限、成本、审计。
- 数据隔离从"元数据过滤"起步，大客户可独立实例。
- 过滤条件必须服务端注入，绝不信任客户端。
- 租户上下文用 ThreadLocal/Reactor Context，注意清理。
- 越权测试是必做项，应纳入 CI 门禁。
