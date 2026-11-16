写一个能返回 JSON 的接口只需求几行代码，但要让它具备合理的路由、中间件顺序、错误处理和 REST 语义，就需要理解 HTTP 的请求响应模型。本篇先用手写的 `http` 服务器打底，再进入 Express，最后落地一个可运行的 REST API。

# 1. 用 http 模块起一个服务器

## 1.1 最小服务器

```javascript
import http from "node:http";

const server = http.createServer((req, res) => {
  res.statusCode = 200;
  res.setHeader("Content-Type", "application/json; charset=utf-8");
  res.end(JSON.stringify({ ok: true, path: req.url }));
});

server.listen(3000, () => {
  console.log("http://localhost:3000");
});
```

`createServer` 接收一个回调，每个请求到来时被调用一次。`req` 是可读流，`res` 是可写流。

★ **关键认知**：HTTP 服务器的本质是**一个长期运行的事件循环 + 每个连接上的两个流**。你不需要为每个请求创建线程，只需要在回调里尽快读完输入、写完输出并返回，剩下的并发由事件循环负责。

## 1.2 请求对象

```javascript
const server = http.createServer((req, res) => {
  console.log(req.method);  // GET / POST
  console.log(req.url);     // /users?page=2
  console.log(req.headers); // 请求头对象

  const url = new URL(req.url, `http://${req.headers.host}`);
  console.log(url.pathname);      // /users
  console.log(url.searchParams.get("page")); // 2
});
```

请求体不会自动解析，需要手动读取流：

```javascript
async function readBody(req) {
  const chunks = [];
  for await (const chunk of req) {
    chunks.push(chunk);
  }
  return Buffer.concat(chunks).toString("utf8");
}
```

⚠ **关键认知**：`req.url` 只包含路径和查询串，不含协议与域名。拼 `new URL` 时必须提供 base，否则会抛 `Invalid URL`。

## 1.3 响应对象

```javascript
res.writeHead(201, {
  "Content-Type": "application/json; charset=utf-8",
  "X-Request-Id": "abc-123",
});
res.end(JSON.stringify({ id: 1 }));
```

注意三点：

- `writeHead` 之后再设置 header 无效；
- `res.end()` 必须调用，否则连接会一直挂着；
- 返回中文时显式声明 `charset=utf-8`，避免乱码。

## 1.4 手动处理路由的痛点

```javascript
if (req.method === "GET" && req.url === "/users") {
  // ...
} else if (req.method === "POST" && req.url === "/users") {
  // ...
} else if (req.url.startsWith("/users/")) {
  // ...
} else {
  res.statusCode = 404;
  res.end("Not Found");
}
```

路由、方法匹配、错误处理、请求体解析全要手写，代码很快失控。这就是 Express 存在的意义。

# 2. Express 基础

## 2.1 安装与最小应用

```bash
npm init -y
npm install express
```

```javascript
import express from "express";

const app = express();

app.get("/", (req, res) => {
  res.json({ message: "hello" });
});

app.listen(3000, () => {
  console.log("http://localhost:3000");
});
```

## 2.2 路由

```javascript
app.get("/users", (req, res) => res.json([]));
app.post("/users", (req, res) => res.status(201).json({ id: 1 }));
app.put("/users/:id", (req, res) => res.json({ id: req.params.id }));
app.delete("/users/:id", (req, res) => res.status(204).end());
```

路由参数与查询参数：

```javascript
app.get("/users/:id/posts/:postId", (req, res) => {
  console.log(req.params.id, req.params.postId); // 路径参数
  console.log(req.query.page);                   // 查询参数,字符串
});
```

**路由注册顺序很重要**：

```javascript
app.get("/users/:id", (req, res) => res.send("详情"));
app.get("/users/me", (req, res) => res.send("当前用户")); // 永远不会命中
```

因为 `/users/:id` 先注册，`/users/me` 会被当成 `id = "me"`。把更具体的路由放前面。

★ **关键认知**：Express 的路由是**按注册顺序匹配**的，不是按"精确度"自动排序。`/users/me` 这类静态路径必须写在 `/users/:id` 之前，这是新手最常见的 404 来源之一。

## 2.3 请求与响应 API

| 需求 | 方法 |
|------|------|
| 读取 JSON 请求体 | `express.json()` 中间件 + `req.body` |
| 读取路径参数 | `req.params` |
| 读取查询参数 | `req.query` |
| 读取请求头 | `req.get("Authorization")` |
| 返回 JSON | `res.json(obj)` |
| 设置状态码 | `res.status(404)` |
| 重定向 | `res.redirect("/login")` |
| 发送文件 | `res.sendFile(absolutePath)` |
| 结束无内容响应 | `res.status(204).end()` |

```javascript
app.use(express.json({ limit: "1mb" }));

app.post("/echo", (req, res) => {
  console.log(req.body); // 解析后的对象
  res.status(201).json(req.body);
});
```

⚠ **关键认知**：`express.json()` 只解析 `Content-Type: application/json` 的请求。表单要 `express.urlencoded({ extended: true })`，文件上传要 `multer` 之类的中间件。没有匹配的解析器时，`req.body` 是 `undefined`。

# 3. 中间件

## 3.1 中间件的执行模型

中间件是形如 `(req, res, next) => {}` 的函数。它可以选择：

- 结束响应：`res.send(...)`；
- 交给下一个：`next()`；
- 抛出错误：`next(err)`。

```javascript
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next();
});
```

执行顺序：

```text
请求
  │
  ▼
┌──────────────────┐
│ 日志中间件        │
└────────┬─────────┘
         ▼
┌──────────────────┐
│ express.json()   │
└────────┬─────────┘
         ▼
┌──────────────────┐
│ 鉴权中间件        │
└────────┬─────────┘
         ▼
┌──────────────────┐
│ 路由处理函数      │
└────────┬─────────┘
         ▼
┌──────────────────┐
│ 错误处理中间件    │
└──────────────────┘
```

★ **关键认知**：中间件按注册顺序形成一条链，`next()` 是"把控制权交给下一个"。如果在中间件里既没有 `next()` 也没有结束响应，请求就会**挂起**，客户端一直等到超时。⚠ 每个分支都必须明确：要么响应，要么 `next`，要么 `next(err)`。

## 3.2 常见中间件

| 中间件 | 作用 |
|--------|------|
| `express.json()` | 解析 JSON 请求体 |
| `express.urlencoded()` | 解析表单请求体 |
| `express.static(dir)` | 托管静态文件 |
| `cors` | 处理跨域 |
| `helmet` | 设置安全相关响应头 |
| `morgan` / `pino-http` | 请求日志 |
| `express-rate-limit` | 限流 |

```javascript
import express from "express";
import cors from "cors";
import helmet from "helmet";
import morgan from "morgan";

const app = express();

app.use(helmet());
app.use(cors());
app.use(morgan("combined"));
app.use(express.json({ limit: "1mb" }));
app.use(express.static("public"));
```

★ **关键认知**：中间件的注册顺序就是**请求的处理顺序**。安全头、日志、跨域、请求体解析都应在路由之前，404 兜底在路由之后，错误处理在最后。⚠ 顺序错了往往不报错，只是行为诡异，例如 `req.body` 一直是 `undefined`。

## 3.3 错误处理中间件

错误处理中间件有**四个参数**，必须显式写全：

```javascript
app.use((err, req, res, next) => {
  console.error(err);

  if (err.type === "entity.too.large") {
    return res.status(413).json({ error: "请求体过大" });
  }

  res.status(err.status ?? 500).json({
    error: err.expose ? err.message : "服务器内部错误",
  });
});
```

它必须注册在所有路由之后。

⚠ **关键认知**：Express 4 不会自动捕获 `async` 函数里 `reject` 的 Promise。下面写法不会进入错误中间件：

```javascript
app.get("/bad", async (req, res) => {
  throw new Error("boom"); // Express 4 捕获不到
});
```

解决方式有三种：

```javascript
// 方式一:自己 try/catch
app.get("/ok", async (req, res, next) => {
  try {
    throw new Error("boom");
  } catch (err) {
    next(err);
  }
});

// 方式二:包装器
const asyncHandler = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);

app.get("/ok2", asyncHandler(async (req, res) => {
  throw new Error("boom");
}));
```

方式三是升级到 Express 5，它原生支持 async 处理器的 rejection。

# 4. 一个 REST API 示例

## 4.1 目录结构

```text
src/
├── app.js          # 创建 app、挂中间件与路由
├── server.js       # 启动监听
├── routes/
│   └── users.js    # 用户路由
└── errors.js       # 自定义错误
```

## 4.2 应用入口

```javascript
// src/app.js
import express from "express";
import usersRouter from "./routes/users.js";

const app = express();

app.use(express.json({ limit: "1mb" }));

app.get("/health", (req, res) => {
  res.json({ status: "ok" });
});

app.use("/users", usersRouter);

app.use((req, res) => {
  res.status(404).json({ error: "接口不存在" });
});

app.use((err, req, res, next) => {
  console.error(err);
  res.status(err.status ?? 500).json({
    error: err.expose ? err.message : "服务器内部错误",
  });
});

export default app;
```

```javascript
// src/server.js
import app from "./app.js";

const port = Number(process.env.PORT ?? 3000);

app.listen(port, () => {
  console.log(`服务已启动: http://localhost:${port}`);
});
```

## 4.3 路由实现

```javascript
// src/routes/users.js
import { Router } from "express";

const router = Router();
const users = new Map();

router.get("/", (req, res) => {
  res.json([...users.values()]);
});

router.get("/:id", (req, res) => {
  const user = users.get(req.params.id);
  if (!user) {
    return res.status(404).json({ error: "用户不存在" });
  }
  res.json(user);
});

router.post("/", (req, res) => {
  const { name } = req.body ?? {};

  if (typeof name !== "string" || name.trim() === "") {
    return res.status(400).json({ error: "name 必填" });
  }

  const id = String(users.size + 1);
  const user = { id, name: name.trim() };
  users.set(id, user);

  res.status(201)
    .location(`/users/${id}`)
    .json(user);
});

router.put("/:id", (req, res) => {
  const user = users.get(req.params.id);
  if (!user) {
    return res.status(404).json({ error: "用户不存在" });
  }

  const { name } = req.body ?? {};
  if (typeof name !== "string" || name.trim() === "") {
    return res.status(400).json({ error: "name 必填" });
  }

  user.name = name.trim();
  res.json(user);
});

router.delete("/:id", (req, res) => {
  if (!users.delete(req.params.id)) {
    return res.status(404).json({ error: "用户不存在" });
  }
  res.status(204).end();
});

export default router;
```

## 4.4 REST 语义对照

| 操作 | 方法 | 路径 | 成功状态码 |
|------|------|------|-----------|
| 列表 | GET | `/users` | 200 |
| 详情 | GET | `/users/:id` | 200 |
| 创建 | POST | `/users` | 201 |
| 全量更新 | PUT | `/users/:id` | 200 |
| 部分更新 | PATCH | `/users/:id` | 200 |
| 删除 | DELETE | `/users/:id` | 204 |

★ **关键认知**：REST 的核心不是"用 JSON"，而是**用 HTTP 方法表达意图、用状态码表达结果**。创建成功返回 201 并带 `Location`，删除成功返回 204 不带响应体，失败用 4xx/5xx，这样客户端和网关才能正确判断。

⚠ **关键认知**：示例用内存 `Map` 只为演示，**重启即清空，也无法多进程共享**。生产环境要接数据库，并且注意第 7 篇讲的 cluster 多进程下内存状态不共享。

# 5. 常见错误与练习

## 5.1 常见错误

**错误一：忘记 `return`，继续写响应**

```javascript
app.get("/users/:id", (req, res) => {
  const user = users.get(req.params.id);
  if (!user) {
    res.status(404).json({ error: "不存在" });
    // 忘记 return,下面继续执行
  }
  res.json(user); // 报错:headers already sent
});
```

**错误二：中间件顺序错误**

```javascript
app.use("/users", usersRouter);
app.use(express.json()); // 太晚了,路由已执行
```

解析请求体的中间件必须在路由之前注册。

**错误三：错误处理中间件少写一个参数**

```javascript
app.use((err, req, res) => { /* 不会被识别为错误中间件 */ });
```

必须写成 `(err, req, res, next)` 四个参数。

**错误四：把敏感信息返回给客户端**

```javascript
res.status(500).json({ error: err.stack }); // 泄漏内部路径与实现细节
```

对外只返回通用信息，详细堆栈写日志。

## 5.2 练习

1. 给示例 API 增加 `PATCH /users/:id`，只更新请求体中出现的字段。

2. 写一个请求 ID 中间件：为每个请求生成 `X-Request-Id`，同时写入 `req.id` 和响应头。

3. 给 `POST /users` 增加长度校验：`name` 长度必须在 1 到 20 之间，超限返回 422。

# 6. 小结

| 概念 | 要点 |
|------|------|
| http 模块 | `req` 可读流，`res` 可写流，需手动解析与路由 |
| Express 路由 | 按注册顺序匹配，静态路径放前面 |
| 中间件 | `(req, res, next)`，必须明确响应或 next |
| 错误中间件 | 四参数，注册在所有路由之后 |
| async 错误 | Express 4 需 try/catch 或包装器 |
| REST | 用方法和状态码表达语义 |

下一篇进入异步与流，讲清楚回调、Promise、async/await 的取舍，并发控制，流与背压，以及常见的内存泄漏。
