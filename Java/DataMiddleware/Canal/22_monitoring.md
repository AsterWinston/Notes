
# 1. 概述

Canal 提供了内置的监控指标，并支持通过 Prometheus + Grafana 进行可视化监控。监控体系覆盖了 Server 运行状态、Instance 消费延迟、系统资源等多个维度。

```
Canal Server ──> Prometheus Exporter ──> Prometheus ──> Grafana
                                                           │
                                                    ┌──────▼──────┐
                                                    │ AlertManager│
                                                    │  (告警通知) │
                                                    └─────────────┘
```

# 2. Prometheus 集成

## 2.1 启用 Prometheus

```properties
# canal.properties
canal.metrics.pull.port = 11112        # Prometheus 拉取端口
canal.metrics.enabled = true
```

## 2.2 暴露的指标

Canal 内置了以下指标（`http://localhost:11112/metrics`）：

```
# HELP canal_instance_subscribe_delay 订阅延迟
# TYPE canal_instance_subscribe_delay gauge
canal_instance_subscribe_delay{instance="example",} 0.0

# HELP canal_instance_client_packets 客户端请求包数
# TYPE canal_instance_client_packets counter
canal_instance_client_packets{instance="example",} 1500.0

# HELP canal_instance_publish_blocking_time Parser 阻塞时间
# TYPE canal_instance_publish_blocking_time gauge
canal_instance_publish_blocking_time{instance="example",} 0.0

# HELP canal_instance_received_binlog_bytes 接收的 binlog 字节数
# TYPE canal_instance_received_binlog_bytes counter
canal_instance_received_binlog_bytes{instance="example",} 1.234567e8

# HELP canal_instance_sink_blocking_time Sink 阻塞时间
# TYPE canal_instance_sink_blocking_time gauge
canal_instance_sink_blocking_time{instance="example",} 0.0

# HELP canal_instance_store_produce_seq Store 生产序列号
# TYPE canal_instance_store_produce_seq gauge
canal_instance_store_produce_seq{instance="example",} 456789.0

# HELP canal_instance_store_consume_seq Store 消费序列号
# TYPE canal_instance_store_consume_seq gauge
canal_instance_store_consume_seq{instance="example",} 456700.0

# HELP canal_instance_store_put_delay Store put 延迟
# TYPE canal_instance_store_put_delay gauge
canal_instance_store_put_delay{instance="example",} 0.0

# HELP canal_instance_store_get_delay Store get 延迟
# TYPE canal_instance_store_get_delay gauge
canal_instance_store_get_delay{instance="example",} 0.0

# HELP canal_instance_traffic_delay 流量延迟
# TYPE canal_instance_traffic_delay gauge
canal_instance_traffic_delay{instance="example",} 0.0

# HELP jvm_memory_bytes_used JVM 内存使用量
# TYPE jvm_memory_bytes_used gauge
jvm_memory_bytes_used{area="heap",} 5.36870912e8
jvm_memory_bytes_used{area="nonheap",} 1.23456789e8
```

## 2.3 指标说明

| 指标 | 类型 | 说明 |
|------|------|------|
| `canal_instance_subscribe_delay` | Gauge | 订阅延迟（秒），当前消费位点与最新位点的时间差 |
| `canal_instance_traffic_delay` | Gauge | 流量延迟（字节），当前消费位点与最新位点的 offset 差 |
| `canal_instance_client_packets` | Counter | 客户端请求包数量 |
| `canal_instance_received_binlog_bytes` | Counter | 接收的 binlog 总字节数 |
| `canal_instance_publish_blocking_time` | Gauge | Parser 发布到 Sink 的阻塞时间 |
| `canal_instance_sink_blocking_time` | Gauge | Sink 处理阻塞时间 |
| `canal_instance_store_produce_seq` | Gauge | RingBuffer 生产者序列号 |
| `canal_instance_store_consume_seq` | Gauge | RingBuffer 消费者序列号 |
| `canal_instance_store_put_delay` | Gauge | Store put 操作延迟 |
| `canal_instance_store_get_delay` | Gauge | Store get 操作延迟 |

# 3. Prometheus 配置

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'canal'
    static_configs:
      - targets:
        - '192.168.1.10:11112'    # Canal Server 1
        - '192.168.1.11:11112'    # Canal Server 2
        - '192.168.1.12:11112'    # Canal Server 3
    metrics_path: '/metrics'
    scrape_interval: 10s
```

# 4. Grafana Dashboard

## 4.1 核心面板

```json
{
  "dashboard": {
    "title": "Canal Monitor",
    "panels": [
      {
        "title": "订阅延迟",
        "targets": [
          {
            "expr": "canal_instance_subscribe_delay{instance=~\"$instance\"}"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Binlog 接收速率",
        "targets": [
          {
            "expr": "rate(canal_instance_received_binlog_bytes{instance=~\"$instance\"}[1m])"
          }
        ],
        "type": "graph",
        "yaxis": {
          "format": "bytes"
        }
      },
      {
        "title": "RingBuffer 水位",
        "targets": [
          {
            "expr": "canal_instance_store_produce_seq{instance=~\"$instance\"} - canal_instance_store_consume_seq{instance=~\"$instance\"}"
          }
        ],
        "type": "graph"
      },
      {
        "title": "客户端请求 QPS",
        "targets": [
          {
            "expr": "rate(canal_instance_client_packets{instance=~\"$instance\"}[1m])"
          }
        ],
        "type": "graph"
      },
      {
        "title": "JVM Heap 内存",
        "targets": [
          {
            "expr": "jvm_memory_bytes_used{area=\"heap\",instance=~\"$instance\"}"
          }
        ],
        "type": "graph"
      }
    ]
  }
}
```

## 4.2 告警规则

```yaml
# prometheus_rules.yml
groups:
  - name: canal_alerts
    interval: 30s
    rules:
      # 消费延迟告警
      - alert: CanalHighDelay
        expr: canal_instance_subscribe_delay > 60
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Canal 消费延迟超过 60 秒"
          description: "Instance {{ $labels.instance }} 消费延迟 {{ $value }} 秒"

      # 严重延迟告警
      - alert: CanalCriticalDelay
        expr: canal_instance_subscribe_delay > 300
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Canal 消费延迟超过 5 分钟！"
          description: "Instance {{ $labels.instance }} 消费延迟 {{ $value }} 秒"

      # RingBuffer 积压告警
      - alert: CanalRingBufferBacklog
        expr: (canal_instance_store_produce_seq - canal_instance_store_consume_seq) > 10000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Canal RingBuffer 积压超过 10000"
          description: "Instance {{ $labels.instance }} RingBuffer 积压 {{ $value }}"

      # Binlog 流量异常
      - alert: CanalNoBinlogTraffic
        expr: rate(canal_instance_received_binlog_bytes[5m]) == 0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Canal 5 分钟内没有收到 binlog 数据"
          description: "Instance {{ $labels.instance }} 可能连接断开"

      # JVM 内存告警
      - alert: CanalHighMemory
        expr: jvm_memory_bytes_used{area="heap"} / jvm_memory_bytes_max{area="heap"} > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Canal JVM 堆内存使用超过 85%"
          description: "Instance {{ $labels.instance }} 堆内存使用率 {{ $value | humanizePercentage }}"
```

# 5. Canal Admin 内置监控

## 5.1 Admin 仪表盘

Canal Admin 提供了基础的可视化监控：

```
Admin UI → 系统监控 → 选择 Instance

展示指标：
- Server 运行状态
- Instance 运行状态
- 当前位点信息
- 消费延迟
- 客户端连接数
- 系统资源使用
```

## 5.2 监控 API

```bash
# 获取 Instance 运行状态
curl http://localhost:8089/api/v1/canal/instance/log/192.168.1.10/example

# 输出：
{
  "code": 20000,
  "data": {
    "instance": "example",
    "serverStatus": "RUNNING",
    "cursorStatus": "OK",
    "currentCursor": {
      "journalName": "mysql-bin.000123",
      "position": 45678901,
      "timestamp": 1672545600000
    },
    "masterPosition": {
      "journalName": "mysql-bin.000123",
      "position": 45789123
    },
    "delay": 110222,  # 延迟字节数
    "delaySeconds": 2  # 延迟秒数
  }
}
```

# 6. 自定义监控指标

## 6.1 自定义指标收集

```java
@Component
public class CanalCustomMetrics {
    
    private final MeterRegistry registry;
    
    // 自定义计数器
    private Counter dmlInsertCounter;
    private Counter dmlUpdateCounter;
    private Counter dmlDeleteCounter;
    private Counter ddlCounter;
    
    // 自定义计时器
    private Timer processTimer;
    
    public CanalCustomMetrics(MeterRegistry registry) {
        this.registry = registry;
        
        this.dmlInsertCounter = Counter.builder("canal.custom.dml.insert")
            .description("INSERT 事件计数")
            .tag("destination", "example")
            .register(registry);
            
        this.dmlUpdateCounter = Counter.builder("canal.custom.dml.update")
            .description("UPDATE 事件计数")
            .tag("destination", "example")
            .register(registry);
            
        this.dmlDeleteCounter = Counter.builder("canal.custom.dml.delete")
            .description("DELETE 事件计数")
            .tag("destination", "example")
            .register(registry);
            
        this.processTimer = Timer.builder("canal.custom.process.time")
            .description("处理延迟")
            .register(registry);
    }
    
    /**
     * 在处理每个 Entry 后调用
     */
    public void recordEvent(CanalEntry.Entry entry) {
        EventType eventType = entry.getHeader().getEventType();
        
        switch (eventType) {
            case INSERT:
                dmlInsertCounter.increment();
                break;
            case UPDATE:
                dmlUpdateCounter.increment();
                break;
            case DELETE:
                dmlDeleteCounter.increment();
                break;
            default:
                ddlCounter.increment();
                break;
        }
    }
    
    /**
     * 记录处理时间
     */
    public <T> T recordProcessTime(Supplier<T> task) {
        return processTimer.record(task);
    }
}
```

## 6.2 Java Agent 埋点

```bash
# 启动 Canal 时加载 Java Agent
java -javaagent:/path/to/jmx_prometheus_javaagent-0.19.0.jar=11113:/path/to/config.yaml \
     -jar canal-server.jar

# config.yaml 配置额外的 JVM 指标
rules:
  - pattern: "java.lang.*"
  - pattern: "java.nio.*"
```

# 7. 日志监控

## 7.1 关键日志事件

```java
public class CanalLogMonitor {
    
    // 需要监控的关键日志事件模式
    public static final String[] KEY_PATTERNS = {
        "connection lost",           // MySQL 连接断开
        "dump error",               // Binlog dump 失败
        "ring buffer full",         // RingBuffer 满
        "sink blocked",             // Sink 阻塞
        "ack failed",               // ACK 失败
        "position reset",           // 位点重置
        "HA switch",                // HA 切换
        "OutOfMemoryError",         // OOM
        "parse error",              // 解析失败
    };
    
    /**
     * 扫描日志中的关键事件
     */
    public List<LogEvent> scanForAnomalies(String logFile) {
        List<LogEvent> events = new ArrayList<>();
        
        try (BufferedReader reader = new BufferedReader(new FileReader(logFile))) {
            String line;
            while ((line = reader.readLine()) != null) {
                for (String pattern : KEY_PATTERNS) {
                    if (line.contains(pattern)) {
                        events.add(new LogEvent(pattern, line));
                        break;
                    }
                }
            }
        }
        
        return events;
    }
}
```

## 7.2 日志聚合（ELK）

```
Canal Server 日志 ──> Filebeat ──> Logstash ──> Elasticsearch ──> Kibana
```

```yaml
# filebeat.yml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /opt/canal/logs/canal/canal.log
      - /opt/canal/logs/example/example.log
    multiline.pattern: '^\d{4}-\d{2}-\d{2}'
    multiline.negate: true
    multiline.match: after
    fields:
      app: canal
      instance: example
```

# 8. 健康检查

## 8.1 Canal 健康检查接口

```java
@RestController
@RequestMapping("/health")
public class CanalHealthController {
    
    @Autowired
    private CanalServerWithEmbedded canalServer;
    
    @GetMapping
    public ResponseEntity<Map<String, Object>> health() {
        Map<String, Object> result = new HashMap<>();
        
        // 1. Server 状态
        result.put("server", canalServer.isStart());
        
        // 2. Instance 状态
        Map<String, Boolean> instances = new HashMap<>();
        for (String dest : canalServer.getInstances().keySet()) {
            try {
                CanalInstance instance = canalServer.getCanalInstance(dest);
                instances.put(dest, instance.getEventParser().isStart());
            } catch (Exception e) {
                instances.put(dest, false);
            }
        }
        result.put("instances", instances);
        
        // 3. 延迟检测
        boolean healthy = result.get("server").equals(true) 
            && !instances.containsValue(false);
        
        return ResponseEntity.status(healthy ? 200 : 503).body(result);
    }
}
```

## 8.2 Kubernetes 健康检查

```yaml
# deployment.yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8089
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /health
    port: 8089
  initialDelaySeconds: 10
  periodSeconds: 5
```

# 9. 钉钉/企业微信告警

```java
@Component
public class DingTalkAlarmHandler {
    
    private static final String WEBHOOK_URL = 
        "https://oapi.dingtalk.com/robot/send?access_token=xxx";
    
    /**
     * 发送告警到钉钉
     */
    public void sendAlarm(String title, String content, AlarmLevel level) {
        
        Map<String, Object> message = new HashMap<>();
        
        // Markdown 格式
        message.put("msgtype", "markdown");
        
        Map<String, String> markdown = new HashMap<>();
        markdown.put("title", "Canal 告警: " + title);
        
        String levelEmoji = level == AlarmLevel.CRITICAL ? "🔴" : "⚠️";
        markdown.put("text", String.format(
            "### %s Canal 告警\n" +
            "**告警级别**: %s\n" +
            "**告警内容**: %s\n" +
            "**发生时间**: %s\n" +
            "**Region**: 生产环境\n",
            levelEmoji, level, content, 
            LocalDateTime.now().format(DateTimeFormatter.ISO_LOCAL_DATE_TIME)
        ));
        
        message.put("markdown", markdown);
        
        // 如果是严重告警，添加 @所有人
        if (level == AlarmLevel.CRITICAL) {
            Map<String, Object> at = new HashMap<>();
            at.put("isAtAll", true);
            message.put("at", at);
        }
        
        // 发送 HTTP 请求
        String response = HttpClient.postJson(WEBHOOK_URL, message);
    }
    
    public enum AlarmLevel {
        WARNING,
        CRITICAL
    }
}
```
