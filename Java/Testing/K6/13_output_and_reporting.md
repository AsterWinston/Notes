# 1. 输出机制概览

## 1.1 内置输出

**k6 的指标输出是一个"一进多出"的管线：脚本产生的每条指标样本，可以同时流向控制台和任意多个 `--out` 后端**。v0.5x 内置的输出能力包括：

| 输出 | 启用方式 | 内容 | 典型用途 |
|------|---------|------|---------|
| 控制台 summary | 默认 | 测试结束时的汇总统计 | 本地调试、CI 日志留档 |
| `--out json` | `--out json=file.json` | 每条指标样本一行 JSON | 事后离线分析、自建报表 |
| `--out csv` | `--out csv=dir/` | 样本写 CSV（含 header 与 meta 行） | 用 Excel/pandas 做对比分析 |
| `--out experimental-prometheus-rw` | Remote Write URL | 聚合指标写 Prometheus 系 | 对接 Grafana 观测（v0.5x 内置，见第 14 篇） |
| `--out experimental-csv` | 同 csv | 新版 CSV 实现 | 高吞吐场景 |
| 扩展输出 | `--out kafka=` 等 | 由 xk6 扩展提供 | 自研指标管道（见第 12 篇） |

`--out` 可以叠加多次，各后端互不影响：

```bash
k6 run script.js \
  --out json=results/run-42.json \
  --out csv=results/csv-42 \
  --out experimental-prometheus-rw=http://prom:9090/api/v1/write
```

## 1.2 输出场景

**不同阶段该看不同的输出，混用是新手最常见的工程混乱来源**。本地调试阶段只需要控制台：跑几十秒、盯 end-of-test summary、改脚本再跑，任何文件输出都是负担。CI 归档阶段的核心诉求是**可追溯与可判定**——JSON 导出供趋势对比，summary 进日志供人读，阈值退出码供流水线判断成败。持续可视化阶段则把指标推进 Prometheus/Grafana，压测进行中就能对照被测系统的资源曲线看（见第 14 篇）。一个原则：**调试用 stdout，判定用退出码，归档用 JSON，观察用监控系统**，四种输出各司其职。

# 2. Summary 与自定义报告

## 2.1 控制台 summary 的结构

**end-of-test summary 是 k6 默认输出的"最终成绩单"，理解它的结构是解读一切报告的前提**。一次带阈值与 check 的测试结束后，控制台输出形如：

```text
     █ GET /api/orders

       ✓ status is 200
       ✗ body has orderId
        ↳  2% — ✓ 98 / ✗ 2

   scenarios: (100.00%) 1 scenario, 20 max VUs, 40s max duration (incl. graceful stop):
            * default: 30s looping VUs=20 (maxVUs 20), exec: default, finished: 0

     data_received................: 1.2 MB   41 kB/s
     data_sent....................: 340 kB  11 kB/s
     http_req_duration............: avg=45.2ms min=12ms med=38ms max=410ms
                                    p(90)=82ms p(95)=120ms
     http_req_failed..............: 0.85%  ✓ 17   ✗ 1983
     http_reqs....................: 2000   68.1/s
     iteration_duration...........: avg=51ms
     vus..........................: 20     min=20 max=20

     ✓ http_req_duration...............: threshold 'p(95)<200'
     ✗ http_req_failed.................: threshold 'rate<0.005'
```

各区块的含义：check 区块给出每个检查点的通过率；scenarios 行确认执行计划是否按预期跑满；指标区块按字母序列出所有内置与自定义指标，每行含聚合统计（count/rate/avg/分位数）；阈值区块用 ✓/✗ 逐条给出达标判定。**阈值任何一条 ✗，进程退出码为 99**——这是 summary 与 CI 的直接连接点（见 2.4）。

## 2.2 handleSummary 自定义报告

**`handleSummary(data)` 是 k6 提供的报告钩子：测试结束时 k6 把完整的汇总数据对象交给它，它返回"路径 → 内容"的映射，k6 据此写文件或写 stdout**。返回多个键即同时产出多份报告。这是实现 HTML/Markdown/JSON 多格式报告的官方机制，也是接管默认控制台 summary 的唯一方式（定义了 handleSummary，默认 summary 就不再输出，除非自己写回 `stdout`）：

```javascript
// summary 报告脚本：同时产出 JSON、Markdown、HTML 三份
import http from "k6/http";
import { textSummary } from "https://jslib.k6.io/k6-summary/0.0.2/index.js";

export const options = {
  vus: 10,
  duration: "30s",
  thresholds: {
    http_req_duration: ["p(95)<500"],   // 95 分位延迟阈值
    http_req_failed: ["rate<0.01"],     // 错误率阈值
  },
};

export default function () {
  http.get("https://test.k6.io");
}

export function handleSummary(data) {
  const runId = __ENV.RUN_ID || "local";
  return {
    // 1. 结构化 JSON：机器可读，供 CI 归档与趋势对比
    [`reports/summary-${runId}.json`]: JSON.stringify(data, null, 2),

    // 2. Markdown：直接贴进 PR 评论或测试报告
    [`reports/summary-${runId}.md`]: renderMarkdown(data),

    // 3. HTML：官方 jslib 模板生成的可读报告
    [`reports/summary-${runId}.html`]: textSummary(data, { indent: " ", enableColors: false }),

    // 4. 保留控制台输出，否则终端什么都不显示
    stdout: textSummary(data, { indent: " ", enableColors: true }),
  };
}

// 从 summary 数据里挑关键字段拼一份精简 Markdown
function renderMarkdown(data) {
  const p95 = data.metrics.http_req_duration.values["p(95)"].toFixed(1);
  const failRate = (data.metrics.http_req_failed.values.rate * 100).toFixed(2);
  const rps = data.metrics.http_reqs.values.rate.toFixed(1);
  return [
    `# 压测报告`,
    ``,
    `| 指标 | 值 |`,
    `|------|-----|`,
    `| 请求总量 | ${data.metrics.http_reqs.values.count} |`,
    `| RPS | ${rps} |`,
    `| p(95) 延迟 | ${p95} ms |`,
    `| 错误率 | ${failRate}% |`,
  ].join("\n");
}
```

`data` 对象的结构要点：`data.metrics` 按指标名索引，每个指标含 `type`（counter/gauge/rate/trend）、`contains`（time/data/default）与 `values`（counter 有 `count`/`rate`，trend 有 `avg`/`min`/`max`/`med`/`p(90)`/`p(95)`，rate 有 `passes`/`fails`/`rate`）；`data.root_group` 是 check 的树状分组；`data.options` 回显本次执行配置；`data.state` 含 VU 数、测试时长等运行时状态。**handleSummary 返回的内容写文件时按字面字符串写入，路径相对于工作目录，目录不存在会报错**——CI 里先 `mkdir -p reports`。

> **行业实践**：成熟的 CI 集成通常三管齐下——`summary.json` 作为 artifact 上传（供历史趋势脚本消费）、Markdown 摘要由机器人贴到 PR 评论（让本次变更的性能影响对 reviewer 可见）、退出码决定流水线红绿。只留控制台输出的做法在流水线里等于"结果靠人翻日志"，不可持续。

## 2.3 summary 导出的两种方式对比

除 `handleSummary` 外，`--summary-export` 是更轻量的官方开关：

```bash
# 方式一：命令行开关，导出默认结构的 JSON，零代码
k6 run --summary-export=summary.json script.js

# 方式二：handleSummary，可编程，格式完全自定义
k6 run script.js   # 报告逻辑在脚本内
```

| 维度 | --summary-export | handleSummary |
|------|-----------------|---------------|
| 侵入性 | 零代码，纯 CLI | 脚本内定义函数 |
| 格式 | 固定 JSON | 任意（HTML/MD/JSON/多文件） |
| 能否替代控制台 summary | 否（两者共存） | 是（定义后默认 summary 消失） |
| 适用 | 临时导出、脚本不归你管 | 报告是工程资产、需要多格式 |

## 2.4 阈值失败与退出码

**k6 的退出码是 CI 判定压测成败的契约**：0 表示正常结束；**99 表示至少一条 threshold 失败**（脚本本身跑完了，是"性能不达标"）；非 0 非 99（如 1、2）表示脚本错误或运行时故障。区分 99 与其他非 0 至关重要——前者是"被测系统性能退化"，应该阻断发布并通知性能负责人；后者是"压测脚本/环境问题"，应该通知脚本维护者。CI 里的用法：

```bash
set +e                                # 不让 k6 的非零退出直接终止脚本
k6 run script.js --out json=res.json
code=$?
set -e
if [ $code -eq 99 ]; then
  echo "阈值未达标，性能回归！" && exit 1
elif [ $code -ne 0 ]; then
  echo "k6 执行故障（脚本或环境问题）" && exit 2
fi
```

另一个易踩的坑：测试中途发生**脚本级错误**（如 init 阶段抛异常）时 k6 也会以非 0 退出，但阈值根本没被评估——流水线里应先确认"测试确实跑完"（summary.json 存在且含 `state.test_run_duration_ms`）再看阈值结果，避免把"没测"当成"测过且不达标"。

# 3. 输出指标选择

## 3.1 关键指标优先级

**报告的骨架应该是"延迟分位数 + 错误率 + 吞吐"三元组，且延迟永远看分位数不看均值**。均值会把长尾抹平——p(99) 已经 3 秒的系统，avg 可能仍然只有 80ms。优先级排序：`http_req_duration` 的 p(90)/p(95)/p(99)（用户体感的直接代理）、`http_req_failed`（正确性底线）、`http_reqs` 的 rate（吞吐，用来标定"上述延迟是在多大压力下测得的"）。三者必须**成对呈现**：单独说"p95 = 200ms"没有意义，要说"在 68 RPS、错误率 0.8% 的条件下 p95 = 200ms"。在此之上叠加业务指标（下单成功率、库存扣减正确性）与资源侧指标（被测机 CPU/内存），才构成完整证据链。

## 3.2 标签化输出

**tags 是控制指标聚合维度的机制，用对了能按接口/场景区分，用错了会撑爆时序数据库**。k6 每条样本都带标签（`name`、`status`、`scenario`、`url` 等内置标签 + 用户自定义标签），JSON/CSV 输出保留全部标签，Prometheus RW 输出则按标签生成时间序列。按业务接口聚合的做法：

```javascript
import http from "k6/http";
import { Counter } from "k6/metrics";

// 业务级自定义指标：下单失败次数
const orderFail = new Counter("order_fail");

export default function () {
  // tags 让同一个 http_req_duration 能按 api 维度拆分统计
  http.get("https://shop.example.com/api/cart", {
    tags: { api: "cart" },
  });

  const res = http.post("https://shop.example.com/api/orders", "{}", {
    tags: { api: "order" },
  });
  if (res === null || res.status !== 200) {
    orderFail.add(1, { api: "order" });
  }
}

// 阈值也可以只作用于某个标签维度
export const options = {
  thresholds: {
    "http_req_duration{api:order}": ["p(95)<800"],   // 下单接口单独约束
    "http_req_duration{api:cart}": ["p(95)<300"],    // 购物车接口更严
  },
};
```

> **注意**：`url` 内置标签默认是**未参数化的完整 URL**——REST 风格路径（`/api/users/12345`）会让每个 ID 成为一条独立时间序列，即所谓**基数爆炸**。对策：用命名请求 `http.get(url, { tags: { name: "/api/users/:id" } })` 或设置 `options.noConnectionReuse` 之外的 `--tag` 黑白名单，输出到 Prometheus 时用扩展的 `K6_PROMETHEUS_RW_TREND_STATS` 与标签过滤配置控制维度。

# 4. 结果解释与呈现

## 4.1 指标与结论

**一份合格的压测报告必须包含"是否达标"的明确判断，而判断只能来自事先定义的阈值**。没有阈值的报告只是数据罗列——"p95 是 350ms"本身不构成结论，"p95 350ms，超出 300ms 的 SLO，不通过"才是。这要求阈值在压测**之前**根据 SLO/容量规划定好（见第 9 篇的测试设计），而不是看到结果后再挑一个"能过"的数——后者是自欺欺人的常见形态。报告结构建议固定为：测试意图与场景描述 → 执行环境（二进制版本、施压机、被测版本）→ 三元组指标 + 阈值判定 → 异常观察（错误样本、延迟毛刺时刻）→ 结论与建议。

## 4.2 可视化建议

**Grafana 面板按业务流程组织，而不是按指标类型组织**。"所有接口的 p95 一张图"看起来整齐，排障时却没法用；"下单流程：延迟 + 错误率 + RPS + 应用 CPU 同屏"才能一眼看出"错误率抬头是否伴随资源打满"。趋势类报告关注两件事：**跨版本对比**（同一场景本次 vs 上次基线，k6 JSON 归档 + 简单脚本即可画趋势线）与**压测进行中的实时曲线**（延迟随时间的漂移往往揭示连接池耗尽、GC 停顿、缓存失效这类时间相关问题，而 end-of-test summary 把它们全部平均掉了）。

# 5. 常见问题

## 5.1 输出过大

**JSON/CSV 输出是每样本一行，规模 = 迭代数 × 每迭代请求数 × 指标数，大压测下轻松上 GB**。控制手段：其一带 `--out json` 时配合 `K6_SUMMARY_TIME` 等无济于事——JSON 输出没有内置采样，真正有效的是**只输出自定义指标**（用 Trend/Counter 记录关心的业务量，不开全量 http 明细输出）；其二控制标签维度（3.2 的基数问题在文件输出里表现为列爆炸，在时序库里表现为存储爆炸）；其三长时间 soak 测试用 Prometheus RW 这类**预聚合**输出替代逐样本文件。

## 5.2 输出格式不一致

**多人多脚本产出的报告格式漂移，会让归档数据无法程序化对比**。治理方式：把 `handleSummary` 抽成共享模块（团队内的 report 库，统一 JSON schema 与 Markdown 模板），所有脚本 import 同一实现；CI 中报告文件名带 run id 与 git sha，目录结构固定；JSON schema 变更走版本字段（`data` 里塞 `"schema": "v2"`），下游解析器按版本适配。

# 6. 小结

k6 的输出体系分四层：**控制台 summary 给人看，退出码（99 = 阈值失败）给 CI 判定，JSON/CSV 给归档与离线分析，Prometheus RW 等后端给实时可视化**。`handleSummary` 是自定义报告的核心钩子，返回"路径 → 内容"映射即可同时产出 HTML/Markdown/JSON 多格式，注意定义后默认控制台 summary 会被接管。报告解读坚持"延迟分位数 + 错误率 + 吞吐"三元组成对呈现、阈值先于测试定义、tags 控制聚合维度并警惕 URL 基数爆炸。CI 集成区分 99 与其他非 0 退出码，并先验证测试确实完成再采信阈值结论。
