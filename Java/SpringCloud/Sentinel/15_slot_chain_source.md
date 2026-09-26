
# 1. 概述

SlotChain 是 Sentinel 的**核心执行引擎**，所有规则检查通过责任链模式串联。理解 SlotChain 是深入掌握 Sentinel 的关键。

> 源码版本：Sentinel 1.8.6
> 核心包：`com.alibaba.csp.sentinel.slots`

# 2. 类层次结构

```
ProcessorSlot<T> (接口)
  └── AbstractLinkedProcessorSlot<T> (抽象基类)
        ├── NodeSelectorSlot        # 构建 Node 调用树
        ├── ClusterBuilderSlot      # 构建 ClusterNode
        ├── LogSlot                 # 日志记录
        ├── StatisticSlot           # 统计引擎
        ├── SystemSlot              # 系统保护
        ├── AuthoritySlot           # 授权控制
        ├── FlowSlot                # 流量控制
        ├── DegradeSlot             # 熔断降级
        └── ParamFlowSlot           # 热点参数流控
```

# 3. ProcessorSlot 接口

```java
public interface ProcessorSlot<T> {
    // 进入资源
    void entry(Context context, ResourceWrapper resourceWrapper,
               T param, int count, boolean prioritized, Object... args)
               throws Throwable;

    // 链式触发下一个 Slot
    void fireEntry(Context context, ResourceWrapper resourceWrapper,
                   T param, int count, boolean prioritized, Object... args)
                   throws Throwable;

    // 退出资源
    void exit(Context context, ResourceWrapper resourceWrapper,
              int count, Object... args);
}
```

# 4. AbstractLinkedProcessorSlot

```java
public abstract class AbstractLinkedProcessorSlot<T> implements ProcessorSlot<T> {

    private AbstractLinkedProcessorSlot<?> next;  // 下一个 Slot

    @Override
    public void fireEntry(Context context, ResourceWrapper resourceWrapper,
                          T param, int count, boolean prioritized,
                          Object... args) throws Throwable {
        if (next != null) {
            // 调用下一个 Slot 的 entry 方法
            next.transformEntry(context, resourceWrapper, param,
                count, prioritized, args);
        }
    }

    // 类型转换 + 调用 entry
    void transformEntry(Context context, ResourceWrapper resourceWrapper,
                        Object param, int count, boolean prioritized,
                        Object... args) throws Throwable {
        @SuppressWarnings("unchecked")
        T t = (T) param;
        entry(context, resourceWrapper, t, count, prioritized, args);
    }
}
```

# 5. DefaultSlotChainBuilder

```java
@Spi(isDefault = true)
public class DefaultSlotChainBuilder implements SlotChainBuilder {

    @Override
    public ProcessorSlotChain build() {
        ProcessorSlotChain chain = new DefaultProcessorSlotChain();

        // 注意：构建顺序即执行顺序
        chain.addLast(new NodeSelectorSlot());       // 1. Node 构建
        chain.addLast(new ClusterBuilderSlot());     // 2. ClusterNode 构建
        chain.addLast(new LogSlot());                // 3. 日志
        chain.addLast(new StatisticSlot());          // 4. 统计（先执行后续，再统计）
        chain.addLast(new SystemSlot());             // 5. 系统保护
        chain.addLast(new AuthoritySlot());          // 6. 授权
        chain.addLast(new FlowSlot());               // 7. 流控
        chain.addLast(new DegradeSlot());            // 8. 熔断
        // ParamFlowSlot 通过 SPI 扩展添加

        return chain;
    }
}
```

# 6. CtSph — 入口

```java
public class CtSph implements Sph {

    @Override
    public Entry entry(String name, EntryType type, int count,
                       Object... args) throws BlockException {
        // 1. 获取或创建 Context
        Context context = ContextUtil.getContext();
        if (context == null) {
            context = InternalContextUtil.internalEnter(Constants.CONTEXT_DEFAULT_NAME);
        }

        // 2. 构建 ResourceWrapper
        StringResourceWrapper resource = new StringResourceWrapper(name, type);

        // 3. 获取 SlotChain（每个资源一个链）
        ProcessorSlot<Object> chain = lookProcessChain(resource);

        // 4. 创建 Entry
        Entry entry = new CtEntry(resource, chain, context);
        try {
            // 5. 执行 Slot 链
            chain.entry(context, resource, null, count, false, args);
        } catch (BlockException e) {
            entry.exit(count, args);  // 清理
            throw e;                  // 向上抛出 BlockException
        } catch (Throwable e) {
            entry.setError(e);
        }
        return entry;
    }
}
```

# 7. 责任链执行流程图

```
CtSph.entry("getUser")
  │
  ├── 1. NodeSelectorSlot.entry()
  │     └── 在 Context 中构建调用树节点
  │
  ├── 2. ClusterBuilderSlot.entry()
  │     └── 确保 ClusterNode 存在（全局统计节点）
  │
  ├── 3. LogSlot.entry()
  │     └── 记录 block 日志的准备
  │
  ├── 4. StatisticSlot.entry()
  │     ├── fireEntry() → 先执行后续 Slot
  │     │     ├── 5. SystemSlot → 系统保护检查
  │     │     ├── 6. AuthoritySlot → 授权检查
  │     │     ├── 7. FlowSlot → 流控检查
  │     │     ├── 8. DegradeSlot → 熔断检查
  │     │     └── 9. ParamFlowSlot → 热点参数检查
  │     │
  │     ├── 通过 → 统计 pass + threadNum++
  │     └── 拒绝 → 统计 block
  │
  └── 返回 Entry
```

# 8. SlotChainProvider — 链的创建与缓存

```java
public final class SlotChainProvider {

    private static volatile SlotChainBuilder builder;

    // 每个资源维护独立的 SlotChain
    private static volatile Map<ResourceWrapper, ProcessorSlotChain> chainMap
        = new HashMap<>();

    public static ProcessorSlotChain newSlotChain() {
        if (builder == null) {
            // SPI 加载 SlotChainBuilder
            builder = SpiLoader.of(SlotChainBuilder.class)
                .loadFirstInstanceOrDefault();
        }
        return builder.build();
    }
}
```

**关键点**：每个 Resource 拥有独立的 SlotChain 实例，但 Slot 内部共享全局统计（ClusterNode）。

# 9. SPI 扩展 SlotChain

```java
// 自定义 SlotChainBuilder
@Spi(order = -100)  // 优先级（默认 0）
public class MySlotChainBuilder implements SlotChainBuilder {

    @Override
    public ProcessorSlotChain build() {
        ProcessorSlotChain chain = new DefaultProcessorSlotChain();

        // 默认的 8 个 Slot
        chain.addLast(new NodeSelectorSlot());
        chain.addLast(new ClusterBuilderSlot());
        chain.addLast(new LogSlot());
        chain.addLast(new StatisticSlot());
        chain.addLast(new SystemSlot());
        chain.addLast(new AuthoritySlot());
        chain.addLast(new FlowSlot());
        chain.addLast(new DegradeSlot());

        // 自定义 Slot
        chain.addLast(new MyCustomSlot());

        return chain;
    }
}

// META-INF/services/com.alibaba.csp.sentinel.slotchain.SlotChainBuilder:
// com.example.MySlotChainBuilder
```

# 10. 设计要点总结

1. **每个 Resource 独立链**：避免锁竞争，提升并发性能
2. **StatisticSlot 的特殊位置**：先执行后续 Slot 再统计，确保指标准确性
3. **责任链可扩展**：通过 SPI 自定义 SlotChainBuilder 或添加自定义 Slot
4. **Context 线程绑定**：每个线程一个 Context，Entry 通过 Context 串联
