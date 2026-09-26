
# 1. 什么是 Dubbo

Dubbo 是阿里巴巴开源的一款**高性能、轻量级的 Java RPC 框架**，致力于提供高性能和透明化的远程服务调用方案，以及 SOA 服务治理方案。2018 年 Apache Dubbo 正式进入 Apache 基金会孵化，2019 年成为顶级项目。

三大核心能力：

- **远程通信**：基于 Netty 实现 NIO 通信，支持多种协议、多种序列化方式
- **集群容错**：提供服务发现、负载均衡、路由、容错等集群能力
- **服务治理**：提供可视化服务治理与运维（Admin）、动态配置、流量管理

# 2. 发展历程

```
Alibaba Dubbo 2.x (2011 开源)
  └─→ 2014-2017 停止维护（当当 dubbox 分支）
       └─→ Apache Dubbo 2.7.x (2019 重启，进入 Apache 孵化)
            └─→ Apache Dubbo 3.x (2021 发布，拥抱云原生)
                 ├─ 3.0: 应用级服务发现、Triple 协议、性能优化
                 ├─ 3.1: Mesh 支持、Proxyless、xDS 协议
                 └─ 3.2+: 原生 JDK 17-21 支持、虚拟线程、Spring Boot 3
```

| 版本 | 服务发现 | 协议 | Spring Boot | JDK |
|------|---------|------|-------------|-----|
| Dubbo 2.5-2.6 | 接口级 | Dubbo/REST | 1.x | 8 |
| Dubbo 2.7 | 接口级（兼容） | Dubbo/REST/gRPC | 2.x | 8 |
| Dubbo 3.0 | 应用级（默认） | Dubbo/Triple | 2.x | 8 |
| Dubbo 3.1-3.2 | 应用级 | Triple/Dubbo | 2.x/3.x | 17+ |
| Dubbo 3.3+ | 应用级 | Triple/Dubbo | 3.x | 17/21 |

# 3. Dubbo vs Spring Cloud

| 维度 | Dubbo | Spring Cloud (Feign + Gateway) |
|------|-------|-------------------------------|
| **通信模型** | RPC（远程过程调用） | HTTP/REST |
| **传输协议** | Dubbo/Triple（二进制） | HTTP 1.1/2 |
| **序列化** | Hessian2/Protobuf（二进制） | JSON（文本） |
| **性能** | 高（二进制协议，IO 线程模型优化） | 中（HTTP 开销大） |
| **服务治理** | 内置丰富治理能力 | 需借助 Sentinel/Hystrix |
| **语言支持** | Java 为主（多语言 SDK 发展中） | 多语言（HTTP 通用） |
| **生态集成** | Spring Cloud Alibaba | Spring Cloud 全家桶 |
| **适用场景** | 内部微服务间高性能调用 | 开放 API、跨语言、BFF |

**不是替代关系，而是互补**：在 Spring Cloud 体系内，Dubbo 可以作为 Feign 的替代方案进行服务间调用，也可以与 Feign 混合部署。

```
┌─────────────────────────────────────────────────┐
│                  Spring Cloud                     │
│  ┌──────────────┐   ┌──────────────┐             │
│  │   Dubbo RPC   │   │  Feign HTTP   │            │
│  │  (内部高性能)   │   │  (跨语言/BFF)  │            │
│  └──────┬───────┘   └──────┬───────┘             │
│         │                   │                    │
│  ┌──────┴───────────────────┴───────┐             │
│  │        Spring Cloud Alibaba       │            │
│  │  Nacos / Sentinel / Seata         │            │
│  └──────────────────────────────────┘             │
└─────────────────────────────────────────────────┘
```

# 4. 核心设计理念

1. **Microkernel + Plugin**：微内核 + 全插件化架构，几乎所有功能点都提供 SPI 扩展点
2. **Proxy-based RPC**：通过动态代理让远程调用像本地调用一样透明
3. **URL-driven**：用 URL（Dubbo URL，非 HTTP URL）作为配置总线贯穿整个调用链
4. **Layered Architecture**：清晰的分层结构，每层职责单一、可替换

# 5. 一句话架构

```
Provider                              Consumer
┌──────────┐                         ┌──────────┐
│  Service  │ ──── register ───────→ │  Proxy    │
│  Export   │                         │  (stub)   │
└─────┬─────┘                         └─────┬─────┘
      │                                     │
      │  ┌──────────────────────┐           │
      └─→│    Registry Center    │←─────────┘
         │  (Nacos/Zookeeper)   │
         └──────────────────────┘
                     ↓ lookup
         ┌──────────────────────┐
         │   Monitor (Admin)     │
         └──────────────────────┘
```

Provider 启动时将服务注册到注册中心，Consumer 从注册中心发现服务列表，通过负载均衡和路由策略选择一个 Provider 发起 RPC 调用，同时上报调用指标到监控中心。
