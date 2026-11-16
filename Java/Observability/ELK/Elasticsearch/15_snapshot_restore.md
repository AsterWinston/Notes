快照（Snapshot）是 Elasticsearch 官方认定的灾备手段：它以 Lucene **不可变的段（segment）文件**为粒度，把索引数据复制到共享文件系统或对象存储，支持整库恢复或部分恢复。副本只能防"单台机器坏掉"，快照才能防"数据被错误删除、索引损坏、集群级乃至区域级灾难"，两者是不可互相替代的关系。本篇按 repository 注册 → 创建快照 → SLM 自动化 → 恢复与提速的路径展开，并把快照、副本、CCR、Reindex 四种数据复制手段的分工讲清楚。

# 1. Snapshot Repository

Repository 并不是物理存储位置本身，而是**注册进集群状态的"备份目的地 + 访问方式"**。创建任何快照之前必须先注册仓库；注册这个动作本身只往 cluster state 里写元数据，真正的读写由各数据节点直接对存储发起——所以**仓库路径或 bucket 必须对所有 master-eligible 节点和全部数据节点可见**，fs 类型还要求每个节点挂载的绝对路径完全一致，否则快照会在部分节点上静默失败。

```json
# 注册文件系统仓库
PUT /_snapshot/my_backup
{
  "type": "fs",
  "settings": {
    "location": "/mount/backups/es_backup",
    "compress": true,
    "chunk_size": "1g",
    "max_snapshot_bytes_per_sec": "50mb",
    "max_restore_bytes_per_sec": "100mb"
  }
}

# 注册 S3 仓库
PUT /_snapshot/s3_backup
{
  "type": "s3",
  "settings": {
    "bucket": "es-backups",
    "region": "us-east-1",
    "base_path": "snapshots"
  }
}
```

## 1.1 仓库类型与插件

ES 内置只有 `fs` 和只读的 `url` 两种仓库，对象存储需要安装对应的 repository 插件：

```bash
bin/elasticsearch-plugin install repository-s3
bin/elasticsearch-plugin install repository-gcs
bin/elasticsearch-plugin install repository-azure
# 插件是节点级依赖,每个数据节点与 master 节点都要装,装完必须重启
```

| 类型 | 后端 | 典型场景 | 注意事项 |
|------|------|----------|----------|
| fs | 共享文件系统（NFS/云盘） | 自建机房、小规模集群 | 所有节点同路径挂载；网络存储吞吐常成为第一瓶颈 |
| s3 / gcs / azure | 对象存储 | 云上生产标准做法 | 用 IAM Role/STS 临时凭证，勿把 AK/SK 写进配置文件 |
| url | 只读 http/ftp 路径 | 把别处做好的快照恢复到新集群 | 只读仓库，不能创建快照 |

> **行业实践**：生产优先用对象存储仓库，再配合 bucket 生命周期规则把快照向更冷的存储层归档；自建机房用 NFS 时要实测带宽能否吃满 `max_snapshot_bytes_per_sec`，否则一次全量快照会拖到小时级，挤占业务 IO。

## 1.2 关键 settings

- **compress**：对元数据与 stored fields 做压缩，约省 20-30% 空间，用 CPU 换带宽，对象存储场景建议默认开启。
- **chunk_size**：分块上传粒度，大索引设 1g 可减少请求数；设得过小会放大对象存储的 PUT 次数与费用。
- **max_snapshot_bytes_per_sec / max_restore_bytes_per_sec**：限速保护线上流量，8.x 默认快照 40mb/s、恢复 500mb/s（7.3 之前恢复也是 40mb/s）。恢复窗口紧张的 Dedicated 集群才考虑临时调大或设为 -1 不限速。

# 2. 创建快照

创建快照是**异步**的：master 把快照任务登记进 cluster state，数据节点在后台复制段文件，同一仓库上同时只允许一个快照在跑，第二个请求会直接被拒。`wait_for_completion=true` 只是让 REST 请求阻塞到完成，便于脚本拿同步语义。

```json
# 创建快照
PUT /_snapshot/my_backup/snapshot_20240115?wait_for_completion=true
{
  "indices": "orders,products",
  "ignore_unavailable": true,
  "include_global_state": false
}
```

快照不需要冻结写入，因为**每个 shard 各自获得一个一致性视图**：段文件不可变，快照只需复制"当下已提交"的那组段文件，之后继续写入产生的是新段，天然不会混进本次快照。代价是各分片的"当下"并不在同一毫秒，快照内容接近某个时间点而非全局严格一致——备份场景下完全可以接受。

`include_global_state` 控制是否把**集群设置、索引模板、ILM/SLM 策略**等全局状态一并存进快照。跨集群恢复时建议置 false，否则恢复会被目标集群中不存在或不兼容的全局配置卡住。

# 3. 查看快照

```json
# 所有快照
GET /_snapshot/my_backup/_all

# 单个快照详情
GET /_snapshot/my_backup/snapshot_20240115

# 快照状态
GET /_snapshot/my_backup/snapshot_20240115/_status
```

`_all` 返回仓库内全部快照的目录与各自包含的索引清单，是"索引 X 存在于哪份快照"的常用排查入口。快照的状态有四种：**SUCCESS、IN_PROGRESS、FAILED、PARTIAL**——PARTIAL 表示个别分片失败但快照整体可用，是最容易被忽略的告警点，务必对 `state != SUCCESS` 做监控。进行中的实时进度改用 `GET /_snapshot/my_backup/_stats`。

# 4. 恢复快照

恢复带回的内容包括**数据段、mapping、索引 settings 与别名**；若快照本身没带 global state，则模板、集群配置不在恢复范围内。恢复在实现上等同于"创建索引 + 走 recovery 流程"，因此有几条硬规则：同名且处于打开状态的索引**必须先 close 或改名**；目标集群版本需不低于快照来源（7.x 快照可直接恢复进 8.x，反之不行）；流量同样受 `max_restore_bytes_per_sec` 与快照恢复线程池约束。

```json
# 恢复全部
POST /_snapshot/my_backup/snapshot_20240115/_restore

# 恢复指定索引
POST /_snapshot/my_backup/snapshot_20240115/_restore
{
  "indices": "orders",
  "rename_pattern": "orders",
  "rename_replacement": "orders_restored",
  "include_global_state": false
}
```

## 4.1 部分恢复与重命名

`rename_pattern` 是正则，`rename_replacement` 支持 `$1` 捕获组。典型玩法是整批加前缀恢复到旁路索引，验证无误后再手动切别名，整个过程生产索引不受影响：

```json
# 通配 + 捕获组:orders-2024.* 恢复为 check-orders-2024.*
POST /_snapshot/my_backup/snapshot_20240115/_restore
{
  "indices": "orders-2024.*",
  "rename_pattern": "(.+)",
  "rename_replacement": "check-$1",
  "index_settings": {
    "index.number_of_replicas": 0
  }
}
```

恢复请求里还有一个 `partial: true` 选项：允许只恢复可恢复的分片，其余保持 unassigned。它适合"先让查询服务活过来"的紧急场景，不适合数据完整性要求高的恢复，用完必须尽快补齐分片。

## 4.2 恢复提速

恢复耗时约等于 max（单分片数据量 ÷ 限速， 副本重建时间）。按收益排序的手段：

1. **先 0 副本恢复，完成后再加副本**：如上例把 `number_of_replicas` 置 0，跳过集群内副本复制，一副本集群耗时直接减半，恢复后再 `PUT /orders/_settings` 改回 1；
2. **临时放开限速**：业务低峰期把 `max_restore_bytes_per_sec` 调到 `-1`，恢复完改回默认；
3. **扩节点分摊流量**：分片恢复天然并行，多一台数据节点就多一份并发带宽；
4. **能挂载就不要复制**：对冷数据改用 searchable snapshot（见第 9 节），彻底跳过"下载到本地盘"这一步。

恢复进度用 `GET /_cat/recovery?active_only=true` 观察，按分片显示速率与剩余字节。

> **注意**：恢复出来的索引会经历集中 refresh 与 merge，短时间内写入压力和 master 队列都会上尖，大索引恢复务必安排在低峰期，盯住 `_cluster/pending_tasks` 是否堆积。

# 5. SLM — 自动快照

SLM（Snapshot Lifecycle Management）把手写 crontab 变成集群内一等公民，策略由四部分组成：**schedule**（Quartz 格式定时）、**name**（日期数学 `<nightly-{now/d}>` 按天展开快照名）、**config**（与手动快照相同的参数）、**retention**（`expire_after` 超期、`min_count` 保底份数、`max_count` 封顶）。

```json
# 创建 SLM 策略
PUT /_slm/policy/nightly_backup
{
  "schedule": "0 30 2 * * ?",      // 每天 2:30AM
  "name": "<nightly-{now/d}>",
  "repository": "my_backup",
  "config": {
    "indices": ["orders", "products"],
    "ignore_unavailable": true
  },
  "retention": {
    "expire_after": "30d",
    "min_count": 5,
    "max_count": 30
  }
}
```

配套的运维命令：

```bash
# 立即手动触发一次
POST /_slm/policy/nightly_backup/_execute

# 全局与单策略执行统计(最近失败次数、下次触发时间)
GET /_slm/stats
GET /_slm/policy/nightly_backup/_stats
```

> **注意**：SLM 属商业订阅功能，开源发行版可用 crontab 定时调用 Snapshot API、再按命名规则删除过期快照来达成同等效果。无论哪种方式，都应监控**"最近一次成功快照的时间戳"**而不是"快照任务有没有报错"——任务显示成功但仓库被写坏的极端情况，只能靠定期恢复演练兜底。

## 5.1 保留策略的引用计数

删除一个快照并不会立刻物理删除数据：仓库对段文件做**引用计数**，只有不再被任何现存快照引用的文件才会被清理。因此删最老的快照通常很快，也不会影响较新快照的完整性；`min_count` 则保证即使全部超过 `expire_after`，至少仍保留 N 份可恢复点。

# 6. 增量快照原理

```text
Snapshot 1: [A, B, C]     → 全量
Snapshot 2: [A, B, C, D]  → 仅新增 D（增量）
Snapshot 3: [B, C, D, E]  → 删除 A，新增 E

原理：ES 对比每个段的 hash，只备份变化的段。
```

展开一层：Lucene 的段文件**一旦生成就永不修改**，merge 也只是"写新段、原子切换引用、旧段稍后物理删除"；每个文件带内容校验和，仓库侧维护一份文件清单（manifest）。所谓增量，就是**新快照只上传清单里不存在的文件**。所以快照的存储与 IO 成本取决于"新增/变化了多少段"，而不是"做过多少次快照"——按天滚动、写完即 forcemerge 的日志索引，每天的快照实际只上传当天的新索引，仓库体积增长非常平缓。

```text
时间线          索引分片的段           本次上传
snap-1         {A1, B1}               A1, B1        ← 首次全量
snap-2         {A1, B1, C1}           C1            ← 新写入形成新段
snap-3(merge后) {D1}                  D1            ← 段重写,近似全量
删除 snap-3     -                      仅清理无人引用的 D1
```

**设计考量**：

- **增量意味着仓库不可被"手工修剪"**：对象存储里任何一份段文件缺失或损坏，会同时破坏所有引用它的快照，建议对 bucket 开启版本控制或 Object Lock（WORM）；
- **forcemerge 会造成"变相全量"**：段被重写后下一个快照几乎要整本上传，ILM 把 forcemerge 放在 warm 阶段、快照排在其后，正是为了利用这一次性的完整上传换来之后长期的只读稳定；
- **快照与副本各管一段风险**，见下一小节。

## 6.1 快照 vs 副本：防灾 vs 防单点

| 维度 | Replica 副本 | Snapshot 快照 |
|------|--------------|---------------|
| 防护目标 | 单机故障、节点重启（可用性） | 误删误改、索引损坏、集群/区域级灾难 |
| 误删是否同步 | 是，主分片删了副本跟着丢 | 否，可回到任意快照时间点 |
| 恢复点（RPO） | 秒级（近实时同步） | 上次快照时刻，小时/天级 |
| 恢复时间（RTO） | 秒~分钟，自动 | 分钟~小时，手动或脚本 |
| 存储成本 | 热存储按副本数翻倍 | 对象存储 + 增量，便宜一个数量级 |

> **行业实践**：副本解决"机器会坏"，快照解决"人或程序会犯错"。只有副本没有快照的集群，一次误操作 `_delete_by_query` 或一次勒索加密就会同时击穿全部副本；反过来，想靠快照替代副本也不可行——小时级的 RTO 在线服务根本等不起。

# 7. 跨集群复制 （CCR）

CCR 在操作日志级别做**主动-被动复制**：follower 分片持续重放 leader 分片的写操作，延迟在秒级。

```json
# 在 Follower 集群上
PUT /orders-ccr/_ccr/follow
{
  "remote_cluster": "remote_cluster_name",
  "leader_index": "orders",
  "max_read_request_operation_count": 5000,
  "max_retry_delay": "60s"
}

# 暂停/恢复
POST /orders-ccr/_ccr/pause_follow
POST /orders-ccr/_ccr/resume_follow
```

要认清它的定位：CCR 是"活镜像"而**不是备份**——误删同样会实时同步过来，也不能回到任意历史时间点。它适合跨机房热备（低 RPO）与就近读取的读写分离，配合 follow pattern 可对新索引自动建立复制；复制延迟通过 `GET /_ccr/stats` 监控。与快照对比：CCR 的代价是**一套同等规模的热集群**，快照的代价只是对象存储账单——灾备方案通常两者并用，CCR 保 RPO，快照保可回滚。

# 8. Reindex — 数据迁移

Reindex 的主战场是**改 mapping、改主分片数**（这两者都无法原地修改）、字段结构变换和跨集群搬迁。

```json
POST /_reindex
{
  "source": {
    "index": "orders_old"
  },
  "dest": {
    "index": "orders_new"
  },
  "script": {
    "source": "ctx._source.amount = ctx._source.amount * 1.1"
  }
}

# 跨集群 reindex
POST /_reindex
{
  "source": {
    "remote": {
      "host": "http://old-cluster:9200"
    },
    "index": "orders"
  },
  "dest": {
    "index": "orders"
  }
}
```

几个必须提前处理的坑：dest 索引若不存在会被按默认设置自动创建，**正确做法是先建好索引或索引模板再 reindex**；任务默认单线程滚动，可加 `?slices=auto` 按分片并行，再用 `requests_per_second` 限速防挤占业务写入；remote reindex 要求在 `elasticsearch.yml` 预配 `reindex.remote.whitelist: "old-cluster:9200"`；reindex **不会自动切换别名**，数据核对完成后需手动 `POST /_aliases` 把写别名指向新索引。

```bash
# 并行 + 限速的标准姿势
POST /orders_old/_reindex?slices=auto&refresh&requests_per_second=10000
{
  "source": { "index": "orders_old" },
  "dest":   { "index": "orders_new" }
}
```

# 9. Searchable Snapshot：把对象存储当本地盘

对冷数据而言，"恢复"不再必须把全部段文件拉回本地磁盘。Searchable snapshot 把快照**直接挂载为可查询索引**，段文件留在对象存储上，本地节点只按需缓存被频繁访问的热块：

```json
# 挂载快照为只读索引
POST /orders-2023-quiet/_searchable_snapshot
{
  "snapshot_name": "snapshot_20240115",
  "target_repository": "s3_backup"
}

# ILM cold/frozen 阶段本质就是自动做这件事
PUT /_ilm/policy/logs_policy
{
  "phases": {
    "cold": {
      "actions": {
        "searchable_snapshot": { "snapshot_repository": "s3_backup" }
      }
    }
  }
}
```

查询延迟受对象存储往返（TTFB 几十到几百毫秒）支配，适合合规归档、低频历史日志这类"一年查不了几次但必须能查"的数据。**设计考量**：挂载索引牺牲查询速度换取近乎归零的本地存储；它依赖所有参与节点都持有仓库凭证，仓库权限回收前不要清理旧快照。

# 10. 小结

| 主题 | 核心要点 |
|------|----------|
| Repository | 先注册后用；fs 要求全节点同路径，云存储装插件 + IAM 凭证 |
| 快照创建 | 异步、同仓库串行；分片级一致视图；跨集群恢复建议 `include_global_state: false` |
| SLM | 定时 + 日期数学 + 引用计数保留；监控"最近成功时间"而非任务报错 |
| 增量原理 | 段不可变 + 清单比对，只上传新段；forcemerge 会造成一次变相全量 |
| 恢复 | 先 0 副本、临时放限速、能挂载不复制；PARTIAL 状态必须告警 |
| 分工 | 副本防单点、快照防灾、CCR 保低 RPO 热备、Reindex 管改结构 |

一句话收束：**快照是唯一的"时间机器"，副本只是"空间冗余"**。下一篇（16）进入安全监控。
