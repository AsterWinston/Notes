
# 1. 主流 APM 全景

| 维度 | SkyWalking | Jaeger | Zipkin | Pinpoint | Datadog |
|------|-----------|--------|--------|----------|---------|
| **开源** | ✓ Apache | ✓ CNCF | ✓ | ✓ | ✗ 商业 |
| **协议** | gRPC | UDP/Thrift | HTTP/Thrift | Thrift | Agent |
| **存储** | ES/MySQL/BanyanDB | ES/Cassandra | ES/MySQL | HBase | 自建 |
| **追踪** | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Metrics** | ✓ 内置 OAL | ✗ | ✗ | ✓ | ✓ |
| **告警** | ✓ | ✗ | ✗ | ✓ | ✓ |
| **日志关联** | ✓ | ✗ | ✗ | ✗ | ✓ |
| **代码级诊断** | ✓ Profiling | ✗ | ✗ | ✓（强） | ✓ |
| **服务拓扑** | ✓ | ✓ | 有限 | ✓ | ✓ |
| **多语言** | Java/.NET/Go/Python... | 多语言 | 多语言 | 仅 Java | 多语言 |
| **性能开销** | 低(3-8%) | 低 | 中 | 低 | 中 |

# 2. 场景推荐

## 2.1 纯 Java 微服务 → SkyWalking / Pinpoint

```
SkyWalking 优势：
  - Apache 顶级项目，社区活跃
  - 内置 Metrics + 告警 + Profiling
  - 无侵入 Java Agent
  - 支持 Spring Cloud/Dubbo 生态

Pinpoint 优势：
  - 代码级诊断（方法参数、返回值查看）
  - 字节码增强更彻底
  - 缺点：仅支持 Java，依赖 HBase
```

## 2.2 多语言混合 → SkyWalking / Jaeger

```
SkyWalking 优势：
  - Java/.NET/Go/Python/PHP/Node.js 等 Agent 覆盖
  - 统一协议，数据格式统一

Jaeger 优势：
  - CNCF 毕业项目，K8s 生态好
  - 多语言 SDK 丰富
  - 缺点：无 Metrics 和告警
```

## 2.3 K8s + Service Mesh → Istio + SkyWalking

```
Istio Envoy 的 ALS → SkyWalking OAP
  优点：零注入（Sidecar 层面采集）
  缺点：未覆盖应用内部数据库/Redis 调用（需补充 Agent）
```

## 2.4 商业 SaaS → Datadog

```
优点：开箱即用，Dashboard 丰富，AI 分析
缺点：按数据量计费，大规模下成本高
```

# 3. 选型矩阵

```
团队技术栈?
  ├── 纯 Java → SkyWalking / Pinpoint
  ├── Java + Go → SkyWalking
  └── 多语言混合 → SkyWalking / Jaeger

是否需要 Metrics + 告警?
  ├── 是 → SkyWalking（内置）
  └── 否 → Jaeger / Zipkin

是否 K8s + Service Mesh?
  ├── 是 → SkyWalking（ALS 集成）
  └── 否 → 任一均可

运维能力?
  ├── 有 ES 经验 → SkyWalking
  ├── 有 HBase 经验 → Pinpoint
  └── 有 Cassandra 经验 → Jaeger

预算?
  ├── 无 → SkyWalking（开源免费）
  └── 充足 → Datadog（商业）
```

# 4. 从 Zipkin/Jaeger 迁移

```
1. SkyWalking OAP 配置 Zipkin Receiver
   receiver-zipkin:
     default:
       host: 0.0.0.0
       port: 9411

2. 应用逐步切换 sw8 Header 传播
3. 保留 Zipkin/Jaeger 数据直到 TTL 过期
4. 完全切换后停用旧系统
```
