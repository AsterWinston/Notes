
# 1. 概述

授权规则（Authority Rule）根据**调用方来源**（origin）进行黑白名单控制。与流控规则配合使用，可以实现**服务间调用的权限管控**。

```java
// 设置调用来源
ContextUtil.enter(resourceName, origin);  // origin = "appA"
Entry entry = SphU.entry(resourceName);
// ... 业务逻辑
entry.exit();
```

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

## 2.2 授权类型

| 类型 | 常量 | 含义 |
|------|------|------|
| 白名单 | `AUTHORITY_WHITE` | 只有白名单中的来源可以通过 |
| 黑名单 | `AUTHORITY_BLACK` | 黑名单中的来源被拒绝 |

# 3. AuthorityRule 配置

```java
AuthorityRule rule = new AuthorityRule();
rule.setResource("getUser");                       // 保护的资源
rule.setLimitApp("appA,appB");                     // 来源应用列表
rule.setStrategy(RuleConstant.AUTHORITY_WHITE);    // 白名单模式
```

**白名单模式**：

```
请求来源 = appA → getUser → 在白名单中 → 通过 ✓
请求来源 = appC → getUser → 不在白名单中 → 拒绝 ✗
```

**黑名单模式**：

```
请求来源 = appB → getUser → 在黑名单中 → 拒绝 ✗
请求来源 = appC → getUser → 不在黑名单中 → 通过 ✓
```

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

# 7. 最佳实践

1. **白名单优先**：未知来源默认拒绝，已知来源显式放行
2. **配合流控**：授权规则决定"谁能调用"，流控规则决定"调用多少"
3. **origin 设计**：使用应用名（如 `order-service`），不要用 IP
4. **不要替代认证**：授权规则是轻量调用方控制，不能替代 OAuth2/JWT 认证
5. **兜底处理**：`parseOrigin` 返回未知来源时，根据安全策略决定默认拒绝或放行
