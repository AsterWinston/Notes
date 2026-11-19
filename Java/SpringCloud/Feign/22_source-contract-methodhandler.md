# 1. Contract 解析流程

```text
@FeignClient 接口
    │
    ▼
Contract.parseAndValidateMetadata(Class targetType)
    │
    ├── 遍历 targetType 的所有方法
    │     │
    │     ├── isDefault() → DefaultMethodHandler (跳过)
    │     │
    │     └── 非 default → 解析注解
    │           │
    │           ├── 处理类级别注解
    │           │     ├── @RequestMapping (类上)
    │           │     ├── @Headers
    │           │     └── 提取类级别统一前缀
    │           │
    │           ├── processAnnotationOnMethod()
    │           │     ├── @RequestMapping / @GetMapping / @PostMapping ...
    │           │     │     ├── method → HTTP 方法
    │           │     │     ├── value/path → URL 模板
    │           │     │     ├── produces → Accept header
    │           │     │     └── consumes → Content-Type header
    │           │
    │           ├── processAnnotationsOnParameter()
    │           │     ├── @PathVariable → indexToName + pathVariableIndex
    │           │     ├── @RequestParam → indexToName + queryParam
    │           │     ├── @RequestHeader → headerMapIndex
    │           │     ├── @RequestBody → bodyIndex
    │           │     ├── @RequestPart → formFieldIndex
    │           │     └── @SpringQueryMap → queryMapIndex
    │           │
    │           └── 构建 MethodMetadata
    │                 ├── configKey
    │                 ├── template (RequestTemplate)
    │                 ├── returnType
    │                 ├── indexToName
    │                 └── ...
    │
    └── 返回 Map<String, MethodMetadata>
```

## 1.1 契约的本质：把"调用声明"预编译成"模板 + 索引表"

图里最值得咀嚼的是输出的两样东西。`template` 是**带占位符的半成品请求**（`/api/users/{id}?page={page}` 这种 URI 模板 + 静态 header），不是字符串——`RequestTemplate` 内部把 URI 拆成 `TemplateUrl` 片段数组，占位符位置预扫描过，调用期只做替换不做解析。`indexToName` 及各 `xxxIndex` 是**参数位置路由表**：第 0 个参数进 path 的 `{id}`、第 1 个进 query 的 `page`、第 2 个是 body……`23_source-request-pipeline.md` 展开的 `buildTemplateFromArgs` 逐参数消费的就是这张表。声明式调用的全部"魔法"到这里已经完成，运行期只剩查表填坑——这就是 §4.1（`21_source-dynamic-proxy.md`）说"调用期无反射解析"的具体含义。

## 1.2 validate 失败即启动失败

方法名里的 `Validate` 不是走过场：没有 HTTP 方法注解的方法抛 `MethodMetadataException`（"Method foo not annotated with..."）；同方法多个 `@RequestBody`、`@param.name` 为空（§2.1）等都在此列。校验发生在**装配期**（代理创建时），所以契约错误是"起不来"级别的问题而不是"调不通"——看到这类异常应该高兴，它拦下的是上线后才炸的错。跨 Contract 实现的差异点：`Contract.Default`（feign 原生注解）要求 @Param 显式给名，`SpringMvcContract` 对 `@PathVariable` 的 value 可空时会尝试从模板变量名回填，两条路线的"注解写法兼容性"因此不同（`5_contract.md` §5 对照表）。

# 2. SpringMvcContract 核心源码

```java
public class SpringMvcContract extends Contract.BaseContract {

    @Override
    public MethodMetadata parseAndValidateMetadata(Class<?> targetType,
                                                    Method method) {
        MethodMetadata data = new MethodMetadata();
        data.targetType(targetType);
        data.method(method);
        data.returnType(Types.resolveReturnType(targetType, method));

        // 1. 提取类上的 @RequestMapping 作为 URL 前缀
        RequestMapping classRequestMapping = targetType
                .getAnnotation(RequestMapping.class);
        if (classRequestMapping != null) {
            // 拼接类级别路径
        }

        // 2. 处理方法注解
        Annotation methodAnnotation = getMethodAnnotation(method);
        processAnnotationOnMethod(data, methodAnnotation, method);

        // 3. 处理参数注解
        Annotation[][] paramAnnotations = method.getParameterAnnotations();
        int count = paramAnnotations.length;
        for (int i = 0; i < count; i++) {
            boolean processed = processAnnotationsOnParameter(data,
                    paramAnnotations[i], i);
            if (!processed) {
                // 未标记任何注解的参数
                // SpringMvcContract 会检查是否是复杂对象
                Type parameterType = method.getGenericParameterTypes()[i];
                if (isHttpEntity(parameterType) || isMultipartFile(parameterType)) {
                    // 特殊类型处理
                }
            }
        }

        return data;
    }

    // 处理方法注解（@RequestMapping 及其缩写）
    @Override
    protected void processAnnotationOnMethod(MethodMetadata data,
                                              Annotation methodAnnotation,
                                              Method method) {
        if (!(methodAnnotation instanceof RequestMapping)) {
            return;
        }
        RequestMapping requestMapping = (RequestMapping) methodAnnotation;

        // HTTP 方法
        if (requestMapping.method().length > 0) {
            String httpMethod = requestMapping.method()[0].name();
            data.template().method(httpMethod);
        }

        // URL 路径
        if (requestMapping.value().length > 0) {
            String pathValue = requestMapping.value()[0];
            data.template().uri(pathValue);
        }

        // produces → Accept header
        if (requestMapping.produces().length > 0) {
            data.template().header("Accept",
                    Arrays.asList(requestMapping.produces()));
        }

        // consumes → Content-Type header
        if (requestMapping.consumes().length > 0) {
            data.template().header("Content-Type",
                    Arrays.asList(requestMapping.consumes()));
        }
    }

    // 处理参数注解
    @Override
    protected boolean processAnnotationsOnParameter(MethodMetadata data,
                                                     Annotation[] annotations,
                                                     int paramIndex) {
        boolean isHttpAnnotation = false;

        for (Annotation annotation : annotations) {
            Class<? extends Annotation> type = annotation.annotationType();

            if (type == PathVariable.class) {
                PathVariable pathVariable = (PathVariable) annotation;
                String name = pathVariable.value();
                // 记录参数索引与名称的映射
                data.indexToName().put(paramIndex,
                        Collections.singletonList(name));
                isHttpAnnotation = true;

            } else if (type == RequestParam.class) {
                RequestParam requestParam = (RequestParam) annotation;
                String name = requestParam.value();
                // 必须是显式指定了 value
                CheckState.checkArgument(!name.isEmpty(),
                        "RequestParam.value() was empty on parameter %s", paramIndex);
                data.indexToName().put(paramIndex,
                        Collections.singletonList(name));
                isHttpAnnotation = true;

            } else if (type == RequestHeader.class) {
                RequestHeader requestHeader = (RequestHeader) annotation;
                String name = requestHeader.value();
                data.headerMapIndex(paramIndex);
                data.indexToName().put(paramIndex,
                        Collections.singletonList(name));
                isHttpAnnotation = true;

            } else if (type == RequestBody.class) {
                data.bodyIndex(paramIndex);
                isHttpAnnotation = true;

            } else if (type == SpringQueryMap.class) {
                data.queryMapIndex(paramIndex);
                isHttpAnnotation = true;
            }
        }

        return isHttpAnnotation;
    }
}
```

（骨架简化还原：真实实现按注解类型走 `AnnotationProcessor` 链，`@GetMapping` 等组合注解经 `AnnotatedParameterProcessor` 归一到 `@RequestMapping` 语义；`method()` 对缩写注解取注解自身推断而非数组首元素；上方 `instanceof RequestMapping` 的直判与 produces/consumes 分支是示意写法。）

## 2.1 最著名的报错："RequestParam.value() was empty"

源码里那行 `checkArgument` 值得单独讲，因为它是 Feign 使用率第一的启动报错。根因是**参数名的可得性**：Spring MVC 在 Controller 侧可以留空 `@RequestParam String name`，靠编译产物里的参数名信息（`-parameters` 或调试符号）反推；Feign 的解析发生在**调用方进程**，接口编译时若没开 `-parameters`，字节码里方法参数就叫 `arg0`——Feign 拒绝猜，强制显式命名。两个修法：注解里写全 `@RequestParam("name")`（推荐，接口契约自文档化）；或编译插件开 `-parameters` 后 SpringMvcContract 能回填（对 `@PathVariable` 尤其有用）。同一个坑的变体：`@RequestHeader`/`@PathVariable` 留空在多数版本同样炸或静默错绑——**给 Feign 接口写注解就全部显式命名**，把它当团队 lint 规则。

## 2.2 未注解参数去哪了

§2 源码 `!processed` 分支只处理 HttpEntity/MultipartFile 特例，其余**未标注解的参数默认被忽略**——不像 Spring MVC 有复杂的 HandlerMethodArgumentResolver 猜测体系。所以"传了对象没生效"（忘写 @RequestBody 的 POJO、忘写 @SpringQueryMap 的查询对象）在 Feign 里是静默丢失 + 模板占位符解析不到值时整个调用抛 `IllegalStateException: ... parameters ... not annotated`（部分版本行为）。参数路由的完整记忆表：

```text
@PathVariable → path 占位符      @RequestParam → query      @RequestHeader → header
@RequestBody  → body(至多一个)   @RequestPart  → multipart part
@SpringQueryMap → 对象字段摊平成 query（至多一个）           Request.Options → 特殊位(§2.3)
其余未注解    → 忽略/报错
```

## 2.3 两个隐形合法参数：Options 与 AsyncRequestOptions

`Request.Options` 作为方法参数出现时不参与 URL 组装，而是**运行时替换该次调用的超时**（`11_timeout-connection.md` §2.1）——它的处理不在上面的注解循环里，而是 Contract 基类的"类型即语义"特判。记住这条是因为接口重构时容易把它误标 `@SpringQueryMap`，一旦当普通对象摊平，超时参数会变成 `?connectTimeout=1000&...` 的荒腔走板 query。

# 3. MethodHandler 接口

```java
public interface MethodHandler {
    Object invoke(Object[] argv) throws Throwable;
}
```

```java
public interface MethodHandler.Factory {
    MethodHandler create(Target<?> target, MethodMetadata metadata,
                         RequestTemplate.Factory buildTemplateFromArgs,
                         Request.Options options,
                         Decoder decoder,
                         ErrorDecoder errorDecoder);
}
```

## 3.1 MethodHandler = "半调用"（partial application）

这个两成员的结构是理解 Feign 运行期的钥匙：`Factory.create` 在**装配期**把"这个方法需要的一切"——元数据、模板工厂、超时、编解码器、拦截器列表、Retryer、Logger、Client——全部捕获进一个对象（真实签名比示意多这些参数，还原度注记），产物 `invoke(argv)` 只剩"喂参数"一个自由度。也就是说 **一个 Feign 接口方法在运行期的化身就是一个闭包**。收益是调用路径零决策（没有"该用哪个 Decoder"的查找，构建时定死），代价在 `21_source-dynamic-proxy.md` §3.1 说过：组件是构建期快照，换 Bean 要重建代理。

# 4. SynchronousMethodHandler — 核心实现

```java
public class SynchronousMethodHandler implements MethodHandler {

    private final MethodMetadata metadata;
    private final Target<?> target;
    private final Client client;
    private final Retryer retryer;
    private final Logger logger;
    private final Level logLevel;
    private final RequestTemplate.Factory buildTemplateFromArgs;
    private final Request.Options options;
    private final Decoder decoder;
    private final ErrorDecoder errorDecoder;
    private final boolean decode404;
    private final boolean dismiss404;
    private final List<RequestInterceptor> requestInterceptors;

    @Override
    public Object invoke(Object[] argv) throws Throwable {
        // 1. 根据方法参数构建 RequestTemplate
        RequestTemplate template = buildTemplateFromArgs.create(argv);

        // 2. 克隆 Retryer（每次调用独立的重试状态）
        Retryer retryerToUse = this.retryer.clone();

        // 3. 重试循环
        while (true) {
            try {
                return executeAndDecode(template, options);
            } catch (RetryableException e) {
                try {
                    retryerToUse.continueOrPropagate(e);
                } catch (RetryableException th) {
                    Throwable cause = th.getCause();
                    if (this.dismiss404 && is404(cause)) {
                        return null;
                    }
                    if (cause instanceof IOException) {
                        // 增强异常信息
                        throw errorExecuting(request, (IOException) cause);
                    }
                    throw cause;
                }
                // 重试间隔内打印日志
                if (logLevel != Level.NONE) {
                    logger.logRetry(metadata.configKey(), logLevel);
                }
            }
        }
    }

    Object executeAndDecode(RequestTemplate template, Request.Options options)
            throws Throwable {
        // 步骤 1: 将 RequestTemplate 转为 Request
        //         同时执行所有 RequestInterceptor
        Request request = targetRequest(template);

        // 步骤 2: 记录请求日志
        if (logLevel != Level.NONE) {
            logger.logRequest(metadata.configKey(), logLevel, request);
        }

        // 步骤 3: 执行 HTTP 请求
        Response response;
        long start = System.nanoTime();
        try {
            response = client.execute(request, options);
        } catch (IOException e) {
            if (logLevel != Level.NONE) {
                logger.logIOException(metadata.configKey(), logLevel, e,
                        elapsedTime(start));
            }
            throw errorExecuting(request, e);
        }

        // 步骤 4: 记录响应日志 + rebuffer body
        long elapsedTime = TimeUnit.NANOSECONDS.toMillis(
                System.nanoTime() - start);
        if (logLevel != Level.NONE) {
            response = logger.logAndRebufferResponse(
                    metadata.configKey(), logLevel, response, elapsedTime);
        }

        // 步骤 5: 处理 404
        if (response.status() >= 200 && response.status() < 300) {
            if (response.status() == 204 || response.body() == null) {
                return null;
            }
            // 正常响应 → Decoder 解码
            return decoder.decode(response, metadata.returnType());
        } else if (decode404 && response.status() == 404) {
            return decoder.decode(response, metadata.returnType());
        }

        // 步骤 6: 错误响应 → ErrorDecoder
        throw errorDecoder.decode(metadata.configKey(), response);
    }

    Request targetRequest(RequestTemplate template) {
        // 执行所有 RequestInterceptor
        for (RequestInterceptor interceptor : requestInterceptors) {
            interceptor.apply(template);
        }
        // Target.apply() → 将模板转为 Request，设置 URL
        return target.apply(template);
    }
}
```

## 4.1 invoke 与 executeAndDecode 的分界线是重试

把两段源码对照着读，管线归属立刻清晰：**循环外**（每逻辑调用一次）——模板构建、Retryer 克隆；**循环内**（每次尝试）——拦截器、编码产物落地、HTTP、日志、解码/错误。三条此前散在各篇的结论在这个结构里会师：拦截器每尝试重跑且作用于同一模板实例（`7_interceptor.md` §8.1 幂等要求的出处）；`Retryer.clone()` 在循环外是"单次调用独享计数"的实现（`9_retry.md` §2.1 线程安全方案的出处）；204/空 body 直接 return null 的路径根本**不经过 Decoder**（`20_response-mapping.md` §6 空响应矩阵第一行的出处）。

## 4.2 errorExecuting：status = -1 的出生地

`client.execute` 抛 IOException 时走 `errorExecuting(request, e)`——它 new 的是 `RetryableException`（status=-1，cause 保留原始 IO 异常），这就是 `8_error-handling.md` §8"ErrorDecoder 管不到网络失败"那条图外路径的确切出处；重试耗尽时示例代码把 `cause` 解包重抛的写法（真实实现此处保留 FeignException 包装并补 message，还原度注记）解释了为什么生产栈里超时类错误有时以 `RetryableException` 结尾、有时是裸 `SocketTimeoutException`——取决于版本对"最终抛出物"的处理。

## 4.3 步骤 5 的一个真实差异

示例把 204 判断放在 2xx 分支内、body==null 统一 return null——真实代码对 `response.body() == null` 的处理会先看返回类型是否要求非空再决定抛不抛，且 204 之外"200 + 空 body"通常仍进 Decoder（由 Jackson 决定给 null 还是炸，`20_response-mapping.md` §6 矩阵第 2/3 行的分界）。读简化源码时要防的正是这类"示意分支合并"造成的口径误记，涉及边界行为以 `23_source-request-pipeline.md` 与真实类为准。

# 5. RequestTemplate → Request 转换

```java
// Target.apply() 方法（HardCodedTarget / EmptyTarget）
public Request apply(RequestTemplate input) {
    // 1. 替换 URL 中的 {变量} 为实际值
    input.target(url);   // 设置 base URL

    // 2. 将 RequestTemplate 转为不可变的 Request
    return input.request();
}

// RequestTemplate.request() 方法
public Request request() {
    // 构建完整的 HTTP 请求对象
    return Request.create(
            httpMethod,          // GET/POST/...
            url,                 // 完整 URL
            headers,             // 不可变 headers
            body,                // 请求体字节
            charset,             // 字符集
            requestInterceptor   // 可选的 per-request 拦截器
    );
}
```

## 5.1 可变模板与不可变报文的双态设计

`RequestTemplate` 是**可变**的（拦截器在 apply 里改 header/query/URI 都发生在这一态），`Request` 是**不可变**快照（`headers` 转 unmodifiableMap、body 已是终态字节）。这个双态解释了三个现象：拦截器永远碰不到"已发出的报文"（签名基于模板态，`7_interceptor.md` §2 的时序约束）；重试重放时 Request 每次**重新生成**（模板不变，`targetRequest` 在循环内）；以及 `input.target(url)` 的副作用——`HardCodedTarget.apply` 直接改传入模板（真实代码如此，示例注释"替换变量"不准确：变量替换早已在 `buildTemplateFromArgs` 完成，这里只是拼 base URL），所以跨尝试对模板的修改会累积，与 `7_interceptor.md` §8.1 的"append 畸形请求"同源。

# 6. MethodMetadata 的 configKey

```java
// configKey 生成规则：TargetType#methodName(ParamType1,ParamType2)
// 示例：
//   UserClient#getUser(Long)
//   UserClient#createUser(User)

// 作用：
// 1. 作为 MethodHandler Map 的 key
// 2. 日志中的方法标识
// 3. ErrorDecoder 中的 methodKey 参数
```

## 6.1 configKey 的完整消费者清单

注释里的三个作用还要补三个：`RetryableException.decode` 的 methodKey（同 ErrorDecoder）、熔断器**实例命名**的组成部分（`14_circuit-breaker.md` §6.1 那些难配的 instances 键名根源在此）、Micrometer 指标 `uri`/`method` 标签与 Feign 自身 `FeignException` message 前缀（`[UserClient#getUser] <--- ...` 的方括号）。一张身份表串起五个系列文件：**configKey 是跨日志、监控、熔断、异常、配置的同一主键**，写灰度/告警/熔断配置的键名时先到日志里抄 `[...]` 原文，比按规则手拼可靠。重载方法注意参数列表参与生成（`Long`/`String` 不撞），但**擦除后同签名**的泛型重载（`get(List<User>)` vs `get(List<Order>)` 擦除后都是 `(List)`）会撞 key——接口设计避免这种重载。

# 7. 完整调用链路总结

```text
userClient.getUser(1L)
  │
  │ JDK Proxy
  ▼
FeignInvocationHandler.invoke(proxy, method, [1L])
  │
  │ dispatch.get(method)
  ▼
SynchronousMethodHandler.invoke([1L])
  │
  ├── buildTemplateFromArgs.create([1L])
  │     │
  │     ├── 读取 MethodMetadata
  │     ├── 替换 URL 模板中的 {id}
  │     ├── 组装 Query Parameters
  │     └── 返回 RequestTemplate
  │
  ├── while(true) { ... }   ← 重试循环
  │
  └── executeAndDecode(template, options)
        │
        ├── targetRequest(template)
        │     ├── for (RequestInterceptor i : interceptors) i.apply(template)
        │     └── target.apply(template) → Request
        │
        ├── logger.logRequest(configKey, level, request)
        │
        ├── client.execute(request, options)
        │     │
        │     ├── FeignBlockingLoadBalancerClient
        │     │     ├── loadBalancerClient.choose(serviceId)
        │     │     └── delegate.execute(newRequest, options)
        │     │           └── ApacheHttpClient / OkHttpClient / Client.Default
        │     │
        │     └── 返回 feign.Response
        │
        ├── logger.logAndRebufferResponse(...)
        │
        └── 状态码判断
              ├── 2xx → decoder.decode(response, returnType)
              ├── 404+decode404 → decoder.decode(response, returnType)
              └── 其他 → errorDecoder.decode(configKey, response)
```

## 7.1 一图收束全系列：各篇在这条线上认领故障

这张图是本系列 25 篇的"主板"，排障时按图上节点对号入座：`buildTemplateFromArgs` 出错（占位符没替换、query 丢失）→ `5_contract.md`；`interceptor.apply` 引入的畸形 → `7_interceptor.md`；`client.execute` 的超时/池满 → `11_timeout-connection.md` + `19_http-client-engines.md`；`choose(serviceId)` 空列表 → `13_loadbalancer.md`/`15_service-discovery.md`；`decoder.decode` 类型错位 → `6_encoder-decoder.md`/`20_response-mapping.md`；`errorDecoder` 产出的异常形态 → `8_error-handling.md`；整节点被熔断短路（图上根本没走到 execute）→ `14_circuit-breaker.md`；循环转多圈 → `9_retry.md`。拿到一条 Feign 异常栈，先在这张图上找到**最深的那个 Feign 帧**，它的名字直接告诉你该翻哪篇——这就是把"源码篇"放在系列末尾的原因：它不是第八种知识，是前七种的索引。
