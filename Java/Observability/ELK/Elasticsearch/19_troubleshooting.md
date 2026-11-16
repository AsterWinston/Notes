排错是 ES 运维的"期末考"：症状高度收敛（集群 red、查询变慢、写入被拒、节点掉线），根因却分布极散（磁盘、堆、分片、配置、上游流量）。高效排错的诀窍不是记住命令，而是**固定一条"状态 → 分片 → 资源 → 任务"的下钻路径**，每一步都有对应的 API 输出可读。本篇按症状分章，最后给出决策流程图与速查命令集。

# 1. 集群状态异常

集群颜色是对"分片安置结果"的摘要，拿到报警第一步永远是确认级别与范围：

```bash
# 查看健康
GET /_cluster/health
GET /_cluster/health?level=indices  # 每个索引健康

# Red: 有主分片未分配
# Yellow: 所有主分片 OK，部分副本未分配
# Green: 正常

# 查看未分配分片原因
GET /_cat/shards?v&h=index,shard,state,unassigned.reason
```

三种颜色的业务含义要分得清：**green** 全部分片安置；**yellow** 数据完整可用，只是冗余度打折（单节点集群常态 yellow，不是故障）；**red** 至少一个主分片不可用——**包含该分片的文档查询直接缺失**，且该索引写入可能受阻。判断影响面用 `level=indices` 找出 red 的具体索引，再决定是"全量响应"还是"局部降级"。

还有一个常被忽略的中间态：集群状态是 green/yellow，但 `_cluster/health` 里的 `relocating_shards`、`initializing_shards`、`unassigned_shards` 持续 nonzero 且 `active_shards_percent_as_number < 100`，说明分配器在反复重试——颜色会骗人，计数器不会。

## 1.1 状态不变化：先看 master 在不在

集群状态由 master 计算下发，如果 `GET /_cat/master` 失败或频繁切换，一切"分片不动"的现象都是master 问题的下游症状。master 节点的 CPU、磁盘（cluster state 写入）、网络延迟应最先排查——这与 18 篇"3 个独立故障域 master"的纪律互为因果。

# 2. Unassigned Shard

unassigned 的原因分类是排错的核心知识，`_cat/shards` 的 `unassigned.reason` 列直接给出：

| 原因 | 含义 | 典型处置 |
|------|------|----------|
| NEW_REPOSITORY / INDEX_CREATED | 新索引/新仓库恢复，等待分配 | 等自动分配，看是否有分配约束（awareness、tier）挡住 |
| CLUSTER_RECOVERED | 集群重启恢复中 | 等 `cluster.routing.allocation.node_concurrent_recoveries` 排队完成 |
| NODE_LEFT | 原持有节点离开，副本/主分片待安置 | 节点回来则自动恢复；不回则等超时后重分配 |
| INDEX_CLOSED / REOPENED | 索引关闭再打开触发重算 | 正常现象 |
| ALLOCATION_FAILED | 分配尝试失败达上限（映射损坏、磁盘不足、shrink 名冲突等 10 次） | **必须查 explain 里的失败原因**，修复后手动 `retry_failed` |
| REALLOCATED_REPLICA | 为均衡负载/升级而主动重定位 | 等待搬迁完成 |
| FORCED_EMPTY_PRIMARY | 人工用 `allocate_empty_primary` 清空（数据已丢） | 确认是否该从快照恢复 |

定位工具是 allocation explain——把分配器的每个否决理由摊开：

```bash
# 1. 查看详细原因
GET /_cluster/allocation/explain
{
  "index": "orders",
  "shard": 0,
  "primary": true
}

# 2. 解决方案：
# - 副本未分配 → 等待自动恢复
# - 磁盘满 → 清理磁盘或扩容
# - 分片超限 → 减少节点分片数或增加节点
# - 节点离线 → 恢复节点或强制重新分配

# 3. 强制分配（慎重！）
POST /_cluster/reroute?retry_failed=true
```

explain 输出读三段：`can_allocate`/`allocate_decision`（NO 就是被挡）、`allocate_allocation_id`、以及逐节点的 `node_allocation_decisions`——每个节点的 `deciders` 数组写明否决者，常见的有 `disk_threshold_decider`（水位）、`shards_limit_decider`（600 上限）、`same_shards_decider`（主副不同盘）、`enable_allocation_decider`（`cluster.routing.allocation.enable` 被临时关了）。

> **注意**：`allocate_empty_primary` / `allocate_stale_primary` 这类强制命令等于在说"我知道这分片数据烂了/丢了，照我说的办"，用错就是永久丢数据；只有两种合法场景——磁盘物理损毁且快照时间点在事故之后要抢救，或测试集群无所谓数据。生产第一步永远回到 15 篇的快照恢复。

主分片与副本的 unassigned 处置节奏完全不同：副本挂了 ES 会自动在别处重建（有源可读）；**主分片挂了要把最好的副本 promote**，若全部副本都不在只能从 snapshot restore——所以 red 索引先查 `GET /orders/_cat/shards` 确认主副还剩几份。

# 3. 写入被拒（429）

429 有两个来源，处置南辕北辙，**先用响应体的 `reason.type` 分清**：

```text
# 错误信息：
TOO_MANY_REQUESTS/12/disk usage exceeded flood stage watermark

# 原因：磁盘使用超过 95%
# 解决：
# 1. 清理磁盘
# 2. 提高 watermarks（临时）
PUT /_cluster/settings
{
  "transient": {
    "cluster.routing.allocation.disk.watermark.flood_stage": "97%"
  }
}

# 3. 删除旧索引
DELETE /old-index-*
```

第一类是 **flood stage 索引只锁**：磁盘越过 95% 后 ES 给所有索引加 `index.blocks.read_only_allow_delete`，写入报 429。解除锁必须两步：先让磁盘真正降到水位线下（清理/扩容/删旧索引），**再手动清掉残留的 block**——ES 知道水位恢复后会自动解锁，但"水位计算周期 + block 清除"偶有不及时，确认手段：

```json
PUT /_all/_settings
{
  "index.blocks.read_only_allow_delete": null
}
```

第二类是 **write 线程池队列打满**：上游并发/批量超过集群处理能力，`_cat/thread_pool/write` 里 `queue` 顶满、`rejected` 上涨。这时改水位没用，方向是限流（调低 bulk 并发、`requests_per_second`）、扩容、或按 13 篇的索引优化（refresh_interval、translog async）。临时把 `write` 线程数调大只治标——线程池容量背后是 CPU 与磁盘 IO。

> **行业实践**：容量与拒绝的告警线要设在 flood stage **之前**——85% 告警、90% 介入、95% 已经只读锁。把 read-only 锁当"紧急刹车"而不是"日常红灯"，刹车用完还得回头查为什么一路没拦。

# 4. OOM / GC 频繁

堆相关的死法主要有四种：大聚合/大排序把 request 熔断打到跳、fielddata 加载 text 字段排序撑爆、分片过多导致元数据+global ordinals 常驻内存、以及 OS Cache 挤占触发系统 OOM killer。先看现场：

```bash
# 1. 查看 JVM
GET /_nodes/stats/jvm

# 2. 热点线程
GET /_nodes/hot_threads

# 3. 解决方案：
# - 增加堆内存（≤ 32GB）
# - 减少分片数
# - 关闭 fielddata（改用 doc_values）
# - 限制聚合桶数量
# - 增大 GC 阈值
```

**fielddata 专项**：正常配置下 fielddata 内存应恒为 0——它只在"对 text 字段排序/聚合/script 取值"时被加载。`GET /_nodes/stats/indices/fielddata?human` 一旦 nonzero，去慢日志和 profile 里揪出那条 `sort: description` 或 `aggs on text` 的查询，改 keyword 子字段。若字段曾被手工设了 `"fielddata": true`，先清缓存止血，再改回 mapping（新数据生效，存量索引需 reindex）：

```bash
POST /orders/_cache/clear?fielddata=true   # 清掉已加载进堆的 fielddata
PUT /orders/_mapping
{ "properties": { "description": { "type": "text", "fielddata": false } } }
```

GC 判断经验：`GET /_nodes/stats/jvm` 看 `gc collectors.old_collection_count` 增速与 `old_collection_time_in_millis`——**old GC 频繁（分钟级）就是堆里常驻物太多**，方向是减分片、清缓存、查 global ordinals（大量 keyword 高基数字段 + 大聚合会重建 ordinals）；young GC 高频则是大结果集/大聚合流量问题，回到查询侧治理。真要 OOM（进程被杀），日志里先看是 java.lang.OutOfMemoryError（堆内）还是被 kernel oom_kill（OS 层，通常因为堆占比超了 50%）。

# 5. 慢查询定位

固定五步：**发现 → 抓现行 → 定性 → 归因 → 处置**。

```bash
# 1. 开启慢查询日志
PUT /orders/_settings
{
  "index.search.slowlog.threshold.query.warn": "2s",
  "index.search.slowlog.level": "info"
}

# 2. 查看正在执行的慢查询
GET /_tasks?actions=*search*&detailed=true

# 3. Kill 慢查询
POST /_tasks/task_id:xxx/_cancel

# 4. Profile 分析
GET /orders/_search
{ "profile": true, "query": { ... } }
```

```text
发现(P99 报警/Dashboard) → _tasks?actions=*search*&detailed 抓现行(能 cancel)
        │                              │
        │ 已结束的慢查询               │ 正在跑的
        ▼                              ▼
 slowlog 取 query 原文 ────────→ 同一份 DSL 拿去 _profile
                                          │
                          分解: query phase(打分/遍历) vs fetch phase(取文档)
                                          │
              归因: 全表扫描型(wildcard/query_string) / 桶爆炸型(聚合) / 深分页型 / 大文档 fetch 型
```

`profile: true` 看每个分片上 query 阶段的 breakdown：`BooleanQuery` 里哪个 clause 的 `seek/count` 最大，就是慢点；配合 `?verbose=true` 能看到具体 scorer 耗时。`_tasks` 的 `running_time_in_nanos` 与 `description`（截断的 DSL）能直接回答"是谁把集群打慢的"，确认后 `POST /_tasks/<id>/_cancel` 止血，再回头改调用方的查询或加 mapping。fetch phase 慢多半是 `_source` 巨大——解法是 `stores._source` includes、或把大文本移出索引只存引用。

> **注意**：cancel 只中止单个请求，不解决"为什么会有这种请求"；慢查询定位闭环必须落到**调用方归因**（API Key 维度）与**护栏设置**（max_buckets、default_timeout），否则下周同一时间还会来一遍。

# 6. 数据不一致

"写进去查不到"大概率不是丢数据，而是**没 refresh**（近实时语义，见 20 篇）或读写走了不同索引/别名。确认顺序：

```bash
# 1. 检查分片分布
GET /_cat/shards/orders?v

# 2. 强制同步
POST /orders/_stats?level=shards

# 3. 重新分配副本
POST /_cluster/reroute?retry_failed=true

# 4. 必要时重建副本
# 关闭索引 → 修改副本数为 0 → 重新设置副本数
```

真正的主副数据分叉（历史故障 + 强制分配遗留）表现为"同一文档在不同分片副本上命中结果不同"，最干净的处理是让坏副本重同步：置 `number_of_replicas: 0` 再恢复，或 reroute 强制 relocate；有快照时用 restore 到旁路索引比对，才是数据正确性的终极裁判。文档 id 级比对可用 `GET /orders/_doc/<id>` 对照 `_seq_no/_primary_term`。

# 7. 脑裂与 master 选举

旧世界（7.7 之前 Zen discovery）的脑裂：网络分区两侧各选出 master，各自接受写入，合并时数据无声丢失。防御三件套：**最少 3 个 master-eligible**（quorum = N/2+1）、`discovery.zen.minimum_master_nodes` 显式设置（当年默认值是 1，默认即脑裂）、独立故障域。

8.x 的 **cluster formation 机制 + voting configuration** 已从原理上消灭脑裂：master 选举需要拿到**多数投票**，votes 名单在配置变更时原子重算，分区少数侧选不出 master 只会停摆，不会出现双主。排错视角因此从"防脑裂"变成"防选不出"：

```bash
GET /_cat/master?v
GET /_cluster/voting_config_exclusions          # 被剔除出投票组的节点
# 下线 master 节点前应先排除投票，防止票数算不平:
POST /_cluster/voting_config_exclusions?node_names=master2
# 移除完成后再取消:
DELETE /_cluster/voting_config_exclusions
```

"集群反复切主 / master 事件刷屏"的三大根因：master 节点 GC 停顿或 CPU 饱和（心跳超时）、网络抖动（transport 延迟 > `cluster.election.initial_timeout` 量级）、以及把高负载数据节点也设为 master-eligible。治理办法就是 14/18 篇的拓扑纪律：3 台专职小规格 master + 独立故障域 + 下线走 voting exclusions 流程。

# 8. 排错决策流程

```text
报警/用户反馈
   │
   ├─ 集群红/黄? ──是──→ _cat/shards 找 unassigned → allocation/explain
   │                        ├─ 磁盘水位 → 清盘/扩容/降水位(见第3章)
   │                        ├─ 节点离线 → 恢复节点 or 等自动重排
   │                        └─ ALLOCATION_FAILED → retry_failed + 修根因
   │
   ├─ 查询慢? ──→ _tasks 抓现行(可 cancel) → slowlog 取历史 → _profile 归因
   │                ├─ 桶爆炸/深分页/通配符 → 查询治理 + max_buckets 护栏
   │                └─ 分片数/段数过多 → forcemerge/重规划(18 篇)
   │
   ├─ 写入 429? ──→ 看 reason.type
   │                ├─ flood stage → 清盘 → 确认 read_only block 解除
   │                └─ write rejected → 限流/扩容/索引写入优化
   │
   ├─ 堆告急/OOM? → _nodes/stats/jvm + fielddata + hot_threads
   │                ├─ fielddata>0 → 揪 text 排序查询, 改 keyword
   │                └─ old GC 频繁 → 减分片/减高基数聚合/扩堆(≤31g)
   │
   └─ 反复切主? → _cat/master + master 节点 GC/网络 → 修拓扑(专职 3 master)
```

# 9. 快速诊断命令

```bash
# 集群状态
GET /_cluster/health
GET /_cat/nodes?v
GET /_cat/indices?v&s=store.size:desc

# 资源
GET /_nodes/stats/jvm,os,process
GET /_cat/thread_pool?v

# 分片
GET /_cat/shards?v&h=index,shard,prirep,state,unassigned.reason

# 任务
GET /_cat/tasks?v
GET /_tasks?actions=*search*,*write*
```

# 10. 小结

| 症状 | 第一命令 | 第二命令 | 多数根因 |
|------|----------|----------|----------|
| red/yellow | `_cluster/health` | `allocation/explain` | 节点离线、磁盘水位、分配失败 |
| 查询慢 | `_tasks?detailed` | `_profile` + slowlog | 桶爆炸、深分页、通配符 |
| 写入 429 | 看 reason.type | `_cat/thread_pool/write` | flood stage、队列打满 |
| OOM/GC | `_nodes/stats/jvm` | fielddata 统计 + hot_threads | 分片过多、text fielddata、大聚合 |
| 反复切主 | `_cat/master` | voting_config_exclusions | master 资源不足、网络抖动 |

排错的通用心法只有一句：**先读集群自己的解释（explain/tasks/stats），再动手改配置**；所有"强制"命令都是最后手段。下一篇（20）把高频考点整理成面试问答。
