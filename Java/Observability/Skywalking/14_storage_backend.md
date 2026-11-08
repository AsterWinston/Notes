
# 1. 存储后端全景

存储是 SkyWalking 架构里**唯一有状态**的组件：OAP 无状态可随便扩，UI 无状态可随便扩，唯有存储的写入能力、查询能力和数据保留成本，共同决定了整套系统的天花板与预算。SkyWalking 把存储抽象成一层 SPI（`StorageModule`），理论上一个实现就能对接任意后端，官方维护的实现有 **H2、MySQL/PostgreSQL（SQL 系）、Elasticsearch（7.x+/8.x）、Infinispan 与 BanyanDB**，各版本还支持程度不同。选型前要先理解一个反直觉的事实：**APM 的数据不是"日志"也不是"业务表"，它是时序数据 + 高基数标签 + 点查 Trace 的混合负载**，这决定了各后端的天然优劣。

## 1.1 五后端对照表

| 维度 | H2 | MySQL/PostgreSQL | Elasticsearch | Infinispan | BanyanDB |
|------|----|------------------|---------------|------------|----------|
| **生产推荐** | 否 | 中小规模可用 | 大规模成熟之选 | 已边缘化 | **官方主推** |
| **定位** | 内存/内嵌库 | 关系型 | 倒排索引检索 | 内存数据网格 | **APM 原生时序库** |
| **写入性能** | 低 | 中 | 高 | 高（内存） | 极高 |
| **查询性能** | 低 | 中（靠索引） | 高（聚合强） | 中 | 高（时序查询强） |
| **水平扩展** | ✗ | 有限（读写分离/分库） | ✓ | ✓ | ✓ |
| **运维复杂度** | 极低 | 低 | 中偏高 | 中 | 低 |
| **资源消耗** | 低 | 中 | 高（JVM 堆大户） | 高（全内存） | 中 |
| **数据持久性** | 重启可丢 | 强 | 强 | 取决于配置 | 强 |
| **适用场景** | 开发/演示 | 小规模、已有 DBA | Trace 量大 + 需要检索 | 早期内存方案 | 新项目默认 |

五者的角色差异远大于参数差异，逐个看：

**H2** 是内嵌库，OAP 启动即带，零部署成本，但容量、并发与持久性都撑不住生产——它存在的意义是让 Quick Start 一条命令跑起来。**MySQL** 把观测数据当普通表写，胜在团队熟悉、事务可靠，败在表膨胀：Trace 与指标是纯 append 负载，单表每天数百万上千万行，索引维护与清理都是 DBA 的噩梦，官方文档也明确 SQL 系只建议中小规模。**Elasticsearch** 曾经长期是生产默认：写入吞吐高、聚合查询强、集群扩展成熟，缺点是大集群的内存成本与运维门槛，以及"通用索引模型跑 APM 负载"带来的映射与分片治理问题。**Infinispan** 是内存数据网格方案，用集群内存换写入速度，但容量受内存上限约束、持久化配置复杂，近年社区投入明显减少，新部署基本不再推荐，老用户建议迁向 BanyanDB。**BanyanDB** 是 SkyWalking 团队**为自身数据模型从零设计的时序库**，下一节展开。

## 1.2 BanyanDB 为什么被官方主推

**BanyanDB 的核心竞争力不是"更快的通用存储"，而是"与 OAP 数据模型完全同构的存储"**，这带来四点结构性优势：

- **时序模型匹配**：SkyWalking 的指标本质是"实体（服务/实例/端点）× 时间窗口"的序列，BanyanDB 以 Group/Series/Block 的组织方式原生表达，一个实体的指标聚在同一时间线序列中连续存放，指标查询退化为"按序列定位 + 顺序扫描"，不需要 ES 那种倒排聚合。
- **schema-on-write 而非动态映射**：指标在 OAL 编译期就确定了字段与类型，BanyanDB 的 measure/record 模型与之对应，写入路径没有 ES 的映射推断开销，也天然规避映射爆炸（见 3.3 节）。
- **Trace 按段流（segment set）存储**：Trace 是典型的"写一次、按 TraceId 点查、按时间范围批量清理"负载，BanyanDB 按时间分片存储段数据，点查走索引、过期 TTL 直接**按时间分片整体删除**，比 ES 删索引或 MySQL 逐行 DELETE 干净得多。
- **资源账**：官方基准的口径是同等负载下显著低于 ES 的资源占用（内存量级差距常见 50% 以上表述），且它专为 SkyWalking 设计，不需要容量规划师额外调优 JVM 堆、分片数这类与业务无关的参数。

> **注意**：BanyanDB 由 SkyWalking 社区孵化、CNCF Sandbox 托管，尚年轻，其强项是"跑 SkyWalking（及 OTLP 时序负载）"，不适合当通用文档库使用；若组织已有深度绑定的 ES 生态（日志检索、安全分析共用一套集群），继续用 ES 完全合理——**存储选型一半是技术，一半是组织已有技能的复用**。

# 2. 数据生命周期：TTL 与分索引

无论选哪个后端，SkyWalking 存储侧的第一治理对象都是**数据生命周期**。OAP 把数据分成两大类，各自独立 TTL：

```text
metricsDataTTL  —— 聚合指标（分钟/小时/天粒度），UI 仪表盘与告警的数据源
recordDataTTL   —— 原始记录：Trace 段(Segment)、日志、Agent 事件，体量大、点查为主
```

**两类数据价值衰减曲线完全不同**：指标是聚合产物，体量小（一条只有几十字节），承载趋势分析与容量规划，留久一点成本不高；Trace 是原始明细，单条动辄数 KB，绝大多数 Trace 只在出问题的当天有查看价值。因此典型配置是 **Trace 保 3~7 天、指标保 30 天**，重要集群可把慢 Trace/错误 Trace 单独延长（依赖采样侧配合，见最佳实践篇）。

分索引（rolling index）是 TTL 的执行机制：OAP 按天（`dayStep`）为每类数据创建带日期后缀的索引/表，如 `sw_segment-20261108`、`sw_metrics-all-20261108`。过期清理变成**整天粒度的 DROP**，写入永远只打当天新索引，避免"一个大索引无限膨胀 + 原地删除"的两种反模式。理解这一点，才能理解为什么各后端的调优参数都在围绕"索引/表的数量与大小"做文章。

> **设计考量**：TTL 不是越短越省钱。TTL 决定的是**可回溯排障的窗口**与**容量规划基线**——把 recordDataTTL 从 3 天压到 1 天省下的存储，可能远小于一次"周一复盘上周故障查不到 Trace"的损失；反过来把 Trace 留 90 天几乎从不值得。先用"故障复盘最多回看几天"倒推 TTL，再乘流量估容量。

# 3. Elasticsearch 配置与治理

## 3.1 接入配置

```yaml
# config/application.yml (9.x/10.x 实际键名；TTL 统一放在 core.default)
storage:
  selector: elasticsearch
  elasticsearch:
    namespace: skywalking
    clusterNodes: es1:9200,es2:9200,es3:9200
    protocol: http
    user: elastic
    password: ${ES_PASSWORD}
    connectTimeout: 5000
    socketTimeout: 30000
    indexReplicasNumber: 1
    indexShardsNumber: 2
    dayStep: 1             # 每天新索引
    superDatasetIndexShardsFactor: 5   # Trace 这类超大数据集的分片放大系数
    bulkActions: 5000      # 批量写入大小
    batchOfBytes: 10485760 # 单批字节上限(10MB)
    flushInterval: 5       # 刷新间隔(秒)
    concurrentRequests: 2  # 并发请求数

core:
  default:
    recordDataTTL: 7       # Trace 数据保留天数
    metricsDataTTL: 30     # 指标数据保留天数
```

`indexShardsNumber × (1 + indexReplicasNumber)` 决定了每天每个逻辑表实际产生的分片数，ES 集群规模应按**单个分片 10~50GB 的健康区间**反推：日增 Trace 大时优先加节点/加 shard，而不是加副本。

## 3.2 索引模板与清理

```bash
# 提前创建 Index Template，避免 OAP 首写时动态创建导致分片数失控
curl -X PUT "localhost:9200/_index_template/skywalking-template" \
  -H "Content-Type: application/json" \
  -d '{
    "index_patterns": ["sw_*"],
    "template": {
      "settings": {
        "number_of_shards": 2,
        "number_of_replicas": 1,
        "refresh_interval": "30s",
        "codec": "best_compression"
      }
    }
  }'
```

```bash
# 常规巡检：按大小排索引，找异常膨胀的表
curl "localhost:9200/_cat/indices/sw_*?v&s=store.size:desc"
# 应急清理过期索引（正常情况由 OAP TTL 任务自动执行）
curl -X DELETE "localhost:9200/sw_segment-20261025"
```

把 `refresh_interval` 调大（如 30s）能显著降低写入放大，代价是新数据可见延迟——观测数据可以接受。若集群启用 ILM（Index Lifecycle Management），可关闭 OAP 的 TTL 删除职责，改由 ILM 统一做 hot→warm→delete，避免两套过期逻辑打架。

## 3.3 映射爆炸的治理

ES 跑 SkyWalking 最常见的两类"失控"都与 mapping/查询模型有关：

- **动态映射失控**：自定义 Meter、Profiling 快照等字段若走动态推断，`long` 会被推成 `text`（附带 norms 与 doc_values），索引大小与聚合成本翻倍。治理方式是让 OAP 通过模板预置严格 mapping（数值字段 `index:false` 或强制 `keyword`），禁止动态扩展。
- **高基数字段的聚合爆炸**：端点名、实例 ID、TraceId 都是高基数 `keyword`，`terms` 聚合依赖全局序数表（global ordinals），端点爆炸（未归一化的 RESTful 路径）会直接反映为序数表膨胀与查询超时；对分词 `text` 字段排序聚合更会触发 fielddata 进堆。**存储层救不了脏数据，必须在 Agent/OAL 层把基数压下来**（路径归一化、忽略端点、控制自定义指标的 label），这也是"存储对查询能力的影响"最典型的体现。
- 附带一提：查询时间范围越宽、`nested`/深分页用得越多，ES 的协调开销越大，UI 上"慢查询"多半是**查询模型不适配倒排索引**而非集群太小——用合理的 TTL + 按天索引把查询范围约束住，比堆硬件有效。

# 4. MySQL 配置

```yaml
# 9.x/10.x 实际键名（TTL 统一在 core.default，见第 2 节）
storage:
  selector: mysql
  mysql:
    properties:
      jdbcUrl: jdbc:mysql://mysql-host:3306/skywalking?useSSL=false&serverTimezone=UTC&rewriteBatchedStatements=true
      dataSource.user: skywalking
      dataSource.password: ${MYSQL_PASSWORD}
      dataSource.cachePrepStmts: true
      dataSource.useServerPrepStmts: true
    maxSizeOfBatchSql: 2000             # 批量 SQL 上限
    asyncBatchPersistentPoolSize: 4     # 异步批量写入线程
```

SQL 模式下 OAP 会按启动脚本自动建表（`config/` 下按数据库分发 `create-table.sql` 风格脚本，MySQL 与 PostgreSQL 方言不同），指标按时间窗聚合后写宽表，Trace 段写入 `segment` 表。**MySQL 的结构性瓶颈在于：时间范围查询依赖 `(实体, 时间)` 联合二级索引，写入依赖 B+ 树随机插入，TTL 清理是逐行 DELETE**——DELETE 大表产生 undo 与主从延迟，一般要配合按天分表 + `DROP TABLE`，或用 `pt-archiver` 异步搬迁。因此经验值口径大致是：**日 Trace 千万条以内、并发查询用户少（内部平台）时可用**；再往上，要么换 ES/BanyanDB，要么接受为 SQL 后端配备专门的清表与索引维护流程。

> **行业实践**：坚持用 MySQL 的团队通常把 SkyWalking 定位为"轻量版：看拓扑、看指标、偶尔查 Trace"，同时依赖 `sample_n_per_3_secs` 强降采样把写入量压到数据库舒适区——存储选型与采样策略从来是联动决策。

# 5. BanyanDB 与 Infinispan 配置

BanyanDB 以独立服务（或 sidecar 同 Pod）部署，OAP 侧只需：

```yaml
# config/application.yml (10.x 实际键名；TTL 统一在 core.default)
storage:
  selector: banyandb
  banyandb:
    targets: banyandb:17912        # server gRPC 地址(多个逗号分隔)
    maxBulkSize: 5000              # 批量写入条数
    flushInterval: 15              # 刷新间隔(秒)
    concurrentWriteThreads: 15     # 并发写线程
    metricsShardsNumber: 1         # 指标分片数
    recordShardsNumber: 1          # 记录分片数
    superDatasetShardsFactor: 2    # Trace 超大数据集分片放大
    blockIntervalHours: 24         # block 生成周期
    segmentIntervalDays: 1         # segment(时间片)生成周期
```

BanyanDB 自身用 `banyandb-server` 无状态前端 + 存储组的形态部署，K8s 上有官方 chart，磁盘建议 NVMe——它的压缩列存对 IO 延迟敏感。Infinispan 配置略（`selector: infinispan`），仅在维护存量部署时参考：指定服务器列表、用户鉴权与 cache 模板，数据持久化依赖其 cache-store，容量规划等于内存规划。

# 6. 存储选型决策

```text
每日 Trace 量?
  ├── < 100 万          → MySQL / H2（演示）
  ├── 100 万 ~ 1000 万  → BanyanDB / Elasticsearch
  └── > 1000 万         → BanyanDB（写优化）或 ES 集群

组织技能栈?
  ├── 有 ES 运维经验、日志平台已用 ES → Elasticsearch
  ├── 只有 MySQL DBA、规模小          → MySQL
  ├── 不想引入新运维面                → BanyanDB（与 OAP 同生命周期管理）
  └── 存量 Infinispan 部署            → 评估迁移 BanyanDB，不建议新上
```

**存储直接决定查询能力的上限**，选型时按"你要在 UI 上做什么"倒推：高频做端点级聚合下钻、按 tag 检索 Trace → ES 或 BanyanDB 强；只做趋势看盘 + 精确 TraceId 点查 → MySQL 也够；要对日志做全文检索级分析 → 那就不是 SkyWalking 存储层的职责，接 ES 日志生态更合适。

# 7. 最佳实践

1. **TTL 分层**：Trace（recordDataTTL）保 3~7 天，指标（metricsDataTTL）保 30 天，用故障复盘窗口倒推而不是拍脑袋
2. **新项目默认 BanyanDB**：时序模型匹配、资源占用低、TTL 按分片删除干净
3. **ES 治理三板斧**：预置索引模板控制分片、调大 refresh_interval、把高基数治理前移到 Agent/OAL 层
4. **MySQL 定期清理**：按天分表 + DROP，避免逐行 DELETE 拖垮主从
5. **Bulk 参数对齐吞吐**：`batchActions`/`flushInterval`/`concurrentRequests` 三者联动，压测确定组合而非单点调大
6. **监控存储本身**：ES 集群健康、BanyanDB 磁盘水位、MySQL 连接池都要纳入告警，存储挂了 = 观测失明
7. **容量测算先行**：按"日 Trace × 平均段大小 × TTL"估算原始量，再乘以索引/副本放大系数（ES 经验 2~3 倍）规划磁盘

# 8. 小结

| 后端 | 一句话定位 |
|------|-----------|
| H2 | 演示专用，生产禁用 |
| MySQL/PostgreSQL | 中小规模 + 强 DBA 文化的团队可用，瓶颈在表膨胀与清理 |
| Elasticsearch | 大规模成熟方案，代价是资源与映射/聚合治理 |
| Infinispan | 内存换速度，社区边缘化，存量迁移对象 |
| BanyanDB | APM 原生时序库，9.x/10.x 官方主推，新项目默认 |

选型的本质是匹配负载形态：**SkyWalking 的数据是"时序聚合 + 明细点查 + 按时间过期"，谁对这个模式表达得最自然，谁就是对的存储**——这正是 BanyanDB 出现的理由。
