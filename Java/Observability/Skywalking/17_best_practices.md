
# 1. Agent 部署策略

```
策略对比：

Sidecar 模式（K8s）：
  ├── Agent 作为 InitContainer 拷贝到共享卷
  ├── 应用容器挂载 -javaagent
  ├── 优点：Agent 版本独立管理
  └── 缺点：Pod 内存额外 +100MB

启动参数模式：
  ├── -javaagent:/path/to/skywalking-agent.jar
  ├── 优点：简单直接
  └── 缺点：Agent 升级需重启

镜像内置模式：
  ├── Agent 打包在应用镜像中
  ├── 优点：最简部署
  └── 缺点：Agent 升级需重新构建镜像
```

# 2. 服务命名规范

```properties
# 推荐命名格式
agent.service_name=${spring.application.name}
# 或
agent.service_name=${ENV}-${APP_NAME}

# 示例
agent.service_name=prod-order-service
agent.service_name=staging-user-service
```

# 3. 采样配置

```properties
# 生产环境采样策略
agent.sample_n_per_3_secs=500       # 每 3 秒最多 500 条
agent.force_tracing=false           # 关闭全局强制追踪

# 忽略端点（大量调用但不重要的端点）
trace.ignore_path=/health,/metrics,/actuator/**,/favicon.ico
```

# 4. 端点爆炸防控

```properties
# 避免 Restful ID 导致端点爆炸
# /api/user/1 → /api/user/{id}
# /api/user/2 → /api/user/{id}

# 方式 1: Agent 配置
plugin.springmvc.urlSuffixRestPath=/api/user/\d+

# 方式 2: 代码中自定义
@GetMapping("/api/user/{id}")
public User getUser(@PathVariable Long id) { ... }
# Agent 自动识别 @PathVariable → /api/user/{id}
```

# 5. 日志关联配置

```xml
<!-- 所有服务统一 Pattern -->
<Pattern>%d [%tid] %-5level %logger{36} - %msg%n</Pattern>
```

```properties
# 日志级别控制
logging.level.org.apache.skywalking=WARN    # 生产环境
```

# 6. 资源容量规划

```
指标计算：
  每日 Trace 量 ≈ Agent 实例数 × QPS × 3600 × 24 × 采样率

  存储容量 ≈ 每日 Trace 量 × 平均 Span 数 × 2KB
             + 指标数 × 服务数 × 200B

  示例：
    10 个服务，每个 500 QPS，采样率 100%（即 500）
    每日 Trace ≈ 10 × 500 × 3600 × 24 = 4.32 亿条 → 需要降采样

    采样率设为 100（每 3 秒 100 条）
    每日 Trace ≈ 10 × 100 × 20 × 24 = 48 万条

    存储 ≈ 48万 × 10 Span × 2KB ≈ 10GB/天
    ES 集群建议 3 节点各 4C8G + 200GB SSD
```

# 7. 安全建议

```
□ Agent 不暴露端口（仅出站 gRPC 连接）
□ OAP gRPC 端口不对外开放（仅内网可访问）
□ UI 通过反向代理暴露（Nginx + Basic Auth）
□ 存储密码使用环境变量或 K8s Secret
□ RBAC 管理 UI 权限
```

# 8. DON'T

```
✗ 不要在数据库事务中创建大量 Span（影响性能）
✗ 不要在生产环境开启所有 SQL 参数追踪
✗ 不要忘记设置 Trace TTL（存储会无限增长）
✗ 不要用 H2 做生产存储（数据重启丢失）
✗ 不要在方法内部循环中创建 Span（每个 Span 有 ~3μs 开销）
✗ 不要遗漏 actuator/health 端点过滤（会污染 Trace）
✗ 不要让 agent.service_name 在多个不相关服务间共用
```
