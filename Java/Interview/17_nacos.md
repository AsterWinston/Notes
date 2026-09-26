# 1. 知识点
- **核心功能二合一**：Nacos（Naming and Configuration Service）同时提供服务注册发现和配置中心两大功能，一个组件替代Eureka+Config。
- **CAP倾向**：默认采用AP模式保证可用性，可切换为CP模式使用Raft一致性协议保证强一致性。
- **注册中心CP/AP选择**：对数据一致性要求高的场景（如金融交易）选CP；对可用性要求高的场景（如电商首页展示）选AP。
- **数据模型-命名空间**：Namespace用于实现多环境隔离（dev/test/prod），不同Namespace的服务和配置完全隔离。
- **数据模型-Group分组**：在同一个Namespace内，Group用于进一步隔离服务或配置，如区分不同业务线。
- **数据模型-Service服务名**：服务的唯一标识，客户端通过服务名获取实例列表。
- **数据模型-Cluster集群**：Service下可划分多个Cluster，实现同机房就近访问，降低跨机房延迟。
- **数据模型-Instance实例**：表示一个具体的服务节点，包含IP、端口、权重、健康状态等元数据。
- **权重**：实例权重决定流量分配比例（0-1之间），权重越大分配流量越多，可实现流量控制和灰度发布。
- **元数据**：实例的扩展KV属性，可自定义信息，如版本号、机房、灰度标签等，用于服务治理。
- **服务注册流程**：客户端启动后向Nacos Server发送注册请求，上报IP、端口、元数据等信息。
- **注册表存储**：Nacos Server使用ConcurrentHashMap存储服务实例，内存中维护全量注册信息。
- **心跳机制**：客户端每5秒向Server发送心跳维持实例存活，Server每15秒扫描实例，超过15秒未收到心跳标记不健康，30秒后剔除。
- **临时实例与持久化实例**：临时实例（ephemeral=true）基于心跳保活，适用微服务；持久化实例（ephemeral=false）不依赖心跳，适用基础服务。
- **服务发现-Pull模式**：客户端定时从Server拉取注册表（默认30s），获取最新实例列表。
- **服务发现-Push模式**：当服务变更时，Server通过UDP推送变更通知给订阅了该服务的客户端，实现近实时更新。
- **自我保护机制**：当健康实例数占总实例数比例低于阈值（默认15%）时，Nacos触发自我保护，不再剔除不健康实例，防止大面积误下线。
- **Distro一致性协议**：Nacos自研的AP协议，用于临时实例的同步，保证最终一致性，各节点都可读写。
- **JRaft CP协议**：基于Raft算法实现强一致性，用于持久化实例的数据同步，保证数据强一致。
- **配置中心功能**：支持配置的动态管理，包括创建、修改、删除、查询、历史版本回滚。
- **配置监听**：客户端通过长轮询（Long Polling）监听配置变更，Server变更后通知客户端刷新。
- **配置文件Profile**：通过`${spring.profiles.active}`实现不同环境的配置管理，Nacos会自动加载对应profile的配置。
- **命名空间读配置**：不同命名空间的配置独立，客户端通过配置namespace参数隔离环境。
- **共享配置**：通过`shared-configs`参数引用多个命名空间的共享配置，多个微服务共用基础配置。
- **配置自动刷新**：Spring Cloud结合`@RefreshScope`实现Bean级别的配置动态刷新。
- **@Value刷新**：添加`@RefreshScope`后，`@Value`注入的属性可在配置变更后自动更新。
- **@ConfigurationProperties刷新**：天然支持配置刷新，变更后自动生效，不需要额外注解。
- **配置优先级**：profile配置 > 扩展配置（extension-configs） > 共享配置（shared-configs） > 本地配置文件。
- **灰度发布配置**：在Nacos控制台可以对配置进行灰度发布，选择指定的IP或标签下放配置，实现配置的灰度验证。
- **控制台管理**：Nacos Server自带Web控制台，支持服务管理、配置管理、权限管理、命名空间管理等。

# 2. 面试题

## 2.1 Nacos 同时做注册中心和配置中心的优势是什么？
1. **架构简化**：一个组件替代Eureka+Config Server，减少运维成本和网络跳数
2. **统一管理**：服务注册和配置在同一个Web控制台，统一数据模型（Namespace、Group）
3. **一致的数据模型**：配置和服务使用相同的Namespace/Group概念，便于多环境管理
4. **减少依赖**：不需要额外部署Config Server和消息中间件，降低系统复杂度
5. **性能优化**：配置和服务数据在同一进程内存中，查询效率更高
6. **统一权限**：一套权限体系管理注册和配置，安全管控更便捷

## 2.2 Nacos CAP 模式如何选择？在什么场景下切换？
**AP模式（默认）**：
- 场景：互联网电商、社交应用等对可用性要求高的场景
- 特点：数据可能短暂不一致，但保证系统可用
- 协议：Distro，各节点平等，数据异步同步
- 风险：短时间可能拿到过期的实例列表

**CP模式**：
- 场景：金融交易、支付结算等对一致性要求高的场景
- 特点：牺牲部分可用性，保证数据强一致
- 协议：JRaft，Leader单一，数据同步写入多数节点才返回
- 风险：Leader选举期间可能短暂不可用

**切换方式**：
```yaml
# AP模式（默认）
spring.cloud.nacos.discovery.ephemeral=true
# CP模式（持久化实例）
spring.cloud.nacos.discovery.ephemeral=false
```

## 2.3 Nacos 服务注册和发现的完整流程是怎样的？
**注册流程**：
1. 服务提供者启动后，构建Instance对象（IP、Port、metadata）
2. 调用Nacos NamingService的registerInstance方法发送注册请求
3. Nacos Server将实例存入内存ConcurrentHashMap（key: namespace@@group@@serviceName）
4. 客户端启动心跳任务（每5秒发送一次心跳）
5. 服务端每15秒检查实例，超过15秒无心跳标记为不健康，超过30秒剔除

**发现流程**：
1. 服务消费者启动后，调用subscribe方法订阅目标服务
2. 客户端首次全量拉取注册表（HTTP GET请求）
3. 后续每30秒增量拉取变更数据
4. 服务端在服务实例变更时通过UDP推送变更通知
5. 客户端更新本地缓存，LoadBalancer根据最新实例列表进行负载均衡

## 2.4 Nacos的心跳机制和自我保护机制详细说明
**心跳机制**：
- 客户端每5秒向Nacos Server发送一次心跳请求
- 请求携带serviceName、ip、port、cluster等信息
- Server更新该实例的最后心跳时间戳
- Server每15秒执行一次健康检查任务
- 超过15秒未收到心跳的实例标记为unhealthy
- 超过30秒未收到心跳的实例直接剔除

**自我保护机制**：
- 触发条件：当前健康实例数 / 总实例数 < 自我保护阈值（默认0.85，即15%实例不健康时触发）
- 触发后行为：
  1. Server不再剔除任何已超时的实例
  2. 控制台显示"服务处于自我保护模式"
- 恢复条件：健康实例比例恢复正常
- 设计目的：防止网络分区导致Server误将大量健康实例下线
- 配置项：`nacos.naming.self-protection.threshold`

## 2.5 Nacos 配置中心的动态刷新原理
1. 客户端启动时根据`dataId`、`group`、`namespace`获取配置
2. 客户端建立与Nacos Server的长轮询连接（HTTP Long Polling）
3. 客户端发送配置监听请求，携带当前配置的MD5值
4. Server对比MD5：
   - 配置未变更：hold住请求29.5秒后返回空
   - 配置已变更：立即返回新配置内容和MD5
5. 客户端收到变更后触发`RefreshEvent`事件
6. `RefreshEventListener`监听到事件，调用`ContextRefresher.refresh()`
7. 如果Bean上有`@RefreshScope`：
   - 清空Scope缓存
   - 下次访问时重新创建Bean实例
8. 如果有`@ConfigurationProperties`：
   - 重新绑定属性值到Bean

## 2.6 Nacos配置管理的配置优先级是怎样的？
从高到低：
1. 当前应用profile的配置（`${spring.application.name}-${profile}.${file-extension}`）
2. 不带profile的应用配置（`${spring.application.name}.${file-extension}`）
3. 扩展配置（extension-configs），按配置顺序，后面覆盖前面
4. 共享配置（shared-configs），按配置顺序，后面覆盖前面
5. 本地配置文件（`application.yml` / `bootstrap.yml`）

实际使用时：
- 扩展配置和共享配置按数组顺序，索引越大优先级越高
- `refresh`属性控制是否动态刷新（默认false）
- 多个微服务共享的公共配置放在shared-configs
- 特定场景的额外配置放在extension-configs

## 2.7 Nacos Distro协议和JRaft协议的区别
| 维度 | Distro | JRaft |
|------|--------|-------|
| 设计目标 | AP（可用性优先） | CP（一致性优先） |
| 一致性 | 最终一致性 | 强一致性 |
| 适用实例 | 临时实例 | 持久化实例 |
| 同步方式 | 异步同步 | 同步写入多数节点 |
| 写可用性 | 任意节点可写 | 仅Leader可写 |
| 数据丢失风险 | 有（网络分区场景） | 无（多数确认才返回） |
| 选举机制 | 无选举 | 有Leader选举 |
| 性能 | 高（无一致性开销） | 较低（有同步开销） |
| 适用场景 | 微服务发现 | 配置中心/关键元数据 |

## 2.8 如何使用Nacos实现灰度发布？
**服务灰度（流量灰度）**：
1. 为灰度实例设置特殊的metadata，如`version=gray`或`env=canary`
2. 使用Nacos的实例元数据筛选功能
3. Consumer端自定义LoadBalancer规则，根据请求头中的`version`匹配灰度实例
4. 或通过Spring Cloud LoadBalancer自定义`ServiceInstanceListSupplier`

**配置灰度**：
1. 在Nacos控制台对目标配置执行"灰度发布"
2. 选择灰度目标（基于IP或自定义标签）
3. 只有匹配灰度条件的客户端才能获取到灰度版本配置
4. 灰度验证通过后可以"全量发布"覆盖原配置
5. 灰度期间正式客户端仍使用原配置，不影响线上

**最佳实践**：
- 结合网关的weight路由实现入口流量灰度
- 通过HTTP Header传递灰度标签，整条链路透传
- 全链路灰度需要所有中间件支持标签透传
