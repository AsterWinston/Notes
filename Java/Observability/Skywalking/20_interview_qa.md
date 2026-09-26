
# 1. 基础题

## 1.1 什么是 SkyWalking？核心架构是什么？

Apache SkyWalking 是开源 APM 系统，提供分布式追踪、指标聚合和告警。核心架构：**Agent（探针）→ OAP Server（分析平台）→ Storage（存储）→ UI（可视化）**。

## 1.2 APM 的三大支柱是什么？

| 支柱 | 说明 | SkyWalking 实现 |
|------|------|----------------|
| Tracing | 分布式调用链路追踪 | Java Agent 字节码增强 |
| Metrics | 系统指标聚合 | OAL 引擎定义 + OAP 聚合 |
| Logging | 日志关联追踪 | TraceId 注入 MDC |

## 1.3 SkyWalking vs Zipkin / Jaeger？

SkyWalking 内置 Metrics + 告警 + Profiling + 日志关联，是**一体化 APM**。Zipkin/Jaeger 仅做分布式追踪，需要配合其他工具才能补全 Metrics 和告警能力。

# 2. 原理题

## 2.1 SkyWalking Java Agent 是如何工作的？

使用 `java.lang.instrument` 的 **premain** 机制，在类加载时通过 ByteBuddy 修改字节码，在目标方法前后插入拦截器代码（before/after/onException），记录 Span 信息。

## 2.2 TraceId 是如何跨服务传播的？

通过 **sw8 协议头**在 HTTP Header 中传播。格式：`1-TRACEID-SEGMENTID-SPANID-SERVICE-INSTANCE-ENDPOINT-PEER`。Dubbo 通过 RpcInvocation.attachment 传播。MQ 通过消息 Header 传播。

## 2.3 Span 的三种类型及区别？

| 类型 | 含义 | 示例 |
|------|------|------|
| Entry | 服务接收外部请求 | HTTP Server / Dubbo Provider |
| Exit | 服务发起外部调用 | HTTP Client / Dubbo Consumer |
| Local | 服务内部方法调用 | Service 方法 |

## 2.4 OAL 引擎是什么？

OAL（Observability Analysis Language）是 SkyWalking 的指标定义 DSL，允许用户用类 SQL 语法定义从 Trace 数据中提取的指标。如 `service_resp_time = from(Service.*).latency()`。

# 3. 实践题

## 3.1 如何排查"慢接口"？

1. **SkyWalking Dashboard** → 找到慢 Endpoint
2. **Trace 列表** → 选中慢 Trace
3. **Span 树** → 找到耗时最长的 Span
4. **Profiling** → 在 Span 上创建 Profiling 任务 → 火焰图定位到方法
5. **关联日志** → 查看该 TraceId 下的日志

## 3.2 端点爆炸如何解决？

```properties
# Agent 配置标准化 API 路径
trace.ignore_path=/actuator/**
plugin.springmvc.urlSuffixRestPath=/api/\w+/\d+
```

## 3.3 如何设置采样策略？

```properties
agent.sample_n_per_3_secs=500     # 高 QPS 场景限制采样
agent.force_tracing=false         # 非 debug 模式不强制全量
```

# 4. 架构题

## 4.1 SkyWalking 如何保证高可用？

- **OAP 无状态**：可水平扩展，前端 LB 分发 Agent 连接
- **存储集群**：ES/BanyanDB 集群保证数据高可用
- **Agent 重连**：OAP 故障时 Agent 自动切到其他节点

## 4.2 OAP 的数据流是怎样的？

```
Agent (gRPC) → Receiver → Analyzer (解析Segment → 构建Trace)
  → Aggregator (L1 秒级 → L2 分钟级聚合)
  → Exporter → Storage (ES/MySQL/BanyanDB)
```

## 4.3 如何降低 SkyWalking 的性能开销？

- 合理设置采样率（`sample_n_per_3_secs`）
- 忽略静态资源和健康检查端点
- 减少不必要的插件加载
- 生产环境关闭 SQL 参数追踪
- OAP 集群水平扩展分摊压力
