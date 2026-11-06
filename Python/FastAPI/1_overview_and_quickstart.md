FastAPI 这几年几乎成了 Python 后端接口的默认选择：写起来像 Flask 一样轻，却自带类型校验、依赖注入和 OpenAPI 文档。但它不是"更快的 Flask"，而是建立在 ASGI 与类型注解之上的一整套接口开发范式。本篇先把定位、运行模型和最小服务讲清楚，后面八篇才有共同的坐标系。

# 1. FastAPI 是什么

## 1.1 一句话定位

FastAPI 是一个**基于标准类型注解的现代 Python Web 框架**，专为构建 API 而生。它把三件事缝合在一起：

| 组成 | 作用 |
|------|------|
| Starlette | 提供 ASGI 应用、路由、中间件、请求响应对象 |
| Pydantic | 提供数据模型、校验、序列化 |
| 类型注解 + 反射 | 把函数签名自动翻译成校验规则和文档 |

所以你在 FastAPI 里写的不是"配置"，而是**普通的带类型注解的 Python 函数**：

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/items/{item_id}")
def read_item(item_id: int, q: str | None = None):
    return {"item_id": item_id, "q": q}  # 返回 dict 会被自动转成 JSON
```

这段代码同时得到了：路径解析、`item_id` 的整数转换、`q` 的可选校验、JSON 序列化、以及 `/docs` 里的一份接口文档。没有任何一行是"为了框架"而写的。

★ **关键认知**：FastAPI 的核心不是"装饰器路由"，而是**把函数签名当成接口契约**。参数名、类型、默认值、返回类型，都会在启动时被读取并转成校验逻辑与 OpenAPI schema。理解这一点，后面所有"为什么这样写"都有了答案。

## 1.2 它解决了什么痛点

传统写法里，一个接口往往要重复四遍同样的信息：

```text
路由声明  →  参数解析  →  参数校验  →  文档描述
   ↓            ↓            ↓            ↓
 容易漏       手写转换      手写 if       手写维护
```

FastAPI 把这四步合并成一处函数签名，改一处就全同步。代价是：你必须认真写类型注解，否则框架帮不上忙。

# 2. ASGI 与运行模型

## 2.1 从 WSGI 到 ASGI

要理解 FastAPI 为什么能异步，先要看它站在什么协议上。

| 协议 | 全称 | 调用形式 | 并发模型 |
|------|------|---------|---------|
| WSGI | Web Server Gateway Interface | 同步函数，一次一个请求 | 线程/进程 |
| ASGI | Asynchronous Server Gateway Interface | 异步可等待对象 | 单线程事件循环，可挂起 |

WSGI 的接口大致是 `app(environ, start_response)`，它是同步阻塞的：请求没处理完，这个调用就不返回。ASGI 的接口是 `async def app(scope, receive, send)`，处理过程中可以在 `await` 处让出控制权。

```text
WSGI：  请求 → 工作线程阻塞处理 → 返回 → 线程才能接下一个
ASGI：  请求 → 协程挂起等待 IO → 事件循环去处理别的请求 → IO 就绪再回来
```

★ **关键认知**：FastAPI 本身不跑服务器，它只是一个 ASGI 应用对象。真正监听端口、解析 HTTP、调度事件循环的是 **uvicorn** 这类 ASGI 服务器。`app` 是"处理器"，uvicorn 是"发动机"。

## 2.2 一个请求的完整链路

```text
客户端
  │  HTTP 请求
  ▼
┌──────────────┐
│   uvicorn    │  解析 HTTP，构造 scope / receive / send
└──────┬───────┘
       ▼
┌──────────────┐
│  Starlette   │  中间件链、路由匹配
└──────┬───────┘
       ▼
┌──────────────┐
│   FastAPI    │  依赖注入、参数解析、Pydantic 校验
└──────┬───────┘
       ▼
┌──────────────┐
│  你的函数     │  def 或 async def
└──────┬───────┘
       ▼
   响应模型序列化 → JSONResponse → 客户端
```

这张图很重要：**依赖注入和校验发生在你的函数之前**，函数拿到的已经是干净、可信的参数。

## 2.3 同步函数与异步函数

FastAPI 允许两种写法：

```python
@app.get("/sync")
def sync_endpoint():
    return {"mode": "sync"}      # 在线程池里执行

@app.get("/async")
async def async_endpoint():
    return {"mode": "async"}     # 在事件循环里执行
```

| 写法 | 执行位置 | 适合场景 |
|------|---------|---------|
| `def` | 线程池（默认约 40 个线程） | 同步库、CPU 轻计算 |
| `async def` | 事件循环 | 异步库、IO 等待 |

⚠ **不要用 `async def` 去包同步阻塞代码**。比如在 `async def` 里调用 `time.sleep` 或 `requests.get`，会卡死整个事件循环，所有并发请求一起停摆。同步代码就用 `def`，让 FastAPI 丢进线程池。

# 3. 与 Flask、Django 的对比

## 3.1 三者的定位

| 维度 | FastAPI | Flask | Django |
|------|---------|-------|--------|
| 定位 | API 优先 | 微框架 | 全栈框架 |
| 协议 | ASGI | WSGI（3.x 起支持 ASGI） | WSGI（4.x 起支持 ASGI） |
| 数据校验 | Pydantic 内建 | 需扩展 | Form/DRF Serializer |
| 自动文档 | 内建 OpenAPI | 需扩展 | 需 DRF + 扩展 |
| ORM | 不自带 | 不自带 | 自带 ORM |
| 后台管理 | 无 | 无 | 自带 admin |
| 学习曲线 | 中（需懂类型注解） | 低 | 高 |
| 典型场景 | 微服务、AI 接口 | 小工具、原型 | 内容型网站、后台系统 |

★ **关键认知**：FastAPI 的"快"不只是性能，更是**开发反馈快**——类型标注一写，校验、文档、序列化全到位。但它**不提供 ORM、迁移、后台管理**，这些要自己选型拼装。它更像"接口层框架"，不是 Django 那样的全家桶。

## 3.2 什么时候不该选 FastAPI

- 需要开箱即用的后台管理、权限、ORM 一体化：选 Django。
- 只有一个页面或极简脚本：Flask 足够。
- 团队不熟悉类型注解和异步：先补基础，否则会写出"伪异步"。
- 重度依赖同步 ORM 且不想改造：FastAPI 能用，但异步收益打折。

# 4. 安装与环境

## 4.1 建议的版本

本篇以 **FastAPI 0.11x、Pydantic 2.x、Python 3.11** 为准。三者版本关系紧密，不要混用旧版：

```bash
python --version                  # 确认 3.11+
python -m venv .venv              # 创建虚拟环境
# Windows PowerShell
.venv\Scripts\Activate.ps1
# macOS / Linux
source .venv/bin/activate

pip install "fastapi[standard]"   # 含 uvicorn、httpx、jinja2 等常用件
```

`fastapi[standard]` 会带上 `uvicorn`、`fastapi-cli`、`httpx` 等，适合大多数项目。只想最小化：

```bash
pip install fastapi uvicorn
```

## 4.2 依赖清单

```text
fastapi==0.115.*      # 框架本体
pydantic==2.*         # 数据校验（FastAPI 0.100+ 要求 2.x）
uvicorn[standard]==0.32.*  # ASGI 服务器，standard 带 uvloop/httptools
```

⚠ **不要把 Pydantic 1.x 和 FastAPI 0.100+ 混用**。Pydantic 2 是一次不兼容重写，`validator`、`Config` 等写法都变了。网上大量旧教程仍是 1.x 语法，照抄会报错。

# 5. 最小可运行服务

## 5.1 目录结构

```text
myapp/
├── .venv/
├── main.py
└── requirements.txt
```

## 5.2 完整代码

```python
# main.py
from fastapi import FastAPI

app = FastAPI(
    title="示例服务",          # 出现在文档标题
    version="0.1.0",          # 出现在文档版本
    description="FastAPI 最小示例",
)

@app.get("/")
def read_root():
    return {"message": "Hello FastAPI"}

@app.get("/hello/{name}")
def say_hello(name: str):
    return {"message": f"你好，{name}"}
```

## 5.3 启动

```bash
uvicorn main:app --reload
```

参数含义：

| 参数 | 含义 |
|------|------|
| `main:app` | 模块 `main` 里的 `app` 对象 |
| `--reload` | 改代码自动重启，仅开发用 |
| `--host 0.0.0.0` | 允许外部访问，默认只监听 127.0.0.1 |
| `--port 8000` | 指定端口 |

启动后访问 `http://127.0.0.1:8000/hello/Alice`，返回：

```json
{"message": "你好，Alice"}
```

## 5.4 自动文档

FastAPI 默认提供两套文档：

| 路径 | 内容 |
|------|------|
| `/docs` | Swagger UI，可展开、可试调 |
| `/redoc` | ReDoc，阅读体验更好 |
| `/openapi.json` | 原始 OpenAPI schema |

★ **关键认知**：文档不是手写的，是**从代码反射出来的**。改了函数签名、加了 Pydantic 模型，文档会自动同步。这也是为什么"类型注解写得越完整，文档质量越高"。

⚠ **生产环境通常要关掉 `/docs`**，避免暴露接口结构。可以这样控制：

```python
app = FastAPI(docs_url=None, redoc_url=None)
# 需要时只在内网环境单独挂载
```

# 6. 常见错误

## 6.1 启动时报 "Could not import module"

- 原因：`uvicorn main:app` 里的 `main` 是**模块路径**，不是文件名随便写。
- 检查：当前目录下是否有 `main.py`，里面是否有名为 `app` 的对象。
- 包结构下要写全路径，例如 `uvicorn app.main:app`。

## 6.2 返回 dict 却报序列化错误

FastAPI 会用 `jsonable_encoder` 转常见类型，但遇到 `datetime`、`Decimal`、自定义类需要模型声明。返回任意对象并不总是可行。

## 6.3 端口被占用

```bash
# Windows 查看占用
netstat -ano | findstr :8000
# 换端口
uvicorn main:app --port 8001
```

## 6.4 改了代码没生效

`--reload` 只监听 Python 文件，且在某些 IDE 的调试模式下失效。确认改动已保存、进程确实是 reload 模式启动的。

# 7. 练习

1. 建一个服务，提供 `/`、`/hello/{name}`、`/add?a=1&b=2` 三个接口，其中 `a`、`b` 声明为 `int`，访问 `/docs` 确认参数类型被正确标注。
2. 故意访问 `/add?a=abc&b=2`，观察返回的状态码和错误结构，理解 FastAPI 自动校验的行为。
3. 把 `title`、`version`、`description` 改成自己的项目信息，对比 `/docs` 与 `/openapi.json` 的变化。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 定位 | 基于类型注解的 API 框架，缝合 Starlette + Pydantic |
| ASGI | 异步网关协议，uvicorn 负责运行，FastAPI 是应用对象 |
| 同步/异步 | `def` 走线程池，`async def` 走事件循环，别互相污染 |
| 对比 | 比 Flask 强在类型与文档，比 Django 少 ORM 与后台 |
| 安装 | Python 3.11 + FastAPI 0.11x + Pydantic 2.x |
| 最小服务 | `app = FastAPI()` + 路由函数 + `uvicorn main:app --reload` |
| 文档 | `/docs`、`/redoc`、`/openapi.json` 自动生成 |

下一篇进入路由与参数，讲清路径参数、查询参数、请求体三者的边界，`status_code` 与 `APIRouter` 的用法，以及依赖注入的第一次登场。
