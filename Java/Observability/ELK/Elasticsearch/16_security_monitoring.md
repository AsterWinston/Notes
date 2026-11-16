从 8.x 开始，**安全与加密默认开启**：TLS、认证与 RBAC 在免费的 Basic license 即可用，过去"开源 ES = 裸奔内网"的前提不再成立；但要认清分层——DLS/FLS 与审计日志属于付费订阅（Gold 起），字段脱敏更是企业级能力。与此同时，ES 的可观测性也需要自建——`_stats`、`_cat`、Task Management、slow log 构成日常监控的四板斧，Watcher/Alerting 负责把指标变成告警。本篇前半讲"怎么锁"（X-Pack 安全），后半讲"怎么盯"（监控与熔断保护）。

# 1. RBAC 安全体系

## 1.1 启用安全与加密

```yaml
# 启用安全（8.x 默认）
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.http.ssl.enabled: true
```

```bash
# 生成证书（自动）
elasticsearch-certutil ca
elasticsearch-certutil cert --ca elastic-stack-ca.p12

# 设置密码（自动生成）
elasticsearch-setup-passwords auto
```

> **注意**：`elasticsearch-setup-passwords` 只适用于**手工生成证书的传统部署**；8.x 容器/自动化配置场景下内置用户密码由 enrollment 流程生成，此时该命令会直接报错，重置密码改用 `bin/elasticsearch-reset-password -u elastic`。密码忘记且无任何可用管理员账号时，只能引导新的原生 realm 或重建安全配置。生产上更推荐把内置超级用户收口保管、应用侧改用 API Key/服务账号，避免共享 `elastic` 口令。

## 1.2 内置用户

| 用户 | 角色 |
|------|------|
| `elastic` | 超级管理员 |
| `kibana_system` | Kibana 连接到 ES |
| `logstash_system` | Logstash 连接到 ES |
| `beats_system` | Filebeat 连接到 ES |

内置系统用户只服务于组件内部上报（monitored、manage_index_patterns 之类最小权限），**不要拿 `elastic` 跑业务写入**——它绕过一切角色设计，既无法审计归因，泄露后果也是全集群级的。

## 1.3 自定义角色

```json
POST /_security/role/order_read_role
{
  "cluster": ["monitor"],
  "indices": [
    {
      "names": ["orders"],
      "privileges": ["read", "view_index_metadata"]
    }
  ]
}

POST /_security/role/order_write_role
{
  "indices": [
    {
      "names": ["orders"],
      "privileges": ["create", "index", "delete", "write"]
    }
  ]
}

# 创建用户并分配角色
POST /_security/user/order_user
{
  "password": "secret123",
  "roles": ["order_read_role", "order_write_role"]
}
```

**设计考量**：

- **按应用而非按人授权**：一个服务一对"读写角色 + 专属账号"，审计日志里才能看清"谁改的数据"；
- **权限最小化的抓手是 application privilege 与 index 模式**：用 `orders-*` 通配限定范围，比授权 `*` 再靠纪律约束可靠得多；
- **集群级权限单独收口**：`monitor` 只读指标，`manage` 能改设置删索引，`all` 基本等于超级用户——运维平台账号给 monitor，业务账号一律不给 cluster 权限；
- 角色可组合可继承（`mappings` 字段可把 LDAP/AD 组直接映射进角色），企业统一认证走 `xpack.security.authc.realms.ldap` 或 SAML；realm 顺序即认证回退链。

## 1.4 API Key 与服务凭证

对短期服务调用，API Key 比自建用户更轻：

```bash
# 以 elastic 申请一个限定 orders 读权限的 API Key
POST /_security/api_key
{
  "name": "order-readonly-key",
  "expiration": "30d",
  "role_descriptors": {
    "reader": {
      "indices": [{ "names": ["orders"], "privileges": ["read"] }]
    }
  }
}
# 返回 id:hash,Base64 后放 Authorization: ApiKey <token>
```

## 1.5 审计日志

```yaml
xpack.security.audit.enabled: true
xpack.security.audit.logfile.events.include: ["access_granted", "access_denied", "connection_denied", "authentication_failed"]
xpack.security.audit.logfile.events.ignore_principals: ["kibana_system", "probe_user"]
```

审计事件写到节点日志文件（也可转发 Kafka），典型用途：追责越权访问、满足等保/GDPR 留痕。**注意**审计量与集群 QPS 成正比，高频 `authentication_success` 除非合规强制否则不要开，用 ignore 名单过滤健康检查与监控账号，否则磁盘会被审计日志反向打满。

# 2. 字段级/文档级安全

## 2.1 FLS — 字段级安全

```json
POST /_security/role/masked_role
{
  "indices": [
    {
      "names": ["orders"],
      "privileges": ["read"],
      "field_security": {
        "grant": ["order_no", "amount", "status"],
        "except": ["customer_phone", "customer_email"]
      }
    }
  ]
}
```

FLS 在**取回阶段**裁剪 `_source` 字段：命中计算仍用全量数据，但响应里看不到被 except 的字段，聚合结果也不会暴露。8.x 还支持**脱敏脚本**（需企业级订阅）：`"masked_fields": { "customer_phone": "expression 'phone-[[masked]]'" }` 可返回部分掩码值而不是直接消失。

> **注意**：FLS 保护的是"查询响应"，不是磁盘与内存——原文仍在倒排索引与 doc_values 里，拿到节点 root 权限的人依然可读；对手机号、身份证这类强合规字段，更稳妥的做法是**写入前在应用层加密/哈希**，ES 里只存密文与可搜索的盲索引（blind index）。

## 2.2 DLS — 文档级安全

```json
POST /_security/role/self_orders
{
  "indices": [
    {
      "names": ["orders"],
      "privileges": ["read"],
      "query": {
        "term": { "user_id": "_user_id" }
      }
    }
  ]
}
```

DLS 把一段 query **自动 AND 进该角色的每一次搜索**，实现行级隔离：`_user_id`、`_username`、`_groups` 等占位符在运行时替换为当前认证主体。典型多租户方案是 `{"term": {"tenant_id": "{{{x_pack_metadata.tenant}}}"}}` 配合 JWT realm 注入 metadata。

**设计考量**：DLS + FLS 组合可以实现"同索引、不同角色看到不同行列"的 SaaS 权限模型，但有代价——DLS 的过滤 query 每次搜索都参与打分前的过滤，复杂 DLS（子查询、脚本）会拖慢缓存命中；能用索引物理隔离的小租户，优先按 `tenant-a-orders` 分索引，把 DLS 留给确实需要混存的大规模场景。

# 3. TLS 配置

```yaml
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: certs/http.p12
xpack.security.http.ssl.truststore.path: certs/http.p12

xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: certs/transport.p12
xpack.security.transport.ssl.truststore.path: certs/transport.p12
```

两条链路分开看：**HTTP（9200）** 面向客户端与 Kibana，证书需要合法的 SAN（把负载均衡/域名写进去，用 `elasticsearch-certutil http` 生成实例证书最容易漏）；**Transport（9300）** 是节点间通信，必须开——它承载分片数据与集群状态，未加密时任何能路由到 9300 的机器都可以伪装成节点加入集群。

> **注意**：`verification_mode` 三档的取舍——`full`（默认）校验 CA + 主机名，生产两端都该用它；`certificate` 只验 CA 不验主机名，常见于内网 IP 频繁变动的 K8s 环境，安全性略降但仍加密；`none` 等于只加密不认证，**等于没开**，仅限开发调试。证书轮换走 `POST /_nodes/reload_security_settings`，不用滚动重启整集群。

# 4. 监控

## 4.1 Cat API

```bash
GET /_cat/health?v
GET /_cat/nodes?v&h=name,heap.percent,ram.percent,cpu,load_1m
GET /_cat/indices?v&h=index,pri,rep,store.size
GET /_cat/shards?v&h=index,shard,prirep,state,unassigned.reason
GET /_cat/thread_pool/write?v
GET /_cat/thread_pool/search?v
GET /_cat/allocation?v
```

Cat 系列适合人眼看、脚本切（`format=json`/`h=` 指定列）；程序化采集用 `/_nodes/stats`、`/_stats` 全量 JSON 更稳。

## 4.2 _stats：索引级与集群级

```bash
# 索引级:分段、缓存、搜索/写入耗时统计
GET /orders/_stats?pretty

# 集群级:一次拿全部节点资源指标
GET /_nodes/stats/jvm,os,fs,thread_pool,indices?pretty
```

值得建立基线的关键指标：`segments.count`（每分片上万段会拖慢搜索并抬高堆占用）、`query_cache.memory_size_in_bytes` 与 hit/miss 比、`fielddata.memory_size_in_bytes`（非零就该查谁在用 doc_values 之外的路径）、`merging.current`、write/search 线程池的 `queue` 与 `rejected`、`fs.total.available_bytes`（对着水位线算剩余额度）。

## 4.3 Task Management

```bash
# 查看活跃任务
GET /_cat/tasks?v

# 取消慢查询
POST /_tasks/task_id:xxx/_cancel

# 查看热点线程
GET /_nodes/hot_threads?interval=1s
```

Task API 覆盖搜索、写入、reindex、分片恢复，`parent_task_id` 可以看层级；`hot_threads` 默认抓 CPU 最高的线程栈，是 CPU 飙高时**不需要重启**就能定位现场的手段。

## 4.4 Circuit Breaker：堆内保险丝

ES 给多个易失控的堆内结构设了熔断器，超阈值直接抛 `CircuitBreakingException` 而不是放任 OOM：

| 熔断器 | 默认阈值 | 保护对象 | 触发后现象 |
|--------|----------|----------|------------|
| `fielddata` | 堆 40% | text 字段排序/聚合加载的 term 索引 | 查询失败，提示 "Data too large" |
| `request`（单请求聚合/排序） | 堆 60% | 单个大聚合的中间桶 | 报 "Total memory used by all existing requests exceeded" |
| `parent`（总合并限额） | 堆 95% | 以上全部 | 任何请求都可能被拒 |
| `accounting`（段级记账） | 100% | 已驻留堆的 fielddata 记账 | 一般只是日志告警 |
| `in_flight_requests` | 堆 100% | 正在处理的传输层请求 | 拒绝新请求 |
| `analysis` | 堆 5% | 表达式/analyze 缓存 | 编译正则类分析失败 |

```json
PUT /_cluster/settings
{
  "persistent": {
    "indices.breaker.request.limit": "55%",
    "indices.breaker.total.limit": "90%"
  }
}
```

> **注意**：调大熔断阈值 = 把熔断换成 OOM，几乎从不解决问题；正确路径永远是**让查询少占堆**：聚合改 `composite` 分页、terms 聚合控制 size、用 `execution_hint: map` 或 global ordinals 评估、去掉 text 字段排序改用 keyword。

# 5. Watcher / Alerting 简介

X-Pack 告警有两条产品线：**Watcher**（老式，JSON DSL，规则自持）与**Alerting + SLO**（新式，规则对象 + Connectors 分发，与 Kibana Stack Management 一体化）。

```json
PUT /_watcher/watch/orders_error_watch
{
  "trigger": { "schedule": { "interval": "5m" } },
  "input": {
    "search": {
      "request": {
        "indices": ["app-logs-*"],
        "body": {
          "query": { "bool": { "filter": [
            { "term": { "level": "ERROR" } },
            { "range": { "@timestamp": { "gte": "now-5m" } } }
          ] } }
        }
      }
    }
  },
  "condition": { "compare": { "ctx.payload.hits.total": { "gt": 100 } } },
  "actions": {
    "notify_webhook": {
      "webhook": { "method": "post", "url": "http://alert-h:8060/es", "body": "{{ctx.payload.hits.total}} errors in 5m" }
    }
  }
}
```

开源替代路径：`elastalert2` 读 ES 做规则匹配，或把 `_prometheus/metrics`/exporter 接进 Prometheus Alertmanager。**设计考量**：告警规则放版本库管理（watch JSON 与规则导出的 NDJSON 可 `POST /_watcher/watch/_bulk` 一键重建），避免"集群重建后告警全丢"；对 red 集群、写入拒绝、fielddata 熔断这三类事件必须设 P0，其余指标只做趋势展示。

# 6. Slow Log

```json
PUT /orders/_settings
{
  "index.search.slowlog.threshold.query.warn": "2s",
  "index.search.slowlog.threshold.query.info": "1s",
  "index.search.slowlog.threshold.fetch.warn": "1s",
  "index.indexing.slowlog.threshold.index.warn": "5s"
}
```

慢日志分 **query phase**（打分耗时）与 **fetch phase**（取 `_source`/stored fields 耗时），两段阈值要分别设——很多"慢"其实慢在大文档 fetch 而不是查询本身。日志落节点文件（`logs/index_index_search_slowlog.json`），8.x 默认带 tracing 开关（`index.search.slowlog.trace_include_source_match:true` 可看命中的具体 query），配合 19 篇的定位流程食用。

# 7. Prometheus Exporter

```bash
# 安装插件
elasticsearch-plugin install \
  https://github.com/vvanholl/elasticsearch-prometheus-exporter/releases/download/8.11.0/prometheus-exporter-8.11.0.zip

# 访问指标
curl http://localhost:9200/_prometheus/metrics
```

第三方插件路线，需与 ES 版本严格对齐、逐节点安装重启。没有订阅又要接 Prometheus 体系的团队，更主流的选择是**独立部署 exporter 进程**：prometheus-community 的 `elasticsearch_exporter`，或 Grafana 自家的 **Grafana Agent integrations.elastic**——不侵入 ES 进程，升级 ES 不需要重编插件，采集 `cluster_health`、`node_stats`、`indices_stats` 三组核心指标即可支撑大盘。

# 8. 小结

```text
安全四层: TLS(传输/HTTP) → 认证(realm/API Key) → 授权(集群/索引/角色) → 数据可见性(DLS/FLS)
监控四板斧: _cat/health + _stats(资源与缓存基线) + tasks/hot_threads(现场) + slowlog(查询质量)
告警三必设: 集群 red / 写入 rejected>0 / fielddata 或 request 熔断触发
```

安全决定"谁能看什么"，监控与熔断决定"集群会不会在你睡觉时翻车"。下一篇（17）把视角拉高到整个 Elastic Stack 的数据流与生态集成。
