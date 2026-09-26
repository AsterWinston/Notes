
# 1. 对比概览

| 特性 | Nacos | Eureka | Consul | ZooKeeper | Apollo |
|------|-------|--------|--------|-----------|--------|
| 开发厂商 | 阿里巴巴 | Netflix (停更) | HashiCorp | Apache | 携程 |
| 语言/框架 | Java | Java | Go | Java | Java |
| CAP 模型 | AP/CP 可切换 | AP | CP | CP | AP |
| 服务注册/发现 | ✅ | ✅ | ✅ | ✅ | ❌ |
| 配置管理 | ✅ | ❌ | ✅ | ❌ | ✅ |
| 健康检查 | TCP/HTTP/MySQL | HTTP | TCP/HTTP/gRPC/Script | TCP | - |
| 一致性协议 | Distro + Raft | 无（最终一致） | Raft | ZAB | 无 |
| 多数据中心 | ✅ | ✅（2.x） | ✅ | ❌ | ❌ |
| DNS 协议支持 | ✅ | ❌ | ✅ | ❌ | ❌ |
| 控制台 | ✅丰富 | ✅基础 | ✅基础 | ❌ | ✅丰富 |
| Spring Cloud 集成 | ✅（Alibaba） | ✅（Netflix） | ✅ | ✅ | ✅ |
| K8s 集成 | ✅（通用） | ✅（K8s首个） | ✅（原生） | ❌ | ❌ |
| 多语言 SDK | Java/Go/Python/C++/C# | Java (已停止) | 全语言 | Java/C/Python | Java/.NET |
| 开源协议 | Apache 2.0 | Apache 2.0 | MPL 2.0 | Apache 2.0 | Apache 2.0 |
| 社区活跃度 | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |

# 2. Nacos vs Eureka

## 2.1 Eureka 概述

Eureka 是 Netflix 开源的基于 REST 的服务注册与发现组件，Spring Cloud Netflix 的核心组件（已进入维护模式，不再开发新功能）。

## 2.2 架构对比

**Eureka 架构**：
```
Eureka Server (集群)
  ├── 节点间采用 P2P 复制
  ├── 自我保护模式（默认开启）
  └── 三级缓存（ReadOnly/ReadWrite/Registry）

Eureka Client
  ├── 30s 拉取一次全量注册表
  ├── 30s 发送一次心跳
  └── 本地保存全量注册表
```

**Nacos 架构**：
```
Nacos Server (集群)
  ├── 节点间 Distro (AP) / Raft (CP)
  ├── 保护阈值机制
  └── 内存注册表 + MySQL 持久化

Nacos Client
  ├── gRPC 实时推送 + 3s 定时兜底
  ├── 5s 心跳 (临时实例)
  └── 本地磁盘缓存
```

## 2.3 核心差异

| 维度 | Nacos | Eureka |
|------|-------|--------|
| **配置中心** | 内置 | 需配合 Spring Cloud Config |
| **推送机制** | gRPC 实时推送 | 客户端定时拉取 (30s) |
| **时效性** | 秒级 (gRPC) | 分钟级 (30s 拉取) |
| **健康检查** | TCP/HTTP/MySQL (临时+持久) | HTTP Health Check |
| **自我保护** | 保护阈值 (可配置) | 自我保护模式 (默认开启) |
| **多注册中心** | 支持多注册中心 | 不支持 |
| **负载均衡** | 权重 + 集群优先 | Ribbon（已淘汰） |
| **CAP** | AP/CP 可切换 | AP |
| **维护状态** | 活跃开发 | 进入维护模式 |

## 2.4 迁移建议

从 Eureka 迁移到 Nacos 的关键步骤：

```yaml
# 替换依赖
# 移除:
# spring-cloud-starter-netflix-eureka-server / client
# 添加:
# spring-cloud-starter-alibaba-nacos-discovery

# 替换配置
# Eureka:
eureka.client.service-url.defaultZone=http://127.0.0.1:8761/eureka
# Nacos:
spring.cloud.nacos.discovery.server-addr=127.0.0.1:8848

# 代码改动
# @EnableEurekaClient → @EnableDiscoveryClient（兼容）
```

# 3. Nacos vs Consul

## 3.1 Consul 概述

Consul 是 HashiCorp 开发的基于 Go 语言的服务网格解决方案，提供服务发现、KV 存储、健康检查、多数据中心支持。

## 3.2 核心差异

| 维度 | Nacos | Consul |
|------|-------|--------|
| **一致性** | AP/CP 可切换 | 仅 CP (Raft) |
| **配置 KV** | 三级模型（NS/Group/DataId） | 扁平 KV + 多级 Key |
| **配置版本** | 历史版本管理 | 无原生版本 |
| **灰度发布** | ✅ 原生支持 | ❌ 需自行实现 |
| **性能** | 高（内存 + 事件驱动） | 中等（Raft 写入瓶颈） |
| **多数据中心** | 支持（有限） | ✅ 原生支持（WAN Federation） |
| **健康检查** | TCP/HTTP/MySQL | TCP/HTTP/gRPC/Script/TTL |
| **一致性协议** | Distro + Raft (Java) | Raft (Golang) |
| **部署复杂度** | 需要 MySQL | 零依赖（内置存储） |
| **控制台** | ✅ 功能全面 | ✅ 基础 |

## 3.3 选型建议

- **需要多数据中心** → Consul（原生 WAN Federation）
- **需要配置灰度发布** → Nacos
- **最小化依赖** → Consul（零外部依赖）
- **只用作服务发现 + 配置中心** → Nacos（配置管理功能更强）

# 4. Nacos vs ZooKeeper

## 4.1 ZooKeeper 概述

ZooKeeper 是 Apache 的分布式协调服务，常被用作注册中心（Dubbo 默认），但设计初衷并非服务发现。

## 4.2 核心差异

| 维度 | Nacos | ZooKeeper |
|------|-------|-----------|
| **设计目标** | 服务发现 + 配置管理 | 分布式协调（锁、选举、配置） |
| **CAP** | AP/CP 可切换 | 仅 CP (ZAB) |
| **配置管理** | ✅ 丰富 | ❌ 需自行实现 |
| **数据模型** | 服务/配置专用模型 | 树形节点（ZNode） |
| **注册方式** | API / SDK / DNS | 临时节点 (EHPEMERAL) |
| **健康检查** | 客户端心跳 / 服务端探测 | Session 超时 |
| **控制台** | ✅ 丰富 | ❌ 无原生 |
| **服务发现性能** | 高（实时推送） | 中（Watch 机制） |
| **K8s 集成** | ✅ | ❌ |
| **学习曲线** | 中等 | 较高 |

## 4.3 ZooKeeper 作为注册中心的痛点

1. **ZNode 容量限制**：每个 ZNode 建议 < 1MB，大量服务实例会导致问题
2. **Session 超时问题**：GC 导致的假死可能触发误剔除
3. **只有 CP**：网络分区时少数派不可用
4. **Watch 一次性**：每次变更后需要重新注册 Watch
5. **无控制台**：运维管理困难

# 5. Nacos vs Apollo

## 5.1 Apollo 概述

Apollo 是携程框架部门开源的配置管理中心，专注配置管理，在配置治理方面功能非常全面。

## 5.2 核心差异

| 维度 | Nacos Config | Apollo |
|------|-------------|--------|
| **定位** | 配置中心 + 注册中心 | 纯配置中心 |
| **部署复杂度** | 中等（MySQL + 集群） | 较高（MetaServer + ConfigService + AdminService + Portal） |
| **配置模型** | Namespace / Group / DataId | AppId / Cluster / Namespace |
| **环境管理** | Namespace 隔离 | 独立部署 + Cluster 隔离 |
| **灰度发布** | ✅ Beta IP | ✅ 完整的灰度策略（IP/比例/标签） |
| **配置版本** | 历史版本列表 | ✅ 完善的版本比对和回滚 |
| **配置审核** | ❌ | ✅ 发布审核机制 |
| **权限管理** | RBAC | ✅ 完善的权限体系 |
| **开放平台** | REST API | ✅ 完整的 OpenAPI |
| **客户端** | Nacos SDK | ✅ Apollo Client (+Meta Server 高可用) |
| **服务发现** | ✅ 内置 | ❌ 需要配合 Eureka/Nacos |

## 5.3 选型建议

- **同时需要服务注册发现 + 配置中心** → Nacos
- **只需要配置中心且对配置治理要求极高** → Apollo
- **简单场景，希望最小化运维** → Nacos
- **已有完善的配置管理流程（审批、审计）** → Apollo

# 6. 综合选型矩阵

## 6.1 按场景选型

| 场景 | 推荐方案 | 理由 |
|------|---------|------|
| 中小型微服务（< 100 服务） | Nacos | 同时解决注册+配置，运维简单 |
| 大型微服务（> 500 服务） | Nacos + Apollo | Apollo 做配置管理，Nacos 做注册中心 |
| Dubbo 生态 | Nacos | 原生支持，推荐替代 ZooKeeper |
| 云原生 / K8s | Nacos + Istio | Nacos 提供服务元数据，Istio 做流量治理 |
| 多数据中心 | Consul + Apollo | Consul WAN Federation + Apollo 配置管理 |
| 全 Java 生态 | Nacos | Spring Cloud Alibaba 全家桶无缝集成 |
| 多语言异构 | Consul | Go 开发，多语言 SDK 支持最好 |

## 6.2 综合评分

| 维度权重 | Nacos | Eureka | Consul | ZooKeeper | Apollo |
|---------|-------|--------|--------|-----------|--------|
| 服务发现 (30%) | 9 | 7 | 8 | 6 | 0 |
| 配置管理 (30%) | 8 | 2 | 6 | 3 | 9 |
| 性能 (15%) | 9 | 7 | 7 | 6 | 8 |
| 运维复杂度 (10%) | 7 | 8 | 7 | 7 | 6 |
| 生态集成 (10%) | 9 | 6 | 8 | 7 | 7 |
| 社区活跃度 (5%) | 9 | 2 | 8 | 8 | 7 |
| **综合得分** | **8.5** | **5.6** | **7.3** | **5.9** | **6.4** |

## 6.3 结论

**Nacos 是当前微服务注册中心和配置中心的一站式最佳选择**：

1. **功能最全面**：同时覆盖服务注册发现 + 配置管理 + DNS
2. **性能最优**：gRPC 实时推送 + 内存注册表 + 异步同步
3. **架构最灵活**：AP/CP 可切换，适应不同场景
4. **生态最完善**：Spring Cloud Alibaba 全家桶、Dubbo、Gateway 深度集成
5. **社区最活跃**：阿里巴巴持续投入，Apache 顶级项目
