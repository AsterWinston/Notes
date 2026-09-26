# 1. 客服服务主流程

## 1.1 设计目标与总体时序

### 1.1.1 结论先行

主流程是一条确定性编排链路，把"情绪识别 → 意图识别 → 分支处理 → 置信度判断 → 转人工"串起来。规则优先、模型兜底，任何输入都必须落到一个明确出口，不允许出现"无响应"的中间态。

### 1.1.2 处理时序

```
用户消息
   │
   ▼
① 情绪识别 ──ANGRY──▶ 转人工
   │ 其它
   ▼
② 意图识别
   ├── COMPLAINT ──▶ 转人工
   ├── CONSULT ────▶ RAG 问答
   ├── AFTER_SALES ▶ 工具 + RAG
   ├── CHITCHAT ───▶ 直接回答
   └── UNKNOWN ────▶ 澄清 / 转人工
   │
   ▼
③ 生成回答
   │
   ▼
④ 置信度判断 ──低──▶ 转人工
   │ 高
   ▼
⑤ 返回用户 + 满意度收集
```

### 1.1.3 关键设计取舍

| 取舍点 | 选择 | 理由 |
|--------|------|------|
| 情绪先于意图 | 是 | 愤怒用户不必再走问答 |
| 投诉是否走模型 | 否 | 合规硬约束，直接转人工 |
| 置信度来源 | 检索分 + 自评 | 单一信号不可靠 |
| 未知意图处理 | 澄清一次再转 | 避免无意义转接 |
| 状态存储 | 数据库 + 会话锁 | 支持多实例部署 |

## 1.2 主流程编排实现

### 1.2.1 核心代码

```java
@Service
@RequiredArgsConstructor
public class CustomerServiceAgent {

    private static final double THRESHOLD = 0.6;

    private final IntentClassifier intentClassifier;
    private final EmotionDetector emotionDetector;
    private final RagService ragService;
    private final ToolRegistry toolRegistry;
    private final HumanTransfer humanTransfer;
    private final ChatClient chatClient;

    public CsResponse handle(String sessionId, String message, UserContext user) {
        // ① 情绪识别
        String emotion = emotionDetector.detect(message);
        if ("ANGRY".equals(emotion)) {
            return humanTransfer.transfer(sessionId, "用户情绪激动", user);
        }

        // ② 意图识别
        String intent = intentClassifier.classify(message);

        // ③ 投诉直接转人工
        if ("COMPLAINT".equals(intent)) {
            return humanTransfer.transfer(sessionId, "投诉", user);
        }

        // ④ 生成回答
        CsResponse response = switch (intent) {
            case "CONSULT" -> handleConsult(sessionId, message, user);
            case "AFTER_SALES" -> handleAfterSales(sessionId, message, user);
            case "CHITCHAT" -> handleChitchat(message);
            default -> CsResponse.clarify("请问您具体想咨询什么问题？");
        };

        // ⑤ 置信度判断
        if (response.confidence() < THRESHOLD) {
            return humanTransfer.transfer(sessionId, "置信度不足", user);
        }
        return response;
    }
}
```

### 1.2.2 分支与出口

| 出口类型 | 触发条件 | 用户可见结果 | 会话状态 |
|----------|----------|--------------|----------|
| 直接回答 | 闲聊 / 高置信咨询 | 正常回复 | AI |
| 工具回答 | 售后且工具成功 | 订单信息 + 说明 | AI |
| 澄清 | UNKNOWN | 追问一次 | AI |
| 转人工 | 愤怒 / 投诉 / 低置信 | 转接提示 + 会话号 | HUMAN |

### 1.2.3 会话锁与幂等

同一 `sessionId` 的并发消息必须串行处理，否则会出现两条回复交错、状态互相覆盖。

```java
public CsResponse handleLocked(String sessionId, String message, UserContext user) {
    Lock lock = sessionLocks.computeIfAbsent(sessionId, k -> new ReentrantLock());
    lock.lock();
    try {
        if (sessionRepository.isClosed(sessionId)) {
            return CsResponse.closed("会话已结束。");
        }
        return handle(sessionId, message, user);
    } finally {
        lock.unlock();
    }
}
```

边界：单机锁在多实例部署下失效，需要换成分布式锁（如 Redis），或按 `sessionId` 做一致性哈希路由。

## 1.3 主流程的生产建议

- 为情绪与意图识别设置独立超时（各约 300ms），失败时降级为 NORMAL / UNKNOWN。
- 埋点每一步的耗时与出口，便于定位"为什么转人工"。
- 阈值 `THRESHOLD` 做成配置，按业务线灰度调整。
- 空消息、超长消息在入口处拦截并裁剪，避免无效模型调用。

# 2. 咨询处理（RAG）

## 2.1 检索增强问答机制

### 2.1.1 结论先行

咨询类问题的核心是"只依据知识库回答"。检索为空或相似度不足时，宁可低置信转人工，也不让模型自由发挥。

### 2.1.2 检索与生成链路

查询改写（多轮指代消解）→ 向量检索 TopK（如 K=5）→ 重排 Rerank 与阈值过滤 → 拼接上下文交给 LLM 生成 → 用检索分均值计算置信度。

### 2.1.3 边界与坑

| 坑 | 表现 | 规避 |
|----|------|------|
| 检索为空 | 模型编造 | 显式低置信拒答 |
| 多轮指代 | "它多少钱" | 查询改写补全实体 |
| 文档过期 | 旧政策 | 知识库版本管理 |
| 分块过大 | 噪声多 | 语义分块 + 重排 |

## 2.2 咨询处理实现

### 2.2.1 核心代码

```java
private CsResponse handleConsult(String sessionId, String message,
                                 UserContext user) {
    List<Document> docs = ragService.retrieve(message, user);

    if (docs.isEmpty()) {
        return CsResponse.lowConfidence("抱歉，我暂时没有找到相关信息。");
    }
    String answer = chatClient.prompt()
            .system("""
                    你是客服助手。只依据资料回答，资料不足时说明。
                    先共情，再给出解决方案。
                    """)
            .user("资料：\n%s\n\n问题：%s".formatted(format(docs), message))
            .call().content();

    double confidence = avgScore(docs);
    return CsResponse.ok(answer, confidence);
}
```

### 2.2.2 置信度计算

检索分取文档相似度均值，但相似度与答案正确率并非线性关系，需用标注数据校准阈值，不能直接当概率使用。

## 2.3 生产建议

- TopK 与阈值通过离线评测确定，不要拍脑袋。
- 对高频问题加语义缓存，命中后跳过模型调用。
- 知识库更新后做回归评测，防止答案漂移。

# 3. 售后处理（工具 + RAG）

## 3.1 工具调用机制

### 3.1.1 结论先行

售后问题需要真实数据，必须通过工具查询，且工具内部强制校验"只能查自己的订单"。模型只负责选择工具与组织话术，不负责鉴权。

### 3.1.2 工具编排

LLM 决策 → `queryOrder(orderId)` → 工具内归属校验 → 通过则返回订单，失败则拒绝提示 → 组织回答（工具结果 + RAG 政策）。

### 3.1.3 边界与坑

- 模型可能编造订单号，工具需返回明确错误而非空结果。
- 工具异常要转成可读话术，不能把堆栈抛给用户。
- 多工具场景需限制调用次数，防止死循环与成本失控。

## 3.2 售后处理实现

### 3.2.1 核心代码

```java
private CsResponse handleAfterSales(String sessionId, String message,
                                    UserContext user) {
    // 挂载工具
    List<ToolCallback> tools = toolRegistry.afterSalesTools(user);

    String answer = chatClient.prompt()
            .system("""
                    你是售后客服。
                    1. 需要查订单时使用 queryOrder；
                    2. 只能查询当前用户自己的订单；
                    3. 无法处理时说明并转人工。
                    """)
            .user(message)
            .toolCallbacks(tools)
            .call().content();

    return CsResponse.ok(answer, 0.85);
}
```

### 3.2.2 工具定义与归属校验

```java
@Component
@RequiredArgsConstructor
public class AfterSalesTools {

    private final OrderRepository orderRepository;

    @Tool("根据订单号查询订单，仅能查询当前用户的订单")
    public OrderView queryOrder(
            @ToolParam(description = "订单号") String orderId,
            UserContext user) {

        Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new BizException("订单不存在"));

        if (!order.userId().equals(user.userId())) {
            throw new BizException("无权查看该订单");
        }
        return OrderView.from(order);
    }
}
```

归属校验必须放在工具实现内部，而不是依赖提示词约束。提示词只是"建议"，代码才是"强制"。

## 3.3 生产建议

- 工具返回值做字段裁剪，只暴露必要信息。
- 记录每次工具调用的入参、结果与耗时，便于审计与复盘。
- 工具失败时给固定话术，避免暴露内部细节。

# 4. 情绪识别

## 4.1 情绪识别机制

### 4.1.1 结论先行

情绪识别是安全阀：ANGRY 一律转人工；NEGATIVE 走共情话术但可继续自动处理。

### 4.1.2 三级情绪

| 情绪 | 含义 | 处理 |
|------|------|------|
| NORMAL | 中性或正面 | 正常处理 |
| NEGATIVE | 不满、失望 | 共情 + 继续 |
| ANGRY | 愤怒、威胁 | 直接转人工 |

### 4.1.3 边界与坑

- 反讽（"服务真好，等了三天"）易误判为 NORMAL。
- 短消息（"？"）信息不足，宜按 NORMAL 处理。
- 情绪会随对话变化，应按最近若干轮而非单条判断。

## 4.2 情绪识别实现

### 4.2.1 核心代码

```java
interface EmotionDetector {
    @UserMessage("""
            判断用户情绪，只输出 NORMAL / NEGATIVE / ANGRY。

            判断依据：
            - ANGRY：明确愤怒、威胁、投诉升级
            - NEGATIVE：不满、失望
            - NORMAL：中性或正面

            用户：{{it}}
            """)
    String detect(String message);
}
```

### 4.2.2 与转人工联动

```java
if ("ANGRY".equals(emotion)) {
    return humanTransfer.transfer(sessionId, "用户情绪激动", user);
}
```

解析模型输出时需容错：结果不属于三值之一时按 NORMAL 处理并告警，避免非法值直接进入分支。

## 4.3 生产建议

- 用小模型或规则先过滤明显情绪词，降低延迟与成本。
- 定期抽样人工复核情绪判定，维护误判样例集。
- 情绪标签写入会话，供转人工摘要与质检使用。

# 5. 转人工

## 5.1 转人工机制

### 5.1.1 结论先行

转人工的体验取决于两件事：用户端有明确提示与会话编号，客服端有可读的会话摘要。缺一都会让转接"断档"。

### 5.1.2 会话摘要

摘要应包含：用户诉求、已尝试的处理、订单/工单号、情绪标签。长度控制在客服 10 秒内可读完。

### 5.1.3 边界与坑

- 无可用客服时不能直接失败，需进入排队并告知预计等待。
- 摘要生成失败要降级为原始消息片段。
- 转人工后 AI 必须停止应答，避免人机双答。

## 5.2 转人工实现

### 5.2.1 核心代码

```java
@Service
@RequiredArgsConstructor
public class HumanTransfer {

    private final TransferService transferService;
    private final SessionRepository sessionRepository;
    private final Summarizer summarizer;
    private final NotificationService notificationService;

    public CsResponse transfer(String sessionId, String reason, UserContext user) {
        // ① 生成会话摘要
        String summary = summarizer.summarize(
                sessionRepository.findMessages(sessionId));

        // ② 创建转人工请求
        TransferRequest request = transferService.create(
                sessionId, user.userId(), reason, summary);

        // ③ 更新会话状态
        sessionRepository.updateStatus(sessionId, "HUMAN");

        // ④ 通知客服
        notificationService.notifyAvailableAgent(request);

        return CsResponse.transfer("""
                非常抱歉给您带来不便。已为您转接人工客服，请稍候。
                您的会话编号：%s
                """.formatted(request.id()));
    }
}
```

### 5.2.2 会话状态与通知

| 状态 | 含义 | 允许操作 |
|------|------|----------|
| AI | AI 应答中 | 收发消息 |
| HUMAN | 已转人工 | 仅人工应答 |
| CLOSED | 会话结束 | 只读 |

## 5.3 生产建议

- 转人工请求进入队列，按技能组路由。
- 通知失败时重试并告警，保证不漏接。
- 摘要限制长度、突出关键信息，便于客服快速接手。

# 6. 满意度收集

## 6.1 满意度收集机制

### 6.1.1 结论先行

满意度是结果指标，但更重要的是把"未解决 + 低分"的会话回流为改进样本，形成闭环。

### 6.1.2 数据回流闭环

```
会话结束
   │
   ▼
邀请评价 ──▶ 评分 + 是否解决
   │
   ▼
未解决或低分 ──▶ 失败案例库
   │
   ▼
聚类分析 ──▶ 知识库补充 / 提示优化
```

### 6.1.3 边界与坑

- 评价率低会导致样本偏差，需引导而非强推。
- 评分口径要统一（1-5 分还是拇指）。
- 恶意刷分需去重与限流。

## 6.2 满意度实现

### 6.2.1 核心代码

```java
@PostMapping("/api/cs/feedback")
public void feedback(@RequestBody CsFeedbackRequest req, UserContext user) {
    feedbackService.save(new CsFeedback(
            req.sessionId(), req.rating(), req.resolved()));

    // 未解决 + 低分 → 回流分析
    if (!req.resolved() || req.rating() < 3) {
        failureCaseService.record(req);
    }
}
```

### 6.2.2 数据模型

`cs_feedback(session_id, rating, resolved, created_at)`，`session_id` 建唯一索引保证幂等。

## 6.3 生产建议

- 评价请求幂等：同一会话只计一次。
- 失败案例按主题聚类，驱动知识库补齐。
- 把满意度与解决率、转人工率一起看，避免单指标误导。

# 7. 数据模型与接口

## 7.1 数据模型

### 7.1.1 会话与消息表

```sql
CREATE TABLE cs_session (
    id          VARCHAR(64) PRIMARY KEY,
    user_id     VARCHAR(64) NOT NULL,
    status      VARCHAR(16) NOT NULL,   -- AI / HUMAN / CLOSED
    intent      VARCHAR(32),
    emotion     VARCHAR(16),
    summary     TEXT,
    created_at  TIMESTAMP NOT NULL
);

CREATE TABLE cs_message (
    id          BIGSERIAL PRIMARY KEY,
    session_id  VARCHAR(64) NOT NULL,
    role        VARCHAR(16) NOT NULL,   -- USER / AI / AGENT
    content     TEXT NOT NULL,
    created_at  TIMESTAMP NOT NULL
);
```

### 7.1.2 转人工与反馈表

| 表 | 关键字段 | 用途 |
|----|----------|------|
| cs_transfer | id, session_id, reason, summary, status | 转人工队列 |
| cs_feedback | session_id, rating, resolved | 满意度统计 |

## 7.2 接口设计

### 7.2.1 接口清单

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /api/cs/chat | 发送消息 |
| POST | /api/cs/feedback | 提交评价 |
| GET | /api/cs/session/{id} | 查询会话 |

### 7.2.2 请求与响应

请求体为 `{ sessionId, message }`，响应体为 `{ type, answer, confidence, sessionId }`；`type` 取 `ANSWER / CLARIFY / TRANSFER`。

## 7.3 部署与配置

关键配置项：`cs.confidence-threshold`（转人工阈值）、`cs.rag.top-k`（检索条数）、`cs.tool.max-calls`（工具调用上限）、`cs.transfer.queue`（转人工队列）。

# 8. 难点与演进

## 8.1 难点与解决

| 难点 | 解决 |
|------|------|
| 情绪升级 | 情绪识别 + 直接转人工 |
| 答错 | 强约束 + 置信度 + 转人工 |
| 话术不合规 | 模板 + 提示约束 |
| 越权 | 归属校验 |
| 转人工体验 | 摘要 + 快速通知 |

高频难点补充：并发同会话会导致消息交错，用会话锁或路由解决；模型漂移会导致答案变化，靠回归评测发现；成本失控来自长上下文，用缓存与分层模型缓解。

## 8.2 演进路线

```
P0 知识库问答
P1 意图识别 + 转人工
P2 工单工具
P3 满意度 + 审计
P4 评测 + 优化
```

# 9. 小结

## 9.1 核心结论

- 主流程：情绪 → 意图 → 处理 → 置信度 → 转人工。
- 投诉与愤怒用户直接转人工。
- 售后用工具 + RAG。
- 转人工生成摘要供客服参考。
- 满意度与未解决案例回流分析。

## 9.2 实现检查清单

- [ ] 情绪/意图解析有兜底值
- [ ] 工具内部做归属校验
- [ ] 同会话并发加锁
- [ ] 转人工状态机正确
- [ ] 全流程可审计
