
# 1. 概述

网关是链路的第一跳，也是可观测性里最容易"断链"的一跳：请求在这里完成协议转换、路由、鉴权、限流，**Trace 上下文若在这一跳没接住，后面整条链路都会变成孤儿**。SkyWalking 对 Spring Cloud Gateway（以及 Zuul、Nginx、Kong、Envoy）都有官方插件；Sentinel 侧则在网关与注解两种拦截形态下把限流/熔断事件落进 Trace 与指标。本篇按"网关 span 设计 → Sentinel 指标 → 限流进 trace → 多网关组合"的顺序展开。

```text
Client ──→ [Nginx/SLB] ──→ [SCG 网关] ──→ [业务服务 A] ──→ [服务 B / DB / Redis]
              │               │
        Entry/Exit Span   Entry Span + Route Exit Span
        (lua/envoy 插件)   （限流检查在路由前：Block 则整条 Trace 只有网关一跳）
```

# 2. Spring Cloud Gateway 插件

SCG 基于 WebFlux/Reactor，agent 的 spring-cloud-gateway 插件增强 `RoutingWebHandler` 等核心类，把"一次转发"拆成**入口 Entry + 路由 Exit** 两个语义：

```text
请求 GET /api/user/1
  │
  ├── Entry Span: GET:/api/user/**      ← 匹配到的 Route id/predicates 路径
  │
  ├── Exit Span: {目标服务}              ← lb://user-service 解析出的服务名或 host
  │     │  写入 sw8 到转发的 HTTP header
  │     ▼
  │   user-service
  │     └── Entry Span: GET:/api/user/{id}   ← 下游正常接链
  │
  └── Response 回流 → 记录 status_code，≥400 标 error
```

```yaml
# 典型路由配置——agent 会自动为每条 Route 生成端点
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/user/**
          filters:
            - StripPrefix=1
```

**两个设计细节**：第一，Exit Span 的 peer 来自 `lb://` 解析出的**服务名**，因此网关拓扑里"网关 → user-service"这条边的对端直接是注册中心语义的服务名，与下游 agent 上报的 service_name 对得上，拓扑图才会接合；第二，Gateway 的端点名取**Route Predicate 的路径模板**（`/api/user/**`）而非原始 URI，天然避免路径变量爆炸——但 `/**` catch-all 路由会把全站流量折叠进一个端点，端点级指标因此失去分辨率，**关键业务路由要显式声明独立 Path**。

> **注意**：Gateway 上的自定义 GlobalFilter 如果自行 `exchange.getRequest().mutate()` 重建请求且不保留 headers，sw8 会丢；鉴权/签名类过滤器改写请求时务必确认 header 透传。这是"网关以下全部断链"最常见的真凶。

# 3. 网关层 Span 设计：入口段耗时切分

网关一条 Trace 里真正有诊断价值的是**入口段耗时分解**：用户感知的 RT = 网关自身 + 下游。把这一段切开，靠的是网关插件在 Route 转发前后自然形成的时间边界：

```text
GET:/api/user/**  (总 210ms)
  ├── [网关自身] GlobalFilter 链（鉴权/限流/改写）    ≈ 10ms   ← Entry 与 Exit 的间隔
  ├── [Exit Span] lb://user-service 网络+序列化        ≈ 20ms   ← 从发请求到收首字节前
  │     └── 下游 Entry Span: 180ms
  │           ├── /api/user/{id} 业务处理      90ms
  │           ├── JDBC select                  60ms
  │           └── Redis get                    20ms
  └── [网关收尾] 响应改写/压缩                        ≈ ?     ← 看 Entry Span 尾部时间
```

实操要点：**Exit Span 耗时 − 下游 Entry Span 耗时 ≈ 网络与网关转发开销**，与 Dubbo 场景的 client/server 双视角同理。若怀疑网关自身慢（线程池排队、Full GC），看网关实例的 `instance_jvm_*` 指标与"网关有 Entry 无 Exit 的那段时间差"。超时配置要与 Trace 对齐：`response-timeout` 触发时网关产生 error Span，下游可能"还在正常执行"——**两侧指标不一致不是 bug，是超时的语义**，排障时以 Trace 的时间轴为准。

# 4. Sentinel 限流指标接入

Sentinel 自身有完整的指标体系（秒级流水、控制台），与 SkyWalking 的接入分两层：**指标层**走 Micrometer/Prometheus 导出 + `prometheus-fetcher`/MAL 进 OAP；**链路层**走 agent 的 sentinel 插件把资源保护动作记进当前 Span。

```yaml
# application.yml — Sentinel 指标暴露（spring-cloud-starter-alibaba-sentinel）
spring:
  cloud:
    sentinel:
      metrics:
        file-extension: .idx
      eager: true
      transport:
        dashboard: sentinel-console:8080
# 配合 micrometer 导出后，OAP prometheus-fetcher 抓取 sentinel_block_qps 等指标
```

| 指标 | 来源 | 用途 |
|------|------|------|
| pass/block/success QPS | Sentinel 秒级统计 → MAL | 限流压力水位 |
| RT / thread count | Sentinel 资源统计 | 熔断决策依据（SkyWalking 侧只用来自检） |
| `sentinel.block=true` tag | agent sentinel 插件写进 Span | Trace 级过滤 |

> **行业实践**：**别把两套告警混用**。Sentinel 控制台管"流控规则是否生效、block 量是否异常"（秒级、规则维度），SkyWalking 告警管"服务 SLA/RT 是否劣化"（分钟级、实体维度）。限流触发的 block 会拉低通过率，若 SkyWalking 侧把 block 当 error 统计会导致 SLA 告警风暴——正确姿势是用 `toolkit.ignore_exceptions` 忽略 `BlockException`，让限流成为"业务预期行为"而不是"故障"。

# 5. 限流事件进 Trace

Sentinel adapter 在资源被拦截时，agent 插件把拦截信息写进当前 Span，UI 可按 tag 过滤出所有被限流的请求：

```java
@RestController
public class UserController {

    @GetMapping("/user/{id}")
    @SentinelResource(value = "getUser", blockHandler = "handleBlock")
    public User getUser(@PathVariable Long id) {
        return userService.getById(id);
    }

    public User handleBlock(Long id, BlockException ex) {
        // SkyWalking sentinel 插件在此请求的 Entry Span 上记录拦截事件
        // Span 日志: exception=FlowException, resource=getUser
        return new User(id, "Blocked", 0);
    }
}
```

```text
正常请求:
Trace: /api/user/{id} (200ms)
  ├── Entry: GET:/api/user/{id}
  │     ├── Sentinel 资源 getUser → entry OK（插件仅记录被 block 的情况）
  │     └── UserService.getById (150ms) → JDBC (100ms)

被限流请求（blockHandler 兜底返回 200）:
Trace: /api/user/{id} (2ms)
  └── Entry: GET:/api/user/{id}
        └── logs: {exception: "FlowException", resource: "getUser"}
        （无下游 Span——整条链路止步于限流）
```

识别方法：**Trace 耗时极短 + Span 异常日志含 FlowException/DegradeException + 无 Exit Span**，三者同时出现即"被限流而非下游故障"。GraphQL 查这类链路用 `queryTrace(traceId)` 或按 endpoint + is_error 过滤后肉眼扫 tags。

> **注意**：blockHandler 返回 HTTP 200 时，SkyWalking 默认按状态码判成功，**SLA 指标不会因限流下降**——这是"限流不算错"的正面效果；但如果业务把 block 记为 error 日志并上报，日志指标与 trace 指标会出现方向相反的统计，汇报口径要提前统一。

# 6. Gateway + Sentinel 联合

SCG 的 Sentinel 适配器在**路由维度**做流控（resource name = route id），网关被限流时链路更短：

```yaml
spring:
  cloud:
    sentinel:
      scg:
        fallback:
          mode: response            # 直接返回响应（另一种是 redirect 到降级路径）
          response-status: 429
          response-body: '{"code":429,"msg":"too many requests"}'
```

```text
Gateway 被限流时:
  Entry Span: GET:/api/user/**  ── logs: FlowException, resource=user-service
  429 直接返回 → 无 Route Exit Span、无下游任何 Span
  网关侧 service_sla 是否下降取决于 429 是否 ≥400（是的，429 会计 error）
```

与第 5 节的取舍在这里反转：网关 fallback 返回 429 是**状态码 ≥400**，会拉低网关 SLA——此时若不想让限流触发告警，把 `threshold` 调高、或按 `sentinel` tag 过滤告警范围，比改 fallback 状态码更诚实：**429 本来就应该被看见**。

# 7. Nginx 与其他边缘网关

边缘层（LB/Nginx）插件的价值是**把 Trace 起点前移到用户入口**，让"网关前的网络耗时"也被度量：

```nginx
# skywalking-nginx-lua：lua 模块生成 Entry Span 并向下游透传 sw8
http {
    lua_shared_dict tracing_buffer 10m;

    server {
        listen 80;
        location / {
            skywalking_trace on;
            skywalking_app edge-nginx;          # 在 OAP 注册为独立服务
            proxy_pass http://gateway;
            # 插件自动把 sw8 设置为 proxy_set_header，无需手写
        }
    }
}
```

| 网关形态 | 官方支持 | 上下文载体 | 备注 |
|----------|---------|-----------|------|
| Spring Cloud Gateway | ✓ java agent | HTTP header | 与下游 Boot 服务同一 agent 体系 |
| Zuul 1 | ✓ java agent | HTTP header | Zuul 2 支持有限，升级前核对矩阵 |
| Nginx/OpenResty | ✓ skywalking-nginx-lua | sw8 header | lua 生态；Envoy 走 istio/mesh 指标 |
| Kong | ✓（lua 插件/zipkin 兼容） | header | 大量团队直接用 OTel 插件转 OAP OTLP |
| Envoy/Istio | ✓ 原生 SW 指标 | sw8 header | mesh 层，指标精度到连接级 |

# 8. 多网关组合与 Zipkin 兼容

现实里常见"公有云 ALB → Nginx → SCG → 服务"多段网关串联。SkyWalking 插件体系按组件类型匹配、不冲突，但要守住两条纪律：

**纪律一：sw8 必须逐跳透传**。任何一跳做了"重建请求头"的操作都会断链；排查时从外到内逐跳看 attachment/header 里 sw8 是否存在。多语言/商业网关给不了 sw8 支持时，退而求其次让它至少把 sw8 **原样透传**——插件不识别它没关系，识别的跳点能接上就行。

**纪律二：混用 Zipkin 生态时走 receiver 而非双挂探针**。OAP 内置 Zipkin sender receiver（9411），Envoy/旧系统上报的 Zipkin v2 span 可以被接收；`zipkin-sw8` 兼容配置让 Zipkin 客户端也能携带 sw8 与 SkyWalking 链路对接。

```text
ALB(透传) → Nginx-lua(Entry,生成sw8) → SCG(Entry+Exit) → Service(Entry)
                 │                          │
            Zipkin 上报的组件 ──→ OAP zipkin receiver ──→ 汇入同一 Trace（靠 sw8 或共享 traceId）
```

> **行业实践**：多网关组合端点名治理比埋点更重要——每一层都用自己的 Path 模板当端点，同一个业务接口会在三四个服务里各有一个端点名，Dashboard 上"哪个是权威"要先讲清楚。常见约定：以**业务服务的端点**为权威口径，网关端点只看流量与状态码分布，不做业务级 SLA。

# 9. 最佳实践

1. **关键路由显式声明 Path**：避免 `/**` 折叠导致端点失去分辨率
2. **鉴权/签名过滤器保留 header**：防止"网关以下全断链"
3. **BlockException 不计 error**：`toolkit.ignore_exceptions` 排除限流异常，防 SLA 告警风暴
4. **边缘 Trace 起点前移**：Nginx/envoy 插件让"用户 RT"与"服务 RT"差额可见
5. **超时语义对齐**：网关超时产生的 error 与下游正常执行并存，以 Trace 时间轴为准下结论

# 10. 小结

| 主题 | 要点 |
|------|------|
| 网关 span | Entry（Route Path 端点）+ Exit（lb 服务名 peer），二者间隔即网关自身开销 |
| 耗时切分 | Exit − 下游 Entry ≈ 网络/转发；网关指标与 Trace 不一致先想超时语义 |
| Sentinel 指标 | pass/block QPS 走 MAL 进 SkyWalking；控制台管规则、SkyWalking 管实体告警 |
| 限流进 trace | Span 异常日志记 FlowException + 无下游 Span = 被限流而非故障 |
| 多网关 | sw8 逐跳透传是底线；Zipkin 组件经 receiver 汇入，端点口径以业务服务为权威 |

下一篇《12. MySQL/Redis 数据库插件》下沉到链路终点——存储访问段的 span 与慢查询治理。
