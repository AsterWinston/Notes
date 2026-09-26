
# 1. Contract 解析流程

```
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
    │           │     └── @Headers (方法级别)
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

# 7. 完整调用链路总结

```
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
