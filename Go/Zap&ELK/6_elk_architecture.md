ELK 不是一个软件，而是 Elasticsearch、Logstash、Kibana 三个组件的合称，再加上采集端的 Beats，实际是四个角色。很多人能照着教程跑通 demo，却说不清一条日志从产生到能在 Kibana 里搜到，中间经过了哪些缓冲、哪一步可能丢数据、分片到底该怎么规划。本篇把架构、数据流和容量规划讲清楚。

# 1. ELK 是什么

## 1.1 四个组件的职责

```text
┌──────────┐   ┌──────────┐   ┌──────────────┐   ┌──────────┐
│ Filebeat │──►│ Logstash │──►│ Elasticsearch│──►│  Kibana  │
│  采集     │   │  解析/过滤│   │   存储/检索   │   │  可视化   │
└──────────┘   └──────────┘   └──────────────┘   └──────────┘
   轻量           重量             有状态             无状态
```

| 组件 | 职责 | 有状态 | 资源特点 |
|------|------|--------|----------|
| Filebeat | 读文件、多行合并、轻量字段处理 | 有（registry 记录读取位置） | 极低，几十 MB |
| Logstash | Grok 解析、字段转换、丰富、路由 | 可选（持久化队列） | 高，JVM 500 MB 起 |
| Elasticsearch | 倒排索引、存储、聚合、副本 | 有 | 很高，磁盘 + 内存 |
| Kibana | 查询界面、可视化、Dashboard | 无（配置存 ES） | 中 |

★ **关键认知**：**Filebeat 和 Logstash 不是替代关系，而是分工关系**。Filebeat 负责"把文件可靠地搬走"，Logstash 负责"把非结构化数据变成结构化数据"。把所有解析都压在 Filebeat 上，会让采集端变重、升级困难；把所有采集都塞给 Logstash，会浪费 JVM 资源。

## 1.2 数据流的完整链路

```text
应用写日志
   │
   ▼
本地文件 / stdout
   │
   ▼
┌─────────────────────────────────────────────────────────┐
│ Filebeat（每台机器/每个 Pod）                             │
│  - 读文件，记录 offset 到 registry                        │
│  - 多行合并（Java 堆栈、panic）                           │
│  - 加 host、container、tags 等元数据                      │
└────────────────────────┬────────────────────────────────┘
                         │ 可选：Kafka / Redis 缓冲
                         ▼
┌─────────────────────────────────────────────────────────┐
│ Logstash（可选，按需部署）                                │
│  input beats → filter grok/mutate/date → output ES       │
└────────────────────────┬────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│ Elasticsearch                                            │
│  主分片写入 → 副本同步 → refresh 后可搜 → segment 合并     │
└────────────────────────┬────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│ Kibana                                                   │
│  Data View → Discover 查询 → Lens 可视化 → Dashboard      │
└─────────────────────────────────────────────────────────┘
```

# 2. Elasticsearch

## 2.1 索引、分片与副本

| 概念 | 说明 | 类比 |
|------|------|------|
| Index | 一类文档的集合 | 数据库里的"表" |
| Document | 一条 JSON 记录 | 一行数据 |
| Field | 文档的字段 | 列 |
| Primary Shard | 索引的水平切分，决定容量上限 | 分库分表的分片 |
| Replica Shard | 主分片的副本，提供高可用和读吞吐 | 从库 |
| Segment | Lucene 的不可变索引块 | 只读的小文件 |

```text
索引 orders（3 主分片，1 副本）
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  P0         │  │  P1         │  │  P2         │
│  Node A     │  │  Node B     │  │  Node C     │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │
       ▼                ▼                ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  R0         │  │  R1         │  │  R2         │
│  Node B     │  │  Node C     │  │  Node A     │
└─────────────┘  └─────────────┘  └─────────────┘
```

★ **关键认知**：**主分片数在索引创建后不能改**（只能 reindex 或 split）。所以分片规划要在建索引前想清楚。副本数可以随时调整，用于扩容读吞吐或提高可用性。

## 2.2 写入流程

```text
1. 客户端/Logstash 发送 bulk 请求到协调节点
2. 协调节点按 _id 或 _routing 计算目标分片：shard = hash(routing) % 主分片数
3. 请求转发到该分片的 Primary
4. Primary 写入内存 buffer + translog（事务日志）
5. Primary 把请求并行转发给所有 Replica
6. Replica 写入成功返回 ACK
7. Primary 收到所有副本 ACK 后返回成功
8. 默认每 1s refresh：buffer → 新 segment，此时文档可被搜索
9. translog 定期 flush 到磁盘，保证持久化
```

| 阶段 | 何时可见 | 说明 |
|------|----------|------|
| 写入成功 | 已持久化到 translog | 进程崩溃可恢复 |
| refresh | 约 1 秒后 | 可被搜索，但未落盘到 segment |
| flush | 默认 30 分钟或 translog 满 | segment 落盘，translog 清空 |

⚠ **关键认知**：ES 是**近实时（NRT）**，不是实时。写完立刻查可能查不到，最多等一个 refresh interval。日志场景通常把 `refresh_interval` 调到 30s，显著降低 segment 数量和合并压力。

## 2.3 查询流程

```text
1. 请求打到协调节点
2. 协调节点把查询广播到索引的每个分片（主或副本）
3. 每个分片本地查询，返回 doc_id + 排序值（query 阶段）
4. 协调节点合并、排序，取前 N 个 doc_id
5. 再向对应分片 fetch 完整 _source（fetch 阶段）
6. 返回结果
```

这解释了一个常见现象：**分片越多，查询的 fan-out 越大，协调节点合并成本越高**。日志索引常见 1 主分片 + 1 副本，就是因为单分片几十 GB 足够，不需要为了并行度切太碎。

## 2.4 索引模板与数据流

8.x 推荐用 **data stream + index template** 管理按日期滚动的日志索引：

```json
PUT _index_template/logs-app
{
  "index_patterns": ["logs-app-*"],
  "data_stream": {},
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1,
      "refresh_interval": "30s",
      "lifecycle": { "name": "logs-ilm" }
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "service": {
          "properties": {
            "name": { "type": "keyword" }
          }
        },
        "request_id": { "type": "keyword" },
        "message": { "type": "text" },
        "log": {
          "properties": {
            "level": { "type": "keyword" }
          }
        }
      }
    }
  }
}
```

写数据时直接写 data stream 名 `logs-app`，ES 会自动路由到当前的后备索引（如 `.ds-logs-app-2026.11.06-000001`），ILM 负责滚动和删除。

★ **关键认知**：**日志索引不要用动态映射（dynamic mapping）裸奔**。第一条日志里 `duration` 是数字，第二条变成 `"1.2s"` 字符串，映射冲突后整个 bulk 都可能失败。生产上必须显式定义模板，至少把 `request_id`、`service.name` 映射成 `keyword`。

# 3. Logstash 与 Filebeat 分工

## 3.1 各自定位

| 维度 | Filebeat | Logstash |
|------|----------|----------|
| 语言 | Go | JRuby / Java |
| 资源 | 几十 MB | 500 MB+ |
| 解析能力 | 基础（JSON、multiline、dissect） | 强（grok、ruby、geoip） |
| 缓冲 | 内存队列 + registry | 持久化队列（PQ） |
| 适合 | 边缘采集、容器 | 中心化解析、丰富、路由 |
| 部署位置 | 每台机器/每个 Pod | 中心节点或独立集群 |

## 3.2 三种典型拓扑

**拓扑一：Filebeat 直连 ES（最简单）**

```text
Filebeat ──► Elasticsearch ──► Kibana
```

适合日志已经是 JSON、不需要复杂解析的场景。少一跳，少一个故障点。

**拓扑二：Filebeat → Logstash → ES（最常见）**

```text
Filebeat ──► Logstash（grok/geoip/路由）──► Elasticsearch
```

适合 Nginx、Java 堆栈等非结构化日志，需要中心化解析。

**拓扑三：Filebeat → Kafka → Logstash → ES（大规模）**

```text
Filebeat ──► Kafka ──► Logstash ──► Elasticsearch
              ▲
              └─ 削峰、解耦、可回放
```

Kafka 在这里承担**缓冲和解耦**：ES 维护时日志不丢，流量突增时不把 Logstash 打挂，出问题还能从 Kafka 重放。

| 拓扑 | 优点 | 缺点 | 适合规模 |
|------|------|------|----------|
| 直连 | 简单、低延迟 | 无缓冲、解析弱 | 小 |
| 经 Logstash | 解析强、可路由 | JVM 资源、需运维 | 中 |
| 经 Kafka | 削峰、解耦、可回放 | 组件多、运维复杂 | 大 |

★ **关键认知**：**不要为了"架构完整"硬上 Kafka**。日量 10 GB 以下，Filebeat → Logstash → ES 完全够用；上了 Kafka 却没有消费监控和磁盘规划，反而多一个会挂的组件。

# 4. 数据流与可靠性

## 4.1 端到端至少一次

日志链路通常承诺 **at-least-once**，不承诺 exactly-once：

| 环节 | 保证 | 可能的问题 |
|------|------|------------|
| 应用写文件 | 由文件系统保证 | 进程崩溃可能丢缓冲区 |
| Filebeat 读 | registry 记录 offset | 轮转/删除过快会丢 |
| Kafka | 副本 + ACK | 配置不当会丢 |
| Logstash | PQ + ACK | 内存队列会丢 |
| ES bulk | 部分成功需检查 | 忽略 `errors` 会丢 |

```json
// bulk 响应里 errors=true 时，必须逐条检查并重试失败项
{
  "took": 30,
  "errors": true,
  "items": [
    { "index": { "_index": "logs-app", "status": 201 } },
    { "index": { "_index": "logs-app", "status": 400, "error": { "type": "mapper_parsing_exception" } } }
  ]
}
```

⚠ **关键认知**：很多"日志丢了"的事故，根因是**客户端忽略了 bulk 响应里的 `errors`**。ES 对单条文档的映射失败不会让整个请求失败，只会在 `items` 里标 400。必须检查并处理。

## 4.2 背压

当下游变慢，压力会沿链路往回传：

```text
ES 写入慢 → Logstash 队列涨 → Filebeat 发送阻塞 → Filebeat 读取变慢 → 本地文件堆积
```

| 环节 | 背压表现 | 应对 |
|------|----------|------|
| Filebeat | 发送重试、harvester 暂停 | 调大队列、加 Logstash |
| Logstash | PQ 满、拒绝新事件 | 扩实例、优化 grok |
| ES | bulk 拒绝（429） | 降 refresh、加节点、调 bulk size |

★ **关键认知**：背压不是故障，是**保护机制**。真正危险的是"没有背压"——Filebeat 读得飞快、Logstash 内存队列无限涨，最后 OOM 一次性丢掉全部缓冲。所以生产上要给每一跳都设置**有界队列**。

# 5. 容量规划

## 5.1 存储估算

```text
每日原始日志量 = QPS × 每条字节数 × 86400
每日 ES 存储   = 原始量 × (1 + 副本数) × 索引膨胀系数
索引膨胀系数   ≈ 1.1 ~ 1.5（倒排索引 + _source + 存储开销）
```

例子：5 万 QPS，每条 400 字节，1 副本：

```text
原始量 = 50000 × 400 × 86400 ≈ 1.7 TB/天
ES 存储 = 1.7 TB × 2 × 1.2 ≈ 4.1 TB/天
保留 7 天 ≈ 29 TB
```

这个数字通常会让人重新考虑采样和字段精简。

## 5.2 分片规划

| 单分片大小 | 评价 |
|------------|------|
| < 1 GB | 太碎，分片开销占比高 |
| 10-50 GB | 推荐区间 |
| 50-100 GB | 可接受，恢复慢 |
| > 100 GB | 太大，合并和恢复都痛苦 |

```text
分片数 ≈ 每日索引大小 / 30 GB
```

日志场景按天滚动，每天一个索引，通常 1-2 个主分片就够。**分片不是越多越并行**，每个分片都是一个 Lucene 实例，有堆内存和文件句柄开销。

## 5.3 节点规划

| 节点角色 | 职责 | 建议 |
|----------|------|------|
| master | 集群状态管理 | 3 个专用，小规格 |
| data hot | 写入 + 近期查询 | 本地 SSD，内存 ≥ 磁盘的 1/30 |
| data warm/cold | 历史查询 | 大磁盘，低配 |
| coordinating | 查询聚合 | 大内存、多核 |
| ingest | 预处理 pipeline | 按解析量 |

★ **关键认知**：ES 的堆内存建议**不超过 31 GB**（压缩指针边界），且不超过物理内存的 50%，剩下的留给文件系统缓存。把 128 GB 机器全给 JVM，反而会因为缺 page cache 导致查询变慢。

# 6. 常见架构误区

| 误区 | 后果 | 正确做法 |
|------|------|----------|
| 所有日志都进 ES，不采样 | 成本爆炸 | Debug 采样、字段精简、分层保留 |
| 分片数按节点数分配 | 后续扩容困难 | 按数据量规划，1 主分片起步 |
| 用 ES 存业务主数据 | 无事务、更新代价高 | 只存检索副本 |
| Logstash 用内存队列 | 重启丢数据 | 开持久化队列 |
| Filebeat 直连 ES 却做复杂 grok | 采集端 CPU 高 | 交给 Logstash 或 ingest pipeline |
| 忽略 bulk errors | 静默丢日志 | 检查响应、失败重试 |
| 所有索引 1 天滚动但量很小 | 大量小索引、分片爆炸 | 按大小或 7 天滚动 |

⚠ **关键认知**：**ELK 的成本大头是存储和查询，不是采集**。优化顺序应该是：先砍无用日志（采样、降级）→ 再精简字段 → 再调 refresh/合并 → 最后才考虑加机器。

# 7. 练习与自检

1. 画出你当前项目的日志链路，标出每一跳的队列类型（内存/磁盘）和容量上限。哪一跳满了会最先丢数据？
2. 日增原始日志 500 GB，1 副本，膨胀系数 1.2，保留 7 天。需要多少存储？如果单分片控制在 30 GB，每天索引应该几个分片？
3. 为什么 ES 写完不能立刻搜到？请从 buffer、refresh、translog、flush 四个概念解释，并说明日志场景为什么可以把 refresh 调到 30s。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 四组件 | Filebeat 采集、Logstash 解析、ES 存储、Kibana 展示 |
| 分片 | 主分片创建后不可改，副本可调 |
| 写入 | translog 持久化，refresh 后可搜 |
| 模板 | 显式 mapping，ID 用 keyword |
| 拓扑 | 直连 / 经 Logstash / 经 Kafka，按规模选 |
| 可靠性 | at-least-once，必须处理 bulk errors |
| 背压 | 每跳有界队列，防止 OOM 丢全部 |
| 容量 | 原始量 × 副本 × 膨胀，分片 10-50 GB |

下一篇进入 Filebeat 与 Logstash 的具体配置，讲清 filestream 输入、多行日志合并、Grok 解析、过滤链、背压参数、断点续传原理，以及最常见的配置错误。
