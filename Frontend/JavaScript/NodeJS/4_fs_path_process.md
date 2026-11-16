服务端代码每天都在和文件系统、路径、进程环境打交道：读配置、写日志、扫描上传目录、根据环境变量切换数据库。看似简单的 `fs.readFile` 里藏着编码、错误码、路径穿越和内存占用等一堆细节。本篇把 `fs`、`path`、`process` 三件套讲清楚。

# 1. fs 模块概览

## 1.1 三种 API 风格

`node:fs` 同时提供三种调用方式：

| 风格 | 示例 | 适用场景 |
|------|------|---------|
| 同步 | `fs.readFileSync(path)` | 启动脚本、一次性小文件 |
| 回调 | `fs.readFile(path, cb)` | 老代码、需要精细控制 |
| Promise | `fs.promises.readFile(path)` | 现代代码首选 |

```javascript
import fs from "node:fs";
import { readFile } from "node:fs/promises";

// 同步:会阻塞事件循环
const syncData = fs.readFileSync("./config.json", "utf8");

// Promise:推荐
const asyncData = await readFile("./config.json", "utf8");
```

★ **关键认知**：同步 API 不是"错误设计"，它在**启动阶段**读配置、CLI 工具里非常合适。真正的问题是**在请求处理路径里使用它**——那会让单个慢磁盘拖住所有并发请求。

## 1.2 fs/promises 的基本用法

```javascript
import { readFile, writeFile, appendFile, mkdir, rm } from "node:fs/promises";
import { join } from "node:path";

const dir = join(process.cwd(), "data");
await mkdir(dir, { recursive: true }); // 已存在也不报错

const file = join(dir, "notes.txt");
await writeFile(file, "第一行\n", "utf8");
await appendFile(file, "第二行\n", "utf8");

const content = await readFile(file, "utf8");
console.log(content);
```

逐段说明：

- `mkdir(..., { recursive: true })` 会创建整条路径，目录已存在时不抛错；
- `writeFile` 默认**覆盖**整个文件，追加要用 `appendFile`；
- 传 `"utf8"` 得到字符串，不传则得到 `Buffer`；
- 路径用 `path.join` 拼接，不要手写 `"/"`，否则 Windows 下容易出问题。

## 1.3 读取目录与文件信息

```javascript
import { readdir, stat } from "node:fs/promises";
import { join } from "node:path";

const entries = await readdir("./data", { withFileTypes: true });

for (const entry of entries) {
  if (entry.isDirectory()) {
    console.log("[目录]", entry.name);
    continue;
  }

  const info = await stat(join("./data", entry.name));
  console.log("[文件]", entry.name, info.size, "字节");
}
```

`stat` 返回 `size`、`mtime`、`isDirectory()` 等信息。需要判断"是否存在"时：

```javascript
import { access, constants } from "node:fs/promises";

try {
  await access("./config.json", constants.R_OK);
  console.log("文件可读");
} catch {
  console.log("文件不存在或不可读");
}
```

⚠ **关键认知**：先 `stat` 再 `readFile` 存在**竞态窗口**——两次调用之间文件可能被删除或替换。安全性要求高的场景应直接操作并在 `catch` 里处理 `ENOENT`，而不是先检查再执行。

# 2. path 模块

## 2.1 路径拼接与解析

```javascript
import path from "node:path";

console.log(path.join("src", "utils", "index.js")); // src/utils/index.js
console.log(path.resolve("src", "index.js"));        // 绝对路径
console.log(path.basename("/tmp/a/b.txt"));          // b.txt
console.log(path.extname("/tmp/a/b.txt"));           // .txt
console.log(path.dirname("/tmp/a/b.txt"));           // /tmp/a
console.log(path.parse("/tmp/a/b.txt"));
```

`path.parse` 的结果：

```text
{
  root: "/",
  dir: "/tmp/a",
  base: "b.txt",
  ext: ".txt",
  name: "b"
}
```

| 方法 | 作用 | 注意 |
|------|------|------|
| `join` | 拼接并规范化 | 相对路径，跟随当前工作目录 |
| `resolve` | 解析成绝对路径 | 从右往左拼接，直到出现绝对路径 |
| `normalize` | 规范化 `..` 与重复分隔符 | 不检查文件是否存在 |
| `basename` | 取文件名 | 可传扩展名以去掉后缀 |
| `extname` | 取扩展名 | 只返回最后一个点之后 |

★ **关键认知**：`join` 与 `resolve` 的区别在"是否依赖当前工作目录"。`join("a", "b")` 就是 `a/b`；`resolve("a", "b")` 会拼成 `process.cwd()/a/b`。⚠ 服务端不要用 `process.cwd()` 之外的方式猜测项目根目录，因为进程的启动目录可以被任意改变。

## 2.2 __dirname 与 import.meta.url

CommonJS 里可以直接用 `__dirname`、`__filename`：

```javascript
const path = require("node:path");
console.log(__dirname); // 当前文件所在目录
console.log(path.join(__dirname, "data", "a.txt"));
```

ESM 没有这两个变量，需要从 `import.meta.url` 推导：

```javascript
import { fileURLToPath } from "node:url";
import { dirname, join } from "node:path";

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);

console.log(join(__dirname, "data", "a.txt"));
```

★ **关键认知**：`__dirname` 是**文件所在目录**，`process.cwd()` 是**进程启动目录**。两者在开发时经常相同，但部署后(如用 systemd 启动、从别的目录执行)可能完全不同。⚠ 读取"跟代码放在一起的资源"时用 `__dirname`，读取"用户当前所在位置"时才用 `cwd`。

# 3. process 与环境变量

## 3.1 常用属性

```javascript
console.log(process.pid);        // 进程 ID
console.log(process.ppid);       // 父进程 ID
console.log(process.platform);   // win32 / linux / darwin
console.log(process.arch);       // x64 / arm64
console.log(process.version);    // v22.x.x
console.log(process.cwd());      // 当前工作目录
console.log(process.uptime());   // 已运行秒数
console.log(process.memoryUsage()); // 内存使用
```

进程事件：

```javascript
process.on("exit", (code) => {
  console.log("进程即将退出,退出码", code); // 只能是同步操作
});

process.on("uncaughtException", (err) => {
  console.error("未捕获异常", err);
  process.exit(1); // 记录后退出,不要继续带病运行
});

process.on("unhandledRejection", (reason) => {
  console.error("未处理的 Promise 拒绝", reason);
});
```

⚠ **关键认知**：`uncaughtException` 不是"恢复"手段。异常发生后进程状态可能已损坏，正确做法是记录日志、尽快退出，由进程管理器重启。唯一可靠的兜底是**在退出前完成同步清理**。

## 3.2 环境变量

```javascript
const port = Number(process.env.PORT ?? 3000);
const dbUrl = process.env.DATABASE_URL;

if (!dbUrl) {
  throw new Error("缺少 DATABASE_URL 环境变量");
}
```

Node 20.6+ 支持 `--env-file`：

```bash
node --env-file=.env src/index.js
```

`.env` 文件：

```text
PORT=3000
DATABASE_URL=postgres://localhost:5432/app
NODE_ENV=development
```

★ **关键认知**：环境变量**永远是字符串**。`process.env.PORT` 拿到的是 `"3000"` 而不是 `3000`，要显式转换。⚠ 不要给环境变量设默认值后再偷偷用于生产，比如 `JWT_SECRET ?? "dev-secret"`——配置缺失时应该在启动阶段直接失败，而不是用弱密钥跑起来。

## 3.3 退出码与信号

```javascript
process.exitCode = 1; // 更温和:让事件循环自然结束
process.exit(0);      // 立即退出,可能丢失未完成的异步操作
```

| 退出码 | 含义 |
|--------|------|
| 0 | 正常结束 |
| 1 | 未捕获异常或一般错误 |
| 2 | 命令行参数错误 |
| 130 | 被 Ctrl+C 中断 |

优雅关闭需要监听信号：

```javascript
process.on("SIGTERM", async () => {
  console.log("收到 SIGTERM,开始关闭");
  await server.close();
  process.exit(0);
});
```

# 4. 文件流

## 4.1 为什么大文件要用流

`readFile` 会把整个文件读进内存。读取 2 GB 日志时，进程内存会瞬间上涨 2 GB，甚至直接 OOM。流把文件切成小块，边读边处理：

```text
readFile:  [====================]  一次性读入内存
createReadStream: [==][==][==][==]  逐块处理
```

★ **关键认知**：判断标准不是"文件多大"，而是**是否需要一次性拿到全部内容**。需要整体解析的小 JSON 用 `readFile` 没问题；日志分析、文件上传、大 CSV 导入必须用流。

## 4.2 可读流与可写流

```javascript
import { createReadStream, createWriteStream } from "node:fs";

const reader = createReadStream("./big.log", { encoding: "utf8" });
const writer = createWriteStream("./copy.log");

reader.on("data", (chunk) => {
  writer.write(chunk);
});

reader.on("end", () => {
  writer.end();
  console.log("复制完成");
});

reader.on("error", (err) => console.error("读取失败", err));
writer.on("error", (err) => console.error("写入失败", err));
```

这段代码能工作，但**没有处理背压**：如果读得比写得快，`writer` 的内部缓冲区会不断膨胀。正确做法是检查 `write()` 的返回值：

```javascript
reader.on("data", (chunk) => {
  if (!writer.write(chunk)) {
    reader.pause(); // 写不动了,先暂停读取
    writer.once("drain", () => reader.resume());
  }
});
```

## 4.3 pipeline

手写背压逻辑容易出错，`stream/promises` 的 `pipeline` 会自动处理错误与背压：

```javascript
import { pipeline } from "node:stream/promises";
import { createReadStream, createWriteStream } from "node:fs";
import { createGzip } from "node:zlib";

await pipeline(
  createReadStream("./big.log"),
  createGzip(),
  createWriteStream("./big.log.gz")
);

console.log("压缩完成");
```

`pipeline` 会在任一环节出错时销毁所有流，并返回一个可 `await` 的 Promise。第 6 篇会展开讲流与背压。

# 5. 错误处理

## 5.1 错误优先回调

老式回调遵循"错误优先"约定：

```javascript
import fs from "node:fs";

fs.readFile("./a.txt", "utf8", (err, data) => {
  if (err) {
    console.error("读取失败", err.code);
    return;
  }
  console.log(data);
});
```

Promise 版本用 `try/catch`：

```javascript
try {
  const data = await readFile("./a.txt", "utf8");
  console.log(data);
} catch (err) {
  if (err.code === "ENOENT") {
    console.log("文件不存在");
  } else {
    throw err;
  }
}
```

## 5.2 常见 err.code

| 错误码 | 含义 | 典型处理 |
|--------|------|---------|
| `ENOENT` | 文件或目录不存在 | 返回默认值或 404 |
| `EACCES` | 权限不足 | 检查权限，返回 403 |
| `EEXIST` | 文件已存在 | 改用追加或覆盖 |
| `EISDIR` | 期望文件却拿到目录 | 修正路径 |
| `ENOTDIR` | 路径中间某段不是目录 | 修正路径 |
| `ENOSPC` | 磁盘空间不足 | 告警并停止写入 |
| `EMFILE` | 打开的文件描述符过多 | 检查是否泄漏 |

★ **关键认知**：不要用错误信息字符串判断错误类型，`err.code` 才是稳定标识。⚠ 也不要吞掉所有错误——只处理你真正能处理的那几种，其余继续抛出，否则线上问题会被静默掩盖。

# 6. 常见错误与练习

## 6.1 常见错误

**错误一：用字符串拼接路径**

```javascript
const file = __dirname + "/data/" + name; // Windows 下分隔符可能不对
const safe = path.join(__dirname, "data", name); // 正确
```

**错误二：把用户输入直接拼进路径**

```javascript
const file = path.join(uploadDir, req.query.name); // 危险:可被 ../ 穿越
```

必须校验解析后的路径仍在目标目录内：

```javascript
const target = path.resolve(uploadDir, req.query.name);
if (!target.startsWith(path.resolve(uploadDir) + path.sep)) {
  throw new Error("非法路径");
}
```

**错误三：忘记 `writeFile` 会覆盖**

向日志追加内容却用了 `writeFile`，每次运行都把历史清空。应使用 `appendFile` 或可写流。

**错误四：读大文件用 `readFile`**

内存占用随文件线性增长，并发几个请求就会 OOM。改用流式处理。

## 6.2 练习

1. 写一个函数 `ensureDir(dir)`，在目录不存在时递归创建，已存在时静默返回。

2. 用 `readdir` 统计某个目录下所有 `.md` 文件的总行数，要求使用流式读取，避免一次性载入大文件。

3. 写一个脚本，从 `process.env.LOG_DIR` 读取日志目录，目录不存在时输出明确错误并以非零退出码结束。

# 7. 小结

| 概念 | 要点 |
|------|------|
| fs 三种风格 | 同步、回调、Promise，请求路径优先 Promise |
| path.join / resolve | join 不依赖 cwd，resolve 会拼成绝对路径 |
| __dirname | 文件所在目录，ESM 用 `import.meta.url` 推导 |
| 环境变量 | 永远是字符串，缺失时启动即失败 |
| 文件流 | 大文件边读边处理，避免整体载入内存 |
| pipeline | 自动处理背压与错误，优于手写 pipe |
| 错误处理 | 用 `err.code` 判断，不吞掉未知错误 |

下一篇进入 HTTP 与 Express，讲清楚 `http` 模块的请求响应模型、Express 的路由与中间件，并落地一个 REST API 示例。
