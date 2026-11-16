# 1. 概述

**读写分离是 Mycat 最成熟、生产使用率最高的特性**——比它的分库分表用得还多，因为"一个主库两个从库"的架构在 MySQL 生态里是绝对主流，而读写分离的风险远比分片低（不涉及数据搬迁）。Mycat 在这件事上的价值主张是：**应用侧完全无感知**，不需要在代码里维护两套 DataSource，不需要 AbstractRoutingDataSource 那种路由 hack，只需要把连接串指向 Mycat，写流量自动去主、读流量自动分摊到从。

```text
Application
    │ SQL
    ▼
┌──────────┐
│  Mycat   │  判断语句类型（读写分类器）
│  Router  │
└────┬─────┘
     ├── INSERT/UPDATE/DELETE/DDL → writeHost (主)
     └── SELECT                  → readHost (从，按 balance 策略负载均衡)
```

**读写分离的边界必须先讲清楚**：Mycat 做的是"基于语句类型的静态分流"，它不理解业务语义。`SELECT ... FOR UPDATE` 会被识别为写走主库，但 `SELECT` 里带一个子查询 UPDATE 语义（如 `SELECT GET_LOCK(...)`）就未必；更关键的是**它不保证"写后读一致"**——刚写完主库立刻读从库，可能读到旧数据，这是所有 Proxy 型读写分离的固有局限（见第 8 节）。

**1.6 与 2.x 的配置形态差异**：1.6 把主从关系写在 schema.xml 的 `<dataHost>` 里（`<writeHost>` 嵌套 `<readHost>`），负载均衡由 `balance`/`writeType`/`switchType` 三个属性控制；2.x 抽象出独立的 `clusters.json`，用 `clusterType: MASTER_SLAVE` + `readBalanceType` 表达同一件事。本篇以 1.6 为主线。

# 2. 1.6 的 dataHost 主从配置

## 2.1 一主两从完整可粘贴配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">

  <schema name="TESTDB" checkSQLschema="false" sqlMaxTimeout="100">
    <table name="orders" primaryKey="id" dataNode="dn1"/>
  </schema>

  <dataNode name="dn1" dataHost="host1" database="test_db"/>

  <dataHost name="host1" maxCon="1000" minCon="10"
            balance="1" writeType="0" switchType="1"
            slaveThreshold="100" dbType="mysql" dbDriver="native"
            initSQL="SET NAMES utf8mb4">
    <heartbeat>select 1</heartbeat>

    <!-- 主库：写流量全部进这里 -->
    <writeHost host="w1" url="192.168.1.10:3306" user="mycat" password="mycat123">
      <readHost host="r1" url="192.168.1.11:3306" user="mycat" password="mycat123"
                weight="10"/>
      <readHost host="r2" url="192.168.1.12:3306" user="mycat" password="mycat123"
                weight="5"/>
    </writeHost>

    <!-- 备用主库：w1 挂掉后按 switchType 提升 -->
    <writeHost host="w2" url="192.168.1.13:3306" user="mycat" password="mycat123"
               using="false"/>
  </dataHost>
</mycat:schema>
```

**结构上的三个要点**：`<readHost>` 必须嵌套在 `<writeHost>` 内部，表达"这对主从的绑定关系"；多个 `<writeHost>` 表示主备（不是多主，默认只有第一个 `using` 生效）；`initSQL` 里的语句会在每条新建后端连接上执行，字符集类设置放这里最稳。

## 2.2 三个决定路由的属性

| 属性 | 取值 | 含义 |
|------|------|------|
| **balance** | 0 | 所有读和写都走 writeHost（读写分离关闭） |
| | 1 | 随机在**所有可用节点**（含 writeHost + 全部 readHost）负载均衡读 |
| | 2 | 读请求按**权重**在所有节点间分配，写仍走当前 writeHost |
| | 3 | 读请求只在 **readHost** 间分配，writeHost 不参与读 |
| **writeType** | 0 | 所有写固定发到第一个可用的 writeHost |
| | 1 | 写请求在所有 writeHost 间负载均衡（真正多主时才用） |
| | 2 | 写请求在 writeHost 间**分布式事务负载均衡**（历史遗留，几乎不用） |
| **switchType** | -1 | 不做主从切换（写库挂了直接报错，交给外部 MHA/Orchestrator） |
| | 1 | 默认值，写库心跳失败即切换到下一个 writeHost |
| | 2 | 基于 MySQL Master 心跳检测切换 |
| | 3 | 写入全局心跳表 `mycatHeartbeat` 检测切换（对主从都探） |

> **行业实践**：`balance=1` 看起来最"均衡"，实际生产更推荐 `balance=3`——它让主库专心写，读全部压到从库。`balance=1` 会把读流量分到主库，在主库本来就是瓶颈的场景里等于自废武功。`switchType=-1` 是很多团队的选择：**把故障切换的决策权交给 MHA/Orchestrator 这类专门组件，Mycat 只做连接发现**，避免 Mycat 与各自动切换工具同时动作导致脑裂。

**slaveThreshold 与延迟感知**：只有当心跳 SQL 配置成能反映复制延迟的形式（1.6 的经典写法是 `<heartbeat>show slave status</heartbeat>`，并要求 Mycat 账号有 `REPLICATION CLIENT` 权限），`slaveThreshold` 才有意义——当 `Seconds_Behind_Master` 超过该阈值，Mycat 把这个 readHost 从读列表摘除。普通 `select 1` 心跳不携带延迟信息，此时 `slaveThreshold` 形同虚设。

# 3. 读负载均衡策略详解

**balance 的四种取值对应不同的流量分布**，用一张 ASCII 图看得更清楚（1 主 2 从，读请求 100 次）：

```text
balance=0   w1 ████████████████████████ (100 次，含全部写)   r1/r2 空闲
balance=1   w1 ~33次   r1 ~33次   r2 ~34次  （随机，主库参与读）
balance=2   w1 全部写 + 按 weight 分摊读：r1(w=10) 67%  r2(w=5) 33%
balance=3   写 → w1；读 → 仅 r1/r2 轮询，w1 完全不承担读
```

**权重（weight）只在 balance=2 时生效**，这是 1.6 的一个反直觉点：很多人给 readHost 配了 `weight` 却发现不生效，原因是 `balance=1/3` 走的是随机/轮询而不是权重。想让异构配置的从库（一台高配、一台低配）按能力分摊读流量，必须 `balance=2`。

Mycat2 把这套逻辑抽象成 `clusters.json` 的 `readBalanceType`：

```json
{
  "clusters": {
    "rw_cluster": {
      "clusterType": "MASTER_SLAVE",
      "readBalanceType": "BALANCE_ALL",
      "master": {
        "name": "m1",
        "url": "jdbc:mysql://192.168.1.10:3306/test_db",
        "user": "mycat",
        "password": "mycat123",
        "type": "JDBC",
        "initSQLs": ["SET NAMES utf8mb4"]
      },
      "replicas": [
        { "name": "s1", "url": "jdbc:mysql://192.168.1.11:3306/test_db",
          "user": "mycat", "password": "mycat123", "type": "JDBC", "weight": 10 },
        { "name": "s2", "url": "jdbc:mysql://192.168.1.12:3306/test_db",
          "user": "mycat", "password": "mycat123", "type": "JDBC", "weight": 5 }
      ]
    }
  }
}
```

| 类型 | Mycat2 配置值 | 对应 1.6 balance | 说明 |
|------|--------------|-----------------|------|
| 全部均衡 | `BALANCE_ALL` | 1 | 所有节点（含主）参与读负载 |
| 不均衡 | `BALANCE_NONE` | 0 | 读也走 Master，读写分离关闭 |
| 权重均衡 | `BALANCE_ALL_READ` | 2 | 按 `weight` 权重分配读请求 |
| 仅从库 | `BALANCE_SLAVE` | 3 | 只分摊到 replicas |
| 就近 | `BALANCE_NEARBY` | 无 | 优先同机房 Slave（2.x 新增） |

# 4. writeHost 挂掉后的切换与 backup 读

**主库故障时 Mycat 的默认行为分三步**：心跳连续失败达到 `retryCount` 次（默认 3）→ 把 w1 标记为不可用 → 按 `switchType` 决定是否把 w2 提升为当前写库。**w2 的 readHost（若配置）会同时加入读列表**——这就是"backup 承担读"：主从都挂一半时，系统靠备用主库硬撑读写，牺牲一致性保可用。

```text
正常态                          w1 故障态（switchType=1）
w1(主) ← 写                    w2(备主) ← 写提升
 ├── r1 ← 读                    ├── r1 ← 读（跨主复制链路可能变长）
 └── r2 ← 读                    └── r2 ← 读
w2 闲置(using=false)            w1 摘除，心跳持续探测待恢复

注意：r1/r2 原本复制自 w1，w1 挂掉后它们并不会自动改挂到 w2
——Mycat 不管复制拓扑，主从切换必须由 DBA/工具先改好复制关系
```

> **设计争议**：Mycat 的自动切换（`switchType=1/2/3`）设计于 MySQL 高可用工具尚不成熟的 2015 年前后，今天被普遍认为是**该关闭的功能**——Mycat 只看"连接是否存活"，无法判断旧主是真的死了还是网络分区，自动提升新写库可能造成双写脑裂。当前主流做法是 `switchType=-1` 禁自动切换 + 外部用 MHA/Orchestrator/RDS 高可用完成主从切换 + Mycat 靠心跳自动发现新拓扑。**行业实践**：切换后务必人工或用脚本校验 Mycat 后端连接指向，`SHOW @@DATAHOST` 里 `ACTIVE backend connections` 的分布会告诉你流量去了哪。

**恢复回切不是自动的**：w1 修好后心跳恢复，Mycat 只是把它重新放回可用池，**不会自动把写切回 w1**（避免反复抖动）。要么接受"w2 长期当主"，要么在低峰期人工 reload 配置或重启回切——这是很多团队"故障后主从角色错乱"的根源。

# 5. 强制路由与事务内的读写规则

除了 balance 的全局策略，Mycat 提供三条细粒度控制通道。**第一是 Hint 注释**，1.6 的经典语法：

```sql
-- 强制走主库（解决写后读不一致的杀手锏）
/*!mycat:sql=SELECT * FROM orders WHERE user_id = 7*/ SELECT * FROM users;  -- 指定路由 SQL
/*!mycat:db_type=master*/ SELECT * FROM users WHERE id = 1;                 -- 强制主库
/*!mycat:db_type=slave*/  SELECT * FROM users WHERE id = 1;                 -- 强制从库

-- 走指定数据节点
/*!mycat:datanode=1*/ SELECT * FROM orders;

-- 开启该语句的事务级主库粘性（会话内后续读也走主）
SET autocommit = 0;   -- 事务内所有语句 Mycat 自动钉在 writeHost
```

**第二是 SQL 类型本身的判定规则**。Mycat 把这些语句无条件路由到 writeHost：所有 INSERT/UPDATE/DELETE/REPLACE/DDL；包含 `FOR UPDATE`、`LOCK IN SHARE MODE` 的 SELECT；调用 `LAST_INSERT_ID()`、`FOUND_ROWS()`、会话变量的 SELECT；显式事务中的所有语句。**在 Mycat 里开启事务（`SET autocommit=0` 或 JDBC `setAutoCommit(false)`）后，事务期间的所有读也会被钉在主库**——这不是保守，而是必需：读从库的事务在跨节点写时会破坏一致性，Mycat 的事务连接本来就是按节点绑定 writeHost 的。

**第三是 ORM 层配合**。ShardingSphere 用 HintManager，Mycat 只能靠 SQL 注释，很多团队封装 `MasterQueryInterceptor`，在 MyBatis 拦截器里给特定方法自动拼 `/*!mycat:db_type=master*/` 前缀：

```java
// MyBatis 拦截器示意：写后 N 秒内的查询强制走主库
public class MasterRouteInterceptor implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        BoundSql sql = ((Executor) invocation.getTarget()).getBoundSql();
        if (RecentWriteCache.hasWritten(sql.getSql())) {   // 3 秒内有相关写入
            sql.setParameter("__mycat_hint", "/*!mycat:db_type=master*/");
        }
        return invocation.proceed();
    }
}
```

# 6. 延迟感知的局限

**Mycat 的"延迟感知"能力必须泼冷水**，这是它与云厂商 Proxy（如 PolarDB Proxy、RDS Proxy）差距最大的地方：

1. **`show slave status` 心跳本身不可靠**：`Seconds_Behind_Master` 反映的是 IO 线程与 SQL 线程的位点差，主库压大时该值抖动剧烈，甚至复制中断仍报 0；Mycat 拿它对比 `slaveThreshold` 只能做到"粗粒度摘除明显落后的从库"
2. **摘除动作有滞后**：心跳默认 10 秒间隔 + 失败重试，从"延迟飙升"到"被摘除"最少半分钟，期间的读请求照常打到落后从库
3. **没有 GTID/位点级一致性读**：Proxy 界的先进做法（写入时记录位点、读时校验从库位点已追平才放行）Mycat 1.6 完全没有；2.x 的延迟检测同样停留在阈值层面
4. **`slaveThreshold` 对匿名复制拓扑无效**：级联复制（主→从→从）下末级从库的 `Seconds_Behind_Master` 语义与直连从库不同，阈值没法统一设

**设计考量**：

- **对延迟敏感的业务链路（下单后查单、支付后查状态）不要依赖 Mycat 的延迟摘除**，用 Hint 强制主库或把该查询改为依赖写路径的返回值，是唯一可靠的方案
- **把主从延迟控制在亚秒级是前提**：并行复制（`slave_parallel_workers`）、row 格式 binlog、log_slave_updates 关闭等 MySQL 侧优化，比中间件的延迟检测更有效
- **监控要独立**：用 PT-heartbeat 或 ` Seconds_Behind_Master ` 的 exporter 直接监控 MySQL，不要等 Mycat 日志里"heartbeat fail"才发现问题

# 7. 心跳检测机制

```xml
<!-- 1.6 dataHost 心跳的两种写法 -->
<!-- 普通探活：只判断连接可用性 -->
<heartbeat key="masterShowSLave">show slave status</heartbeat>

<!-- 推荐生产写法：写心跳表探测主从真实可写性（switchType=3 配合） -->
<heartbeat>select 1</heartbeat>
```

心跳线程的工作循环：按 `heartbeat` 间隔（server.xml `idleTimeout` 之外另有系统属性控制）对每个 writeHost/readHost 执行心跳 SQL → 成功则标记 `IS_AVAILABLE`，失败计数 +1 → 连续失败达到 `retryRetry`（系统属性 `heartbeatRetryCount`，默认 3）标记宕机并触发 switchType 决策 → 恢复后自动重新纳入均衡池。

```sql
-- 9066 管理端口观察心跳结果
SHOW @@HEARTBEAT;
SHOW @@DATAHOST;
-- NAME      TYPE  URL                 IS_ACTIVE  IN_USE  ACTIVE_CONN
-- w1        write 192.168.1.10:3306   1          0       12
-- r1        read   192.168.1.11:3306   1          0       8
-- r2        read   192.168.1.12:3306   0          0       0   ← 已被心跳摘除
```

> **注意**：心跳 SQL 用 `show slave status` 时，Mycat 连的账号必须在该节点上能看到复制状态；更隐蔽的坑是**某些云 RDS 禁止普通账号执行 `show slave status`**，导致心跳永远失败、从库全部被摘除，表现为"配了读写分离但读全打到主库"。云环境一律用 `select 1` 探活 + 云监控管延迟。

# 8. 小结

**Mycat 读写分离 = dataHost 的 writeHost/readHost 拓扑 + balance 负载均衡策略 + 心跳摘除机制**三件套。配置层面记住四点：`balance=3` 让主库专职写、异构从库按权重分流必须 `balance=2`、`switchType=-1` 把切换交给外部 HA 工具、心跳账号权限与云 RDS 限制要先验证。能力层面记住一句：**它能摘"死的"从库，但对"慢的"从库只有粗粒度阈值感知，写后读一致必须靠 Hint 或事务钉主自行解决**。与 ShardingSphere-Proxy 相比，Mycat 的读写分离胜在配置简单直白，输在没有位点一致性读与强制读主策略的会话级自动化。

