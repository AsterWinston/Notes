
# 1. 告警架构

```
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

# 2. 规则类型

| Rule Type | 说明 | 适用场景 |
|-----------|------|---------|
| **Elasticsearch query** | 基于查询条件匹配文档数 | 错误日志告警、安全事件 |
| **Index threshold** | 聚合指标超过阈值 | 订单量下降、响应时间超标 |
| **Metric threshold** | 单指标阈值判断 | CPU > 90%、磁盘 > 85% |
| **Geo-containment** | 地理围栏 | 车辆进出围栏 |
| **Machine Learning** | ML 异常分数阈值 | 异常检测告警 |

## 2.1 Elasticsearch Query Rule

```
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

## 2.2 Index Threshold Rule

```
Rule → Create rule → Index threshold

Index: orders*
Time field: created_at
Aggregation: SUM(amount)
Threshold: < 10000 (for last 15 minutes)

Action:
  - Connector: Email
  - Subject: "订单金额异常下降"
  - Body: "过去 15 分钟订单总额: ¥{{context.value}}"
```

# 3. Connectors（连接器）

| Connector | 配置 |
|-----------|------|
| **Email** | SMTP 服务器 + 收件人 |
| **Slack** | Webhook URL + 频道 |
| **Webhook** | HTTP URL + Method + Headers |
| **ServiceNow** | ITSM 集成 |
| **PagerDuty** | 事件管理 |
| **Jira** | 创建 Issue |
| **Microsoft Teams** | Webhook |
| **Server Log** | 写入 Kibana 日志（调试用） |

# 4. 告警生命周期

```
Rule Check (定时执行)
  │
  ├── 条件不满足 → OK (绿色)
  │
  ├── 条件满足 → Active (红色)
  │     ├── Action 执行
  │     └── 持续满足 → > Throttle 间隔再次 Action
  │
  ├── 条件恢复 → Recovered (通知恢复)
  │
  └── 手动 Mute → Muted (静默)
```

# 5. 告警管理

```
Stack Management → Rules and Connectors
  ├── 查看所有规则及其状态
  ├── Enable/Disable 规则
  ├── Mute 静默（临时关闭）
  ├── Snooze 定时静默（接下来 1h 不告警）
  └── 查看告警历史

API:
  GET /api/alerting/rules
  POST /api/alerting/rule/{id}/_mute_all
```

# 6. 告警变量模板

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

# 7. 最佳实践

1. **分级告警**：P0 立即通知（Slack + PagerDuty），P2 仅邮件
2. **避免告警风暴**：设置 Throttle（同一规则 5 分钟内不重复告警）
3. **静默期**：计划内的维护窗口提前 Mute
4. **告警摘要**：多个规则汇总到同一 Connector，减少噪音
5. **恢复通知**：开启 Recovered Action，被告知问题已解决
