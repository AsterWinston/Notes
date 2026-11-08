读写分离是最早在生产普及的数据库扩展手段：写走主库、读走从库，用复制把读流量摊到多台机器。ShardingSphere 把它做成**协议层透明**的能力——应用照常发 SQL，内核按语句类型路由到写库或读库，再由负载均衡算法挑选具体从库。看起来简单，真正决定体验的是三件事：**事务内读去哪、主从延迟带来的"读不到自己刚写的"如何兜底、从库故障如何摘除**。本篇围绕这三点展开，配置为 5.5.x 格式。

# 1. 概述

ShardingSphere 提供透明读写分离，写操作自动路由到主库，读操作负载均衡到从库。

```text
Application (JDBC / Proxy)
    │ SQL
    ▼
ShardingSphere
    │
    ├── INSERT/UPDATE/DELETE → writeDataSource (Master)
    │
    └── SELECT → readDataSource (Slave) 负载均衡
              ┌──────────┬──────────┐
              ▼          ▼          ▼
           ds_slave0  ds_slave1  (WEIGHT/ROUND_ROBIN 选择)
```

判定规则很直白：**SELECT 走读库，其余走写库**。两类语句需额外说明：`SELECT ... FOR UPDATE` 按写处理走主库；事务内 SELECT 的去向由策略决定（第 4 节）。读写分离与分片是两个正交维度——分片决定"去哪个库"，读写分离决定"进该库的主还是从"，可叠加（第 6 节）。

> **设计争议**：读写分离该在**中间件层**（ShardingSphere/应用路由）做还是**基础设施层**（ProxySQL、MySQL Router、云厂商代理）做？中间件层复用连接池、免一跳、可与分片/Hint 联动，但受形态语言限制（JDBC 仅 Java）；Proxy 层语言无关但要自己保高可用。云托管库通常把主从路由做在连接串层，此时再叠 ShardingSphere 读写分离属重复建设，选一层即可。

# 2. 读写分离配置（5.5.x）

5.5.x 用 `dataSourceGroups` 扁平声明一组主从（**取代 4.x/5.0 的 `staticStrategy`/`dynamicStrategy` 嵌套写法**）：

```yaml
rules:
  - !READWRITE_SPLITTING
    dataSourceGroups:
      readwrite_ds:
        writeDataSourceName: ds_master
        readDataSourceNames:
          - ds_slave0
          - ds_slave1
        loadBalancerName: round_robin

    loadBalancers:
      round_robin:
        type: ROUND_ROBIN
      random_lb:
        type: RANDOM
      weight_lb:
        type: WEIGHT
        props:
          ds_slave0: 10      # 属性名=从库名，值=权重
          ds_slave1: 5
```

`writeDataSourceName`/`readDataSourceNames` 默认走**行表达式**解析，因此可以批量声明：`writeDataSourceName: write_ds${0..1}`、`readDataSourceNames: read_ds${0..1}_${0..1}`，一套规则撑起多组分库的主从。

## 2.1 负载均衡算法

| 算法 | 说明 | 适用 |
|------|------|------|
| `ROUND_ROBIN` | 轮询（默认） | 同构从库 |
| `RANDOM` | 随机 | 同构、实现简单 |
| `WEIGHT` | 按权重（props 以从库名为 key 配 double 权重） | 从库规格不一 |

> **注意**：三种算法都**不感知后端实际负载**——轮询打到一台 CPU 90% 的从库照样继续发。想要"按真实压力分流"，要么实现 `LoadBalanceAlgorithm` SPI（结合 `Seconds_Behind_Master`、连接数加权），要么用自定义算法读指标动态调整权重。

# 3. 动态读写分离的演进（DB_DISCOVERY 已移除）

4.x 到 5.3.x 提供过独立的 `!DB_DISCOVERY` 规则：用 `dynamicStrategy + autoAwareDataSourceName` 让 ShardingSphere 通过心跳探测 MGR/主从组，自动感知主从拓扑与选主：

```yaml
# 仅 ≤5.3.x 有效；5.4+ 已移除 DB_DISCOVERY 模块，照抄会启动报错
rules:
  - !READWRITE_SPLITTING
    dataSources:
      readwrite_ds:
        dynamicStrategy:
          autoAwareDataSourceName: ds_mgr
  - !DB_DISCOVERY
    dataSources:
      ds_mgr:
        dataSourceNames: [ds_master, ds_slave0, ds_slave1]
        discoveryTypeName: mgr
        discoveryHeartbeatName: mgr_heartbeat
    discoveryTypes:
      mgr:
        type: MGR
```

**5.4 起数据库发现模块被整体移除**，读写分离统一收敛到第 2 节的 `dataSourceGroups` 静态声明——`writeDataSourceName`/`readDataSourceNames` 直接列出物理数据源。设计取向是"**主从拓扑与选主交给数据库层/云 HA，中间件不再自造一套发现**"：MGR 选主、ProxySQL/云 RDS 代理本身就是更专业的拓扑管理器，ShardingSphere 重复实现既难维护又易与真拓扑不一致。

| 能力 | ≤5.3.x（DB_DISCOVERY） | 5.4+ / 5.5.x |
|------|----------------------|-------------|
| 拓扑来源 | 中间件心跳探测 | 配置文件声明 + 数据库层选主 |
| 主库故障 | 跟随 MGR 新主 | 交 Proxy/MHA/云 HA，ShardingSphere 跟随重连 |
| 配置复杂度 | 高（发现类型+心跳） | 低（静态名单） |
| 推荐度 | 旧版本遗留 | **当前唯一方式** |

> **行业实践**：迁移旧 dynamic 配置时，把发现组名拆成显式的 `writeDataSourceName`/`readDataSourceNames`，主从切换交由数据库高可用层完成——5.5.x 读写分离自身只保证"选对读库、摘掉故障读库"，不保证"自动选主"。

# 4. 事务内读写策略

5.x 在数据源组上提供 `transactionalReadQueryStrategy`，控制**事务内 SELECT** 的去向：

```yaml
rules:
  - !READWRITE_SPLITTING
    dataSourceGroups:
      readwrite_ds:
        writeDataSourceName: ds_master
        readDataSourceNames: [ds_slave0, ds_slave1]
        transactionalReadQueryStrategy: PRIMARY   # PRIMARY / FIXED / DYNAMIC（默认）
        loadBalancerName: round_robin
```

```text
事务中的读写行为（strategy = PRIMARY）：
  BEGIN → SELECT (走主库！避免从库延迟读旧数据)
       → INSERT (走主库)
       → SELECT (仍走主库，同一主库连接内完成)
  COMMIT
```

三种取值：`PRIMARY` 事务内全部读走主库（最保守，主库压力大）；`FIXED` 事务首次读锁定一个从库、后续复用（同事务快照一致，但锁不住延迟）；`DYNAMIC` 每读随机选（吞吐最高、一致性最弱）。

**设计考量**：

- **默认值是 DYNAMIC**，与很多团队直觉相反——老教程"事务里写过再读会走主库"来自 4.x 隐式规则；5.x 把它显式化为 `PRIMARY`，需自己声明。
- **支付、库存等"读己所写"敏感链路配 PRIMARY 或用 Hint**，报表/列表容忍延迟的场景保持 DYNAMIC 换主库减压，别用一个全局开关糊弄两类需求。

# 5. Hint 强制路由

不依赖 SQL 形态、代码显式点名主库，是"读己所写"最可靠的兜底：

```java
// Java 代码强制主库
HintManager hintManager = HintManager.getInstance();
hintManager.setWriteRouteOnly();       // 强制走主库（写库路由）
try {
    List<User> users = jdbcTemplate.query("SELECT * FROM t_user", rowMapper);
} finally {
    hintManager.close();               // ThreadLocal，必须关闭
}
```

Proxy/JDBC 通用写法是 **SQL Hint 注释**，便于上游网关打标：

```sql
/* SHARDINGSPHERE_HINT: WRITE_ROUTE_ONLY=true */
SELECT * FROM t_user WHERE id = 1;
```

> **注意**：HintManager 是 **ThreadLocal 语义**。线程池里必须"用完即关"，异常路径也要放进 finally；`CompletableFuture` 换线程后 Hint 会丢，跨线程要么显式传递要么用 SQL 注释版。

# 6. 与分片的组合

读写分离组本质是"一个逻辑数据源"，可被分片规则当数据节点引用：

```yaml
rules:
  - !SHARDING
    tables:
      t_order:
        actualDataNodes: readwrite_ds.t_order_${0..1}   # 引用读写分离组
        tableStrategy:
          standard:
            shardingColumn: order_id
            shardingAlgorithmName: tbl_inline
    shardingAlgorithms:
      tbl_inline:
        type: INLINE
        props:
          algorithm-expression: t_order_${order_id % 2}

  - !READWRITE_SPLITTING
    dataSourceGroups:
      readwrite_ds:
        writeDataSourceName: ds_master
        readDataSourceNames: [ds_slave0, ds_slave1]
        loadBalancerName: round_robin
```

**效果**：分片路由先把 SQL 定到逻辑组 `readwrite_ds`，再由读写分离层决定进 `ds_master` 还是 `ds_slaveX`。生产形态常是"**每个分库一组主从**"，用行表达式批量声明：

```text
t_order (逻辑) ──分片(库)──► readwrite_ds0 / readwrite_ds1
                                │读写分离              │
                          master0├─slave00,slave01   master1├─slave10,slave11
```

# 7. 从库故障处理与熔断

```text
心跳/连接检测失败 → 标记不可用 → 从读负载中摘除（写主库不受影响）
       ↑                                   │
       └──── 探测恢复 → 自动重新加入读负载 ◄┘
```

静态名单下 ShardingSphere 通过连接池探活把故障从库移出候选；运维也可用 DistSQL 显式熔断（Cluster 模式全节点生效，5.5.x 用 ALTER ... ENABLE/DISABLE）：

```sql
-- 手动摘除某个从库
ALTER READWRITE_SPLITTING RULE readwrite_ds DISABLE ds_slave0;
-- 查看各读源状态
SHOW STATUS FROM READWRITE_SPLITTING RULE readwrite_ds;
-- 恢复
ALTER READWRITE_SPLITTING RULE readwrite_ds ENABLE ds_slave0;
```

> **注意**：主库故障时读写分离**不会自动把从库升主**——升主是数据库层（MGR 选主、MHA、云 HA）的职责，ShardingSphere 只跟随拓扑。指望中间件"主挂自动写从库"是危险的误解，可能双写脑裂。

# 8. 主从延迟与数据一致性取舍

读写分离的坑几乎都汇聚到复制延迟这一个物理事实上。按需求强度给决策表：

| 场景 | 一致性要求 | 推荐做法 |
|------|-----------|---------|
| 支付/下单后回显 | 读己所写 | `transactionalReadQueryStrategy: PRIMARY` 或 Hint |
| 事务内先查后改 | 快照一致 | PRIMARY；或把查询并入同一语句 |
| 列表页/搜索 | 容忍秒级延迟 | DYNAMIC，负载均衡到从库 |
| 报表/导出 | 容忍 + 怕压主 | 从库，专设分析从库（WEIGHT 调高） |
| 强一致 + 高读压 | 都不能放 | 半同步复制 + 位点等待，或改读缓存/ES |

**设计考量**：

- **"查不到刚写的单"八成是读写分离延迟**，先拿 `Seconds_Behind_Master` 与业务时间线对账，再决定升一致还是降读从。
- 延迟敏感但必须读从库的，可做**会话粘滞**：写成功后把用户临时钉在主库 N 秒，比全局 PRIMARY 省资源。
- 从库变多 ≠ 吞吐线性增长：慢 SQL 会占满从库连接反噬主库复制线程，限制大查询比加机器有效。

# 9. 小结

| 主题 | 一句话结论 |
|------|-----------|
| 判定 | SELECT→读库，其余→写库；FOR UPDATE 按写 |
| 配置 | 5.5.x 用 `dataSourceGroups` + `writeDataSourceName`/`readDataSourceNames`，可行表达式批量 |
| 负载 | ROUND_ROBIN/RANDOM/WEIGHT（不感知真实负载，需 SPI 扩展） |
| 事务读 | `transactionalReadQueryStrategy`：PRIMARY/FIXED/DYNAMIC，默认 DYNAMIC |
| Hint | `setWriteRouteOnly()` / SQL 注释，读己所写最可靠兜底 |
| 组合 | 读写分离组被分片当数据节点引用，生产每分库一组主从 |
| 故障 | 连接探活自动摘 + DistSQL DISABLE 手动熔断；主库不自动升 |

下一篇（6_data_encryption）看字段级透明加密：加解密改写原理、LIKE 检索的密文方案与密钥管理纪律。
