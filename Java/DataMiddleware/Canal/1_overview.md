
# 1. 什么是 Canal

Canal 是阿里巴巴开源的一款基于 MySQL 数据库增量日志（Binary Log）解析的中间件，提供增量数据订阅和消费能力。其名称"Canal"意为"运河/管道"，形象地表达了它作为数据传输管道的作用。

Canal 的核心工作原理是**模拟 MySQL Slave 的交互协议**，伪装成一个 MySQL 从库，向 MySQL Master 发送 dump 协议请求，MySQL Master 收到请求后开始推送 Binary Log 给 Canal（Slave），Canal 解析 Binary Log 对象，还原出原始 SQL 的变更数据。

```
┌─────────────┐     Binary Log Dump      ┌──────────────┐
│ MySQL Master │ ──────────────────────> │ Canal Server │
│  (Source)    │                          │  (Simulated  │
│              │                          │   Slave)     │
└─────────────┘                          └──────┬───────┘
                                                │
                                         Parse & Transform
                                                │
                                         ┌──────▼───────┐
                                         │ Canal Client │
                                         │   (Consumer) │
                                         └──────────────┘
```

# 2. 背景与历史

## 2.1 起源

早期阿里巴巴在杭州和美国双机房部署，存在跨机房数据同步的业务需求。最初的解决方案是业务层双写（写入杭州和美国两个数据库），但这带来了数据一致性和维护上的巨大挑战。

从 2010 年开始，阿里巴巴开始尝试基于数据库日志解析获取增量变更进行同步，由此诞生了 Canal 项目（内部代号）。2013 年 1 月，Canal 正式开源在 GitHub 上。

## 2.2 版本演进

| 版本 | 发布时间 | 主要特性 |
|------|---------|---------|
| v1.0.x | 2013 | 基本功能：binlog 解析、客户端订阅 |
| v1.1.0 | 2018 | Canal Admin 管理平台、多实例管理 |
| v1.1.3 | 2019 | 支持 MySQL 8.0、GTID 模式、阿里云 RDS |
| v1.1.4 | 2020 | Prometheus 监控、Kafka 消息格式增强 |
| v1.1.5 | 2021 | Canal Adapter 增强、性能优化 |
| v1.1.6 | 2023 | 稳定性提升、bug 修复 |
| v1.1.7 | 2024 | 最新稳定版，兼容 MySQL 8.3 |

## 2.3 社区与生态

- **GitHub Stars**: 27k+（Java 语言 CDC 领域 Star 数最高）
- **Fork**: 7.5k+
- **Contributors**: 70+
- **典型用户**: 阿里巴巴、美团、滴滴、字节跳动、快手、小米等

# 3. 核心能力

## 3.1 Binlog 伪装与订阅

Canal 模拟 MySQL Slave 的交互协议，向 MySQL Master 发起 dump 请求：

1. Canal 向 Master 发送 `COM_REGISTER_SLAVE` 注册为 Slave
2. 发送 `COM_BINLOG_DUMP` 请求，指定 binlog 文件名和偏移量
3. Master 认证通过后，持续推送 binlog 事件流
4. Canal 接收并解析 binlog event，还原为结构化的变更数据

## 3.2 Binlog 数据解析

Canal 支持解析三种 MySQL binlog 格式：

- **ROW 格式** (推荐)：记录每行数据变更前后的完整信息，Canal 可获取字段级别的变更
- **STATEMENT 格式**：记录 SQL 语句原文，解析复杂，不推荐
- **MIXED 格式**：MySQL 自动选择 STATEMENT 或 ROW，Canal 兼容处理

## 3.3 增量订阅与消费

Canal 提供 Client SDK，支持多种消费模式：

```
Canal Server (push binlog event)  <----  Canal Client (pull & ack)
```

- **单机消费**：单个 Client 连接到 Canal Server
- **集群消费**：多个 Client 组成消费者集群，通过 ZooKeeper 协调
- **多消费者组**：不同 Client 组可以独立消费同一份增量数据

## 3.4 数据投递

Canal Server 可以将解析后的数据投递到多种下游系统：

```
┌──────────┐    ┌──────────┐    ┌───────────┐
│  Kafka   │    │ RocketMQ │    │ RabbitMQ  │
└────┬─────┘    └────┬─────┘    └─────┬─────┘
     │               │                │
     └───────────────┼────────────────┘
                     │
              ┌──────▼───────┐
              │ Canal Server │
              └──────┬───────┘
                     │
     ┌───────────────┼────────────────┐
     │               │                │
┌────▼─────┐   ┌────▼─────┐   ┌──────▼──────┐
│  MySQL   │   │  ES      │   │ Custom      │
│ Adapter  │   │ Adapter  │   │ Client      │
└──────────┘   └──────────┘   └─────────────┘
```

# 4. 典型应用场景

## 4.1 数据库实时同步

最核心的场景之一。通过 Canal 将源数据库的变更实时同步到目标数据库：

```
MySQL (Source) --> Canal --> MySQL (Target)
                         --> Oracle (Target)
                         --> PostgreSQL (Target)
```

**示例**：跨机房数据同步、读写分离架构中的主从延迟补偿、数据库迁移。

## 4.2 缓存一致性维护

通过监听数据库变更，实时更新或失效缓存：

```java
// 伪代码示例
canalClient.subscribe("order_db\\.*");
while (running) {
    Message message = connector.getWithoutAck(batchSize);
    for (Entry entry : message.getEntries()) {
        if (entry.getEntryType() == EntryType.ROWDATA) {
            RowChange rowChange = RowChange.parseFrom(entry.getStoreValue());
            String tableName = entry.getHeader().getTableName();
            // 更新缓存
            updateCache(tableName, rowChange);
        }
    }
    connector.ack(message.getId());
}
```

**典型架构**：
```
MySQL --> Canal --> Kafka --> 缓存更新服务 --> Redis
```

## 4.3 搜索索引实时构建

将数据库变更同步到搜索引擎，实现准实时搜索：

```
MySQL --> Canal --> Elasticsearch/Solr
```

常见于电商商品搜索、日志检索系统等场景。

## 4.4 数据仓库增量抽取

作为 ETL 的增量数据源：

```
MySQL --> Canal --> Kafka --> Flink/Spark Streaming --> Hive/Hudi/Iceberg
```

## 4.5 业务解耦与事件驱动

数据库变更是最可靠的事实来源，Canal 将其转化为事件流：

```
MySQL (Order表) --> Canal --> Kafka -->  订单通知服务
                                      -->  库存扣减服务
                                      -->  积分计算服务
                                      -->  数据分析服务
```

## 4.6 数据库审计与监控

记录所有数据变更，用于合规审计或数据回溯：

```sql
-- 通过 Canal 捕获所有 DML，写入审计表
INSERT INTO audit_log (table_name, operation, old_data, new_data, timestamp)
VALUES ('user', 'UPDATE', '{"name":"old"}', '{"name":"new"}', NOW());
```

# 5. 项目模块概览

Canal 项目包含以下几个核心子模块：

| 模块 | 说明 |
|------|------|
| **canal-server** | Canal 服务端，核心模块，负责 binlog 订阅、解析、存储 |
| **canal-client** | Java 客户端 SDK，提供消费 API |
| **canal-admin** | Web 管理平台，管理 Canal 实例和集群 |
| **canal-adapter** | 数据适配器，将增量数据同步到各类目标存储 |
| **canal-protocol** | 通信协议定义（基于 Protobuf） |
| **canal-common** | 公共工具类、配置管理 |
| **canal-deployer** | 部署包，包含 Server 和 Admin |
| **connector** | Kafka/RocketMQ/RabbitMQ 连接器 |
| **dbsync** | 数据库同步驱动（基于 open-replicator 演化） |

```
canal
├── canal-server        # 服务端核心
│   ├── canal-instance  # 实例管理
│   ├── canal-parser    # 解析器
│   ├── canal-sink      # 数据下沉
│   └── canal-store     # 数据存储
├── canal-client        # 客户端SDK
├── canal-admin         # 管理后台
├── canal-adapter       # 数据适配器
├── canal-protocol      # 通信协议
├── canal-common        # 公共组件
├── canal-deployer      # 部署包
├── connector           # MQ连接器
│   ├── kafka-connector
│   ├── rocketmq-connector
│   └── rabbitmq-connector
└── dbsync              # 同步驱动
```

# 6. Canal 的核心优势

## 6.1 业务无侵入

Canal 基于 MySQL binlog 工作，不需要修改业务代码，不需要在应用层引入 SDK 或 AOP 切面。对业务系统完全透明。

## 6.2 数据完整性强

基于 ROW 格式的 binlog 可以获取到每一行数据的完整变更前和变更后镜像，不会丢失任何变更。相比应用层双写，数据一致性和完整性更有保障。

## 6.3 低延迟

Canal 采用 Netty + 内存队列的架构，从 MySQL binlog 产生到 Canal Client 消费，延迟通常在毫秒级。

```
Binlog产生 --> Canal解析 --> Client消费
   < 50ms       < 10ms       < 50ms
         总延迟通常 < 100ms
```

## 6.4 丰富的数据投递能力

通过 Canal Adapter 和 Connector 模块，支持投递到：
- 关系型数据库（MySQL、Oracle、PostgreSQL、SQLServer）
- 搜索引擎（Elasticsearch）
- 消息队列（Kafka、RocketMQ、RabbitMQ、Pulsar）
- 大数据存储（HBase、HDFS、Kudu）
- 自定义扩展（SPI 机制）

## 6.5 生产级高可用

- 基于 ZooKeeper 的 HA 切换
- 消费位点持久化与自动恢复
- 支持 GTID 模式，切换更安全

## 6.6 活跃的开源社区

27k+ GitHub Stars，70+ 贡献者，持续维护更新，社区活跃度高。文档完善，有大量中文和英文使用案例。

# 7. 与同类产品的定位

| 维度 | Canal | Debezium | Maxwell | Flink CDC |
|------|-------|----------|---------|-----------|
| 语言 | Java | Java | Java | Java |
| 架构 | Server-Client | Kafka Connect | Standalone | Flink Job |
| 协议模拟 | 自行实现 | 自行实现 | 基于 open-replicator | 基于 Debezium |
| 目标投递 | Adapter/MQ | Kafka | Kafka/Kinesis | Flink Sink |
| DDL支持 | 部分支持 | 支持 | 部分支持 | 支持 |
| Schema变更 | 手动处理 | 自动 | 手动 | 自动 |
| 管理界面 | Canal Admin | Kafka Connect UI | 无 | Flink UI |
| 学习成本 | 中等 | 中等 | 低 | 较高 |

Canal 最突出的优势在于：**丰富的数据投递能力（Adapter）+ 完善的管理平台（Admin）+ 成熟的国内生产实践**。

# 8. 学习路线建议

对于 Canal 的初学者，建议按以下顺序学习：

1. **第 2 章**：理解 MySQL Binlog 基础，这是 Canal 的底层依赖
2. **第 3 章**：掌握 Canal 的整体架构设计
3. **第 4 章**：动手实践，完成安装和快速开始
4. **第 5-7 章**：深入源码，理解 EventParser、EventSink、MetaManager
5. **第 8-9 章**：理解 MySQL 协议模拟和 Binlog 解析的实现细节
6. **第 10-12 章**：学习 Client 使用、协议定义和数据过滤
7. **第 13-17 章**：掌握 Admin 管理和 Adapter 同步
8. **第 18-19 章**：了解 MQ 集成方案
9. **第 20-25 章**：学习运维实战内容（HA、性能、监控、排障、最佳实践）
