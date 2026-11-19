# 1. 核心类关系

```text
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

## 1.1 用三段生命周期读这张图

这条链横跨三个时机，混着看会把"启动报错"和"运行报错"的归因搅浑：

| 阶段 | 时机 | 关键产物 | 典型故障 |
|------|------|----------|----------|
| **注册期** | 容器刷新，`@EnableFeignClients` 触发 Registrar | BeanDefinition（此时**没有**代理） | `basePackages` 没覆盖到接口 → "没有这个 Bean" |
| **装配期** | 首次 `getBean`（FactoryBean.getObject） | Contract 解析 + 代理对象 | Contract 校验失败（§4.1）、子上下文组件缺失 |
| **调用期** | 每次业务方法调用 | 模板构建 → HTTP | 超时/熔断/解码类问题（另见 `23_source-request-pipeline.md`） |

注册期只下"配方"不"做菜"，装配期才真正建代理——这解释了为什么把某个接口挪出扫描范围，报错出现在**注入它的那个服务创建时**而不是启动扫描时；也解释了 Contract 类错误为什么有时"启动没报、上线才报"：装配期是惰性的（§3.1）。

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

## 2.1 为什么是 Registrar + FactoryBean 的两段式

`@EnableFeignClients` 经 `@Import` 挂上 `ImportBeanDefinitionRegistrar`，这是 Spring 里"用户只写接口、容器凭空多出 Bean"的标准配方（MyBatis MapperScannerConfigurer 同款）。拆两段的原因各有必然性：**Registrar 阶段**只有元数据（接口还没被 ClassLoader 完整加载），而构造代理需要容器就绪（子上下文、组件 Bean），所以把"发现"和"制造"分开；**FactoryBean 阶段**则解决"Bean 的类型是接口、产物却是代理"的错位——容器按 `type`（接口 Class）登记 FactoryBean 产物类型，注入点才能直接写 `UserClient`。顺带解释了注入歧义问题：`@FeignClient` 同名多接口（不同 contextId）能共存，是因为 BeanDefinition 的注册名按 `name#接口全名` 规整过，而不是按类型注册。

## 2.2 扫描的两条路径与占位符时机

真实实现（还原度注记：上方是骨架简化）对 `@EnableFeignClients` 的 `clients` 与 `basePackages` 二选一：显式列 `clients` 走快速路径直接读注解属性；否则用 `ClassPathScanningCandidateComponentProvider` 配 `AnnotationTypeFilter(FeignClient.class)` 扫包——**接口能否被扫到依赖 scanner 的独立 MetadataReader**（接口默认不是 candidate，Registrar 里专门放宽了 interface），扫 jar 里嵌套路径、以及 `basePackage` 缺省取注解所在类包这两个坑都在这条路径上。注解属性里的 `${...}` 占位符（`name = "${user.service-id}"`）在注册期就用 `Environment` 解析定稿——所以**配置中心热更新服务名不会改变已有代理**，这类"动态化"需求要走 url 占位符回落（`13_loadbalancer.md` §8）或重建上下文，而不是指望扫描属性。

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

        // 2. 没有 url：服务发现 + 负载均衡模式
        if (!StringUtils.hasText(url)) {
            return (T) loadBalance(builder, context,
                    new Target.HardCodedTarget<>(type, name, url));
        }

        // 3. 有 url：直连模式（不走 LoadBalancer）
        return (T) targeter.target(this, builder, context,
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

（简化还原；§3 的 2/3 分支按真实语义修正了原稿颠倒的 url 判断——真实代码是"**无 url 才 loadBalance**"，`HardCodedTarget` 的 url 参数在无 url 模式下传空串、host 即服务名。）

## 3.1 getObject 的触发时机：代理是懒的

FactoryBean 产物在**第一次被注入/获取**时才创建：接口没有任何消费者 → Contract 永远不解析 → 注解写错也不报错；某服务的接口首次被使用发生在运行期（如反射加载）→ 报错出现在那条业务调用而不是启动。排障口诀："Contract 校验失败类异常出现在**启动日志**说明有组件在启动期注入它（或配了 `@Lazy` 关闭/预实例化单例检查），出现在**业务线程**说明代理直到那时才建"。同一个 FactoryBean 在容器内单例缓存，代理进程内**只造一次**——组件被换后"行为没变"多半是这个缓存要重启才重建（`4_configuration.md` §6 的动态性边界）。

## 3.2 feign(context)：组件从子上下文逐个取

`context.getInstance(name, Xxx.class)` 就是 `4_configuration.md` §1.1 客户端子上下文的消费现场：每个 name/contextId 一个 `NamedContextFactory` 子容器，`FeignClientsConfiguration` 提供默认组件，用户在专属配置类里的 @Bean 覆盖默认。两个由此可推导的行为：取不到任何 Encoder 时 `Builder` 保留**自己的出厂默认**（`Encoder.Default` toString 级）——"对象序列化成 `User@1a2b`"的根因链（`6_encoder-decoder.md` §9）；`getInstance(s)` 每次 getTarget 都会解析一遍，代价只付在装配期。

## 3.3 Targeter：熔断装饰的插入点

`loadBalance` 末行的 `Targeter.target(...)` 是原生 `DefaultTargeter` 与 `FeignCircuitBreakerTargeter` 的分岔：前者直接 `builder.target(target)` 出裸代理，后者构建**双层代理**（§8）。`Client` 用 `getOptional`（可为 null，退回 Builder 默认 `Client.Default`）、`Targeter` 用 `get`（必须有）——这个不对称解释"排除熔断依赖后行为回退成裸调用"而非启动失败。

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

## 4.1 一次性解析：声明式客户端的性能底牌

每个方法的注解解析、URL 模板预编译都发生在这里（`22_source-contract-methodhandler.md` §1 的 parse 阶段），产物 `SynchronousMethodHandler` 连同捕获的组件引用（Encoder/Decoder/Client…）被**永久缓存**在 dispatch Map 里。两个推论：调用期没有注解反射开销，Feign 的运行时成本集中在网络与编解码，"Feign 慢"基本不用怀疑代理层；组件引用是**构建期快照**——给子上下文塞了新 Decoder 不影响已建代理（§3.1 的缓存语义在 JDK 层的体现）。真实代码的 `parseAndValidateMetadata` 是整类一次调用返回全部方法元数据（示例逐方法调用是简化），校验失败在任何一次网络 IO 之前抛出——"接口缺 @RequestMapping"类错误属于 fail-fast。

## 4.2 default 方法：把计算留给本地

`method.isDefault()` 的分支让接口里的 `default` 方法**不走远程**、按本地语义执行（示例代码为简化还原；`DefaultMethodHandler` 在 JDK 的 MethodHandles  specials 上调用父类实现）。实用姿势：接口里放 `default List<User> listUsersPaged(int page) { return listUsers(query(page)); }` 这类参数换算/组合逻辑，调用方复用且零代理成本。边界两个：default 方法本身也经代理路由到 `DefaultMethodHandler` 再本地执行；在其内部调用本接口的其他方法时，`this` 就是代理对象，**这些内部调用仍会完整经过 handler 走远程**（与 Spring AOP 的自调用失效问题相反，方向是"全部生效"）——依赖这个行为时写测试固化；Java 9+ 的 private 接口方法不在 `getMethods()` 里，天然安全。

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

## 5.1 Object 三兄弟为什么要特判

dispatch Map 里只有接口方法，`equals/hashCode/toString` 来自 `Object`，不拦截就会 NPE（`dispatch.get(method)` 返回 null）。拦截语义也讲究：equals 比较的是 **InvocationHandler 同一性**——同一个 FactoryBean 产物（单例）自比恒 true，不同代理互比 false。运维含义：把 Feign 客户端放进 Set/当 Map key 是安全的（按引用语义），但 **调试时打印代理对象得到 `feign....FeignInvocationHandler@xxx` 而不是接口名**属正常，别期待 toString 输出方法清单——想看代理背后配置，去 actuator 或子上下文拿 Bean。

## 5.2 dispatch 的 key 是 Method 对象

`Method → MethodHandler` 的直接映射让路由是 **HashMap 查找 O(1)**（示例第一阶段用 configKey 字符串聚合、装配时转 Method 键，是骨架简化——真实实现把 `Map<String, MethodHandler> nameToHandler` 再转 `Map<Method, MethodHandler>`，configKey 字符串不进调用热路径）。这带来一个隐蔽但真实的坑：**接口重载时 `Method` 对象唯一，configKey 字符串必须含参数签名**才不撞（`UserClient#getUser(Long)` vs `#getUser(String)`）——§6.1 与 `5_contract.md` 的"重载方法 configKey 相同 → 后注册覆盖"风险在字符串键时代存在，Method 键时代消解，但日志/熔断配置仍以 configKey 为身份，撞名照样在**配置维度**出事。

# 6. 完整代理创建时序图

```text
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

## 6.1 启动耗时花在哪

对上图做耗时预算（百接口规模的经验值）：Registrar 扫描是 classpath IO 的大头（jar 多、包深时 `findAllCandidates` 可占启动数百 ms）；getObject/newInstance 阶段每个客户端一次子上下文创建 + Contract 解析，单接口微秒到毫秒级、总量可忽略；真正拖启动的是**装配期连带组件的初始化**——取 Encoder 触发 HttpMessageConverters 装配、取 Client 触发引擎 Bean 与连接管理器、Nacos/Eureka 客户端首次 registry 拉取。冷启动优化优先级因此是"砍依赖装配 > 砍扫描范围 > 砍代理数"，后者几乎白做。

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

## 7.1 JDK 代理的四条能力边界

`$Proxy` 只能**实现接口**，由此锁定 Feign 客户端的形态约束：① 接口可以有 default 方法（§4.2），代理类会重写它们并同样进入 handler——所以 default 方法也是"经过代理再本地执行"；② 方法参数被自动装箱成 `Object[]`（泛型返回值在 handler 里靠 `metadata.returnType` 才还原类型线索，桥接方法 synthetic 会被 Contract 过滤）；③ **无接口的类无法代理**——`@FeignClient` 标在类上报的"不是接口"错误在 Registrar 校验期；④ 每次调用多一跳 `invoke` 反射分发与装箱，量级微秒，对 RT 毫秒级的远程调用不构成性能议题（`26_performance-tuning.md` 的结论：代理层从来不是瓶颈）。

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

## 8.1 双层代理的洋葱结构

Targeter 路线实际产出**两个代理**：内层 `ReflectiveFeign` 造的执行代理持有完整 dispatch；外层熔断代理把"内层代理的每个方法"再包一层 `circuitBreaker.run`（真实实现为**每个 configKey 一个注册好的熔断器**，`factory.create(name, group)` 后 run ——示例按 target.name() 粒度是简化，粒度差异正是 `14_circuit-breaker.md` §6.1 命名问题的机制根源）。三层对齐 `16_fallback.md` §5：`invoke` 里的 supplier 是"一次完整 Feign 方法执行（含 Retryer 循环）"，所以熔断只数最终结果；fallback 函数在熔断外侧执行，故降级不受熔断统计影响但会被 TimeLimiter 计时（装饰器嵌套顺序 TimeLimiter→CircuitBreaker→...）。default 方法直通不经熔断——接口里的本地组合逻辑被排除在故障统计外，这是特性。

# 9. 关键设计模式

| 模式 | 应用 |
|------|------|
| **代理模式** | JDK 动态代理，`@FeignClient` 接口 → `$Proxy` |
| **工厂模式** | `FeignClientFactoryBean`、`InvocationHandlerFactory` |
| **建造者模式** | `Feign.Builder` 链式构建 |
| **策略模式** | `Contract`、`Encoder`、`Decoder`、`Client` 可插拔 |
| **模板方法** | `ReflectiveFeign.newInstance()` 固定流程，组件可替换 |
| **装饰器模式** | `FeignBlockingLoadBalancerClient` 包装原始 `Client` |

## 9.1 把模式表反向用作排障地图

异常栈里出现哪个类，就知道问题在哪一层、该翻哪篇：`FeignClientsRegistrar`/`NoSuchBeanDefinition` → 注册期扫描与注入（§2）；`FeignClientFactoryBean`/Contract 校验异常 → 装配期配置（§3、`5_contract.md`）；`$Proxy`/`FeignInvocationHandler` 本身几乎不出错，出现即是被下面两层顶上来的：`SynchronousMethodHandler` 栈帧 → 管线逻辑（超时、拦截器、重试，`23_source-request-pipeline.md`）；`FeignBlockingLoadBalancerClient` → 选址（`13_loadbalancer.md`）；`FeignCircuitBreakerInvocationHandler` → 熔断与降级（`14_circuit-breaker.md`、`16_fallback.md`）。读栈时注意 Spring 会把 `method.invoke` 的异常包一层 `UndeclaredThrowableException`/`InvocationTargetException` 再解包——`8_error-handling.md` 里"为什么我 catch 不到 FeignException"的最后一环，就是代理层解包语义（checked 异常必须声明在接口方法上，否则被包装）。
