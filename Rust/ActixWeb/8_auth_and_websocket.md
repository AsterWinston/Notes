鉴权回答"你是谁"，WebSocket 回答"如何保持长连接"。前者在 Actix Web 里有提取器和中间件两种落点，后者要回到 actor 模型：连接本身是一个 actor，心跳靠定时消息，广播靠中心 actor 转发。本篇把两条线各走一遍。

# 1. 鉴权方案怎么选

| 方案 | 存储位置 | 优点 | 缺点 |
|------|----------|------|------|
| JWT | 客户端 | 无状态、易横向扩展 | 无法主动失效、令牌易泄漏 |
| Cookie Session | 服务端/签名 Cookie | 可主动注销 | 需要共享存储、CSRF 防护 |
| Opaque Token | 服务端 | 可撤销、可审计 | 每次请求要查存储 |

★ **关键认知**：JWT 的"无状态"是双刃剑。它不需要查库就能验签，但也意味着**签发后无法单方面作废**，只能等过期。需要"立即踢下线"时，必须配合黑名单或改用会话。

```toml
[dependencies]
jsonwebtoken = "9"
argon2 = "0.5"
chrono = "0.4"
actix-session = { version = "0.10", features = ["cookie-session"] }
actix-web-actors = "4"
actix = "0.13"
```

# 2. JWT 的签发与校验

## 2.1 Claims 与签发

```rust
use jsonwebtoken::{encode, EncodingKey, Header};
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
pub struct Claims {
    pub sub: i64,   // 用户 ID
    pub iat: usize, // 签发时间
    pub exp: usize, // 过期时间
    pub role: String,
}

pub fn issue_token(secret: &[u8], user_id: i64, role: &str) -> Result<String, AppError> {
    let now = chrono::Utc::now().timestamp() as usize;
    let claims = Claims {
        sub: user_id,
        iat: now,
        exp: now + 3600, // 1 小时过期
        role: role.to_string(),
    };

    encode(&Header::default(), &claims, &EncodingKey::from_secret(secret))
        .map_err(|e| AppError::Internal(e.into()))
}
```

`Header::default()` 使用 HS256。密钥至少 32 字节，从环境变量读取，绝不写进代码或仓库。

## 2.2 校验

```rust
use jsonwebtoken::{decode, Algorithm, DecodingKey, Validation};

pub fn verify_token(secret: &[u8], token: &str) -> Result<Claims, AppError> {
    let mut validation = Validation::new(Algorithm::HS256);
    validation.validate_exp = true; // 显式开启过期校验

    let data = decode::<Claims>(token, &DecodingKey::from_secret(secret), &validation)
        .map_err(|_| AppError::Unauthorized)?; // 具体原因只进日志

    Ok(data.claims)
}
```

★ **关键认知**：一定要**指定算法**再校验。只信任令牌头里的 `alg` 会导致算法混淆攻击（攻击者把 RS256 改成 HS256，用公钥当 HMAC 密钥）。`Validation::new(Algorithm::HS256)` 固定了算法，才安全。⚠ 校验失败时对外统一返回 401，不要区分"签名错误""已过期"，这些细节只进日志。

## 2.3 登录时签发

登录接口先用 Argon2 校验密码哈希（`PasswordHash::new` + `Argon2::default().verify_password`），失败统一返回 401，成功再调用 `issue_token` 签发令牌。**绝不存明文密码**，也不要把"用户不存在"和"密码错误"区分返回。

# 3. 鉴权提取器

```rust
use actix_web::{dev::Payload, error, web, Error, FromRequest, HttpRequest};
use std::future::{ready, Ready};

pub struct AuthUser {
    pub id: i64,
    pub role: String,
}

impl FromRequest for AuthUser {
    type Error = Error;
    type Future = Ready<Result<Self, Self::Error>>;

    fn from_request(req: &HttpRequest, _payload: &mut Payload) -> Self::Future {
        let secret = req.app_data::<web::Data<AppState>>()
            .map(|s| s.jwt_secret.clone());

        let token = req.headers().get("Authorization")
            .and_then(|v| v.to_str().ok())
            .and_then(|v| v.strip_prefix("Bearer "));

        match (secret, token) {
            (Some(secret), Some(token)) => match verify_token(&secret, token) {
                Ok(c) => ready(Ok(AuthUser { id: c.sub, role: c.role })),
                Err(_) => ready(Err(error::ErrorUnauthorized("令牌无效"))),
            },
            _ => ready(Err(error::ErrorUnauthorized("缺少令牌"))),
        }
    }
}
```

★ **关键认知**：把鉴权做成提取器，函数签名即文档。看到 `user: AuthUser` 就知道这个接口需要登录；漏写参数就不会有鉴权，也不会出现"中间件挂错 scope"的静默漏洞。中间件适合整组路由统一拦截，提取器适合把用户信息带进 Handler，两者可组合，但都调用同一个 `verify_token`。需要角色校验时，在 Handler 里判断 `user.role`，不满足返回 `AppError::Forbidden`。

# 4. 会话方案

不需要 JWT 无状态特性时，`actix-session` 更简单：`SessionMiddleware::new(CookieSessionStore::default(), Key::from(&[0u8; 64]))` 挂到 `App`，Handler 里用 `session.insert("user_id", 42i64)?` 写、`session.get::<i64>("user_id")?` 读、`session.purge()` 注销。

⚠ Cookie 会话需要 CSRF 防护（`SameSite`、CSRF token）。`CookieSessionStore` 的 Cookie 有 4KB 上限，别往里塞大对象。

# 5. WebSocket 基础

## 5.1 握手入口

```rust
use actix_web::{web, Error, HttpRequest, HttpResponse};
use actix_web_actors::ws;

pub async fn ws_route(req: HttpRequest, stream: web::Payload) -> Result<HttpResponse, Error> {
    ws::start(EchoSession, &req, stream) // EchoSession 实现 Actor + StreamHandler
}
```

连接本身是一个 actor：`impl Actor for EchoSession { type Context = ws::WebsocketContext<Self>; }`，收帧实现 `StreamHandler<Result<ws::Message, ws::ProtocolError>>`，在 `handle` 里按 `Text`、`Ping`、`Close` 分支处理，主动发消息用 `ctx.text()`，停止用 `ctx.stop()`。

★ **关键认知**：WebSocket 连接在 Actix 里就是 actor，`WebsocketContext` 是它的执行上下文。不需要手写读写循环。浏览器发 WebSocket 不能自定义请求头，握手阶段通常从查询参数取令牌，校验通过后再 `ws::start`。真实项目里 `ws_route` 还要提取 `web::Data<Addr<ChatServer>>`，用它构造 `ChatSession`。⚠ 令牌放 URL 会被访问日志、代理、浏览器历史记录捕获，生产建议用一次性短期票据换取连接。

# 6. 心跳

```rust
use std::time::{Duration, Instant};

const HEARTBEAT_INTERVAL: Duration = Duration::from_secs(5);
const CLIENT_TIMEOUT: Duration = Duration::from_secs(10);

struct ChatSession {
    hb: Instant,
    user_id: i64,
    id: usize,
    server: actix::Addr<ChatServer>,
}

impl ChatSession {
    fn start_heartbeat(&self, ctx: &mut ws::WebsocketContext<Self>) {
        ctx.run_interval(HEARTBEAT_INTERVAL, |act, ctx| {
            if Instant::now().duration_since(act.hb) > CLIENT_TIMEOUT {
                ctx.stop(); // 超时断开，释放连接
                return;
            }
            ctx.ping(b""); // 发送 Ping，客户端应回 Pong
        });
    }
}
```

在 `StreamHandler` 里刷新 `hb`：收到 `Ping` 时 `ctx.pong`，收到 `Pong` 或 `Text` 时 `self.hb = Instant::now()`。任何入站消息都算活跃。⚠ 只在 `started` 里 `run_interval` 一次即可，重复调用会注册多个定时器。令牌放 URL 会被日志捕获，生产建议用一次性短期票据换取连接。

# 7. 广播

## 7.1 中心 actor

```rust
use actix::prelude::*;
use std::collections::HashMap;

#[derive(Message, Clone)]
#[rtype(result = "()")]
pub struct Broadcast(pub String);

#[derive(Message)]
#[rtype(result = "usize")]
pub struct Connect { pub addr: Recipient<Broadcast> }

#[derive(Message)]
#[rtype(result = "()")]
pub struct Disconnect { pub id: usize }

#[derive(Default)]
pub struct ChatServer {
    sessions: HashMap<usize, Recipient<Broadcast>>,
    next_id: usize,
}

impl Actor for ChatServer { type Context = Context<Self>; }

impl Handler<Connect> for ChatServer {
    type Result = usize;
    fn handle(&mut self, msg: Connect, _: &mut Context<Self>) -> usize {
        let id = self.next_id;
        self.next_id += 1;
        self.sessions.insert(id, msg.addr);
        id
    }
}

impl Handler<Disconnect> for ChatServer {
    type Result = ();
    fn handle(&mut self, msg: Disconnect, _: &mut Context<Self>) {
        self.sessions.remove(&msg.id);
    }
}

impl Handler<Broadcast> for ChatServer {
    type Result = ();
    fn handle(&mut self, msg: Broadcast, _: &mut Context<Self>) {
        for addr in self.sessions.values() {
            let _ = addr.do_send(msg.clone()); // 发送失败说明对端已断开
        }
    }
}
```

## 7.2 注册、回写与清理

```rust
impl actix::Actor for ChatSession {
    type Context = ws::WebsocketContext<Self>;

    fn started(&mut self, ctx: &mut Self::Context) {
        self.start_heartbeat(ctx); // 启动心跳，见上一节
        let addr = ctx.address();
        self.server.send(Connect { addr: addr.recipient() })
            .into_actor(self)
            .then(|res, act, ctx| {
                match res {
                    Ok(id) => act.id = id,
                    Err(_) => ctx.stop(), // 注册失败，直接断开
                }
                fut::ready(())
            })
            .wait(ctx);
    }

    fn stopped(&mut self, _ctx: &mut Self::Context) {
        self.server.do_send(Disconnect { id: self.id }); // 必须清理，否则泄漏
    }
}

impl Handler<Broadcast> for ChatSession {
    type Result = ();
    fn handle(&mut self, msg: Broadcast, ctx: &mut Self::Context) {
        ctx.text(msg.0);
    }
}
```

启动时把 actor 地址注入应用：

```rust
let chat_server = ChatServer::default().start();
let chat_data = web::Data::new(chat_server);

HttpServer::new(move || {
    App::new().app_data(chat_data.clone()).route("/ws", web::get().to(ws_route))
})
```

★ **关键认知**：`Addr<ChatServer>` 是 actor 的地址，克隆它不会复制邮箱，而是得到指向同一 actor 的句柄。所有连接通过 `Recipient<Broadcast>` 注册到中心，中心负责扇出。⚠ 广播是 O(n) 操作，连接数上万时要考虑分房间、批量发送、背压；无节制地 `do_send` 会让 actor 邮箱堆积，最终内存暴涨。

# 8. 常见坑

| 问题 | 后果 | 正确做法 |
|------|------|----------|
| JWT 密钥写死在代码里 | 泄漏后可任意伪造令牌 | 环境变量 + 定期轮换 |
| 不校验 `exp` | 过期令牌永久有效 | `validate_exp = true` |
| 不固定算法 | 算法混淆攻击 | `Validation::new(Algorithm::HS256)` |
| 密码明文或弱哈希 | 拖库即泄露 | Argon2id |
| WebSocket 不做心跳 | 僵尸连接占内存 | Ping/Pong + 超时 |
| `run_interval` 重复注册 | 心跳风暴 | 只在 `started` 注册一次 |

# 9. 动手练习

1. 实现注册与登录，密码用 Argon2 哈希，登录成功返回 1 小时有效的 JWT。
2. 写一个 `AuthUser` 提取器，让 `/me` 返回当前用户 ID 与角色，令牌无效返回 401。
3. 在 Echo WebSocket 基础上加心跳与广播，用两个浏览器标签页验证消息互达。

验收标准：过期令牌被拒绝，关闭标签页后中心 actor 的会话数下降，心跳超时连接自动断开。

# 10. 小结

| 概念 | 要点 |
|------|------|
| JWT | 无状态、验签即信任，无法主动作废 |
| Claims | `sub`、`exp`、`iat`、自定义角色 |
| 算法固定 | 必须指定 HS256，防算法混淆 |
| 提取器 | `AuthUser` 把鉴权变成函数签名 |
| Session | 可主动注销，注意 CSRF 与 4KB 限制 |
| WebSocket | 连接即 actor，`StreamHandler` 收帧 |
| 心跳 | Ping/Pong + 超时 `ctx.stop()` |
| 广播 | 中心 actor + `Recipient`，断开要清理 |

下一篇进入测试与部署，讲清楚 `actix_web::test` 的测试客户端、如何给 Handler 注入测试状态、配置与日志的分环境管理、优雅关闭，以及把服务打包上线的注意事项。
