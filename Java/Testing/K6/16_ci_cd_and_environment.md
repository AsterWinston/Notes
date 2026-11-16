# 1. CI/CD 集成目标

## 1.1 为什么要集成

**把性能测试放进 CI/CD 的核心动机是让性能回归和功能回归一样"提交即验证"**。传统做法是发布前由测试团队手动压测一轮，周期长、频次低，一次引入 300ms 延迟的慢查询可能在主干上存活数周才被发现。k6 的设计从一开始就面向这个场景：脚本即代码（与业务代码同仓库、同评审、同版本管理）、阈值即门禁（thresholds 不达标时进程以非零退出码结束，天然适配流水线）、结果是机器可读的（JSON/CSV 输出可归档可对比）。这三点合起来，性能验证就从"发布前的一次性活动"变成了"每次合并都执行的持续反馈"。

需要提前想清楚的是收益边界：CI 中的共享 runner 资源不稳定、被测环境往往是低配 staging，**CI 性能测试的定位是"防回归的趋势门禁"，不是"精确的容量结论"**。绝对数值（P95 = 180ms）在不同 runner 上会漂移，但相对变化（本次比基线慢 40%）是可靠的信号。

## 1.2 适用场景

适合放进流水线的场景按价值排序：

- **关键 API 回归测试**：核心交易/查询链路每次合并跑一遍低并发冒烟，阈值宽松但趋势可查，主要拦截"新增 N+1 查询""误删缓存"这类明显劣化。
- **版本发布前的性能门禁**：release 分支或 tag 触发一轮接近真实负载的压测，阈值严格（如 `http_req_failed < 1%` 且 `P95 < SLO`），失败直接阻断发布。
- **基础设施变更验证**：数据库升配、中间件换版本、K8s HPA 策略调整时，用同一份脚本压同一组接口，对比变更前后。

不适合放进常规 CI 的场景：长时浸泡测试（soak，动辄数小时）、需要专属硬件的大规模压测（上万 VU）、破坏性测试（chaos）。这些放到定时任务或独立环境执行，见 2.7 的分层策略。

# 2. 集成策略

## 2.1 关键原则

- **选择稳定可重复的测试场景**：CI 门禁脚本必须幂等——每次运行创建的数据要清理（teardown 或专用测试租户），依赖的下游要可控（mock 或专用环境），否则失败率上升后团队会迅速失去对门禁的信任，开始"重跑直到绿"。
- **设定清晰阈值，避免误报**：阈值基于历史基线留足余量（例如基线 P95 = 200ms，门禁设 350ms 而非 210ms）。共享 runner 的 CPU 争抢会造成 20%~50% 的正常波动，阈值太紧就是狼来了。
- **PR 门禁必须快**：控制在 1~3 分钟内。冒烟用 1~5 个 VU 跑几十个迭代即可，目的不是打出负载，而是验证"链路通、无异常、无明显劣化"。
- **失败信息可定位**：把 summary、原始指标、k6 退出码都归档为 artifact，让人不用重跑就能看到失败细节。

## 2.2 结果归档

**归档的目标是"每个构建版本都能追溯到一份性能报告"**。k6 提供三种可归档产物：`--out json=result.json` 输出逐条样本（事后可用脚本做任意维度对比）、`handleSummary()` 在测试结束时生成自定义报告文件、以及标准输出里的文本 summary（CI 日志自带）。推荐 handleSummary 方式：

```javascript
import { textSummary } from 'https://jslib.k6.io/k6-summary/0.1.0/index.js';

// 测试结束时由 k6 调用，返回值决定 summary 写到哪里
export function handleSummary(data) {
  return {
    stdout: textSummary(data, { indent: ' ', enableColors: false }), // CI 日志里可读
    'summary.json': JSON.stringify(data, null, 2),                   // 落盘，供 CI 归档对比
  };
}
```

> **注意**：`--summary-export=summary.json` 在 v0.5x 中仍可用但已标记废弃，官方推荐迁移到 `handleSummary()`。另外 jslib 的 `textSummary` 是运行时从网络拉取的，离线 CI 环境要把该文件下载到仓库里本地 import。

归档时把 git commit SHA / 镜像 tag 写进测试标签（`k6 run --tag version=${GIT_SHA}`），summary.json 随 artifact 保存，后续可以用脚本拉取多版本数据画趋势线。

## 2.3 GitHub Actions 集成示例

用官方 action 最省事，`grafana/k6-action` 会把 action 级环境变量透传给脚本（`__ENV` 可直接读取）：

```yaml
name: perf-smoke
on:
  pull_request:
    branches: [main]

jobs:
  k6-smoke:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run k6 smoke test
        uses: grafana/k6-action@v0.3.1
        with:
          filename: perf/smoke.js            # 相对仓库根目录的脚本路径
        env:
          BASE_URL: https://staging.example.com
          K6_VUS: "5"                        # K6_ 前缀变量直接覆盖 options
          K6_DURATION: "1m"
          API_TOKEN: ${{ secrets.STAGING_TOKEN }}

      - name: Upload summary
        if: always()                         # 失败也要归档，方便定位
        uses: actions/upload-artifact@v4
        with:
          name: k6-summary
          path: summary.json
```

不用 action、手动安装 k6 的写法（apt 官方源，版本可控）：

```yaml
      - name: Install k6
        run: |
          sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg \
            --keyserver hkp://keyserver.ubuntu.com:80 \
            --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
          echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" \
            | sudo tee /etc/apt/sources.list.d/k6.list
          sudo apt-get update && sudo apt-get install -y k6
          k6 version                         # 打印版本，排查兼容问题用

      - name: Run test
        run: k6 run --out json=result.json perf/smoke.js
```

## 2.4 GitLab CI 集成示例

直接用官方 Docker 镜像，把 entrypoint 清空以便在 script 里写完整命令：

```yaml
k6_smoke:
  stage: test
  image:
    name: grafana/k6:latest
    entrypoint: [""]                         # 镜像默认 entrypoint 是 k6，清空后自己写
  variables:
    BASE_URL: "https://staging.example.com"
    K6_VUS: "5"
    K6_DURATION: "1m"
  script:
    - k6 run --out json=result.json perf/smoke.js
  artifacts:
    when: always                             # GitLab 同样要 always，失败时保留现场
    paths:
      - summary.json
      - result.json
    expire_in: 30 days
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
```

## 2.5 Jenkins 集成示例

Jenkins 场景下 k6 通常预装在 agent 上，重点是**正确区分退出码**，把"阈值失败"和"k6 自身崩溃"分开处理：

```groovy
stage('Performance smoke') {
  steps {
    withCredentials([string(credentialsId: 'staging-api-token', variable: 'API_TOKEN')]) {
      sh '''
        set +e
        k6 run --out json=result.json \
          -e BASE_URL=https://staging.example.com \
          -e API_TOKEN="$API_TOKEN" \
          perf/smoke.js
        K6_EXIT=$?
        set -e
        if [ $K6_EXIT -eq 99 ]; then
          echo "阈值未达标，性能门禁失败"; exit 99
        elif [ $K6_EXIT -ne 0 ]; then
          echo "k6 异常退出（code=$K6_EXIT），疑似脚本或环境问题"; exit $K6_EXIT
        fi
      '''
    }
  }
  post {
    always { archiveArtifacts artifacts: 'result.json,summary.json', allowEmptyArchive: true }
  }
}
```

## 2.6 Docker 镜像运行 k6

官方镜像 `grafana/k6` 基于 Alpine，适合本地统一环境和自建 runner。两种典型用法：

```bash
# 脚本从 stdin 传入（适合单文件脚本，无需挂载）
docker run --rm -i grafana/k6:latest run - <perf/smoke.js

# 挂载脚本目录（多文件工程、带数据文件时的标准做法）
docker run --rm -v "$PWD/perf:/scripts" \
  -e BASE_URL=https://staging.example.com \
  grafana/k6:latest run /scripts/smoke.js
```

> **注意**：容器里的 `localhost` 指向容器自身，不是宿主机。压测跑在宿主机上的服务时，Linux 用 `--network host`，macOS/Windows 用 `host.docker.internal` 替代 localhost，否则会得到一片 `connection refused` 却误以为服务挂了。

镜像 tag 建议固定版本（`grafana/k6:v0.57.0`）而不是 `latest`——k6 次版本间偶有行为变化，门禁脚本对版本敏感。

## 2.7 分层测试策略

**冒烟测试放 PR 流水线，负载测试放定时任务，二者用同一份脚本、不同的参数**。这是成本与信号质量的平衡：PR 门禁追求快而稳定，夜间负载追求真实而全面。

| 层级 | 触发方式 | 负载规模 | 时长 | 阈值策略 | 目标 |
|------|----------|----------|------|----------|------|
| 冒烟 smoke | 每个 PR | 1~5 VU，几十次迭代 | < 2 分钟 | 宽松（错误率、明显劣化） | 链路可用性、防低级回归 |
| 负载 load | 夜间 cron / release 前 | 接近生产峰值 | 10~30 分钟 | 严格（对齐 SLO） | 真实性能门禁 |
| 浸泡 soak | 每周定时 | 中等持续负载 | 1~4 小时 | 内存泄漏、连接泄漏趋势 | 稳定性问题 |
| 压力 stress | 手动触发 | 递增直至崩溃 | 按需 | 不设门禁，只记录拐点 | 容量规划 |

夜间负载测试的 GitHub Actions 定时触发（cron 表达式为 UTC 时间）：

```yaml
on:
  schedule:
    - cron: '30 2 * * 1-5'    # 每周一至周五 UTC 02:30，即北京时间 10:30
```

同一份脚本通过环境变量切换强度：

```javascript
// PR 冒烟与夜间负载共用脚本，仅参数不同
export const options = {
  vus: Number(__ENV.K6_VUS || 5),
  duration: __ENV.K6_DURATION || '1m',
};
```

> **行业实践**：定时负载测试的结果写入时序库（`--out experimental-prometheus-rw` 或 InfluxDB），在 Grafana 上建"每日 P95 趋势"面板。门禁只看单次结果容易被噪声误伤，趋势面板 + 单次门禁的组合才是可持续的运营方式。

# 3. 环境管理

## 3.1 多环境配置

**k6 通过全局对象 `__ENV` 读取环境变量，同一份脚本适配 dev/staging/prod 多环境**。注入方式有两种：`-e KEY=VALUE` 命令行参数，或直接使用系统环境变量（`k6 run` 会默认把进程环境变量暴露进 `__ENV`）。此外 `K6_` 前缀的环境变量会被解释为 k6 options（如 `K6_VUS=10` 等价于 `--vus 10`），这是在不改脚本的情况下调整负载的正规途径：

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

// 环境变量读取 + 默认值兜底，本地开发与 CI 共用同一脚本
const BASE_URL = __ENV.BASE_URL || 'http://localhost:8080';
const API_TOKEN = __ENV.API_TOKEN;

export default function () {
  const res = http.get(`${BASE_URL}/api/v1/items`, {
    headers: { Authorization: `Bearer ${API_TOKEN}` },
  });
  check(res, { 'status is 200': (r) => r.status === 200 });
  sleep(1);
}
```

```bash
# 方式一：-e 显式传入（推荐，意图清晰）
k6 run -e BASE_URL=https://staging.example.com -e API_TOKEN=xxx perf/smoke.js

# 方式二：K6_ 前缀覆盖 options，脚本零改动
K6_VUS=50 K6_DURATION=10m k6 run -e BASE_URL=https://staging.example.com perf/smoke.js
```

**设计考量**：不要在仓库里维护 `config-dev.js` / `config-prod.js` 多份配置文件——脚本分叉后必然漂移。正确做法是单脚本 + 环境变量注入差异项（地址、凭据、负载强度），环境差异全部外置。

## 3.2 机密与权限

**token、密码等敏感信息一律走 CI 的 secrets 机制，运行时注入，绝不写入脚本或提交到仓库**。各平台的对应物：GitHub Actions 的 `${{ secrets.XXX }}`、GitLab 的 masked CI/CD variables、Jenkins 的 `withCredentials`。k6 侧只需从 `__ENV` 读取。

几个容易踩的泄密点：

- 脚本里 `console.log(__ENV.API_TOKEN)` 调试后忘记删除，token 进入 CI 公开日志。日志脱敏只能兜底，不能依赖。
- `k6 archive` / `k6 cloud run` / `k6 inspect` **默认不包含系统环境变量**——这是官方故意的安全措施，防止把本机环境里的敏感变量打包上传到 k6 Cloud。确有需要时用 `--include-system-env-vars` 显式开启，但更稳妥的做法是只用 `-e` 传入明确需要的那几个变量。
- 使用 Grafana Cloud k6 时，把 secret 配置为云端环境变量（支持标记 secret），脚本同样从 `__ENV` 读，避免凭据出现在上传的脚本内容里。

# 4. 失败处理

## 4.1 阈值失败

**阈值失败时 k6 以退出码 99 结束，流水线据此直接阻断**——这是把性能纳入门禁的关键机制。threshold 在测试结束时统一判定；配置了 `abortOnFail` 的阈值一旦在运行中失败会立即中止测试（同样退出码 99），适合"错误率超 5% 就别浪费时间跑完"的场景。

必须记住的退出码语义（v0.5x）：

| 退出码 | 常量名 | 含义 | CI 处理建议 |
|--------|--------|------|-------------|
| 0 | Success | 正常完成且阈值全部通过 | 绿灯 |
| 97 | InvalidConfig | 配置非法（options 解析失败等） | 脚本 bug，立即修 |
| 98 | InvalidUsage | 命令用法错误 | 脚本 bug，立即修 |
| 99 | ThresholdsHaveFailed | 阈值未达标 | 性能门禁失败，阻断 |
| 100 | SetupExhausted | `setup()` 抛异常 | 环境或前置数据问题 |
| 101 | TeardownExhausted | `teardown()` 抛异常 | 清理逻辑问题 |
| 102 | GoError | 脚本触发 Go 层错误 | 多为 API 误用 |
| 103 | SystemError | 系统级错误 | 查 runner 资源 |
| 105 | InvalidArguments | 无效参数 | 脚本 bug |
| 106 | ExternalSystemError | 外部系统错误（如 k6 Cloud 不可达） | 重试或报平台 |
| 107 | ScriptException | 未捕获的 JS 异常 | 脚本 bug，按堆栈修 |
| 108 | InternalError | k6 自身 bug | 升级版本 / 提 issue |

> **注意**：99 和 107 是 CI 中最常见的两种非零退出码，含义完全不同——99 是"测试跑完了但性能不达标"（真实的性能问题），107 是"脚本自己崩了"（代码 bug）。流水线里应当区分处理：99 通知性能负责人，107 直接当构建失败找脚本作者。

探索性压测、容量摸底这类**不希望被阈值阻断**的运行，加 `--no-thresholds`：它禁用全部阈值判定（包括 abortOnFail），k6 只采集数据、退出码不受阈值影响。适合"先看看系统极限在哪"的阶段，避免为跑完一次探索而反复注释脚本里的 thresholds。

## 4.2 非预期故障

**流水线里要区分"性能不达标"与"测试没跑成"**。退出码 100/103/106、大面积 `connection refused`、DNS 解析失败都属于后者——此时性能数据无意义，正确动作是重试并修环境，而不是把失败记入趋势。实践上可以：

- 对退出码 103/106 允许自动重试 1 次（网络抖动、平台瞬时故障）；99/107/97 永不自动重试（重试只会掩盖问题）。
- 在 summary 中检查 `http_req_failed` 与错误类型分布：如果失败集中在 `connection refused` / `timeout`，先怀疑被测服务没起来或 runner 网络不通，而非性能劣化。
- 门禁脚本开头加一个前置探活步骤（curl 健康检查接口），失败则直接报"环境问题"，不进入压测阶段。

# 5. 常见问题

## 5.1 环境不一致

**环境差异导致的性能结论失真，是 CI 性能测试最大的坑**。三个层面：runner 层（共享 runner 的 CPU 型号、核数、邻居负载都不可控，同样的脚本 P95 能差 50%）、网络层（runner 到 staging 的链路延迟与生产用户完全不同）、被测环境层（staging 数据库数据量只有生产的 1%、下游是 mock）。对策按优先级：阈值只做趋势对比不做绝对值判定；重要门禁用自托管 runner（固定硬件）；staging 尽量灌入接近生产量级的数据。如果都做不到，至少在报告里记录 runner 规格，让数据可解释。

## 5.2 测试耗时过长

CI 中的性能测试要克制：**PR 门禁只跑短链路、核心场景，完整场景矩阵留给定时任务**。控制手段包括：冒烟用 `per-vu-iterations` 执行器精确限定迭代数（`iterations: 20, vus: 2`）；砍掉 `sleep` 思考时间（冒烟阶段不需要模拟真人节奏）；把多场景脚本拆分为 smoke.js / load.js 两个入口，共用模块化的业务函数。一个健康的状态是：PR 门禁 90 秒内出结果，夜间负载 30 分钟内跑完全部核心链路。

# 6. 小结

CI/CD 集成的本质是把 k6 的三个特性接入流水线：脚本即代码（进仓库、随版本演进）、阈值即门禁（退出码 99 阻断构建）、结果即数据（JSON 归档画趋势）。三大平台的接入方式殊途同归——GitHub Actions 用官方 action 或 apt 安装，GitLab 直接跑 `grafana/k6` 镜像，Jenkins 预装二进制并精细处理退出码。环境差异全部通过 `__ENV` 外置，机密走 secrets 机制注入。分层策略上，PR 跑 2 分钟内的冒烟门禁，夜间定时跑对齐 SLO 的负载测试，浸泡与压力测试按周或手动执行。失败处理的关键是分得清 99（性能不达标，真信号）与 107/103（脚本或环境故障，假信号），前者阻断、后者重试并修复。
