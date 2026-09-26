# 1. 知识点
- **单体架构 vs 微服务架构**：单体将所有功能打包在一个应用中，部署简单但耦合度高；微服务将应用拆分为多个独立服务，每个服务独立开发、部署、扩展。
- **独立部署**：微服务每个服务可独立构建、测试、部署，不影响其他服务，提高发布频率和灵活性。
- **独立扩展**：可针对瓶颈服务单独扩容，而非整体扩容，资源利用率更高。
- **分布式复杂度**：微服务引入分布式系统问题，包括网络延迟、分布式事务、服务发现、容错等。
- **数据一致性**：分布式环境下无法依赖单机ACID事务，需引入最终一致性、Saga、TCC等方案。
- **网络延迟**：服务间RPC调用增加网络开销，需考虑超时、重试、熔断等机制。
- **运维成本**：微服务数量多，需要自动化部署、监控、日志收集、链路追踪等基础设施。
- **服务粒度评估**：按业务域拆分（DDD限界上下文）、按团队组织拆分、按变更频率拆分、按数据依赖拆分。
- **Spring Cloud版本命名**：从2020.0开始采用日历版本命名（以前为伦敦地铁站名：Greenwich、Hoxton等），使用`<major>.<minor>.<patch>`格式。
- **Spring Cloud 2020.x（Ilford）**：基于Spring Boot 2.4.x-2.5.x，移除了spring-cloud-netflix大部分模块。
- **Spring Cloud 2021.x（Jubilee）**：基于Spring Boot 2.6.x-2.7.x，全面推荐使用Spring Cloud LoadBalancer替代Ribbon。
- **Spring Cloud 2022.x（Kilburn）**：基于Spring Boot 3.0.x，全面升级至Spring Framework 6和Jakarta EE。
- **Spring Boot与Spring Cloud版本对应**：必须严格匹配，Spring Cloud每个Release Train规定了对应的Spring Boot版本范围，不匹配会导致兼容性问题。
- **Netflix技术栈停更**：Eureka 2.0停止开发、Hystrix进入维护模式、Ribbon进入维护模式、Zuul 1.x被Gateway替代、Archaius进入维护模式。
- **迁移至Alibaba方案**：Nacos替代Eureka+Config、Sentinel替代Hystrix、Spring Cloud LoadBalancer替代Ribbon、Gateway替代Zuul。
- **服务注册中心对比（Nacos）**：CAP倾向AP/CP可切换，健康检查通过TCP/HTTP/MySQL上报心跳，一致性协议为Distro（AP）或JRaft（CP），支持多数据中心。
- **服务注册中心对比（Eureka）**：CAP倾向AP，健康检查为客户端心跳+自我保护机制，一致性协议为Peer to Peer弱一致性，多数据中心支持较弱。
- **服务注册中心对比（Consul）**：CAP倾向CP，健康检查支持HTTP/TCP/Script/gRPC等多种方式，一致性协议为Raft，原生支持多数据中心。
- **服务注册中心对比（ZooKeeper）**：CAP倾向CP，健康检查通过临时节点+Session心跳，一致性协议为ZAB，不支持多数据中心。
- **注册中心增量拉取**：客户端首次全量拉取注册表后，后续通过增量方式获取变更信息，减少带宽和延迟。
- **心跳续约**：服务提供者定时向注册中心发送心跳（默认30s），注册中心超过一定时间未收到心跳则剔除该实例。
- **自我保护机制**：Eureka在15分钟内心跳失败比例低于85%时触发，不再剔除过期实例，防止因网络分区导致大面积下线。
- **Ribbon停更**：已进入维护模式，官方建议迁移到Spring Cloud LoadBalancer，但仍可在旧项目中使用。
- **RestTemplate vs WebClient**：RestTemplate为同步阻塞HTTP客户端（Servlet方式），WebClient为异步非阻塞（Reactive），官方推荐新项目使用WebClient。
- **Spring Cloud LoadBalancer**：替代Ribbon的负载均衡组件，默认使用轮询算法，支持自定义切换到随机等策略，可通过`@LoadBalancerClient`注解配置。
- **OpenFeign**：声明式HTTP客户端，通过接口+注解的方式定义远程调用，内部集成负载均衡，简化微服务间调用。
- **Feign vs OpenFeign**：Feign是Netflix的开源项目（已停更），OpenFeign是Spring Cloud社区继续维护的版本，增加了Spring MVC注解支持等特性。
- **Sentinel vs Hystrix**：Hystrix已停更，Sentinel是阿里开源的替代方案，功能更丰富（热点限流、系统规则、控制台可视化），支持规则动态配置。
- **Resilience4j**：轻量级容错库，为Java 8+函数式编程设计，提供熔断、限流、重试、隔离等能力，可作为Sentinel的替代。
- **Nacos Config**：Naocs提供的配置中心功能，支持配置的动态刷新、灰度发布、命名空间隔离、配置回滚等。
- **Apollo配置中心**：携程开源的配置管理平台，支持多环境、多集群、灰度发布、权限管理、配置审计等企业级功能。
- **Spring Cloud Config**：基于Git/SVN存储配置，通过Config Server统一管理，结合Spring Cloud Bus实现配置动态刷新。
- **动态刷新**：使用`@RefreshScope`注解标记需要动态刷新的Bean，通过`/actuator/bus-refresh`端点触发配置刷新（需Spring Cloud Bus支持）。
- **Spring Cloud Gateway vs Zuul**：Zuul 1.x为同步阻塞（Servlet），Gateway基于Spring WebFlux异步非阻塞，性能更优；Zuul 2.x一直未发布正式版。
- **链路追踪变更**：Spring Cloud Sleuth已停更，3.x版本改用Micrometer Tracing + Brave + Zipkin实现分布式链路追踪。
- **SkyWalking**：Apache开源APM工具，支持多语言探针、服务拓扑图、性能指标分析、告警等，无需代码侵入。
- **Seata**：阿里巴巴开源的分布式事务解决方案，支持AT、TCC、Saga、XA四种模式，通过TC协调全局事务。
- **Spring Cloud vs Spring Cloud Alibaba**：Spring Cloud是标准规范，Netflix组件逐渐停更；Spring Cloud Alibaba是阿里实现，组件活跃度高、功能丰富、中文文档完善。
- **微服务 vs Dubbo**：Spring Cloud基于HTTP/RESTful，生态完善（网关、配置中心、链路追踪）；Dubbo基于RPC，性能更高，但生态需自行整合。
- **Service Mesh**：通过Sidecar代理（如Istio+Envoy）处理服务间通信，将网络层下沉到基础设施，应用代码无侵入，但增加运维复杂度。
- **不同服务之间的通信方式**: OpenFeign(声明式HTTP客户端，基于REST API调用，适用于Spring Cloud微服务间通信，简单易用); Dubbo(高性能RPC框架，基于TCP协议，效率更高，适用于对性能要求较高的内部服务通信); gRPC(基于HTTP2 + ProtoBuf，跨语言，支持双向流); Spring Cloud默认用OpenFeign
- **限流/降级/熔断的区别**: 限流—限制访问频率; 熔断—直接切断故障服务的调用; 降级—降低服务质量保护核心功能

# 2. 面试题

## 2.1 微服务架构相比单体架构有哪些优缺点？
优点：
1. 独立部署：每个服务可独立发布，不影响其他服务，发布频率更高
2. 独立扩展：可按服务粒度扩缩容，资源利用率高
3. 技术异构：不同服务可使用不同技术栈和数据库
4. 团队自治：每个微服务由小团队独立负责，开发效率高
5. 故障隔离：一个服务故障不会导致整个系统瘫痪

缺点：
1. 分布式复杂度：引入网络通信、数据一致性、分布式事务等问题
2. 运维成本高：需要自动化CI/CD、容器编排、监控告警、链路追踪等基础设施
3. 调试困难：请求跨多个服务，问题定位复杂
4. 网络延迟：服务间RPC调用增加额外开销
5. 数据一致性：数据库被拆分，跨服务数据操作需要分布式事务

## 2.2 如何合理评估微服务粒度？
1. 业务领域驱动：按照DDD限界上下文划分，一个限界上下文对应一个微服务
2. 团队规模：遵循"两个披萨原则"，一个团队能维护的微服务数量不宜过多
3. 数据独立性：每个微服务应拥有自己的数据库，避免多个服务共享数据库
4. 变更频率：将频繁变更的模块独立成服务，稳定模块可合并
5. 可独立测试：一个服务应能独立完成功能测试，无需依赖其他服务全部启动
6. 过小的问题：服务过小会导致"微服务爆炸"，RPC调用链过长，事务管理复杂
7. 过大的问题：服务过大会丧失微服务优势，部署慢、耦合高

## 2.3 Nacos、Eureka、Consul、ZooKeeper如何选型？
| 对比维度 | Nacos | Eureka | Consul | ZooKeeper |
|---------|-------|--------|--------|-----------|
| CAP倾向 | AP/CP可切换 | AP | CP | CP |
| 一致性协议 | Distro(AP)/JRaft(CP) | Peer to Peer | Raft | ZAB |
| 健康检查 | TCP/HTTP/MySQL | 客户端心跳 | HTTP/TCP/Script | 临时节点+Session |
| 多数据中心 | 支持 | 弱支持 | 原生支持 | 不支持 |
| 配置中心 | 内置 | 无 | 内置KV | 可用但不推荐 |
| 易用性 | 高（中文文档） | 高 | 中 | 低 |
| 推荐场景 | 国内主流选择 | 纯注册中心 | 多数据中心 | 成熟ZooKeeper技术栈 |

选型建议：
- 新项目国内首选Nacos（注册+配置二合一，阿里维护活跃）
- 需多数据中心选Consul
- 已有ZooKeeper技术栈可继续使用
- Eureka已停更，不建议新项目使用

## 2.4 Eureka的自我保护机制是什么？
自我保护机制是Eureka在CAP中偏向AP的重要体现：

1. 触发条件：15分钟内，心跳续约失败的比例低于85%
2. 触发行为：Eureka Server进入自我保护模式，不再剔除超过TTL的实例
3. 设计目的：防止因网络分区导致Eureka Server误将大量健康实例剔除
4. 副作用：可能保留已下线的实例，导致调用方拿到不可用节点
5. 配置参数：
   - `eureka.server.enable-self-preservation=true`（默认开启）
   - `eureka.server.renewal-percent-threshold=0.85`（触发阈值）
6. 生产建议：一般建议开启，避免网络不稳定时大面积误下线

## 2.5 OpenFeign的核心原理是什么？
OpenFeign基于动态代理和声明式注解实现远程调用：

1. **接口扫描**：`@EnableFeignClients`触发`@FeignClient`接口扫描，将接口的BeanDefinition注册到Spring容器
2. **代理创建**：`FeignClientFactoryBean`通过FactoryBean机制，为每个`@FeignClient`接口创建JDK动态代理对象
3. **契约解析**：`Contract`组件解析接口上的Spring MVC注解（`@RequestMapping`、`@GetMapping`等），生成`MethodMetadata`
4. **请求模板**：将方法调用信息封装为`RequestTemplate`，包括URL、请求方法、请求参数、请求头等
5. **拦截器链**：`RequestInterceptor`在请求发送前修改`RequestTemplate`，常用于添加认证Token、TraceId等
6. **执行请求**：`Client`组件通过HTTP客户端（默认URLConnection，可替换为OkHttp或Apache HttpClient）发送请求
7. **解码响应**：`Decoder`将HTTP响应Body反序列化为方法返回值类型

## 2.6 @RefreshScope实现配置自动刷新的原理？
1. `@RefreshScope`注解的Bean属于`refresh`作用域，其创建时通过CGLIB生成代理对象
2. Bean实例存储在`ScopeCache`中，默认使用ConcurrentHashMap
3. 当配置发生变更（如Nacos发布配置），`RefreshEventListener`监听到RefreshEvent事件
4. 调用`ContextRefresher.refresh()`方法重新创建Environment对象
5. 清空RefreshScope的缓存，当下次访问Bean时会重新创建，使用新的配置值
6. 注意：`@ConfigurationProperties`标注的Bean天然支持配置刷新，无需`@RefreshScope`

## 2.7 Sentinel 和 Hystrix 的区别是什么？
| 对比维度 | Sentinel | Hystrix |
|---------|----------|---------|
| 隔离策略 | 信号量隔离 | 线程池隔离（默认） |
| 熔断降级 | 慢调用比例/异常比例/异常数 | 异常比例 |
| 控制台 | 可视化Dashboard | 简单Dashboard（已不维护） |
| 规则配置 | 动态规则、控制台推送 | 代码硬编码或配置文件 |
| 统计精度 | 滑动窗口（秒级） | 滑动窗口（粗略） |
| 额外功能 | 热点限流、系统规则、网关限流 | 无 |
| 维护状态 | 活跃维护（阿里巴巴） | 已停更 |

## 2.8 Spring Cloud Gateway 的工作流程是什么？
1. 客户端请求到达Gateway
2. `RoutePredicateHandlerMapping`根据路由断言匹配目标Route，若未匹配返回404
3. 匹配成功后，`FilteringWebHandler`执行前置过滤器链（`GatewayFilter` + `GlobalFilter`）
4. 过滤器链中的过滤器可以修改请求（添加Header、重写路径、鉴权等）
5. 请求通过`NettyRoutingFilter`转发到后端微服务
6. 后端服务处理请求返回响应
7. 响应经过后置过滤器链处理后返回给客户端

## 2.9 Seata AT 模式的一阶段和二阶段分别做什么？
**一阶段**：
1. 解析SQL语句，获取SQL类型（INSERT/UPDATE/DELETE）、表名、条件等
2. 查询SQL执行前的数据镜像（before-image），保存为undo_log
3. 执行原SQL，将数据写入数据库（未提交）
4. 查询SQL执行后的数据镜像（after-image）
5. 将before-image和after-image以及XID、BranchID等存入undo_log表
6. 注册分支事务到TC，获取全局行锁
7. 提交本地事务（一阶段即提交，这是AT模式的核心特点）

**二阶段-提交**：
1. TC通知各RM全局提交
2. RM异步删除对应XID的undo_log记录
3. 提交本地事务（实际上一阶段已提交，此处只需清理undo_log）

**二阶段-回滚**：
1. TC通知各RM全局回滚
2. RM根据undo_log中的before-image构建反向补偿SQL
3. 校验当前数据与after-image是否一致（防止脏写）
4. 若一致则执行补偿SQL，将数据恢复为before-image
5. 删除undo_log记录

## 2.10 微服务实施中如何选择 Spring Cloud 还是 Spring Cloud Alibaba？
**选 Spring Cloud（Netflix版）的场景**：
- 团队已有Eureka/Hystrix等Netflix组件经验
- Spring Boot版本较低（2.3及以下），升级成本高
- 项目规模小，无需复杂治理功能
- 注意：Netflix组件已基本停更，长期维护风险高

**选 Spring Cloud Alibaba 的场景**：
- 新项目启动，组件活跃维护
- 需要注册中心+配置中心一体化（Nacos）
- 需要完善的限流熔断功能（Sentinel+控制台）
- 需要分布式事务支持（Seata）
- 中文社区、文档丰富，国内使用广泛
- Spring Boot 2.4+推荐

**建议**：新项目普通推荐Spring Cloud Alibaba，组件维护活跃、功能完整、学习成本低。

## 2.11 什么是 Service Mesh？它和 Spring Cloud 微服务体系有何区别？
**Service Mesh**：通过Sidecar代理（如Envoy）透明处理服务间通信，将网络层（负载均衡、熔断、限流、链路追踪、安全等）下沉到基础设施层。

**核心区别**：
| 维度 | Spring Cloud | Service Mesh |
|------|-------------|--------------|
| 实现位置 | 应用内（SDK/库） | 基础设施（Sidecar） |
| 语言耦合 | Java专属 | 语言无关 |
| 升级方式 | 修改代码+重新部署 | Sidecar更新，应用无感知 |
| 性能开销 | 较小（进程内） | 较大（两次网络代理） |
| 运维复杂度 | 低 | 高（需管理Sidecar+控制面） |
| 生态 | Spring + Netflix/Alibaba | Istio + Envoy + K8s |
| 学习成本 | 中 | 高 |

**适用场景**：
- Spring Cloud：中小团队，Java技术栈统一，快速开发
- Service Mesh：多语言异构系统，大型组织，已有K8s基础设施
- 两者也可以混合使用（Spring Cloud应用部署在Service Mesh中）

## 2.12 不同服务之间如何进行通信？
| 方式 | 协议 | 特点 | 适用场景 |
|------|------|------|----------|
| **OpenFeign** | HTTP/REST | 声明式HTTP客户端，注解驱动，简单易用，集成负载均衡和熔断 | Spring Cloud微服务间通信（默认方案） |
| **Dubbo** | TCP/自定义协议 | 高性能RPC框架，支持多种序列化，服务治理完善 | 内部服务高性能调用，Java技术栈 |
| **gRPC** | HTTP/2 + ProtoBuf | 跨语言，支持双向流，基于Protocol Buffers序列化 | 多语言异构系统，需要流式传输 |
| **RestTemplate/WebClient** | HTTP/REST | 原始HTTP调用，灵活但需手动编码 | 简单场景，非声明式需求 |

Spring Cloud生态默认使用OpenFeign。对性能要求极高的内部服务间调用可考虑Dubbo。

## 2.13 限流、降级和熔断有什么区别？
- **限流（Rate Limiting）**：限制访问频率，超出阈值的请求被拒绝或排队。属于主动预防机制，防止系统被流量冲垮。如Sentinel基于QPS/并发线程数限流。
- **熔断（Circuit Breaking）**：当依赖服务不可用或响应过慢时，直接切断对该服务的调用，快速失败。属于被动止损机制，防止故障扩散（级联故障/雪崩效应）。如Sentinel基于慢调用比例/异常比例/异常数触发熔断。
- **降级（Degradation）**：降低服务质量，返回默认值/缓存数据/简化功能，保证核心功能可用。属于妥协兜底机制。如Feign Fallback返回兜底数据。

三者协同工作：限流在入口拦截流量 → 超出系统能力时触发降级 → 下游不可用时触发熔断并执行降级。

## 2.14 Spring Cloud 和 Spring Boot 的版本对应关系
Spring Cloud 和 Spring Boot 有严格的版本匹配要求，版本不对应会导致兼容性问题：

| Spring Cloud 版本 | 版本号规则 | 对应 Spring Boot |
|------------------|-----------|-----------------|
| 2023.0.x (Leyton) | 23 → 2023.0.x | 3.2.x |
| 2022.0.x (Kilburn) | 22 → 2022.0.x | 3.0.x |
| 2021.0.x (Jubilee) | 21 → 2021.0.x | 2.6.x / 2.7.x |
| 2020.0.x (Ilford) | 20 → 2020.0.x | 2.4.x / 2.5.x |
| Hoxton（末代旧规则） | 字母命名 | 2.2.x / 2.3.x |
| Greenwich | 字母命名 | 2.1.x |

**命名规则变化**：Spring Cloud 从 Hoxton 之后放弃了伦敦地铁站字母命名，改为日历版本号 YYYY.MINOR.MICRO（同 Spring Boot）。版本号前缀对应年份。

**关键对应关系**：
- Spring Cloud 2020.0.x → Spring Boot 2.4.x / 2.5.x（Netflix 组件停更，集成 Spring Cloud LoadBalancer、Spring Cloud Gateway）
- Spring Cloud 2021.0.x → Spring Boot 2.6.x / 2.7.x（Eureka 2.0 移除）
- Spring Cloud 2022.0.x → Spring Boot 3.0.x（Jakarta EE 迁移，Spring Cloud Alibaba 2022.x）
- Spring Cloud 2023.0.x → Spring Boot 3.2.x（虚拟线程支持）

**如何确认版本**：使用 Spring Initializr 或官方兼容性表；直接引用 `spring-cloud-dependencies` BOM 并指定版本。

## 2.15 Spring Cloud LoadBalancer 的工作原理和与 Ribbon 的区别
**为什么替代 Ribbon**：Netflix 于 2018 年宣布 Ribbon 进入维护模式（不再新增特性），Spring Cloud 2020.0 起不再支持 Ribbon。

| 维度 | Ribbon | Spring Cloud LoadBalancer |
|------|--------|--------------------------|
| 状态 | 停更维护 | 官方推荐 |
| 负载均衡算法 | 丰富（轮询、加权、随机、一致性哈希等） | 基础（轮询 + 随机 + 自定义） |
| 响应式 | 不支持 | 原生支持 Spring WebFlux |
| 健康检查 | 内置（定期 Ping） | 依赖服务发现组件的健康检查 |
| 缓存 | 自带服务列表缓存 | 利用 Spring Cloud 服务发现缓存 |
| 配置方式 | Ribbon 专有配置 | Spring 标准配置 + 自定义 Bean |

**Spring Cloud LoadBalancer 工作流程**：
1. 与 OpenFeign / WebClient 集成：通过 `LoadBalancerClient` 拦截请求 URL（`http://service-name/api/...`）。
2. 从服务发现（Nacos/Eureka）获取 `service-name` 对应的所有实例列表。
3. 根据 `ReactorLoadBalancer<ServiceInstance>` 选择具体实例（默认轮询）。
4. 替换 URL 为 `http://实际IP:Port/api/...` 后发出请求。

**自定义负载均衡策略**：
```java
@Bean
public ReactorLoadBalancer<ServiceInstance> customLoadBalancer(Environment env, LoadBalancerClientFactory factory) {
    String name = env.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
    return new RandomLoadBalancer(factory.getLazyProvider(name, ServiceInstanceListSupplier.class), name);
}
```

## 2.16 配置中心组件的技术选型（Nacos Config vs Apollo vs Spring Cloud Config）
| 维度 | Spring Cloud Config | Nacos Config | Apollo（携程） |
|------|-------------------|-------------|--------------|
| 存储后端 | Git / 本地文件 / Vault | 内置 MySQL/内嵌存储 | 内置 MySQL |
| 动态刷新 | 需配合 Bus + MQ（RabbitMQ/Kafka） | 原生支持（长轮询） | 原生支持（长轮询+HTTP） |
| 灰度发布 | 支持（Profile/Label） | 支持（命名空间+Group） | 原生支持 |
| 权限管理 | 简单（Git 权限） | 基础 | 完善（多环境/多集群/多项目） |
| 配置界面 | 无（需单独部署 Config Server） | 内置 Web 管理界面 | 专业级 Portal |
| 多环境 | Profile + Git 分支 | Namespace | Namespace + Env |
| 审计日志 | 无 | 无 | 完整的配置发布审计 |
| 部署复杂度 | 中（需 Config Server + Git/MQ） | 低（Nacos 自带） | 高（需 MySQL + Portal + Admin + Config Service） |
| 适用团队 | 小型团队，Git 管理配置 | 企业级，希望配置和注册中心统一 | 大企业，对配置权限/审计要求高 |

**选型建议**：
- 如果已经用了 Nacos 作为注册中心 → 直接用 Nacos Config（一个管控台统一管理注册+配置）。
- 大企业需要配置发布审批、版本对比、操作审计 → Apollo。
- 团队规模小、不额外部署中间件 → Spring Cloud Config + Git 够用。
