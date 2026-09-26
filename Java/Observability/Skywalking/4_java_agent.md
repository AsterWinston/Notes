
# 1. Java Agent 基础

Java Agent 是 JVM 提供的**运行时字节码增强**机制，允许在类加载时修改字节码。SkyWalking Agent 基于此实现零侵入监控。

```
JVM 启动
  │
  ├── 1. premain() → 注册 ClassFileTransformer
  │
  ├── 2. 类加载时 → 每个类经过 Transformer
  │     └── transform(className, classBytes)
  │           └── ByteBuddy 增强
  │                 └── 返回增强后的字节码
  │
  └── 3. JVM 使用增强后的类
```

# 2. Agent 启动流程

```java
// skywalking-agent.jar → MANIFEST.MF
// Premain-Class: org.apache.skywalking.apm.agent.SkyWalkingAgent

public class SkyWalkingAgent {

    public static void premain(String agentArgs, Instrumentation inst) {
        // 1. 加载核心配置
        SnifferConfigInitializer.initializeCoreConfig(agentArgs);

        // 2. 初始化插件体系
        PluginFinder pluginFinder = new PluginFinder(
            new PluginBootstrap().loadPlugins()
        );

        // 3. 注册 ClassFileTransformer
        inst.addTransformer(new ClassEnhancePluginDefine() {
            @Override
            protected ClassMatch enhanceClass() {
                return // ByteBuddy 匹配规则
            }

            @Override
            protected InstanceMethodsAroundInterceptor getInterceptor() {
                return new MyInterceptor();  // 自定义拦截器
            }
        });

        // 4. JVM 关闭钩子 → 清理资源
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            ServiceManager.INSTANCE.shutdown();
        }));
    }
}
```

# 3. 插件体系架构

```
┌──────────────────────────────────────────┐
│              Plugin System                │
│                                           │
│  ┌────────────────────────────────────┐  │
│  │       PluginBootstrap               │  │
│  │  (扫描 skywalking-plugin.def)       │  │
│  └──────────────┬─────────────────────┘  │
│                 │                         │
│  ┌──────────────▼─────────────────────┐  │
│  │       PluginDefine                  │  │
│  │  define() → 类匹配 + 方法拦截       │  │
│  └──────────────┬─────────────────────┘  │
│                 │                         │
│  ┌──────────────▼─────────────────────┐  │
│  │    AbstractClassEnhancePluginDefine │  │
│  │  - enhanceClass()   → 匹配目标类    │  │
│  │  - getInterceptor() → 拦截器实例     │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
```

## 3.1 插件定义文件

```
# resources/skywalking-plugin.def
spring-webmvc=org.apache.skywalking.apm.plugin.spring.mvc.define.RestControllerInstrumentation
spring-mvc-annotation=org.apache.skywalking.apm.plugin.spring.mvc.define.RestControllerAdviceInstrumentation
dubbo=org.apache.skywalking.apm.plugin.dubbo.DubboInstrumentation
jdbc=org.apache.skywalking.apm.plugin.jdbc.define.JDBCDriverInterceptor
```

## 3.2 ClassMatch — 类匹配策略

```java
// 按类名匹配
byName("org.springframework.web.servlet.DispatcherServlet")

// 按注解匹配
byClassAnnotationMatch("org.springframework.stereotype.Controller")

// 按方法注解匹配
byMethodAnnotationMatch("org.springframework.web.bind.annotation.GetMapping")

// 按父类/接口匹配
byHierarchyMatch("java.sql.Statement")
```

# 4. 方法拦截器

```java
// 环绕拦截
public interface InstanceMethodsAroundInterceptor {

    // 方法执行前
    void beforeMethod(EnhancedInstance objInst, Method method,
                      Object[] allArguments, Class<?>[] argumentsTypes,
                      MethodInterceptResult result);

    // 方法执行后（正常返回）
    Object afterMethod(EnhancedInstance objInst, Method method,
                       Object[] allArguments, Class<?>[] argumentsTypes,
                       Object ret);

    // 方法执行异常
    void handleMethodException(EnhancedInstance objInst, Method method,
                               Object[] allArguments,
                               Class<?>[] argumentsTypes, Throwable t);
}
```

## 4.1 示例：Spring MVC 拦截

```java
public class RestControllerInterceptor implements InstanceMethodsAroundInterceptor {

    @Override
    public void beforeMethod(EnhancedInstance objInst, Method method,
            Object[] allArguments, Class<?>[] argumentsTypes,
            MethodInterceptResult result) {
        // 1. 从请求中获取 URL
        HttpServletRequest request = (HttpServletRequest) allArguments[0];
        String operationName = request.getMethod() + ":" + request.getRequestURI();

        // 2. 创建 Entry Span
        AbstractSpan span = ContextManager.createEntrySpan(
            operationName, null);
        span.setComponent(ComponentsDefine.SPRING_MVC);
        span.setLayer(SpanLayer.HTTP);
        span.tag(Tags.URL, request.getRequestURL().toString());
        span.tag(Tags.HTTP.METHOD, request.getMethod());
    }

    @Override
    public Object afterMethod(EnhancedInstance objInst, Method method,
            Object[] allArguments, Class<?>[] argumentsTypes, Object ret) {
        // 1. 获取 HTTP 状态码
        int statusCode = getStatusCode(allArguments, ret);
        AbstractSpan span = ContextManager.activeSpan();
        span.tag(Tags.HTTP_RESPONSE_STATUS_CODE, String.valueOf(statusCode));
        if (statusCode >= 400) {
            span.errorOccurred();
        }

        // 2. 停止 Span
        ContextManager.stopSpan();
        return ret;
    }

    @Override
    public void handleMethodException(EnhancedInstance objInst, Method method,
            Object[] allArguments, Class<?>[] argumentsTypes, Throwable t) {
        AbstractSpan span = ContextManager.activeSpan();
        span.log(t);
        span.errorOccurred();
    }
}
```

# 5. BootClassLoader 机制

SkyWalking Agent 的类使用独立 ClassLoader 加载，避免与应用类冲突：

```
Bootstrap ClassLoader (JVM)
  │
  ├── SkyWalking Agent JAR 在此加载
  │     └── Plugin 类也在 Agent ClassLoader
  │
  ├── App ClassLoader
  │     └── 应用类
  │
  └── Agent 通过 ByteBuddy 在 App 类中插入代码
        插入的代码通过 Agent ClassLoader 桥接访问
```

# 6. Optional 插件

```bash
# 可选插件默认不激活（可能影响性能），需手动拷贝
cp agent/optional-plugins/apm-spring-annotation-plugin-*.jar \
   agent/plugins/
```

| Optional 插件 | 用途 |
|--------------|------|
| `spring-annotation-plugin` | @Bean/@Service/@Repository 方法级追踪 |
| `trace-ignore-plugin` | 自定义忽略追踪的端点 |
| `quartz-scheduler-plugin` | Quartz 定时任务追踪 |

# 7. 性能开销

| 操作 | 开销 | 说明 |
|------|------|------|
| Span 创建 | ~1μs | ThreadLocal 操作 |
| 方法拦截 | ~3μs | ByteBuddy 拦截器调用 |
| Segment 序列化 | ~50μs | 仅上报时执行（异步） |
| 整体 CPU 开销 | 3-8% | 正常负载下 |
