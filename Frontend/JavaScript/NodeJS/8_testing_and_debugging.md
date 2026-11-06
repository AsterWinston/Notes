代码能跑不等于正确。上线后才发现的问题，往往在测试里就能拦住；已经出问题的服务，则要靠日志、断点和性能分析定位。本篇把测试、调试、日志、CPU 与内存诊断串起来，给出一套可落地的排查流程。

# 1. 测试基础

## 1.1 测试金字塔

```text
        ┌──────────┐
        │  E2E 少   │  真实浏览器/网络,慢且脆
        ├──────────┤
        │ 集成测试  │  路由 + 数据库,验证协作
        ├──────────┤
        │ 单元测试多│  纯函数、模块,快而稳定
        └──────────┘
```

★ **关键认知**：单元测试验证"逻辑是否正确"，集成测试验证"组件能否协作"，E2E 验证"用户流程是否走得通"。三者不是替代关系，基层越多，反馈越快、定位越准。

## 1.2 node:test 入门

Node 18+ 内置 `node:test`，无需安装依赖：

```javascript
// math.js
export function add(a, b) {
  return a + b;
}
```

```javascript
// math.test.js
import test from "node:test";
import assert from "node:assert/strict";
import { add } from "./math.js";

test("add 返回两数之和", () => {
  assert.equal(add(1, 2), 3);
});

test("add 处理负数", () => {
  assert.equal(add(-1, -2), -3);
});
```

运行：

```bash
node --test
node --test math.test.js
node --test --experimental-test-coverage
```

## 1.3 常用断言

```javascript
import assert from "node:assert/strict";

assert.equal(actual, expected);          // 宽松相等(==)
assert.strictEqual(actual, expected);    // 严格相等(===)
assert.deepEqual(obj1, obj2);            // 深度比较
assert.ok(value);                        // 真值判断
assert.throws(() => fn(), /错误信息/);    // 断言抛错
assert.rejects(promise, /错误信息/);      // 断言 Promise 拒绝
```

异步测试：

```javascript
test("异步读取配置", async () => {
  const config = await loadConfig();
  assert.equal(config.port, 3000);
});
```

## 1.4 钩子与组织

```javascript
import test, { describe, beforeEach, after } from "node:test";

describe("用户服务", () => {
  let db;

  beforeEach(async () => {
    db = await createTestDb(); // 每个用例前重建,保证隔离
  });

  after(async () => {
    await db.close();
  });

  test("创建用户", async () => {
    const user = await db.create({ name: "Ada" });
    assert.equal(user.name, "Ada");
  });
});
```

## 1.5 测试替身

外部依赖(数据库、第三方 API)在单元测试里应替换掉：

```javascript
import test from "node:test";
import assert from "node:assert/strict";

test("天气服务返回温度", async () => {
  const fakeFetch = async () => ({
    ok: true,
    json: async () => ({ temperature: 25 }),
  });

  const weather = await getWeather("Beijing", fakeFetch); // 注入依赖
  assert.equal(weather.temperature, 25);
});
```

★ **关键认知**：优先通过**参数注入**替换依赖，而不是全局 mock。注入让依赖关系显式，测试不需要关心模块加载顺序；全局 mock 容易互相污染，尤其在并发执行的测试里。

## 1.6 Jest 对照

| 能力 | node：test | Jest |
|------|-----------|------|
| 运行器 | 内置 | 需安装 |
| 断言 | `node:assert` | `expect` |
| 监听模式 | `--watch` | `--watch` |
| 覆盖率 | `--experimental-test-coverage` | `--coverage` |
| 快照 | 无内置 | 内置 |
| Mock | `mock` API | `jest.mock` |
| 配置复杂度 | 低 | 中 |

Jest 写法：

```javascript
describe("add", () => {
  test("返回两数之和", () => {
    expect(add(1, 2)).toBe(3);
  });
});
```

选型建议：小项目、纯 Node 服务优先 `node:test`，零依赖、启动快；需要快照、复杂 mock、成熟生态时选 Jest 或 Vitest。

# 2. 调试

## 2.1 console 的进阶用法

```javascript
console.log("普通日志");
console.error("错误日志,输出到 stderr");
console.table([{ id: 1, name: "Ada" }, { id: 2, name: "Bob" }]);
console.dir(obj, { depth: null, colors: true }); // 完整展开对象
console.time("query");
await db.query();
console.timeEnd("query"); // 输出耗时
console.trace("调用栈");
```

⚠ **关键认知**：`console.log` 在生产环境是**同步写 stdout**，高频调用会拖慢服务。生产应使用结构化日志库，并按级别控制输出。

## 2.2 内置调试器

```bash
node --inspect-brk src/index.js
```

打开 Chrome，访问 `chrome://inspect`，连接后可以：

- 在源码里打断点；
- 单步执行、查看调用栈；
- 查看作用域变量；
- 在控制台里执行表达式。

也可以只用命令行调试器：

```bash
node inspect src/index.js
```

常用命令：`c` 继续、`n` 下一步、`s` 进入函数、`o` 跳出、`bt` 打印调用栈。

## 2.3 VS Code 调试配置

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "启动服务",
      "program": "${workspaceFolder}/src/index.js",
      "envFile": "${workspaceFolder}/.env",
      "console": "integratedTerminal",
      "skipFiles": ["<node_internals>/**"]
    },
    {
      "type": "node",
      "request": "attach",
      "name": "附加到进程",
      "port": 9229
    }
  ]
}
```

★ **关键认知**：`--inspect-brk` 会在第一行暂停，适合调试启动阶段的问题；`--inspect` 不暂停，适合附加到已运行进程。⚠ 生产环境不要开放调试端口，`--inspect=0.0.0.0` 等于把代码执行权限暴露给网络。

# 3. 日志

## 3.1 日志级别

| 级别 | 用途 | 生产默认 |
|------|------|---------|
| `fatal` | 进程无法继续 | 输出 |
| `error` | 请求失败、异常 | 输出 |
| `warn` | 可恢复的异常情况 | 输出 |
| `info` | 关键业务流程 | 输出 |
| `debug` | 调试细节 | 关闭 |
| `trace` | 极细粒度 | 关闭 |

```javascript
import pino from "pino";

const logger = pino({ level: process.env.LOG_LEVEL ?? "info" });

logger.info({ userId: 1 }, "用户登录");
logger.error({ err }, "查询失败");
```

## 3.2 结构化日志

不要拼接字符串，输出 JSON：

```javascript
// 差:难以检索和聚合
console.log(`用户 ${userId} 在 ${time} 下单 ${orderId}`);

// 好:字段独立,可被日志系统索引
logger.info({ userId, orderId, time }, "用户下单");
```

★ **关键认知**：结构化日志的价值在于**可查询**。`userId` 是独立字段，才能按用户聚合、按错误码统计；拼进一句话里，就只能全文搜索。

## 3.3 请求追踪

```javascript
import { randomUUID } from "node:crypto";

app.use((req, res, next) => {
  req.id = req.get("X-Request-Id") ?? randomUUID();
  res.setHeader("X-Request-Id", req.id);

  const start = performance.now();
  res.on("finish", () => {
    logger.info({
      requestId: req.id,
      method: req.method,
      path: req.path,
      status: res.statusCode,
      durationMs: Math.round(performance.now() - start),
    }, "请求完成");
  });

  next();
});
```

有了 `requestId`，一次请求在多个服务、多条日志之间就能串起来。

⚠ **关键认知**：不要在日志里记录密码、token、完整身份证号、银行卡号。需要排查时做脱敏，例如只保留后四位。日志系统往往权限更宽、留存更久，泄漏面比数据库更大。

# 4. 性能与内存诊断

## 4.1 CPU profiling

```bash
node --cpu-prof --cpu-prof-dir=./profiles src/index.js
```

生成 `.cpuprofile` 后，在 Chrome DevTools 的 Performance 面板加载，看火焰图：

```text
[================ 请求处理 ================]
  [==== JSON.parse ====]
      [== 校验 ==]
  [============ 数据库查询 ============]
```

火焰图里越宽的横条耗时越长，层层堆叠表示调用关系。优先优化"宽且深"的路径。

## 4.2 内存快照

```bash
node --heapsnapshot-signal=SIGUSR2 src/index.js
```

向进程发送 `SIGUSR2` 会生成堆快照：

```bash
kill -USR2 <pid>
```

在 Chrome DevTools 的 Memory 面板加载 `.heapsnapshot`，对比两个时间点的快照：

- **Shallow Size**：对象自身占用；
- **Retained Size**：该对象被回收后能释放的总内存；
- **Retainers**：谁在持有这个对象。

★ **关键认知**：排查内存泄漏的关键不是"哪个对象最大"，而是**哪个对象在两次快照之间持续增加，以及谁在持有它**。找到 Retainer 链，通常就能定位到未清理的缓存、监听器或定时器。

## 4.3 异步资源追踪

```bash
node --trace-warnings src/index.js
```

```javascript
import async_hooks from "node:async_hooks";
```

`async_hooks` 可以追踪异步资源的生命周期，适合排查"回调为何一直不执行""哪个异步操作泄漏了句柄"。它开销较大，只在诊断时启用。

## 4.4 常见问题定位表

| 现象 | 可能原因 | 排查手段 |
|------|---------|---------|
| 内存持续上涨 | 缓存无上限、监听器未移除 | 堆快照对比 |
| CPU 长期 100% | 死循环、正则回溯、重计算 | CPU profile |
| 请求偶发超时 | 事件循环被阻塞、下游慢 | 事件循环延迟、链路追踪 |
| 进程突然退出 | 未捕获异常、OOM | 退出码、`dmesg`、`--trace-uncaught` |
| 句柄泄漏 | 定时器/连接未关闭 | `process._getActiveHandles()` |

```javascript
setInterval(() => {
  console.log("活跃句柄数", process._getActiveHandles().length);
}, 10000);
```

⚠ **关键认知**：`process._getActiveHandles()` 是内部 API，字段与行为可能变化，只适合临时排查，不要写进生产监控的核心逻辑。

# 5. 练习

1. 为一个"计算订单总价"的纯函数写 3 个 `node:test` 用例，覆盖正常、空数组、含非法数量三种情况。

2. 给一个 Express 应用加上请求日志中间件，记录 `requestId`、路径、状态码与耗时，并要求日志为 JSON 格式。

3. 写一个会持续向全局数组 push 数据的脚本，用两次堆快照对比，找出增长的对象与持有链。

# 6. 小结

| 概念 | 要点 |
|------|------|
| 测试金字塔 | 单元多、集成中、E2E 少 |
| node：test | 内置运行器，`node --test` 即可 |
| 依赖注入 | 优于全局 mock，隔离更干净 |
| 断点调试 | `--inspect-brk` 调启动，`--inspect` 附加 |
| 结构化日志 | JSON 字段可检索，带 requestId 串联 |
| CPU profile | 火焰图看宽而深的调用路径 |
| 堆快照 | 找持续增长的对象与 Retainer 链 |

下一篇进入安全，讲清楚输入校验、SQL 与命令注入、路径穿越、SSRF、依赖供应链、密钥管理与限流。
