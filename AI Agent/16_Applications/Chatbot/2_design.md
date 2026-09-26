# 1. 整体架构

聊天机器人（Chatbot）看似简单，生产化却涉及流式、记忆、RAG、工具、成本与安全六大工程问题。本节先给出整体架构，再逐层拆解。

## 1.1 分层视图

```
┌─────────────────────────────────────────┐
│  前端（Web/App）                         │
│  EventSource ← SSE                       │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│  API 网关（鉴权 / 限流 / 路由）           │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│  对话服务                                 │
│  ├── 会话管理（Redis）                    │
│  ├── 记忆（窗口 + 摘要）                  │
│  ├── RAG（PGVector）                      │
│  ├── 工具（订单、库存）                   │
│  └── Advisor（日志/护栏/成本）            │
└───────┬──────────┬──────────┬────────────┘
        ▼          ▼          ▼
   ┌────────┐ ┌────────┐ ┌────────┐
   │ 模型    │ │ 向量库  │ │ Redis  │
   └────────┘ └────────┘ └────────┘
```

## 1.2 组件职责

| 组件 | 职责 |
|------|------|
| 网关 | 鉴权、限流、路由、TLS 终止 |
| 对话服务 | 编排：记忆 → 意图 → RAG/工具 → 生成 |
| 会话存储（Redis） | 会话状态、短期记忆、缓存 |
| 向量库（PGVector） | 知识库检索 |
| 模型服务 | 外部 LLM API（可多模型路由） |

## 1.3 技术选型

| 能力 | 选型 | 理由 |
|------|------|------|
| 流式 | WebFlux + SSE | 非阻塞、天然支持背压 |
| 会话 | Redis | 低延迟、支持 TTL |
| 检索 | PGVector | 与业务库同源，运维简单 |
| 编排 | Spring AI | 抽象统一、生态成熟 |
| 可观测 | Micrometer + OTel | 标准、易接入 |

# 2. 数据流

## 2.1 一次对话的完整链路

```
用户消息
  │
  ▼
① 鉴权 + 限流
  │
  ▼
② 加载会话记忆
  │
  ▼
③ 意图识别（是否需要 RAG/工具）
  │
  ├── 需要 RAG → 检索 → 注入上下文
  ├── 需要工具 → 工具调用循环
  └── 直接回答
  │
  ▼
④ 流式生成
  │
  ▼
⑤ 保存记忆 + 记录指标
```

## 2.2 各阶段说明

| 阶段 | 关键点 |
|------|--------|
| ① 鉴权限流 | 租户/用户级配额，防止滥用 |
| ② 加载记忆 | 取最近 N 轮 + 会话摘要 |
| ③ 意图识别 | 用小模型或规则分类，决定挂载哪些能力 |
| ④ 流式生成 | SSE 增量推送，降低首字延迟 |
| ⑤ 保存记忆 | 异步写入，不阻塞响应 |

## 2.3 异常分支

- **模型超时**：返回已生成内容 + 提示"回答不完整"；
- **RAG 无结果**：降级为通用回答并说明"未找到相关资料"；
- **工具失败**：捕获错误，回填结构化错误给模型，由其决定重试或转人工；
- **限流触发**：返回 429 与重试建议。

### 2.3.1 降级原则

**宁可给出不完整但真实的回答，也不要静默失败**：所有降级都要明确告知用户，并记录指标用于事后分析。

# 3. 接口设计

## 3.1 流式对话接口（SSE）

```java
// 流式对话
GET /api/chat/stream?message=xxx
Header: X-Session-Id, Authorization

Response: SSE
  event: start    data: {"sessionId":"..."}
  event: sources  data: [{...}]        // RAG 来源
  event: message  data: 内容增量
  event: done     data: {"tokens":..,"cost":..}
  event: error    data: {"code":"..","message":".."}
```

### 3.1.1 事件设计要点

- `sources` 在生成前推送，便于前端展示引用；
- `message` 为增量文本，前端做拼接；
- `done` 携带 Token/成本，便于前端与运营观测。

## 3.2 会话管理接口

```java
GET    /api/sessions              // 列出会话
GET    /api/sessions/{id}         // 会话历史
DELETE /api/sessions/{id}         // 删除会话
POST   /api/feedback              // 反馈（点赞/点踩）
```

## 3.3 接口约定与错误码

| 错误码 | 含义 | 处理 |
|--------|------|------|
| 400 | 参数错误 | 提示用户 |
| 401/403 | 未认证/无权限 | 跳登录 |
| 429 | 限流 | 退避重试 |
| 500 | 服务异常 | 提示稍后重试 |
| 503 | 模型不可用 | 降级或转人工 |

# 4. 数据模型

## 4.1 表结构

```sql
-- 会话
CREATE TABLE conversation (
    id          VARCHAR(64) PRIMARY KEY,
    tenant_id   VARCHAR(64) NOT NULL,
    user_id     VARCHAR(64) NOT NULL,
    title       VARCHAR(256),
    status      VARCHAR(16) NOT NULL,
    summary     TEXT,
    created_at  TIMESTAMP NOT NULL,
    updated_at  TIMESTAMP NOT NULL
);

-- 消息
CREATE TABLE message (
    id              BIGSERIAL PRIMARY KEY,
    conversation_id VARCHAR(64) NOT NULL,
    role            VARCHAR(16) NOT NULL,
    content         TEXT NOT NULL,
    tokens          INT,
    created_at      TIMESTAMP NOT NULL
);

-- 反馈
CREATE TABLE feedback (
    id              BIGSERIAL PRIMARY KEY,
    message_id      BIGINT NOT NULL,
    user_id         VARCHAR(64) NOT NULL,
    rating          SMALLINT NOT NULL,   -- 1 / -1
    comment         TEXT,
    created_at      TIMESTAMP NOT NULL
);
```

### 4.1.1 表关系

```
conversation 1 ──< message 1 ──< feedback
```

一个会话含多条消息，一条消息可有多条反馈。

## 4.2 索引与约束

```sql
CREATE INDEX idx_conv_tenant_user ON conversation(tenant_id, user_id, updated_at DESC);
CREATE INDEX idx_msg_conv ON message(conversation_id, created_at);
CREATE INDEX idx_feedback_msg ON feedback(message_id);
```

- 查询会话列表按 `tenant_id + user_id + updated_at` 走索引；
- 拉取历史按 `conversation_id + created_at` 排序。

## 4.3 数据保留与清理

| 数据 | 保留策略 |
|------|----------|
| 会话/消息 | 按合规要求保留（如 180 天） |
| Redis 短期记忆 | TTL 自动过期 |
| 日志 | 脱敏后保留，敏感字段不入库 |

# 5. 关键技术设计

| 问题 | 方案 |
|------|------|
| 流式 | SSE + WebFlux |
| 记忆 | Redis 窗口 + 摘要 |
| 会话并发 | 会话级分布式锁 |
| RAG | PGVector + 混合检索 |
| 工具 | 按意图动态挂载 |
| 成本 | 模型分层 + 缓存 |
| 安全 | 护栏 Advisor + 租户隔离 |

## 5.1 流式响应

```java
@GetMapping(value = "/api/chat/stream",
        produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> stream(
        @RequestParam String message,
        @RequestHeader("X-Session-Id") String sessionId) {
    return chatService.stream(sessionId, message)
            .map(chunk -> ServerSentEvent.builder(chunk).event("message").build())
            .onErrorResume(e -> Flux.just(
                    ServerSentEvent.builder("{\"code\":\"500\"}").event("error").build()));
}
```

## 5.2 会话并发控制

同一会话的并发写入会导致消息乱序，需**会话级锁**：

```java
public <T> T withSessionLock(String sessionId, Supplier<T> action) {
    String key = "lock:session:" + sessionId;
    boolean locked = redis.opsForValue().setIfAbsent(key, "1", Duration.ofSeconds(10));
    if (!locked) throw new BusyException("会话正忙，请稍后");
    try {
        return action.get();
    } finally {
        redis.delete(key);
    }
}
```

### 5.2.1 锁的粒度与超时

锁粒度取**会话级**（而非用户级或全局），兼顾隔离与并发；必须设置过期时间，避免死锁。

## 5.3 RAG 集成

- 采用混合检索（稠密 + BM25）+ 重排，提升召回；
- 检索结果以 `sources` 事件推送，回答中标注引用编号；
- 无结果时降级为通用回答并明确告知。

> 检索细节见 `5_RAG/Retrieval/3_hybrid_search.md`。

## 5.4 动态工具挂载

按意图只挂载相关工具，避免工具过多干扰规划：

```java
List<ToolCallback> tools = switch (intent) {
    case "ORDER" -> orderTools;
    case "AFTER_SALE" -> afterSaleTools;
    default -> List.of();
};
```

## 5.5 成本控制

| 手段 | 说明 |
|------|------|
| 模型分层 | 简单意图用小模型，复杂用强模型 |
| 缓存 | 相同/相似问题命中缓存 |
| 上下文裁剪 | 只带必要历史与检索片段 |
| 预算告警 | 单会话/单租户成本上限 |

### 5.5.1 模型分层策略

按意图与复杂度路由：闲聊/FAQ → 小模型；检索问答 → 中模型；多步推理 → 强模型。路由规则集中配置，便于灰度调整。

## 5.6 安全与隔离

- **租户隔离**：检索强制按 `tenant_id` 过滤，越权测试纳入 CI；
- **提示注入防御**：用户输入永远放 User 角色，检索内容视为普通文本；
- **护栏**：输入/输出双向护栏，拦截敏感与违规内容。

> 详见 `15_Security/2_prompt_injection.md` 与 `15_Security/5_guardrails.md`。

# 6. 记忆策略

## 6.1 三层记忆

```
短期：最近 10 轮消息（Redis）
长期：会话摘要（超 20 轮后压缩）
用户：用户画像（跨会话）
```

| 层级 | 存储 | 作用 |
|------|------|------|
| 短期 | Redis | 保证多轮连贯 |
| 长期 | 摘要 | 保留关键信息、控制 Token |
| 用户 | 画像库 | 个性化（偏好、历史问题） |

### 6.1.1 三层如何协同

每轮对话：读短期 + 摘要 → 拼装上下文 → 生成 → 写回短期；当短期超阈值，触发压缩写入长期摘要。

## 6.2 压缩策略

超过阈值后，把早期消息交给模型压缩为摘要，替换原文。摘要需保留：用户目标、已确认事实、未决问题。

# 7. 工具设计

## 7.1 工具清单

| 工具 | 场景 |
|------|------|
| queryOrder | 查询订单 |
| queryStock | 查询库存 |
| queryPolicy | 查询政策 |
| createTicket | 创建工单 |

## 7.2 动态挂载

按意图识别动态挂载，避免工具过多；写操作类工具需审批或二次确认。

# 8. 部署

## 8.1 部署拓扑

```
无状态服务 × N（K8s）
  ├── Redis（会话 + 缓存）
  ├── PostgreSQL + pgvector（知识库）
  └── 模型 API（外部）
```

## 8.2 扩缩容

- 对话服务**无状态**，按 QPS 水平扩缩；
- Redis 用哨兵/集群保证高可用；
- 模型 API 侧做限流与多供应商容灾。

# 9. 可观测与运维

| 维度 | 指标 |
|------|------|
| 延迟 | 首字延迟（TTFT）、端到端 P99 |
| 质量 | 解决率、转人工率、点踩率 |
| 成本 | 单会话 Token/成本、缓存命中率 |
| 稳定性 | 模型错误率、工具失败率、限流触发数 |

> 详见 `13_Observability/9_cost_and_token_monitoring.md`。

# 10. 演进路线

1. **MVP**：单模型 + 滑动窗口记忆 + 无 RAG；
2. **增强**：接入 RAG（混合检索 + 重排）与工具调用；
3. **工程化**：流式、会话锁、成本控制、可观测；
4. **智能化**：多模型路由、语义缓存、主动追问；
5. **规模化**：多租户、灰度发布、A/B 评测。

# 11. 小结

- 架构：前端 SSE → 网关 → 对话服务 → 模型/向量库/Redis。
- 数据流：鉴权 → 记忆 → 意图 → RAG/工具 → 生成 → 保存。
- 接口：流式对话 + 会话管理 + 反馈，事件驱动、错误码规范。
- 关键设计：流式、记忆分层、会话锁、混合检索、动态工具、成本与安全。
- 无状态部署，状态外置；用指标驱动持续演进。
