
# 1. 概述

SkyWalking Java Agent 内置 Dubbo 2.7.x/3.x 插件，自动为 Dubbo RPC 调用创建 Span 并完成跨进程上下文透传。RPC 栈的可观测性与 HTTP 栈同构，只是**载体从 HTTP Header 换成了 Dubbo 的 attachment**：

```text
Consumer                                       Provider
    │ 创建 Exit Span                              │
    │   operation: {interface}:{method}           │
    │   peer: {provider-ip}:{port}                │
    │                                            │
    │ 将 sw8 序列化写入 RpcInvocation.attachment   │
    │ ────────────  Dubbo 请求 ───────────────→   │ 从 attachment 提取 sw8
    │                                            │ 创建 Entry Span（新 Segment，同 TraceId）
    │ ←──────────  Response ──────────────────   │ 结束 Entry Span
    │ 结束 Exit Span                              │
    │                                            │
    └── Segment 之间用 ref 指向父 Segment ─────────┘
```

**关键实现路径**：skywalking-java 的 dubbo 插件不是靠注册 Dubbo SPI Filter，而是**直接增强 Invoker 层**——consumer 侧匹配 `DubboInvoker#doInvoke`（真正发出网络请求的位置），provider 侧匹配 `AbstractProxyInvoker#invoke`（业务实现被调用前的最后统一入口）。这样做的收益在下文第 4 节展开。

# 2. 为什么增强 Invoker 而不是写一个 Filter SPI

这是 Dubbo 集成里最值得理解的一次**设计取舍**。社区自研 trace 常走 `@SPI` 注册自定义 Filter（如 `TraceFilter`），SkyWalking 官方插件却选了字节码增强：

| 维度 | Filter SPI（@SPI 注册） | Invoker 字节码增强（官方做法） |
|------|------------------------|------------------------------|
| 接入成本 | 需在 `dubbo.properties`/`@Activate` 配置生效链 | 挂 agent 即生效，零配置 |
| 顺序可控性 | Filter 链顺序受 order/@Order 影响，易被业务 filter 覆盖 attachment | 位于链路最外层，先于所有 Filter |
| 侵入用户代码 | 需打包 SPI 扩展 jar 进应用 | 完全不进应用依赖 |
| 版本适配 | Filter API 相对稳定 | 需为 2.5/2.7/3.x 各写 instrumentation |
| 拿到的信息 | attachment、invocation | 同上 + 真实对端地址（invoker.getUrl()） |

**核心考量是顺序与可靠性**：sw8 必须"最先写入、最后不被覆盖"。Filter 链里任何业务 filter 重设 attachment 或提前 return（例如鉴权失败短路），链路就可能断裂或串味；增强 Invoker 相当于把自己放在 Dubbo 抽象的最外圈，无论 Filter 链怎么折腾，进出都绕不过它。代价是版本适配矩阵必须维护得很细——这也解释了"某插件只对 Dubbo ≥ 2.7.4 生效"这类限制声明从何而来。

# 3. 上下文透传：attachment 里的 sw8

Dubbo 的 `RpcInvocation#attachments` 是 Map<String,String>，SkyWalking 复用它携带序列化的 `ContextCarrier`：

```java
// Consumer 侧拦截器核心逻辑（简化）
public void beforeMethod(...) {
    Invocation invocation = (Invocation) allArguments[0];
    // 1. 以接口方法名创建 Exit Span
    AbstractSpan span = ContextManager.createExitSpan(
        "Dubbo/" + invocation.getMethodName(),
        new DubboRpcContextTransporting(invocation));   // 把 attachment 当作 Header 载体
    // 2. 注入 sw8：TraceId / SegmentId / SpanId / Service / Instance / Endpoint
    ContextManager.inject(span);
    span.setComponent(ComponentsDefine.DUBBO);
    span.tag(PEER, invoker.getUrl().getAddress());       // provider 地址
}

// Provider 侧：从 attachment 提取，恢复父上下文
ContextCarrier carrier = ContextManager.extract(attachment ->
    attachment.get(HeaderNames.sw8));
AbstractSpan span = ContextManager.createEntrySpan(operationName, carrier /* sw8 */);
```

写入的实际键与 HTTP 一致——`sw8`（链路主数据）、`sw8-correlation`（业务透传数据）、`sw8-x`（扩展示意）。所以 Dubbo 场景排查丢链，思路和 HTTP 一样：**抓一次请求看 attachment 里有没有 sw8、值是否完整**。

> **注意**：sw8 值里含 `-`、`.` 分隔的多段字符串，如果 provider 侧有 Filter 对 attachment 做了 key 校验/白名单过滤（安全加固常见做法），`sw8` 会被静默丢弃，表现就是 **provider 侧开了一条全新 Trace**。把 `sw8*` 前缀加入 attachment 白名单是必配项。

# 4. Span 生成与标签

| 位置 | Span 类型 | Layer | Component |
|------|----------|-------|-----------|
| Consumer 发起 | Exit | RPC | `Dubbo` |
| Provider 接收 | Entry | RPC | `Dubbo` |
| Provider 业务实现内部 | Local（需注解/插件） | — | — |

operationName 形如 `DubboProvider/{interface}.{method}`（consumer 侧为 `Dubbo/{interface}.{method}`，具体前缀随插件版本微调），provider 端点名因此和接口签名强绑定，**接口方法重载会导致端点合并**，需要靠参数类型或 `@Trace` 重命名区分。

自动标签：

| 标签 | 说明 |
|------|------|
| `rpc.provider` / peer | 对端 `ip:port`（拓扑边的依据） |
| `dubbo.interface` | 接口全限定名 |
| `dubbo.method` | 方法名 |
| `dubbo.parameters` | RPC 参数（需开启，生产慎用） |

**拓扑指标的来源**：client 与 server 两侧的 Exit/Entry Span 各带一次记录，OAP 据此生成 `ServiceRelation` 的 `detectPoint=CLIENT/SERVER` 双视角指标——`service_relation_server_resp_time` 是 provider 视角耗时，`client` 视角多了一跳网络与序列化，两者差值就是 RPC 框架自身+网络的开销，这是判断"是服务慢还是网络慢"的标准手法。

# 5. 集群路由标签（zone / cluster）

Dubbo 的多集群、多地域部署需要把"这次调用打到了哪个 zone"表达出来。SkyWalking 的做法不是给每个 zone 建独立服务，而是**实例属性 + 标签**：

```properties
# agent.config — 实例级属性（进入 Instance Properties，UI 可按属性过滤拓扑）
agent.instance_properties_json={"zone":"sh-a","cluster":"prod-1","group":"trade"}

# 服务层分组（可选）
agent.namespace=${SW_AGENT_NAMESPACE:prod}   # 同 service_name 在不同 namespace 下互不干扰
```

```text
┌──────────────────────────────────────────────┐
│  order-service                                │
│    Instance-A  zone=sh-a cluster=prod-1       │
│    Instance-B  zone=sh-b cluster=prod-1       │  ← 拓扑仍是一个服务
│    Instance-C  zone=sh-b cluster=gray         │     但可按标签过滤/分组看调用边
└──────────────────────────────────────────────┘
```

配合 Dubbo 自身的**标签路由（tag/router）**排障时，把命中的路由标签用 `ActiveSpan.tag("dubbo.tag", tag)` 打进 Span，就能在 Trace 里回答"这次调用走了哪个路由"。

> **行业实践**：不要把机房/环境写进 `service_name`（如 `order-service-sh-a`）。服务名一旦承载拓扑维度，跨环境的调用关系就再也拼不到一张图上——正确做法是服务名保持稳定，环境差异交给 `agent.namespace` 或多套 OAP，实例差异交给 instance_properties。

# 6. Dubbo 3 Triple 协议

Dubbo 3 主推 Triple（HTTP/2 承载，gRPC 兼容）。9.x agent 支持 Triple 的原理是：Triple 复用了 Dubbo 3 的 `Invoker` 抽象与 `RpcInvocation` attachment 机制，因此 consumer/provider 侧的增强点仍然命中；同时 Triple 底层是 HTTP/2，**sw8 也能以 HTTP header 形式在跨语言（gRPC）调用里传播**。

```properties
# agent.config 无需额外配置，插件按依赖的 dubbo 包名与协议自动匹配
# Triple 场景 Span 标签额外包含：
#   rpc.protocol = tri
#   http.method  = POST（Triple 以 HTTP POST 承载）
```

需要留意的兼容性边界：**gRPC stub 形式的 Triple 调用（走 grpc-java 的 Client/Server Call）不会命中 dubbo 插件**，此时要挂 grpc 插件；服务混合暴露 `dubbo` 与 `tri` 两个协议时，两端协议必须一致，否则 provider 侧拿到的是 HTTP header 而非 attachment，提取不到 sw8。

# 7. 与 Seata / Sentinel 共存

多个组件都要往 attachment 塞东西时，冲突点是"键名"和"谁先返回"，不是链路本身：

```text
Consumer (Seata TM)                    Provider (Seata RM)
  │ @GlobalTransactional
  │ ── Dubbo Invoker（SkyWalking 最外层）
  │      attachment:
  │        sw8:        ......          ← SkyWalking 写
  │        TX_XID:     1.2.3.xxx       ← Seata 写
  │ ── Sentinel Dubbo Filter（业务 Filter 链内）
  ▼
  各自读各自的 key，互不干扰；Span 与事务上下文正交
```

Sentinel 的 Dubbo 适配器在 Filter 链内做限流，被 Block 时会在 provider/consumer 侧产生异常；由于 SkyWalking 增强在 Invoker 外层，**BlockException 依然会被记录到当前 Exit/Entry Span 的异常日志里**，Trace 能看到"这次 RPC 被限流了"而不是"RPC 报未知错"。

# 8. 常见丢链原因

按出现频率排序，逐条对应到可操作的检查：

| 症状 | 原因 | 定位与修复 |
|------|------|-----------|
| provider 侧开新 Trace | 业务 Filter/安全组件过滤了 `sw8` attachment key | attachment 白名单放行 `sw8`/`sw8-correlation` |
| consumer 有 Span、provider 无 | provider 未挂 agent，或接口走的是 `LocalInvoker`(injvm) | injvm 是同进程调用，本就不跨 Segment，属正常 |
| 异步调用后半段丢链 | `CompletableFuture`/`AsyncContext` 在别的线程结束 span | 升级 agent 用官方异步支持；自定义线程池加 `@TraceCrossThread` |
| 泛化调用无端点 | `GenericService.$invoke` 方法名在参数里 | 用 `ActiveSpan.tag` 补业务方法名，或让端点名取 `$invoke` 后自行归一 |
| 部分 Span 缺失 | 超过 `agent.span_limit_per_segment` 配额 | 调大配额或减少方法级埋点 |
| 链路有 TraceId 但 UI 查不到 | Trace 未命中采样（未采样只传播不落库） | `agent.sample_n_per_3_secs` 调大或设 -1 |
| 三方 SDK 内部 Dubbo 调用丢链 | 该 SDK 自带 shaded Dubbo 类，包名不匹配增强点 | 核对插件对 shaded 包名的支持，必要时自写 instrumentation |

**最容易误判的是前两条**：它们看起来像"SkyWalking 不支持 Dubbo 某场景"，实际是上下文在传输层被吞掉了。抓一次真实 invocation 打印 attachment 的 key 集合，比读十遍文档有用。

> **注意**：`injvm`（同 JVM 内 consumer/provider 共存）走本地调用不经过网络层，两侧本就在同一 Segment 语境里，不存在"跨进程断链"；若误以为丢链而给 injvm 加强透传，反而制造重复 span。判断是否 injvm，看日志里 invoker 类型即可。

# 9. 配置速查

```properties
# agent.config — Dubbo 场景常用（键名与官方配置表一致）
plugin.dubbo.collect_consumer_arguments=false      # 采集入参（Object#toString），默认关
plugin.dubbo.consumer_arguments_length_threshold=256  # 开启时参数截断长度
plugin.dubbo.collect_provider_arguments=false      # provider 侧同理，生产保持关闭
plugin.dubbo.provider_arguments_length_threshold=256
statuscheck.ignored_exceptions=FlowException,DegradeException  # 限流等控制流异常不计 error
agent.instance_properties_json={"zone":"sh-a","cluster":"prod-1"}  # 集群/机房标签
agent.span_limit_per_segment=300                 # RPC+DB 扇出大时防 span 截断
```

参数采集会把 `Object#toString` 的结果送进上报包体，大对象撑爆包体并泄露业务数据，**生产默认关闭**；确需排查时用长度阈值兜底，或只对特定接口用 `@Tag` 精确取值。`statuscheck.ignored_exceptions` 正是第 7 节 Sentinel BlockException 不计 SLA 损失的配置位。

# 10. 小结

| 主题 | 要点 |
|------|------|
| 实现方式 | 增强 `DubboInvoker#doInvoke` / `AbstractProxyInvoker#invoke`，比 Filter SPI 更靠外、更抗干扰 |
| 透传载体 | `RpcInvocation.attachment` 存 `sw8`，键名与 HTTP header 一致；白名单过滤是头号丢链原因 |
| 指标视角 | client/server 双 detectPoint，差值即网络+框架开销 |
| 集群标签 | 用 instance_properties/namespace，不要污染 service_name |
| Dubbo 3 | Triple 复用 attachment 与 Invoker；gRPC stub 走 grpc 插件；shaded 包名需核对支持矩阵 |
| 共存 | Seata `TX_XID` 与 sw8 各占各 key；Sentinel BlockException 仍进 Span 异常日志 |

下一篇《11. 网关与 Sentinel》把链路的第一跳——网关入口——补全，并看限流事件如何落到 Trace 与指标上。
