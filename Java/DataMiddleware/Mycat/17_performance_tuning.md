
# 1. 调优全景图

```
Mycat2 性能调优维度：
┌─────────────┬──────────────┬──────────────┬─────────────┐
│  JVM 层      │   Reactor    │   连接池      │  SQL 路由    │
├─────────────┼──────────────┼──────────────┼─────────────┤
│ 堆内存 / GC  │ 线程数        │ 前端/后端连接  │ 分片命中     │
│ 堆外内存     │ Buffer 大小  │ 超时配置      │ 避免全扫描   │
│ 启动参数     │ 序列化框架    │ 连接复用      │ 结果合并     │
└─────────────┴──────────────┴──────────────┴─────────────┘
```

# 2. JVM 调优

```bash
# Mycat2 启动 JVM 参数
java -server -Xms2g -Xmx4g \
     -Xmn1g \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -XX:InitiatingHeapOccupancyPercent=45 \
     -XX:+DisableExplicitGC \
     -XX:MaxDirectMemorySize=1g \
     -Dio.netty.allocator.type=pooled \
     -jar mycat2.jar
```

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `-Xms/-Xmx` | 堆内存 | 2G/4G |
| `-Xmn` | 年轻代 | 堆的 1/4 |
| `-XX:MaxDirectMemorySize` | 堆外内存（Netty/Vert.x 使用） | 1G |
| `-Dio.netty.allocator.type` | Netty 内存分配器 | pooled |

# 3. Reactor 线程调优

```yaml
# mycat.yml
server:
  reactorNumber: 8          # Reactor 线程数
  bufferPoolPageSize: 4194304    # 4MB 页大小
  bufferPoolChunkSize: 4096      # 4KB 块大小
  processorBufferPoolType: "direct"  # direct / heap
```

**Reactor 线程数建议**：

| CPU 核心数 | 推荐线程数 | 说明 |
|-----------|-----------|------|
| 4 | 4 | — |
| 8 | 8-16 | I/O 密集型可多配 |
| 16 | 16-32 | — |
| 32 | 32-48 | — |

```java
// 原则：Reactor 线程数 = CPU 核心数 × 2（I/O 密集型）
// 过多：上下文切换开销 > 并行收益
// 过少：无法充分利用多核
```

# 4. 连接池调优

```yaml
server:
  maxCon: 10000              # 最大前端连接数

# 数据源连接池
datasources:
  ds1:
    minCon: 5                # 最小空闲连接
    maxCon: 100              # 最大连接
    maxRetryCount: 3         # 获取连接重试
    idleTimeout: 60000       # 空闲超时(ms)
    connectionTimeout: 3000  # 获取连接超时(ms)
```

**连接数计算公式**：

```
后端连接数 ≈ (前端 QPS × 平均 RT / 1000) / 每个分片

示例：
  前端 QPS = 5000
  平均 RT = 10ms
  分片数 = 4

  每分片连接数 = (5000 × 0.01) / 4 ≈ 12.5 → 配 20
  总后端连接数 = 20 × 4 = 80
```

# 5. SQL 路由优化

```sql
-- ✓ 最优：精确分片查询（只访问 1 个分片）
SELECT * FROM orders WHERE id = 1;

-- △ 可接受：IN 查询（访问已知的 N 个分片）
SELECT * FROM orders WHERE id IN (1, 5, 9);

-- ✗ 差：全分片扫描（访问所有分片）
SELECT * FROM orders WHERE user_id = 100;  -- user_id 不是分片键
SELECT * FROM orders ORDER BY create_time;  -- 无分片键
```

**分片命中率监控**：

```sql
/*!mycat:showSQLStat*/
-- +------------------+--------+----------+
-- | sql              | count  | avgShards|
-- +------------------+--------+----------+
-- | SELECT ... id=1  | 10000  | 1.0      | ← 精确路由
-- | SELECT ... uid=? | 5000   | 4.0      | ← 全分片扫描
-- +------------------+--------+----------+
```

# 6. Buffer Pool 调优

```yaml
server:
  bufferPool:
    type: direct
    pageSize: 4194304      # 4MB 页大小
    chunkSize: 4096        # 4KB 块大小
    maxPages: 256          # 最大页数（256 × 4MB = 1GB）
```

**Buffer Pool 用途**：Mycat2 内部数据缓冲（SQL 解析、结果集临时存储、协议编解码）。

**建议**：`pageSize × maxPages` ≈ `-XX:MaxDirectMemorySize`

# 7. 压测基准

```
测试环境：Mycat2 4C8G，后端 4 个 MySQL 5.7

| 场景                      | QPS    | AVG RT | P99 RT |
|--------------------------|--------|--------|--------|
| 精确分片 (WHERE id=?)    | 25000  | 3ms    | 8ms    |
| IN 查询（3 个 ID）        | 12000  | 8ms    | 20ms   |
| 全分片 COUNT(*)          | 5000   | 20ms   | 50ms   |
| 全分片 ORDER BY LIMIT    | 3000   | 35ms   | 80ms   |
| 读写分离 SELECT          | 20000  | 4ms    | 10ms   |
| XA 事务（2 分片）         | 3000   | 30ms   | 60ms   |
```

# 8. 常见性能问题

| 问题 | 原因 | 优化 |
|------|------|------|
| 全分片扫描过多 | WHERE 缺少分片键 | 优化 SQL，添加分片键 |
| 后端连接池满 | minCon/maxCon 不足 | 按公式计算增大 |
| 排序/分页慢 | 深度分页 + 跨分片排序 | 使用游标分页替代 OFFSET |
| GC 频繁 | 堆内存不足 | 增大 -Xmx，使用 G1GC |
| Reactor 线程满 | 线程数不足 | 增大 reactorNumber |

# 9. 快速调优清单

```
□ JVM: -Xms2g -Xmx4g -XX:+UseG1GC -XX:MaxDirectMemorySize=1g
□ Reactor: 线程数 = CPU 核数 × 2
□ 连接池: 后端连接数 = 前端 QPS × RT / 1000 / 分片数
□ Buffer: pageSize × maxPages ≈ MaxDirectMemorySize
□ SQL: 确保 WHERE 包含分片键
□ 监控: Prometheus + Grafana 实时观察
□ 日志: 生产环境设置 WARN 级别
```
