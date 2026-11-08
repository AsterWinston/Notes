# 1. 脚本结构与模块

## 1.1 最小脚本结构

k6 脚本本质是一个 **ES Module**：用 `import` 引入内置或第三方模块，用 `export default` 导出默认函数作为迭代入口，用 `export const options` 导出运行配置。最小可运行脚本只需要一个 default 函数：

```javascript
import http from "k6/http";

export default function () {
  http.get("https://test.k6.io");
}
```

运行 `k6 run script.js` 时，若不指定任何负载参数，k6 使用缺省执行器 shared-iterations，以 **1 个 VU 跑 1 次迭代**——这就是"最小脚本 + 默认参数"的完整语义，适合冒烟验证脚本逻辑。

模块系统值得多说一句：k6 的 JS 环境由 Goja 提供，**支持 CommonJS 与 ESM 混用、支持 npm 包**（通过 `npm install` 后用 webpack/rollup/esbuild 打包，或直接被 k6 的实验性 ESM 解析加载）、支持相对路径的多文件拆分。大型压测套件通常拆成 `config.js`（环境与常量）、`utils/`（登录、签名等可复用函数）、`scenarios/`（各业务流脚本）的目录结构，用 Git 管理——这正是"压测即代码"的组织形态。

## 1.2 options 与脚本配置

`options` 是脚本的自描述配置，涵盖负载模型、阈值、TLS、输出等全部运行选项：

```javascript
export const options = {
  // 负载模型（简单形式；复杂形式见 scenarios）
  vus: 10,
  duration: "30s",

  // 质量闸门：任一阈值突破则退出码 99
  thresholds: {
    http_req_failed: ["rate<0.01"],          // 错误率 < 1%
    http_req_duration: ["p(95)<500"],        // P95 延迟 < 500ms
  },

  // 其他常用项
  insecureSkipTLSVerify: false,              // 仅限测试环境可置 true
  discardResponseBodies: true,               // 不保留响应体，省内存
  summaryTrendStats: ["avg", "min", "med", "max", "p(90)", "p(95)", "p(99)"],
};
```

options 里定义的默认值可以被环境变量（`K6_VUS`）与命令行参数（`--vus`）逐层覆盖，优先级：**命令行 > 环境变量 > options**。把负载模型和阈值写进 options 而不是命令行，是保证脚本可复现的前提——`git checkout` 任意历史版本都能还原当时的完整测试定义。

## 1.3 常用内置变量

k6 向脚本暴露三个全局内置变量，它们不是 import 来的，直接可用：

**`__VU`** 是当前虚拟用户的编号，从 1 开始，常用于数据分片——`users[__VU % users.length]` 让每个 VU 固定使用用户池中的一个账号，避免所有 VU 挤同一个账号触发服务端风控或行锁竞争。

**`__ITER`** 是当前 VU 已完成的迭代序号，从 1 开始，常用于构造不重复的业务数据（订单号、幂等键）：`const orderNo = \`ORD-${__VU}-${__ITER}\``。

**`__ENV`** 是环境变量对象，所有 `-e KEY=VALUE` 传入的变量与系统环境变量都能从中读取。

> **注意**：`__VU` 与 `__ITER` 在 init 阶段均为 0——**任何依赖它们的逻辑必须写在 default 函数内部**，写在模块顶层（init context）会得到 0 并静默产生错误数据，这是新手最常踩的坑之一。

# 2. 生命周期模型

## 2.1 两个世界：init context 与 VU 阶段

k6 脚本的执行分为截然不同的两个阶段，理解这条分界线是掌握 k6 脚本模型的钥匙：

```text
┌──────────────────────────────────────────────────────────┐
│ Init Context（每 VU 启动时执行一次，之后不再执行）           │
│  · 所有 import 语句                                       │
│  · 模块顶层代码：常量、new SharedArray、new Trend(...)      │
│  · options 的求值                                        │
│  ✗ 不能调用 http.* / sleep / check 等运行期 API            │
├──────────────────────────────────────────────────────────┤
│                    setup()  （全局一次）                    │
│  · 准备数据、取 token；返回值 JSON 序列化后传给 VU           │
├──────────────────────────────────────────────────────────┤
│ VU 阶段（每个 VU 循环执行，压测主体）                        │
│  · default(data) 反复执行 = 迭代                           │
│  · http/check/sleep/group/fail 等全部可用                  │
├──────────────────────────────────────────────────────────┤
│                  teardown(data) （全局一次）               │
│  · 清理数据、输出汇总；接收 setup 的返回值                    │
└──────────────────────────────────────────────────────────┘
```

**init context 里只允许"纯声明"性质的代码**。所有 `import`、顶层 `const`、`new SharedArray`、`new Trend` 在这里执行；每个 VU 创建时都会完整跑一遍 init context（所以模块顶层代码会被执行 N 次）。此阶段调用 `http.get` 或 `sleep` 会直接报错 `Some HTTP functions were not reachable in the init context`——设计上禁止，是因为 init 代码若做网络 I/O，几千个 VU 的启动风暴会瞬间打垮被测系统，且这部分流量无法计入正常指标。

**顶层代码被每 VU 重复执行**这一点直接引出一条性能纪律：模块顶层不要做重计算（大 JSON 解析、复杂循环），要么用 `SharedArray`（跨 VU 共享一份底层数据），要么挪进 `setup()`（全局只跑一次）。

## 2.2 setup 阶段

`setup()` 在 init 之后、VU 启动之前**全局执行一次**，运行在独立的 VU 上下文里，可以使用 http、check 等全部运行期 API。它的典型用途是准备测试前置状态：造测试数据、登录换取全局 token、初始化配额。setup 的返回值必须是**可 JSON 序列化**的（对象、数组、字符串、数字；函数、undefined、循环引用不行），k6 会把它序列化后分发给每个 VU 作为 default 函数的入参：

```javascript
import http from "k6/http";
import { fail } from "k6";

const BASE_URL = __ENV.BASE_URL || "https://api.example.com";

export function setup() {
  const res = http.post(`${BASE_URL}/login`, { user: "loadtest", pass: __ENV.PASSWORD });
  const token = res.json("token");
  if (!token) {
    fail("登录失败，无法取得 token，终止测试");   // 快速失败，别带着空 token 跑完全场
  }
  return { token, startedAt: Date.now() };       // 必须可 JSON 序列化
}
```

setup 阶段自身产生的指标默认会被打上 `group:::setup` 标签计入统计，若不想让准备流量污染压测数据，可以用 `--no-setup` 跳过或在分析时按 group 过滤。setup 也有独立超时（默认 `setupTimeout: "30s"`），数据准备耗时长的场景要显式调大。

## 2.3 VU 执行阶段

default 函数是每个迭代的主体，接收 setup 返回值作为第一个参数。迭代内可以使用全部运行期 API，两个实践要点：**思考时间用 `sleep`** 表达（单位秒，支持小数；arrival-rate 执行器下 sleep 仍会拉长单次迭代耗时，需要一并计入 preAllocatedVUs 估算）；**迭代必须能独立成败**——一次迭代内抛出未捕获异常，只中断该 VU 的当前迭代，VU 记录一次失败后继续下一轮，不会导致整个测试崩溃。

## 2.4 teardown 阶段

`teardown(data)` 在全部 VU 结束后**全局执行一次**，入参同样是 setup 的返回值，适合做数据清理、注销会话、发送测试完成通知。teardown 抛异常不会改变测试的阈值判定结果，但会记录在输出中；`--no-teardown` 可跳过。teardown 与 setup 有同样的 JSON 序列化约束与独立超时（`teardownTimeout`）。

完整生命周期示例：

```javascript
import http from "k6/http";
import { sleep, fail } from "k6";

const BASE_URL = __ENV.BASE_URL || "https://test.k6.io";

export const options = {
  vus: 5,
  duration: "10s",
  setupTimeout: "60s",                            // 准备数据可能较慢，放宽超时
};

export function setup() {
  const res = http.post(`${BASE_URL}/login`, { user: "u", pass: "p" });
  if (res.status !== 200) fail("login failed");    // 前置失败立即终止
  return { token: res.json("token") };
}

export default function (data) {
  const headers = { Authorization: `Bearer ${data.token}` };
  const res = http.get(`${BASE_URL}/api/items`, { headers });
  if (res.status !== 200) fail(`unexpected status ${res.status}`);
  sleep(1);                                        // 模拟用户思考时间
}

export function teardown(data) {
  http.post(`${BASE_URL}/logout`, null, {
    headers: { Authorization: `Bearer ${data.token}` },
  });
}
```

> **行业实践**：多个 scenario 可以各自指定 `exec` 入口函数，但 **setup/teardown 永远只有一对且全局共享**。若不同业务流需要不同的前置数据，惯例是在 setup 里返回一个大对象（`{ orderToken, userToken, seedData }`），各 default 函数按需取用；或者干脆把前置逻辑放进各场景的 default 函数首次迭代里用 `__ITER === 1` 判断执行。

# 3. 数据共享与状态

## 3.1 每 VU 隔离

**每个 VU 拥有独立的 Goja 运行时实例，JS 层面的变量空间完全隔离**。模块顶层声明的 `let counter = 0` 在每个 VU 里是各自的一份副本，VU1 加到 100 时 VU2 可能才到 7，且互相不可见——这不是竞态，是根本不存在共享。这条隔离规则简化了并发心智（无需锁），但也意味着**跨 VU 传递数据只有三条正路**：setup 返回值（一次性分发）、`SharedArray`（只读共享）、外部存储（Redis/数据库，经扩展读写）。试图用全局变量在 VU 间做计数器或队列，得到的永远是各自为政的假数据。

## 3.2 SharedArray 使用

`SharedArray` 是官方提供的**跨 VU 只读共享数据结构**：数据只在 init 阶段加载一次，所有 VU 共享同一份底层内存（读取时按需复制到各自运行时），专门解决"每个 VU 都 parse 一遍大 JSON 文件"的内存与 CPU 浪费：

```javascript
import http from "k6/http";
import { SharedArray } from "k6/data";

const BASE_URL = __ENV.BASE_URL || "https://api.example.com";

// 第一个参数是全局唯一名字（用于内存复用标识），第二个是加载函数
const users = new SharedArray("users", () => JSON.parse(open("./users.json")));

export default function () {
  // __VU 从 1 开始，取模映射到用户池；每个 VU 固定使用同一账号
  const user = users[__VU % users.length];
  http.post(`${BASE_URL}/login`, { user: user.name, pass: user.pass });
}
```

`open()` 是 k6 提供的文件读取函数，**只能在 init context 使用**，路径相对于脚本文件；文件会被打进 `k6 archive` 归档，保证脚本自包含。SharedArray 支持按索引与 `length` 访问，不支持 push/splice 等修改操作——它是只读的，这是共享内存安全的前提。

## 3.3 随机与可复现

压测数据里的随机性需要可控：随机选品、随机金额能让负载更真实，但**问题复现时需要知道当时的随机序列**。k6 没有全局随机种子机制，实践中的可复现方案有三种：用 `__VU`/`__ITER` 派生确定性伪随机（同参数重跑得到同序列）；记录关键随机值的采样日志；对必须精确复现的场景直接固定数据文件。随机数生成本身用标准 `Math.random()` 即可，注意它在每个 VU 的运行时里独立播种。

# 4. 错误处理与超时

## 4.1 fail 与异常

k6 区分两种"失败"：**HTTP 层失败**（连接错误、超时、4xx/5xx，后者需配 `setResponseCallback` 才计入 `http_req_failed`）与**脚本层失败**（未捕获异常、显式 `fail()`）。`fail(reason)` 抛出特殊异常终止当前迭代并计入 `iteration_errors`；而 `check` 失败**什么都不终止**，只更新 `checks` 指标的通过率。选择原则：**预期内的业务分支用 check 标记**（例如"库存不足返回 409"是合法结果），**违反前置条件、继续跑没有意义的情况用 fail**（例如登录拿不到 token）：

```javascript
import http from "k6/http";
import { check, fail } from "k6";

const res = http.post(`${BASE_URL}/orders`, payload);

const ok = check(res, {
  "创建订单成功": (r) => r.status === 201,
  "返回订单号": (r) => r.json("orderNo") !== undefined,
});

if (!ok && res.status === 500) {
  fail("服务端 500，订单创建流程无法继续");   // 后续步骤依赖订单号，fail 快速止损
}
```

## 4.2 请求超时

超时控制有三层粒度：**全局** `options.timeout`（默认 "1m"，作用于所有网络操作）；**单请求** `params.timeout`（如 `http.get(url, { timeout: "5s" })`）；**setup/teardown** 的独立超时。超时值的设定要结合业务 SLO 与链路环境——把超时设得比 SLO 短会让慢请求变成错误请求，掩盖真实的延迟分布；设得过长则慢请求占住 VU，闭合模型下直接拉低吞吐。经验做法：超时设为 P99 SLO 的 2-3 倍，让"超出容忍但真实发生"的延迟留在 Trend 指标里而不是被截断成 error。

## 4.3 失败策略

成熟的脚本要明确回答三个问题：单次请求失败后**是否重试**（k6 不内置重试，需显式循环实现，但重试会掩盖错误率，回归测试中应关闭）；单迭代失败后**是否继续**（默认继续，VU 进入下一迭代）；整体错误率超限后**是否中止**（用 thresholds 的 `abortOnFail` 让 k6 主动停止测试而不是硬跑完）：

```javascript
export const options = {
  thresholds: {
    // 错误率持续 30s 超过 10% 则立即中止，避免浪费时间和服务端资源
    http_req_failed: [{ threshold: "rate<0.10", abortOnFail: true, delayAbortEval: "30s" }],
  },
};
```

# 5. 常见坑与最佳实践

## 5.1 在 init 中做 I/O

init context 禁止网络调用是硬性规则，但**变通误区**更常见：有人在顶层用 `SharedArray` 的加载函数里发 HTTP 请求——同样会失败，因为 SharedArray 的回调也在 init 执行。正确的分层是：**静态数据（文件、常量表）放 init + SharedArray；动态数据（token、种子记录）放 setup；每迭代数据放 default**。另一个隐性坑是顶层执行重计算：模块顶层的循环、正则预编译会被每个 VU 重复执行，5000 VU 就是 5000 遍——预计算结果应放进 SharedArray 或 setup 返回值。

## 5.2 大对象与内存压力

响应体默认完整保存在内存中，`res.body` 是字符串——**高 QPS 下大响应体会成为压测机内存的头号杀手**。全局 `discardResponseBodies: true` 丢弃所有响应体，再对少数需要解析的请求单独 `responseType: "text"` 捞回来；JSON 解析用 `res.json("path.to.field")` 按路径取值，避免 `JSON.parse(res.body)` 全量反序列化后再丢弃。此外每个 VU 的运行时里累积的数组、Map（例如把每次迭代的响应 push 进顶层数组）会随迭代数线性增长直至 OOM——**迭代内产生的数据要么当场断言完丢弃，要么聚合进 Counter/Gauge 指标，不要囤积原始对象**。

## 5.3 过多日志

`console.log` 在压测中是昂贵的：字符串拼接发生在 JS 层，输出跨越 Go 层写入 stderr，且默认同步刷盘。1000 VU × 每迭代 3 条日志 × 每秒 10 迭代 = 每秒 3 万行日志，足以让客户端自身成为瓶颈并扭曲全部延迟指标。纪律是：**日志只属于调试阶段**（`k6 run --vus 1 --iterations 5 -v`），进入正式压测前移除所有 console 调用；确需长期观测的信息改为自定义指标（Counter/Trend），走指标引擎聚合而非日志流。

# 6. 小结

k6 脚本模型的骨架是"init context 声明 + setup 准备 + default 迭代 + teardown 清理"的四段生命周期：**init 每 VU 一次且禁止 I/O，setup/teardown 全局各一次且返回值经 JSON 序列化传递，default 是唯一的循环主体**。VU 间的 JS 隔离决定了跨 VU 数据共享只有 SharedArray（只读静态）、setup 返回值（一次性分发）与外部存储三条路。工程实践的重心在三件事：负载模型与阈值写进 options 保证可复现、用 check 与 fail 区分业务分支和致命错误、管住内存与日志让压测机自己不成为瓶颈。
