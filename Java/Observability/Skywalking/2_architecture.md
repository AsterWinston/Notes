
# 1. 整体架构

SkyWalking 采用**三层架构**：探针层 → 分析平台 → 存储/UI。

```
┌──────────────────────────────────────────────────────────────┐
│                        Probe Layer                            
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │  Java    │ │  .NET    │ │  Go      │ │  eBPF    │       │
│  │  Agent   │ │  Agent   │ │  Agent   │ │  Agent   │       │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘       │
│       │            │            │            │               │
├───────┼────────────┼────────────┼────────────┼───────────────┤
│       └────────────┴─────────┬──┴────────────┘  gRPC/HTTP    │
│                              ▼                                │
│  ┌───────────────────────────────────────────────────────┐   │
│  │                   OAP Server                           │   │
│  │                                                        │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │   │
│  │  │ Receiver │→│ Analyzer │→│ Aggregator│→│ Exporter │ │   │
│  │  │  (接收)  │ │  (分析)  │ │  (聚合)   │ │  (导出)  │ │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └────┬─────┘ │   │
│  │                                              │        │   │
│  │  ┌──────────┐ ┌──────────┐   ┌───────────────▼─────┐ │   │
│  │  │   Core   │ │   OAL    │   │     Storage          │ │   │
│  │  │  (核心)  │ │  Engine  │   │ (ES/MySQL/BanyanDB)  │ │   │
│  │  └──────────┘ └──────────┘   └─────────────────────┘ │   │
│  └───────────────────────────────────────────────────────┘   │
│                              │                                │
├──────────────────────────────┼────────────────────────────────┤
│                              ▼                                │
│  ┌───────────────────────────────────────────────────────┐   │
│  │                   UI Layer                              │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐               │   │
│  │  │Dashboard │ │ Topology │ │  Trace   │               │   │
│  │  │  (仪表盘) │ │  (拓扑)  │ │  (追踪)  │               │   │
│  │  └──────────┘ └──────────┘ └──────────┘               │   │
│  └───────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

# 2. OAP Server 内部管道

```
Agent (gRPC) 
  │ TraceSegment / Meter
  ▼
┌────────────────┐
│   Receiver      │  ← 接收 gRPC 流，反序列化
│  (GRPCServer)   │
└───────┬────────┘
        │ TraceSegment
        ▼
┌────────────────┐
│   Analyzer      │  ← 解析 Segment → 构建 Trace
│  (SegmentParser)│     端点/服务/实例识别
└───────┬────────┘
        │ Metrics
        ▼
┌────────────────┐
│   Aggregator    │  ← L1 聚合（内存中秒级）
│  (OAL Engine)   │     L2 聚合（分钟级）
└───────┬────────┘
        │ Aggregated Metrics
        ▼
┌────────────────┐
│   Exporter      │  ← 写入存储后端
│  (StorageDAO)   │     ES/MySQL/BanyanDB
└────────────────┘
```

# 3. 数据模型

## 3.1 Trace — 一次完整的请求调用链

```
Trace (全局唯一 TraceId)
  ├── Segment 1 (Service A — entry)
  │     ├── Span: /api/user (Entry)
  │     │     └── Span: UserService.getUser (Local)
  │     │           └── Span: jdbc/select (Exit)
  │     └── Span: Dubbo/AccountService.debit (Exit)
  │           │
  │           ▼ (跨进程传播)
  └── Segment 2 (Service B — Dubbo Provider)
        └── Span: Dubbo/AccountService.debit (Entry)
              └── Span: jdbc/update (Exit)
```

## 3.2 Span 类型

| 类型 | 含义 | 示例 |
|------|------|------|
| **Entry** | 入口 Span，服务接收外部请求 | HTTP Server / Dubbo Provider |
| **Exit** | 出口 Span，服务发起外部调用 | HTTP Client / Dubbo Consumer / JDBC |
| **Local** | 本地 Span，方法级跟踪 | 业务方法、内部逻辑 |

## 3.3 Segment — 单个服务内的 Trace 片段

```java
// Segment 是 JVM 进程内的 Trace 片段
// 一个 Trace 由多个 Segment 组成（每个服务一个 Segment）
// Segment 包含多个 Span
// Segment 之间通过 TraceId + SegmentId 关联
```

# 4. 通信协议

```
Agent ──gRPC Streaming──→ OAP Server
  ├── TraceSegmentReportService (追踪数据)
  ├── JVMMetricReportService (JVM 指标)
  ├── MeterReportService (自定义指标)
  └── ProfileTaskQueryService (性能剖析)

OAP Server ←──GraphQL── UI Dashboard
  ├── 查询 Trace / Metrics / Topology
  └── 订阅告警事件
```

**为什么选 gRPC？**
- 双向流：Agent 可以持续推送数据，OAP 可以下发指令（如 Profiling 任务）
- 高性能：Protobuf 序列化 + HTTP/2 多路复用
- 跨语言：多语言 Agent 共享同一协议

# 5. 服务层级（Layer）

SkyWalking 9.x 引入 Layer 概念，按层次组织服务：

| Layer | 说明 | 示例 |
|-------|------|------|
| `OS_LINUX` | 操作系统层 | — |
| `K8S` | Kubernetes 层 | k8s-service |
| `MESH` | Service Mesh 层 | Istio Envoy |
| `GENERAL` | 通用服务 | Spring Boot 应用 |
| `FAAS` | 函数即服务 | AWS Lambda |
| `DATABASE` | 数据库 | MySQL/PG |
| `CACHE` | 缓存 | Redis |

# 6. OAP 集群

```
                    Load Balancer
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
        ┌──────────┐┌──────────┐┌──────────┐
        │ OAP-1    ││ OAP-2    ││ OAP-3    │  ← 无状态，水平扩展
        │(Receiver) ││(Receiver)││(Receiver)│
        └────┬─────┘└────┬─────┘└────┬─────┘
             │           │           │
             └───────────┼───────────┘
                         ▼
              ┌──────────────────────┐
              │    Storage Cluster   │  ← 共享存储
              └──────────────────────┘
```

OAP 无状态设计 → 可水平扩展，仅受存储后端写入能力限制。
