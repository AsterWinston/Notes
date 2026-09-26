
# 1. Observability 总览

Kibana Observability 提供统一的**应用性能监控（APM）、基础设施监控和用户监控**界面。

```
Observability 三大支柱:
  ├── APM           → 应用性能（延迟/错误/事务）
  ├── Infrastructure → 主机/容器/K8s 指标
  ├── Uptime        → 服务可用性检测
  └── User Experience → 真实用户监控（RUM）
```

# 2. APM

## 2.1 Service Map

```
APM → Services → 选择 Service → Service Map

自动生成服务依赖拓扑:
  ┌────────┐     ┌────────┐
  │Gateway │────→│Order   │──→ MySQL
  └────────┘     │Service │──→ Redis
                 └────────┘──→ Kafka
```

## 2.2 Transactions — 事务

```
APM → Services → order-service → Transactions

Transaction 列表:
  Name                Avg. Duration  Throughput  Error %
  GET /api/order/{id}    120ms         50 tpm      0.2%
  POST /api/order         85ms         20 tpm      0.0%

点击 Transaction → 查看:
  ├── Timeline (Waterfall): Span 耗时瀑布图
  │     ├── Span: Controller (5ms)
  │     ├── Span: Service (10ms)
  │     │     └── Span: JDBC select (80ms)  ← 瓶颈
  │     └── Span: Redis get (3ms)
  ├── Metadata: HTTP Headers, User Agent
  └── Labels: 自定义标签
```

## 2.3 Errors

```
APM → Services → order-service → Errors

Error 列表:
  Type                  Count  Message
  NullPointerException  12     at com.example.UserService.getUser
  TimeoutException       5     Connection timeout to redis
  
点击 Error → 查看:
  ├── Stack Trace
  ├── Occurrences (发生时间分布)
  ├── Affected Transactions
  └── Context (请求参数 / 用户信息)
```

# 3. Infrastructure

```
Observability → Infrastructure

Hosts:
  ├── CPU Usage (折线图)
  ├── Memory Usage (折线图)
  ├── Disk Usage (折线图)
  └── Network Traffic (折线图)

Containers (K8s):
  ├── Pod CPU / Memory
  ├── Pod Status (Running / Failed)
  └── Pod Logs (关联到 Discover)
```

# 4. Uptime

```
Observability → Uptime

Monitor 列表:
  Name             URL                  Status    P50      P99
  API Health       /api/health          Up        45ms     120ms
  Homepage         /home               Up        230ms    800ms

点击 Monitor → Duration trends → TSL/SSL 证书到期提醒
```

# 5. User Experience (RUM)

```
真实用户监控（需集成 APM RUM Agent）:

Observability → User Experience
  ├── Page Load Duration (P50/P75/P90/P95)
  ├── Core Web Vitals: LCP / FID / CLS
  ├── Breakdown (每个步骤的耗时)
  └── Browser/OS/Geo 分布
```

# 6. SLO 管理

```
Observability → SLO

创建 SLO:
  - Service: order-service
  - Indicator: Latency < 500ms
  - Objective: 99.5% in 30-day rolling window

Dashboard:
  SLI value:
  ┌─────────────────────────────────────────────────┐
  │  ████████████████████████████████████████░░ 99.2% │
  │  ─────────────────────────────────────────────── │
  │         Target: 99.5%         Budget: -0.3%      │
  └─────────────────────────────────────────────────┘
```
