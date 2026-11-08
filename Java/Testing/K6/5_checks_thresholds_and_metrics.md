# 1. 指标体系概览

## 1.1 四类指标类型

k6 的一切测量结果都归结为**指标（Metric）**，而所有指标只有四种类型，理解这四个原语就理解了 k6 的整个结果模型：

**Counter（计数器）**：单调递增的累计值。k6 按测试时长计算速率（`count/s`）并在结束时给出总数。典型如 `http_reqs`（总请求数）、`iterations`（总迭代数）、`data_sent`（发送字节数）。

**Gauge（瞬时值）**：可增可减的当前值，结束时输出最后值、最小值、最大值。典型如 `vus`（当前活跃 VU 数）、`go_goroutines`（运行时内部指标）。

**Rate（比率）**：布尔事件的通过比例，值域 0-1，输出 `rate`（总体通过率）与非零个数。典型如 `http_req_failed`（请求失败率）、`checks`（检查通过率）。

**Trend（分布）**：一组数值的分布统计，输出 avg/min/med/max 与任意分位数 p(N)。这是压测中信息量最大的类型——延迟类指标全部是 Trend，因为**平均值会掩盖长尾，分位数才是用户体验的真实度量**。

```text
Counter  ▮▮▮▮▮▮▮▮▮▮→ 只增不减，看速率与总量
Gauge    ▂▅▇▃▆▂▄▁▃→ 上下波动，看瞬时水位
Rate     0.987    → 通过/失败二值流，看比例
Trend    ▁▂▃▅▇█   → 数值分布，看均值与 p(90)/p(95)/p(99) 长尾
```

## 1.2 常见内置指标

内置指标按前缀分组，HTTP 相关的一组最重要，因为它们的分段 timing 与第 4 篇的 `res.timings` 一一对应：

| 指标 | 类型 | 含义 |
|------|------|------|
| `http_req_duration` | Trend | 请求总耗时（blocked 之后到 receiving 结束） |
| `http_req_blocked` | Trend | 客户端排队等待连接的时间 |
| `http_req_connecting` | Trend | TCP 建连耗时（复用连接时为 0） |
| `http_req_tls_handshaking` | Trend | TLS 握手耗时 |
| `http_req_sending` | Trend | 发送请求体耗时 |
| `http_req_waiting` | Trend | TTFB，等待首字节 ≈ 服务端处理时间 |
| `http_req_receiving` | Trend | 接收响应体耗时 |
| `http_req_failed` | Rate | 请求失败率（需配 `setResponseCallback` 才含 4xx/5xx） |
| `http_reqs` | Counter | 总请求数（除以时长即 RPS） |
| `iterations` | Counter | 总迭代数 |
| `iteration_duration` | Trend | 单次迭代耗时（含 sleep 与批处理） |
| `checks` | Rate | check 断言通过率 |
| `vus` / `vus_max` | Gauge | 当前 / 峰值 VU 数 |
| `data_sent` / `data_received` | Counter | 收发字节数 |

> **注意**：`http_req_failed` 默认**只统计传输层失败**（连接错误、超时），HTTP 4xx/5xx 状态码不算失败——k6 认为"服务器明确回了 500"与"根本没连上"是两类事件。要把状态码纳入失败率，必须注册回调：`import { setResponseCallback } from "k6/http"; setResponseCallback(http.expectedStatuses(200, 302));`。不知道这一点的团队常对着 0% 的错误率自我感觉良好，而实际服务端正在批量吐 500——业务正确性应靠 check 或该回调兜底。

## 1.3 指标采样与统计

Trend 指标的分位数由 **t-digest 算法流式估算**：k6 不在内存里保存每个样本，而是维护一个有界的概要结构，因此 p(95)/p(99) 是近似值（误差通常在 1% 量级），高并发下内存占用恒定。这个设计取舍值得知道：**分位数不可跨"聚合边界"精确合并**——把测试拆成多段分别输出再人工平均 p(95) 是统计错误（分位数不满足可加性），要合并就合并原始数据流重新估算。summary 默认展示 `avg min med max p(90) p(95)`，用 `--summary-trend-stats` 或 options 的 `summaryTrendStats` 自定义，例如加入 `p(99)` 与 `count`。**高并发场景永远看分位数不看平均值**：1000 个请求里 950 个 50ms、50 个 5s，平均值漂亮地显示 297ms，而 p(95)=5s 才是用户感受到的真相。

# 2. Checks 机制

## 2.1 check 用法

`check` 是 k6 的**软断言**：接收一个值与一组"描述 → 布尔函数"，返回整体是否全部通过，成败计入 `checks` 指标（Rate 类型）：

```javascript
import http from "k6/http";
import { check, sleep } from "k6";

export default function () {
  const res = http.get("https://api.example.com/items/1");

  const ok = check(res, {
    "status is 200": (r) => r.status === 200,          // 状态码断言
    "latency < 300ms": (r) => r.timings.duration < 300, // 延迟断言
    "返回合法 JSON": (r) => r.json("data.id") !== undefined, // 业务字段断言
  });

  if (!ok) {
    // check 失败不中断迭代，是否 fail 由业务逻辑决定
    console.warn(`iteration ${__ITER} checks failed`);
  }
  sleep(1);
}
```

check 的第二个参数是**任意 JS 值**（通常是 Response，也可以是 JSON 对象、数组），断言函数接收该值返回布尔。多个断言短路求值：第一个失败后仍会继续评估其余项（全部结果都计入），所以一次 check 调用能同时报告多个失败维度。

## 2.2 checks 与 thresholds 的本质区别

**check 失败不影响测试通过与否，threshold 被突破才决定成败**——这是 k6 结果模型里最重要的一条分界线：

| 维度 | checks | thresholds |
|------|--------|------------|
| 性质 | 迭代内软断言 | 测试级硬阈值 |
| 失败后果 | `checks` 指标通过率下降，迭代继续 | 测试判定失败，退出码 99 |
| 求值时机 | 每次调用即时 | 测试结束时（或 abortOnFail 持续评估） |
| 作用对象 | 单次响应/数据 | 指标聚合值（p(95)、rate） |
| CI 语义 | 参考信息（除非再对 checks 设阈值） | 质量闸门 |

checks 本身是 Rate 指标，因此**可以对 checks 设阈值**把软断言升级成硬闸门：`thresholds: { checks: ["rate>0.99"] }`。这是把"业务正确性"纳入 CI 判定的标准做法——只对 `http_req_failed` 设阈值会漏掉"HTTP 200 但响应体是错误页"这类静默失败。

> **行业实践**：成熟团队的阈值组合通常是三层——`http_req_failed`（传输与状态码错误）、`checks`（业务正确性）、`http_req_duration` 的 p(95)/p(99)（性能 SLO），三者任一突破都红灯。只设延迟阈值是最常见的缺口：系统可以在 P95 达标的同时把一半请求返回成 500。

# 3. Tags 与 Groups

## 3.1 tags 的作用

**tag 是指标的维度标签**，附加后该请求/迭代产生的所有指标都带上这组键值，从而支持按维度聚合与过滤：

```javascript
import http from "k6/http";

export default function () {
  // 请求级 tag：区分同一脚本里的不同接口
  http.get("https://api.example.com/items", {
    tags: { endpoint: "list_items", scenario: "browse" },
  });
}

export const options = {
  scenarios: {
    browsing: { executor: "constant-vus", vus: 10, duration: "1m",
                tags: { flow: "browse" } },        // scenario 级 tag，覆盖其全部指标
  },
  thresholds: {
    // 只对打了 endpoint=order 标签的请求设更严格的阈值
    "http_req_duration{endpoint:order}": ["p(95)<800"],
    "http_req_failed{flow:browse}": ["rate<0.01"],
  },
};
```

阈值语法 `metric{tag:value}` 就是标签过滤的应用。**tag 值必须低基数**——用 `user_id`、`order_no` 这类唯一值做 tag 会让每个请求生成一条独立时间序列，输出到 Prometheus/InfluxDB 时直接打爆时序库（k6 官方将此类事故称为 cardinality explosion）。维度标签的正确粒度是接口名、场景名、区域这类有限枚举。

## 3.2 group 的作用

`group` 把一段业务流程聚合成命名分组，**组内所有指标自动获得 `group:::路径` 标签**，用于按业务步骤统计整体性能：

```javascript
import { group } from "k6";
import http from "k6/http";
import { sleep } from "k6";

export default function () {
  group("下单流程", function () {
    group("加入购物车", function () {
      http.post("https://api.example.com/cart", { sku: "A1" });
    });
    group("提交订单", function () {
      http.post("https://api.example.com/orders", { sku: "A1" });
    });
    sleep(1);
  });
}

export const options = {
  thresholds: {
    // group 嵌套路径用 ":::" 连接，可对整段流程设阈值
    "http_req_duration{group:::下单流程:::提交订单}": ["p(95)<1000"],
    "checks{group:下单流程}": ["rate>0.95"],
  },
};
```

group 与 tag 的分工：**group 表达"流程结构"**（有层级、影响 summary 展示分组），**tag 表达"数据维度"**（平铺、供后端聚合）。两者可叠加。group 名同样忌动态拼接（`group(\`user-${id}\`)` 是基数爆炸的又一来源）。

# 4. 阈值（Thresholds）

## 4.1 语法与示例

阈值声明在 options 中，键是指标名（可带标签过滤），值是**阈值表达式字符串数组**。表达式语法为 `聚合函数 比较符 数值`：

```javascript
export const options = {
  thresholds: {
    // Trend 指标：对分布统计量设限
    http_req_duration: ["p(95)<500", "avg<200"],

    // Rate 指标：对比率设限（rate 是 0-1 小数，0.01 = 1%）
    http_req_failed: ["rate<0.01"],
    checks: ["rate>0.99"],

    // Counter 指标：对计数/速率设限
    http_reqs: ["count>1000"],              // 总请求数必须大于 1000（防止压测没真正跑起来）
    errors: ["count<100"],

    // Gauge 指标：对瞬时值设限
    vus: ["value<=50"],

    // 带标签过滤：只约束特定接口
    "http_req_duration{endpoint:checkout}": ["p(95)<800"],

    // 对象形式：附加控制项
    "http_req_duration{scenario:soak}": [
      { threshold: "p(99)<2000", abortOnFail: true, delayAbortEval: "60s" },
    ],
  },
};
```

聚合函数按指标类型取用：Trend 支持 `avg`/`min`/`med`/`max`/`p(N)`/`count`；Rate 支持 `rate`（通过率）/`count`；Counter 支持 `count`；Gauge 支持 `value`（最后值）/`min`/`max`。`p(N)` 的 N 是任意百分位——`p(95)<300` 读作"P95 延迟必须小于 300 毫秒"。**任一阈值在测试结束时不满足，k6 以退出码 99 结束**，CI 据此判定失败；`abortOnFail: true` 则让 k6 在测试进行中就持续评估并提前中止（`delayAbortEval` 给评估一个宽限期，避免开局抖动误杀）。

## 4.2 设计原则

**先有基线再设阈值**：对新服务先跑一轮探索性负载测试，记录当前 p(95) 与错误率，再据此设定"不退化"阈值（如基线 × 1.2），而不是抄别人的 `p(95)<300`。**关键链路阈值更严格**：核心交易接口的 p(99) 应比边缘查询接口严格数倍，用标签过滤分别声明。**阈值即 SLO 的代码化**：写不出阈值的团队通常也说不清自己的 SLO，阈值声明过程本身就是逼团队量化目标的仪式。另外注意 `http_reqs: ["count>N"]` 这类**下限阈值**的价值——它防的不是"系统太慢"而是"测试根本没生效"（目标地址配错、脚本逻辑短路导致零请求，其余阈值全部空转通过）。

## 4.3 常见阈值配方

```javascript
export const options = {
  thresholds: {
    // 配方一：API SLO 三件套（最常用）
    http_req_failed: ["rate<0.01"],                 // 错误率 < 1%
    http_req_duration: ["p(95)<500"],               // P95 < 500ms
    checks: ["rate>0.99"],                          // 业务断言通过率 > 99%

    // 配方二：严格核心链路
    "http_req_duration{endpoint:payment}": ["p(99)<1000", "p(95)<600"],

    // 配方三：稳定性测试（长时运行，关注错误与资源）
    http_req_failed: [{ threshold: "rate<0.05", abortOnFail: true, delayAbortEval: "120s" }],
    vus: ["value<=100"],

    // 配方四：验证测试确实执行（配合任意配方使用）
    http_reqs: ["count>10000"],
    iterations: ["count>100"],
  },
};
```

# 5. 自定义指标

## 5.1 四种自定义指标

`k6/metrics` 模块导出四个构造函数，参数是指标名（须全局唯一，输出到后端时即序列名）：

```javascript
import { Trend, Rate, Counter, Gauge } from "k6/metrics";
import http from "k6/http";
import { sleep } from "k6";

// 必须在 init context（模块顶层）创建——运行期重复 new 同名指标会报错
const orderLatency = new Trend("order_latency", true);   // true = 时间单位（ms）
const bizFailure = new Rate("biz_failure");              // 业务失败率
const ordersCreated = new Counter("orders_created");     // 成功下单计数
const activeCarts = new Gauge("active_carts");           // 瞬时购物车数

export default function () {
  const start = Date.now();
  const res = http.post("https://api.example.com/orders", { sku: "A1" });

  orderLatency.add(Date.now() - start, { step: "create" });  // add(值, 可选标签)
  ordersCreated.add(res.status === 201 ? 1 : 0);
  bizFailure.add(res.status !== 201);                        // Rate.add(布尔)
  activeCarts.add(res.json("cartCount") ?? 0);
  sleep(1);
}

export const options = {
  thresholds: {
    // 自定义指标与内置指标享受完全相同的阈值待遇
    order_latency: ["p(95)<1200"],
    biz_failure: ["rate<0.02"],
  },
};
```

`Trend` 构造函数第二个参数 `isTime=true` 让 summary 以时间格式（ms/s）展示数值，延迟类自定义指标务必开启。**自定义指标的典型用途**是测量 k6 内置指标覆盖不到的业务量：消息队列消费延迟、批处理作业耗时、业务成功率——把"HTTP 层正常"与"业务层正常"分开度量。

## 5.2 采集与性能影响

每次 `add()` 都要跨越 JS→Go 边界并进入聚合引擎，单条开销微秒级但**高频调用累积可观**：万级 RPS 下每个请求打 5 个自定义指标，聚合压力与基数（tag 组合数）都会侵蚀压测精度。纪律与 tag 一致：**指标名与 tag 值必须是有限枚举**，禁止把 ID 类动态值放进 add 的第二个参数；能用一个 Trend + tag 区分步骤的，不要建十个指标。若发现开启自定义指标后 `http_reqs` 速率明显下降，先砍指标粒度再怀疑被测系统。

# 6. 结果输出与分析

## 6.1 summary 输出与归档

测试结束时 k6 在 stderr 打印 **end-of-test summary**：按指标与 group 组织的统计表，Trend 显示 `summaryTrendStats` 指定的各统计量，阈值行标注 ✓/✗。机器可读的归档方式有三种：`--summary-export=summary.json`（汇总表快照）、`--out json=results.json`（逐条指标事件流，含全部 tag，供入库分析）、`--out csv=results.csv`（扁平化 CSV，适合 Excel 快速查看）。长期趋势分析的正路是 `--out experimental-prometheus-rw` 或 InfluxDB 接 Grafana 面板，把每次压测的指标与构建号 tag 关联存档，让"这个版本比上个版本 P95 恶化了 15%"可以被自动发现。

```bash
# CI 标准输出组合：汇总表 + JSON 归档 + 阈值判定
k6 run --summary-export=summary.json \
       --out json=results-$BUILD_NUMBER.json \
       --tag commit=$GIT_SHA \
       script.js
```

## 6.2 指标解读方法

指标组合的形状比单个数值更有诊断价值，几组经典模式的读法：**延迟上升且吞吐（http_reqs 速率）下降**——服务端到达瓶颈或触发限流，请求在排队；**延迟上升但吞吐不变**——服务端处理变慢但容量未饱和（慢查询、GC），继续加压会雪崩；**`http_req_blocked` 升高**——压测机连接池排队或 fd/端口耗尽，客户端瓶颈（对照第 4 篇 6.3 节）；**`http_req_waiting` 变长而 connecting/receiving 正常**——纯服务端处理时间劣化，直接查应用与数据库；**`checks` 通过率下降而 `http_req_failed` 为 0**——服务返回 200 但内容错误，典型的"静默失败"，查业务逻辑而非网络。解读顺序建议固定为：先看测试是否有效（http_reqs 下限阈值、Insufficient VUs 警告），再看错误面（http_req_failed、checks），最后看延迟分布（duration 及各分段 timing）。

> **注意**：summary 里的 `iterations` 与 `http_reqs` 比值异常（如 1 次迭代产生 20 个请求）往往暴露脚本问题——重定向风暴、意外的 batch 展开或重试循环失控。读懂指标的前提是先确认负载模型本身符合预期。

# 7. 小结

k6 的结果模型建立在四种指标原语上：Counter 计数、Gauge 水位、Rate 比率、Trend 分布，HTTP 请求的六段 timing 全部以 Trend 呈现，分位数用 t-digest 流式估算因此不可跨段合并。**checks 是迭代内软断言，thresholds 是测试级硬闸门**，前者失败不改变测试结论，后者突破即退出码 99——把 `checks` 指标本身纳入阈值是业务正确性进 CI 的标准姿势。tags 与 groups 提供指标维度，但基数必须受控。阈值表达式 `p(95)<500`、`rate<0.01` 是 SLO 的代码化，配方上要同时覆盖错误率、业务断言、延迟分布与"测试确实执行"的下限校验。自定义 Trend/Rate/Counter/Gauge 补足业务量度量，最终经 summary/json/Prometheus 归档，让每次压测成为可对比、可追责的历史数据点。
