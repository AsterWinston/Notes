# 1. 为什么 LLM 应用容量难算

**结论先行**：LLM（Large Language Model，大语言模型）应用的容量不能用传统 Web 的「QPS × 平均耗时」单点公式估算。它同时受请求时长、Token（词元）用量、并发模型与下游配额四条相互独立的约束，必须分层建模，并取各层容量的最小值。

## 1.1 与传统 Web 容量的差异

### 1.1.1 请求耗时长且方差大

传统接口耗时通常在 10~200 ms，方差小，平均值即可代表容量。LLM 请求耗时 1~30 s，且同一接口在不同输入下可能相差数十倍，平均值会严重失真。

| 维度 | 传统 Web | LLM 应用 |
|------|----------|----------|
| 单请求耗时 | 10~200 ms | 1~30 s |
| 耗时方差 | 小 | 极大 |
| 主要资源 | CPU / 内存 | 线程 / 连接 + 下游配额 |
| 成本模型 | 固定 | 按 Token 计费 |
| 重试代价 | 低 | 高（重复计费） |

### 1.1.2 Token 用量决定成本与延迟

输出 Token 越多，生成时间越长、费用越高。容量规划必须同时给出「请求数」与「Token 数」两套口径，缺一不可。

### 1.1.3 下游限流成为硬约束

模型厂商通常以 RPM（Requests Per Minute，每分钟请求数）与 TPM（Tokens Per Minute，每分钟词元数）限流。本地资源再充裕，也绕不过这道墙。

## 1.2 并发模型的三种形态

### 1.2.1 同步阻塞

每个请求独占一个 Web 线程直到模型返回，线程池大小即并发上限，扩容等于加线程或加副本。

### 1.2.2 流式长连接

SSE（Server-Sent Events，服务器推送事件）或 WebSocket（全双工通信协议）保持长连接，占用的是连接与内存，而非线程。

### 1.2.3 异步任务

提交后立即返回 taskId，由队列与 Worker 异步执行，适合超过 30 s 的长任务。

## 1.3 容量估算的常见误区

| 误区 | 后果 | 纠正 |
|------|------|------|
| 只按 QPS 估算 | 忽略 Token 与配额 | 双口径建模 |
| 只用平均延迟 | 低估尾延迟 | 用 P95 / P99 |
| 忽略每请求调用次数 | Agent 场景严重低估 | 乘以调用次数 |
| 忽略流式连接内存 | OOM | 单独估算连接内存 |
| 无压测直接上线 | 拐点未知 | 定期压测 |

> 容量是「各层约束的最小值」，不是单一公式的结果。

# 2. 关键容量指标

**结论先行**：容量规划围绕五类指标展开——并发、吞吐、延迟、Token 吞吐与队列积压，它们通过利特尔法则（Little's Law）相互换算。

## 2.1 核心指标定义

### 2.1.1 并发与吞吐

| 指标 | 说明 | 单位 |
|------|------|------|
| 并发请求数 | 同时在处理的请求 | 个 |
| 吞吐（QPS） | 每秒完成的请求数 | req/s |
| 平均 / P99 延迟 | 响应时间 | ms / s |
| Token 吞吐 | 每秒处理的 Token | tok/s |
| 队列积压 | 待处理任务数 | 个 |

### 2.1.2 延迟分位

平均值掩盖长尾。对话类应用关注首字节延迟（TTFT，Time To First Token）与端到端延迟，二者口径不同，不能混用。

### 2.1.3 Token 吞吐与队列积压

Token 吞吐决定成本与模型层压力；队列积压是背压（Backpressure）的触发信号，超过阈值即应拒绝或降级。

## 2.2 指标之间的关系

### 2.2.1 利特尔法则

在稳态系统中，并发数、吞吐与延迟满足：

```
吞吐 = 并发数 / 平均延迟
并发数 = 吞吐 × 平均延迟
平均延迟 = 并发数 / 吞吐
```

### 2.2.2 计算示例

例：目标 50 QPS，平均延迟 2 s，则所需并发 ≈ 100。若延迟升至 5 s，同样 50 QPS 就需要 250 并发，容量需求翻 2.5 倍。

| 目标 QPS | 平均延迟 | 所需并发 |
|:--------:|:--------:|:--------:|
| 50 | 1 s | 50 |
| 50 | 2 s | 100 |
| 50 | 5 s | 250 |
| 100 | 2 s | 200 |

## 2.3 指标的采集与观测

容量规划要基于真实观测而非猜测。建议暴露以下指标：

| 指标名 | 类型 | 说明 |
|--------|------|------|
| `http_server_requests_seconds` | Histogram | 端到端延迟分位 |
| `llm_tokens_total` | Counter | Token 累计用量 |
| `llm_calls_total` | Counter | 模型调用次数 |
| `queue_depth` | Gauge | 当前队列积压 |
| `inflight_requests` | Gauge | 当前并发 |

```java
// 用 Micrometer 记录并发与延迟（JDK 17+）
private final AtomicInteger inflight = new AtomicInteger();

public String handle(String q) {
    inflight.incrementAndGet();
    try {
        return chatClient.prompt().user(q).call().content();
    } finally {
        inflight.decrementAndGet();
    }
}
```

> 采集指标是分层计算的前提：没有数据，任何公式都是空谈。

# 3. 分层容量计算

**结论先行**：把请求路径拆成接入、Agent、模型、检索、工具、状态六层，逐层计算容量，最终取各层最小值作为系统上限。

## 3.1 分层模型

### 3.1.1 六层结构

```
① 接入层：Web 线程 / 连接数
② Agent 层：每请求的模型调用次数
③ 模型层：RPM / TPM 限制
④ 检索层：向量库 QPS
⑤ 工具层：下游 API 限流
⑥ 状态层：存储读写 QPS
```

### 3.1.2 木桶原理

系统吞吐等于最慢那一层。即使接入层能扛 500 并发，模型层 RPM 只允许 100，真实上限就是 100。

## 3.2 逐层计算

### 3.2.1 接入层

接入层容量 ≈ 线程数（同步）或连接数上限（流式）。Tomcat 默认 `max-threads=200`，即同步模式下最多 200 并发。

### 3.2.2 Agent 层与模型层

Agent 每处理一个用户请求，可能调用模型多次（规划、工具、总结）。模型层并发需求：

```
模型并发 = 目标 QPS × 每请求调用次数 × 平均延迟
```

### 3.2.3 检索、工具与状态层

检索层、工具层、状态层各自有 QPS 上限，需按调用放大系数折算。

| 层 | 放大系数 | 说明 |
|----|:--------:|------|
| 接入 | 1.0 | 每请求 1 次 |
| 模型 | 2~5 | 每请求多次调用 |
| 检索 | 1~3 | 可能多路检索 |
| 工具 | 0~5 | 依赖工具数量 |
| 状态 | 2~10 | 频繁读写 |

## 3.3 Java 实现

```java
public record LayerCapacity(String name, int concurrency) {}

public final class CapacityCalculator {

    public static int modelConcurrency(
            int targetQps, double callsPerRequest, double avgLatencySec) {
        return (int) Math.ceil(targetQps * callsPerRequest * avgLatencySec);
    }

    public static int concurrencyByRpm(int rpm, double avgLatencySec) {
        return (int) (rpm / 60.0 * avgLatencySec);
    }

    public static int bottleneck(List<LayerCapacity> layers) {
        return layers.stream()
                .mapToInt(LayerCapacity::concurrency)
                .min()
                .orElse(0);
    }

    public static void main(String[] args) {
        int targetQps = 50;
        double callsPerRequest = 3.0;
        double avgLatencySec = 2.0;

        int modelConc = modelConcurrency(targetQps, callsPerRequest, avgLatencySec);
        int modelByRpm = concurrencyByRpm(10_000, avgLatencySec);

        int limit = bottleneck(List.of(
                new LayerCapacity("access", 200),
                new LayerCapacity("model", Math.min(modelConc, modelByRpm)),
                new LayerCapacity("vector", 150)));

        System.out.printf("model=%d, byRpm=%d, system=%d%n",
                modelConc, modelByRpm, limit);
    }
}
```

## 3.4 边界与坑

- 放大系数是均值，Agent 场景方差大，应按 P95 而非均值估算。
- RPM 与 TPM 是两把锁，Token 长的请求可能先撞 TPM。
- 模型厂商按组织级限流，多实例共享配额，不能按副本数线性外推。

> 生产建议：把每层容量写成配置项并纳入监控，任一层的实际使用率超过 70% 即预警。

# 4. 流式场景容量

**结论先行**：流式连接是长连接，瓶颈在连接数与内存，而非线程数，必须单独建模。

## 4.1 流式的资源模型

### 4.1.1 连接而非线程

同步请求结束即释放线程；流式连接在整个生成期间持续存在，占用文件描述符与连接对象。

### 4.1.2 内存与缓冲区

```
1000 个并发流式连接：
- 连接数：1000
- 每连接内存：几十 KB ~ 几 MB
- 总内存：取决于缓冲区与框架
```

若每连接缓冲 64 KB，1000 连接约需 64 MB，叠加堆外缓冲可能更高。

## 4.2 优化手段

| 手段 | 说明 |
|------|------|
| WebFlux 响应式 | 少量线程支撑大量连接 |
| 连接数上限 | 防止无限增长 |
| 心跳与超时 | 清理死连接 |
| 分片部署 | 按连接数扩容 |
| 减小缓冲区 | 降低单连接内存 |

```yaml
  # WebFlux / Netty 连接与缓冲配置
server:
  netty:
    connection-timeout: 5s
  port: 8080
spring:
  codec:
    max-in-memory-size: 256KB
```

## 4.3 边界与坑

- 反向代理（Nginx / 网关）的连接数与超时往往先于应用成为瓶颈。
- 客户端断开后服务端若不及时释放，会累积「僵尸连接」。
- 流式响应的首字节延迟与整体延迟都要监控，不能只看总时长。

> 生产建议：对流式接口单独设置连接数配额与超时，并对断开事件计数告警。

# 5. 压测

**结论先行**：压测是容量规划的唯一实证手段，目标是找到性能拐点并验证各层约束。

## 5.1 压测工具选型

| 工具 | 特点 |
|------|------|
| k6 | 脚本化，支持 SSE |
| JMeter | 功能全，GUI |
| Gatling | Scala DSL，报告好 |
| wrk | 轻量高性能 |

## 5.2 k6 压测脚本

### 5.2.1 基础脚本

```javascript
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  stages: [
    { duration: '1m', target: 20 },    // 爬坡
    { duration: '3m', target: 50 },    // 稳态
    { duration: '1m', target: 0 },     // 回落
  ],
  thresholds: {
    http_req_duration: ['p(95)<3000'],  // P95 < 3s
    http_req_failed: ['rate<0.01'],     // 错误率 < 1%
  },
};

export default function () {
  const res = http.post('http://localhost:8080/api/chat',
    JSON.stringify({ message: '你好' }),
    { headers: { 'Content-Type': 'application/json' } });

  check(res, { 'status is 200': (r) => r.status === 200 });
}
```

### 5.2.2 SSE 压测要点

流式接口需测量 TTFT 与总时长，而非仅看状态码：

```javascript
import sse from 'k6/x/sse';

export default function () {
  const res = sse.open('http://localhost:8080/api/stream?q=hello', (client) => {
    client.on('event', () => {});
    client.on('error', () => {});
  });
  check(res, { 'sse connected': (r) => r.status === 200 });
}
```

## 5.3 压测方法

### 5.3.1 分级压测

先压单接口，再压全链路；先小并发找基线，再逐步加压找拐点。

### 5.3.2 拐点识别

| 阶段 | 吞吐 | 延迟 | 表现 |
|------|:----:|:----:|------|
| 线性区 | 随并发上升 | 稳定 | 健康 |
| 拐点 | 增速放缓 | 开始上升 | 临界 |
| 饱和区 | 不再上升 | 急剧上升 | 危险 |

## 5.4 压测注意

| 注意 | 说明 |
|------|------|
| Mock 模型 | 否则压测成本极高 |
| 分级压测 | 先压单接口再压全链路 |
| 观察下游 | 是否触发限流 |
| 记录拐点 | 找到性能拐点 |
| 真实数据 | 请求体大小接近真实 |
| 隔离环境 | 避免污染生产 |

> 生产建议：把压测纳入 CI，每次容量相关变更后重跑，拐点数据存档对比。

# 6. 成本容量估算

**结论先行**：容量不仅是「能不能扛住」，还是「扛住要花多少钱」。成本估算需把模型、嵌入与基础设施三类费用相加。

## 6.1 成本构成

| 成本项 | 驱动因素 |
|--------|----------|
| 模型输入 | 月请求数 × 平均输入 Token |
| 模型输出 | 月请求数 × 平均输出 Token |
| 嵌入 | 新增文档数 × 平均文档 Token |
| 基础设施 | 实例数 × 实例月价 |

## 6.2 Java 实现

```java
import java.math.BigDecimal;

public record CapacityPlan(
        BigDecimal monthlyRequests,
        int avgInputTokens,
        int avgOutputTokens,
        BigDecimal inputPrice,
        BigDecimal outputPrice,
        BigDecimal monthlyNewDocs,
        int avgDocTokens,
        BigDecimal embeddingPrice,
        int instances,
        BigDecimal instanceMonthlyPrice) {}

public final class CostCapacityEstimator {

    public BigDecimal monthlyCost(CapacityPlan plan) {
        BigDecimal modelInput = plan.monthlyRequests()
                .multiply(BigDecimal.valueOf(plan.avgInputTokens()))
                .multiply(plan.inputPrice());
        BigDecimal modelOutput = plan.monthlyRequests()
                .multiply(BigDecimal.valueOf(plan.avgOutputTokens()))
                .multiply(plan.outputPrice());
        BigDecimal embeddingCost = plan.monthlyNewDocs()
                .multiply(BigDecimal.valueOf(plan.avgDocTokens()))
                .multiply(plan.embeddingPrice());
        BigDecimal infraCost = BigDecimal.valueOf(plan.instances())
                .multiply(plan.instanceMonthlyPrice());

        return modelInput.add(modelOutput).add(embeddingCost).add(infraCost);
    }
}
```

## 6.3 边界与坑

- 用 `BigDecimal` 而非 `double`，避免金额精度丢失。
- 重试会重复计费，成本估算要计入失败率。
- 价格表会变动，应外置为配置而非硬编码。

> 生产建议：把成本估算做成定时任务，每日对账实际账单与预测值。

# 7. 容量规划表

**结论先行**：规划表把「当前 → 目标」的差距量化为可执行的扩容动作。

## 7.1 规划表示例

| 指标 | 当前 | 规划（6 个月） | 扩容方式 |
|------|:----:|:-------------:|----------|
| 日请求数 | 10 万 | 50 万 | 水平扩容 |
| 并发 | 100 | 500 | 增加副本 |
| 向量量 | 100 万 | 500 万 | 向量库扩容 |
| 日 Token | 5000 万 | 2.5 亿 | 配额 + 缓存 |
| 日成本 | ¥500 | ¥2500 | 优化手段 |

## 7.2 规划方法

1. 采集当前真实峰值（非平均）。
2. 按增长率外推目标值。
3. 逐层计算所需资源。
4. 标注每项的扩容方式与前置条件。

> 规划表要按季度滚动更新，避免一次性预测过远导致失真。

# 8. 弹性与预留

**结论先行**：基线负载用固定副本预留，峰值负载用弹性扩容，突发流量用队列吸收。

## 8.1 三种负载

```
基线负载 → 固定副本（预留）
峰值负载 → HPA 弹性扩容
突发流量 → 队列缓冲 + 限流保护
```

## 8.2 弹性配置

### 8.2.1 HPA 配置

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: agent-api
spec:
  minReplicas: 3      # 覆盖基线
  maxReplicas: 20     # 应对峰值
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
```

### 8.2.2 队列缓冲

```yaml
queue:
  max-depth: 10000        # 有界队列
  reject-threshold: 8000  # 超过即返回 429
  consumer-concurrency: 8
```

## 8.3 边界与坑

- 弹性扩容有冷启动延迟，LLM 场景副本启动慢，需预热。
- HPA 依赖指标，若用 CPU 而瓶颈在下游配额，扩容无效。
- 扩到 maxReplicas 后若仍不够，只能靠队列与限流兜底。

> 生产建议：预留副本数覆盖日常峰值，弹性只用于应对超预期增长。

# 9. 降本与扩容的权衡

**结论先行**：不是所有增长都靠扩容解决，降本与扩容是一组可权衡的杠杆。

## 9.1 决策矩阵

| 场景 | 选择 |
|------|------|
| 成本敏感 | 优化（缓存 / 分层 / 裁剪） |
| 增长确定 | 预留容量（更便宜） |
| 波动大 | 弹性扩容 |
| 突发极端 | 队列 + 限流 + 降级 |

## 9.2 混合策略

多数系统需要组合：基线预留 + 缓存降本 + 峰值弹性 + 队列兜底。纯扩容成本随容量线性上升，纯优化则有天花板，混合策略在两者之间取平衡。

> 生产建议：每季度复核一次「优化 vs 扩容」的性价比，优化收益会随规模放大。

# 10. 生产注意事项

**结论先行**：容量规划失败的根因往往不是算错，而是观测缺失与假设未验证。

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 只按 QPS 估算 | 分层计算 |
| 忽略 Token | Token 决定成本与延迟 |
| 无压测 | 定期压测 |
| 压测用真模型 | 成本高，用 Mock |
| 无拐点数据 | 记录性能拐点 |
| 无预留 | 基线用固定副本 |
| 无监控 | 每层使用率纳入监控 |

## 10.2 落地检查表

- 五类核心指标是否已采集？
- 六层容量是否都有上限配置？
- 是否做过分级压测并记录拐点？
- 弹性扩缩容是否经过验证？
- 队列是否有界、是否配置背压？

> 生产建议：把检查表固化为上线门禁，缺一项不予发布。

# 11. 小结

- LLM 容量要分层计算，取各层最小值。
- 吞吐 = 并发 / 平均延迟，据此反推所需并发。
- 流式场景关注连接数而非线程数。
- 压测用 Mock 模型，找性能拐点。
- 基线预留 + 峰值弹性 + 队列吸收突发。
- 成本与容量是一体两面，需同时规划。
