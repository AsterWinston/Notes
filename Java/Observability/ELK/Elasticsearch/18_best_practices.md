前面各篇解决"ES 会做什么"，本篇解决"生产上应该怎么做"。ES 的绝大多数线上事故——分片爆炸、mapping 漂移、堆内存打满、磁盘只读——都不是引擎缺陷，而是**索引设计、容量规划、查询与运维纪律**四处失守。本篇按这五块给出可执行的规则，并在篇末汇总成一张对照大表。

# 1. 容量规划

容量规划的正确顺序是：**先估数据量与增速，再定分片，最后倒推节点数与堆大小**，而不是先买机器再塞分片。

```text
分片数计算:
  单分片推荐大小: 30-50GB（搜索密集）/ 50-100GB（日志密集）
  分片数 ≈ 总数据量(TB) × 1024 / 单分片大小(GB)

  示例: 5TB 订单数据, 单分片 50GB → 约 100 个主分片

节点数计算:
  每个节点 ≤ 600 分片（总主+副）
  节点数 ≥ (总分片数 / 600) × 1.5（冗余）

内存计算:
  堆内存: 节点内存的 50%, 上限 32GB（压缩指针限制）
  OS Cache: 用于 Lucene 文件缓存（至少留 50% 内存）
```

三条规则各自的原理值得展开。**分片大小**：每个分片是一个 Lucene 实例，搜索时"逐分片建堆、协调节点合并"的成本随分片数线性上涨，而单分片过大又会让恢复、迁移、ILM shrink 都变慢，30-50GB 是搜索与运维弹性的平衡点。**每节点 600 分片**：官方以"每 GB 堆内存不超过 20 个分片"为经验值（16GB 堆 ≈ 320 分片，600 是绝对上限），分片元数据（segment 信息、global ordinals）常驻堆，分片过万是 master 压力与堆爆炸的头号来源。**堆 32GB 上限**：JVM 压缩对象指针（compressed oops）在堆 ≤ 31GB（实际取 30-31g 留安全余量）时按 4 字节寻址，越过界限指针膨胀，多给的内存反而浪费了堆内有效对象容量——这就是"给 64GB 堆只配 31GB，剩下全留给 OS Cache"的原因。

> **注意**：日志类索引要按**保留期总数据量**而不是单日数据量算分片。单日 100GB 保留 90 天就是 9TB；此时正确解不是每天 100 个分片，而是"每天滚动索引 + 按天定分片数（如 1-3 个）+ ILM 逐层搬迁 shrink"，让老索引自动瘦身。

## 1.1 副本与磁盘冗余

副本数直接乘在磁盘账上（1 副本 = 2 倍存储）。写多读少的日志集群 1 副本足够，绝不能为省磁盘上 0 副本——单节点故障即 red。规划磁盘时为**段合并、转储、快照缓存、Lucene 临时文件**预留至少 20-25% 余量，并确保磁盘水位线（85%/90%/95%）落在真实容量规划之内，而不是等 flood stage 来当容量告警。

# 2. JVM 调优

```bash
# config/jvm.options
-Xms16g -Xmx16g
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:G1HeapRegionSize=8m
-XX:InitiatingHeapOccupancyPercent=30
-XX:+DisableExplicitGC

# 堆内存 ≤ 32GB（compressed oops 限制）
# 堆内存 = 物理内存的 50%
```

8.x 默认就是 G1，**堆大小是真正需要决策的旋钮，GC 参数基本不用动**。`-Xms` 与 `-Xmx` 相等避免运行时伸缩；`DisableExplicitGC` 防止 System.gc()（NIO 直接内存回收触发）打断查询造成毛刺。ES 官方明确建议：**不要换成 ZGC/Shenandoah 或自调一堆 GC flag**——G1 在 ES 场景（大堆、高分配率、段缓存驻留）下久经验证，收益最大的永远是把查询和分片数治理好，而不是调 GC。另外配合系统层：`bootstrap.memory_lock: true` 锁定内存防 swap（配 `LimitMEMLOCK=infinity`），关闭 swap 分区，`vm.max_map_count=262144`（Lucene mmap 默认值不够）。

# 3. 索引设计：滚动、别名、ILM

时序数据的第一设计原则：**不要把日志写进一个大索引**。主分片数不可修改，单索引会无限膨胀；按时间滚动 + 别名统一读写，是所有日志/指标集群的地基：

```text
写入 ──→ 别名 logs（指向当前写索引 logs-000001）
                │  rollover（超 30GB 或 30 天）
                ▼
ILM 生命周期:  logs-000001  hot → warm → cold → frozen → delete
               新数据永远只进"写索引"，历史索引只读
```

- **按天滚动还是按大小 rollover**：数据量波动大的业务（日志）用 rollover 条件 `max_size/max_primary_shard_size + max_age` 兜底，保证分片均匀；合规要求"整天在一个索引"就用按天 date-math 命名。两者都以别名收口，查询端不需要感知物理索引名。
- **ILM 分层 hot-warm-cold-frozen**：hot 承接写入与高频查询（SSD、副本高）；warm 做 forcemerge + shrink 降副本（HDD）；cold 只读、更低硬件；frozen 用 searchable snapshot 把数据留在对象存储、本地只缓存热块。层级就是"数据价值随时间衰减"的定价表。

```json
PUT /_ilm/policy/logs_policy
{
  "phases": {
    "hot":    { "actions": { "rollover": { "max_primary_shard_size": "50gb", "max_age": "30d" } } },
    "warm":   { "min_age": "2d", "actions": {
                 "shrink": { "number_of_shards": 1 },
                 "forcemerge": { "max_num_segments": 1 } } },
    "cold":   { "min_age": "30d", "actions": { "set_priority": { "priority": 50 } } },
    "delete": { "min_age": "90d", "actions": { "delete": {} } }
  }
}
```

> **行业实践**：rollover 之后再做 shrink/forcemerge（ILM 顺序保证这点）——在正在写入的分片上 forcemerge 是重 IO 灾难；warm 阶段"一次 merge 成单段"后数据不可变，后续快照也只复制增量，多个最佳实践在这里咬合。

# 4. Mapping 设计纪律

```json
{
  "properties": {
    "order_no":    { "type": "keyword" },      // 精确匹配、聚合、排序 → keyword
    "description": { "type": "text" },         // 全文检索 → text
    "status": {
      "type": "keyword",
      "norms": false                            // keyword 不需要评分
    },
    "created_at":  { "type": "date", "format": "strict_date_optional_time||epoch_millis" }
  },
  "dynamic": "strict"                           // 生产环境禁止动态映射
}
```

**设计考量**：

- **keyword vs text 是第一决策**：text 分词后进倒排索引，支持 match 相关性检索但不能直接排序/聚合（需 fielddata，危险）；keyword 不分词，精确 term 查询、聚合排序的载体。"既要点搜索又要聚合"用多字段 `fields: { keyword: { type: keyword, ignore_above: 256 } }`；
- **dynamic 一律 strict 或 false**：动态映射会在脏字段写入时把 `status` 推断成 long、把对象数组推断成 nested 之外的形态，之后只能 reindex 返工；确需灵活字段就显式用 `dynamic_templates` 定规则；
- **能不存就不存**：不检索的字段 `"index": false`，不需要 `_source` 之外的 stored fields 别开，`ignore_above` 截断超长 keyword，`enabled: false` 让纯展示对象跳过索引；
- **nested 是成本项**：每个 nested 子文档是隐藏文档，数量翻倍、写入放大；数组只有"需要子元素间关联查询"时才用 nested，否则 plain object/keyword 数组即可；
- **索引永远由模板管理**：`composable template + aliases` 进 CI，禁止业务代码"写第一条时顺手建索引"。

# 5. 查询纪律

| 纪律 | 反例 | 正解 |
|------|------|------|
| 分页深度 ≤ 10000 | `from: 900000` | `search_after` + PIT；导出用 scroll/async |
| 聚合桶数受控 | 大时间范围 date_histogram(1m)+terms 嵌套 | 提高 interval、composite 分页、估算桶数 |
| 排序字段轻量 | 按 text 排序 | keyword/doc_values 排序，避免 script sort |
| 通配符不打头 | `query_string: "*phone*"` | prefix / `wildcard` 字段类型（8.x）/ ngrams |
| 精确定分用 filter | 全 bool+must | filter/rank：可缓存、不计分 |
| 大结果集瘦身 | fetch 全部字段 | `_source` includes 只取所需 |

**设计考量**：查询治理最有效的抓手其实只需三个集群级护栏：**慢查询日志**（warn 阈值 1s 起步）、**请求熔断 request circuit breaker**（兜大聚合）、**API Key 分应用授权**（出问题能定位到调用方）；再用 `search.max_buckets`、`index.max_result_window`、`search.default_search_timeout` 三个 setting 把最危险的查询形态直接拒掉。与其培训所有人写好 DSL，不如让坏查询被快速拒绝并可见。

# 6. 集群安全
```text
□ 开启 X-Pack Security（认证 + 授权）
□ 启用 TLS（传输 + HTTP 两层）
□ 非 root 用户运行 ES 进程
□ IPTables / Security Group 限制访问 9200/9300
□ 定期审计日志
□ 固定版本（不用 :latest 镜像）
□ 备份 snapshot 每天自动
□ 监控磁盘水印（disk watermark）
```

补充两条 8.x 特有的：**默认配置即安全**（security + TLS 开箱开启，别再复制 6.x 时代的裸奔教程）；**9300 端口绝不出内网**——历史上大量 ES 挖矿事件都是 9200/9300 暴露公网导致。

# 7. 监控告警

```text
必须监控的指标:
  □ 集群状态（必须 green 或 yellow）
  □ 未分配分片数（> 0 告警）
  □ 磁盘使用率（> 85% 告警）
  □ JVM 堆使用率（> 75% 告警）
  □ 写入拒绝率（> 0 告警）
  □ 搜索延迟 P99（> 1s 告警）
  □ 节点离线告警
```

在原有清单上再加三个高价值指标：**分片总数/单节点分片数**（漂移说明 rollover 或 ILM 失效）、**search/write 线程池 queue 持续非零**（容量不足的先行指标）、**最近一次成功快照时间**（>26h 告警，防灾最后一道保险）。阈值原则：对 red、拒绝、水位线用**即时告警**，对延迟、堆使用用**趋势告警**，避免全员对抖动脱敏。

# 8. DON'T

```text
✗ 不要混用不同版本的 ES 节点
✗ 不要让堆内存超过 32GB（丢失 compressed oops）
✗ 不要在单节点部署多个角色
✗ 不要在生产环境用 dynamic mapping
✗ 不要设置过大的 from+size（> 10000）
✗ 不要在高负载时 forcemerge
✗ 不要在磁盘满时继续写入（flood_stage 会只读）
✗ 不要只用 1 个 Master 节点（脑裂风险）
✗ 不要在已运行的索引上修改主分片数
```

其中两条常被低估："单节点多角色"在 K8s/单机 all-in-one 里很普遍，代价是该节点一挂整簇无 master 且数据无处恢复；"1 个 master"在早期 discovery 配置下会脑裂，8.x 用 voting config 后不会分裂但会**停摆**——风险从"数据写花"变成"集群不可用"，3 台独立故障域的 master 仍是标配。

# 9. 生产配置模板

```yaml
cluster.name: prod-cluster
node.name: data-node-1
node.roles: [data, ingest]

path.data: /data/elasticsearch
path.logs: /var/log/elasticsearch

network.host: 0.0.0.0
http.port: 9200
transport.port: 9300

discovery.seed_hosts: ["master1:9300", "master2:9300", "master3:9300"]
cluster.initial_master_nodes: ["master1", "master2", "master3"]

xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.http.ssl.enabled: true

indices.memory.index_buffer_size: 20%
bootstrap.memory_lock: true
```

> **注意**：`cluster.initial_master_nodes` **只属于首次引导**——它仅在集群第一次创建时生效，由它引导出的投票名单固化进 cluster state；此后保留该配置无副作用，但拿"旧配置文件 + 全新 cluster.name"在新环境启动，会按名单拼出一个与现网互不相干的"影子集群"，业务流量被误导入空集群，是重建/搬迁时的经典事故。8.x 支持不带该配置的自动引导（单节点自动成簇、多节点靠 seed_hosts 发现），模板里保留写法是为兼容多节点冷启动。`discovery.seed_hosts` 用静态 IP 或稳定 DNS，别用滚动会变的 Pod IP。

# 10. 小结

| 领域 | 一句话纪律 | 关键数字 |
|------|-----------|----------|
| 分片规划 | 按总量定分片，宁少勿多 | 搜索 30-50GB/分片；每 GB 堆 ≤ 20 分片；每节点 ≤ 600 |
| 内存 | 堆 ≤ 31GB，其余给 OS Cache | 堆 = 物理内存 50% |
| 索引结构 | 时序数据滚动 + 别名 + ILM | rollover max_primary_shard_size=50GB |
| Mapping | dynamic strict；keyword/text 各司其职 | ignore_above 截断；nested 慎用 |
| 写入 | bulk 5-15MB、乱序可接受时提高并发 | refresh_interval 写入期可放大 |
| 查询 | 深分页禁止、filter 优先、桶数受控 | max_result_window=10000；search_after 替代 |
| 安全 | 默认全开，9300 不出内网 | 3 master + API Key 分应用 |
| 监控 | red/拒绝/水位/快照四必告 | 快照超 26h 即告警 |
| 备份 | SLM 每日快照，副本防灾不重叠 | expire 30d + min_count 5 |

**小结**：容量规划决定"会不会撑爆"，索引与 mapping 纪律决定"会不会返工"，查询与运维纪律决定"半夜会不会被叫起来"。三件事各守住一条线，ES 集群就能长期维持在"无感"状态。下一篇（19）讲已经出事之后怎么排。
