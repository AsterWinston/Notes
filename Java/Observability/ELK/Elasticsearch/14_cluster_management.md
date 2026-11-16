# 1. 节点角色配置

集群管理的第一步是把"职责"显式分配给"机器"。ES 节点默认什么都能干，生产规模下会造成三类资源互相踩踏：master 选举被 GC 停顿拖死、写入挤占查询堆、协调合并压垮数据节点。8.x 用 `node.roles` 拆开角色：

```yaml
node.roles: [master]   # Master-only：2C4G, 堆小但必须稳定, 不存数据
node.roles: [data, ingest]   # Data：8C32G+, SSD; 堆 = 物理内存一半且 ≤31GB
node.roles: []         # Coordinating-only：4C8G, 无磁盘需求, 挡在 LB 后面
node.roles: [ml]       # ML 节点（X-Pack 授权）
node.roles: [data_hot] # 8.x 分层角色: data_content/data_hot/data_warm/data_cold/data_frozen
```

**master 节点的独立性是第一位的**：它维护集群状态并做分片分配决策，状态广播与选举对延迟极其敏感，数据节点兼任 master 的风险是"一次 Full GC 就触发选举 + 分片重分配雪崩"。**堆大小铁律：物理内存的一半且不超过 31GB（压缩指针边界），剩下的留给 Lucene 文件系统缓存——页缓存比堆更重要**。`bootstrap.memory_lock: true`（禁 swap）与关闭 THP 是必配项。

## 1.1 生产推荐拓扑

请求链路： LB/VIP → 2× Coordinating（4C8G，无状态）→ Data tier（8C32G+, NVMe）← 集群状态/选举 —— 3× Master-only（2C4G）。

| 规模 | 推荐拓扑 | 说明 |
|------|---------|------|
| ≤3 节点 | master + data 混部 ×3 | 最小可行；注意选举配置 |
| 中等（6-20） | 3 专用 master + data + 2 协调 | 主流形态 |
| 大（>20） | 3 master + 分层 data + 独立协调/ingest | 按 tier 独立扩缩 |

master 固定 3 个（奇数、过半才能选举），data 至少 3 个，单副本才有意义（主副必须分居不同节点）。

# 2. Hot-Warm-Cold 冷热架构

冷热用节点属性 + 分配规则实现：新数据跑 NVMe、老数据沉 HDD 甚至对象存储，本质是"数据价值随时间衰减，而介质成本/性能不同"。8.x 用内置分层角色即自动带上 `_tier` 属性：

```yaml
node.roles: [data_hot, ingest]    # SSD, 高吞吐
node.roles: [data_warm]           # HDD 大容量
node.roles: [data_cold]           # 只读层
node.roles: [data_frozen]         # 不下载到本地, 直接从对象存储按需读
```

```json
# 索引分配(通常由 ILM 自动完成, 手工指定如下)
PUT /logs-2024.01.15
{
  "settings": {
    "index.routing.allocation.include._tier_preference": "data_hot,data_warm,data_cold",
    "index.routing.allocation.require.box_type": "ssd"
  }
}
```

**`_tier_preference`（include 语义）与 `require` 是软硬之别**：前者"优先热层、放不下自动降层"，后者是硬约束，目标层没有节点时分片直接卡 unassigned。分层不只是换机器：warm/cold 层索引应在降层时做 forcemerge、副本降为 0、转 searchable snapshot，否则冷数据仍以"热索引形态"占着便宜机器的 IO。

> **行业实践**：热层按峰值写入吞吐定节点数，冷层按容量定；日志场景常见热冷容量比 1:3~1:6。降层动作交给 ILM 统一做，不要手工搬。

# 3. ILM — 索引生命周期管理

ILM 把"索引何时换机器、何时合并、何时删除"变成策略，是集群长期不烂掉的保障。

```json
PUT /_ilm/policy/logs_policy
{
  "phases": {
    "hot": {
      "actions": {
        "rollover": {
          "max_primary_shard_size": "30GB",
          "max_docs": 50000000,
          "max_age": "1d"
        }
      }
    },
    "warm": {
      "min_age": "3d",
      "actions": {
        "allocate": { "number_of_replicas": 1, "include": { "tier": "warm" } },
        "forcemerge": { "max_num_segments": 1 },
        "shrink": { "number_of_shards": 1 }
      }
    },
    "cold": {
      "min_age": "30d",
      "actions": {
        "searchable_snapshot": { "snapshot_repository": "my_s3_repo" }
      }
    },
    "delete": {
      "min_age": "90d",
      "actions": { "delete": {} }
    }
  }
}

# 关联到索引模板(composable, 8.x 推荐)
PUT /_index_template/logs_template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "index.lifecycle.name": "logs_policy",
      "index.lifecycle.rollover_alias": "logs"
    }
  }
}

# 排错入口: 看索引卡在哪个 phase/step 及原因
GET /_ilm/explain/logs-000001
```

机制细节三条：**min_age 从 rollover 时刻起算，不是文档时间**，日志"保留 90 天"要靠 `max_age: 1d` 滚动 + delete 阶段 90d 共同保证；每个阶段由有序 step 组成（如 forcemerge 依赖磁盘水位未触发），**卡步的唯一排查入口是 `_ilm/explain` 的 `step_info`**;allocate 用 include 是软偏好，配成 require 而目标层无节点时索引会红。日志/时序场景 8.x 首选 data stream（自动 rollover + 生命周期绑定，只能 append）。

> **注意**："索引 3 天了还在 hot"最常见的原因是 ILM 被停（`GET /_ilm/stop` 历史）或 rollover 条件写错导致别名长期不滚。给 rollover 配监控：写索引变更频率异常低就该报警。

# 4. Rollover

Rollover 让"写入永远打到别名，索引在幕后不断换新"，是控制分片大小与索引数量的日常动作。

```json
# 1. 创建带写别名的首个索引(命名零填充保证自然序)
PUT /logs-000001
{
  "aliases": { "logs": { "is_write_index": true } },
  "settings": { "number_of_shards": 3 }
}

# 2. 条件任一满足即滚动
POST /logs/_rollover/logs-000002
{
  "conditions": {
    "max_age": "1d",
    "max_docs": 10000000,
    "max_primary_shard_size": "30gb"
  }
}
```

写别名（`is_write_index: true`）有且只有一个，索引命名零填充保证自然序。滚动条件首选 **`max_primary_shard_size`**——直接对齐"单分片 30-50GB"红线，不受分片数影响；`max_docs` 防文档爆，`max_age` 保证保留期语义，三者常同时配；单纯整索引 `max_size` 在不同分片数的索引间不可比。

> **行业实践**：日志"每天一个索引"是历史习惯，更优解是按分片大小滚动 + max_age 兜底——流量大时一天滚多个，流量小也不会攒出巨型索引。

# 5. 分片数上限与水位线

## 5.1 分片预算

```text
单分片推荐大小: 10-50GB(搜索型) / 50-100GB(日志冷数据)
分片数 ≈ 总数据量 / 单分片大小

硬约束(默认):
  cluster.max_shards_per_node = 1000   # 单节点总分片上限
  经验红线: 每 1GB 堆 ≤ 20 个分片
```

**"分片越多越好"是最普遍的误解**：每个分片是独立的 Lucene 实例，有固定开销（堆内 segment 元数据、文件句柄、集群状态体积），查询时"每分片各排一个 size 的堆再全局归并"。分片数触顶时新索引分配失败并触发 red，治理入口是 shrink（目标分片数须为原数的因子）、split 与 searchable snapshot，前两者要求索引先只读：

```json
PUT /logs-2024.01.15/_settings
{ "index.blocks.write": true }
POST /logs-2024.01.15/_shrink/logs-small
```

## 5.2 磁盘水位线与 flood_stage 只读死锁

```yaml
# elasticsearch.yml
cluster.routing.allocation.disk.watermark.low: 85%          # 不再把新分片分配进来
cluster.routing.allocation.disk.watermark.high: 90%         # 开始把分片迁走
cluster.routing.allocation.disk.watermark.flood_stage: 95%  # 触发只读保护
```

```json
# 动态修改
PUT /_cluster/settings
{
  "transient": {
    "cluster.routing.allocation.disk.watermark.low": "80%"
  }
}
```

**flood_stage 的真实后果常被低估**：超过阈值时，ES 给相关索引自动加上 `index.blocks.read_only_allow_delete: true`——写入、删除全部失败，而"清理数据腾空间"本身需要写能力，**形成死锁：盘满 → 只读 → 无法删 → 继续满**。恢复动作有固定顺序：

```json
# 1. 先腾空间: 删旧索引/扩盘/临时调高水位线
# 2. 再显式解除只读锁(水位回落不会自动解除!)
PUT /_all/_settings
{ "index.blocks.read_only_allow_delete": null }

# 3. 验证
GET /_cat/allocation?v
```

> **警告**：解锁前必须确认空间真的腾出来了，否则锁会原样加回来。根治靠"提前量"：low/high/flood 之间的余量要能容纳分片迁移的双份占用与 ILM 删除延迟。

# 6. 集群健康与 yellow/red 排查

```bash
GET /_cluster/health
GET /_cat/shards?v&h=index,shard,prirep,state,unassigned.reason
GET /_cluster/allocation/explain        # 权威答案: 分片为什么不分配
```

| 状态 | 含义 | 影响 | 常见原因 |
|------|------|------|---------|
| **green** | 主分片 + 全部副本已分配 | 无 | 正常态 |
| **yellow** | 主分片全分配，有副本未分配 | 读写正常，冗余降级 | 只剩 1 个数据节点；水位线阻止分配；刚挂节点 |
| **red** | 有主分片未分配 | 部分数据查不到、写可能失败 | 节点数据丢失；分片数触顶；只读锁卡删除 |

**yellow 常是正常态**（单数据节点 dev 集群永远 yellow），要不要管看 `unassigned.reason` 与持续时间。red 的标准动线：

```text
red
 ├─ 1. GET /_cluster/allocation/explain → 读 explanation 第一条原因
 ├─ 2. 节点掉线? 能回来就等(delayed_timeout 默认 1m); 回不来从 snapshot restore
 ├─ 3. max_shards_per_node 触顶? → 提高上限或删旧索引/降副本
 ├─ 4. 水位线/flood 只读? → 腾空间 + 解 read_only_allow_delete
 └─ 5. 原因修复后: POST /_cluster/reroute?retry_failed=true
```

> **注意**：`retry_failed=true` 清掉"分配失败黑名单"让 ES 重试，能解一部分卡死，但不解决根因（盘满、节点没了）。对 red 索引的 `allocate_empty_primary` 等于放弃该分片数据，做之前必须有快照恢复方案。

# 7. Reroute 与分配控制

```json
# 手动移动分片(目标节点已有该分片副本时)
POST /_cluster/reroute
{
  "commands": [
    { "move": { "index": "orders", "shard": 0, "from_node": "node-old", "to_node": "node-new" } }
  ]
}

# 下线节点的标准姿势: 用 exclude 赶分片, 而不是直接停进程
PUT /_cluster/settings
{ "persistent": { "cluster.routing.allocation.exclude._name": "node-to-remove" } }

# 分配开关与恢复限速(滚动升级窗口)
PUT /_cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.enable": "new_primaries",
    "cluster.routing.rebalance.enable": "none",
    "indices.recovery.max_bytes_per_sec": "100mb"
  }
}

# 修改副本数
PUT /orders/_settings
{ "number_of_replicas": 2 }
```

**reroute 是手术刀，不是日常工具**：手动命令的合理用途只有三类——节点下线（exclude）、取消坏副本（cancel）、放弃数据的紧急恢复（allocate_stale/empty_primary，必须带 `accept_data_loss: true`）。滚动升级期间先关 rebalance、限 recovery 速率，节点到位后再放开，否则每个节点重启都触发一轮搬迁风暴。

## 7.1 分配感知

```yaml
# elasticsearch.yml
cluster.routing.allocation.awareness.attributes: rack_id
node.attr.rack_id: rack1
cluster.routing.allocation.awareness.force.rack_id.values: rack1,rack2
```

**效果**：主分片和副本不会分配到同一个 rack，提高可用性。不配 force 时，只有 rack1 在线也会照常分配（集群可用但容灾承诺失效）；配了 force 后 rack2 必须有节点，否则分片未分配——**用可用性换"绝不妥协的容灾布局"**。

# 8. Snapshot 与 SLM

集群管理的兜底能力是备份。SLM（Snapshot Lifecycle Management）把"定期快照 + 自动过期"做成内置策略，替代手工 crontab。

```json
# 1. 注册仓库(S3 示例, 需 repository-s3 插件)
PUT /_snapshot/my_s3_repo
{
  "type": "s3",
  "settings": { "bucket": "es-backups", "compress": true, "server_side_encryption": true }
}
PUT /_snapshot/my_s3_repo/_verify     # 验证仓库可用

# 2. SLM 策略: 每 30 分钟增量快照, 保留 21 天
PUT /_slm/policy/prod-snapshots
{
  "schedule": "0 */30 * * * ?",
  "name": "<prod-snap-{now/d}>",
  "repository": "my_s3_repo",
  "config": { "indices": "*", "include_global_state": true },
  "retention": { "expire_after": "21d", "min_count": 5, "max_retries": 5 }
}

# 3. 运维: 状态与手动执行
GET /_slm/policy/prod-snapshots/_stats
POST /_slm/policy/prod-snapshots/_execute
```

**快照是增量且段级去重的**：重复的 segment 文件不复制，"每 30 分钟一次"的存储成本远低于直觉；但它不是实时备份，最近一次快照之后的写入不在里面。`include_global_state` 决定是否备份模板/ILM/SLM/安全配置。恢复时可用 `rename_pattern` 把索引恢复到新名字做验证，避免与在线数据冲突。

> **行业实践**：备份的验收标准是**定期恢复演练**：隔离集群里 restore + 核对文档数。从未验证过的快照等于没有；仓库最好加对象锁定（WORM）与独立凭据，否则误删/勒索会把备份一起带走。

# 9. 跨集群能力

| 能力 | 机制 | 数据搬迁 | 典型用途 |
|------|------|---------|---------|
| CCR（跨集群复制） | leader/follower 索引按 seq_no 追 | ✓ | 异地容灾、就近读 |
| CCS（跨集群搜索） | 查询扇出到远程集群 | ✗ | 多机房统一查询 |
| 快照恢复 | 共享 repository | ✓ | 迁移、归档 |

CCS 的性能前提是**用明确索引名剪枝**：远端 `logs-*` 通配会向所有匹配索引扇出，一个远端抖动拖慢全局。CCR 的 follower 索引只读，leader 端必须开安全认证，且 leader 的 translog 留存要覆盖 follower 的最大落后时长，否则复制断链只能 unfollow + 重建。**拆多集群的边界是"数据主权/故障域/独立扩缩"，不是"装不下就拆"**。

# 10. 扩缩容与维护实践

```bash
# 扩容: 新节点直接作为目标 tier 的 data 节点加入
# 先禁止搬迁让节点"到齐"再统一平衡, 避免来回搬
PUT /_cluster/settings
{ "persistent": { "cluster.routing.allocation.enable": "new_primaries" } }

# ... 节点到位后恢复 all, 观察搬迁进度
GET /_cat/recovery?active_only=true&v
GET /_cluster/health?wait_for_status=green&timeout=1h

# 缩容: 先赶分片再停机
PUT /_cluster/settings
{ "persistent": { "cluster.routing.allocation.exclude._name": "node-9" } }
```

日常维护：全集群同版本 + 同插件（升级走滚动重启）；监控盯 `number_of_pending_tasks`、unassigned_shards、heapPercent 持续 >75%、thread_pool rejected、ILM 卡步数、SLM 失败数。容量规划三条红线——单分片 ≤30-50GB、每 GB 堆 ≤20 分片、数据盘长期 ≤70%，任何一条被突破，问题会以"偶发超时"或"随机 red"出现，排查成本远高于当初多买一台机器。

# 11. 小结

| 主题 | 核心结论 |
|------|---------|
| 角色与堆 | 3 专用 master + 分层 data + 独立协调；堆 = RAM 一半 ≤31GB，页缓存优先 |
| 冷热 | `_tier_preference` 是软偏好，require 是硬约束会卡分配；降层伴随 forcemerge/降副本 |
| ILM | min_age 从 rollover 时刻起算；卡步看 `_ilm/explain` 的 step_info |
| Rollover | 条件首选 max_primary_shard_size + max_age 兜底；命名零填充、写索引唯一 |
| 分片预算 | 每 GB 堆 ≤20 分片、max_shards_per_node 默认 1000；治理靠 shrink/split |
| 水位线 | flood_stage 加只读锁且**不会自动解除**；先腾空间再显式清 null |
| health | yellow 常为正常；red 从 allocation/explain 起步；allocate_empty_primary = 放弃数据 |
| reroute | 手术刀；滚动变更先关 rebalance、限 recovery 速率，下线用 exclude |
| SLM | 增量快照便宜，include_global_state 别漏；必须定期恢复演练 |
| 跨集群 | CCS 靠明确索引名剪枝；CCR 只读 follower，注意 translog 留存覆盖落后时长 |

集群管理收束了 ES 主线的"单文档 → 分片 → 集群 → 生命周期"；15_snapshot_restore 与 16_security_monitoring 分别展开备份与安全的细节，18_best_practices/20_interview_qa 做全链路复盘。
