# 1. Kernel 的职责

Kernel 是 Semantic Kernel 的中央容器与执行入口。它的价值在于**把服务、插件、记忆、过滤器集中装配**，使调用方只需面向 Kernel，而不必关心模型与函数的创建细节。理解 Kernel 的职责边界，是正确组织 SK 应用的第一步。

## 1.1 职责总览

### 1.1.1 结构

Kernel 承担五项职责：注册 AI 服务、注册插件、管理记忆、管理过滤器、提供调用入口。

```
Kernel = 中央容器
  ├── 注册 AI 服务（模型）
  ├── 注册插件（函数）
  ├── 管理记忆
  ├── 管理过滤器（拦截）
  └── 提供调用入口
```

### 1.1.2 构建代码

Kernel 通过 Builder 构建，注册 AI 服务与插件后即可使用。

```java
Kernel kernel = Kernel.builder()
        .withAIService(ChatCompletionService.class, chatService)
        .withPlugin(plugin)
        .build();
```

## 1.2 服务与插件管理

### 1.2.1 服务注册

AI 服务按接口类型注册，支持同类型多实现并按名称选择，便于多模型并存与故障切换。

```java
Kernel kernel = Kernel.builder()
        .withAIService(ChatCompletionService.class, primary, "primary")
        .withAIService(ChatCompletionService.class, backup, "backup")
        .build();
```

### 1.2.2 插件解析

Kernel 负责在函数调用时定位插件与函数，并把模型生成的参数解析为 Java 参数。函数名冲突时以插件名为命名空间区分。

```java
// 通过 插件名 + 函数名 定位
FunctionResult result = kernel.invokeAsync("weather", "getWeather",
        Map.of("city", "北京")).block();
```

## 1.3 生产建议

Kernel 建议在应用启动时构建一次并复用；若插件依赖请求级上下文（如租户），应通过参数或派生 Kernel 传递，而不是让插件持有可变全局状态。

## 1.4 Kernel 生命周期

### 1.4.1 构建期与运行期

Kernel 的构建期负责装配服务与插件，运行期只负责解析与调用。把可变配置（模型名、超时）放在构建期之后通过配置刷新注入，可避免频繁重建容器。

| 阶段 | 动作 | 是否可变 |
|------|------|----------|
| 构建期 | 注册服务、插件 | 否 |
| 运行期 | 解析函数、执行 | 是 |

### 1.4.2 线程安全

Kernel 在无状态前提下可被多线程共享；若插件持有可变状态，则需自行加锁或改为无状态设计。

```java
// 无状态插件：可安全共享
public class WeatherPlugin {
    // 仅依赖入参，无字段状态
}
```

# 2. 插件（Plugin）

插件是函数的分组容器，把相关能力聚合成一个可注册、可授权、可复用的单元。

## 2.1 插件定义

### 2.1.1 结构

```
Plugin = 一组相关函数的集合
  ├── 原生函数（Java 方法）
  └── 语义函数（Prompt 模板）
```

### 2.1.2 创建插件

插件由普通 Java 对象转换而来，插件名会成为函数命名空间。

```java
KernelPlugin plugin = KernelPluginFactory.createFromObject(
        new WeatherPlugin(), "weather");

Kernel kernel = Kernel.builder()
        .withAIService(ChatCompletionService.class, chatService)
        .withPlugin(plugin)
        .build();
```

## 2.2 原生函数

### 2.2.1 注解与参数

原生函数通过 `@DefineKernelFunction` 暴露，参数通过 `@KernelFunctionParameter` 描述。描述文本会进入函数 schema，直接影响模型的选择与传参质量。

```java
public class WeatherPlugin {

    @DefineKernelFunction(name = "getWeather",
            description = "查询指定城市的天气")
    public String getWeather(
            @KernelFunctionParameter(name = "city", description = "城市名")
            String city) {
        return city + "：晴，25℃";
    }

    @DefineKernelFunction(name = "getForecast",
            description = "查询未来天气预报")
    public String getForecast(String city, int days) {
        return city + " 未来 " + days + " 天：晴转多云";
    }
}
```

### 2.2.2 完整示例

把上述对象注册为插件并调用，验证函数可用性。

```java
WeatherPlugin weather = new WeatherPlugin();
KernelPlugin weatherPlugin = KernelPluginFactory.createFromObject(weather, "weather");

Kernel kernel = Kernel.builder()
        .withAIService(ChatCompletionService.class, chatService)
        .withPlugin(weatherPlugin)
        .build();

FunctionResult r = kernel.invokeAsync("weather", "getWeather",
        Map.of("city", "上海")).block();
```

## 2.3 插件分组

### 2.3.1 按领域分组

```
Plugin 1（weather）：getWeather、getForecast
Plugin 2（order）：queryOrder、cancelOrder
Plugin 3（summarize）：summarize
```

> 按领域分组，便于管理与权限控制。

### 2.3.2 命名与权限

插件名应稳定、语义清晰，且作为权限最小单元：可整体授予或收回某租户对某插件的访问权。

| 插件 | 函数 | 敏感度 | 授权策略 |
|------|------|--------|----------|
| weather | getWeather | 低 | 默认开放 |
| order | queryOrder | 中 | 登录用户 |
| order | cancelOrder | 高 | 需二次确认 |

## 2.4 生产建议

插件不宜过大。一个插件包含过多函数会降低模型选择准确率，也削弱权限粒度。建议每个插件聚焦单一领域，函数数量控制在 5 到 10 个以内。命名上，插件名用小写下划线、函数名用小驼峰、描述用动宾短语，可显著提升模型选择函数的准确率。

# 3. 语义函数

语义函数把 Prompt 模板与参数绑定，由 LLM 执行，是 SK 中承接生成类任务的方式。

## 3.1 定义

### 3.1.1 Prompt 模板

```
语义函数 = Prompt 模板 + 参数
由 LLM 执行，非确定性
```

### 3.1.2 示例

在 Java 中语义函数通常以返回 Prompt 的形式声明，实际执行交给模型。

```java
public class SummarizePlugin {

    @DefineKernelFunction(name = "summarize",
            description = "总结文本，输出 3 个要点")
    public String summarize(
            @KernelFunctionParameter(name = "text") String text) {
        // 这个函数由 LLM 执行（通过 Prompt）
        return "总结以下文本，输出 3 个要点：\n" + text;
    }
}
```

## 3.2 结构化输出

### 3.2.1 约束与解析

当语义函数需要被程序消费时，应在 Prompt 中约束为结构化格式，并在调用后解析与校验。

```java
// 伪代码：约束 JSON 并解析
String prompt = "输出 JSON：{\"title\": string, \"points\": string[]}\n" + text;
// 解析失败 -> 重试或降级为纯文本
```

| 输出格式 | 适用 | 风险 |
|----------|------|------|
| 纯文本 | 展示 | 难以程序消费 |
| JSON | 结构化消费 | 可能格式非法 |
| 函数调用 | 强约束 | 依赖模型支持 |

## 3.3 与原生函数协作

### 3.3.1 组合模式

典型组合是：原生函数取数，语义函数生成，再由编排层串联。

```
语义函数（理解意图）
      │
      ▼
原生函数（查询/计算）
      │
      ▼
语义函数（生成回答）
```

## 3.4 生产建议

为语义函数设置超时、重试与输出校验；把 Prompt 模板版本化，便于回滚与 A/B 测试。

## 3.5 语义函数参数化

### 3.5.1 模板变量

语义函数的参数应显式声明并注入模板，避免字符串拼接带来的注入与歧义。

```java
// 伪代码：模板变量注入
String template = "请用 {{style}} 风格总结：\n{{text}}";
String prompt = template
        .replace("{{style}}", style)
        .replace("{{text}}", text);
```

| 参数 | 作用 | 校验 |
|------|------|------|
| text | 待处理文本 | 非空、长度上限 |
| style | 输出风格 | 白名单 |
| language | 输出语言 | 枚举 |

# 4. 调用函数

SK 支持两种调用方式：直接调用与模型自动调用。

## 4.1 直接调用

### 4.1.1 invokeAsync

直接调用绕过模型，适合确定性流程或测试。

```java
// 直接调用
FunctionResult result = kernel.invokeAsync("weather", "getWeather",
        Map.of("city", "北京")).block();

System.out.println(result.getResult());
```

## 4.2 模型自动调用

### 4.2.1 函数调用

把插件交给模型，模型根据用户意图自行决定是否调用及传参。

```java
// 通过模型自动调用（函数调用）
ChatCompletionService chat = kernel.getService(ChatCompletionService.class);

ChatHistory history = new ChatHistory();
history.addUserMessage("北京天气如何？");

FunctionResult answer = chat.getChatMessageContentsAsync(
        history,
        new PromptExecutionSettings(...),   // 启用函数调用
        kernel).block();
```

### 4.2.2 执行设置

执行设置控制温度、最大 token、是否启用函数调用等，应按场景区分。

```java
// 伪代码：启用函数调用的设置
PromptExecutionSettings settings = PromptExecutionSettings.builder()
        .withFunctionChoiceBehavior(FunctionChoiceBehavior.AUTO)
        .build();
```

## 4.3 结果处理

### 4.3.1 FunctionResult

`FunctionResult` 同时承载返回值与元数据，需做空值与类型判断。

| 字段 | 含义 | 注意事项 |
|------|------|----------|
| getResult | 返回值 | 可能为空 |
| getFunction | 函数名 | 用于审计 |
| 元数据 | token、耗时 | 用于监控 |

## 4.4 生产建议

模型自动调用要设置最大轮次，防止模型在多个函数间无限循环；对带副作用的函数（下单、发送）必须加幂等键与人工确认。对互不依赖的查询可并行调用以降低延迟，但写操作必须串行，避免竞态。

# 5. 插件分组与治理

插件是治理的最小单元，分组方式直接决定权限与可观测粒度。

## 5.1 分组策略

### 5.1.1 领域分组

| 分组依据 | 示例 | 优点 |
|----------|------|------|
| 业务领域 | order、user | 权限清晰 |
| 数据源 | crm、erp | 便于隔离 |
| 敏感度 | public、internal | 便于风控 |

## 5.2 权限控制

### 5.2.1 按插件授权

在调用前校验当前用户是否有权访问目标插件与函数，实现最小权限。

```java
// 伪代码：调用前鉴权
if (!authorizer.allow(user, pluginName, functionName)) {
    throw new AccessDeniedException(pluginName + "." + functionName);
}
```

## 5.3 生产建议

把插件名与函数名纳入审计日志，便于事后追溯"哪个租户在何时调用了哪个敏感函数"。审计字段至少包含调用主体、插件.函数、脱敏后的参数摘要、耗时与结果状态。

# 6. 过滤器

过滤器是 SK 的横切机制，等价于 Spring AI 的 Advisor，用于在不侵入业务函数的前提下实现日志、鉴权、限流与缓存。

## 6.1 过滤器类型

### 6.1.1 三类过滤器

```
过滤器 = 拦截器（类似 Spring AI 的 Advisor）
  ├── 函数调用过滤器
  ├── Prompt 过滤器
  └── 结果过滤器
```

| 类型 | 拦截点 | 典型用途 |
|------|--------|----------|
| 函数调用 | 函数执行前后 | 日志、鉴权、限流 |
| Prompt | 请求模型前 | 内容审核、脱敏 |
| 结果 | 模型返回后 | 格式化、审计 |

## 6.2 实现示例

### 6.2.1 日志过滤器

```java
public class LoggingFilter implements FunctionInvocationFilter {

    @Override
    public FunctionResult invoke(FunctionInvocationContext context) {
        log.info("调用函数: {}", context.getFunction().getName());
        FunctionResult result = context.next();
        log.info("结果: {}", result.getResult());
        return result;
    }
}
```

### 6.2.2 鉴权过滤器

在 `context.next()` 之前校验权限，不通过则直接抛出异常，避免真正执行函数。

```java
public class AuthFilter implements FunctionInvocationFilter {

    private final Authorizer authorizer;
    private final UserContext user;

    public AuthFilter(Authorizer authorizer, UserContext user) {
        this.authorizer = authorizer;
        this.user = user;
    }

    @Override
    public FunctionResult invoke(FunctionInvocationContext context) {
        if (!authorizer.allow(user, context.getFunction().getName())) {
            throw new AccessDeniedException(context.getFunction().getName());
        }
        return context.next();
    }
}
```

## 6.3 执行顺序

### 6.3.1 链式顺序

过滤器按注册顺序构成责任链，先注册的先进入、后退出。

```
Filter A 进入
  Filter B 进入
    函数执行
  Filter B 退出
Filter A 退出
```

## 6.4 生产建议

过滤器要轻量且不抛未捕获异常，否则会影响主流程；对过滤器本身的耗时也要监控，避免横切逻辑成为性能瓶颈。

## 6.5 缓存过滤器

### 6.5.1 缓存示例

对幂等且高频的函数调用，可在过滤器层加缓存，命中时直接返回，避免重复执行。

```java
public class CacheFilter implements FunctionInvocationFilter {

    private final Cache<String, FunctionResult> cache;

    public CacheFilter(Cache<String, FunctionResult> cache) {
        this.cache = cache;
    }

    @Override
    public FunctionResult invoke(FunctionInvocationContext context) {
        String key = context.getFunction().getName() + context.getArguments();
        FunctionResult cached = cache.getIfPresent(key);
        if (cached != null) {
            return cached;
        }
        FunctionResult result = context.next();
        cache.put(key, result);
        return result;
    }
}
```

| 函数类型 | 可否缓存 | 说明 |
|----------|----------|------|
| 查询类 | 可 | 注意 TTL |
| 生成类 | 谨慎 | 结果非确定 |
| 写操作 | 否 | 有副作用 |

# 7. 生产注意事项

## 7.1 风险清单

### 7.1.1 风险与建议

| 风险 | 建议 |
|------|------|
| 插件过大 | 按领域分组 |
| 函数描述不清 | 明确描述与参数 |
| 无鉴权 | 函数内校验 |
| 无日志 | 加过滤器 |
| 无观测 | 加过滤器 |
| 权限不清 | 按插件分组授权 |
| 参数校验 | 函数内校验 |
| 版本不兼容 | 锁定版本 |

## 7.2 上线检查清单

### 7.2.1 检查项

上线前确认：插件是否按领域分组、函数描述是否准确、敏感函数是否有鉴权、是否配置过滤器与日志、是否限制模型自动调用轮次、依赖版本是否锁定。

## 7.3 生产建议

把检查清单固化为发布门禁，并对过滤器链路与函数调用成功率建立监控与告警。回滚方面，新增插件可下线注册、修改描述可回退版本、新增过滤器可移除注册、升级 SK 可回退依赖版本。

# 8. 小结

## 8.1 要点回顾

- Kernel 是中央容器，管理服务与插件。
- Plugin 是一组相关函数的集合。
- 区分原生函数（代码）与语义函数（Prompt）。
- 过滤器实现横切关注点。
- 插件按领域分组，便于管理与权限控制。
- 函数要清晰描述并校验参数。

## 8.2 演进建议

后续可把插件按敏感度分层授权、为函数调用建立端到端追踪，并引入自动化测试覆盖函数 schema 与过滤器链，使插件体系在规模增长时仍可维护。
