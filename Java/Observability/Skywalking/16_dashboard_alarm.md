
# 1. UI Dashboard 部署

```bash
# 独立部署 UI
java -jar skywalking-webapp.jar \
  --server.port=8080 \
  --spring.cloud.discovery.client.simple.instance.oap-service=http://oap:12800
```

```yaml
# 或修改 webapp/webapp.yml
server:
  port: 8080

collector:
  path: /graphql
  ribbon:
    listOfServers: oap1:12800,oap2:12800
```

# 2. Dashboard 功能

| 面板 | 说明 |
|------|------|
| **General Service** | 服务 QPS/RT/SLA/拓扑 |
| **Service Instance** | JVM CPU/Memory/GC/Thread |
| **Endpoint** | 端点粒度指标（P99/成功率） |
| **Topology** | 服务依赖拓扑图 |
| **Trace** | 完整调用链查询 |
| **Log** | 关联日志查看 |
| **Profile** | 性能剖析火焰图 |

# 3. 告警规则 DSL

```yaml
# config/alarm-settings.yml
rules:
  # 服务平均响应时间告警
  service_resp_time_rule:
    metrics-name: service_resp_time
    op: ">"
    threshold: 1000              # 超过 1s
    period: 10                   # 持续 10 分钟
    count: 3                     # 连续 3 次
    silence-period: 10           # 静默 10 分钟
    message: "服务 {name} 响应时间超过 1000ms"

  # 服务成功率告警
  service_sla_rule:
    metrics-name: service_sla
    op: "<"
    threshold: 99
    period: 10
    count: 2
    message: "服务 {name} 成功率低于 99%"

  # 端点响应时间告警
  endpoint_resp_time_rule:
    metrics-name: endpoint_resp_time
    op: ">"
    threshold: 3000
    period: 5
    count: 3
    message: "端点 {name} 响应时间超过 3s"

  # 实例 JVM 内存告警
  instance_jvm_old_gc_count_rule:
    metrics-name: instance_jvm_old_gc_count
    op: ">"
    threshold: 5
    period: 1
    count: 1
    message: "实例 {name} Full GC 频繁"
```

# 4. 告警通知 Hook

```yaml
# config/alarm-settings.yml
webhooks:
  - http://alert-manager:9093/api/v1/alerts         # AlertManager
  - http://wechat-webhook.example.com/send           # 企业微信
  - http://dingtalk-webhook.example.com/send         # 钉钉
  - http://feishu-webhook.example.com/send           # 飞书

grpchooks:
  - target-host: notification-service
    target-port: 11800
```

## 4.1 Webhook 格式

```json
{
  "scope": "Service",
  "name": "order-service",
  "id": "abc123",
  "alarmMessage": "服务 order-service 响应时间超过 1000ms",
  "startTime": 1705302000000
}
```

# 5. 自定义 Dashboard

```yaml
# config/ui-initialized-templates/custom-dashboard.yml
templates:
  - name: "订单服务监控"
    type: "DASHBOARD"
    configuration:
      children:
        - name: "订单 QPS"
          type: "Widget"
          metrics:
            - "endpoint_cpm"
        - name: "订单 RT"
          type: "Widget"
          metrics:
            - "endpoint_resp_time"
```

# 6. RBAC 权限

```yaml
# config/application.yml
authorization:
  enabled: true
  admin-token: admin-token-xxx       # 管理 API 令牌
```

**使用**：
- Admin Token 可执行 Profiling 任务、修改告警规则
- 普通用户仅查看 Dashboard

# 7. 最佳实践

1. **告警分级**：P0（服务 SLA < 99%）、P1（慢响应）、P2（慢端点）
2. **静默期设置**：避免告警风暴（一次问题 → 一条告警）
3. **Webhook 多通道**：企业微信 + 钉钉 + AlertManager 三通道覆盖
4. **条件触发**：连续 N 次告警才触发，避免瞬时波动
