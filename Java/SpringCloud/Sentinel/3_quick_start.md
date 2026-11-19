# 1. 环境准备

| 组件 | 版本要求 |
|------|---------|
| JDK | 17+（推荐 21） |
| Spring Boot | 3.2+ |
| Spring Cloud | 2023.x |
| Spring Cloud Alibaba | 2023.0.1+ |
| Sentinel | 1.8.6+ |

## 1.1 版本对应关系的读法

这张表约束的是 **Spring Cloud Alibaba（SCA）starter** 这条线，不是 Sentinel 内核——`sentinel-core` 本身兼容 JDK 8+，Boot 3 适配的关键在 SCA 2022.0.x 起全面切到 `jakarta.*` 命名空间：Boot 2.7/Cloud 2021 用 SCA 2021.0.6.0（javax），Boot 3.x 必须 SCA 2022.0.x/2023.x（jakarta），混用的症状是 Web 适配器 `ClassNotFoundException: javax.servlet.Filter`。另一条容易忽视：**Dashboard 与客户端版本相互独立**，Dashboard 1.8.6 可以管 1.8.x 全系客户端，协议向后兼容是有意设计——不需要为了升控制台重发所有应用。

# 2. 依赖引入

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>2023.0.1.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- Sentinel 核心依赖 -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
    </dependency>

    <!-- Spring Boot Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Sentinel Dashboard 通信（推荐） -->
    <dependency>
        <groupId>com.alibaba.csp</groupId>
        <artifactId>sentinel-transport-simple-http</artifactId>
    </dependency>
</dependencies>
```

## 2.1 starter 里装了什么、还要单独加什么

`spring-cloud-starter-alibaba-sentinel` 已经传递带入 `sentinel-core`、`sentinel-transport-simple-http`（原注释"推荐单独引入"在 starter 时代是冗余动作，留着只为看清依赖）和自动配置类。真正要**按需补**的是三件：`sentinel-annotation-aspectj`（SCA 部分版本不自动传递，`@SentinelResource` 需要 `SentinelResourceAspect` Bean 生效，注解不生效时第一查这里，§4.3）、`sentinel-parameter-flow-control`（热点参数规则，同时决定 `ParameterFlowSlot` 是否进链，`2_architecture.md` §3.3）、`spring-cloud-alibaba-sentinel-datasource` + `sentinel-datasource-nacos`（规则持久化，§5.3）。**Dashboard 不在依赖里**——它是独立进程：`java -Dserver.port=8080 -Dcsp.sentinel.dashboard.server=localhost:8080 -jar sentinel-dashboard.jar`（后一个参数对 dashboard 自身无意义，可省；示例取默认 8080）。

# 3. 配置文件

```yaml
server:
  port: 8080

spring:
  application:
    name: sentinel-demo

  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080     # Dashboard 地址
        port: 8719                    # 客户端与 Dashboard 通信端口

      # 取消 Dashboard 懒加载（启动即注册）
      eager: true

      # 数据源（Nacos 持久化）
      datasource:
        flow:
          nacos:
            server-addr: localhost:8848
            data-id: ${spring.application.name}-flow-rules
            group-id: SENTINEL_GROUP
            data-type: json
            rule-type: flow

      # 全局默认规则
      filter:
        enabled: true                  # 启用 Servlet Filter 自动拦截
```

## 3.1 先解决示例自身的端口冲突

`server.port: 8080` 与 Dashboard 默认端口 8080 **撞车**——照抄会有一方起不来。实践改法：应用换 `server.port: 18080`（`transport.dashboard` 仍指 `localhost:8080` 指向 Dashboard），或者反过来给 Dashboard `--server.port=9000` 并把 dashboard 地址同步改掉。两个 8080 的混淆还常出现在排障语境里，先分清这台机器上其实有**三个端口**：

| 端口 | 归属 | 用途 |
|------|------|------|
| 18080（示例改后） | 业务应用 | 业务 HTTP |
| 8080 | Dashboard | 控制台 Web UI |
| 8719 | 应用内 Command Center | 接收心跳应答方的指标拉取/规则命令 |

`transport.port` 的语义是**本机 Command Center 监听端口**（Dashboard 反向连它拉 `/metrics`），被占用时客户端自动 +1 递增尝试（8720、8721…），注册信息随心跳带过去，所以端口漂移不影响连通；防火墙/安全组场景要保证 Dashboard → 各应用 8719± 网段可达，这常是"控制台有心跳但曲线全是空"的真凶。

## 3.2 每个配置项配错的症状

| 配置项 | 默认值 | 配错/不配的症状 |
|--------|--------|----------------|
| `transport.dashboard` | 无（不配则完全不注册） | 控制台永远看不到应用 |
| `eager` | `false` | 首次请求前不注册（懒加载），"部署完控制台没应用"多半是这个 |
| `filter.enabled` | `true` | 关掉后 URL 不再自动成资源，只剩手动/注解资源 |
| `datasource.flow.*` | 无 | 不配则规则只有内存态：Dashboard 裸推、重启归零 |
| `transport.heartbeat-interval-ms` | 5000 | 调大可减心跳流量，代价是应用死后控制台滞留显示 |

`eager: true` 强烈建议生产常开：懒加载的本意是避免未用 Sentinel 的机器白起 HTTP server，但代价是"发布后没人访问 → 控制台无实例 → 运维无法预配规则"的鸡生蛋问题，开源自检脚本也会因首请求被计入统计而失真。

# 4. 第一个限流示例

## 4.1 使用 @SentinelResource 注解

```java
@RestController
@RequestMapping("/api")
public class DemoController {

    @GetMapping("/hello")
    @SentinelResource(
        value = "sayHello",              // 资源名称
        blockHandler = "handleBlock",    // 限流/降级后的处理方法
        fallback = "handleFallback"      // 业务异常后的处理方法
    )
    public String sayHello(@RequestParam(defaultValue = "World") String name) {
        return "Hello, " + name;
    }

    // blockHandler: 参数与返回值与原方法一致，额外 BlockException
    public String handleBlock(String name, BlockException ex) {
        return "Too many requests! Please try later.";
    }

    // fallback: 参数与返回值与原方法一致，额外 Throwable
    public String handleFallback(String name, Throwable ex) {
        return "Service error, please try later.";
    }
}
```

## 4.2 使用 API 方式

```java
@RestController
@RequestMapping("/api")
public class DemoController {

    @GetMapping("/user/{id}")
    public String getUser(@PathVariable Long id) {
        // 定义资源
        try (Entry entry = SphU.entry("getUser")) {
            // 业务逻辑
            return "User: " + id;
        } catch (BlockException e) {
            // 被拦截
            return "Request blocked!";
        }
    }
}
```

## 4.3 两个易踩的第一坑

**坑一：注解没生效、资源没出现在控制台。** `@SentinelResource` 靠 AspectJ 切面实现，需要容器里有 `SentinelResourceAspect` Bean（`@Bean public SentinelResourceAspect sentinelResourceAspect() { return new SentinelResourceAspect(); }`，部分 SCA 版本自动配置、部分要手加——`sentinel-annotation-aspectj` 依赖同时在位）。另一个隐雷是**同类内部调用不走切面**（Spring AOP 代理语义，`20_annotation_source.md`）。

**坑二：URL 资源与注解资源是两套。** §3 的 `filter.enabled=true` 已让 `GET:/api/hello` 自动生成 Web 资源，再标 `@SentinelResource("sayHello")` 会出现**簇点链路里两个资源保护同一段代码**——给"注解资源"配规则拦不住直接打 URL 的流量之外的语义差异（Web filter 资源在请求级、注解资源在方法级，嵌套时形成调用树父子，`2_architecture.md` §5）。起步期的纪律：**对外入口用 Web 资源名配规则，业务方法用注解资源**，别让两者管同一个粒度。

## 4.4 blockHandler 与 fallback 的分工表

| | blockHandler | fallback |
|--|--------------|----------|
| 触发 | `BlockException`（限流/熔断/系统保护/授权） | 业务异常（非 Block，默认任何异常） |
| 签名 | 原参数 + `BlockException`，返回一致 | 原参数（可选 `Throwable`），返回一致 |
| 位置 | 同类；或 `blockHandlerClass` 且方法 static | 同类；或 `fallbackClass` 且方法 static |
| 缺失时 | Block 异常**原样抛给调用方** | 异常原样抛出 |
| 收窄 | 不区分（想按类型分流用 `ex instanceof`） | `exceptionsToIgnore` 排除不降级的异常 |

两者同时存在时各管各的门：被限流的请求**不会**进 fallback（它没有业务异常），业务抛错的请求**不会**进 blockHandler——"降级了但控制台 block 指标没涨/涨了但没走降级"类困惑先对照本表定位走错了门。方法签名校验发生在**运行期反射查找**（找不到就按"缺失"行处理），没有编译期保护，签名写错是静默失效的，上线前对每条规则做一次超阈值演练（`25_interview_qa.md` 高频题）。

# 5. 配置流控规则

## 5.1 通过 Dashboard 配置（推荐）

1. 启动 Dashboard：`java -jar sentinel-dashboard.jar`
2. 访问 `http://localhost:8080`，默认账号/密码：`sentinel/sentinel`
3. 找到应用 → 簇点链路 → 选择资源 → 新增流控规则

## 5.2 通过代码配置

```java
@PostConstruct
public void initFlowRules() {
    List<FlowRule> rules = new ArrayList<>();

    FlowRule rule = new FlowRule();
    rule.setResource("sayHello");                    // 资源名
    rule.setGrade(RuleConstant.FLOW_GRADE_QPS);     // QPS 限流
    rule.setCount(10);                               // 阈值 10
    rule.setLimitApp("default");                     // 默认对所有调用方生效
    rules.add(rule);

    FlowRuleManager.loadRules(rules);
}
```

## 5.3 通过 Nacos 持久化

发送 POST 请求到 Nacos 创建配置：

```bash
curl -X POST "http://localhost:8848/nacos/v1/cs/configs" \
  -d "dataId=sentinel-demo-flow-rules" \
  -d "group=SENTINEL_GROUP" \
  -d "content=[
    {
      \"resource\": \"sayHello\",
      \"grade\": 1,
      \"count\": 10,
      \"limitApp\": \"default\"
    }
  ]"
```

## 5.4 三入口的优先级与覆盖规则

三条路最终都调用同一句 `FlowRuleManager.loadRules()`，**后执行者整表替换前一位**（不是按 resource 合并——这是最容易记错的语义）。时序上：`@PostConstruct` 的 loadRules 在启动早期执行 → 数据源初始化（`InitFunc`/`ReadableDataSource` 首次 getProperty + watch 回调）随后到达并**覆盖**代码规则 → 运行期 Dashboard 裸推再次覆盖、直到下一次数据源变更又把它冲掉。由此得出两条生产纪律：**代码硬编码只当"数据源读不到时的种子"看待，不要当并行规则源**；接了 Nacos 数据源后，控制台修改必须经"写回 Nacos"通道（默认不写回，`2_architecture.md` §6.1）。验证规则到底加载没加载，别看控制台——直接 `FlowRuleManager.getRules()` 打在日志或调试断点里，它是唯一事实源（控制台显示的可能是 Dashboard 自己缓存的视图，`9_dashboard.md`）。

# 6. 验证效果

```bash
# 正常请求
curl http://localhost:8080/api/hello?name=Test
# Hello, Test

# 高频请求（超过 10 QPS 阈值）
for i in {1..20}; do
  curl http://localhost:8080/api/hello &
done
# Too many requests! Please try later.   ← blockHandler 生效

# 查看实时监控
curl http://localhost:8719/metrics
```

## 6.1 压测方法的两个纠正

`for + curl &` 验证"超过 10 QPS"多半**压不出效果**：每个 curl 进程启动+建连耗时 10ms~秒级，20 个并发进程的真实到达速率远低于阈值，且 shell 后台任务分布极不均匀。快速验证用 `ab -n 100 -c 30 URL`（或 `wrk -t2 -c30 -d5s`），30 并发打本地服务轻松过 10 QPS。另外首次调用有资源懒创建（ClusterNode/DefaultNode 建立）和 JIT 预热，前几十毫秒的通过率不代表稳态——**验证动作看第二段流量**。`/metrics` 那行给出的是最近窗口的原始秒级数据（`18_statistics_engine.md` 格式），Dashboard 曲线就是它聚合出来的，控制台看不到曲线时这条 curl 能区分"客户端没数据"还是"Dashboard 拉不到"。

## 6.2 "配了规则没限住"排查顺序

1. **资源名对得上吗**——规则的 `resource` 与簇点链路里的名字逐字比对（`GET:` 前缀、context-path、大小写），错名限流是发生率第一的原因；
2. **origin/limitApp**——规则 `limitApp=default` 只匹配"默认来源"，Web filter 未配 origin parser 时来源为空，用 `other` 规则或 `default` 的语义在 `7_authority_rule.md` §4；
3. **阈值单位**——grade 0 是**并发线程数**不是 QPS，接口 RT 10ms 时线程数模式几乎限不住（`4_flow_control.md` §1.2）；
4. **是否被更靠前的拦截掩盖**——系统保护先于流控触发时异常类型是 `SystemBlockException`（`2_architecture.md` §3.3），你以为流控没生效其实是系统规则先动手；
5. **多实例**——规则只推到了部分机器（`14_dynamic_rules.md` §10 集群一致性）。

# 7. 关键注解速查

| 注解 | 作用 | 位置 |
|------|------|------|
| `@SentinelResource` | 声明资源、指定 blockHandler/fallback | 方法上 |
| `@SentinelRestTemplate` | 为 RestTemplate 添加 Sentinel 保护 | Bean 声明上 |
| `blockHandler` | 流控/降级时的处理方法 | `@SentinelResource` 属性 |
| `fallback` | 业务异常时的处理方法 | `@SentinelResource` 属性 |

## 7.1 @SentinelResource 全属性一览

速查表只列了高频四行，完整属性集（排障与写中间件时需要）：`resourceType`（资源分类，透传给规则匹配）、`entryType`（IN/OUT，标记入口或出口调用）、`blockHandler`/`blockHandlerClass`、`defaultBlockHandler`（兜底方法，签名参数更少时优先匹配专用、找不到落到 default）、`fallback`/`fallbackClass`/`defaultFallback`（default 版连原参数都可以不带）、`exceptionsToTrace`/`exceptionsToIgnore`（白/黑名单控制哪些异常计入 error 统计——熔断按异常比例决策时这两个属性就是你的投票器，`5_circuit_breaker.md`）。注意 `defaultFallback` 的优先级低于 `fallback`，且找不到精确签名时的退化顺序：专用 → default → 抛原异常，写公共降级方法（返回统一 `Result.fail("繁忙")`）时挂 default 属性即可全类共享。

# 8. 常见启动问题

| 问题 | 原因 | 解决 |
|------|------|------|
| Dashboard 看不到应用 | 未配置 `transport.port` | 添加 `spring.cloud.sentinel.transport.port=8719` |
| 规则不生效 | 资源未触发首次调用 | 先调用一次接口，Sentinel 默认懒加载 |
| 端口冲突 | 8719 已占用 | 更换 `transport.port` |
| 日志刷屏 | 默认打印拦截日志 | 配置 `csp.sentinel.log.dir` 指定目录 |

## 8.1 表的第一行归因要纠正

"Dashboard 看不到应用"的原因**不是没配 `transport.port`**——该配置有默认值 8719 且占用时自动递增，不配也能注册。真实高频原因按概率排序：① 没配 `transport.dashboard`（应用不知道向谁心跳）；② 懒加载未触发首次请求（与第二行同源，解法 `eager: true`，§3.2）；③ Dashboard→应用的 Command Center 端口反向不通（容器网络/防火墙，症状是"应用列表有、点进去曲线空白"）；④ 应用名/分组认知偏差——多环境共用一个 Dashboard 时在搜索框按确切 `spring.application.name` 找。第二行"规则不生效=未触发首次调用"同理要纠偏：懒加载只影响**控制台可见性**，规则本身在数据源回调时就已 load 进 Manager，首次请求进来立刻生效——"没限住"请按 §6.2 五步排查，不要等"先调一次"。第四行补充：刷屏的多为 `sentinel-record.log`（信息级）与块日志，除指定目录外可用 `-Dcsp.sentinel.log.output.type=console` 或调 `csp.sentinel.log.level` 控制量级；`csp.sentinel.log.use.pid` 多实例同机时防互相覆盖。

# 9. 起步之后

到这里你已具备一条完整链路：资源定义 → 规则加载 → 拦截验证。下一步按方向深入：规则语义全集 `4_flow_control.md`~`8_hotspot_param.md`；Spring Cloud 生态接线（Feign/RestTemplate/Gateway）`10`~`13`；生产化（持久化、集群、多语言控制台替代方案）`14`、`9`；机制层从样板篇 `15_slot_chain_source.md` 进，统计引擎 `18` 是其地基。常见事故与解法速查在 `24_troubleshooting.md`。
