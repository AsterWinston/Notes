代码写完只是开始，真正决定服务是否稳定的是部署方式：配置是否分层、进程崩了能否自动重启、容器镜像是否安全、上线时旧连接能否优雅收尾、出问题能否第一时间发现。本篇把部署链路走一遍，并收束整个 Node.js 模块。

# 1. 环境配置

## 1.1 配置分层

一套代码要跑在开发、测试、生产多个环境，差异只能放在配置里：

```text
代码(不变) + 环境变量(按环境变化) = 运行实例
```

```text
开发:  NODE_ENV=development  DATABASE_URL=localhost
测试:  NODE_ENV=test         DATABASE_URL=test-db
生产:  NODE_ENV=production   DATABASE_URL=prod-db (来自密钥管理)
```

★ **关键认知**：凡是"不同环境不一样"的东西——数据库地址、密钥、第三方开关、日志级别——都必须外部化。⚠ 不要把生产配置写进代码分支，也不要用 `if (process.env.NODE_ENV === "production")` 在代码里散落环境判断，配置应在启动时一次性读取并校验。

## 1.2 环境变量实践

```javascript
function requireEnv(name) {
  const value = process.env[name];
  if (!value) {
    throw new Error(`缺少环境变量 ${name}`);
  }
  return value;
}

export const config = {
  port: Number(process.env.PORT ?? 3000),
  nodeEnv: process.env.NODE_ENV ?? "development",
  databaseUrl: requireEnv("DATABASE_URL"),
  jwtSecret: requireEnv("JWT_SECRET"),
  logLevel: process.env.LOG_LEVEL ?? "info",
};
```

启动时校验的好处是**快速失败**：配置缺失立刻退出，而不是运行到某个请求才报错。

```bash
node --env-file=.env src/index.js
```

Node 20.6+ 支持 `--env-file`，本地开发不需要额外装 `dotenv`。⚠ 生产环境不要把 `.env` 打进镜像，应由平台注入。

## 1.3 十二要素应用

Node 服务常用的部署原则可归纳为：

| 原则 | 含义 |
|------|------|
| 一份代码库 | 同一份代码部署到多个环境 |
| 显式依赖 | 依赖写进 `package.json` 与锁文件 |
| 配置外置 | 环境变量，不写死 |
| 无状态 | 进程不依赖本地磁盘保存会话 |
| 可丢弃 | 进程随时可被杀掉重启 |
| 日志即事件流 | 输出到 stdout，由平台收集 |
| 优雅退出 | 收到信号后完成存量请求 |

# 2. 进程管理

## 2.1 为什么需要进程管理器

`node src/index.js` 有两个问题：

- 进程崩溃后不会自动重启；
- 单进程只能用到一个 CPU 核心。

进程管理器负责：**守护、重启、多实例、日志、开机自启**。

## 2.2 PM2

```bash
npm install -g pm2
pm2 start src/index.js --name notes-api -i max
pm2 logs notes-api
pm2 status
pm2 reload notes-api
pm2 save
pm2 startup
```

`-i max` 会按 CPU 核心数启动多个实例，内部使用 cluster。

配置文件：

```javascript
// ecosystem.config.cjs
module.exports = {
  apps: [
    {
      name: "notes-api",
      script: "src/index.js",
      instances: "max",
      exec_mode: "cluster",
      max_memory_restart: "512M",
      env: {
        NODE_ENV: "production",
        PORT: 3000,
      },
    },
  ],
};
```

```bash
pm2 start ecosystem.config.cjs
pm2 reload notes-api   # 零停机重载
```

⚠ **关键认知**：`restart` 会先杀进程再启动，期间服务不可用；`reload` 会逐个替换实例，适合生产。配置了 `max_memory_restart` 后，内存超限会触发重启，这能兜底泄漏，但**只是缓解，不是修复**。

## 2.3 systemd

不用 PM2 时，可以直接交给 systemd：

```ini
[Unit]
Description=Notes API
After=network.target

[Service]
Type=simple
User=notes
WorkingDirectory=/srv/notes-api
EnvironmentFile=/etc/notes-api/env
ExecStart=/usr/bin/node src/index.js
Restart=always
RestartSec=3
KillSignal=SIGTERM
TimeoutStopSec=30

[Install]
WantedBy=multi-user.target
```

这组配置的关键点：

- `Restart=always`：崩溃自动拉起；
- `KillSignal=SIGTERM`：先发优雅关闭信号；
- `TimeoutStopSec=30`：30 秒内没退出则强杀；
- `EnvironmentFile`：配置与代码分离。

# 3. 容器化

## 3.1 最小 Dockerfile

```dockerfile
FROM node:22-alpine AS base
WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci --omit=dev

COPY src ./src

ENV NODE_ENV=production
EXPOSE 3000

USER node
CMD ["node", "src/index.js"]
```

要点：

- 先拷贝 `package.json` 与锁文件，利用 Docker 层缓存；
- 用 `npm ci` 保证可复现；
- `--omit=dev` 不装开发依赖；
- 用非 root 用户运行；
- 基础镜像选 `alpine` 或 `slim`，减小体积与攻击面。

## 3.2 多阶段构建

TypeScript 项目需要构建，但生产镜像不该包含编译工具：

```dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
COPY --from=build /app/dist ./dist
USER node
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

最终镜像只保留运行所需的文件，体积更小，也不会把源码与开发依赖带上线。

## 3.3 .dockerignore

```text
node_modules
dist
.git
.env
*.log
coverage
```

★ **关键认知**：`node_modules` 必须在 `.dockerignore` 里排除。否则本地依赖会被复制进镜像，覆盖容器内 `npm ci` 安装的结果，导致平台相关的二进制不匹配。⚠ 同时排除 `.env`，避免密钥被打进镜像层。

## 3.4 容器运行建议

```bash
docker build -t notes-api:1.0.0 .
docker run -d --name notes-api -p 3000:3000 \
  --env-file .env \
  --memory 512m \
  --cpus 1 \
  notes-api:1.0.0
```

- 限制内存与 CPU，防止单容器拖垮宿主机；
- 配置日志驱动，避免容器日志无限增长；
- 挂载只读文件系统，减少运行时可写面；
- 用编排平台管理副本、滚动更新与健康检查。

# 4. 健康检查与优雅关闭

## 4.1 健康检查

健康检查分两种：

| 类型 | 回答的问题 | 检查内容 |
|------|-----------|---------|
| Liveness | 进程还活着吗 | 进程能否响应 |
| Readiness | 能接收流量吗 | 数据库、缓存等依赖是否就绪 |

```javascript
app.get("/health/live", (req, res) => {
  res.json({ status: "ok" });
});

app.get("/health/ready", async (req, res) => {
  try {
    await db.query("SELECT 1");
    res.json({ status: "ready" });
  } catch (err) {
    res.status(503).json({ status: "not ready" });
  }
});
```

★ **关键认知**：`liveness` 失败会触发重启，`readiness` 失败只是暂时摘流量。⚠ 不要把数据库检查放进 `liveness`：数据库抖动会导致所有实例被重启，反而放大故障。

## 4.2 优雅关闭

进程收到 `SIGTERM` 后应该：

```text
1. 停止接受新连接
2. 继续处理存量请求
3. 关闭数据库连接、消息队列、定时器
4. 在超时时间内退出,超时则强杀
```

```javascript
import http from "node:http";

const server = http.createServer(app);

server.listen(3000, () => {
  console.log("服务已启动");
});

let shuttingDown = false;

async function shutdown(signal) {
  if (shuttingDown) return;
  shuttingDown = true;

  console.log(`收到 ${signal},开始优雅关闭`);

  server.close(async (err) => {
    if (err) {
      console.error("关闭服务器失败", err);
      process.exit(1);
    }

    try {
      await db.close();
      console.log("资源已释放");
      process.exit(0);
    } catch (closeErr) {
      console.error("关闭资源失败", closeErr);
      process.exit(1);
    }
  });

  // 兜底:超过 30 秒仍未关闭则强制退出
  setTimeout(() => {
    console.error("优雅关闭超时,强制退出");
    process.exit(1);
  }, 30_000).unref();
}

process.on("SIGTERM", () => shutdown("SIGTERM"));
process.on("SIGINT", () => shutdown("SIGINT"));
```

⚠ **关键认知**：`server.close()` 只停止接收**新连接**，不会强制中断已有连接。如果有长连接(WebSocket)、未完成的请求，需要额外跟踪并主动结束。容器编排的 `terminationGracePeriodSeconds` 要大于应用内的关闭超时，否则会在应用还没收尾时被强杀。

## 4.3 滚动更新

```text
旧实例 v1 [运行中] ──► [停止接收新流量] ──► [等待存量请求] ──► [退出]
新实例 v2 [启动]     ──► [就绪检查通过]  ──► [接入流量]
```

前提是**服务无状态**：会话放 Redis，文件放对象存储，任何实例都能处理任何请求。

# 5. 监控与可观测性

## 5.1 三大支柱

| 支柱 | 回答的问题 | 典型工具 |
|------|-----------|---------|
| Metrics 指标 | 系统整体是否健康 | Prometheus、Grafana |
| Logs 日志 | 某个请求发生了什么 | Loki、ELK |
| Traces 链路 | 请求在哪些服务耗时 | OpenTelemetry、Jaeger |

## 5.2 关键指标

**RED 方法**(面向服务)：

- **Rate**：每秒请求数；
- **Errors**：错误率；
- **Duration**：响应耗时分布。

**USE 方法**(面向资源)：

- **Utilization**：CPU、内存使用率；
- **Saturation**：队列长度、事件循环延迟；
- **Errors**：系统级错误。

Node 特别值得盯的指标：

| 指标 | 说明 |
|------|------|
| 事件循环延迟 | 主线程是否被阻塞 |
| `heapUsed` | 是否存在内存泄漏 |
| 活跃句柄数 | 是否有连接/定时器泄漏 |
| HTTP 5xx 比例 | 服务健康度 |
| P99 响应时间 | 用户体验 |
| 进程重启次数 | 是否在崩溃循环 |

```javascript
import client from "prom-client";

client.collectDefaultMetrics();

const httpDuration = new client.Histogram({
  name: "http_request_duration_seconds",
  help: "HTTP 请求耗时",
  labelNames: ["method", "route", "status"],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 5],
});

app.use((req, res, next) => {
  const end = httpDuration.startTimer();
  res.on("finish", () => {
    end({ method: req.method, route: req.route?.path ?? req.path, status: res.statusCode });
  });
  next();
});

app.get("/metrics", async (req, res) => {
  res.set("Content-Type", client.register.contentType);
  res.end(await client.register.metrics());
});
```

## 5.3 告警原则

- 告警要**可行动**，不能"知道了但什么也做不了"；
- 优先基于**症状**(错误率、延迟)而不是原因(CPU 高)；
- 设置合理的持续时间，避免瞬时抖动触发；
- 分级：致命立即通知，警告进日报；
- 每条告警附上排查入口(仪表盘、日志查询链接)。

★ **关键认知**：监控的价值不在"数据好看"，而在**缩短故障发现与定位时间**。没有 requestId、没有结构化日志、没有关键指标的服务，出问题时只能靠猜。

# 6. 上线检查清单

## 6.1 构建与依赖

- [ ] 锁文件已提交，CI 使用 `npm ci`；
- [ ] 生产安装使用 `--omit=dev`；
- [ ] `engines` 声明的 Node 版本与运行环境一致；
- [ ] 镜像使用非 root 用户，`.dockerignore` 排除 `node_modules` 与 `.env`；
- [ ] 依赖审计无高危漏洞，或有明确豁免记录。

## 6.2 配置与安全

- [ ] 所有配置来自环境变量，启动时校验；
- [ ] 密钥来自密钥管理服务，未写入镜像与仓库；
- [ ] 生产关闭调试端口与详细错误堆栈；
- [ ] 已配置限流、请求体上限、CORS 白名单；
- [ ] 安全响应头(helmet)已启用。

## 6.3 运行与观测

- [ ] 有 `liveness` 与 `readiness` 两个健康检查；
- [ ] 收到 `SIGTERM` 能优雅关闭，超时时间与编排配置匹配；
- [ ] 进程崩溃能自动重启，且有重启次数告警；
- [ ] 日志为结构化 JSON，带 requestId；
- [ ] 关键指标(RED + 事件循环延迟 + 内存)已接入监控；
- [ ] 有回滚方案，且回滚步骤演练过。

## 6.4 常见上线事故

| 事故 | 根因 | 预防 |
|------|------|------|
| 启动即崩溃 | 缺少环境变量 | 启动时校验配置 |
| 内存 OOM | 无内存上限、泄漏 | 限制内存 + 监控 + 堆快照 |
| 更新期间 502 | 先杀旧再起新 | 滚动更新 + readiness |
| 日志爆盘 | 未配置轮转 | 输出 stdout，由平台轮转 |
| 时区错误 | 容器默认 UTC | 统一用 UTC 存储，展示层转换 |
| 静态资源 404 | 构建产物未进镜像 | 多阶段构建检查产物 |

# 7. 练习

1. 给一个 Node 服务编写最小 `Dockerfile`，要求使用多阶段构建、非 root 用户运行，并通过 `.dockerignore` 排除 `node_modules` 与 `.env`。

2. 实现一个优雅关闭流程：收到 `SIGTERM` 后停止接收新连接、等待 10 秒内的存量请求完成、关闭数据库连接，超时则以退出码 1 结束。

3. 为服务设计三个监控指标（请求错误率、P99 延迟、事件循环延迟），说明各自的采集方式与告警阈值。

# 8. 总结与下一步

## 8.1 模块知识地图

```text
运行时与事件循环
      │
      ├── 模块系统 ──► 包管理
      │
      ├── fs/path/process
      │
      ├── HTTP/Express ──► 异步与流
      │                         │
      │                         ▼
      │                  cluster/worker
      │
      ├── 测试与调试
      │
      ├── 安全
      │
      └── 部署与监控
```

## 8.2 核心结论回顾

| 主题 | 一句话结论 |
|------|-----------|
| 事件循环 | JS 单线程，靠 libuv 把等待外包出去 |
| 模块 | CJS 同步快照，ESM 静态绑定，`type` 决定 `.js` |
| 包管理 | 锁文件是契约，CI 用 `npm ci` |
| 文件 | 大文件用流，路径用 `path`，错误看 `code` |
| HTTP | 路由按顺序，中间件要明确 `next` |
| 异步 | 并发要有上限，流要处理背压 |
| 多核 | HTTP 用 cluster，计算用 worker_threads |
| 调试 | 火焰图看 CPU，堆快照看内存 Retainer |
| 安全 | 输入校验 + 参数化 + 最小权限 |
| 部署 | 配置外置、无状态、优雅关闭、可观测 |

## 8.3 继续深入的方向

- **数据库**：连接池、事务、索引、迁移工具；
- **消息队列**：RabbitMQ、Kafka，削峰与解耦；
- **缓存**：Redis 的缓存穿透、击穿、雪崩；
- **网关与编排**：Nginx、Kubernetes、服务发现；
- **可观测性**：OpenTelemetry 全链路追踪；
- **运行时进阶**：诊断报告、GC 调优、V8 内存模型。

★ **关键认知**：Node.js 的难点从来不是语法，而是**并发模型、资源边界与生产工程**。把事件循环、流、多核、安全、部署这几条主线吃透，再看任何框架和工具，都能快速判断它在哪一层、解决什么问题、代价是什么。

# 9. 小结

| 主题 | 关键实践 |
|------|---------|
| 配置 | 环境变量外部化，启动即校验 |
| 进程 | PM2/systemd 守护，`reload` 而非 `restart` |
| 容器 | 多阶段构建、非 root、排除 `node_modules` |
| 健康检查 | liveness 探活、readiness 探依赖 |
| 优雅关闭 | 停新连接、等存量、释放资源、超时兜底 |
| 监控 | RED + 事件循环延迟 + 内存，日志带 requestId |
| 上线 | 清单化检查，回滚方案提前演练 |

至此，Node.js 模块的十篇笔记完成。从运行时到部署，主线始终是"理解机制、守住边界、让问题可观测"。接下来可以挑一个方向深入，把知识落到真实项目里。
