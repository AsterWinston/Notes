
# 1. 调优全景图

```
Sentinel 性能调优维度：
┌─────────────┬──────────────┬──────────────┬─────────────┐
│  规则配置    │   统计窗口    │   线程模型    │   Dashboard  │
├─────────────┼──────────────┼──────────────┼─────────────┤
│ 规则数量控制  │ 窗口大小/Len  │ Netty 线程    │ 部署资源     │
│ 规则复杂度    │ Bucket 数量  │ 全局线程池    │ 心跳频率     │
│ 规则分组优化  │ 指标精度权衡  │ 同步/异步     │ 集群模式     │
└─────────────┴──────────────┴──────────────┴─────────────┘
```

# 2. 规则性能优化

## 2.1 规则数量

Sentinel 为每个资源的每个 Rule 执行一次检查。规则数量影响 SlotChain 执行开销：

```java
// ❌ 避免：为每个接口创建独立规则（100 个接口 × 3 条规则 = 300 条）
for (String api : apis) {
    FlowRule rule = new FlowRule(api);
    rule.setCount(100);
    FlowRuleManager.loadRules(Collections.singletonList(rule));
}

// ✓ 推荐：使用网关级 + API 分组级规则
// 路由级：每个路由 1 条规则（10 条）
// API 分组级：每个分组 1-2 条规则（5 条）
// 共 15 条规则，而非 300 条
```

**性能指标**：100 条 FlowRule 匹配耗时约 1-2 μs，300 条约 5-10 μs。

## 2.2 规则评估开销

```java
// FlowSlot 的执行时间可在极端情况下触发
// 每个请求的 SlotChain 执行总耗时 ≈ 5-15 μs（正常）
// 规则过多时可能增长到 50-100 μs
```

# 3. 滑动窗口调优

## 3.1 窗口参数配置

```java
// 调整 Sentinel 统计精度与内存的平衡
// 默认：2 样本 / 1 秒（500ms 窗口 × 2）
SentinelConfig.setConfig(SampleCountProperty.SAMPLE_COUNT, "4");    // 4 样本
SentinelConfig.setConfig(IntervalProperty.INTERVAL, "2000");       // 2 秒统计周期

// 修改后：4 窗口 × 500ms = 2 秒统计周期
// 内存增加约 2×，但统计精度更高
```

## 3.2 窗口数 vs 内存

| SampleCount | Interval | 窗口长度 | 内存占用（单资源） | 精度 |
|-------------|----------|---------|------------------|------|
| 2（默认） | 1000ms | 500ms | ~200B | 低 |
| 10 | 1000ms | 100ms | ~1KB | 高 |
| 20 | 1000ms | 50ms | ~2KB | 极高 |
| 60 | 60000ms | 1000ms | ~6KB | 中（分钟级） |

## 3.3 降低内存占用

```java
// 控制统计资源数量
SentinelConfig.setConfig("csp.sentinel.statistic.max.rt", "5000");  // 最大记录 RT

// 关闭每分钟统计（仅保留秒级）
// 减少 StatisticNode 中 rollingCounterInMinute 的内存开销
```

# 4. Dashboard 性能

## 4.1 JVM 参数

```bash
java -Xms512m -Xmx1g \
     -Xss256k \
     -XX:+UseG1GC \
     -Dserver.port=8080 \
     -jar sentinel-dashboard.jar
```

## 4.2 机器数量限制

Dashboard 管理的机器数量影响性能：

| 机器数 | 推荐配置 | 建议 |
|--------|---------|------|
| < 50 | 默认配置 | — |
| 50-200 | 增加内存至 2G | 调整心跳间隔 |
| > 200 | 水平扩展 Dashboard + 分区管理 | 按环境拆 Dashboard |

## 4.3 心跳频率

```yaml
spring:
  cloud:
    sentinel:
      transport:
        heartbeat-interval-ms: 30000  # 默认 10s，可调整到 30s
```

# 5. 集群流控性能

## 5.1 Token Server 配置

```java
// Token Server JVM
java -Xms1g -Xmx2g \
     -Dproject.name=sentinel-token-server \
     -Dserver.port=18730 \
     -jar token-server.jar

// Client 连接配置
ClusterClientConfig config = new ClusterClientConfig();
config.setRequestTimeout(200);  // Token 请求超时(ms)
```

## 5.2 集群模式选择

| 模式 | 适用场景 | 延迟 | 可靠性 |
|------|---------|------|--------|
| 独立 Token Server | 限流精度要求高 | +1-5ms | 高 |
| 嵌入 Token Server | 小规模集群（<10 实例） | +0-1ms | 中 |

# 6. 高并发场景优化

## 6.1 无锁统计

Sentinel 使用 `LongAdder` 而非 `AtomicLong`，在 100+ 线程并发时吞吐量提升约 3-5×。

## 6.2 减少 Entry 创建

```java
// ❌ 避免：循环中重复创建 Entry
for (Item item : items) {
    try (Entry e = SphU.entry("processItem")) {
        process(item);
    }
}

// ✓ 推荐：批量处理
try (Entry e = SphU.entry("processItems")) {
    for (Item item : items) {
        process(item);
    }
}
```

## 6.3 Context 复用

```java
// 同一请求链路中复用 Context
ContextUtil.enter("order-context", "order-app");
try {
    // 多次 entry 共享同一个 Context
    try (Entry e1 = SphU.entry("getUser")) { ... }
    try (Entry e2 = SphU.entry("getOrder")) { ... }
} finally {
    ContextUtil.exit();
}
```

# 7. 压测基准参考

```
测试环境：4C8G，100 并发连接，接口 RT=5ms

| 配置 | 无 Sentinel | Sentinel(2规则) | Sentinel(10规则) |
|------|------------|----------------|-----------------|
| QPS | 25000 | 24000 (-4%) | 22000 (-12%) |
| P99 RT | 8ms | 9ms | 12ms |
| CPU | 60% | 65% | 75% |
```

结论：Sentinel 在规则数量合理时（< 10 条/资源），性能损耗可控制在 5% 以内。

# 8. 最佳实践

1. **规则总量控制**：单个应用规则总数建议 < 200 条
2. **窗口参数默认即可**：生产环境无需修改，除非有特殊精度要求
3. **Dashboard 独立部署**：不与业务服务混合部署
4. **关闭 Debug 日志**：生产环境 `csp.sentinel.log.level=WARN`
5. **监控 Sentinel 自身**：关注 `FlowSlot.check` 和 `StatisticSlot.entry` 的耗时
