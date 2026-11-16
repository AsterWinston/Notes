# 1. 分区表

当一张表的数据量达到几千万甚至上亿行，索引变高、全表扫描变慢、维护窗口不够用，**分区（Partitioning）** 是最直接的应对手段。它的核心思想是**把一张大表逻辑上当成一张表，物理上拆成多个独立段**，查询时通过分区键做**分区裁剪（Partition Pruning）**，只读相关分区。

需要先说明一点：Oracle 的分区是**付费选件（Partitioning Option）**，虽然 12c 之后部分版本把基础分区下放，但企业版生产环境仍要确认授权。MySQL、PostgreSQL 的分区是免费的，这是选型时绕不开的成本差异。

## 1.1 分区类型

| 类型 | 依据 | 典型场景 |
|------|------|---------|
| RANGE | 列值范围 | 按时间分区，最常用 |
| LIST | 枚举值列表 | 按地区、状态、租户分区 |
| HASH | 哈希函数 | 无自然分区键，追求均匀分布 |
| INTERVAL | 范围 + 自动建分区 | 按时间自动扩展，免手工加分区 |
| 复合分区 | 两种组合 | RANGE-LIST、RANGE-HASH 等 |
| REFERENCE | 外键关联 | 子表跟随父表分区 |
| 虚拟列分区 | 表达式结果 | 按 `created_at` 的月份分区 |

## 1.2 范围分区

```sql
CREATE TABLE orders (
    id         NUMBER(19)    NOT NULL,
    user_id    NUMBER(19)    NOT NULL,
    amount     NUMBER(18, 2) NOT NULL,
    created_at DATE          NOT NULL
)
PARTITION BY RANGE (created_at) (
    PARTITION p2025 VALUES LESS THAN (DATE '2026-01-01'),
    PARTITION p2026_q1 VALUES LESS THAN (DATE '2026-04-01'),
    PARTITION p2026_q2 VALUES LESS THAN (DATE '2026-07-01'),
    PARTITION pmax VALUES LESS THAN (MAXVALUE)
);
```

`MAXVALUE` 分区是兜底，防止插入超出范围的数据时报错。但兜底分区会破坏"分区裁剪"的收益，应配合定期 `SPLIT PARTITION` 把它拆开。

```sql
-- 把兜底分区拆出一个新分区
ALTER TABLE orders SPLIT PARTITION pmax AT (DATE '2026-10-01')
INTO (PARTITION p2026_q3, PARTITION pmax);
```

## 1.3 间隔分区（自动扩展）

```sql
CREATE TABLE logs (
    id         NUMBER,
    created_at DATE NOT NULL
)
PARTITION BY RANGE (created_at)
INTERVAL (NUMTOYMINTERVAL(1, 'MONTH')) (
    PARTITION p_init VALUES LESS THAN (DATE '2026-01-01')
);
```

插入 `2026-02-15` 的数据时，Oracle 会自动创建对应月份的分区，不需要 DBA 手工维护。这是 11g 之后最推荐的时间分区写法。

## 1.4 分区裁剪

分区裁剪是分区最大的收益来源：查询条件命中分区键，优化器只访问相关分区。

```sql
-- 能裁剪：只扫 p2026_q1
SELECT * FROM orders
WHERE created_at >= DATE '2026-02-01' AND created_at < DATE '2026-03-01';

-- 不能裁剪：对分区键用了函数，优化器无法判断范围
SELECT * FROM orders WHERE TO_CHAR(created_at, 'YYYY-MM') = '2026-02';

-- 改写为范围条件后可以裁剪
SELECT * FROM orders
WHERE created_at >= DATE '2026-02-01' AND created_at < DATE '2026-03-01';
```

```sql
-- 查看执行计划里的分区裁剪信息
EXPLAIN PLAN FOR
SELECT * FROM orders WHERE created_at >= DATE '2026-02-01';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL, NULL, 'BASIC +PARTITION'));
```

> **注意**：分区键上做函数、隐式转换，都会让裁剪失效，退化成全分区扫描。这和普通索引失效是同一类问题，但后果更严重——扫描的是多个分区的全部数据。

## 1.5 本地索引与全局索引

| 索引类型 | 结构 | 优点 | 缺点 |
|---------|------|------|------|
| 本地索引（Local） | 每个分区一个索引分区，与表分区一一对应 | 分区维护简单，裁剪后只读对应索引 | 唯一性约束必须包含分区键 |
| 全局索引（Global） | 一个索引跨所有表分区 | 可对非分区键建唯一索引 | 分区维护会导致全局索引失效 |

```sql
-- 本地索引
CREATE INDEX idx_orders_created_local ON orders(created_at) LOCAL;

-- 全局索引
CREATE INDEX idx_orders_user_global ON orders(user_id) GLOBAL;
```

> **行业实践**：时间分区表的主查询通常带时间条件，用本地索引最省心；如果业务大量按 `user_id` 查询，而 `user_id` 不是分区键，就需要全局索引，并接受"分区维护后重建索引"的代价（可用 `UPDATE GLOBAL INDEXES` 减少失效）。

## 1.6 分区维护

```sql
-- 新增分区
ALTER TABLE orders ADD PARTITION p2026_q4 VALUES LESS THAN (DATE '2027-01-01');

-- 删除历史分区（比 DELETE 快得多，直接释放段）
ALTER TABLE orders DROP PARTITION p2025;

-- 交换分区：把普通表的数据"换"进分区，适合批量加载
ALTER TABLE orders EXCHANGE PARTITION p2026_q1 WITH TABLE stage_orders;
```

`DROP PARTITION` 和 `TRUNCATE PARTITION` 都是 DDL，速度快、不产生大量 undo，是分区表做数据生命周期管理的主要手段。这比 `DELETE FROM ... WHERE created_at < ...` 高效得多。

# 2. RAC

**RAC（Real Application Clusters）** 让多个实例通过共享存储同时打开同一个数据库。它的目标是**高可用 + 横向扩展计算能力**，不是"把数据复制多份"。

```text
        ┌──────────────┐      ┌──────────────┐
        │  实例 Node1  │      │  实例 Node2  │
        │  SGA + 进程  │◀────▶│  SGA + 进程  │
        └──────┬───────┘ 内联 └──────┬───────┘
               │   Cache Fusion       │
               └──────────┬───────────┘
                          ▼
                ┌──────────────────┐
                │   共享存储         │
                │ 数据文件/日志/OCR │
                └──────────────────┘
```

## 2.1 核心组件

| 组件 | 作用 |
|------|------|
| Clusterware（GI） | 集群管理、节点监控、资源调度 |
| OCR | 集群注册表，存集群配置 |
| Voting Disk | 投票盘，用于脑裂判断 |
| Cache Fusion | 通过内联网络在实例间传输数据块 |
| GCS / GES | 全局缓存服务 / 全局队列服务，管理块与锁 |
| SCAN | 单一客户端访问名，把连接分发到各节点 |
| VIP | 虚拟 IP，节点故障时漂移到其他节点 |
| Service | 把工作负载绑定到指定节点，便于管理 |

## 2.2 Cache Fusion 的代价

RAC 最容易被误解的地方是"加了节点就一定更快"。Cache Fusion 让一个实例可以读取另一个实例缓存中的块，但这个跨节点传输依赖**内联网络（Interconnect）**，延迟远高于本地内存。如果业务是"多个节点频繁读写同一批热块"，跨节点争用会让性能不升反降。

```text
节点 1 修改块 A → 块 A 在节点 1 的 Buffer Cache 中变为最新
节点 2 要读块 A → 通过内联向节点 1 请求 → 节点 1 传输块 → 节点 2 才能读
```

> **行业实践**：RAC 适合"**读多写少、或不同节点访问不同数据分区**"的场景。序列主键、频繁更新的统计表、热点账户，都是 RAC 的典型痛点。建 RAC 前先问：业务能不能按用户/租户/地区拆分到不同节点？如果所有节点都抢同一批行，RAC 帮不上忙。

## 2.3 RAC 与"高可用"的边界

RAC 能扛住**单节点故障**：节点挂了，VIP 漂移、服务在其他节点拉起、连接通过 TAF 重连。但它**扛不住存储故障**——所有节点共享同一份存储，存储坏了整个集群一起完蛋。所以生产上的高可用通常是 **RAC + Data Guard**：RAC 解决节点级故障，Data Guard 解决站点级/存储级故障。

# 3. Data Guard

**Data Guard** 通过**重做日志传输**把主库的变更同步到备库，形成主备架构。它是 Oracle 最核心的容灾方案，和 RAC 是互补关系，不是替代关系。

```text
┌──────────┐   redo 传输   ┌──────────┐
│  主库     │──────────────▶│  备库     │
│ Primary  │   (SYNC/ASYNC) │ Standby  │
└──────────┘                └──────────┘
      │                           ▲
      │                           │ Redo Apply
      └──── 归档日志 / 实时传输 ────┘
```

## 3.1 备库类型

| 类型 | 同步方式 | 是否可读 | 适用 |
|------|---------|---------|------|
| 物理备库（Physical） | 块级重做应用 | 默认不可读，Active Data Guard 可读 | 容灾、读写分离 |
| 逻辑备库（Logical） | SQL 应用 | 可读可写（有限） | 滚动升级、部分表复制 |
| 快照备库（Snapshot） | 临时转为可写 | 可读写 | 临时测试、演练 |

物理备库是绝对主流：它按块同步，主备完全一致，切换快、运维简单。逻辑备库维护成本高，只在特定场景用。

## 3.2 保护模式

| 模式 | 行为 | 数据丢失风险 | 性能影响 |
|------|------|-------------|---------|
| 最大保护（Maximum Protection） | 主库提交前必须收到备库确认 | 零丢失 | 最大，备库不可用则主库停写 |
| 最大可用（Maximum Availability） | 正常时同步，备库不可用时降级为异步 | 极端情况可能少量丢失 | 中等，推荐 |
| 最大性能（Maximum Performance） | 默认异步传输 | 可能丢失部分事务 | 最小 |

> **设计考量**：金融核心系统常用"最大可用"，在"零丢失"和"可用性"之间取平衡；一般业务用"最大性能"，接受分钟级 RPO。选模式前先和业务确认 **RPO（能丢多少数据）** 和 **RTO（多久能恢复）**，而不是直接选"最安全的"。

## 3.3 切换与故障转移

| 操作 | 触发条件 | 特点 |
|------|---------|------|
| Switchover | 计划内，主备角色互换 | 无数据丢失，可双向 |
| Failover | 主库故障，备库强制接管 | 可能丢数据，原主库需重建 |

```sql
-- 查看备库状态与延迟
SELECT database_role, open_mode, protection_mode FROM v$database;
SELECT name, value, unit FROM v$dataguard_stats;

-- 查看归档日志传输情况
SELECT dest_id, status, applied_sequence#, gap_status
FROM v$archive_dest_status WHERE dest_id = 2;
```

**Fast-Start Failover（FSFO）** 配合 Observer 进程，可以在主库故障时自动触发 failover。但自动切换有风险：网络抖动可能导致误判，所以需要 Observer 部署在第三方站点，并配置合理的阈值。

# 4. 读写分离

Oracle 的读写分离和 MySQL 主从思路类似，但手段不同：

| 方案 | 原理 | 延迟 | 一致性 | 成本 |
|------|------|------|--------|------|
| Active Data Guard | 物理备库实时应用，只读打开 | 秒级 | 最终一致 | 需 ADG 授权 |
| GoldenGate | 逻辑复制，可跨异构库 | 毫秒到秒级 | 最终一致 | 额外授权 |
| 物化视图 | 定时刷新 | 分钟级 | 最终一致 | 免费 |
| 应用层双写 | 应用同时写多个库 | 取决于实现 | 难保证 | 开发成本高 |

**Active Data Guard（ADG）** 是最常用的方案：备库在应用 redo 的同时对外提供只读查询，报表、大查询、数据导出都可以走备库，减轻主库压力。

> **注意**：读写分离最大的坑是**读到自己刚写的数据**。主库写入后，备库可能还没应用完，应用立刻去备库查会查不到。解决办法是"写后读走主库"、"按业务容忍度选择读库"，或者用 ADG 的 `DBMS_DG` 等待应用到位（会牺牲延迟）。

# 5. 故障转移

故障转移要分层看：

```text
客户端
  │ 1. 连接层：SCAN / VIP / TNS 多地址
  ▼
监听器
  │ 2. 服务层：Service + TAF / FAN
  ▼
实例
  │ 3. 节点故障：RAC 接管
  ▼
存储/站点
  │ 4. 站点故障：Data Guard failover
  ▼
数据库
```

## 5.1 TAF 与 FAN

- **TAF（Transparent Application Failover）**：连接断开后自动重连到可用节点，可选"只重连"或"重连并重放查询"。
- **FAN（Fast Application Notification）**：数据库主动通知应用节点状态变化，应用可更快响应。
- **Application Continuity**：12c 之后的能力，能在连接中断后重放事务，对应用几乎透明，但需要配置和授权。

> **行业实践**：不要只依赖 TAF。客户端连接串应配置多个地址（`ADDRESS_LIST`），应用侧配好重试与超时。真正的故障转移能力，要靠**定期演练**验证，而不是看架构图画得漂亮。

## 5.2 演练清单

- 拔掉 RAC 一个节点，观察 VIP 漂移、服务切换、应用重连时间。
- 主库执行 `ALTER DATABASE COMMIT TO SWITCHOVER`，验证 switchover 无数据丢失。
- 模拟主库宕机，手动 failover，记录 RTO 与数据丢失量。
- 验证备库只读查询在切换后是否正常。
- 检查告警日志、Data Guard Broker 状态、归档 gap 是否自动修复。

# 6. 选型

| 需求 | 推荐方案 | 说明 |
|------|---------|------|
| 单机高可用 | 单实例 + Data Guard | 成本最低，能扛站点故障 |
| 节点级高可用 + 计算扩展 | RAC | 需要共享存储与内联网络 |
| 容灾 + 读写分离 | RAC + ADG | 金融、电信常见组合 |
| 跨异构数据库同步 | GoldenGate | 迁移、双跑、数据汇聚 |
| 只读报表 | ADG 或物化视图 | 按实时性要求选 |
| 预算有限 | 单实例 + 备份 + 定期演练 | 高可用不等于必须 RAC |

> **设计考量**：架构复杂度要和团队能力匹配。RAC 的运维门槛（集群、OCR、投票盘、Cache Fusion 调优）远高于单实例。一个没有 RAC 运维经验的团队上 RAC，故障时的恢复时间可能比单实例还长。**能演演练、能恢复的简单架构，优于图纸上完美但没人会修的复杂架构**。

# 7. 练习

- **练习一**：创建按 `created_at` 月份做间隔分区的表，插入跨 3 个月的数据，用 `DBMS_XPLAN` 的 `+PARTITION` 查看分区裁剪；再改成 `TO_CHAR(created_at, 'YYYY-MM') = ?`，观察裁剪失效并解释原因。
- **练习二**：画出"RAC + Data Guard"架构图，标出 Cache Fusion、redo 传输、归档应用三条链路，说明单节点故障与站点故障分别由哪层接管。
- **练习三**：对比"最大保护"与"最大性能"两种 Data Guard 模式，从 RPO、RTO、主库可用性三个维度列一张表，并给出一个业务场景的选型理由。

# 8. 小结

| 主题 | 核心结论 |
|------|---------|
| 分区 | 逻辑一张表、物理多段，靠分区裁剪减少扫描 |
| 分区类型 | 时间用 RANGE/INTERVAL，枚举用 LIST，均匀分布用 HASH |
| 本地/全局索引 | 本地易维护，全局支持非分区键唯一索引 |
| RAC | 多实例共享存储，靠 Cache Fusion，适合节点级高可用 |
| Data Guard | redo 传输的主备容灾，物理备库为主流 |
| 保护模式 | 最大保护/最大可用/最大性能，按 RPO/RTO 选 |
| 读写分离 | ADG 最常用，注意主备延迟导致的读不一致 |
| 选型 | 架构复杂度匹配团队能力，定期演练比图纸重要 |

高可用解决的是"坏了怎么继续提供服务"，但**数据本身丢了怎么办**？下一篇进入**备份与恢复**，讲清归档模式、RMAN 全备与增量、PITR 的原理与操作，以及为什么"备份成功"不等于"能恢复"。
