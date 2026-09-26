
# 1. 规则设计原则

## 1.1 分层防护体系

```
第一层：网关限流（Route/API Group 级）
  ├── 防止突发流量进入系统
  └── 统一管控入口流量

第二层：服务限流（资源级）
  ├── 保护核心接口不被冲垮
  └── 按接口差异化设置阈值

第三层：系统保护（全局级）
  └── 最后一层兜底，防止系统崩溃
```

## 1.2 阈值设定方法论

```java
// 1. 压测获取基准
// 基准 QPS = 压测最大 QPS × 0.8（安全系数）

// 2. 核心接口 vs 非核心接口
// 核心接口（支付/下单）：阈值 = 基准 QPS
// 非核心接口（查询/统计）：阈值 = 基准 QPS × 0.3

// 3. 超时保护
// Read Timeout < Degrade TimeWindow
// 确保超时发生在熔断之前
```

## 1.3 流控效果选择

| 场景 | 流控效果 | 理由 |
|------|---------|------|
| 通用 HTTP 接口 | 快速失败 | 快速响应，不堆积 |
| 秒杀/抢购 | 匀速排队 | 平滑流量，保护下游 |
| 冷启动/新扩容 | WarmUp | 避免瞬间打满 |
| 定速写入 DB | 匀速排队 | 匹配 DB 写入能力 |

# 2. 降级方案设计

## 2.1 多级降级策略

```
用户请求
  ├── 一级降级：流控拒绝 → 返回"排队中"
  ├── 二级降级：熔断拒绝 → 返回缓存数据
  ├── 三级降级：系统保护 → 返回静态兜底
  └── 正常处理
```

## 2.2 降级返回值设计

```java
// ❌ 错误：返回 null
@SentinelResource(value = "getUser", fallback = "fallback")
public User getUser(Long id) { ... }

public User fallback(Long id, Throwable ex) {
    return null;  // 调用方 NPE
}

// ✓ 正确：返回有意义的兜底对象
public User fallback(Long id, Throwable ex) {
    return User.builder()
        .id(id)
        .name("用户信息暂时不可用")
        .build();
}
```

# 3. 生产配置清单

```yaml
spring:
  application:
    name: order-service

  cloud:
    sentinel:
      # ========== 通信配置 ==========
      transport:
        dashboard: sentinel-dashboard.prod:8080
        port: 8719
        heartbeat-interval-ms: 10000

      # ========== 初始化配置 ==========
      eager: true
      filter:
        enabled: true
        order: -2147483648     # 最高优先级

      # ========== 日志配置 ==========
      log:
        dir: /var/log/sentinel
        switch-pid: true

      # ========== 数据源配置 ==========
      datasource:
        flow:
          nacos:
            server-addr: nacos.prod:8848
            namespace: production
            data-id: ${spring.application.name}-flow-rules
            group-id: SENTINEL_GROUP
            data-type: json
            rule-type: flow

        degrade:
          nacos:
            server-addr: nacos.prod:8848
            namespace: production
            data-id: ${spring.application.name}-degrade-rules
            group-id: SENTINEL_GROUP
            rule-type: degrade

        system:
          nacos:
            server-addr: nacos.prod:8848
            namespace: production
            data-id: ${spring.application.name}-system-rules
            group-id: SENTINEL_GROUP
            rule-type: system

      # ========== 全局异常配置 ==========
      block-page: /sentinel/block

  # ========== 业务超时配置 ==========
  cloud:
    openfeign:
      client:
        config:
          default:
            connect-timeout: 3000
            read-timeout: 5000       # < Sentinel 熔断 TimeWindow

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
```

# 4. 接口分级管理

```java
// 接口分级枚举
public enum ApiLevel {
    CORE(100),       // 核心接口：支付、下单
    IMPORTANT(50),   // 重要接口：查询、校验
    NORMAL(30),      // 普通接口：列表、搜索
    LOW(10);         // 低优先级：统计、日志

    final int defaultQps;
}

// 统一限流配置
@PostConstruct
public void initFlowRules() {
    FlowRuleManager.loadRules(Arrays.asList(
        createRule("POST:/order/create", ApiLevel.CORE),
        createRule("GET:/order/{id}", ApiLevel.IMPORTANT),
        createRule("GET:/order/list", ApiLevel.NORMAL)
    ));
}
```

# 5. 灰度发布中的流量控制

```yaml
# 灰度发布场景：新版本只接收 10% 流量
# 方案：通过 origin 区分灰度版本
spring:
  cloud:
    sentinel:
      datasource:
        flow:
          nacos:
            data-id: order-service-flow-rules

# Nacos 规则配置：
# [
#   { "resource": "POST:/order/create",
#     "limitApp": "order-service-canary",  # 灰度版本
#     "count": 10 },                       # 限 10 QPS
#   { "resource": "POST:/order/create",
#     "limitApp": "default",               # 正式版本
#     "count": 100 }                       # 限 100 QPS
# ]
```

# 6. 开发规范

## 6.1 DO

- ✓ 为每个对外接口添加 `@SentinelResource`
- ✓ 始终提供有意义的 fallback 返回值
- ✓ 规则持久化到 Nacos/Apollo
- ✓ 核心接口单独配置熔断规则
- ✓ 使用 `fallbackFactory` 记录降级原因日志
- ✓ 生产环境关闭 Dashboard 原始推送模式

## 6.2 DON'T

- ✗ 不要返回 null 作为降级值
- ✗ 不要在 blockHandler 中调用远程服务
- ✗ 不要为每个 URL 参数组合创建独立规则
- ✗ 不要在 Sentinel 规则检查中做耗时操作
- ✗ 不要依赖 Dashboard 内存存储规则到生产
- ✗ 不要把 Sentinel 当作业务异常处理工具

# 7. 容器化部署建议

```yaml
# K8s Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: order-service
          env:
            - name: JAVA_OPTS
              value: >-
                -Dcsp.sentinel.dashboard.server=sentinel-dashboard:8080
                -Dcsp.sentinel.log.dir=/var/log/sentinel
                -Dproject.name=order-service
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: 2000m
              memory: 2Gi
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
```
