# 1. 概述

Sentinel 通过 Dubbo Filter SPI 机制透明集成 Dubbo，对 Provider 和 Consumer 两侧均提供流量防护能力。

```text
Consumer                                Provider
┌──────────────────────┐               ┌──────────────────────┐
│  SentinelDubboConsumer│              │  SentinelDubboProvider│
│  Filter               │              │  Filter               │
│  ┌──────────────────┐ │   RPC Call   │  ┌──────────────────┐ │
│  │ 流量控制          │ │ ──────────→ │  │ 流量控制          │ │
│  │ 熔断降级          │ │ ←────────── │  │ 系统保护          │ │
│  │ 统计监控          │ │              │  │ 统计监控          │ │
│  └──────────────────┘ │              │  └──────────────────┘ │
└──────────────────────┘               └──────────────────────┘
```

## 1.1 两侧规则的归属与"哪一侧该配什么"

图里两侧格子画得对称，实际分工要按"谁保护谁"划：**Provider 侧**配流控（QPS/线程）、系统保护、授权/热点——拦"进来的流量"，资源名是接口方法签名；**Consumer 侧**配熔断降级——防"出去的依赖"，同一名资源在调用方进程里各有一套独立统计与规则（provider 的 ClusterNode 与每个 consumer 的 ClusterNode 互不相干，规则文件也是各应用的）。两侧都配流控会出经典怪象：consumer 侧 QPS 阈值 50、provider 侧 100，集群 3 个 consumer 实例各自 50 → provider 视角最高 150，provider 先炸——**容量类规则只应有一侧是权威**（通常 provider 自护），consumer 侧专注熔断与超时（§5）。

# 2. 依赖引入

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>

<!-- Sentinel Dubbo Adapter -->
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-sentinel-spring-cloud-starter</artifactId>
    <version>${dubbo.version}</version>
</dependency>
```

## 2.1 【纠偏】adapter 的权威坐标在 com.alibaba.csp

第二个依赖的坐标有误：Sentinel 官方 Dubbo 适配器的发布物是 **`com.alibaba.csp:sentinel-apache-dubbo3-adapter`**（Dubbo 3.x）或 `sentinel-apache-dubbo-adapter`（Dubbo 2.7，javax 包名系），由 Sentinel 侧版本线管理、不是 `${dubbo.version}`；`org.apache.dubbo` 下确有 dubbo-sentinel 集成模块但社区/Spring 生态的主流接法是前者（SCA 文档同款）。选错纪元的 adapter 症状是包名对不上（`adapter.dubbo` vs `adapter.dubbo3`）或 Filter 类加载不到；两代 adapter 与 Dubbo 的 jakarta/javax、Triple 协议兼容性要一起核对（`3_quick_start.md` §1.1 的版本矩阵思路）。**且注意**：`spring-cloud-starter-alibaba-sentinel` 不会自动带 Dubbo adapter，这个依赖必须显式加——"引了 starter 但 RPC 资源不出现在簇点链路"的第一归因。

# 3. Filter 注册机制

Sentinel 通过 Dubbo SPI 自动注册 Filter：

```text
META-INF/dubbo/org.apache.dubbo.rpc.Filter:

sentinel.dubbo.provider=com.alibaba.csp.sentinel.adapter.dubbo3.SentinelDubboProviderFilter
sentinel.dubbo.consumer=com.alibaba.csp.sentinel.adapter.dubbo3.SentinelDubboConsumerFilter
```

**Provider 端资源名**：`{interfaceName}:{methodName}({paramTypes})`

```java
// 资源名示例
com.example.UserService:getUser(Long)
com.example.OrderService:createOrder(Order, Long)
```

**Consumer 端资源名**：`{interfaceName}:{methodName}({paramTypes})`

## 3.1 注册≠激活：Consumer 侧要手动点名

SPI 文件让 Filter **可被发现**，是否**进链**由 `@Activate` 决定，两侧待遇完全不同：Provider Filter 是 `@Activate(group = PROVIDER)`——provider 侧所有服务自动生效；Consumer Filter 是 `@Activate(group = CONSUMER, value = "sentinel")`——**必须显式激活**：引用处 `@DubboReference(filter = "sentinel.dubbo.consumer")`（SPI key 以所用 adapter 版本 §3 文件为准）或全局 `dubbo.consumer.filter=sentinel.dubbo.consumer`，也可以给 URL 加 `sentinel=true` 参数（对应 value 语义）。不激活的原因与开销有关：consumer 侧每次出 RPC 都建 Entry/统计，官方选择让调用方显式承诺。**症状对照**：provider 有资源、consumer 配了熔断规则永不触发——十有八九是这里没点名，这是 Dubbo 集成排障率第一的坑。

## 3.2 资源名示例要加一层核对

示例格式方向正确（`接口全名:方法(参数列表)`），但**参数类型是全限定名**（`getUser(java.lang.Long)`，不是 `(Long)`），且重载/泛型擦除会体现在这一段——手拼规则名极易差一个字符后静默失效。可靠取法：**先真实调一次，从本机簇点链路（`curl :8719/metrics` 或 Dashboard）里复制现名**，与 §3.1 的激活检查一并做。origin 侧顺带说清：consumer Filter 会把调用方应用名（`project.name`）塞进隐式参数，provider 侧的授权规则/按来源限流自动拿到它（`7_authority_rule.md` §2.2 通道③的兑现处）——这是 Dubbo 比 Feign 省事的少数点之一。

# 4. Provider 端限流

## 4.1 配置方式

```yaml
spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080
      datasource:
        flow:
          nacos:
            server-addr: localhost:8848
            data-id: ${spring.application.name}-flow-rules
            group-id: SENTINEL_GROUP
            rule-type: flow
```

## 4.2 通过 Dashboard 配置

在 Dashboard 中针对资源名 `com.example.UserService:getUser(Long)` 配置流控规则。

## 4.3 通过 @DubboService 配置

```java
@DubboService(parameters = "sentinel.flow.rules=[{\"resource\":\"" +
    "com.example.UserService:getUser(Long)\",\"grade\":1,\"count\":100}]")
public class UserServiceImpl implements UserService { ... }
```

## 4.4 【勘误】§4.3 的"注解挂规则"机制不存在

`sentinel.flow.rules` 这个 parameters 键**不是 sentinel adapter 的能力**（形似 Dubbo 自带 limit 过滤器的 `parameters={"limit","100"}` 写法，疑似混淆而来）：adapter 只从 RuleManager/数据源消费规则，注解上挂 JSON 不会有任何效果且静默。Provider 限流的合法入口就三个——§4.1 数据源（生产推荐）、`FlowRuleManager.loadRules()`（代码，注意懒注册窗口 `9_dashboard.md` §4.4）、Dashboard 裸推（临时，重启丢）。Dubbo 原生 `limit` 过滤器与 Sentinel 流控**功能重叠、口径互盲**（各限各的、互相看不到对方拦了多少），同链路二选一，选了 Sentinel 就把原生 limit 关掉。

# 5. Consumer 端熔断

Consumer 端 Sentinel Filter 对 RPC 调用进行熔断降级：

```java
@DubboService
public class OrderServiceImpl implements OrderService {

    @DubboReference(
        timeout = 3000,
        retries = 0,  // 开启 Sentinel 熔断后建议关闭 Dubbo 自身重试
        parameters = "sentinel.enabled=true"
    )
    private UserService userService;

    @Override
    public OrderResult createOrder(Order order) {
        try {
            // RPC 调用，自动受 Sentinel Dubbo Consumer Filter 保护
            User user = userService.getUser(order.getUserId());
            // ... 业务逻辑
        } catch (RpcException e) {
            // 熔断或流控拒绝
            return OrderResult.fallback(order);
        }
    }
}
```

## 5.1 这段代码的三处校准

**① `parameters = "sentinel.enabled=true"` 同样是无此机制的写法**：激活 consumer filter 的合法写法见 §3.1（`filter` 属性点名），这个参数不会产生任何效果——示例里它与 §4.3 同族，都是"看着像配置其实不存在"。② **被拦异常的形态**：BlockException 会被包成 `SentinelRpcException`（RpcException 的一个 cause 携带 Block 类型的子类），所以 `catch (RpcException)` 接得住，但想区分"被熔断"与"网络故障"要解包看 cause——`e.getCause() instanceof DegradeException`；把它一律当"下游挂了"重试会造成 §5.2 的预算错觉。③ **fallback 的正路**不是 catch 手写：adapter 提供 `DubboAdapterGlobalConfig.setConsumerFallback(...)`（block 时返回什么由它统一决定，含降级值与异常语义），写 catch 的方案每个调用点都要重复，且漏一个点就是裸异常上抛——全局 fallback 与 Feign 的 `fallbackFactory` 是平行思路（Feign 系列 `16_fallback.md`）。

## 5.2 retries=0 的理由展开

Dubbo consumer 默认 `retries=2`（连同首次共 3 次尝试，且**故障切换会换 provider**）。它与 Sentinel 的交互是三笔账：熔断统计上，一次业务调用最多产生 3 个 Entry 事件，下游过载时"重试打到别的 provider"把故障面扩大——这正是熔断要防的事，Sentinel 在第一次尝试被拦后不会重投，重试语义完全归 Dubbo；预算上，`timeout × (retries+1)` 是该调用点的真实最坏耗时，上层任何 SLA 都要按这个乘积核对（Feign 系列 `9_retry.md` §9.1 同款不等式）；幂等上，写接口带 retries 的重复提交事故与 Sentinel 无关但同场加演。**方向**：要么 `retries=0` 让 Sentinel 的熔断+快速失败做主，要么保留 Dubbo 重试就别在 consumer 侧配熔断（统计会被重试稀释）——两边容错机制各留一层。

# 6. 嵌套调用与 Context

Dubbo 调用链中的 Context 传递：

```java
// Consumer 端
ContextUtil.enter("order-context", "order-service");
try (Entry entry = SphU.entry("orderEntry")) {
    // 调用 UserService（Sentinel Dubbo Filter 自动创建嵌套 Entry）
    User user = userService.getUser(userId);
}
ContextUtil.exit();
```

**注意**：Dubbo Filter 中的 Entry 会在 Filter 调用链中自动管理，不需要手动 release。

## 6.1 provider 侧 Context 的自动化与一处遗留手工

Provider Filter 对每次调用自动 `ContextUtil.enter(接口名, 调用方app名)` 并在 invoke 收尾 exit——所以 **provider 侧的链路模式（chain）规则的 refResource 是接口方法签名而非入口 URL**，与 Web 场景的语境完全不同（`4_flow_control.md` §3.4 的匹配对象差异在 RPC 上的对应形态）。示例的手工 enter 属于 consumer 侧业务想要"把多次 RPC 归到一个上下文"才需要；不加时 consumer Filter 用的是默认 context，嵌套关系照样成立（每个 RPC 一个 Entry，挂在当前线程上下文）。Dubbo 线程模型要留意：provider 业务线程执行 Filter 收尾，**异步 Triple/网关转发场景下 Context 与 exit 的配对按线程走**（`2_architecture.md` §5.2 的纪律在 RPC 同样适用）；"不需要手动 release"这句对 Filter 内建的 Entry 成立，对 §6 示例手工建的 Entry 不成立——try-with-resources 保持。

# 7. Filter 执行顺序

```java
// Consumer 端 Filter 链
ConsumerContextFilter → SentinelDubboConsumerFilter → ... → Transport

// Provider 端 Filter 链
Transport → ... → SentinelDubboProviderFilter → Business Method
```

Sentinel Filter 应尽可能靠近业务方法，确保其他 Filter（如 Context、Trace）在 Sentinel 判断之前执行。

## 7.1 【勘误】实现是"尽可能靠外"，理由也应是尽早拒绝

两个 sentinel Filter 的 `@Activate(order = -10000)` 把它们钉在链的**前段**（order 越小越靠外/越先执行）：provider 侧收到请求先问"该不该拦"，再走后续 Filter 和业务；consumer 侧出站先问熔断器"配不配调"。末句"应尽可能靠近业务方法、其他 Filter 先执行"与实现相反。设计理由也值得记：拦截的价值在**省掉后续一切开销**（序列化、路由、线程占用），放在尾部等于"全付了钱再退票"。唯一需要留意的是靠外带来的统计口径：被 Sentinel 拒的调用不会经过 metrics/trace 类靠内 Filter，链路追踪里看不到这些"被拦请求"——对账"网关侧请求数 vs provider 业务日志数"的差值就是拦截量（也是 `9_dashboard.md` §4.4 block 指标单独成列的意义）。若确需调整顺序（让某认证 Filter 先跑），用自定义 Filter 包一层，改 adapter 的 order 常量不如加显式链声明。

# 8. 配置示例

```yaml
dubbo:
  application:
    name: order-service
  registry:
    address: nacos://localhost:8848

spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080
        port: 8719
      eager: true
      datasource:
        flow:
          nacos:
            server-addr: localhost:8848
            data-id: ${spring.application.name}-flow-rules
            group-id: SENTINEL_GROUP
            rule-type: flow
        degrade:
          nacos:
            server-addr: localhost:8848
            data-id: ${spring.application.name}-degrade-rules
            group-id: SENTINEL_GROUP
            rule-type: degrade
```

## 8.1 这份配置的生效核对清单

RPC 场景比 Web 多三个核对点：**① adapter 依赖在位**（§2.1，`sentinel-apache-dubbo3-adapter` 进 classpath）；**② consumer 侧 filter 已点名**（§3.1，本 yaml 没有 dubbo.consumer.filter 配置——这份文件属于 provider 应用，consumer 角色的应用还要加）；**③ 资源名从真实簇点链路复制**（§3.2，规则 JSON 里的 resource 与签名全限定名逐字符一致）。RPC 应用同样吃 `eager: true`（Dubbo provider 启动即有注册心跳，但簇点链路资源要等首调用——lazy 语义与 Web 一致）。`dubbo.application.name` 与 `spring.application.name` 建议同值，否则 Dashboard 应用列表与 RPC origin 出现两个名字，来源类规则按后者匹配。

# 9. 最佳实践

1. **Dubbo retries=0**：启用 Sentinel 熔断后，关闭 Dubbo 自身的重试机制
2. **资源粒度**：接口 + 方法粒度即可，过细会带来管理负担
3. **Provider 侧流控**：保护 Provider 不被 Consumer 海量请求击垮
4. **Consumer 侧熔断**：保护 Consumer 线程不被慢 Provider 阻塞
5. **异常透传**：`BlockException` 会自动转换为 `RpcException`，Consumer 可捕获处理

## 9.1 逐条注脚

**1** 的完整论证在 §5.2，执行时用 `retries=0`（引用级）而不是全局 0——纯读接口保留 1 次故障切换重试也合理，但要明白那会改变熔断统计分母（§1.1 权威一侧配容量）。**2** 补上"过细"的具体形态：想按方法参数再拆资源（RPC 版热点）应走 `@SentinelResource` 手工埋点或热点参数规则，而不是给 Filter 资源改名——Filter 资源名是自动生成的，改名即失联。**3/4** 与 §1.1 同一条纪律的两半，评审时用"每个资源名只在一侧有容量规则"检查。**5** 的精确形态是 `SentinelRpcException(cause=BlockException)`（§5.1 ②），且推荐用全局 ConsumerFallback 替代逐点 catch（§5.1 ③）；最后补一条 Dubbo 特有的：**Triple/新协议与网关转发场景**（HTTP 转 RPC）在网关侧已有一层 Sentinel 资源时，provider 侧规则阈值应把"网关层已拦掉的量"排除在外，两层容量规则各管各层、绝不互相引用对方的数字。

# 10. 相关阅读

origin 与授权体系在 `7_authority_rule.md`；熔断策略与状态机在 `5_circuit_breaker.md`（源码层 `17_circuit_breaker_source.md`）；与 Feign/Web 集成对照在 `13_feign_integration.md`、`10_spring_cloud_integration.md`；集群 token server 在 RPC 场景同样适用（`9_dashboard.md` §6.2）。
