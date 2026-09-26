
> 版本说明：本文档基于 **RabbitMQ 3.13.x**，分析其 Erlang/OTP 源码架构、消息流转全链路、存储引擎、流量控制、网络分区处理等核心原理。

---

# 1. 整体系统架构

RabbitMQ 运行在 **Erlang/OTP** 虚拟机上，充分利用 OTP 的 Actor 并发模型（进程间消息传递）、监督树（Supervisor Tree）、热代码升级等能力。

## 1.1 Erlang/OTP 进程模型

```
┌─────────────────────────────────────────────────────────┐
│                     BEAM VM (Erlang)                     │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ Scheduler│  │ Scheduler│  │ Scheduler│  ... (N个)    │
│  │   #1     │  │   #2     │  │   #3     │              │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘              │
│       │              │              │                    │
│       ▼              ▼              ▼                    │
│  ┌─────────────────────────────────────────┐            │
│  │          Erlang Processes Pool          │            │
│  │  (轻量级线程, ~2KB 创建, μs 级切换)      │            │
│  └─────────────────────────────────────────┘            │
└─────────────────────────────────────────────────────────┘
```

| 特性 | 说明 |
|------|------|
| **Actor 模型** | 每个 Erlang 进程独立运行，通过消息传递通信（无共享内存） |
| **监督树** | 父子进程监控，子进程崩溃自动重启（Let it Crash 哲学） |
| **热升级** | 不停止服务即可升级模块代码 |
| **轻量级并发** | 创建百万级 Erlang 进程无压力（每个 ~2KB 内存） |

---

# 2. RabbitMQ 核心模块

## 2.1 模块架构

```
rabbitmq-server (Erlang/OTP)
│
├── rabbit_common        # 核心数据结构与通用逻辑（AMQP 帧编解码、类型定义）
├── rabbit               # Broker 主模块
│   ├── rabbit_reader    # AMQP 连接读写（每个 Connection 一个 reader 进程）
│   ├── rabbit_channel   # 信道进程（每个 Channel 一个 Erlang 进程）
│   ├── rabbit_amqqueue  # 队列进程（每个 Queue 一个 Erlang 进程）
│   ├── rabbit_exchange  # 交换机路由逻辑
│   ├── rabbit_binding   # 绑定管理
│   ├── rabbit_mgmt      # Management Plugin 内核
│   ├── rabbit_mnesia    # Mnesia 元数据存储
│   ├── rabbit_khepri    # Khepri 元数据存储（3.13+）
│   ├── rabbit_mirror_queue_master / slave  # 镜像队列（已弃用）
│   ├── rabbit_fifo      # Quorum Queue Raft 实现
│   └── rabbit_stream_queue  # Stream Queue 实现
│
├── rabbitmq_prelaunch   # 启动引导
└── plugins              # 插件系统
```

## 2.2 源码关键模块说明

| 模块文件 | 功能 | 关键进程数 |
|----------|------|:---:|
| `rabbit_reader.erl` | TCP 连接管理，AMQP 帧解析与编解码 | 1/Connection |
| `rabbit_channel.erl` | 信道逻辑：发布、消费、ACK、QoS、事务等 | 1/Channel |
| `rabbit_amqqueue.erl` | 队列核心：入队、出队、持久化、TTL、DLX | 1/Queue |
| `rabbit_exchange.erl` | 交换机的路由逻辑分发 | N/A（纯逻辑） |
| `rabbit_variable_queue.erl` | Classic Queue 内部实现（索引、消息存储） | — |
| `rabbit_fifo.erl` | Quorum Queue 实现（Raft 日志复制状态机） | — |
| `rabbit_msg_store.erl` | 消息持久化存储引擎 | 1/Broker |

---

# 3. 消息流转全链路

## 3.1 从 Publisher 发布到消息落盘

```
Publisher                              RabbitMQ Broker
   │                                        │
   │──── TCP Connect ──────────────────────→│ rabbit_reader 创建
   │                                        │
   │──── AMQP Connection.Open ────────────→│ 连接认证、VHost 验证
   │                                        │
   │──── AMQP Channel.Open ───────────────→│ rabbit_channel 创建
   │                                        │
   │──── AMQP Exchange.Declare ───────────→│ 元数据写入 Khepri/Mnesia
   │──── AMQP Queue.Declare ──────────────→│ 元数据写入 Khepri/Mnesia
   │──── AMQP Queue.Bind ─────────────────→│ 绑定关系写入 Khepri/Mnesia
   │                                        │
   │──── AMQP Basic.Publish ──────────────→│
   │    [Exchange + RoutingKey + Body]      │
   │                                        ├─→ rabbit_channel:handle_publish()
   │                                        │      ├─ 权限检查
   │                                        │      └─→ rabbit_exchange:route()
   │                                        │            ├─ 匹配 BindingKey
   │                                        │            └─→ rabbit_amqqueue:deliver()
   │                                        │                  ├─ 入队（内存/磁盘）
   │                                        │                  ├─ 持久化（msg_store）
   │                                        │                  ├─ 镜像复制（如启用）
   │                                        │                  └─→ 通知消费者 / 返回 Confirm
   │                                        │
   │←─── AMQP Basic.Ack (Confirm) ─────────│
   │     或 Basic.Nack (失败)              │
```

## 3.2 从 Consumer 消费到消息删除

```
Consumer                               RabbitMQ Broker
   │                                        │
   │──── AMQP Basic.Consume ───────────────→│ 注册消费者
   │                                        │  rabbit_amqqueue:basic_consume()
   │                                        │
   │←─── AMQP Basic.Deliver ────────────────│ 投递消息（Push）
   │                                        │
   │──── 业务处理...                          │
   │                                        │
   │──── AMQP Basic.Ack ───────────────────→│ rabbit_amqqueue:ack()
   │                                        │   ├─ 从队列删除消息
   │                                        │   ├─ 更新持久化存储
   │                                        │   └─→ 检查 TTL / DLX / 队列长度
```

## 3.3 Queue 进程内部状态

```
rabbit_amqqueue 进程内部：
┌────────────────────────────────────┐
│  Queue State                       │
│                                    │
│  msg_index: ETS Table              │  ← 消息索引（内存）
│  msg_store: 持久化存储引用          │  ← 磁盘消息文件
│  consumers: [Pid1, Pid2, ...]      │  ← 消费者进程列表
│  unacked: #{DeliveryTag => Msg}    │  ← 未确认消息
│  ready_queue: Priority Queue       │  ← 等待投递消息
│  ttl_timer: Timer                  │  ← TTL 定时器
│  policy: Policy Map                │  ← 策略配置
└────────────────────────────────────┘
```

---

# 4. 存储引擎原理

## 4.1 Classic Queue 存储架构

```
┌──────────────────────────────────────────────┐
│              Classic Queue 存储               │
│                                              │
│  ┌──────────────────┐   ┌────────────────┐  │
│  │  Queue Index     │   │  Message Store  │  │
│  │  (队列索引)       │   │  (消息存储)      │  │
│  │                  │   │                │  │
│  │  消息元数据:      │   │  消息体:        │  │
│  │  - SeqID         │──→│  - 完整消息内容  │  │
│  │  - 消息大小       │   │  - 分段存储     │  │
│  │  - 存储位置       │   │  - 可 GC 回收   │  │
│  │  - TTL           │   │                │  │
│  └──────────────────┘   └────────────────┘  │
│                                              │
│  小消息（< embed_size）直接嵌入 Index         │
│  大消息（>= embed_size）引用 Message Store    │
└──────────────────────────────────────────────┘
```

| 组件 | 说明 |
|------|------|
| **Queue Index** | 每个队列一个索引文件，记录消息元数据（SeqID、存储位置、大小） |
| **Message Store** | 全局共享的消息体存储，所有队列的消息体统一存放，使用引用计数管理生命周期 |
| **embed_msgs_below** | 默认 4096 字节，小于此值的消息体直接嵌入索引，减少 I/O |

## 4.2 消息持久化流程

```
消息发布
    │
    ▼
│ 确认消息     │ ← 消息写入 msg_store
│ 需要持久化?   │
    │
    ├─ 是 → disk write → fsync (取决配置) → Confirm
    └─ 否 → 内存 → Confirm
```

## 4.3 Quorum Queue 存储（Raft Log）

```
┌──────────────────────────────────────────────┐
│             Quorum Queue 存储                  │
│                                              │
│  Raft Log (持久化)  │  Segment Files（磁盘）  │
│  ┌──────────────┐   │  ┌──────────────────┐  │
│  │ Entry 1      │   │  │ 消息1完整内容     │  │
│  │ Entry 2      │   │  │ 消息2完整内容     │  │
│  │ Entry 3      │   │  │ 消息3完整内容     │  │
│  │ ...          │   │  │ ...              │  │
│  └──────────────┘   │  └──────────────────┘  │
│       ↑              └─────────────────────┘  │
│   WAL (预写日志)      Segment 文件存储消息体    │
│   复制到 Follower                               │
└──────────────────────────────────────────────┘
```

Quorum Queue 使用两个组件：
1. **Raft WAL**：记录所有操作（入队、出队），复制到 Follower，用于共识
2. **Segment Files**：存储实际消息体，定期被消费者确认后删除

---

# 5. 流量控制机制

RabbitMQ 使用基于**信用（Credit）**的流量控制，防止生产者和消费者之间的速度不匹配。

## 5.1 信用流控模型

```
┌─────────────────────────────────────────┐
│          Credit Flow Control            │
│                                         │
│  ┌──────────┐           ┌──────────┐   │
│  │ Producer │──Credit──→│  Queue   │   │
│  │          │←─耗尽─阻塞─│  Process │   │
│  └──────────┘           └──────────┘   │
│                                         │
│  初始信用: {InitialCredit, MoreCredit}   │
│  默认: {200, 50}                        │
│                                         │
│  Producer 每发一条消息消耗 1 个信用       │
│  信用耗尽 → 阻塞 Producer               │
│  Queue 处理 N 条后 → 发放 MoreCredit    │
└─────────────────────────────────────────┘
```

## 5.2 三层流控

| 层次 | 触发条件 | 效果 | 配置 |
|------|----------|------|------|
| **连接级** | Channel 信用耗尽 | 阻塞该 Channel 的 publish | `credit_flow_default_credit` |
| **队列级** | 队列进程消息积压 | 阻塞向该队列的发布 | 内部机制 |
| **全局级** | 内存或磁盘告警 | 阻塞所有连接的 publish | `vm_memory_high_watermark` / `disk_free_limit` |

## 5.3 内存流控详细流程

```
监控进程定期检查内存使用
    │
    ▼
内存使用 > high_watermark (默认 40%)?
    ├─ 否 → 什么都不做
    └─ 是 → 触发内存告警
              │
              ▼
         发布者列表排序（根据发布速度排名）
              │
              ▼
         按顺序阻塞发布者（速度快的先阻塞）
    │
    ▼
内存使用 < high_watermark * 0.8 (恢复阈值)?
    └─ 是 → 解除告警 → 逐步恢复发布者
```

---

# 6. 网络分区处理原理

## 6.1 什么是网络分区

当集群节点间的网络连接中断时，节点虽然仍在运行，但无法互相通信，形成**分区（Partition）**。

## 6.2 RabbitMQ 检测机制

```
心跳检测（默认 60s）
    ↓
节点 A 15s 内无法收到节点 B 的心跳
    ↓
节点 A 判定节点 B 可能宕机或网络分区
    ↓
根据 cluster_partition_handling 策略处理
```

## 6.3 Pause-Minority 策略（推荐）

```
┌──────────────────────────────────────────┐
│        分区前: 3 个节点                    │
│    [Node1]──[Node2]──[Node3]             │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│     分区后: 分成 2 个分区                   │
│                                          │
│  分区A: [Node1]──[Node2]  (2 个节点)       │
│     ↓ 多数方 → 继续运行                    │
│                                          │
│  分区B: [Node3]            (1 个节点)       │
│     ↓ 少数方 → 暂停服务 ⏸                 │
└──────────────────────────────────────────┘
```

## 6.4 Autoheal 策略

选择节点数量较多的一方恢复，少的一方重启，重新加入集群。

## 6.5 Ignore 策略（不推荐）

```
节点各自独立运行，分区恢复后会出现：
  - 消息重复（两边都认为自己是正确的）
  - 数据不一致
  - 脑裂
```

---

# 7. 元数据存储：Mnesia → Khepri

## 7.1 Mnesia（传统方案）

```erlang
% Mnesia 是 Erlang 内置的分布式数据库
% RabbitMQ 用于存储：
%   - 用户、权限（rabbit_user）
%   - VHost（rabbit_vhost）
%   - Exchange / Queue / Binding 元数据
%   - Policy / Parameter
%
% 问题：
%   - 网络分区后需要手动恢复
%   - 大规模集群（>7 节点）性能下降
%   - Schema 变更需要所有节点在线
```

## 7.2 Khepri（3.13+ 新方案）

Khepri 是 RabbitMQ 团队开发的基于 **Raft 共识协议**的分布式元数据存储，替代 Mnesia。

```erlang
% Khepri 特性：
%   - 基于 Raft，自动处理网络分区
%   - 支持更大规模集群
%   - 更好的性能和可靠性
%   - 向后兼容 Mnesia API
```

| 维度 | Mnesia | Khepri |
|------|--------|--------|
| 共识协议 | 无（依赖表复制策略） | Raft |
| 网络分区 | 需手动或 Autoheal | 自动处理 |
| 大规模集群 | 差（>7 节点性能问题） | 好（Raft 设计） |
| Schema 变更 | 需所有节点在线 | 单 Leader 执行 |
| 引入版本 | 初始 | RabbitMQ 3.12（实验），3.13（GA） |

---

# 8. AMQP 协议帧格式

RabbitMQ 的原生协议是 AMQP 0-9-1。AMQP 帧结构如下：

```
┌────────────────────────────────────────────────────────┐
│  AMQP Frame                                            │
│                                                        │
│  ┌──────────┬─────────┬───────────┬──────────┬───────┐ │
│  │   Type   │ Channel │   Size    │ Payload  │  End  │ │
│  │  (1 byte)│(2 bytes)│ (4 bytes) │ (variable)│(1 byte)│ │
│  ├──────────┼─────────┼───────────┼──────────┼───────┤ │
│  │  0x01    │   0~65535│ 0~4294MB  │          │ 0xCE  │ │
│  └──────────┴─────────┴───────────┴──────────┴───────┘ │
└────────────────────────────────────────────────────────┘
```

| 字段 | 大小 | 说明 |
|------|------|------|
| **Type** | 1 byte | 1=Method, 2=Header, 3=Body, 4=Heartbeat |
| **Channel** | 2 bytes | 信道编号（0~65535） |
| **Size** | 4 bytes | 有效载荷大小 |
| **Payload** | 变长 | 帧类型对应的内容 |
| **End** | 1 byte | 固定值 0xCE（帧尾标记） |

---

# 9. 核心数据结构（Erlang 源码片段解读）

## 9.1 队列状态定义

```erlang
% rabbit_amqqueue.erl (简化)
-record(q, {
    name,              % #resource{Name, VHost}
    durable,           % boolean
    auto_delete,       % boolean
    exclusive_owner,   % pid() | none
    arguments,         % [{atom(), any()}]
    pid,               % 队列进程 pid
    slave_pids,        % 镜像从节点 pid 列表
    policy,            % 策略映射
    state              % live | crashed | down
}).
```

## 9.2 消息投递流程（简化）

```erlang
% rabbit_channel.erl 的 handle_publish 核心逻辑:
handle_publish(ExchangeName, RoutingKey, Properties, Body, ChannelState) ->
    % 1. 权限检查
    ok = check_write_permission(ExchangeName, ChannelState),
    % 2. 查找交换机
    case rabbit_exchange:lookup(ExchangeName) of
        {ok, Exchange} ->
            % 3. 路由消息
            case rabbit_exchange:route(Exchange, RoutingKey, Properties) of
                [] -> 
                    % 无匹配队列 → 根据 mandatory 决定行为
                    handle_unroutable(Properties, ChannelState);
                Queues ->
                    % 4. 投递到每个匹配的队列
                    lists:foreach(fun(Q) -> 
                        rabbit_amqqueue:deliver(Q, Properties, Body) 
                    end, Queues),
                    % 5. Confirm（如果启用）
                    maybe_confirm(ChannelState)
            end;
        not_found ->
            channel_exception(not_found, ChannelState)
    end.
```

---

# 10. 并发模型深度解析

## 10.1 每个 Connection 一个 Reader 进程

```erlang
% rabbit_reader.erl
% 每个 TCP 连接创建一个 reader 进程
% reader 负责：
%  - AMQP 帧解析
%  - 心跳管理
%  - 从 TCP Socket 读取 → 分发到对应 Channel 进程
```

## 10.2 每个 Channel 一个进程

```erlang
% rabbit_channel.erl
% 每个 Channel 一个 Erlang 进程
% Channel 进程负责：
%  - 处理 AMQP 方法调用（Publish / Consume / Ack / Declare 等）
%  - 维护信道级别的状态（Confirm 序列号、QoS Prefetch 计数等）
%  - 与 Queue 进程通信（投递消息、获取消息）
```

## 10.3 每个 Queue 一个进程

```erlang
% rabbit_amqqueue.erl
% 每个 Queue 一个 Erlang 进程
% Queue 进程负责：
%  - 消息入队/出队
%  - 管理消费者列表
%  - 检查 TTL 过期
%  - 处理 DLX 死信
%  - 维护 Index 和 MsgStore
```

---

# 11. 原理面试要点

## 11.1 为什么 RabbitMQ 单线程模型性能很高？

1. **Erlang 轻量级进程**：每个连接/信道/队列都是独立的 Erlang 进程，天然并发
2. **I/O 多路复用**：Erlang VM 使用 epoll/kqueue 管理 TCP 连接
3. **内存操作**：消息优先存储在内存，磁盘 I/O 异步完成
4. **无锁设计**：Actor 模型天然避免锁竞争

## 11.2 消息如何保证不丢失？（三层保障）

```
1. Publisher Confirm  → 生产者知道消息是否到达 Broker
2. 持久化（Queue/Exch/Persistent） → Broker 重启后消息仍在
3. Consumer Manual ACK → 消费者确认后才删除消息
```

## 11.3 RabbitMQ 为什么用 Erlang 实现？

| 原因 | 说明 |
|------|------|
| **并发模型** | Actor 模型天然适合消息中间件的并发场景 |
| **容错性** | Let it Crash 理念 + 监督树，单进程崩溃不影响整体 |
| **热升级** | 可以在不停机的情况下更新代码和修复 Bug |
| **软实时** | Erlang 的调度器保证低延迟响应 |

## 11.4 网络分区时如何保证数据一致性？

- **Quorum Queue**：Raft 共识协议，多数派确认，少数分区自动停止服务
- **Pause-Minority**：少数节点暂停，防止脑裂
- **Khepri**：基于 Raft 的元数据存储，自动处理分区
