# 1. 概述

Sentinel 支持将规则存储在外部数据源（Nacos、Apollo、ZooKeeper 等），实现**规则持久化**和**动态推送**，避免应用重启后规则丢失。

# 2. 推送模式对比

```text
原始模式（内存）:
  Dashboard ──推送──→ Client 内存 ──重启──→ 规则丢失 ✗

Pull 模式:
  Client ──定时拉取──→ Nacos/Apollo ──重启──→ Client 重新拉取 ✓

Push 模式:
  Nacos/Apollo ──变更推送──→ Client ──重启──→ 数据源推送 ✓
```

## 2.1 三模式的两个隐藏维度：时延与失联行为

对比表按"重启后还在不在"切分，运维上更要紧的是另外两列：**生效时延**（Push 模式经配置中心 watch，端到端亚秒~秒级；Pull 模式吃轮询周期，止血操作最慢等一个 interval；Dashboard 裸推即时但覆盖面=逐机器串行推送）；**数据源失联时行为**——三个都是 fail-open，且形态不同：启动时读不到数据源，Sentinel 用**空规则**继续跑（不是拒绝启动，除非配了 `rule-property` 初始化校验），运行中 watch 断连则**保留最后一份内存规则**继续生效。由此得出规则系统的第一审计问题："此刻生效的规则是谁写的"——数据源、代码 loadRules、Dashboard 裸推三股势力在 `FlowRuleManager` 汇合成一份内存态（`2_architecture.md` §6.1 的整表替换时序），**排障永远以目标机器实际内容为准**（`FlowRuleManager.getRules()`/Command Center `/rules`，`9_dashboard.md` §8.1），任何"配置文件里写了"都不构成证据。

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

## 3.3 配置键的三个语义要点

**① 数据源名（flow/degrade/…）是自定义标签**，SCA 按 `datasource.<任意名>.nacos` 的后缀类型装配，真正决定转换器的是 `rule-type`——写漏 `rule-type` 时 SCA 按数据源名猜测（flow 猜对、`param-flow` 这类带横杠名猜错则规则加载失败且只在 debug 日志可见），五个块都显式写全是对的。**② namespace 用命名空间 ID 不是显示名**（dev 环境名≠ID，填显示名静默读不到——Nacos 侧配置在 public 而应用指着一个不存在的 ID，症状=启动空规则）。**③ 同 rule-type 出现两个数据源是雷**：两份 `DynamicSentinelProperty` 各自独立把 `loadRules` 打进同一个 Manager（整表替换，`2_architecture.md` §6.1），谁后到谁赢、watch 回调还会互相踩踏——一个规则类型全局只接一个数据源。

## 3.4 Nacos 中的规则 JSON

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

## 3.5 JSON 的加载期行为：坏条目静默丢、空文件清空

转换器对每个元素单独反序列化，**非法条目跳过、合法条目照常生效**（`"grade": "QPS"` 这种错会整条丢；全文件 JSON 语法错则回调收到 null/异常，Manager 保持旧表或空表）——"改了没报错但没生效"的第三种形态。空数组 `[]` 是合法的"清空全部规则"，误删文件内容等于全局裸奔，数据源侧要有版本回滚意识（Nacos 历史版本恰好免费给了这条）。字段名 camelCase 对应 Java 属性（`refResource`/`warmUpPeriodSec`/`paramFlowItemList`…），Dashboard 表单字段与之一一映射（`3_quick_start.md` §5.4 的加载验证纪律、`4_flow_control.md` §5.1 字段表）。

## 3.6 通过 API 发布规则

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

## 3.7 OpenAPI 发布的两个生产前提

Nacos 服务端开了鉴权（生产标配）时此 curl 会 403，要先 `POST /nacos/v1/auth/login` 换 accessToken 拼进请求；`content` 大 JSON 建议 `-d @rules.json` 走文件而非内联转义（示例的裸转义超过几十行极易引号事故）。更稳的工程形态是流水线里用 nacos-client SDK 发布（天然带鉴权/重试/校验），curl 留给应急值班手册——发布后验证三步：Nacos 控制台确认版本号 → 应用日志 `configUpdate`（§7 监听器打的）→ 目标机器 `/rules` 现值（`9_dashboard.md` §8.1）。

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

## 4.1 键位形态差异与灰度用法

Apollo 模型里 dataId/group 被折叠成 `namespace + key`（一个 namespace 下多个 key 分别挂各规则类型，键名族如 `flow-rules-key`/`degrade-rules-key` 逐类不同，抄 Nacos 习惯配不出 Apollo 源）。`default-flow-rule-value` 承担 §2.1 的启动兜底语义——写 `"[]"` 是显式"读不到就空规则"，想留种子规则可以填一份保守 JSON（与 §3.3③ 不冲突，因为它是同源的初值不是第二数据源）。Apollo 的**灰度发布/秒级回滚**恰好补上 Sentinel 数据源模型缺的"按实例生效"能力：规则灰度交给 Apollo 灰度规则，比给 Sentinel 造实例级开关正统得多——用 Apollo 的团队值得把这条写进变更流程。

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

## 5.1 zk 源的两个提醒

`group-id` 在 zk 语境**无意义可删**（数据组织是 znode path，示例留着纯属从 Nacos 模板抄惯）；`path` 就是规则 JSON 存放的节点内容。选型现状要有数：ZK 源是早期生态产物，新团队用 Nacos 居多；已有 zk 基建（如同时跑 dubbo）复用它合理，但注意 zk 的 watch 一次性触发语义已由 curator 封装成持久监听，节点删除事件=规则清空（§3.5 的 `[]` 等价事故面，znode 权限收紧到发布流水线可写）。

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

## 6.1 【勘误】接口方法是 readConfig()，且惯用写法是继承 AbstractDataSource

两处修正：① `ReadableDataSource` 的读方法是 **`T readConfig() throws Exception`**（示例的 `loadConfig()` 编译不过或只是没实现到接口上）；② 官方 adapter 的实现范式不直接 implement 双接口，而是**继承 `AbstractDataSource<String, List<FlowRule>>`**——基类持有 `SentinelProperty` 并提供 `updateValue()`，子类在存储回调里调它：

```java
// 正规形态（还原）：构造时传入转换器
public class MyDataSource extends AbstractDataSource<String, List<FlowRule>> {
    public MyDataSource(String configKey, Function<String, List<FlowRule>> ruleParser) {
        super(ruleParser);                                  // parser 负责 JSON→规则
        this.configKey = configKey;
        loadInitialConfig();                                // 首读 + updateValue(current)
        myStorage.watch(configKey, content -> {             // 变更回调
            T parsed = parser.apply(content);
            updateValue(parsed);                            // 失败会抛 IllegalStateException（fail 语义可配）
        });
    }
}
```

转换器复用 `RuleConstant` 家族的 `jsonConverter`（各 Manager 提供 `FlowRuleManager.CONVERTER` 一类静态件），别自己重写解析——§3.5 的逐条容错行为要靠它保持一致。`WritableDataSource` 只在"控制台/Dashboard 要写回"时才需要（§7.1 写回链路），纯读场景可以不实现。

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

## 7.1 这个示例混用了两套接线，单用一种

监听器里手动 `FlowRuleManager.loadRules(rules)` 与末行 `register2Property(property)` 是**重复接线**：`register2Property` 会给同一 property 挂上 Manager 内置的 `FlowPropertyListener`（它负责调 loadRules 并做校验转换）——两条同时在，每次变更 load 两遍（幂等但日志翻倍，且手写 listener 若吞异常会让人误以为"接线坏了"）。正确心智：**`register2Property` 是唯一的"规则生效通道"**，自己的 listener 只配观测/审计（§9.5 的变更日志用它打，但里面不碰 Manager）。两个时序细节：`addListener` 时 property 已有值会**立刻补发一次 configLoad**（示例先 addListener 后 register 的顺序安全的原因）；`updateValue(null)` 在 DynamicSentinelProperty 的 `isWildcard`/相等性判断下可能不发事件——"改了配置没回调"要查新旧值是否 equals、以及解析层是否返回了 null（§3.5 的全文件坏 JSON 形态在 Property 层的表现就是 updateValue(null) 保持旧值，fail-open）。

# 8. 规则推送流程

```text
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

## 8.1 图的下游补全：load 之后到生效之间还有三跳

Manager 收下规则不是终点：**① 校验与索引重建**（`convertAndInit` 一族：逐条合法性过滤、按资源建 Map，§3.5 的逐条丢弃发生在这）；**② 既有代理零感知**——内存规则表被 Checker 每次实时读取（FlowSlot 拿 resource 现查 Map），**不需要**重建 Feign 代理/Entry，这是"规则秒级生效"与"配置项要重启"的分界（组件 Bean 是装配期快照，规则是运行期查表，`2_architecture.md` §3.1 的缓存语义只绑组件不绑规则）；**③ 集群规则的第三通道**——thresholdType=GLOBAL 的规则还要经 Dashboard/推送把 flowId 同步到 token server 侧（server 没收到同 flowId 配置时 client 请求被拒或退化，§9.6），普通单机规则没这一跳。多实例一致性：watch 断连期间实例用的是旧值，Nacos 长轮询重连后**自动补拉最新**（失联期间发布的变更不丢）；但**发布瞬间**各机器收到回调有秒级参差，大促"整点生效"类规则提前 5 分钟发、留验证窗口。

# 9. 最佳实践

1. **生产环境必须持久化**：不配置数据源则规则重启丢失
2. **Nacos 优先**：Spring Cloud Alibaba 生态中 Nacos 是首选
3. **规则变更验证**：发布前在测试环境验证 JSON 格式
4. **分组隔离**：不同环境的规则使用不同的 Nacos namespace/group
5. **监听规则变更**：通过日志或 Metrics 确认规则推送生效
6. **常规规则变更**：推荐通过 API/Nacos Console 更新，不通过 Dashboard 原始模式

## 9.1 六条的落地增强

**1** 的完整表述含另一半：数据源解决"存活"，**写回通道**解决"控制台改动去哪"——裸 Dashboard 在接了数据源后仍是危险按钮（§2.1 三势力汇合点），要么改造 Dashboard 写回数据源（`9_dashboard.md` §5.2 三档谱系），要么在权限上禁用控制台编辑。**3** 升级为自动化：JSON schema/最简 parse 校验进 CI（§3.5 的静默丢条目机器测得出），资源名对簇点链路的"存在性断言"进发布流水线（`4_flow_control.md` §6.1 引用过的规则审计思路）。**4** 补一个反直觉点：namespace 隔离的是**配置存储**，隔离不了"同一应用名跨 namespace 被同两个环境注册进同一 Dashboard"的显示混淆——环境前缀写进 data-id（`${app}-flow-rules` 的 app 本身含 env 后缀）比只靠 namespace 更防呆。**5** 的具体挂点就是 §7.1 的观测 listener + 版本号日志（把 Nacos 配置 MD5 打进 configLoad/configUpdate，事后对账有据）。**6** 之外补一条边界：紧急止血允许 Dashboard 裸推，但按 `2_architecture.md` §6.1 纪律**24 小时内回写数据源**，且回写前禁止任何例行发布（否则被 watch 冲掉，止血效果随下一次发布蒸发）。

# 10. 相关阅读

规则流转全景在 `2_architecture.md` §6；"改规则是否要重启"的组件/规则分界在 `4_configuration` 语义的 Sentinel 版即 §8.1；集群流控 flowId 同步与 token server 治理在 `9_dashboard.md` §6.2；各规则类型 JSON 字段总表散布于 `4`~`8` 与 `12_gateway_integration.md` §8；加载失败排障案例在 `24_troubleshooting.md`。
