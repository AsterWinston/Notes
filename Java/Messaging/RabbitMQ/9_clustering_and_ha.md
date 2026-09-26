
> 版本说明：本文档基于 **RabbitMQ 3.13.x**，涵盖集群搭建、Quorum Queue、镜像队列（已不推荐）、Federation/Shovel 插件、高可用架构设计。

---

# 1. 集群概述

RabbitMQ 集群是**多个 Erlang 节点组成的逻辑整体**，共享用户、VHost、Exchange、Queue 元数据，但 Queue 内容默认只存在于声明它的节点上（镜像队列除外）。

## 1.1 集群特性

| 特性 | 说明 |
|------|------|
| **元数据共享** | 所有节点的 Exchange、Queue、Binding、VHost、用户信息一致 |
| **消息不共享** | 默认情况下 Queue 的内容只存在于声明该队列的节点 |
| **客户端透明** | 客户端连接任意节点都可以访问整个集群的资源 |
| **节点发现** | 通过 Erlang Cookie + 节点名称互相发现 |
| **默认端口** | 25672（集群通信）、4369（EPMD 发现服务） |

---

# 2. Docker Compose 搭建三节点集群

```yaml
version: "3.8"
services:
  rabbitmq1:
    image: rabbitmq:3.13-management
    hostname: rabbit1
    container_name: rabbitmq1
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin123
      RABBITMQ_ERLANG_COOKIE: SECRET_COOKIE_VALUE
    ports:
      - "5672:5672"
      - "15672:15672"
    volumes:
      - rabbitmq1_data:/var/lib/rabbitmq

  rabbitmq2:
    image: rabbitmq:3.13-management
    hostname: rabbit2
    container_name: rabbitmq2
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin123
      RABBITMQ_ERLANG_COOKIE: SECRET_COOKIE_VALUE
    ports:
      - "5673:5672"
      - "15673:15672"
    volumes:
      - rabbitmq2_data:/var/lib/rabbitmq

  rabbitmq3:
    image: rabbitmq:3.13-management
    hostname: rabbit3
    container_name: rabbitmq3
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin123
      RABBITMQ_ERLANG_COOKIE: SECRET_COOKIE_VALUE
    ports:
      - "5674:5672"
      - "15674:15672"
    volumes:
      - rabbitmq3_data:/var/lib/rabbitmq

volumes:
  rabbitmq1_data:
  rabbitmq2_data:
  rabbitmq3_data:
```

## 2.1 组建集群

```bash
# 进入 node2 和 node3，分别执行：
docker exec -it rabbitmq2 rabbitmqctl stop_app
docker exec -it rabbitmq2 rabbitmqctl reset
docker exec -it rabbitmq2 rabbitmqctl join_cluster rabbit@rabbit1
docker exec -it rabbitmq2 rabbitmqctl start_app

docker exec -it rabbitmq3 rabbitmqctl stop_app
docker exec -it rabbitmq3 rabbitmqctl reset
docker exec -it rabbitmq3 rabbitmqctl join_cluster rabbit@rabbit1
docker exec -it rabbitmq3 rabbitmqctl start_app

# 查看集群状态
docker exec -it rabbitmq1 rabbitmqctl cluster_status
```

---

# 3. 集群类型

## 3.1 三种集群模式

| 模式 | 元数据共享 | 消息共享 | 故障转移 | 适用版本 |
|------|:---:|:---:|------|----------|
| **普通集群** | 是 | 否（仅本地节点） | 手动处理 | 所有版本 |
| **镜像队列** | 是 | 是（异步/同步复制到所有镜像） | 自动（提升最旧的镜像） | 3.0 ~ 3.12（3.13 不推荐） |
| **Quorum Queue** | 是 | 是（Raft 共识复制到多数派） | 自动（Raft Leader 选举） | 3.8+（推荐） |

## 3.2 普通集群的消息路由

```
客户端连接 node2，发送消息到 queue_on_node1：

 ┌────────┐           ┌────────┐
 │ node1  │           │ node2  │
 │ queue1 │           │        │
 └───▲────┘           └───▲────┘
     │                    │
     │ 消息被路由到 node1  │ 客户端
     └────────────────────┘
```

客户端连接任意节点都能访问 `queue1`，但消息实际存储在 node1 上。如果 node1 宕机，`queue1` 不可用。

---

# 4. Quorum Queue 深入

## 4.1 架构

```
┌───────────────────────────────────────────┐
│           Quorum Queue (3 Nodes)           │
│                                           │
│  写入请求                                  │
│     ↓                                     │
│  ┌─────────┐  (Leader)                    │
│  │ Node 1  │  接收所有写入                 │
│  └────┬────┘                              │
│       │ Raft 日志复制                      │
│  ┌────▼────┐ ┌─────────┐                  │
│  │ Node 2  │ │ Node 3  │                  │
│  │Follower │ │Follower │                  │
│  └─────────┘ └─────────┘                  │
│       ↑           ↑                       │
│   多数派确认（≥2）后写入成功                 │
└───────────────────────────────────────────┘
```

## 4.2 策略声明

```bash
# 通过策略让所有以 "order." 开头的队列成为仲裁队列
rabbitmqctl set_policy quorum-policy "^order\." \
  '{"queue-type":"quorum"}' \
  --priority 10 --apply-to queues
```

```java
// Java 声明
Map<String, Object> args = new HashMap<>();
args.put("x-queue-type", "quorum");
channel.queueDeclare("order.quorum.queue", true, false, false, args);
```

## 4.3 Quorum Queue 的成员管理

```bash
# 查看队列状态（含 Raft 信息）
rabbitmq-queues quorum_status order.quorum.queue

# 添加成员（必须有奇数个成员）
rabbitmq-queues add_member order.quorum.queue rabbit@node3

# 删除成员
rabbitmq-queues delete_member order.quorum.queue rabbit@node3

# 重新平衡 Leader
rabbitmq-queues rebalance quorum
```

## 4.4 关键参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `x-quorum-initial-group-size` | 3 | 初始成员数 |
| `x-quorum-target-group-size` | 0 | 目标成员数（0=初始大小） |
| `x-dead-letter-exchange` | — | 死信交换机 |
| `x-max-length` | — | 最大消息数 |

---

# 5. 镜像队列（Mirror Queue — 不推荐）

> **注意**：镜像队列在 RabbitMQ 3.13 中被标记为不推荐（deprecated），将在未来版本中移除。推荐使用 Quorum Queue 替代。

## 5.1 策略配置

```bash
# 所有队列都镜像到所有节点（不推荐生产环境使用）
rabbitmqctl set_policy ha-all "^" '{"ha-mode":"all","ha-sync-mode":"automatic"}'

# 指定复制到固定数量的节点
rabbitmqctl set_policy ha-exactly "^ha\." \
  '{"ha-mode":"exactly","ha-params":2,"ha-sync-mode":"automatic"}'
```

## 5.2 Ha Mode 说明

| 模式 | 含义 | 适用场景 |
|------|------|----------|
| `ha-mode: all` | 镜像到所有集群节点 | 小集群（≤5 节点） |
| `ha-mode: exactly` | 镜像到固定数量节点（含 master） | 大集群，按需分配资源 |
| `ha-mode: nodes` | 镜像到指定节点列表 | 精确控制镜像位置 |
| `ha-sync-mode: automatic` | 新镜像加入时自动同步 | 推荐 |
| `ha-sync-mode: manual` | 手动触发同步 | 不推荐 |
| `ha-promote-on-shutdown: always` | Master 宕机时提升最旧镜像 | 提高可用性 |

---

# 6. Federation 插件（跨集群联合）

Federation 用于**跨数据中心、跨 WAN** 的 RabbitMQ 消息传递，不要求节点在同集群中。

```
┌──────────────────────┐         ┌──────────────────────┐
│  数据中心 A (上游)     │         │  数据中心 B (下游)     │
│                      │         │                      │
│  ┌────────────────┐  │  WAN    │  ┌────────────────┐  │
│  │ upstream.ex    │──┼────────┼─→│ federated.ex    │  │
│  │ upstream.queue │  │         │  │ federated.queue │  │
│  └────────────────┘  │         │  └────────────────┘  │
└──────────────────────┘         └──────────────────────┘
```

## 6.1 配置步骤

```bash
# 1. 启用插件
rabbitmq-plugins enable rabbitmq_federation
rabbitmq-plugins enable rabbitmq_federation_management

# 2. 定义上游（在下游 Broker 执行）
rabbitmqctl set_parameter federation-upstream my-upstream \
  '{"uri":"amqp://admin:admin123@upstream-host:5672","expires":3600000}'

# 3. 定义策略（在下游 Broker 执行）
rabbitmqctl set_policy federate-me "^federated\." \
  '{"federation-upstream":"my-upstream"}' \
  --priority 10 --apply-to exchanges
```

## 6.2 Federation vs Cluster

| 维度 | Cluster | Federation |
|------|---------|------------|
| 适用网络 | LAN（低延迟） | WAN（高延迟/不可靠） |
| 节点关系 | 紧密耦合（同 Erlang Cookie） | 松耦合（独立集群） |
| 带宽要求 | 高 | 低（断点续传） |
| 故障隔离 | 单集群故障影响全体 | 各集群独立恢复 |
| 复制方向 | 双向 | 单向（上游→下游） |
| 使用场景 | 单数据中心 | 多数据中心 / 跨区域 |

---

# 7. Shovel 插件（消息搬运）

Shovel 用于在 Broker/集群/队列之间**单向搬运消息**，比 Federation 更灵活（支持任意源和目标）。

```
源 Broker: source.queue ──→ Shovel ──→ 目标 Broker: destination.queue
```

## 7.1 配置

```bash
# 启用插件
rabbitmq-plugins enable rabbitmq_shovel
rabbitmq-plugins enable rabbitmq_shovel_management

# 定义 Shovel
rabbitmqctl set_parameter shovel my-shovel \
  '{
    "src-protocol": "amqp091",
    "src-uri": "amqp://admin:admin123@source-host:5672",
    "src-queue": "source.queue",
    "dest-protocol": "amqp091",
    "dest-uri": "amqp://admin:admin123@dest-host:5672",
    "dest-queue": "destination.queue",
    "ack-mode": "on-confirm",
    "reconnect-delay": 5
  }'
```

## 7.2 Shovel vs Federation

| 维度 | Shovel | Federation |
|------|--------|------------|
| 粒度 | Queue 级别 | Exchange 级别 |
| 方向 | 单向搬运 | 单向同步 |
| 转发模式 | 订阅/消费 | 拉取 |
| 灵活性 | 高（可跨协议） | 中 |
| 管理复杂度 | 简单 | 中等 |

---

# 8. 高可用架构设计

## 8.1 典型三层架构

```
                    ┌──────────────┐
                    │   Load       │
                    │   Balancer   │  (HAProxy / Nginx / LVS)
                    └──┬───┬───┬──┘
                       │   │   │
              ┌────────▼───▼───▼────────┐
              │    RabbitMQ Cluster     │
              │  ┌─────┐┌─────┐┌─────┐ │
              │  │Node1││Node2││Node3│ │
              │  │QC_1 ││QC_2 ││QC_3 │ │  Quorum Queues
              │  └─────┘└─────┘└─────┘ │
              └─────────────────────────┘
                         │
              ┌──────────▼──────────┐
              │   Application       │
              │   (Spring AMQP)     │
              │   + 重试 / 断路     │
              └─────────────────────┘
```

## 8.2 客户端连接配置

```yaml
spring:
  rabbitmq:
    addresses: rabbit1:5672,rabbit2:5672,rabbit3:5672
    username: admin
    password: admin123
    virtual-host: /
    connection-timeout: 30000
    requested-heartbeat: 60
    cache:
      connection:
        mode: CONNECTION
      channel:
        size: 25
```

---

# 9. 故障场景与处理

## 9.1 常见故障处理

| 故障场景 | 影响 | 处理方案 |
|----------|------|----------|
| **单节点宕机（普通队列）** | 该节点的队列不可用 | 客户端重连到其他节点，队列丢失 |
| **单节点宕机（Quorum Queue）** | 自动 Leader 选举 | 无损（有足够副本），自动切换 |
| **网络分区** | 集群分裂，脑裂 | `cluster_partition_handling=autoheal` |
| **磁盘满** | 阻止所有发布 | 磁盘告警 → 扩容 → 清理 |
| **内存超限** | 阻止所有发布 | 触发流控，扩容/清理 |

## 9.2 网络分区处理策略

```ini
# rabbitmq.conf
cluster_partition_handling = autoheal      # 自动选择一方恢复
# cluster_partition_handling = pause_minority  # 少数方暂停
# cluster_partition_handling = ignore          # 不处理（不推荐）
```

| 策略 | 行为 | 推荐度 |
|------|------|:---:|
| `autoheal` | 自动选择节点较多的一方恢复，少的一方重启 | ⭐⭐⭐ |
| `pause_minority` | 少数方暂停服务，多数方继续 | ⭐⭐⭐⭐⭐ |
| `pause_if_all_down` | 无法连接其他节点时暂停 | ⭐⭐⭐ |
| `ignore` | 不处理，可能导致脑裂 | ❌ |

---

# 10. 索引 — 队列类型选择

```
┌─────────────────────────────────────────────────────┐
│                  队列类型选择决策树                    │
│                                                     │
│  需要消息可重复消费（类似 Kafka Offset）？              │
│   └─ 是 → Stream Queue                                │
│   └─ 否 → 继续                                        │
│                                                     │
│  需要强一致性和高可用？                                 │
│   └─ 是 → Quorum Queue ⭐（Raft 共识）                 │
│   └─ 否 → 继续                                        │
│                                                     │
│  消息量大但可容忍消息丢失（纯缓存）？                     │
│   └─ 是 → Classic Queue（不持久化）                     │
│   └─ 否 → 继续                                        │
│                                                     │
│  消息堆积大、内存紧张？                                 │
│   └─ 是 → Lazy Queue / Classic+Lazy                    │
│   └─ 否 → Classic Queue                               │
└─────────────────────────────────────────────────────┘
```
