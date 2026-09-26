# 1. 概述与动机

结论先行：向量记忆（Vector Memory）擅长"语义相似召回"，却不擅长"关系推理"；知识图谱记忆（Knowledge Graph Memory）用实体-关系（Entity-Relation）结构补足这一短板，两者结合才能覆盖完整记忆需求。

## 1.1 向量记忆的能力边界

### 1.1.1 相似召回与关系推理的差异

向量记忆把文本编码为稠密向量（Dense Vector），通过余弦相似度（Cosine Similarity）召回语义相近的片段。它天然适合"找相关内容"，却无法回答"张三的直属上级是谁"这类需要沿关系链精确查找的问题。

```
问："用户的直属上级是谁？"
向量记忆：召回含"上级"字样的片段（可能不完整、可能张冠李戴）
知识图谱：(用户) -[汇报给]-> (张经理)   ← 直接可查，结果确定
```

### 1.1.2 典型失效场景

| 场景 | 向量记忆表现 | 原因 |
|------|:------------:|------|
| 相似召回 | 优 | 语义匹配是天然优势 |
| 关系查询 | 差 | 关系未被显式建模 |
| 多跳推理 | 差 | 需跨片段组合，易断链 |
| 结构化更新 | 中 | 旧片段仍被召回，无法失效 |
| 聚合统计 | 差 | 无法按字段过滤与求和 |

## 1.2 知识图谱记忆的定位

### 1.2.1 能力对比

| 能力 | 向量记忆 | 知识图谱 |
|------|:--------:|:--------:|
| 相似召回 | 是 | 一般 |
| 关系查询 | 否 | 是 |
| 多跳推理 | 否 | 是 |
| 结构化更新 | 一般 | 是 |
| 可解释性 | 低 | 高 |

### 1.2.2 适用与不适用

- 适用：用户画像、组织关系、稳定偏好、可验证的长期事实。
- 不适用：大段叙述、情感化对话、需要模糊语义匹配的开放内容。
- 结论：图谱不是替代向量，而是与向量互补，构成"结构化 + 语义"双通道记忆。

## 1.3 本文范围

本文覆盖记忆图谱的模型、抽取、存储、查询、Java 集成、与向量混合、时效更新、规模控制与生产实践，均以 JDK 17/21 与 Spring AI、LangChain4j 风格示例展开。

# 2. 记忆图谱模型

结论先行：记忆图谱由节点（Node，实体）与边（Edge，关系）组成；边带方向、类型、置信度与时效，从而支持精确、可解释、可回溯的关系查询。

## 2.1 节点与边

### 2.1.1 数据模型

```java
public record MemoryNode(
        String id,
        String label,                 // 实体类型：User / Company / Preference
        String name,
        Map<String, Object> properties
) {}

public record MemoryEdge(
        String fromId,
        String relation,              // 关系类型
        String toId,
        double confidence,
        Instant validFrom,
        Instant validTo              // 时效（可选，null 表示长期有效）
) {}
```

### 2.1.2 时效语义

- `validFrom`：关系生效时间，可为抽取时间或对话中提及的时间。
- `validTo`：关系失效时间，`null` 表示仍然有效。
- 判定"当前有效"的统一条件：`validTo == null || validTo > now`。
- 进阶可用双时态（Bi-temporal）：区分"事件发生时间"与"系统记录时间"，便于审计与回溯。

## 2.2 图谱示例

### 2.2.1 用户记忆图

```
(用户:张三) ──[职位]──▶ (后端工程师)
     │
     ├──[偏好]──▶ (简洁回答)
     ├──[所在公司]──▶ (Acme)
     │                    │
     │                    └──[技术栈]──▶ (Java 21)
     └──[汇报给]──▶ (李经理)
```

## 2.3 建模原则

### 2.3.1 实体归一化

同一实体可能以多种写法出现（"张三"、"张先生"、"Zhang San"）。写入前需归一化（Normalization）：统一大小写、去除称谓、同义词映射到规范名，避免图中出现重复节点。

### 2.3.2 关系白名单

并非所有关系都值得入图。只保留高价值关系类型，可显著抑制图膨胀并降低注入风险。白名单在抽取后、写入前执行。

## 2.4 边界与坑

- 关系方向易错：`A 汇报给 B` 与 `B 管理 A` 语义等价但方向相反，需固定约定。
- 属性爆炸：把大量临时字段塞进节点属性，会让图退化为宽表。
- 缺省时效：若不支持 `validTo`，历史关系会与新关系同时命中。

# 3. 构建记忆图谱

结论先行：构建分两步——先用大模型从对话中抽取三元组（Triple），再经校验与归一化写入图数据库；抽取质量与写入安全是成败关键。

## 3.1 从对话抽取三元组

### 3.1.1 抽取接口

```java
public record Triple(String subject, String subjectType,
                     String relation, String object, String objectType) {}

interface MemoryGraphExtractor {

    @UserMessage("""
            从对话中抽取实体与关系，构建记忆图谱。

            对话：
            {{conversation}}

            输出 JSON 数组：
            [{"subject":"张三","subjectType":"User",
              "relation":"职位","object":"后端工程师","objectType":"Role"}]
            """)
    List<Triple> extract(String conversation);
}
```

### 3.1.2 抽取提示词设计

- 显式给出关系白名单，减少模型自由发挥。
- 要求输出严格 JSON，并附示例（Few-shot），提升可解析率。
- 对"不确定"的关系允许模型省略，而非编造。

### 3.1.3 抽取后校验

- JSON 反序列化失败即丢弃该批次并告警。
- 关系不在白名单内则丢弃。
- 主语/宾语为空或超长则丢弃。

## 3.2 写入图

### 3.2.1 upsert 流程

```java
public void buildFrom(UserContext user, String conversation) {
    List<Triple> triples = extractor.extract(conversation);

    for (Triple t : triples) {
        if (!shouldStore(t.relation())) {
            continue;                                  // 白名单过滤
        }
        String subjectId = graph.upsertNode(t.subject(), t.subjectType());
        String objectId = graph.upsertNode(t.object(), t.objectType());

        graph.upsertEdge(subjectId, t.relation(), objectId, 0.9);
    }
}
```

### 3.2.2 去重与冲突

`upsertNode` 以（label, 归一化 name）为唯一键，命中则复用节点；`upsertEdge` 以（from, relation, to）为唯一键，命中则更新置信度与时间戳，避免重复边。

## 3.3 边界与坑

- 同一次对话抽取到矛盾关系（既"在职"又"离职"）时，需以时间较晚者为准。
- 批量写入应放在单个事务（Transaction）中，避免半写状态。
- 抽取是额外 LLM 调用，需控制频率，可对短对话合并后批量抽取。

## 3.4 生产建议

- 抽取与写入解耦，走消息队列异步执行。
- 对抽取结果做抽样人工复核，监控准确率。
- 记录每次写入的来源对话 ID，便于溯源与回滚。

# 4. 存储选型

结论先行：中小规模优先 Neo4j，超大规模选 NebulaGraph；若已有 PostgreSQL 且规模小，可用递归 CTE（Common Table Expression，公用表表达式）临时替代，避免引入新组件。

## 4.1 图数据库对比

| 存储 | 说明 |
|------|------|
| Neo4j | 最成熟，Cypher 查询，生态完善 |
| NebulaGraph | 分布式，适合超大规模图 |
| JanusGraph | 可扩展，需外部存储后端 |
| PostgreSQL + 递归 CTE | 小规模可用，运维成本低 |

## 4.2 小规模替代方案

用一张边表 `memory_edge(from_id, relation, to_id, valid_to)`，配合递归 CTE 实现多跳：

```sql
WITH RECURSIVE reach(id, depth) AS (
    SELECT to_id, 1 FROM memory_edge
    WHERE from_id = :userId AND relation = '所在公司'
    UNION ALL
    SELECT e.to_id, r.depth + 1 FROM memory_edge e
    JOIN reach r ON e.from_id = r.id
    WHERE r.depth < 3
)
SELECT * FROM reach;
```

## 4.3 Cypher 基础

```cypher
// Neo4j：创建节点与关系
MERGE (u:User {name: '张三'})
MERGE (r:Role {name: '后端工程师'})
MERGE (u)-[:职位]->(r)
```

## 4.4 边界与坑

- 递归 CTE 在深链与高并发下性能骤降，深度需设上限。
- 图数据库的索引需建在查询入口字段（如 `User.id`）上，否则全表扫描。
- 图库与业务库混部会争抢资源，导致在线查询抖动。

# 5. 关系查询

结论先行：图查询的核心是"从入口节点出发，沿指定关系走一跳或多跳，并按时效过滤"，配合索引即可获得毫秒级响应。

## 5.1 单跳与多跳

### 5.1.1 属性查询

```cypher
// 查询用户的所有属性
MATCH (u:User {name: $name})-[r]->(o)
RETURN type(r) AS relation, o.name AS value
```

### 5.1.2 多跳查询

```cypher
// 多跳查询：用户所在公司的技术栈
MATCH (u:User {name: $name})-[:所在公司]->(c)-[:技术栈]->(t)
RETURN t.name
```

## 5.2 时效过滤

```cypher
// 带时效：只取有效的
MATCH (u:User {name: $name})-[r]->(o)
WHERE r.validTo IS NULL OR r.validTo > datetime()
RETURN type(r), o.name
```

## 5.3 查询优化

- 为 `User.name`、`User.id` 建唯一约束与索引。
- 限制跳数（一般不超过 3），防止笛卡尔积爆炸。
- 结果设 `LIMIT`，避免一次拉回整张子图。

## 5.4 边界与坑

- 多跳若无方向约束，可能命中反向路径导致错误结果。
- 循环关系（A→B→A）会让多跳查询陷入重复遍历。
- 变量名与参数占位符混用（如 `$name` 与字面量）易造成查询缓存失效。

# 6. Java 集成

结论先行：用官方 Driver 封装一个 `GraphMemoryStore`，对外只暴露查询与写入两类方法；所有动态拼接的关系类型必须经白名单校验，防止 Cypher 注入。

## 6.1 GraphMemoryStore 实现

### 6.1.1 查询

```java
@Component
@RequiredArgsConstructor
public class GraphMemoryStore {

    private final Driver neo4jDriver;

    public List<MemoryRelation> queryRelations(String userId) {
        try (Session session = neo4jDriver.session()) {
            return session.executeRead(tx -> {
                Result result = tx.run("""
                        MATCH (u:User {id: $userId})-[r]->(o)
                        WHERE r.validTo IS NULL OR r.validTo > datetime()
                        RETURN type(r) AS relation, o.name AS value
                        """, Map.of("userId", userId));

                return result.stream()
                        .map(record -> new MemoryRelation(
                                record.get("relation").asString(),
                                record.get("value").asString()))
                        .toList();
            });
        }
    }
```

### 6.1.2 写入

```java
    public void upsert(String userId, String relation, String value) {
        try (Session session = neo4jDriver.session()) {
            session.executeWrite(tx -> tx.run("""
                    MERGE (u:User {id: $userId})
                    MERGE (o:MemoryValue {name: $value})
                    MERGE (u)-[r:%s]->(o)
                    SET r.updatedAt = datetime()
                    """.formatted(sanitizeRelation(relation)),
                    Map.of("userId", userId, "value", value)));
        }
    }
}
```

## 6.2 注入安全

### 6.2.1 白名单校验

```java
private static final Set<String> ALLOWED_RELATIONS = Set.of(
        "职位", "所在公司", "技术栈", "偏好", "汇报给");

private String sanitizeRelation(String relation) {
    if (!ALLOWED_RELATIONS.contains(relation)) {
        throw new IllegalArgumentException("非法的关系类型：" + relation);
    }
    return relation;
}
```

> `relation` 不能直接拼接进 Cypher，需**白名单校验**防注入。参数化查询只能保护"值"，保护不了被当作"模式"（Pattern）拼接的标识符。

## 6.3 边界与坑

- 忘记关闭 `Session` 会耗尽连接池，务必用 try-with-resources。
- 读写应分别使用 `executeRead` / `executeWrite`，以利用集群路由。

## 6.4 生产建议

- 加超时与重试，图库抖动时降级为仅用向量记忆。
- 对写入方法加指标埋点，监控每秒写入与失败率。

# 7. 与向量检索结合

结论先行：关系查询与语义召回各有所长，混合召回（Hybrid Recall）先取图谱的确定关系，再补向量的相关历史，可同时兼顾精确与覆盖。

## 7.1 混合召回流程

```
混合召回：
① 向量召回：语义相关的记忆片段
② 图查询：相关实体的关系
③ 合并注入上下文
```

## 7.2 上下文组装

### 7.2.1 buildMemoryContext

```java
public String buildMemoryContext(UserContext user, String question) {
    StringBuilder sb = new StringBuilder();

    // ① 图：用户属性
    List<MemoryRelation> relations = graphStore.queryRelations(user.userId());
    if (!relations.isEmpty()) {
        sb.append("【用户信息】\n")
          .append(relations.stream()
                  .map(r -> "%s：%s".formatted(r.relation(), r.value()))
                  .collect(Collectors.joining("\n")))
          .append("\n\n");
    }

    // ② 向量：相关历史
    List<MemoryEntry> memories = vectorStore.recall(user, question, 5);
    if (!memories.isEmpty()) {
        sb.append("【相关历史】\n")
          .append(formatMemories(memories));
    }
    return sb.toString();
}
```

## 7.3 边界与坑

- 两路结果可能冲突（图谱说"在职"，向量说"已离职"），需以时效较新者为准。
- 合并后 Token 可能超限，需在最后统一裁剪。

## 7.4 生产建议

- 对图谱与向量分别统计命中率，定位召回瓶颈。
- 给图谱块加固定前缀（如"【用户信息】"），便于模型区分来源。

# 8. 时效与更新

结论先行：关系会随时间变化，更新应采用"软失效 + 新边"策略，保留历史以便审计与回溯，而非直接删除。

## 8.1 更新策略

### 8.1.1 软失效

```java
public void updateRelation(String userId, String relation, String newValue) {
    // ① 旧关系标记失效
    graphStore.expireRelation(userId, relation, Instant.now());

    // ② 建立新关系
    graphStore.upsert(userId, relation, newValue);
}
```

### 8.1.2 硬覆盖与版本化

| 策略 | 说明 | 适用 |
|------|------|------|
| 软失效 | `validTo` 标记，保留历史 | 大多数场景 |
| 硬覆盖 | 直接替换 | 无需历史 |
| 版本化 | 保留多版本 | 审计要求高 |

## 8.2 冲突消解

### 8.2.1 以时间为准

同一（subject, relation）出现多个 object 时，取 `validFrom` 最新且当前有效的边作为答案。

## 8.3 边界与坑

- 批量软失效需与写入同事务，否则可能出现"双有效"或"双失效"。
- 时钟不一致（多实例）会导致 `validFrom` 乱序，建议统一用数据库时间。
- 只更新边而不更新节点属性，会造成"边新值旧"的不一致。

## 8.4 生产建议

- 更新走统一入口，禁止多处代码直接改图。
- 为每次变更记录审计日志（操作者、时间、旧值、新值）。
- 定期校验是否存在同一关系的多条有效边，发现即修复。

# 9. 规模控制

结论先行：图的价值随规模上升而稀释，必须用白名单、上限与定期清理三道闸门，把图约束在"高价值、可维护"的范围内。

## 9.1 关系白名单

```java
// 只保留重要关系，避免图膨胀
private static final Set<String> IMPORTANT_RELATIONS = Set.of(
        "职位", "所在公司", "技术栈", "偏好", "汇报给");

public boolean shouldStore(String relation) {
    return IMPORTANT_RELATIONS.contains(relation);
}
```

## 9.2 清理与归档

- 定期清理长期 `validTo` 已过期的边。
- 对低频访问的子图做冷归档，需要时再恢复。

## 9.3 边界与坑

- 白名单过窄会漏掉有价值的新关系，需定期评审扩充。
- 清理任务需幂等，避免重复执行造成误删。
- 度上限过严会丢失真实偏好，需按关系类型分别设置。

## 9.4 生产建议

- 把规模阈值写入配置中心，随业务增长动态调整。
- 清理任务在低峰期执行，避免影响在线查询。
- 归档数据可保留在对象存储，需要时再回灌热库。

# 10. 生产注意事项

结论先行：图记忆的风险集中在"注入、膨胀、时效、重复、质量、隔离"六类，逐项都有成熟的工程对策。

## 10.1 风险与对策

| 风险 | 建议 |
|------|------|
| 图无限膨胀 | 白名单关系 + 定期清理 |
| 关系注入 | 白名单校验，禁止拼接 |
| 无时效 | 支持 `validTo` 与时间过滤 |
| 实体重复 | 实体归一化（同义词映射） |
| 抽取质量差 | 用强模型 + 结构校验 |
| 无隔离 | 按用户 / 租户隔离命名空间 |

## 10.2 监控指标

| 指标 | 含义 |
|------|------|
| 节点数 / 边数 | 图规模趋势 |
| 抽取成功率 | 抽取链路健康度 |
| 查询 P99 延迟 | 图库性能 |
| 写入失败率 | 存储稳定性 |
| 无效边占比 | 时效清理效果 |

## 10.3 边界与坑

- 多租户若共用图而未隔离，会串数据且难以清理。
- 监控缺失时，图膨胀往往在数月后才被发现。

## 10.4 生产建议

- 上线前做容量评估，设定图规模告警阈值。
- 关键关系变更保留操作日志，支持回溯。

# 11. 小结

结论先行：知识图谱记忆是向量记忆的结构化补充，核心在于"实体-关系-时效"的建模与安全写入。

## 11.1 要点回顾

- 知识图谱记忆擅长关系查询与多跳推理。
- 模型：节点（实体）+ 边（关系），支持时效。
- 从对话抽取三元组构建，用图数据库存储。
- 与向量检索结合，兼顾关系与语义。
- 关系类型需白名单，防注入与图膨胀。

## 11.2 落地清单

- 定义关系白名单与实体归一化规则。
- 实现抽取、校验、写入三段式流水线。
- 为查询入口建索引，限制跳数与结果数。
- 更新采用软失效，保留历史。
- 建立规模、延迟、失败率监控。
