
# 1. 环境要求

| 组件 | 版本要求 |
|------|----------|
| JDK | 1.8+ |
| Maven | 3.2+ |
| OS | Linux / macOS / Windows |
| 内存 | 最低 2G，推荐 4G+ |

# 2. 单机部署（源码构建）

## 2.1 下载源码

```bash
git clone https://github.com/apache/rocketmq.git
cd rocketmq

# 切换分支（以 4.9.x 为例）
git checkout release-4.9.4

# Maven 编译，跳过测试
mvn -Prelease-all -DskipTests clean install -U
```

## 2.2 修改配置

```bash
# 修改 Broker 启动内存
vi distribution/target/rocketmq-4.9.4/rocketmq-4.9.4/bin/runbroker.sh
# JAVA_OPT="${JAVA_OPT} -server -Xms1g -Xmx1g -Xmn512m"

# 修改 NameServer 启动内存
vi distribution/target/rocketmq-4.9.4/rocketmq-4.9.4/bin/runserver.sh
# JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx256m -Xmn128m"
```

## 2.3 启动服务

```bash
# 1. 启动 NameServer
nohup sh bin/mqnamesrv &
tail -f ~/logs/rocketmqlogs/namesrv.log
# 出现 "The Name Server boot success." 表示启动成功

# 2. 启动 Broker
nohup sh bin/mqbroker -n localhost:9876 &
tail -f ~/logs/rocketmqlogs/broker.log
# 出现 "The broker[...] boot success." 表示启动成功

# 3. 验证
jps
# NamesrvStartup
# BrokerStartup
```

## 2.4 关闭服务

```bash
sh bin/mqshutdown broker
sh bin/mqshutdown namesrv
```

# 3. Docker 部署（推荐测试用）

```yaml
version: '3'
services:
  namesrv:
    image: apache/rocketmq:4.9.4
    container_name: rmqnamesrv
    command: sh mqnamesrv
    ports:
      - "9876:9876"

  broker:
    image: apache/rocketmq:4.9.4
    container_name: rmqbroker
    depends_on:
      - namesrv
    ports:
      - "10909:10909"
      - "10911:10911"
    command: sh mqbroker -n namesrv:9876 -c /home/rocketmq/conf/broker.conf
    volumes:
      - ./broker.conf:/home/rocketmq/conf/broker.conf

  dashboard:
    image: apacherocketmq/rocketmq-dashboard:latest
    container_name: rmqdashboard
    depends_on:
      - namesrv
    ports:
      - "8080:8080"
    environment:
      JAVA_OPTS: -Drocketmq.namesrv.addr=namesrv:9876
```

## 3.1 broker.conf

```properties
brokerClusterName = DefaultCluster
brokerName = broker-a
brokerId = 0
deleteWhen = 04
fileReservedTime = 48
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH
brokerIP1 = 宿主机的IP
autoCreateTopicEnable = true
```

```bash
docker-compose up -d
```

# 4. 第一个示例：Hello RocketMQ

## 4.1 Maven 依赖

```xml
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-client</artifactId>
    <version>4.9.4</version>
</dependency>
```

## 4.2 Producer 发送消息

```java
import org.apache.rocketmq.client.producer.DefaultMQProducer;
import org.apache.rocketmq.client.producer.SendResult;
import org.apache.rocketmq.common.message.Message;

public class SyncProducer {

    public static void main(String[] args) throws Exception {
        // 1. 创建 Producer，指定 Group 名称
        DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
        // 2. 指定 NameServer 地址
        producer.setNamesrvAddr("127.0.0.1:9876");
        // 3. 启动 Producer
        producer.start();

        for (int i = 0; i < 100; i++) {
            // 4. 创建消息：Topic + Tag + Body
            Message msg = new Message(
                "TopicTest",                // Topic
                "TagA",                     // Tag
                ("Hello RocketMQ " + i).getBytes()  // Body
            );
            // 5. 同步发送
            SendResult result = producer.send(msg);
            System.out.printf("发送结果: %s%n", result);
        }

        // 6. 关闭 Producer
        producer.shutdown();
    }
}
```

## 4.3 Consumer 消费消息

```java
import org.apache.rocketmq.client.consumer.DefaultMQPushConsumer;
import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyContext;
import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyStatus;
import org.apache.rocketmq.client.consumer.listener.MessageListenerConcurrently;
import org.apache.rocketmq.common.message.MessageExt;

import java.util.List;

public class SyncConsumer {

    public static void main(String[] args) throws Exception {
        // 1. 创建 Consumer，指定 Group
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("please_rename_unique_group_name");
        // 2. 指定 NameServer
        consumer.setNamesrvAddr("127.0.0.1:9876");
        // 3. 订阅 Topic 和 Tag
        consumer.subscribe("TopicTest", "*");
        // 4. 注册消息监听器
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(
                    List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                for (MessageExt msg : msgs) {
                    System.out.printf("收到消息: %s, Topic: %s, Tag: %s%n",
                        new String(msg.getBody()), msg.getTopic(), msg.getTags());
                }
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        // 5. 启动 Consumer
        consumer.start();
        System.out.println("Consumer 启动成功");
    }
}
```

## 4.4 运行结果

```
# Producer 输出：
发送结果: SendResult [sendStatus=SEND_OK, msgId=..., offsetMsgId=..., messageQueue=..., queueOffset=0]

# Consumer 输出：
收到消息: Hello RocketMQ 0, Topic: TopicTest, Tag: TagA
收到消息: Hello RocketMQ 1, Topic: TopicTest, Tag: TagA
...
```

# 5. RocketMQ Dashboard（管理控制台）

**访问**: `http://localhost:8080`

关键功能：
- **运维**：查看集群状态、Broker 状态、NameServer 信息
- **驾驶舱**：查看 Topic 消息量、写入 TPS
- **消息**：按 Topic/MessageId/Key 查询消息
- **消费者**：查看 ConsumerGroup、消费延迟

# 6. 常见启动问题

| 问题 | 原因 | 解决 |
|------|------|------|
| `No route info of this topic` | 未配置 `autoCreateTopicEnable` | Broker 配置开启自动创建 |
| `connect to <ip:9876> failed` | NameServer 未启动 | 先启动 NameServer |
| `NoSuchMethodError` | rockermq-client 版本与 Broker 不一致 | 对齐版本 |
| `Not enough memory` | JVM 内存不足 | 降低 `runbroker.sh` 中 Xms/Xmx |

# 7. 参考

- [RocketMQ Quick Start](https://rocketmq.apache.org/docs/quick-start/)
