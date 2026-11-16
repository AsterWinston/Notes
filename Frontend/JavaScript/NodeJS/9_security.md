Node 服务直接面向公网，任何一处"把用户输入当代码或路径用"都可能变成漏洞。安全不是加一个中间件就完事，而是贯穿输入、执行、文件、网络、依赖、密钥的整套习惯。本篇按攻击面逐项拆解，每项都给出可落地的防护写法。

# 1. 输入校验

## 1.1 永远不要信任客户端

客户端的校验只是"提升体验"，**真正的边界在服务端**。攻击者可以绕过页面，直接构造请求。

```javascript
// 错误:直接使用请求体
app.post("/users", async (req, res) => {
  await db.insert(req.body); // 可能包含多余字段、错误类型
});
```

至少要做三件事：

- **类型校验**：`age` 必须是数字，不是字符串 `"18"`；
- **范围校验**：`age` 在 0 到 150 之间；
- **白名单字段**：只接收允许的字段，其余丢弃。

## 1.2 用 schema 校验

```javascript
import { z } from "zod";

const createUserSchema = z.object({
  name: z.string().trim().min(1).max(20),
  email: z.string().email(),
  age: z.number().int().min(0).max(150).optional(),
});

app.post("/users", (req, res) => {
  const parsed = createUserSchema.safeParse(req.body);

  if (!parsed.success) {
    return res.status(422).json({
      error: "参数不合法",
      details: parsed.error.issues.map((i) => i.path.join(".")),
    });
  }

  // parsed.data 只包含 schema 声明的字段
  return res.status(201).json(parsed.data);
});
```

★ **关键认知**：schema 校验同时解决"类型安全"和"字段白名单"两个问题。`z.object` 默认会**丢弃未声明的字段**，从而阻止攻击者塞入 `isAdmin: true` 这类越权参数。

## 1.3 常见校验遗漏

| 字段 | 常见遗漏 | 风险 |
|------|---------|------|
| 分页参数 | 未限制上限 | 一次拉取百万条，拖垮数据库 |
| ID | 未校验格式 | 注入、类型混淆 |
| 排序字段 | 直接拼进 SQL | SQL 注入 |
| URL | 未限制协议与域名 | SSRF |
| 文件名 | 未过滤 `../` | 路径穿越 |

# 2. 注入类风险

## 2.1 SQL 注入

```javascript
// 错误:字符串拼接
const sql = `SELECT * FROM users WHERE name = '${req.query.name}'`;
await db.query(sql);
```

输入 `' OR '1'='1` 会让条件恒真，输入 `'; DROP TABLE users; --` 可能直接删表。

正确做法是**参数化查询**：

```javascript
// 正确:占位符 + 参数
await db.query("SELECT * FROM users WHERE name = $1", [req.query.name]);
```

```javascript
// 正确:ORM 的参数绑定
const user = await prisma.user.findUnique({
  where: { email: req.body.email },
});
```

⚠ **关键认知**：参数化查询之所以安全，是因为数据库把**参数当成值**，而不是 SQL 语法的一部分。任何"把用户输入拼进 SQL 文本"的做法，无论怎么转义都不如占位符可靠。表名、列名无法参数化时，必须用**白名单映射**，不能直接拼接。

## 2.2 命令注入

```javascript
import { exec } from "node:child_process";

// 错误:拼接用户输入
exec(`ls ${req.query.dir}`, (err, stdout) => {
  res.send(stdout);
});
```

输入 `; rm -rf /` 会在 shell 里执行额外命令。

正确做法是**避免 shell**，用 `execFile` 或 `spawn` 传参数数组：

```javascript
import { execFile } from "node:child_process";

execFile("ls", [req.query.dir], (err, stdout) => {
  if (err) return res.status(400).json({ error: "执行失败" });
  res.send(stdout);
});
```

★ **关键认知**：`exec` 会启动 shell，因此 `;`、`|`、`$()` 都有特殊含义；`execFile` / `spawn` 不经过 shell，参数是独立数组，天然阻断命令拼接。⚠ 即便如此，也要限制参数内容，例如目录必须是白名单内的路径。

## 2.3 模板注入与 XSS

服务端渲染模板时，不要关闭自动转义：

```javascript
// 错误:把用户输入当 HTML
res.send(`<h1>${req.query.name}</h1>`);
```

输入 `<script>alert(1)</script>` 会直接执行。API 返回 JSON 时，主要风险转移到前端；如果前端用 `innerHTML` 渲染用户内容，同样会触发 XSS。

防护要点：

- 输出到 HTML 时转义 `<`、`>`、`&`、`"`、`'`；
- 前端优先用 `textContent` 而不是 `innerHTML`；
- 设置 `Content-Security-Policy` 限制脚本来源；
- 富文本用成熟的白名单清洗库，不要自己写正则。

# 3. 路径穿越

```javascript
// 错误:用户输入直接拼路径
const file = path.join(uploadDir, req.query.name);
res.sendFile(file);
```

输入 `../../etc/passwd` 会读出上传目录之外的文件。

正确的校验是**解析后判断前缀**：

```javascript
import path from "node:path";

const base = path.resolve(uploadDir);
const target = path.resolve(base, req.query.name);

if (target !== base && !target.startsWith(base + path.sep)) {
  return res.status(400).json({ error: "非法路径" });
}
```

★ **关键认知**：`path.join` 只做拼接与规范化，**不会阻止 `..` 逃逸**。必须把路径解析成绝对路径后，判断它是否仍在允许的根目录内。⚠ 只检查字符串里是否包含 `..` 不够，编码、符号链接、大小写都可能绕过。

文件上传还要额外注意：

- 用随机文件名，不要直接使用用户提供的文件名；
- 限制扩展名与 MIME 类型；
- 上传目录不要设置可执行权限；
- 文件内容做类型嗅探，防止伪装成图片的脚本。

# 4. SSRF

SSRF(服务端请求伪造)是让服务器替攻击者发起请求，访问内网服务。

```javascript
// 危险:用户提供 URL,服务端直接请求
app.get("/proxy", async (req, res) => {
  const response = await fetch(req.query.url);
  res.send(await response.text());
});
```

攻击者传入 `http://169.254.169.254/latest/meta-data/` 可能读到云主机元数据，传入 `http://169.254.169.254/latest/meta-data/` 可能探测内网端口。

防护措施：

| 措施 | 说明 |
|------|------|
| 协议白名单 | 只允许 `http` / `https` |
| 域名白名单 | 只允许明确的业务域名 |
| 解析后校验 IP | 拒绝私网、回环、链路本地地址 |
| 禁止重定向 | 或对重定向目标重新校验 |
| 独立出口 | 代理服务放在受限网络 |

```javascript
function isPrivateIp(ip) {
  return (
    ip.startsWith("10.") ||
    ip.startsWith("127.") ||
    ip.startsWith("192.168.") ||
    /^172\.(1[6-9]|2\d|3[01])\./.test(ip) ||
    ip.startsWith("169.254.")
  );
}
```

⚠ **关键认知**：只在字符串层面校验域名不够，攻击者可以用 DNS 重绑定让域名先解析到公网、再解析到内网。稳妥做法是**解析出 IP 后校验，并让实际连接使用同一个 IP**，或直接走受控的出口代理。

# 5. 依赖与供应链安全

## 5.1 依赖风险

- 直接依赖有漏洞；
- 间接依赖被投毒；
- `postinstall` 脚本执行恶意代码；
- 包名抢注(typosquatting)，如把 `lodash` 写成 `1odash`。

```bash
npm audit
npm audit fix
npm outdated
```

## 5.2 防护实践

- 提交并锁定 lockfile，CI 用 `npm ci`；
- 使用 `overrides` 修复间接依赖漏洞；
- 安装时评估维护者、下载量、最近更新；
- CI 中用 `--ignore-scripts` 执行不可信依赖的安装；
- 接入 Dependabot / Snyk 持续扫描；
- 私有包使用内部 registry，并配置作用域。

★ **关键认知**：依赖安全的核心是**减少不可信代码的执行面**。每多一个依赖，就多一份需要审计的代码、一个可能被接管的维护者账号。⚠ 不要为了一个小功能引入一个庞大的包，标准库和几十行代码往往足够。

# 6. 密钥与配置

## 6.1 不要把密钥写进代码

```javascript
// 错误:密钥进入 Git 历史
const client = new Client({ secret: "sk-live-123456" });
```

正确做法：

```javascript
const secret = process.env.CLIENT_SECRET;
if (!secret) {
  throw new Error("缺少 CLIENT_SECRET");
}
```

- `.env` 加入 `.gitignore`；
- 生产环境用密钥管理服务或容器注入；
- 不同环境使用不同密钥；
- 定期轮换，泄漏后立即吊销。

⚠ **关键认知**：密钥一旦提交进 Git，即使后来删除文件，历史记录里仍然存在。必须**立即吊销并重新生成**，只删文件等于没处理。

## 6.2 配置校验

```javascript
const required = ["DATABASE_URL", "JWT_SECRET", "PORT"];

for (const key of required) {
  if (!process.env[key]) {
    throw new Error(`缺少环境变量 ${key}`);
  }
}
```

启动即失败，比运行到一半才发现配置缺失安全得多。

## 6.3 安全响应头

```javascript
import helmet from "helmet";

app.use(helmet());
```

`helmet` 会设置 `Content-Security-Policy`、`X-Content-Type-Options`、`X-Frame-Options` 等头，降低 XSS、点击劫持、MIME 混淆风险。

# 7. 权限与 DoS

## 7.1 认证与授权

认证回答"你是谁"，授权回答"你能做什么"。两者都不能省：

```javascript
app.delete("/posts/:id", async (req, res) => {
  const post = await db.post.findUnique({ where: { id: req.params.id } });

  if (!post) return res.status(404).json({ error: "不存在" });

  // 关键:校验归属,不能只校验登录
  if (post.authorId !== req.user.id) {
    return res.status(403).json({ error: "无权操作" });
  }

  await db.post.delete({ where: { id: post.id } });
  res.status(204).end();
});
```

★ **关键认知**：最常见的越权漏洞不是"没登录"，而是**只校验了登录，没校验资源归属**。任何按 ID 操作资源的接口，都要问一句："这个资源属于当前用户吗？"

## 7.2 限流与资源上限

```javascript
import rateLimit from "express-rate-limit";

app.use("/api", rateLimit({
  windowMs: 60 * 1000,
  limit: 100,           // 每个 IP 每分钟 100 次
  standardHeaders: true,
}));
```

还要限制：

- 请求体大小：`express.json({ limit: "1mb" })`；
- 上传文件大小与数量；
- 分页大小上限；
- 查询超时与并发数；
- 正则与递归深度，防止 ReDoS。

⚠ **关键认知**：限流要放在**认证之前还是之后**，取决于目标。防止暴力破解应针对 IP + 账号；保护后端资源应针对用户身份。单机内存限流在 cluster 下会失真，多实例部署要用 Redis 等共享计数。

## 7.3 错误信息与日志

```javascript
app.use((err, req, res, next) => {
  logger.error({ err, requestId: req.id }, "请求失败");

  res.status(err.status ?? 500).json({
    error: "服务器内部错误", // 对外不暴露堆栈
    requestId: req.id,
  });
});
```

对外返回通用信息 + 请求 ID，内部日志保留完整堆栈，兼顾安全与排查效率。

# 8. 练习

1. 找出下面代码的漏洞并修复：

   ```javascript
   app.get("/file", (req, res) => {
     res.sendFile(path.join("/srv/uploads", req.query.name));
   });
   ```

2. 写一个 URL 校验函数，只允许 `https` 协议，并拒绝解析到 `127.0.0.1`、`10.0.0.0/8`、`192.168.0.0/16` 的地址。

3. 给一个删除评论的接口补上归属校验，要求非作者返回 403、资源不存在返回 404。

# 9. 小结

| 风险 | 核心防护 |
|------|---------|
| 输入不可信 | schema 校验 + 字段白名单 |
| SQL 注入 | 参数化查询，不拼字符串 |
| 命令注入 | 用 `execFile` / `spawn`，避免 shell |
| 路径穿越 | 解析绝对路径 + 前缀校验 |
| SSRF | 协议/域名白名单 + 解析后校验 IP |
| 供应链 | lockfile、审计、少依赖 |
| 密钥泄漏 | 环境变量/密钥服务，泄漏即轮换 |
| 越权 | 认证之后必须校验资源归属 |
| DoS | 限流 + 体积/并发/超时上限 |

下一篇进入部署与总结，讲清楚环境配置、进程管理、Docker 镜像、健康检查、优雅关闭与监控，并给出上线检查清单。
