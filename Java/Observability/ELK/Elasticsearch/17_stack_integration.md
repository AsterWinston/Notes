Elasticsearch 很少孤立部署：日志采集（Beats/Logstash）、可视化（Kibana）、链路追踪（APM）与业务代码（Java Client / Spring Data）围绕它组成一个完整观测体系。本篇先画出全家桶的数据流，再对比"传统 Beats 分散采集"与"新 Fleet/Elastic Agent 统一采集"两代架构，最后落到 Java 应用侧：如何用 8.x Java Client 与 Spring Data Elasticsearch 把 ES 接进业务。

# 1. ELK 架构

```text
┌──────────────────────────────────────────────────────┐
│                     ELK Stack                         │
│                                                       │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐        │
│  │ Filebeat │   │ Logstash │   │   APM    │        │
│  │ (采集)    │   │ (处理)    │   │  Agent   │        │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘        │
│       │              │              │                │
│       └──────────────┼──────────────┘                │
│                      ▼                                │
│              ┌──────────────┐                         │
│              │ Elasticsearch│ ← 存储 + 搜索 + 分析     │
│              └──────┬───────┘                         │
│                     │                                 │
│              ┌──────▼───────┐                         │
│              │    Kibana    │ ← 可视化                │
│              └──────────────┘                         │
└──────────────────────────────────────────────────────┘
```

各组件的职责边界值得先说清，因为**部署形态直接决定运维成本**：Beats 是 Go 写的轻量采集器，只负责"读源、简单加工、发送"，常驻内存几十 MB；Logstash 是 JVM 管道，承担 grok 解析、字段改写、路由分流等**重加工**，同时天然是个缓冲区，能吸收流量尖峰；Elasticsearch 只管存储与查询，**不要在它前面裸接原始日志**——解析失败的文件映射（dynamic mapping）会把索引 schema 搞乱；Kibana 是唯一的"读端"，Dashboards、Discover、Dev Tools 都从这里出。

> **行业实践**：小规模（单机日志、量小）用 Filebeat → ES 直连，跳过 Logstash 减少一层运维；量大且格式脏（多语言混排、需要清洗）才上 Logstash，并且至少部署 2 个实例 + 队列（RabbitMQ/Kafka）做削峰。ES 前端再加一层协调节点，把"解析与 fan-out"和"数据节点"的故障域隔开。

# 2. Filebeat → ES

```yaml
# filebeat.yml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/app/*.log
    fields:
      app: order-service
      env: production

output.elasticsearch:
  hosts: ["es1:9200", "es2:9200", "es3:9200"]
  username: "beats_system"
  password: "${BEATS_PASSWORD}"
  index: "app-logs-%{+yyyy.MM.dd}"
```

**设计考量**：

- **registry 文件决定"读到哪了"**：Filebeat 按 inode 记录 offset，容器日志轮转（inode 复用）场景必须配 `close.on_state_change.inactive` 与 `clean_inactive`，否则会漏读或重读；
- **直连 ES 就没有 grok 能力**：`processors` 只支持 drop/decode/add_fields 等轻量操作，复杂解析请走 `output.logstash`；
- **凭证别写死**：`${BEATS_PASSWORD}` 环境变量或 keystore（`elasticsearch-keystore add filebeat.password`）都行，`beats_system` 这类系统账号权限不匹配，生产建议单独建只允许写 `app-logs-*` 的角色；
- 8.x 起 Filebeat 自带 ECS（Elastic Common Schema）字段命名，自建 index pattern 前先确认 `@timestamp` 时区与 ECS 版本兼容性。

配套的索引模板让滚动 + ILM 一步到位：

```json
PUT /_index_template/app-logs
{
  "index_patterns": ["app-logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 3,
      "index.lifecycle.name": "logs_policy",
      "index.lifecycle.rollover_alias": "app-logs"
    }
  }
}
```

# 3. Logstash 数据处理

```ruby
# logstash.conf
input {
  beats { port => 5044 }
}

filter {
  grok {
    match => { "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} %{GREEDYDATA:message}" }
  }
  date { match => ["timestamp", "ISO8601"] }
  mutate {
    remove_field => ["@version", "host"]
    add_field => { "processed_by" => "logstash" }
  }
}

output {
  elasticsearch {
    hosts => ["es:9200"]
    index => "processed-logs-%{+yyyy.MM.dd}"
  }
}
```

Logstash 性能调优的三个旋钮：`pipeline.workers`（filter 线程数，默认 = CPU 核数）、`pipeline.batch.size`（每批事件数，调大提吞吐但增延迟）、`pipeline.ecs_compatibility`。grok 是多行合并与字段解析的重头戏，`%{GREEDYDATA}` 这类贪婪模式放在 pattern 末尾，顺序错会导致整行匹配失败落入 `_grokparsefailure` 标签——**给失败事件加独立输出路径**是排错第一步：

```ruby
if "_grokparsefailure" in [tags] {
  elasticsearch { index => "dlq-logs-%{+yyyy.MM.dd}" hosts => ["es:9200"] }
}
```

> **注意**：Logstash 的 ES 输出默认走 bulk，`index` 里的 `%{+yyyy.MM.dd}` 按 **Logstash 进程本地时区**展开，跨时区团队常出现"索引日期比日志时间早/晚一天"；稳妥做法是用 date filter 解析出事件时间后写入 `[@metadata][index_date]`，输出 index 改用该字段拼接，与进程时区彻底解耦。

# 4. Kibana

## 4.1 Discover — 日志检索

```text
Kibana → Discover
  - 选择 Index Pattern: app-logs-*
  - KQL 搜索: level:ERROR AND app:order-service
  - 时间过滤
  - 字段过滤
```

KQL（Kibana Query Language）是 Lucene query string 的安全替代：不支持通配裸奔、不支持脚本，语法只有 `field:value`、`and/or/not`、范围与 `"短语"`，对非技术同学友好且不会因为一个 `*` 打挂集群。Discover 保存的 filter + 列布局可作为团队共享的"日志视图"。

## 4.2 Dashboard — 可视化

```text
常用面板:
  - 日志量趋势（折线图）
  - 错误率（饼图）
  - Top 10 慢接口（数据表）
  - P99 响应时间（折线图）
  - 按服务分布（柱状图）
```

面板背后全是 ES 聚合：趋势线是 `date_histogram`，Top 10 是 `terms`，P99 是 `percentiles`。理解这一层映射，才能解释"为什么 Dashboard 在大时间范围下会卡"——本质是聚合桶数超过了 `search.max_buckets`（默认 65535）或 terms 精度丢失，治理手段回落到 18 篇的查询纪律。

## 4.3 Canvas — 大屏展示

创建可视化大屏展示实时业务指标。

## 4.4 Lens — 拖拽分析

拖拽字段即可生成图表，无需编写聚合 JSON。

**Kibana Space** 是多团队隔离的正解：每个 Space 有独立的 Dashboard/索引模式，配 RBAC 角色（`kibana_dashboard_only_user` 给只读观众），避免"一个所有人共享的大空间"被误删。告警（Rules & Alerting）、Connectors（webhook/Slack/邮件）也挂在 Kibana 上，与 16 篇的 Watcher 二选一即可，新集群一律用 Kibana Alerting。

# 5. Fleet / Elastic Agent — 新采集架构

传统模式的痛点：每种数据源一个 Beats（Filebeat、Metricbeat、Auditbeat……），各自一套 YAML，升级要挨个滚，字段规范靠自觉。Fleet 把这一切收敛成**一个 Elastic Agent + 服务端集中管理**：

```text
传统:  Filebeat ─┐
       Metricbeat ─┼─→ (手写多份配置,各自升级)
       Auditbeat ─┘

新架构: Elastic Agent ──(以 Integration 为单位装载 filebeat/metricbeat 模块)
              │  策略下发: Fleet Server → Kibana UI 统一管理
              ▼
          ES / ES|QL
```

| 维度 | 传统 Beats | Fleet + Elastic Agent |
|------|-----------|----------------------|
| 部署单元 | 每种数据源一个 beat | 每台主机一个 agent |
| 配置方式 | 本地 YAML，自行分发（Ansible/Puppet） | Kibana 里定义 Policy，服务端下推 |
| 集成生态 | 无统一打包 | Integration 包（Systemv2）：采集 + 解析 + 模板 + Dashboard 一键装 |
| 升级 | 逐个滚 | Fleet 批量滚动升级，支持维护窗口 |
| 传输链路 | beat → LS → ES 或 beat → ES | agent → Fleet Server → ES（或 dist 直连） |
| 适合 | 单一场景、老集群存量 | 新集群、多数据源、主机数大 |

> **行业实践**：存量 Beats 部署不会被强推迁移，两者长期共存；但新数据源一律优先走 Elastic Agent + Integration，因为**解析规则与 Dashboard 由官方随 Integration 升级维护**，自维护 grok 的成本会在半年内反超。Fleet Server 自身也要高可用（≥3 实例 + ES 存储策略），它挂了不影响已运行的 agent 采集，但策略无法下发。

APM 同样能由 Elastic Agent 以 `apm` 集成承载，替代独立 apm-server 进程。

# 6. APM 集成

APM（Elastic APM）采的是**分布式链路**：Java Agent 字节码增强，自动跟踪 HTTP/RPC/DB 调用，数据经 APM Server（或 agent）写入 `traces-apm.*` 索引，Kibana APM 页展示火焰图与服务拓扑。

```bash
# APM Server
docker run -d --name apm-server \
  -p 8200:8200 \
  docker.elastic.co/apm/apm-server:8.11.0

# Java Agent 启动参数
java -javaagent:elastic-apm-agent.jar \
  -Delastic.apm.service_name=order-service \
  -Delastic.apm.server_url=http://apm:8200 \
  -jar app.jar
```

**设计考量**：

- 日志-链路打通靠 **trace_id 关联**：Java Agent 会把 active trace id 写进 MDC，logback pattern 加 `%X{trace_id} %X{span_id}`，Filebeat 用 `decode_json_fields` + `processor.enrich` 或 Kibana 的 rule & connector 跳转，即可从一条 ERROR 日志一键跳到对应 waterfall——这是 ELK + APM 组合的核心价值；
- APM 采样：`transaction_sample_rate=0.1` 对高 QPS 服务是常态，全采会写爆 traces 索引；
- APM 数据同样吃 ILM，traces/metrics 索引通常保留 7-14 天即可。

# 7. Java 应用侧集成

## 7.1 Elasticsearch Java Client 8.x

8.x 官方统一为 `elasticsearch-java`（与 server 版本对齐，坐标 `co.elastic.clients:elasticsearch-java`），底层基于 HttpClient 5 + Jackson：

```java
// 客户端构建：认证 + TLS，线程安全，全局单例复用连接池
var client = ElasticsearchClient.of(b -> b
        .host("https://es1:9200")
        .withBasicAuth("app_writer", "secret")   // 服务间调用推荐改用 apiKey("id:hash")
        .setHttpClientConfigCallback(hc -> hc
                .setTlsStrategy(caFingerprintStrategy)  // 加载企业 CA
                .setConnectionPoolConfig(pool -> pool.setMaxConnTotal(100)))
        .build());

// 写入一条文档
IndexResponse resp = client.index(i -> i
    .index("orders")
    .id(String.valueOf(order.getId()))
    .document(order));

// 组合查询：status 是 keyword 用 term 精确过滤，正文用 match 全文检索
SearchResponse<Order> search = client.search(s -> s
    .index("orders")
    .query(q -> q.bool(b -> b
        .filter(f -> f.term(t -> t.field("status").value("PAID")))
        .must(m -> m.match(t -> t.field("description").query("phone 15")))
        .filter(f -> f.range(r -> r.field("createdAt")
            .gte(JsonData.of(start)))))),
    Order.class);
```

连接池复用即可（客户端线程安全），不要每请求 new client；connect/socket 超时显式设置，防 ES 慢查询拖死业务线程池。

## 7.2 Spring Data Elasticsearch

Spring Boot 3.x + Spring Data Elasticsearch 5.x 内部即走新版 Java Client，配置收敛在 `application.yml`：

```yaml
spring:
  elasticsearch:
    uris: https://es1:9200
    username: app_writer
    password: ${ES_PASSWORD}
    restclient.ssl.bundle: es-ca     # 结合 Spring 的 SSL bundle 配置 CA
```

```java
// 实体映射:注意 @Document 的分片/副本设置与 18 篇 mapping 纪律一致
@Document(indexName = "orders", shards = 3, replicas = 1)
public class Order {
    @Id
    private String id;
    @Field(type = FieldType.Keyword)
    private String orderNo;
    @Field(type = FieldType.Text, analyzer = "ik_max_word")
    private String description;
    @Field(type = FieldType.Date)
    private LocalDateTime createdAt;
}

public interface OrderRepository extends ElasticsearchRepository<Order, String> {
    List<Order> findByOrderNo(String orderNo);   // 方法名派生查询
}

// 复杂查询用 ElasticsearchClient 或 NativeQuery 模板
@Autowired ElasticsearchOperations ops;

String json = """
    {"bool": {"filter": [{"term": {"status": "PAID"}}]}}
    """;
var query = new NativeQueryBuilder()
    .withQuery(q -> q.withJsonString(json))
    .withPageable(PageRequest.of(0, 20))
    .build();
SearchHits<Order> hits = ops.search(query, Order.class);
```

> **注意**：Spring Data 的自动索引创建（`spring.elasticsearch.*` + `@Document(createIndex=false)` 权衡）只适合开发环境；生产 mapping 应由 **索引模板 + CI 发布**管理，让应用启动时"顺手建索引"迟早造成 mapping 漂移。深度分页、scroll、聚合在 Spring Data 里表达力有限，复杂检索直接用底层 `ElasticsearchClient`——`ElasticsearchClients.elasticsearchClient(transport)` 可从 ops 里取，两套并存不冲突。

## 7.3 选型对照

| 维度 | Java Client 8.x | Spring Data ES |
|------|-----------------|----------------|
| 抽象层级 | 原生 API（与 REST 一一对应） | Repository + 注解映射 |
| 功能完整度 | 全（REST 全部能力，含 EQL、CCS、ESQL） | 子集，复杂聚合要绕 |
| 上手成本 | 需熟悉 DSL | CRUD 快 |
| 推荐场景 | 搜索中台、网关、写入管道 | 常规业务 CRUD + 简单检索 |

# 8. Grafana ES 数据源

```text
Grafana → Configuration → Data Sources → Elasticsearch
  URL: http://es:9200
  Index: orders-*
  Time field: created_at
```

**Grafana vs Kibana**：
- Kibana：ES 原生，功能全
- Grafana：跨数据源聚合（Prometheus + ES 同一面板）

**设计考量**：指标走 Prometheus、日志/链路走 ES 的团队，Grafana 做统一"告警 + 总览大盘"、Kibana 做日志深挖与 ES 运维界面，是双 UI 共存的合理分工；数据源账号给只读角色（仅 `monitor` + 目标索引 `read`），Grafana 匿名 viewer 角色保持最小权限。

# 9. 小结

| 组件 | 一句话定位 | 关键落地要点 |
|------|-----------|--------------|
| Beats/Agent | 采集端 | 新集群一律 Elastic Agent + Integration；registry/时区是两大暗坑 |
| Logstash | 加工与缓冲 | grok 失败进 DLQ；batch/workers 决定吞吐上限 |
| Elasticsearch | 存储分析 | 模板 + ILM 先行，别让动态映射接管 schema |
| Kibana/Fleet/APM | 读端与管控 | KQL 面向普通人；APM 用 trace_id 与日志互跳 |
| Java 集成 | 业务接入 | 搜索中台用 Java Client，常规 CRUD 用 Spring Data，mapping 交给模板 |

下一篇（18）把这些组件的运维约束汇总成一套可执行的最佳实践清单。
