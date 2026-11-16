本地 `uvicorn main:app --reload` 跑通，只是开发的起点。上线要考虑的是一组完全不同的问题：怎么起进程、如何利用多核、容器怎么构建、健康检查怎么配、重启时如何不丢请求、线上怎么观测。这一篇把这些工程问题串起来，最后对整个 FastAPI 模块做收束。

# 1. 生产进程模型

## 1.1 uvicorn 与 gunicorn 的分工

| 组件 | 角色 | 特点 |
|------|------|------|
| uvicorn | ASGI 服务器 | 原生支持 async，单进程 |
| gunicorn | 进程管理器 | 负责 fork、监控、重启 worker |
| uvicorn worker | gunicorn 的 worker 类型 | 每个 worker 跑一个 uvicorn |

两种常见组合：

```bash
# 方式一：纯 uvicorn 多 worker（简单）
uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4

# 方式二：gunicorn 管理 uvicorn worker（更成熟）
gunicorn app.main:app \
  -k uvicorn.workers.UvicornWorker \
  -w 4 \
  -b 0.0.0.0:8000
```

★ **关键认知**：**worker 数不是越多越好**。每个 worker 是独立进程，有自己的内存和事件循环。经验值是 `CPU 核数` 到 `2 × 核数 + 1`，但要看内存和 IO 占比。数据库连接池也要按 `worker 数 × 每 worker 连接数` 估算，否则会打爆数据库连接上限。

## 1.2 单进程 vs 多进程

```text
单进程：
  ┌──────────────────────────┐
  │ uvicorn (1 个事件循环)    │
  └──────────────────────────┘

gunicorn 多 worker：
  ┌────────────┐
  │ gunicorn   │  负责调度与重启
  └──┬──┬──┬───┘
     ▼  ▼  ▼
  worker worker worker
  各自一个事件循环，互不共享内存
```

⚠ **多 worker 下内存状态不共享**。内存限流、本地缓存、进程内锁都会失效，必须换成 Redis 等外部存储。第 6 篇的限流示例在多 worker 部署下就不成立。

生产必须关 `--reload`、按 CPU 配 worker 数、关闭或限制 `/docs`、提升日志级别并脱敏错误详情。

# 2. 配置管理

## 2.1 环境变量

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    app_name: str = "fastapi-service"
    debug: bool = False
    database_url: str
    secret_key: str
    workers: int = 4

settings = Settings()
```

生产环境不挂 `.env` 文件，直接用容器/平台注入的环境变量；`.env` 只服务本地开发。

通过 `APP_ENV` 区分环境。**不要把生产密钥写进镜像**，构建时只打包代码，运行时再注入。

# 3. Docker 部署

## 3.1 Dockerfile

```dockerfile
FROM python:3.11-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app ./app

EXPOSE 8000

CMD ["gunicorn", "app.main:app", \
     "-k", "uvicorn.workers.UvicornWorker", \
     "-w", "4", "-b", "0.0.0.0:8000", \
     "--access-logfile", "-", "--error-logfile", "-"]
```

| 指令 | 作用 |
|------|------|
| `PYTHONUNBUFFERED=1` | 日志实时输出，不被缓冲 |
| `PYTHONDONTWRITEBYTECODE=1` | 不生成 `.pyc` |
| 先 COPY requirements | 利用层缓存，代码变动不重装依赖 |
| `slim` 镜像 | 体积小，够用 |

⚠ **不要用 `root` 运行**。生产镜像应创建普通用户：

```dockerfile
RUN useradd -m appuser
USER appuser
```

## 3.2 docker-compose

```yaml
services:
  api:
    build: .
    ports: ["8000:8000"]
    environment:
      DATABASE_URL: postgresql+asyncpg://app:secret@db:5432/app
      SECRET_KEY: ${SECRET_KEY}
    depends_on:
      - db
    restart: unless-stopped
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data
volumes:
  pgdata:
```

`depends_on` 只保证启动顺序；要等数据库就绪，需给 db 加 healthcheck 并用 `condition: service_healthy`，否则会出现"启动时连不上库"。

镜像瘦身靠 `slim` 基础镜像、多阶段构建、`.dockerignore` 和只装运行依赖。

# 4. 健康检查

## 4.1 两种探针

| 探针 | 回答的问题 | 失败后果 |
|------|-----------|---------|
| liveness | 进程还活着吗 | 重启容器 |
| readiness | 能接流量吗 | 从负载均衡摘除 |

```python
from fastapi import FastAPI
from sqlalchemy import text

app = FastAPI()

@app.get("/health/live")
async def liveness():
    return {"status": "ok"}          # 只看进程是否存活

@app.get("/health/ready")
async def readiness(session: SessionDep):
    try:
        await session.execute(text("SELECT 1"))
    except Exception:
        raise HTTPException(status_code=503, detail="数据库不可用")
    return {"status": "ready"}
```

★ **关键认知**：**liveness 不要检查外部依赖**。数据库短暂抖动时，如果 liveness 也失败，容器会被反复重启，反而放大故障。依赖检查放 readiness，让流量暂时绕开即可。

健康检查不要做重查询或写操作，超时控制在 1-3 秒，返回码明确为 200 或 503。

# 5. 优雅关闭

## 5.1 为什么需要

容器重启、滚动发布时会发 `SIGTERM`。进程若立刻退出，正在处理的请求会被截断；正确顺序是停止接新请求、等待进行中的请求完成、释放资源再退出。

## 5.2 lifespan 管理资源

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.pool = await create_pool()     # 启动：建连接池
    yield
    await app.state.pool.close()             # 关闭：释放资源

app = FastAPI(lifespan=lifespan)
```

`yield` 之前的代码在启动时执行，之后的代码在关闭时执行。

## 5.3 给 gunicorn 留时间

```bash
gunicorn app.main:app \
  -k uvicorn.workers.UvicornWorker \
  -w 4 \
  --graceful-timeout 30 \
  --timeout 60
```

| 参数 | 含义 |
|------|------|
| `--graceful-timeout` | 等待 worker 处理完请求的上限 |
| `--timeout` | worker 无响应多久被强杀 |

⚠ **优雅关闭时间要小于编排平台的强杀时间**。Kubernetes 默认 `terminationGracePeriodSeconds` 是 30 秒，如果 gunicorn 也设 30 秒，可能来不及清理就被 SIGKILL。

# 6. 监控与可观测性

## 6.1 三根支柱

Metrics 看趋势（Prometheus + Grafana），Logs 看细节（ELK / Loki），Traces 看调用链（OpenTelemetry + Jaeger）。

## 6.2 关键指标

重点盯 QPS、P95/P99 延迟、5xx 与 4xx 比例、连接池使用率、事件循环延迟。

## 6.3 Prometheus 指标接入

```bash
pip install prometheus-fastapi-instrumentator
```

```python
from prometheus_fastapi_instrumentator import Instrumentator

Instrumentator().instrument(app).expose(app, endpoint="/metrics")
```

默认会采集请求数、延迟直方图、状态码分布。`/metrics` 应只对内网开放。

## 6.4 结构化日志

用 JSON 输出日志字段（`request_id`、`method`、`path`、`status`、`duration_ms`），采集系统按字段检索，比纯文本 grep 高效得多。

★ **关键认知**：**可观测性不是"多打日志"，而是"能回答未知问题"**。线上出问题时，你需要在不改代码的前提下回答：哪个接口慢、慢在哪一段、影响多少用户。Metrics 看面，Traces 看线，Logs 看点，三者配合。

# 7. 安全上线清单

## 7.1 部署前检查

| 项目 | 检查点 |
|------|--------|
| 密钥 | 环境变量注入，未进镜像/仓库 |
| 调试 | `debug=False`，`--reload` 已关 |
| 文档 | `/docs` 关闭或限内网 |
| CORS | 明确来源，未用通配 + 凭证 |
| HTTPS | 由反向代理或网关终止 TLS |
| 依赖 | 无已知高危漏洞（`pip-audit`） |
| 权限 | 容器非 root，数据库最小权限 |
| 日志 | 敏感字段脱敏 |

生产通常在 uvicorn 前放 Nginx / Traefik / 云负载均衡，负责终止 TLS、限流和转发真实 IP。FastAPI 取客户端 IP 时要信任代理头，可用 `--proxy-headers` 配合 `--forwarded-allow-ips`。

# 8. 整个模块的收束

## 8.1 九篇回顾

| 篇目 | 核心问题 | 一句话答案 |
|------|---------|-----------|
| 1 概览与快速开始 | FastAPI 是什么 | 类型注解驱动的 ASGI 框架 |
| 2 路由与参数 | 参数从哪来 | 路径必填、查询可选、请求体用模型 |
| 3 Pydantic 模型 | 数据怎么校验 | 模型即契约，输入输出分离 |
| 4 依赖注入 | 逻辑怎么复用 | `Depends` 声明前置逻辑，`yield` 管资源 |
| 5 异步与数据库 | 怎么不阻塞 | 真异步用 async，同步库用 def |
| 6 认证与安全 | 怎么保护接口 | OAuth2 + JWT + 权限依赖 |
| 7 错误处理 | 出错怎么返回 | 领域异常 + 统一处理器 |
| 8 测试 | 怎么保证不坏 | 依赖覆盖 + 事务回滚 + 契约断言 |
| 9 部署与总结 | 怎么上线 | 多 worker + 容器 + 健康检查 + 监控 |

## 8.2 一条完整的学习路径

学习路径是：写得出（1-3 路由/参数/模型）→ 组织好（4 依赖注入）→ 跑得动（5 异步/数据库）→ 守得住（6-7 认证/错误）→ 测得全（8 测试）→ 上得去（9 部署/监控）。

继续深入可看 Alembic 迁移、分层架构、压测与缓存、OpenTelemetry、CI/CD 与任务队列。三条建议：类型注解写完整；不要过早追求全异步；测试从第一天开始写。

# 9. 练习

1. 给现有项目写一个多阶段 `Dockerfile`，用非 root 用户运行，构建后用 `docker run` 启动并访问 `/health/live`。
2. 配置 `/health/live` 与 `/health/ready` 两个端点，readiness 检查数据库连通性，用 `docker compose` 验证数据库未就绪时 API 被标记为不健康。
3. 接入 `prometheus-fastapi-instrumentator`，访问 `/metrics` 确认出现请求计数与延迟指标，并写一条 5xx 比例告警的 PromQL。

# 10. 小结

| 主题 | 要点 |
|------|------|
| 进程模型 | gunicorn 管进程，uvicorn worker 跑事件循环 |
| worker 数 | 按 CPU 与内存估算，连接池同步放大 |
| 配置 | 环境变量注入，生产不挂 `.env` |
| 容器 | slim 镜像、非 root、先装依赖、健康检查 |
| 健康检查 | liveness 只看存活，readiness 查依赖 |
| 优雅关闭 | lifespan 释放资源，留足 graceful 时间 |
| 监控 | Metrics 看面、Traces 看线、Logs 看点 |
| 上线 | HTTPS、CORS、文档、密钥、依赖审计逐项确认 |

至此 FastAPI 模块结束。建议回到第 1 篇，用"类型注解即契约"的视角重读一遍，再挑一个真实小项目，把路由、模型、依赖、数据库、测试和部署完整走一遍。
