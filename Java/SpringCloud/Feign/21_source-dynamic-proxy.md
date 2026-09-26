
# 1. 核心类关系

```
@FeignClient 接口
    │
    │ FeignClientsRegistrar 扫描注册
    ▼
FeignClientFactoryBean (FactoryBean)
    │
    │ getObject() → getTarget()
    ▼
Feign.Builder.target(Class<T>, String url)
    │
    │ 或 Spring Cloud 中走 FeignCircuitBreaker.Builder
    ▼
ReflectiveFeign.newInstance(Target<T> target)
    │
    │ 1. Contract.parseAndValidateMetadata() → Map<String, MethodHandler>
    │ 2. InvocationHandlerFactory.create()    → FeignInvocationHandler
    │ 3. Proxy.newProxyInstance()             → JDK 动态代理
    ▼
返回代理对象 (注入到 Spring 容器)
```

# 2. FeignClientsRegistrar — Bean 注册入口

```java
class FeignClientsRegistrar
        implements ImportBeanDefinitionRegistrar, ResourceLoaderAware, EnvironmentAware {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata,
                                        BeanDefinitionRegistry registry) {
        // 1. 注册默认配置
        registerDefaultConfiguration(metadata, registry);
        // 2. 注册 FeignClient
        registerFeignClients(metadata, registry);
    }

    private void registerFeignClients(AnnotationMetadata metadata,
                                      BeanDefinitionRegistry registry) {
        // 扫描 @FeignClient 注解的接口
        Set<BeanDefinition> candidates = scanner
                .findCandidateComponents(basePackage);

        for (BeanDefinition candidate : candidates) {
            String className = candidate.getBeanClassName();
            // 动态注册 BeanDefinition，beanClass = FeignClientFactoryBean
            BeanDefinitionBuilder builder = BeanDefinitionBuilder
                    .genericBeanDefinition(FeignClientFactoryBean.class);

            // 将 @FeignClient 的属性注入到 FactoryBean
            builder.addPropertyValue("name", ...);
            builder.addPropertyValue("url", ...);
            builder.addPropertyValue("path", ...);
            builder.addPropertyValue("type", className);
            builder.addPropertyValue("fallback", ...);
            builder.addPropertyValue("fallbackFactory", ...);

            registry.registerBeanDefinition(beanName, builder.getBeanDefinition());
        }
    }
}
```

**关键点**：每个 `@FeignClient` 接口都会被注册为一个 `FeignClientFactoryBean` 的 BeanDefinition。Spring 容器在实例化时会调用 `FeignClientFactoryBean.getObject()`。

# 3. FeignClientFactoryBean.getObject()

```java
class FeignClientFactoryBean implements FactoryBean<Object>, InitializingBean {

    @Override
    public Object getObject() {
        return getTarget();
    }

    <T> T getTarget() {
        FeignContext context = applicationContext.getBean(FeignContext.class);

        // 1. 构建 Feign.Builder
        Feign.Builder builder = feign(context);

        // 2. 如果没有 URL，构建为负载均衡目标
        if (StringUtils.hasText(url)) {
            // 直连模式
            return (T) loadBalance(builder, context,
                    new HardCodedTarget<>(type, name, url));
        }

        // 3. 服务发现 + 负载均衡模式
        return (T) loadBalance(builder, context,
                new Target.HardCodedTarget<>(type, name, cleanUrl()));
    }

    protected Feign.Builder feign(FeignContext context) {
        Feign.Builder builder = Feign.builder();

        // 从 FeignContext 获取各组件（支持按 serviceId 隔离）
        builder.contract(context.getInstance(name, Contract.class));
        builder.encoder(context.getInstance(name, Encoder.class));
        builder.decoder(context.getInstance(name, Decoder.class));
        builder.errorDecoder(context.getInstance(name, ErrorDecoder.class));
        builder.retryer(context.getInstance(name, Retryer.class));
        builder.logger(context.getInstance(name, Logger.class));
        builder.requestInterceptors(context.getInstances(name, RequestInterceptor.class));
        builder.options(context.getInstance(name, Request.Options.class));
        // ...

        return builder;
    }

    protected <T> T loadBalance(Feign.Builder builder, FeignContext context,
                                HardCodedTarget<T> target) {
        // 获取 Client（可能是 FeignBlockingLoadBalancerClient）
        Client client = getOptional(context, Client.class);
        if (client != null) {
            builder.client(client);
        }

        // 带熔断的构建器
        Targeter targeter = get(context, Targeter.class);
        return (T) targeter.target(this, builder, context, target);
    }
}
```

# 4. ReflectiveFeign.newInstance() — 代理创建核心

```java
public class ReflectiveFeign extends Feign {

    @Override
    public <T> T newInstance(Target<T> target) {

        // ===== 第一阶段：解析注解，生成 MethodHandler =====
        Map<String, MethodHandler> nameToHandler = new LinkedHashMap<>();
        // 处理接口上的 @Headers 注解
        Map<String, Header> headerMap = new LinkedHashMap<>();
        // 处理 default 方法
        Map<Method, MethodHandler> defaultMethodHandlers = new LinkedHashMap<>();

        for (Method method : target.type().getMethods()) {
            if (method.isDefault()) {
                // Java 8+ 接口默认方法 → DefaultMethodHandler
                DefaultMethodHandler handler = new DefaultMethodHandler(method);
                defaultMethodHandlers.put(method, handler);
            } else {
                // 解析注解 → MethodMetadata → 构建 SynchronousMethodHandler
                MethodMetadata metadata = contract.parseAndValidateMetadata(
                        target.type(), method);
                nameToHandler.put(metadata.configKey(),
                        factory.create(target, metadata));
            }
        }

        // ===== 第二阶段：创建 InvocationHandler =====
        InvocationHandler handler = factory.create(target, nameToHandler);
        // factory 默认是 InvocationHandlerFactory.Default

        // ===== 第三阶段：创建 JDK 动态代理 =====
        T proxy = (T) Proxy.newProxyInstance(
                target.type().getClassLoader(),
                new Class<?>[]{target.type()},
                handler);

        // 将 MethodHandler 映射绑定到代理上
        for (Map.Entry<String, MethodHandler> entry : nameToHandler.entrySet()) {
            MethodHandler methodHandler = entry.getValue();
            if (methodHandler instanceof FactoryAware) {
                ((FactoryAware) methodHandler).setFactory(proxy, target);
            }
        }

        return proxy;
    }
}
```

# 5. FeignInvocationHandler — JDK 代理调用入口

```java
public static class Default implements InvocationHandlerFactory {

    @Override
    public InvocationHandler create(Target target,
                                     Map<String, MethodHandler> dispatch) {
        return new FeignInvocationHandler(target, dispatch);
    }
}

static class FeignInvocationHandler implements InvocationHandler {

    private final Target target;
    private final Map<Method, MethodHandler> dispatch;  // Method → MethodHandler

    @Override
    public Object invoke(Object proxy, Method method, Object[] args)
            throws Throwable {
        // hashCode / equals / toString → 跳过
        if ("equals".equals(method.getName()) && args != null) {
            try {
                Object other = args[0];
                if (other == null) return false;
                Object otherHandler = Proxy.getInvocationHandler(other);
                return this.equals(otherHandler);
            } catch (IllegalArgumentException e) {
                return false;
            }
        } else if ("hashCode".equals(method.getName()) && args == null) {
            return this.hashCode();
        } else if ("toString".equals(method.getName()) && args == null) {
            return this.toString();
        }

        // 实际业务方法 → 委托给 MethodHandler
        return dispatch.get(method).invoke(args);
    }
}
```

# 6. 完整代理创建时序图

```
Spring 容器启动
    │
    ├── FeignClientsRegistrar.registerFeignClients()
    │     └── 为每个 @FeignClient 注册 FeignClientFactoryBean BeanDefinition
    │
    ├── Spring 实例化 FeignClientFactoryBean
    │     │
    │     ├── FeignClientFactoryBean.getObject()
    │     │     │
    │     │     ├── feign(context) → 构建 Feign.Builder（装配组件）
    │     │     │
    │     │     └── loadBalance(builder, target)
    │     │           │
    │     │           ├── FeignCircuitBreakerTargeter.target()
    │     │           │     │
    │     │           │     ├── builder.target(target)
    │     │           │     │     │
    │     │           │     │     └── ReflectiveFeign.newInstance(target)
    │     │           │     │           │
    │     │           │     │           ├── Contract.parseAndValidateMetadata()
    │     │           │     │           │     └── 返回 MethodMetadata (每方法)
    │     │           │     │           │
    │     │           │     │           ├── MethodHandler.Factory.create()
    │     │           │     │           │     └── SynchronousMethodHandler (每方法)
    │     │           │     │           │
    │     │           │     │           ├── InvocationHandlerFactory.create()
    │     │           │     │           │     └── FeignInvocationHandler
    │     │           │     │           │
    │     │           │     │           └── Proxy.newProxyInstance()
    │     │           │     │                 └── JDK 代理对象
    │     │           │     │
    │     │           │     └── 如果有 fallback，用 FeignCircuitBreakerInvocationHandler 包裹
    │     │           │
    │     │           └── 返回代理对象
    │     │
    │     └── Spring 将代理对象作为 Bean 注入到其他组件
    │
    └── 应用启动完成
```

# 7. Proxy.newProxyInstance() 本质

```java
// 等价于运行时动态生成了如下类：
public class $Proxy0 implements UserClient {

    private final InvocationHandler handler;

    public $Proxy0(InvocationHandler handler) {
        this.handler = handler;
    }

    @Override
    public User getUser(Long id) {
        // 所有方法调用都转发到 handler.invoke()
        return (User) handler.invoke(this,
                method,         // UserClient.getUser Method 对象
                new Object[]{id});
    }
}
```

# 8. Spring Cloud 增强：FeignCircuitBreakerInvocationHandler

当启用熔断时，`FeignCircuitBreakerTargeter` 会替换默认的 InvocationHandler：

```java
class FeignCircuitBreakerInvocationHandler implements InvocationHandler {

    private final CircuitBreakerFactory factory;
    private final Target<?> target;
    private final Map<Method, InvocationHandlerFactory.MethodHandler> dispatch;
    private final FallbackFactory<?> nullableFallbackFactory;

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 非业务方法直接转发
        if (isDefaultMethod(method)) return dispatch.get(method).invoke(args);

        // 创建 CircuitBreaker
        CircuitBreaker circuitBreaker = factory.create(target.name());

        // 用 CircuitBreaker 的 run 包装调用
        return circuitBreaker.run(
                () -> dispatch.get(method).invoke(args),   // 实际调用
                throwable -> {
                    // fallback 逻辑
                    if (nullableFallbackFactory != null) {
                        Object fallback = nullableFallbackFactory.create(throwable);
                        return method.invoke(fallback, args);
                    }
                    throw new FeignCircuitBreakerException(throwable);
                }
        );
    }
}
```

# 9. 关键设计模式

| 模式 | 应用 |
|------|------|
| **代理模式** | JDK 动态代理，`@FeignClient` 接口 → `$Proxy` |
| **工厂模式** | `FeignClientFactoryBean`、`InvocationHandlerFactory` |
| **建造者模式** | `Feign.Builder` 链式构建 |
| **策略模式** | `Contract`、`Encoder`、`Decoder`、`Client` 可插拔 |
| **模板方法** | `ReflectiveFeign.newInstance()` 固定流程，组件可替换 |
| **装饰器模式** | `FeignBlockingLoadBalancerClient` 包装原始 `Client` |
