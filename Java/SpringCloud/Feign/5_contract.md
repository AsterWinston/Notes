# 1. Contract 是什么

## 1.1 职责：把注解翻译成执行计划

`Contract` 是 Feign 的核心组件之一，负责将 Java 接口上的注解**解析为 `MethodMetadata`**（方法元数据），包含：

- HTTP 方法（GET/POST/PUT/DELETE 等）
- 请求路径模板（如 `/users/{id}`）
- 请求头（Header）映射
- 查询参数映射
- 请求体（Body）位置
- 返回值类型信息

Contract 的产出物是"静态的 HTTP 执行计划"：它不知道参数值是什么，只记录"第 0 个参数对应路径占位符 id、第 1 个参数进 query 且名字叫 source"这类**索引级映射**。因此解析只发生在构建期一次，每次调用只是把运行时参数填进模板——这层分工是 Feign 热路径轻量的根本原因（运行期细节见 §5 与 `22_source-contract-methodhandler.md`）。

# 2. 两种内置 Contract

| Contract | 使用场景 | 支持的注解 |
|----------|----------|------------|
| `Contract.Default` | 纯 OpenFeign 环境 | Feign 原生注解（`@RequestLine`, `@Param`, `@Headers`, `@Body`） |
| `SpringMvcContract` | Spring Cloud 环境（**默认**） | Spring MVC 注解（`@RequestMapping`, `@PathVariable`, `@RequestParam` 等） |

Spring Cloud OpenFeign 默认使用 `SpringMvcContract`，通过 `FeignClientsRegistrar` 自动注入。

## 2.1 选 SpringMvcContract 的隐性红利

接口注解与提供方 Controller 注解同族，意味着客户端接口可以**直接对照甚至复制**提供方的映射注解，路径漂移肉眼可查；这也是很多团队用"提供方 Controller 实现共享接口"来强制契约一致的前提。代价是 Spring MVC 注解体系里的一些能力（如 `params`/`headers` 条件匹配、` consumes 多值路由）Feign 并不支持——`SpringMvcContract` 只提取映射语义的子集，写了也不报错但会被忽略，契约设计时要避免依赖这些高级条件。

# 3. SpringMvcContract 解析流程

```text
@FeignClient 接口
  │
  ▼
SpringMvcContract.parseAndValidateMetadata(Class<?> targetType, Method method)
  │
  ├── 解析 @RequestMapping / @GetMapping / @PostMapping ...
  │     ├── 提取 HTTP 方法 (method)
  │     ├── 提取 URL 模板 (value/path)
  │     ├── 提取 produces → Accept 头
  │     └── 提取 consumes → Content-Type 头
  │
  ├── 解析参数注解
  │     ├── @PathVariable → indexToName 映射 + path variable
  │     ├── @RequestParam → indexToName 映射 + query parameter
  │     ├── @RequestHeader → indexToName 映射 + header
  │     ├── @RequestBody → bodyIndex
  │     ├── @RequestPart → formFieldIndex
  │     └── @SpringQueryMap → queryMapIndex
  │
  ├── 解析 @Headers 注解（类级别和方法级别）
  │
  └── 返回 MethodMetadata
        ├── configKey (类名#方法名(参数类型))
        ├── template (RequestTemplate)
        ├── returnType
        ├── headerMapIndex
        ├── queryMapIndex
        └── indexToName / indexToExpander
```

## 3.1 构建期与运行期的分界线

流程图里所有产物都冻结在 `MethodMetadata`。运行期 `SynchronousMethodHandler` 每次调用只做三件事：克隆模板（`targetRequestTemplate`）、按 `indexToName` 等映射把实参展开进模板、把请求体交给 Encoder。两个高频推论：解析错误（漏写 HTTP 注解、`@RequestParam` 无名）**在应用启动阶段就会抛出**而不是第一次调用；改 `MethodMetadata` 的自定义 Contract 影响所有后续调用，而不可能对单个方法调用生效。

## 3.2 configKey 不止是个标识

`configKey = 类名#方法(参数类型)` 是全链路的"方法身份证"：Feign 的日志（`10_logging.md`）、ErrorDecoder 上下文、Resilience4j 熔断器命名（`14_circuit-breaker.md` 中 `<contextId>#<methodKey>`）、性能打点都按它聚合。重载方法会得到不同的 configKey，熔断/日志粒度天然是方法级——但注意接口改名或换包会让监控面板上的序列全部漂移，可观测性口径要预先约定。

# 4. 自定义 Contract

## 4.1 扩展 SpringMvcContract

```java
public class MySpringMvcContract extends SpringMvcContract {

    public MySpringMvcContract() {
        // parameterAnnotationsProcessors 默认会处理 @PathVariable 等
        // 可以在此注册自定义参数注解处理器
    }

    @Override
    protected void processAnnotationOnMethod(MethodMetadata data,
                                             Annotation methodAnnotation,
                                             Method method) {
        // 处理自定义注解
        if (methodAnnotation instanceof MyApiVersion) {
            MyApiVersion version = (MyApiVersion) methodAnnotation;
            data.template().header("X-Api-Version", version.value());
            return;
        }
        super.processAnnotationOnMethod(data, methodAnnotation, method);
    }

    @Override
    protected boolean processAnnotationsOnParameter(MethodMetadata data,
                                                    Annotation[] annotations,
                                                    int paramIndex) {
        boolean processed = super.processAnnotationsOnParameter(data, annotations, paramIndex);
        // 自定义参数注解处理...
        return processed;
    }
}
```

两个覆写点的选择：方法级注解（版本头、签名策略这类"整方法生效"的标记）走 `processAnnotationOnMethod`；参数级注解优先用 §6 的处理器体系而不是覆写这里——`processAnnotationsOnParameter` 一旦覆写，责任就变成你自己保证不破坏内置注解的解析顺序。

## 4.2 注册自定义 Contract

```java
@FeignClient(name = "user-service",
             configuration = MyContractConfig.class)
public interface UserClient { }
```

```java
public class MyContractConfig {
    @Bean
    public Contract feignContract() {
        return new MySpringMvcContract();
    }
}
```

`Contract` Bean 属于组件覆盖（同 `4_configuration.md` §4 语义），只作用于指定 `configuration` 的客户端；若放到主上下文则全局替换，效果类似 §5.1 讲过的作用域污染，位置选择规则一致。

# 5. MethodMetadata 关键字段

```java
public final class MethodMetadata implements Serializable {

    // 方法唯一标识：ClassName#methodName(ParamType1,ParamType2)
    private String configKey;

    // 返回值类型
    private Type returnType;

    // 请求模板（URL, Headers, Query 等）
    private RequestTemplate template;

    // 参数索引 → 参数名称映射
    private Map<Integer, Collection<String>> indexToName;

    // 参数索引 → 参数值展开器（处理数组/集合等）
    private Map<Integer, Param.Expander> indexToExpander;

    // @SpringQueryMap 参数索引
    private Integer queryMapIndex;

    // @RequestHeader @HeaderMap 参数索引
    private Integer headerMapIndex;

    // 请求体参数索引
    private int bodyIndex;

    // 表单字段索引
    private int formFieldIndex;

    // 先序列化为 JSON 再编码为 URL 的查询参数
    private Map<Integer, EncodedQueryParam> encodedQueryParams;

    // 可忽略的类型（用于继承接口的场景）
    private Set<Integer> ignoreParams;

    // ...
}
```

读这份字段清单的正确方式是"每种参数去向一个索引字段"：path/query/header 三个通道靠 `indexToName`（区分 `paramNameParser` 记录的位置类型），body 靠 `bodyIndex`，表单靠 `formFieldIndex`，Map 摊平靠 `queryMapIndex`/`headerMapIndex`。`indexToExpander` 回答的是"集合参数怎么变成字符串"（默认 `Expander.Defaults`：String 直出，其余 `toString`，需要 `a=1&a=2` 展开 List 时要注册自定义 `Param.Expander`）。`returnType` 保留完整泛型信息，Decoder 靠它还原 `List<User>` 这类嵌套类型——这也是 Feign 能直接返回泛型集合的实现基础。

# 6. 自定义参数注解处理器

```java
@Component
public class MyRequestHeaderParameterProcessor
        extends AnnotatedParameterProcessor {

    @Override
    public Class<? extends Annotation> getAnnotationType() {
        return MyHeader.class;
    }

    @Override
    public boolean processArgument(AnnotatedParameterContext context,
                                   Annotation annotation,
                                   Method method) {
        MyHeader myHeader = AnnotatedParameterProcessor
                .findMethodAnnotation(MyHeader.class, context.getParameterIndex(), method);
        if (myHeader != null) {
            context.setParameterSpecification(
                    new SimpleParameterSpecification(
                            Header.class.getSimpleName(),
                            myHeader.value()
                    )
            );
        }
        return true;
    }
}
```

## 6.1 注册方式：它是 Bean，不是配置键

`FeignClientsConfiguration` 在创建默认的 `SpringMvcContract` 时会注入容器中所有 `AnnotatedParameterProcessor` Bean 并传给其构造器，所以上面的 `@Component` 写法即可生效（主容器注册 = 全局客户端可用；想按客户端隔离，就在 §4.2 的配置类里手工 `new MySpringMvcContract(自定义处理器列表, ...)` 覆盖 Contract）。注意处理器**只能被注册进默认 Contract 的列表**，如果谁配置了不含这些处理器的自定义 Contract，自定义注解会静默不解析——两条扩展路径同时使用时要检查装配关系。

# 7. Contract.Default 解析过程（对比）

```java
Contract.Default 对应的注解体系：

@RequestLine("GET /users/{id}")        → HTTP method + URL template
@Param("id")                           → named parameter
@Headers("X-Token: {token}")           → header template
@Body("%7B\"name\": \"{name}\"%7D")   → body template
@QueryMap(encoded = true)              → query parameter map
```

与 `SpringMvcContract` 本质相同，只是注解类型/解析规则不同。

值得对比的其实是表达哲学：`Contract.Default` 把 URL 与 method 写成一条"请求行"字符串，参数统一叫 `{token}` 占位符（不区分它在 path/query/header/body 哪个通道，占位符名字全局解析）；Spring MVC 注解则显式声明参数通道，占位符名可跨通道重名。前者适合脱离 Spring 的轻量集成（比如直接给一个 Retrofit 风格的 OpenFeign 客户端复用注解），后者在 Spring 生态里歧义更少。

# 8. 常见问题

**Q：为什么 `@RequestParam` 不传 value 会报错？**

`SpringMvcContract` 解析 `@RequestParam` 时强制要求 `name()` 不为空。这是因为编译后的接口方法参数名默认不保留（除非加 `-parameters` 编译参数），无法通过反射获取。

**Q：`@PostMapping` 同时有 `@RequestParam` 和 `@RequestBody`，body 会被如何处理？**

```java
@PostMapping("/users")
User create(@RequestParam String source, @RequestBody User user);
```

- `@RequestParam` → Query parameter
- `@RequestBody` → HTTP body

生成的请求：`POST /users?source=web` + `{"name":"xxx","age":20}`

两者可以共存，互不影响。

**Q：接口里的 `default`/`static` 方法会被解析吗？**

不会。Contract 只处理需要远程化的抽象方法；`default` 方法在代理上调用时直接在本地执行（可用于做参数组装、结果后处理的语法糖），`static` 方法同理。这是 Feign 接口里写"非 HTTP 辅助方法"的合法姿势。

**Q：父接口的注解会被继承吗？**

方法注解不会被"自动继承"：子接口重写父接口方法时，注解必须重新声明，Contract 解析的是具体方法上的标注。契约复用应通过拆分接口（读写分离、版本分组）实现，而不是指望注解随签名继承——这是"用共享接口强制提供方/调用方一致"方案最容易踩空的地方（提供方 Controller 类走的是 Spring MVC 的路由体系，两边解析器不同，行为并不对称）。
