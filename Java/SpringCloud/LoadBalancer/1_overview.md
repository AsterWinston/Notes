
# 1. 概述

Spring Cloud LoadBalancer 是 Spring Cloud 官方提供的**客户端负载均衡器**，用以替代进入维护模式的 Netflix Ribbon。它从 Spring Cloud 2020.0（Ilford）版本开始成为默认的负载均衡实现。

```
┌──────────────────────────────────────────────────────────┐
│                      服务消费者                           │
│  ┌────────────────────────────────────────────────────┐  │
│  │            Spring Cloud LoadBalancer               │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐ │  │
│  │  │ Instance │  │  Load    │  │   Http Client    │ │  │
│  │  │ Supplier │→│ Balancer │→│ (RestTemplate/    │ │  │
│  │  │          │  │          │  │  WebClient/Feign) │ │  │
│  │  └──────────┘  └──────────┘  └──────────────────┘ │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
   ┌──────────┐        ┌──────────┐        ┌──────────┐
   │ 实例 A    │        │ 实例 B    │        │ 实例 C    │
   └──────────┘        └──────────┘        └──────────┘
```

# 2. 为什么要用 LoadBalancer

## 2.1 Ribbon 的落幕

Netflix Ribbon 曾是 Spring Cloud 生态中客户端负载均衡的事实标准，但存在以下问题：

| 问题 | 说明 |
|------|------|
| **停止维护** | Netflix 于 2018 年宣布 Ribbon 进入维护模式，不再添加新功能 |
| **阻塞式架构** | 基于 RestTemplate 的阻塞 I/O 模型，不支持响应式编程 |
| **代码耦合** | Ribbon 内部模块耦合度高，扩展复杂 |
| **API 陈旧** | 部分 API 设计过时，不符合现代 Spring 风格 |

## 2.2 LoadBalancer 的优势

| 维度 | Spring Cloud LoadBalancer | Netflix Ribbon |
|------|--------------------------|----------------|
| **维护状态** | Spring 官方活跃维护 | 停止维护 |
| **响应式支持** | ✓ 原生支持 WebFlux / Reactor | ✗ 仅支持阻塞 |
| **架构风格** | 响应式 + 阻塞桥接 | 纯阻塞 |
| **API 设计** | 函数式、接口精简 | 类层次深、API 复杂 |
| **Spring 集成** | 原生 Spring Boot AutoConfiguration | 基于 `@RibbonClient` |
| **配置风格** | `spring.cloud.loadbalancer.*` | `ribbon.*`（已弃用） |
| **扩展性** | ServiceInstanceListSupplier 链式组合 | IRule / IPing / ServerListFilter |
| **体积** | 轻量（核心模块 ~200KB） | 较重 |

# 3. 核心设计理念

1. **响应式优先**：核心 API 基于 Project Reactor，天然支持异步非阻塞，同时提供 `LoadBalancerClient` 阻塞桥接接口。

2. **供应器模式（Supplier）**：服务实例列表通过 `ServiceInstanceListSupplier` 函数式接口获取，支持链式组合装饰。

3. **负载均衡器抽象**：核心接口 `ReactorLoadBalancer<T>` 将负载均衡算法与实例获取解耦。

4. **客户端工厂模式**：`LoadBalancerClientFactory` 为每个服务（serviceId）创建独立的 Spring 子容器，实现**服务级别**的配置隔离。

5. **自动装配**：通过 `spring-cloud-starter-loadbalancer` 自动引入，整合 RestTemplate / WebClient / Feign。

# 4. 架构概览

```
                        ┌──────────────────────┐
                        │   Service Discovery  │  ← Nacos / Eureka / Consul
                        │   (注册中心)          │
                        └──────────┬───────────┘
                                   │ 服务发现
                                   ▼
┌─────────────────────────────────────────────────────────┐
│               LoadBalancerClientFactory                  │
│  ┌──────────────────┐  ┌───────────────────┐            │
│  │ user-service     │  │ order-service     │  ...       │
│  │  └─ 独立子容器   │  │  └─ 独立子容器    │            │
│  │     ├─ Supplier  │  │     ├─ Supplier   │            │
│  │     └─ Balancer  │  │     └─ Balancer   │            │
│  └──────────────────┘  └───────────────────┘            │
└─────────────────────────────────────────────────────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    ▼              ▼              ▼
          ┌──────────────┐ ┌────────────┐ ┌──────────────┐
          │ RestTemplate  │ │  WebClient │ │    Feign     │
          │ (阻塞)        │ │  (响应式)  │ │   (声明式)   │
          └──────────────┘ └────────────┘ └──────────────┘
```

# 5. LB 协议与请求流程

Spring Cloud LoadBalancer 使用 `lb://service-id` 协议标识需要负载均衡的请求：

```
请求: http://order-service/api/orders
  │
  ▼
① 提取 serviceId = "order-service"
  │
  ▼
② ServiceInstanceListSupplier 从注册中心获取实例列表
  │  [192.168.1.10:8080, 192.168.1.11:8080, 192.168.1.12:8080]
  ▼
③ ReactorLoadBalancer.choose() 执行负载均衡算法
  │  → 192.168.1.11:8080
  ▼
④ 替换 URL → http://192.168.1.11:8080/api/orders
  │
  ▼
⑤ HTTP Client 发出实际请求
```

# 6. 版本兼容矩阵

| Spring Cloud | Spring Boot | LoadBalancer 版本 | Ribbon 状态 |
|-------------|-------------|-------------------|------------|
| Hoxton.SR1+ | 2.2.x | 2.2.x（可选） | 可选使用 |
| 2020.0.x (Ilford) | 2.4.x / 2.5.x | 3.0.x | 默认移除 |
| 2021.0.x (Jubilee) | 2.6.x / 2.7.x | 3.1.x | 不复存在 |
| 2022.0.x (Kilburn) | 3.0.x | 4.0.x | — |
| 2023.0.x (Leyton) | 3.1.x / 3.2.x | 4.1.x | — |

# 7. 使用场景

| 场景 | 适用 | 说明 |
|------|------|------|
| Spring Cloud 微服务调用 | ✓ | 与 RestTemplate / WebClient / Feign 无缝集成 |
| Spring Cloud Gateway | ✓ | 通过 `lb://` 协议前缀自动集成 |
| 响应式/WebFlux 项目 | ✓ | 核心基于 Reactor，天然适配 |
| 传统 Servlet 项目 | ✓ | 通过 `LoadBalancerClient` 阻塞桥接 |
| 非 Spring 项目 | ✗ | 强依赖 Spring 容器 |
| 服务端负载均衡 | ✗ | 仅客户端负载均衡 |
