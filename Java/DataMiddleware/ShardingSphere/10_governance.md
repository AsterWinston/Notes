治理与弹性伸缩是 ShardingSphere 从"客户端库"走向"数据库平台"的关键：Cluster 运行模式让多个 JDBC/Proxy 实例共享同一份元数据、任一节点改动全网生效；Migration Pipeline 让"在线扩容、异构迁库"这类原本要停机导数据的操作变成可控、可校验、可回滚的任务。本篇先讲 Mode 的三种运行形态与注册中心，再讲元数据如何持久化与协调，最后重点讲弹性伸缩——以及它背后一段重要的演进史（治理中心/ElasticJob 的退场与 Pipeline 的上位）。

# 1. 运行模式演进：从 Governance Center 到 Mode

理解 5.5.x 的治理，必须先厘清一段被大量过时教程混淆的历史：

| 阶段 | 机制 | 现状 |
|------|------|------|
| 4.x | **Governance Center**（治理中心）：注册中心 + 状态/配置/熔断，内嵌 ElasticJob 做协调 | 已废弃 |
| 5.0 | 重构为 **Mode（运行模式）**：Standalone / Cluster，元数据持久化到 `repository` | 现行框架 |
| 5.0–早期 | 弹性伸缩/定时任务依赖 **ElasticJob**（ShardingSphere 曾把它做成 shardingsphere-elasticjob） | 已剥离 |
| 5.1+ | **Migration Pipeline**（DistSQL `MIGRATE TABLE`）取代 ElasticJob 成为在线数据迁移主力 | 现行方案 |
| 2022 | **ElasticJob 独立从 Apache 孵化器毕业为顶级项目**，与 ShardingSphere 解耦为"各自独立但可搭配"的两个 Apache 项目 | 独立 TLP |

> **注意**：题目里"governance 并入 elasticjob"的说法方向反了——真实演进是 **ShardingSphere 把治理中心能力内化为 Mode，并把 ElasticJob 从自身数据迁移路径中移除**，ElasticJob 反过来毕业成了独立 Apache 顶级项目。二者现在是"平台 + 调度"的可组合关系，而非从属。写笔记、面试都别答反。

Mode 是"要不要中心协调"的开关：Standalone 一切本地/自带库，Cluster 才有跨实例元数据同步与在线治理。

# 2. Mode — 三种运行模式

## 2.1 Standalone（默认，无外部依赖）

```yaml
mode:
  type: Standalone
  repository:
    type: JDBC            # 元数据存内嵌 H2，或外挂 MySQL
```

Standalone 用 JDBC 仓库把元数据落到内嵌 H2（默认 `jdbc:h2:mem:config`，**内存库重启即失，配置仍要随应用/YAML 提供**；换成文件模式或外挂 MySQL 才能持久化）。适合开发、单 Proxy 实例、或配置完全走 YAML 的场景。外挂 MySQL 持久化：

```yaml
mode:
  type: Standalone
  repository:
    type: JDBC
    props:
      provider: MySQL                       # 可选 H2/MySQL/EmbeddedDerby/DerbyNetworkServer/HSQLDB
      jdbc_url: jdbc:mysql://localhost:3306/shardingsphere
      username: root
      password: root
```

> **注意**：Standalone 的 props 键名是 `provider`/`jdbc_url`（下划线风格，对齐仓库 SPI 文档），与数据源配置里的 `jdbcUrl` 不同；用 MySQL provider 时 classpath 需自备 `mysql-connector-j` 依赖。

## 2.2 Cluster（生产推荐，注册中心协调）

```yaml
mode:
  type: Cluster
  repository:
    type: ZooKeeper       # 官方推荐注册中心
    props:
      namespace: governance
      server-lists: zk1:2181,zk2:2181,zk3:2181
      retryIntervalMilliseconds: 500
      maxRetries: 3
      timeToLiveSeconds: 60
      operationTimeoutMilliseconds: 500
```

Cluster 模式把元数据写入注册中心，所有 Proxy/开启 Cluster 的 JDBC 实例监听同一份数据；任一实例执行 DistSQL 改规则，写注册中心后**其他实例 Watch 到变更自动热加载**，这就是"一处改、全网生效"的实现。`namespace` 是 ZK 根路径前缀，多套 ShardingSphere 共用一套 ZK 时要靠它隔离。

## 2.3 注册中心类型

| repository type | 依赖 artifact | 适用 |
|-----------------|--------------|------|
| `ZooKeeper` | shardingsphere-cluster-mode-repository-zookeeper | **推荐**，成熟稳定 |
| `etcd` | ...repository-etcd | K8s/云原生环境 |
| `JDBC`（Standalone） | ...repository-jdbc | 单机持久化 |

> **注意**：`mode.repository.type` 与早期 4.x 治理中心的注册中心配置项**不通用**，迁移时以 5.5.x `mode` 段为准；Cluster 仓库类型以 ZooKeeper（`shardingsphere-cluster-mode-repository-zookeeper`）为主，使用其他仓库需确认对应 artifact 在你的小版本中存在。

# 3. 元数据结构与持久化

Cluster 模式下注册中心保存的是一棵分层树（以 `namespace` 为根，示意结构，不同小版本节点名略有差异）：

```text
/{namespace}/
  ├── metadata/          # 逻辑库、存储单元、表结构等元数据
  ├── rules/             # 各功能规则（分片/读写分离/加密/影子…）
  ├── props/             # 全局属性（sql-show 等）
  ├── authority/         # 用户与权限
  ├── snapshot/          # 元数据快照，加速启动恢复
  └── nodes/             # 集群节点信息（计算节点在线状态、worker-id 分配等）
```

- **状态维度**：实例注册（`nodes/compute_nodes`）、数据源可用性、`worker_id` 分配（雪花算法在 Cluster 下由注册中心协调，避免多实例 worker-id 撞号——这是第 4 节的关键能力）。
- **配置维度**：`rules/props/authority/metadata` 全部持久化，Proxy 重启从注册中心重建内存配置；JDBC 实例改配置回写注册中心并广播 Watch 事件。

> **设计争议**：把整份配置放注册中心带来便利，也带来"注册中心成为强依赖"的风险——ZK 不可用时，已运行实例靠内存快照继续工作，但**新启动实例会失败、DistSQL 变更被拒**。因此注册中心必须 3 节点起 + 独立于数据库的部署域，别把 ZK 和 MySQL 混在同一台机器。

# 4. 集群协调与治理能力

## 4.1 主键 worker-id 协调

Cluster 下 SNOWFLAKE 的 `worker-id` 若手工配置极易冲突。ShardingSphere 用注册中心做**分布式 worker-id 分配**：实例启动抢占一个临时节点编号，宕机释放——这是"配了 Cluster 就别再手填 worker-id"的原因。

## 4.2 熔断与节点管理（DistSQL RAL）

治理动作统一由 DistSQL RAL 表达（详见 9_distsql）：

```sql
-- 摘除/恢复故障从库（全集群生效）
ALTER READWRITE_SPLITTING RULE readwrite_ds DISABLE ds_slave0;
ALTER READWRITE_SPLITTING RULE readwrite_ds ENABLE ds_slave0;

-- 实例（计算节点）上下线、打标分组
SHOW COMPUTE NODES;
DISABLE COMPUTE NODE '0699e636-ade9-4681-b37a-65240c584bb3';
-- 给实例打标签（做流量分组/灰度），语法为 LABEL COMPUTE NODE "<instance_id>" WITH <label>
LABEL COMPUTE NODE '0699e636-ade9-4681-b37a-65240c584bb3' WITH gray;

-- 集群锁：WRITE 排他锁 / READ_WRITE 读写锁，可带 TIMEOUT 毫秒
LOCK CLUSTER WITH WRITE TIMEOUT 2000;
UNLOCK CLUSTER;
```

## 4.3 元数据刷新

后端库 DDL 变了，让 ShardingSphere 重采样表结构：

```sql
REFRESH TABLE METADATA t_order;
REFRESH DATABASE METADATA sharding_db;   -- 从治理中心拉取
FORCE REFRESH DATABASE METADATA;          -- 把本地最新元数据写回治理中心
```

# 5. 弹性伸缩：Migration Pipeline

在线扩缩容与异构迁库由 **数据迁移（Migration）Pipeline** 完成，它是一条"存量全量 + 增量 binlog + 一致性校验 + 切流"的可控流水线，取代早期 ElasticJob 方案。核心机制：

```text
1. 注册迁移源            REGISTER MIGRATION SOURCE STORAGE UNIT
2. 建目标规则            CREATE SHARDING TABLE RULE（新分片布局）
3. 启动迁移             MIGRATE TABLE ds0.t_order INTO t_order
4. Pipeline 分阶段：
   ① inventory 全量复制   ② commit/rollback point
   ③ increment 增量同步(binlog)   ④ check 一致性校验
5. 校验通过 → 切读写到新布局 → 删旧数据
```

## 5.1 迁移命令序列

```sql
-- 1. 注册待迁移的来源库（迁移专用，与业务存储单元分开）
REGISTER MIGRATION SOURCE STORAGE UNIT ds_src (
    URL="jdbc:mysql://127.0.0.1:3306/legacy_ds?serverTimezone=UTC",
    USER="root", PASSWORD="root"
);

-- 2. 把一张逻辑表从源迁到当前库布局（自动全量+增量+校验）
MIGRATE TABLE ds_src.t_order INTO t_order;

-- 3. 查任务与进度
SHOW MIGRATION LIST;
SHOW MIGRATION STATUS 'j0101...';

-- 4. 一致性校验（CRC32_MATCH 快、DATA_MATCH 准）
CHECK MIGRATION 'j0101...' BY TYPE (NAME = 'CRC32_MATCH');
SHOW MIGRATION CHECK STATUS 'j0101...';

-- 5. 速率/线程可调
ALTER MIGRATION RULE (
    READ(WORKER_THREAD=20, BATCH_SIZE=1000, SHARDING_SIZE=10000000,
         RATE_LIMITER(TYPE(NAME="QPS", PROPERTIES("qps"="500")))),
    WRITE(WORKER_THREAD=20, BATCH_SIZE=1000)
);

-- 6. 完成或中止
COMMIT MIGRATION 'j0101...';
ROLLBACK MIGRATION 'j0101...';
```

## 5.2 扩缩容代价对比

| 方案 | 停机 | 数据搬迁 | 回滚能力 | 适用 |
|------|------|---------|---------|------|
| Migration Pipeline | 否 | 全量+增量 | ROLLBACK | 在线扩容、异构迁库 |
| 停机导数据（mysqldump） | 是 | 全量 | 靠备份 | 一次性小库迁移 |
| 翻倍分裂（预留 2 的幂） | 否 | 部分 | 需自研 | 规划期预留再翻倍 |

> **行业实践**：扩容优先在**分片数设计时按 2 的幂预留**（见 4_sharding），未来"翻倍分裂"只搬一半数据；已经分得很满、或要改分片算法（如从取模换范围）时，用 Migration Pipeline 全量重分布。迁移期一定开 **流量双写或增量校验**，`CRC32_MATCH` 做初筛、`DATA_MATCH` 做终检，切读前先低峰灰度。

# 6. 权限与连接治理

Proxy 作为流量入口自带鉴权与连接治理（配置见 9_distsql 权限节）。治理相关的属性：

```yaml
# conf/global.yaml（Proxy）——以下为 5.5.2 文档中的真实键名
props:
  max-connections-size-per-query: 1              # 单查询每实例最大连接数（防打爆后端）
  proxy-frontend-max-connections: 0              # 前端最大连接数，0 不限
  proxy-backend-query-fetch-size: -1             # 后端游标每次取数行数，-1 取驱动最小值
  proxy-frontend-flush-threshold: 128            # 前端 I/O 刷新阈值
  proxy-frontend-database-protocol-type: MySQL   # 前端协议：MySQL/PostgreSQL/openGauss
  kernel-executor-size: 16                       # SQL 执行线程池大小
  system-log-level: INFO                         # 支持 DistSQL#RAL 动态改
```

- `max-connections-size-per-query`：MEMORY_STRICTLY 模式的核心闸门，限制单逻辑查询占用的后端连接，防止一个全路由大查询吃光某分库连接池。
- 前端连接数、后端连接池是两个独立资源域，Proxy 高可用要分别设置告警阈值。

# 7. 集群可观测与状态

```sql
SHOW DIST VARIABLE;            -- 集群拓扑、规则、属性全景
SHOW COMPUTE NODES;            -- 实例列表（instance_id/host/status/version）
SHOW COMPUTE NODE INFO;
SHOW COMPUTE NODE MODE;        -- 当前节点 Standalone/Cluster
```

配合 Prometheus 指标（11_observability）中的 `proxy_state`（0=OK、1=熔断、2=锁定）可把治理状态纳入监控大盘，`proxy_meta_data_info` 直接暴露逻辑库数与存储单元数。

# 8. 最佳实践

1. **生产必用 Cluster + ZooKeeper**：Standalone 仅开发；ZK 至少 3 节点独立部署。
2. **worker-id 交给注册中心**：Cluster 下别手填 SNOWFLAKE `worker-id`。
3. **变更走 DistSQL + EXPORT 备份**：改前 `EXPORT DATABASE CONFIGURATION`，改后 `PREVIEW` 验证路由。
4. **迁移必做一致性校验**：CRC32 初筛 + DATA_MATCH 终检，切读前灰度。
5. **注册中心高可用 + 快照**：理解 ZK 不可用时"存量可跑、新启动失败"的行为边界。
6. **全局锁用于批量变更**：`LOCK CLUSTER` 冻结路由，避免半更新状态下新旧规则混跑。

# 9. 小结

| 主题 | 结论 |
|------|------|
| 演进 | 4.x Governance → 5.0 Mode；ElasticJob 剥离并独立毕业为 Apache TLP；在线迁移由 Migration Pipeline 承担 |
| Mode | Standalone（JDBC/H2 或外挂库）/ Cluster（ZooKeeper/etcd）；生产 Cluster |
| 元数据 | rules/props/authority/metadata/nodes 持久化到注册中心，Watch 同步热加载 |
| 协调 | worker-id 分配、实例注册、数据源状态、熔断、全局锁 |
| 弹性伸缩 | Migration Pipeline：全量+增量+校验+回滚，扩容首选预翻倍数分片 |
| 风险 | 注册中心强依赖；`max-connections-size-per-query` 护后端连接池 |

下一篇（11_observability）看指标、链路与慢 SQL 观测的落地方式。
