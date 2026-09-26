
# 1. 心跳检测机制

Mycat2 通过定期心跳检测后端 MySQL 节点的可用性：

```
Mycat2
  │
  │ 每 10s 发送心跳
  │
  ├──→ MySQL Master  ── SELECT 1 → OK ✓
  ├──→ MySQL Slave1  ── SELECT 1 → OK ✓
  └──→ MySQL Slave2  ── 超时 ✗ → 标记为不可用，自动摘除
```

# 2. 心跳配置

```json
// datasources.json 中的心跳配置
{
  "datasources": {
    "ds1": {
      "heartbeat": {
        "heartbeatSQL": "SELECT 1",
        "maxRetry": 3,
        "minSwitchTimeInterval": 30000,
        "heartbeatTimeout": 1000,
        "slaveThreshold": 60
      }
    }
  }
}
```

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| `heartbeatSQL` | 心跳检测 SQL | `SELECT 1` |
| `maxRetry` | 失败重试次数 | 3 |
| `minSwitchTimeInterval` | 最小切换间隔(ms) | 30000 |
| `heartbeatTimeout` | 心跳超时(ms) | 1000 |
| `slaveThreshold` | Slave 延迟阈值(秒) | 60 |

# 3. 故障检测与摘除

```
状态机：
  ONLINE ──心跳失败 (连续 3 次)──→ OFFLINE
  OFFLINE ──心跳恢复─────────────→ ONLINE
  OFFLINE ──超过 10 分钟─────────→ PERMANENTLY_OFFLINE
```

```java
// 伪代码：心跳检测逻辑
public class HeartbeatDetector {

    public void check() {
        for (DataSource ds : allDataSources) {
            try {
                long start = System.currentTimeMillis();
                ds.executeQuery(heartbeatSQL);
                long elapsed = System.currentTimeMillis() - start;

                ds.setHealthy(true);
                ds.setResponseTime(elapsed);
                ds.resetFailCount();

            } catch (Exception e) {
                ds.incrementFailCount();
                log.warn("heartbeat fail: {}, count: {}",
                    ds.getName(), ds.getFailCount());

                if (ds.getFailCount() >= maxRetry) {
                    // 摘除节点
                    ds.setHealthy(false);
                    // 更新路由表，停止向该节点发送请求
                    router.removeDataNode(ds);
                    // 发送告警
                    alertService.alert("DataNode OFFLINE: " + ds.getName());
                }
            }
        }
    }
}
```

# 4. 主从延迟检测

```sql
-- Mycat2 内部执行
SHOW SLAVE STATUS;

-- 解析 Seconds_Behind_Master
-- 如果 > slaveThreshold → 标记 Slave 不可用
```

```json
{
  "heartbeat": {
    "masterHeartbeatSQL": "SHOW MASTER STATUS",
    "slaveHeartbeatSQL": "SHOW SLAVE STATUS",
    "slaveThreshold": 60
  }
}
```

# 5. Prometheus 集成

## 5.1 暴露 Metrics

```yaml
# mycat.yml
server:
  prometheus:
    enabled: true
    port: 9090
    path: /metrics
```

## 5.2 内置指标

| 指标 | 说明 | 类型 |
|------|------|------|
| `mycat_frontend_connections` | 前端连接数 | Gauge |
| `mycat_backend_connections` | 后端连接数 | Gauge |
| `mycat_qps` | 每秒查询数 | Counter |
| `mycat_rt_seconds` | 查询响应时间 | Histogram |
| `mycat_error_total` | 错误总数 | Counter |
| `mycat_datanode_status` | 数据节点状态(0=离线,1=在线) | Gauge |

## 5.3 PromQL 查询

```prometheus
# QPS
rate(mycat_qps[1m])

# P99 响应时间
histogram_quantile(0.99, rate(mycat_rt_seconds_bucket[1m]))

# 后端连接利用率
mycat_backend_connections / mycat_backend_max_connections

# 数据节点在线率
avg(mycat_datanode_status)
```

# 6. Grafana 大盘建议

```
推荐面板：
┌─────────────────────────────────────┐
│ 1. QPS + 错误率（折线图）            │
│ 2. P50/P99 响应时间（折线图）         │
│ 3. 前后端连接数（折线图）              │
│ 4. 数据节点状态（状态面板）            │
│ 5. SQL 路由分布（分片命中率）          │
│ 6. Heartbeat 延迟（热力图）           │
└─────────────────────────────────────┘
```

# 7. 管理端口监控命令

```sql
-- 查看前端连接
/*!mycat:showFrontends*/

-- 查看后端连接
/*!mycat:showBackends*/

-- 查看 Reactor 线程
/*!mycat:showReactors*/

-- 查看心跳状态
/*!mycat:showHeartbeatStatus*/
-- +----------+---------+--------+--------+
-- | name     | status  | rtime  | delay  |
-- +----------+---------+--------+--------+
-- | master0  | ONLINE  | 2ms    | —      |
-- | slave1   | ONLINE  | 3ms    | 5s     |
-- | slave2   | OFFLINE | —      | —      |
-- +----------+---------+--------+--------+

-- 查看慢 SQL
/*!mycat:showSlowSQL*/
```

# 8. 告警建议

| 告警项 | 阈值 | 严重程度 |
|--------|------|---------|
| 数据节点 OFFLINE | 任意 | 紧急 |
| 前端连接数 > 80% | maxCon × 0.8 | 严重 |
| P99 响应时间 > 1s | — | 警告 |
| 错误率 > 1% | — | 警告 |
| Slave 延迟 > 60s | — | 警告 |
| QPS 突然下降 > 50% | — | 紧急 |
