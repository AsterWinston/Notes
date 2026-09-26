
# 1. 性能瓶颈分析

```
Seata 分布式事务的性能开销主要来自：
┌────────────────────────────────────────────┐
│ 1. undo_log 写入 (AT) — 额外 INSERT 操作     │
│ 2. 前置/后置镜像查询 — BEFORE/AFTER SELECT   │
│ 3. TC 通信 — 注册分支/提交/回滚 RPC           │
│ 4. 全局锁竞争 — lock_table 查询和等待         │
│ 5. undo_log 表膨胀 — 磁盘和索引开销            │
└────────────────────────────────────────────┘
```

# 2. AT 模式 SQL 开销分析

一次 AT 模式 UPDATE 实际执行的 SQL：

```sql
-- 业务 SQL
UPDATE account SET balance = balance - 50 WHERE user_id = 1;

-- Seata 额外 SQL：
-- 1. 前置镜像查询
SELECT * FROM account WHERE user_id = 1 FOR UPDATE;

-- 2. 执行业务 SQL（同上）

-- 3. 后置镜像查询
SELECT * FROM account WHERE user_id = 1;

-- 4. undo_log 写入
INSERT INTO undo_log (...) VALUES (...);

-- 5. 全局锁注册到 TC（RPC）
-- 6. Phase 2: TC 通知提交 → undo_log DELETE

-- 总计：1 → 4 条 SQL + 2 次 RPC
```

# 3. 批量优化

```java
// ❌ 逐条插入（每条都生成 undo_log）
@GlobalTransactional
public void batchInsert(List<Order> orders) {
    for (Order order : orders) {
        orderMapper.insert(order);  // N 次 undo_log
    }
}

// ✓ 批量插入（减少 undo_log 次数）
@GlobalTransactional
public void batchInsert(List<Order> orders) {
    orderMapper.batchInsert(orders);  // 1 次 undo_log（整个批次的前后镜像）
}
```

**优化效果**：100 条记录的批量插入，SQL 数从 400+ 降至 ~8。

# 4. undo_log 表优化

## 4.1 定期清理

```sql
-- 创建定时任务清理已提交的 undo_log
DELETE FROM undo_log
WHERE log_status = 1
  AND log_modified < DATE_SUB(NOW(), INTERVAL 7 DAY)
LIMIT 1000;

-- 添加索引加速清理
CREATE INDEX idx_log_status_modified
  ON undo_log(log_status, log_modified);
```

## 4.2 表分区

```sql
-- 按月分区 undo_log
ALTER TABLE undo_log PARTITION BY RANGE (TO_DAYS(log_created)) (
  PARTITION p202401 VALUES LESS THAN (TO_DAYS('2024-02-01')),
  PARTITION p202402 VALUES LESS THAN (TO_DAYS('2024-03-01')),
  PARTITION p202403 VALUES LESS THAN (TO_DAYS('2024-04-01')),
  PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

## 4.3 归档策略

```yaml
seata:
  server:
    undo:
      log-save-days: 7          # 保留天数
      log-delete-period: 86400000 # 清理周期(ms) = 24h
```

# 5. TC 通信优化

```yaml
seata:
  client:
    rm:
      # 异步提交（AT 模式 Phase 2 异步执行）
      async-commit-buffer-limit: 10000

    tm:
      # 减少不必要的重试快速失败
      commit-retry-count: 3
      rollback-retry-count: 3

server:
  # TC 端性能参数
  max-commit-retry-timeout: 60000
  max-rollback-retry-timeout: 60000

# RPC 线程池
transport:
  type: TCP
  server: NIO
  heartbeat: true
  thread-factory:
    boss-thread-prefix: NettyBoss
    worker-thread-prefix: NettyServerNIOWorker
    boss-thread-size: 1
    worker-thread-size: default
```

# 6. 连接池配置

```yaml
seata:
  store:
    db:
      datasource: druid
      min-conn: 10
      max-conn: 100
      max-wait: 5000

# 业务数据库连接池
spring:
  datasource:
    hikari:
      minimum-idle: 10
      maximum-pool-size: 50
      connection-timeout: 3000
```

**AT 模式连接池建议**：连接池大小 = 普通模式的 1.5~2 倍，因为 AT 模式的额外 SQL 会占用更多连接时间。

# 7. GC 调优

```bash
# TC Server JVM 参数
java -Xms2g -Xmx4g \
     -Xmn1g \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -XX:InitiatingHeapOccupancyPercent=45 \
     -jar seata-server.jar
```

# 8. 性能基准参考

```
测试环境：4C8G，MySQL 5.7，单表 10w 行，100 并发

| 模式   | TPS    | AVG RT | 额外SQL/RPC |
|--------|--------|--------|------------|
| 无事务  | 5000   | 20ms   | 0          |
| AT     | 3800   | 26ms   | +3 SQL + 2 RPC |
| TCC    | 4500   | 22ms   | +2 RPC     |
| Saga   | 4800   | 20ms   | +1 RPC     |
| XA     | 2200   | 45ms   | XA 协议开销  |

AT 模式性能损耗约 24%，在可接受范围内。
```

# 9. 最佳优化策略

| 优先级 | 优化项 | 预期收益 |
|--------|--------|---------|
| 1 | 批量操作代替逐条操作 | 30-50% |
| 2 | undo_log 定期清理 + 索引 | 10-20% |
| 3 | AT 模式异步提交 | 5-10% |
| 4 | 连接池扩容 | 5-15% |
| 5 | TC 集群水平扩展 | 线性提升 |
