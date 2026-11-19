# 1. 概述与设计动机

## 1.1 SlotChain 是什么

SlotChain 是 Sentinel 的**核心执行引擎**，所有规则检查通过责任链模式串联。理解 SlotChain 是深入掌握 Sentinel 的关键。

> 源码版本：Sentinel 1.8.6
> 核心包：`com.alibaba.csp.sentinel.slots`

从调用点看，`SphU.entry("res")` 只是一次方法调用，但内部实际走过一条近十个节点的检查流水线：构建统计树、绑定线程上下文、记录指标、系统保护、黑白名单、流控、熔断、热点参数。SlotChain 的职责就是把这组**职责互不相同的"检查站点"按固定顺序串成一条链**，让每个请求依次通过。

读本篇的两个主线：

- **装配**：链由 `DefaultSlotChainBuilder` 组装，装配顺序即执行顺序（§3）；
- **运行**：`CtSph.entry()` 的每次请求沿链走一遍，通过则返回 `Entry`，拒绝则抛 `BlockException`（§4）。

## 1.2 为什么用责任链模式

Sentinel 在执行路径设计上要同时满足四类诉求：

| 诉求 | 责任链如何解决 |
|------|----------------|
| 检查类型多，且要不断新增（热点参数、自适应保护……） | 新检查加一个 Slot 即可，无需改动既有代码（开闭原则） |
| 检查顺序敏感（统计必须"观察"规则决定） | 链表结构把顺序固化下来，`addLast` 的调用序列自文档化 |
| 三方希望插入自定义逻辑 | SPI 可追加单个 Slot，也可整体替换链（§6） |
| 热路径必须快 | Slot 无状态、链构建后只读，遍历全程无锁，没有共享可变状态 |

一个常见误解是"所有请求共享一条全局链"。实际上**每个资源拥有自己的链**（§5.1），不同资源的链彼此独立；真正共享的是链背后的统计数据（ClusterNode）。

和 Netty 的 `ChannelPipeline` 对比：Sentinel 的链是**固定结构**——资源第一次被访问时装配一次，之后只读，不能在运行中增删节点；Netty 的 pipeline 可以按 Channel 动态调整。Sentinel 牺牲灵活性，换来"热路径无锁遍历 + 结构可预期"。

# 2. 核心类层次与接口

## 2.1 类层次结构

```text
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

图中每个类名即一个检查站点的职责。泛型参数 `T` 是**沿链传递的上下文参数**——它会随着遍历推进被逐层"加工"（Object → DefaultNode → Node），类型演进见 §2.4。

## 2.2 ProcessorSlot 接口

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

三个方法的语义分工：

| 方法 | 调用时机 | 行为约定 |
|------|----------|----------|
| `entry` | 请求进入，链遍历 | 做本节点检查；要拒绝就抛 `BlockException`；放行靠调 `fireEntry` 接力给下一节点 |
| `fireEntry` | 在自己 `entry` 内部 | "接力"动作：调用下一个 Slot 的 `entry`；若已是链尾，遍历结束 = 请求通过 |
| `exit` | 请求结束（`Entry#exit`） | 同样沿链传递，做"收尾"：减线程数、统计 RT、清理上下文 |

注意**"通过"没有专门的返回值**——Slot 想拦截就抛 `BlockException`，否则必须自己调 `fireEntry` 接力。责任链的"决定"被惯用成异常控制流，这是读 Sentinel 源码时最容易看漏的约定。

## 2.3 AbstractLinkedProcessorSlot

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

这个抽象类只负责"链表"这一件事：持有 `next` 引用，`fireEntry`/`fireExit` 做接力（exit 侧同理，此处省略）。两个实现细节值得注意：

1. `transformEntry` 是包私级的桥方法，负责把 `Object` 向下转成具体 Slot 声明的 `T`，再调它的 `entry`；
2. `DefaultProcessorSlotChain` 通过 `addLast` 维护头尾指针完成装配，**链头是一个空转的中继节点**：`chain.entry(...)` 穿过它直接进入第一个业务 Slot——这让"整条链"和"单个节点"在类型上都是 `ProcessorSlot`，调用方无需区分。

## 2.4 链上参数的类型演进

各 Slot 之间靠 `T param` 传递信息，实际类型沿链变化：

| Slot | 声明泛型 | 入参 | 向下游传递 |
|------|----------|------|------------|
| NodeSelectorSlot | `<Object>` | null | 本 (Context, 资源) 的 DefaultNode |
| ClusterBuilderSlot | `<DefaultNode>` | DefaultNode | 不变（同时绑定 ClusterNode、origin 来源节点） |
| LogSlot / StatisticSlot | `<DefaultNode>` | DefaultNode | 不变 |
| SystemSlot / AuthoritySlot / FlowSlot / DegradeSlot | `<Node>` | DefaultNode | 不变 |
| ParamFlowSlot（SPI 追加） | `<Node>` | Node | 不变 |

这套机制**完全靠装配顺序约定保证**：`transformEntry` 的强转没有运行时类型校验。如果把 `FlowSlot`（`<Node>`）排到了 `NodeSelectorSlot` 前面，它收到的还是 `Object`，强转直接 `ClassCastException`；即使类型碰巧兼容，拿到的也可能是错误的统计节点。这就是 §7.2 里"Slot 顺序不能乱动"的根源。

# 3. 链的装配：DefaultSlotChainBuilder

## 3.1 装配顺序即执行顺序

```java
@Spi(isDefault = true)
public class DefaultSlotChainBuilder implements SlotChainBuilder {

    @Override
    public ProcessorSlotChain build() {
        ProcessorSlotChain chain = new DefaultProcessorSlotChain();

        // 注意：构建顺序即执行顺序
        chain.addLast(new NodeSelectorSlot());       // 1. Node 构建
        chain.addLast(new ClusterBuilderSlot());     // 2. ClusterNode 构建
        chain.addLast(new LogSlot());                // 3. 日志（历史占位，基本透传）
        chain.addLast(new StatisticSlot());          // 4. 统计（先执行后续，再统计）
        chain.addLast(new SystemSlot());             // 5. 系统保护
        chain.addLast(new AuthoritySlot());          // 6. 授权
        chain.addLast(new FlowSlot());               // 7. 流控
        chain.addLast(new DegradeSlot());            // 8. 熔断

        // 1.8.0+：SPI 追加的 ProcessorSlot 挂在内置 Slot 之后
        for (ProcessorSlot slot : SpiLoader.loadInstanceList(ProcessorSlot.class)) {
            chain.addLast(slot);
        }

        return chain;
    }
}
```

`addLast` 的调用顺序就是请求的遍历顺序。这段代码有两个容易忽略的事实：

1. **内置链只装配 8 个 Slot**。`ParamFlowSlot` 不在名单里——它由 `sentinel-parameter-flow-control` 模块通过上面的 `ProcessorSlot` SPI 注册（`META-INF/services/com.alibaba.csp.sentinel.slotchain.ProcessorSlot`），引入依赖即自动生效；
2. builder 每次返回的都是**新建**的链，由调用方（`CtSph`）按资源缓存（§5.1），builder 本身不持有链。

## 3.2 各 Slot 为什么排在这个位置

| # | Slot | 职责 | 位置依据 |
|---|------|------|----------|
| 1 | NodeSelectorSlot | 构建 (Context, 资源) 调用树 DefaultNode | 统计与检查的前置条件：后续所有 Slot 都要用 Node |
| 2 | ClusterBuilderSlot | 构建/获取 ClusterNode 与 origin 节点 | 全局统计入口，也是流控"应用级效果"的依据 |
| 3 | LogSlot | 历史占位（block 日志已迁到异步组件写 sentinel-block.log） | 位置不再敏感，保留只为兼容 |
| 4 | StatisticSlot | pass/block/exception/线程数/RT 统计 | 必须在规则 Slot 之前，才能"观察"下游的决定结果（§4.3） |
| 5 | SystemSlot | 系统自适应保护（Load/CPU/RT/线程数/总 QPS） | 最粗粒度的否决先行，系统已过载时不再浪费后续细检查；只检查 `EntryType.IN` 入口流量 |
| 6 | AuthoritySlot | 黑名单/白名单（按 origin 来源） | 身份是"能不能进门"的判断，先于一切流量判断 |
| 7 | FlowSlot | 流控规则检查 | 常规流量控制 |
| 8 | DegradeSlot | 熔断降级 | 依赖实时统计（异常率/RT）决定开合 |
| SPI | ParamFlowSlot | 热点参数限流/排队 | 细粒度在粗粒度之后；限流排队可能抛 `PriorityWaitException` |

一句话记忆：**建树 → 建全局节点 → 立统计观察员 → 由粗到细否决**。

## 3.3 链的不可变性与线程安全

链在装配期只被 `addLast` 追加，构建完成后被多个线程的请求**只读共享**。它敢这么做的两个前提：Slot 实例自身无状态（所有可变数据都在 Node、Context 里，参数从方法入参传入），以及"构建 → 发布"发生在 `lookProcessChain` 的锁内、经 volatile 引用替换发布（§5.2），保证链结构对其他线程安全可见。反过来说，如果你在自定义 Slot 里放了成员变量存请求状态，整条链的线程安全假设就被破坏了。

# 4. 请求执行路径

## 4.1 CtSph.entry() 主干流程

```java
public class CtSph implements Sph {

    @Override
    public Entry entry(String name, EntryType type, int count,
                       Object... args) throws BlockException {
        // 1. 构建 ResourceWrapper
        StringResourceWrapper resource = new StringResourceWrapper(name, type);

        // 2. 获取 Context（NullContext = 上层显式关闭记录，裸放行）
        Context context = ContextUtil.getContext();
        if (context instanceof NullContext) {
            return new CtEntry(resource, null, context);
        }
        if (context == null) {
            context = InternalContextUtil.internalEnter(Constants.CONTEXT_DEFAULT_NAME);
        }

        // 3. 全局开关关闭：同样裸放行，不检查不统计
        if (!Constants.IS_GLOBAL_ENABLE) {
            return new CtEntry(resource, null, context);
        }

        // 4. 获取 SlotChain（每个资源一条链；资源数超限返回 null）
        ProcessorSlot<Object> chain = lookProcessChain(resource);
        if (chain == null) {
            return new CtEntry(resource, null, context);
        }

        // 5. 创建 Entry 并执行 Slot 链
        Entry entry = new CtEntry(resource, chain, context);
        try {
            chain.entry(context, resource, null, count, false, args);
        } catch (BlockException e) {
            entry.exit(count, args);  // 清理
            throw e;                  // 向上抛出 BlockException
        } catch (Throwable e) {
            // 链内部错误：仅记录，不阻断业务（fail-open）
            entry.setError(e);
        }
        return entry;
    }
}
```

分步解读，重点看三种"裸放行"场景：

1. **NullContext**：入口名称为空等场景下 Sentinel 会给出 NullContext，此时完全跳过记录与统计；
2. **全局开关** `Constants.IS_GLOBAL_ENABLE`：关闭后所有资源都不走链，等于把 Sentinel 从调用路径里摘出来；
3. **chain == null**：资源数达到上限（§5.2），该资源放弃保护直接放行。

异常处理分两种命运：`BlockException` 是**正常的控制流**——`entry.exit` 清理现场后原样抛出给业务；**其他 Throwable 被视为链的内部错误**——只记录 `error`，请求照常通过。这个不对称是理解 Sentinel "fail-open 哲学"（观测组件不许因自身故障阻断业务）的钥匙。

## 4.2 链执行流程图

```text
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

这张图实际是**嵌套**而非平铺的：`StatisticSlot` 调 `fireEntry` 后游标才进入下游规则 Slot；下游全部正常返回，StatisticSlot 回到自己的代码继续加 pass；下游抛 BlockException，它捕获后加 block 再重抛。所以同一个请求的 pass/block/exception 指标都出自第 4 站——"先放行后记账"的位置安排是整条链最精妙的一点，下节展开。

## 4.3 StatisticSlot：先发射，后统计

StatisticSlot 的 entry 骨架（简化）：

```java
try {
    // 先让下游规则 Slot 决定命运
    fireEntry(context, resourceWrapper, node, count, prioritized, args);
    // 下游全部通过，此时才记账
    node.increaseThreadNum();
    node.addPassRequest(count);
} catch (PriorityWaitException ex) {
    // 排队限流场景：已通过等待"计过数"，这里只补线程数
    node.increaseThreadNum();
} catch (BlockException e) {
    context.getCurEntry().setError(e);
    node.increaseBlockQps(count);
    throw e;
} catch (Throwable e) {
    context.getCurEntry().setError(e);
    node.increaseExceptionQps(count);
    throw e;
}
```

为什么要"后统计"？答案藏在指标定义里：`pass` 必须是**通过了全部检查**的数量，线程数必须是**真正被放进来**的并发量。如果在规则检查之前记账，被流控拦掉的请求也会推高 pass 与并发线程数——而这两个值恰恰是流控（线程数模式）和熔断（RT/异常比例）的输入，等于自己污染自己的决策依据。

`PriorityWaitException` 是个精致分支：当流控/热点参数选择了 RateLimiter（匀速排队）策略时，请求会在检查点内 sleep 到配额允许的时刻，然后以这个"非 Block 异常"通知 StatisticSlot"我在排队阶段已经记过 pass 了，别再记一遍"，只需补线程数。这也是"通过链参数传决定结果"的又一例。

## 4.4 exit 的回程

`entry` 成功后业务调用 `entry.exit()`（应放 `finally` 或用 try-with-resources）。exit 沿链同样从前到后传递，做三类收尾：

- **StatisticSlot.exit**：`decreaseThreadNum` 归还并发额度；请求正常完成时用 `completeTime - createTime` 记 RT 和 success 计数；若 `Entry` 上挂了 `Tracer.trace()` 记录的异常，计入异常指标；
- **Context 层级**：把当前 Entry 从上下文的 Entry 链表摘除、父 Entry 恢复为 `curEntry`——嵌套资源（一个 entry 里再 entry）就是靠 Context 内的双向链表找到"我的父节点"，构建调用树；
- **ContextUtil.exit**：最外层退出时清理线程绑定的上下文，防止线程池复用导致上下文串味。

忘记 exit 是最隐蔽的误用：线程数只增不减，"并发线程数"模式的流控会误判系统满载而持续拦截，且故障点离真正的肇事代码很远（参见 `24_troubleshooting.md`）。

# 5. 链的缓存与生命周期

## 5.1 每个资源一条链

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

"一资源一链"的成本是每个资源一套 8+ 个 Slot 外壳（实例本身很小，规则、Node 引用都是共享的）；收益是热路径的**局部性与零查找**——遍历链时每个 Slot 拿到的参数就是当前资源的统计对象，链不需要再"判断现在在处理哪个资源"。真正大的共享数据（ClusterNode、各规则 List）全部搬到了链外。

## 5.2 lookProcessChain：双检锁与 64000 上限

`CtSph` 缓存链的核心方法（还原）：

```java
ProcessorSlot<Object> lookProcessChain(ResourceWrapper resourceWrapper) {
    ProcessorSlotChain chain = chainMap.get(resourceWrapper);
    if (chain == null) {
        synchronized (LOCK) {
            chain = chainMap.get(resourceWrapper);      // 双重检查
            if (chain == null) {
                if (chainMap.size() >= MAX_RESOURCE_COUNT) {   // 64000
                    return null;                        // 放弃保护，"裸放行"
                }
                chain = SlotChainProvider.newSlotChain();
                Map<ResourceWrapper, ProcessorSlotChain> newMap =
                    new HashMap<>(chainMap.size() + 1);
                newMap.putAll(chainMap);
                newMap.put(resourceWrapper, chain);
                chainMap = newMap;                      // copy-on-write 发布
            }
        }
    }
    return chain;
}
```

三个设计点：

1. **双检锁**：已缓存资源的查找完全无锁，只有首次构建才进全局锁，避免同一资源被并发重复装配；
2. **写时复制**：扩容不是原地 `put` 而是整表重建后替换 volatile 引用，读线程要么看到旧表要么看到新表，永远不会读到结构性损坏的 HashMap；
3. **上限 fail-open**：资源数超过 `MAX_RESOURCE_COUNT`（64000）后直接返回 null 链——宁可这个资源不设防，也不允许链缓存把内存吃穿。

第 3 点是真实的高频事故来源：把带路径变量的原始 URI（`/user/1001`、`/user/1002`…）或动态拼接串当资源名，资源数无限膨胀；撑满 64000 后，**之后所有新资源全部失去保护**，表现为"规则明明配了却拦不住"。资源名必须收敛为模板（`/user/{id}`），排查方法参见 `24_troubleshooting.md`。

## 5.3 链的构建时机

链是**首次 entry 时才懒构建**的：`@SentinelResource` 声明资源、甚至推送规则，都不会生成链。由此有两个实用推论——"配置了规则但 Dashboard 没有数据"多半只是还没有真实请求触发构建；而集群冷启动瞬间，第一批请求会集中走锁构建链，属于一次性的微小毛刺，无需担心。

# 6. SPI 扩展

## 6.1 自定义 SlotChainBuilder

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

注意这个方案的代价：`SpiLoader.loadFirstInstanceOrDefault` 按 order **只取一个**生效 builder，自定义者**整体替换**默认 builder。你的 `build()` 漏掉任何一个内置 Slot，就等于全局所有资源少了那个机制——漏 `StatisticSlot` 则 Dashboard 空白，漏 `FlowSlot` 则流控全线失效。实践中除非需要重排链结构，不要用这条路，走 §6.2。

## 6.2 更优雅：SPI 追加单个 Slot（1.8.0+）

1.8.0 起可以在 `META-INF/services/com.alibaba.csp.sentinel.slotchain.ProcessorSlot` 注册独立 Slot，由默认 builder 追加到内置 8 个之后（§3.1 的循环）。自定义检查逻辑只写一个 Slot，零替换风险：

```java
public class MyAuditSlot extends AbstractLinkedProcessorSlot<Node> {

    @Override
    public void entry(Context context, ResourceWrapper rw, Node node,
                      int count, boolean prioritized, Object... args)
                      throws Throwable {
        if (needBlock(args)) {
            // 拦截 = 抛 BlockException 家族，而不是普通异常
            throw new FlowException("my-audit-slot");
        }
        fireEntry(context, rw, node, count, prioritized, args);
    }
}
```

两条约束：追加的 Slot 永远排在内置 8 个**之后**（无法插队到统计之前，声明泛型按 §2.4 选 `<Node>`）；若在自己的 Slot 里抛出非 `BlockException` 的异常，它会按链内部错误处理——计入 exception 但**不拦截**请求（§4.1 第 5 步），拦截必须用 BlockException 家族。SPI 全景（含规则持久化、心跳等扩展点）见 `19_spi_extension.md`。

# 7. 设计要点总结与常见坑

## 7.1 设计要点

1. **每个 Resource 独立链**：避免链结构上的锁竞争，提升并发性能；大的共享数据（ClusterNode、规则表）全部外置到链外；
2. **StatisticSlot 的特殊位置**：先执行后续规则、后统计，确保 pass/block/exception/线程数这些"被别人依赖的指标"不被自己的决策输入污染；
3. **责任链可扩展**：优先用 `ProcessorSlot` SPI 追加单个 Slot，`SlotChainBuilder` 整体替换仅作兜底；
4. **Context 线程绑定**：每线程一个 Context，Entry 以双向链表挂在其下，支撑嵌套资源建树与 exit 清理；
5. **fail-open 贯穿全局**：NullContext、全局开关、64000 上限、内部异常吞掉——四处兜底都在"业务照常"一侧，Sentinel 把自己定位为可牺牲的观测件而非必达的守门件。

## 7.2 常见坑

| 坑 | 现象 | 原因与对策 |
|----|------|------------|
| 自定义 SlotChainBuilder 漏装配内置 Slot | 某类规则全线失效 / Dashboard 空白 | builder 是整体替换语义（§6.1）；改用 ProcessorSlot SPI 追加 |
| 调整 Slot 装配顺序 | `ClassCastException` 或统计错乱 | Slot 间类型靠顺序约定，无运行时校验（§2.4）；保持 StatisticSlot 位于规则 Slot 之前 |
| 资源名含变量、粒度过细 | 新资源完全不设防 | chainMap 达 64000 上限后 fail-open（§5.2）；资源名收敛为模板 |
| `entry` 后不调 `exit` | 并发线程数只增不减，线程模式流控误拦截 | 线程数在 exit 时归还（§4.4）；用 try-with-resources / finally 统一收尾 |
| 在自定义 Slot 里抛普通异常想拦截 | 请求照常通过，只有异常计数上涨 | 只有 BlockException 才是"拦截"语言（§6.2）；普通异常按内部错误放行 |

相关阅读：`18_statistics_engine.md`（StatisticSlot 统计落点的滚动窗口实现）、`19_spi_extension.md`（SPI 扩展全景）、`16_flow_control_source.md`（FlowSlot 之后的规则检查细节）。
