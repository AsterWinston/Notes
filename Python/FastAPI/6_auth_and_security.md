接口一旦上线，第一道门槛就是"你是谁、你能做什么"。FastAPI 的安全工具链围绕 OAuth2 和依赖注入展开：认证负责识别身份，授权负责限制动作，两者都不能省。本篇从密码哈希讲到 JWT、权限依赖、CORS、限流和密钥管理，给出一套能落地的组合。

# 1. 认证与授权的基本模型

## 1.1 两个概念

| 概念 | 回答的问题 | 手段 |
|------|-----------|------|
| 认证（Authentication） | 你是谁 | 密码、Token、OAuth2 |
| 授权（Authorization） | 你能做什么 | 角色、权限、作用域 |

```text
请求 ──▶ 认证：解析 Token / 校验密码 ──▶ 授权：检查角色/权限 ──▶ 业务
              │ 失败 401                     │ 失败 403
              ▼                              ▼
          未登录响应                     无权限响应
```

★ **关键认知**：**401 与 403 不能混用**。401 表示"没认证或认证失效"，客户端应去登录；403 表示"认证了但没权限"，再登录也没用。返回错状态码会让前端做错误的跳转。

本篇以 **OAuth2 密码流 + JWT** 为主线，这是 FastAPI 官方示例的默认组合。

# 2. 密码哈希

## 2.1 为什么不能存明文

数据库泄露时，明文密码等于把所有用户的账号交出去。哈希不可逆，泄露后攻击者只能拿到摘要。但普通哈希（如 SHA-256）太快，容易被彩虹表和暴力破解，所以要用**慢哈希**。

| 算法 | 特点 | 建议 |
|------|------|------|
| MD5 / SHA-1 | 已不安全 | 禁用 |
| SHA-256 | 快，易被爆破 | 不用于密码 |
| bcrypt | 慢，带盐 | 可用 |
| Argon2 | 现代推荐 | 首选 |

## 2.2 使用 pwdlib

```bash
pip install "pwdlib[argon2]"
```

```python
from pwdlib import PasswordHash

password_hash = PasswordHash.recommended()   # 默认 Argon2

def hash_password(plain: str) -> str:
    return password_hash.hash(plain)

def verify_password(plain: str, hashed: str) -> bool:
    return password_hash.verify(plain, hashed)
```

存放时只存 `hashed`，登录时用 `verify` 比较。**永远不要自己拼接盐或比较哈希字符串**。

⚠ 老教程常用 `passlib[bcrypt]`，但 passlib 维护停滞，且与新版 bcrypt 存在兼容告警。新项目直接用 `pwdlib`；已有项目可保留，但要锁定版本。

# 3. JWT 基础

## 3.1 结构

JWT 由三段组成，用 `.` 分隔：

```text
header.payload.signature
  │       │         │
  │       │         └─ 用密钥对前两段签名
  │       └─ 载荷：sub、exp、role 等
  └─ 算法声明：{"alg": "HS256", "typ": "JWT"}
```

载荷是 **Base64URL 编码，不是加密**。任何人都能解开看内容，所以**不要放密码、身份证号等敏感信息**。

## 3.2 签发与校验

```bash
pip install pyjwt
```

```python
from datetime import datetime, timedelta, timezone
import jwt

SECRET_KEY = "从环境变量读取，不要硬编码"
ALGORITHM = "HS256"

def create_access_token(subject: str, expires_minutes: int = 30) -> str:
    now = datetime.now(timezone.utc)
    payload = {
        "sub": subject,                          # 主体，通常是用户 ID
        "iat": now,                              # 签发时间
        "exp": now + timedelta(minutes=expires_minutes),
    }
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

def decode_token(token: str) -> dict:
    return jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
```

`exp` 会被 `jwt.decode` 自动校验，过期抛 `ExpiredSignatureError`；签名不对抛 `InvalidTokenError`。

★ **关键认知**：JWT 的"无状态"意味着**服务端不保存会话**，好处是水平扩展简单，坏处是**无法主动让某个 Token 失效**。要支持登出/踢人，得引入黑名单或短过期 + 刷新令牌。

# 4. OAuth2 密码流

## 4.1 认证依赖

```python
from typing import Annotated
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/token")

async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],
) -> dict:
    credentials_error = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="认证失败",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = decode_token(token)
    except jwt.InvalidTokenError:
        raise credentials_error

    user_id = payload.get("sub")
    if user_id is None:
        raise credentials_error

    user = await load_user(user_id)     # 伪代码：查库
    if user is None:
        raise credentials_error
    return user

CurrentUser = Annotated[dict, Depends(get_current_user)]
```

`OAuth2PasswordBearer` 做两件事：从 `Authorization: Bearer <token>` 里取 Token；在文档里生成"Authorize"按钮。

## 4.2 登录接口

```python
from fastapi import APIRouter
from fastapi.security import OAuth2PasswordRequestForm

router = APIRouter(prefix="/auth", tags=["auth"])

@router.post("/token")
async def login(
    form: Annotated[OAuth2PasswordRequestForm, Depends()],
):
    user = await find_user_by_name(form.username)
    if user is None or not verify_password(form.password, user["password_hash"]):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="用户名或密码错误",
        )
    token = create_access_token(subject=str(user["id"]))
    return {"access_token": token, "token_type": "bearer"}
```

`OAuth2PasswordRequestForm` 接收 `application/x-www-form-urlencoded` 的 `username`、`password` 字段，这是 OAuth2 规范要求，不是 JSON。

⚠ **登录失败的错误信息要模糊**。不要区分"用户不存在"和"密码错误"，否则攻击者可以枚举用户名。

# 5. 权限控制

## 5.1 基于角色的检查

```python
def require_role(role: str):
    async def checker(user: CurrentUser) -> dict:
        if role not in user.get("roles", []):
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="权限不足",
            )
        return user
    return checker

AdminUser = Annotated[dict, Depends(require_role("admin"))]

@router.delete("/users/{user_id}")
async def delete_user(user_id: int, admin: AdminUser):
    return {"deleted": user_id}
```

`require_role("admin")` 返回一个依赖函数，把配置闭包进去。这是第 4 篇"类依赖"之外的另一种常见写法。

★ **关键认知**：权限检查**必须放在服务端**。前端隐藏按钮只是体验优化，不能当作安全措施。任何"只有管理员能看到这个接口"的假设，都要在服务端用依赖再确认一次。

# 6. CORS

## 6.1 为什么需要

浏览器同源策略会拦截跨域请求。前端跑在 `http://localhost:5173`，后端在 `http://localhost:8000`，就是跨域。

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],   # 生产环境写具体域名
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

## 6.2 安全边界

⚠ **不要把 `allow_origins=["*"]` 和 `allow_credentials=True` 一起用**。规范禁止这种组合，浏览器会直接拒绝响应。需要携带 Cookie 时，必须列出具体来源。

# 7. 限流

## 7.1 策略

限流按"键 + 窗口 + 上限"实现，键可以是 IP、用户 ID 或接口名。单机内存计数简单，但多 worker 不共享，生产用 Redis 的 `INCR` + `EXPIRE`，或直接在网关层限流。登录、发短信等敏感接口要单独收紧。

# 8. 密钥管理

## 8.1 不要硬编码

```python
# 错误示范
SECRET_KEY = "my-secret-key-123"
```

密钥进 Git 历史后，即使删除也留在提交里。正确做法：

```bash
# .env（加入 .gitignore）
SECRET_KEY=生成的随机值
DATABASE_URL=postgresql+asyncpg://...
```

```bash
python -c "import secrets; print(secrets.token_urlsafe(48))"
```

## 8.2 用 pydantic-settings 读取

```bash
pip install pydantic-settings
```

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    secret_key: str
    database_url: str
    access_token_minutes: int = 30

settings = Settings()      # 缺少必填项时启动即失败
```

启动即校验，比运行到一半才发现配置缺失好得多。

## 8.3 轮换与最小权限

密钥要定期轮换、按环境隔离；数据库账号只给必要权限；打印配置时屏蔽敏感字段。

# 9. 常见错误

| 问题 | 后果 | 正确做法 |
|------|------|---------|
| 存明文密码 | 泄露即失守 | Argon2/bcrypt 哈希 |
| JWT 放敏感信息 | 载荷可被解码 | 只放 ID、角色等非敏感数据 |
| access token 有效期过长 | 泄露窗口大 | 15-30 分钟 + refresh |
| 401 和 403 混用 | 前端跳转错误 | 未认证 401，无权限 403 |
| CORS 通配 + 凭证 | 浏览器拒绝 | 明确列出源 |
| 密钥硬编码 | 进 Git 历史 | 环境变量 + `.gitignore` |
| 登录错误提示过细 | 用户名可枚举 | 统一"用户名或密码错误" |

# 10. 练习

1. 用 `pwdlib` 实现注册与登录：注册时哈希密码并存库，登录时校验并签发 JWT，Token 里带 `sub` 和 `exp`。
2. 写一个 `require_role("admin")` 依赖，保护删除接口；用普通用户和管理员各请求一次，确认分别得到 403 和 200。
3. 配置 CORS 只允许 `http://localhost:5173`，用浏览器或 `curl -H "Origin: ..."` 验证预检请求的响应头。

# 11. 小结

| 主题 | 要点 |
|------|------|
| 认证/授权 | 401 未认证，403 无权限 |
| 密码 | 慢哈希，Argon2 优先，只存摘要 |
| JWT | 三段结构，载荷可解码，签名防篡改 |
| OAuth2 | 密码流用表单取 Token，Bearer 头传递 |
| 权限 | 依赖封装角色/作用域检查 |
| CORS | 明确来源，通配与凭证不可同用 |
| 限流 | 内存仅演示，生产用 Redis |
| 密钥 | 环境变量 + 启动校验 + 定期轮换 |

下一篇进入错误处理，讲清 `HTTPException`、全局异常处理器、统一错误结构、校验错误格式化和日志记录。
