# 1. 故障排查思路

## 1.1 先区分客户端与服务端问题

**排错的第一步永远不是看报错文本，而是判定问题出在施压链路还是被测系统**。两条经验法则：

- **客户端瓶颈**：吞吐（RPS）上不去或下降，但服务端 CPU/内存/延迟曲线平稳，k6 侧 `http_req_blocked` 升高、施压机 CPU 接近打满——压力根本没送达。
- **服务端瓶颈**：延迟上升伴随错误率升高，服务端资源指标（CPU、线程池、连接池、DB）同步恶化，k6 侧 `http_req_waiting` 是延迟大头。

还有一类最容易误判的第三种情况：**脚本或环境错误**——服务端好好的，但脚本写错、地址配错、证书不对，表现为大面积、瞬间出现的同一种错误。判据是错误率曲线：性能劣化的错误率是随负载爬坡逐渐上升的，配置错误的错误率是从第一秒就接近 100% 的。

## 1.2 结合指标定位

k6 把一次 HTTP 请求的耗时拆成了串行的几段，**看哪一段变大，问题就在哪一段**：

```text
http_req_duration = blocked + connecting + tls_handshaking + sending + waiting + receiving

┌──────────┬───────────────┬────────────────┬──────────┬─────────────┬────────────┐
│ blocked  │  connecting   │ tls_handshaking│ sending  │   waiting   │ receiving  │
│ 取连接/排队│  TCP 三次握手  │    TLS 握手     │ 发送请求体 │ 等服务端首字节│  下载响应体  │
├──────────┼───────────────┼────────────────┼──────────┼─────────────┼────────────┤
│ 大 = 客户端│ 大 = 网络/     │ 大 = 加密开销/  │ 大 = 上行 │ 大 = 服务端   │ 大 = 响应体 │
│ fd/连接池 │  服务端 backlog│  证书链/握手参数 │  带宽小   │  处理慢      │  大或下行慢 │
└──────────┴───────────────┴────────────────┴──────────┴─────────────┴────────────┘
```

- `http_req_blocked` 升高 → 客户端资源不足（fd、连接池、临时端口）或 DNS 慢，参见第 17 篇施压机调优。
- `http_req_connecting` 升高 → 服务端 accept 队列满（`somaxconn`/backlog）或网络路径丢包。
- `http_req_tls_handshaking` 升高 → 握手量太大（连接没复用）或加密套件不匹配导致多次协商。
- `http_req_waiting`（TTFB）升高 → **几乎总是服务端处理慢**，这是最值得盯的指标。
- `http_req_receiving` 升高 → 响应体过大或服务端流式输出慢。

## 1.3 排错流程图

拿到一个"压测结果不对劲"的现场，按这张图走：

```text
压测异常（错误率高 / 吞吐低 / 延迟大）
        │
        ▼
第一秒就 100% 错误？──是──► 配置/脚本问题：查 BASE_URL、证书、代理、
        │                  init context 报错（见第 3、4 节）
        否
        ▼
施压机 CPU > 70% 或 blocked 升高？──是──► 压测端瓶颈：调内核参数、
        │                                SharedArray、加机器（第 17 篇）
        否
        ▼
错误类型分布？
   │
   ├─ connection refused ──► 服务没监听 / 端口错 / 容器网络（localhost 陷阱）
   ├─ timeout ─────────────► 服务端过载或超时配置过短（对照服务端指标）
   ├─ reset / EOF ─────────► LB 空闲超时踢连接 / 服务端崩溃重启
   ├─ 429 / 503 ───────────► 限流、熔断、服务降级（找业务侧确认策略）
   └─ 5xx 且 waiting 大 ───► 服务端真瓶颈：查 DB 慢查询、线程池、GC
        │
        ▼
仍无法定位 ──► 降 VU 到 1 复现（k6 run --vus 1 --iterations 5 --console-output d.log）
              区分"负载相关"与"始终发生"
```

# 2. 常见错误与处理

## 2.1 连接超时（connection refused / TimeoutError）

**"connection refused"和"timeout"是两种完全不同的故障**。`dial tcp 10.0.0.5:8080: connect: connection refused` 表示 TCP SYN 到达了对端机器但端口上没有进程监听——服务没起、端口写错、或者容器网络里把 `localhost` 当成了宿主机（Docker 中 `localhost` 是容器自身，压宿主机服务要用 `host.docker.internal` 或 `--network host`）。而 `context deadline exceeded` / k6 的 `request timeout` 表示 SYN 发出去石沉大海：防火墙 DROP、服务端 accept 队列溢出、或纯粹是服务端处理时间超过了 `timeout` 选项。

排查顺序：先 `curl -v $BASE_URL/健康检查接口` 手工验证连通性；再确认超时配置是否合理（k6 全局默认 60s，脚本里 `timeout: '10s'` 会把 15s 的正常慢响应记成错误）；最后对照服务端 access log——**请求根本没到服务端是网络问题，到了但处理超时是服务端问题**。

## 2.2 TLS 握手失败

典型报错与对应根因：

| 报错文本 | 根因 | 处理 |
|----------|------|------|
| `x509: certificate signed by unknown authority` | 自签/私有 CA，k6 不信任 | 用 `--blacklist-ip` 之外的正道：把 CA 加进施压机信任链，或测试期 `insecureSkipTLSVerify: true` |
| `tls: failed to find any PEM data` | `--tls-cert`/`--tls-key` 文件路径或格式错 | 确认 PEM 格式、路径相对运行目录 |
| `remote error: tls: handshake failure` | 协议版本/加密套件不匹配（如服务端只开 TLS1.3，老版本客户端） | `--tls-version tls1.2:tls1.3` 显式指定范围 |
| `tls: first record does not look like a TLS handshake` | 用 `https://` 打了纯 HTTP 端口 | 核对 URL scheme 与端口 |

mTLS 场景记得同时配 `tls-cert`/`tls-key` 与信任的 CA。

> **注意**：`insecureSkipTLSVerify: true` 只用于隔离排查，不要留在正式脚本里——它会掩盖真实的证书过期问题，压测全绿、上线当天证书过期是真实发生过的事故。

## 2.3 429/503 大量出现

**429/503 不是 k6 的错误，是被测系统的限流与降级在起作用**——压测角度看甚至是"成功"的信号（系统有自保能力）。处理要点：先与业务侧确认限流维度（按 IP？施压机出口 IP 固定，很容易整体被限；按用户？检查参数化数据是否集中在少数账号）；确认 CDN/WAF/API 网关哪一层返回的（看响应头 `Retry-After`、`X-RateLimit-*` 或服务端日志）。若目标是压透限流之后的真实服务，需要业务侧为压测流量开白名单；若目标就是验证限流本身，把 429 计入预期结果而非失败。

## 2.4 connection reset / unexpected EOF

`read: connection reset by peer` 与 `unexpected EOF` 的共同点是：**连接建立后被对端强行关闭**。压测场景下的高频根因：负载均衡器（Nginx/ALB）的 keepalive 空闲超时或 `worker_connections` 上限，k6 复用了一条 LB 已经决定关闭的连接；服务端进程 OOM 被杀或崩溃重启；连接数超过服务端 `max_connections`。鉴别方法：如果 reset 集中在测试中途突发（而非随负载平滑上升），大概率是服务端进程出事了，去查服务端的重启记录与 OOM 日志；如果错误率与 VU 数强相关，查 LB 的连接数配置。客户端侧缓解可以用 `discardResponseBodies` 配合场景重跑，但根因要在服务端/LB 上修。

## 2.5 goeventloop 与 "Uncaught exception"

**`level=error msg="Uncaught exception"` 是 k6 日志系统（logrus）对未捕获 JS 异常的标准输出格式**，后面跟着 `source=stacktrace` 字段和完整 JS 堆栈，进程随后以退出码 107 结束（若脚本 catch 了异常则只记录不退出）。最常见来源：

```javascript
// 反例一：async 函数里抛错没人接 —— 表现为 Uncaught (in promise)
export default async function () {
  const data = await someHelper();   // someHelper 抛错时无人 catch
  http.post(BASE, data);
}

// 反例二：对空响应调用 .json() —— connection refused 时 res.status === 0
export default function () {
  const res = http.get(`${BASE}/api`);
  const body = res.json();           // 请求失败时 body 为空，这里抛
  // 正确做法：先判断请求是否成功
  // if (res.error) { return; }
  // const body = res.json();
}
```

而 `goeventloop` 是 k6 内部的 JS 事件循环实现（`internal/js/eventloop` 包）——Goja 本身没有事件循环，k6 自己写了一个来支撑 Promise/async/await。**日志里出现 goeventloop 相关错误（如 "event loop terminated" 或事件循环排空异常）通常意味着异步代码的 Promise 被拒绝且未处理，或者在事件循环已停止后仍有回调尝试执行**。处理原则：所有 async 路径要么 try/catch 要么返回 Promise 链给 k6；对 `res.json()`/`res.body` 的访问前一律先检查 `res.error` 与 `res.status`；如果堆栈指向 k6 内部且脚本无异步代码，检查 k6 版本是否为已知 bug（`k6 version` + 搜索对应 issue）。

## 2.6 内存 OOM

施压机上 k6 进程 OOM 的两种表现：Go 运行时直接 `fatal error: runtime: out of memory`（进程退出码 2），或被容器/系统 OOM killer 杀掉（Docker 里退出码 137，`dmesg | grep -i oom` 有记录）。根因排查按概率排序：

1. **init 数据文件太大且没用 SharedArray**——每个 VU 各持一份副本，内存 = VU 数 × 文件大小，这是头号杀手（见第 17 篇 3.4）。
2. **响应体堆积**——大响应 + 高 VU 数 + 未开 `discardResponseBodies`。
3. **脚本级泄漏**——把每次迭代的结果 push 进模块级数组（`allResults.push(res)`），数组随迭代无限增长；k6 脚本里模块级变量是 VU 生命周期常驻的，**绝对不要在迭代中向长生命周期容器里追加数据**，要聚合就用 `Trend`/`Counter` 自定义指标。

验证手段：压测中另开终端 `watch -n 1 'ps -o rss= -p $(pgrep k6)'` 观察 RSS 曲线——锯齿状波动正常（GC 在工作），单调爬升就是泄漏。

## 2.7 HTTP/2 兼容问题

k6 对 HTTPS 请求默认通过 ALPN 协商 HTTP/2，且没有暴露"强制 HTTP/1.1"的开关。HTTP/2 相关的疑难症状包括：连接被 `GOAWAY` 后流全部重置、某些 LB 对 h2 的流控窗口处理不当导致请求挂起、`http2: server sent GOAWAY and closed the connection` 错误间歇出现。定位方法：用 `curl -v --http2 $URL` 与 `curl -v --http1.1 $URL` 分别验证服务端两种协议的行为是否一致；若确认是服务端/LB 的 h2 实现问题，短期规避是让服务端 ALPN 只宣告 http/1.1（服务端配置），k6 会自动回落。

> **设计考量**：h2 多路复用意味着"一个 TCP 连接承载多个并发流"——k6 的连接数指标会比 h1 场景少得多，吞吐却可能卡在服务端/LB 的 `MaxConcurrentStreams` 上限上。压测 h2 服务时不要拿连接数与 h1 的经验值对比，吞吐卡住时优先查流并发上限。

# 3. 脚本问题

## 3.1 语法错误

`SyntaxError: Unexpected token` 出现在 init 阶段，进程直接退出（退出码 97 或 107）。两个常见根因：脚本用了 Goja 原生不支持的新语法而当前跑在 `--compatibility-mode=base`（换回默认 `full` 让 Babel 转译）；或 TypeScript 没编译就直接 `k6 run script.ts`（k6 v0.5x 原生支持直接跑 TS，但依赖实验性的 JS 运行时特性，稳妥做法仍是 esbuild/tsc 编译为 JS 再跑，见第 11 篇）。运行前先做低成本验证：

```bash
k6 inspect script.js          # 只解析脚本与 options，不真正压测，语法错误立刻暴露
k6 run --vus 1 --iterations 1 script.js   # 单迭代冒烟，验证逻辑链路
```

## 3.2 逻辑错误

**逻辑错误的典型表现是"请求都成功了但失败率异常"或"业务数据不对"**：登录拿到的 token 没带到后续请求、关联提取的正则在新版响应里匹配不到、数据文件读串了行。防线是给每个关键步骤加 `check` 并细分命名——`check(res, { 'login ok': ... })` 失败时 summary 里对应检查项的失败率会单独展示，比笼统的 `http_req_failed` 好定位得多。数据相关问题用 `--console-output` 打开单 VU 调试日志，打印实际发送的请求体与提取结果，肉眼比对一次就能发现"提取到 undefined 还在继续发请求"这类问题。

## 3.3 init context 里调用 VU 函数

**init 上下文（模块顶层代码）里调用 `http.get`、`check` 等 VU 专属 API 会直接报错**，形如 `http.get() can only be called in the VU context` / `Cannot use ... in the init context`。原因是 init 代码在每个 VU 启动前（以及 `k6 inspect`、`k6 archive` 时）执行，此时没有 VU 运行时环境。正确分工：

```javascript
import http from 'k6/http';
import { SharedArray } from 'k6/data';

// init 上下文：只能做 import、常量定义、加载数据文件
const BASE = __ENV.BASE_URL || 'http://localhost:8080';
const users = new SharedArray('users', () => JSON.parse(open('./users.json')));

// 错误示范（取消注释即报错）：
// const res = http.get(`${BASE}/health`);        // init 里发请求
// const token = login(users[0]);                 // init 里调自定义 VU 函数

export default function () {
  // VU 上下文：所有请求、check、sleep 都在这里
  const res = http.get(`${BASE}/health`);
  check(res, { 'health ok': (r) => r.status === 200 });
}
```

需要"测试开始前执行一次的请求"（如全局登录、准备数据），用 `setup()` 函数——它运行在专门的 setup VU 里，可以使用全部 VU API，返回值会传给每个 VU 的 default 函数和 `teardown()`。

# 4. 环境问题

## 4.1 DNS 解析异常

DNS 问题的两种面孔：`dial tcp: lookup api.example.com: no such host`（解析彻底失败——域名拼错、内网域名在公网 runner 上不可见、/etc/resolv.conf 配置错）；以及更隐蔽的**DNS 抖动推高延迟**——解析结果没有缓存或 TTL 过短，每次建连都走一次 DNS 查询，`http_req_blocked` 里混入了几毫秒到几百毫秒的解析时间。排查：`dig api.example.com +stats` 看解析耗时与 TTL；容器环境里注意 ndots 配置（K8s 默认 `ndots:5` 会让短域名尝试多次搜索域拼接，解析慢一个量级，可用 FQDN 结尾加点规避）。k6 侧可在 options 里用 `hosts` 选项做静态映射绕过 DNS：`hosts: { 'api.example.com:443': '10.0.0.5:443' }`。

## 4.2 代理与防火墙

k6 遵循 `HTTP_PROXY`/`HTTPS_PROXY`/`NO_PROXY` 环境变量。**压测流量走代理是大忌**：代理服务器的连接数限制、缓存行为、额外一跳的延迟都会污染数据，而且代理先于被测系统成为瓶颈。确认方法：`env | grep -i proxy` 检查施压机环境；内网压测把目标网段加进 `NO_PROXY`。防火墙的坑更阴险：安全组只放行了部分源 IP（新扩容的施压机被拦）、出方向流量被 DPI 设备限速、conntrack 表满导致间歇性丢包（`dmesg` 里出现 `nf_conntrack: table full, dropping packet`，处理见第 17 篇 4.3 的 `nf_conntrack_max` 调优）。

## 4.3 版本兼容检查

**排错无果时先看版本**：`k6 version` 确认施压机版本，再对照三个兼容面——脚本里 import 的远程 jslib 模块版本（`https://jslib.k6.io/k6-summary/0.1.0/index.js` 这类 URL 里带版本号，k6 v0.5x 起支持不带版本号的 latest 形式但不推荐，构建不可重现）；xk6 扩展编译时链接的 k6 版本（**扩展必须用与运行环境相同版本的 k6 源码编译**，`xk6 build v0.57.0 --with ...`，版本错配会直接 panic）；以及 CI 镜像 tag（`grafana/k6:latest` 在流水线里是隐患，固定到具体版本）。k6 各版本的 breaking change 集中在 options 语义与输出格式上，升级前过一遍官方 release notes 的 "Breaking changes" 段落。

# 5. 排查清单

## 5.1 快速检查项

> **行业实践**：把"单 VU 最小复现"固化为排错的第一动作而不是最后手段——绝大多数"诡异"的压测异常在 `--vus 1` 下要么立刻复现（脚本/配置问题），要么直接消失（并发/容量问题），十分钟内就能把问题空间砍掉一半。

- **脚本版本与配置是否正确**：git 分支/commit 对不对、`BASE_URL` 指向的环境是不是想压的那个、`k6 inspect` 确认 options 实际生效值。
- **目标服务是否可访问**：施压机上 `curl -v` 手工请求健康检查接口，覆盖协议、证书、代理、DNS 全链路。
- **客户端资源是否充足**：CPU < 70%、fd 余量、TIME_WAIT 数量、带宽，对照第 17 篇的标定基线。
- **错误是从第一秒就有还是随负载出现**：前者查配置，后者查容量。
- **单 VU 能否稳定复现**：`--vus 1 --iterations 5 --console-output debug.log`，能复现是脚本/环境问题，不能复现是并发/容量问题。

## 5.2 常用排查命令速查

```bash
k6 version                                          # 版本确认
k6 inspect script.js                                # 静态解析脚本与 options
k6 run --vus 1 --iterations 1 --console-output d.log script.js   # 最小复现
curl -v --http2 $BASE_URL/health                    # 全链路连通性与协议协商
dig api.example.com +stats                          # DNS 解析耗时与 TTL
ss -s && ss -tan state time-wait | wc -l            # 施压机 socket 状态总览
ls /proc/$(pgrep k6)/fd | wc -l                     # k6 进程 fd 占用
dmesg -T | grep -iE 'oom|conntrack'                 # OOM 与 conntrack 丢包记录
pidstat -p $(pgrep k6) 1                            # k6 进程 CPU/内存曲线
```

# 6. 小结

k6 排错的方法论是"三段定位"：先用错误率曲线形态区分配置错误（第一秒就全错）与性能问题（随负载爬坡），再用施压机资源指标排除客户端瓶颈，最后用 `http_req_*` 分段耗时把问题钉到具体环节——blocked 看客户端、connecting 看网络与 backlog、waiting 看服务端处理。高频错误各有明确指纹：refused 查监听与容器网络、timeout 查防火墙与服务端负载、reset/EOF 查 LB 空闲超时与进程崩溃、429/503 查限流策略、TLS 报错对证书链与协议版本、goeventloop/Uncaught exception 查未处理的 Promise 与空响应解析、OOM 查 SharedArray 缺失与脚本级数组堆积、init context 报错查 VU API 误用。版本兼容问题优先核对 k6 二进制、jslib 模块、xk6 扩展三者的版本一致性。所有排查最后都收敛到同一个动作：降到单 VU 最小复现——能稳定复现的问题都不难修。
