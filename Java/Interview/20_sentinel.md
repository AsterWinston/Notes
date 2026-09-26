# 1. 知识点
- **核心功能**：流量控制（流控）、熔断降级、系统自适应保护、热点参数限流、网关流量控制、集群流控。
- **工作流程概述**：Sentinel拦截资源调用 → 经过Slot责任链处理 → 每个Slot检查对应的规则 → 通过则放行，拒绝则抛出BlockException或执行降级逻辑。
- **资源定义**：资源是Sentinel中的核心概念，可以是Java方法、HTTP接口、代码块，通过`SphU.entry("资源名")`或`@SentinelResource`定义。
- **Slot责任链**：Sentinel核心处理框架，每个Slot负责一种规则检查（如流控、降级、系统规则、统计等），链表式串联执行。
- **ProcessorSlotChain**：Slot责任链的构建器，由`SlotChainBuilder`创建，包含NodeSelectorSlot、ClusterBuilderSlot、StatisticSlot、FlowSlot、DegradeSlot、SystemSlot、AuthoritySlot等。
- **NodeSelectorSlot**：构建资源调用树，为每个资源创建不同的DefaultNode，记录资源的调用链路。
- **ClusterBuilderSlot**：构建资源的集群节点ClusterNode，同一个资源所有链路的统计汇总到ClusterNode。
- **StatisticSlot**：统计核心，记录调用结果（成功/失败/耗时等），为后续Slot的规则判断提供数据基础。
- **FlowSlot**：流量控制Slot，根据流控规则判断是否放行请求。
- **DegradeSlot**：熔断降级Slot，检查资源是否处于熔断状态。
- **SystemSlot**：系统规则Slot，根据系统整体负载判断是否限流。
- **滑动时间窗口算法**：Sentinel使用滑动窗口做指标统计，将一个时间窗口（如1秒）划分为多个小的采样窗口，统计精度更高。
- **LeapArray环形数组**：核心数据结构，使用环形数组存储窗口数据，固定数组长度，通过时间戳定位窗口位置实现滑动。
- **窗口长度（windowLengthInMs）**：每个采样窗口的时间跨度，默认500ms（1秒划分为2个窗口，SampleCountProperty默认为2）。
- **采样计数（SampleCount）**：时间窗口被划分为多少个采样窗口，默认2，即每秒2个窗口，每500ms一个。
- **流控规则（FlowRule）**：定义资源的流量控制策略，包括阈值类型、流控模式、流控效果。
- **阈值类型-QPS**：每秒查询数，限制每秒最大请求数量，超过阈值时拒绝新请求。
- **阈值类型-并发线程数**：限制同时处理请求的线程数，适用于需隔离慢调用的场景。
- **流控模式-直接**：对当前资源直接限流，不关心其他资源。
- **流控模式-关联**：当关联资源达到阈值时，限流当前资源，如支付接口关联下单接口，支付压力大时限流下单。
- **流控模式-链路**：只针对指定入口链路的调用进行限流（需要SphU.entry("入口")提前标记入口）。
- **流控效果-快速失败**：流量超阈值时直接抛出FlowException，拒绝请求（默认方式）。
- **流控效果-Warm Up（预热/冷启动）**：系统启动时让流量缓慢增长到阈值，防止冷系统被瞬间流量冲垮，基于令牌桶算法。
- **流控效果-匀速排队**：使用漏桶算法让请求以均匀速度通过，适用于消息队列、削峰填谷，需设置超时时间。
- **熔断降级-慢调用比例**：当资源的慢调用比例（响应时间超过设定RT的请求占比）超过阈值时触发熔断。
- **熔断降级-异常比例**：当请求异常比例超过阈值时触发熔断。
- **熔断降级-异常数**：当1分钟内异常数量超过阈值时触发熔断。
- **断路器状态机**：OPEN（熔断打开，拒绝请求）→ HALF_OPEN（半开，允许探测请求通过）→ CLOSED（熔断关闭，恢复正常）或回退到OPEN。
- **半开探测**：熔断时间窗口过后进入半开状态，允许少量探测请求（通常是1-5个），若成功则CLOSED恢复，若失败则重新OPEN等待下一时间窗口。
- **系统规则（SystemRule）**：保护应用整体，当系统指标超过阈值时对所有入口流量限流。
- **系统规则-LOAD**：系统负载（load1），仅对Linux/Unix有效。
- **系统规则-RT**：所有入口接口的平均响应时间（毫秒）。
- **系统规则-线程数**：所有入口流量的并发线程数。
- **系统规则-入口QPS**：所有入口流量每秒请求数。
- **系统规则-CPU使用率**：系统CPU使用率（%）。
- **热点规则（ParamFlowRule）**：针对特定的方法参数值设置不同的限流阈值，如针对热门商品ID单独限制QPS。
- **ParamFlowItem**：热点参数的特例配置，为特定参数值指定单独的限流阈值。
- **@SentinelResource注解**：声明式定义Sentinel资源，配置资源名、blockHandler、fallback等属性，简化资源定义。
- **blockHandler**：处理Sentinel异常（BlockException），如流控、降级、热点参数等被拒时的降级逻辑，不处理业务异常。
- **fallback**：处理业务异常（Throwable），当方法执行抛出非BlockException异常时执行fallback逻辑。
- **blockHandlerClass/fallbackClass**：将降级逻辑提取到独立工具类中，通过class属性引用。
- **defaultFallback**：通用fallback方法，当没有指定具体的fallback时使用，方法签名可以是`defaultFallback()`或`defaultFallback(Throwable)`。
- **规则持久化-默认模式**：规则存储在JVM内存中，应用重启后规则丢失，适用于开发测试环境。
- **规则持久化-推模式**：通过Sentinel控制台将规则推送到Nacos/Apollo等配置中心，应用从配置中心拉取规则，控制台修改→推送Nacos→应用实时更新。
- **规则持久化-拉模式**：应用从本地文件或配置中心定时拉取规则，支持动态更新但时效性弱于推模式。
- **Sentinel vs Hystrix对比**：Sentinel采用信号量隔离（轻量），Hystrix默认线程池隔离；Sentinel规则更丰富（热点、系统规则）；Sentinel有可视化控制台；Sentinel活跃维护而Hystrix已停更。
- **隔离机制差异**：线程池隔离资源占用大但隔离彻底，信号量隔离轻量但存在线程阻塞风险，Sentinel可选信号量或线程池。
- **Sentinel Dashboard**：可视化控制台，支持实时监控（秒级QPS、响应时间、线程数）、规则配置（流控/降级/热点/系统/授权）、集群流控管理等。
- **实时监控**：Dashboard展示秒级粒度的监控数据，包括通过QPS、拒绝QPS、响应时间等，支持多维度筛选。
- **集群流控**：通过Token Server统一管理全局流量配额，Token Client向Token Server请求Token，实现分布式环境下的精准流控。
- **与OpenFeign整合**：`feign.sentinel.enabled=true`后，每个Feign方法自动成为Sentinel资源，可在控制台配置流控/降级规则。
- **与Gateway整合**：通过`sentinel-spring-cloud-gateway-adapter`将Gateway路由接入Sentinel管控，支持API分组和网关流控规则。
- **授权规则**：基于调用来源的黑白名单控制（AuthorityRule），通过`RequestOriginParser`解析调用来源。
- **限流、降级和熔断的区别**: 限流(Rate Limiting) — 限制访问频率，超出阈值的请求被拒绝或排队; 熔断(Circuit Breaking) — 服务不可用时暂时切断对该服务的调用，快速失败; 降级(Degradation) — 服务压力大时降低服务质量(返回默认值/缓存/简化功能)，保护核心功能
- **Sentinel限流规则持久化**: 默认Dashboard配置的规则重启后丢失。解决方案: 将流控/降级规则持久化到Nacos配置中心，应用启动时从Nacos加载规则，Dashboard修改后同步推送到Nacos

# 2. 面试题

## 2.1 Sentinel 的核心功能有哪些？
1. **流量控制（Flow Control）**：控制资源的QPS或并发线程数，支持多种流控模式（直接/关联/链路）和效果（快速失败/Warm Up/匀速排队）
2. **熔断降级（Circuit Breaking）**：基于慢调用比例、异常比例、异常数触发熔断，断路器状态自动切换（OPEN→HALF_OPEN→CLOSED）
3. **热点参数限流（Hotspot Param Flow）**：针对特定参数值（如商品ID）设置不同的限流阈值，防止"热点"请求集中
4. **系统自适应保护（System Rules）**：根据系统整体负载（LOAD、RT、线程数、CPU、QPS）自适应限流，保护系统不崩溃
5. **网关流量控制（Gateway Flow Control）**：针对Gateway的路由/API组进行流量控制
6. **集群流控（Cluster Flow Control）**：分布式环境下的全局精确流量控制，通过Token Server统一配额
7. **授权规则**：基于调用来源的黑白名单控制

## 2.2 Sentinel 的工作流程和 Slot 责任链是怎样的？
**整体流程**：
1. 业务代码调用`SphU.entry("资源名")`定义受保护资源
2. Sentinel核心加载该资源对应的ProcessorSlotChain
3. 依次执行Slot链中的每个Slot，检查规则
4. 所有Slot通过 → 正常执行 → `entry.exit()`统计结果
5. 某个Slot拒绝 → 抛出`BlockException` → 执行降级逻辑

**Slot责任链**（按执行顺序）：
| Slot | 功能 |
|------|------|
| NodeSelectorSlot | 构建资源调用树，记录调用路径 |
| ClusterBuilderSlot | 构建ClusterNode，汇总同一资源所有链路数据 |
| LogSlot | 记录BlockException日志 |
| StatisticSlot | **核心Slot**，统计调用信息（成功/失败/RT） |
| AuthoritySlot | 授权规则（黑白名单）检查 |
| SystemSlot | 系统规则检查 |
| FlowSlot | 流控规则检查 |
| DegradeSlot | 熔断降级规则检查 |
| ParamFlowSlot | 热点参数规则检查 |

每个Slot处理后就调用`fireEntry(context, resourceWrapper, next, count, args)`传递给下一个Slot。

## 2.3 Sentinel 的滑动时间窗口统计是如何实现的？
Sentinel使用**LeapArray（跳跃数组）**实现滑动窗口统计，核心设计如下：

**数据结构**：
- 环形数组：`WindowWrap<T>[] array`，长度固定为`sampleCount`
- 数组长度 = `intervalInMs` / `windowLengthInMs`（默认 1000ms / 500ms = 2）
- 每个`WindowWrap`包含：窗口起始时间戳、窗口统计值（MetricBucket）

**滑动机制**：
1. 根据当前时间戳计算目标窗口索引：`idx = (time / windowLength) % array.length`
2. 检查当前索引位置的窗口是否在有效期内：
   - 若窗口起始时间等于预期 → 直接返回（刷新窗口数据）
   - 若窗口已过期 → 重置窗口（清空数据，更新起始时间）
   - 若时间跨度过大 → 重置整个数组
3. 在目标窗口的MetricBucket中累计统计数据（通过QPS、异常数、RT等）

**MetricBucket内部**：
- 使用LongAdder记录统计指标
- 指标包括：PASS（通过）、BLOCK（阻塞）、SUCCESS（成功）、EXCEPTION（异常）、RT（响应时间，存储在数组中）

**优点**：相比固定时间窗口，滑动窗口解决了临界突变问题，统计更平滑精确；环形数组复用内存，避免频繁创建销毁。

## 2.4 @SentinelResource 注解如何定义降级逻辑？
```java
@Service
public class OrderService {

    // 方式1：blockHandler + fallback 分离
    @SentinelResource(
        value = "getOrder",
        blockHandler = "handleBlock",      // 处理流控/降级
        fallback = "handleFallback",        // 处理业务异常
        fallbackClass = OrderFallbackHandler.class  // fallback所在类
    )
    public Order getOrder(Long orderId) {
        // 业务逻辑
        if (orderId == null) throw new IllegalArgumentException("orderId不能为空");
        return orderMapper.selectById(orderId);
    }

    // blockHandler方法签名要求：
    //   返回值类型一致 + 参数列表(原参数 + BlockException)
    public Order handleBlock(Long orderId, BlockException ex) {
        log.warn("订单查询被限流 orderId={}", orderId, ex);
        return new Order(); // 返回兜底数据
    }
}

// 方式2：外部工具类（fallbackClass）
public class OrderFallbackHandler {
    // 必须是static方法
    public static Order handleFallback(Long orderId, Throwable ex) {
        log.error("订单查询异常 orderId={}", orderId, ex);
        return new Order();
    }
}
```

**注意事项**：
- `blockHandler`只处理`BlockException`及其子类（FlowException、DegradeException、ParamFlowException等）
- `fallback`处理所有异常（除ExceptionsToIgnore配置的异常外）
- `blockHandlerClass`/`fallbackClass`中的方法必须是`public static`
- 降级方法参数列表要与原方法一致（或去掉参数），末尾可加BlockException/Throwable

## 2.5 Sentinel 如何实现规则持久化？
Sentinel默认将规则存储在内存中，应用重启丢失。持久化方案如下：

**方案一：推模式（推荐）**
使用Sentinel Dashboard → 配置中心推送

工作流程：
1. Sentienl Dashboard配置规则后，调用配置中心API将规则写入Nacos/Apollo等
2. 应用启动时从配置中心读取规则到Sentinel内存
3. Dashboard修改规则 → 推送到配置中心 → 配置中心通知应用 → 应用实时更新规则

配置示例：
```yaml
spring:
  cloud:
    sentinel:
      datasource:
        flow:
          nacos:
            server-addr: localhost:8848
            data-id: ${spring.application.name}-flow-rules
            group-id: DEFAULT_GROUP
            data-type: json
            rule-type: flow
```

**方案二：拉模式（非实时）**
```yaml
spring:
  cloud:
    sentinel:
      datasource:
        flow:
          file:
            file: classpath:sentinel-rules.json
            rule-type: flow
```
应用定时从文件读取规则并更新内存。

**方案三：数据库持久化**
自定义DataSource，从数据库加载规则，通过定时轮询或推拉结合方式实现动态更新。

## 2.6 Sentinel 与 Hystrix 的详细对比
| 维度 | Sentinel | Hystrix |
|------|----------|---------|
| 隔离策略 | 信号量（默认），也可线程池 | 线程池（默认）|
| 资源开销 | 低（信号量基于计数器） | 高（线程池上下文切换） |
| 熔断策略 | 慢调用比例/异常比例/异常数 | 异常比例（仅一种） |
| 流控效果 | 快速失败/Warm Up/匀速排队 | 仅快速失败 |
| 系统保护 | 支持（LOAD/RT/CPU等） | 不支持 |
| 热点限流 | 支持 | 不支持 |
| 实时监控 | 秒级Dashboard | 较粗粒度（Hystrix Dashboard） |
| 控制台 | 功能丰富、活跃维护 | 简单、已停更 |
| 规则管理 | 控制台动态配置 | 代码配置（需重启） |
| 规则持久化 | 支持多种持久化 | 不支持 |
| 集群流控 | 支持 | 不支持 |
| 网关限流 | 原生支持Gateway | 仅Zuul 1.x |
| 维护状态 | 活跃（阿里巴巴） | 停止维护 |

## 2.7 Sentinel 的熔断降级三种策略分别如何工作？
**1. 慢调用比例（SLOW_REQUEST_RATIO）**：
- 触发条件：`count`（最大RT，毫秒）内，慢调用比例超过`slowRatioThreshold`
- 如：RT>500ms的请求占比超过50%时触发
- 适用于：依赖服务变慢但不报错的场景

**2. 异常比例（ERROR_RATIO）**：
- 触发条件：异常请求占总请求比例超过`count`
- 如：异常比例超过0.5时（50%）
- 适用于：依赖服务偶发错误的场景

**3. 异常数（ERROR_COUNT）**：
- 触发条件：1分钟内异常总数超过`count`
- 如：1分钟内异常数超过10次时触发
- 适用于：依赖服务持续错误的场景

**断路器状态转换**：
```
CLOSED（正常）→ 触发条件满足 → OPEN（拒绝请求）
  ↑                                     ↓
  └── 探测全部成功 ← HALF_OPEN ← 熔断时间窗口到期
       (或探测失败 → 继续OPEN)
```

## 2.8 如何将 Sentinel 与 OpenFeign 整合？
```yaml
# application.yml
feign:
  sentinel:
    enabled: true  # 开启Sentinel对Feign的集成

spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080
```

```java
// Feign接口定义
@FeignClient(
    name = "order-service",
    fallback = OrderClientFallback.class,           // 降级处理类
    fallbackFactory = OrderClientFallbackFactory.class // 可获取异常原因
)
public interface OrderClient {
    @GetMapping("/orders/{id}")
    Order getOrder(@PathVariable("id") Long id);
}

// Fallback实现
@Component
public class OrderClientFallback implements OrderClient {
    @Override
    public Order getOrder(Long id) {
        // Sentinel触发降级或流控时执行此方法
        return new Order();
    }
}

// FallbackFactory实现（推荐，可获取异常信息）
@Component
public class OrderClientFallbackFactory implements FallbackFactory<OrderClient> {
    @Override
    public OrderClient create(Throwable cause) {
        log.error("order-service调用失败", cause);
        return id -> new Order();
    }
}
```

**原理**：
- Feign客户端被Sentinel包装为Sentinel资源
- 资源名格式：`GET:http://order-service/orders/{id}`
- 可在Sentinel Dashboard中看到该资源并配置流控/降级规则
- 触发规则时执行Fallback逻辑

## 2.9 Sentinel 的集群流控是如何工作的？
**核心概念**：
- **Token Server**：负责管理全局Token配额的服务端，运行在独立模式或内嵌模式
- **Token Client**：需要申请Token的客户端，即业务应用

**工作流程**：
1. Token Client接收到请求，向Token Server请求token
2. Token Server根据集群流控规则判断是否发放token
3. 若token发放成功，请求正常通过；若发放失败，执行限流逻辑

**两种模式**：
- **独立模式（Alone）**：Token Server作为独立进程运行，所有Token Client向其请求
- **内嵌模式（Embedded）**：集群中选举一台机器作为Token Server，同时兼顾业务和token发放

**配置步骤**：
1. 配置Token Server地址
2. 在Sentinel Dashboard中添加集群流控规则
3. Dashboard将规则推送到Nacos/配置中心
4. Token Server和Client从配置中心加载规则

**优缺点**：
- 优点：实现精确的全局流量控制，解决单机限流不准问题
- 缺点：引入额外复杂度，Token Server单点问题（需集群部署）

## 2.10 限流、降级和熔断有什么区别？
- **限流（Rate Limiting）**：限制访问频率，超出阈值的请求被拒绝或排队等待。目的是保护系统不被流量冲垮，属于"预防"机制。如Sentinel基于QPS/并发线程数限流。
- **熔断（Circuit Breaking）**：当依赖服务不可用或响应过慢时，暂时切断对该服务的调用，快速失败并执行降级逻辑。目的是防止故障扩散（级联故障/雪崩），属于"止损"机制。如Sentinel基于慢调用比例/异常比例/异常数自动熔断。
- **降级（Degradation）**：当系统压力大或依赖服务不可用时，降低服务质量，返回默认值/缓存数据/简化功能，保证核心功能可用。属于"妥协"机制。如Feign Fallback返回兜底数据。

三者关系：限流在入口处控制流量 → 超出系统能力时执行降级 → 下游服务不可用时触发熔断并降级。

## 2.11 Sentinel限流规则怎么进行持久化？
Sentinel默认将规则存储在JVM内存中，应用/Dashboard重启后规则丢失。持久化方案：

**推荐方案：推模式（Nacos持久化）**
1. 引入依赖：`sentinel-datasource-nacos`
2. 配置数据源：
```yaml
spring:
  cloud:
    sentinel:
      datasource:
        flow:
          nacos:
            server-addr: localhost:8848
            data-id: ${spring.application.name}-flow-rules
            group-id: DEFAULT_GROUP
            data-type: json
            rule-type: flow
```
3. 工作原理：应用启动时从Nacos拉取规则 → 加载到Sentinel内存 → Dashboard修改规则后调用Nacos API推送 → Nacos通知应用实时更新规则

**其他方案**：
- **拉模式**：应用定时从本地文件/配置中心拉取规则，时效性弱
- **数据库持久化**：自定义DataSource从DB加载规则，定时轮询或推拉结合
