
# 1. 概述

ShardingSphere 提供 Agent 插件体系，支持 Prometheus + OpenTelemetry + 日志的三位一体可观测性。

```
┌──────────────────────────────────────────────┐
│           ShardingSphere Agent                │
│                                               │
│  ┌──────────┐ ┌──────────┐ ┌──────────────┐  │
│  │ Metrics  │ │ Tracing  │ │   Logging    │  │
│  │(Prometh) │ │ (OTLP)   │ │  (File/ELK)  │  │
│  └──────────┘ └──────────┘ └──────────────┘  │
└──────────────────────────────────────────────┘
```

# 2. Agent 插件体系

```yaml
# agent.yaml
plugins:
  metrics:
    type: Prometheus
    props:
      host: 0.0.0.0
      port: 9090
  tracing:
    type: OpenTelemetry
    props:
      otlp-traces-exporter: otlp
      otlp-traces-endpoint: http://collector:4317
  logging:
    type: File
    props:
      log-file: /var/log/shardingsphere/sql.log
```

# 3. Prometheus Metrics

## 3.1 内置指标

```prometheus
# 请求总数
shardingsphere_proxy_requests_total{type="select"}

# 请求延迟
shardingsphere_proxy_requests_latency_millis{quantile="0.99"}

# 当前连接数
shardingsphere_proxy_current_connections

# SQL 解析错误
shardingsphere_sql_parsing_error_total

# 事务状态
shardingsphere_transaction_total{status="committed"}
shardingsphere_transaction_total{status="rolled_back"}
```

## 3.2 Grafana Dashboard

推荐面板：

```
1. QPS（按 SQL 类型） — 折线图
2. P50/P99 延迟 — 折线图
3. 连接数（前端/后端） — 折线图
4. 错误率 — 折线图
5. 事务提交/回滚比 — 折线图
6. 各数据源状态 — 状态面板
```

# 4. OpenTelemetry Tracing

## 4.1 自动埋点

ShardingSphere Agent 自动在五个处理阶段创建 Span：

```
Root Span: ShardingSphere-JDBC
  ├── Span: SQL Parse
  ├── Span: SQL Rewrite
  ├── Span: SQL Route
  ├── Span: SQL Execute
  │     ├── Span: ds0.t_order_0
  │     └── Span: ds1.t_order_0
  └── Span: Result Merge
```

## 4.2 集成 Jaeger

```yaml
plugins:
  tracing:
    type: OpenTelemetry
    props:
      otlp-traces-exporter: jaeger
      otlp-traces-endpoint: http://jaeger:14250
```

# 5. SQL 日志

```yaml
props:
  sql-show: true           # 打印 SQL（开发调试）
  sql-simple: true         # 简单模式（不打印参数值）
```

```log
# 日志输出
[INFO] Logic SQL: SELECT * FROM t_order WHERE order_id = ?
[INFO] Actual SQL: ds0 :: SELECT * FROM t_order_0 WHERE order_id = ? ::: [1]
[INFO] Actual SQL: ds0 :: SELECT * FROM t_order_1 WHERE order_id = ? ::: [1]
```

# 6. 慢 SQL 监控

```yaml
props:
  sql-show: true
  max-connections-size-per-query: 1
  slow-sql-millis-threshold: 1000    # 超过 1s 记录为慢 SQL
```

# 7. 健康检查

```bash
# Proxy 健康检查
curl http://localhost:3307/health
# → {"status":"UP"}
```

# 8. 最佳实践

1. **Prometheus + Grafana**：生产环境标配
2. **Tracing 选 OpenTelemetry**：业界标准，无关厂商
3. **SQL 日志分级**：开发 FULL，生产仅慢 SQL
4. **告警规则**：连接数 > 80%、错误率 > 1%、P99 > 1s
