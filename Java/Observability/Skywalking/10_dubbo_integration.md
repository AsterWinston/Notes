
# 1. 概述

SkyWalking Java Agent 内置 Dubbo 2.7/3.x 插件，自动为 Dubbo RPC 调用创建 Span。

```
Consumer                                    Provider
    │                                          │
    │ Dubbo Invocation                         │
    │ 创建 Exit Span                           │
    │   operation: {interface}:{method}         │
    │   peer: provider-ip:port                 │
    │                                          │
    │ 注入 sw8 到 RpcInvocation.attachment     │
    │ ─────────────────────────────────────→   │
    │                                          │ 提取 sw8 → 创建 Entry Span
    │                                          │   operation: {interface}:{method}
    │ ←──────────── Response ────────────────  │
    │                                          │
    │ 结束 Exit Span                          │ 结束 Entry Span
```

# 2. 自动埋点

Dubbo 插件拦截 Dubbo Filter 链，自动生成 Span：

| 位置 | Span 类型 | 操作名示例 |
|------|----------|-----------|
| Consumer 发送请求 | Exit | `Dubbo/com.example.UserService.getUser(Long)` |
| Provider 接收请求 | Entry | `Dubbo/com.example.UserService.getUser(Long)` |

# 3. Span 标签

Agent 自动添加以下标签：

| 标签 | 说明 |
|------|------|
| `dubbo.url` | Dubbo URL |
| `dubbo.version` | Dubbo 版本 |
| `dubbo.parameter` | RPC 参数 |
| `dubbo.interface` | 接口全限定名 |
| `dubbo.method` | 方法名 |

# 4. Filter 链顺序

SkyWalking Dubbo Filter 的优先级：

```
Consumer Filter 链:
  1. ConsumerContextFilter
  2. SkyWalkingDubboConsumerFilter  ← 在 Context 后立即执行
  3. ... 业务 Filter ...
  4. DubboInvoker

Provider Filter 链:
  1. ContextFilter
  2. SkyWalkingDubboProviderFilter  ← 在业务逻辑前创建 Span
  3. ... 业务 Filter ...
  4. 业务实现类
```

# 5. Dubbo 3 Triple 协议

Dubbo 3 的 Triple 协议（基于 HTTP/2 + gRPC）同样支持：

```properties
# agent.config — 无需额外配置，Agent 自动识别 Triple 协议
# Triple 的 Span 标签额外包含：
# - dubbo.protocol=triple
# - http.method=POST
```

# 6. Dubbo + Seata xid 传播

```
Consumer (TM)                          Provider (RM)
  │ @GlobalTransactional                 │
  │ xid = xxx                           │
  │                                     │
  │ Dubbo RPC                           │
  │ attachment:                          │
  │   sw8: ......                       │
  │   TX_XID: xxx                       │  ← Seata 与 SkyWalking 共存
  │ ──────────────────────────────→     │
  │                                     │
```

两个传播协议互不冲突，分别在 `attachment` 中传递。

# 7. Dubbo + Sentinel 联动

```
Dubbo Consumer
  │
  ├── Sentinel Exit Span（流控前）
  │     └── 如果被 BlockException → 标记 Span 为 error
  │
  ├── Dubbo Exit Span（实际 RPC）
  │
  └── Sentinel + Dubbo Span 关联在同一 Trace
```

# 8. 配置

```properties
# agent.config — Dubbo 相关
plugin.dubbo.collect_consumer_arguments=true
plugin.dubbo.collect_provider_arguments=false   # 生产通常关闭
```

# 9. 最佳实践

1. **Provider 端参数关闭**：生产环境 `collect_provider_arguments=false` 防止敏感数据泄露
2. **异步 Dubbo**：Agent 自动处理 `CompletableFuture` 异步调用的 Trace 传播
3. **泛化调用**：Dubbo 泛化调用（GenericService）同样被追踪
4. **Filter 顺序**：确保 SkyWalking Filter 在 Context Filter 之后立即执行
