# 1. 初始Redis

## 1.1 认识NoSQL

- 定义：Not Only SQL，用于高并发、高可用、海量数据场景的非关系型数据库。
- 典型特征：
	- 弱一致或最终一致性
	- 数据模型多样：Key-Value、文档、列族、图
	- 易扩展、读写吞吐高
- 适用场景：缓存、会话、排行榜、计数器、消息队列、实时推荐等。

### 1.1.1 SQL vs NoSQL 对比

| 维度 | SQL (关系型数据库) | NoSQL (非关系型数据库) |
|---|---|---|
| 数据模型 | 结构化，表+行+列，Schema 严格 | Key-Value / 文档 / 列族 / 图，Schema 灵活 |
| 一致性 | ACID 强一致 | BASE 最终一致（部分支持强一致） |
| 扩展方式 | 垂直扩展（升级硬件）为主 | 水平扩展（加节点）为主 |
| 典型产品 | MySQL、PostgreSQL、Oracle | Redis、MongoDB、Cassandra、Neo4j |
| 查询语言 | SQL 标准 | 各自 API / DSL |
| 事务支持 | 完整事务 | 部分支持或不支持 |
| 适用场景 | ERP、财务、订单等强一致场景 | 缓存、社交、IoT、实时分析等海量并发场景 |

### 1.1.2 CAP 定理

CAP 定理指出：分布式系统最多只能同时满足以下三者中的两个：

- **C (Consistency) 一致性**：所有节点在同一时间看到的数据相同。
- **A (Availability) 可用性**：每个请求都能收到（成功或失败的）响应。
- **P (Partition Tolerance) 分区容错性**：网络分区故障时系统仍能继续运行。

Redis 在 CAP 中的定位：
- **单机 Redis**：偏向 C + A，分区容错性弱（单点故障风险）。
- **Redis Sentinel（哨兵模式）**：追求 A + P，主从切换期间可能出现短暂数据不一致。
- **Redis Cluster（集群模式）**：P + 最终一致性，通过分片容忍分区，但跨分片操作可能不一致。

实际建议：**P 必须满足（网络不可靠），Redis 在 C 与 A 之间根据部署模式做权衡。**

---

## 1.2 认识Redis

- Redis 是内存型 Key-Value 数据库，支持持久化与多种数据结构。
- 特性：
	- 内存读写，速度快（微秒级）
	- 丰富数据结构（String、Hash、List、Set、ZSet）
	- 持久化（RDB、AOF）与主从复制
	- 单线程模型 + I/O 多路复用
- 常见场景：缓存、分布式锁、限流、计数、排行榜、消息队列（Stream/发布订阅）。

### 1.2.1 Redis 版本历史

| 版本 | 发布时间 | 关键特性 |
|---|---|---|
| 1.0 | 2009年 | 初始版本，支持 String/List/Set 类型 |
| 2.0 | 2010年 | Pub/Sub、虚拟内存 |
| 2.2 | 2011年 | AOF 持久化优化 |
| 2.4 | 2011年 | Hash 类型优化 |
| 2.6 | 2012年 | Lua 脚本、PEXPIRE |
| 2.8 | 2013年 | SCAN 系列命令、Sentinel 哨兵高可用 |
| 3.0 | 2015年 | Redis Cluster 正式版（分布式） |
| 3.2 | 2016年 | GEO 地理信息索引 |
| 4.0 | 2017年 | Modules 模块系统、Lazy Free、混合持久化 |
| 5.0 | 2018年 | Stream 数据类型、ZPOPMIN/ZPOPMAX |
| 6.0 | 2020年 | ACL 权限控制、多线程 I/O、RESP3 |
| 6.2 | 2021年 | ZDIFF/ZINTER/ZUNION、LPOS、SMISMEMBER |
| 7.0 | 2022年 | Redis Functions、ACL v2、Sharded Pub/Sub、MP/SP/Cluter 改进 |
| 7.2 | 2023年 | 哨兵模式增强、性能优化 |
| 7.4 | 2024年 | Hash 过期、性能与稳定性改进 |

> **重要变更**：Redis 从 7.4 开始采用 RSALv2 + SSPLv1 双协议许可，不再使用 BSD 许可。

### 1.2.2 Redis vs Memcached 对比

| 对比维度 | Redis | Memcached |
|---|---|---|
| 数据结构 | String、Hash、List、Set、ZSet、Stream、Geo、Bitmap、HyperLogLog | 仅 String (Key-Value) |
| 持久化 | 支持 RDB / AOF / 混合持久化 | 不支持（重启数据丢失） |
| 线程模型 | 单线程主处理 + 多线程 I/O（6.0+） | 多线程 |
| 内存管理 | jemalloc，支持多种淘汰策略（LRU/LFU/TTL） | Slab Allocation，仅 LRU |
| 集群支持 | 原生 Cluster / Sentinel | 需客户端一致性 Hash |
| 事务 | MULTI/EXEC (轻量) | 不支持 |
| 发布订阅 | 支持 Pub/Sub、Stream | 不支持 |
| 性能参考 | 单机 10W+ QPS | 单机 10W+ QPS |
| Value 上限 | 512MB | 1MB |

### 1.2.3 Redis 基准性能参考

- **单机 QPS**：约 10万~15万（简单 SET/GET，无持久化）
- **延迟**：微秒级（局域网 < 1ms）
- **Pipeline 批量**：可达 50W~100W QPS
- **影响性能的因素**：持久化策略、网络延时、大 Key、慢查询命令

---

## 1.3 安装Redis以及启动的三种方式

- Windows 建议使用 WSL2 或 Docker。
- Linux 常见安装方式：
	- 源码编译：下载源码 -> make -> make install
	- 包管理器：apt/yum 安装

启动方式（示例）：
- 直接前台启动：`redis-server`
- 指定配置文件：`redis-server /etc/redis/redis.conf`
- 守护进程启动：配置 `daemonize yes` 后 `redis-server /path/redis.conf`

### 1.3.1 Windows — Docker Desktop 安装

**前置条件**：
1. 在 Windows 中启用 Hyper-V 或 WSL2。
2. 下载并安装 [Docker Desktop for Windows](https://www.docker.com/products/docker-desktop/)。
3. 安装完成后重启，确保 Docker Desktop 运行状态正常。

**验证安装**：
```powershell
docker --version
docker run hello-world
```

### 1.3.2 Windows — Docker 运行 Redis

```powershell
# 拉取 Redis 镜像
docker pull redis:7.4

# 启动 Redis 容器
docker run -d --name redis -p 6379:6379 redis:7.4

# 查看运行状态
docker ps

# 进入容器执行 redis-cli
docker exec -it redis redis-cli
```

### 1.3.3 完整 docker-compose.yml（Redis + RedisInsight）

```yaml
version: "3.8"
services:
  redis:
    image: redis:7.4
    container_name: redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
      - ./redis.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf
    restart: unless-stopped
    networks:
      - redis-net

  redis-insight:
    image: redislabs/redisinsight:latest
    container_name: redis-insight
    ports:
      - "8001:8001"
    volumes:
      - redisinsight_data:/db
    restart: unless-stopped
    networks:
      - redis-net

volumes:
  redis_data:
  redisinsight_data:

networks:
  redis-net:
    driver: bridge
```

启动：
```powershell
docker compose up -d
```

### 1.3.4 Linux — Ubuntu/Debian 通过 apt 安装

```bash
# 更新软件包索引
sudo apt update

# 安装 Redis
sudo apt install redis-server -y

# 启动 Redis 服务
sudo systemctl start redis-server

# 设置开机自启
sudo systemctl enable redis-server

# 查看运行状态
sudo systemctl status redis-server

# 测试连接
redis-cli ping
```

**配置文件位置**：`/etc/redis/redis.conf`

### 1.3.5 Linux — CentOS/RHEL 通过 yum 安装

```bash
# 安装 EPEL 仓库（CentOS 7）
sudo yum install epel-release -y

# 安装 Redis
sudo yum install redis -y

# 启动 Redis 服务
sudo systemctl start redis

# 设置开机自启
sudo systemctl enable redis

# 查看运行状态
sudo systemctl status redis

# 测试连接
redis-cli ping
```

### 1.3.6 macOS — 通过 Homebrew 安装

```bash
# 安装 Homebrew（如果未安装）
# /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 安装 Redis
brew install redis

# 启动 Redis（前台）
redis-server

# 启动 Redis（后台服务）
brew services start redis

# 停止 Redis
brew services stop redis

# 查看服务列表
brew services list
```

### 1.3.7 Docker 容器常用命令

```bash
# 启动容器
docker start redis

# 停止容器
docker stop redis

# 重启容器
docker restart redis

# 查看日志（实时）
docker logs -f redis

# 查看最近 100 条日志
docker logs --tail 100 redis

# 进入容器内部
docker exec -it redis bash

# 在容器内执行 redis-cli
docker exec -it redis redis-cli

# 删除容器（先停止）
docker stop redis; docker rm redis

# 删除镜像
docker rmi redis:7.4

# 查看容器资源占用
docker stats redis

# 仅查看数据卷
docker volume ls
```

### 1.3.8 redis.conf 关键配置参数解释

| 配置项 | 默认值 | 说明 |
|---|---|---|
| `bind 127.0.0.1` | `127.0.0.1` | 绑定 IP，生产环境建议设为内网 IP |
| `port 6379` | `6379` | 监听端口 |
| `daemonize no` | `no` | 是否以守护进程方式运行 |
| `requirepass foobared` | 注释 | 访问密码，生产环境必须设置 |
| `maxmemory <bytes>` | `0` (不限制) | 最大内存使用量，超过后触发淘汰策略 |
| `maxmemory-policy noeviction` | `noeviction` | 内存淘汰策略：`allkeys-lru`、`volatile-lru`、`allkeys-lfu` 等 |
| `save 900 1` | 注释 | RDB 快照条件：900秒内至少1个key变化则触发 |
| `appendonly no` | `no` | 是否开启 AOF 持久化 |
| `appendfsync everysec` | `everysec` | AOF 同步策略：`always` / `everysec` / `no` |
| `logfile ""` | `""` (stdout) | 日志文件路径 |
| `loglevel notice` | `notice` | 日志级别：debug/verbose/notice/warning |
| `databases 16` | `16` | 数据库数量（0~15） |
| `timeout 0` | `0` (不超时) | 客户端空闲超时（秒） |
| `tcp-keepalive 300` | `300` | TCP keepalive 探测间隔（秒） |
| `maxclients 10000` | `10000` | 最大客户端连接数 |
| `slowlog-log-slower-than 10000` | `10000` (微秒) | 慢查询阈值 |
| `slowlog-max-len 128` | `128` | 慢查询日志最大条数 |
| `replica-read-only yes` | `yes` | 从节点是否只读 |

**典型生产配置片段**：
```conf
bind 0.0.0.0          # 允许外部访问（需配合防火墙）
port 6379
daemonize yes          # 后台运行
requirepass Str0ngP@ss # 强密码
maxmemory 4gb          # 限制最大内存
maxmemory-policy allkeys-lru  # LRU 淘汰
appendonly yes         # 开启 AOF
appendfsync everysec   # 每秒同步一次 AOF
logfile /var/log/redis/redis.log
```

---

## 1.4 Redis命令行客户端

- 连接：`redis-cli -h 127.0.0.1 -p 6379`
- 常用命令：
	- `ping` -> `PONG`
	- `select 0` 切换库
	- `auth <password>` 认证
	- `quit` 退出

### 1.4.1 认证流程

```bash
# 连接 Redis（服务端已设置 requirepass）
redis-cli -h 127.0.0.1 -p 6379

# 此时执行任何命令都会报错：(error) NOAUTH Authentication required.
ping

# 方式一：进入 cli 后认证
auth <password>

# 方式二：连接时直接认证
redis-cli -h 127.0.0.1 -p 6379 -a <password>

# 方式三：通过环境变量（避免密码泄露在命令行历史）
# 设置 REDISCLI_AUTH 环境变量
redis-cli -h 127.0.0.1 -p 6379
```

> **安全提示**：避免使用 `-a` 参数传递密码，该方式会将密码暴露在进程列表和 Shell 历史中。推荐进入 cli 后使用 `auth` 命令。

### 1.4.2 数据库切换 SELECT

```bash
# Redis 默认有 16 个数据库（0~15）
SELECT 0   # 切换到 0 号库
SELECT 1   # 切换到 1 号库
SELECT 15  # 切换到 15 号库

# 注意：Redis Cluster 模式下不支持 SELECT，仅使用 db0
```

### 1.4.3 INFO 命令 — 获取服务器状态

```bash
# 获取全部信息
INFO

# 获取指定模块信息
INFO server      # 服务器基本信息（版本、运行时间等）
INFO clients     # 客户端连接信息
INFO memory      # 内存使用详情
INFO stats       # 命令统计
INFO replication # 主从复制状态
INFO cpu         # CPU 使用情况
INFO keyspace    # 各数据库 Key 数量
INFO persistence # 持久化状态

# 示例：查看内存使用
127.0.0.1:6379> INFO memory
# used_memory:1048576
# used_memory_human:1.00M
# used_memory_rss:2097152
# mem_fragmentation_ratio:2.00
```

### 1.4.4 CONFIG GET / SET

```bash
# 查看当前所有配置
CONFIG GET *

# 查看指定配置
CONFIG GET maxmemory
CONFIG GET maxmemory-policy
CONFIG GET appendonly
CONFIG GET dir

# 运行时修改配置（无需重启）
CONFIG SET maxmemory 2gb
CONFIG SET maxmemory-policy allkeys-lru
CONFIG SET timeout 300

# 将运行时修改写入配置文件（持久化配置）
CONFIG REWRITE
```

### 1.4.5 MONITOR 命令 — 实时监控

```bash
# 开启实时监控（在生产环境慎用，会消耗约 50% 性能）
MONITOR

# 输出示例：
# 1715678901.123456 [0 127.0.0.1:54321] "SET" "user:1001" "tom"
# 1715678902.234567 [0 127.0.0.1:54322] "GET" "user:1001"

# 适用于调试问题，执行完后按 Ctrl+C 退出监控
```

### 1.4.6 CLIENT LIST 查看客户端连接

```bash
# 查看所有已连接客户端
CLIENT LIST

# 输出示例：
# id=3 addr=127.0.0.1:54321 fd=8 name= age=120 idle=0 flags=N db=0 sub=0 psub=0

# 查看客户端数量
CLIENT LIST | grep -c "id="   # Linux
CLIENT LIST                      # 手动统计

# 关闭指定客户端连接
CLIENT KILL 127.0.0.1:54321

# 关闭指定 IP 的所有客户端
CLIENT KILL ADDR 127.0.0.1

# 设置当前连接的名称
CLIENT SETNAME my-app-server

# 获取当前连接名称
CLIENT GETNAME
```

---

## 1.5 Redis的图形化界面客户端

- 推荐：RedisInsight、Another Redis Desktop Manager
- 功能：可视化 Key 管理、命令执行、监控、慢查询分析。

### 1.5.1 RedisInsight — 通过 Docker 部署

RedisInsight 是 Redis 官方出品的免费图形化管理工具。如果使用了上面的 `docker-compose.yml`，RedisInsight 已经随 Redis 一起启动了。

访问地址：`http://localhost:8001`

**手动启动 RedisInsight（不使用 compose）：**
```powershell
docker run -d --name redis-insight -p 8001:8001 redislabs/redisinsight:latest
```

### 1.5.2 RedisInsight 基本使用

1. **连接 Redis**：
   - 打开 `http://localhost:8001`
   - 点击 "Add Redis Database"
   - Host: `host.docker.internal`（Windows）或 `redis`（同一 compose 网络）
   - Port: `6379`
   - 如有密码，填写密码
   - 点击 "Add Redis Database"

2. **浏览 Key**：
   - 左侧 Browser 面板可展开键空间
   - 支持按前缀筛选、按类型过滤
   - 点击 Key 查看值（支持 JSON/Hex 等视图）

3. **执行命令**：
   - 进入 CLI 选项卡
   - 输入命令实时执行，支持语法高亮

4. **监控内存**：
   - Overview 面板查看内存使用趋势图
   - Key 详情页可查看每个 Key 的内存占用

5. **慢查询分析**：
   - Slowlog 面板查看慢查询记录
   - 分析执行慢的命令并优化

### 1.5.3 Another Redis Desktop Manager

另一款流行的开源 Redis 桌面管理工具：
- 下载地址：https://github.com/qishibo/AnotherRedisDesktopManager
- 支持 Windows / macOS / Linux
- 功能：多连接管理、Key 树形浏览、批量删除、SSH 隧道、数据导出

---

# 2. Redis命令

## 2.1 通用命令

- `keys *`：匹配所有 Key（线上慎用）
- `scan 0 match user:* count 100`：增量扫描
- `type key`：查看数据类型
- `del key`：删除
- `exists key`：存在性判断
- `expire key 60` / `ttl key`：过期设置与查看
- `rename key newKey`：重命名

### 2.1.1 SCAN 命令详解

`KEYS *` 会阻塞 Redis，生产环境绝对禁止使用。`SCAN` 是安全的游标迭代方式。

```bash
# 基本用法：SCAN cursor [MATCH pattern] [COUNT count]
SCAN 0 MATCH user:* COUNT 100

# 返回结果：
# 1) "6"           -- 下一次迭代的游标（返回 0 表示遍历完成）
# 2) 1) "user:1"   -- 匹配到的 Key 列表
#    2) "user:2"

# 完整遍历步骤
SCAN 0 MATCH order:* COUNT 1000     # 第一次
SCAN 16 MATCH order:* COUNT 1000    # 用上一次返回的游标继续
SCAN 0 MATCH order:* COUNT 1000     # 游标返回 0 表示结束
```

**SCAN 注意事项**：
- 单次 `SCAN` 不一定返回 `COUNT` 个元素，`COUNT` 只是建议值。
- 遍历过程中新添加的 Key 可能不会被返回，已删除的 Key 可能仍被返回（增量迭代无快照隔离）。
- 游标值不确定，不要改游标为 0 强制结束循环。

### 2.1.2 EXISTS — 多 Key 存在性判断

```bash
# 单个 Key
EXISTS user:1
# (integer) 1   -- 存在返回 1，不存在返回 0

# 多个 Key（返回存在的数量）
EXISTS user:1 user:2 user:999
# (integer) 2   -- user:1 和 user:2 存在，user:999 不存在
```

### 2.1.3 过期时间相关命令

```bash
# EXPIRE — 设置相对过期时间（秒）
EXPIRE session:token123 3600

# EXPIREAT — 设置绝对过期时间（Unix 时间戳，秒）
EXPIREAT coupon:2024 1715692800

# PEXPIRE — 设置相对过期时间（毫秒）
PEXPIRE api:ratelimit:uid:1001 60000

# PEXPIREAT — 设置绝对过期时间（Unix 时间戳，毫秒）
PEXPIREAT event:2024 1715692800000

# PERSIST — 移除过期时间，变为永久 Key
PERSIST session:token123

# PTTL — 查看剩余过期时间（毫秒，-1 永久，-2 不存在）
PTTL session:token123
```

### 2.1.4 TTL 返回值含义表

| 返回值 | 含义 |
|---|---|
| `正整数` | 剩余过期时间（秒） |
| `-1` | Key 存在但没有设置过期时间（永久有效） |
| `-2` | Key 不存在 |

```bash
SET noex key1 value   # 不设置过期时间
TTL key1              # -1（永久有效）

SETEX key2 60 value2  # 60 秒过期
TTL key2              # 58（剩余秒数）

DEL key3
TTL key3              # -2（Key 不存在）
```

### 2.1.5 DUMP / RESTORE — Key 序列化与恢复

```bash
# DUMP — 将 Key 序列化为二进制数据
DUMP user:1
# "\x00\x03tom\t\..."

# RESTORE — 从序列化数据恢复 Key（可指定 TTL）
RESTORE user:backup 0 "\x00\x03tom\t\..."
# 参数：RESTORE key ttl serialized-value [REPLACE]
# ttl=0 表示无过期，REPLACE 覆盖已存在的 Key
```

### 2.1.6 OBJECT 命令 — 查看 Key 的底层信息

```bash
# OBJECT ENCODING — 查看 Key 的底层编码
OBJECT ENCODING count
# "int"       -- 整数编码
OBJECT ENCODING name
# "embstr"    -- 短字符串编码（<=44字节）
OBJECT ENCODING longtext
# "raw"       -- 长字符串编码
OBJECT ENCODING user:1
# "ziplist" 或 "listpack" 或 "hashtable"

# OBJECT REFCOUNT — 查看引用计数
OBJECT REFCOUNT name
# (integer) 1

# OBJECT IDLETIME — 查看 Key 闲置时间（秒）
OBJECT IDLETIME name
# (integer) 120  -- 已闲置 120 秒
```

### 2.1.7 数据库管理命令

```bash
# DBSIZE — 查看当前数据库 Key 数量
DBSIZE
# (integer) 1024

# FLUSHDB — 清空当前数据库（危险！）
FLUSHDB

# FLUSHALL — 清空所有数据库（极度危险！）
FLUSHALL

# FLUSHDB ASYNC / FLUSHALL ASYNC — 异步清空（Redis 4.0+，不阻塞）
FLUSHDB ASYNC
FLUSHALL ASYNC
```

> **警告**：FLUSHDB / FLUSHALL 是不可逆操作。生产环境建议在 redis.conf 中禁用：`rename-command FLUSHDB ""` `rename-command FLUSHALL ""`

---

## 2.2 String类型

- 适用：缓存、计数、对象序列化。
- 常用：
	- `set k v` / `get k`
	- `setnx k v`（不存在才设置）
	- `incr k` / `incrby k 10`
	- `decr k` / `append k "x"`

Java 示例（Jedis）：
```java
import redis.clients.jedis.Jedis;

public class RedisStringDemo {
		public static void main(String[] args) {
				try (Jedis jedis = new Jedis("localhost", 6379)) {
						jedis.set("counter", "1");
						jedis.incrBy("counter", 5);
						String value = jedis.get("counter");
						System.out.println(value);
				}
		}
}
```

### 2.2.1 子串操作

```bash
# GETRANGE — 获取字符串的子串（索引从 0 开始）
SET message "Hello Redis"
GETRANGE message 0 4       # "Hello"
GETRANGE message 6 -1      # "Redis"
GETRANGE message -5 -1     # "Redis"（负数表示倒数）

# SETRANGE — 覆盖字符串指定位置的子串
SET phone "138-0000-0000"
SETRANGE phone 4 "1234"    # 从索引 4 开始覆盖
GET phone                   # "138-1234-0000"
```

### 2.2.2 高级字符串命令

```bash
# GETEX — 获取值并设置过期时间（Redis 6.2+）
SET token "abc123"
GETEX token EX 60          # 获取值并设置 60 秒过期
GETEX token PX 60000       # 获取值并设置 60000 毫秒过期
GETEX token PERSIST        # 获取值并移除过期时间
GETEX token                # 仅获取值，不改变过期时间

# GETDEL — 获取值并删除 Key（Redis 6.2+）
SET temp "one-time-use"
GETDEL temp                # "one-time-use"
GET temp                   # (nil)  -- 已被删除

# MSET / MGET — 批量操作（减少网络往返）
MSET user:1:name "tom" user:1:age "25" user:1:city "Beijing"
MGET user:1:name user:1:age user:1:city
# 1) "tom"
# 2) "25"
# 3) "Beijing"

# MSETNX — 批量设置（只有所有 Key 都不存在时才成功）
MSETNX lock:a val1 lock:b val2   # 返回 1（成功）
MSETNX lock:a val3 lock:c val4   # 返回 0（失败，lock:a 已存在）
```

### 2.2.3 Spring Data Redis 操作 String

```java
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.ValueOperations;
import org.springframework.stereotype.Service;

import java.time.Duration;

@Service
public class StringRedisService {

    private final StringRedisTemplate redisTemplate;

    public StringRedisService(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public void stringOperations() {
        ValueOperations<String, String> ops = redisTemplate.opsForValue();

        // 基本 SET / GET
        ops.set("product:1001:name", "iPhone 15");

        // SET 带过期时间
        ops.set("session:token123", "uid=1001", Duration.ofHours(2));

        // SETNX 等价操作（仅在 Key 不存在时设置）
        Boolean success = ops.setIfAbsent("lock:order:1001", "locked", Duration.ofSeconds(30));

        // INCR / DECR
        ops.increment("article:1001:views");
        ops.increment("article:1001:views", 10L);
        ops.decrement("stock:1001");

        // MSET / MGET（批量）
        redisTemplate.opsForValue().multiSet(
            java.util.Map.of("k1", "v1", "k2", "v2", "k3", "v3")
        );
        java.util.List<String> values = redisTemplate.opsForValue()
            .multiGet(java.util.List.of("k1", "k2", "k3"));
    }
}
```

### 2.2.4 实战场景：分布式计数器

```java
@Service
public class DistributedCounterService {

    private final StringRedisTemplate redisTemplate;

    public DistributedCounterService(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    // 文章阅读计数
    public Long incrementArticleViews(Long articleId) {
        String key = "article:" + articleId + ":views";
        return redisTemplate.opsForValue().increment(key);
    }

    // 获取计数
    public Long getArticleViews(Long articleId) {
        String key = "article:" + articleId + ":views";
        String value = redisTemplate.opsForValue().get(key);
        return value != null ? Long.parseLong(value) : 0L;
    }

    // 库存扣减（原子操作）
    public Long decreaseStock(Long productId, int quantity) {
        String key = "product:" + productId + ":stock";
        return redisTemplate.opsForValue().decrement(key, quantity);
    }
}
```

### 2.2.5 实战场景：分布式锁

```java
@Service
public class DistributedLockService {

    private final StringRedisTemplate redisTemplate;

    public DistributedLockService(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    // 尝试获取锁（SET key value NX PX milliseconds）
    public boolean tryLock(String lockKey, String lockValue, long expireMillis) {
        return Boolean.TRUE.equals(
            redisTemplate.opsForValue()
                .setIfAbsent(lockKey, lockValue, Duration.ofMillis(expireMillis))
        );
    }

    // 释放锁（Lua 脚本保证原子性：只有值匹配才释放）
    public boolean releaseLock(String lockKey, String lockValue) {
        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                        "return redis.call('del', KEYS[1]) " +
                        "else return 0 end";
        Long result = redisTemplate.execute(
            new org.springframework.data.redis.core.script.DefaultRedisScript<>(script, Long.class),
            java.util.Collections.singletonList(lockKey), lockValue
        );
        return result != null && result == 1;
    }
}
```

### 2.2.6 实战场景：缓存模式

```java
@Service
public class CacheService {

    private final StringRedisTemplate redisTemplate;

    public CacheService(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    // 缓存读取模式（Cache-Aside）
    public String getWithCache(String key) {
        // 1. 先查缓存
        String cached = redisTemplate.opsForValue().get(key);
        if (cached != null) {
            return cached;
        }
        // 2. 缓存未命中，查数据库（此处模拟）
        String dbValue = queryFromDatabase(key);
        if (dbValue != null) {
            // 3. 写入缓存，设置过期时间
            redisTemplate.opsForValue().set(key, dbValue, Duration.ofMinutes(30));
        }
        return dbValue;
    }

    private String queryFromDatabase(String key) {
        // 模拟数据库查询
        return "db_value_for_" + key;
    }

    // 缓存更新：先更新数据库，再删除缓存
    public void updateWithCacheInvalidation(String key, String newValue) {
        updateDatabase(key, newValue);
        redisTemplate.delete(key);  // 删除缓存，而非更新
    }

    private void updateDatabase(String key, String value) {
        // 模拟数据库更新
    }
}
```

---

## 2.3 Key的层级格式

- 约定：业务:对象:主键:字段
- 示例：
	- `user:1001:name`
	- `order:20240512:status`
- 优点：避免冲突、便于批量扫描与治理。

### 2.3.1 更多业务域示例

```bash
# 用户模块
user:1001:name          # 用户名
user:1001:email         # 用户邮箱
user:1001:session       # 用户会话
user:1001:cart          # 购物车

# 订单模块
order:202405121001      # 订单详情
order:202405121001:items # 订单项
order:202405121001:status # 订单状态

# 商品模块
product:1001:detail     # 商品详情
product:1001:stock      # 商品库存
product:1001:price      # 商品价格

# 缓存模块
cache:hot:articles      # 热门文章列表
cache:user:1001:feed    # 用户推荐流
cache:dict:province     # 省份字典

# 分布式锁
lock:order:create:1001  # 下单锁
lock:task:daily:report  # 定时任务锁

# 限流
ratelimit:api:/login:1001       # 登录接口限流（按用户）
ratelimit:sms:phone:13800138000 # 短信验证码限流
```

### 2.3.2 Key 命名最佳实践

1. **长度控制**：Key 不宜过长（占用内存），也不宜过短（可读性差），建议在 30-50 字符以内。
2. **分隔符**：统一使用冒号 `:` 分隔层级（Redis 社区惯例），避免使用 `/`、`_` 等。
3. **大小写规范**：全部使用小写字母，避免大小写混淆（`USER:1` 和 `user:1` 是不同的 Key）。
4. **命名一致性**：同一个业务域内，Key 格式必须统一。
5. **避免特殊字符**：不要使用空格、中文等特殊字符作为 Key 名。
6. **前缀策略**：按业务域设计 Key 前缀，方便：
   - 使用 `SCAN` 按前缀批量扫描
   - 使用 `redis-cli --bigkeys` 分析大 Key 来源
   - 在 RedisInsight 中按前缀分组浏览
   - 配置淘汰策略时区分 `volatile-lru` vs `allkeys-lru`
7. **反例**：
   ```
   key1 = "user1"                         # 不直观，不知道是什么业务
   key2 = "product_1001"                  # 分隔符不统一
   key3 = "Order:2024:Status"             # 大小写不一致
   key4 = "user:1001:very-very-long-name" # 过长
   ```

---

## 2.4 Hash类型

- 适用：对象存储（字段-值）
- 常用：
	- `hset user:1 name "tom" age 18`
	- `hget user:1 name`
	- `hmget user:1 name age`
	- `hgetall user:1`

Java 示例（Jedis）：
```java
import redis.clients.jedis.Jedis;
import java.util.Map;

public class RedisHashDemo {
		public static void main(String[] args) {
				try (Jedis jedis = new Jedis("localhost", 6379)) {
						jedis.hset("user:1", "name", "tom");
						jedis.hset("user:1", "age", "18");
						Map<String, String> map = jedis.hgetAll("user:1");
						System.out.println(map);
				}
		}
}
```

### 2.4.1 Hash 高级命令

```bash
# HSETNX — 仅当字段不存在时才设置
HSET user:1 name "tom"
HSETNX user:1 name "jerry"    # 返回 0（字段已存在，设置失败）
HSETNX user:1 email "a@b.com" # 返回 1（字段不存在，设置成功）

# HINCRBY — 对 Hash 中的数值字段进行原子自增
HSET product:1001 stock 100
HINCRBY product:1001 stock -5   # stock: 95
HINCRBY product:1001 sold 3     # sold: 3

# HINCRBYFLOAT — 浮点数自增
HSET product:1001 price 99.99
HINCRBYFLOAT product:1001 price 10.01  # price: 110.00
HINCRBYFLOAT product:1001 price -0.5   # price: 109.5

# HEXISTS — 判断字段是否存在
HEXISTS user:1 name    # 1（存在）
HEXISTS user:1 phone   # 0（不存在）

# HDEL — 删除指定字段
HDEL user:1 email phone   # 返回删除成功的字段数量

# HLEN — 获取字段数量
HSET user:1 name "tom" age "25" city "Beijing"
HLEN user:1   # 3

# HKEYS — 获取所有字段名
HKEYS user:1   # 1) "name"  2) "age"  3) "city"

# HVALS — 获取所有字段值
HVALS user:1   # 1) "tom"  2) "25"  3) "Beijing"

# HSCAN — 增量遍历 Hash 的键值对（大 Hash 必用）
HSET large:hash f1 v1 f2 v2 f3 v3 ... f10000 v10000
HSCAN large:hash 0 MATCH f1* COUNT 10
```

### 2.4.2 Hash 与 String 的选择

| 场景 | 推荐类型 | 原因 |
|---|---|---|
| 对象字段频繁单独更新 | Hash | 原子操作单个字段，节省内存 |
| 对象整体读写，字段少变化 | String (JSON) | 一次 GET/SET 完成 |
| 对象字段很多（> 500 个） | 拆分多个 Hash | 避免 HGETALL 阻塞 |
| 需要设置过期时间 | String 或 Hash（Redis 7.4+ 支持 Hash 字段过期） | Hash 不支持整体 TTL 外独立字段 TTL（7.4 之前） |

### 2.4.3 Spring Data Redis 操作 Hash

```java
import org.springframework.data.redis.core.HashOperations;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

import java.util.Map;

@Service
public class RedisHashService {

    private final StringRedisTemplate redisTemplate;

    public RedisHashService(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public void hashOperations() {
        HashOperations<String, Object, Object> ops = redisTemplate.opsForHash();

        // HSET 设置单个字段
        ops.put("user:1001", "name", "张三");
        ops.put("user:1001", "email", "zhangsan@example.com");
        ops.put("user:1001", "age", "28");

        // HGET 获取单个字段
        Object name = ops.get("user:1001", "name");

        // HMSET 批量设置
        ops.putAll("user:1002", Map.of("name", "李四", "age", "30", "city", "上海"));

        // HMGET 批量获取
        java.util.List<Object> fields = ops.multiGet("user:1001",
            java.util.List.of("name", "email"));

        // HGETALL 获取所有字段
        Map<Object, Object> all = ops.entries("user:1001");
        all.forEach((k, v) -> System.out.println(k + " = " + v));

        // HEXISTS 判断字段是否存在
        Boolean hasField = ops.hasKey("user:1001", "phone");

        // HDEL 删除字段
        ops.delete("user:1001", "email");

        // HINCRBY 数值自增
        ops.increment("user:1001", "login_count", 1L);
        ops.increment("user:1001", "balance", -50L);

        // HSCAN 增量遍历
        try (org.springframework.data.redis.core.Cursor<Map.Entry<Object, Object>> cursor =
                ops.scan("large:hash", org.springframework.data.redis.core.ScanOptions.scanOptions()
                    .match("f*").count(100).build())) {
            while (cursor.hasNext()) {
                Map.Entry<Object, Object> entry = cursor.next();
                System.out.println(entry.getKey() + " = " + entry.getValue());
            }
        }

        // HKEYS / HVALS / HLEN
        java.util.Set<Object> keys = ops.keys("user:1001");
        java.util.List<Object> values = ops.values("user:1001");
        Long size = ops.size("user:1001");
    }
}
```

### 2.4.4 实战场景：购物车

```java
@Service
public class ShoppingCartService {

    private final StringRedisTemplate redisTemplate;

    public ShoppingCartService(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    // 添加到购物车
    public void addItem(String userId, String productId, int quantity) {
        String key = "cart:" + userId;
        redisTemplate.opsForHash().put(key, productId, String.valueOf(quantity));
    }

    // 修改商品数量
    public void updateQuantity(String userId, String productId, int delta) {
        String key = "cart:" + userId;
        redisTemplate.opsForHash().increment(key, productId, delta);
    }

    // 删除购物车商品
    public void removeItem(String userId, String productId) {
        String key = "cart:" + userId;
        redisTemplate.opsForHash().delete(key, productId);
    }

    // 获取购物车所有商品
    public java.util.Map<Object, Object> getCart(String userId) {
        String key = "cart:" + userId;
        return redisTemplate.opsForHash().entries(key);
    }

    // 获取购物车商品数量
    public Long getCartItemCount(String userId) {
        String key = "cart:" + userId;
        return redisTemplate.opsForHash().size(key);
    }

    // 清空购物车
    public void clearCart(String userId) {
        redisTemplate.delete("cart:" + userId);
    }
}
```

### 2.4.5 实战场景：用户资料缓存

```java
@Service
public class UserProfileCache {

    private final StringRedisTemplate redisTemplate;

    public UserProfileCache(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    // 缓存用户资料
    public void cacheProfile(String userId, String name, String email, String phone,
                             String avatar, Long loginCount) {
        String key = "user:" + userId + ":profile";
        redisTemplate.opsForHash().putAll(key, java.util.Map.of(
            "name", name,
            "email", email,
            "phone", phone,
            "avatar", avatar,
            "login_count", String.valueOf(loginCount)
        ));
        redisTemplate.expire(key, Duration.ofHours(24));
    }

    // 获取某个字段
    public String getField(String userId, String field) {
        Object value = redisTemplate.opsForHash().get("user:" + userId + ":profile", field);
        return value != null ? value.toString() : null;
    }

    // 更新登录次数（原子操作）
    public Long incrementLoginCount(String userId) {
        return redisTemplate.opsForHash()
            .increment("user:" + userId + ":profile", "login_count", 1L);
    }
}
```

---

## 2.5 List类型

- 适用：消息队列、时间线
- 常用：
	- `lpush list a b c`
	- `rpush list x y z`
	- `lpop list` / `rpop list`
	- `lrange list 0 -1`

### 2.5.1 List 高级命令

```bash
# LINSERT — 在指定元素之前/之后插入
LPUSH mylist a b c
# mylist: c b a
LINSERT mylist BEFORE b x     # 在 b 之前插入 x
# mylist: c x b a
LINSERT mylist AFTER b y      # 在 b 之后插入 y
# mylist: c x b y a

# LREM — 删除指定值（可指定删除个数和方向）
RPUSH scores 90 80 90 70 90
LREM scores 2 90               # 从左删除 2 个 90
# scores: 80 70 90
LREM scores -1 70              # 从右删除 1 个 70
LREM scores 0 100              # 删除所有 100

# LTRIM — 保留指定范围的元素（用于固定长度队列）
RPUSH logs "log1" "log2" "log3" "log4" "log5"
LTRIM logs 0 2                 # 只保留前 3 个
# logs: log1 log2 log3

# LINDEX — 按索引取值
LINDEX logs 0    # log1
LINDEX logs -1   # log3（最后一个）

# LLEN — 获取列表长度
LLEN logs        # 3

# LPOS — 查找元素位置（Redis 6.2+）
RPUSH letters a b c a b c
LPOS letters a           # 0（第一个匹配位置）
LPOS letters a RANK 2    # 3（第二个匹配位置）
LPOS letters a COUNT 2   # [0, 3]（返回所有匹配位置）
LPOS letters z            # (nil)（不存在）

# BLPOP / BRPOP — 阻塞式弹出（超时时间，秒）
# 队列为空时阻塞等待，有新元素时立即返回
BLPOP task_queue 30      # 从左侧弹出，最多等 30 秒
# 返回：1) "task_queue"  2) "task_data"
BRPOP task_queue 0       # 从右侧弹出，0 表示永不超时
```

### 2.5.2 Jedis 操作 List

```java
import redis.clients.jedis.Jedis;
import java.util.List;

public class JedisListDemo {
    public static void main(String[] args) {
        try (Jedis jedis = new Jedis("localhost", 6379)) {

            // LPUSH / RPUSH
            jedis.lpush("tasks", "task3", "task2", "task1");
            jedis.rpush("tasks", "task4", "task5");

            // LRANGE
            List<String> tasks = jedis.lrange("tasks", 0, -1);
            System.out.println(tasks);

            // LPOP / RPOP
            String task = jedis.lpop("tasks");
            String lastTask = jedis.rpop("tasks");

            // BLPOP 阻塞弹出
            List<String> result = jedis.blpop(30, "task_queue");

            // LTRIM 固定长度
            jedis.ltrim("logs", 0, 99);  // 只保留最近 100 条

            // LLEN
            Long size = jedis.llen("tasks");
        }
    }
}
```

### 2.5.3 Spring Data Redis 操作 List

```java
@Service
public class ListRedisService {

    private final StringRedisTemplate redisTemplate;

    public ListRedisService(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public void listOperations() {
        var ops = redisTemplate.opsForList();

        // LPUSH / RPUSH
        ops.leftPushAll("queue:orders", "order3", "order2", "order1");
        ops.rightPush("queue:orders", "order4");

        // LPOP / RPOP
        String order = ops.leftPop("queue:orders");
        String lastOrder = ops.rightPop("queue:orders");

        // LRANGE 获取范围
        java.util.List<String> allOrders = ops.range("queue:orders", 0, -1);

        // LINDEX
        String first = ops.index("queue:orders", 0);

        // LLEN
        Long size = ops.size("queue:orders");

        // LREM
        ops.remove("queue:orders", 1, "order2");

        // LTRIM 固定长度
        ops.trim("queue:orders", 0, 99);

        // BLPOP / BRPOP（阻塞弹出）
        String task = ops.leftPop("task_queue", Duration.ofSeconds(30));
    }
}
```

### 2.5.4 实战场景：消息队列

```java
@Service
public class MessageQueueService {

    private final StringRedisTemplate redisTemplate;

    public MessageQueueService(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    // 生产者：LPUSH 写入队列
    public void publish(String queueName, String message) {
        redisTemplate.opsForList().leftPush(queueName, message);
    }

    // 消费者：BRPOP 阻塞读取
    public String consume(String queueName, int timeoutSeconds) {
        return redisTemplate.opsForList()
            .rightPop(queueName, Duration.ofSeconds(timeoutSeconds));
    }

    // 批量生产
    public void publishBatch(String queueName, java.util.List<String> messages) {
        redisTemplate.opsForList().leftPushAll(queueName, messages);
    }
}
```

### 2.5.5 实战场景：最新 N 条动态

```java
@Service
public class RecentItemsService {

    private final StringRedisTemplate redisTemplate;

    public RecentItemsService(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    // 添加新动态（LPUSH + LTRIM 保持固定长度）
    public void addFeed(String userId, String feedItem) {
        String key = "feed:" + userId;
        redisTemplate.opsForList().leftPush(key, feedItem);
        redisTemplate.opsForList().trim(key, 0, 49);  // 保留最近 50 条
    }

    // 获取最近 N 条动态
    public java.util.List<String> getRecentFeeds(String userId, int count) {
        String key = "feed:" + userId;
        return redisTemplate.opsForList().range(key, 0, count - 1);
    }

    // 示例：记录最新操作日志
    public void addOperationLog(String userId, String operation) {
        String key = "log:" + userId + ":operations";
        redisTemplate.opsForList().leftPush(key, operation);
        redisTemplate.opsForList().trim(key, 0, 199);  // 保留最近 200 条
    }
}
```

---

## 2.6 Set类型

- 适用：标签、去重、交并差
- 常用：
	- `sadd tag:java a b c`
	- `smembers tag:java`
	- `sismember tag:java a`
	- `sinter s1 s2` / `sunion s1 s2` / `sdiff s1 s2`

### 2.6.1 Set 高级命令

```bash
# SRANDMEMBER — 随机返回元素（不删除）
SADD colors red blue green yellow orange purple
SRANDMEMBER colors        # 随机 1 个
SRANDMEMBER colors 3      # 随机 3 个（可能重复）
SRANDMEMBER colors -3     # 随机 3 个（不重复）

# SPOP — 随机弹出并删除元素
SPOP colors               # 随机弹出 1 个
SPOP colors 2             # 随机弹出 2 个

# SMOVE — 将元素从一个 Set 移动到另一个 Set
SADD set1 a b c
SADD set2 x y z
SMOVE set1 set2 a         # 将 a 从 set1 移到 set2

# SCARD — 获取 Set 元素总数
SCARD colors              # 6

# SSCAN — 增量遍历大 Set
SSCAN big_set 0 MATCH item:* COUNT 1000

# SISMEMBER — 判断元素是否在 Set 中
SISMEMBER colors red      # 1（是）
SISMEMBER colors black    # 0（否）

# SMISMEMBER — 批量判断（Redis 6.2+）
SMISMEMBER colors red black blue  # 1) 1  2) 0  3) 1
```

### 2.6.2 Jedis 操作 Set

```java
import redis.clients.jedis.Jedis;
import java.util.Set;

public class JedisSetDemo {
    public static void main(String[] args) {
        try (Jedis jedis = new Jedis("localhost", 6379)) {

            // SADD
            jedis.sadd("tag:java", "spring", "mybatis", "junit");

            // SMEMBERS
            Set<String> members = jedis.smembers("tag:java");

            // SISMEMBER
            boolean isMember = jedis.sismember("tag:java", "spring");

            // SCARD
            Long size = jedis.scard("tag:java");

            // SRANDMEMBER
            String random = jedis.srandmember("tag:java");
            Set<String> randoms = jedis.srandmember("tag:java", 2);

            // SPOP
            String popped = jedis.spop("tag:java");

            // 集合运算
            Set<String> inter = jedis.sinter("tag:java", "tag:python");
            Set<String> union = jedis.sunion("tag:java", "tag:python");
            Set<String> diff = jedis.sdiff("tag:java", "tag:python");

            // SREM 删除元素
            jedis.srem("tag:java", "junit");
        }
    }
}
```

### 2.6.3 Spring Data Redis 操作 Set

```java
@Service
public class SetRedisService {

    private final StringRedisTemplate redisTemplate;

    public SetRedisService(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public void setOperations() {
        var ops = redisTemplate.opsForSet();

        // SADD
        ops.add("tag:spring", "web", "data-jpa", "security");
        ops.add("tag:frontend", "vue", "react", "web");

        // SMEMBERS
        java.util.Set<String> all = ops.members("tag:spring");

        // SISMEMBER
        Boolean exists = ops.isMember("tag:spring", "web");

        // SCARD
        Long size = ops.size("tag:spring");

        // SRANDMEMBER
        String rand = ops.randomMember("tag:spring");
        java.util.Set<String> randoms = ops.randomMembers("tag:spring", 2);

        // SPOP
        String popped = ops.pop("tag:spring");

        // 集合运算
        java.util.Set<String> inter = ops.intersect("tag:spring", "tag:frontend");
        java.util.Set<String> union = ops.union("tag:spring", "tag:frontend");
        java.util.Set<String> diff = ops.difference("tag:spring", "tag:frontend");

        // 集合运算并存储结果
        ops.intersectAndStore("tag:spring", "tag:frontend", "tag:common");
        ops.unionAndStore("tag:spring", "tag:frontend", "tag:all");

        // SMOVE
        ops.move("tag:spring", "web", "tag:frontend");

        // SSCAN
        try (org.springframework.data.redis.core.Cursor<String> cursor =
                ops.scan("tag:spring", org.springframework.data.redis.core.ScanOptions.scanOptions()
                    .match("web*").count(10).build())) {
            while (cursor.hasNext()) {
                System.out.println(cursor.next());
            }
        }
    }
}
```

### 2.6.4 实战场景：用户标签系统

```java
@Service
public class UserTagService {

    private final StringRedisTemplate redisTemplate;

    public UserTagService(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    // 为用户添加标签
    public void addTags(String userId, String... tags) {
        redisTemplate.opsForSet().add("user:" + userId + ":tags", tags);
    }

    // 删除用户标签
    public void removeTags(String userId, String... tags) {
        redisTemplate.opsForSet().remove("user:" + userId + ":tags", (Object[]) tags);
    }

    // 获取用户所有标签
    public java.util.Set<String> getUserTags(String userId) {
        return redisTemplate.opsForSet().members("user:" + userId + ":tags");
    }

    // 查找带标签的用户（反向索引）
    public java.util.Set<String> getUsersByTag(String tag) {
        return redisTemplate.opsForSet().members("tag:" + tag + ":users");
    }

    // 打标签时同时维护反向索引
    public void addTagsWithIndex(String userId, String... tags) {
        for (String tag : tags) {
            redisTemplate.opsForSet().add("user:" + userId + ":tags", tag);
            redisTemplate.opsForSet().add("tag:" + tag + ":users", userId);
        }
    }
}
```

### 2.6.5 实战场景：共同好友

```java
@Service
public class CommonFriendService {

    private final StringRedisTemplate redisTemplate;

    public CommonFriendService(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    // 添加好友关系
    public void addFriend(String userId, String friendId) {
        redisTemplate.opsForSet().add("friends:" + userId, friendId);
    }

    // 查找共同好友
    public java.util.Set<String> getCommonFriends(String userId1, String userId2) {
        return redisTemplate.opsForSet()
            .intersect("friends:" + userId1, "friends:" + userId2);
    }

    // 可能认识的人（好友的好友，但排除已是好友和自己）
    // = union(all friends' friends) - my friends - myself
    public java.util.Set<String> getRecommendFriends(String userId) {
        java.util.Set<String> myFriends = redisTemplate.opsForSet()
            .members("friends:" + userId);

        java.util.Set<String> candidates = new java.util.HashSet<>();
        var ops = redisTemplate.opsForSet();
        for (String friendId : myFriends) {
            java.util.Set<String> fof = ops.members("friends:" + friendId);
            if (fof != null) candidates.addAll(fof);
        }

        candidates.removeAll(myFriends);
        candidates.remove(userId);
        return candidates;
    }
}
```

---

## 2.7 SortedSet类型

- 适用：排行榜、延迟队列
- 常用：
	- `zadd rank 100 user1 98 user2`
	- `zrange rank 0 -1 withscores`
	- `zrevrange rank 0 9 withscores`
	- `zincrby rank 10 user1`

### 2.7.1 SortedSet 高级命令

```bash
# ZADD — 支持 NX / XX / GT / LT / CH 等选项
ZADD leaderboard NX 100 Alice         # 仅新增
ZADD leaderboard XX 200 Bob           # 仅更新已有元素
ZADD leaderboard XX GT 150 Charlie    # 仅当新分数大于当前分数时更新
ZADD leaderboard XX LT 80 David       # 仅当新分数小于当前分数时更新
ZADD leaderboard CH 90 Eve            # 返回被修改的成员数（含新增和更新）

# ZRANK / ZREVRANK — 获取排名（从 0 开始，升序）
ZADD rank 100 a 200 b 150 c 80 d
ZRANK rank c            # 2（c 排第 3，分数 150）
ZREVRANK rank c         # 1（倒序排名：第 2 高分）

# ZSCORE — 获取元素分数
ZSCORE rank b           # "200"

# ZREM — 删除元素
ZREM rank a c           # 返回 2

# ZCARD — 获取元素总数
ZCARD rank              # 4

# ZCOUNT — 按分数范围计数
ZCOUNT rank 100 200     # 分数在 [100, 200] 之间
ZCOUNT rank (100 200    # 分数在 (100, 200] 之间（左开右闭）

# ZPOPMIN / ZPOPMAX — 弹出最小/最大分数的元素（Redis 5.0+）
ZPOPMIN rank            # 弹出最低分 1 个
ZPOPMAX rank 3          # 弹出最高分 3 个

# ZRANGEBYSCORE — 按分数范围查询
ZRANGEBYSCORE rank 100 200            # [100, 200]
ZRANGEBYSCORE rank 100 (200           # [100, 200)
ZRANGEBYSCORE rank 100 +inf LIMIT 0 10  # 前 10 个 >=100 的

# ZREMRANGEBYRANK — 按排名范围删除
ZREMRANGEBYRANK rank 0 2   # 删除排名 0~2 的元素（最低分 3 个）

# ZREMRANGEBYSCORE — 按分数范围删除
ZREMRANGEBYSCORE rank 0 50  # 删除分数 <=50 的元素

# ZDIFF / ZINTER / ZUNION — 集合运算（Redis 6.2+）
ZADD z1 1 a 2 b 3 c
ZADD z2 1 a 3 c 5 e
ZDIFF 2 z1 z2                  # 差集：b (分数 2)
ZINTER 2 z1 z2 AGGREGATE SUM   # 交集并求和分数：a:2, c:6
ZUNION 2 z1 z2 AGGREGATE MAX   # 并集取最大值：a:1, b:2, c:3, e:5
ZINTERSTORE dest 2 z1 z2 WEIGHTS 2 3  # 交集并加权求和

# ZRANGE WITHSCORES — 正序查询（带分数）
ZRANGE rank 0 -1 WITHSCORES
# 1) "d" 2) "80"  3) "a" 4) "100"  5) "c" 6) "150"  7) "b" 8) "200"

# ZREVRANGE WITHSCORES — 倒序查询（带分数）
ZREVRANGE rank 0 2 WITHSCORES
# 1) "b" 2) "200"  3) "c" 4) "150"  5) "a" 6) "100"
```

### 2.7.2 Jedis 操作 SortedSet

```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.resps.Tuple;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class JedisZSetDemo {
    public static void main(String[] args) {
        try (Jedis jedis = new Jedis("localhost", 6379)) {

            // ZADD
            Map<String, Double> members = new HashMap<>();
            members.put("Alice", 95.0);
            members.put("Bob", 88.5);
            members.put("Charlie", 92.0);
            jedis.zadd("leaderboard", members);

            // ZINCRBY
            jedis.zincrby("leaderboard", 10, "Alice");

            // ZRANGE WITHSCORES
            List<Tuple> tuples = jedis.zrangeWithScores("leaderboard", 0, -1);
            for (Tuple t : tuples) {
                System.out.println(t.getElement() + " -> " + t.getScore());
            }

            // ZREVRANGE
            List<String> top3 = jedis.zrevrange("leaderboard", 0, 2);

            // ZRANK / ZREVRANK
            Long rank = jedis.zrank("leaderboard", "Alice");
            Long revRank = jedis.zrevrank("leaderboard", "Alice");

            // ZSCORE
            Double score = jedis.zscore("leaderboard", "Alice");

            // ZCOUNT
            Long count = jedis.zcount("leaderboard", 80, 100);

            // ZPOPMIN / ZPOPMAX
            Tuple min = jedis.zpopmin("leaderboard");
            Tuple max = jedis.zpopmax("leaderboard");
        }
    }
}
```

### 2.7.3 Spring Data Redis 操作 SortedSet

```java
@Service
public class ZSetRedisService {

    private final StringRedisTemplate redisTemplate;

    public ZSetRedisService(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public void zsetOperations() {
        var ops = redisTemplate.opsForZSet();

        // ZADD
        ops.add("leaderboard", "Alice", 95);
        ops.add("leaderboard", "Bob", 88.5);
        ops.add("leaderboard", "Charlie", 92);

        // 批量 ZADD
        java.util.Set<org.springframework.data.redis.core.ZSetOperations.TypedTuple<String>> tuples =
            new java.util.HashSet<>();
        tuples.add(org.springframework.data.redis.core.DefaultTypedTuple.of("David", 75.0));
        tuples.add(org.springframework.data.redis.core.DefaultTypedTuple.of("Eve", 85.0));
        ops.add("leaderboard", tuples);

        // ZINCRBY
        ops.incrementScore("leaderboard", "Alice", 10);

        // ZRANGE / ZREVRANGE
        java.util.Set<String> asc = ops.range("leaderboard", 0, -1);
        java.util.Set<String> desc = ops.reverseRange("leaderboard", 0, 9);

        // ZRANGE WITHSCORES
        java.util.Set<org.springframework.data.redis.core.ZSetOperations.TypedTuple<String>> withScores =
            ops.rangeWithScores("leaderboard", 0, 2);
        for (var t : withScores) {
            System.out.println(t.getValue() + " -> " + t.getScore());
        }

        // ZRANK / ZREVRANK
        Long rank = ops.rank("leaderboard", "Alice");
        Long revRank = ops.reverseRank("leaderboard", "Alice");

        // ZSCORE
        Double score = ops.score("leaderboard", "Alice");

        // ZCOUNT
        Long count = ops.count("leaderboard", 80, 100);

        // ZPOPMIN / ZPOPMAX
        var min = ops.popMin("leaderboard");
        var max = ops.popMax("leaderboard", 3);

        // ZREMRANGEBYRANK / ZREMRANGEBYSCORE
        ops.removeRange("leaderboard", 100, 999);
        ops.removeRangeByScore("leaderboard", 0, 60);

        // ZCARD
        Long size = ops.size("leaderboard");
    }
}
```

### 2.7.4 实战场景：排行榜

```java
@Service
public class LeaderboardService {

    private final StringRedisTemplate redisTemplate;

    public LeaderboardService(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    // 更新用户分数
    public void updateScore(String userId, double score) {
        redisTemplate.opsForZSet().add("game:leaderboard", userId, score);
    }

    // 增加分数
    public void addScore(String userId, double delta) {
        redisTemplate.opsForZSet().incrementScore("game:leaderboard", userId, delta);
    }

    // 获取 Top N
    public java.util.Set<String> getTopN(int n) {
        return redisTemplate.opsForZSet()
            .reverseRange("game:leaderboard", 0, n - 1);
    }

    // 获取 Top N（带分数）
    public java.util.Set<org.springframework.data.redis.core.ZSetOperations.TypedTuple<String>>
            getTopNWithScores(int n) {
        return redisTemplate.opsForZSet()
            .reverseRangeWithScores("game:leaderboard", 0, n - 1);
    }

    // 获取用户排名（倒序，第 1 名 rank=0）
    public Long getUserRank(String userId) {
        Long rank = redisTemplate.opsForZSet().reverseRank("game:leaderboard", userId);
        return rank != null ? rank + 1 : null;  // +1 转为从 1 开始
    }

    // 获取用户周围排名（前后各 2 名）
    public java.util.List<String> getAroundUser(String userId, int range) {
        Long rank = redisTemplate.opsForZSet().reverseRank("game:leaderboard", userId);
        if (rank == null) return java.util.Collections.emptyList();

        long start = Math.max(0, rank - range);
        long end = rank + range;
        return new java.util.ArrayList<>(
            redisTemplate.opsForZSet().reverseRange("game:leaderboard", start, end)
        );
    }
}
```

### 2.7.5 实战场景：延迟队列

```java
@Service
public class DelayQueueService {

    private final StringRedisTemplate redisTemplate;

    public DelayQueueService(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    // 添加延迟任务（score = 执行时间的时间戳毫秒）
    public void addTask(String taskId, long executeAtMillis) {
        redisTemplate.opsForZSet().add("delay:queue", taskId, executeAtMillis);
    }

    // 添加指定秒后执行的任务
    public void addTaskWithDelay(String taskId, long delaySeconds) {
        long executeAt = System.currentTimeMillis() + delaySeconds * 1000;
        redisTemplate.opsForZSet().add("delay:queue", taskId, executeAt);
    }

    // 消费已到期的任务
    public java.util.Set<String> pollExpiredTasks() {
        long now = System.currentTimeMillis();
        // 获取分数在 [0, now] 之间的任务
        return redisTemplate.opsForZSet().rangeByScore("delay:queue", 0, now);
    }

    // 消费并删除已到期任务
    public java.util.Set<String> consumeExpiredTasks(int limit) {
        long now = System.currentTimeMillis();
        // 使用 Lua 脚本保证原子性
        String script = "local tasks = redis.call('ZRANGEBYSCORE', KEYS[1], 0, ARGV[1], 'LIMIT', 0, ARGV[2]) " +
                        "if #tasks > 0 then redis.call('ZREM', KEYS[1], unpack(tasks)) end " +
                        "return tasks";
        return (java.util.Set<String>)(java.util.Set) redisTemplate.execute(
            new org.springframework.data.redis.core.script.DefaultRedisScript<>(script, java.util.Set.class),
            java.util.Collections.singletonList("delay:queue"),
            String.valueOf(now), String.valueOf(limit)
        );
    }
}
```

---

# 3. Redis的Java客户端

## 3.1 客户端对比

- Jedis：经典、轻量、API 直观
- Lettuce：基于 Netty，支持异步/响应式
- Spring Data Redis：对 Spring 生态友好，封装统一模板

### 3.1.1 详细对比表

| 维度 | Jedis | Lettuce | Redisson | Spring Data Redis |
|---|---|---|---|---|
| 底层网络 | BIO (阻塞 I/O) | Netty (NIO) | Netty (NIO) | 抽象层（换底层实现） |
| 线程安全 | 不安全（需连接池） | 安全（天生线程安全） | 安全 | 安全 |
| 异步支持 | 不支持（4.0+ 开始支持） | 原生支持异步/响应式 | 原生支持异步/响应式 | 取决于底层实现 |
| 连接池 | JedisPool | 无需连接池（连接复用） | 无需连接池 | 封装 Lettuce 连接 |
| API 风格 | 接近 Redis 原生命令 | 同步/异步/响应式三套 API | 分布式对象抽象（Lock、Map、Queue） | 模板方法 + opsForXxx |
| 分布式功能 | 仅命令执行 | 仅命令执行 | 分布式锁、布隆过滤器、限流器等 | 无（定位于操作层） |
| Spring 集成 | 需手动配置 | Spring Boot 默认集成 | Spring Boot Starter 可用 | 官方 Spring 项目 |
| 序列化 | 手动 | 手动 | 内置多种编解码器 | 可配置 Serializer |
| 适用场景 | 简单项目、学习 | 高并发 Web 应用 | 分布式场景、服务治理 | Spring Boot 项目 |

### 3.1.2 如何选择

- **Jedis**：传统项目、API 与 Redis 命令高度对齐、学习成本最低。适合小型项目或不依赖 Spring 的项目。
- **Lettuce**：Spring Boot 2.0+ 的默认客户端、高并发场景、需要异步/响应式编程。
- **Redisson**：需要分布式锁、布隆过滤器、原子长整型等高级分布式对象，适合微服务分布式场景。
- **Spring Data Redis**：Spring Boot 项目的标准选择，底层默认使用 Lettuce。

---

## 3.2 Jedis快速入门

依赖（Maven）：
```xml
<dependency>
		<groupId>redis.clients</groupId>
		<artifactId>jedis</artifactId>
		<version>5.1.0</version>
</dependency>
```

示例：
```java
import redis.clients.jedis.Jedis;

public class JedisQuickStart {
		public static void main(String[] args) {
				try (Jedis jedis = new Jedis("localhost", 6379)) {
						jedis.set("hello", "redis");
						System.out.println(jedis.get("hello"));
				}
		}
}
```

### 3.2.1 Jedis 带完整参数连接

```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.Protocol;

public class JedisWithParams {
    public static void main(String[] args) {
        // 方式一：构造参数
        Jedis jedis = new Jedis("192.168.1.100", 6379);
        jedis.auth("mypassword");
        jedis.select(2);  // 选择 2 号库
        System.out.println(jedis.ping());

        // 方式二：带超时参数的构造
        try (Jedis j = new Jedis("192.168.1.100", 6379, 2000, 2000)) {
            // 参数：host, port, connectionTimeout(ms), socketTimeout(ms)
            j.auth("mypassword");
            j.set("key1", "value1");
            System.out.println(j.get("key1"));
        }

        // 方式三：使用 URI 连接
        // redis://[:password@]host[:port][/database]
        try (Jedis j = new Jedis(
                java.net.URI.create("redis://:mypassword@192.168.1.100:6379/3"))) {
            System.out.println(j.ping());
        }
    }
}
```

### 3.2.2 Jedis 错误处理

```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.exceptions.JedisConnectionException;
import redis.clients.jedis.exceptions.JedisDataException;

public class JedisErrorHandling {
    public static void main(String[] args) {
        Jedis jedis = null;
        try {
            jedis = new Jedis("localhost", 6379, 5000);
            jedis.ping();
            jedis.set("key", "value");
            System.out.println(jedis.get("key"));
        } catch (JedisConnectionException e) {
            System.err.println("无法连接 Redis：" + e.getMessage());
            // 执行降级策略或重试
        } catch (JedisDataException e) {
            System.err.println("Redis 命令执行异常：" + e.getMessage());
        } catch (Exception e) {
            System.err.println("未知异常：" + e.getMessage());
        } finally {
            if (jedis != null) {
                jedis.close();
            }
        }
    }
}
```

---

## 3.3 Jedis的连接池

```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;
import redis.clients.jedis.JedisPoolConfig;

public class JedisPoolDemo {
		public static void main(String[] args) {
				JedisPoolConfig config = new JedisPoolConfig();
				config.setMaxTotal(20);
				config.setMaxIdle(10);

				try (JedisPool pool = new JedisPool(config, "localhost", 6379)) {
						try (Jedis jedis = pool.getResource()) {
								jedis.set("pool:key", "v1");
								System.out.println(jedis.get("pool:key"));
						}
				}
		}
}
```

### 3.3.1 连接池参数详解

| 参数 | 默认值 | 说明 |
|---|---|---|
| `maxTotal` | 8 | 连接池最大连接数 |
| `maxIdle` | 8 | 连接池最大空闲连接数 |
| `minIdle` | 0 | 连接池最小空闲连接数 |
| `maxWaitMillis` | `-1`（无限） | 获取连接最大等待时间（ms），`-1` 表示无限等待 |
| `minEvictableIdleTimeMillis` | 1800000 (30min) | 连接空闲最小驱逐时间 |
| `timeBetweenEvictionRunsMillis` | `-1`（不执行） | 空闲连接检测间隔（ms） |
| `numTestsPerEvictionRun` | 3 | 每次驱逐检测的连接数 |
| `testOnBorrow` | `false` | 获取连接时是否检测可用性（影响性能） |
| `testOnReturn` | `false` | 归还连接时是否检测可用性 |
| `testWhileIdle` | `false` | 空闲时是否检测可用性（推荐 `true`） |
| `blockWhenExhausted` | `true` | 连接耗尽时是否阻塞等待 |
| `jmxEnabled` | `true` | 是否注册 JMX MBean 监控 |

### 3.3.2 生产级连接池配置 + Shutdown Hook

```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;
import redis.clients.jedis.JedisPoolConfig;

import java.time.Duration;

public class ProductionJedisPool {

    private static JedisPool pool;

    static {
        JedisPoolConfig config = new JedisPoolConfig();

        // 连接池基本配置
        config.setMaxTotal(50);             // 最大连接数
        config.setMaxIdle(20);              // 最大空闲连接
        config.setMinIdle(5);               // 最小空闲连接（预热）
        config.setMaxWait(Duration.ofSeconds(3));  // 获取连接最大等待 3 秒

        // 连接验证
        config.setTestOnBorrow(true);       // 借用时验证（生产建议开启）
        config.setTestOnReturn(false);      // 归还时验证（性能影响小，可不开启）
        config.setTestWhileIdle(true);      // 空闲检测（配合驱逐线程）

        // 驱逐策略
        config.setTimeBetweenEvictionRuns(Duration.ofSeconds(30));  // 每 30s 检测一次
        config.setMinEvictableIdleDuration(Duration.ofMinutes(5));  // 空闲超过 5 分钟即驱逐
        config.setNumTestsPerEvictionRun(10);                       // 每次检测 10 个连接

        // 连接耗尽策略
        config.setBlockWhenExhausted(true);

        pool = new JedisPool(config, "localhost", 6379, 2000, "mypassword");

        // JVM 关闭时清理连接池
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            if (pool != null && !pool.isClosed()) {
                pool.close();
                System.out.println("JedisPool closed.");
            }
        }));
    }

    public static Jedis getResource() {
        return pool.getResource();
    }

    public static void main(String[] args) {
        try (Jedis jedis = getResource()) {
            jedis.set("prod:test", "hello");
            System.out.println(jedis.get("prod:test"));
        }
    }
}
```

---

## 3.4 认识SpringDataRedis

- Spring Data Redis 封装了序列化、连接池、模板操作。
- 常用入口：`RedisTemplate`、`StringRedisTemplate`。

### 3.4.1 Spring Boot 自动配置默认值

| 属性 | 默认值 | 说明 |
|---|---|---|
| `spring.data.redis.host` | `localhost` | Redis 服务器地址 |
| `spring.data.redis.port` | `6379` | Redis 端口 |
| `spring.data.redis.password` | 空 | 连接密码 |
| `spring.data.redis.database` | `0` | 数据库索引 |
| `spring.data.redis.timeout` | 空 | 连接超时时间 |
| `spring.data.redis.lettuce.pool.max-active` | `8` | 连接池最大连接数 |
| `spring.data.redis.lettuce.pool.max-idle` | `8` | 连接池最大空闲连接数 |
| `spring.data.redis.lettuce.pool.min-idle` | `0` | 连接池最小空闲连接数 |
| `spring.data.redis.lettuce.pool.max-wait` | `-1ms` | 最大等待时间 |
| `spring.data.redis.lettuce.shutdown-timeout` | `100ms` | 关闭超时 |

### 3.4.2 application.yml 配置示例

**单机模式**：
```yaml
spring:
  data:
    redis:
      host: 192.168.1.100
      port: 6379
      password: mypassword
      database: 0
      timeout: 3000ms
      lettuce:
        pool:
          max-active: 20
          max-idle: 10
          min-idle: 5
          max-wait: 2000ms
```

**哨兵模式**：
```yaml
spring:
  data:
    redis:
      password: mypassword
      sentinel:
        master: mymaster
        nodes:
          - 192.168.1.101:26379
          - 192.168.1.102:26379
          - 192.168.1.103:26379
      lettuce:
        pool:
          max-active: 20
          max-idle: 10
```

**集群模式**：
```yaml
spring:
  data:
    redis:
      password: mypassword
      cluster:
        nodes:
          - 192.168.1.101:6379
          - 192.168.1.102:6379
          - 192.168.1.103:6379
        max-redirects: 3
      lettuce:
        pool:
          max-active: 20
          max-idle: 10
```

### 3.4.3 Lettuce 连接工厂默认行为

Spring Boot 2.x 开始默认使用 Lettuce 作为 Redis 客户端：

- **连接共享**：Lettuce 使用单连接线程安全，默认无需连接池即可复用。
- **连接池**：启用连接池需添加 `commons-pool2` 依赖：
  ```xml
  <dependency>
      <groupId>org.apache.commons</groupId>
      <artifactId>commons-pool2</artifactId>
  </dependency>
  ```
- **异步/响应式**：Lettuce 原生支持 `ReactiveRedisTemplate`，适合 WebFlux 项目。

---

## 3.5 RedisTemplate快速入门

```java
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

@Service
public class RedisTemplateService {
		private final StringRedisTemplate template;

		public RedisTemplateService(StringRedisTemplate template) {
				this.template = template;
		}

		public void demo() {
				template.opsForValue().set("k1", "v1");
				String v = template.opsForValue().get("k1");
				System.out.println(v);
		}
}
```

### 3.5.1 opsForXxx 各类型操作速览

```java
@Service
public class OpsOverview {

    private final StringRedisTemplate template;

    public OpsOverview(StringRedisTemplate template) {
        this.template = template;
    }

    public void allOpsDemo() {
        // String（opsForValue）
        template.opsForValue().set("str:key", "value");
        String str = template.opsForValue().get("str:key");

        // Hash（opsForHash）
        template.opsForHash().put("hash:key", "field1", "value1");
        Object hashVal = template.opsForHash().get("hash:key", "field1");

        // List（opsForList）
        template.opsForList().leftPush("list:key", "item1");
        String listVal = template.opsForList().rightPop("list:key");

        // Set（opsForSet）
        template.opsForSet().add("set:key", "a", "b", "c");
        java.util.Set<String> setMembers = template.opsForSet().members("set:key");

        // ZSet（opsForZSet）
        template.opsForZSet().add("zset:key", "member1", 100);
        java.util.Set<String> zsetRange = template.opsForZSet().range("zset:key", 0, -1);

        // Geo（opsForGeo）
        template.opsForGeo().add("geo:cities",
            new org.springframework.data.geo.Point(116.40, 39.90), "Beijing");
        template.opsForGeo().add("geo:cities",
            new org.springframework.data.geo.Point(121.47, 31.23), "Shanghai");
        org.springframework.data.geo.Distance dist = template.opsForGeo()
            .distance("geo:cities", "Beijing", "Shanghai",
                org.springframework.data.redis.connection.RedisGeoCommands.DistanceUnit.KILOMETERS);

        // HyperLogLog（opsForHyperLogLog）
        template.opsForHyperLogLog().add("hll:visits", "user1", "user2", "user3");
        Long uv = template.opsForHyperLogLog().size("hll:visits");

        // 通用操作
        Boolean exists = template.hasKey("str:key");
        Boolean expired = template.expire("str:key", Duration.ofMinutes(10));
        Long ttl = template.getExpire("str:key");
        Boolean deleted = template.delete("str:key");
        org.springframework.data.redis.core.DataType type = template.type("hash:key");
    }
}
```

---

## 3.6 RedisTemplate的RedisSerializer

- 默认 JDK 序列化可读性差。
- 常见替代：String、JSON（Jackson）。
- 需要在配置类中设置 key/value 的序列化器。

String 序列化（全 String KV）：
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.StringRedisSerializer;

@Configuration
public class RedisStringSerializerConfig {
		@Bean
		public RedisTemplate<String, String> stringRedisTemplate(RedisConnectionFactory factory) {
				RedisTemplate<String, String> template = new RedisTemplate<>();
				template.setConnectionFactory(factory);

				StringRedisSerializer stringSerializer = new StringRedisSerializer();
				template.setKeySerializer(stringSerializer);
				template.setValueSerializer(stringSerializer);
				template.setHashKeySerializer(stringSerializer);
				template.setHashValueSerializer(stringSerializer);

				template.afterPropertiesSet();
				return template;
		}
}
```

JSON 序列化（对象/复杂值）：
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

@Configuration
public class RedisJsonSerializerConfig {
		@Bean
		public RedisTemplate<String, Object> jsonRedisTemplate(RedisConnectionFactory factory) {
				RedisTemplate<String, Object> template = new RedisTemplate<>();
				template.setConnectionFactory(factory);

				StringRedisSerializer keySerializer = new StringRedisSerializer();
				GenericJackson2JsonRedisSerializer valueSerializer =
						new GenericJackson2JsonRedisSerializer();

				template.setKeySerializer(keySerializer);
				template.setValueSerializer(valueSerializer);
				template.setHashKeySerializer(keySerializer);
				template.setHashValueSerializer(valueSerializer);

				template.afterPropertiesSet();
				return template;
		}
}
```

配置方式要点：
- String 类型：优先直接使用 `StringRedisTemplate`；需要自定义时，定义 `RedisTemplate<String, String>` 并设置 `StringRedisSerializer`。
- JSON 类型：定义 `RedisTemplate<String, Object>`，Key 用 `StringRedisSerializer`，Value/HashValue 用 `GenericJackson2JsonRedisSerializer`。
- 放在 `@Configuration` 类中注册为 `@Bean`，Spring Boot 自动装配后即可注入使用。

### 3.6.1 序列化方案对比

| 序列化器 | Key 可读性 | Value 可读性 | 跨语言 | 空间占用 | 反序列化要求 | 推荐场景 |
|---|---|---|---|---|---|---|
| `JdkSerializationRedisSerializer` | 差 | 差 | 否 | 大 | 同版本 JDK | 简单原型（不推荐） |
| `StringRedisSerializer` | 高 | 高 | 是 | 中 | 无 | 纯字符串 KV |
| `GenericJackson2JsonRedisSerializer` | — | 高（含类型信息） | 是 | 较大 | 同包名类 | 通用 JSON 对象 |
| `Jackson2JsonRedisSerializer` | — | 高（不含类型信息） | 是 | 中 | 需指定 Class | 固定类型 JSON |
| `ProtobufRedisSerializer` / `KryoRedisSerializer` | — | 差（二进制） | 否 | 小 | 需 Schema | 高性能场景 |

### 3.6.2 Jackson2JsonRedisSerializer vs GenericJackson2JsonRedisSerializer

- **GenericJackson2JsonRedisSerializer**：JSON 中附带 `@class` 类型信息，反序列化时自动识别类型，无需手动指定 Class。缺点：额外存储类型信息，增大约 30% 体积。
- **Jackson2JsonRedisSerializer**：纯 JSON，不存储类型信息，反序列化时需手动指定 Class（如 `new Jackson2JsonRedisSerializer<>(User.class)`）。优点：体积小，适合固定类型的缓存。

### 3.6.3 自定义 ObjectMapper（支持 Java 8 时间）

```java
import com.fasterxml.jackson.annotation.JsonAutoDetect;
import com.fasterxml.jackson.annotation.PropertyAccessor;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.databind.jsontype.impl.LaissezFaireSubTypeValidator;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

@Configuration
public class RedisCustomJsonConfig {

    @Bean
    public RedisTemplate<String, Object> customJsonRedisTemplate(RedisConnectionFactory factory) {
        // 构建自定义 ObjectMapper
        ObjectMapper mapper = new ObjectMapper();

        // 注册 Java 8 时间模块（支持 LocalDate, LocalDateTime 等）
        mapper.registerModule(new JavaTimeModule());

        // 禁用将日期序列化为时间戳
        mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);

        // 序列化所有字段（包括无 getter 的字段）
        mapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);

        // 启用类型信息（用于 GenericJackson2JsonRedisSerializer 反序列化时自动识别类型）
        mapper.activateDefaultTyping(
            LaissezFaireSubTypeValidator.instance,
            ObjectMapper.DefaultTyping.NON_FINAL
        );

        GenericJackson2JsonRedisSerializer valueSerializer =
            new GenericJackson2JsonRedisSerializer(mapper);

        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);

        StringRedisSerializer keySerializer = new StringRedisSerializer();
        template.setKeySerializer(keySerializer);
        template.setValueSerializer(valueSerializer);
        template.setHashKeySerializer(keySerializer);
        template.setHashValueSerializer(valueSerializer);

        template.afterPropertiesSet();
        return template;
    }
}
```

---

## 3.7 StringRedisTemplate（SpringBoot提供，**其继承**RedisTemplate<String, String>）

- 仅处理 String 类型，默认 String 序列化。
- 适合简单 KV 场景。

### 3.7.1 更多使用示例

```java
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.Arrays;
import java.util.List;

@Service
public class StringRedisService {

    private final StringRedisTemplate template;

    public StringRedisService(StringRedisTemplate template) {
        this.template = template;
    }

    // 基本 SET/GET
    public String getValue(String key) {
        return template.opsForValue().get(key);
    }

    public void setValue(String key, String value, long ttlSeconds) {
        template.opsForValue().set(key, value, Duration.ofSeconds(ttlSeconds));
    }

    // 批量操作
    public List<String> multiGet(List<String> keys) {
        return template.opsForValue().multiGet(keys);
    }

    // INCR 原子计数
    public Long incr(String key) {
        return template.opsForValue().increment(key);
    }

    // 位图操作
    public void bitSet(String key, long offset, boolean value) {
        template.opsForValue().setBit(key, offset, value);
    }

    public Boolean bitGet(String key, long offset) {
        return template.opsForValue().getBit(key, offset);
    }

    // 删除 Key
    public Boolean delete(String key) {
        return template.delete(key);
    }

    // 批量删除
    public Long deleteAll(List<String> keys) {
        return template.delete(keys);
    }
}
```

---

## 3.8 RedisTemplate操作Hash类型

```java
import org.springframework.data.redis.core.HashOperations;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

@Service
public class RedisHashService {
		private final StringRedisTemplate template;

		public RedisHashService(StringRedisTemplate template) {
				this.template = template;
		}

		public void saveUser() {
				HashOperations<String, Object, Object> ops = template.opsForHash();
				ops.put("user:2", "name", "lucy");
				ops.put("user:2", "age", "20");
		}
}
```

### 3.8.1 更多 Hash 操作示例

```java
@Service
public class RedisHashAdvancedService {

    private final StringRedisTemplate template;

    public RedisHashAdvancedService(StringRedisTemplate template) {
        this.template = template;
    }

    public void hashAdvancedOps() {
        HashOperations<String, Object, Object> ops = template.opsForHash();

        // 批量 PUT
        ops.putAll("user:3", java.util.Map.of(
            "name", "张三",
            "email", "zhangsan@example.com",
            "phone", "13800138000"
        ));

        // GET 单个字段
        Object name = ops.get("user:3", "name");

        // MULTIGET 批量获取
        java.util.List<Object> fields = ops.multiGet("user:3",
            java.util.Arrays.asList("name", "email", "phone"));

        // GET ALL
        java.util.Map<Object, Object> all = ops.entries("user:3");

        // DELETE 字段
        ops.delete("user:3", "phone");

        // INCREMENT 数值字段
        ops.increment("user:3", "login_count", 1L);

        // SCAN 遍历
        try (org.springframework.data.redis.core.Cursor<java.util.Map.Entry<Object, Object>> cursor =
                ops.scan("user:3", org.springframework.data.redis.core.ScanOptions
                    .scanOptions().match("*").count(10).build())) {
            while (cursor.hasNext()) {
                java.util.Map.Entry<Object, Object> entry = cursor.next();
                System.out.println(entry.getKey() + ": " + entry.getValue());
            }
        }

        // EXISTS
        Boolean hasEmail = ops.hasKey("user:3", "email");
    }
}
```

---

## 3.9 RedisTemplate操作List / Set / ZSet

### 3.9.1 RedisTemplate操作List

```java
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.concurrent.TimeUnit;

@Service
public class RedisTemplateListService {

    private final StringRedisTemplate template;

    public RedisTemplateListService(StringRedisTemplate template) {
        this.template = template;
    }

    public void listOperations() {
        var ops = template.opsForList();

        // LPUSH — 从左侧推入
        ops.leftPush("mylist", "c");
        ops.leftPush("mylist", "b");
        ops.leftPush("mylist", "a");
        // mylist: [a, b, c]

        // LPUSH 批量推入
        ops.leftPushAll("mylist", "d", "e", "f");
        // mylist: [f, e, d, a, b, c]

        // RPUSH — 从右侧推入
        ops.rightPush("mylist", "x");
        ops.rightPushAll("mylist", "y", "z");

        // LPOP / RPOP — 弹出
        String leftPopped = ops.leftPop("mylist");
        String rightPopped = ops.rightPop("mylist");

        // BLPOP — 阻塞弹出
        String blockingPop = ops.leftPop("mylist", 30, TimeUnit.SECONDS);

        // LRANGE — 范围查询
        List<String> all = ops.range("mylist", 0, -1);
        List<String> firstThree = ops.range("mylist", 0, 2);

        // LINDEX — 按索引取值
        String first = ops.index("mylist", 0);
        String last = ops.index("mylist", -1);

        // LLEN — 长度
        Long size = ops.size("mylist");

        // LREM — 删除元素
        ops.remove("mylist", 1, "x");

        // LTRIM — 裁剪
        ops.trim("mylist", 0, 99);

        // LSET — 按索引设值
        ops.set("mylist", 0, "newFirst");
    }
}
```

### 3.9.2 RedisTemplate操作Set

```java
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

import java.util.Set;

@Service
public class RedisTemplateSetService {

    private final StringRedisTemplate template;

    public RedisTemplateSetService(StringRedisTemplate template) {
        this.template = template;
    }

    public void setOperations() {
        var ops = template.opsForSet();

        // SADD
        ops.add("set:tech", "java", "golang", "python", "rust");

        // SMEMBERS
        Set<String> members = ops.members("set:tech");

        // SISMEMBER
        Boolean isJava = ops.isMember("set:tech", "java");

        // SCARD
        Long size = ops.size("set:tech");

        // SRANDMEMBER
        String random = ops.randomMember("set:tech");
        Set<String> randomTwo = ops.randomMembers("set:tech", 2);
        // 获取一个随机元素（不重复）:
        Set<String> distinctRandoms = ops.distinctRandomMembers("set:tech", 3);

        // SPOP
        String popped = ops.pop("set:tech");
        Set<String> poppedTwo = ops.pop("set:tech", 2);

        // SMOVE
        ops.move("set:tech", "java", "set:favorite");

        // SREM
        ops.remove("set:tech", "python");

        // 集合运算：交集
        Set<String> inter = ops.intersect("set:tech", "set:favorite");

        // 集合运算：并集
        Set<String> union = ops.union("set:tech", "set:favorite");

        // 集合运算：差集
        Set<String> diff = ops.difference("set:tech", "set:favorite");

        // 集合并存储结果
        ops.intersectAndStore("set:tech", "set:favorite", "set:common");
        ops.unionAndStore("set:tech", "set:favorite", "set:all");
        ops.differenceAndStore("set:tech", "set:favorite", "set:onlyTech");
    }
}
```

### 3.9.3 RedisTemplate操作ZSet

```java
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.ZSetOperations;
import org.springframework.stereotype.Service;

import java.util.Set;

@Service
public class RedisTemplateZSetService {

    private final StringRedisTemplate template;

    public RedisTemplateZSetService(StringRedisTemplate template) {
        this.template = template;
    }

    public void zSetOperations() {
        var ops = template.opsForZSet();

        // ZADD 单个
        ops.add("leaderboard", "Alice", 95.0);

        // ZADD 批量
        Set<ZSetOperations.TypedTuple<String>> tuples = new java.util.HashSet<>();
        tuples.add(org.springframework.data.redis.core.DefaultTypedTuple.of("Bob", 88.5));
        tuples.add(org.springframework.data.redis.core.DefaultTypedTuple.of("Charlie", 92.0));
        tuples.add(org.springframework.data.redis.core.DefaultTypedTuple.of("David", 78.0));
        ops.add("leaderboard", tuples);

        // ZINCRBY
        ops.incrementScore("leaderboard", "Alice", 5.0);

        // ZRANGE（升序）
        Set<String> asc = ops.range("leaderboard", 0, -1);

        // ZREVRANGE（降序）
        Set<String> desc = ops.reverseRange("leaderboard", 0, 2);

        // ZRANGE WITHSCORES
        Set<ZSetOperations.TypedTuple<String>> withScores =
            ops.rangeWithScores("leaderboard", 0, 2);
        for (ZSetOperations.TypedTuple<String> t : withScores) {
            System.out.println(t.getValue() + " -> " + t.getScore());
        }

        // ZREVRANGE WITHSCORES
        Set<ZSetOperations.TypedTuple<String>> top3WithScores =
            ops.reverseRangeWithScores("leaderboard", 0, 2);

        // ZRANK（升序排名，从 0 开始）
        Long aliceRank = ops.rank("leaderboard", "Alice");

        // ZREVRANK（降序排名，从 0 开始）
        Long aliceRevRank = ops.reverseRank("leaderboard", "Alice");

        // ZSCORE
        Double aliceScore = ops.score("leaderboard", "Alice");

        // ZCOUNT
        Long count80to100 = ops.count("leaderboard", 80, 100);

        // ZRANGEBYSCORE
        Set<String> range80to100 = ops.rangeByScore("leaderboard", 80, 100);
        // 分页
        Set<String> page1 = ops.rangeByScore("leaderboard", 80, 100, 0, 10);

        // ZPOPMIN / ZPOPMAX
        ZSetOperations.TypedTuple<String> min = ops.popMin("leaderboard");
        ZSetOperations.TypedTuple<String> max = ops.popMax("leaderboard");

        // ZREM — 移除元素
        ops.remove("leaderboard", "David");
        // 按排名移除
        ops.removeRange("leaderboard", 0, 9);
        // 按分数移除
        ops.removeRangeByScore("leaderboard", 0, 60);

        // ZCARD
        Long size = ops.size("leaderboard");
    }
}
```

---

## 3.10 第一个完整Redis项目：用户信息缓存

以下是一个完整的 Spring Boot 项目，演示如何使用 Redis 缓存用户信息。

### 3.10.1 Maven 依赖 (pom.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.5</version>
        <relativePath/>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>redis-user-cache</artifactId>
    <version>1.0.0</version>
    <name>redis-user-cache</name>

    <properties>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <!-- Spring Boot Web -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Spring Data Redis -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>

        <!-- Lettuce 连接池支持 -->
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-pool2</artifactId>
        </dependency>

        <!-- Jackson Java 8 时间支持 -->
        <dependency>
            <groupId>com.fasterxml.jackson.datatype</groupId>
            <artifactId>jackson-datatype-jsr310</artifactId>
        </dependency>

        <!-- Lombok（简化实体类代码） -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- Spring Boot Test -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

### 3.10.2 配置文件 (application.yml)

```yaml
# src/main/resources/application.yml
server:
  port: 8080

spring:
  data:
    redis:
      host: localhost
      port: 6379
      password:           # 如果设置了密码，在此填写：mypassword
      database: 0
      timeout: 3000ms
      lettuce:
        pool:
          max-active: 20
          max-idle: 10
          min-idle: 5
          max-wait: 2000ms

logging:
  level:
    com.example: DEBUG
```

### 3.10.3 用户实体 (User.java)

```java
package com.example.entity;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.io.Serializable;
import java.time.LocalDateTime;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class User implements Serializable {

    private static final long serialVersionUID = 1L;

    private Long id;
    private String username;
    private String email;
    private String phone;
    private Integer age;
    private LocalDateTime createTime;
}
```

### 3.10.4 Redis 配置类 (RedisConfig.java)

```java
package com.example.config;

import com.fasterxml.jackson.annotation.JsonAutoDetect;
import com.fasterxml.jackson.annotation.PropertyAccessor;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.databind.jsontype.impl.LaissezFaireSubTypeValidator;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        ObjectMapper mapper = new ObjectMapper();

        // 支持 Java 8 时间
        mapper.registerModule(new JavaTimeModule());
        mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
        mapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        mapper.activateDefaultTyping(
            LaissezFaireSubTypeValidator.instance,
            ObjectMapper.DefaultTyping.NON_FINAL
        );

        GenericJackson2JsonRedisSerializer valueSerializer =
            new GenericJackson2JsonRedisSerializer(mapper);
        StringRedisSerializer keySerializer = new StringRedisSerializer();

        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        template.setKeySerializer(keySerializer);
        template.setValueSerializer(valueSerializer);
        template.setHashKeySerializer(keySerializer);
        template.setHashValueSerializer(valueSerializer);
        template.afterPropertiesSet();

        return template;
    }
}
```

### 3.10.5 用户服务 (UserService.java)

```java
package com.example.service;

import com.example.entity.User;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.time.LocalDateTime;
import java.util.Map;
import java.util.concurrent.TimeUnit;

@Service
public class UserService {

    private static final String USER_KEY_PREFIX = "user:";
    private static final Duration USER_CACHE_TTL = Duration.ofHours(2);

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    /**
     * 根据 ID 获取用户（先查缓存，缓存未命中则查数据库）c
     */
    public User getUserById(Long userId) {
        String key = USER_KEY_PREFIX + userId + ":info";

        // 1. 先查缓存
        User cachedUser = (User) redisTemplate.opsForValue().get(key);
        if (cachedUser != null) {
            System.out.println("缓存命中: " + key);
            return cachedUser;
        }

        // 2. 缓存未命中，查数据库（模拟）
        System.out.println("缓存未命中，查询数据库: " + key);
        User dbUser = queryUserFromDatabase(userId);

        // 3. 写入缓存
        if (dbUser != null) {
            redisTemplate.opsForValue().set(key, dbUser, USER_CACHE_TTL);
            System.out.println("用户信息已缓存: " + key);
        }

        return dbUser;
    }

    /**
     * 创建或更新用户，同时更新缓存
     */
    public User saveUser(User user) {
        // 1. 持久化到数据库（模拟）
        System.out.println("保存用户到数据库: " + user.getId());

        // 2. 更新缓存
        String key = USER_KEY_PREFIX + user.getId() + ":info";
        redisTemplate.opsForValue().set(key, user, USER_CACHE_TTL);

        return user;
    }

    /**
     * 删除用户，同时删除缓存
     */
    public void deleteUser(Long userId) {
        // 1. 从数据库删除（模拟）
        System.out.println("从数据库删除用户: " + userId);

        // 2. 删除缓存
        String key = USER_KEY_PREFIX + userId + ":info";
        redisTemplate.delete(key);
        System.out.println("缓存已删除: " + key);
    }

    /**
     * 模拟从数据库查询
     */
    private User queryUserFromDatabase(Long userId) {
        // 模拟数据库延迟
        try {
            TimeUnit.MILLISECONDS.sleep(200);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        // 模拟一条数据库记录
        User user = new User();
        user.setId(userId);
        user.setUsername("user_" + userId);
        user.setEmail("user" + userId + "@example.com");
        user.setPhone("1380013" + String.format("%04d", userId));
        user.setAge(20 + (int)(userId % 30));
        user.setCreateTime(LocalDateTime.of(2024, 1, 15, 10, 0).plusDays(userId));
        return user;
    }

    /**
     * 查看缓存中的原始 key
     */
    public Map<Object, Object> getUserFromHash(Long userId) {
        String key = USER_KEY_PREFIX + userId;
        return redisTemplate.opsForHash().entries(key);
    }
}
```

### 3.10.6 用户控制器 (UserController.java)

```java
package com.example.controller;

import com.example.entity.User;
import com.example.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/users")
public class UserController {

    @Autowired
    private UserService userService;

    /**
     * GET /api/users/{id} — 查询用户
     * 示例：curl http://localhost:8080/api/users/1
     */
    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        User user = userService.getUserById(id);
        if (user == null) {
            return ResponseEntity.notFound().build();
        }
        return ResponseEntity.ok(user);
    }

    /**
     * POST /api/users — 创建/更新用户
     * 示例：curl -X POST http://localhost:8080/api/users \
     *        -H "Content-Type: application/json" \
     *        -d '{"id":1,"username":"tom","email":"tom@example.com","phone":"13800000000","age":25}'
     */
    @PostMapping
    public ResponseEntity<User> saveUser(@RequestBody User user) {
        User saved = userService.saveUser(user);
        return ResponseEntity.ok(saved);
    }

    /**
     * DELETE /api/users/{id} — 删除用户
     * 示例：curl -X DELETE http://localhost:8080/api/users/1
     */
    @DeleteMapping("/{id}")
    public ResponseEntity<String> deleteUser(@PathVariable Long id) {
        userService.deleteUser(id);
        return ResponseEntity.ok("Deleted: " + id);
    }

    /**
     * GET /api/users/{id}/from-hash — 查看 Hash 形式的缓存
     */
    @GetMapping("/{id}/from-hash")
    public ResponseEntity<?> getUserFromHash(@PathVariable Long id) {
        return ResponseEntity.ok(userService.getUserFromHash(id));
    }
}
```

### 3.10.7 启动类 (RedisUserCacheApplication.java)

```java
package com.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class RedisUserCacheApplication {
    public static void main(String[] args) {
        SpringApplication.run(RedisUserCacheApplication.class, args);
    }
}
```

### 3.10.8 项目结构与测试

项目目录结构：
```
redis-user-cache/
├── pom.xml
└── src/main/
    ├── java/com/example/
    │   ├── RedisUserCacheApplication.java    # 启动类
    │   ├── config/
    │   │   └── RedisConfig.java              # Redis 配置
    │   ├── entity/
    │   │   └── User.java                     # 用户实体
    │   ├── service/
    │   │   └── UserService.java              # 业务逻辑
    │   └── controller/
    │       └── UserController.java           # REST 控制器
    └── resources/
        └── application.yml                   # 配置文件
```

**使用 Postman 或 curl 测试**：

```bash
# 1. 创建用户
curl -X POST http://localhost:8080/api/users \
  -H "Content-Type: application/json" \
  -d '{
    "id": 1,
    "username": "tom",
    "email": "tom@example.com",
    "phone": "13800000001",
    "age": 25
  }'

# 2. 查询用户（第一次查数据库，之后命中缓存）
curl http://localhost:8080/api/users/1

# 3. 再次查询（应命中缓存）
curl http://localhost:8080/api/users/1

# 4. 更新用户
curl -X POST http://localhost:8080/api/users \
  -H "Content-Type: application/json" \
  -d '{
    "id": 1,
    "username": "tom-updated",
    "email": "new_email@example.com",
    "phone": "13800000001",
    "age": 26
  }'

# 5. 删除用户（删除数据库记录和缓存）
curl -X DELETE http://localhost:8080/api/users/1

# 6. 验证缓存已清除
curl http://localhost:8080/api/users/1

# 7. 直接通过 redis-cli 查看缓存中的 key
# redis-cli KEYS "user:*"
```

---

> **全文完。本文涵盖了 Redis 从基础概念、安装部署、五大数据类型的完整命令体系，到 Java 三大客户端（Jedis/Lettuce/Spring Data Redis）的实战应用。**
>
> **推荐学习路径**：
> 1. 搭建本地 Redis 环境（Docker 或 Linux）
> 2. 熟记五大数据类型的核心命令
> 3. 先用 Jedis 手写 CRUD 理解原理
> 4. 再切换到 Spring Data Redis 实现生产级代码
> 5. 最后学习 Redis 持久化、集群、分布式锁等进阶主题
