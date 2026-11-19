# 1. 概述

授权规则（Authority Rule）根据**调用方来源**（origin）进行黑白名单控制。与流控规则配合使用，可以实现**服务间调用的权限管控**。

```java
// 设置调用来源
ContextUtil.enter(resourceName, origin);  // origin = "appA"
Entry entry = SphU.entry(resourceName);
// ... 业务逻辑
entry.exit();
```

## 1.1 一句话定位与它的适用边界

授权规则回答的是"**这个调用方准不准进**"——纯放行/拦截，没有速率概念；资源级流控的 `limitApp` 回答"**这个调用方准进多少**"。两者共用同一套 origin 数据源（§2.1），所以治理上通常成对出现：黑名单先斩掉不该来的，按来源流控再限住该来的量。边界要认清：origin 是**自报家门**（头/参数里写什么就算什么，§2.2），不校验真实性——它是流量治理里的"粗围栏"，防误调不防恶意调，安全边界始终属于认证体系（§7.4）。链上位置在 AuthoritySlot（FlowSlot 之前，`2_architecture.md` §3.3），被拒抛 `AuthorityException`，且**被拦请求不进任何通过统计**（§4.4 推论）。

# 2. 核心概念

## 2.1 调用来源（origin）

origin 是 Sentinel 中标识调用方来源的字符串，通常设为应用名：

```java
// 通过 RequestOriginParser 解析来源
@Component
public class MyRequestOriginParser implements RequestOriginParser {
    @Override
    public String parseOrigin(HttpServletRequest request) {
        // 从 Header 获取调用方应用名
        return request.getHeader("X-Origin-App");
    }
}
```

## 2.2 origin 的三条来源通道与"没传"的语义

origin 进入 Context 的全部通道：① **手工 API**：`ContextUtil.enter(name, origin)` 第二参（§1 示例）；② **Web 适配器**：`RequestOriginParser` 解析请求（SCA 自动装配容器里的实现 Bean；没有注册 parser 时默认实现读 `x-sentinel-request-origin` 头，都没有则 origin 为空串）；③ **RPC 适配器**：Dubbo/gRPC 从调用方隐式附件取 `Constants.REQUEST_ORIGIN`，即调用方的 `project.name`/`csp.sentinel.app.type` 配置值——**Feign 不在③里**，Feign→Feign 的 origin 不会自动传播，要靠调用方拦截器塞头 + 提供方 parser 读头这条手工链（Feign 系列同款纪律，`13_feign_integration.md`）。"没传"的语义必须钉死：**空 origin 是一个合法来源值，匹配规则时按空串处理**——白名单规则下空 origin 一律被拒（不在名单里），黑名单规则下一律放行（不在名单里），这是"来源限流不生效"排查（`3_quick_start.md` §6.2）的本体。origin 是**请求级**的：Context 定了 origin，该请求内所有嵌套 `SphU.entry` 共享它——黑白名单对内部资源同样生效，配置时想清楚资源粒度。

## 2.3 授权类型

| 类型 | 常量 | 含义 |
|------|------|------|
| 白名单 | `AUTHORITY_WHITE` | 只有白名单中的来源可以通过 |
| 黑名单 | `AUTHORITY_BLACK` | 黑名单中的来源被拒绝 |

## 2.4 limitApp 的三种写法与匹配语义

`limitApp` 不是"一个来源"，是**逗号分隔的来源列表**，另有三个特殊值（真实匹配逻辑见 §5.1 还原）：

```text
"appA,appB"    多个具名来源
"*"            匹配所有来源（含空 origin）——黑名单里等于全拦、白名单里等于全放
"other"        特指"名单外其他所有来源"（沿用 FlowRule 语义体系）
```

黑名单 + `other` 是"只许 appA、其余全拒"的又一种写法，与白名单 `[appA]` 语义相同——但两者**空 origin 归属不同**（白名单拒空、other 是否含空取决于解析，实践上把需要拦截匿名流量的一方显式列进名单），混用两套表达前先对空 origin 做一次实测。

# 3. AuthorityRule 配置

```java
AuthorityRule rule = new AuthorityRule();
rule.setResource("getUser");                       // 保护的资源
rule.setLimitApp("appA,appB");                     // 来源应用列表
rule.setStrategy(RuleConstant.AUTHORITY_WHITE);    // 白名单模式
```

**白名单模式**：

```text
请求来源 = appA → getUser → 在白名单中 → 通过 ✓
请求来源 = appC → getUser → 不在白名单中 → 拒绝 ✗
```

**黑名单模式**：

```text
请求来源 = appB → getUser → 在黑名单中 → 拒绝 ✗
请求来源 = appC → getUser → 不在黑名单中 → 通过 ✓
```

## 3.1 同资源多条规则的叠加语义

AuthorityRuleManager 按资源聚合规则列表（与流控一样是多条并存），**任一规则拒绝即拦截**（短路求值，第一条抛异常后面不看）。所以"白名单允许 appA"+"黑名单禁止 appA"同配一个资源 = appA 进不来（黑名单那条先撞上或后撞上都会拒），**规则集没有优先级、没有例外语法**——想表达"白名单全体但豁免某人"只能把豁免者从黑名单资源规则里拆走或给它单配资源名。跨规则的一致性审计（谁还能进这个资源）建议做成小脚本读数据源 JSON 回答，规则一多人工脑补必错。

# 4. 完整示例

## 4.1 配置 Origin Parser

```java
@Component
public class HeaderOriginParser implements RequestOriginParser {

    @Override
    public String parseOrigin(HttpServletRequest request) {
        // 1. 从请求头获取
        String origin = request.getHeader("X-Source-App");
        if (origin != null) return origin;

        // 2. 从请求参数获取
        origin = request.getParameter("sourceApp");
        if (origin != null) return origin;

        // 3. 默认值
        return "unknown";
    }
}
```

## 4.2 配置授权规则

```java
@PostConstruct
public void initAuthorityRules() {
    List<AuthorityRule> rules = new ArrayList<>();

    // 白名单：只允许 appA 和 appB 调用
    AuthorityRule whiteRule = new AuthorityRule();
    whiteRule.setResource("getUser");
    whiteRule.setLimitApp("appA,appB");
    whiteRule.setStrategy(RuleConstant.AUTHORITY_WHITE);
    rules.add(whiteRule);

    // 黑名单：禁止 appX 调用
    AuthorityRule blackRule = new AuthorityRule();
    blackRule.setResource("deleteUser");
    blackRule.setLimitApp("appX");
    blackRule.setStrategy(RuleConstant.AUTHORITY_BLACK);
    rules.add(blackRule);

    AuthorityRuleManager.loadRules(rules);
}
```

## 4.3 控制台配置

在 Sentinel Dashboard 中：
1. 进入**授权规则**页面
2. 新增规则 → 填写：资源名、流控应用（来源列表）、授权类型

## 4.4 示例的三处工程注脚

**① parser 的返回纪律**：示例的三级回退是好的，但 `"unknown"` 兜底值要在规则侧显式表态——匿名流量是拒是放必须由白/黑名单里的 `unknown`（或依赖 `other` 的写法）说清，否则它作为"又一个来源"静默绕过你的具名名单。parser 每请求执行、开销敏感操作（正则/查表）自己做缓存；返回 `null` 会被按空处理，别指望 null 触发 NPE 暴露问题。**② 懒注册的窗口**：授权规则在资源**首次被统计**前后才建 OriginNode（`2_architecture.md` §2.4），但判定本身不依赖节点存在（规则匹配是字符串级），不像来源限流要 OriginNode 计数——所以授权规则是五类规则里"启动即生效"最干净的，`@PostConstruct` 配完第一批请求就会被拦，没有统计预热期。**③ 被拒请求的可见性**：AuthorityException 计入 block 指标但不计入 pass/error（`5_circuit_breaker.md` §4.1 同源规则），监控上"该资源 QPS 掉了但调用方报错率没涨"就是被拦在门外——给被拒流量单独打点（自定义 BlockExceptionHandler 计数）是权限治理常态需求。

# 5. AuthoritySlot 源码

```java
public class AuthoritySlot extends AbstractLinkedProcessorSlot<DefaultNode> {

    @Override
    public void entry(Context context, ResourceWrapper resourceWrapper,
                      DefaultNode node, int count, boolean prioritized, Object... args) {
        // 根据资源名获取授权规则列表
        List<AuthorityRule> rules = AuthorityRuleManager.getRules(resourceWrapper.getName());
        for (AuthorityRule rule : rules) {
            String origin = context.getOrigin();  // 当前请求的来源
            if (rule.getLimitApp().equals(origin)) {
                if (rule.getStrategy() == RuleConstant.AUTHORITY_BLACK) {
                    throw new AuthorityException(origin + " is in blacklist");
                }
            } else {
                if (rule.getStrategy() == RuleConstant.AUTHORITY_WHITE) {
                    throw new AuthorityException(origin + " is not in whitelist");
                }
            }
        }
        fireEntry(context, resourceWrapper, node, count, prioritized, args);
    }
}
```

## 5.1 【勘误】真实匹配不是 equals，是 token 集合

示例的 `getLimitApp().equals(origin)` 若当真，`"appA,appB"` 这种多来源配置永远匹配不上任何一个 origin——1.8.x 的真实实现（`AuthorityRuleManager.convertAndInit` + `AuthorityStateController`，还原注记）是**加载期把 limitApp 按逗号拆成 `Set<String>` 存进规则条目**，运行期 `matchToken(origin, tokens)`：先精确 contains，再处理 `*`（视为匹配一切）与 `other` 归组。三点由此修正的心智模型：**① 逗号列表合法且是标准写法**（§2.4）；**② 拆分成本付在规则加载期**，热路径是 Set.contains，origin 基数再大无性能议题；**③ 规则 JSON 手写时 limitApp 里的空格不会自动 trim**（`"appA, appB"` 中 `" appB"` 是另一个 token，匹配不上 appB）——Dashboard 表单会规整，手写文件要自己保证。其余骨架语义（逐条短路、黑白分支、空 origin 走 else 分支被白名单拒）示例是对的。

# 6. Gateway 中的授权

在 Spring Cloud Gateway 中，`RequestOriginParser` 通过 ServerWebExchange 实现：

```java
@Component
public class GatewayOriginParser implements RequestOriginParser {

    @Override
    public String parseOrigin(HttpServletRequest request) {
        // Gateway 场景下，origin 通常从 Header 获取
        return request.getHeader("X-Gateway-Origin");
    }
}

// 或用 SentinelGatewayFilter 配合
@Bean
public GlobalFilter sentinelGatewayFilter() {
    return new SentinelGatewayFilter();
}
```

## 6.1 【勘误】Gateway 场景接口不是这个

代码块第一段的实现**在 Gateway 里根本不会被调用**：Gateway 基于 WebFlux，Servlet 的 `HttpServletRequest` 不存在；`sentinel-spring-cloud-gateway-adapter` 用的是自己的回调体系（`SentinelSCGAutoConfiguration`/`WebFluxCallbackManager` 一族，注册函数式 parser，入参 `ServerWebExchange`），§2.2 的 Servlet 版接口只在 MVC/WebFlux 应用侧成立。正确姿势与资源命名（route id / API 分组、`SentinelGatewayBlockExceptionHandler`）在 `12_gateway_integration.md` 展开，本篇只纠正"两份 parser 接口同名不同物、别跨场景抄类"这一条。另一个 Gateway 特有的 origin 语义：经网关进来的流量 origin 天然是**网关自己**（下游只见到网关 IP/头），要做"终端调用方"级授权，必须网关侧把来源写进透传头、下游 parser 读取——两层名单的口径要事先约定。

# 7. 最佳实践

1. **白名单优先**：未知来源默认拒绝，已知来源显式放行
2. **配合流控**：授权规则决定"谁能调用"，流控规则决定"调用多少"
3. **origin 设计**：使用应用名（如 `order-service`），不要用 IP
4. **不要替代认证**：授权规则是轻量调用方控制，不能替代 OAuth2/JWT 认证
5. **兜底处理**：`parseOrigin` 返回未知来源时，根据安全策略决定默认拒绝或放行

## 7.1 五条实践各配一条落地判据

**第 1 条的例外**：公共服务入口（网关、开放 API）无法穷举来源，白名单会误伤——这类资源退回黑名单 + 按来源限流，把"拒绝"交给认证层。**第 3 条"不要用 IP"**的理由要补全：容器网络下 IP 不复用不可读、NAT 后多应用同 IP、名单维护跟不上扩缩容——应用名是唯一稳定身份；确需网络维隔离用 NetworkFile 类规则思路（1.8.3+ 的 `AuthorityBaseRule` 无此能力，别硬造）。**第 5 条的机制支点**：白名单 + parser 兜底 `"unknown"` = 默认拒绝，这组是"安全缺省"；黑名单 + 兜底 = 默认放行，这组是"可用缺省"——一个资源的两种缺省别混配，且**新增资源时缺省走哪个要进评审清单**（新资源没配规则时是全放行的，授权体系靠"资源全量纳管"才有闭环，漏一个端点=洞一个端点）。fail-open 全局提醒：授权规则解析失败（JSON 坏/字段错）时 Manager 保留上一份或空表——**空表=全放行**，改完规则文件必须验证加载成功（`FlowRuleManager.getRules()` 同款自检，`14_dynamic_rules.md` §10）。

# 8. 相关阅读

origin 的统计侧（OriginNode、按来源限流 limitApp）在 `4_flow_control.md` §2 与 `16_flow_control_source.md`；Slot 位置与异常链上优先级在 `15_slot_chain_source.md`；各适配器 parser 全景在 `10`/`11`/`12`/`13` 四篇集成；黑白名单误拦排障案例在 `24_troubleshooting.md`。
