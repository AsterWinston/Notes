# 1. MemGPT 概述

结论先行：MemGPT 借鉴操作系统的虚拟内存（Virtual Memory）机制，把 LLM 的上下文窗口当作"内存"、外部存储当作"磁盘"，通过分页与换入换出（Paging / Swap）让模型自主管理记忆，从而突破固定窗口的限制。

## 1.1 核心思想

### 1.1.1 虚拟内存类比

```
┌──────────────────────────────────────┐
│  上下文窗口（内存）                    │
│  ├── System 指令（固定）               │
│  ├── 工作上下文（当前对话）             │
│  └── FIFO 消息队列                    │
└──────────────────────────────────────┘
              ▲ 换入 / 换出 ▼
┌──────────────────────────────────────┐
│  外部存储（磁盘）                      │
│  ├── 归档记忆（历史对话）              │
│  └── 召回记忆（长期事实）              │
└──────────────────────────────────────┘
```

### 1.1.2 关键洞察

> 关键洞察：**模型自己决定什么时候把信息写入长期存储、什么时候召回**。

传统方案由外部代码决定"记什么、忘什么"；MemGPT 把这一决策权交给模型，通过函数调用（Function Calling）完成。

### 1.1.3 与 RAG 的区别

检索增强生成（RAG，Retrieval-Augmented Generation）只在回答前检索一次，且检索由外部触发；MemGPT 让模型在对话中多次、自主地读写记忆，记忆本身可被更新，这是两者最本质的区别。

## 1.2 与传统记忆管理的差异

### 1.2.1 对比

| 维度 | 传统方案 | MemGPT |
|------|----------|--------|
| 决策者 | 外部代码 | 模型自身 |
| 触发方式 | 固定规则 | 模型按需调用 |
| 记忆粒度 | 消息级 | 事实 / 段落级 |
| 灵活性 | 低 | 高 |
| 成本 | 低 | 较高（多轮工具调用） |

### 1.2.2 适用场景

- 长周期个人助理、需要跨会话记住用户偏好与事实。
- 需要模型主动"记笔记"的复杂任务。
- 不适用：短任务、对延迟与成本极敏感的场景。

## 1.3 本文范围

本文覆盖 MemGPT 的三层记忆结构、自管理记忆、换入换出策略、Letta 工程化实现，以及自建系统的借鉴方式与生产实践。

# 2. 三层记忆结构

结论先行：MemGPT 把记忆分为核心记忆（常驻上下文）、工作上下文（当前对话）与归档记忆（可检索历史）三层，各层容量与访问方式不同，共同构成分级存储。

## 2.1 结构总览

### 2.1.1 层次表

| 层 | 内容 | 管理方式 |
|----|------|----------|
| System 指令 | 角色、规则 | 固定 |
| 工作上下文 | 当前对话 | 自动 |
| FIFO 队列 | 近期消息 | 队列淘汰 |
| 归档记忆 | 完整历史 | 按需检索 |
| 召回记忆 | 关键事实 | 常驻 / 按需 |

### 2.1.2 容量分配

上下文窗口是稀缺资源，需按比例分配：系统指令与核心记忆固定占用，FIFO 队列占用中等，剩余留给本轮推理与工具调用。

## 2.2 各层职责

### 2.2.1 核心记忆

存放"必须时刻可见"的事实，如用户姓名、偏好、当前目标。体量小但价值高，随对话动态更新。

### 2.2.2 工作上下文

当前对话的即时消息，直接参与推理，超出容量后进入 FIFO 队列淘汰。

### 2.2.3 归档记忆

完整历史对话与长期事实，默认不占上下文，仅在模型主动检索时按需换入。

### 2.2.4 三者协同

```
三层协同（一次请求的生命周期）：
  1) pageIn ：核心记忆 + 近期消息 + 按需归档 → 组装上下文
  2) 推理   ：模型生成回答，可能调用记忆工具
  3) pageOut：超容消息归档，核心记忆更新
  4) 落库   ：持久化消息历史与记忆状态
```

## 2.3 边界与坑

- 核心记忆若无上限，会像内存泄漏一样慢慢挤占窗口。
- FIFO 淘汰可能丢掉尚未归档的近期重要消息。

## 2.4 生产建议

- 为核心记忆设硬性条数或 Token 上限。
- 淘汰前先尝试归档，确保"换出即可再召回"。

# 3. 自管理记忆

结论先行：模型通过函数调用主动管理记忆，核心是五个记忆函数；Java 侧用工具（Tool）暴露这些能力，模型在对话中自行决定何时调用。

## 3.1 记忆函数

### 3.1.1 函数清单

```
可调用的记忆函数：
- core_memory_append    ：向核心记忆追加
- core_memory_replace   ：修改核心记忆
- archival_memory_insert：写入归档记忆
- archival_memory_search：检索归档记忆
- conversation_search   ：搜索历史对话
```

### 3.1.2 Java 工具实现

```java
// 用工具让模型自主管理记忆
@Component
public class MemoryTools {

    @Tool("向核心记忆追加信息（用户偏好、关键事实）")
    public String coreMemoryAppend(@P("要记住的内容") String content) {
        coreMemory.append(content);
        return "已记住：" + content;
    }

    @Tool("把当前对话归档到长期存储")
    public String archiveConversation(@P("归档内容") String content) {
        archiveStore.save(content);
        return "已归档";
    }

    @Tool("搜索历史归档记忆")
    public String searchArchive(@P("查询") String query) {
        return formatResults(archiveStore.search(query));
    }
}
```

## 3.2 自管理的收益与代价

| 维度 | 说明 |
|------|------|
| 收益 | 记忆决策贴合语义，无需人工规则 |
| 代价 | 每次记忆操作都是一次工具调用，增加 Token 与延迟 |
| 风险 | 模型可能滥用工具或写入噪声 |

### 3.2.1 收益量化

| 指标 | 传统 | MemGPT |
|------|:----:|:------:|
| 跨会话记忆 | 无 | 有 |
| 主动记忆 | 无 | 有 |
| 每轮 Token | 低 | 中 |
| 实现复杂度 | 低 | 高 |

## 3.3 边界与坑

- 工具返回值需简短，否则"记忆操作"本身又占满上下文。
- 模型可能反复写入重复内容，需在工具内做去重。
- 若模型从不调用记忆工具，等于退化为普通对话。

## 3.4 生产建议

- 限制每轮记忆工具的最大调用次数。
- 在系统提示中显式指导"何时该记忆、何时该检索"。
- 记录工具调用日志，分析模型的记忆行为模式。

# 4. 换入换出策略

结论先行：换入（pageIn）按"核心记忆 → 近期消息 → 按需归档"的顺序装配上下文，换出（pageOut）把超出队列容量的消息归档，两者共同维持窗口不溢出。

## 4.1 换入

### 4.1.1 代码

```java
public class MemoryPager {

    private static final int MAX_CONTEXT_TOKENS = 8000;

    public List<Message> pageIn(String sessionId, String question) {
        List<Message> context = new ArrayList<>();

        // ① 核心记忆（常驻）
        context.add(SystemMessage.from(coreMemory.render()));

        // ② 近期消息（FIFO 队列）
        context.addAll(fifoQueue.recent(sessionId, 10));

        // ③ 按需召回归档
        if (needsArchive(question)) {
            List<ArchiveEntry> relevant = archive.search(sessionId, question, 3);
            context.add(SystemMessage.from(formatArchive(relevant)));
        }

        return trimToFit(context, MAX_CONTEXT_TOKENS);
    }
```

### 4.1.2 装配顺序

核心记忆优先级最高，其次近期消息，最后按需归档；`trimToFit` 在超限时从低优先级部分开始裁剪。

## 4.2 换出

```java
    // 换出：把超出的消息归档
    public void pageOut(String sessionId) {
        List<Message> overflow = fifoQueue.overflow(sessionId);
        if (!overflow.isEmpty()) {
            archive.save(sessionId, overflow);
            fifoQueue.remove(overflow);
        }
    }
}
```

### 4.2.1 换出时机

在每轮对话结束后、追加新消息前执行，保证队列始终不超容。

### 4.2.2 换入换出时序

```
一次对话的换入换出时序：
  用户提问 ──▶ pageIn ──▶ 组装上下文 ──▶ 模型推理
                                          │
                            调用记忆工具 ◀─┘
                                          │
  回答返回 ◀── pageOut ◀── 归档超容消息 ◀─┘
```

## 4.3 边界与坑

- 换出与换入若并发，可能读到不一致的队列状态，需按会话加锁。
- 归档失败却已从队列移除，会造成消息永久丢失。

## 4.4 生产建议

- 先归档成功、再从队列删除，保证至少一次（At-least-once）。
- `trimToFit` 需按 Token 而非条数裁剪。
- 对换入换出打点，监控每轮上下文占用。

# 5. Letta 工程化实现

结论先行：Letta 是 MemGPT 的工程化实现（原 MemGPT 团队），把"有状态 Agent + 内置记忆管理"封装为可部署服务，免去自研换入换出的复杂度。

## 5.1 特性

### 5.1.1 特性表

| 特性 | 说明 |
|------|------|
| Agent 服务 | 有状态 Agent 服务 |
| 记忆管理 | 内置核心 / 归档记忆 |
| 多 Agent | 支持 Agent 间通信 |
| 工具 | 内置记忆工具 |
| 持久化 | 数据库持久化状态 |

### 5.1.2 架构图

```
Letta 架构：
  Agent（有状态）
    ├── 核心记忆（常驻上下文）
    ├── 归档记忆（可检索）
    └── 消息历史（持久化）
```

### 5.1.3 使用示例

```java
// 伪代码：通过 Letta SDK 创建有状态 Agent
LettaClient client = LettaClient.builder()
        .baseUrl("http://localhost:8283")
        .build();

AgentState agent = client.agents().create(
        CreateAgentRequest.builder()
                .name("assistant")
                .memoryBlocks(List.of(
                        MemoryBlock.builder().label("persona").build(),
                        MemoryBlock.builder().label("human").build()))
                .build());

// 后续每次对话只需带 agentId，状态由服务端维护
client.agents().messages().create(agent.id(),
        MessageCreate.builder().content(question).build());
```

## 5.2 与 MemGPT 的关系

MemGPT 是论文与原型，Letta 是其产品化延续：把记忆管理、状态持久化、多 Agent 通信做成开箱即用的能力。

## 5.3 边界与坑

- 引入 Letta 意味着新增一个外部依赖与运维面。
- 状态持久化在数据库中，迁移与备份需一并考虑。

## 5.4 生产建议

- 评估自研成本与 Letta 运维成本后再选型。
- 若采用 Letta，仍需在应用层做记忆质量监控。

# 6. 对自建系统的启发

结论先行：不必引入 MemGPT/Letta，也可借鉴其"核心记忆常驻 + 归档按需召回 + 模型主动管理"三条思想，用简单组件实现。

## 6.1 借鉴要点

### 6.1.1 三条原则

- 核心记忆：把高价值事实常驻上下文。
- 归档记忆：完整历史落库，按需检索。
- 模型主动管理：用工具让模型自己决定记与取。

### 6.1.2 最小实现

```java
public class InspiredMemoryManager {

    // ① 核心记忆：常驻上下文的关键事实
    private final CoreMemory coreMemory;

    // ② 归档：完整历史
    private final ArchiveStore archive;

    // ③ 让模型主动管理记忆（工具）
    private final MemoryTools memoryTools;

    public String chat(String sessionId, String question) {
        // 组装：核心记忆 + 近期消息 + 按需归档
        List<Message> context = assemble(sessionId, question);

        return chatClient.prompt()
                .messages(context)
                .tools(memoryTools)          // 模型可主动记忆
                .call().content();
    }
}
```

## 6.2 核心记忆实现

### 6.2.1 CoreMemory

```java
@Component
public class CoreMemory {

    private final Map<String, List<String>> memory = new ConcurrentHashMap<>();

    public void append(String userId, String content) {
        memory.computeIfAbsent(userId, k -> new ArrayList<>()).add(content);
        limitSize(userId, 20);              // 限制条数
    }

    public String render(String userId) {
        List<String> items = memory.getOrDefault(userId, List.of());
        if (items.isEmpty()) return "";

        return """
                关于用户的已知信息：
                %s
                """.formatted(items.stream()
                .map(i -> "- " + i)
                .collect(Collectors.joining("\n")));
    }
}
```

### 6.2.2 容量控制

`limitSize` 超出上限时淘汰最旧或最不常访问的条目，避免核心记忆无限增长。

```java
private void limitSize(String userId, int max) {
    List<String> items = memory.get(userId);
    if (items == null || items.size() <= max) {
        return;
    }
    // 简单策略：保留最近 max 条
    List<String> kept = new ArrayList<>(
            items.subList(items.size() - max, items.size()));
    memory.put(userId, kept);
}
```

## 6.3 边界与坑

- 用内存 `Map` 存核心记忆，进程重启即丢失，生产必须落库。
- 追加无去重，模型重复记忆会产生冗余条目。

## 6.4 生产建议

- 核心记忆持久化到数据库，并加缓存。
- 写入时做去重与归一化，淘汰策略显式可配。
- 定期人工审阅核心记忆，剔除过期事实。

# 7. 归档检索

结论先行：归档检索把历史对话向量化（Embedding）存入向量库，查询时按相似度召回，并用会话 ID 过滤保证隔离。

## 7.1 归档写入

### 7.1.1 代码

```java
@Component
public class ArchiveStore {

    public void save(String sessionId, List<Message> messages) {
        String content = messages.stream()
                .map(m -> m.role() + ": " + m.text())
                .collect(Collectors.joining("\n"));

        // 向量化存储，便于后续检索
        TextSegment segment = TextSegment.from(content,
                Metadata.from(Map.of("sessionId", sessionId)));
        vectorStore.add(embeddingModel.embed(segment).content(), segment);
    }
```

### 7.1.2 切分策略

按对话轮次或语义段落切分，片段过大会稀释相似度、过小会丢失上下文。

### 7.1.3 元数据设计

| 字段 | 说明 |
|------|------|
| sessionId | 会话隔离键 |
| userId | 用户隔离键 |
| timestamp | 写入时间，用于时间衰减 |
| role | 消息角色（user / assistant） |

## 7.2 归档检索

### 7.2.1 代码

```java
    public List<String> search(String sessionId, String query, int topK) {
        return vectorStore.similaritySearch(SearchRequest.builder()
                        .query(query)
                        .topK(topK)
                        .filterExpression("sessionId == '" + sessionId + "'")
                        .build())
                .stream().map(Document::getText).toList();
    }
}
```

### 7.2.2 过滤与排序

必须用 `sessionId` 过滤避免跨会话串扰；可用时间衰减加权，让近期记忆排序更靠前。

### 7.2.3 混合检索

先按 `sessionId` 过滤缩小范围，再按向量相似度排序，最后对时间做加权融合，兼顾"相关"与"新鲜"两个目标。

## 7.3 边界与坑

- `filterExpression` 直接拼接字符串存在注入风险，需用参数化或白名单。
- 归档写入与队列移除若不在同一事务，可能丢消息。

## 7.4 生产建议

- 为向量库建会话级分区或命名空间，强化隔离。
- 对检索结果做相关性阈值过滤，避免召回噪声。
- 监控召回命中率与延迟。

# 8. 生产注意事项

结论先行：MemGPT 类系统的风险集中在"工具滥用、记忆膨胀、检索慢、换入换出复杂、状态一致性、成本"六类，需逐项设防。

## 8.1 风险与对策

| 风险 | 建议 |
|------|------|
| 模型滥用记忆工具 | 限制调用频率 |
| 核心记忆膨胀 | 限制条数 |
| 归档检索慢 | 向量索引 |
| 换入换出复杂 | 从简单策略起步 |
| 状态一致性 | 持久化 + 事务 |
| 成本 | 记忆调用增加 Token |

## 8.2 监控指标

| 指标 | 含义 |
|------|------|
| 记忆工具调用频次 | 是否被滥用 |
| 核心记忆条数 | 是否膨胀 |
| 归档检索 P99 | 检索性能 |
| 上下文占用率 | 换入换出是否合理 |
| 每轮 Token 成本 | 成本趋势 |

## 8.3 边界与坑

- 状态存内存则重启即失忆，必须持久化。
- 只监控成本不监控记忆质量，会积累噪声。

## 8.4 生产建议

- 从"核心记忆 + 简单向量归档"起步，逐步演进。
- 为记忆工具设配额，超限时拒绝并提示模型。
- 定期审计归档内容，清理无效记忆。

# 9. 小结

结论先行：MemGPT 用操作系统式分级存储让模型自主管理记忆，Letta 将其工程化；自建系统可按需借鉴其思想而非照搬实现。

## 9.1 要点回顾

- MemGPT 借鉴虚拟内存思想，让模型自主管理记忆。
- 三层结构：核心记忆、工作上下文、归档记忆。
- 通过工具让模型主动写入 / 召回记忆。
- Letta 是其工程化实现，可参考其架构。
- 自建系统可借鉴思想：核心记忆常驻 + 归档按需召回。

## 9.2 落地清单

- 定义核心记忆字段与容量上限。
- 用工具暴露记忆写入与检索能力。
- 归档走向量库，按会话隔离。
- 换入换出先保证"归档成功再删除"。
- 建立调用频次、上下文占用与成本监控。
