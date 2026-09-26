
# 1. 日志体系

Sentinel 提供多层次的日志记录，用于问题排查和运行监控。

```
${user.home}/logs/csp/
  ├── sentinel-record.log.{yyyy-MM-dd}       # 拦截/流控日志
  ├── {appName}-metrics.log.{yyyy-MM-dd}     # 指标日志
  ├── sentinel-block.log                     # 旧版阻塞日志
  └── sentinel-record.log                    # 旧版记录日志
```

# 2. 拦截日志

## 2.1 日志格式

```
# sentinel-record.log
2024-01-15 10:30:00.123|order-service|getUser|FlowException|default|100|0
  │                      │            │      │           │      │  │
  │                      │            │      │           │      │  └─ 额外信息
  │                      │            │      │           │      └─ 规则阈值
  │                      │            │      │           └─ 来源应用
  │                      │            │      └─ 异常类型
  │                      │            └─ 资源名
  │                      └─ 应用名
  └─ 时间戳
```

## 2.2 日志配置

```yaml
spring:
  cloud:
    sentinel:
      log:
        dir: /var/log/sentinel          # 日志目录
        switch-pid: true                # 是否在文件名中加 PID
```

```properties
# JVM 参数方式
csp.sentinel.log.dir=/var/log/sentinel
csp.sentinel.log.use.pid=true
```

# 3. Metrics 指标日志

## 3.1 日志格式

```
# {appName}-metrics.log
1705302000|1705302001000|getUser|10|5|0|2|2|25|0|0|0
  │         │             │       │  │ │ │ │ │  │ │ │
  │         │             │       │  │ │ │ │ │  │ │ └─ occupied_pass
  │         │             │       │  │ │ │ │ │  │ └─ exception
  │         │             │       │  │ │ │ │ │  └─ block
  │         │             │       │  │ │ │ │ └─ pass
  │         │             │       │  │ │ │ └─ cluster
  │         │             │       │  │ │ └─ 资源名
  │         │             │       │  │ └─ id
  │         │             │       │  └─ minute pass
  │         │             │       └─ rt
  │         │             └─ 资源名
  │         └─ 窗口结束时间
  └─ 窗口开始时间
```

## 3.2 解析 Metrics 日志

```java
// 官方 MetricsReader 解析
public class MetricLogReader {

    public List<MetricNode> readMetrics(String filePath) {
        List<MetricNode> metrics = new ArrayList<>();
        try (BufferedReader reader = new BufferedReader(
                new FileReader(filePath))) {
            String line;
            while ((line = reader.readLine()) != null) {
                String[] parts = line.split("\\|");
                MetricNode node = new MetricNode();
                node.setTimestamp(Long.parseLong(parts[0]));
                node.setPassQps(Long.parseLong(parts[3]));
                node.setBlockQps(Long.parseLong(parts[7]));
                node.setRt(Double.parseDouble(parts[4]));
                metrics.add(node);
            }
        }
        return metrics;
    }
}
```

# 4. Prometheus 集成

## 4.1 依赖

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

## 4.2 暴露 Metrics

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
```

## 4.3 自定义 Sentinel 指标

```java
@Component
public class SentinelMetricsExporter {

    private final MeterRegistry registry;

    public SentinelMetricsExporter(MeterRegistry registry) {
        this.registry = registry;
        initMetrics();
    }

    private void initMetrics() {
        // 注册 Sentinel 指标到 Prometheus
        Gauge.builder("sentinel_pass_qps", () -> 
            ClusterNodeManager.getOriginCountSummary().values()
                .stream().mapToLong(Metric::pass).sum())
            .register(registry);

        Gauge.builder("sentinel_block_qps", () ->
            ClusterNodeManager.getOriginCountSummary().values()
                .stream().mapToLong(Metric::block).sum())
            .register(registry);
    }
}
```

# 5. Grafana 大盘

```prometheus
# PromQL 查询示例

# 每秒通过请求数
rate(sentinel_pass_qps[1m])

# 每秒阻塞请求数
rate(sentinel_block_qps[1m])

# 平均响应时间
rate(sentinel_rt_total[1m]) / rate(sentinel_pass_qps[1m])

# 阻塞率
rate(sentinel_block_qps[1m]) / (rate(sentinel_pass_qps[1m]) + rate(sentinel_block_qps[1m]))
```

# 6. Dashboard 监控面板

Dashboard 提供以下实时监控：

| 面板 | 指标 | 用途 |
|------|------|------|
| 实时监控 | 通过 QPS、拒绝 QPS、RT | 快速发现异常 |
| 簇点链路 | 资源调用链树 | 理解流量链路 |
| 机器列表 | 机器健康状态 | 实例管理 |
| 规则管理 | 规则配置与推送 | 规则运维 |

# 7. 日志级别控制

```java
// 动态调整 Sentinel 日志级别
System.setProperty("csp.sentinel.log.level", "DEBUG");
// 可选: TRACE, DEBUG, INFO, WARN, ERROR
```

# 8. 告警集成

```java
@Component
public class SentinelAlertHandler {

    @PostConstruct
    public void init() {
        // 监听 BlockEvent
        BlockEventHandler.register(new BlockEventHandler() {
            @Override
            public void onEvent(BlockEvent event) {
                if (event.getBlockThreshold() > 100) {
                    // 大流量阻塞告警
                    alertService.send("Sentinel Block Alert",
                        String.format("Resource: %s, QPS: %d, Threshold: %d",
                            event.getResource(),
                            event.getCurrentQps(),
                            event.getBlockThreshold()));
                }
            }
        });
    }
}
```

# 9. 最佳实践

1. **生产日志目录**：指定独立日志目录，避免与业务日志混合
2. **日志清理策略**：配置 `csp.sentinel.metric.file.total.count` 限制文件数量
3. **Prometheus 监控**：生产环境推荐 Prometheus + Grafana 方案
4. **告警阈值**：阻塞 QPS 持续 > 0 超过 5 分钟应触发告警
5. **Dashboard 仅监控**：生产环境使用 Nacos 持久化规则，Dashboard 只用于可视化
