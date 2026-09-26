# 1. 元数据的三大作用

```
① 过滤   按租户/权限/时效筛选检索范围
② 引用   回答时展示来源，提升可信度
③ 治理   版本管理、失效清理、审计
```

> 元数据必须在**入库前**设计好，事后补需要全量重建索引。

## 1.1 过滤

过滤（Filtering）是元数据最核心的作用。它把"向量相似度"这一单一维度，扩展为"相似度 + 结构化条件"的联合检索，从而保证：

- **隔离**：用户只能检索到有权访问的文档；
- **时效**：只命中当前生效的版本；
- **范围**：限定文档类型、来源、语言等。

## 1.2 引用

回答时附带来源（标题、章节、URL），让用户可核实，显著提升信任度。引用信息全部来自元数据，因此元数据的完整度直接决定回答的可信度。

## 1.3 治理

元数据支撑知识库的长期运营：

- **版本管理**：同一文档的多次修订可追溯；
- **失效清理**：按过期时间批量下架；
- **审计**：记录文档来源、变更时间、操作人。

## 1.4 为什么必须前置设计

元数据一旦入库后才发现缺失，往往需要**全量重建索引**，代价极高：

| 时机 | 代价 | 说明 |
|------|------|------|
| 入库前设计 | 低 | 只影响写入逻辑 |
| 入库后补充 | 高 | 需重跑分块 + 重新嵌入 + 重建索引 |
| 上线后补充 | 极高 | 需停机或双写迁移 |

> 结论：把"元数据 schema"当作数据库表结构一样，在项目初期就冻结核心字段。

# 2. 元数据分类

## 2.1 分类总览

| 类别 | 字段示例 | 作用 |
|------|----------|------|
| 来源 | `source`, `url`, `docId` | 引用、溯源 |
| 定位 | `title`, `section`, `page` | 展示、定位 |
| 权限 | `tenantId`, `department`, `acl` | 隔离、过滤 |
| 时效 | `createdAt`, `updatedAt`, `effectiveDate` | 时效管理 |
| 治理 | `version`, `status`, `docType` | 版本、分类 |
| 内容 | `language`, `keywords`, `summary` | 检索辅助 |

## 2.2 字段设计示例

```java
public record DocMetadata(
        String docId,          // 来源：唯一标识
        String source,         // 来源：原始路径/URL
        String title,          // 定位：标题
        String section,        // 定位：章节
        int page,              // 定位：页码
        String tenantId,       // 权限：租户
        String department,     // 权限：部门
        List<String> acl,      // 权限：允许的角色
        String createdAt,      // 时效：创建时间（ISO 8601）
        String effectiveDate,  // 时效：生效时间
        String expireDate,     // 时效：失效时间
        String version,        // 治理：版本
        String status,         // 治理：DRAFT/PUBLISHED
        String docType,        // 治理：policy/manual/...
        String language,       // 内容：语言
        List<String> keywords  // 内容：关键词
) {}
```

## 2.3 必填与可选

| 字段 | 必填 | 理由 |
|------|:----:|------|
| docId | 是 | 去重与引用定位 |
| source | 是 | 引用可追溯 |
| tenantId | 是 | 权限隔离 |
| status | 是 | 控制是否可检索 |
| effectiveDate | 政策类必填 | 时效管理 |
| keywords | 否 | 增强项，可后补 |
| summary | 否 | 增强项，可后补 |

> 必填字段应在**入库管道**中强校验，缺字段直接拒绝写入。

## 2.4 元数据的生命周期

元数据不是写入即结束，它随文档一起经历完整的生命周期：

```
创建 → 入库 → 检索 → 更新 → 失效 → 清理
```

| 阶段 | 关键动作 | 常见问题 |
|------|----------|----------|
| 创建 | 采集来源、定位信息 | 来源缺失 |
| 入库 | 补全权限、时效、治理字段 | 字段未强校验 |
| 检索 | 注入过滤条件 | 越权、命中过期 |
| 更新 | 升版本、改生效日期 | 旧版本未失效 |
| 失效 | 标记 status 或 expireDate | 未批量处理 |
| 清理 | 物理删除与审计留存 | 误删、审计缺失 |

> 设计时应把"失效"与"清理"当作一等公民：一个只增不减的知识库，最终会被过期内容淹没。

# 3. 权限过滤（最关键）

## 3.1 元数据过滤方案

```java
// Spring AI
SearchRequest.builder()
        .query(question)
        .filterExpression("tenantId == 'acme' && department == 'HR'")
        .topK(5)
        .build();
```

```java
// LangChain4j
EmbeddingStoreContentRetriever.builder()
        .embeddingStore(store)
        .embeddingModel(embeddingModel)
        .dynamicFilter(query -> metadataKey("tenantId")
                .isEqualTo(TenantContext.currentTenant()))
        .build();
```

## 3.2 隔离方案对比

| 方案 | 隔离强度 | 运维成本 | 适用 |
|------|:--------:|:--------:|------|
| 同集合 + 元数据过滤 | 中 | 低 | 多数场景 |
| 每租户独立集合/表 | 高 | 高 | 大租户、强合规 |
| 每租户独立实例 | 最高 | 最高 | 金融/政务 |

> 无论哪种，都必须写**越权检索测试**：用户 A 不能检索到用户 B 的内容。

## 3.3 复杂权限模型

```java
// ACL 列表：文档允许的角色
// acl = "HR,MANAGER,ADMIN"
// 检索时：当前用户角色 ∈ acl
public Filter.Expression buildFilter(UserContext user) {
    return new FilterExpressionBuilder()
            .and(
                    b("tenantId").eq(user.tenantId()),
                    b("status").eq("PUBLISHED"),
                    b("acl").in(user.roles())        // 任一角色匹配
            ).build();
}
```

## 3.4 权限注入的位置

权限过滤绝不能依赖"调用方自觉传入"，否则极易越权。正确做法是**在检索封装层强制注入**：

```java
public List<Document> retrieve(String query) {
    UserContext user = SecurityContext.current();       // 从上下文取，不信参数
    Filter.Expression filter = buildFilter(user);       // 强制注入权限
    return store.search(SearchRequest.builder()
            .query(query)
            .filterExpression(filter)
            .topK(5)
            .build());
}
```

| 注入位置 | 安全性 | 说明 |
|----------|:------:|------|
| 业务代码手写 | 低 | 容易漏写 |
| 检索封装层强制 | 高 | 唯一入口 |
| 向量库侧策略 | 最高 | 依赖库能力 |

> 边界：`acl.in(user.roles())` 的语义是"任一角色匹配"。若需"所有角色都满足"，需改用 `and` 展开，不要想当然。

# 4. 时效过滤

## 4.1 生效与失效

```java
// 只检索生效中的文档
filterExpression("effectiveDate <= '2026-09-21' && (expireDate == null || expireDate >= '2026-09-21')")

// 只检索最新版本
filterExpression("version == 'latest'")
```

> 政策类文档必须支持"生效日期"，否则会检索到已废止的旧版本。

## 4.2 版本管理

时效与版本常常同时使用，形成"当前有效版本"的完整约束：

```java
public Filter.Expression activeFilter(String today) {
    return new FilterExpressionBuilder()
            .and(
                    b("status").eq("PUBLISHED"),
                    b("effectiveDate").lte(today),
                    b("version").eq("latest")
            ).build();
}
```

| 场景 | 过滤条件 |
|------|----------|
| 只查当前有效 | `effectiveDate <= today && version == latest` |
| 查历史版本 | `version == 'v2'` |
| 查全部（含失效） | 不加时效条件（仅审计用） |

> 边界：`expireDate == null` 的处理要显式写出，不同向量库对 null 比较的支持不一。

## 4.3 时区与格式陷阱

时效过滤高度依赖时间字符串的可比较性，常见陷阱：

| 陷阱 | 后果 | 对策 |
|------|------|------|
| 格式不统一 | 字符串比较结果错误 | 统一 ISO 8601 |
| 时区不一致 | 边界日期误判 | 统一存 UTC 或明确时区 |
| 精度不一致 | `2026-09-21` 与 `2026-09-21T00:00:00` 不等 | 统一到同一精度 |
| 本地化格式 | `2026/9/21` 排序错乱 | 禁止本地化格式入库 |

```java
// 统一以 UTC + ISO 8601 写入
String now = Instant.now().toString();     // 2026-09-21T08:00:00Z
```

> 记住：过滤是**字符串/数值比较**，不是日期解析。格式不统一时，比较结果没有任何保证。

# 5. 结构化过滤

## 5.1 常用过滤维度

```java
// 按文档类型
filterExpression("docType == 'policy'")

// 按时间范围
filterExpression("updatedAt >= '2026-01-01'")

// 组合
filterExpression("docType == 'policy' && updatedAt >= '2026-01-01'")
```

## 5.2 过滤表达式语法

主流向量库多用类 SQL 或类 Mongo 表达式：

| 操作 | 表达式示例 | 含义 |
|------|------------|------|
| 等于 | `docType == 'policy'` | 精确匹配 |
| 范围 | `updatedAt >= '2026-01-01'` | 时间/数值范围 |
| 集合 | `acl in ['HR','ADMIN']` | 任一匹配 |
| 逻辑 | `A && B` / `A \|\| B` | 与/或 |
| 非空 | `expireDate != null` | 存在性判断 |

> 边界：不同向量库对字符串比较、大小写、时区的处理不同。上线前必须对**目标库**做一次语法与语义验证。

# 6. 元数据增强

## 6.1 LLM 增强

用 LLM 自动生成元数据（有成本，适合高价值知识库）：

```java
// Spring AI：关键词增强
KeywordMetadataEnricher enricher = new KeywordMetadataEnricher(chatModel, 5);
List<Document> enriched = enricher.apply(documents);

// 摘要增强
SummaryMetadataEnricher summaryEnricher = new SummaryMetadataEnricher(
        chatModel, List.of(SummaryType.CURRENT, SummaryType.PREVIOUS));
```

> 增强元数据可用于**过滤、重排、引用展示**，但要评估成本收益。

## 6.2 增强的边界

| 风险 | 说明 | 对策 |
|------|------|------|
| 成本 | 每篇文档额外一次 LLM 调用 | 只对高价值库启用 |
| 幻觉 | 生成的关键词可能不准确 | 作为辅助信号，不作为硬过滤 |
| 不一致 | 不同批次风格不一 | 固定提示词与模型版本 |
| 延迟 | 入库变慢 | 异步增强 + 回填 |

> 关键原则：**增强元数据只用于"软过滤/重排"，权限、时效等硬约束必须来自可信来源，不能由 LLM 生成。**

# 7. 元数据设计原则

## 7.1 原则清单

| 原则 | 说明 |
|------|------|
| 前置设计 | 入库前确定，事后补代价高 |
| 必填校验 | 关键字段（tenantId/source）强制校验 |
| 类型一致 | 时间统一 ISO 格式，避免字符串比较出错 |
| 避免高基数 | 不要放随机 ID 作为过滤主键 |
| 可扩展 | 预留字段，避免频繁改 schema |

## 7.2 反例

**反例一：时间格式混乱**

```
createdAt = "2026/9/21"     ← 与 "2026-09-21" 无法直接比较
```

**反例二：高基数过滤字段**

```
traceId = "a1b2c3..."        ← 每个分块都不同，建索引无意义且拖慢写入
```

**反例三：把权限信息交给 LLM 生成**

```
acl = llmGenerated(...)      ← 越权风险的根源
```

**反例四：必填字段留空**

```
tenantId = null              ← 过滤条件失效，导致跨租户泄漏
```

# 8. 过滤的性能影响

## 8.1 预过滤与后过滤

```
过滤越复杂 → 检索越慢
```

两种执行顺序的差异：

```
预过滤（Pre-filter）          后过滤（Post-filter）
先按元数据缩小候选集           先做向量检索取 TopK
再在候选集内做向量检索         再按元数据过滤
→ 结果数可控                  → 过滤后可能不足 K 条
```

优化手段：

| 手段 | 说明 |
|------|------|
| 预过滤 vs 后过滤 | 优先支持预过滤的向量库（如 Qdrant、Milvus） |
| 分区 | 按租户分区，缩小搜索范围 |
| 索引 | 为过滤字段建索引 |
| 减少过滤字段 | 只保留必要维度 |

> 部分向量库是"先检索后过滤"，过滤后结果可能不足；支持"预过滤"的库更可靠。

## 8.2 过滤字段的索引

| 字段类型 | 是否建索引 | 理由 |
|----------|:----------:|------|
| tenantId | 是 | 几乎每次过滤都用 |
| status | 是 | 基数低、命中率高 |
| docType | 是 | 常用维度 |
| createdAt | 视情况 | 若按时间范围查询则建 |
| docId | 否（作主键） | 高基数，且用作文档标识 |
| keywords | 谨慎 | 高基数，建议用于重排而非过滤 |

### 8.2.1 分区与集合划分

比索引更粗粒度的优化是"分区"，把大集合按高选择性字段切成若干子集合：

```java
// 按租户分区，检索时只进入对应分区
String partition = "tenant_" + user.tenantId();
VectorStore store = storeRegistry.get(partition);
```

| 划分维度 | 优点 | 代价 |
|----------|------|------|
| 按租户 | 隔离强、范围小 | 小租户碎片化 |
| 按时间 | 便于冷热分层 | 跨期查询需合并 |
| 按类型 | 各类型独立调参 | 类型间无法联合检索 |

> 分区与元数据过滤并不互斥：通常是"先按租户分区，再在分区内做元数据过滤"。

# 9. 元数据的展示（引用）

## 9.1 引用格式

```java
public String formatCitation(Document doc) {
    Metadata meta = doc.getMetadata();
    return "[%s · %s] %s".formatted(
            meta.get("title"),
            meta.get("section"),
            meta.get("source"));
}
```

回答时附来源，让用户可核实，显著提升信任度。

## 9.2 可追溯性

引用要能落到"可打开的原文位置"：

| 层级 | 提供的信息 | 价值 |
|------|------------|------|
| 文档级 | title + url | 可打开文档 |
| 章节级 | section / page | 可定位段落 |
| 分块级 | chunkId | 可精确回看 |

> 边界：若 `source` 指向内网地址或已删除文件，引用会失效。上线前需校验 source 的可访问性，或提供代理跳转。

## 9.3 引用的去重与聚合

一次回答常命中同一文档的多个分块，直接罗列会显得冗余。应按文档聚合后再展示：

```java
public List<String> citations(List<Document> docs) {
    return docs.stream()
            .collect(Collectors.groupingBy(d -> d.getMetadata().get("docId")))
            .values().stream()
            .map(group -> {
                Document first = group.get(0);
                List<String> sections = group.stream()
                        .map(d -> (String) d.getMetadata().get("section"))
                        .distinct().toList();
                return formatCitation(first, sections);   // 合并章节
            }).toList();
}
```

| 策略 | 效果 |
|------|------|
| 按 docId 去重 | 同一文档只出现一次 |
| 合并 section | 展示命中段落范围 |
| 按相关度排序 | 最相关来源在前 |

> 引用既要不遗漏，也要不冗余。聚合后的引用更便于用户快速定位。

# 10. 生产注意事项

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 元数据缺失 | 入库前强制校验 |
| 越权检索 | 过滤条件强制注入 + 越权测试 |
| 旧版本未失效 | 版本 + 生效日期管理 |
| 过滤拖慢检索 | 选支持预过滤的向量库 |
| 时间格式混乱 | 统一 ISO 8601 |
| 引用不可用 | 保证 source 可访问 |

## 10.2 测试与监控

| 测试/监控项 | 内容 |
|-------------|------|
| 越权测试 | 构造跨租户查询，断言结果为空 |
| 时效测试 | 构造过期文档，断言不被检索 |
| 召回监控 | 过滤后结果数是否低于阈值 |
| 过滤耗时 | P95/P99 延迟 |
| 元数据完整率 | 必填字段缺失比例 |

> 越权测试应作为**回归测试**常驻 CI，而非一次性验证。

# 11. 小结

## 11.1 关键要点

- 元数据三大作用：过滤、引用、治理。
- 权限过滤是 RAG 生产化的硬性要求。
- 隔离方案按合规强度选择，越权测试必不可少。
- 时效与版本管理防止检索到过期内容。
- 元数据要前置设计、强制校验、格式统一。

## 11.2 检查清单

- [ ] 核心字段（docId/source/tenantId/status）已定义且必填。
- [ ] 权限过滤在检索封装层强制注入。
- [ ] 时间字段统一 ISO 8601。
- [ ] 选用支持预过滤的向量库，过滤字段已建索引。
- [ ] 越权与时效测试进入 CI。
- [ ] 引用中的 source 可访问。
