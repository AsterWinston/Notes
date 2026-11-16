# 1. 告警架构

Kibana Alerting 是 Elastic Stack 8.x 的**原生告警框架**，其核心设计是把"什么时候判断"、"判断什么"、"通知谁"三段职责彻底解耦：Rule（规则）负责周期性求值，Alert（告警实例）负责跟踪一次从触发到恢复的完整状态机，Connector（连接器）负责把消息投递到外部系统。这种解耦带来的直接好处是**同一个 Connector 可以被任意多条规则复用**，修改 SMTP 密码或 Slack Webhook 地址时只需改一处，而不需要在几十条规则的 YAML 里逐个替换。

```text
┌──────────────────────────────────────────────────────┐
│                   Kibana Alerting                      │
│                                                       │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐       │
│  │   Rule   │───→│  Action  │───→│ Connector│       │
│  │ (规则)    │    │  (动作)   │    │ (连接器)  │       │
│  │ WHEN...  │    │  THEN... │    │ Email    │       │
│  └──────────┘    └──────────┘    │ Slack    │       │
│                                  │ Webhook  │       │
│                                  └──────────┘       │
└──────────────────────────────────────────────────────┘
```

规则的周期性求值由 **Task Manager** 驱动，任务记录持久化在 `.kibana_task_manager` 索引中，因此多台 Kibana 实例可以组成集群共同分担告警调度，某台实例宕机后任务会被其他实例接管，**告警不会因为单实例故障而丢失**。这也是 8.x 告警与 Watcher 在部署形态上的本质差异——Watcher 只能在 master 节点上运行，而 Alerting 天然是分布式调度的。

> **注意**：Alerting 框架本身在 Basic License 下可用（如 Elasticsearch query、Index threshold 这类通用规则），但**按功能分层的规则类型受 License 限制**——Watcher、ML 异常检测规则、SIEM 检测规则、Service Level Objective 规则分别需要 Platinum 或 Enterprise。选型时先确认 License 档位，避免规则创建到一半发现不可用。

# 2. 规则类型

8.x 内置的规则类型按"数据源 + 判断方式"划分，最常用的是前三类：

| Rule Type | 说明 | 适用场景 | License |
|-----------|------|---------|---------|
| **Elasticsearch query** | 基于查询条件匹配文档数 | 错误日志告警、安全事件 | Basic |
| **Index threshold** | 按时间字段 + 分组维度聚合，指标超过阈值 | 订单量下降、响应时间超标 | Basic |
| **Metric threshold** | 单指标阈值判断（基于 Metrics Data View） | CPU > 90%、磁盘 > 85% | Basic |
| **Custom** | 自定义执行器，用 ES 查询 + Mustache 完全掌控判定逻辑 | 复杂业务规则 | Basic |
| **Geo-containment** | 地理围栏进出判断 | 车辆进出围栏、骑手轨迹 | Platinum |
| **Machine Learning** | ML 异常分数阈值 | 无固定阈值的异常检测告警 | Platinum |
| **Watcher** | 直接复用 ES 端 Watch 作为告警源 | 存量 Watch 迁移过渡 | Platinum |
| **Service Level Objective** | SLO 预算燃烧率告警 | 错误预算快烧完时提前预警 | Enterprise |

## 2.1 Elasticsearch Query Rule

Elasticsearch query 规则是最直白的类型：**在给定时间窗口内跑一个查询，对命中总数做阈值比较**。它只关心"命中了多少条"，不关心是哪台机器、哪个接口贡献的，所以适合总量型告警，不适合需要定位到具体维度的告警——那种场景应该用 Index threshold。

```text
Rule → Create rule → Elasticsearch query

Name: "High error rate"
Index: logs-*
Query: level:ERROR
Threshold: count > 100 (for last 5 minutes)
Check every: 1 minute

Action:
  - Connector: Slack
  - Message: "High error rate detected: {{context.value}} errors in last 5min"
```

两个窗口参数的配合关系值得注意：`Check every` 决定**求值频率**，`Threshold` 的时间窗口决定**每次往回看多远**。两者可以不相等——每 1 分钟检查一次"过去 5 分钟"意味着同一条错误日志会被计入 5 次连续的求值窗口。这不是 bug，但会导致告警恢复时间被拉长：故障停止后还需要等窗口完全滑出，告警才会从 Active 转为 Recovered。

## 2.2 Index Threshold Rule

Index threshold 规则在查询之上**加了聚合与分组**：指定时间字段、聚合函数（count/sum/avg/max/min）、分组 by 字段，然后对**每个分组的聚合值独立判断阈值**。这意味着"每 15 分钟订单总额低于 10000"和"每个服务的 P95 延迟超过 500ms"这类带维度的告警都能一条规则覆盖，且告警消息里能拿到具体是哪个分组越界。

```text
Rule → Create rule → Index threshold

Index: orders*
Time field: created_at
Aggregation: SUM(amount)
Group by: category
Threshold: < 10000 (for last 15 minutes)

Action:
  - Connector: Email
  - Subject: "订单金额异常下降"
  - Body: "过去 15 分钟订单总额: ¥{{context.value}}"
```

**设计考量**：

- **分组字段基数要可控**：by 字段如果是 user_id 这种高基数字段，每次求值会产生海量分组，聚合查询本身会变成 ES 负担；按 host、service、status 这类低基数维度分组是常态做法。
- **阈值方向选对类型**：大于阈值（延迟、错误数）和小于阈值（订单量、成交量骤降）用同一个规则即可，Index threshold 支持 min/max 双向比较，业务量下跌告警不要用"查不到数据"来实现。

## 2.3 Custom Rule

当内置类型表达不了判定逻辑时，Custom 规则允许分别自定义**执行器（Executor）**、**调度器（Scheduler）**和**触发器（Trigger）**三层：执行器写 ES 查询 DSL，触发器用 Mustache 模板表达"什么条件下产生告警"。它是能力上限最高、也最容易写出难维护告警的类型。

```json
POST /api/alerting/rule
{
  "name": "payment-timeout-custom",
  "rule_type_id": ".index-threshold",
  "consumer": { "id": "alerting" },
  "schedule": { "interval": "1m" },
  "params": {
    "index": "logs-payment-*",
    "timeField": "@timestamp",
    "threshold": [100],
    "metrics": [{ "name": "count", "aggregation": "count" }],
    "groupBy": ["service.name"],
    "aggType": "s"
  },
  "actions": []
}
```

> **行业实践**：先穷尽 query/threshold 两类内置规则，确实表达不了再上 Custom。Custom 规则的排障成本明显更高——它的判定结果不再能从 UI 上的"阈值线"直接读出，需要点开告警详情看 Mustache 求值上下文。团队内应约定 Custom 规则必须附注释说明业务含义。

# 3. Connectors（连接器）

Connector 是**独立于规则保存的配置对象**，存的是目标系统的地址与凭据。8.x 在 Stack Management 中统一管理，规则里只做引用，凭据字段（SMTP 密码、PagerDuty routing key）以 `xpack.encryptedSavedObjects.encryptionKey` 加密存储——**没配这个加密密钥时连接器根本无法创建**，这是很多环境初次搭建时的第一个报错点。

| Connector | 配置 | 典型用途 |
|-----------|------|---------|
| **Email** | SMTP 服务器 + 收件人 | 正式告警、值班邮件 |
| **Indexer** | 写入指定 ES 索引 | 告警落库、二次加工 |
| **Webhook** | HTTP URL + Method + Headers | 对接自研平台、钉钉/企微机器人 |
| **PagerDuty** | Integration Key | P0 值班呼叫 |
| **Jira** | Site + 项目 | 自动创建工单 |
| **ServiceNow** | ITSM 实例 | 生成 Incident |
| **Slack / Teams** | Webhook URL + 频道 | 团队群通知 |
| **Server Log** | 无需配置 | 调试规则触发（写入 Kibana 日志） |
| **Connector UI / Sideline** | 自定义前端 | 告警详情旁展示业务上下文 |

以最常见的自建 IM 集成为例，Webhook 连接器配合 Mustache 模板可以把告警上下文原样透传：

```bash
curl -X POST "https://kibana:5601/api/connectors/webhook" \
  -H "kbn-xsrf: true" -H "Authorization: ApiKey <key>" \
  -d '{
    "name": "dingtalk-ops",
    "connector_type_id": ".webhook",
    "config": {
      "url": "https://oapi.dingtalk.com/robot/send?access_token=xxx",
      "method": "post",
      "headers": { "Content-Type": "application/json" },
      "hasAuthConnectors": [],
      "auth": { "authenticationType": "none" }
    },
    "secrets": {}
  }'
```

> **注意**：`kbn-xsrf: true` 请求头是 Kibana 所有非 GET API 的硬性要求，缺失会返回 401 或 409，与权限无关，是最容易被误判为认证问题的报错。另外出网告警（如公网 SaaS）要走代理时，需要在 `kibana.yml` 里配置 `xpack.actions.proxyUrl`，连接器本身不感知代理。

# 4. 告警生命周期

每条规则产生的不是一个"事件"，而是一个**有状态的 Alert 对象**，状态机如下：

```text
Rule Check (定时执行)
  │
  ├── 条件不满足 → New (观察) → 首次满足 → Active (告警中)
  │                                  ├── Action 执行
  │                                  └── 持续满足 → 按 Throttle 间隔更新上下文
  │
  ├── 条件恢复 → 停止周期内未再触发 → Recovered (通知恢复)
  │
  └── 手动 Mute / Snooze → Muted / Snoozed (静默, 状态保留)
```

8.x 的告警状态比 7.x 更丰富：`Active`、`Recovered`、`Acknowledged`（值班已认领）、`Closed`（确认处理完毕不再通知）、`Muted`、`Warning`（规则自身执行出错，比如查询超时——**规则坏了本身也是一种告警**，务必给 Warning 状态配置兜底通知，否则"沉默的告警系统"比误报更危险）。

**Throttle 与 Group 的分工**是理解这个生命周期的关键：Throttle 控制**同一规则、同一分组**在恢复前重复通知的最小间隔（旧版按分钟数配置，新版按"每 N 个检查周期"配置）；而 **Groups by 抑制**则解决"一个规则同时炸出 200 个分组"的横向风暴——每个分组独立成为一条 Alert，各自走自己的 Throttle 时钟，互不干扰。

```text
Rule: per-group alerting
  Groups by: host.name
  ┌─ host-A 越界 → Alert#1 (Active, 每 10 min 重复通知)
  ├─ host-B 越界 → Alert#2 (独立时钟)
  └─ host-A 恢复 → Alert#1 → Recovered, 发恢复通知
```

# 5. 告警管理

日常的告警治理入口集中在 Stack Management → Rules and Connectors：

```text
Stack Management → Rules and Connectors
  ├── 查看所有规则及其状态
  ├── Enable/Disable 规则
  ├── Mute 静默（临时关闭）
  ├── Snooze 定时静默（接下来 1h/8h/本周不告警）
  ├── 查看告警历史 (Alerts 页签, 可按状态/规则过滤)
  └── 规则执行诊断 (Metrics: 执行耗时、失败次数)
```

```bash
# API 管理
GET  /api/alerting/rules/_find
POST /api/alerting/rules/{rule_id}/_mute_all
POST /api/alerting/alerts/{alert_id}/_ack   # 值班认领
# 告警历史（含完整状态迁移记录）
GET  /api/alert_history?perPage=20
```

> **行业实践**：告警历史默认保留 15 天（`xpack.alerting.maxAlertHistory` 可调），治理告警噪音（误报率、重复率统计）要靠这份历史数据，所以**不要把保留期改得更短**；做大促封网时优先用 Snooze 而不是 Disable 规则——Snooze 期间规则仍在求值、状态仍在维护，只是不发通知，封网结束自动恢复，避免忘记重新启用。

# 6. 告警变量模板

Action 消息用 **Mustache** 模板渲染，可用变量随规则类型不同而不同，通用的一组如下：

```mustache
# 邮件模板中可用的变量
Rule: {{rule.name}}
Time: {{context.date}}
Value: {{context.value}}
Message: {{context.message}}
Threshold: {{context.threshold}}

# 示例
Subject: "[{{rule.severity}}] {{rule.name}}"
Body: |
  Alert triggered at {{context.date}}
  Current value: {{context.value}}
  Threshold: {{context.threshold}}
  Link: {{context.link}}
```

三个高价值变量常被忽略：`{{state.tags}}` 携带告警对象的自定义标签（可用于路由到不同值班群）、`{{context.hits}}` 在 query 规则里直接内嵌**触发告警的原始文档样例**（值班人员不用先登录再查日志）、`{{context.link}}` 是**跳转到 Discover 精确定位这批数据的深链**，一条告警消息里带上它，平均定位时间可以从分钟级降到秒级。

**设计考量**：

- **消息要自带上下文**：告警文案 = 什么指标 + 当前值 + 阈值 + 时间窗 + 深链，五要素缺一，值班就多花一次跳板操作。
- **模板里不要放重查询**：Mustache 只在已求出的上下文上渲染，不要在模板语义上依赖二次查询，Action 阶段的延迟直接计入告警时延。

# 7. Alerting 与 Watcher 对比

Watcher 是 ES 端的**老一代告警机制**，Watch 定义、执行、审计全部发生在 Elasticsearch 内部。8.x 依然支持，但 Elastic 官方立场是**新功能一律使用 Kibana Alerting**，Watcher 主要保留给存量迁移与需要深度脚本定制的场景。

| 维度 | Kibana Alerting (8.x) | Watcher (ES 端) |
|------|----------------------|-----------------|
| 定义方式 | Kibana UI / REST API | Watch JSON + Watcher API |
| License | Basic 起（部分规则类型 Platinum+） | 全程 Platinum 起 |
| 执行调度 | Task Manager，多 Kibana 实例分担 | ES master 节点单点执行 |
| 状态机 | Alert 对象，含 Ack/Closed 工作流 | Watch 状态较粗糙 |
| 抑制/分组 | Groups by 原生分组告警 | `condition` + `throttle_period` 手工实现 |
| 行动器 | Connector 统一管理，加密凭据 | `input/actions` 内联写死 |
| 模板 | Mustache | Mustache |
| 审计 | Alert History + `.kibana-event-log` | `.watcher-watch-history` / `.watcher-audit` |
| 脚本能力 | 受限（自定义规则类型也有限） | 可嵌 painless 全脚本，灵活性极高 |
| 官方定位 | 推荐方向 | 维护状态，存量兼容 |

> **注意**：两种机制并存时最容易出现**重复告警**——同一个业务条件既写了 Watch 又建了 Rule。迁移期建议用 Indexer 连接器把 Alerting 事件写回索引，与 Watch 的 audit 索引对账一周，确认覆盖等价后逐个停用 Watch，而不是直接一刀切。

# 8. 生产告警设计

告警系统的质量不由"规则数量"衡量，而由**每条告警的可执行性**衡量。阈值怎么定是第一个问题：**静态阈值应参考历史分位数而非拍脑袋**——响应时间类指标用 p95/p99 的历史基线上浮一定比例，业务量类指标用同比（昨天同一时刻）与环比（前 15 分钟）双重条件，避免"凌晨低峰误报"和"高峰真故障漏报"。

```text
阈值选择经验:
  延迟类    → p95 > 基线×1.5 持续 2 个周期
  错误率类  → error_rate > max(0.5%, 7日均值×2)
  业务量类  → 环比跌 50% AND 同比跌 50%   (两个条件同时满足才告)
  资源类    → 绝对阈值 (disk > 85%) + 增长率 (预计 4h 内写满)
```

**避免告警风暴**的核心手段按优先级排列：

1. **分组告警而非全局阈值**：一条规则 Groups by service，每服务独立 Alert，天然带上定位维度；
2. **合理 Throttle**：恢复前同一分组最多每 5–10 分钟通知一次；
3. **依赖抑制**：机房断电时先告"网络不可达"，其下游的百条"服务超时"规则应通过 tag 条件被抑制——用 Custom 规则读取上游告警状态做条件判断可以实现；
4. **升级策略分层**：P0 走 PagerDuty 呼叫 + Slack @值班，P2 只进邮件汇总，**同一事件的各级通知必须收敛到同一 Connector 群**，让所有人看到同一份上下文。

> **行业实践**：每季度统计一次 Alert History 中的"触发次数 / 被 Ack 次数 / 触发了但无人处理数"，**处理率低于 30% 的规则要么改阈值要么删掉**。告警疲劳的根源不是告警多，而是"大家默认它又是误报"——一条没人理的告警不仅无效，还在稀释所有告警的可信度。规则数量控制在值班人员"每条都认识"的规模，通常是每团队 20–40 条。

# 9. 小结

- **架构上**：Rule（求值）→ Alert（状态机）→ Connector（投递）三层解耦，Task Manager 分布式调度，凭据统一加密管理。
- **选型上**：query 规则管总量、threshold 规则管带维度的指标、Custom 兜底复杂逻辑；Watcher 仅作存量兼容，新功能一律走 Alerting。
- **状态机是重点**：Throttle 控制纵向重复，Groups by 实现横向分组抑制，Recovered/Ack/Closed 构成值班工作流，规则自身的 Warning 状态也要有人收通知。
- **生产设计上**：阈值来自 p95 基线与同环比而非直觉，消息五要素自带上下文与深链，治理告警靠定期统计处理率、淘汰无人认领的规则——**可信度是告警系统唯一的资本**。
