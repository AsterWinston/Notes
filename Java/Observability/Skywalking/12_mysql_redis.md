
# 1. 概述

数据库与缓存访问通常占业务请求耗时的 50% 以上，是 Trace 里必须看清的"最后一公里"。SkyWalking 对 MySQL/Redis 的追踪思路与其他中间件一致：**在 driver/client 的关键方法上生成 Exit Span**，SQL/命令文本作为 tag 挂上去，peer 指向目标实例。本篇覆盖四件事：插件原理、慢 SQL 阈值与采集、SQL 参数上报的安全边界、span 粒度与采样的相互作用。

```text
Service Entry Span: GET:/api/order/{id}
  ├── Exit Span: Mysql/JDBC/PreparedStatement/execute   ← mysql 插件
  │     tags: db.type=mysql, db.instance=order_db,
  │           db.statement=SELECT ... WHERE id=?
  ├── Exit Span: Redis/Jedis/get                        ← redis 插件
  │     tags: db.type=Redis, db.instance=r-xxx:6379/0
  └── Local Span: 业务方法（若有）
```

# 2. MySQL：JDBC Driver 插件原理

mysql 插件不解析 SQL、不碰 ORM，**拦截点在 JDBC 规范的两个位置**：

```text
1) java.sql.DriverManager / mysql ConnectorJ Driver#connect
      → 拿到 Connection 时把 URL 解析成 peer（host:port）与 db.instance（库名）
2) Statement#execute* / PreparedStatement#execute*
      → beforeMethod: 以 SQL 摘要为 operationName 创建 Exit Span
        afterMethod : 结束 Span；异常 → errorOccurred + log(exception)
```

这样设计的收益是**与 ORM 完全解耦**：MyBatis、JPA/Hibernate、jOOQ 最终都要落到 `PreparedStatement.execute`，插件天然全覆盖；ORM 层不需要任何感知。代价是**批量执行/预编译复用场景的 span 语义**——`addBatch + executeBatch` 只产生一个 Span（拦截点在 `executeBatch`），N 条 SQL 的耗时被折叠成一个 Exit Span，这是设计使然不是丢数据。

## 2.1 Span 标签

| 标签 | 说明 | 示例 |
|------|------|------|
| `db.type` | 数据库类型 | `mysql` |
| `db.instance` | 库名 | `order_db` |
| `db.statement` | SQL 文本（参数默认打码） | `SELECT * FROM t_order WHERE id = ?` |
| `peer` | 目标地址 | `10.0.1.5:3306` |

`db.statement` 记录的是** PreparedStatement 的原始模板**（带 `?` 占位符），这就是参数化上报的安全基线：**参数值默认永远不上报**。

## 2.2 SQL 参数上报与脱敏

开启参数采集后，插件用 `PreparedStatement` 记录下来的 `setXxx` 值回填占位符：

```properties
# agent.config（键名与官方配置表一致）
plugin.jdbc.trace_sql_parameters=true            # 回填 SQL 参数，默认 false（SW_JDBC_TRACE_SQL_PARAMETERS）
plugin.jdbc.sql_parameters_max_length=512        # 单个参数最大长度，超长截断
plugin.jdbc.sql_body_max_length=2048             # SQL 语句本体最大长度
```

> **警告**：`trace_sql_parameters=true` 会把身份证、手机号、密码明文送进 ES——而 ES 里的数据默认按"可观测数据"的权限管理，通常比生产库宽松得多。行业通行做法是**仅预发/灰度环境开启**，或开启前确认插件版本是否支持字段级脱敏（较新版本 agent 提供 `op_name_collect`/自定义 converter 能力，但对"哪些参数要洗"没有万能答案）；更稳妥的替代是业务侧用 `@Tag` 显式选择要上报的安全参数，而不是全量回填。

# 3. 慢 SQL 阈值与采集

慢 SQL 的判定发生在 **OAP 侧 trace receiver**，而不是 agent——agent 只负责把带耗时的 DB span 完整上报，OAP 在分析 Span 时按阈值打标：

```yaml
# OAP config/application.yml — receiver-trace 段
receiver-trace:
  default:
    slowDbThreshold: 200      # DB span 超过 200ms 判为慢 SQL（毫秒）
    slowHttpThreshold: 1000   # 跨进程 http 调用的 RPC 耗时阈值
    slowCustomThreshold: 500  # 自定义/local span 阈值
```

效果链路（OAP 侧两条独立产出）：

```text
上报的 segment 中的 DB span
  ├── TraceAnalyzer 按 slowDbThreshold 打标 → UI Trace 列表可筛"慢链路"
  └── SQLStatementAnalyzer 提取 db.statement → sw_sql_statement-YYYYMMDD 索引
        └── UI: Database → Slow SQL statements 榜单（按 latency 排序，含耗时/次数）
```

**注意慢 SQL 榜单与采样正交**：慢 SQL 判定基于**上报到 OAP 的 segment**，**如果整条 Trace 未命中采样，它的慢 SQL 也不会出现在榜单里**。高流量服务把采样调低后慢 SQL 榜单"变干净了"往往是假象——真实结论应该去数据库侧（`long_query_time` + 慢日志）交叉验证。

> **行业实践**：SkyWalking 慢 SQL 榜单适合回答"这段时间哪些语句在拖后腿"，不适合替代 DBA 的慢日志体系：它看不到没有走应用的 SQL（跑批、DMS 手工查询），也没有执行计划与锁等待信息。两边用 traceId/时间窗互查，才是完整拼图。

# 4. Redis 客户端插件

Redis 插件按 client 类型分别增强：Jedis 拦 `Jedis#sendCommand`、Lettuce 拦 `RedisChannelWriter#write`（异步命令完成时结束 span）、Redisson 拦命令执行入口。span 模板统一为 `Redis/{命令}`：

| 客户端 | 拦截点 | 异步/批处理表现 |
|--------|--------|----------------|
| Jedis | `Connection#sendCommand` | Pipeline 合并为命令序列的多 span（逐命令） |
| Lettuce | `RedisChannelWriter` | 异步 future 完成点收尾，跨线程不断链 |
| Redisson | 命令 API 入口 | 带 `RQueue` 等结构语义 |

```properties
# agent.config — 参数采集按客户端类型分别配置（默认全部关闭）
plugin.jedis.trace_redis_parameters=false        # Jedis 命令参数
plugin.jedis.redis_parameter_max_length=1024     # 超长截断
plugin.lettuce.trace_redis_parameters=false      # Lettuce 同理
plugin.redisson.trace_redis_parameters=false     # Redisson 同理
```

Lettuce 的响应式 API 与 Spring WebFlux 组合是最容易丢链的场景：命令发出在 Netty 写线程、回调在另一线程，若业务链路的上下文没被 Reactor 传播接住，`GET` 的 span 会变成孤儿或消失——**排查看 agent.log 的 instrumentation 命中记录，再确认 reactor 插件版本在支持矩阵内**。

> **注意**：MGET/HMGET 这类批量命令产生一个 Span，但 keys/field 列表可能很长，`db.statement` 有截断；缓存 key 命名不规范（含用户 ID 高基数）不会造成端点爆炸——Redis 的 Span operationName 只到命令级（`Redis/GET`），**参数 key 只在 statement 里**，端点维度天然安全。

# 5. 连接池指标

连接池是"DB 慢"最常见的替罪羊：SQL 本身 5ms，但 `getConnection` 排队 200ms。JDBC span 只覆盖执行段，**拿不到连接的时间表现为 Entry 与 JDBC Exit 之间的"空白"**，要靠连接池指标显形。agent 对主流池的增强方式：

| 连接池 | 指标通道 |
|--------|---------|
| HikariCP | agent 插件直接采集池状态（active/idle/pending/wait），经 Meter 通道上报 |
| Druid | 同上（复用 Druid 自身 JMX/统计） |
| DBCP2 / Tomcat JDBC Pool | 增强对应 MBean/PoolStats 上报 |

UI 的 Instance Dashboard 展示 `connection_pool` 相关面板；告警建议盯两条：**pending（等待连接的线程数）** 持续 >0 即池不足，**wait duration（平均获取等待）** 突增通常是池内连接被慢 SQL 长期占用——先看 SQL 再看池参数，方向别搞反。

# 6. 数据库端 Span 粒度控制

高频循环查询会把一个 segment 塞满几百个 JDBC span，三个旋钮按优先级组合：

1. **`agent.span_limit_per_segment=300`**：超配额后新 span 不再记录（Trace 上表现为 span 数量整齐地断在某个数），批处理场景优先调大它；
2. **忽略配置**：`plugin.jdbc.ignore_statements`（按语句/正则忽略心跳 SQL、框架内部探测 SQL），把噪音源直接掐掉；
3. **Redis 类阈值**：`plugin.redis.command_threshold` 把同段内同类命令合并，防"一次循环 50 个 GET"刷屏。

```text
一条消息处理任务的 Trace（调优前）：
  Entry(1) + Kafka(1) + JDBC(1) + Redis×80 = 83 spans，UI 无法阅读
调优后：
  command_threshold=10 折叠 Redis；ignore_statements 干掉框架心跳
  Entry(1) + Kafka(1) + JDBC(1) + Redis(汇总10) ≈ 13 spans
```

**数据库虚拟实体（DatabaseAccess）的聚合粒度由 OAL 自动完成**，与 Trace 侧的 span 折叠互不影响——折叠只影响链路可读性，`database_access_cpm/resp_time` 指标仍按全量采样统计。

# 7. 采样对 DB span 的影响

这条规则要刻进脑子：**SkyWalking 头部采样以 Entry 为决策点，整条 Trace 要么全采要么全不采**（agent 默认 `sample_n_per_3_secs` 窗口计数，非概率抽样）。对数据库可观测性的推论：

```text
未命中采样的 Trace：
  ├── 无 segment 上报 → 无 DB span 明细 → 无慢 SQL 记录
  ├── DB 访问不生成任何 database_access 指标（指标从 Trace 流衍生）
  └── 但下游服务仍通过 sw8 知道"我在链路中"（传播继续，只是不落库）

命中采样的 Trace：DB span 完整上报
```

所以"采样调低后 DB 面板指标掉了一个数量级"不是丢数据，而是**指标本就构建在采样的 Trace 流上**——这是 SkyWalking 指标体系（《6. 指标》一节 L1 聚合）与 DB 插件的耦合点。若 DB 指标必须全量，正确做法不是把采样拉满，而是把关键 DB 访问拆独立服务级统计（或用 MAL 导入数据库 exporter 指标）。

# 8. 性能开销

| 操作 | 额外开销 | 说明 |
|------|---------|------|
| JDBC 每次 execute | < 5μs（约 0–1%） | 拦截器 + span 创建为 ThreadLocal 操作 |
| Redis 每命令 | < 3μs | 命令名解析极轻 |
| SQL 参数采集开启 | 明显上升 | setXxx 全量缓存 + 回填字符串拼接 |

拦截器本身的成本几乎可以忽略；**真正要警惕的是"参数采集 + 大 IN 列表 + 超长 SQL 文本"三件套**——statement 字符串会占上报包体的大头，让 agent 的 buffer 更快轮转、序列化更慢，压测时按打开该开关的真实配置来。

# 9. 小结

| 主题 | 要点 |
|------|------|
| mysql 插件 | 拦 Driver connect 与 Statement.execute，与 ORM 解耦；batch 折叠为单 span |
| SQL 参数 | 默认只报模板 `?`；开启回填 = 明文入库风险，仅灰度/预发使用 |
| 慢 SQL | OAP `receiver-trace` 的 `slowDbThreshold` 判定，依赖 Trace 命中采样；不能替代 DB 慢日志 |
| redis 插件 | Jedis/Lettuce/Redisson 全覆盖，operationName 到命令级天然防端点爆炸 |
| 连接池 | Hikari/Druid 等池指标看 pending 与 wait duration，定位"拿不到连接"型慢 |
| 粒度控制 | span_limit + ignore_statements + command_threshold 三旋钮，指标不受折叠影响 |
| 采样 | DB span 与指标全部依赖头部采样；"未采样不落库"是所有 DB 排障假象的根源 |

数据库段补全后，一个请求从入口到存储的每个耗时来源都有了归属——剩下的问题从"在哪里慢"变成"怎么配、怎么运维"，参见后续《14. 存储后端》《17. 最佳实践》。
