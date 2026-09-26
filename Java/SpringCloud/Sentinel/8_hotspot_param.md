
# 1. 概述

热点参数流控（Hotspot Parameter Flow Control）是对**资源调用中的参数**进行精细化限流。普通流控以资源整体为粒度，热点参数流控按**参数值**分别统计和限流。

**典型场景**：

```
商品详情接口：/product/detail?id=1001
  ├── id=1001（热门商品）→ QPS 限制 5000
  ├── id=1002（冷门商品）→ QPS 限制 100（默认）
  └── id=1003 → QPS 限制 100（默认）
```

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

# 5. ParamFlowSlot 执行流程

```java
public class ParamFlowSlot extends AbstractLinkedProcessorSlot<DefaultNode> {

    @Override
    public void entry(Context context, ResourceWrapper resourceWrapper,
                      DefaultNode node, int count, boolean prioritized,
                      Object... args) {
        List<ParamFlowRule> rules = ParamFlowRuleManager.getRules(resourceWrapper.getName());
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

# 8. 最佳实践

1. **适用场景**：商品详情、秒杀商品、热门内容 — 对热点资源的差异化防护
2. **阈值设定**：热点商品例外阈值 = 压测基准 × 1.5，普通商品 = 压测基准 × 0.5
3. **参数选择**：选择区分度高的参数作为热点维度（如商品ID、用户ID）
4. **避免过多例外项**：超过 100 个例外项考虑用其他策略（如缓存 + 流控组合）
5. **配合缓存**：热点参数限流 + 本地缓存，双重保护
