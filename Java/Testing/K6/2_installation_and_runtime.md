# 1. 安装与版本管理

## 1.1 二进制安装

k6 以**单个静态编译二进制**的形式分发，无运行时依赖（不需要 JVM、Python、Node.js），从 GitHub Releases 下载对应平台的压缩包，解压后把 `k6`（或 `k6.exe`）放入 PATH 即可。这是所有安装方式中最推荐的基线做法——二进制版本明确、可校验、可在团队内统一分发：

```bash
# 下载并解压后验证
k6 version
# 输出形如：k6 v0.55.0 (go1.22.x, linux/amd64)
```

建议团队**固定使用同一版本号**（例如统一 v0.55.0），把版本号写进 README 或 Dockerfile，避免成员之间因小版本差异导致脚本行为或输出格式不一致。k6 的 minor 版本偶有破坏性变更（废弃 API、指标改名），major 版本前不承诺完全兼容。

## 1.2 包管理器安装

各主流平台都有官方维护的包管理渠道，命令如下：

```bash
# macOS (Homebrew)
brew install k6

# Windows (Chocolatey)
choco install k6

# Windows (winget，Win10 1809+/Win11 内置)
winget install k6 --source winget

# Linux - Debian/Ubuntu（官方 APT 源）
sudo gpg -k
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg \
  --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D68
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" \
  | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update && sudo apt-get install k6

# Linux - RHEL/CentOS/Fedora（官方 RPM 源）
sudo dnf install https://dl.k6.io/rpm/repo.rpm
sudo dnf install k6
```

包管理器的优势是升级方便（`brew upgrade k6`、`apt-get upgrade k6`），劣势恰恰也是"方便升级"——**CI 环境里不应让包管理器隐式决定版本**。生产压测流水线推荐两种方式锁死版本：下载指定版本的二进制归档，或用固定 tag 的 Docker 镜像。

## 1.3 Docker 方式

官方镜像 `grafana/k6` 适用于 CI/CD 与无本地环境场景，避免在构建机上安装任何工具链。镜像内置 k6 二进制与 ca-certificates，体积很小：

```bash
# 从标准输入读脚本运行（脚本不落盘）
docker run --rm -i grafana/k6 run - < script.js

# 挂载脚本目录运行（推荐，脚本可引用同目录数据文件）
docker run --rm -v "$(pwd):/scripts" -w /scripts \
  grafana/k6 run --vus 10 --duration 30s script.js

# 固定版本 tag，避免 latest 漂移
docker run --rm -i grafana/k6:0.55.0 run - < script.js
```

> **注意**：容器内压测**被测目标若是宿主机服务**，URL 不能用 `localhost`（那是容器自己的回环地址），Linux 上要加 `--add-host=host.docker.internal:host-gateway` 或用宿主机内网 IP；macOS/Windows 的 Docker Desktop 可直接使用 `host.docker.internal`。另一个常被忽略的点：容器默认的文件描述符与网络配额可能低于宿主机，高 VU 数压测前应确认 `--ulimit nofile` 设置。

## 1.4 版本选择与升级策略

版本管理的三条纪律：**生产压测固定版本**，升级前先在低风险环境用存量脚本回归一遍；**关注 CHANGELOG 中的三类变更**——执行器语义调整、内置指标改名/废弃、summary 输出格式变化，这三类直接影响历史数据可比性与 CI 判定；**保留旧版本二进制**，当新版本结果与基线出现无法解释的偏差时，能用旧版本交叉验证是工具问题还是被测系统问题。

# 2. 运行时基础

## 2.1 运行命令与参数

`k6 run` 是最核心的子命令，其余子命令（`k6 archive` 打包脚本、`k6 cloud` 上云执行、`k6 inspect` 查看脚本元数据、`k6 convert` 转换 HAR）使用频率低得多。常用参数按用途分组：

```bash
# 负载控制（覆盖脚本内 options 的简写形式）
k6 run --vus 50 --duration 2m script.js
k6 run --iterations 1000 --vus 20 script.js    # 总迭代数封顶，跑完即止
k6 run --stage 30s:20,1m30s:50,10s:0 script.js # 爬坡：30s 升到 20 VU，保持到 1m30s 升 50，10s 归零

# 结果输出
k6 run --out json=result.json script.js        # 逐条指标写入 JSON 流
k6 run --out csv=result.csv script.js          # CSV 格式
k6 run --summary-export=summary.json script.js # 仅导出结束时汇总表
k6 run --summary-trend-stats="avg,min,med,max,p(90),p(95),p(99)" script.js

# 运行行为
k6 run --no-thresholds script.js               # 忽略阈值判定（调试用）
k6 run --no-summary script.js                  # 不打印汇总表
k6 run --paused script.js                      # 启动后暂停，配合 REST API 手动控制
k6 run --quiet script.js                       # 抑制非必要输出（CI 降噪）
k6 run --tag run_id=$BUILD_NUMBER script.js    # 给所有指标打全局标签
```

`--stage` 的语法是 `持续时间:目标VU数` 的逗号分隔序列，等价于脚本里的 ramping-vus 执行器，适合命令行快速试探负载曲线。

> **行业实践**：CI 里的标准形态是 `k6 run --out json --summary-export --tag`，三者齐备：json 流供后续入库做趋势分析，summary 供人当场看，tag（构建号、commit SHA）供关联代码变更。只跑 `k6 run script.js` 让结果烂在控制台日志里，是压测流水线的常见反模式。

## 2.2 配置来源与管理策略

k6 的配置来源有四层，**优先级从高到低：命令行参数 > 环境变量 > 配置文件 > 脚本 options**。环境变量与配置文件中的选项名遵循"K6_ 前缀 + 驼峰转大写下划线"的规则，例如脚本里的 `vus` 对应 `K6_VUS`、`summaryTrendStats` 对应 `K6_SUMMARY_TREND_STATS`、`insecureSkipTLSVerify` 对应 `K6_INSECURE_SKIP_TLS_VERIFY`。

管理策略上建议：**脚本 options 承载"这个测试是什么"**（负载模型、阈值——它们是测试定义的一部分，必须进 Git）；**环境变量承载"这次在哪跑"**（目标地址、凭据、代理）；**命令行只做临时覆盖与调试**。反过来把负载模型写在命令行里，脚本进 Git 也没法复现测试，会造成"参数漂移"——半年后没人知道当初到底跑了多少 VU。

## 2.3 环境变量与机密

通过环境变量注入运行参数与敏感信息，避免密码进代码库：

```bash
# 覆盖 k6 运行选项（K6_ 前缀）
K6_VUS=20 K6_DURATION=30s k6 run script.js
K6_OUT=csv=output.csv k6 run script.js

# 注入业务变量（-e 参数，脚本内经 __ENV 读取）
k6 run -e BASE_URL=https://staging.example.com -e API_TOKEN=secret script.js

# 也可用系统环境变量（需 k6 v0.55+ 配合 --env-file 或直接导出）
export BASE_URL=https://staging.example.com
k6 run --env-file .env.local script.js
```

脚本内读取：

```javascript
// __ENV 是全局对象，未设置的变量取值为 undefined，务必给默认值
const baseUrl = __ENV.BASE_URL || "https://test.k6.io";
const token = __ENV.API_TOKEN;

if (!token) {
  // init 阶段直接抛错，快速失败优于跑到一半发现 401
  throw new Error("缺少 API_TOKEN 环境变量");
}
```

> **注意**：`k6 run -e` 传入的值会出现在进程列表（`ps` 输出）与 k6 的归档文件（`k6 archive` 会把 `__ENV` 快照打进去）中。**真正的机密应通过 CI 平台的 Secret 机制注入环境变量**，而不是拼在命令行上；归档文件若包含机密，绝不能上传共享。

## 2.4 日志与输出

k6 自身的日志走 stderr，级别用 `--console-output`/`K6_CONSOLE_OUTPUT` 与 `--log-level`（或 `-v` 详细、`-l debug` 调试）控制。脚本里的 `console.log` 默认输出到 stdout，可用 `--console-output=console.txt` 重定向到文件。两条纪律：**压测执行期禁止高频 console.log**——每条日志都要跨越 JS 与 Go 层边界并做字符串拼接，1000 VU 每秒打一行就足以显著拉低吞吐并扭曲延迟测量；**结果输出统一归档**，JSON 流按 run_id 命名入库，否则趋势分析无从谈起。

# 3. 执行器与 VU 概念

## 3.1 VU 与迭代的运行时语义

**VU（Virtual User）在运行时是一个 goroutine 加一个独立的 Goja JS 运行时实例**。理解这一点能解释 k6 的多数行为：VU 之间不共享任何 JS 变量（各自解释器隔离），所以全局可变状态天然不存在竞态；VU 的数量可以在测试中动态变化（ramping），新 VU 的创建成本极低；每个 VU 顺序执行迭代，一个 VU 在同一时刻只有一个进行中的迭代。`--vus 50 --duration 2m` 的含义是：维持 50 个这样的执行单元循环跑 `default` 函数 2 分钟，总迭代数取决于每次迭代多快跑完。

## 3.2 执行器类型总览

执行器（Executor）声明"负载如何随时间变化"。k6 v0.5x 内置六种，按闭合/开放模型分类：

| 执行器 | 模型 | 关键参数 | 负载语义 |
|--------|------|----------|----------|
| shared-iterations | 闭合 | vus, iterations, maxDuration | N 个 VU 共享总共 M 次迭代，跑完即停 |
| per-vu-iterations | 闭合 | vus, iterations, maxDuration | 每个 VU 各自跑 M 次迭代 |
| constant-vus | 闭合 | vus, duration | 恒定 N 个 VU 跑 T 时长 |
| ramping-vus | 闭合 | stages | VU 数按阶段爬坡/回落 |
| constant-arrival-rate | 开放 | rate, timeUnit, duration, preAllocatedVUs, maxVUs | 恒定每秒启动 M 次迭代 |
| ramping-arrival-rate | 开放 | stages, preAllocatedVUs, maxVUs | 迭代到达率按阶段爬坡 |

```javascript
export const options = {
  scenarios: {
    // 闭合模型：恒定 20 个 VU 跑 5 分钟
    steady_browsing: {
      executor: "constant-vus",
      vus: 20,
      duration: "5m",
    },
    // 开放模型：每秒稳定发起 100 次迭代，不管上次是否完成
    api_traffic: {
      executor: "constant-arrival-rate",
      rate: 100,             // 到达率
      timeUnit: "1s",        // 到达率的时间单位
      duration: "5m",
      preAllocatedVUs: 50,   // 预分配 VU 池（至少 rate × 平均迭代秒数）
      maxVUs: 200,           // 系统变慢时允许扩容到的上限
    },
  },
};
```

**shared-iterations 是 `--vus` + `--iterations` 的默认映射**，也是不写 scenarios 时 k6 的缺省执行器；`--vus` + `--duration` 则映射到 constant-vus。多 scenario 可以同时声明，并行执行、各自独立的负载曲线，且通过 `exec` 字段指定各自的入口函数、通过 `tags`/`env` 区分环境。

## 3.3 开放模型的 preAllocatedVUs 陷阱

开放模型下**到达率是硬承诺**：如果系统响应变慢，已分配的 VU 都在等待响应，k6 必须创建新 VU 来维持到达率，直到 maxVUs。预分配不足时日志会出现 `Insufficient VUs` 警告——这表示 k6 没兑现你要求的到达率，测试结果已经失真。经验公式：**preAllocatedVUs ≥ rate × P99 迭代耗时（秒）**，maxVUs 留 2-4 倍余量。这个警告必须当作测试无效信号处理，而不是噪声。

> **设计考量**：闭合模型测的是"50 个在线用户会给系统带来什么"，开放模型测的是"每秒 100 个请求砸过来系统扛不扛得住"。两者不可互相替代——用闭合模型压测 API 会因协调遗漏低估延迟（系统变慢→发压变慢→排队被掩盖），用开放模型模拟在线用户又会施加超出真实的负载。写 scenarios 前先回答：被测系统是"用户驻留型"还是"流量到达型"。

# 4. 运行资源与限制

## 4.1 CPU 与内存估算

经验数据：**每个空闲 VU 约占 1-2MB 内存，发压中的 VU 视响应体大小可达数 MB**；CPU 消耗主要由请求速率与 JS 逻辑复杂度决定，纯 HTTP 压测下 8 核机器驱动 3000-5000 VU、上万 RPS 是常见水平。长时间稳定性测试要额外预留 30%-50% 内存余量——Go 运行时归还内存给操作系统是惰性的，加上指标聚合缓冲，内存水位会缓慢爬升后被误判为泄漏。压测机自身 CPU 若被打满，测量出的延迟里混入了客户端排队时间，结果不可信；**压测期间必须同时监控压测机自身的负载**。

## 4.2 连接数与系统限制

Linux 上的三个经典限制：**文件描述符上限**（每个 TCP 连接占一个 fd，`ulimit -n` 默认 1024 远不够，压测前 `ulimit -n 250000`）；**临时端口范围**（`net.ipv4.ip_local_port_range` 默认约 28000 个端口，高频短连接会在几分钟内耗尽，需扩到 `1024 65535`）；**TIME_WAIT 堆积**（短连接关闭后端口要等 2MSL 才释放，可开启 `net.ipv4.tcp_tw_reuse` 缓解）。这些限制的症状都是"连接到 5 万级并发后突然大量 connection refused / timeout"，而被测系统监控却显示一切正常——排查瓶颈时先看压测机，再看服务端。

## 4.3 网络与代理

k6 默认识别 `HTTP_PROXY`/`HTTPS_PROXY`/`NO_PROXY` 环境变量；内网压测要确认 DNS 解析路径（容器内 DNS、/etc/hosts）与防火墙策略。若被测服务只接受白名单来源，压测机的出口 IP 需要提前报备。压测流量走代理会引入额外延迟与代理自身的吞吐上限——**除非链路必须，压测应直连目标**，并把代理带宽计入容量评估。

# 5. 常见问题与排查

## 5.1 启动失败

`k6: command not found` 是 PATH 未配置或二进制无执行权限（Linux/macOS 下 `chmod +x`）；脚本解析失败会报 Goja 语法错误并指向行列号，常见原因是用了 k6 不支持的新 JS 语法（Goja 对部分 ES2020+ 特性如 top-level await、class fields 支持不完整）；`unknown executor` 则多半是执行器名拼写错误或版本过旧。**用 `k6 inspect script.js` 可以在不执行的情况下验证脚本能否被解析**，CI 里适合做前置冒烟检查。

## 5.2 HTTPS/TLS 问题

自签名证书与证书链不全是压测环境的头号 TLS 故障，报错形如 `x509: certificate signed by unknown authority`。正解是把内部 CA 加入信任：脚本 options 里配置 `blacklistIPs` 之外，用 `K6_INSECURE_SKIP_TLS_VERIFY=true`（或 options 的 `insecureSkipTLSVerify: true`）**仅限测试环境临时使用**——跳过校验的同时也跳过了对证书有效期、域名的检查，可能掩盖真实的配置错误。mTLS 双向认证通过 options 的 `tlsVersion`、`tlsCipherSuites` 与命令行证书参数配置。

## 5.3 时间与结果偏差

脚本内时间处理统一用 UTC 或显式时区，避免夏令时与跨时区团队的日志对不上；压测机与服务端时钟必须 NTP 同步，否则关联两侧日志排查慢请求时时间轴会错位。另一类隐性偏差来自**测试时长不足**：短于 1 分钟的测试里，TCP 连接建立、TLS 握手、服务端的缓存冷启动都摊不薄，P95 会显著高于稳态值——建立基线的测试至少要跑 5 分钟以上，并丢弃前 30-60 秒的预热段数据（可用 ramping 阶段自然覆盖）。

# 6. 小结

k6 的安装是"下载一个二进制"级别的简单，但**运行时配置管理的纪律决定了压测结果的可复现性**：版本固定、负载模型进脚本、机密走环境变量、结果带 tag 归档。执行器体系是运行时的核心概念——闭合模型（constant-vus 系）模拟在线用户，开放模型（arrival-rate 系）模拟真实到达流量，API 压测优先开放模型并警惕 `Insufficient VUs` 失真信号。资源层面，压测机的 fd 上限、临时端口、CPU 水位都可能先于被测系统成为瓶颈，"结果异常先看客户端"应成为排查本能。
