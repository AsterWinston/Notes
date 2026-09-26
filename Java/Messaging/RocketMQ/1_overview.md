
# 1. 什么是 RocketMQ

Apache RocketMQ 是一个**分布式消息和流处理平台**，以低延迟、高吞吐、高可靠著称。最初由阿里巴巴开发并用于支撑双11等大规模场景，2016年捐赠给 Apache 软件基金会并于2017年成为顶级项目。

## 1.1 发展历史

| 阶段 | 时间 | 里程碑 |
|------|------|--------|
| 起源 | 2011 | 阿里内部 MetaQ 1.0，用于交易核心链路 |
| 演进 | 2012-2014 | MetaQ 2.0 → RocketMQ 3.0，支持事务消息、顺序消息 |
| 开源 | 2016 | 进入 Apache 孵化器 |
| 毕业 | 2017.9 | 成为 Apache 顶级项目 |
| 5.0 | 2022 | RocketMQ 5.0 发布，架构重构，存算分离，支持 Pop 消费模式 |

## 1.2 核心特性

- **削峰填谷**：处理瞬时流量高峰，保护下游系统
- **解耦**：生产者和消费者独立演进
- **异步通信**：非阻塞、最终一致性
- **顺序消息**：全局有序和分区有序
- **事务消息**：分布式事务最终一致性
- **定时/延迟消息**：18 个延迟级别，精确到秒级
- **消息过滤**：Tag 过滤、SQL92 表达式过滤
- **消息重试与死信**：自动重试 + 死信队列兜底
- **高可用**：主从架构 + DLedger 自动选主
- **海量堆积**：基于磁盘存储，支持千亿级消息堆积

## 1.3 适用场景

- 异步解耦（订单系统 → 积分系统）
- 流量削峰（秒杀 → 抢购请求缓冲）
- 数据分发（Binlog 同步 → 多下游消费）
- 日志处理（日志收集 → 实时分析）
- 分布式事务（订单+库存+余额一致性）

# 2. 主流消息中间件对比

| 维度 | RocketMQ | Kafka | RabbitMQ | Pulsar |
|------|----------|-------|----------|--------|
| 架构 | 注册中心 + Broker 主从 | 分区多副本 + ZooKeeper | AMQP Broker | 存算分离 BookKeeper |
| 吞吐量 | 十万级 TPS | 百万级 TPS | 万级 TPS | 十万级 TPS |
| 延迟 | 毫秒级 | 毫秒级 | 微秒级 | 毫秒级 |
| 事务消息 | 原生支持 | 支持（0.11+） | 不支持 | 原生支持（4.0+） |
| 顺序消息 | 支持 | 分区内有序 | 不支持 | 支持 |
| 延迟消息 | 18个级别 | 不支持（需外部实现） | 插件支持 | 支持 |
| 消息过滤 | Tag/SQL92 | 不支持（消费端过滤） | 不支持 | 支持 |
| 消息重试 | 支持 | 不支持（需手动） | 支持 | 支持 |
| 社区/生态 | Apache，国内活跃 | Apache，全球最大 | VMware，成熟 | Apache，增长快 |
| 主要场景 | 电商/金融/物流 | 大数据/日志 | 通用消息路由 | 云原生 |

## 2.1 选择建议

- **高吞吐 + 流处理**：Kafka
- **事务/顺序/延迟 + 电商金融**：RocketMQ
- **复杂路由 + 轻量消息**：RabbitMQ
- **存算分离 + 多租户**：Pulsar

# 3. RocketMQ 5.x 新特性

## 3.1 架构变化

```
RocketMQ 4.x:  Producer → NameServer → Broker(M/S) → Consumer
RocketMQ 5.x:  Producer → Proxy → NameServer → Broker → Consumer
                              ↓
                         gRPC 协议
```

- **Proxy 组件**：解耦协议层与内核，支持 gRPC/HTTP/Remoting
- **存算分离**：Broker 拆分为逻辑计算层 + 物理存储层
- **Pop 消费模式**：Broker 代理拉取，降低 Consumer 逻辑复杂度
- **轻量级 SDK**：多语言 SDK 通过 Proxy 接入

## 3.2 关键改进

| 特性 | 4.x | 5.x |
|------|-----|-----|
| 协议层 | 自定义 Remoting | gRPC + Proxy |
| 消费模式 | Push/Pull | 新增 Pop 模式 |
| 重平衡 | Client 端 | 可迁移至 Proxy |
| 多语言 | Java SDK | 各语言轻量 SDK |
| 存储 | 本地文件 | 支持对象存储扩展 |

# 4. 参考

- [Apache RocketMQ 官网](https://rocketmq.apache.org/)
- [RocketMQ GitHub](https://github.com/apache/rocketmq)
- [RocketMQ 5.0 设计文档](https://github.com/apache/rocketmq/blob/master/docs/cn/rocketmq.pdf)
