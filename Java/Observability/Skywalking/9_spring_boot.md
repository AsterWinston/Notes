
# 1. 概述

SkyWalking Java Agent 提供 Spring Boot 的**全自动集成**，无需修改代码：Agent 在类加载期拦截 Spring 的关键扩展点，自动识别 Spring MVC/WebFlux、RestTemplate、Feign、`@Scheduled`、`@Async` 等组件并生成对应 Span。理解这张"自动埋点地图"是做好 Spring Boot 可观测性的前提——知道哪些环节免费拿到，哪些环节需要额外动手：

```text
Spring Boot 应用
  │
  ├── Spring MVC / WebFlux   → Entry Span（HTTP 端点，自动）
  ├── @Service / @Repository → Local Span（方法级，需 optional 注解插件）
  ├── @Scheduled             → Local/Entry Span（定时任务，自动）
  ├── @Async / 线程池         → 跨线程 Trace 传播（自动，覆盖主流池）
  ├── RestTemplate / Feign   → Exit Span（HTTP 调用，自动 + 头透传）
  └── JDBC / Redis / Kafka   → Exit Span（中间件访问，自动）
```

> **注意**：自动 ≠ 全量。**纯业务方法默认没有 Span**（无插件的类不会被增强），方法级细节要么用 `@Trace` 注解，要么用 Profiling 任务临时抓栈——不要为了"看得细"就把 spring-annotation 插件全量打开，那会让每个 Bean 方法都产生 Local Span，配额和存储都撑不住。

# 2. Agent 挂载与目录结构

挂载方式是 JVM 标准的 `-javaagent`，Spring Boot  fat jar 同样适用（agent 先于应用 premain）：

```bash
java -javaagent:/opt/skywalking/agent/skywalking-agent.jar \
     -DSW_AGENT_NAME=order-service \
     -DSW_AGENT_COLLECTOR_BACKEND_SERVICES=oap.prod:11800 \
     -jar order-service.jar
```

agent 发行目录值得逐层认识一遍，排障时 90% 的操作都落在这里：

```text
skywalking-agent/
├── skywalking-agent.jar     # premain 入口（MANIFEST 指定 Premain-Class）
├── config/
│   ├── agent.config         # 全部配置项，${ENV:默认值} 形式
│   └── logback.xml          # agent 自身日志配置
├── plugins/                 # 默认启用插件（springmvc/feign/jdbc/...）
├── optional-plugins/        # 可选插件，拷入 plugins/ 才生效
├── bootstrap-plugins/       # 增强 JDK 核心类（HttpURLConnection 等）
├── activations/             # toolkit 桥接激活包（注解/日志/Micrometer）
├── expired/                 # 不再维护的旧版插件
└── logs/                    # agent.log，排障第一现场
```

**插件生效链**是 `plugins/ 内 jar + skywalking-plugin.def 声明 + 目标类真实存在` 三者同时满足——Spring Boot 应用若没引入 Feign 依赖，agent 有插件也不会产生任何增强，因此"某框架没数据"先查依赖版本是否在插件支持矩阵内，再查 agent.log。

# 3. 启动配置

```properties
# agent/config/agent.config（与 Boot 应用最常打交道的几项）
agent.service_name=${SW_AGENT_NAME:order-service}
collector.backend_service=${SW_AGENT_COLLECTOR_BACKEND_SERVICES:oap.prod:11800}
agent.sample_n_per_3_secs=${SW_AGENT_SAMPLE:-1}
agent.ignore_suffix=${SW_AGENT_IGNORE_SUFFIX:.jpg,.ico,.css,.js}
agent.span_limit_per_segment=300                # 单次请求 span 配额，超出丢弃

plugin.springmvc.collect_http_params=true       # 采集 URL query 参数
plugin.feign.request_header_collection=true     # Feign 透传/收集请求头（部分版本为 collect_request_body）
logging.level=INFO
```

配置优先级仍是 `-D > 环境变量 > agent.config`。**Boot 3 / Jakarta 支持要求 skywalking-java 9.0+**：9.0 起插件定义全面适配 `jakarta.*` 命名空间与 Spring 6 的 API 变化（如 Controller 增强点、`ServerWebExchange` 内部结构），8.x agent 挂 Boot 3 的典型症状是 **agent 启动正常但 HTTP 入口无 span**——不是报错，而是增强点匹配不到类。Boot 3 + JDK 17/21 还要留意 `--add-opens`：个别反射式插件（涉及 JDK 内部包的）需要放开模块封装，agent.log 会明确给出 `Illegal reflective access` 提示。

# 4. Spring MVC 自动埋点

Agent 拦截 `DispatcherServlet` 与 HandlerMapping 相关路径，Entry Span 的创建时机在请求进入 Servlet 容器后、路由到 Controller 前：

```text
HTTP 请求 → DispatcherServlet.doDispatch()
  │
  ├── 拦截点 1: 获得 Handler → 取最佳匹配 PathPattern（端点名来源）
  ├── 拦截点 2: HandlerAdapter.handle() → 执行 Controller
  │     └── （spring-annotation 插件开启时才有方法级 Local Span）
  └── 拦截点 3: 响应完成 → 记录 HTTP 状态码，≥400 标记 error
```

**自动记录的标签**：

| 标签 | 说明 |
|------|------|
| `url` | 请求 URL |
| `http.method` | HTTP 方法 |
| `status_code` | 响应状态码 |
| `http.params` | 请求参数（`collect_http_params=true` 时） |

端点名默认是 `HTTP方法:最佳匹配路径`（如 `GET:/api/user/{id}`），路径来自 Spring 的 PathPattern 匹配结果而非原始 URI——这天然规避了"路径变量导致端点爆炸"的问题，前提是你用 `@PathVariable` 声明了模板路径；**正则通配或 Ant 风格混写时仍可能出现高基数端点**，上线前在 UI 端点列表里数一遍。

# 5. 出向调用与透传：RestTemplate / Feign / WebFlux

跨进程链路能否接上，取决于**出向调用是否被识别 + `sw8` 头是否写入请求**。三类主流客户端都是官方插件，行为各有细节：

```text
OrderService ──RestTemplate/Feign/WebClient──→ UserService
   │                    │
   │            拦截点：exchange/doExecute/execute
   │            1. 创建 Exit Span（peer = 目标 host:port）
   │            2. CarrierItem 写入 header：sw8 / sw8-correlation / sw8-extends
   ▼                    ▼
  Entry Span      下游 springmvc Entry Span 解析 sw8 → 同一 TraceId、新 SegmentId
```

- **RestTemplate**：增强 `RestTemplate#doExecute`，标准场景全覆盖；但用自定义 `ClientHttpRequestInterceptor` 重写字节流、或走非默认的 `AsyncClientHttpRequestFactory` 时，注意拦截顺序与插件兼容性。
- **OpenFeign**：增强 Feign 的 `Target#apply(RequestTemplate)` 阶段写头；`plugin.feign.opt_collection_headers` 可指定透传的业务头（经 `sw8-correlation` 编码进链路）。网关聚合类服务里 Feign 调用密集，记得核对 correlation 头大小限制，超大自定义头会被丢弃。
- **WebFlux/WebClient**：基于 Reactor 的**上下文传播不依赖 ThreadLocal**，9.x 插件通过 `Context` 机制接上下文，SSE/长连接场景 span 结束时机跟随响应体流完成——若下游首包后即断流，span 可能提前或异常结束，这类"链路时序怪异"优先怀疑背压/取消信号。
- **`@Async`/线程池**：主流线程池（JDK、Spring ThreadPoolTaskExecutor）的提交-执行传播由 `apm-jdk-threadpool-plugin` 自动处理；自建包装类提交（lambda 被自定义 Executor 套了两层）时用 toolkit 的 `@TraceCrossThread`/`SkyWalkingRunnable` 显式桥接。

# 6. Micrometer 指标接入

Spring Boot 的 Actuator 指标生态是 Micrometer。SkyWalking 有两种接入姿势，按需选择：

| 方式 | 原理 | 适用 |
|------|------|------|
| **toolkit 推送** | 引 `apm-toolkit-micrometer-1.5`，注册 `MeterRegistry` 增强，指标经 agent 的 Meter 通道周期上报 OAP，MAL 规则转成 SkyWalking 指标 | 想要统一在 SkyWalking UI 看板展示 |
| **Prometheus 抓取** | 暴露 `/actuator/prometheus`，OAP `prometheus-fetcher` 周期拉取 + MAL 分析 | 已有 Prometheus 生态，只做旁路融合 |

```xml
<!-- 方式一：toolkit 依赖（不改变 Micrometer 使用方式） -->
<dependency>
    <groupId>org.apache.skywalking</groupId>
    <artifactId>apm-toolkit-micrometer-1.5</artifactId>
    <version>${skywalking.version}</version>
</dependency>
```

> **设计争议**：Boot 3 的 `micrometer-tracing`（Observability API）是否要配合 SkyWalking agent？两套追踪体系（sw8 上下文 vs traceparent 上下文）并行时，agent 只认 `sw8`，micrometer 只认 `traceparent`，**混用会产生断裂的双链路**。工程上的收敛方案是二选一：要么全栈用 agent（放弃 micrometer-tracing 的导出能力，只留它的 metrics），要么让 micrometer-tracing + OTLP exporter 直连 OAP 的 OTLP receiver。别指望它们互相无缝翻译。

# 7. @Trace 注解增强

方法级 Local Span 的标准做法，需要 `apm-toolkit-trace` 依赖 + agent 的 annotation 插件（位于 `activations/`，默认已激活）：

```java
@Service
public class OrderService {

    @Trace                    // 方法进出自动创建/停止 Local Span
    @Tag(key = "userId", value = "arg[0]")           // SpEL 式取值：入参
    @Tag(key = "result", value = "returnedObj.status") // 返回值
    public Order createOrder(Long userId, CreateOrderRequest req) {
        return doCreate(req);
    }
}
```

三个高频坑：**其一**，注解对"被增强的 Bean 方法"生效——同类内 `this.method()` 自调用绕过代理，不会产 span；**其二**，`@Tag` 的取值表达式在方法执行后求值，`arg[0]` 若指向可变对象记录的是事后状态；**其三**，高频调用方法慎用（每请求都建 span），先问自己"这 5% 的请求细节值得 100% 的开销吗"。

# 8. 端点规范化与忽略

即使有 PathPattern 兜底，仍要治理这两类端点爆炸源：**外部传入的任意路径**（catch-all `/**` 转发、老接口 `/user/{id}.htm` 参数混在路径里）和**健康检查噪音**：

```properties
# 忽略静态资源与常见后缀（内置能力）
agent.ignore_suffix=.jpg,.ico,.css,.js

# 忽略指定路径：trace-ignore 插件（从 optional-plugins 拷入 plugins/）
# 配置文件 trace-ignore.conf 放到 agent/config/ 后编辑
skywalking.exclude_path=/actuator/**,/health/**,/api/legacy/**
```

catch-all 转发场景推荐在网关/聚合层做**路径前缀映射**（把动态段折叠成固定 endpoint），或者接受它们被归入 `GET:/**` 这类粗粒度端点——SkyWalking 不会自动折叠未声明模板的路径，这是 Spring MVC"最佳匹配"机制的另一面。

# 9. 本地 Docker 联调

开发机最常见的拓扑是"OAP+UI 在容器、应用在 IDE"，两个易错点：**容器端口要映射出来**（`-p 11800:11800`）、**IDE 应用指 `localhost:11800`**（宿主机视角）。容器里的应用指 OAP 则必须用服务名。完整最小验证链：

```bash
# 1) 起 OAP + UI（standalone/ES 均可，参考《3. 快速开始》；两容器同网络才能互访）
docker network create sw
docker run -d --name oap --network sw -p 11800:11800 -p 12800:12800 \
  -e SW_STORAGE=banyan apache/skywalking-oap-server:9.7.0
docker run -d --name ui --network sw -p 8080:8080 \
  -e SW_OAP_ADDRESS=http://oap:12800 apache/skywalking-ui:9.7.0

# 2) IDE 的 VM options 指向宿主机端口
-javaagent:D:/skywalking/agent/skywalking-agent.jar
-DSW_AGENT_NAME=order-local
-DSW_AGENT_COLLECTOR_BACKEND_SERVICES=127.0.0.1:11800
```

> **行业实践**：本地联调建议固定一套"开发专用 agent.config 副本"（采样全量、`collect_http_params=true`、日志 DEBUG），与生产模板 diff 管理；千万不要用生产 config 调试后忘了改回来，`collect_http_params` 把身份证号码带进 ES 是真实发生过的数据泄漏。

# 10. Spring Boot Actuator 集成

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus   # 需要指标导出时加 prometheus
  metrics:
    tags:
      application: ${spring.application.name}      # 指标统一打服务名标签
```

Actuator 端点本身会被 agent 当作普通 MVC/WebFlux 端点生成 Trace——量大且无业务意义，**务必按第 8 节忽略**；若走第 6 节的 Prometheus 抓取方式，`/actuator/prometheus` 端点也要一并忽略，否则 OAP 每次 fetch 都给自己制造一条 Trace。

# 11. 最佳实践

1. **agent.service_name 唯一且稳定**：拓扑分组键，改名等于断史
2. **HTTP 参数采集谨慎开启**：`collect_http_params` 可能泄露敏感信息，生产按需白名单开启
3. **端点规范化**：用 `@PathVariable` 模板 + trace-ignore 治理高基数路径
4. **健康检查忽略**：`/actuator/**`、`/health` 不追踪
5. **Boot 3 用 9.x agent**：jakarta 命名空间支持是硬门槛
6. **线程池覆盖验证**：上线前用一条异步链路实测 sw8 是否贯通，别信"应该支持"

# 12. 小结

| 主题 | 要点 |
|------|------|
| 挂载与目录 | `-javaagent` + plugins/optional-plugins/activations 三层插件目录，生效需依赖真实存在 |
| Boot 3 | 必须 skywalking-java 9.x；症状常为"agent 正常但无 span"而非报错 |
| 透传 | RestTemplate/Feign/WebClient 自动写 `sw8`；WebFlux 走 Reactor Context；自定义池用 `@TraceCrossThread` |
| Micrometer | toolkit 推送或 prometheus-fetcher 抓取，MAL 收敛；与 micrometer-tracing 双体系要二选一 |
| 治理 | 端点靠 PathPattern 天然规范 + ignore 兜底；Actuator 噪音必须忽略 |

下一篇《10. Dubbo 集成》把视线从 HTTP 栈挪到 RPC 栈，看 SPI Filter 如何复制同样的透传逻辑。
