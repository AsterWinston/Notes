# 1. 性能开销结构

一次全局事务相对本地事务的额外成本，可拆为五项：

```text
Seata 分布式事务的性能开销主要来自：
┌────────────────────────────────────────────┐
│ 1. undo_log 写入 (AT) — 额外 INSERT 操作     │
│ 2. 前置/后置镜像查询 — BEFORE/AFTER SELECT   │
│ 3. TC 通信 — 注册分支/提交/回滚 RPC           │
│ 4. 全局锁竞争 — lock_table 查询和等待         │
│ 5. undo_log 表膨胀 — 磁盘和索引开销            │
└────────────────────────────────────────────┘
```

其中 1、2、4 为 AT 特有，3 是所有模式共有的 TC 侧开销，5 是随时间累积的存储侧隐性开销。

> **注意**：调优顺序是 **先业务链路（分支数、批量、热点）→ 再存储（undo_log/lock_table/TC store）→ 再集群（TC 分片、连接池）→ 最后 JVM**。典型反例：TC 扩到 6 节点吞吐纹丝不动——瓶颈在每个分支多查两次镜像的 SQL 上。

# 2. AT 模式 SQL 开销分析

一次 AT 模式 UPDATE 实际执行的 SQL：

```sql
-- 业务 SQL
UPDATE account SET balance = balance - 50 WHERE user_id = 1;

-- Seata 代理层额外动作（同连接、同本地事务内）：
-- 1. 前置镜像查询
SELECT * FROM account WHERE user_id = 1 FOR UPDATE;
-- 2. 执行业务 SQL
-- 3. 后置镜像查询
SELECT * FROM account WHERE id IN (...);
-- 4. undo_log 写入（序列化前后镜像）
INSERT INTO undo_log (...) VALUES (...);

-- TC 侧（RPC，占 RT 不占业务 SQL）：
-- 5. 分支注册 + 全局锁检查/加锁（合并为一次 RPC）
-- 6. Phase 2: TC 通知分支提交 → 异步 DELETE undo_log

-- 总计：1 → 4 条 SQL + 2 次 RPC
```

量化口径：单分支 UPDATE 的业务 SQL 从 1 条变成 4 条，本地事务时长放大约 2-3 倍；两次 TC RPC 同城各占 1-3ms——这就是 AT 约 20-30% TPS 折损的来源。优化方向一一对应：**减少 SQL 条数（批量）、降低每条成本（索引与主键规范）、隐藏 RPC 延迟（连接复用与异步化）**。

# 3. 批量与 SQL 拦截器开销

```java
// ❌ 错误示范：逐条插入（每条都触发前后镜像 + undo_log）
@GlobalTransactional
public void batchInsert(List<Order> orders) {
    for (Order order : orders) {
        orderMapper.insert(order);  // N 次 undo_log
    }
}

// ✓ 正确做法：真批量（一次拦截，1 条 undo_log 装下整批镜像）
@GlobalTransactional
public void batchInsert(List<Order> orders) {
    orderMapper.batchInsert(orders);  // 单条 multi-values INSERT
}
```

**优化效果**：100 条记录的批量插入，SQL 数从 400+ 降至 ~8，且省掉 N 次全局锁 RPC 往返。

SQL 拦截器本身的开销常被低估。DataSourceProxy 的完整链路是：**SQL 解析（构建 AST）→ 表元数据获取 → 前置镜像 SELECT → 业务执行 → 后置镜像 SELECT → 镜像序列化 → undo_log INSERT**。解析与元数据都有缓存（TableMetaCache），但解析在热点路径上逐条执行；镜像查询在**没有合适索引时会退化为主键全表扫描**（复杂 WHERE 无法下推时按主键回表收集行）。

**设计考量**：

- **表必须有显式主键且尽量单列**：Seata AT 硬性要求主键+唯一索引之一；前后镜像以主键定位行，复合主键/无索引大表会让镜像查询成本失控。
- **控制单语句影响行数**：一条 UPDATE 命中 10 万行会生成 10 万行镜像（undo_log 单条可达几十 MB），镜像查询、序列化、回滚三处同时爆炸——大批量变更要分片为多条语句/多个本地事务分批提交。
- **拦截器白名单**：纯查询服务不需要事务代理，别把 `DataSourceProxy` 无脑套在只读数据源上，SELECT 同样有解析成本。

# 4. 全局锁竞争热点治理

AT 的写隔离靠 TC 的 lock_table 全局锁，**热点行是全链路吞吐的天花板**。秒杀场景下同一库存行的全局锁排队，是 AT 压测 TPS 曲线断崖的第一嫌疑。

```text
热点行冲突时间线（库存行 product:999 被 100 并发争抢）：

分支1 ──检查锁──→ 获取成功 ──持有──→ 全局提交释放
分支2 ──检查锁──→ 冲突 → 重试(lockRetryTimes) → 获取 → ...
分支3 ──检查锁──→ 冲突 → 重试超限 → LockKeyConflictException
结果：全局锁串行化窗口 ≈ 单事务时长 × 并发数
```

治理手段按优先级：**业务分桶（治本）**——热点行拆成 N 个子行（库存分 16 桶、账户分子账户），冲突面缩小 N 倍；**缩短持锁时间**——全局事务只圈"扣减"动作，查询、校验、发消息移出边界；**调重试参数**：

```yaml
seata:
  client:
    rm:
      lock:
        retry-interval: 10        # 每次重试间隔(ms)
        retry-times: 30           # 热点高建议降并发而不是盲目调大
        retry-policy: branchRollback  # 冲突时快速失败回滚全局 vs 等待
```

热点行链路的**终解是 TCC 业务预留**（冻结字段原子 UPDATE），不是把 AT 调到极限。

> **行业实践**：出现大量 `LockKeyConflictException` 或 lock retry 计数飙升时，先查"全局事务是否圈大了"（事务边界），再查"是否真热点"（业务分桶），最后才调重试参数——调大 retry-times 只是把快速失败变成排队超时，P99 只会更差。

# 5. 存储侧优化：undo_log 与 TC store

## 5.1 undo_log 表索引与清理

undo_log 是"写多删也多"的高频表，Phase 2 的 DELETE 与清理任务都靠索引吃饭：

```sql
-- 官方表自带 uk_xid_branch_id（回滚定位用），清理任务需补时间索引：
CREATE INDEX idx_log_modified ON undo_log(log_modified);

-- 兜底清理（防漏删堆积；正常提交路径 Phase 2 已异步删除）
DELETE FROM undo_log
WHERE log_status = 1
  AND log_modified < DATE_SUB(NOW(), INTERVAL 7 DAY)
LIMIT 1000;

-- 超大规模按月分区，清理变成 DROP PARTITION，瞬时完成且无 binlog 洪峰
ALTER TABLE undo_log PARTITION BY RANGE (TO_DAYS(log_created)) (
  PARTITION p202610 VALUES LESS THAN (TO_DAYS('2026-11-01')),
  PARTITION p202611 VALUES LESS THAN (TO_DAYS('2026-12-01')),
  PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

```yaml
# TC 端自动清理任务（application.yml 的 store 段）
seata:
  server:
    store:
      undo:
        log-save-days: 7              # 保留天数
        log-delete-period: 86400000   # 清理周期(ms) = 24h
```

> **注意**：undo_log 膨胀的真实杀伤路径是**索引深度与 DELETE 锁**，而不是磁盘。单表过亿后 Phase 2 的 DELETE 与恢复扫描都会拖慢业务库，堆积还会放大备份与主从延迟。"定期清理 + 时间索引"是 AT 上线 checklist 项。

## 5.2 TC store 模式与分片

TC 的状态存储是集群吞吐的第二瓶颈。db 模式下 global_table/branch_table/lock_table 的读写都压在同一个 MySQL 上：

| store 模式 | 吞吐特征 | 适用 |
|-----------|---------|------|
| file | 单节点本地磁盘，不可横向扩 | 开发/测试 |
| db | 依赖共享 MySQL，单库极限≈万级 TPS | 中小规模生产 |
| redis | 内存级读写，注意持久化 | 高吞吐 |
| raft | TC 自管复制状态机，免外部存储 | 2.x 主推，中大规模 |

**TC 集群与分片要点**：db/redis 模式下多 TC 节点无状态对等部署，全局事务按 xid 哈希分片，扩 TC 提升协调吞吐（前提是存储跟得上）。raft 模式奇数节点（3/5）组集群，`apply-batch`、`disruptor-buffer-size` 是吞吐旋钮：

```yaml
seata:
  server:
    raft:
      group: default
      server-addr: 10.0.1.10:7091,10.0.1.11:7091,10.0.1.12:7091
      apply-batch: 32                 # Raft 批量应用条数
      max-replicator-inflight-msgs: 256
      disruptor-buffer-size: 16384    # 事务处理环形队列容量
```

> **行业实践**：TC 数量不是越多越好——单全局事务由一个 TC 节点全程协调，扩节点提升的是**并发会话总量**；热点在大事务上时扩 TC 无用，要减分支数、缩事务时长。容量翻倍优先加 store 分片，其次加 TC 节点。

# 6. 异步提交与 TC 通信优化

```yaml
seata:
  client:
    rm:
      async-commit-buffer-limit: 10000   # 异步分支提交队列容量
      report-retry-count: 5
    tm:
      commit-retry-count: 30
      rollback-retry-count: 30
  transport:
    heartbeat: true
```

异步提交是 AT 吞吐的关键机制：Phase 2 的 undo_log DELETE 与分支提交回执不阻塞全局事务返回——TM 拿到 TC 的"已受理"即成功；队列打满时提交延迟陡增，瓶颈通常在 RM 端 DELETE 能力。

通信层抓手在**连接复用**：客户端到 TC 是 Netty 长连接多路复用，`boss-thread-size: 1` 配 NIO worker（CPU 核数 × 2）足够；RPC 占比高时先查网络 RTT（是否跨 AZ），再查 TC 线程池排队。

# 7. 超时参数：三层预算

超时配置决定故障止损速度，设错比不设更糟。三层关系必须**由内向外递增**：`innodb_lock_wait_timeout` > 单条 SQL 最坏耗时，分支本地事务 > 业务 SQL，全局超时 > 单分支最坏耗时：

```yaml
seata:
  client:
    tm:
      default-global-transaction-timeout: 30000   # 全局超时(ms)：故障兜底
```

注解级覆盖：`@GlobalTransactional(timeoutMills = 10000)`。

**设计考量**：

- 全局超时不是性能参数而是**恢复触发器**：超时后 TC 驱动回滚——设太长故障悬挂占锁，设太短正常慢请求被误回滚，按 P99 事务时长的 2-3 倍设定。
- 分支超时的隐形来源是 `max-commit-retry-timeout / max-rollback-retry-timeout`（TC 端，超时进入悬挂检测），出现大量 `TimeoutRollbacking` 先看这两个而不是业务代码。
- 重试参数与超时预算联动：重试耗尽进入人工兜底，监控盯 `AsyncCommitting/TimeoutRollbacking` 状态存量，这是"最终一致性欠账"的直接读数。

# 8. 连接池与 JVM

```yaml
# TC Server 的存储连接池
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

**AT 模式连接池建议**：连接池大小 = 普通模式的 1.5~2 倍——前后镜像与 undo_log 写入把单分支**连接占用时长**拉长，池子小了请求在排队等连接而不是等锁，表现为 `connection-timeout` 而不是锁冲突。XA 更苛刻：prepare 后连接不能归还，连接需求≈并发分支数。

```bash
# TC Server JVM 参数
java -Xms2g -Xmx4g \
     -Xmn1g \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -XX:InitiatingHeapOccupancyPercent=45 \
     -jar seata-server.jar
```

TC 是"高吞吐小对象"型负载（事务会话、RPC buffer），GC 停顿直接映射为全局提交 RT 的毛刺；G1 默认够用，Full GC 频繁先查 `disruptor-buffer-size` 是否过小导致积压、超时未清理事务是否堆积。

# 9. 压测基线与监控指标

调优必须有仪表盘。最小监控集：

| 指标 | 来源 | 告警参考 |
|------|------|---------|
| 全局事务提交时间 P50/P99 | TC 统计 `/metrics` 或埋点 | P99 > 全局超时的 1/3 |
| 事务重试率 | TC 日志与状态计数 | > 1% 持续 5min |
| 状态分布 | global_table 按 status 聚合 | Committing/Rollbacking 存量超阈值 |
| 全局锁冲突次数/耗时 | RM 日志 LockConflict | 突增即热点告警 |
| async-commit 队列水位 | TC buffer 指标 | 接近 buffer-limit |
| undo_log 行数与最大行长度 | 业务库巡检 | 行数周环比 > 20% 增长 |

**压测基线**（模拟口径：4C8G，MySQL 5.7，单表 10w 行，100 并发，无热点）：

```text
| 模式   | TPS    | AVG RT | 额外SQL/RPC     |
|--------|--------|--------|-----------------|
| 无事务 | 5000   | 20ms   | 0               |
| AT     | 3800   | 26ms   | +3 SQL + 2 RPC  |
| TCC    | 4500   | 22ms   | +2 RPC          |
| Saga   | 4800   | 20ms   | +1 RPC          |
| XA     | 2200   | 45ms   | XA 协议 + 锁窗口 |
```

AT 损耗约 24%，可接受。压测纪律：**每个变量单独打轴**——并发数、分支数（1/3/6）、热点与否、事务时长分别跑曲线。

# 10. 容量估算

估算公式（AT 单 TC 集群口径）：

```text
单全局事务成本 ≈ 分支数 × (本地SQL 4~6 条 + 镜像 2~3 条)
              + 分支数 × 1 次 branchRegister RPC
              + 1 次 globalCommit RPC

TC 容量上限 ≈ min(单节点 RPC 处理能力 × 节点数, store 层 TPS)
业务库容量 ≈ 峰值 TPS × 平均分支数 × 镜像/undo SQL 放大系数(3~4)
```

算例：目标峰值 3000 TPS、平均 3 分支的 AT 链路——业务库需承受约 3000×3×4=36000 SQL/s，undo_log 写入 ≈ 9000/s，TC 需处理 ≈ 9000 branchRegister + 3000 globalCommit RPC/s。结论：业务库先做分库/读写分离评估；TC 侧 2~3 节点 db 或 raft 集群起步。**估算的价值不在精确，在让"扩谁"有数据可依。**

# 11. 优化策略优先级

| 优先级 | 优化项 | 预期收益 |
|--------|--------|---------|
| 1 | 批量操作代替逐条操作 | 30-50% |
| 2 | 缩小全局事务边界（只圈写操作） | 20-40% |
| 3 | 热点分桶/业务拆分 | 冲突场景 5-10 倍 |
| 4 | undo_log 定期清理 + 索引 | 10-20% |
| 5 | AT 异步提交 + 队列监控 | 5-10% |
| 6 | 连接池扩容（AT 1.5-2 倍） | 5-15% |
| 7 | TC 集群水平扩展 / store 分片 | 线性提升（受限于存储） |

# 12. 小结

Seata 性能优化沿开销模型展开：分支侧省 SQL（批量合并、镜像查询靠主键、事务边界收缩），协调侧省 RPC（长连接复用、注册与锁检查合并、异步提交），存储侧防累积（undo_log 索引+清理+分区、TC store 选型与分片），运行侧守预算（三层超时由内向外递增、连接池按模式放大）。顺序与证据比参数更重要：**先热点与事务边界，再存储，再集群，最后 JVM**，每一步以监控指标验证。调优的终局目标不是把 AT 调到接近本地事务——那说明业务本不需要分布式事务——而是让模式与热点设计匹配：能用边界收缩解决的不上参数，能用分桶解决的不上集群，扛不住的热点链路换 TCC 或重构聚合根。
