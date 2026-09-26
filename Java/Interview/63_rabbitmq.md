# 1. 知识点
- **AMQP协议(Advanced Message Queuing Protocol)**: RabbitMQ基于AMQP 0-9-1协议的实现，是应用层标准高级消息队列协议。定义消息格式、路由规则、可靠性传输机制，跨语言跨平台
- **核心组件-VirtualHost(vhost)**: 虚拟主机，是RabbitMQ的顶层逻辑隔离单元。每个vhost拥有独立的Exchange、Queue、Binding和权限体系，用户绑定到vhost。多个应用通过不同vhost在同一个RabbitMQ实例上做到逻辑隔离
- **核心组件-Connection**: AMQP长连接，TCP连接。创建和销毁开销大(需三次握手)，需要一个长连接保活。但一个TCP连接上可以复用多个Channel
- **核心组件-Channel**: 轻量级虚拟连接，复用同一个TCP连接，每个Channel有独立ID。创建和销毁开销极小，生产者和消费者都通过Channel与RabbitMQ交互。Channel不是线程安全的，多线程需各自创建Channel
- **核心组件-Exchange(交换机)**: 接收生产者发来的消息，根据RoutingKey和Binding规则将消息路由到匹配的Queue。不存储消息，只做路由转发
- **核心组件-Queue(队列)**: 存储消息的缓冲区。生产者消息经Exchange路由最终到达Queue，消费者从Queue中拉取或由RabbitMQ推送消息
- **核心组件-Binding(绑定)**: Exchange与Queue之间的关联关系。Binding时指定RoutingKey和BindingKey的匹配规则
- **核心组件-RoutingKey**: 消息携带的路由键。生产者发送消息时指定RoutingKey，Exchange根据Binding规则决定消息路由到哪些Queue
- **Exchange类型-Direct**: 精确匹配RoutingKey。消息的RoutingKey必须和BindingKey完全一致，消息才转发到对应Queue。适用于单播/精确路由(如下单通知)
- **Exchange类型-Fanout**: 广播模式，忽略RoutingKey。将消息投递到所有绑定的Queue，每个Queue都能收到全量消息。适用于广播通知(如配置更新、日志分发)
- **Exchange类型-Topic**: 通配符模式匹配RoutingKey。`*`(星号)匹配恰好一个词，`#`(井号)匹配零个或多个词，词间用`.`分隔。如`order.*`匹配order.create/order.pay，`order.#`匹配order.create.log等所有子级
- **Exchange类型-Headers**: 根据消息Headers属性匹配，忽略RoutingKey。Binding时指定一组KV匹配条件(支持x-match=all全部匹配或x-match=any任一匹配)。很少使用，性能差于Topic
- **默认Exchange**: RabbitMQ内置的nameless Exchange(名称为空字符串)，类型为Direct。每个Queue自动以Queue名称作为BindingKey绑定到默认Exchange，实现快速开发
- **消息模型**: 生产者(Producer)创建消息→发送到Exchange(指定RoutingKey)→Exchange根据Binding规则匹配→消息投递到Queue→消费者(Consumer)从Queue消费
- **生产者Confirm确认机制**: ack/nack回调确认。开启Confirm模式后，消息到达Broker后会给生产者异步回调ack(确认)/nack(拒绝)。注意：如果已投递到Queue但未持久化就宕机，消息仍可能丢失
- **消费者ACK机制-手动ACK**: `basicAck(long deliveryTag, boolean multiple)` 确认单条/多条消息。消费者处理完毕后显式确认，RabbitMQ收到ACK后从Queue中删除消息。保证消息不丢失
- **消费者ACK机制-自动ACK**: 消费者收到消息立即自动确认，不管业务是否处理成功。如果消费者宕机，已推送但未处理的消息会丢失。生产环境不推荐
- **消费者ACK机制-basicReject/basicNack**: `basicReject(deliveryTag, requeue)` 拒绝单条；`basicNack(deliveryTag, multiple, requeue)` 拒绝一条或多条。requeue=true重新入队，requeue=false成为死信或丢弃
- **消息持久化**: Exchange持久化(durable=true)、Queue持久化(durable=true)、Message持久化(MessageProperties.PERSISTENT_TEXT_PLAIN → DeliveryMode=2)。三者全开启+Confirm手动ACK机制=消息不丢失
- **死信队列(DLQ-Dead Letter Queue)**: 普通队列的消息变为死信(Dead Message)时，被重新投递到死信交换机(Dead Letter Exchange)，再由死信交换机路由到死信队列。可在死信队列中人工/自动处理失败消息
- **死信产生的三种情况**: 1)消息被消费者`basicReject/basicNack`且`requeue=false`；2)消息TTL过期(队列/消息设置了`x-message-ttl`)；3)队列达到最大长度(x-max-length)后，队头消息被挤出成为死信
- **延迟队列**: RabbitMQ原生不支持任意延迟，通过TTL+DLQ组合实现：消息发到设了TTL的队列→超时未消费变为死信→死信交换机投递到实际消费队列；或安装`rabbitmq_delayed_message_exchange`插件，直接在Exchange层延迟指定时间后投递
- **优先级队列**: 通过参数`x-max-priority`设置队列最大优先级(0-255，推荐0-10)。生产者发消息时设置优先级值，数值越大越优先被消费。只有消息堆积时优先级才有效
- **消息幂等性**: 消息可能重复投递(网络超时重试等)，消费端需保证消费幂等。方案：每条消息携带全局唯一ID(如UUID/NanoID)，消费前在Redis/DB中查询该ID是否已处理，已处理则直接ACK跳过
- **消息顺序性**: 单Queue单Consumer保证FIFO(先进先出)。多个Consumer轮询分发(Round-Robin)会破坏顺序，因为每个Consumer处理速度不同；带线程池异步处理也会打乱顺序。保序需单线程串行消费
- **消息堆积**: 生产者速度远超消费者速度时堆积。解决：1)增加消费者实例并发消费；2)设置消息TTL过期丢弃；3)死信队列分流(堆积后成为死信转到DLQ异步处理)；4)增加Queue数量分摊压力；5)消费者端限流(`basicQos`设置prefetch_count每次拉取数量)；6)批量ACK减少网络开销
- **QoS/prefetch_count**: `channel.basicQos(int prefetchCount)` 设置消费者最多同时接收多少条未ACK的消息。prefetch_count=1时轮询分发变为能者多劳(处理完才能接收下一条)。防止消费者处理慢导致消息全部积压在一个消费者
- **集群模式-普通集群**: 元数据(Exchange/Queue/Binding)在所有节点同步共享，但Queue数据只存在一个节点。访问其他节点时自动转发到Queue所属节点，存在单点故障风险。元数据同步不会有瓶颈
- **集群模式-镜像集群(Mirrored Queue)**: Queue在多个节点有Master+Slave副本。Master接收写入，同步到所有Slave。Master宕机时自动选最老的Slave提升为Master。保证数据高可用，但有性能损耗(同步开销)
- **集群模式-Quorum Queue(RabbitMQ 3.8+)**: 基于Raft共识协议的队列，替代镜像队列的官方推荐方案。强一致性、自动故障转移、数据安全
- **RabbitMQ vs RocketMQ vs Kafka**:
  | 维度 | RabbitMQ | RocketMQ | Kafka |
  |------|----------|----------|-------|
  | 协议 | AMQP | 自研(类JMS) | 自研 |
  | 吞吐量 | 万级 | 十万级 | 百万级 |
  | 可靠性 | 高(持久化+ACK) | 极高(同步刷盘) | 极高(分区副本) |
  | 延时 | 微秒级(极低) | 毫秒级 | 毫秒级(批处理高) |
  | 事务消息 | 不支持 | 支持(半消息) | 支持(幂等) |
  | 社区 | 活跃(全球) | 阿里开源 | Apache顶级 |
  | 适用场景 | 低延迟实时、RPC调用 | 金融/交易/可靠通知 | 大数据/日志/流处理 |
- **为什么用MQ-异步处理**: 非核心流程走异步MQ，接口快速返回，提升用户体验和响应速度。如注册成功后异步发短信、发积分
- **为什么用MQ-削峰填谷**: 高峰期流量存入MQ缓冲，后端按自己能力匀速消费，平滑流量，防止瞬时高峰打垮下游服务
- **为什么用MQ-解耦**: 服务间不直接调用，通过MQ订阅/发布模型松耦合。上游发布事件，下游根据自身需求订阅，增加/减少消费者不影响上游
- **多线程异步 vs MQ**: 多线程在应用进程内执行/受CPU资源限制/任务不能被持久化(进程崩溃丢失)/不适合海量任务和分布式系统；MQ独立部署/支持消息堆积和持久化/跨服务分布式/适合大规模异步解耦场景
- **消息补偿/重试机制**: 消费失败时做有限次数重试(如3次)，超过重试次数后进入死信队列，人工介入处理
- **消息追踪**: 每条消息携带全局唯一traceId，在生产者/Exchange/Queue/消费者各环节记录日志，方便排查消息丢失和重复消费问题

# 2. 面试题

## 2.1 RabbitMQ中有哪几种Exchange类型？
RabbitMQ提供4种核心Exchange类型，外加内置的默认Exchange：

1) **Direct Exchange(直连交换机)**：精确匹配RoutingKey。消息的RoutingKey必须与BindingKey完全一致。适用于单播、精确路由场景(如订单通知、私信)

2) **Fanout Exchange(扇出交换机)**：广播模式，忽略RoutingKey。消息投递到所有绑定的Queue。适用于广播通知、配置更新推送、日志分发

3) **Topic Exchange(主题交换机)**：通配符模式匹配RoutingKey。`*`匹配恰好一个词，`#`匹配零个或多个词，词之间用`.`分隔。如`order.*`匹配`order.create`/`order.pay`，`order.#`匹配`order.create.log`等。适用于灵活路由、事件订阅

4) **Headers Exchange(头交换机)**：根据消息Headers属性匹配，忽略RoutingKey。Binding时指定一组KV条件，支持x-match=all(全部匹配)或x-match=any(任一匹配)。性能差于Topic，极少使用

5) **Default Exchange(默认交换机)**：RabbitMQ内置的无名交换机(空字符串)，类型为Direct。每个Queue自动以Queue名称作为RoutingKey绑定到默认交换机，无需显式声明Binding。

实际选型：单播用Direct，广播用Fanout，灵活路由用Topic。

## 2.2 什么是死信队列？什么情况下消息会变成死信？
**死信(Dead Letter)**是Normal Queue中无法被正常消费的消息。当消息变为死信时，会被重新投递到该Queue配置的**死信交换机(DLX-Dead Letter Exchange)**，再由DLX路由到**死信队列(DLQ-Dead Letter Queue)**进行后续处理。

消息变成死信的三种情况：
1) **消息被拒绝(Reject/Nack)**：消费者调用`basicReject`或`basicNack`且设置`requeue=false`，消息不被重新入队
2) **消息TTL过期**：队列设置`x-message-ttl`或消息单独设置`expiration`，超过指定时间未被消费
3) **队列达到最大长度**：队列设置`x-max-length`，消息数超过限制后，队头最早的消息被挤出成为死信

死信队列的作用：
- 异常消息的兜底处理(如消费多次失败的消息)，人工介入排查修复
- 结合TTL实现延迟队列(下文详述)
- 消息审计和监控告警(死信率异常时触发告警)

配置死信队列的方式：声明Queue时指定`x-dead-letter-exchange`和`x-dead-letter-routing-key`参数。

## 2.3 如何保证RabbitMQ消息的可靠性？
消息可靠性保证贯穿消息从生产到消费的全链路，需要三端配合：

**1) 生产者端——Confirm确认机制**：
开启`Publisher Confirm`模式。消息发送到Exchange后，RabbitMQ异步回调ack/nack。收到ack后生产者可删除本地消息记录，收到nack或超时则重发。Spring Boot中配置`spring.rabbitmq.publisher-confirm-type=correlated`，配合`RabbitTemplate.ConfirmCallback`处理。

**2) Broker端——持久化**：
- 声明Exchange时设置`durable=true`
- 声明Queue时设置`durable=true`
- 消息发送时设置`DeliveryMode=2`(MessageProperties.PERSISTENT_TEXT_PLAIN)
三者全部持久化才能保证Broker重启后消息不丢失。注意：持久化性能开销大，需权衡。

**3) 消费者端——手动ACK**：
设置`spring.rabbitmq.listener.simple.acknowledge-mode=manual`，消费者处理完业务逻辑后调用`channel.basicAck(deliveryTag, false)`确认消费。只有收到ACK后RabbitMQ才删除消息。如果消费者宕机，未ACK的消息会重新投递给其他消费者。

**完整可靠链路**：
生产者发送→Confirm回调确认→Broker持久化→消费者手动ACK，任何一个环节失败都能发现并重试。

**注意**：即使以上三步都做到，也可能因集群中主节点宕机且持久化未完成刷盘导致极小概率的丢失。金融级别可靠性建议使用RocketMQ的同步刷盘机制。

## 2.4 RabbitMQ中如何解决消息堆积问题？
消息堆积的根本原因是**生产速度 > 消费速度**。解决方案按优先级排序：

**1) 增加消费者**：
- 增加消费者实例数量，水平扩展消费能力
- 注意：Queue中的消息默认轮询分发给消费者，增加消费者直接线性提升消费能力

**2) 消费者端优化**：
- 批量ACK，减少网络往返次数
- 使用线程池并发处理消息(注意顺序性要求)
- 优化业务逻辑(如减少DB请求、使用缓存)

**3) 限流(Flow Control)**：
- 设置`basicQos(prefetchCount)`控制消费者每次最多接收的消息数。prefetchCount过大会导致消息积压在消费者内存中，过小导致处理效率低。一般设为消费能力×平均处理时间的2-3倍

**4) 消息TTL**：
- 设置消息过期时间，超时未消费自动转为死信或丢弃
- 适用于实时性要求高但过期无意义的消息(如实时股价推送)

**5) 死信队列分流**：
- 堆积严重的消息成为死信后转入DLQ，由独立服务异步处理，不影响主流程

**6) 增加Queue数量**：
- 使用Topic Exchange将消息按RoutingKey分摊到不同Queue，每个Queue由独立消费者组处理

**7) 监控告警**：
- 设置Queue length告警阈值，提前发现堆积趋势，在堆积严重前处理

## 2.5 RabbitMQ中如何保证消息有序性？
RabbitMQ本身保证**单个Queue内的消息按入队顺序投递给同一个消费者**时为FIFO。破坏顺序的场景及解决方案：

**破坏顺序的场景**：
1) **多Consumer轮询分发(Round-Robin)**：消息被分发给多个Consumer并行处理，处理速度不同导致乱序
2) **消费者端多线程/线程池**：单个Consumer收到消息后交给线程池并发处理
3) **消息重试**：失败的消息被重新入队/被其他消费者消费，与时序消息交叉

**保证顺序的方案**：
1) **单Queue+单Consumer**：一个Queue只绑定一个Consumer，单线程串行消费。消息严格按FIFO
2) **分区顺序**：按业务Key(如用户ID/订单ID)路由到不同Queue，每个Queue只一个Consumer。同类消息顺序保证，不同类消息并行处理
3) **RabbitMQ插件-Single Active Consumer**：Queue设置`x-single-active-consumer=true`，集群中只有一个活跃Consumer，故障时自动切换到备用Consumer

注意：RabbitMQ不擅长严格全局顺序消息(不像RocketMQ有顺序消息特性)。如果业务强依赖全局顺序，建议使用RocketMQ的FIFO队列或Kafka的单分区。

## 2.6 如何防止消息重复消费？(如何保证消息幂等性)
消息重复消费的根源：生产者重试(网络超时)、Broker故障转移重新投递、消费者ACK丢失(网络断开)。MQ大多保证至少一次投递(At-least-once)，所以消费端必须保证幂等。

**方案1——消息唯一ID+Redis/DB去重**（最常用）：
生产者发送时为每条消息生成全局唯一ID(如雪花算法/UUID)，消费者收到消息后：
1) 尝试在Redis中`SET key value NX EX 3600`(key为消息ID)，如果不成功说明已处理过
2) 或在DB中`INSERT IGNORE INTO consumed_msgs(msg_id)`，利用唯一索引保证不重复插入
3) 已处理的消息直接ACK跳过

**方案2——业务状态机**：
利用业务数据的唯一约束。如订单状态流转`待支付→已支付`，更新时加条件`UPDATE order SET status='已支付' WHERE id=1 AND status='待支付'`，第二次更新影响行数为0，天然幂等。

**方案3——版本号/乐观锁**：
每条数据加version字段，更新时`UPDATE ... SET version=version+1 WHERE id=? AND version=?`，第二次执行不满足条件。

需根据业务场景选择：通用方案用消息ID去重；有状态流转的业务用状态机。两者可结合，消息ID去重做第一层兜底。

## 2.7 多线程异步和MQ的区别？
| 维度 | 多线程异步 | MQ |
|------|-----------|-----|
| 执行范围 | 应用进程内 | 独立中间件，跨服务 |
| 资源竞争 | 共享CPU/内存，资源紧张时互相影响 | 独立资源，不互相影响 |
| 可靠性 | 无持久化，进程崩溃任务丢失 | 持久化到磁盘，支持重试和死信 |
| 扩展性 | 单机垂直扩展，有上限 | 水平扩展，分布式架构 |
| 任务堆积 | 内存队列满则OOM/丢弃 | 支持大量消息堆积 |
| 解耦程度 | 强耦合(同进程内) | 完全解耦(发布订阅模型) |
| 管理能力 | 无 | 可视化管理、监控、回溯 |
| 适用场景 | 轻量、实时、进程内任务 | 大规模、分布式、可靠性要求高 |

**什么时候用多线程异步**：
- 任务简单、执行时间短、不需要持久化
- 进程内事件通知(如发布-订阅模式中通知多个监听器)
- 轻量级异步：如`@Async`注解处理发送短信

**什么时候用MQ**：
- 跨服务异步通信(订单系统通知库存系统)
- 削峰填谷(秒杀场景)
- 需要可靠性保证(消息不丢、可重试)
- 需要广播/多订阅者

## 2.8 为什么用MQ？异步、削峰、解耦
**1) 异步处理——提升响应速度**：
将非核心/耗时流程从同步调用改为异步消息。如用户注册：同步只需完成注册落库并返回成功，异步发MQ处理发送欢迎短信、积分发放、数据分析等后续任务。用户感知的响应时间大幅缩短。

**2) 削峰填谷——平滑流量**：
秒杀/大促场景，流量瞬时暴增远超后端处理能力。MQ作为缓冲区暂存请求，后端以恒定速度消费。高峰期入队(TPS=10w)、低谷期消费(TPS=5k)。避免流量直接击垮数据库和服务。

**3) 解耦——系统间松耦合**：
同步调用时，上游需知道所有下游地址，下游变更影响上游。引入MQ后，上游只发消息到Topic，下游自行订阅。新增/下线消费者对上游零影响。如订单创建后，库存、物流、通知、数据分析各系统独立订阅`order_created`事件。

**额外好处**：
- **数据分发**：一条消息被多个下游消费(如大数据平台和监控平台同时订阅)
- **故障隔离**：下游宕机不影响上游，消息堆积在MQ中待恢复后继续消费

## 2.9 RabbitMQ和Kafka、RocketMQ有什么区别？如何选型？
| 维度 | RabbitMQ | RocketMQ | Kafka |
|------|----------|----------|-------|
| 语言 | Erlang | Java | Java/Scala |
| 协议 | AMQP 0-9-1 | 自研(类JMS) | 自研 |
| 吞吐量 | 万级(单机约2-3w/s) | 十万级(单机约10w+/s) | 百万级(单机约100w+/s) |
| 延时 | 微秒级(极低) | 毫秒级 | 毫秒级(批处理) |
| 可靠性 | 高(持久化+镜像队列) | 极高(同步刷盘+同步复制) | 极高(分区副本+ISR) |
| 事务消息 | 不支持 | 支持(半消息回查) | 支持(幂等生产者) |
| 顺序消息 | 弱(单Queue+FIFO) | 强(分区顺序+全局顺序) | 强(单分区有序) |
| 延迟消息 | 插件支持 | 原生支持(18级) | 不支持 |
| 消息回溯 | 不支持(消费即删除) | 支持(按时间回溯) | 支持(按offset回溯) |
| 消息追踪 | 需要额外开发 | 原生消息轨迹 | 需额外配置 |
| 集群管理 | 简单(内置) | NameServer+Broker | ZK/KRaft+Broker |
| 社区生态 | 全球活跃/中小型 | 阿里开源/国内活跃 | Apache顶级/大数据生态 |

**选型建议**：
- **RabbitMQ**：对延迟敏感、实时性高、消息量不大(万级)、需要灵活路由的互联网应用。如实时推送、微服务内部RPC
- **RocketMQ**：金融/电商/交易场景，需要事务消息、顺序消息、高可靠性。阿里/滴滴等大厂标配。如订单支付、库存扣减
- **Kafka**：大数据/日志/流处理/事件溯源场景，超高吞吐。如用户行为埋点、日志采集ELK、实时计算

简单口诀：**实时用Rabbit，金融用Rocket，大数据用Kafka**。

## 2.10 RabbitMQ如何实现延迟队列？
RabbitMQ实现延迟队列有两种方式：

**方式1——TTL + 死信队列(DLQ)组合**（原生方式）：
1) 声明一个TTL队列`delay_queue`，设置`x-message-ttl`为延迟时间，并设置`x-dead-letter-exchange`指向目标交换机
2) 生产者将消息发到`delay_queue`
3) 消息在`delay_queue`中没人消费，超过TTL后过期变为死信
4) 死信自动投递到DLX，再路由到真正的消费队列`target_queue`
5) 消费者监听`target_queue`消费，实现延迟效果

缺点：若有多个不同延迟时间的需求，需创建多个TTL队列(每个队列TTL固定)。且消息在TTL队列中按FIFO，队头消息未过期时，新到达的短TTL消息会被堵住。

**方式2——rabbitmq_delayed_message_exchange插件**（更灵活）：
1) 安装插件：`rabbitmq-plugins enable rabbitmq_delayed_message_exchange`
2) 声明Exchange时设置类型为`x-delayed-message`，参数`x-delayed-type`指定原始Exchange类型(如direct/topic)
3) 生产者发送消息时设置`x-delay`Header为延迟毫秒数
4) 消息先到延迟交换机，交换机在内部保持消息到期后才投递到目标Queue

优点：不需要额外创建TTL队列和死信队列，每条消息可单独设置延迟时间，灵活度极高。缺点：依赖插件，非官方内置。

## 2.11 RabbitMQ的集群模式有哪些？
**1) 普通集群(元数据共享)**：
- VHost/Exchange/Queue/Binding/User等元数据在所有节点同步共享
- Queue的**消息内容只存在一个节点上**，其他节点只有元数据(指向消息所在节点)
- 消费者连任意节点都能消费，但若非本地节点，请求被转发到Queue实际所在的节点
- 优缺点：配置简单，但无数据冗余，消息所在节点宕机后Queue不可用(元数据还在但消息丢失)

**2) 镜像集群(Mirrored Queue)**：
- Queue在多个节点上设有副本——一个Master和多个Slave
- Master接收所有写入，并将消息同步到所有Slave
- Master宕机时，存续时间最长的Slave自动提升为Master
- 配置策略(policy)：`ha-mode: all`(所有节点镜像) / `ha-mode: exactly`(指定数量) / `ha-mode: nodes`(指定节点列表)
- 优缺点：高可用、数据不丢失。但性能有损耗(同步开销)、网络开销大、磁盘占用翻倍

**3) Quorum Queue(RabbitMQ 3.8+)**：
- 基于Raft共识协议的新型队列，替代镜像队列的官方推荐方案
- 强一致性、自动Leader选举、自动故障转移
- 比镜像队列性能更好、更安全，支持Poison Message处理

**其他集群特性**：
- **节点类型**：RAM节点(元数据内存)/Disk节点(元数据磁盘)。至少一个Disk节点
- **脑裂防护**：`cluster_partition_handling`策略配置(ignore/autoheal/pause_minority)
- **联邦/Shovel**：跨数据中心的集群联动(广域网消息转发)

生产环境建议：使用镜像集群或Quorum Queue保证高可用；至少3节点部署避免单点故障。

## 2.12 什么是Channel？为什么用Channel而不是直接用TCP连接？
**Channel(信道)**是建立在TCP连接之上的虚拟轻量级连接。每个Channel有独立ID，复用一个TCP连接。

**为什么不用TCP连接直接交互**：
1) **连接开销大**：TCP建立需要三次握手、端口分配、内核资源。每个生产者和消费者若都建立独立TCP连接，连接数爆炸(如1000个生产者=1000个TCP连接)
2) **资源消耗大**：每个TCP连接占用文件描述符、内存缓冲区。RabbitMQ单机能支撑的TCP连接数有限
3) **Channel开销极轻**：创建/销毁只需要在TCP连接上分配/释放一个ID号，不涉及握手和内核资源分配。在一条TCP连接上可以创建**数万个Channel**
4) **复用提高性能**：所有Channel共享一条TCP连接的带宽和缓冲，减少系统调用和数据拷贝

**类比**：TCP连接=高速公路，Channel=车道。建一条高速路代价高，但路上可以划很多车道。

**注意事项**：
- Channel**不是线程安全的**，每个线程应使用独立Channel
- Channel有连接上限(默认每个连接最多65535个Channel，受2字节ChannelID限制)
- Channel异常关闭时，其上所有未ACK的消息会重新入队

**Spring AMQP中的Channel管理**：`RabbitTemplate`内部通过`CachingConnectionFactory`管理Channel的创建和复用(缓存Channel提高性能)，开发者一般不需要手动管理Channel。

## 2.13 Session和Cookie的关系是什么？
Session和Cookie是配合使用的。**Session的运行依赖Cookie**：

1) 用户首次访问，服务器创建Session对象(生成唯一JSESSIONID)
2) 服务器通过HTTP响应的`Set-Cookie`头将`JSESSIONID=xxxxx`发送给浏览器
3) 浏览器存储该Cookie，后续每次请求自动在`Cookie`请求头中携带JSESSIONID
4) 服务器收到JSESSIONID后从Session存储(内存/Redis)中查找对应Session对象，识别用户身份

**如果禁用Cookie怎么办**：
通过**URL重写(URL Rewriting)**将JSESSIONID附加在URL后面，如`http://example.com/index;jsessionid=ABC123`。Servlet中通过`response.encodeURL()`自动添加。但URL重写不美观、容易泄露(被复制分享)、不利于SEO，一般不建议。

## 2.14 Cookie与Session有什么区别？
| 维度 | Cookie | Session |
|------|--------|---------|
| 存储位置 | 浏览器端(客户端) | 服务器端 |
| 大小限制 | 单个Cookie不超过4KB，每个域名最多20-50个 | 无限制(取决于服务器内存/存储) |
| 数据类型 | 仅字符串 | 任意Java Object |
| 安全性 | 较低(存储在客户端可被查看/篡改) | 较高(数据在服务端，客户端仅有SessionID) |
| 生命周期 | 可通过Expires/Max-Age设置长期保存(如"记住我"7天) | 默认30分钟(inactive超时)，浏览器关闭后JSESSIONID Cookie被清除(会话Cookie) |
| 网络开销 | 每次请求都携带Cookie，增加请求大小 | 仅传递SessionID，开销极小 |
| 跨域 | 不支持跨域名(可通过Domain属性控制) | 不支持(需处理跨域Session共享) |

## 2.15 如何实现Session共享？
分布式架构中多台服务器需要共享Session，否则用户请求被Nginx轮询到不同服务器时登录态丢失。

**方案1——Session持久化到Redis/DB(推荐)**：
所有服务将Session统一存储在Redis(内存)或MySQL中。请求到达任意服务器时，服务器使用JSESSIONID到Redis查找Session对象。
- 优点：集中管理、性能高(Redis内存操作)、可持久化
- 缺点：Redis故障影响全局，需Redis集群保证高可用

**方案2——无状态JWT替代Session**：
不使用服务端Session，改用JWT令牌。用户登录后服务端签发JWT返回给客户端，客户端每次请求携带JWT(Header Authorization)。服务端不存储任何会话状态，只验证签名。
- 优点：天然支持分布式/跨域、服务端无状态、不依赖存储
- 缺点：无法主动踢出用户(需额外黑名单)、令牌一经签发在有效期内一直有效

**方案3——Spring Session框架**：
Spring生态的首选方案，只需简单配置即可将Session存储到Redis/JDBC/MongoDB等后端，对业务代码透明。

```java
// application.yml
spring:
  session:
    store-type: redis  # 存储类型
    timeout: 30m       # Session超时

// Redis存储需引入: spring-session-data-redis
```

- 注解`@EnableRedisHttpSession`即可启用
- 对开发者完全透明，仍使用`HttpSession session = request.getSession()`读写Session
- Spring Session自动处理序列化/反序列化

## 2.16 什么是CSRF攻击？如何防护？
**CSRF(Cross-Site Request Forgery)**跨站请求伪造是一种利用用户已登录身份在第三方站点发起恶意请求的攻击方式。

**攻击原理**：
1) 用户登录了银行网站A(获取登录态Cookie)
2) 在未退出A的情况下去浏览一个黑客网站B
3) B网站中嵌入隐藏的表单/图片：`<img src="http://bank.com/transfer?to=hacker&amount=10000" />`
4) 浏览器加载图片时自动携带A网站的Cookie发起`/transfer`请求
5) 银行服务器认为这是已登录用户本人的合法操作

**核心**：浏览器默认行为——发起跨域请求时也会自动携带目标域名的Cookie。

**为什么JWT可以避免CSRF**：
普通Session Cookie在跨域请求时自动携带。而JWT通常存储在localStorage或由前端手动添加到Authorization请求头中，**跨域请求不会自动携带JWT**，攻击者无法伪装。

**防护措施**：
1) **CSRF Token**：服务器生成随机Token放在页面(隐藏字段/Header)，请求时提交Token，服务端验证。攻击者的伪造表单无法获取Token
2) **SameSite Cookie**：`Set-Cookie: session=xxx; SameSite=Strict/Lax`。SameSite=Strict跨站请求完全不发送Cookie；SameSite=Lax仅GET导航请求发送。从根源上阻止Cookie携带
3) **Referer/Origin校验**：服务端检查请求头Referer或Origin是否是可信域名。但部分浏览器/网络环境不发送Referer头，有局限性
4) **验证码**：关键操作(转账/支付/修改密码)要求输入验证码或二次认证
5) **自定义请求头**：前端发AJAX请求时加自定义Header头，浏览器会先发OPTIONS预检请求，若服务端不返回对应CORS头则请求被拦截

## 2.17 为什么要用双Token无感刷新？解决了什么问题？
双Token策略即 **Access Token(访问令牌) + Refresh Token(刷新令牌)**，解决的是JWT过期后用户无感知续期的体验问题。

**痛点**：如果只有单一Token：
- Token有效期设短(如15分钟)：用户频繁需要重新登录，体验极差
- Token有效期设长(如7天)：Token一旦泄露，攻击者可长期冒用，安全风险大

**双Token方案**：
| 维度 | Access Token | Refresh Token |
|------|-------------|---------------|
| 用途 | API请求认证 | 刷新Access Token |
| 有效期 | 短(15-30分钟) | 长(7-30天) |
| 存储 | 内存/变量 | HttpOnly Cookie/安全存储 |
| 暴露风险 | 高(每次请求携带) | 低(仅在刷新接口使用) |

**工作流程**：
1) 用户登录 → 服务端返回Access Token(15分钟) + Refresh Token(7天)
2) 前端每次API请求携带Access Token在Authorization头
3) 服务端返回401(过期)时，前端自动用Refresh Token调`/refresh`接口换取新的Access Token
4) 用户完全无感知，无需重新登录
5) Refresh Token也过期时，才跳转到登录页

**解决的问题**：
- **用户体验**：Access Token频繁过期，前端自动刷新，用户无感知
- **安全性**：Access Token短有效期即使泄露影响也有限；Refresh Token更长但使用频率低、暴露面小
- **权限控制**：可将Refresh Token也加入黑名单实现强制踢人

## 2.18 JWT和普通Token有什么区别？
| 维度 | 普通Token | JWT(Json Web Token) |
|------|-----------|---------------------|
| 定义 | 无固定格式，只是一个随机字符串 | 三段式结构(Header.Payload.Signature)的JSON格式令牌 |
| 结构 | 无固定结构，如`abc123xyz` | `eyJhbGciOiJIUzI1NiJ9.eyJ1c2VySWQiOjF9.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c` |
| 签名 | 不内置签名(安全性靠实现) | 内置Signature，使用密钥对Header+Payload签名，防篡改 |
| 状态性 | 通常需要将Token和用户信息在服务端关联(查DB/Redis) | **无状态(Stateless)**，Payload中可携带用户信息，验证签名后直接使用，不需要查数据库 |
| 安全性 | 依赖实现 | 签名防篡改，支持多种算法(HS256/RS256/ES256) |
| 适用场景 | 简单Token认证(如API密钥) | 分布式SSO、微服务间认证、移动端 |
| 实现复杂度 | 简单 | 较复杂(需管理密钥、过期、刷新) |

JWT组成：`Header.Payload.Signature`
- **Header**：指定签名算法(`{"alg":"HS256","typ":"JWT"}`)
- **Payload(Claims)**：存储业务数据(过期时间/用户ID/角色等)，是Base64编码而非加密，**不要存敏感信息**
- **Signature**：`HMACSHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload), secret)`，防止数据篡改
