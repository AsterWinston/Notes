
# 1. 路由概述

路由（Router）负责在负载均衡之前，根据规则对 Provider 列表进行**过滤**，是流量管控的核心手段。

```java
public interface Router extends Comparable<Router> {
    URL getUrl();
    <T> List<Invoker<T>> route(List<Invoker<T>> invokers, 
        URL url, Invocation invocation) throws RpcException;
    int getPriority();
    boolean isRuntime();   // 是否需要实时过滤
    boolean isForce();     // 无匹配时是否强制返回空
}
```

**路由执行链**：

```
目录获取的全量 Invoker List
    │
    ├── ConditionRouter     (条件路由)
    ├── TagRouter           (标签路由)
    ├── ServiceRouter       (服务路由 - 3.x)
    ├── MeshAppRouter       (Mesh 应用路由 - 3.x)
    ├── MeshEnvRouter       (Mesh 环境路由 - 3.x)
    ├── AppRouter           (应用路由 - 3.x)
    └── ScriptRouter        (脚本路由)
    │
    ▼
过滤后的 Invoker List → 交给 LoadBalance
```

# 2. ConditionRouter（条件路由）

## 2.1 规则语法

```
条件路由规则字符串:
  "规则1 => 规则2"
  规则1: 匹配条件 (消费端匹配)
  规则2: 过滤条件 (提供者过滤)
```

## 2.2 规则格式

```
[服务消费者匹配条件] => [服务提供者过滤条件]

# 示例
host = 192.168.1.* => host = 192.168.1.*   # 优先调用同机房
method = find* => host = 192.168.2.*        # find* 方法优先调用机房2
host != 192.168.2.*                         # 白名单（不支持 2 号机房 Provider）
                                            # 黑名单（无匹配条件时作用于所有 Consumer）
```

## 2.3 源码实现

```java
public class ConditionRouter implements Router {
    private static final Pattern ROUTE_PATTERN = 
        Pattern.compile("([&!=,]*)s*=>s*(.*)");
    
    // 解析后的条件
    private final Map<String, MatchPair> whenCondition;   // 匹配条件
    private final Map<String, MatchPair> thenCondition;   // 过滤条件

    public ConditionRouter(URL url) {
        String rule = url.getParameter("rule");
        // 解析 "key1=val1 & key2!=val2 => key3=val3"
        Matcher matcher = ROUTE_PATTERN.matcher(rule);
        matcher.matches();
        
        whenCondition = parseCondition(matcher.group(1));  // 消费端匹配
        thenCondition = parseCondition(matcher.group(2));  // 提供者过滤
    }

    @Override
    public <T> List<Invoker<T>> route(List<Invoker<T>> invokers, 
            URL url, Invocation invocation) {
        
        // 1. 消费端不匹配 → 不应用该路由规则
        if (!matchWhen(url, invocation)) {
            return invokers;
        }
        
        // 2. 匹配了消费端，对 Provider 列表按 thenCondition 过滤
        List<Invoker<T>> result = new ArrayList<>();
        for (Invoker<T> invoker : invokers) {
            if (matchThen(invoker.getUrl(), url)) {
                result.add(invoker);
            }
        }
        
        // 3. 过滤后为空 AND 非强制 → 返回全量（不路由）
        if (result.isEmpty() && !force) {
            return invokers;
        }
        
        return result;
    }

    private boolean matchWhen(URL url, Invocation invocation) {
        // 匹配:
        //   host = 192.168.1.*    → url.getHost() matches "192.168.1.*"
        //   method = find*        → invocation.getMethodName() matches "find*"
        //   application = myapp   → url.getParameter("application") == "myapp"
        return matchCondition(whenCondition, url, invocation);
    }

    private boolean matchThen(URL url, URL consumerUrl) {
        return matchCondition(thenCondition, url, null);
    }

    private static Map<String, MatchPair> parseCondition(String condition) {
        // "host=192.168.1.* & method!=find*"
        // → {host: {matches:["192.168.1.*"]}, method: {mismatches:["find*"]}}
        Map<String, MatchPair> map = new HashMap<>();
        for (String kv : condition.split("&")) {
            // 解析 key=value or key!=value
        }
        return map;
    }
}
```

## 2.4 配置示例

**管理后台配置**（自动推送到注册中心）：

```yaml
# 读写分离：读方法到性能较弱的服务器
rule: "method=get* => host=10.0.0.1*"

# 灰度发布：v2 版本只路由到 v2 Provider
rule: "consumer.version=2.0.0 => provider.version=2.0.0"

# 黑名单：排除某个 Provider
rule: "host!=192.168.1.100"

# 白名单：只允许特定 Provider
rule: "host=192.168.1.10*,host=192.168.1.20*"
```

# 3. TagRouter（标签路由）

## 3.1 概念

标签路由用于**流量分组隔离**，如灰度发布、多套环境隔离。

```
Consumer (tag=gray) → TagRouter → 只路由到 Provider (tag=gray)
Consumer (tag=prod) → TagRouter → 只路由到 Provider (tag=prod)
```

## 3.2 源码实现

```java
public class TagRouter implements Router {
    
    @Override
    public <T> List<Invoker<T>> route(List<Invoker<T>> invokers, 
            URL url, Invocation invocation) {
        
        String tag = getTag(invocation);
        
        if (StringUtils.isEmpty(tag)) {
            // 未指定 tag → 只调用无 tag 的 Provider
            return filterTag(invokers, "");
        }
        
        // 优先匹配同 tag 的 Provider
        List<Invoker<T>> tagged = filterTag(invokers, tag);
        if (!tagged.isEmpty()) {
            return tagged;
        }
        
        // 同 tag 没有 Provider → 降级到无 tag Provider
        List<Invoker<T>> untagged = filterTag(invokers, "");
        if (!untagged.isEmpty()) {
            return untagged;
        }
        
        // 无 tag Provider 也没有 → 抛出异常
        throw new RpcException("No provider for tag: " + tag);
    }

    private String getTag(Invocation invocation) {
        // 1. 从 RpcContext 获取（代码 set）
        String tag = RpcContext.get().getAttachment("dubbo.tag");
        // 2. 从 URL 参数获取（全局配置）
        if (StringUtils.isEmpty(tag)) {
            tag = url.getParameter("dubbo.provider.tag");
        }
        return tag;
    }
}
```

## 3.3 配置

Provider 端：

```yaml
dubbo:
  provider:
    tag: gray     # 标记为灰度节点
```

Consumer 端：

```yaml
dubbo:
  consumer:
    tag: gray     # 只调用灰度节点
```

或代码中动态设置：

```java
RpcContext.getClientAttachment().setAttachment("dubbo.tag", "gray");
User user = userService.getUser(1L);
```

# 4. ServiceRouter（服务路由 - Dubbo 3）

Dubbo 3 应用级发现下，Provider 实例上可能有多个服务，需要根据服务名过滤：

```java
public class ServiceRouter implements Router {
    
    @Override
    public <T> List<Invoker<T>> route(List<Invoker<T>> invokers, 
            URL url, Invocation invocation) {
        
        // 在应用级发现下，一个实例可能有多个服务
        // 过滤出真正提供目标服务的实例
        return invokers.stream()
            .filter(inv -> inv.getUrl().getServiceInterface()
                .equals(invocation.getServiceName()))
            .collect(Collectors.toList());
    }
}
```

# 5. 应用级路由（AppRouter - Dubbo 3）

```java
public class AppRouter implements Router {

    @Override
    public <T> List<Invoker<T>> route(List<Invoker<T>> invokers, 
            URL url, Invocation invocation) {
        
        // 根据应用名筛选 Provider
        String targetApp = url.getParameter("application");
        if (StringUtils.isNotEmpty(targetApp)) {
            return invokers.stream()
                .filter(inv -> targetApp.equals(
                    inv.getUrl().getRemoteApplication()))
                .collect(Collectors.toList());
        }
        return invokers;
    }
}
```

# 6. ScriptRouter（脚本路由 - 已废弃）

通过脚本语言（Java/Groovy/JavaScript）动态决策路由：

```java
public class ScriptRouter implements Router {
    
    @Override
    public <T> List<Invoker<T>> route(List<Invoker<T>> invokers, 
            URL url, Invocation invocation) {
        
        String script = url.getParameter("rule");
        
        // 将 invokers 和 invocation 暴露给脚本
        ScriptEngine engine = new ScriptEngineManager()
            .getEngineByName("javascript");
        engine.put("invokers", invokers);
        engine.put("invocation", invocation);
        engine.put("context", RpcContext.get());
        
        // 脚本返回过滤后的 Invoker 列表
        return (List<Invoker<T>>) engine.eval(script);
    }
}
```

**注意**：ScriptRouter 在 Dubbo 3.2+ 中已标记 `@Deprecated`，推荐使用 ConditionRouter 或 StateRouter。

# 7. StateRouter（状态路由 — Dubbo 3 新特性）

StateRouter 是 Dubbo 3 引入的新一代路由机制，基于**状态机**实现更灵活的路由策略：

```java
public interface StateRouter<T> {
    void notify(List<URL> urls);
    List<Invoker<T>> route(List<Invoker<T>> invokers, 
        URL url, Invocation invocation, 
        boolean needToPrintMessage, Holder<String> messageHolder);
    boolean isRuntime();
    boolean isForce();
}
```

与旧 Router 的区别：

| 特性 | Router (旧) | StateRouter (新) |
|------|-----------|----------------|
| 路由粒度 | 接口级 | 应用级 |
| 状态感知 | 无 | 有 (健康/不健康) |
| 性能 | 每次全量过滤 | 缓存过滤结果 |
| 扩展性 | 有限 | 更好 |

# 8. 路由规则的生命周期

## 8.1 发布流程

```
1. 管理后台 (Dubbo Admin) 创建路由规则
2. 持久化到注册中心（如 Nacos/ZK 的 /routers 节点）
3. RegistryDirectory 通过 notify() 收到变更通知
4. refreshRouter() → 重新构建 Router Chain
5. 下一次 RPC 调用时生效
```

## 8.2 RegistryDirectory 中的 Router 管理

```java
public class RegistryDirectory<T> extends AbstractDirectory<T> {
    
    private void refreshRouter(List<URL> routerUrls) {
        if (routerUrls.isEmpty()) {
            return;
        }
        // 将 URL 转换为 Router 实例
        List<Router> newRouters = toRouters(routerUrls);
        // CAS 替换
        this.routers = newRouters;
    }

    @Override
    public List<Router> getRouters() {
        return routers;
    }
}
```

# 9. 路由规则实战

## 9.1 灰度发布

```yaml
# 条件路由规则
# 消费端 version=v2 的请求 → 只路由到 version=v2 的提供者
enabled: true
force: false           # 非强制：无 v2 Provider 时降级到其他版本
runtime: true          # 运行时实时匹配
priority: 1
rule: "consumer.version=2.0.0 => provider.version=2.0.0"
```

## 9.2 同机房优先

```yaml
# host 匹配规则
# 同机房的 Consumer 优先访问同机房 Provider
enabled: true
force: false
runtime: true
priority: 2
rule: "host=192.168.1.* => host=192.168.1.*"
```

## 9.3 流量隔离

```yaml
# 标签路由
# 设置 Provider 标签
dubbo:
  provider:
    tag: zone-a
```

```yaml
# 设置 Consumer 标签
dubbo:
  consumer:
    tag: zone-a
```

# 10. 自定义 Router

```java
public class CustomRouter implements Router {
    private final URL url;
    private final int priority;
    private final boolean force;

    public CustomRouter(URL url) {
        this.url = url;
        this.priority = url.getParameter("priority", 0);
        this.force = url.getParameter("force", false);
    }

    @Override
    public <T> List<Invoker<T>> route(List<Invoker<T>> invokers, 
            URL url, Invocation invocation) {
        
        // 自定义路由逻辑
        // 例如：根据请求中的 userId 哈希值决定路由到哪个 Provider
        String userId = invocation.getAttachment("userId");
        if (StringUtils.isNotEmpty(userId)) {
            int hash = Math.abs(userId.hashCode()) % invokers.size();
            return Collections.singletonList(invokers.get(hash));
        }
        return invokers;
    }

    @Override
    public int getPriority() { return priority; }

    @Override
    public boolean isRuntime() { return true; }

    @Override
    public boolean isForce() { return force; }

    @Override
    public URL getUrl() { return url; }
}
```

SPI 注册：

```
# META-INF/dubbo/org.apache.dubbo.rpc.cluster.RouterFactory
custom=com.example.CustomRouterFactory
```

通过 Dubbo Admin 管理后台发布路由规则即可生效。
