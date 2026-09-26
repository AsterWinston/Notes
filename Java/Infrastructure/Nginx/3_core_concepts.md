
---

# 1. Master / Worker 进程模型

Nginx 启动后会生成两种进程：

```
          ┌─────────┐
          │  Master  │  （1个）—— 管理 Worker、读取配置、处理信号
          └────┬─────┘
    ┌──────────┼──────────┐
    ▼          ▼          ▼
┌───────┐ ┌───────┐ ┌───────┐
│Worker1│ │Worker2│ │Worker3│  （N个，通常 = CPU 核数）
└───────┘ └───────┘ └───────┘
```

| 进程 | 数量 | 职责 |
|------|------|------|
| **Master** | 1 个 | 读取并验证配置、管理工作进程、接收外部信号（`reload`/`stop` 等） |
| **Worker** | N 个 | 处理客户端连接与请求、网络 I/O 读写、与后端通信 |
| **Cache Manager** | 1 个 | 管理缓存（过期检查、缓存大小控制） |
| **Cache Loader** | 1 个（启动时） | 加载磁盘缓存信息到内存 |

> Master 以 **root** 运行（监听 80/443 端口），Worker 以普通用户（如 `nginx`）运行，提升安全性。

---

# 2. 事件驱动模型

## 2.1 对比传统服务器模型

| 模型 | 代表 | 原理 | 并发能力 | 内存消耗 |
|------|------|------|----------|----------|
| 多进程（prefork） | Apache (prefork) | 每个连接 fork 一个子进程 | 低 | 极高 |
| 多线程（worker） | Apache (worker) | 每个连接分配一个线程 | 中 | 中 |
| **事件驱动** | **Nginx** | 单线程异步处理所有连接 | 极高 | 极低 |

## 2.2 Nginx 事件驱动流程

```
1. Worker 进程在 listen 端口上等待事件（epoll/kqueue/select）
2. 新连接到达 → 事件通知 Worker
3. Worker 将连接加入事件循环，异步处理 I/O
4. 请求处理完毕 → 释放连接资源
5. 继续等待新事件
```

- **epoll**（Linux 2.6+）：最高效，无文件描述符数量限制
- **kqueue**（FreeBSD/macOS）：BSD 平台的 epoll 等价物
- **select / poll**：跨平台但效率低，连接数受限

---

# 3. 连接处理机制

## 3.1 Worker 竞争锁

```
多个 Worker 进程通过共享锁（accept_mutex）竞争新连接：
  - accept_mutex on   → 同一时刻只有一个 Worker 接收新连接（默认开启）
  - accept_mutex off  → 所有 Worker 同时接收（可能惊群，Linux 3.9+ 内核已解决）
```

```nginx
events {
    accept_mutex on;    # 互斥锁（默认开启）
}
```

## 3.2 连接数限制

```nginx
events {
    worker_connections 10240;   # 单个 Worker 最大并发连接数
    multi_accept on;            # 一次接收所有新连接（非逐一）
    use epoll;                  # 指定事件模型
}
```

**最大并发连接数计算公式**：

```
最大并发连接数 = worker_processes × worker_connections

作为反向代理时：
最大有效并发 = worker_processes × worker_connections / 2
（因为每个客户端连接对应一条到后端的连接）
```

---

# 4. 信号控制机制

Nginx 通过向 Master 进程发送信号来控制行为：

| 信号 | 命令等价 | 作用 |
|------|----------|------|
| `TERM / INT` | `nginx -s stop` | 快速关闭 |
| `QUIT` | `nginx -s quit` | 优雅关闭（处理完当前请求） |
| `HUP` | `nginx -s reload` | 重新加载配置（平滑重启） |
| `USR1` | — | 重新打开日志文件（日志切割用） |
| `USR2` | — | 平滑升级可执行文件 |
| `WINCH` | — | 优雅关闭 Worker（配合 USR2 升级用） |

```bash
# 向 Master 发送 HUP 信号平滑重载
kill -HUP $(cat /var/run/nginx.pid)
```

---

# 5. 平滑升级流程

```
1. 准备新版 Nginx 二进制文件
2. kill -USR2 <旧master_pid>     → 启动新 Master（新旧并存）
3. kill -WINCH <旧master_pid>    → 优雅关闭旧 Worker
4. kill -QUIT <旧master_pid>     → 完全关闭旧 Master
```

---

# 6. 请求处理阶段

Nginx 处理一个 HTTP 请求依次经过 **11 个阶段**：

| 阶段 | 名称 | 常用模块 |
|------|------|----------|
| 1 | `NGX_HTTP_POST_READ_PHASE` | realip |
| 2 | `NGX_HTTP_SERVER_REWRITE_PHASE` | rewrite（server 块内） |
| 3 | `NGX_HTTP_FIND_CONFIG_PHASE` | location 匹配 |
| 4 | `NGX_HTTP_REWRITE_PHASE` | rewrite（location 块内） |
| 5 | `NGX_HTTP_POST_REWRITE_PHASE` | 防止 rewrite 死循环 |
| 6 | `NGX_HTTP_PREACCESS_PHASE` | limit_req, limit_conn |
| 7 | `NGX_HTTP_ACCESS_PHASE` | allow/deny, auth_basic |
| 8 | `NGX_HTTP_POST_ACCESS_PHASE` | satisfy 指令 |
| 9 | `NGX_HTTP_PRECONTENT_PHASE` | try_files |
| 10 | `NGX_HTTP_CONTENT_PHASE` | index, static, proxy_pass, fastcgi 等（内容处理） |
| 11 | `NGX_HTTP_LOG_PHASE` | access_log |

> **理解请求处理阶段**是深入掌握 Nginx 配置的关键。同一阶段内的模块按配置顺序执行，不同阶段则严格按阶段顺序执行。
