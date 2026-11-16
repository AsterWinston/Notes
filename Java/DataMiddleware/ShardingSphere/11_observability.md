可观测性回答三个问题：**系统现在健康吗（Metrics）、一条慢请求卡在哪（Tracing、出错的 SQL 是什么（Logging）**。ShardingSphere 5.5.x 的可观测能力主要通过 **ShardingSphere-Agent**（Java Agent，字节码增强）落地，Proxy 侧也内置了一套 Prometheus 指标。这一篇讲三支柱各自的落地方式、如何接 Prometheus/Grafana、如何接 OpenTelemetry/SkyWalking 链路、以及最实用的"慢 SQL / 错误 SQL 观测"。先纠正一处常见误解：**Proxy 默认没有 `/health` HTTP 端口**，健康检查靠连上端口执行 `SELECT 1` 或看指标端点，别照抄带 `/health` 的过时示例。

# 1. 观测架构

```text
┌───────────────────────────────────────────────────────────┐
│              ShardingSphere-Agent (premain 增强)           │
│  ┌────────────┐   ┌──────────────┐   ┌────────────────┐   │
│  │  Metrics   │   │   Tracing    │   │    Logging     │   │
│  │ Prometheus │   │ OpenTelemetry│   │  File/Console  │   │
│  │  exporter  │   │   OTLP span  │   │                │   │
│  └─────┬──────┘   └──────┬───────┘   └───────┬────────┘   │
└────────┼─────────────────┼───────────────────┼────────────┘
         ▼                 ▼                   ▼
   Prometheus ──► Grafana  Jaeger/SkyWalking   sql.log
                                        告警       链路查询      慢 SQL 复盘
```

三种接入形态各有取指标方式：**Proxy 内置**（5.5.x 起直接暴露 Prometheus 端点，最省事）、**Agent 增强 JDBC**（给 Spring Boot 应用挂 `-javaagent`）、**Agent 增强 Proxy**（更细的方法级埋点）。埋点是旁路的——不改变 SQL 执行逻辑，采集失败不应影响主链路，这也是选 Agent 而非改内核的核心理由。

# 2. Agent 配置

Agent 用独立 `conf/agent.yaml`（与业务 YAML 分离），三大插件各自开关：

```yaml
# shardingphere-agent/conf/agent.yaml
plugins:
  metrics:
    Prometheus:
      host: "0.0.0.0"
      port: 9091
      props:
        jvm-information-collector-enabled: "true"   # 附带 JVM 指标
  tracing:
    OpenTelemetry:
      props:
        otel.service.name: "shardingsphere-proxy"
        otel.traces.exporter: "otlp"
        otel.exporter.otlp.traces.endpoint: "http://otel-collector:4317"
        otel.traces.sampler: "always_on"
  logging:
    File:
      props:
        level: "INFO"
```

启用方式按形态不同：

```bash
# JDBC / Spring Boot：挂 javaagent
java -javaagent:/path/shardingsphere-agent.jar -jar app.jar

# Proxy：用 -g 参数加载 Agent（或环境变量）
./bin/start.sh -g

# Docker Sidecar/嵌入
COPY --from=ghcr.io/apache/shardingsphere-agent:latest /usr/agent/ /shardingsphere-agent/
ENTRYPOINT ["java","-javaagent:/shardingsphere-agent/shardingsphere-agent.jar","-jar","/app.jar"]
```

> **注意**：Agent 与 SkyWalking/其他 javaagent 同时挂载时，增强顺序可能冲突（都会插桩 JDBC/网络层），要在一方排掉重复埋点；生产 tracing 用 `always_on` 采样开销大，高 QPS 场景把 `otel.traces.sampler` 降为比例采样（如 `parentbased_traceidratio`）。

# 3. Prometheus Metrics

## 3.1 指标清单

5.5.x Proxy/JDBC 内置的关键指标（名称以前缀区分形态）：

| 指标 | 类型 | 含义 |
|------|------|------|
| `proxy_requests_total` | COUNTER | 请求总数（按 SQL 类型 INSERT/UPDATE/DELETE/SELECT/DDL… 分维度） |
| `parsed_sql_total` | COUNTER | 解析计数，按语句类型 |
| `routed_sql_total` | COUNTER | 路由计数，判断是否全路由 |
| `routed_result_total` | COUNTER | 命中数据源数/表数 |
| `proxy_current_connections` | GAUGE | 当前前端连接数 |
| `proxy_execute_latency_millis` | HISTOGRAM | 执行延迟直方图（算 P99） |
| `proxy_execute_errors_total` | COUNTER | 执行错误数 |
| `proxy_transactions_total` | COUNTER | 事务数，按 commit/rollback |
| `proxy_state` | GAUGE | 0=OK、1=熔断、2=锁定 |
| `proxy_meta_data_info` | GAUGE | 逻辑库数、存储单元数 |

JDBC 形态对应 `jdbc_state`、`jdbc_statement_execute_total`、`jdbc_statement_execute_latency_millis`、`jdbc_transactions_total` 等。抓取端点形如：

```bash
curl http://localhost:9091/metrics | grep proxy_execute_latency
# proxy_execute_latency_millis_bucket{le="1",...} 30
# proxy_execute_latency_millis_count 4200
```

## 3.2 Grafana 面板建议

```text
1. QPS（按 SQL 类型）           — 折线，requests_total rate
2. P50/P95/P99 延迟             — 直方图 histogram_quantile
3. 连接数（前端 vs 后端池）      — 折线，容量告警
4. 错误率                       — errors_total / requests_total
5. 事务 commit/rollback 比       — 回滚率突增预警
6. proxy_state                  — 熔断/锁定状态置面板
7. 全路由占比                    — routed_result 判断扫描宽度
```

> **行业实践**：最有价值的分片专项指标是 **`routed_result_total` / `routed_sql_total` 比值**——它反映"平均每条查询扫了多少分片"，比值持续爬升说明业务出现了越来越多全路由 SQL（往往是新接口漏带分片键），是比延迟更早的预警信号。

# 4. OpenTelemetry Tracing

Agent 在 SQL 处理各阶段自动建 Span，一条跨分片查询的链路形态：

```text
Root Span: SQL Route
  ├── Span: SQL Parse
  ├── Span: SQL Rewrite
  ├── Span: SQL Route
  ├── Span: SQL Execution
  │     ├── ds0.t_order_0   (子 span，含物理 SQL 与耗时)
  │     └── ds1.t_order_1
  └── Span: Result Merge
```

链路能直观暴露"慢在哪个分片"：某个后端 Span 特别长 = 该分片是瓶颈（数据倾斜/缺索引）。OTLP 可直接送 Jaeger，也可送 SkyWalking 的 OTEL Receiver：

```yaml
tracing:
  OpenTelemetry:
    props:
      otel.traces.exporter: "otlp"
      # 送 Jaeger（OTLP/gRPC）
      otel.exporter.otlp.traces.endpoint: "http://jaeger:4317"
```

> **设计考量**：
>
> - **SkyWalking 集成**有两条路：一是继续用 SkyWalking 自带 javaagent（ShardingSphere 早期有 apm-toolkit 埋点，现统一走 OTel），二是让 ShardingSphere Agent 用 OTLP 输出、SkyWalking OAP 开 OTEL 接收端口消费。混用两套 tracer 会导致 traceId 不通，选其一贯通全链路。
> - **链路只在跨分片查询、XA 事务这类"多段执行"场景最有价值**；单分片简单查询的 span 开销大于收益，采样率按接口分级设置。

# 5. SQL 日志

```yaml
props:
  sql-show: true      # 打印 Logic SQL + Actual SQL + 参数（开发调试）
  sql-simple: true    # 只打印原始逻辑 SQL，不打参数与物理 SQL
```

```text
# sql-show 输出样例
Logic SQL: SELECT * FROM t_order WHERE order_id = ?
Actual SQL: ds1 ::: SELECT * FROM t_order_1 WHERE order_id = ? ::: [8888888888]
```

- `sql-show` 走 INFO 日志，**生产高并发下日志量巨大**，要么关、要么 `sql-simple`、要么单独路由到 sql.log 并配滚动策略。
- 想抓"参数 + 物理 SQL"又不想全开，用 Agent 的 logging 插件按级别/文件隔离，比全局 `sql-show` 可控。

# 6. 慢 SQL / 错误观测

5.5.x **没有** 独立 `slow-sql-millis-threshold` 这类阈值开关（网上旧例常见此臆造键）。慢 SQL 观测靠三条真实途径组合：

1. **Prometheus 延迟直方图**：`proxy_execute_latency_millis` 的 P99 报警，定位到"哪个时间点变慢"。
2. **OpenTelemetry Trace**：抓一条慢请求，看是哪个分片 Span 长——是数据问题还是全路由问题。
3. **后端数据库慢日志**：ShardingSphere 是"转发层"，单条物理 SQL 慢要在 **MySQL `slow_query_log`**（`long_query_time`）抓，因为真正执行在分片库。

```sql
-- 变更前后自查路由宽度（最便宜的"慢 SQL 预防"）
PREVIEW SELECT * FROM t_order WHERE order_id = 8888888888;
```

> **注意**：分片环境下"MySQL 慢查询日志"分散在每个分库，排查要遍历所有分片；更实用的做法是把 `proxy_execute_latency_millis` + `routed_result_total` 接进告警，先判断是"某分片慢"还是"扫的分片多"，再决定去哪个库捞慢日志。

# 7. 健康检查与就绪

```bash
# Proxy 存活：用 MySQL 客户端做轻量探活（无独立 /health 端口）
mysql -h 127.0.0.1 -P 3307 -u root -p -e "SELECT 1;"

# 节点状态：Cluster 下查注册中心里的计算节点
#   SHOW COMPUTE NODES → status=OK / mode_type=Cluster
# 指标端点：Agent/Proxy 的 prometheus 端口
curl -s http://localhost:9091/metrics | grep proxy_state
```

K8s 就绪探针用 TCP 3307 + 一条 `SELECT 1`，别依赖不存在的 HTTP 健康路径。`proxy_state` 变 1（熔断）或 2（锁定）可作为告警触发条件，联动 10_governance 的 `UNLOCK CLUSTER`/从库恢复流程。

# 8. 最佳实践

1. **生产标配 Prometheus + Grafana**：延迟直方图、连接数、错误率、`proxy_state` 四件套。
2. **Tracing 用 OpenTelemetry**：厂商中立，Jaeger/SkyWalking 都能接；跨分片链路才开全采样。
3. **SQL 日志分级**：开发 `sql-show`，生产关或 `sql-simple` + 独立 sql.log 滚动。
4. **慢 SQL 三方联查**：直方图报警 → Trace 定位分片 → 后端慢日志取证，缺一环都查不透。
5. **告警阈值**：连接数 > 池 80%、错误率 > 1%、P99 > 1s、`proxy_state != 0`、全路由占比突增。

# 9. 小结

| 支柱 | 落地 | 关键看点 |
|------|------|---------|
| Metrics | Agent Prometheus 或 Proxy 内置端点 | requests/latency 直方图/connections/state |
| Tracing | Agent OpenTelemetry → Jaeger/SkyWalking | 各处理阶段 Span，跨分片定位慢片 |
| Logging | `sql-show`/`sql-simple` + Agent File 插件 | Logic SQL → Actual SQL 映射 |
| 慢 SQL | 直方图报警 + Trace + MySQL 慢日志（无内置阈值键） | 先判"慢片"还是"扫片多" |
| 健康 | TCP 3307 + `SELECT 1` + `proxy_state`（无 /health） | 就绪探针用轻量查询 |

至此 ShardingSphere 主篇（1–11）结束：从定位、架构、快速上手到六大功能与治理/观测。配套的 Proxy vs JDBC 细节、Spring 集成、多库支持、源码解析与面试问答见本目录 12–21 篇。
