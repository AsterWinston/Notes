# 1. 分片设计原则

```text
分片键选择的黄金法则：
  1. 高频查询 WHERE 条件中的列
  2. 不影响业务逻辑的不可变列
  3. 区分度高的列（避免数据倾斜）
  4. 尽量与 JOIN 关联键一致（避免跨分片 JOIN）

推荐：业务主键（订单号/用户ID/SKU）
不推荐：状态、类型等低区分度列
```

分片键是整个分片体系里**唯一不可在线修改的东西**（改它意味着全量数据重分布），所以它值得在设计评审里花最多时间。四条法则各自展开：**高频查询列**决定"多少比例的 SQL 能单片路由"，用慢查询日志/全量 SQL 统计 WHERE 列出现频率来定，而不是拍脑袋；**不可变性**是硬约束——分片键一旦能 UPDATE，改值即改归属，ShardingSphere 不自动跨片搬行，数据会留在错误的片上；**区分度**避免大商家/热点租户把一个分片撑爆，可用"分片键值 → 行数"直方图验证（TOP 1% 键值占比 >30% 即倾斜预警）；**JOIN 对齐**决定订单-明细这类表组是否需要绑定表/基因法补丁。

## 1.1 基因法解决买家卖家双维度分片

交易类系统的经典矛盾：买家要按 `buyer_id` 查"我的订单"，卖家要按 `seller_id` 查"店铺订单"，一个分片键服务不了两个维度。基因法的思路是**让 order_id 内嵌 buyer_id（或 seller_id）的"基因片段"**，使另一维度查询能反解路由：

```text
分片规则：
  买家维度：db_index = buyer_id % 16
  订单号生成：order_id = snowflake_id 高位 + 基因段
             基因段 = buyer_id % 16（与库路由同函数）

查询路由：
  WHERE buyer_id = ?          → 正常分片键路由
  WHERE order_id = ?          → 取 order_id 末4位基因 → 同库路由（1 片）
  WHERE seller_id = ?         → 无法路由 → 该维度走 ES/异构索引（异步 binlog 同步）
```

```java
// 基因嵌入主键：生成订单号时把分片基因写入低位
long orderId = (snowflakeId << 4) | (buyerId % shardTotal); // 低 4 位存基因
// 反解路由（INLINE 表达式示意）：
// db_index = orderId % 16  与  buyerId % 16 恒等
```

**基因法的边界要认清**：基因只能种一种维度（通常买家），卖家维度仍需搜索/数仓异构索引兜底；订单号格式对外暴露（对账、发票）时基因段泄露分片规模，扩容改模数会波及号段语义；本质是"用 ID 格式换一次路由能力"，属于建模期一次性决策。对照方案是**双写两份索引表**（buyer→order、seller→order 各按各自键分片），读写都单片但写放大一倍、一致性需事务/MQ 保证——数据量与一致性要求决定选哪个。

> **设计争议**：是否所有"两个高频查询维度"都值得上基因法？反对派主张：卖家维度查询天然是运营/商家后台流量，走 ES 反而是正确的读写分离（OLTP 与检索各归各位），把路由能力硬塞进主键格式属于过度设计；支持派强调中小团队没有 ES 运维能力，基因法成本更低。两派共同前提：**主查询维度必须先定，次维度才谈补丁**。

# 2. 容量规划

```text
分片数计算公式：
  分片数 = 预估总数据量 / 单表舒适容量
  单表舒适容量 ≈ 1000万 ~ 2000万行

扩容预留：
  初始配置 N 个分片
  预留 N 个空分片（避免紧急扩容）
  使用 2 的幂次方分片数（方便后续倍增）
```

## 2.1 为什么分片数用 2 的幂

倍增扩容（8→16）时，**取模语义下旧数据的去向有干净规律：每个旧片恰好对半迁移到新片**（`x % 16` 与 `x % 8` 的关系决定搬迁集合），配合 ShardingSphere Migration 的 `MIGRATE TABLE INTO` 与一致性校验可做到在线渐进；非 2 幂（如 10→20）虽然也成对，但 %10 与 %20 的数据分布规律不如二进制对齐直观，且 INLINE 表达式手工扩片容易写错。另一派做法是**预分片（pre-sharding）**：物理表一次性建好 1024 张，初期所有"逻辑分片"映射到同几个库，扩容只改映射不改数据——迁移成本一次性付清，代价是小表碎片多、目录缓存大。选型：数据可预测增长选幂次倍增，增长陡峭且不可预测选预分片。

单表舒适容量不是玄学：以 InnoDB 为例，16KB 页 × B+ 树三层（典型行宽 1KB）可寻址约 2000 万行级别；**真正先崩的是 DDL 与局部性**——大表 pt-online-schema-change 一跑几小时、二级索引深度增加、备份窗口拉长。所以"1000-2000 万"背后还要叠加"单表数据文件 ≤ 10GB"与"单片 QPS ≤ 库容量 60%"两条硬线。

# 3. 配置管理

```yaml
# 推荐：YAML 初始部署 + DistSQL 运维

# 初始：YAML 定义完整规则
rules:
  - !SHARDING
    tables: ...
    shardingAlgorithms: ...
    keyGenerators: ...

# 运维：DistSQL 在线调整
ALTER SHARDING TABLE RULE t_order (
    DATANODES("ds${0..3}.t_order_${0..1}")
);
```

**YAML 是声明式基线，DistSQL 是命令式变更，两者必须以注册中心为唯一事实源**：Cluster 模式下 DistSQL 写入注册中心、各实例 watch 生效，YAML 只在首次 `overwrite` 或实例冷启动时灌入。纪律是"谁改、从哪改、怎么回滚"三件事写进团队规范：变更从 Proxy 走 DistSQL（有审计、有 PREVIEW 可验），YAML 仓库同步 PR 保持一致（否则下次扩容新实例按旧 YAML 起，规则被打回）；回滚手段是变更前 `EXPORT DATABASE CONFIGURATION` 存档、异常时 `IMPORT`。Standalone 模式没有 watch，YAML 改动必须重启，适合低变更频率的小规模场景，别把它误用于需要在线扩缩容的体系。

# 4. 连接池配置

```yaml
dataSources:
  ds0:
    connectionTimeoutMilliseconds: 3000
    idleTimeoutMilliseconds: 60000
    maxLifetimeMilliseconds: 1800000
    maxPoolSize: 50          # 根据 QPS 计算
    minPoolSize: 5
```

**连接数计算**：`maxPoolSize ≈ (QPS × 平均RT/1000) + 10`

分片环境要把公式升级成**二维账本**：每个应用实例的池大小 × 实例数 × 平均扇出片数 = 数据库总连接压力。例：200 实例 × 池 50 × 平均路由 2 片 = 20000 个潜在连接，MySQL `max_connections` 通常只有几千——**应用侧池"合理"不等于数据库侧总账合理**。控制手段：JDBC 侧压低 maxPoolSize（配合 `max-connections-size-per-query` 限扇出）、或改走 Proxy 做 M:1 收敛（12 篇）；数据库侧用 `SELECT COUNT(*) FROM information_schema.processlist` 与 `SHOW STATUS LIKE 'Threads_connected'` 常看水位。minPoolSize 在流量有波谷但重连成本高的场景拉高到稳态值的一半，避免整点活动冷启动时惊群建连。

# 5. JDBC vs Proxy 选择

| 场景 | 推荐 |
|------|------|
| 纯 Java 应用，性能优先 | JDBC |
| 多语言混合 | Proxy |
| 需要 DistSQL 在线变更 | Proxy |
| 需要统一连接治理 | Proxy |
| 只需分库分表 | JDBC |
| 已有 MySQL 运维经验 | Proxy（协议兼容） |

展开的判定维度与混合部署模式见 12 篇，这里只补一句业务视角：**选型不是一次定终身，但"接入形态迁移"要做在数据迁移之外的独立窗口里**。

# 6. 索引与跨片代价

```sql
-- 每个物理表独立建索引
-- 分片键在主键中 → 已覆盖

-- 非分片键的查询：考虑以下策略
-- 1. 避免（让查询包含分片键）
-- 2. 如果无法避免 → 使用 ES 等搜索引擎作为二级索引
-- 3. 广播表存储小量级数据

-- 分页查询优化：使用游标分页
SELECT * FROM t_order
WHERE order_id > ? AND user_id = ?
ORDER BY order_id LIMIT 20;
```

## 6.1 跨片查询的四种收口姿势

| 姿势 | 机制 | 代价 | 适用 |
|------|------|------|------|
| 带分片键 | 单片路由 | 无 | 主查询维度 |
| 游标分页 | `WHERE id > last` 替代 OFFSET | 无放大 | 列表翻页 |
| 异构索引 | binlog→ES/ClickHouse | 延迟 + 同步链路 | 任意维度检索 |
| 广播表/基因法 | 全片冗余 / ID 内嵌片号 | 写放大 / 格式耦合 | 小维表 / 第二维度 |

索引设计本身没有分布式特殊性（每物理表独立建），特殊的是**优化器看不到全局**：`WHERE status=? AND ctime>?` 不带分片键就是 N 片各扫本地索引再归并，N 倍 IO + N 倍传输；带分片键则退化为单片本地查询。"跨片代价"在评审时按公式估：**扇出数 ×（单片行扫描成本 + 归并读放大 offset+limit）**。深分页红线：`offset > 10000` 的分片表查询直接在代码规范/SQL 审计里拦截（SQLAuditor SPI 可挂自定义规则，见 15 篇），逼出游标方案。

> **行业实践**：绝大多数"分片后查询变慢"的工单最终归因于**没带分片键的隐式全路由**（ORM 动态 SQL 拼漏条件最常见）。治理三板斧：上线前 PREVIEW 批量预检 + 灰度期抽样 sql-show 比对 + 生产中路由单元数指标化告警（Actual SQL 条数 > 阈值即报警）。

# 7. 热点与倾斜治理

热点治理分三层，成本依次升高：**① 算法层**——对倾斜键换复合分片（`user_id + order_type` 拼接取模）让同一用户不同业务落不同片，只治表不治本；**② 数据层**——识别出的超级大客户单独"拆户"：规则里为该键值配置专用数据源（ShardingSphere 可自定义算法做例外路由），长尾继续走公共片，本质是"分片 + 独占库"混合布防；**③ 架构层**——热点读（大促爆款 SKU）走缓存/Proxy 读写分离加从库，热点写靠 MQ 削峰把瞬时脉冲摊平到片的稳态容量内。

倾斜的度量口径建议固定为：`单片行数 / 平均片行数 > 3` 或 `单片容量增速 > 平均 2 倍` 即预警，接入 ShardingSphere 的 metrics 或定时巡检脚本统计 `SELECT COUNT(*)`（低峰）。**发现扩容窗口比发现倾斜更难**，所以容量巡检报表必须把"预计触达舒适线时间"作为一等公民，而不是等到磁盘告警。

# 8. 生产配置模板

```yaml
mode:
  type: Cluster
  repository:
    type: ZooKeeper
    props:
      namespace: shardingsphere
      server-lists: zk1:2181,zk2:2181,zk3:2181

props:
  sql-show: false                       # 生产关闭 SQL 打印
  sql-simple: false
  max-connections-size-per-query: 1
  kernel-executor-size: 16              # 执行线程数
  proxy-frontend-executor-size: 32      # Proxy 前端线程
  proxy-backend-executor-suitable: true
  check-table-metadata-enabled: false   # 关闭元数据检查（性能优化）

authority:
  users:
    - user: root@%
      password: ${ROOT_PASSWORD}        # 环境变量注入
  privilege:
    type: ALL_PRIVILEGES
```

## 8.1 关键项的设计考量

- **sql-show 生产不开**：每条 SQL 的 Actual SQL 打印在高 QPS 下是显著 IO + 日志噪音，用 `sql-show-export` 抽样（Agent 可观测链路，11 篇）或临时动态开启排查
- **check-table-metadata-enabled**：开启时 DML 前逐表校验列元数据一致性，跨库 DDL 未对齐的"保险丝"，代价是每次解析多一轮元数据查询；稳态 DDL 流程受控后关闭提速，但灰度期建议开着
- **max-connections-size-per-query**：默认 1 是保守值，读多写少的报表类应用调到 CPU 核数级可提速明显，交易类保持默认
- **worker-id**：多实例 SNOWFLAKE 必须保证 worker-id 互异（ZooKeeper 模式自动从注册中心抢占分配，Standalone 需手工规划），这是分布式主键冲突的根治项（20 篇）

# 9. 监控告警

分片体系的可观测面比普通单库多了三段：**路由扇出、改写质量、分布式事务状态**，任何一段劣化都可能不体现在数据库指标上。Agent（11 篇）产出 route/rewrite/execute 三段 metrics，Prometheus 抓取后按下表配置告警基线：

```text
必须告警的场景：
□ Proxy 进程 OOM
□ 某个存储单元被 DISABLE
□ 连接数 > 80% maxPoolSize
□ SQL 解析错误持续增长
□ P99 RT 突增 2 倍以上
□ ZooKeeper 会话超时
```

**设计考量**：

- **扇出数是分片系统的一级指标**：逻辑 SQL 与 Actual SQL 条数比值突增 = 某条 SQL 丢了分片键，比数据库慢查询告警更早暴露问题
- **XA 悬挂数单独盯**：`XA RECOVER` 结果行数 > 0 持续 5 分钟即告警，prepared 事务持锁不超时，晚处理一小时等于锁表一小时
- **注册中心会话超时要区分网络抖动与真故障**：ZK session expired 后实例需重启重新注册，短 timeout 报警 + 长失联自动重启动作要分开
- **Proxy 堆外内存（Netty direct）独立监控**：只盯 heap 会漏掉 ByteBuf 泄漏，`-XX:MaxDirectMemorySize` 与实际用量水位都要有指标

# 10. DON'T

```text
✗ 不要用分片键以外的列做高频全表扫描
✗ 不要在 YAML 中写死 IP（用环境变量或配置中心）
✗ 不要在生产环境 sql-show=true（影响性能）
✗ 不要在建表后频繁改分片规则（数据迁移成本高）
✗ 不要用 OFFSET 深度分页（用游标）
✗ 不要忘记定期备份注册中心元数据
✗ 不要单点部署 Proxy（至少 2 实例 + LB）
```

补三条同样高频：**不要在未验证 SQL 解析覆盖度前迁移存量复杂 SQL**（先 PARSE/PREVIEW 全量预检，解析失败会静默降级直通，15 篇）；**不要在分片表上保留数据库 AUTO_INCREMENT 依赖**（跨片重号/空洞语义变化，14 篇）；**不要跨形态拼分布式事务**（Java 线程本地事务无法与经 Proxy 的调用链组成同一个 XA，12 篇）。

# 11. 上线 Checklist

```text
□ 建模评审：分片键不可变/区分度报告/双维度是否需要基因法或异构索引
□ 预检：全量业务 SQL PREVIEW，路由单元数 = 1 的占比 ≥ 95%
□ 预检：SQL 解析零失败（PARSE 批跑），深 OFFSET 用例清零
□ 容量：分片数按 3 年增长 + 2 幂预留；单片 ≤ 舒适线 60%
□ 倾斜：TOP 键值分布复核，超级客户拆户规则就位
□ 主键：SNOWFLAKE worker-id 唯一性（Cluster 自动/Standalone 手工表）
□ 事务：跨库事务清单与类型（LOCAL/XA/BASE）逐场景标注，无跨形态拼事务
□ 连接：二维账本核算（实例×池×扇出 ≤ DB max_connections 70%）
□ 高可用：Proxy ≥ 2 实例 + LB，ZK 3 节点，元数据导出备份定时任务
□ 观测：路由扇出数/解析失败数/连接水位/XA 悬挂 四类告警上线
□ 回滚：EXPORT 配置存档、双写/迁移的回切脚本演练通过
```

> **行业实践**：把 checklist 里前两条（PREVIEW 覆盖 + 单片路由占比）做成 CI 化脚本——存量 SQL 资产从 ORM 导出跑批比对，上线准入以"路由报告"签字为准。分片事故复盘里九成能追溯到"预检缺失"，这一步的 ROI 高于任何调参技巧。

# 12. 小结

1. **分片键是唯一不可在线修改的决策**，SQL 画像 + 不可变 + 区分度 + JOIN 对齐四律缺一不可
2. **双维度用基因法或双份索引**，主维度定生死、次维度靠补丁
3. **扩容数学提前定**：2 幂倍增或预分片，触达舒适线的时间要进容量报表
4. **连接账本是二维的**（实例×池×扇出），扇出与深分页是两大隐形税
5. **上线以 PREVIEW 路由报告为准入门禁**，分片事故九成死在建模与预检缺失