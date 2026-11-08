影子库（Shadow Database）是 ShardingSphere 的**全链路压测**数据层方案：生产系统直面大促前，必须用真实流量模型压一遍，但压测数据绝不能污染生产数据。思路是给压测流量打标记，识别出标记的 SQL 整体改道到一套结构相同的影子库，生产库毫发无损。它解决的核心问题是"**数据隔离**"而非"流量注入"——流量打标、透传是链路层（网关/RPC/中间件）的事，ShardingSphere 只负责"见到标记就换库"。本篇讲影子规则怎么配、流量怎么判定、与分片/读写分离怎么叠加、以及和传统"单独建测试库"方案的对比。

# 1. 概述

```
┌─────────────────────────────────────────────┐
│              Application                     │
│  ┌──────────────┐        ┌──────────────┐   │
│  │ 正常流量      │        │ 压测流量      │   │
│  │ (shadow=未命中)│        │ (shadow=命中)  │   │
│  └──────┬───────┘        └──────┬───────┘   │
└─────────┼───────────────────────┼────────────┘
          │ ShardingSphere        │ 影子算法判定
          ▼                       ▼
   ┌────────────┐         ┌──────────────┐
   │ 生产数据库  │         │  影子数据库    │
   │ (真实数据)  │         │ (压测数据)    │
   └────────────┘         └──────────────┘
```

生产与影子在**同一个逻辑数据源组**里成对声明，SQL 到达时影子算法逐条判定：命中→影子库，未命中→生产库。判定发生在路由阶段之前，因此影子对上层完全透明——业务系统不用为压测改一行代码，压测标记由链路注入。

> **行业实践**：全链路压测的标配是"标记随请求走完全程"：压测平台在 HTTP Header 打标 → 网关透传 → RPC（Dubbo/gRPC attachment）透传 → 线程池透传（TransmittableThreadLocal）→ 最终由 ShardingSphere 影子规则消费标记落影子库。任何一环丢了标，压测数据就会写进生产库，这是全链路压测最严重的事故类型。

# 2. 配置

数据源成对声明（生产 + 影子，表结构必须一致），规则用 `!SHADOW`：

```yaml
dataSources:
  ds_prod:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.cj.jdbc.Driver
    jdbcUrl: jdbc:mysql://prod-db:3306/order_db
    username: root
    password: root
  ds_shadow:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.cj.jdbc.Driver
    jdbcUrl: jdbc:mysql://shadow-db:3306/order_db
    username: root
    password: root

rules:
  - !SHADOW
    dataSources:
      shadowDataSource:                 # 生产-影子映射对
        productionDataSourceName: ds_prod
        shadowDataSourceName: ds_shadow

    tables:
      t_order:
        dataSourceNames:
          - shadowDataSource            # 哪些表参与影子
        shadowAlgorithmNames:
          - user_id_insert_match        # 判定算法（可多个，命中其一即影子）
      t_order_item:
        dataSourceNames:
          - shadowDataSource
        shadowAlgorithmNames:
          - user_id_insert_match
          - sql_hint_algorithm

    shadowAlgorithms:
      user_id_insert_match:
        type: VALUE_MATCH               # 列值匹配
        props:
          operation: insert
          column: user_id
          value: 1
      sql_hint_algorithm:
        type: SQL_HINT                  # SQL 注释打标
```

两个结构细节：**shadow 算法按表挂载**（`tables.<逻辑表>.shadowAlgorithmNames`），不同表可用不同判定规则；**多算法是 OR 关系**，任一算法判定为影子流量即进影子库。

# 3. 影子算法：流量如何打标

| type | 判定依据 | props | 适用 |
|------|---------|-------|------|
| `VALUE_MATCH` | 指定列的值等于固定值 | column、operation、value | 压测用户 ID 预留段（如固定 1） |
| `REGEX_MATCH` | 指定列的值匹配正则 | column、operation、regex | 压测账号段号区间 |
| `SQL_HINT` | SQL 前注释含 `/* SHARDINGSPHERE_HINT: SHADOW=true */` | 无 | 网关统一打标 |

```yaml
shadowAlgorithms:
  user_id_match:
    type: REGEX_MATCH
    props:
      operation: insert          # 只判定 INSERT
      column: user_id
      regex: "^[9]\\d{6,}$"      # 9 开头的 7 位以上 ID 视为压测账号
```

SQL Hint 形态（Proxy/JDBC 通用，压测标记从网关直接拼进注释）：

```sql
/* SHARDINGSPHERE_HINT: SHADOW=true */
INSERT INTO t_order (user_id, status) VALUES (9000001, 'PAID');
```

```java
// Java 侧没有 setShadow() 这类专用 API——5.x 的 HintManager
// 只提供分片/读写路由相关的 Hint；影子流量打标请统一使用 SQL_HINT 注释形态
// （由网关/拦截器在 SQL 前拼接 /* SHARDINGSPHERE_HINT: SHADOW=true */），
// 或给压测账号预留 ID 段走 VALUE_MATCH/REGEX_MATCH。
// 旧笔记/教程中的 HINT_MATCH 类型与 HintManager.setShadowHuntEnabled()
// 属于 4.x 时代写法，5.5.x 已不存在，迁移时一律换成上述两种方式。
```

**设计考量**：

- **REGEX/VALUE_MATCH 是"数据自带标记"**：压测账号段与真实账号段物理隔离，链路上任何一跳丢标都不影响影子判定正确性，最稳；代价是要预分配压测用户 ID 段。
- **SQL_HINT/Java Hint 是"链路带标记"**：不依赖数据形态，适合无法预留账号段的存量系统；但打标能力必须全链路打通，丢标即污染生产。
- 判定只作用于 `operation` 指定的语句类型，SELECT 是否也进影子取决于挂载的算法——生产压测通常读写都要隔离，否则压测读会放大到生产库。

# 4. 全链路压测流程

```
1. 影子库初始化：结构=生产，灌入必要基础数据（商品、库存种子）
2. 压测平台构造压测账号/参数 → 请求携带影子标记
3. 网关接收标记 → 透传至 RPC 上下文 → 线程池透传
4. ShardingSphere 影子算法识别流量（按表+列值/Hint）
5. 命中 → 该 SQL 读写整体路由影子库；未命中 → 生产库
6. Prometheus/Grafana 监控影子链路与生产链路各自的指标
7. 压测结束 → 清理影子库数据 → 复盘瓶颈
```

影子库的容量、连接池参数与生产**同规格**才有压测意义；影子库资源（磁盘、复制带宽）要与生产物理隔离，否则"压垮影子"等于压垮生产。

# 5. 与分片、读写分离组合

影子与分片天然可叠加——先按影子判定选"生产组/影子组"，再在组内做分片路由：

```yaml
rules:
  - !SHARDING
    tables:
      t_order:
        actualDataNodes: ds_prod${0..1}.t_order_${0..1}
        databaseStrategy:
          standard:
            shardingColumn: user_id
            shardingAlgorithmName: db_inline
    shardingAlgorithms:
      db_inline:
        type: INLINE
        props:
          algorithm-expression: ds_prod${user_id % 2}

  - !READWRITE_SPLITTING
    dataSourceGroups:
      readwrite_ds0:
        writeDataSourceName: ds_prod0
        readDataSourceNames: [ds_prod0]

  - !SHADOW
    dataSources:
      shadow_ds0:
        productionDataSourceName: readwrite_ds0
        shadowDataSourceName: ds_shadow0    # 生产组也可以是读写分离组
    tables:
      t_order:
        dataSourceNames: [shadow_ds0]
        shadowAlgorithmNames: [sql_hint_algorithm]
    shadowAlgorithms:
      sql_hint_algorithm:
        type: SQL_HINT
```

组合时的顺序：**影子判定在最外层**（改道整个数据源组），分片/读写分离在组内继续生效。影子组内的分片算法表达式要与生产组保持一致（t_order_${user_id % 2}），否则"同一条数据在生产落 ds_prod0、在影子落 ds_shadow1"，压测结果失真。

# 6. 与"单独测试库"方案对比

| 维度 | 影子库（ShardingSphere） | 独立压测环境 | 生产表内打标记列 |
|------|------------------------|-------------|----------------|
| 数据真实性 | 配置/路由与生产一致 | 需人工同步，易漂移 | 与生产同源同库 |
| 隔离强度 | 实例级物理隔离 | 环境级隔离 | **无隔离**，索引/锁竞争仍在生产 |
| 流量改造 | 打标即可，代码零改动 | 需要环境路由改造 | 每条 SQL 都要带标记条件 |
| 链路覆盖 | 只覆盖到数据层 | 依赖全环境搭建 | 仅数据层 |
| 成本 | 一套影子集群 | 一套完整环境 | 低 |
| 污染风险 | 丢标即污染生产（需校验） | 不污染生产 | 删除标记数据时误删风险 |

结论很直接：**追求真实容量评估选影子库；合规不允许压测流量进生产链路、或只做组件级压测选独立环境；表内标记列方案在生产系统上基本不可取**。

# 7. 注意事项

1. **数据隔离**：影子库必须是独立实例（独立主机/存储），只做逻辑分库不算隔离。
2. **DDL 同步**：生产每次 DDL 变更同步到影子库，纳入发布流程——影子表落后会导致压测期间整链路 SQL 报错。
3. **清理策略**：压测结束定时清空影子库（影子数据无长期价值），防止存储膨胀拖垮备份体系。
4. **监控区分**：影子流量与生产流量指标分开采集（可按影子数据源名区分维度），避免压测告警淹没生产告警。
5. **资源隔离**：影子库的计算/IO 资源不能与生产争抢，混布实例是常见事故源。
6. **防呆校验**：上线影子规则后先在影子库查数验证"压测账号确实不在生产表出现"，长期跑一个"生产库扫描压测 ID 段"的巡检任务兜底。

> **警告**：影子规则是**旁路改道**，不是权限管控——一旦影子算法 props 写错（正则漏锚点 `^$`、Hint 未透传），压测数据直接写进生产库且事后极难剥离。任何影子配置变更按生产变更流程对待。

# 8. 小结

| 主题 | 一句话结论 |
|------|-----------|
| 定位 | 全链路压测的数据层隔离：命中标记的 SQL 整体改道影子库 |
| 配置 | `!SHADOW`：生产/影子数据源成对 + 按表挂算法，多算法 OR |
| 打标 | VALUE_MATCH/REGEX（数据自带，最稳）vs SQL_HINT/Hint（链路带，需透传） |
| 组合 | 影子在最外层，分片/读写分离在组内继续；算法表达式须一致 |
| 对比 | 容量评估选影子库，组件压测选独立环境，标记列方案不可取 |
| 纪律 | 实例物理隔离、DDL 同步、压后清理、监控分渠道、变更防呆 |

下一篇（8_distributed_transaction）看跨分片写的一致性：LOCAL、XA、BASE 三种事务模型如何选。
