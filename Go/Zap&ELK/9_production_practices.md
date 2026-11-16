前八篇分别讲了原则、Zap、字段、采样、轮转、ELK 架构、采集和 Kibana。但真实的生产系统不是八个独立知识点，而是一条端到端的链路：日志怎么产生、怎么搬运、怎么存、怎么查、怎么控成本、怎么合规、怎么告警、出故障怎么定位。本篇把这些串起来，并给出可直接落地的检查清单与模块总结。

# 1. 日志链路设计

## 1.1 端到端架构

```text
┌─────────────────────────────────────────────────────────────────────┐
│ 应用层                                                               │
│  Zap（结构化 JSON）→ 本地文件 / stdout                                │
│  字段：service、env、version、request_id、trace_id、event、level      │
└───────────────────────────────┬─────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 采集层                                                               │
│  Filebeat（filestream）                                              │
│  - 多行合并、加 host/container 元数据                                 │
│  - registry 记录 offset，断点续传                                     │
└───────────────────────────────┬─────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 缓冲层（可选）                                                        │
│  Kafka：削峰、解耦、可回放                                            │
└───────────────────────────────┬─────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 处理层                                                               │
│  Logstash / ES Ingest Pipeline                                       │
│  - Grok/dissect 解析、date 覆盖时间、脱敏、路由                       │
└───────────────────────────────┬─────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 存储层                                                               │
│  Elasticsearch                                                       │
│  - data stream + ILM：hot → warm → cold → delete                     │
└───────────────────────────────┬─────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 消费层                                                               │
│  Kibana（Discover/Lens/Alerting） + APM + 外部告警平台                │
└─────────────────────────────────────────────────────────────────────┘
```

## 1.2 每一跳的职责与容量

| 环节 | 职责 | 关键容量指标 | 满了会怎样 |
|------|------|--------------|------------|
| 应用 | 产生结构化日志 | 磁盘、写入 QPS | 写失败，业务可能受影响 |
| Filebeat | 读取、合并、发送 | 内存队列、发送重试 | 读取变慢，文件堆积 |
| Kafka | 缓冲、解耦 | 分区数、磁盘、保留时间 | 生产被限流 |
| Logstash | 解析、丰富、路由 | PQ 大小、worker 数、CPU | 拒绝新事件 |
| ES | 索引、存储、查询 | 磁盘、堆内存、分片数 | 写入 429、查询超时 |
| Kibana | 查询、展示 | 并发查询数 | 页面卡顿 |

★ **关键认知**：**每一跳都必须有有界队列和明确的溢出策略**。"内存无限缓冲"是最危险的设计，因为它把"慢慢堆积"变成了"某一刻 OOM 丢掉全部"。宁可让 Filebeat 读慢一点，也不要让 Logstash 无限制吃内存。

## 1.3 一个可落地的参考架构

| 规模 | 日日志量 | 推荐架构 |
|------|----------|----------|
| 小 | < 10 GB | Filebeat → ES → Kibana |
| 中 | 10-200 GB | Filebeat → Logstash → ES → Kibana |
| 大 | > 200 GB | Filebeat → Kafka → Logstash → ES → Kibana |
| 超大 | > 1 TB | 多 Kafka 集群 + 多 Logstash + ES 冷热分层 + 对象存储 |

⚠ **关键认知**：**不要一步到位上最复杂的架构**。组件越多，故障面越大。先跑通 Filebeat 直连 ES，遇到解析需求再加 Logstash，遇到削峰需求再加 Kafka。每次加组件都要能回答"它解决了什么具体问题"。

# 2. 成本控制

## 2.1 成本构成

```text
总成本 = 计算（应用日志格式化 + Filebeat + Logstash + ES）
       + 存储（ES 主副本 + 快照 + 对象存储）
       + 网络（跨可用区传输 + 出口流量）
       + 人力（运维、调优、排障）
```

经验上，**存储通常占 50-70%，计算占 20-30%，网络和人力占其余**。所以降本的第一优先级是"少存"，而不是"换更便宜的机器"。

## 2.2 降本手段

| 手段 | 节省 | 代价 | 优先级 |
|------|------|------|--------|
| Debug 采样/关闭 | 30-60% | 少看细节 | 高 |
| 删除无用字段 | 10-30% | 需改代码 | 高 |
| 缩短热数据保留 | 20-50% | 历史查得慢 | 高 |
| refresh 调大 | 10-20% CPU | 可见延迟增加 | 中 |
| 压缩（best_compression） | 20-30% 存储 | 写入 CPU 增加 | 中 |
| 冷热分层 | 30-50% 存储 | 架构复杂 | 中 |
| 采样（Info） | 20-40% | 可能漏低频问题 | 中 |
| 换对象存储归档 | 60-80% 长期 | 查询需恢复 | 低 |

```json
// 对只读的历史索引启用更高压缩
PUT logs-order-2026.10.01/_settings
{
  "index": {
    "codec": "best_compression",
    "refresh_interval": "-1"
  }
}
```

## 2.3 分层保留

```text
Hot    （7 天）  ：SSD，1 副本，可查可写，refresh 30s
Warm   （30 天） ：HDD，1 副本，只读，force merge 成少量 segment
Cold   （90 天） ：大容量 HDD / 对象存储，按需恢复
Delete （180 天）：删除或归档到合规存储
```

```json
PUT _ilm/policy/logs-ilm
{
  "policy": {
    "phases": {
      "hot":    { "actions": { "rollover": { "max_primary_shard_size": "30gb", "max_age": "1d" } } },
      "warm":   { "min_age": "7d",  "actions": { "forcemerge": { "max_num_segments": 1 }, "set_priority": { "priority": 50 } } },
      "cold":   { "min_age": "30d", "actions": { "searchable_snapshot": { "snapshot_repository": "backup" } } },
      "delete": { "min_age": "180d", "actions": { "delete": {} } }
    }
  }
}
```

★ **关键认知**：**成本优化要按"数据价值"分层，而不是一刀切**。错误日志、审计日志值得留 180 天；`cache miss` 这类 Debug 留 3 天都嫌多。把所有日志放一个保留策略，要么贵得离谱，要么合规过不了。

# 3. 安全与合规

## 3.1 数据分级

| 级别 | 例子 | 处理方式 |
|------|------|----------|
| 公开 | 服务名、版本号 | 正常存储 |
| 内部 | 请求路径、耗时 | 正常存储，限内部访问 |
| 敏感 | 用户 ID、IP、手机号 | 脱敏或哈希后存储 |
| 机密 | 密码、Token、身份证、银行卡 | 禁止落日志 |

★ **关键认知**：**机密数据一旦进了日志，就等于进了多个副本、多台机器、多个备份**。删除一条 ES 文档容易，删除所有快照、Kafka 消息、Logstash 队列里的副本几乎不可能。所以唯一可靠的合规手段是**入口就不写**。

## 3.2 传输与存储加密

```yaml
# Filebeat → Logstash 开启 TLS
output.logstash:
  hosts: ["logstash:5044"]
  ssl:
    enabled: true
    certificate_authorities: ["/etc/pki/ca.crt"]
    certificate: "/etc/pki/filebeat.crt"
    key: "/etc/pki/filebeat.key"
```

```yaml
# Elasticsearch 8.x 默认开启安全
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.http.ssl.enabled: true
```

| 环节 | 加密方式 |
|------|----------|
| 应用到 Filebeat | 本地文件权限 0600 |
| Filebeat 到 Logstash/Kafka | TLS |
| Logstash 到 ES | HTTPS + 账号 |
| ES 静态数据 | 磁盘加密 + snapshot 加密 |
| Kibana 访问 | HTTPS + SSO |

## 3.3 访问控制与审计

| 对象 | 控制手段 |
|------|----------|
| 索引 | Role + `field_security` + `document_level_security` |
| Dashboard | Space + feature privileges |
| 查询 | 禁止普通用户写 raw DSL |
| 调级接口 | 内网 + Token + 审计 |
| 导出 | 限制批量导出，记录操作人 |

```json
// 文档级安全：租户只能看自己的日志
PUT _security/role/tenant_reader
{
  "indices": [
    {
      "names": ["logs-*"],
      "privileges": ["read"],
      "query": "{\"term\": {\"tenant_id\": \"{{_user.name}}\"}}"
    }
  ]
}
```

⚠ **关键认知**：**字段级安全不能替代入口脱敏**。`field_security` 只控制"查询返回什么"，不控制"数据是否已经存在于索引和快照里"。合规审计看的是后者。

# 4. 告警

## 4.1 告警指标

| 类别 | 指标 | 阈值示例 |
|------|------|----------|
| 业务 | 错误日志速率 | > 10 条/分钟 |
| 业务 | 错误率 | > 1% 持续 5 分钟 |
| 业务 | 关键事件缺失 | 10 分钟无 `order_created` |
| 链路 | Filebeat 发送失败 | > 0 持续 5 分钟 |
| 链路 | Logstash PQ 使用率 | > 80% |
| 存储 | ES 磁盘使用率 | > 75% 预警，85% 严重 |
| 存储 | ES 写入拒绝（429） | > 0 |
| 查询 | Kibana 查询 P95 | > 5s |

## 4.2 告警规则示例

Kibana 8.x 内置 Alerting，可以直接对日志做阈值告警：

```text
Rule type: Elasticsearch query
Index: logs-order-*
Query: level: "error" or level: "fatal"
Threshold: count > 10
Time window: 5 minutes
Group by: service.name
Action: 企业微信 / PagerDuty
```

用 ES 的 DSL 写更精确的规则：

```json
POST logs-order-*/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "range": { "@timestamp": { "gte": "now-5m" } } },
        { "terms": { "level": ["error", "fatal"] } },
        { "term": { "service.name": "order-api" } }
      ]
    }
  },
  "aggs": {
    "by_error_type": {
      "terms": { "field": "error_type", "size": 10 }
    }
  }
}
```

## 4.3 告警治理

| 问题 | 后果 | 治理 |
|------|------|------|
| 所有 Error 都告警 | 告警疲劳 | 只对 `error_type` 白名单告警 |
| 阈值太敏感 | 频繁抖动 | 加持续时间、加分组 |
| 无 Runbook | 值班不知道怎么处理 | 每条告警附排查链接 |
| 告警不闭环 | 没人认领 | 接入 on-call，记录处理结果 |
| 只告警不总结 | 同类故障反复 | 每月复盘 Top 3 |

★ **关键认知**：**告警的价值在于"可行动"**。一条告警如果值班人看了也不知道该做什么，它就是噪声。设计告警时先问："收到这条告警的人，第一步该做什么？"答不上来就不要配。

# 5. 故障排查

## 5.1 常见故障矩阵

| 现象 | 可能原因 | 排查入口 |
|------|----------|----------|
| Kibana 完全没数据 | Filebeat 没启动 / 输出配错 | `filebeat test output` |
| 某服务没数据 | 路径不对 / 权限不足 / 多行合并卡住 | Filebeat 日志、`ls -l` |
| 数据延迟很大 | 背压、ES 写入慢、Kafka 积压 | 各跳队列长度、ES 429 |
| 字段搜不到 | mapping 错 / Data View 时间字段错 | `_mapping`、Discover |
| 日志重复 | filestream `id` 改了 / registry 丢失 | registry 文件、offset |
| 时间差 8 小时 | 时区未配置 | `date` filter、浏览器时区 |
| ES 磁盘涨得快 | 无 ILM / 副本过多 / Debug 未关 | `_cat/indices`、ILM 状态 |
| 查询超时 | 分片过多 / 通配符查询 / 冷数据 | 慢查询日志、`_cat/shards` |

## 5.2 排查手册

**第一步：确认数据源**

```bash
# 日志文件在不在、有没有在增长
ls -lh /var/log/order-api/
tail -f /var/log/order-api/app.log
```

**第二步：确认采集端**

```bash
# 测试 Filebeat 到 ES/Logstash 的连通性
filebeat test output

# 看 Filebeat 自己的日志
journalctl -u filebeat -f
# 关注 Harvester started / Failed to publish events
```

**第三步：确认处理端**

```bash
# Logstash 管道状态
curl -s localhost:9600/_node/stats/pipelines?pretty | jq '.pipelines.main.events'

# 重点看 in / filtered / out 是否一致，queue 是否在涨
```

**第四步：确认存储端**

```bash
# 集群健康和索引状态
curl -s 'localhost:9200/_cluster/health?pretty'
curl -s 'localhost:9200/_cat/indices/logs-*?v&s=store.size:desc'

# 看最近的写入拒绝
curl -s 'localhost:9200/_cat/thread_pool/write?v'
```

**第五步：确认查询端**

```text
1. Data View 的 pattern 是否匹配到新索引
2. Time field 是否是 @timestamp
3. 时间范围是否覆盖
4. 字段类型是否 keyword
```

★ **关键认知**：**排查要沿链路单向推进，不要跳跃**。最常见的错误是"Kibana 搜不到"就直接去改查询语句，结果真正的问题是 Filebeat 根本没读到文件。每一段都用该段的工具验证，才能快速定位。

# 6. 上线检查清单

| 类别 | 检查项 | 是否必须 |
|------|--------|----------|
| 日志 | 使用结构化 JSON，禁止 `fmt.Println` | 必须 |
| 日志 | `msg` 固定短语，变量放字段 | 必须 |
| 日志 | 每条日志带 `service`、`env`、`version` | 必须 |
| 日志 | 请求链路带 `request_id` / `trace_id` | 必须 |
| 日志 | Error 带 `error_type`、`error`、业务主键 | 必须 |
| 安全 | 禁止打印密码、Token、完整请求体 | 必须 |
| 安全 | 敏感字段入口脱敏 | 必须 |
| 轮转 | 配置 `MaxSize` + `MaxBackups` + `MaxAge` | 必须 |
| 采集 | Filebeat `id` 稳定，多行规则有测试样例 | 必须 |
| 存储 | 显式索引模板，ID 字段为 `keyword` | 必须 |
| 存储 | ILM 策略覆盖 hot/warm/cold/delete | 推荐 |
| 告警 | 错误率、链路健康、磁盘水位 | 必须 |
| 权限 | Space + Role + 字段脱敏 | 必须 |
| 成本 | Debug 采样/关闭，字段精简 | 推荐 |
| 运维 | 调级接口鉴权 + 审计 | 必须 |
| 运维 | 有 Runbook 和排障入口 | 推荐 |

# 7. 模块总结

## 7.1 九篇回顾

| 篇目 | 核心结论 |
|------|----------|
| 1 日志原则 | 结构化优先、级别表达"谁来看"、脱敏在入口 |
| 2 Zap 核心 | Logger/Core/Encoder/WriteSyncer 四层，生产用 JSON |
| 3 结构化日志 | 三层字段、request_id 全链路、错误分类记录 |
| 4 糖与采样 | 热路径用强类型，Error 不采样，Hook 拿不到字段 |
| 5 轮转归档 | 大小+时间组合，轮转不是备份 |
| 6 ELK 架构 | 分片不可改、近实时、背压保护、容量估算 |
| 7 Filebeat/Logstash | filestream 续传、多行规则、Grok vs dissect |
| 8 Kibana | Data View、KQL、text/keyword、权限 |
| 9 生产实践 | 链路有界、按价值分层、告警可行动、排查单向推进 |

## 7.2 能力地图

```text
写日志（Zap）
  ├─ 结构化字段 ──► 能被机器解析
  ├─ 级别与采样 ──► 控制量与噪声
  ├─ 上下文 ──────► 能串起一次请求
  └─ 脱敏 ────────► 合规底线
        │
        ▼
搬日志（Filebeat / Logstash）
  ├─ 可靠读取 ────► registry + 轮转保留
  ├─ 解析 ────────► dissect 优先，Grok 兜底
  └─ 背压 ────────► 有界队列，不丢不炸
        │
        ▼
存日志（Elasticsearch）
  ├─ 模板 ────────► 显式 mapping
  ├─ 分片与 ILM ──► 容量与成本
  └─ 副本 ────────► 可用性
        │
        ▼
用日志（Kibana）
  ├─ 查询 ────────► KQL + keyword
  ├─ 可视化 ──────► 分层 Dashboard
  └─ 告警 ────────► 可行动、可闭环
```

★ **关键认知**：**日志系统的成熟度不取决于用了多少组件，而取决于"一条日志能否在 5 分钟内回答一个具体问题"**。如果链路很豪华，但排障时还要靠人肉 grep，那是本末倒置。

# 8. 练习与自检

1. 为你的项目画一条完整的日志链路，标出每一跳的队列类型、容量上限和溢出策略。指出哪一跳最可能成为单点。
2. 日日志成本突然翻倍，请给出一个排查顺序（从应用、采集、存储到查询），并说明每一步看什么指标。
3. 设计一套告警：要求覆盖"业务错误率""采集链路中断""ES 磁盘水位"三类，每条告警写出查询、阈值、持续时间和第一步处理动作。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 链路 | 应用 → 采集 → 缓冲 → 处理 → 存储 → 消费 |
| 有界 | 每跳都要有队列上限和溢出策略 |
| 成本 | 先少存，再分层，最后才加机器 |
| 安全 | 机密不落日志，入口脱敏，字段级权限 |
| 告警 | 可行动、有 Runbook、能闭环 |
| 排查 | 沿链路单向推进，用每段的工具验证 |
| 上线 | 结构、上下文、轮转、模板、权限、告警缺一不可 |
| 成熟度 | 一条日志 5 分钟内回答一个具体问题 |

至此本模块结束。下一步可以进入 OpenTelemetry，把日志、指标、链路真正用 `trace_id` 串成一套完整的可观测性体系。
