
# 1. 概述

Sentinel 的 SPI（Service Provider Interface）机制是框架可扩展性的基础。通过 SPI，开发者可以自定义 SlotChain、数据源、Transport、日志等核心模块。

# 2. Sentinel SPI vs Java SPI

| 特性 | Java SPI | Sentinel SPI |
|------|---------|-------------|
| 配置文件位置 | `META-INF/services/` | `META-INF/services/`（兼容） |
| 全量加载 | 是 | 否（按需加载） |
| 优先级排序 | 不支持 | `@Spi(order = -100)` 支持 |
| 默认实现 | 不支持 | `@Spi(isDefault = true)` 支持 |
| 单例/多例 | 多例 | 单例（默认） |

# 3. SPI 核心类

```java
@Component
@Spi(isDefault = true, order = Constants.ORDER_DEFAULT)
public class DefaultSlotChainBuilder implements SlotChainBuilder { ... }

// SpiLoader 加载 SPI
public class SpiLoader {

    public static <T> T loadFirstInstanceOrDefault(Class<T> clazz) {
        List<T> instances = load(clazz);
        if (instances.isEmpty()) {
            return null;
        }
        // 返回第一个（已按 order 排序）
        return instances.get(0);
    }

    public static <T> List<T> load(Class<T> clazz) {
        List<SpiWrapper<T>> wrappers = new ArrayList<>();
        
        // 扫描 META-INF/services/{interface_name}
        // 读取实现类名 → 反射实例化
        ServiceLoader<T> loader = ServiceLoader.load(clazz);
        for (T instance : loader) {
            Spi spi = instance.getClass().getAnnotation(Spi.class);
            int order = (spi != null) ? spi.order() : 0;
            wrappers.add(new SpiWrapper<>(instance, order));
        }

        // 按 order 排序
        wrappers.sort(Comparator.comparingInt(SpiWrapper::getOrder));
        return wrappers.stream().map(SpiWrapper::getInstance).collect(toList());
    }
}
```

# 4. 内置 SPI 扩展点

| SPI 接口 | 说明 | 默认实现 |
|---------|------|---------|
| `SlotChainBuilder` | 构建 SlotChain | `DefaultSlotChainBuilder` |
| `InitFunc` | 初始化回调 | — |
| `DataSource` | 规则数据源 | Nacos/Apollo/ZK |
| `CommandCenter` | Dashboard 通信 | `NettyHttpCommandCenter` |
| `HeartbeatSender` | 心跳发送 | `SimpleHttpHeartbeatSender` |
| `TransportObserver` | 网络传输监控 | — |

# 5. InitFunc — 初始化扩展

```java
// 自定义初始化逻辑
@Spi(order = -100)  // 高优先级
public class MyInitFunc implements InitFunc {

    @Override
    public void init() throws Exception {
        // 1. 设置全局配置
        SentinelConfig.setConfig(Constants.USER_ORIGIN, "my-service");

        // 2. 注册自定义规则
        initFlowRules();

        // 3. 注册自定义 SlotChainBuilder（如果未通过 SPI 注册）
        SlotChainProvider.setBuilder(new MySlotChainBuilder());

        log.info("Custom InitFunc initialized");
    }

    private void initFlowRules() {
        List<FlowRule> rules = new ArrayList<>();
        FlowRule rule = new FlowRule("default");
        rule.setCount(1000);  // 默认全局限流
        rules.add(rule);
        FlowRuleManager.loadRules(rules);
    }
}

// META-INF/services/com.alibaba.csp.sentinel.init.InitFunc
// com.example.MyInitFunc
```

# 6. 自定义 ProcessorSlot

```java
// 1. 定义自定义 Slot
public class TracingSlot extends AbstractLinkedProcessorSlot<DefaultNode> {

    private static final Tracer tracer = Tracing.getTracer();

    @Override
    public void entry(Context context, ResourceWrapper resourceWrapper,
                      DefaultNode node, int count, boolean prioritized,
                      Object... args) throws Throwable {
        // 在 SlotChain 中创建 Tracing Span
        Span span = tracer.spanBuilder(resourceWrapper.getName())
            .setAttribute("entryType", resourceWrapper.getEntryType().name())
            .startSpan();

        try (Scope scope = span.makeCurrent()) {
            // 添加当前 Entry 信息到 Span
            span.setAttribute("threadCount", node.curThreadNum());
            span.setAttribute("passQps", node.passQps());

            // 继续执行后续 Slot
            fireEntry(context, resourceWrapper, node, count, prioritized, args);
        } catch (Throwable t) {
            span.setStatus(StatusCode.ERROR);
            span.recordException(t);
            throw t;
        } finally {
            span.end();
        }
    }
}

// 2. 自定义 SlotChainBuilder
@Spi(order = -200)
public class TracingSlotChainBuilder implements SlotChainBuilder {

    @Override
    public ProcessorSlotChain build() {
        ProcessorSlotChain chain = new DefaultProcessorSlotChain();
        chain.addLast(new NodeSelectorSlot());
        chain.addLast(new ClusterBuilderSlot());
        chain.addLast(new TracingSlot());         // ← 插入自定义 Slot
        chain.addLast(new LogSlot());
        chain.addLast(new StatisticSlot());
        chain.addLast(new SystemSlot());
        chain.addLast(new AuthoritySlot());
        chain.addLast(new FlowSlot());
        chain.addLast(new DegradeSlot());
        return chain;
    }
}
```

# 7. 自定义 Transport

```java
// 实现自定义心跳发送
@Component
public class CustomHeartbeatSender implements HeartbeatSender {

    private final RestTemplate restTemplate = new RestTemplate();

    @Override
    public boolean sendHeartbeat() throws Exception {
        HeartbeatMessage message = new HeartbeatMessage();
        message.setApp(SentinelConfig.getAppName());
        message.setPort(TransportConfig.getPort());
        message.setHostname(HostNameUtil.getHostname());

        // 发送到自定义的监控平台
        ResponseEntity<String> response = restTemplate.postForEntity(
            "http://my-monitor/api/heartbeat",
            message,
            String.class
        );
        return response.getStatusCode().is2xxSuccessful();
    }

    @Override
    public long intervalMs() {
        return 5000;  // 5s 心跳间隔
    }
}
```

# 8. 自定义过滤器扩展

```java
// Dubbo Filter 扩展
@Activate(group = {CommonConstants.PROVIDER, CommonConstants.CONSUMER})
public class CustomDubboFilter implements Filter {

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation)
            throws RpcException {
        String resourceName = invoker.getInterface().getName()
            + ":" + invocation.getMethodName();

        ContextUtil.enter(resourceName, invoker.getUrl()
            .getParameter("application", "unknown"));

        try (Entry entry = SphU.entry(resourceName,
                EntryType.OUT, 1, invocation.getArguments())) {
            return invoker.invoke(invocation);
        } catch (BlockException e) {
            throw new RpcException(RpcException.FORBIDDEN_EXCEPTION,
                "Blocked by Sentinel: " + resourceName);
        } finally {
            ContextUtil.exit();
        }
    }
}
```

# 9. 规则监听扩展

```java
@Component
public class RuleChangeListener implements InitFunc {

    @Override
    public void init() {
        // 监听流控规则变更
        FlowRuleManager.getProperty().addListener(
            new PropertyListener<List<FlowRule>>() {
                @Override
                public void configLoad(List<FlowRule> rules) {
                    log.info("Flow rules loaded: {} rules", rules.size());
                }

                @Override
                public void configUpdate(List<FlowRule> rules) {
                    log.warn("Flow rules UPDATED: {} rules", rules.size());
                    // 发送通知、记录审计日志等
                    notifyAdmin(rules);
                }
            });
    }

    private void notifyAdmin(List<FlowRule> rules) {
        // 规则变更通知
    }
}
```

# 10. 最佳实践

1. **优先 SPI**：优先使用 SPI 扩展，避免修改 Sentinel 源码
2. **InitFunc 初始化**：统一在 InitFunc 中完成初始化，确保执行顺序
3. **自定义 Slot 位置**：统计类 Slot 放在第一个，检查类 Slot 放在最后
4. **避免重量级操作**：Slot 在每次调用时执行，保持轻量
5. **异常处理**：自定义 Slot 中的异常不应影响后续 Slot 执行
