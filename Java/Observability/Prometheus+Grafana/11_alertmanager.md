
# 1. 架构与原理

Alertmanager 是 Prometheus 生态中的**告警管理组件**，负责接收 Prometheus Server 发送的告警，进行去重、分组、路由、抑制、静默处理后，通过多种渠道发送通知。

```
Prometheus Server     Alertmanager                       Notification
+-------------+     +------------------+              +---------------+
| Alert Rule  |     | 接收告警          |              | Email         |
| --> Firing  |---->| --> 分组          |---- 发送 --> | Slack/DingTalk|
|             |     | --> 抑制          |              | PagerDuty     |
|             |     | --> 静默          |              | Webhook       |
+-------------+     | --> 去重          |              +---------------+
                    +------------------+
```

# 2. 安装与部署

## 2.1 二进制

```bash
wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz
tar xvf alertmanager-0.27.0.linux-amd64.tar.gz
cd alertmanager-0.27.0.linux-amd64
./alertmanager --config.file=alertmanager.yml
```

## 2.2 Docker

```bash
docker run -d \
  --name alertmanager \
  -p 9093:9093 \
  -v /path/to/alertmanager.yml:/etc/alertmanager/alertmanager.yml \
  prom/alertmanager:v0.27.0
```

## 2.3 Kubernetes Helm

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm upgrade --install alertmanager prometheus-community/alertmanager
```

# 3. 配置文件结构

```yaml
global:
  resolve_timeout: 5m         # 告警自动解决时间
  smtp_from: 'alertmanager@example.com'
  smtp_smarthost: 'smtp.example.com:587'
  smtp_auth_username: 'user'
  smtp_auth_password: 'password'
  slack_api_url: 'https://hooks.slack.com/services/...'

route:
  receiver: 'default'
  group_by: ['alertname', 'cluster']
  group_wait: 30s           # 组内第一条告警等待时间
  group_interval: 5m        # 组内新告警发送间隔
  repeat_interval: 4h       # 重复告警发送间隔
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty'
      repeat_interval: 10m

receivers:
  - name: 'default'
    email_configs:
      - to: 'team@example.com'

  - name: 'pagerduty'
    pagerduty_configs:
      - routing_key: '...'
```

# 4. 路由树（Route Tree）

Alertmanager 使用**树形路由结构**进行告警分发，这是其最核心的配置：

```yaml
route:
  receiver: 'default'
  group_by: ['alertname', 'cluster']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty'
      repeat_interval: 5m
      routes:
        - match:
            team: 'infra'
          receiver: 'pagerduty_infra'
        - match:
            team: 'sre'
          receiver: 'pagerduty_sre'

    - match_re:
        severity: 'warning|info'
      receiver: 'slack-notifications'

    - match:
        job: 'pushgateway'
      receiver: 'noop'   # 忽略 pushgateway 的告警
```

## 4.1 路由匹配优先级

1. 从根节点开始，逐级向下匹配
2. 同级路由按配置顺序匹配，**第一个匹配生效**
3. 如果子路由未匹配，使用父路由的 receiver

## 4.2 路由配置字段

| 字段 | 说明 |
|------|------|
| `receiver` | 接收者名称 |
| `group_by` | 分组标签列表 |
| `match` | 精确匹配 |
| `match_re` | 正则匹配 |
| `continue` | 匹配后是否继续（默认 false） |

# 5. 告警分组（Grouping）

分组是 Alertmanager 最重要的功能之一，可将相似的告警合并为一个通知。

## 5.1 分组流程

```
原始告警流：
  HostA: CPU > 80%
  HostB: CPU > 80%
  HostC: CPU > 80%
  HostA: Mem > 90%
  HostB: Mem > 90%

按 alertname 分组：
  组1: [CPU > 80%]
    - HostA: CPU > 80%
    - HostB: CPU > 80%
    - HostC: CPU > 80%
  组2: [Mem > 90%]
    - HostA: Mem > 90%
    - HostB: Mem > 90%

按 alertname + severity 分组：
  组1: [CPU > 80%, warning]
    ...
  组2: [CPU > 90%, critical]
    ...
```

## 5.2 分组时间参数

```yaml
route:
  group_wait: 30s      # 组内第一条告警的等待时间（积累更多告警再发送）
  group_interval: 5m   # 组内已有告警后，新告警的等待发送间隔
  repeat_interval: 4h  # 已发送的告警重复通知间隔
```

## 5.3 合理的参数设置

| 场景 | group_wait | group_interval | repeat_interval |
|------|-----------|---------------|-----------------|
| 常规 | 30s | 5m | 4h |
| 紧急(critical) | 10s | 1m | 30m |
| 通知(info) | 1m | 30m | 24h |

# 6. 抑制（Inhibition）

抑制是指当某个告警已触发时，抑制其他相关的告警，减少告警风暴。

## 6.1 抑制规则

```yaml
inhibit_rules:
  # 如果节点挂了，抑制该节点上的所有其他告警
  - source_match:
      severity: critical
    source_match_re:
      alertname: 'NodeDown|InstanceDown'
    target_match_re:
      severity: 'warning|info'
    equal: ['instance']   # 只有 instance 相同时才抑制

  # 如果告警已解决（resolved），不再抑制
  - source_match:
      alertname: 'DatacenterDown'
    target_match:
      severity: 'warning'
    equal: ['datacenter']
```

## 6.2 抑制参数

| 字段 | 说明 |
|------|------|
| `source_match` | 源告警匹配条件 |
| `source_match_re` | 源告警正则 |
| `target_match` | 被抑制目标匹配 |
| `target_match_re` | 被抑制目标正则 |
| `equal` | 指定标签相同才生效 |

# 7. 静默（Silencing）

静默是在指定时间内**静音特定的告警**，通常用于维护窗口或已知问题。

## 7.1 创建静默

Alertmanager UI：`http://alertmanager:9093/#/silences`

```bash
# 使用 amtool 创建静默
amtool silence add \
  --alertmanager.url=http://localhost:9093 \
  --author="admin" \
  --comment="Scheduled maintenance" \
  --duration=2h \
  severity=critical \
  instance=server1

# 查看静默列表
amtool silence list --alertmanager.url=http://localhost:9093

# 过期静默
amtool silence expire <silence_id> --alertmanager.url=http://localhost:9093
```

# 8. 通知模板

## 8.1 默认模板

Alertmanager 使用 **Go 模板** 格式化通知内容。

## 8.2 自定义模板

```yaml
templates:
  - '/etc/alertmanager/templates/*.tmpl'
```

```gotmpl
{{/* /etc/alertmanager/templates/my.tmpl */}}
{{ define "my_custom_slack" }}
[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}
{{ if eq .Status "firing" }}
**告警列表：**
{{ range .Alerts }}
• {{ .Labels.instance }} - {{ .Annotations.summary }}
  Value: {{ .Annotations.value }}
{{ end }}
{{ end }}
{{ end }}
```

## 8.3 模板中的变量

| 变量 | 说明 |
|------|------|
| `{{ .Status }}` | firing / resolved |
| `{{ .Alerts }}` | 告警列表 |
| `{{ .GroupLabels }}` | 分组标签 |
| `{{ .CommonLabels }}` | 公共标签 |
| `{{ .CommonAnnotations }}` | 公共注解 |
| `{{ .ExternalURL }}` | Alertmanager URL |

# 9. 通知渠道配置

## 9.1 Email

```yaml
receivers:
  - name: 'email-team'
    email_configs:
      - to: 'team@example.com'
        headers:
          subject: '{{ template "email.subject" . }}'
        html: '{{ template "email.html" . }}'
        send_resolved: true
```

## 9.2 Slack

```yaml
receivers:
  - name: 'slack-alerts'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/...'
        channel: '#alerts'
        title: '{{ template "slack.title" . }}'
        text: '{{ template "slack.text" . }}'
        send_resolved: true
```

## 9.3 企业微信

```yaml
receivers:
  - name: 'wechat'
    wechat_configs:
      - api_secret: '...'
        corp_id: '...'
        agent_id: '...'
        to_user: '@all'
        message: '{{ template "wechat.message" . }}'
```

## 9.4 Webhook

```yaml
receivers:
  - name: 'webhook'
    webhook_configs:
      - url: 'http://webhook.example.com/alerts'
        send_resolved: true
        http_config:
          basic_auth:
            username: 'admin'
            password: 'password'
```

# 10. 高可用部署

Alertmanager 支持**基于 Gossip 协议的高可用集群**，自动去重告警。

```yaml
# 集群模式启动
./alertmanager \
  --config.file=alertmanager.yml \
  --cluster.listen-address=0.0.0.0:9094 \
  --cluster.peer=alertmanager-1:9094 \
  --cluster.peer=alertmanager-2:9094
```

## 10.1 高可用配置

```yaml
# Prometheus 配置多个 Alertmanager
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - 'alertmanager-1:9093'
          - 'alertmanager-2:9093'
          - 'alertmanager-3:9093'
```

## 10.2 Gossip 优势

- 告警去重：集群中只会发送一次通知
- 故障转移：任一节点故障不影响告警发送
- 数据同步：静默、抑制等状态自动同步

# 11. amtool

```bash
# 查看告警状态
amtool alert list --alertmanager.url=http://localhost:9093

# 查看配置
amtool config show --alertmanager.url=http://localhost:9093

# 验证配置
amtool check-config alertmanager.yml

# 查询告警
amtool alert query --alertmanager.url=http://localhost:9093 --severity=critical
```
