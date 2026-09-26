
# 1. 概述

SkyWalking Profiling 提供**方法级性能诊断**，帮助定位 CPU 热点、内存泄漏和线程阻塞。

```
传统 APM 只能看到：
  /api/user 端点耗时 200ms

Profiling 可以看到：
  /api/user 耗时 200ms
    ├── UserService.getById() 120ms
    │     └── JDBC select 100ms
    ├── AccountService.debit() 60ms
    │     └── JDBC update 50ms
    └── JSON.toJSONString() 20ms
```

# 2. Profiling 任务创建

```bash
# 通过 CLI 创建 Profiling 任务
# 针对 /api/user 端点，监控 50 条 Trace
curl -X POST http://oap:12800/graphql \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation {
      createProfileTask(input: {
        serviceId: \"my-service\",
        endpointName: \"/api/user\",
        monitorDuration: 10,
        minDurationThreshold: 100,
        dumpPeriod: 10,
        maxSamplingCount: 50
      }) { id }
    }"
  }'
```

| 参数 | 说明 |
|------|------|
| `monitorDuration` | 监控持续时间（分钟） |
| `minDurationThreshold` | 最小耗时阈值（ms），低于此值不采样 |
| `dumpPeriod` | 线程栈采样间隔（ms） |
| `maxSamplingCount` | 最大采样 Trace 数 |

# 3. Profiling 工作流程

```
1. OAP 创建 Profiling Task
2. Agent 通过 gRPC 查询待执行的 Task
3. Agent 开始监控匹配的 Trace
   - 每个满足条件的 Span 触发线程栈快照
   - 快照存储在 Segment 中
4. Segment 上报到 OAP
5. OAP 聚合分析 → 生成性能报告
6. UI 展示火焰图 / 调用树
```

# 4. 性能剖析可视化

```
UI 展示：
┌─────────────────────────────────┐
│        Flame Graph               │
│                                    │
│  main()                           │
│  ├── service() ─── 80%            │
│  │     ├── db.query() ─── 50%     │
│  │     └── redis.get() ─── 30%    │
│  └── controller() ─── 20%        │
│        └── jsonSerialize() ─── 20%│
└─────────────────────────────────┘
```

# 5. 连续剖析（eBPF）

SkyWalking 9.x 集成 eBPF 支持内核级监控：

```yaml
# eBPF Profiling 可以监控：
# - 网络 I/O 延迟（TCP 连接建立、数据传输）
# - 文件 I/O 延迟（磁盘读写）
# - CPU 使用率和上下文切换
# - 内存分配和回收
```

```bash
# 部署 eBPF Agent
# 需在宿主机上加载 eBPF 程序
docker run -d \
  --privileged \
  -v /sys/kernel/debug:/sys/kernel/debug \
  apache/skywalking-rover:latest
```

# 6. 自定义方法追踪

```java
// @Trace 注解追踪方法耗时
@Service
public class UserService {

    @Trace(operationName = "getUserById")
    @Tag(key = "userId", value = "arg[0]")
    @Tag(key = "result", value = "returnedObj.name")
    public User getUserById(Long userId) {
        // 方法执行时间自动记录到 Span
        return userMapper.selectById(userId);
    }
}
```

# 7. 最佳实践

1. **按需开启**：Profiling 有性能开销（2-5%），生产环境仅在排查问题时启用
2. **阈值筛选**：`minDurationThreshold` 过滤掉正常耗时，聚焦长耗时 Trace
3. **采样控制**：`maxSamplingCount` 限制采样数，避免 OAP 存储压力
4. **定期诊断**：每月的性能回归测试时开启 Profiling
5. **结合 Tracing**：先在 Tracing 中找到慢端点，再用 Profiling 深入分析
