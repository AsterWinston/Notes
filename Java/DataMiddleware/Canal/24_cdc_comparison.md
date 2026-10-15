
# 1. CDC 工具总览

Change Data Capture（CDC）是一种捕获数据库变更数据的技术。当前主流的开源 CDC 工具包括：

| 工具 | 语言 | 架构 | GitHub Stars | 社区 |
|------|------|------|-------------|------|
| **Canal** | Java | Server-Client | 27k+ | 阿里巴巴维护 |
| **Debezium** | Java | Kafka Connect | 10k+ | Red Hat 维护 |
| **Maxwell** | Java | Standalone | 4k+ | Zendesk 维护 |
| **Flink CDC** | Java | Flink Job | 5k+ | Apache Flink 社区 |
| **Databus** | Java | Relay-Client | 3.6k+ | LinkedIn 维护 |

# 2. Canal

## 2.1 核心特点

```
Canal: Alibaba 开源，专注 MySQL Binlog 解析
├── 架构: Server-Client 模式
├── 协议: 自研 Binlog Dump + Protobuf
├── 投递: Adapter（MySQL/Oracle/ES/HBase）+ MQ（Kafka/RocketMQ）
├── 管理: Canal Admin Web UI
├── 定位: 阿里系生态中间件
└── 用户: 阿里巴巴、美团、滴滴、字节跳动
```

## 2.2 优势
- 国内文档丰富，中文社区活跃
- Canal Adapter 支持多种目标（RDB/ES/HBase）
- Canal Admin 提供完善的管理界面
- 与 Kafka/RocketMQ 深度集成
- 阿里云 RDS 原生支持

## 2.3 局限
- 仅支持 MySQL（及其衍生版本）
- Schema 变更需手动处理
- DDL 同步能力有限
- 依赖 ZooKeeper 实现高可用
- 非 Kafka Connect 生态

# 3. Debezium

## 3.1 核心特点

```
Debezium: Red Hat 维护，Kafka Connect 生态
├── 架构: Kafka Connect Source Connector
├── 协议: 自研数据库连接协议
├── 数据库支持: MySQL / PostgreSQL / MongoDB / SQL Server / Oracle / DB2
├── 管理: Kafka Connect REST API
├── 定位: 云原生 CDC 标准
└── 用户: Red Hat、Confluent、Zalando
```

## 3.2 优势
- 支持多种数据库（覆盖面最广）
- 原生 Kafka 集成，无需额外部署
- 支持 Schema 变更自动演进（Schema Registry）
- 快照（Snapshot）机制完善
- 活跃的国际化社区，文档英文为主
- 支持 Kubernetes Operator 部署

## 3.3 局限
- 必须依赖 Kafka 和 Kafka Connect
- 中文文档和社区较少
- 架构较重（需要 Kafka + Connect + Schema Registry）
- 没有 Canal Admin 那样的管理 UI
- DDL 变更可能影响连接器稳定性

# 4. Maxwell

## 4.1 核心特点

```
Maxwell: Zendesk 开发，轻量级 Binlog 解析
├── 架构: 单进程 Standalone 模式
├── 协议: 基于 open-replicator
├── 投递: Kafka / Kinesis / Google Pub/Sub / RabbitMQ / Redis
├── 管理: 无（依赖日志）
├── 定位: 简单、轻量的 MySQL CDC
└── 用户: Zendesk、GitHub、Etsy
```

## 4.2 优势
- 极其简单，一个 JAR + 一个配置文件即可运行
- 支持多种数据投递目标
- 代码量小，容易理解
- 支持 `bootstrap` 全量初始化
- 学习成本低

## 4.3 局限
- 仅支持 MySQL
- 不支持高可用（单点）
- 功能相对简单（无过滤引擎、无 Adapter 体系）
- 无管理 UI
- 社区相对较小

# 5. Flink CDC

## 5.1 核心特点

```
Flink CDC: Apache Flink 生态的 CDC 连接器
├── 架构: Flink Source Connector
├── 协议: 基于 Debezium Engine
├── 数据库支持: MySQL / PostgreSQL / MongoDB / Oracle / TiDB
├── 管理: Flink Web UI
├── 定位: 流式计算 + CDC 一体化
└── 用户: Flink 生态用户
```

## 5.2 优势
- 与 Flink 深度集成，天然支持流式计算
- 支持 Exactly-Once 语义（Flink Checkpoint）
- Schema Evolution 自动处理
- 无需部署独立 CDC 服务
- 支持多种数据库

## 5.3 局限
- 必须依赖 Flink 集群
- 只适用于已有 Flink 基础设施的场景
- 资源消耗较大
- SQL CDC Connector 仍在快速演进中
- 中文文档以 Flink 为主，CDC 文档较少

# 6. 综合对比

## 6.1 功能维度

| 功能 | Canal | Debezium | Maxwell | Flink CDC |
|------|-------|----------|---------|-----------|
| 数据库支持 | MySQL | MySQL/PG/Mongo/Oracle/SQLSrv/DB2 | MySQL | MySQL/PG/Mongo/Oracle |
| Binlog 解析 | 自研 | 自研 | open-replicator | Debezium Engine |
| 高可用 | ZooKeeper | Kafka Connect 分布式 | ❌ | Flink HA |
| DDL 处理 | 部分支持 | 支持 | 部分支持 | 支持 |
| Schema 变更 | 手动 | 自动(Schema Registry) | 手动 | 自动 |
| 全量+增量 | ETL 命令 | Snapshot | bootstrap | Startup Mode |
| 数据过滤 | Aviator | SMT | 配置过滤 | SQL WHERE |
| 管理 UI | Canal Admin | Kafka Connect UI | ❌ | Flink UI |
| Prometheus | ✅ | ✅ | ❌ | ✅ |

## 6.2 性能维度

| 维度 | Canal | Debezium | Maxwell | Flink CDC |
|------|-------|----------|---------|-----------|
| TPS上限 | 10k+ | 10k+ | 5k+ | 10k+ |
| 延迟 | < 100ms | < 100ms | < 100ms | < 100ms |
| 内存占用 | 中等(2-4GB) | 中等(2-4GB) | 低(1-2GB) | 高(4-8GB+) |
| 资源开销 | 中等 | 中等 | 低 | 高(需要Flink集群) |

## 6.3 运维维度

| 维度 | Canal | Debezium | Maxwell | Flink CDC |
|------|-------|----------|---------|-----------|
| 部署复杂度 | 中等 | 高(多组件) | 低 | 高(需要Flink) |
| 学习成本 | 中等 | 中等 | 低 | 高 |
| 中文文档 | 丰富 | 较少 | 较少 | 中等 |
| 国内实践 | 非常多 | 中等 | 少 | 快速增长 |
| 监控完善度 | 中等 | 高 | 低 | 高 |
| 社区活跃度 | 高 | 高 | 中等 | 高 |

# 7. 选型建议

## 7.1 按场景选型

```
场景: 阿里云/国内生产环境
推荐: Canal
理由: 阿里云支持、国内文档丰富、Admin 管理平台、社区活跃

场景: 国际化/多数据库环境
推荐: Debezium
理由: 数据库支持广、Kafka 原生集成、Schema Registry、云原生

场景: 已有 Flink 基础设施
推荐: Flink CDC
理由: 流计算一体化、Exactly-Once、无需独立 CDC 服务

场景: 简单快速验证
推荐: Maxwell
理由: 单 JAR 启动、配置简单、快速上手

场景: 复杂数据同步（多目标）
推荐: Canal
理由: Canal Adapter 的目标覆盖面最广（RDB/ES/HBase/Kafka 等）
```

## 7.2 架构决策树

```
需要多数据库支持？
├── 是 → Debezium 或 Flink CDC
└── 否 → 继续
         │
    已有 Kafka 基础设施？
    ├── 是 → Debezium
    └── 否 → 继续
             │
        已有 Flink 基础设施？
        ├── 是 → Flink CDC
        └── 否 → 继续
                 │
            国内环境？
            ├── 是 → Canal
            └── 否 → Debezium
```

## 7.3 迁移注意事项

从其他 CDC 工具迁移到 Canal 时需要注意：

1. **位点迁移**：确保从正确的 Binlog 位点开始
2. **数据格式差异**：Canal 的 FlatMessage vs Debezium 的 Change Event
3. **表结构处理**：Schema 变更的处理方式不同
4. **重试语义**：At-Least-Once vs Exactly-Once
5. **监控切换**：从 Kafka Connect Metrics 迁移到 Prometheus + Grafana
