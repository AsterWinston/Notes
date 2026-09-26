
# 1. 消息过滤概述

RocketMQ 支持在 Consumer 端（也支持在 Broker 端）按规则过滤消息，减少不必要的数据传输和消费端处理开销。

## 1.1 过滤方式对比

| 方式 | 过滤位置 | 语法 | 准确性 | 性能影响 |
|------|----------|------|--------|----------|
| **Tag 过滤** | Broker 端（ConsumeQueue Tag Hash） | `tagA \|\| tagB` | 哈希冲突可能误匹配 | 低开销 |
| **SQL92 过滤** | Broker 端（BloomFilter + 属性过滤） | `a > 5 AND b = 'x'` | 精确 | 中开销 |
| **类过滤** | Consumer 端 | 自定义 Java Filter | 最精确 | 高开销（全量接收） |

# 2. Tag 过滤

## 2.1 基本原理

```
消息存储时：
  消息的 Tag HashCode 存储在 ConsumeQueue 条目中 (20B 中的后 8B)

消费过滤时：
  1. Broker 从 ConsumeQueue 读取 entries → 比对 Tag HashCode
  2. 匹配 → 读取 CommitLog → 返回给 Consumer
  3. 不匹配 → 跳过（不读 CommitLog）

注意: Tag 过滤基于 HashCode，存在哈希冲突 → Consumer 端会二次过滤
```

## 2.2 Producer 端设置 Tag

```java
// 单 Tag
Message msg = new Message("TopicTest", "TagA", "body".getBytes());

// 消息属性中也可以携带过滤标记
msg.putUserProperty("level", "1");
```

## 2.3 Consumer 端使用 Tag 过滤

```java
// 订阅表达式
consumer.subscribe("TopicTest", "TagA");           // 只消费 TagA
consumer.subscribe("TopicTest", "TagA || TagB");   // TagA 或 TagB
consumer.subscribe("TopicTest", "*");              // 全部 Tag
consumer.subscribe("TopicTest", "TagA||*");        // 等同于 "*"
```

## 2.4 Tag 过滤源码流程

```java
// ExpressionMessageFilter.java — Broker 端过滤
public boolean isMatchedByConsumeQueue(Long tagsCode, ConsumeQueueExt.CqExtUnit cqExtUnit) {
    if (SubscriptionData.isNullTag() || subscriptionData.isSubSetEmpty()) {
        return true;
    }
    // 逐个比对订阅的 Tag HashCode
    for (String tag : subscriptionData.getTagsSet()) {
        if (tagsCode.equals(Long.valueOf(tag.hashCode()))) {
            return true;
        }
    }
    return false;
}

// 消费者端二次过滤 (防哈希冲突)
public boolean isMatchedByCommitLog(ByteBuffer msgBuffer, Map<String, String> properties) {
    // 从消息体中解析实际 Tag
    String realTag = properties.get(MessageConst.PROPERTY_TAGS);
    // 精确比对
    return subscriptionData.getTagsSet().contains(realTag);
}
```

# 3. SQL92 过滤

## 3.1 启用 SQL92 过滤

```properties
# Broker 配置开启 SQL92 过滤支持
enablePropertyFilter = true
```

## 3.2 Producer 设置属性

```java
Message msg = new Message("TopicTest", "TagA", "body".getBytes());
msg.putUserProperty("region", "beijing");
msg.putUserProperty("level", "3");
msg.putUserProperty("price", "100.5");
```

## 3.3 Consumer 使用 SQL92 过滤

```java
consumer.subscribe("TopicTest", 
    MessageSelector.bySql("region = 'beijing' AND level > 2"));

// 支持的操作符：
// 比较: =, <>, >, >=, <, <=
// 逻辑: AND, OR, NOT
// 类型: IS NULL, IN, BETWEEN
```

## 3.4 SQL92 过滤原理

```java
// Broker 端执行 SQL92 过滤
// ExpressionFilter 类

// 1. 解析 SQL 表达式 → 语法树
// 2. 从消息属性中提取字段值
// Build FilterData:
//    - properties (Map)
//    - tagsCode
//    - bloomFilterData
// 3. 执行表达式计算
// 4. 匹配 → 返回消息，不匹配 → 跳过
```

## 3.5 SQL92 过滤局限性

- 需要 Broker 端 `enablePropertyFilter=true`（默认关闭，因为额外开销）
- 仅支持消息属性过滤，不支持消息体内容过滤
- 复杂表达式可能影响 Broker 性能
- 属性值限定为数字或字符串

# 4. 类过滤（Server 端 Filter Class）

## 4.1 使用方式

```java
// 1. 定义过滤类（需要在 Broker 的 classpath 中）
public class MyMessageFilter implements MessageFilter {
    @Override
    public boolean match(MessageExt msg, FilterContext context) {
        String body = new String(msg.getBody());
        return body.contains("important");
    }
}

// 2. Consumer 指定过滤类
consumer.subscribe("TopicTest", "com.example.MyMessageFilter");
```

## 4.2 注意

- Filter 类需要部署在所有 Broker 上（运维成本高）
- 不推荐在生产环境使用，已被 SQL92 方案替代
- RocketMQ 5.x 已移除 FilterServer 组件

# 5. Filter 机制架构

```
Broker 端过滤流程：

Consumer 拉取请求
    │
    ▼
PullMessageProcessor.processRequest()
    │
    ├── 1. 获取消费组订阅关系 SubscriptionData
    │      (订阅的 Topic, Tag, SQL 等)
    │
    ├── 2. 获取 ConsumeQueue 数据
    │
    ├── 3. Broker 端过滤 (Tag Hash / SQL92)
    │      ├── 不匹配 → 跳过
    │      └── 匹配 → 继续
    │
    ├── 4. 读取 CommitLog 消息内容
    │
    ├── 5. Consumer 端二次过滤 (防 Tag Hash 冲突)
    │
    └── 6. 返回过滤后的消息
```

# 6. 最佳实践

1. **Tag 数量限制**：每个 Topic 建议 Tag 数量 < 100，Tag 过多影响 ConsumeQueue
2. **Tag 命名规范**：使用业务含义明确的短命名，如 `ORDER_PAID`, `ORDER_CLOSED`
3. **SQL92 前开启 enablePropertyFilter**：否则退化为 Tag 过滤
4. **避免全量接收 + 消费端过滤**：浪费带宽和消费资源
5. **属性名使用简短字符串**：减少存储和传输开销
6. **Tag 规划设计**：先规划好 Tag 体系再上线，避免频繁新增

# 7. 参考

- [RocketMQ Filter 文档](https://rocketmq.apache.org/docs/4.x/producer/04message2/)
