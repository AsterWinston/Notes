
# 1. 仪表盘与告警在体系中的位置

OAP 把数据聚合好写进存储之后，用户感知到的 SkyWalking 其实就是两块：**UI 仪表盘（看）与 Alarm（喊）**。前者决定"出了事能不能三分钟定位到"，后者决定"没盯着屏幕时出事能不能找到你"。这两块的配置分布在三个地方：UI 侧的 `webapp.yml` 与初始化模板（`config/ui-initialized-templates`），OAP 侧的 `alarm-settings.yml`，以及两端之间的 GraphQL 查询协议。本篇按"部署 → 默认仪表盘 → 自定义 → 告警规则 → 告警通道 → 使用动线"的顺序展开。

# 2. UI 部署

```bash
# 独立部署 UI：9.x 的 UI 是 Spring Boot 应用，配置项在 webapp.yml 或同名环境变量
java -jar skywalking-ui.jar
# 环境变量形式（对应 webapp.yml 的 collector.ribbon.listOfServers）
export COLLECTOR_RIBBON_LISTOFSERVERS=oap-host:12800
export SERVER_PORT=8080
```

```yaml
# 或修改 dists/ui/webapp.yml（9.x 布局）
server:
  port: 8080

collector:
  path: /graphql
  ribbon:
    listOfServers: oap1:12800,oap2:12800
```

UI 是无状态的 GraphQL 代理 + 前端，可以随意横向扩容、放在 Nginx/Ingress 后面。`listOfServers` 配多个 OAP 地址时 UI 侧做轮询，等价于把查询压力也分摊到 OAP 集群。9.x 起 UI 还支持 **Helmx/自托管模式**之外的 Grafana 方案（OAP 提供 PromQL/LogQL 兼容 API 后，可用 Grafana 直连查询指标），但默认仪表盘生态仍以自带 UI 为主。

# 3. 默认仪表盘的组织方式

SkyWalking 的仪表盘不是"一堆图"，而是按 **General 与 Layer 模板 + 实体级别**组织的三层结构，UI 初始化时从 `config/ui-initialized-templates` 下的 yml 模板批量注册：

| 面板 | 说明 |
|------|------|
| **General Service** | 服务 QPS/RT/SLA/慢端点 + 服务拓扑入口 |
| **Service Instance** | 单实例 JVM CPU/Memory/GC/Thread + 实例调用端点 |
| **Endpoint** | 端点粒度指标（响应时间 percentiles/成功率/CALLS） |
| **Topology** | 服务依赖拓扑图（节点带 SLA/RT 着色） |
| **Trace** | 完整调用链查询（按服务/端点/tag/时间过滤） |
| **Log** | 关联日志查看（按 TraceId 跳转） |
| **Profile** | 线程快照火焰图分析任务 |
| **Event / Alarm** | 事件时间线与告警历史 |

模板按 layer 分目录（general、k8s、mesh、database、cache、mq、os 等），装了 eBPF/K8s 采集后对应 layer 的仪表盘自动出现在下拉里。**每个模板内部又分 Service/Instance/Endpoint 三层页面**，页面间通过点击实体自然下钻，这个结构是官方设计的"排障动线"的骨架。

> **注意**：仪表盘指标能不能显示，取决于 **OAL 脚本是否产出该指标**——比如默认模板里的 `service_sla_percent` 系列需要 percentile 类指标开启；自定义 layer 的图如果对应指标没定义，UI 只会显示空白而不是报错，这是"图不见了"类问题最常见的根因。

# 4. 自定义仪表盘

自定义有三种深度：**UI 上拖拽**（临时看盘，重启丢失，除非开启持久化）、**初始化模板 yml**（推荐，版本化管理）、**GraphQL 直接查询**（嵌入自建平台）。

模板方式在 `config/ui-initialized-templates/` 放一个 yml，OAP 启动时自动注册：

```yaml
# config/ui-initialized-templates/order-service.yml
templates:
  - id: order-service-dashboard
    title: "订单服务监控"
    layer: GENERAL
    configuration: |-
      children:
        - id: order-qps
          title: "订单 QPS"
          type: Line
          metrics:
            - name: service_cpm
              expression: service_cpm
              label: CPM
          widget: WidgetLine
        - id: order-rt
          title: "订单 RT"
          type: Line
          metrics:
            - name: service_resp_time
              expression: service_resp_time
              label: ms
          widget: WidgetLine
```

单个 widget 的核心是 **metrics 表达式（MQE）**，10.x 的 MQE 支持四则运算与聚合函数，比如用 `service_resp_time / service_sla` 组合出复合指标，或用 `avg(service_percentile{p='95'})` 取分位。自定义指标本身要先经 OAL/MAL 定义产出（见 Metrics 篇），仪表盘只是消费端——**"先有指标，后有图"的顺序不能反**。

> **行业实践**：把团队核心 SLO（下单成功率、支付 RT P99）做成独立模板进 Git 管理，随 OAP 配置一起发布；临场排障用的拖拽图用完即弃，不值得维护。

# 5. 拓扑图的使用

拓扑图数据来自 OAP 的 **STAM（流式拓扑分析）**：从每条 Trace 的 Exit/Entry Span 配对中提取"调用关系 + 调用指标"，因此它展示的是**真实流量构成的依赖**，而不是注册表意义上的依赖。三个使用要点：

- **边的方向与含义**：箭头从调用方指向被调方，边上的指标是 `service_relation_*`（客户端/服务端视角各一套），**两端看同一条边的延迟可能不同**——差值大通常指向网络/中间件（网关、MQ）而非服务本身。
- **无流量的服务不出现**：拓扑是流量驱动的，冷服务、刚上线未调用的服务不在图上属正常，不要当作"监控断了"。
- **下钻动线**：拓扑节点右键 → 该服务仪表盘 → 慢端点列表 → 单条 Trace 的 Span 树 → 日志/Profiling。这是官方刻意设计的排障路径，日常演练就按这条线走。

**设计考量**：跨层依赖（Java 服务调用 Mesh 服务再落到 DB）在混合探针环境下会拆成"General 拓扑 + Mesh 拓扑"两张图，靠虚拟服务（Virtual Database/Cache 机制）串起来；如果发现某条依赖只在一张图里出现，先检查两侧探针的 layer 配置而不是怀疑丢数据。

# 6. 告警规则

告警引擎是 OAP 内置的 **alarm 模块**，内存中的时间窗口队列，按分钟粒度检查指标值——**这意味着告警最快也是分钟级**，别拿它做秒级熔断。配置文件 `config/alarm-settings.yml`，规则名必须以 `_rule` 结尾。

9.x 早期（≤9.5）的经典格式（阈值 + 持续次数）：

```yaml
# config/alarm-settings.yml (9.0~9.5)
rules:
  # 服务平均响应时间告警
  service_resp_time_rule:
    metrics-name: service_resp_time
    op: ">"
    threshold: 1000              # 超过 1s
    period: 10                   # 检查最近 10 分钟窗口
    count: 3                     # 窗口内满足 3 次才告警
    silence-period: 10           # 告警后静默 10 分钟
    message: "服务 {name} 响应时间超过 1000ms"
    tags:
      level: WARNING

  # 服务成功率告警
  service_sla_rule:
    metrics-name: service_sla
    op: "<"
    threshold: 8000              # SLA 以万分比计，8000 = 80%
    period: 10
    count: 2
    message: "服务 {name} 成功率低于 80%"

  # 百分位响应时间（P50/75/90/95/99 任一超标）
  service_percentile_rule:
    metrics-name: service_percentile
    threshold: 1000
    labels: p50,p75,p90,p95,p99
    op: ">"
    period: 10
    count: 3
    message: "服务 {name} 响应时间分位超 1000ms"
```

9.6 起规则改为 **MQE 表达式**驱动（根运算必须是比较/布尔操作，结果为 SINGLE_VALUE）：

```yaml
# config/alarm-settings.yml (9.6+/10.x)
rules:
  service_resp_time_rule:
    # 最近窗口平均 RT 超过 1s
    expression: avg(service_resp_time / 1000) > 1
    period: 10
    silence-period: 10
    message: "服务 {name} 响应时间超过 1000ms"
    tags:
      level: CRITICAL
  # 环比/基线：与历史基线比较（10.2.0+ 支持 baseline 操作）
  service_resp_time_baseline_rule:
    expression: sum(service_resp_time > baseline(service_resp_time, upper)) > 3
    period: 10
    message: "服务 {name} 响应时间高于基线预测值（近 10 分钟内 3 分钟超标）"
```

常用规则类型与对应写法：

| 类型 | ≤9.5 写法要点 | 9.6+ 表达式示例 |
|------|-------------|----------------|
| 阈值 | `metrics-name + op + threshold` | `avg(service_resp_time) > 1000` |
| 慢 SQL/DB | `database_access_resp_time > 1000`（DB scope 同 service） | `sum(database_access_resp_time > 500) >= 3` |
| 成功率 | `service_sla`（万分比） | `avg(service_sla / 100) < 85` |
| 百分位 | `labels: p50,p95,p99` | `sum(service_percentile{p='95'} > 1000) >= 3` |
| 环比/基线 | 无原生支持，只能双指标规则近似 | `baseline(metric, upper/lower)` 操作 |
| 多条件复合 | 拆多条规则 | `(avg(service_sla/100) > 80) && (avg(service_percentile{p='0'}) > 1000)` |

几个容易踩的语义细节：**`period` 是缓存窗口分钟数，告警不支持比分钟更细的周期**；`count`（≤9.5）/表达式里的 `sum(... ) >= N`（9.6+）是"窗口内命中 N 个分钟点"，用持续命中过滤瞬时毛刺；**`silence-period` 默认等于 period**，同一规则同一实体在静默期内只发一次；10.4 起告警内核支持**恢复通知**（recovery 消息带 `recoveryTime`，条件转好即通知），配合 webhook 的 recovery 通道能显著减少"值班不知道服务恢复了"的来回确认。SLA 指标的万分比单位也常被误写（阈值 99 实际等于 0.99%），`op` 与单位要成对检查。

> **注意**：一条规则只针对**同一实体层级**，不要在表达式里混用 service 与 endpoint 级指标；`include-names/exclude-names`（及对应 regex）用于圈定实体范围，核心链路和非核心链路分别设不同阈值就靠它。告警规则支持动态配置热更新（`configuration` 模块接 zookeeper/apollo 等），生产上改规则不该重启 OAP。

# 7. 告警通知通道

9.6 起（与 MQE 同期）通知端重构为 **hooks 体系**：按类型（webhook、slack、wechat、dingtalk、feishu、welink、pagerduty、discord、gRPC）配多个具名 hook，规则用 `hooks: ["slack.custom1"]` 绑定，`is-default: true` 的全局兜底（更早版本是全局 `webhooks` 列表）：

```yaml
# hooks 节选：webhook + 钉钉 + 企业微信
hooks:
  webhook:
    default:
      is-default: true
      urls:
        - http://alertmanager:9093/api/v1/alerts
      recovery-urls:
        - http://alertmanager:9093/api/v2/recovery
    ops-dingtalk:
      urls:
        - http://webhook-gateway:8080/dingtalk/ops
  dingtalk:
    custom1:
      text-template: |-
        { "msgtype": "text", "text": { "content": "SkyWalking 告警: \n %s." } }
      webhooks:
        - url: https://oapi.dingtalk.com/robot/send?access_token=xxx
          secret: dummysecret   # 钉钉加签密钥
  wechat:
    custom1:
      text-template: |-
        { "msgtype": "text", "text": { "content": "SkyWalking 告警: \n %s." } }
      webhooks:
        - https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx
```

裸 webhook 收到的消息体是 **AlarmMessage 数组**，字段结构化程度足以直接路由：

```json
[{
  "scopeId": 1,
  "scope": "SERVICE",
  "name": "order-service",
  "id0": "12",
  "id1": "",
  "ruleName": "service_resp_time_rule",
  "alarmMessage": "服务 order-service 响应时间超过 1000ms",
  "startTime": 1762598400000,
  "tags": [{ "key": "level", "value": "CRITICAL" }]
}]
```

`scope/name/id0/id1/tags` 足够让对接服务（如自研告警平台或 Alertmanager receiver）做**按团队路由、按 level 分级、去重聚合**；10.4+ 的恢复消息还会带上 `recoveryTime`（新版另有 `uuid` 关联触发与恢复两条消息）。gRPC hook 则对应 `alarm-hook.proto`，适合接入内部 RPC 体系的通知服务。HTTP 类 hook 有统一超时 `alarm.default.httpTimeout`（默认 12s，在 `application.yml` 不在 `alarm-settings.yml`）。

> **行业实践**：通道配置常见分层——P0 级规则绑定 PagerDuty/电话网关，业务群绑定钉钉/企微/飞书，机器可处理的原始告警走 webhook 进 Alertmanager 统一去重降噪。**不要**把所有规则都发到同一个群：告警疲劳会让真正致命的告警一起被屏蔽。

# 8. 指标下钻动线

仪表盘的价值在"下钻是否顺手"，推荐把标准动线固化成团队 SOP：

```text
告警消息(带 scope/name/ruleName)
   │
   ▼
服务仪表盘 ──确认指标异常──▶ 拓扑图(看依赖是否连带异常)
   │                          │
   ▼                          ▼
慢端点 TopN 列表          异常下游服务(循环同一下钻)
   │
   ▼
Trace 列表(按端点+时间+tag 过滤) ──▶ 单条 Trace Span 树
   │                                   │
   ▼                                   ▼
日志页(按 TraceId)                耗时最长 Span → Profiling 火焰图
```

配套习惯：Trace 检索优先用 **searchable tags**（`core.default.searchableTracesTags` 默认含 http.status_code、db.instance 等，自定义 tag 要显式加入才可按 tag 查询）；告警消息里的 `name` 直接就是下钻入口的实体名；UI 右上角时间范围与"聚合粒度分钟/小时"影响图表形态，看瞬时尖刺时把粒度调细。

# 9. 最佳实践

1. **告警分级**：用 `tags.level`（CRITICAL/WARNING/INFO）标记，通道按级别路由；P0 只给 SLA 与核心 RT
2. **持续命中才告警**：`count`/`sum(... ) >= N` 过滤瞬时毛刺，宁可晚一分钟也不要狼来了
3. **静默期必须设**：避免一次故障刷屏（同一规则在 silence-period 内只发一条）
4. **恢复通知**：10.4+ 开启恢复通知，配合 webhook 通道闭环告警生命周期
5. **规则外置进 Git**：alarm-settings.yml 与 ui-initialized-templates 同 OAP 配置一起评审发布，改规则走动态配置热更新
6. **每季度做一次告警审计**：合并长期不触发的僵尸规则，拆分一触发就淹没值班的大杂烩规则

# 10. 小结

| 主题 | 关键结论 |
|------|---------|
| 仪表盘组织 | General + Layer 模板 × Service/Instance/Endpoint 三层下钻，模板进 Git 管理 |
| 自定义图 | 先 OAL/MAL 定义指标，再模板注册 widget；MQE 表达式做复合指标 |
| 拓扑 | 流量驱动（STAM），客户端/服务端视角延迟差值指向网络层；是下钻动线的中转站 |
| 告警规则 | 9.x 阈值式、10.x MQE 表达式式；分钟级窗口 + count 持续命中 + silence-period 防刷屏 |
| 通知通道 | 10.x hooks 具名绑定 + is-default 兜底；webhook 消息体结构化，适合对接 Alertmanager 分级路由 |

下一篇（17 最佳实践）把这些组件在生产环境的配置纪律与反模式汇总成清单。
