会写 TOML 语法，不等于会管配置。真实项目里配置通常有默认值、环境差异、本地覆盖和密钥注入四层来源，还要保证读出来的值类型正确、缺失时能快速失败。本篇讲一套可以落地到 Python、Rust 项目的 TOML 配置组织方式。

# 1. 配置为什么要分层

## 1.1 五个层级与优先级

把所有东西塞进一个 `config.toml`，很快会遇到三个问题：生产参数不敢改、本地调试要手改、密钥容易被提交。分层让每一层只负责一件事：

```text
优先级从高到低：

  命令行参数        --port 9000
       │
       ▼
  环境变量          APP_SERVER__PORT=9000
       │
       ▼
  本地覆盖          config/local.toml（不进仓库）
       │
       ▼
  环境配置          config/production.toml
       │
       ▼
  默认配置          config/default.toml
```

★ **关键认知**：配置系统的核心不是"支持多少种来源"，而是**优先级明确且可预测**。同样一个 `port`，如果两个来源冲突时行为不确定，排查成本会远高于多写几行文档。⚠ 应用启动时最好把最终生效的配置（脱敏后）打印一次。

优先级从高到低是：命令行 > 环境变量 > `local.toml` > `<env>.toml` > `default.toml`。把这套规则写进 README，并在启动日志里输出当前环境和实际加载的文件列表，能省掉大量"为什么线上端口不对"的排查。

# 2. 目录与命名

## 2.1 推荐的目录结构

```text
project/
├── config/
│   ├── default.toml          # 默认值，必须提交
│   ├── production.toml       # 生产环境差异，必须提交
│   ├── local.toml            # 个人覆盖，加入 .gitignore
│   └── local.toml.example    # 示例，提交给新同事参考
└── .gitignore
```

把 `config/local.toml`、`*.secret.toml`、`.env` 写进 `.gitignore`。

★ **关键认知**：`default.toml` 应该**能独立跑起来**：一份默认配置加上必需的环境变量，就能在开发机启动。环境文件只写差异，不复制整份默认配置，否则改一个默认值要改五份文件。

## 2.2 键的命名

推荐用 `snake_case`，和 Python、Rust 字段命名一致；布尔键用 `is_` / `enable_` 前缀；**带单位的数值把单位写进键名**。

| 不推荐 | 推荐 | 原因 |
|--------|------|------|
| `timeout = 3000` | `timeout_ms = 3000` | 看不出是秒还是毫秒 |
| `size = 100` | `max_size_mb = 100` | 单位不明 |
| `DBHost` | `db_host` | 大小写混用易错 |
| `retry` | `max_retries` | 需要明确是次数还是开关 |
| `url` | `database_url` | 多个 url 时更清楚 |

★ **关键认知**：配置键是**跨语言契约**。TOML 里叫 `timeout_ms`，Rust 结构体里叫 `timeout_ms`，Python 字典里也读 `timeout_ms`，三处一致才不会在改名时漏掉一方。⚠ 不要用缩写堆砌，`db_conn_max` 不如 `database_max_connections` 直观。

## 2.3 按领域分组

表的划分按**业务领域**，不要按"类型"：

```toml
# 推荐：按领域
[server]
host = "0.0.0.0"
port = 8080

[database]
url = "postgres://localhost/app"
max_connections = 20

# 不推荐：按类型
[strings]
host = "0.0.0.0"

[numbers]
port = 8080
```

★ **关键认知**：按领域分组后，改数据库相关配置只需要看一个表；按类型分组则要在多个表之间跳转，而且新增字段时很难判断该放哪。⚠ 嵌套不要超过三层，超过时通常说明该拆成多个配置对象了。

# 3. 环境区分

## 3.1 默认配置 + 环境覆盖

`config/default.toml`：

```toml
[server]
host = "127.0.0.1"
port = 8080

[database]
url = "postgres://localhost/app"
max_connections = 5
```

`config/production.toml` 只写差异：`server.host = "0.0.0.0"`、`database.max_connections = 20`。合并后 `host` 变为 `0.0.0.0`，`port` 和 `url` 保持默认，`max_connections` 变为 20。

## 3.2 深度合并的规则

合并不是简单替换整个文件，而是递归合并表：

| 值类型 | 合并行为 |
|--------|---------|
| 标量（字符串、数字、布尔） | 高优先级直接替换 |
| 表 | 递归合并，键取并集 |
| 数组 | **整体替换**，不拼接 |
| 表数组 | 整体替换 |

```toml
# 默认配置 default.toml
[server]
allowed_hosts = ["localhost", "127.0.0.1"]
```

```toml
# 生产环境 production.toml
[server]
allowed_hosts = ["example.com"]
```

最终只有 `["example.com"]`。⚠ **关键认知**：想追加而不是替换时，要么在环境文件里写完整列表，要么在加载器里定义明确的追加语法，不能假设合并会帮你拼接。

## 3.3 环境变量覆盖

TOML 本身**没有环境变量插值**：

```toml
port = "${APP_PORT}"        # 解析结果就是字符串 "${APP_PORT}"
```

环境变量覆盖必须由**加载器**完成。常见约定是用双下划线表示层级：

```text
APP_SERVER__HOST=0.0.0.0         →  server.host
APP_SERVER__PORT=9000            →  server.port
APP_DATABASE__URL=postgres://... →  database.url
```

加载器遍历 `APP_` 前缀变量，按 `__` 拆分路径后写回嵌套字典，`set_nested` 需要自己实现。

★ **关键认知**：环境变量读进来永远是字符串，`APP_SERVER__PORT=9000` 得到的是 `"9000"`，不是整数 `9000`。⚠ 覆盖后必须重新做类型转换和校验，否则 `port` 会以字符串身份流进业务代码。

## 3.4 环境选择

用一个单独的环境变量决定加载哪份文件：

```python
env = os.environ.get("APP_ENV", "development")
if env not in {"development", "staging", "production"}:
    raise SystemExit(f"未知环境：{env}")
```

不要让代码根据主机名、当前目录去"猜"环境。显式配置比魔法推断可靠。

# 4. 密钥与敏感信息

## 4.1 什么不该进仓库

| 内容 | 是否进仓库 | 存放位置 |
|------|-----------|---------|
| 数据库密码 | 否 | 环境变量 / 密钥管理服务 |
| API Token | 否 | 环境变量 / 密钥管理服务 |
| 私钥、证书私钥 | 否 | 密钥管理服务 |
| 普通主机名、端口 | 是 | 环境 TOML |
| 日志级别、超时 | 是 | 默认 TOML |
| 本地个人路径 | 否 | `local.toml` |

★ **关键认知**：配置文件里应该存**密钥的引用**，而不是密钥本身。引用可以是环境变量名、密钥服务里的路径，或者一个占位符；真实值在启动时注入。⚠ 一旦密钥进过 Git 历史，即使后来删掉也仍然留在历史里，必须视为已泄露并轮换。

## 4.2 引用式写法

```toml
[database]
url = "postgres://app@db.internal:5432/orders"
password_env = "DB_PASSWORD"        # 只存环境变量名
```

读取时从 `os.environ` 取变量，取不到就抛错退出。也可以让加载器统一处理 `${VAR}` 占位符，但必须明确：**这是加载器的功能，不是 TOML 的语法**。TOML 解析器只会把它当普通字符串。

# 5. 默认值与可选性

## 5.1 默认值放在哪

| 位置 | 优点 | 缺点 |
|------|------|------|
| `default.toml` | 非代码人员也能改，集中可见 | 需要随项目分发 |
| 代码里的 `default` | 类型安全，测试方便 | 改默认值要发版 |
| 环境变量 | 容器友好 | 分散，难审计 |

推荐组合：**业务默认值放 `default.toml`，结构性默认值放代码**。例如 `port` 默认 8080 适合放 TOML；`workers` 默认等于 CPU 核数，适合放代码，因为它依赖运行时环境。

## 5.2 可选字段的表达

TOML 没有 `null`，可选字段的处理方式是"允许缺失"：

```toml
[cache]
enabled = false
# ttl_s 不写，表示使用程序默认值
```

```python
ttl = config.get("cache", {}).get("ttl_s", 60)
```

Rust 侧对应 `#[serde(default = "default_ttl")]`，缺失字段时调用默认函数。

★ **关键认知**：默认值也是**契约的一部分**。如果文档写"不填默认 60 秒"，代码就必须真的是 60；⚠ 默认值散落在多个 `unwrap_or` 里时，很容易出现文档、代码、测试三份不一致。

# 6. 可读性

## 6.1 注释与单位

同一张表内，把相关键放在一起；表之间空一行；每个不直观的键上方写一行注释，说明单位和取值影响。

```toml
[server]
host = "0.0.0.0"              # 监听所有网卡
port = 8080                   # 对外 HTTP 端口
read_timeout_ms = 5_000       # 单次读超时，超过即断开
max_connections = 1_000       # 同时保持的连接上限
```

不要用注释写"这是端口"，要写"对外暴露的 HTTP 端口，0 表示随机"。

## 6.2 数组与嵌套

短数组一行写完，长数组每行一个并保留尾逗号，例如 `tags = ["api", "orders"]`。超过三层的嵌套通常难读也难覆盖，建议把稳定的一组参数收成一张表。

# 7. 启动时校验

## 7.1 快速失败

配置错误应该在**进程启动阶段**暴露，而不是等第一个请求进来才崩：

```python
from pydantic import BaseModel, Field

class Server(BaseModel):
    host: str = "127.0.0.1"
    port: int = Field(default=8080, ge=1, le=65535)

server = Server(**config["server"])   # 类型错误、越界会立即抛出
```

## 7.2 校验清单

| 检查项 | 例子 |
|--------|------|
| 必填键存在 | 缺少 `database.url` 直接退出 |
| 类型正确 | `port` 必须是整数 |
| 取值范围 | `port` 在 1 到 65535 之间 |
| 枚举合法 | `logging.level` 只能是已知级别 |
| 路径可访问 | 日志目录存在且可写 |
| 跨字段一致 | `min_connections <= max_connections` |

★ **关键认知**：校验失败时输出**字段路径和期望值**，例如 `server.port: 期望 1-65535，实际 70000`，不要只说"配置错误"。⚠ 不要把密钥原值打进日志。

# 8. 常见错误

| 错误 | 后果 | 修复 |
|------|------|------|
| 密钥写进 `default.toml` | 泄露到 Git 历史 | 改用环境变量或密钥服务 |
| 数组覆盖当成合并 | 线上白名单变短 | 环境文件写完整列表 |
| 环境变量覆盖后不转类型 | `port` 变成字符串 | 覆盖后重新校验 |
| 默认值散落在代码各处 | 文档与行为不一致 | 集中到 `default.toml` |
| 配置文件按类型分组 | 难以定位字段 | 按业务领域分组 |
| 启动时不校验 | 运行到一半才报错 | 启动即校验并退出 |

# 9. 动手练习

1. 为一个小服务设计 `default.toml`、`production.toml`、`local.toml.example`，要求默认配置能直接启动，生产只覆盖 host 和连接数。

2. 写一个 Python 加载器，按 `default -> $APP_ENV -> local` 的顺序深度合并，并支持 `APP_SERVER__PORT` 形式的环境变量覆盖。

3. 找出一份你项目里的配置，检查是否有密钥、单位不明、数组覆盖这三类问题，并给出修改方案。

# 10. 小结

| 主题 | 要点 |
|------|------|
| 分层 | 命令行 > 环境变量 > 本地 > 环境文件 > 默认 |
| 目录 | `default.toml` 提交，`local.toml` 忽略 |
| 命名 | `snake_case`，单位写进键名 |
| 合并 | 表递归合并，数组整体替换 |
| 环境变量 | TOML 不插值，由加载器转换类型 |
| 密钥 | 只存引用，不存值 |
| 默认值 | 集中、可测试、与文档一致 |
| 校验 | 启动即校验，失败快速退出 |

下一篇进入格式对比与工具链，讲清楚 TOML 和 YAML、JSON、INI 各自适合什么场景，以及 `taplo` 格式化、schema 校验和编辑器支持怎么配。
