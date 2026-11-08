# 1. HTTP 模块基础

## 1.1 常见请求方法

`k6/http` 模块为每个 HTTP 动词提供了便捷方法：`http.get`、`http.post`、`http.put`、`http.patch`、`http.del`（注意是 del 不是 delete，因为 delete 是 JS 保留字）、`http.head`、`http.options`，以及万能入口 `http.request(method, url, body, params)`。所有方法**同步阻塞**当前 VU 直到响应返回——k6 的 JS 层没有 Promise/async-await，并发靠 VU 与 `http.batch` 实现，这是它与前端 JS 心智模型最大的差异：

```javascript
import http from "k6/http";

const BASE = "https://api.example.com";

// GET：无请求体
http.get(`${BASE}/items?page=1`);

// POST JSON：body 必须是字符串，headers 手动声明 Content-Type
http.post(`${BASE}/items`, JSON.stringify({ name: "widget", price: 9.9 }), {
  headers: { "Content-Type": "application/json" },
});

// POST 表单：body 传对象时默认按 application/x-www-form-urlencoded 编码
http.post(`${BASE}/login`, { user: "u", pass: "p" });

// 通用形式：method 任意，适合动态动词
http.request("PURGE", `${BASE}/cache/key`, null, null);
```

一个高频错误值得单独强调：**POST JSON 时 body 传对象不会自动序列化**——对象会被字符串化成 `[object Object]` 发出去。必须 `JSON.stringify` 并手动设置 `Content-Type: application/json`，这是 Java 开发者从 RestTemplate/Jackson 世界过来最容易踩的坑。

## 1.2 params 参数

每个请求方法的最后一个参数是 `params` 对象，控制本次请求的行为细节：

```javascript
const res = http.get(`${BASE}/items/1`, {
  headers: { "X-Trace-Id": `vu${__VU}-iter${__ITER}` },  // 请求头
  tags: { endpoint: "item_detail" },                      // 指标标签，用于分组统计与阈值过滤
  timeout: "5s",                                          // 本次请求超时，覆盖全局默认
  redirects: 3,                                           // 最大重定向次数
  cookies: { session: ["sid-abc"] },                      // 附加 cookie
  responseType: "text",                                   // 响应体保留方式（默认继承全局）
});
```

`tags` 是其中最有工程价值的字段：**每个 tag 都会附加到该请求产生的全部指标上**，使得"只统计某个接口""只对下单链路设阈值"成为可能（阈值侧写法 `http_req_duration{endpoint:order}<800`，见第 5 篇）。动态值（订单号、URL 路径参数）**不应**直接出现在 URL 统计维度里——k6 默认对每个不同 URL 生成一条时间序列，路径里带 ID 会导致指标基数爆炸，正确做法是 URL 归一化加 tags 标注。

## 1.3 批量请求与并发

**`http.batch` 在单个 VU 内并行发起多个请求**——请求在 Go 层并发执行，JS 层阻塞等待全部完成，返回值是按入参顺序排列的响应数组。它模拟的是"浏览器同时加载页面资源""客户端并发调用多个微服务"这类真实场景：

```javascript
import http from "k6/http";

export default function () {
  // 数组形式：每项是 [method, url, body?, params?]
  const responses = http.batch([
    ["GET", "https://api.example.com/a", null, { tags: { api: "a" } }],
    ["GET", "https://api.example.com/b"],
    ["POST", "https://api.example.com/c", JSON.stringify({ id: 1 }), {
      headers: { "Content-Type": "application/json" },
    }],
  ]);

  // 对象形式：键名即结果属性名，可读性更好
  const { user, cart } = http.batch({
    user: ["GET", "https://api.example.com/user"],
    cart: ["GET", "https://api.example.com/cart"],
  });
}
```

> **设计考量**：batch 的并发度默认无上限，一次 batch 100 个请求会瞬间打出 100 条并发连接。`maxParallelRequests`（脚本 options，默认 20）可以限流，模拟浏览器的每域名 6 连接限制或保护被测系统。注意 batch 内的请求**共享同一 VU 的 cookie jar**，并行写 cookie 有竞态可能；另外 batch 的整体耗时计入 `http_req_blocked` 之外，但会拉长迭代时长——开放模型下要计入 preAllocatedVUs 估算。

# 2. 会话与认证

## 2.1 Cookie 管理

k6 为**每个 VU 维护独立的 cookie jar**，行为对齐真实浏览器：响应的 `Set-Cookie` 自动存储，后续同域请求自动携带，跨迭代保持会话。这意味着"登录一次、后续请求自动带会话"在 k6 里零成本：

```javascript
import http from "k6/http";

export default function () {
  // 首次登录后，session cookie 自动进入本 VU 的 jar
  http.post("https://example.com/login", { user: "u", pass: "p" });
  // 后续请求自动携带 cookie，无需手动处理
  http.get("https://example.com/profile");

  // 显式读写 jar（跨域带 cookie、预置会话时需要）
  const jar = http.cookieJar();
  jar.set("https://other.example.com", "sid", "abc123");
  const sid = jar.cookiesForURL("https://example.com").sid;
}
```

两个边界情况：**跨域请求默认不带 cookie**（jar 按域隔离，与浏览器同源策略一致），需要跨子域共享会话时用 `jar.set` 显式写入目标域；**每个 VU 的 jar 独立**意味着 100 个 VU 登录会创建 100 个会话——这通常正是压测想要的（模拟 100 个真实用户），但若被测系统按会话数收费或限流，需提前评估。

## 2.2 认证方式

**HTTP Basic 认证**有两种写法：params 里的 `auth` 字段，或手动拼 Authorization 头。**Bearer Token** 则总是手动拼头。token 的生命周期决定获取位置：长期有效的服务账号 token 放 `__ENV`；短时效 token 放 `setup()` 全局取一次；会过期的用户级 token 在迭代内按需刷新：

```javascript
import http from "k6/http";

export default function () {
  // Basic 认证的便捷写法
  http.get("https://example.com/api", { auth: "basic:user:password" });

  // 等价的显式写法
  const basicAuth = "Basic " + __ENV.BASIC_CREDENTIALS;   // 预先 base64(user:pass)
  http.get("https://example.com/api", {
    headers: { Authorization: basicAuth },
  });

  // Bearer token：从 setup 返回值取（见第 3 篇生命周期）
  http.get("https://example.com/api", {
    headers: { Authorization: `Bearer ${__ENV.API_TOKEN}` },
  });
}
```

> **行业实践**：Java 生态压测常复用业务 SDK（用 Spring 的 RestClient/WebClient 发压测请求），k6 则要求**用 http 模块重新表达认证逻辑**——签名算法（如 AWS SigV4、自定义 HMAC）没有现成 SDK 可抄时，要么用 JS 在脚本里实现（k6 内置了 crypto 模块支持 HMAC/SHA），要么封装成 xk6 扩展。评估压测工具时，"我们的网关签名逻辑能否在脚本里低成本复现"应该是选型检查项之一。

## 2.3 重定向

重定向默认自动跟随，上限 10 次（options 的 `maxRedirects`），单请求可用 `params.redirects` 覆盖。`redirects: 0` 表示**不跟随**，用于专门验证 301/302 本身（例如检查跳转目标是否正确、重定向是否泄漏会话）。跟随产生的每一跳都是独立请求，各自计入 `http_reqs` 与 `http_req_duration`——统计接口 QPS 时要意识到 302 跳转会让请求数虚高，必要时用 tags 过滤主请求。

# 3. 连接与协议

## 3.1 Keep-Alive 与连接复用

k6 的 HTTP 客户端构建在 Go `net/http` 之上，**默认启用连接池与 Keep-Alive**：同一 VU 对同一目标的连续请求复用 TCP 连接，省去握手开销。这贴近真实浏览器与长连接客户端的行为，但压测语义上要注意：**连接复用会让第二次起的请求不产生 `http_req_connecting` 与 `http_req_tls_handshaking` 耗时**（指标为 0），如果目标是评估"每次新建连接"的短连接服务性能，需要用 `noConnectionReuse: true` 关闭复用，让每个请求走完整的 DNS + TCP + TLS 流程。

## 3.2 HTTP/2

k6 默认走 HTTP/1.1，对 HTTPS 目标可显式开启 HTTP/2（options 里 `httpVersion: "h2"` 或按 ALPN 协商 `"any"`）。HTTP/2 的多路复用把"每域名多连接"压缩为单连接多流，**同样 QPS 下连接数大幅下降**，这对连接数敏感的服务端（Nginx worker_connections、网关连接配额）意味着完全不同的压测结果。原则：**用与服务端真实客户端一致的协议版本压测**——App 端走 h2、老 SDK 走 1.1，就分别建模，混用会得到无法解释的基线。

## 3.3 TLS 与双向认证

TLS 相关配置集中在 options：`insecureSkipTLSVerify`（跳过证书校验，仅限测试环境）、`tlsVersion`（限定 `min`/`max`，如只允许 tlsv1.2 以上）、`tlsCipherSuites`（限定套件）、`tlsAuth`（**mTLS 客户端证书**，PEM 格式内嵌或文件路径）：

```javascript
export const options = {
  tlsVersion: { min: "tls1.2", max: "tls1.3" },
  tlsAuth: [
    {
      domains: ["internal.example.com"],
      cert: open("./client.crt"),    // open() 仅 init context 可用
      key: open("./client.key"),
    },
  ],
};
```

# 4. 响应处理

## 4.1 常用响应字段

每个请求返回 `Response` 对象，字段与 Java 世界的 `HttpResponse` 高度同构：

```javascript
const res = http.get("https://api.example.com/items/1");

res.status;              // 200，状态码
res.body;                // 响应体字符串（受 responseType 控制）
res.headers;             // 响应头对象，键名小写化，多值头合并为逗号串
res.timings.duration;    // 请求总耗时（ms），另有 blocked/connecting/tls_handshaking/sending/waiting/receiving
res.url;                 // 最终 URL（含重定向后）
res.error_code;          // 非 HTTP 层错误码（如 1000 连接超时），无错误为 0
res.request;             // 对应的请求元信息

// JSON 解析：整体或按路径（路径支持 "data.items.0.id" 形式）
const item = res.json();                 // 全量对象
const name = res.json("data.name");      // 按路径取单个字段，避免全量反序列化
```

`res.json(path)` 的路径取值是 k6 对高频操作的优化：**只解析到目标路径即返回**，大响应体下比 `JSON.parse(res.body).data.name` 省内存省 CPU。解析失败（非 JSON 响应）会返回 undefined 而非抛异常，务必配合 check 断言存在性。

## 4.2 HTML 处理

对 HTML 响应，k6 提供 `res.html(selector)`——底层是 Go 实现的 goquery（jQuery 语法的子集），支持 CSS 选择器查询 DOM：

```javascript
const res = http.get("https://example.com/login-page");

const csrfToken = res.html().find('input[name="csrf_token"]').attr("value");
const title = res.html("title").text();

// 典型用途：抓取表单隐藏字段后提交
http.post("https://example.com/login", {
  csrf_token: csrfToken,
  user: "u",
  pass: "p",
});
```

`html()` 同样是**惰性解析**：不调用就不构建 DOM 树，纯 API 压测零开销。需要说明的是，DOM 查询能力仅覆盖选择器/属性/文本提取，不支持执行页面 JS——这也是 k6 不适合重前端 E2E 场景的技术根源。

## 4.3 大响应与内存

高 QPS 压测的内存纪律：**默认丢弃响应体，按需捞回**。全局 `discardResponseBodies: true` 让所有响应只保留元数据（状态码、头、timing），单请求需要体时声明 `responseType: "text"`（或二进制场景 `"binary"`）：

```javascript
import http from "k6/http";
import { check } from "k6";

export const options = {
  discardResponseBodies: true,       // 全局：响应体不保留，内存占用骤降
};

export default function () {
  http.get("https://api.example.com/health");                 // 只关心状态码，零内存
  const res = http.get("https://api.example.com/items/1", {
    responseType: "text",                                      // 仅本请求保留响应体
  });
  check(res, { "item 存在": (r) => r.json("data.id") !== undefined });
}
```

## 4.4 响应时间构成

`res.timings` 把一次请求的耗时拆成六段，与浏览器 DevTools 的 Timing 面板一致：

```text
├─ blocked ─┤
            ├─ dns ─┤
                      ├── connecting ──┤
                      │                ├─ tls_handshaking ─┤
                                                           ├─ sending ─┤
                                                                       ├─ waiting ─┤
                                                                                   ├─ receiving ─┤
└────────────────────────────── duration ─────────────────────────────────────────┘
```

`waiting` 即 TTFB（发出请求到收到首字节），**基本等价于服务端处理时间**；`blocked` 是客户端排队（连接池等待）；`connecting`/`tls_handshaking` 只在建立新连接时非零。每一段都有对应的内置指标（`http_req_blocked`、`http_req_waiting` 等），排查慢请求时先看分段再下结论——waiting 长是服务端慢，blocked 长是压测机自己连接不够。

# 5. 高级客户端：httpx 与扩展

## 5.1 httpx 模块

`httpx` 是官方 jslib 提供的**带默认配置的 HTTP 客户端封装**，解决"每个请求重复写 headers/tags/baseURL"的问题，用法与 axios 神似：

```javascript
import { httpx } from "https://jslib.k6.io/httpx/0.1.0/index.js";

// 创建带默认配置的会话（在 init context 创建，VU 间隔离由各自运行时保证）
const session = new httpx.Session({
  baseURL: "https://api.example.com",
  headers: { Authorization: `Bearer ${__ENV.TOKEN}` },
});

export default function () {
  const res = session.post("/orders", JSON.stringify({ sku: "A1" }), {
    headers: { "Content-Type": "application/json" },
  });
  // session.get / put / del / request 同理；支持 batch：session.batch([...])
}
```

jslib 的模块在首次运行时从 CDN 拉取，**离线环境应把 jslib 源码下载到本地**或改用 npm 打包，否则 CI 会因外网依赖抖动而失败。

## 5.2 与 Java 压测客户端的对比

Java 团队做压测时常见两条路：JMeter 的 HTTP Sampler，或直接用业务代码里的 HTTP 客户端（OkHttp/WebClient）写压测程序。与 k6 http 模块的差异值得对照：

| 维度 | k6 http | JMeter HTTP Sampler | Java 客户端（OkHttp 等） |
|------|---------|--------------------|--------------------------|
| 配置方式 | JS 代码 + params 对象 | GUI/表单字段 | 代码 + Builder |
| 连接池 | 每 VU 独立，Go 层管理 | 全局可配（httpclient4 实现） | 应用级单例 |
| 响应断言 | check 函数 | 响应断言组件（配置式） | 手写断言代码 |
| 指标采集 | 自动分段 timing + 内置指标 | 采样器结果 + 聚合报告 | 需自行埋点（Micrometer） |
| 并发模型 | VU 同步阻塞，batch 并行 | 线程 + 异步 Sampler | 线程池/Reactor |
| 复用业务代码 | 不可（协议要重写） | 不可 | 完全复用 |

**复用业务代码是 Java 自研压测程序的唯一真优势**——签名、加密、序列化逻辑直接调生产 SDK，不会出现"压测脚本实现的签名和线上不一致"这种致命偏差。代价是并发模型、指标采集、负载曲线全要自建。k6 的交换条件是：放弃代码复用，换来开箱即用的分段 timing、指标体系与负载模型。签名逻辑复杂且多变的项目，用 xk6 把签名封装成 Go 扩展是兼顾两者的折中。

# 6. 性能与稳定性要点

## 6.1 超时与重试

k6 **不内置重试机制**——请求失败就是失败，这是刻意设计：自动重试会掩盖错误率，让"99% 请求重试两次才成功"的系统在报表上显示为健康。确需模拟客户端重试行为时显式写循环，并把重试次数计入自定义指标：

```javascript
import http from "k6/http";
import { sleep } from "k6";
import { Counter } from "k6/metrics";

const retries = new Counter("http_retries");

function getWithRetry(url, maxRetries = 2) {
  for (let i = 0; i <= maxRetries; i++) {
    const res = http.get(url, { timeout: "5s" });
    if (res.status === 200) return res;
    if (i < maxRetries) {
      retries.add(1);
      sleep(Math.pow(2, i) * 0.5);       // 指数退避：0.5s、1s
    }
  }
  return null;                            // 重试用尽仍失败，由调用方 check/fail
}
```

超时设置过小会把网络抖动放大成错误率，过大则让慢请求占住 VU——原则同第 3 篇：超时约为 P99 SLO 的 2-3 倍。

## 6.2 DNS 与连接错误

DNS 解析失败或缓慢会让**所有请求**的 blocked/dns 段激增，症状是延迟整体抬升而非个别接口变慢——k6 默认每次新建连接都解析 DNS（TTL 行为受 `noConnectionReuse` 影响），可用 `hosts` 选项把关键域名钉死到 IP 排除 DNS 因素：`options.hosts = { "api.example.com:443": "10.0.0.5:443" }`（注意带端口）。连接错误要区分 `error_code`：1xxx 段是网络层（超时、拒绝），2xxx 是 HTTP 协议层——connection refused 通常指向服务端端口/防火墙，timeout 更多指向容量或链路。

## 6.3 客户端自身瓶颈

**压测机先于被测系统到达极限**是高并发压测的常态结局：CPU 打满导致 JS 执行变慢、带宽饱和导致 receiving 段拉长、fd/端口耗尽导致批量连接失败。判别方法：观察 `http_req_blocked`（客户端排队）与 `http_req_connecting` 是否异常升高，同时看压测机自身的 CPU/带宽监控——**被测系统指标正常而延迟上涨，几乎必然是客户端瓶颈**。解法依次是：开启 `discardResponseBodies` 省内存 CPU、限制 `maxParallelRequests`、多台压测机分片（k6-operator）。

# 7. 小结

k6 的 HTTP 能力围绕"同步阻塞 + VU 并发 + batch 局部并行"的模型展开：便捷方法覆盖全部动词但 JSON body 必须手动序列化，params 的 tags 字段是指标细分与阈值过滤的基础，cookie jar 按 VU 隔离自动管理会话。协议层默认连接复用与 HTTP/1.1，短连接压测与 h2 场景需显式切换。响应处理的核心纪律是 `discardResponseBodies` 全局丢弃 + 按需 `responseType` 捞回，`res.timings` 六段分解是定位慢请求的第一工具。与 Java 压测客户端相比，k6 放弃业务代码复用换取完整的指标与负载模型，签名逻辑复杂的系统可用 xk6 扩展折中。
