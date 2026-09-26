
> 版本说明：本文档基于 **RabbitMQ 3.13.x**，覆盖 Management HTTP API、Prometheus + Grafana 监控方案、关键指标告警策略。

---

# 1. Management HTTP API

## 1.1 基础信息

- 基础 URL：`http://localhost:15672/api/`
- 认证方式：HTTP Basic Auth
- 认证格式：`-u username:password`
- 响应格式：JSON

## 1.2 核心 API 端点

### 1.2.1 概览

```bash
# 集群概览
curl -u admin:admin123 http://localhost:15672/api/overview

# 节点信息
curl -u admin:admin123 http://localhost:15672/api/nodes

# 特定节点详情
curl -u admin:admin123 http://localhost:15672/api/nodes/rabbit@rabbit1
```

### 1.2.2 交换机

```bash
# 列出所有交换机
curl -u admin:admin123 http://localhost:15672/api/exchanges
curl -u admin:admin123 http://localhost:15672/api/exchanges/%2F  # / VHost（URL编码）

# 特定交换机
curl -u admin:admin123 http://localhost:15672/api/exchanges/%2F/order.direct

# 创建交换机
curl -u admin:admin123 -X PUT \
  -H "Content-Type: application/json" \
  -d '{"type":"direct","durable":true}' \
  http://localhost:15672/api/exchanges/%2F/my.exchange
```

### 1.2.3 队列

```bash
# 列出所有队列
curl -u admin:admin123 http://localhost:15672/api/queues
curl -u admin:admin123 http://localhost:15672/api/queues/%2F

# 特定队列
curl -u admin:admin123 http://localhost:15672/api/queues/%2F/order.queue

# 创建队列
curl -u admin:admin123 -X PUT \
  -H "Content-Type: application/json" \
  -d '{"durable":true,"arguments":{"x-queue-type":"quorum"}}' \
  http://localhost:15672/api/queues/%2F/my.queue

# 清空队列（Purge）
curl -u admin:admin123 -X DELETE \
  http://localhost:15672/api/queues/%2F/my.queue/contents
```

### 1.2.4 绑定

```bash
# 列出交换机的所有绑定
curl -u admin:admin123 http://localhost:15672/api/exchanges/%2F/order.direct/bindings/source

# 列出队列的所有绑定
curl -u admin:admin123 http://localhost:15672/api/queues/%2F/order.queue/bindings

# 创建绑定
curl -u admin:admin123 -X POST \
  -H "Content-Type: application/json" \
  -d '{"routing_key":"order.create","arguments":{}}' \
  http://localhost:15672/api/bindings/%2F/e/order.exchange/q/order.queue
```

### 1.2.5 连接与信道

```bash
# 连接列表
curl -u admin:admin123 http://localhost:15672/api/connections

# 删除连接
curl -u admin:admin123 -X DELETE \
  http://localhost:15672/api/connections/connection-name

# 信道列表
curl -u admin:admin123 http://localhost:15672/api/channels
```

### 1.2.6 发布与消费

```bash
# 向交换机发布消息
curl -u admin:admin123 -X POST \
  -H "Content-Type: application/json" \
  -d '{"properties":{},"routing_key":"order.create","payload":"hello","payload_encoding":"string"}' \
  http://localhost:15672/api/exchanges/%2F/order.direct/publish

# 从队列获取消息（不删除）
curl -u admin:admin123 -X POST \
  -H "Content-Type: application/json" \
  -d '{"count":10,"ackmode":"ack_requeue_true","encoding":"auto","truncate":50000}' \
  http://localhost:15672/api/queues/%2F/order.queue/get
```

### 1.2.7 VHost 和用户

```bash
# VHost
curl -u admin:admin123 http://localhost:15672/api/vhosts
curl -u admin:admin123 -X PUT http://localhost:15672/api/vhosts/dev

# 用户
curl -u admin:admin123 http://localhost:15672/api/users
curl -u admin:admin123 -X PUT \
  -H "Content-Type: application/json" \
  -d '{"password":"pass123","tags":"management"}' \
  http://localhost:15672/api/users/newuser

# 权限
curl -u admin:admin123 -X PUT \
  -H "Content-Type: application/json" \
  -d '{"configure":".*","write":".*","read":".*"}' \
  http://localhost:15672/api/permissions/%2F/newuser
```

---

# 2. Prometheus 监控

## 2.1 安装 Prometheus 插件

```bash
rabbitmq-plugins enable rabbitmq_prometheus
```

> 插件启用后，Prometheus 指标端点为 `http://localhost:15692/metrics`

## 2.2 Prometheus 配置

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'rabbitmq'
    static_configs:
      - targets:
          - 'rabbit1:15692'
          - 'rabbit2:15692'
          - 'rabbit3:15692'
    metrics_path: '/metrics'
```

## 2.3 关键指标

### 2.3.1 节点指标

| 指标名 | 说明 | 告警建议 |
|--------|------|----------|
| `rabbitmq_node_mem_used` | 节点内存使用（字节） | > 80% 告警 |
| `rabbitmq_node_mem_limit` | 节点内存上限 | — |
| `rabbitmq_node_disk_space_free` | 磁盘剩余空间 | < 2GB 告警 |
| `rabbitmq_node_fd_used` | 文件描述符使用数 | > 80% 告警 |
| `rabbitmq_node_proc_used` | Erlang 进程数 | > 80% 告警 |
| `rabbitmq_node_sockets_used` | Socket 连接数 | > 80% 告警 |
| `rabbitmq_node_running` | 节点是否运行 | 0 告警 |
| `rabbitmq_node_uptime_seconds` | 节点运行时间 | 突然归零告警 |

### 2.3.2 队列指标

| 指标名 | 说明 | 告警建议 |
|--------|------|----------|
| `rabbitmq_queue_messages_ready` | 待消费消息数 | 持续增长告警（堆积） |
| `rabbitmq_queue_messages_unacked` | 已投递未确认消息数 | 持续增长告警（消费慢） |
| `rabbitmq_queue_messages` | 队列消息总数 | — |
| `rabbitmq_queues` | 队列总数 | — |
| `rabbitmq_queue_consumers` | 消费者数量 | 0 告警（无消费者） |

### 2.3.3 消息速率指标

| 指标名 | 说明 |
|--------|------|
| `rabbitmq_queue_messages_published_total` | 发布消息总量 |
| `rabbitmq_queue_messages_delivered_total` | 投递消息总量 |
| `rabbitmq_queue_messages_acknowledged_total` | 确认消息总量 |
| `rabbitmq_channel_messages_published_total` | 信道级别发布统计 |
| `rabbitmq_channel_messages_confirmed_total` | 确认统计 |

## 2.4 Grafana 告警规则示例

```yaml
# Prometheus 告警规则
groups:
  - name: rabbitmq_alerts
    rules:
      - alert: RabbitMQNodeDown
        expr: rabbitmq_node_running == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "RabbitMQ 节点 {{ $labels.node }} 已停止"

      - alert: RabbitMQMemoryHigh
        expr: (rabbitmq_node_mem_used / rabbitmq_node_mem_limit) > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "RabbitMQ 内存使用超过 85%"

      - alert: RabbitMQDiskLow
        expr: rabbitmq_node_disk_space_free < 2 * 1024 * 1024 * 1024
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "RabbitMQ 磁盘剩余空间小于 2GB"

      - alert: RabbitMQQueueMessagesGrowing
        expr: rate(rabbitmq_queue_messages_ready[5m]) > 0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "队列 {{ $labels.queue }} 消息持续堆积 10 分钟"

      - alert: RabbitMQNoConsumer
        expr: rabbitmq_queue_consumers == 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "队列 {{ $labels.queue }} 无消费者超过 5 分钟"

      - alert: RabbitMQHighUnacked
        expr: rabbitmq_queue_messages_unacked > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "队列 {{ $labels.queue }} 未确认消息超过 1000 条"
```

---

# 3. Grafana Dashboard

推荐的社区 Grafana Dashboard：
- **RabbitMQ-Overview** (ID: 10991) — 集群总览仪表盘
- **RabbitMQ-Queues** (ID: 4279) — 队列级别监控

## 3.1 自建 Dashboard 核心面板

```
┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│   Nodes Up   │ │  Memory %    │ │  Disk Free   │ │  Total Msgs  │
│     3/3      │ │    42%      │ │   5.2 GB     │ │    12,345    │
└──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Message Rates (publish / deliver / ack)  [Graph]           │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  ▓▓ publish  ▓▓ deliver  ▓▓ ack                      │    │
│  └─────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────┤
│  Top 10 Queues by Message Count  [Table]                    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ Queue Name        │ Ready │ Unacked │ Consumers │    │    │
│  │ order.queue       │ 5000  │ 200     │ 3        │    │    │
│  │ notification.q    │ 1200  │ 50      │ 2        │    │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

---

# 4. 日志管理

## 4.1 日志配置

```ini
# rabbitmq.conf
log.dir = /var/log/rabbitmq
log.file = rabbitmq.log
log.file.level = info
log.file.rotation.date = $D0
log.file.rotation.count = 7

# JSON 格式日志（便于 ELK）
log.file.formatter = json
```

## 4.2 关键日志排查

```bash
# 查看最新日志
tail -f /var/log/rabbitmq/rabbitmq.log

# 搜索错误
grep -i "error" /var/log/rabbitmq/rabbitmq.log

# 搜索特定队列相关日志
grep "order.queue" /var/log/rabbitmq/rabbitmq.log

# 检查启动日志
grep "started" /var/log/rabbitmq/startup_log
```

---

# 5. 消息追踪（Firehose Tracer）

Firehose Tracer 用于追踪每条消息的发布和投递过程。

```bash
# 启用追踪
rabbitmqctl trace_on

# 关闭追踪
rabbitmqctl trace_off

# 查看追踪队列
rabbitmqctl list_queues name | grep amq.rabbitmq.trace
```

> **性能影响**：追踪会显著增加 CPU 和网络开销，仅在调试时临时启用。

## 5.1 追踪使用（Java）

```java
// 声明追踪队列
channel.queueDeclare("trace.queue", true, false, false, null);

// 绑定到 amq.rabbitmq.trace topic 交换机
channel.queueBind("trace.queue", "amq.rabbitmq.trace", "#");

// 消费追踪消息
channel.basicConsume("trace.queue", true,
    (consumerTag, delivery) -> {
        String routingKey = delivery.getEnvelope().getRoutingKey();
        // routingKey: publish.{exchangeName} 或 deliver.{queueName}
        String body = new String(delivery.getBody(), StandardCharsets.UTF_8);
        System.out.printf("[Trace] %s → %s%n", routingKey, body);
    },
    consumerTag -> {}
);
```

---

# 6. 生产巡检 Checklist

| 检查项 | 方法 | 正常标准 |
|--------|------|----------|
| 节点运行状态 | `rabbitmq-diagnostics check_running` | OK |
| 端口连通性 | `rabbitmq-diagnostics check_port_connectivity` | OK |
| 内存告警 | `rabbitmq-diagnostics memory_breakdown` | Used < Limit |
| 磁盘告警 | `rabbitmq-diagnostics check_disk_alarms` | No alarms |
| 队列堆积 | API: `queues/{name}` → `messages_ready` | 无异常增长 |
| 消费者数量 | API: `queues/{name}` → `consumers` | > 0 |
| 死信队列 | API: `queues/{dlq}` → `messages_ready` | 0 或可控范围 |
| 连接数 | API: `connections` → count | < 上限 |
| 网络分区 | `rabbitmqctl cluster_status` → partitions | [] |
| 日志异常 | 扫描 ERROR/WARN 日志 | 无异常 |

---

# 7. 自动诊断脚本

```bash
#!/bin/bash
# rabbitmq-health-check.sh

echo "===== RabbitMQ 健康检查 $(date) ====="

# 1. 节点状态
echo "[1/6] 节点状态:"
rabbitmq-diagnostics check_running
if [ $? -ne 0 ]; then echo "ERROR: 节点未运行"; fi

# 2. 集群状态
echo "[2/6] 集群状态:"
PARTITIONS=$(rabbitmqctl cluster_status | grep partitions | grep -v '\[\]')
if [ -n "$PARTITIONS" ]; then
    echo "ERROR: 检测到网络分区: $PARTITIONS"
fi

# 3. 磁盘告警
echo "[3/6] 磁盘状态:"
rabbitmq-diagnostics check_disk_alarms

# 4. 内存
echo "[4/6] 内存使用:"
MEM_USED=$(rabbitmq-diagnostics memory_breakdown 2>/dev/null | grep "Total memory used" || echo "N/A")
echo "$MEM_USED"

# 5. 队列堆积
echo "[5/6] 消息堆积（前5）:"
curl -s -u admin:admin123 http://localhost:15672/api/queues | \
    jq -r 'sort_by(-.messages_ready) | .[:5] | .[] |
    "\(.name): ready=\(.messages_ready) unacked=\(.messages_unacknowledged) consumers=\(.consumers)"'

# 6. 死信队列
echo "[6/6] 死信队列:"
curl -s -u admin:admin123 http://localhost:15672/api/queues | \
    jq -r '.[] | select(.name | test("dlx|dead")) | "\(.name): ready=\(.messages_ready)"'

echo "===== 检查完成 ====="
```
