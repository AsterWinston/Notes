
# 1. 配置来源与优先级

Dubbo 支持多层配置，优先级从高到低：

```
┌────────────────────────────────────────────────────┐
│  优先级                        来源                    │
├────────────────────────────────────────────────────┤
│  1. 方法级                        @DubboService.methods               │
│  2. 接口级                        @DubboService / @DubboReference      │
│  3. 消费者/提供者全局              dubbo.consumer / dubbo.provider       │
│  4. JVM 系统属性                  -Ddubbo.protocol.port=20880          │
│  5. 配置中心（Nacos Config）      外部动态配置                          │
│  6. application.yml / properties                                 │
│  7. dubbo.properties              类路径根目录                   │
└────────────────────────────────────────────────────┘
```

**验证方式**：

```yaml
# application.yml
dubbo:
  protocol:
    port: 20880          # 最终值由优先级最高的配置决定
```

启动时添加 JVM 参数 `-Ddubbo.protocol.port=20881`，最终端口为 20881（JVM 属性优先级更高）。

# 2. 配置方式对比

| 方式 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| **XML** | 早期项目、需要可视化配置 | 结构清晰，IDE 有提示 | 配置冗长，不支持动态 |
| **注解** | 类/方法级个性化配置 | 简洁，就近配置 | 硬编码，修改需重启 |
| **属性文件** | 全局默认配置 | 集中管理，环境隔离 | 不支持方法级 |
| **API** | 框架集成、特殊场景 | 最灵活 | 侵入性强 |
| **配置中心** | 动态配置、集中治理 | 热更新，全局管控 | 有外部依赖 |

# 3. 属性配置（YAML）

## 3.1 基础配置

```yaml
dubbo:
  application:
    name: user-service
    version: 1.0.0
    owner: team-a
    organization: example
    logger: slf4j
    dump-directory: /tmp/dubbo
    qos-enable: true
    qos-port: 22222
    qos-accept-foreign-ip: false

  registry:
    address: nacos://127.0.0.1:8848
    protocol: nacos
    timeout: 30000
    session: 60000
    file: ${user.home}/.dubbo/dubbo-registry.cache
    parameters:
      namespace: dev
      group: DEFAULT_GROUP

  protocol:
    name: dubbo
    port: 20880
    server: netty               # netty / mina / grizzly
    serialization: hessian2
    charset: UTF-8
    threadpool: fixed
    threads: 200
    iothreads: 8
    queues: 0
    accepts: 0
    dispatcher: all

  provider:
    timeout: 3000
    retries: 0
    loadbalance: random
    cluster: failover
    delay: -1
    weight: 100
    executes: 200
    actives: 0
    deprecated: false
    dynamic: true
    accesslog: /tmp/dubbo-access.log

  consumer:
    timeout: 3000
    retries: 2
    loadbalance: random
    cluster: failover
    check: false
    validation: false
    cache: false
    lazy: false
```

## 3.2 多协议配置

```yaml
dubbo:
  protocols:
    dubbo:
      name: dubbo
      port: 20880
    triple:
      name: tri
      port: 50051
```

## 3.3 多注册中心配置

```yaml
dubbo:
  registries:
    nacos:
      address: nacos://10.0.0.1:8848
      parameters:
        namespace: prod
    zk:
      address: zookeeper://10.0.0.2:2181
      default: false
```

# 4. 注解配置

## 4.1 @DubboService（Provider 端）

```java
@DubboService(
    interfaceClass = UserService.class,  // 暴露的接口（默认自动检测）
    interfaceName = "org.example.UserService",
    version = "1.0.0",
    group = "dev",
    path = "user-service",
    timeout = 3000,
    retries = 0,
    loadbalance = "roundrobin",
    cluster = "failover",
    validation = "true",
    proxy = "javassist",
    actives = 100,
    executes = 200,
    weight = 100,
    delay = -1,
    dynamic = true,
    register = true,                     // 是否注册到注册中心
    protocol = "dubbo,tri",
    registry = "nacos",
    // 方法级配置
    methods = {@Method(
        name = "createUser",
        timeout = 5000,
        retries = 0,
        executes = 50,
        validation = "true"
    )}
)
public class UserServiceImpl implements UserService {
    // ...
}
```

## 4.2 @DubboReference（Consumer 端）

```java
@DubboReference(
    interfaceClass = UserService.class,
    version = "1.0.0",
    group = "dev",
    timeout = 3000,
    retries = 2,
    loadbalance = "random",
    cluster = "failover",
    check = false,
    mock = "return null",
    cache = "lru",
    async = false,
    validation = "true",
    // 方法级配置
    methods = {@Method(
        name = "getUser",
        timeout = 1000,
        retries = 0
    )}
)
private UserService userService;
```

# 5. 配置中心集成

Dubbo 支持将配置外部化到 Nacos/Apollo/Zookeeper 等配置中心，实现动态配置。

## 5.1 依赖

```xml
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-configcenter-nacos</artifactId>
</dependency>
```

## 5.2 配置

```yaml
dubbo:
  config-center:
    address: nacos://127.0.0.1:8848
    namespace: dev
    group: dubbo
    highest-priority: true        # 外部配置优先级是否最高
    check: true                   # 启动时是否检查配置中心可用
```

## 5.3 动态配置生效流程

```
Nacos Config Center
│
├─ 外部化配置（external-configuration）
│   └─ dataId: {app-name}.properties  → 覆盖本地 dubbo 配置
│
├─ 服务治理（configurators）
│   └─ 覆盖提供者 URL 参数 → Router / LoadBalance 实时生效
│
└─ 应用级配置（app-config）
    └─ dataId: {app-name}-dubbo.properties  → 应用级共享配置
```

## 5.4 动态配置中心源码

```java
public interface Configuration {
    Object getInternalProperty(String key);
    // 支持获取方法级配置: key = "dubbo.service.{method}.timeout"
}

// 配置优先级组合
public abstract class AbstractDynamicConfiguration 
    implements Configuration {

    // 组合多个 Configuration 源
    protected CompositeConfiguration compositeConfig;

    // 优先级: JVM > XML > application.yml > 配置中心 > dubbo.properties
}
```

# 6. XML 配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
       xsi:schemaLocation="http://dubbo.apache.org/schema/dubbo
         http://dubbo.apache.org/schema/dubbo/dubbo.xsd">

    <!-- 应用名 -->
    <dubbo:application name="user-service"/>

    <!-- 注册中心 -->
    <dubbo:registry address="nacos://127.0.0.1:8848"/>

    <!-- 协议 -->
    <dubbo:protocol name="dubbo" port="20880"/>

    <!-- 服务提供者 -->
    <dubbo:service interface="org.example.UserService"
                   ref="userServiceImpl"
                   version="1.0.0"
                   timeout="3000"/>

    <bean id="userServiceImpl" class="org.example.provider.UserServiceImpl"/>

    <!-- 服务消费者 -->
    <dubbo:reference id="userService"
                     interface="org.example.UserService"
                     version="1.0.0"
                     timeout="3000"/>
</beans>
```

# 7. API 配置

```java
public class ApiConfigExample {
    public static void main(String[] args) {
        // 注册中心
        RegistryConfig registry = new RegistryConfig();
        registry.setAddress("nacos://127.0.0.1:8848");

        // 协议
        ProtocolConfig protocol = new ProtocolConfig();
        protocol.setName("dubbo");
        protocol.setPort(20880);

        // 应用
        ApplicationConfig application = new ApplicationConfig();
        application.setName("user-service");

        // Provider: 暴露服务
        ServiceConfig<UserService> service = new ServiceConfig<>();
        service.setApplication(application);
        service.setRegistry(registry);
        service.setProtocol(protocol);
        service.setInterface(UserService.class);
        service.setRef(new UserServiceImpl());
        service.setVersion("1.0.0");
        service.setTimeout(3000);
        service.export();

        // Consumer: 引用服务
        ReferenceConfig<UserService> reference = new ReferenceConfig<>();
        reference.setApplication(application);
        reference.setRegistry(registry);
        reference.setInterface(UserService.class);
        reference.setVersion("1.0.0");
        reference.setTimeout(3000);

        UserService userService = reference.get();
        User user = userService.getUser(1L);

        new CountDownLatch(1).await();
    }
}
```

# 8. 配置覆盖规则

同一属性在不同层级的覆盖关系以 `timeout` 为例：

```
方法级 timeout                    ← 最高优先级 (方法维度)
  ↓
@DubboService(timeout=3000)       ← 接口级
  ↓
dubbo.provider.timeout=2000       ← 提供者全局
  ↓
-Ddubbo.provider.timeout=1500     ← JVM 属性
  ↓
dubbo.properties: timeout=1000    ← 配置文件
```

**消费端跨越多层的超时优先级**：

```
Consumer 方法级 > Consumer 接口级 > Consumer 全局 > Provider 方法级 > Provider 接口级 > Provider 全局
```

# 9. 配置类继承体系

```
AbstractConfig
├── ApplicationConfig          (应用配置)
├── RegistryConfig             (注册中心配置)
├── ProtocolConfig             (协议配置)
├── ProviderConfig             (提供者默认配置)
├── ConsumerConfig             (消费者默认配置)
├── MonitorConfig              (监控中心配置)
├── ConfigCenterConfig         (配置中心配置)
├── MetadataReportConfig       (元数据中心配置)
├── SslConfig                  (SSL 配置)
├── MethodConfig               (方法级配置)
├── ArgumentConfig             (方法参数回调配置)
│
├── AbstractServiceConfig
│   ├── ServiceConfig          (服务提供者)
│   └── ServiceConfigBase
│
└── AbstractReferenceConfig
    └── ReferenceConfig         (服务消费者)
```

# 10. 配置属性覆盖 API

```java
@Service
public class CustomConfigPostProcessor 
        implements BeanFactoryPostProcessor {

    @Override
    public void postProcessBeanFactory(
            ConfigurableListableBeanFactory beanFactory) {
        // 动态覆盖协议端口
        System.setProperty("dubbo.protocol.port", 
            String.valueOf(determinePort()));
    }
}
```

也可以通过 `DubboBootstrap` API 直接设置：

```java
DubboBootstrap.getInstance()
    .application(new ApplicationConfig("my-app"))
    .registry(new RegistryConfig("nacos://127.0.0.1:8848"))
    .protocol(new ProtocolConfig("dubbo", 20880))
    .service(serviceConfig)
    .reference(referenceConfig)
    .start();
```
