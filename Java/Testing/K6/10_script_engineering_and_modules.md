单个几百行的 k6 脚本能跑，不代表一个团队几十条业务线、上百个压测场景的脚本库能维护。当同一段登录逻辑在十个脚本里复制了十遍、当测试地址散落在各处硬编码、当老板要一份自定义格式的压测报告时，脚本工程化的欠账就集中爆发。本章讲 k6 的模块系统与它的限制、如何用 ES Modules 做分层拆分与公共封装、`handleSummary` 自定义汇总、`__ENV` 配置管理，以及脚本纳入版本控制与 CI 复用的实践。

# 1. 脚本工程化目标

## 1.1 可维护性

**可维护性的核心是分层抽象：协议层、业务层、场景层各司其职，改动被隔离在单一层级。**协议层封装"怎么发请求"——统一的 HTTP 客户端、超时、重试、基础断言、鉴权头注入；业务层封装"业务流程是什么"——登录、加购、下单这类由多个请求组成的用户动作；场景层只描述"什么负载跑什么流程"——执行器参数、场景组合、阈值。分层的好处用一次变更就能体会：全站接口加了签名头，只改协议层一个文件；下单流程插入风控步骤，只改业务层；大促要把负载翻倍，只动场景层的 options，业务代码零改动。

反面形态是"面条脚本"：请求、断言、数据、配置、流程全部平铺在一个文件里，复制粘贴出十几个变体。它的维护成本不是线性增长而是组合爆炸——一处公共逻辑变更要同步 N 个文件，漏改的那个在下一次压测里产出错误数据，且没人知道它错了。

## 1.2 可复用性

**复用的单位是"业务动作"而不是"请求"：封装到用户操作粒度（login、placeOrder），调用方才能像搭积木一样组合场景。**k6 生态为此提供了两条现成路径：官方 `jslib`（CDN 引入的公共库，如 papaparse、cheerio）与 **k6 扩展包机制**——把公共业务模块发布成 npm 包或私有 Git 仓库，脚本里 `import` 使用（如 `@grafana/k6-utils`）。团队内自建的公共库通常包含：带默认超时与断言的 HTTP 客户端、统一的登录/token 管理、数据加载工具（SharedArray 包装）、环境配置解析、自定义指标工厂。复用性的检验标准很朴素：**新写一个业务线的压测脚本，理想状态下只需要写业务动作和场景配置，协议与工具代码全部 import。**

# 2. 模块化设计

## 2.1 ES Modules 与 k6 的模块限制

**k6 使用 Goja 实现 ECMAScript 5.1 + 大部分 ES6 语法，模块系统是 ES Modules（`import`/`export`），但与 Node.js 存在几处关键差异。**写工程化脚本前必须记住这张限制清单：

| 能力 | Node.js | k6 | 说明 |
|------|---------|-----|------|
| ES Modules（import/export） | 支持 | 支持 | k6 的主力模块方式，静态解析 |
| CommonJS（require） | 支持 | **不支持** | npm 上的 CJS 包不能直接 import |
| Node 内置模块（fs/path） | 支持 | **不支持** | 文件读取只有 `open()`，且限 init 上下文 |
| 动态 import()/require | 支持 | **不支持** | 模块路径必须是静态字符串字面量 |
| npm 依赖 | 原生 | 仅纯 JS 且 ESM 兼容的包 | CJS 包需先用 webpack/esbuild 打包转换（见第 11 篇） |
| URL 远程模块 | 无 | 支持（jslib/HTTPS） | `import x from "https://jslib.k6.io/..."` |

**动态 import 不被支持**这条对工程化的影响最大：不能根据运行时条件加载不同模块，"按环境切换实现"只能靠 `__ENV` 变量 + 条件分支在同一模块内完成，或者在构建期用打包器处理。另外，每个模块顶层代码在每个 VU 的 init 上下文各执行一遍（第 6 篇），模块里做重初始化（大 JSON 解析、SharedArray 构建）时要意识到成本按 VU 计——这正是公共数据加载模块统一走 SharedArray 的原因。

## 2.2 目录组织建议

**目录结构按"层"组织而不是按"项目"组织，公共资产（lib/data/config）与场景脚本分离：**

```text
k6/
├── lib/                        # 协议层 + 工具（跨业务复用）
│   ├── http_client.js          # 统一 HTTP 封装：超时、断言、鉴权
│   ├── auth.js                 # 登录与 token 管理
│   ├── metrics.js              # 自定义指标工厂
│   └── data.js                 # SharedArray 数据加载工具
├── flows/                      # 业务层（按业务域组织用户动作）
│   ├── order_flow.js           # 下单流程：加购 → 创建订单 → 支付
│   └── user_flow.js            # 用户流程：注册 → 登录 → 改资料
├── scenarios/                  # 场景层（可直接 k6 run 的入口脚本）
│   ├── order_load_test.js      # 订单链路负载测试
│   ├── order_stress_test.js    # 订单链路压力测试
│   └── smoke_test.js           # 全链路冒烟
├── data/                       # 参数化数据文件（大文件走 Git LFS）
│   ├── users.json
│   └── products.csv
├── config/
│   └── env.js                  # 环境配置解析（读 __ENV，给默认值）
├── package.json                # 依赖 jslib 工具包 / 构建脚本（TS 时用）
└── README.md                   # 数据生成方式、运行命令、报告说明
```

**设计考量**：场景层的每个文件都应该是"一次完整测试"的入口——`k6 run scenarios/order_load_test.js -e ENV=staging` 一条命令可执行；业务层与协议层永远不被直接运行。依赖方向严格单向（scenarios → flows → lib），禁止 lib 反向 import flows，这是防止"依赖层级过深"（见 5.1）的结构保证。

## 2.3 封装请求与断言

**协议层封装的价值在于把团队的断言纪律变成代码默认值：超时、状态码检查、错误分类写一次，所有脚本自动继承。**

```javascript
// lib/http_client.js —— 协议层：统一请求封装
import http from "k6/http";
import { check } from "k6";
import { Trend } from "k6/metrics";
import { getConfig } from "../config/env.js";

const config = getConfig();                 // init 上下文解析环境配置
const businessErrors = new Trend("business_errors", true); // 业务错误单独统计

export function getJson(path, params = {}) {
  const res = http.get(`${config.baseUrl}${path}`, withDefaults(params));
  assertOk(res, `GET ${path}`);             // 统一断言，失败自动分类计数
  return res.json();
}

export function postJson(path, body, params = {}) {
  const res = http.post(
    `${config.baseUrl}${path}`,
    JSON.stringify(body),
    withDefaults({ ...params, headers: { "Content-Type": "application/json",
      ...(params.headers || {}) } })
  );
  assertOk(res, `POST ${path}`);
  return res.json();
}

function withDefaults(params) {             // 注入全局默认：超时、鉴权头
  return {
    timeout: config.timeout,                // 默认 "10s"，环境可覆盖
    ...params,
    headers: { ...(config.authHeader && { Authorization: config.authHeader }),
      ...(params.headers || {}) },
  };
}

function assertOk(res, label) {             // 断言纪律集中在此
  const ok = check(res, {
    [`${label} status 2xx`]: (r) => r.status >= 200 && r.status < 300,
  });
  if (!ok) businessErrors.add(1, { endpoint: label });
}
```

```javascript
// flows/order_flow.js —— 业务层：组合协议层原语成用户动作
import { getJson, postJson } from "../lib/http_client.js";

export function placeOrder(user, sku) {     // 一个完整的下单动作
  const cart = postJson("/cart", { sku, qty: 1 });
  const order = postJson("/orders", { cartId: cart.data.cartId });
  return order.data.orderId;                // 关联值向上传递（第 8 篇）
}
```

```javascript
// scenarios/order_load_test.js —— 场景层：只剩负载配置与流程编排
import { sleep } from "k6";
import { placeOrder } from "../flows/order_flow.js";
import { getUsers } from "../lib/data.js";

const users = getUsers();                   // SharedArray 加载，见第 8 篇

export const options = {
  scenarios: {
    order: {
      executor: "constant-arrival-rate",
      rate: 200, timeUnit: "1s", duration: "10m",
      preAllocatedVUs: 100, maxVUs: 400,
    },
  },
  thresholds: { "http_req_duration": ["p(95)<500"], "http_req_failed": ["rate<0.01"] },
};

export default function () {
  const user = users[__VU % users.length];
  placeOrder(user, "A-100");
  sleep(1);                                 // 思考时间（第 9 篇 3.2）
}
```

# 3. handleSummary 自定义汇总

**`handleSummary()` 是 k6 留给脚本的"报告出口"：测试结束时 k6 把全部汇总数据作为一个 JSON 对象传给它，返回值决定生成哪些文件、什么格式。**默认的终端表格汇总只是它的内置特例；接管后可以同时输出多种格式，喂给不同消费方：

```javascript
// scenarios/order_load_test.js（追加）
import { textSummary } from "https://jslib.k6.io/k6-summary/0.1.0/index.js";

export function handleSummary(data) {
  // data 结构：data.metrics（每个指标的 count/avg/p95/rate 等）、
  //            data.root_group（check 分组通过率）、data.state（运行状态）
  const p95 = data.metrics.http_req_duration.values["p(95)"];
  const verdict = p95 < 500 ? "PASS" : "FAIL";   // 自定义判定逻辑

  return {
    stdout: textSummary(data, { indent: " ", enableColors: true }), // 终端保留默认表格
    "reports/summary.json": JSON.stringify(data, null, 2),         // 完整数据落盘
    "reports/verdict.txt": `${verdict} p95=${p95.toFixed(1)}ms`,   // CI 可解析的判定文件
  };
}
```

返回对象的键有三种形态：`stdout`/`stderr` 写入标准输出；普通路径写文件；还可以配合 `--summary-export`（旧参数，已被 handleSummary 取代）迁移。工程上的两个典型用法：**给 CI 产判定**——verdict 文件让流水线不依赖 exit code 也能读到"通过与否 + 关键数字"，用于评论回写 PR 或推送 IM；**给报告系统产结构化数据**——summary.json 入库后做历史趋势（与第 9 篇的基线对比机制衔接）。

> **注意**：handleSummary 里抛异常会吞掉正常退出流程，且它运行在独立上下文（测试已结束），不能发请求、不能用 VU 级 API——汇总数据的对外推送应该交给 `--out` 输出通道或 CI 脚本，而不是塞进 handleSummary。

# 4. 配置管理

## 4.1 分环境配置与 __ENV

**`__ENV` 是 k6 的环境变量入口：命令行 `-e KEY=VALUE`、shell 环境变量、CI Secrets 三条注入通道最终都汇聚到它，脚本代码本身不感知环境差异。**工程化的关键是在 config 层给 `__ENV` 包一层解析与默认值，禁止业务代码直接散落 `__ENV.XXX`：

```javascript
// config/env.js —— 环境配置的唯一解析点
const ENVIRONMENTS = {
  dev:     { baseUrl: "http://localhost:8080",  timeout: "5s"  },
  staging: { baseUrl: "https://staging.example.com", timeout: "10s" },
  prod:    { baseUrl: "https://api.example.com",     timeout: "10s" },
};

export function getConfig() {
  const env = __ENV.ENV || "staging";              // 默认打 staging，防误伤生产
  const base = ENVIRONMENTS[env];
  if (!base) throw new Error(`unknown ENV: ${env}`);
  return {
    env,
    ...base,
    // 单变量覆盖：临时改地址不用改代码
    baseUrl: __ENV.BASE_URL || base.baseUrl,
    timeout: __ENV.TIMEOUT || base.timeout,
    authHeader: __ENV.AUTH_TOKEN ? `Bearer ${__ENV.AUTH_TOKEN}` : null,
  };
}
```

```bash
# 运行方式：环境切换只动命令行，脚本零改动
k6 run scenarios/order_load_test.js -e ENV=staging          # 打预发
k6 run scenarios/order_load_test.js -e ENV=dev -e TIMEOUT=30s  # 本地调试，放宽超时
ENV=staging k6 run scenarios/order_load_test.js             # shell 环境变量同样生效
```

**设计考量**：默认环境故意不设成 prod——配置解析层的"防呆默认值"比口头约定可靠得多；对 prod 的压测应当要求显式 `-e ENV=prod` 并配合白名单校验，这个开关值得在 config 层写死一道确认逻辑。

## 4.2 机密管理

**凭证的唯一合法通道是运行时注入：CI 的 Secrets 管理（GitHub Actions Secrets、GitLab CI Variables）在流水线里以 `-e` 或环境变量传给 k6，代码库、数据文件、报告里都不出现明文。**配套的三条纪律：token 不进 `console.log`（日志会进 CI 存档与报告）；数据文件里的账号密码用脱敏占位 + 运行时映射；`handleSummary` 输出的 summary.json 入库前确认不含请求头回显。一旦凭证入库，Git 历史里就永远有它——轮换密钥的成本远高于从一开始就用 Secrets。

# 5. 复用与发布

## 5.1 版本化管理

**压测脚本必须和被测应用一样对待：入 Git、走 PR 评审、打标签、与被测版本建立关联。**版本关联的实用做法：脚本仓库的 release 标签对齐被测应用的重大版本（应用 v2.3 上线前，压测脚本也出 v2.3 标签），报告里同时记录两个版本号——半年后回看"v2.3 为什么容量下降"，能确切知道当时的脚本逻辑、数据文件与阈值配置。脚本变更中影响结论可比性的部分（负载参数、数据文件、断言口径）要在 PR 描述里显式声明，评审人据此判断历史基线是否作废（第 9 篇 1.2）。

## 5.2 TypeScript 支持方式

**k6 原生只执行 JavaScript，TypeScript 需要编译期转换，官方推荐两条路径。**其一是 `@types/k6` 提供类型声明，开发时用 TS 写、`tsc`/`esbuild` 编译成 JS 再运行；其二是官方的 k6 项目模板（`jslib` 生态的 typescript 模板）内置 esbuild 打包流程，顺带解决了 npm CJS 依赖的转换问题：

```bash
npm install --save-dev typescript @types/k6 esbuild

# 编译：TS → ESM JS，k6 直接可跑
npx esbuild src/test.ts --bundle --format=esm --outfile=dist/test.js
k6 run dist/test.js -e ENV=staging
```

```typescript
// src/test.ts —— TS 脚本示例：options 与 API 全程有类型提示
import http from "k6/http";
import { sleep, check } from "k6";
import { Options } from "k6/options";     // @types/k6 提供的类型

export const options: Options = {
  vus: 10,
  duration: "30s",
  thresholds: { http_req_duration: ["p(95)<500"] },
};

export default function (): void {
  const res = http.get("https://test.k6.io/");
  check(res, { "status 200": (r) => r.status === 200 });
  sleep(1);
}
```

选型建议：单人临时脚本直接写 JS，省去构建链；团队维护的脚本库上 TS——执行器参数拼错、`res.json()` 返回类型误用这类错误在编译期就被拦住，这对"半年才跑一次"的大促压测脚本尤其值钱。构建管线的完整展开见第 11 篇。

## 5.3 脚本发布流程与 CI 复用

**CI 复用的关键是"一次构建、多处消费"：脚本仓库产出可执行工件，各业务流水线按环境注入变量运行。**标准发布流程：PR 阶段跑 lint（eslint + prettier）与冒烟场景（1 VU × 30s，验证脚本可执行、check 全过）；合并后打标签并归档工件（脚本 + 数据文件 + 编译产物）；被测应用的性能流水线拉取工件，按测试类型触发——每次发布跑冒烟与短负载，每晚跑完整负载测试，每周跑浸泡。运行结果经 handleSummary 的 verdict 文件回写 PR 评论或推送 IM，summary.json 入库存趋势：

```yaml
# .github/workflows/perf-smoke.yml —— 消费脚本工件的冒烟流水线
name: perf-smoke
on:
  workflow_dispatch:            # 手动触发，也常被部署流水线调用
jobs:
  smoke:
    runs-on: self-hosted        # 压测机要固定规格，保证结果可比（第 9 篇基线）
    steps:
      - uses: actions/checkout@v4
      - uses: grafana/setup-k6-action@v1     # 官方 action 安装 k6
      - name: Run smoke scenario
        env:
          AUTH_TOKEN: ${{ secrets.PERF_AUTH_TOKEN }}   # 机密走 Secrets 注入
        run: k6 run scenarios/smoke_test.js -e ENV=staging -e BASE_URL=${{ vars.PERF_TARGET }}
      - name: Publish verdict
        if: always()
        run: cat reports/verdict.txt >> "$GITHUB_STEP_SUMMARY"
```

**设计考量**：CI 里跑压测有两条铁律——**跑在固定规格的 self-hosted runner 上**（共享 runner 的资源抖动会让结果不可比，基线机制失效）；**默认只打 staging**，对 prod 的压测走独立的手动审批工作流，且流水线层面限制触发权限。

# 6. 常见问题

## 6.1 模块依赖过深

**依赖链超过三层（scenario → flow → lib）就该警惕：调用栈变深后，一个请求的最终参数由四五处代码共同决定，排查"这个头是谁加的"要跨文件追溯。**治理手段：lib 层内部禁止互相 import 形成网状依赖（工具函数扁平化，公共常量收敛到单一模块）；flow 层只依赖 lib 不依赖其他 flow（流程间复用抽到 lib）；出现第四层需求时，通常说明抽象错了层——把逻辑上移或合并，而不是加一层目录。ESM 的静态依赖图可以用 `k6 inspect` 或打包器的 analyze 功能可视化，定期看一眼有没有意外的环。

## 6.2 配置混乱

**配置混乱的标志是同一参数有多个事实来源：URL 一半在 `__ENV` 一半硬编码，超时写在三个地方各不相同。**收敛原则是"单一配置出口"（4.1 的 config/env.js 模式）：所有环境相关值必须经它解析，代码审查时对裸 `__ENV.XXX` 和字面量 URL 一票否决。另一个高频坑是**布尔/数字型环境变量的类型**：`__ENV` 里一切皆字符串，`__ENV.DEBUG === false` 永远为假（那是字符串 "false"），解析层要做显式转换（`__ENV.DEBUG === "true"`、`parseInt`），这类转换也只允许出现在 config 层。

# 7. 小结

| 主题 | 要点 |
|------|------|
| 模块系统 | ESM only：无 require、无动态 import、无 Node 内置模块；npm CJS 包需打包器转换；模块顶层代码按 VU 重复执行 |
| 分层结构 | scenarios（负载配置）→ flows（业务动作）→ lib（协议与工具），依赖严格单向 |
| 自定义汇总 | `handleSummary(data)` 返回 { stdout, 文件路径: 内容 }，产 verdict 给 CI、产 JSON 给趋势库；不能发请求 |
| 配置管理 | config 层统一解析 `__ENV` 并给防呆默认值（默认 staging）；凭证只走 CI Secrets，全程无明文 |
| TypeScript | `@types/k6` + esbuild 编译为 ESM；团队脚本库推荐 TS，构建细节见第 11 篇 |
| CI 复用 | 固定规格 self-hosted runner + 工件化脚本 + 分级触发（发布跑冒烟/每晚负载/每周浸泡），verdict 回写 PR |

至此 k6 的使用与设计方法论（第 1-10 篇）完整收尾：从运行时、协议、指标，到执行模型、负载建模、数据工程、测试设计、脚本工程化。后续篇章进入周边设施：TypeScript 构建管线（第 11 篇）、xk6 扩展（第 12 篇）、输出与报告（第 13 篇）、可观测性集成（第 14 篇）与分布式压测（第 15 篇）。
