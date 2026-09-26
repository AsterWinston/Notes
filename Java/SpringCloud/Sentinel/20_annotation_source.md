
# 1. 概述

`@SentinelResource` 是 Sentinel 面向业务开发者的主要入口。其底层通过 Spring AOP 切面拦截注解方法，自动调用 `SphU.entry()` 创建资源并执行 SlotChain。

# 2. SentinelResourceAspect — AOP 切面

```java
@Aspect
public class SentinelResourceAspect extends AbstractSentinelAspectSupport {

    // 切点：标注 @SentinelResource 的方法
    @Pointcut("@annotation(com.alibaba.csp.sentinel.annotation.SentinelResource)")
    public void sentinelResourcePointcut() { }

    @Around("sentinelResourcePointcut()")
    public Object invokeResourceWithSentinel(ProceedingJoinPoint pjp) throws Throwable {
        Method originMethod = resolveMethod(pjp);

        // 1. 获取注解信息
        SentinelResource annotation = originMethod
            .getAnnotation(SentinelResource.class);
        String resourceName = getResourceName(annotation.value(),
            originMethod);

        // 2. 获取 Entry 类型
        EntryType entryType = annotation.entryType();

        // 3. 创建 Entry（进入 SlotChain）
        Entry entry = null;
        try {
            entry = SphU.entry(resourceName, entryType, 1,
                pjp.getArgs());
            // 执行原方法
            return pjp.proceed();
        } catch (BlockException ex) {
            // 4. 处理流控/熔断拒绝
            return handleBlockException(pjp, annotation, ex);
        } catch (Throwable ex) {
            // 5. 处理业务异常
            return handleFallback(pjp, annotation, ex);
        } finally {
            if (entry != null) {
                entry.exit(1, pjp.getArgs());
            }
            ContextUtil.exit();
        }
    }
}
```

# 3. blockHandler 处理

```java
// 处理 BlockException
private Object handleBlockException(ProceedingJoinPoint pjp,
        SentinelResource annotation, BlockException ex) throws Throwable {
    if (annotation.blockHandler().isEmpty()) {
        throw ex;  // 未指定 blockHandler，向上抛
    }

    // 查找 blockHandler 方法
    Class<?>[] paramTypes = getParamTypes(pjp, ex.getClass());
    Method blockMethod = findBlockHandler(annotation, paramTypes);

    if (blockMethod == null) {
        throw new NoSuchMethodException("blockHandler not found");
    }

    // 构建参数：原方法参数 + BlockException
    Object[] args = buildArgs(pjp.getArgs(), ex, paramTypes);

    // 调用 blockHandler
    if (isStatic(blockMethod)) {
        return blockMethod.invoke(null, args);
    }
    return blockMethod.invoke(getBean(annotation.blockHandlerClass()), args);
}
```

# 4. fallback 处理

```java
private Object handleFallback(ProceedingJoinPoint pjp,
        SentinelResource annotation, Throwable ex) throws Throwable {
    if (annotation.fallback().isEmpty()
            && annotation.defaultFallback().isEmpty()) {
        throw ex;  // 未指定 fallback，向上抛
    }

    // 查找 fallback 方法
    Method fallbackMethod;
    if (!annotation.fallback().isEmpty()) {
        fallbackMethod = findFallbackMethod(pjp, annotation.fallback(),
            annotation.fallbackClass());
    } else {
        // defaultFallback: 单参数 Throwable
        fallbackMethod = findDefaultFallback(annotation);
    }

    if (fallbackMethod == null) {
        throw ex;
    }

    Object[] args = buildFallbackArgs(pjp.getArgs(), ex,
        fallbackMethod.getParameterTypes());

    if (isStatic(fallbackMethod)) {
        return fallbackMethod.invoke(null, args);
    }
    return fallbackMethod.invoke(
        getBean(annotation.fallbackClass()), args);
}
```

# 5. SphU.entry — 资源入口

```java
// SphU 是 Sentinel 的核心 API
public class SphU {

    private static final Sph sph = new CtSph();

    // 定义资源
    public static Entry entry(String name) throws BlockException {
        return sph.entry(name, EntryType.OUT, 1);
    }

    public static Entry entry(String name, EntryType type,
            int count, Object... args) throws BlockException {
        return sph.entry(name, type, count, args);
    }

    // 异步调用
    public static AsyncEntry asyncEntry(String name) throws BlockException {
        return sph.asyncEntry(name, EntryType.OUT, 1);
    }
}
```

# 6. ContextUtil — 调用上下文

```java
public class ContextUtil {

    private static ThreadLocal<Context> contextHolder = new ThreadLocal<>();

    // 进入上下文（设置 origin 和调用链名称）
    public static Context enter(String name, String origin) {
        if (Constants.CONTEXT_DEFAULT_NAME.equals(name)) {
            throw new ContextNameDefineException(
                "context name cannot be default sentinel_default_context");
        }
        return trueEnter(name, origin);
    }

    protected static Context trueEnter(String name, String origin) {
        Context context = contextHolder.get();
        if (context == null) {
            // 创建根 Context
            context = createEntranceContext(name, origin);
            contextHolder.set(context);
        }
        return context;
    }

    // 退出上下文
    public static void exit() {
        Context context = contextHolder.get();
        if (context != null) {
            contextHolder.remove();
        }
    }
}
```

# 7. SentinelWebInterceptor — Web 自动拦截

```java
public class SentinelWebInterceptor extends AbstractSentinelInterceptor {

    @Override
    protected String getResourceName(HttpServletRequest request) {
        // 默认资源名：HTTP_METHOD:URL_PATH
        String resourceName = request.getMethod() + ":" +
            FilterUtil.filterTarget(request);
        
        // 支持 UrlCleaner 自定义
        if (urlCleaner != null) {
            resourceName = urlCleaner.clean(resourceName);
        }
        return resourceName;
    }

    @Override
    protected void handleBlockException(HttpServletRequest request,
            HttpServletResponse response, BlockException e) {
        // 默认返回 429 Too Many Requests
        response.setStatus(429);
        response.setContentType("application/json;charset=utf-8");
        response.getWriter().write("{\"code\":429,\"message\":\"Blocked by Sentinel\"}");
    }
}
```

# 8. Sentinel 过滤器自动配置

```java
@Configuration
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass(AbstractSentinelInterceptor.class)
public class SentinelWebAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public SentinelWebInterceptor sentinelWebInterceptor(
            Optional<UrlCleaner> urlCleaner,
            Optional<BlockExceptionHandler> blockExceptionHandler) {
        SentinelWebInterceptor interceptor = new SentinelWebInterceptor();
        urlCleaner.ifPresent(interceptor::setUrlCleaner);
        blockExceptionHandler.ifPresent(interceptor::setBlockExceptionHandler);
        return interceptor;
    }

    @Bean
    public SentinelWebMvcConfig sentinelWebMvcConfig(
            SentinelWebInterceptor interceptor) {
        return new SentinelWebMvcConfig(interceptor);
    }
}
```

# 9. 注解声明与执行流程

```
@SentinelResource("getUser")
public User getUser(Long id)
  │
  ├── SentinelResourceAspect.invokeResourceWithSentinel()
  │     │
  │     ├── SphU.entry("getUser", ...)
  │     │     ├── ContextUtil.getContext()
  │     │     ├── lookProcessChain(resource)
  │     │     │     └── DefaultSlotChainBuilder.build()
  │     │     └── chain.entry()
  │     │           ├── NodeSelectorSlot → 构建 Node 树
  │     │           ├── ClusterBuilderSlot → 构建 ClusterNode
  │     │           ├── StatisticSlot → 统计（先执行后统计）
  │     │           ├── SystemSlot → 系统保护检查
  │     │           ├── FlowSlot → 流量控制检查
  │     │           └── DegradeSlot → 熔断降级检查
  │     │
  │     ├── 通过 → pjp.proceed() → 执行业务方法
  │     │
  │     ├── BlockException → handleBlockException()
  │     │     └── 调用 blockHandler 方法
  │     │
  │     └── Throwable → handleFallback()
  │           └── 调用 fallback 方法
  │
  └── entry.exit()
```

# 10. 最佳实践

1. **注解优先于 API**：`@SentinelResource` 比 `SphU.entry()` 更简洁
2. **blockHandler 方法签名**：必须与源方法参数一致 + `BlockException`
3. **分离 blockHandler 类**：使用 `blockHandlerClass` 保持 Controller 清晰
4. **fallback 与 blockHandler 分离**：流量管控和业务异常用不同方法处理
5. **Context 管理**：`SphU.entry()` 自动管理 Context，不需要手动 `ContextUtil.enter()`
