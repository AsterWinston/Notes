错误处理是接口质量的分水岭。写得好的服务，出错时返回结构稳定、可定位、不泄露内部细节；写得差的服务，要么到处 `try/except` 吞异常，要么把堆栈直接甩给用户。FastAPI 提供了分层机制：`HTTPException` 表达业务错误，异常处理器统一收口，校验错误有独立入口。本篇把它们组织成一套可复用的错误体系。

# 1. 错误的分类

## 1.1 三类错误

| 类型 | 例子 | 谁负责 |
|------|------|--------|
| 客户端错误 | 参数缺失、资源不存在、无权限 | 4xx，业务代码主动抛 |
| 服务端错误 | 空指针、数据库断连、第三方超时 | 5xx，兜底处理器 |
| 校验错误 | 类型不符、字段越界 | 422，框架自动产生 |

★ **关键认知**：**能被预期的错误是业务的一部分，不是异常**。用户 ID 不存在应该返回 404，而不是让 `None` 继续往下跑直到崩在某个属性访问上。主动抛错比被动崩溃更可控。

## 1.2 统一错误结构

对外错误响应建议统一成固定形状：

```json
{
  "code": "USER_NOT_FOUND",
  "message": "用户不存在",
  "detail": null,
  "request_id": "8f3c2a"
}
```

`code` 给程序判断，`message` 给人阅读，`detail` 放字段级细节，`request_id` 用来串联日志。

⚠ **不要把内部异常文本直接塞进 `message`**。`KeyError: 'user_id'` 对用户毫无意义，还可能泄露表结构。`message` 用稳定文案，`detail` 放脱敏后的细节。

# 2. HTTPException

## 2.1 基本用法

```python
from fastapi import FastAPI, HTTPException, status

app = FastAPI()

@app.get("/users/{user_id}")
def get_user(user_id: int):
    if user_id != 1:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="用户不存在",
        )
    return {"id": 1, "name": "alice"}
```

返回：

```json
{"detail": "用户不存在"}
```

`HTTPException` 会被 FastAPI 的默认处理器捕获，转成对应状态码的 JSON 响应。

## 2.2 常用参数

```python
raise HTTPException(
    status_code=401,
    detail={"code": "TOKEN_EXPIRED", "message": "登录已过期"},
    headers={"WWW-Authenticate": "Bearer"},
)
```

## 2.3 自定义异常 + 处理器

业务代码里到处 `raise HTTPException` 会让 HTTP 细节渗透到服务层。更好的做法是定义领域异常，再统一映射：

```python
class AppError(Exception):
    def __init__(self, code: str, message: str, status_code: int = 400):
        self.code = code
        self.message = message
        self.status_code = status_code

class UserNotFound(AppError):
    def __init__(self, user_id: int):
        super().__init__(
            code="USER_NOT_FOUND",
            message=f"用户 {user_id} 不存在",
            status_code=404,
        )
```

处理器：

```python
from fastapi import Request
from fastapi.responses import JSONResponse

@app.exception_handler(AppError)
async def app_error_handler(request: Request, exc: AppError):
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "code": exc.code,
            "message": exc.message,
            "request_id": getattr(request.state, "request_id", None),
        },
    )
```

★ **关键认知**：**领域异常负责"是什么错"，处理器负责"怎么呈现"**。服务层不 import FastAPI，就能被单元测试直接调用；换 Web 框架时也只需改处理器。

# 3. 全局异常处理器

## 3.1 覆盖默认 HTTPException

```python
from fastapi import HTTPException
from fastapi.exception_handlers import http_exception_handler

@app.exception_handler(HTTPException)
async def custom_http_exception_handler(request: Request, exc: HTTPException):
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "code": f"HTTP_{exc.status_code}",
            "message": exc.detail if isinstance(exc.detail, str) else "请求失败",
            "detail": exc.detail if isinstance(exc.detail, dict) else None,
        },
        headers=exc.headers,
    )
```

## 3.2 兜底 Exception

```python
import logging

logger = logging.getLogger(__name__)

@app.exception_handler(Exception)
async def unhandled_exception_handler(request: Request, exc: Exception):
    logger.exception(
        "未处理异常 path=%s method=%s",
        request.url.path,
        request.method,
    )
    return JSONResponse(
        status_code=500,
        content={
            "code": "INTERNAL_ERROR",
            "message": "服务器内部错误",
            "request_id": getattr(request.state, "request_id", None),
        },
    )
```

⚠ **兜底处理器不能吞掉所有信息**。响应里不暴露堆栈，但**日志必须记录完整堆栈**（`logger.exception` 会自动带上）。否则线上 500 无从排查。

Starlette 按 MRO 查找最近的处理器：精确类型优先，其次父类，最后落到 `Exception` 兜底。

# 4. 校验错误处理

## 4.1 默认格式

当请求体或参数不合法，FastAPI 返回 422：

```json
{
  "detail": [
    {
      "type": "greater_than",
      "loc": ["body", "price"],
      "msg": "Input should be greater than 0",
      "input": -1
    }
  ]
}
```

`loc` 是字段路径，`body` 表示来自请求体。前端可以据此定位表单字段。

## 4.2 自定义校验错误

默认格式带英文且结构固定，很多项目会重写：

```python
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(
    request: Request,
    exc: RequestValidationError,
):
    details = []
    for err in exc.errors():
        loc = err["loc"]
        field = ".".join(str(x) for x in loc if x != "body")
        details.append({
            "field": field,
            "message": translate_error(err["type"]),
        })

    return JSONResponse(
        status_code=422,
        content={
            "code": "VALIDATION_ERROR",
            "message": "请求参数校验失败",
            "detail": details,
        },
    )

def translate_error(error_type: str) -> str:
    mapping = {
        "missing": "该字段必填",
        "greater_than": "必须大于最小值",
        "string_too_short": "字符串过短",
        "int_parsing": "必须是整数",
    }
    return mapping.get(error_type, "字段不合法")
```

★ **关键认知**：校验错误是**框架在进入你的函数之前**产生的，业务代码根本无法捕获。要改格式，只能在 `RequestValidationError` 处理器里做。这也是唯一能统一 422 结构的地方。

# 5. 日志与可观测性

## 5.1 请求 ID

给每个请求分配唯一 ID，贯穿日志与错误响应：

```python
import uuid
from starlette.middleware.base import BaseHTTPMiddleware

class RequestIdMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        request_id = request.headers.get("X-Request-ID") or uuid.uuid4().hex[:8]
        request.state.request_id = request_id
        response = await call_next(request)
        response.headers["X-Request-ID"] = request_id
        return response

app.add_middleware(RequestIdMiddleware)
```

客户端报错时提供 `X-Request-ID`，服务端就能定位到那一条日志。

## 5.2 结构化日志

```python
import logging, json

class JsonFormatter(logging.Formatter):
    def format(self, record):
        data = {"level": record.levelname, "message": record.getMessage()}
        if record.exc_info:
            data["traceback"] = self.formatException(record.exc_info)
        return json.dumps(data, ensure_ascii=False)

handler = logging.StreamHandler()
handler.setFormatter(JsonFormatter())
logging.basicConfig(level=logging.INFO, handlers=[handler])
```

日志按级别使用：DEBUG 开发、INFO 业务、WARNING 降级、ERROR 失败、CRITICAL 不可用。⚠ **不要记录密码、Token、完整身份证号**，日志会集中采集，写入后难以清除。

# 6. 实践建议

## 6.1 错误码与状态码

异常适合"意外"，不适合预期分支。错误码采用 `<模块>_<原因>`，保持稳定、唯一、可读；HTTP 状态码给网关和监控看，业务码给前端分支用。

# 7. 常见错误

| 问题 | 后果 | 正确做法 |
|------|------|---------|
| 到处 `try/except: pass` | 错误被吞，问题难查 | 只捕获能处理的异常 |
| 把堆栈返回给客户端 | 泄露内部结构 | 统一文案，堆栈进日志 |
| 所有错误都返回 400 | 监控与前端判断失效 | 用语义正确的状态码 |
| 校验错误格式不一致 | 前端要写多套解析 | 统一 422 处理器 |
| 日志无 request_id | 无法串联一次请求 | 中间件注入 |
| 异常处理器里再抛异常 | 500 套 500 | 处理器本身要健壮 |

# 8. 练习

1. 定义 `AppError` 基类和 `OrderNotFound`、`OrderAlreadyPaid` 两个子类，分别映射 404 与 409，并写一个统一处理器输出 `code`、`message`、`request_id`。
2. 自定义 `RequestValidationError` 处理器，把默认英文消息换成中文，并把 `loc` 转成 `items.0.price` 这种点号路径。
3. 写一个中间件给每个请求加 `X-Request-ID`，并在兜底 500 处理器里返回同一个 ID，用日志验证能对上。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 错误分类 | 4xx 客户端、5xx 服务端、422 校验 |
| HTTPException | 快速返回，`detail` 可为字符串或 dict |
| 领域异常 | 服务层不依赖框架，处理器负责呈现 |
| 全局处理器 | 按 MRO 匹配，兜底 `Exception` 返回 500 |
| 校验错误 | 只能通过 `RequestValidationError` 处理器改写 |
| 日志 | request_id + 结构化 + 脱敏 |
| 错误码 | 稳定、唯一、可读 |

下一篇进入测试，讲清 `TestClient`、pytest fixture、依赖覆盖、异步测试、数据库隔离和契约测试。
