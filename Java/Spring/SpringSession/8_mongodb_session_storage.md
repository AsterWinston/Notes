# 1. MongoDB 作为 Session 存储

## 1.1 适用场景

| 场景 | 说明 |
|------|------|
| **技术栈已包含 MongoDB** | 不希望引入额外存储组件 |
| **文档型存储需求** | Session 数据结构灵活，适合文档型数据库 |
| **高可用部署** | MongoDB 副本集天然支持故障转移 |
| **TTL 索引** | MongoDB 原生支持 TTL 索引自动过期 |

## 1.2 依赖引入

```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-mongodb</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

---

# 2. 数据模型

## 2.1 MongoDB 文档结构

Session 数据映射为 MongoDB 集合 `sessions` 中的一个文档：

```json
{
  "_id": "abc123-def456-ghi789",
  "created": "2023-11-15T10:30:00Z",
  "accessed": "2023-11-15T10:35:00Z",
  "interval": 1800,
  "expireAt": "2023-11-15T11:00:00Z",
  "principalName": "zhangsan",
  "attrs": {
    "SPRING_SECURITY_CONTEXT": "...序列化数据...",
    "username": "zhangsan"
  }
}
```

## 2.2 TTL 索引

MongoDB 通过 `expireAt` 字段 + TTL 索引实现自动过期：

```javascript
// Spring Session 自动创建
db.sessions.createIndex(
  { "expireAt": 1 },
  { expireAfterSeconds: 0 }
)
```

当文档的 `expireAt` 时间到达后，MongoDB 后台线程自动删除文档。

**TTL 处理注意事项**：
- 后台线程每 60 秒扫描一次，存在最多 60 秒的过期延迟
- TTL 索引不能用于固定集合（capped collection）
- 删除操作会占用 I/O 资源

---

# 3. 配置详解

## 3.1 application.yml

```yaml
spring:
  data:
    mongodb:
      uri: mongodb://localhost:27017/session_db
      # 或详细配置
      # host: localhost
      # port: 27017
      # database: session_db
      # username: admin
      # password: admin

  session:
    timeout: 30m
    mongodb:
      collection-name: sessions          # 集合名称
      flush-mode: on_save
      save-mode: on_set_attribute
```

## 3.2 Java 配置

```java
package com.example.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.session.data.mongo.config.annotation.web.http.EnableMongoHttpSession;

@Configuration
@EnableMongoHttpSession(
    maxInactiveIntervalInSeconds = 1800,
    collectionName = "sessions"
)
public class MongoSessionConfig {

    // 自定义 MongoDB 的 Session 序列化方式
    @Bean
    public AbstractMongoSessionConverter mongoSessionConverter() {
        return new JacksonMongoSessionConverter();
    }
}
```

## 3.3 自定义 AbstractMongoSessionConverter

```java
@Component
public class CustomMongoSessionConverter extends AbstractMongoSessionConverter {

    private final ObjectMapper objectMapper;

    public CustomMongoSessionConverter() {
        this.objectMapper = new ObjectMapper();
        this.objectMapper.registerModule(new JavaTimeModule());
        this.objectMapper.configure(
            SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);
    }

    @Override
    public Document convert(MongoSession session) {
        Document doc = new Document();
        doc.put("_id", session.getId());
        doc.put("created", session.getCreationTime());
        doc.put("accessed", session.getLastAccessedTime());
        doc.put("interval", session.getMaxInactiveInterval().getSeconds());

        // 序列化属性
        Document attrs = new Document();
        for (String name : session.getAttributeNames()) {
            attrs.put(name, session.getAttribute(name));
        }
        doc.put("attrs", attrs);

        return doc;
    }

    @Override
    public MongoSession convert(Document document) {
        // 从 Document 反序列化为 MongoSession
        return new MongoSession(
            document.getString("_id"),
            document.getDate("created").toInstant(),
            document.getDate("accessed").toInstant(),
            Duration.ofSeconds(document.getLong("interval")),
            document.get("attrs", Document.class)
        );
    }
}
```

---

# 4. MongoSessionRepository 核心实现

```java
public class MongoSessionRepository
        implements FindByIndexNameSessionRepository<MongoSession> {

    private final MongoOperations mongoOperations;
    private final AbstractMongoSessionConverter converter;

    @Override
    public MongoSession createSession() {
        MongoSession session = new MongoSession();
        session.setMaxInactiveInterval(defaultMaxInactiveInterval);
        return session;
    }

    @Override
    public void save(MongoSession session) {
        Document doc = converter.convert(session);
        doc.put("expireAt",
            Date.from(Instant.now().plus(session.getMaxInactiveInterval())));

        // Upsert 操作
        mongoOperations.save(doc, collectionName);
    }

    @Override
    public MongoSession findById(String id) {
        Document doc = mongoOperations.findById(id, Document.class, collectionName);
        if (doc == null) return null;

        MongoSession session = converter.convert(doc);
        if (session.isExpired()) {
            deleteById(id);
            return null;
        }
        return session;
    }

    @Override
    public void deleteById(String id) {
        mongoOperations.remove(
            Query.query(Criteria.where("_id").is(id)), collectionName);
    }

    @Override
    public Map<String, MongoSession> findByIndexNameAndIndexValue(
            String indexName, String indexValue) {
        // 通过 principalName 字段查询
        Query query = Query.query(Criteria.where("principalName").is(indexValue));
        List<Document> docs = mongoOperations.find(query, Document.class,
            collectionName);

        Map<String, MongoSession> result = new HashMap<>();
        for (Document doc : docs) {
            MongoSession session = converter.convert(doc);
            result.put(session.getId(), session);
        }
        return result;
    }
}
```

---

# 5. MongoDB 副本集与分片

## 5.1 副本集连接

```yaml
spring:
  data:
    mongodb:
      uri: mongodb://host1:27017,host2:27017,host3:27017/session_db?replicaSet=rs0
```

## 5.2 分片集群

对于大规模 Session 存储，可以使用 MongoDB 分片：

```javascript
// 按 Session ID 分片
sh.shardCollection("session_db.sessions", { "_id": 1 })
```

## 5.3 读写关注

```yaml
spring:
  data:
    mongodb:
      uri: >-
        mongodb://host1:27017/session_db?
        w=majority&
        wtimeoutMS=5000&
        readPreference=primaryPreferred
```

| 参数 | 说明 |
|------|------|
| `w=majority` | 写入确认：多节点确认后才返回 |
| `readPreference=primaryPreferred` | 优先从主节点读，主节点不可用时读从节点 |
| `readConcern=majority` | 读关注：保证读到的数据已被多数节点确认 |

---

# 6. MongoDB vs Redis 对比

| 维度 | MongoDB | Redis |
|------|---------|-------|
| **存储介质** | 磁盘 + 内存（WiredTiger） | 纯内存 |
| **数据持久化** | 天然持久化 | RDB/AOF 持久化 |
| **查询能力** | 丰富的查询语法 | 仅支持 Key 查询 |
| **过期处理** | TTL 索引（最多 60s 延迟） | TTL Key（即时） |
| **扩展性** | 分片集群 | Redis Cluster / 哨兵 |
| **性能** | 读 ms 级（热数据在内存） | 读 μs 级 |
| **数据大小** | 单文档最大 16MB | 无硬性限制（受内存制约） |
| **适用场景** | Session 需要复杂查询 | Session 高频读写 |
