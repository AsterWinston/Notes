
# 1. 概述

Sentinel 支持将规则存储在外部数据源（Nacos、Apollo、ZooKeeper 等），实现**规则持久化**和**动态推送**，避免应用重启后规则丢失。

# 2. 推送模式对比

```
原始模式（内存）:
  Dashboard ──推送──→ Client 内存 ──重启──→ 规则丢失 ✗

Pull 模式:
  Client ──定时拉取──→ Nacos/Apollo ──重启──→ Client 重新拉取 ✓

Push 模式:
  Nacos/Apollo ──变更推送──→ Client ──重启──→ 数据源推送 ✓
```

# 3. Nacos 数据源

## 3.1 依赖

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-nacos</artifactId>
</dependency>
```

## 3.2 YAML 配置

```yaml
spring:
  cloud:
    sentinel:
      datasource:
        # 流控规则
        flow:
          nacos:
            server-addr: localhost:8848
            namespace: dev                      # Nacos 命名空间
            data-id: ${spring.application.name}-flow-rules
            group-id: SENTINEL_GROUP
            data-type: json                     # 规则格式
            rule-type: flow                     # 规则类型

        # 熔断规则
        degrade:
          nacos:
            server-addr: localhost:8848
            namespace: dev
            data-id: ${spring.application.name}-degrade-rules
            group-id: SENTINEL_GROUP
            data-type: json
            rule-type: degrade

        # 系统规则
        system:
          nacos:
            server-addr: localhost:8848
            namespace: dev
            data-id: ${spring.application.name}-system-rules
            group-id: SENTINEL_GROUP
            data-type: json
            rule-type: system

        # 授权规则
        authority:
          nacos:
            server-addr: localhost:8848
            namespace: dev
            data-id: ${spring.application.name}-authority-rules
            group-id: SENTINEL_GROUP
            data-type: json
            rule-type: authority

        # 热点规则
        param-flow:
          nacos:
            server-addr: localhost:8848
            namespace: dev
            data-id: ${spring.application.name}-param-flow-rules
            group-id: SENTINEL_GROUP
            data-type: json
            rule-type: param-flow
```

## 3.3 Nacos 中的规则 JSON

```json
// dataId: my-service-flow-rules, group: SENTINEL_GROUP
[
  {
    "resource": "getUser",
    "grade": 1,
    "count": 100,
    "strategy": 0,
    "controlBehavior": 0,
    "limitApp": "default"
  },
  {
    "resource": "createOrder",
    "grade": 1,
    "count": 50,
    "strategy": 0,
    "controlBehavior": 0
  }
]
```

## 3.4 通过 API 发布规则

```bash
curl -X POST "http://localhost:8848/nacos/v1/cs/configs" \
  -d "dataId=my-service-flow-rules" \
  -d "group=SENTINEL_GROUP" \
  -d "content=[
    {
      \"resource\": \"getUser\",
      \"grade\": 1,
      \"count\": 100,
      \"strategy\": 0,
      \"controlBehavior\": 0,
      \"limitApp\": \"default\"
    }
  ]"
```

# 4. Apollo 数据源

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-apollo</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    sentinel:
      datasource:
        flow:
          apollo:
            namespace-name: application       # Apollo namespace
            flow-rules-key: sentinel-flow-rules
            default-flow-rule-value: "[]"
```

# 5. ZooKeeper 数据源

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-zookeeper</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    sentinel:
      datasource:
        flow:
          zk:
            server-addr: localhost:2181
            path: /sentinel/flow/${spring.application.name}
            group-id: SENTINEL_GROUP
            data-type: json
            rule-type: flow
```

# 6. 自定义数据源

实现 `ReadableDataSource` 和 `WritableDataSource` 接口：

```java
public class MyDataSource implements ReadableDataSource<String, List<FlowRule>>,
                                      WritableDataSource<List<FlowRule>> {

    private final String configKey;
    private SentinelProperty<List<FlowRule>> property;

    @Override
    public List<FlowRule> loadConfig() {
        // 从自定义存储中读取规则
        String config = myStorage.read(configKey);
        return JSON.parseArray(config, FlowRule.class);
    }

    @Override
    public void write(List<FlowRule> rules) {
        // 将规则写回自定义存储
        String config = JSON.toJSONString(rules);
        myStorage.write(configKey, config);
    }

    @Override
    public void close() { }
}
```

注册自定义数据源：

```java
@PostConstruct
public void initDataSource() {
    MyDataSource ds = new MyDataSource("sentinel-flow-rules");
    FlowRuleManager.register2Property(ds.getProperty());
}
```

# 7. SentinelProperty 与监听器

```java
// SentinelProperty 是规则变更的核心抽象
SentinelProperty<List<FlowRule>> property = new DynamicSentinelProperty<>();

// 添加监听器
property.addListener(new PropertyListener<List<FlowRule>>() {
    @Override
    public void configLoad(List<FlowRule> rules) {
        // 首次加载
        log.info("Flow rules loaded: {} rules", rules.size());
    }

    @Override
    public void configUpdate(List<FlowRule> rules) {
        // 规则更新
        log.info("Flow rules updated: {} rules", rules.size());
        FlowRuleManager.loadRules(rules);
    }
});

// 更新规则
property.updateValue(newRules);
FlowRuleManager.register2Property(property);
```

# 8. 规则推送流程

```
┌──────────┐   变更规则   ┌──────────┐   推送通知    ┌──────────┐
│  Nacos   │ ──────────→ │  Client  │ ───────────→ │ Sentinel │
│ Console  │             │  Listener│              │ Property │
│  (管理员) │             │          │              │          │
└──────────┘             └──────────┘              └────┬─────┘
                                                        │ configUpdate()
                                                        ▼
                                                 ┌──────────────┐
                                                 │ FlowRuleManager│
                                                 │   .loadRules() │
                                                 └──────────────┘
```

# 9. 最佳实践

1. **生产环境必须持久化**：不配置数据源则规则重启丢失
2. **Nacos 优先**：Spring Cloud Alibaba 生态中 Nacos 是首选
3. **规则变更验证**：发布前在测试环境验证 JSON 格式
4. **分组隔离**：不同环境的规则使用不同的 Nacos namespace/group
5. **监听规则变更**：通过日志或 Metrics 确认规则推送生效
6. **常规规则变更**：推荐通过 API/Nacos Console 更新，不通过 Dashboard 原始模式
