鉴权解决"你是谁"，WebSocket 解决"连接不断"。这两件事在 Axum 里都不需要额外框架：JWT 用 `jsonwebtoken` 签发校验，中间件把身份写进请求扩展，WebSocket 用原生 `WebSocketUpgrade` 升级，广播用 Tokio 的 `broadcast` 通道。本篇把它们串成一条可运行的链路。

# 1. 鉴权的两种形态

## 1.1 会话与 JWT

| 维度 | 会话（Session） | JWT |
|------|-----------------|-----|
| 状态存放 | 服务端 | 客户端令牌 |
| 撤销 | 删会话即可 | 需要黑名单或短过期 |
| 扩展性 | 多实例要共享存储 | 天然无状态 |
| 适合 | 后台管理、同域 Web | API、移动端、跨域 |

## 1.2 选择

后台管理系统用会话更简单：服务端能随时踢人，权限变更立即生效。开放 API、移动端、微服务之间用 JWT 更合适：不依赖共享存储，网关也能校验。

★ **关键认知**：JWT 的"无状态"是优点也是缺点。签发后服务端无法主动失效，除非维护黑名单或把过期时间压得很短。⚠ 不要因为"听起来现代"就选 JWT；需要即时撤销的场景，会话或"短 JWT + 刷新令牌"更稳妥。

# 2. JWT

## 2.1 签发

依赖 `jsonwebtoken = "9"` 与 `chrono`。

```rust
use jsonwebtoken::{encode, EncodingKey, Header};
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Claims {
    pub sub: String, // 用户 ID
    pub iat: usize,  // 签发时间（Unix 秒）
    pub exp: usize,  // 过期时间（Unix 秒）
}

pub fn issue(secret: &[u8], user_id: &str) -> Result<String, jsonwebtoken::errors::Error> {
    let now = chrono::Utc::now().timestamp() as usize;
    let claims = Claims {
        sub: user_id.to_string(),
        iat: now,
        exp: now + 3600, // 1 小时后过期
    };
    encode(&Header::default(), &claims, &EncodingKey::from_secret(secret))
}
```

## 2.2 校验

```rust
use jsonwebtoken::{decode, DecodingKey, Validation};

pub fn verify(secret: &[u8], token: &str) -> Result<Claims, jsonwebtoken::errors::Error> {
    let data = decode::<Claims>(
        token,
        &DecodingKey::from_secret(secret),
        &Validation::default(), // 默认校验 exp，算法为 HS256
    )?;
    Ok(data.claims)
}
```

★ **关键认知**：签发和校验必须使用**同一个算法和同一份密钥**。`Header::default()` 是 HS256，`Validation::default()` 也只接受 HS256；换算法时两边都要改，否则会出现"自己签的 token 自己验不过"。⚠ 密钥不要硬编码，长度至少 32 字节，生产环境从密钥管理服务注入。

## 2.3 Claims 设计

| 字段 | 作用 | 注意 |
|------|------|------|
| `sub` | 用户标识 | 不要放手机号、邮箱等敏感信息 |
| `exp` | 过期时间 | 必填，且不要太长 |
| `iat` | 签发时间 | 用于排查与刷新 |
| `jti` | 令牌唯一 ID | 做黑名单时需要 |
| `role` | 角色 | 权限变更后旧令牌仍有效，需配合短过期 |

JWT 的 payload 只是 Base64 编码，**不是加密**。任何人都能解出内容，所以不能放密码、密钥、完整身份证号。

# 3. 鉴权中间件

## 3.1 from_fn_with_state

```rust
use axum::{
    extract::{Request, State},
    http::StatusCode,
    middleware::Next,
    response::{IntoResponse, Response},
};

pub async fn auth(
    State(state): State<AppState>,
    mut request: Request,
    next: Next,
) -> Response {
    let token = request
        .headers()
        .get(axum::http::header::AUTHORIZATION)
        .and_then(|value| value.to_str().ok())
        .and_then(|value| value.strip_prefix("Bearer "));

    let Some(claims) = token.and_then(|t| verify(&state.jwt_secret, t).ok()) else {
        return (StatusCode::UNAUTHORIZED, "未登录").into_response();
    };

    request.extensions_mut().insert(claims); // 传给后续 Handler
    next.run(request).await
}
```

## 3.2 提取器方案

中间件负责"拦"，提取器负责"取"：

```rust
use axum::extract::FromRequestParts;
use axum::http::{request::Parts, StatusCode};

pub struct CurrentUser {
    pub id: String,
}

impl<S> FromRequestParts<S> for CurrentUser
where
    S: Send + Sync,
{
    type Rejection = (StatusCode, &'static str);

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        parts
            .extensions
            .get::<Claims>()
            .map(|claims| CurrentUser {
                id: claims.sub.clone(),
            })
            .ok_or((StatusCode::UNAUTHORIZED, "未登录"))
    }
}
```

## 3.3 保护路由

```rust
use axum::{middleware, routing::{get, post}, Router};

let public = Router::new().route("/login", post(login));

let protected = Router::new()
    .route("/me", get(me))
    .route_layer(middleware::from_fn_with_state(state.clone(), auth));

let app = public.merge(protected).with_state(state);
```

★ **关键认知**：用 `route_layer` 而不是 `layer`，鉴权只对已注册的路由生效，未匹配的请求仍然返回 404 而不是 401。⚠ 顺序上先 `route_layer` 再合并，不要先 `with_state` 再挂需要状态的中间件，否则类型对不上。

# 4. WebSocket

## 4.1 升级握手

```rust
use axum::{
    extract::{
        ws::{Message, WebSocket, WebSocketUpgrade},
        State,
    },
    response::IntoResponse,
};

async fn ws_handler(
    ws: WebSocketUpgrade,
    State(state): State<AppState>,
) -> impl IntoResponse {
    ws.on_upgrade(move |socket| handle_socket(socket, state))
}
```

`WebSocketUpgrade` 会校验 `Upgrade`、`Connection`、`Sec-WebSocket-Key` 等头，失败直接返回 400/426。`on_upgrade` 里的闭包在握手完成后执行。

## 4.2 心跳与收发

```rust
use std::time::Duration;

let mut tick = tokio::time::interval(Duration::from_secs(30));

loop {
    tokio::select! {
        incoming = socket.recv() => {
            match incoming {
                Some(Ok(Message::Text(text))) => {
                    let _ = socket.send(Message::Text(text)).await; // 先做回显
                }
                Some(Ok(Message::Close(_))) | None => break,
                Some(Err(_)) => break,
                _ => {}
            }
        }
        _ = tick.tick() => {
            // 定期 Ping，客户端/中间层不回 Pong 就说明连接可能已死
            if socket.send(Message::Ping(Vec::new().into())).await.is_err() {
                break;
            }
        }
    }
}
```

★ **关键认知**：`WebSocket::recv()` 返回 `Option<Result<Message, Error>>`。`None` 表示连接关闭，`Err` 表示协议或 IO 错误，两者都应该结束循环。⚠ 0.8 的 `Message::Text` 包的是 `Utf8Bytes`，不再是 `String`；从旧代码迁移时，`Message::Text(s)` 的构造需要 `s.into()`。

反向代理和负载均衡器通常会在 60 秒左右断开空闲连接。心跳让连接保持活跃，也能及时发现半开连接。⚠ Axum 会自动回复对端发来的 Ping，但不会替你定期发 Ping；心跳要么应用层做，要么依赖底层 TCP keepalive，后者粒度太粗。

## 4.3 广播

```rust
use tokio::sync::broadcast;

#[derive(Clone)]
struct AppState {
    jwt_secret: std::sync::Arc<Vec<u8>>,
    tx: broadcast::Sender<String>,
}

let (tx, _rx) = broadcast::channel(100); // _rx 要活到进程结束
let state = AppState { jwt_secret: std::sync::Arc::new(secret), tx };
```

每个连接订阅一份接收端：

```rust
async fn handle_socket(mut socket: WebSocket, state: AppState) {
    let mut rx = state.tx.subscribe();

    loop {
        tokio::select! {
            incoming = socket.recv() => {
                match incoming {
                    Some(Ok(Message::Text(text))) => {
                        let _ = state.tx.send(text.to_string()); // 广播给所有连接
                    }
                    Some(Ok(Message::Close(_))) | None => break,
                    Some(Err(_)) => break,
                    _ => {}
                }
            }
            broadcast = rx.recv() => {
                match broadcast {
                    Ok(text) => {
                        if socket.send(Message::Text(text.into())).await.is_err() {
                            break;
                        }
                    }
                    Err(broadcast::error::RecvError::Lagged(n)) => {
                        tracing::warn!(missed = n, "客户端消费过慢，丢弃了部分消息");
                        continue;
                    }
                    Err(broadcast::error::RecvError::Closed) => break,
                }
            }
        }
    }
}
```

★ **关键认知**：`broadcast` 是"每个订阅者都收到一份"的多播，不是队列。慢客户端会触发 `Lagged`，旧消息被丢弃；如果业务不能丢消息，要用持久化队列或为每个连接单独缓冲。⚠ 频道容量是背压的上限，容量太小会频繁 `Lagged`，太大则占内存。

# 5. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| 已登录仍 401 | 中间件没把 `Claims` 放进 extensions | 检查 `extensions_mut().insert` |
| token 永远验不过 | 签发/校验算法或密钥不一致 | 对齐 HS256 与密钥 |
| 404 变 401 | 鉴权用了 `.layer` | 改用 `.route_layer` |
| WebSocket 握手 400 | 反向代理没转发 Upgrade 头 | 配置代理转发 `Upgrade` / `Connection` |
| 消息发不出去 | `broadcast` 没有接收者 | 保留 `_rx` 或至少一个订阅 |
| 客户端频繁重连 | 没有心跳，空闲被断开 | 定期 Ping 或应用层心跳 |
| 旧 token 权限不生效 | JWT 无状态 | 短过期 + 刷新令牌或黑名单 |

⚠ 鉴权失败不要返回具体原因（"用户不存在"还是"密码错误"），统一 401 可以避免账号枚举。

# 6. 动手练习

1. 实现 `/login` 签发 JWT、`/me` 校验 JWT，要求未带 token 返回 401，过期 token 也返回 401。
2. 把用户信息通过 `Extension<Claims>` 传给 `CurrentUser` 提取器，确认 Handler 不需要手动解析请求头。
3. 实现 `/ws` 聊天室：一个客户端发消息，所有客户端都能收到；加上 30 秒 Ping 心跳，观察代理空闲超时是否还会断开。

验收标准：登录、鉴权、WebSocket 广播、心跳四条链路都能用 `curl` 或浏览器控制台验证。

# 7. 小结

| 概念 | 要点 |
|------|------|
| JWT | 无状态令牌，payload 只编码不加密 |
| 签发/校验 | 算法与密钥必须一致，`exp` 必填 |
| 鉴权中间件 | 读 `Authorization`，写 `Extensions` |
| 提取器 | 从 extensions 取 `Claims`，Handler 只声明 `CurrentUser` |
| 保护路由 | `route_layer`，避免 404 变 401 |
| WebSocket | `WebSocketUpgrade` + `on_upgrade` |
| 心跳 | 定期 Ping，防止空闲连接被断开 |
| 广播 | `broadcast::channel`，注意 `Lagged` |

下一篇进入测试与部署，讲清楚如何用 `oneshot` 直接测 Router、如何构造 JSON 请求、测试数据库怎么隔离，以及日志、优雅关闭和容器部署的检查清单。
