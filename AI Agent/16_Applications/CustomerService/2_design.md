# 1. 整体架构与流程

结论先行：客服 Agent 的主链路是"身份 → 情绪 → 意图 → 回答 → 置信度 → 转人工"，每一步都可能提前短路转人工，形成多重兜底。

## 1.1 总体流程

```
用户消息
   │
   ▼
① 身份识别（登录用户）
   │
   ▼
② 情绪识别（不满/愤怒 → 直接转人工）
   │
   ▼
③ 意图识别
   ├── 咨询类 → RAG 问答
   ├── 售后类 → 工具（查订单）+ RAG
   ├── 投诉类 → 直接转人工
   └── 闲聊   → 直接回答
   │
   ▼
④ 生成回答（话术模板）
   │
   ▼
⑤ 置信度判断
   ├── 高 → 返回
   └── 低 → 转人工
   │
   ▼
⑥ 满意度收集
```

### 1.1.1 短路点

| 短路点 | 条件 | 结果 |
|--------|------|------|
| ② | 情绪为 ANGRY | 转人工 |
| ③ | 意图为 COMPLAINT | 转人工 |
| ⑤ | 置信度低 | 转人工 |

## 1.2 分层架构

```
接入层   ── HTTP / WebSocket 网关
编排层   ── 对话编排(Dialogue Orchestrator)
能力层   ── 意图/情绪/置信度/话术/RAG
工具层   ── queryOrder / queryTicket / createTicket
数据层   ── Postgres + PGVector + 工单系统
```

| 层 | 职责 | 关键类 |
|----|------|--------|
| 接入层 | 协议、鉴权 | `ChatController` |
| 编排层 | 状态机、上下文 | `DialogueOrchestrator` |
| 能力层 | 单点能力 | `IntentClassifier` |
| 工具层 | 外部调用 | `OrderTool` |
| 数据层 | 持久化 | `SessionRepository` |

# 2. 意图识别

结论先行：意图识别是路由中枢，分类错误会直接导致后续链路错配，因此要"宁可判未知，不可乱归类"。

## 2.1 意图分类体系

| 意图 | 处理 |
|------|------|
| 咨询 | RAG 问答 |
| 售后 | 工具 + RAG |
| 投诉 | 直接转人工 |
| 闲聊 | 直接回答 |
| 未知 | 澄清 / 转人工 |

### 2.1.1 意图边界

| 易混对 | 区分点 |
|--------|--------|
| 咨询 vs 售后 | 是否涉及具体订单 |
| 售后 vs 投诉 | 是否表达不满与追责 |
| 闲聊 vs 咨询 | 是否包含业务诉求 |

## 2.2 分类实现

```java
interface IntentClassifier {
    @UserMessage("""
            判断用户意图，只输出其中一个：
            CONSULT / AFTER_SALES / COMPLAINT / CHITCHAT / UNKNOWN
            用户：{{it}}
            """)
    String classify(String message);
}
```

### 2.2.1 结构化输出

```java
public record IntentResult(Intent intent, double confidence, String reason) {}

public enum Intent { CONSULT, AFTER_SALES, COMPLAINT, CHITCHAT, UNKNOWN }
```

## 2.3 边界处理

- 低置信：进入澄清，而非强行路由。
- 多意图：取最高优先级（投诉 > 售后 > 咨询 > 闲聊）。
- 未知：默认转人工或澄清一次。

# 3. 情绪识别

结论先行：情绪识别的价值在于"抢在答错之前转人工"，愤怒用户不应再由 AI 继续应答。

## 3.1 情绪分类

```java
interface EmotionDetector {
    @UserMessage("""
            判断用户情绪，输出 NORMAL / NEGATIVE / ANGRY。
            用户：{{it}}
            """)
    String detect(String message);
}
```

| 情绪 | 处理 |
|------|------|
| NORMAL | 正常应答 |
| NEGATIVE | 共情 + 正常应答 |
| ANGRY | 直接转人工 |

## 3.2 情绪处理策略

```java
// 愤怒用户直接转人工
if ("ANGRY".equals(emotion)) {
    return transferToHuman(message, "用户情绪激动");
}
```

### 3.2.1 共情话术

| 情绪 | 话术示例 |
|------|----------|
| NEGATIVE | "理解您的心情，我们马上帮您处理。" |
| ANGRY | "非常抱歉给您带来不便，已为您转接人工。" |

## 3.3 边界与坑

- 反例：把"我有点生气"误判为 ANGRY 会浪费人工资源。
- 坑：情绪随多轮变化，需按会话维度而非单句判断。

# 4. 置信度判断

结论先行：置信度是"拒答闸门"，用于把不可靠的回答挡在用户面前。

## 4.1 置信度来源

```java
public record ConfidenceScore(double score, String reason) {}

// 综合判断：
// ① 检索相似度
// ② 模型自评置信度
// ③ 是否命中知识库
```

| 来源 | 权重建议 |
|------|----------|
| 检索相似度 | 0.4 |
| 模型自评 | 0.4 |
| 命中知识库 | 0.2 |

## 4.2 阈值与决策

```java
if (confidence.score() < THRESHOLD) {
    return transferToHuman(message, "置信度不足");
}
```

### 4.2.1 阈值选择

| 阈值 | 效果 |
|------|------|
| 过高 | 转人工率高，成本升 |
| 过低 | 答错率升，投诉增 |
| 建议 | 0.6–0.7 起步，按数据调 |

## 4.3 边界与坑

- 坑一：模型自评普遍偏乐观，需校准。
- 坑二：检索相似度高但内容不匹配（同义不同指）。

# 5. 话术模板与生成

结论先行：话术模板是品牌一致性与合规的载体，提示词必须显式约束语气、称谓与承诺边界。

## 5.1 系统提示词

```java
@SystemMessage("""
        你是「小助手」，企业客服。

        话术规范：
        1. 称呼用户为"您"；
        2. 先共情再解决（"理解您的心情"）；
        3. 不使用网络流行语；
        4. 涉及承诺时谨慎表述；
        5. 无法处理时主动转人工。
        """)
```

## 5.2 话术约束

| 约束 | 说明 |
|------|------|
| 称谓 | 统一"您" |
| 语气 | 共情优先 |
| 承诺 | 不越权承诺 |
| 兜底 | 主动转人工 |

### 5.2.1 反例

| 反例 | 问题 |
|------|------|
| "包退款，绝对没问题" | 越权承诺 |
| "亲，这边建议您呢" | 网络化、不规范 |
| "我不知道" | 无兜底动作 |

# 6. 工具设计

结论先行：工具是 Agent 触达业务系统的唯一入口，必须做身份与归属双重校验。

## 6.1 工具清单

| 工具 | 说明 |
|------|------|
| `queryOrder` | 查询订单（校验归属） |
| `queryTicket` | 查询工单 |
| `createTicket` | 创建工单 |
| `transferToHuman` | 转人工 |

## 6.2 工具实现与鉴权

```java
@Component
public class OrderTool {

    private final OrderService orderService;

    public OrderTool(OrderService orderService) {
        this.orderService = orderService;
    }

    @Tool("查询订单，仅能查询当前用户自己的订单")
    public OrderView queryOrder(@ToolParam("订单号") String orderId,
                                UserContext user) {
        OrderView order = orderService.find(orderId);
        if (order == null || !order.userId().equals(user.userId())) {
            throw new SecurityException("无权访问该订单");
        }
        return order;
    }
}
```

### 6.2.1 鉴权规则

| 规则 | 说明 |
|------|------|
| 身份校验 | 必须登录 |
| 归属校验 | 资源属于当前用户 |
| 最小权限 | 工具仅暴露必要字段 |

## 6.3 边界与坑

- 坑：把 userId 作为模型可填参数会导致越权，必须由上下文注入。
- 反例：工具返回完整用户对象，泄露无关字段。

# 7. 转人工机制

结论先行：转人工不是失败，而是产品能力的一部分；关键是把上下文摘要完整交给坐席，避免用户重复描述。

## 7.1 触发条件

| 条件 | 优先级 |
|------|--------|
| 情绪 ANGRY | 高 |
| 意图 COMPLAINT | 高 |
| 置信度低 | 中 |
| 用户主动要求 | 中 |
| 连续澄清失败 | 低 |

## 7.2 转人工流程

```java
public class HumanTransfer {

    public TransferResult transfer(String sessionId, String reason,
                                   UserContext user) {
        // ① 创建转人工请求
        TransferRequest request = transferService.create(
                sessionId, user.userId(), reason);

        // ② 生成会话摘要（供人工参考）
        String summary = summarizeConversation(sessionId);

        // ③ 通知客服
        notificationService.notifyAgent(request.id(), summary);

        // ④ 返回给用户
        return new TransferResult("已为您转接人工客服，请稍候...");
    }
}
```

### 7.2.1 摘要内容

```
摘要包含：
  - 用户身份与联系方式
  - 意图与情绪
  - 关键事实（订单号、问题描述）
  - 已尝试的解答与失败原因
```

## 7.3 边界与坑

- 坑：坐席全部忙时需排队提示，避免用户以为无响应。
- 反例：转人工后 AI 仍在应答，造成双头对话。

# 8. 数据模型

结论先行：会话、消息、工单、反馈四张表构成最小闭环，所有关键动作可审计。

## 8.1 会话与消息表

```sql
CREATE TABLE cs_session (
    id          VARCHAR(64) PRIMARY KEY,
    user_id     VARCHAR(64) NOT NULL,
    status      VARCHAR(16) NOT NULL,   -- AI / HUMAN / CLOSED
    intent      VARCHAR(32),
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

## 8.2 工单与反馈表

```sql
CREATE TABLE cs_ticket (
    id          VARCHAR(64) PRIMARY KEY,
    session_id  VARCHAR(64) NOT NULL,
    user_id     VARCHAR(64) NOT NULL,
    type        VARCHAR(32),
    status      VARCHAR(16),
    created_at  TIMESTAMP NOT NULL
);

CREATE TABLE cs_feedback (
    id          BIGSERIAL PRIMARY KEY,
    session_id  VARCHAR(64) NOT NULL,
    rating      SMALLINT NOT NULL,
    resolved    BOOLEAN,
    created_at  TIMESTAMP NOT NULL
);
```

### 8.2.1 字段说明

| 表 | 关键字段 | 说明 |
|----|----------|------|
| cs_session | status | AI/HUMAN/CLOSED |
| cs_message | role | USER/AI/AGENT |
| cs_ticket | type | 工单类型 |
| cs_feedback | rating | 满意度评分 |

## 8.3 边界与坑

- 坑：message 表增长快，需按时间分区或归档。
- 反例：summary 为空导致坐席无上下文。

# 9. 关键技术与部署

结论先行：技术落地要解决"答错、情绪、合规、越权、转人工"五个核心问题，部署上优先保证低延迟与可观测。

## 9.1 关键技术点

| 问题 | 方案 |
|------|------|
| 答错 | 强约束 + 拒答 |
| 情绪升级 | 情绪识别 + 直接转人工 |
| 话术不合规 | 模板 + 提示约束 |
| 越权 | 身份校验 + 归属校验 |
| 转人工 | 摘要 + 通知 |

## 9.2 部署与性能

| 项 | 建议 |
|----|------|
| 部署形态 | 无状态服务 + 外部会话存储 |
| 缓存 | 意图/情绪结果短期缓存 |
| 限流 | 按用户与会话限流 |
| 可观测 | 全链路 Trace + 审计日志 |

### 9.2.1 降级策略

```
RAG 不可用 → 拒答 + 转人工
模型超时   → 转人工
工单系统异常 → 提示稍后重试
```

## 9.3 可观测性与容量

结论先行：可观测性决定问题能否被定位，容量规划决定高峰期能否稳住。

| 观测项 | 采集点 | 告警 |
|--------|--------|------|
| 首响延迟 | 编排层埋点 | P95 > 2s |
| 转人工率 | 会话表统计 | > 25% |
| 置信度分布 | 置信度服务 | 异常偏移 |
| 工具失败率 | 工具层 | > 5% |

### 9.3.1 容量估算

```
峰值会话数 = 在线用户 × 并发率
模型并发   = 峰值会话数 × 每会话调用次数
实例数     = 模型并发 / 单实例承载
```

### 9.3.2 成本控制

| 手段 | 说明 |
|------|------|
| 分层模型 | 简单任务用小模型 |
| 结果缓存 | 意图/情绪短期缓存 |
| 上下文裁剪 | 只带相关历史 |

# 10. 小结

## 10.1 关键结论

- 流程：身份 → 情绪 → 意图 → 回答 → 置信度 → 转人工。
- 投诉与愤怒用户直接转人工。
- 置信度低时转人工。

## 10.2 落地清单

- 话术模板保证品牌一致性。
- 转人工时生成摘要供客服参考。
- 工具鉴权做身份与归属双重校验。
