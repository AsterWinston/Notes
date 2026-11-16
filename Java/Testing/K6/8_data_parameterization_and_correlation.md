一个所有 VU 都用同一个账号、同一份商品数据打出去的压测，测到的很可能是缓存命中率和单行锁竞争，而不是系统容量。数据参数化让每个虚拟用户携带各自的身份与输入，关联（correlation）则让登录 token、订单 id 这类运行时才产生的值在业务流程中正确传递——这两件事决定了压测流量的"真实性"下限。本章讲数据的加载、共享、分片、随机化与关联提取。

# 1. 数据驱动与参数化

## 1.1 为什么要数据驱动

**参数化的第一动机是打破缓存，第二动机是模拟真实分布。**固定输入的压测请求会命中服务端每一层缓存——CPU 缓存、Redis、数据库 buffer pool、CDN，测出的延迟远好于真实用户看到的；固定账号还会把写操作集中到同一行数据上，锁竞争被人为放大。除了正确性问题，参数化也是业务仿真的需要：真实用户查询不同的商品、翻不同的页、带不同的会员等级，请求的成本分布（有的命中索引、有的走全表）只有靠多样化输入才能复现。

**注意**：参数化不是越多越好——把主键类输入完全随机化，可能测出"缓存全不命中"的另一个极端。正确做法是按生产画像控制多样性比例（比如 80% 热门数据 + 20% 长尾数据），这同样需要数据文件支持。

## 1.2 常见数据来源

| 来源 | 加载方式 | 适用场景 | 限制 |
|------|---------|---------|------|
| JSON 文件 | `open()` + `JSON.parse`，包进 SharedArray | 结构化用户池、商品池（首选） | 只能 init 上下文读取 |
| CSV 文件 | papaparse（jslib）解析 | 从数据库/Excel 导出的扁平数据 | 需引入第三方解析库 |
| 环境变量 | `__ENV.XXX` | 少量标量配置：地址、账号、开关 | 不适合大批量数据 |
| setup() 返回值 | HTTP/API 动态获取 | 运行时才存在的数据：注册账号、预置订单 | 返回数据会被 JSON 序列化，别塞大对象 |
| 远程接口 | VU 内实时调用 | 数据无法预生成时 | 把依赖系统的延迟与故障引入被测链路 |

> **行业实践**：数据文件随脚本入版本库（Git LFS 管大文件），并在 README 里记录生成方式与数据版本；临时压测也建议用脚本从生产库脱敏导出生成，而不是手工捏几十个账号——手工数据的分布失真往往在分析结果时才暴露。

# 2. 数据加载与分配策略

## 2.1 init 阶段加载

**所有文件读取都必须发生在 init 上下文，`open()` 是唯一入口，且只在 init 阶段可用。**init 上下文包括模块顶层代码和它调用的函数——第 3 篇讲过，这段代码在每个 VU 的 Goja 运行时里各执行一遍。这带来一个直接的内存陷阱：模块顶层写 `const users = JSON.parse(open("./users.json"))`，每个 VU 都会解析并持有一份完整数组，1000 VU × 50MB JSON = 内存爆炸。

**SharedArray 是官方解法：数据只在 Go 侧解析、存储一份，所有 VU 的 JS 运行时共享访问。**

```javascript
import { SharedArray } from "k6/data";

// 第二个参数是数据工厂函数：只在第一个用到它的 VU 里执行一次，
// 返回值存入 Go 侧共享内存，之后所有 VU 按名取用
const users = new SharedArray("users", function () {
  return JSON.parse(open("./users.json")); // open() 仍须在 init 上下文调用
});

export const options = { vus: 100, duration: "1m" };

export default function () {
  const user = users[__VU % users.length]; // VU 执行阶段按索引读取
  // user 是 JS 对象的深拷贝（读时复制），可安全修改，不影响共享数据
}
```

SharedArray 的原理与代价需要说清楚：

- **共享的是底层存储，读取返回拷贝**：每次 `users[i]` 都会把该元素从 Go 侧数据深拷贝进当前 VU 的运行时。单次读取比原生数组慢一个量级（涉及跨运行时序列化），高频循环里应先取到局部变量再用。
- **元素必须是可 JSON 序列化的值**：函数、循环引用存不进去。
- **数据工厂只执行一次，但要求确定性**：工厂里放随机数会导致"哪个 VU 先触发"影响数据内容，破坏可复现性。
- **小数据不必用 SharedArray**：几百条记录直接模块顶层解析即可，共享机制的读取开销反而不划算；它的适用线大约在"数据体积 × VU 数会威胁压测机内存"时。

CSV 数据没有内置加载器，官方推荐引入 papaparse：

```javascript
import { SharedArray } from "k6/data";
import papaparse from "https://jslib.k6.io/papaparse/5.1.1/index.js";

// papaparse 在 init 上下文解析 CSV 字符串，header: true 把首行作为字段名
const rows = new SharedArray("rows", function () {
  return papaparse.parse(open("./accounts.csv"), { header: true }).data;
});
```

## 2.2 数据分片与分配

**分片策略决定"哪个 VU 用哪份数据"，核心约束是业务唯一性：凡是不能复用的数据（账号、卡号、库存单元），必须保证同一时刻只有一个 VU 持有。**三种常用策略：

```javascript
// 策略一：取模轮转——数据可复用时最简单
const user = users[__VU % users.length];

// 策略二：按 VU 编号切片——每个 VU 独占一段数据，互不重叠
const sliceSize = Math.ceil(users.length / __ENV.VUS);
const myUsers = users.slice((__VU - 1) * sliceSize, __VU * sliceSize);

// 策略三：场景感知的稳定索引——多场景下用 exec.scenario 提供的编号
import exec from "k6/execution";

export default function () {
  // idInTest：本场景内的 VU 序号（从 1 开始，稳定不变）
  // iterationInTest：整个测试内的全局迭代序号（跨场景递增，适合全局唯一取样）
  const user = users[(exec.scenario.idInTest - 1) % users.length];
  const record = users[exec.scenario.iterationInTest % users.length];
}
```

`k6/execution` 模块（v0.34+）提供的场景感知计数器，是替代老式 `__VU`/`__ITER` 的推荐写法——`__VU` 在到达率执行器下会随 VU 扩容变化，`__ITER` 在共享迭代池下跨 VU 不唯一，而 `idInTest`/`iterationInTest` 的语义在六种执行器下一致：

| 属性 | 语义 | 典型用途 |
|------|------|---------|
| exec.scenario.iterationInInstance | 当前 VU 内的迭代序号 | 单 VU 视角的流程计数 |
| exec.scenario.iterationInScenario | 当前场景内的迭代序号 | 场景内均匀取样 |
| exec.scenario.iterationInTest | 全测试范围的迭代序号 | 全局唯一数据消费（不重复取样） |
| exec.scenario.idInInstance | 当前 VU 在本运行时实例内的编号 | 分布式压测下的实例内分片 |
| exec.scenario.idInTest | 当前 VU 在全测试内的编号 | VU 级数据独占分片 |

> **注意**：数据量必须 ≥ 并发 VU 数（独占场景）或 ≥ 总迭代数（不重复消费场景），否则只能退回复用数据。压测前的容量核对应该像 checklist 一样固定执行：`users.length` 与 `vus`、`iterations`、`rate × duration` 三个数字对一遍。

# 3. 关联与动态提取

## 3.1 关联的意义

**关联（correlation）指把上一步响应中的动态值提取出来、注入下一步请求**——登录后拿到的 token、下单返回的 orderId、页面里的一次性 CSRF 令牌都属于此类。它是链路型压测正确性的第一杀手：token 没关联，后续请求全部 401，测出来的是错误处理路径的性能；orderId 关联错了，压的是同一笔订单的并发更新。判断脚本是否需要关联的方法很朴素：**用浏览器开无痕窗口手工走一遍业务流程，凡是"刷新一次就变"的值，都必须关联提取。**

## 3.2 JSON 提取

**`res.json(selector)` 是 JSON 关联的主路径，selector 支持点路径直达嵌套字段**，免去手工层层判空：

```javascript
import http from "k6/http";
import { check, fail } from "k6";

export default function () {
  // 步骤一：登录，提取动态 token
  const loginRes = http.post("https://api.example.com/login", JSON.stringify({
    username: "loadtest_user",
    password: __ENV.TEST_PASSWORD,          // 凭证走环境变量，不入代码库
  }), { headers: { "Content-Type": "application/json" } });

  const ok = check(loginRes, {
    "login status 200": (r) => r.status === 200,
    "login has token": (r) => r.json("data.token") !== undefined,
  });
  if (!ok) {
    fail("login failed, abort iteration");   // 关联值拿不到时立即失败，
  }                                          // 避免带着 undefined 继续打流量

  const token = loginRes.json("data.token"); // 点路径提取嵌套字段

  // 步骤二：带 token 访问业务接口，再提取业务 id
  const orderRes = http.post("https://api.example.com/orders",
    JSON.stringify({ sku: "A-100" }),
    { headers: { Authorization: `Bearer ${token}` } }
  );
  const orderId = orderRes.json("data.orderId");

  // 步骤三：关联值注入后续请求
  http.get(`https://api.example.com/orders/${orderId}`, {
    headers: { Authorization: `Bearer ${token}` },
  });
}
```

两条工程纪律：**关联失败要快速失败**（`fail()` 终止迭代，而不是带着 `undefined` 继续），否则指标里混入大量无意义的 4xx 样本，错误率失真且难归因；**token 这类多迭代复用的关联值优先放 `setup()`**——登录一次、全 VU 共享，比每个迭代都登录省掉一整类流量（前提是被测系统允许 token 复用，且要注意 token 过期问题，见 5.2）。

## 3.3 HTML/文本提取

**HTML 响应的关联有三条路：正则、DOM 解析库、服务端开 JSON 接口，优先级从低到高。**正则适合结构稳定的简单取值（隐藏表单字段、meta 标签）；复杂页面用 jslib 上的 cheerio 做类 jQuery 解析：

```javascript
import http from "k6/http";
import * as cheerio from "https://jslib.k6.io/cheerio/1.0.0/index.js";

export default function () {
  const res = http.get("https://example.com/form-page");
  const $ = cheerio.load(res.body);            // 把 HTML 解析成 DOM
  const csrf = $('input[name="csrf_token"]').attr("value"); // 选择器提取
  // …csrf 注入后续 POST…
}
```

**设计考量**：cheerio 解析整个 DOM 的 CPU 成本不低，高 RPS 下负载发生器会先扛不住；正则虽快但页面结构一变就静默取错值。压测自有系统时，最干净的方案是让服务端为压测流量提供 JSON API 或稳定的取值端点——把解析成本从每次迭代移到开发期一次性的接口约定上。

# 4. 数据一致性与可复现

## 4.1 固定随机策略

**随机化数据的价值在分布，可复现的价值在归因，两者用"固定种子 + 版本化数据文件"调和。**k6 没有全局随机种子开关，`Math.random()` 每次运行都不同；需要可复现时，把随机性收敛到确定性的伪随机函数上（以 `__VU`/迭代号做种子的哈希取模），或者干脆离线生成好数据文件——文件即种子，配合 Git 版本号，任何一次历史压测都能用同一份数据重放。

数据文件的版本管理要点：**记录生成时间、来源查询、脱敏规则**；数据文件变更后旧报告的结论即失效，报告里应引用数据文件的版本号（commit hash）而不是笼统的"用了 users.json"。

## 4.2 数据污染与回收

**写入型压测必然产生脏数据，方案要在压测前设计，而不是压测后手工删。**常用手段按侵入性排序：压测专用账号段（用户名统一 `loadtest_` 前缀，teardown 或离线任务按前缀清理）；影子库/影子表（中间件层按流量标记路由，业务无感知，成本高但最干净）；独立的压测环境（数据随便写，但要接受环境与生产的差异，见第 9 篇）。k6 侧的配套动作是在 `teardown()` 里调用清理接口，并把"清理是否成功"也作为 check 上报。

> **行业实践**：绝不在生产库直接做写入型压测；只读型压测打生产也要求流量打标（特殊 Header + 网关识别），确保监控与风控能把压测流量从业务数据里剔除。

# 5. 常见问题与最佳实践

## 5.1 数据量不足

**数据池小于并发需求时，所有"独占数据"的假设集体失效**：账号被多个 VU 同时登录触发互踢、库存单元被重复扣减、唯一约束冲突批量报错。症状是错误率随 VU 数上升而非随负载上升，且错误类型集中在业务校验层。核对公式：独占型数据量 ≥ maxVUs；不重复消费型数据量 ≥ rate × duration；可复用型无硬性要求，但要评估缓存命中率对结论的影响。数据不够时的正确动作是停下来补数据，而不是把校验关掉。

## 5.2 动态数据失效

**token/会话类关联值都有寿命，长时间测试（浸泡尤其）必须处理过期。**三种策略：setup() 获取 + teardown 前刷新（简单，但 setup 数据是启动时快照，中途过期无解）；迭代内登录（永远新鲜，代价是登录接口混入业务流量，需按 scenario 标签分开统计）；VU 内缓存 + 按 TTL 主动刷新（平衡做法）——

```javascript
let tokenCache = { token: null, expiresAt: 0 };

function getToken() {
  const now = Date.now();
  // 提前 60 秒刷新，避免边界过期
  if (!tokenCache.token || now > tokenCache.expiresAt - 60_000) {
    const res = http.post(`${__ENV.BASE_URL}/login`, JSON.stringify({
      username: `loadtest_${__VU}`, password: __ENV.TEST_PASSWORD,
    }), { headers: { "Content-Type": "application/json" } });
    tokenCache = {
      token: res.json("data.token"),
      expiresAt: now + res.json("data.expiresIn") * 1000,
    };
  }
  return tokenCache.token;
}
```

（每个 VU 独立运行时，模块级变量天然是 VU 私有的，这个缓存模式不需要加锁。）

## 5.3 关联链路过长

**关联是串联系统：单步成功率 99% 的五步链路，整体成功率只剩 95%，链路越长失败被逐级放大。**长链路的治理手段：把前置步骤（注册、实名、开通权限）移出压测脚本、离线预生成，压测只跑真正关心的核心段；链路上每一步都加 check 并在关联失败时 `fail()` 短路，防止失败样本污染下游指标；对稳定的中间值（商品目录 id 这类变化缓慢的数据）定期快照成数据文件，替代每次迭代实时拉取。判断标准：**压测脚本里只保留"被测行为"必需的关联，其余一律前置。**

# 6. 小结

| 主题 | 要点 |
|------|------|
| 数据加载 | 文件读取仅限 init 上下文的 `open()`；大数据必须 SharedArray（Go 侧存一份，读取返回深拷贝） |
| 分片策略 | 取模轮转（可复用）/ VU 切片（独占）/ `exec.scenario.idInTest`、`iterationInTest`（场景感知，推荐替代 `__VU`/`__ITER`） |
| 容量核对 | 独占数据量 ≥ maxVUs；不重复消费 ≥ rate × duration |
| 关联 | JSON 用 `res.json("a.b")` 点路径；HTML 用 cheerio 或推动服务端出 JSON；关联失败必须 `fail()` 短路 |
| token 生命周期 | setup 一次登录 / 迭代内登录 / VU 内 TTL 缓存刷新，按测试时长选 |
| 可复现 | 数据文件入版本库并记录生成方式；随机性收敛为确定性取样 |

数据让流量"长得真"，但测试设计的另一半问题是"测得对"：目标怎么定、结果怎么读、哪些统计陷阱会让结论作废——下一篇进入测试设计方法论。
