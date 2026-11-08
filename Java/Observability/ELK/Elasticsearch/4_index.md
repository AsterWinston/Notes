# 1. 索引概念

```
Index (索引) = 一个独立的"数据库"
  ├── Settings: 分片数、副本数、分析器配置
  ├── Mappings: 字段类型定义
  └── Documents: 实际数据（存储在 Shard 中）
```

ES 里没有"建库""建表"的动作，**索引就是唯一的数据组织单元**，它把关系数据库里 schema（Mappings）、表参数（Settings）、表数据（Documents）三件事打包在一起。索引名必须全小写，不能以 `_`、`-`、`+` 开头。

一个必须尽早建立的观念是：**索引是重决策对象，改起来大多要推倒重来**。主分片数创建后不可变、mapping 里已写数据字段的类型不可变（只能新增字段），因此"什么时候建新索引、怎么命名、分多大"决定了后面两年的运维成本。日志类场景几乎没人手写索引——靠索引模板 + Data Stream + ILM 自动生成与轮转，手工创建只用于业务文档类索引。本篇按"手工建 → 别名 → 模板 → Data Stream/ILM → 日常操作"的顺序展开，正好是复杂度递增的四种使用形态。

# 2. 创建索引

```json
PUT /orders
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "refresh_interval": "30s",
    "analysis": {
      "analyzer": {
        "my_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "order_no": { "type": "keyword" },
      "user_name": { "type": "text", "analyzer": "my_analyzer" },
      "amount": { "type": "double" },
      "status": { "type": "keyword" },
      "created_at": { "type": "date", "format": "yyyy-MM-dd HH:mm:ss" }
    }
  },
  "aliases": {
    "order_alias": {}
  }
}
```

创建请求的三段结构各管一摊：**settings 管分布式形态与性能参数**（分片/副本数、refresh、自定义分析器），**mappings 管字段语义**（详见第 6 篇），**aliases 管访问入口**。一个容易忽略的机制是：`PUT /orders` 不带任何 body 也能"成功"——ES 会用默认 settings（1 主 1 副）加 dynamic mapping 兜底创建。这种"空手创建"在生产上是事故源头：等数据写进去才发现默认 1 分片太小、dynamic mapping 把 status 猜成了 text，而两者都已不可改。

**分析器（analyzer）只能在创建时定义**：写入时用什么分词、查询时就必须用什么分词，中途改分析器会让老段里的 term 和新查询的 term 对不上——所以改分词逻辑的唯一出路是重建索引（reindex），这也是别名方案（第 4 节）如此重要的原因。

> **注意**：创建索引与写入文档的顺序不能反。先 `PUT` 出完整 settings/mappings，再写第一批数据；一旦某个字段因首条文档被 dynamic mapping 猜错类型，纠正它的代价是 reindex 全量数据。

# 3. 索引 Setting

Setting 分静态与动态两类，界限来自"改了会不会让已有数据的物理布局失效"。

## 3.1 静态 Setting（索引创建时定，不可在线改）

```json
{
  "number_of_shards": 3,              // 主分片数：路由公式 hash(routing) % N 的模数，改了老数据就找不到
  "codec": "best_compression"          // 压缩算法：DEFLATE 换磁盘省 ~30%，代价是解压 CPU
  "analyzer": { ... },                 // 分析器定义
  "index.routing_partition_size": 2    // 分区路由参数
}
```

静态 setting 并非绝对无解——`_shrink`（减分片）、`_split`（翻倍加分片）、`_clone` 三个 API 能在受控条件下改写拓扑，但它们的本质是"关闭写入 → 物理重排段 → 产出新索引"，属于重建操作而非参数调整，代价见第 5.1 节。

## 3.2 动态 Setting（可在线修改）

```json
PUT /orders/_settings
{
  "number_of_replicas": 2,             // 副本数：调大后立刻触发副本重建与复制流量
  "refresh_interval": "10s",           // refresh 间隔：批量导入时先 -1，导完恢复
  "max_result_window": 20000,          // from+size 最大值：拦住深分页
  "max_terms_count": 65536             // terms 查询最大词项数
}
```

动态 setting 的"能改"也有运维陷阱：`number_of_replicas` 从 1 调到 2 会让集群凭空多出一倍的数据复制流量，业务高峰操作可能把磁盘和网络打满；`refresh_interval` 改成 30s 后要记得批量导入结束**改回来**，否则搜索永远看不到新数据——把"关刷新 → 导入 → 开刷新 → 手动 _refresh"做成固定脚本，而不是靠人肉记忆。

**设计考量**：

- **codec=best_compression**：日志类索引（写多读少、磁盘贵）值得开；搜索类索引（读多）保持默认 lz4，用 CPU 换低延迟。
- **max_result_window 慎调大**：它是保护协调节点堆内存的闸门，调大等于邀请深分页 OOM；真有翻页需求走 search_after。

# 4. 索引别名

别名（alias）是索引的**间接层**——应用永远访问别名，物理索引名随便换：

```json
// 创建别名
POST /_aliases
{
  "actions": [
    { "add": { "index": "orders-2024-01", "alias": "orders" } }
  ]
}

// 切换别名（原子操作）
POST /_aliases
{
  "actions": [
    { "remove": { "index": "orders-2024-01", "alias": "orders" } },
    { "add": { "index": "orders-2024-02", "alias": "orders" } }
  ]
}
```

**别名优势**：无缝切换索引、零停机。`/_aliases` 这个 API 的精髓在于**整个 actions 数组在一个集群状态事务里生效**——不存在"先 remove 后 add"之间的空窗期，客户端视角里 `orders` 要么指向老索引要么指向新索引。若改用"删除别名"再"添加别名"两次请求，中间任何一次搜索都会报错，这正是旧式 `PUT _alias` 单索引 API 被弃用的原因。

别名还可以携带查询条件，把"视图"做进 ES：

```json
// 带过滤的别名：访问 vip_orders 自动附加 filter
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "orders",
        "alias": "vip_orders",
        "filter": { "term": { "level": "vip" } }
      }
    }
  ]
}

// 读写分离别名：一个别名读、另一个别名写
{ "actions": [
    { "add": { "index": "logs-2024.01", "alias": "logs-read" } },
    { "add": { "index": "logs-2024.02", "alias": "logs-read" } },
    { "add": { "index": "logs-2024.02", "alias": "logs-write", "is_write_index": true } }
]}
```

alias filter 让多租户（每个租户一个别名，锁定 `tenant_id` 条件）无需改应用查询语句；`is_write_index` 让一个读别名横跨多个索引、写别名只指向最新一个，这是日志按天滚动的标准接线方式。**注意**：别名不可嵌套别名，指向的索引被删除时别名自动消失，一个别名可以指向多个索引（查询时扇出到全部）。

## 4.1 零停机重建索引

索引建立后的两大不可变项（分片数、字段类型）决定了**改 schema 必须重建**。别名是重建的落地工具，标准五步：

```
┌─────────────┐  ①建v2(新settings/mapping)  ┌─────────────┐
│ orders-v1    │                            │ orders-v2    │
│ (alias:orders│  ②_reindex v1→v2 (后台搬运) │  空→追数据    │
│  读写)       │ ─────────────────────────→ │              │
└──────┬──────┘                              └──────┬──────┘
       │ ③双写窗口: 新数据同时写 v1、v2                │
       ▼                                            ▼
   ┌───────────────────────┐   ④原子切换别名: orders 从 v1 移到 v2
   │ 应用 (只访问 orders)    │   ⑤观察后下线 v1、撤掉双写
   └───────────────────────┘
```

1. `PUT orders-v2` 按新 schema 建索引；
2. `POST /_reindex` 把 v1 数据搬进 v2（可分批、可限速）；
3. 应用开启**双写**：新数据同时写 v1 与 v2（或同步链路写两次）；
4. 数据追平后用一次 `/_aliases` 原子切换 `orders` 指向 v2；
5. 观察无误后删除 v1、关闭双写。

> **行业实践**：reindex 期间用 `requests_per_second` 给搬运限速，避免挤占在线查询；切换动作放在业务低峰，并保留"一键切回 v1"的路径（v1 在观察期内不删）。电商大促前的分片扩容、搜索改分词器，走的都是这套流程。

# 5. 索引模板

模板解决"每天/每月自动出生的索引必须长得一样"的问题，避免人工创建走样：

```json
PUT /_index_template/order_template
{
  "index_patterns": ["orders-*"],
  "priority": 100,
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1
    },
    "mappings": {
      "properties": {
        "order_no": { "type": "keyword" },
        "amount": { "type": "double" },
        "created_at": {
          "type": "date",
          "format": "yyyy-MM-dd HH:mm:ss"
        }
      }
    },
    "aliases": {
      "orders-search": {}
    }
  }
}

// 创建索引时自动应用模板
PUT /orders-2024-02-01   // 匹配 orders-* 模式
```

8.x 只有 composable template 这一代（`_index_template` API），旧版 `_template`（legacy）与 v2（`_index_template` 之前的 index templates API）已淘汰。几个关键语义：**模板只在索引创建瞬间生效，改模板不会回改已存在的索引**；多个模板匹配同一索引名时按 `priority` 竞争、components 按声明顺序合并，**后应用的同名字段覆盖先应用的**；`composed_of` 允许把公共 settings/mappings 抽成组件复用（比如所有日志索引共享一套 ILM 与默认字段）。

模板 + rollover + ILM 三者接力，负责索引"一生"的自动化：模板定义出生形态，ILM 定义成长与死亡：

```
             出生               成长                   死亡
        ┌─────────────┐   ┌──────────────┐   ┌───────────────────┐
        │ Index Template│──→│  Hot: 写入+    │──→│ Warm: forcemerge, │
        │ (匹配 *-logs) │   │  rollover 按   │   │ 只读、可减副本      │
        └─────────────┘   │  大小/天数滚动   │   ├───────────────────┤
                          └──────────────┘   │ Delete: 到期删除     │
                               ▼              └───────────────────┘
                    logs-000001 → logs-000002 → ...
```

ILM 策略通过 `{"policy": "logs-policy"}` 写进模板的 settings；rollover 让别名下的索引写满阈值（主分片大小、文档数或最大年龄）后自动 `logs-000001 → 000002` 滚动，应用完全无感。详细阶段配置在生命周期篇展开，这里记住：**生产日志集群没有手工建索引这回事，一切从模板出**。

# 6. Data Stream（8.x 推荐）

```json
// 创建 Index Template（必须含 data_stream）
PUT /_index_template/logs_template
{
  "index_patterns": ["my-logs*"],
  "data_stream": {},
  "template": {
    "settings": {
      "index.lifecycle.name": "logs-policy"   // 挂 ILM 策略
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "message": { "type": "text" }
      }
    }
  }
}

// 创建 Data Stream
PUT /_data_stream/my-logs

// 写入（自动路由到最新 backing index）
POST /my-logs/_doc
{ "message": "log entry", "@timestamp": "2024-01-15T10:00:00Z" }
```

Data Stream 是把"别名 + 模板 + rollover + 时间分区"四件事打包的高层抽象：对它写入永远落到最新的隐藏 backing index（`logs-my-logs-000001`），查询自动横跨全部 backing index，每个文档强制携带 `@timestamp` 与 `_doc` 语义（**只允许 append，禁止按 ID 更新删除**）。它诞生于 ECS/Beats 日志管道，现在 Metrics、 traces、审计事件都走这套。

**选择依据**：时序追加型数据（日志、指标、事件）用 Data Stream + ILM，运维心智负担最低；业务文档型数据（商品、订单，需要按 ID 增删改查）用普通索引 + 读写别名。两者的分片规划思路一致：**单分片目标 10–50GB，总大小 ÷ 目标分片大小 = 主分片数**，宁可低估分片数（可 split），不要一上来 50 分片造成 small shards problem；每个节点承载的分片数建议不超过堆大小(GB) × 10~20 的经验上限。

> **注意**：DataStream 写入若缺 `@timestamp` 会被 400 拒绝——这是特性不是 bug，时序索引没有事件时间就无法分区与过期。压测或造数时也常忘指定 `@timestamp` 为当前时间，导致数据全挤进同一个 backing index。

# 7. 索引操作

```bash
# 查看
GET /_cat/indices?v                    # 健康/文档数/存储大小总览
GET /orders/_settings                  # 看实际生效的 settings（含模板合并结果）
GET /orders/_mapping                   # 看字段类型（dynamic 猜出来的也在这里现形）

# 删除（不可恢复，副本一起删！备份靠 snapshot）
DELETE /orders

# 关闭/打开（释放堆与文件句柄，关闭后不可读写）
POST /orders/_close
POST /orders/_open

# 刷新/强制合并
POST /orders/_refresh                                  # 手动触发可搜索化
POST /orders/_forcemerge?max_num_segments=1            # 仅限只读索引！

# 清空索引
POST /orders/_delete_by_query
{ "query": { "match_all": {} } }
```

日常运维围绕 `_cat/indices` 的几列信号展开：`health` 看 yellow/red；`store.size` 与 `pri.store.size` 的比值应等于 `1 + 副本数`，不对说明有分片没分配；`docs.count` 在 delete_by_query 后不会立刻下降（计数含尚未被 merge 清除的删除文档）。**`_close` 的定位是归档**：旧索引留在磁盘但不再占堆和句柄，需要再开。

`_forcemerge` 是第 2 篇 merge 机制的人工版：对**已经只读**的索引（时间滚动的旧一天）压到 1 段，搜索延迟与磁盘占用都会肉眼可见地下降；对仍在写入的索引执行它，会把热段冻成巨型段，后续每次更新都要重写整个段，属于典型的"好心办坏事"。`_delete_by_query` 内部是"scroll 查出 → 批量 delete"，大表清空建议直接删索引，删部分数据要带 `conflicts=proceed` 与限速参数，并在完成后手动 refresh。

> **注意**：`DELETE /orders` 没有回收站。误删之后唯一的救命稻草是 snapshot（第 15 篇）；删别名指向的索引时别名一起消失，删之前先确认应用访问的是别名还是裸索引名。

# 8. 小结

| 主题 | 核心结论 |
|------|---------|
| 索引三要素 | settings 管分布式形态（静态多）、mappings 管字段语义、documents 是数据本体 |
| 静态 vs 动态 | 分片数/分析器不可在线改，只能 shrink/split/reindex；副本/refresh/窗口可在线调但有流量陷阱 |
| 别名 | 间接层 + 原子切换 + filter 视图 + is_write_index 读写分离，一切零停机变更的地基 |
| 零停机重建 | 新索引 → reindex → 双写追平 → 原子切别名 → 观察后下线，reindex 必须限速 |
| 模板 | 只在索引出生时生效，改模板不回改历史索引；components 负责复用 |
| Data Stream | 时序追加型数据的最优形态：强制 @timestamp、禁止改删、自动滚动、天生配 ILM |
| 分片规划 | 单分片 10–50GB；宁少勿多，节点分片数不超堆 GB 数的 10–20 倍 |
| 高频操作 | forcemerge 只对只读索引；DELETE 无回收站，备份另靠 snapshot |

索引的生命周期管起来之后，第 5 篇聚焦索引里的最小单元——文档：CRUD API 的完整语义、_bulk 的正确姿势、以及 `_seq_no`/`_primary_term` 乐观并发控制如何在没有事务的引擎上做出"不丢更新"。
