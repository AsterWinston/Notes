
# 1. 环境准备

## 1.1 依赖要求

| 依赖 | 版本要求 | 说明 |
|------|---------|------|
| JDK | 1.8+ | ZooKeeper 3.5+ 需要 JDK 8；3.6+ 推荐 JDK 11 |
| 操作系统 | Linux / macOS / Windows | Linux 生产环境首选 |
| 内存 | 至少 512MB（推荐 2G+） | 生产环境建议 4G 以上堆内存 |
| 磁盘 | 至少 1GB 可用（SSD 优先） | 事务日志写磁盘，SSD 可显著提升性能 |

## 1.2 下载 ZooKeeper

访问 [Apache ZooKeeper Releases](https://zookeeper.apache.org/releases.html) 下载最新稳定版本。

```bash
# 下载（以 3.9.2 为例）
wget https://dlcdn.apache.org/zookeeper/zookeeper-3.9.2/apache-zookeeper-3.9.2-bin.tar.gz

# 解压
tar -xzf apache-zookeeper-3.9.2-bin.tar.gz -C /usr/local/
ln -s /usr/local/apache-zookeeper-3.9.2-bin /usr/local/zookeeper
```

> **注意**：一定要下载 `-bin.tar.gz` 结尾的文件（包含所有依赖），不要下载源码包。

# 2. 单机模式部署

## 2.1 基础配置

```bash
# 创建数据目录
mkdir -p /var/lib/zookeeper/data

# 复制配置文件
cd /usr/local/zookeeper/conf
cp zoo_sample.cfg zoo.cfg
```

编辑 `zoo.cfg`：

```properties
# 基本时间单位，毫秒。用于心跳和超时，默认 2000ms
tickTime=2000

# Follower 连接 Leader 的初始化超时时间（tickTime 的倍数）
# initLimit * tickTime = 10 * 2000ms = 20s
initLimit=10

# Follower 同步 Leader 数据的超时时间（tickTime 的倍数）
# syncLimit * tickTime = 5 * 2000ms = 10s
syncLimit=5

# 数据目录（内存数据库快照存放位置）
dataDir=/var/lib/zookeeper/data

# 事务日志目录（推荐与 dataDir 分开，放在独立磁盘上）
dataLogDir=/var/lib/zookeeper/log

# 客户端连接端口
clientPort=2181

# 单节点最大客户端连接数
maxClientCnxns=60

# 自动清理快照和事务日志
autopurge.snapRetainCount=3
autopurge.purgeInterval=1

# 管理员服务端口（ZooKeeper 3.5+）
admin.serverPort=8080
```

## 2.2 启动与验证

```bash
# 启动 ZooKeeper
cd /usr/local/zookeeper
bin/zkServer.sh start

# 查看状态
bin/zkServer.sh status

# 预期输出：
# ZooKeeper JMX enabled by default
# Using config: /usr/local/zookeeper/bin/../conf/zoo.cfg
# Mode: standalone
```

## 2.3 连接测试

```bash
# 使用自带 CLI 连接
bin/zkCli.sh -server 127.0.0.1:2181

# 连接成功后进入交互模式
[zk: 127.0.0.1:2181(CONNECTED) 0] ls /
[zookeeper]

[zk: 127.0.0.1:2181(CONNECTED) 1] create /hello world
Created /hello

[zk: 127.0.0.1:2181(CONNECTED) 2] get /hello
world

[zk: 127.0.0.1:2181(CONNECTED) 3] quit
```

# 3. 集群模式部署

## 3.1 规划

以 3 节点集群为例：

| 节点 | 主机名 | clientPort | 通信端口 | 选举端口 | myid |
|------|--------|-----------|---------|---------|------|
| Server1 | zk1.example.com | 2181 | 2888 | 3888 | 1 |
| Server2 | zk2.example.com | 2181 | 2888 | 3888 | 2 |
| Server3 | zk3.example.com | 2181 | 2888 | 3888 | 3 |

## 3.2 每节点创建 myid 文件

在每个节点的 `dataDir` 目录下创建 `myid` 文件，内容为该节点的唯一 ID：

```bash
# 在 zk1 上执行
echo "1" > /var/lib/zookeeper/data/myid

# 在 zk2 上执行
echo "2" > /var/lib/zookeeper/data/myid

# 在 zk3 上执行
echo "3" > /var/lib/zookeeper/data/myid
```

> **重要**：`myid` 文件必须与 `zoo.cfg` 中 `server.X` 的 `X` 对应，且内容不能有多余的空白字符。

## 3.3 集群配置文件

所有节点的 `zoo.cfg` 基础配置相同，并增加集群成员配置：

```properties
# ============== 基础配置 ==============
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/var/lib/zookeeper/data
dataLogDir=/var/lib/zookeeper/log
clientPort=2181
maxClientCnxns=60
autopurge.snapRetainCount=3
autopurge.purgeInterval=1
admin.serverPort=8080

# ============== 集群配置 ==============
# 格式：server.X=host:port1:port2[:role]
#   X     — 服务器 ID，对应 myid
#   host  — 服务器地址
#   port1 — Follower 连接 Leader 的端口
#   port2 — Leader 选举端口
#   role  — 可选，值为 observer 或 participant（默认）
server.1=zk1.example.com:2888:3888
server.2=zk2.example.com:2888:3888
server.3=zk3.example.com:2888:3888
```

## 3.4 启动集群

按任意顺序启动所有节点（ZooKeeper 会自动选举 Leader）：

```bash
# 在每个节点上分别执行
bin/zkServer.sh start

# 检查各节点状态
bin/zkServer.sh status

# zk1 可能的输出：
# Mode: leader

# zk2 可能的输出：
# Mode: follower

# zk3 可能的输出：
# Mode: follower
```

## 3.5 集群连接测试

```bash
# 连接集群（指定多个节点，客户端会自动选择可用的连接）
bin/zkCli.sh -server zk1.example.com:2181,zk2.example.com:2181,zk3.example.com:2181
```

# 4. CLI 命令详解

## 4.1 创建节点

```bash
# 创建持久节点
create /config "server port=8080"
# 输出：Created /config

# 创建临时节点（客户端断开后自动删除）
create -e /temp "temp data"

# 创建持久顺序节点
create -s /seq/node "data"
# 输出：Created /seq/node0000000000

# 创建临时顺序节点
create -e -s /seq_ephemeral/node "data"

# 创建节点并指定 ACL
create /secure "secret" digest:user:password:cdrwa

# 创建带 TTL 的持久节点（ZooKeeper 3.5+，需启用扩展类型）
create /ttl_node "data" -t 5000
```

## 4.2 读取节点

```bash
# 获取节点数据
get /config
# 输出：
# server port=8080

# 获取节点数据及 Stat 信息
get -s /config
# 输出：
# server port=8080
# cZxid = 0x100000001
# ctime = Mon Jan 15 10:30:00 CST 2024
# mZxid = 0x100000001
# mtime = Mon Jan 15 10:30:00 CST 2024
# pZxid = 0x100000001
# cversion = 0
# dataVersion = 0
# aclVersion = 0
# ephemeralOwner = 0x0
# dataLength = 17
# numChildren = 0

# 获取节点状态信息（不获取数据）
stat /config

# 列出子节点
ls /config
ls -s /config   # 带 Stat 信息
```

## 4.3 更新节点

```bash
# 更新节点数据（CAS 乐观锁 — 期望版本匹配才更新）
set /config "server port=9090"
set /config "server port=9090" 1   # 期望版本为 1

# 批量更新（需要 Curator 或原生 API，CLI 不支持批量）
```

## 4.4 删除节点

```bash
# 删除节点（不能有子节点）
delete /config

# 删除节点及版本控制
delete /config 1   # 期望版本为 1

# 递归删除所有子节点（ZooKeeper 3.5+）
deleteall /config
```

## 4.5 Watch 操作

```bash
# 监听节点数据变化（一次性）
get /config -w

# 监听子节点变化
ls /config -w

# 监听节点是否存在
stat /config -w

# 在另一个终端中修改 /config：
set /config "new value"
# 第一个终端会收到：
# WATCHER::
# WatchedEvent state:SyncConnected type:NodeDataChanged path:/config
```

## 4.6 权限操作

```bash
# 查看节点 ACL
getAcl /config

# 设置节点 ACL
setAcl /config digest:user:password:cdrwa

# 认证
addauth digest user:password
```

## 4.7 系统命令

```bash
# 查看服务端配置
config -c

# 查看所有客户端连接
cons

# 查看当前 Watch 数量
wchc

# 查看当前 Watch 路径
wchp

# 重置统计
srst

# 查看节点状态快照
stat

# 帮助
help
```

# 5. Docker 部署

## 5.1 单机部署

```bash
# 拉取镜像
docker pull zookeeper:3.9

# 启动
docker run -d \
  --name zk-standalone \
  -p 2181:2181 \
  -p 8080:8080 \
  -e ZOO_MY_ID=1 \
  -e ZOO_SERVERS="server.1=0.0.0.0:2888:3888" \
  zookeeper:3.9
```

## 5.2 Docker Compose 集群部署

创建 `docker-compose.yml`：

```yaml
version: '3.8'

services:
  zoo1:
    image: zookeeper:3.9
    restart: always
    hostname: zoo1
    container_name: zoo1
    ports:
      - "2181:2181"
      - "8081:8080"
    environment:
      ZOO_MY_ID: 1
      ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=zoo3:2888:3888;2181
    volumes:
      - ./data/zoo1/data:/data
      - ./data/zoo1/datalog:/datalog

  zoo2:
    image: zookeeper:3.9
    restart: always
    hostname: zoo2
    container_name: zoo2
    ports:
      - "2182:2181"
      - "8082:8080"
    environment:
      ZOO_MY_ID: 2
      ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=zoo3:2888:3888;2181
    volumes:
      - ./data/zoo2/data:/data
      - ./data/zoo2/datalog:/datalog

  zoo3:
    image: zookeeper:3.9
    restart: always
    hostname: zoo3
    container_name: zoo3
    ports:
      - "2183:2181"
      - "8083:8080"
    environment:
      ZOO_MY_ID: 3
      ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=zoo3:2888:3888;2181
    volumes:
      - ./data/zoo3/data:/data
      - ./data/zoo3/datalog:/datalog
```

```bash
# 启动集群
docker-compose up -d

# 查看状态
docker-compose ps

# 连接测试
docker exec -it zoo1 zkCli.sh -server localhost:2181
```

# 6. 配置参数速查表

| 参数 | 默认值 | 说明 |
|------|-------|------|
| `tickTime` | 2000 | 基本时间单位（ms），心跳和超时都以 tick 为单位 |
| `initLimit` | 10 | Follower 初始化连接 Leader 超时（tickTime 倍数） |
| `syncLimit` | 5 | Follower 与 Leader 同步超时（tickTime 倍数） |
| `dataDir` | — | 数据快照存放目录（必配） |
| `dataLogDir` | dataDir | 事务日志目录（建议独立磁盘） |
| `clientPort` | 2181 | 客户端连接端口 |
| `maxClientCnxns` | 60 | 单个 IP 最大连接数 |
| `autopurge.snapRetainCount` | 3 | 自动清理时保留的快照数量 |
| `autopurge.purgeInterval` | 0 | 自动清理间隔（小时），0 表示禁用 |
| `snapCount` | 100000 | 多少次事务生成一个快照 |
| `globalOutstandingLimit` | 1000 | 最大未完成请求数（Leader） |
| `maxSessionTimeout` | 40 * tickTime | Session 最大超时时间 |
| `minSessionTimeout` | 2 * tickTime | Session 最小超时时间 |
| `admin.serverPort` | 8080 | 管理服务端口（3.5+） |
| `peerType` | participant | 节点类型：participant 或 observer |

# 7. 常见启动问题

## 7.1 myid 文件问题

```bash
# 错误日志：
# "myid file is missing" 或 "myid file is not a number"

# 解决：
echo "1" > /var/lib/zookeeper/data/myid
# 确保 myid 文件只有 ID 数字，无多余空格和换行
```

## 7.2 端口被占用

```bash
# 错误日志：
# "Address already in use"

# 检查端口占用
netstat -tuln | grep -E "2181|2888|3888|8080"

# 修改 zoo.cfg 中的端口号或关闭占用进程
```

## 7.3 集群节点无法通信

```bash
# 错误日志：
# "Cannot open channel to X at election address"

# 检查：
# 1. 防火墙是否开放了 2181/2888/3888 端口
# 2. 主机名是否能正确解析
# 3. server.X 配置是否正确
```

## 7.4 磁盘空间不足

```bash
# 错误日志：
# "No space left on device"

# 解决：
# 1. 清理旧快照和日志：zkCleanup.sh
# 2. 启用自动清理：autopurge.purgeInterval=1
# 3. 扩展磁盘空间
```
