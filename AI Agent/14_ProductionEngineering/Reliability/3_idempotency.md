# 1. 为什么幂等是刚需

结论先行：在 Agent 与分布式系统中，重复执行几乎必然发生；没有幂等（Idempotency）保护，重试、恢复、重放都会变成重复下单、重复扣款、重复发消息。

Agent 场景下，重复执行几乎必然发生：

```
- 工具超时后重试
- 服务崩溃后恢复重放
- 消息队列重复投递（至少一次语义）
- 用户重复点击
- 分布式故障转移
```

没有幂等 = 重复下单、重复扣款、重复发消息。

## 1.1 重复执行的来源

结论先行：重复来自"至少一次投递"的普遍语义与故障恢复机制，而非偶发错误。

| 来源 | 说明 |
|------|------|
| 超时重试 | 请求已成功但响应丢失 |
| 崩溃恢复 | 恢复时重放未确认操作 |
| 消息队列 | 至少一次（At-least-once）投递 |
| 用户操作 | 重复点击、重复提交 |
| 故障转移 | 主备切换后重复执行 |

## 1.2 没有幂等的后果

结论先行：重复副作用会直接造成资金与数据错误，且难以自动修复。

```
一次下单意图
├── 执行 1 次  → 正常
└── 执行 2 次  → 重复订单 / 重复扣款 / 重复通知
```

| 操作 | 重复执行后果 |
|------|-------------|
| 创建订单 | 重复订单 |
| 扣款 | 多扣资金 |
| 发消息 | 用户被打扰 |
| 发券 | 资产损失 |

## 1.3 幂等与其他机制的关系

结论先行：幂等是重试、恢复、分布式一致性的公共前提，没有它这些机制都不安全。

| 机制 | 对幂等的依赖 |
|------|-------------|
| 重试 | 写操作重试的前提 |
| 恢复重放 | 重放安全的前提 |
| 消息消费 | 至少一次语义的兜底 |
| 故障转移 | 重复执行的兜底 |

# 2. 幂等的定义

结论先行：幂等的数学定义是"同一操作执行一次与执行多次，效果相同"；关键在于"效果"而非"返回值"。

```
f(f(x)) = f(x)
同一请求执行一次与执行多次，效果相同
```

## 2.1 数学定义

结论先行：幂等指对同一输入重复应用函数，结果不变。

| 表达式 | 含义 |
|--------|------|
| `f(f(x)) = f(x)` | 重复应用不改变结果 |
| 一次与多次等价 | 副作用只发生一次 |

## 2.2 操作的幂等性分类

结论先行：查询、覆盖更新、按 ID 删除天然幂等；插入、累加、转账、发消息需显式实现。

| 操作 | 天然幂等 |
|------|:--------:|
| 查询（GET） | 是 |
| 覆盖式更新（SET x=1） | 是 |
| 删除（按 ID） | 是 |
| 插入（INSERT） | 否 |
| 累加（x += 1） | 否 |
| 转账 | 否 |
| 发消息 | 否 |

> 写操作、累加操作、有副作用的操作必须显式实现幂等。

## 2.3 业务幂等与请求幂等的区别

结论先行：请求幂等关注"同一请求不重复执行"，业务幂等关注"同一业务意图只生效一次"，后者更接近需求本质。

| 维度 | 请求幂等 | 业务幂等 |
|------|---------|---------|
| 粒度 | 一次调用 | 一次业务意图 |
| 标识 | 请求 ID | 业务唯一键 |
| 范围 | 单接口 | 跨接口/跨服务 |

# 3. 幂等键设计

结论先行：幂等键（Idempotency Key）用来唯一标识"一次业务意图"，其设计质量直接决定幂等方案是否有效。

```
幂等键 = 唯一标识"一次业务意图"
```

## 3.1 幂等键的含义

### 3.1.1 唯一标识一次业务意图

结论先行：键相同的请求被视为同一次意图，服务端只应让第一次真正生效。

| 属性 | 要求 |
|------|------|
| 唯一性 | 不同意图不同键 |
| 稳定性 | 同一意图重试时键不变 |
| 可存储 | 长度可控、可索引 |

### 3.1.2 生成时机

结论先行：幂等键必须在"发起业务意图时"生成一次，并在整个重试周期内复用。

```
生成 key ──▶ 第一次调用 ──▶ 失败 ──▶ 重试（复用同一 key）
```

## 3.2 生成方案

结论先行：按是否有天然唯一标识选择方案，无标识时用内容哈希。

| 方案 | 说明 | 适用 |
|------|------|------|
| 客户端生成 UUID | 一次请求一个 key | 通用 |
| 业务唯一键 | 订单号、流水号 | 有天然唯一标识 |
| 内容哈希 | 参数哈希 | 无天然标识 |
| 组合键 | taskId + step + tool | Agent 工具 |

## 3.3 Agent 场景的键设计

结论先行：Agent 工具用"任务 + 步骤 + 工具名 + 参数哈希"组合，能精确对应一次工具意图。

```java
// Agent 工具：taskId + step + 工具名 + 参数哈希
String idempotencyKey = "%s:%d:%s:%s".formatted(
        taskId, step, toolName, sha256(arguments));
```

> 关键：重试时必须复用同一个幂等键，不能每次重试都生成新 key。

# 4. 实现方案一：唯一约束

结论先行：利用数据库唯一约束（Unique Constraint）拦截重复插入，是最强一致的方案。

## 4.1 原理

结论先行：给幂等键加唯一索引，重复插入会抛约束冲突，捕获后返回已有记录。

```
插入 key=abc ──▶ 成功
插入 key=abc ──▶ 唯一约束冲突 ──▶ 查询已有记录返回
```

## 4.2 表结构与代码

结论先行：幂等键列加 `UNIQUE` 约束，代码捕获冲突后回查。

```sql
CREATE TABLE orders (
    id              BIGSERIAL PRIMARY KEY,
    idempotency_key VARCHAR(128) NOT NULL UNIQUE,   -- 唯一约束
    user_id         BIGINT       NOT NULL,
    amount          DECIMAL(10,2) NOT NULL,
    status          VARCHAR(32)  NOT NULL,
    created_at      TIMESTAMP    NOT NULL DEFAULT now()
);
```

```java
import org.springframework.dao.DataIntegrityViolationException;

public OrderResult createIdempotent(String key, OrderRequest req) {
    try {
        Order order = orderRepository.save(new Order(key, req));
        return OrderResult.ok(order);
    } catch (DataIntegrityViolationException e) {
        // 唯一约束冲突 → 已创建，返回已有记录
        Order existing = orderRepository.findByIdempotencyKey(key);
        return OrderResult.ok(existing);
    }
}
```

优点：强一致；缺点：依赖数据库唯一约束。

## 4.3 边界与坑

结论先行：唯一约束方案需处理"回查为空"与"并发插入"的竞态。

| 坑 | 解决 |
|----|------|
| 冲突后回查为空 | 重试回查或短事务 |
| 键列过长 | 用哈希压缩 |
| 分库分表 | 唯一约束跨库失效 |

生产建议：回查不到时短暂重试，避免把异常抛给用户。

# 5. 实现方案二：Redis SETNX

结论先行：用 Redis 的 `SET NX`（不存在才设置）实现幂等锁，性能高且支持 TTL，但需处理"处理中"状态与降级。

## 5.1 原理

结论先行：以幂等键为 Redis Key，首次写入成功者获得执行权，其余返回已有结果或"处理中"。

```
SETNX idem:order:abc = PROCESSING
├── 成功 → 本请求执行
└── 失败 → 已存在，读取状态/结果
```

## 5.2 完整实现

### 5.2.1 正常流程

结论先行：先抢占，执行成功后写入 DONE 与结果。

```java
import java.time.Duration;

public OrderResult createIdempotent(String key, OrderRequest req) {
    String redisKey = "idem:order:" + key;

    Boolean first = redis.opsForValue()
            .setIfAbsent(redisKey, "PROCESSING", Duration.ofHours(24));

    if (!Boolean.TRUE.equals(first)) {
        // 已存在：返回上次结果或提示处理中
        String status = redis.opsForValue().get(redisKey);
        if ("PROCESSING".equals(status)) {
            throw new IllegalStateException("请求处理中，请勿重复提交");
        }
        return deserialize(redis.opsForValue().get(redisKey + ":result"));
    }

    try {
        Order order = doCreate(req);
        redis.opsForValue().set(redisKey, "DONE", Duration.ofHours(24));
        redis.opsForValue().set(redisKey + ":result", serialize(order),
                Duration.ofHours(24));
        return OrderResult.ok(order);
    } catch (Exception e) {
        redis.delete(redisKey);          // 失败则释放，允许重试
        throw e;
    }
}
```

### 5.2.2 处理中与失败

结论先行：PROCESSING 状态用于拦截并发重复执行，失败则释放锁允许重试。

| 状态 | 行为 |
|------|------|
| PROCESSING | 拒绝或等待 |
| DONE | 返回缓存结果 |
| 执行异常 | 删除 key，允许重试 |

## 5.3 边界与坑

结论先行：Redis 方案的核心风险是"锁过期但业务未完成"与"Redis 不可用"。

| 坑 | 后果 | 解决 |
|----|------|------|
| TTL 太短 | 执行中锁过期 | TTL 覆盖最长执行时间 |
| 释放锁误删 | 删除他人锁 | 用 value 校验或 Lua |
| Redis 宕机 | 幂等失效 | 降级到数据库约束 |

生产建议：Redis 只做第一道闸门，数据库唯一约束做最终兜底。

# 6. 实现方案三：状态机 + 版本号

结论先行：用状态机（State Machine）与乐观锁版本号实现幂等，适合状态流转类操作，无需额外存储。

## 6.1 原理

结论先行：带前置状态与版本条件的更新，只有第一次能匹配成功，后续更新影响行数为 0。

```sql
UPDATE orders
SET status = 'PAID', version = version + 1
WHERE order_id = ? AND status = 'PENDING' AND version = ?
```

## 6.2 实现

结论先行：影响行数为 0 时视为"已被处理"，返回当前状态即可。

```java
int updated = jdbc.update("""
        UPDATE orders SET status = ?, version = version + 1
        WHERE order_id = ? AND status = ? AND version = ?
        """, newStatus, orderId, expectedStatus, expectedVersion);

if (updated == 0) {
    // 状态已变（可能已被处理）→ 幂等返回
    return OrderResult.ok(repository.get(orderId));
}
```

## 6.3 边界与坑

结论先行：该方案只适合有明确状态流转的操作，且需防止 ABA 问题。

| 坑 | 说明 |
|----|------|
| 无状态流转 | 不适用（如累加） |
| 版本号回绕 | 需防 ABA |
| 并发更新 | 依赖数据库行锁 |

优点：无需额外存储；缺点：只适合状态流转类操作。

# 7. 三种方案对比

结论先行：三种方案各有适用边界，生产上常组合使用。

## 7.1 对比表

| 方案 | 一致性 | 性能 | 适用 |
|------|-------|------|------|
| 唯一约束 | 强 | 中 | 数据库写 |
| Redis SETNX | 最终 | 高 | 高并发、有 TTL |
| 状态机版本号 | 强 | 高 | 状态流转 |

## 7.2 组合策略

结论先行：Redis 做前置拦截、数据库约束做最终兜底，兼顾性能与一致性。

```
请求 ──▶ Redis 幂等锁 ──▶ 业务执行 ──▶ 数据库唯一约束兜底
```

生产建议：核心资金链路必须保留数据库层的强一致兜底。

# 8. 工具幂等（Agent 场景）

结论先行：Agent 工具执行需在框架层统一做幂等，避免每个工具各自实现。

## 8.1 幂等执行器

结论先行：以 taskId + step + 工具签名作为键，命中缓存直接返回结果。

```java
@Component
@RequiredArgsConstructor
public class IdempotentToolExecutor {

    private final AgentStateStore stateStore;
    private final IdempotencyStore idempotencyStore;

    public ToolResult execute(String taskId, int step, ToolCall call) {
        String key = "%s:%d:%s".formatted(taskId, step, call.signature());

        // 1. 检查是否已执行
        Optional<ToolResult> cached = idempotencyStore.get(key);
        if (cached.isPresent()) {
            log.info("工具已执行，返回缓存: {}", key);
            return cached.get();
        }

        // 2. 执行
        ToolResult result = doExecute(call);

        // 3. 记录结果（只读工具也记录，避免重复调用）
        idempotencyStore.put(key, result, Duration.ofHours(24));
        return result;
    }
}
```

## 8.2 只读工具去重

结论先行：只读工具重试虽无副作用，但浪费成本与时间，应做签名去重。

```java
private final Set<String> calledSignatures = ConcurrentHashMap.newKeySet();

public String execute(ToolCall call) {
    String sig = call.name() + ":" + call.arguments();

    if (!calledSignatures.add(sig)) {
        return "该查询已执行过，结果如上，请勿重复调用。";
    }
    return doExecute(call);
}
```

> 在提示中也声明"不要重复调用相同参数的工具"。

## 8.3 提示词配合

结论先行：在系统提示中明确幂等语义，减少模型重复调用与无意义重试。

- 声明相同参数工具不重复调用。
- 返回缓存结果时标注"已执行过"。
- 对写工具说明重试需复用幂等键。

# 9. 幂等键的生命周期

结论先行：幂等键需按状态设置不同 TTL，且 TTL 必须覆盖最大重试窗口。

## 9.1 状态流转

结论先行：幂等键经历创建、处理中、完成或失败四个阶段。

```
创建 ──▶ 处理中 ──▶ 已完成（保留 TTL）
            │
            └──▶ 失败（释放，允许重试）
```

| 状态 | TTL | 说明 |
|------|-----|------|
| PROCESSING | 短（如 5 分钟） | 超时自动释放 |
| DONE | 长（24 小时~7 天） | 覆盖重试窗口 |
| FAILED | 立即释放 | 允许重试 |

> TTL 必须覆盖最大重试窗口，否则重试会重复执行。

## 9.2 TTL 设计

结论先行：TTL 过短会导致重试时键已过期，过长则浪费存储。

| 业务 | 建议 TTL |
|------|---------|
| 即时支付 | 24 小时 |
| 批量任务 | 7 天 |
| 短时查询 | 5 分钟 |

## 9.3 存储降级

结论先行：幂等存储不可用时，必须明确选择"拒绝"或"放行"，不能静默失效。

| 策略 | 说明 | 适用 |
|------|------|------|
| 拒绝 | 返回不可用，要求重试 | 资金类 |
| 放行 | 依赖下游唯一约束兜底 | 有兜底 |
| 本地缓存 | 单机兜底 | 低风险 |

# 10. 监控与审计

结论先行：幂等的命中率与冲突率需要监控，冲突情况需要审计留痕。

## 10.1 指标

结论先行：监控命中率、冲突率与存储可用性。

| 指标 | 含义 |
|------|------|
| `idem.hit` | 幂等命中次数 |
| `idem.conflict` | 并发冲突次数 |
| `idem.store.error` | 存储异常 |

## 10.2 审计

结论先行：记录幂等键与对应结果，便于事后追溯重复请求。

- 记录幂等键、请求参数摘要、结果状态。
- 保留时间覆盖业务对账周期。
- 冲突事件单独告警与归档。

# 11. 生产注意事项

结论先行：幂等事故多源于键设计错误与 TTL 不足，需按清单逐项检查。

## 11.1 常见坑

| 坑 | 后果 | 解决 |
|----|------|------|
| 每次重试生成新 key | 幂等失效 | 复用同一 key |
| TTL 太短 | 重试时 key 已过期 | TTL ≥ 重试窗口 |
| 只加唯一约束不处理冲突 | 抛异常给用户 | 冲突时返回已有结果 |
| 幂等键不含用户 | 跨用户冲突 | key 含 tenantId+userId |
| 忽略"处理中"状态 | 并发重复执行 | 用 SETNX + 状态 |
| 只读工具不去重 | 浪费成本 | 签名去重 |

## 11.2 检查清单

结论先行：上线前逐项确认，可覆盖绝大多数幂等缺陷。

- 写操作是否实现幂等。
- 幂等键是否唯一标识业务意图。
- 重试是否复用同一幂等键。
- TTL 是否覆盖最大重试窗口。
- 存储不可用时降级策略是否明确。
- 是否监控命中率与冲突率。
- 是否保留幂等审计记录。

# 12. 小结

## 12.1 要点回顾

- 幂等是重试、恢复、分布式的前提。
- 幂等键标识"一次业务意图"，重试必须复用。
- 三种实现：唯一约束、Redis SETNX、状态机版本号。

## 12.2 落地行动

- 只读工具也应签名去重，节省成本。
- TTL 必须覆盖最大重试窗口。
