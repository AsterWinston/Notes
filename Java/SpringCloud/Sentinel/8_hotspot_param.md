# 1. 概述

热点参数流控（Hotspot Parameter Flow Control）是对**资源调用中的参数**进行精细化限流。普通流控以资源整体为粒度，热点参数流控按**参数值**分别统计和限流。

**典型场景**：

```text
商品详情接口：/product/detail?id=1001
  ├── id=1001（热门商品）→ QPS 限制 5000
  ├── id=1002（冷门商品）→ QPS 限制 100（默认）
  └── id=1003 → QPS 限制 100（默认）
```

## 1.1 它解决的是资源级限流的两难

对 `/product/detail` 整体限 100 QPS，爆款一来正常商品全被误伤；放宽到 5000，长尾商品刷不出量也照单全收。热点参数把限流粒度下沉到**参数值的维度**：每个被统计到的值各有一套计数器，阈值可以按值例外——本质是"同资源内的差异化配额"。两条与前四篇规则的路线差异要刻在脑子里：**① 判定对象是 `args` 数组的一个位置**（资源级规则看窗口计数，它看"这个值在这个窗口过了几个"）；**② 它是独立模块**——实现不在 sentinel-core，`ParameterFlowSlot` 与规则类都来自 `sentinel-parameter-flow-control` 依赖，不引 jar 则规则数据源配了也没人消费（链上装配见 `2_architecture.md` §3.3，`3_quick_start.md` §2.1 列过这条依赖）。

# 2. ParamFlowRule 核心属性

```java
public class ParamFlowRule extends AbstractRule {
    private int grade = RuleConstant.FLOW_GRADE_QPS;     // 阈值类型
    private double count;                                  // 阈值
    private int paramIdx;                                  // 参数索引（从 0 开始）
    private int controlBehavior;                           // 流控效果
    private int maxQueueingTimeMs;                         // 排队时间
    private int burstCount;                                // 突发流量容忍
    private long durationInSec;                            // 统计时长
    private List<ParamFlowItem> paramFlowItemList;         // 参数例外项
}
```

## 2.1 参数索引（paramIdx）

指定对方法的第几个参数进行热点限流：

```java
// @SentinelResource 方法签名
@SentinelResource("getProduct")
public Product getProduct(Long id, String name) {
    // paramIdx = 0 → 对 id 参数限流
    // paramIdx = 1 → 对 name 参数限流
}
```

## 2.2 可当热点维度的参数类型（硬约束）

args 位置的元素不是任意对象都能被按值统计，支持集：**8 种基本类型及其包装 + String**；数组/集合参数**按长度**当值统计（`Object[]`、`List` 进 map 的 key 是 size，不是内容——想用集合内容做热点维度的配置从第一行就错了）；其他对象要实现 `ParamFlowArgument` 接口（`hotKey()` 返回可统计值），没实现的对象按 `hashCode` 归并成整桶限流——所有实例挤同一个计数器，效果近似资源级流控。`paramIdx` 越界（args 没那么长）时该规则对该次调用**直接放行**（越界视为无参数，不抛错），所以"注解方法参数与 paramIdx 对不上=规则静默失效"是头号陷阱：方法签名重构加参数时把热点规则一起审，或按 §2.3 用兜底例外项断言。

## 2.3 字段语义速查

| 字段 | 语义要点 |
|------|---------|
| `grade` | QPS/线程数都支持（线程数按"该参数值的并发占用"计，值粒度） |
| `durationInSec` | 统计窗口（默认 1s；窗口越大统计越平滑、响应越钝） |
| `burstCount` | 容忍度：窗口内在 count 之外多放 burst 个（应对压测/整点脉冲，不改变 count 语义） |
| `controlBehavior` | 字段在，但控制台热点表单不提供流控效果、实践主流只有快速失败语义；想排队先想清楚是不是该用 MQ 削峰 |
| `paramFlowItemList` | 例外项列表：指定值单独阈值（§3） |
| `clusterMode`/`clusterConfig` | 集群全局阈值（§6 勘误后看这里） |

# 3. 参数例外项（ParamFlowItem）

为**特定参数值**设置独立的阈值：

```java
ParamFlowRule rule = new ParamFlowRule();
rule.setResource("getProduct");
rule.setParamIdx(0);         // 对第 0 个参数（id）
rule.setCount(100);          // 默认阈值 100 QPS
rule.setGrade(RuleConstant.FLOW_GRADE_QPS);

// 例外项：id=1001 的商品热点
ParamFlowItem hotItem = new ParamFlowItem();
hotItem.setObject("1001");   // 匹配 id=1001
hotItem.setClassType("long");
hotItem.setCount(5000);      // 独立阈值 5000 QPS

ParamFlowItem hotItem2 = new ParamFlowItem();
hotItem2.setObject("1002");
hotItem2.setClassType("long");
hotItem2.setCount(3000);

rule.setParamFlowItemList(Arrays.asList(hotItem, hotItem2));
ParamFlowRuleManager.loadRules(Collections.singletonList(rule));
```

## 3.1 匹配与类型归一规则

例外项的 `object` 恒是**字符串字面量**，`classType` 决定解释方式（默认 `"long"`——漏写时 String 参数的例外项永远配不上）；数值族（byte/short/int/long/float/double）做**加宽归一比较**（int 实参 1001 能配 long 例外项），String/char/boolean 走字面精确相等。两个边界：**例外项 count 可以与默认阈值反向**（给冷门值配比默认更低的阈值合法，长尾保护用）；一个资源+paramIdx 命中多个例外项时取**解析后的最大者**（不存在"后配置覆盖"的歧义）。规则加载期对每个 item 做解析校验，**解析失败的条目被静默丢弃**（`"abc"` 配成 long 不会报错，只是那个例外永远不生效）——例外项是"救火阀门"，每次加完必须实测那个热点值真的走高了阈值。

# 4. 完整示例

## 4.1 定义资源方法

```java
@RestController
@RequestMapping("/api")
public class ProductController {

    @GetMapping("/product/{id}")
    @SentinelResource(value = "getProduct", blockHandler = "handleHotBlock")
    public Product getProduct(@PathVariable Long id,
                              @RequestParam(required = false) String category) {
        return productService.getById(id);
    }

    // 热点参数限流回调
    public Product handleHotBlock(Long id, String category, BlockException ex) {
        ParamFlowException pfe = (ParamFlowException) ex;
        log.warn("热点限流: resource={}, paramIdx={}, id={}",
            pfe.getResourceName(), pfe.getParamIdx(), id);
        return Product.fallback(id);  // 返回降级商品信息
    }
}
```

## 4.2 配置规则

```java
@Configuration
public class HotspotRuleConfig {

    @PostConstruct
    public void initParamFlowRules() {
        ParamFlowRule rule = new ParamFlowRule();
        rule.setResource("getProduct");
        rule.setParamIdx(0);
        rule.setCount(100);                          // 默认 100 QPS
        rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
        rule.setDurationInSec(1);

        // 热点商品例外
        rule.setParamFlowItemList(Arrays.asList(
            createItem("1001", Long.class, 5000),    // 秒杀商品
            createItem("1002", Long.class, 3000),    // 促销商品
            createItem("1003", Long.class, 2000)     // 新品首发
        ));

        ParamFlowRuleManager.loadRules(Collections.singletonList(rule));
    }

    private ParamFlowItem createItem(String value, Class<?> clazz, int count) {
        ParamFlowItem item = new ParamFlowItem();
        item.setObject(value);
        item.setClassType(clazz.getSimpleName().toLowerCase());
        item.setCount(count);
        return item;
    }
}
```

## 4.3 示例的两处核对项

**① `(ParamFlowException) ex` 裸强转会炸**：blockHandler 接的是所有 BlockException 家族（这条资源同时可能被普通流控/熔断拦，`2_architecture.md` §3.3 的链序），强转前先 `instanceof`，否则一次 FlowException 就把降级方法本身搞成 500。**② args 是怎么进到 slot 的**：`@SentinelResource` 的切面把方法入参数组作为 `entry(..., args)` 传入——这解释了 §2.2 的"以方法签名为准"：参数被拆成两个重载？paramIdx 语义跟着变。而 **Web filter 生成的 URL 资源**想吃到热点参数，靠的是 SCA 的 `spring.cloud.sentinel.web-sync`（默认 true，把 query 参数值按序塞进 args）：参数顺序来自 request parameter map 的遍历序，**多 query 参数时第 0 位不保证是你要的那一个**——URL 资源的热点规则要么只有一个参数，要么改用注解资源；`web-sync=false` 时 URL 资源根本没有 args，配了热点也永不触发。

# 5. ParamFlowSlot 执行流程

```java
public class ParamFlowSlot extends AbstractLinkedProcessorSlot<DefaultNode> {

    @Override
    public void entry(Context context, ResourceWrapper resourceWrapper,
                      DefaultNode node, int count, boolean prioritized,
                      Object... args) {
        List<ParamFlowRule> rules = ParamFlowRuleManager.getRulesOfResource(resourceWrapper.getName());
        for (ParamFlowRule rule : rules) {
            // 1. 提取指定索引的参数值
            Object param = args[rule.getParamIdx()];
            
            // 2. 检查是否有例外项匹配
            ParamFlowItem matchedItem = matchExceptionItem(rule, param);
            
            // 3. 使用例外项的阈值或默认阈值
            double threshold = (matchedItem != null) 
                ? matchedItem.getCount() 
                : rule.getCount();
            
            // 4. 检查是否超过阈值
            if (!ParamFlowChecker.passCheck(rule, param, threshold)) {
                throw new ParamFlowException(resourceWrapper.getName(), 
                    param, rule.getParamIdx());
            }
        }
        fireEntry(context, resourceWrapper, node, count, prioritized, args);
    }
}
```

## 5.1 类名与数据结构纠偏

真实类名是 **`ParameterFlowSlot`**（示例写 `ParamFlowSlot`，SPI 注册与栈帧里找前者），且如 §1.1 所说在独立模块、按依赖进链。伪码骨架方向正确，补三处真实结构：① 统计载体是**每资源一个 `ParameterMetric`**，内部按 paramIdx 分槽，每槽一张"参数值 → 滑动窗口"的 Map（`ObjectObjMap` 形态，值为 `HotParameterLeapArray`）+ 一张"参数值 → 通过计数"的 `ObjectToLongCounter`——这就是高基数参数的内存账所在：实现设有条目上限保护（值数量超限时按低计数清理），但**清理本身会丢掉热点记忆**，用 userId 这类无界值当热点维度，限流精度和内存都在漏（§8.3 的"高区分度"要反过来读：选**热点集中**的值域）；② 无例外项匹配的值走**默认桶**（所有长尾共享 rule.count 一个计数器，不是"每个值各 100"——§1 图里"冷门商品各自 100"的说法要精化为"冷门商品合计 100"，这是新手最大的语义误读）；③ 参数值统计跨窗口重置，`burstCount` 加在窗口额度上。`ParamFlowException` 携带被限的具体值（`getParamValue()` 取到的是对象），§4.1 日志用它比打印入参更准。

# 6. 集群热点流控

Sentinel 支持**集群维度**的热点流控，针对分布式场景中的热点参数做全局限流：

```java
// 集群流控规则通过 Dashboard 配置
// 指定 Token Server 来协调集群范围的热点判断
FlowRule rule = new FlowRule();
rule.setClusterMode(true);              // 开启集群模式
rule.setClusterConfig(new ClusterFlowConfig()
    .setThresholdType(ClusterRuleConstant.FLOW_THRESHOLD_GLOBAL)  // 全局阈值
    .setFlowId(1L));
```

集群热点流控需要部署**Token Server**，负责全局统计和令牌分配。

## 6.1 【勘误】这段示例是普通集群流控，不是集群热点

代码块配的是 `FlowRule`——热点参数的全局阈值要用 **`ParamFlowRule` 自己的集群字段**：`rule.setClusterMode(true)` + `ClusterParamFlowConfig`（`flowId` 同样必须全局唯一且与 Dashboard/Nacos 对齐，`thresholdType` 全局/单机，`fallbackToLocalWhenFail`——它决定 token server 失联时退化为**单机热点语义**还是直接拒绝，生产几乎都选退化）。集群模式下每个参数值的令牌判定发往 server 集中计数，`tokenCalculateType`（direct 立即发/滑动均值/最小方差等策略）控制对 server 统计的宽容度——多实例并发打同一热点值时，direct 阈值贴着峰值容易误杀，配了滑动均值类策略才是"温和的总量控制"。代价侧核对：全局热点的每参数值判定都多一次 client→server RPC（同进程内嵌模式也走 loopback 网络栈），RT 敏感链路慎用集群热点，能靠"单机阈值 × 实例数换算 + §3.1 例外项"解决就不要上全局计数（token server 部署形态与故障演练在 `14_dynamic_rules.md` §10，机制源码在 `16_flow_control_source.md` 的 cluster 分支）。

# 7. Nacos 持久化配置

```json
// dataId: ${app}-param-flow-rules
[
  {
    "resource": "getProduct",
    "paramIdx": 0,
    "count": 100,
    "grade": 1,
    "durationInSec": 1,
    "paramFlowItemList": [
      { "object": "1001", "classType": "long", "count": 5000 },
      { "object": "1002", "classType": "long", "count": 3000 }
    ]
  }
]
```

## 7.1 数据源接线的两件套

SCA 侧 `spring.cloud.sentinel.datasource.flowparam.nacos.*`（名字随意，键名自定）配 `rule-type: param-flow`，注意 `rule-type` 的值是 **param-flow** 不是 paramFlowRule；两件套缺一都不转：**依赖 `sentinel-parameter-flow-control`**（§1.1，决定 Slot 与 Checker 在不在）+ 这份 datasource 配置（决定规则进不进 Manager）。Nacos JSON 里 `paramFlowItemList` 的每个例外同样过 §3.1 的静默解析校验——手改文件后到 Dashboard 簇点链路确认该资源出现"热点规则"条目，或 `ParamFlowRuleManager.hasRules(resource)` 打点，别信"保存即生效"。

# 8. 最佳实践

1. **适用场景**：商品详情、秒杀商品、热门内容 — 对热点资源的差异化防护
2. **阈值设定**：热点商品例外阈值 = 压测基准 × 1.5，普通商品 = 压测基准 × 0.5
3. **参数选择**：选择区分度高的参数作为热点维度（如商品ID、用户ID）
4. **避免过多例外项**：超过 100 个例外项考虑用其他策略（如缓存 + 流控组合）
5. **配合缓存**：热点参数限流 + 本地缓存，双重保护

## 8.1 逐条校准

**2** 的"普通商品 ×0.5"注意口径：默认 count 是**所有未列出值共享的总闸**（§5.1 ②），"普通商品=压测×0.5"意味着长尾**合计**只给一半容量，这是刻意"宠溺热点"的容量分配，上线前对长尾转化率做一次评估。**3** 与实现限制正相反（§5.1 ① 已析）：正确表述是选**基数可控 + 热点集中**的参数——商品 ID（百万级但同时在热的 <1%）合适；裸用户 ID/订单号（近乎无界）配"默认大阈值 + 只对已知大 V 建例外项"才可用。**4** 的 100 与实现的例外项数量级警戒线一致，但比数量更早触发的问题是**运营化例外项的归属**：热点名单随活动每天变，例外项必须进数据源（Nacos）管理并有增删流程，写在代码里的 `@PostConstruct` 版撑不过第二个促销日——这也正好和 `14_dynamic_rules.md` 的持久化闭环接上。**5** 展开一层：本地缓存（Caffeine `refreshAfterWrite` + singleflight）把热点值挡在限流之前，热点参数规则兜住"缓存也失守"的尾部——顺序上永远让缓存吃正常流量、限流器只面对超额流量，两者阈值不应互相抄。加一条**6**：热点限流触发的 `ParamFlowException` 在 §4.1 之外还应有**被限值维度的指标**（哪个 id 被限了多少），否则大促复盘只知道"限了"不知道"谁被限"。

# 9. 相关阅读

Slot 进链机制与 SPI 装配在 `15_slot_chain_source.md`/`19_spi_extension.md`；每值窗口的统计底座在 `18_statistics_engine.md`；集群模式全貌在 `14_dynamic_rules.md` §10；Web 资源与注解资源的 args 来源差异在 `10_spring_cloud_integration.md`。
