# 1. 心跳检测机制

**心跳（Heartbeat）是 Mycat 感知后端 MySQL 存活与主从状态的唯一常规手段**：中间件与后端之间没有控制面协议，连接池里躺着的空闲连接无法证明数据库还活着，只有周期性发一条探测 SQL 才能及时把故障节点摘除、把新主识别出来。心跳质量直接决定故障发现时间（RTO 的第一段）与读写分离的正确性。

Mycat2 通过定期心跳检测后端 MySQL 节点的可用性：

```text
Mycat2
  │
  │ 每 10s 发送心跳
  │
  ├──→ MySQL Master  ── SELECT 1 → OK ✓
  ├──→ MySQL Slave1  ── SELECT 1 → OK ✓
  └──→ MySQL Slave2  ── 超时 ✗ → 标记为不可用，自动摘除
```

**心跳与业务流量的隔离设计**值得注意：心跳使用**独立的探测连接**，不占用业务连接池，也不排队等业务高峰——否则数据库被业务打满时心跳超时，会造成"忙被误判为死"的错误摘除。反过来这也意味着：数据库活着但业务连接池耗尽时，心跳显示 ONLINE，请求依然失败；**心跳探测的是"实例可达 + 可响应"，不是"业务可用"**，监控上两者要分开看。

Mycat 1.6.x 的心跳配置在 schema.xml 的 `<dataHost>` 里（`<heartbeat>select user()</heartbeat>` + `heartbeat-frequency`），2.x 移到数据源级 JSON 配置，语义一致、粒度更细——每个物理数据源独立探测，而不是每个 dataHost 共享一条心跳线。

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

**参数取值的权衡**集中在两个：

- **心跳周期与超时**：周期越短故障发现越快，但对后端的探测压力和网络抖动敏感度上升。`heartbeatTimeout=1000ms` 意味着跨机房部署（RTT > 几百 ms 或偶发抖动）会频繁误报，同城多可用区常放宽到 2000-3000ms；`maxRetry=3` 起平滑作用——**判定下线需要连续 maxRetry 次失败，最坏故障发现时间 ≈ 周期 × maxRetry + 超时**，按默认值约 30 秒出头，这是"MySQL 挂了 Mycat 大约半分钟才摘"的来源。
- **heartbeatSQL 选什么**：`SELECT 1` 最轻，但只证明 mysqld 网络栈和解析层活着；1.6.x 默认 `select user()` 会触碰 mysql.user 表，能多探测一层权限/存储引擎异常。若要探测"InnoDB 可写"，可以在主库心跳加 `SET SESSION TRANSACTION READ ONLY` 之类的会话级检查或写入哨兵表——但**写心跳要慎之又慎**（对延迟敏感、binlog 膨胀、半同步阻塞），通常留给专门的探活脚本，中间件心跳保持只读。

# 3. 故障检测与摘除

```text
状态机：
  ONLINE ──心跳失败 (连续 3 次)──→ OFFLINE
  OFFLINE ──心跳恢复─────────────→ ONLINE
  OFFLINE ──超过 10 分钟─────────→ PERMANENTLY_OFFLINE
```

摘除动作的实质是**更新路由/负载均衡视图**：OFFLINE 节点不再参与读权重分配；如果是主库节点，则触发主从切换判定（见第 4 节）。恢复同理——心跳连续成功后节点自动回 ONLINE 并重新接入，无需人工，这一点对 Slave 短暂重启、网络闪断的自愈很重要。

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

> **注意**：心跳摘除只影响**新请求的分发**，不能拯救**已在故障连接上执行的请求**——正在等结果的前端会话会 hang 到读超时才报错。应用侧的 connectTimeout/socketTimeout 与重试策略仍然必须配置，"Mycat 会自动摘坏节点"不等于"业务无感知"。另外连接池里存量的空闲连接在节点 OFFLINE 时会被批量销毁重建，这一下抖动在监控上表现为错误尖刺 + 后端重连风暴，属于预期行为。

# 4. 主从切换判定

心跳的另一职责是**发现新主**。当配置了 `switchType` 时，Mycat 会在"心跳视角"里做角色一致性检查：配置标记的 Master 心跳失败、而某台 Slave 心跳正常且其 `SHOW SLAVE STATUS` 显示复制线程已停/源不可达，即判定主库故障，进入切换流程。

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

**两类判定的分工**要分清：

- **延迟判定（读路由）**：`slaveThreshold` 控制的是"这台 Slave 还配不配读"——`Seconds_Behind_Master > 60` 的从库被临时踢出读池，追平后自动回来。大事务、DDL、备份都会把延迟瞬间拉高，这是"读请求偶发跳主库"的正常来源。`Seconds_Behind_Master` 本身有坑：它衡量的是 IO 线程与 SQL 线程的时间差，**主从结构异常时可能显示 0 而实际数据不同步**（如 SQL 线程报错停下后该值长期 NULL/0），严格场景应配合 `Master_Log_File/Exec_Master_Log_Pos` 与 Relay_Log 位点差判断。
- **切换判定（写路由）**：`minSwitchTimeInterval` 是切换防抖——两次自动切换之间至少间隔 30s，防止网络分区反复横跳导致的**乒乓切换**。

> **设计争议**：Mycat 要不要自己做主从切换？Mycat 1.6.x 内置的 switch（选延迟最小的 Slave 提升、对其余节点执行 `CHANGE MASTER TO`）本质是一个简化版 MHA，它**无法可靠回答"旧主是真的死了还是只是我看不到它"**——这正是脑裂温床。业界主流答案是：**HA 决策交给专职组件**（MHA/Orchestrator/Patroni 类，或云上 RDS 的高可用体系），Mycat 只做"跟随者"：通过心跳发现新主并切流量。Mycat2 的 AUTO 切换适合无外部 HA 体系的小规模场景，生产集群建议 `switchType` 设为被动模式，让 Mycat 只探测、只跟随，不做提升。

**连接失效处理**是切换的最后一公里：新主确认后，Mycat 需要（1）销毁指向旧主的连接池；（2）对使用旧主的进行中事务，写请求可能"提交状态未知"——Mycat 会向该前端会话返回错误而非假装成功，**事务的最终一致性由应用补偿逻辑兜底**（幂等重试、对账），这一点在异步复制未追平丢事务时同样成立（见第 9 篇分布式事务）。

# 5. 与 Druid 连接池心跳的差异

应用侧普遍用 Druid/HikariCP 管理直连池，二者与 Mycat 心跳机制的对比是高频困惑点，本质区别在于**角色**：Druid 是"客户端的连接质量保障"，Mycat 心跳是"服务拓扑的成员管理"。

| 维度 | Druid testOnBorrow/validationQuery | Mycat 心跳 |
|------|-----------------------------------|-----------|
| 探测对象 | 池内**一条连接**是否可用 | **整个实例**是否存活、是否主/从 |
| 探测方式 | 借出连接前执行 validationQuery | 独立连接周期执行 heartbeatSQL |
| 失败后果 | 丢弃该连接、重建 | 摘除节点、调整读权重、触发切换判定 |
| 影响延迟 | testOnBorrow 每次借出都加一次 RTT（常见性能杀手） | 与业务请求无关 |
| 拓扑感知 | 无（不关心这个库是主是从） | 有（主从角色、延迟阈值） |
| 恢复 | 连接重建即恢复 | 节点级连续心跳成功后回池 |

> **行业实践**：用了 Mycat 之后，应用侧连接池应**弱化探活职责**——`testOnBorrow=false`，改用 `testWhileIdle=true` + `timeBetweenEvictionRunsMillis`，把"后端死活"的判断留给 Mycat 心跳（它离数据库更近、有全局视图），应用池只管"自己这条 TCP 连接没烂"。两层都做重探测，等于把 validation 流量放大 N 倍打在 MySQL 上，还互相干扰故障判定。

# 6. Prometheus 集成

## 6.1 暴露 Metrics

```yaml
# mycat.yml
server:
  prometheus:
    enabled: true
    port: 9090
    path: /metrics
```

## 6.2 内置指标

| 指标 | 说明 | 类型 |
|------|------|------|
| `mycat_frontend_connections` | 前端连接数 | Gauge |
| `mycat_backend_connections` | 后端连接数 | Gauge |
| `mycat_qps` | 每秒查询数 | Counter |
| `mycat_rt_seconds` | 查询响应时间 | Histogram |
| `mycat_error_total` | 错误总数 | Counter |
| `mycat_datanode_status` | 数据节点状态(0=离线,1=在线) | Gauge |

## 6.3 PromQL 查询

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

心跳相关最该盯的是 `mycat_datanode_status` 的**翻转频率**而不仅是当前值：节点 ONLINE/OFFLINE 反复横跳（flapping）通常意味着网络抖动、超时阈值过紧或数据库处于临界负载，比一次性 OFFLINE 更需要人工介入。

# 7. Grafana 大盘建议

```text
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

# 8. 管理端口监控命令

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

`showHeartbeatStatus` 的 `delay` 列就是 slaveThreshold 判定使用的实时复制延迟，排"读到了旧数据"类问题时第一时间看这里（详见第 20 篇排错）。Mycat 1.6.x 的对应命令是 `show @@heartbeat`、`show @@datasource`，运维脚本迁移 2.x 时需要同步替换。

# 9. 告警建议

| 告警项 | 阈值 | 严重程度 |
|--------|------|---------|
| 数据节点 OFFLINE | 任意 | 紧急 |
| 前端连接数 > 80% | maxCon × 0.8 | 严重 |
| P99 响应时间 > 1s | — | 警告 |
| 错误率 > 1% | — | 警告 |
| Slave 延迟 > 60s | — | 警告 |
| QPS 突然下降 > 50% | — | 紧急 |

告警分级的一条经验：**心跳类告警按"节点×时间"去重**——一次网络分区会让一台 Mycat 对所有后端同时报 OFFLINE，多实例部署时又乘以实例数，不去重就是告警风暴。正确做法是在 Prometheus 侧按数据源聚合、加 `for: 2m` 持续性条件，只报"这个数据源在多数 Mycat 实例视角下都不可达"。

# 10. 小结

| 要点 | 结论 |
|------|------|
| 心跳的本质 | 独立探测连接周期执行 heartbeatSQL，判定"实例可达"，最坏发现时间 = 周期 × maxRetry + 超时 |
| 摘除与恢复 | 连续 maxRetry 失败摘除、恢复后自动回池；只影响新请求分发，存量请求靠应用超时/重试兜底 |
| 延迟与切换 | slaveThreshold 管读路由资格，minSwitchTimeInterval 防乒乓；Seconds_Behind_Master 有假 0 陷阱 |
| 切换归属争议 | 生产建议 Mycat 只探测只跟随，提升决策交给 MHA/Orchestrator/云 RDS，避免脑裂 |
| 与 Druid 差异 | Druid 管连接质量、Mycat 心跳管成员拓扑；用了 Mycat 后应用池应关 testOnBorrow 只留 testWhileIdle |

下一篇（14 高可用部署）把视角从"后端节点探活"拉到"Mycat 自身与整个链路的部署拓扑"。
