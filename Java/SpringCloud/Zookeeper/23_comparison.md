
# 1. 对比概览

ZooKeeper 不是唯一的分布式协调服务。以下从多个维度对比主流产品：

| 产品 | 开发方 | 语言 | 协议 | CAP | 定位 |
|------|--------|------|------|-----|------|
| **ZooKeeper** | Apache | Java | ZAB | CP | 分布式协调服务 |
| **etcd** | CNCF | Go | Raft | CP | 分布式 K-V 存储 |
| **Consul** | HashiCorp | Go | Raft | CP(默认) | 服务网格基础设施 |
| **Eureka** | Netflix | Java | 对等复制 | AP | 服务注册发现 |
| **Nacos** | Alibaba | Java | Distro/Raft | CP+AP | 动态服务发现与配置管理 |

# 2. ZooKeeper vs etcd

## 2.1 架构对比

```text
ZooKeeper                               etcd
+---------------------------+           +---------------------------+
| 类文件系统的层次 ZNode      |           | 扁平的 Key-Value 存储       |
| /config/db/url = "xxx"     |           | /config/db/url = "xxx"     |
+---------------------------+           +---------------------------+
| ZAB 协议（类 Paxos）        |           | Raft 协议                   |
|                             |           |                             |
| Leader 广播 Proposal        |           | Leader 复制日志条目         |
| Follower ACK                |           | Follower 确认                |
+---------------------------+           +---------------------------+
| Java 实现                   |           | Go 实现                     |
| 客户端：NIO 长连接          |           | 客户端：HTTP/gRPC 长连接    |
+---------------------------+           +---------------------------+
```

## 2.2 特性对比

| 维度 | ZooKeeper | etcd |
|------|-----------|------|
| **Watch 机制** | 一次性触发 | 长连接流式推送（Watch API） |
| **数据模型** | 层次树形（类文件系统） | 扁平的 K-V + 不透明数据 |
| **API 风格** | 原生 Java / Curator | HTTP REST + gRPC |
| **事务支持** | multi 批量操作 | 支持事务（if/then/else） |
| **Lease** | Session 超时 | Lease（租约） |
| **TTL** | 3.5+ 支持持久节点 TTL | 原生支持 Key TTL |
| **服务发现** | 需二次开发（Dubbo/Curator） | 需二次开发 |
| **多语言客户端** | Java 为主，其他语言有社区客户端 | Go 为主，多语言官方客户端 |
| **运维复杂度** | 中（需管理 JVM） | 低（Go 单二进制） |
| **内存占用** | 高（JVM 开销） | 低（Go 运行时） |
| **生态** | Hadoop/Kafka/Dubbo/HBase 等 | Kubernetes/容器生态 |
| **GUI** | 无内置（社区项目） | etcd-manager 等 |

**选型建议**：

- 已有 Java 技术栈，需要深度集成（如 Dubbo/Kafka） → **ZooKeeper**
- Kubernetes/容器化环境，需要轻量级 K-V → **etcd**
- 需要原生 TTL 和简单部署 → **etcd**

# 3. ZooKeeper vs Consul

## 3.1 定位差异

```text
ZooKeeper：分布式协调工具包          Consul：服务网格一体化平台
+---------------------------+       +---------------------------+
| 需要自己构建上面层能力       |       | 开箱即用的服务网格能力     |
|                             |       |                             |
| 服务发现：需二次开发        |       | 服务发现：原生支持          |
| 健康检查：仅 Session 心跳    |       | 健康检查：HTTP/TCP/Script  |
| 负载均衡：需客户端实现      |       | DNS 负载均衡：原生支持      |
| 配置管理：需二次开发        |       | K-V 存储：原生支持          |
| 多数据中心：不支持          |       | 多数据中心：原生支持        |
+---------------------------+       +---------------------------+
```

## 3.2 特性对比

| 维度 | ZooKeeper | Consul |
|------|-----------|--------|
| **CAP 模型** | CP | CP（默认，可通过配置切换为 AP） |
| **服务发现** | 需要二次开发 | 原生支持，内置 DNS/HTTP API |
| **健康检查** | Session 心跳 | 丰富：HTTP/TCP/gRPC/Script/OS/Time-to-Live |
| **K-V 存储** | 层次 ZNode | 扁平 K-V |
| **多数据中心** | 不支持 | 原生 WAN 联邦 |
| **Service Mesh** | 不支持 | 内置 Connect（Sidecar Proxy） |
| **Web UI** | 无（需第三方） | 内置精美 Web UI |
| **ACL** | 5 种权限（cdrwa） | 基于 Token 的 ACL 策略 |
| **一致性** | ZAB（强一致） | Raft（CP 时强一致） |
| **部署** | Java 应用 | Go 单二进制 |
| **Spring Cloud** | 支持 | 支持 |

**选型建议**：

- 需要一站式服务网格解决方案 → **Consul**
- 已有成熟的服务发现框架（如 Dubbo），只需要底层协调 → **ZooKeeper**
- 需要多数据中心支持 → **Consul**
- 需要 Web UI 和开箱即用的监控 → **Consul**

# 4. ZooKeeper vs Eureka

## 4.1 CAP 模型差异

这是两者最大的区别：

```text
ZooKeeper (CP)                          Eureka (AP)
+---------------------------+           +---------------------------+
| 强一致性                    |           | 最终一致性                  |
|                             |           |                             |
| Leader 选举期间集群不可写    |           | Eureka Server 不可用时      |
|（牺牲可用性）               |           | 仍可注册和查询               |
|                             |           |（牺牲一致性）               |
|                             |           |                             |
| 客户端故障检测：Session 超时 |           | 客户端故障检测：心跳续约     |
| + 临时节点自动删除          |           | + 自我保护机制               |
+---------------------------+           +---------------------------+
```

## 4.2 特性对比

| 维度 | ZooKeeper | Eureka |
|------|-----------|--------|
| **CAP 模型** | CP（强一致） | AP（高可用） |
| **自我保护** | 无 | 有（网络分区时保留过期实例） |
| **服务注册** | 临时节点 | 心跳续约 |
| **服务发现** | Watch 通知 | Delta 更新 + 定期拉取 |
| **健康检查** | Session 心跳 | 客户端心跳（默认 30s） |
| **不可用时的行为** | 写操作阻塞 | 仍可读写（可能读脏数据） |
| **配置管理** | 支持（需二次开发） | 不支持 |
| **集群复制** | ZAB 主从 | 对等复制 |
| **Spring Cloud 集成** | 支持 | 原生（但已停维） |
| **维护状态** | Apache 活跃维护 | 已停维（进入维护模式） |
| **新项目推荐** | 推荐 | 不推荐 |

**选型建议**：

- 历史项目使用 Eureka → 考虑迁移到 Nacos 或 ZK
- 新项目 → **ZooKeeper** 或 **Nacos**（不再推荐 Eureka）

# 5. ZooKeeper vs Nacos

## 5.1 定位差异

```text
ZooKeeper：通用分布式协调服务        Nacos：一站式微服务平台
+---------------------------+       +---------------------------+
| 底层原语级                  |       | 应用级框架集成                |
|                             |       |                             |
| 提供：                      |       | 提供：                      |
|  · 分布式锁                 |       |  · 服务发现（+ 健康检查）    |
|  · 选主                     |       |  · 配置管理                  |
|  · 队列                     |       |  · DNS 服务                  |
|  · ...                      |       |  · 动态路由                  |
|                             |       |  · 流量管理                  |
| 需要自己组合使用             |       | 开箱即用                     |
+---------------------------+       +---------------------------+
```

## 5.2 特性对比

| 维度 | ZooKeeper | Nacos |
|------|-----------|-------|
| **CAP 模型** | CP | CP + AP（混合架构，可切换） |
| **服务发现** | 需二次开发 | 原生支持 |
| **配置管理** | 需二次开发 | 原生支持，多格式 |
| **健康检查** | Session 心跳 | TCP/HTTP/MySQL（主动探测） |
| **动态路由** | 不支持 | 支持 |
| **DNS-F** | 不支持 | 支持（DNS-based 服务发现） |
| **Web 控制台** | 无内置 | 内置功能丰富的控制台 |
| **集群** | ZAB 协议集群 | Distro（AP）+ Raft（CP） |
| **Spring Cloud** | spring-cloud-zookeeper | spring-cloud-alibaba-nacos |
| **多数据中心** | 不支持 | 支持 |
| **灰度发布** | 不支持 | 支持 |
| **生态契合度** | 与 Dubbo/Kafka 完美配合 | Spring Cloud Alibaba 首选 |

**选型建议**：

- Spring Cloud Alibaba 微服务体系 → **Nacos**（首选）
- 纯 Dubbo 体系 → **ZooKeeper**（经典）
- 需要同时具备服务发现和配置管理 → **Nacos**（开箱即用）
- 需要分布式锁/选主等底层协调能力为主 → **ZooKeeper**

# 6. 综合选型决策树

```text
                              开始
                               |
                  需要一站式微服务解决方案？
                    /                 \
                  是                   否
                   |                    |
         Spring Cloud Alibaba?    需要底层协调原语？
          /          \             /           \
        是           否          是              否
         |            |          |               |
       Nacos      Consul     ZooKeeper      需要简单 K-V？
                                            /           \
                                          是             否
                                          |              |
                                        etcd          重新评估需求

特殊情况：
  · 已有 Kafka/Hadoop/HBase → ZooKeeper
  · Kubernetes 环境 → etcd
  · 多数据中心 → Consul 或 Nacos
  · 纯服务发现且对一致性要求不高 → Eureka (仅维护存量)
```

# 7. 迁移建议

```text
常见迁移路径：

1. Eureka → Nacos
   最平滑的替代方案，Spring Cloud 生态兼容性好

2. Eureka → ZooKeeper
   切换到 CP 模型，适合对一致性要求更高的场景

3. ZooKeeper → Nacos
   想要更丰富的平台能力和 Web 管理界面

4. etcd → ZooKeeper
   从 Go/K8s 生态迁移到 Java 生态

迁移注意事项：
  · 评估 CAP 模型变更带来的影响
  · API 和数据模型差异需要代码适配
  · 灰度迁移，逐步切换流量
  · 保留回滚能力
```
