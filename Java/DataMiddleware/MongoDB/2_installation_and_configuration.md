
# 1. 安装方式概览

| 安装方式 | 适用场景 | 复杂度 |
|---------|---------|--------|
| 二进制包安装 | 生产环境Linux部署 | 中 |
| 包管理器安装（apt/yum） | Linux快速安装 | 低 |
| Docker安装 | 开发测试、CI/CD | 低 |
| Docker Compose部署集群 | 开发环境模拟生产 | 中 |
| Windows MSI安装 | Windows开发 | 低 |
| MongoDB Atlas云服务 | 不想管理运维 | 最低 |

# 2. Windows安装

## 2.1 下载与安装

从MongoDB官网下载MSI安装包，选择对应版本（推荐7.0+或最新稳定版）：

```
https://www.mongodb.com/try/download/community
```

安装时选择"Complete"完整安装，勾选"Install MongoDB as a Service"将MongoDB安装为Windows服务。

## 2.2 配置Windows服务

默认安装路径：`C:\Program Files\MongoDB\Server\<version>\`

默认数据目录：`C:\Program Files\MongoDB\Server\<version>\data\`

如需自定义配置，编辑`mongod.cfg`文件：

```yaml
# mongod.cfg
systemLog:
  destination: file
  logAppend: true
  path: D:\mongo\log\mongod.log

storage:
  dbPath: D:\mongo\data

net:
  port: 27017
  bindIp: 127.0.0.1

# 可选：开启安全认证
# security:
#   authorization: enabled
```

## 2.3 启动与连接

```powershell
# 启动MongoDB服务
net start MongoDB

# 停止MongoDB服务
net stop MongoDB

# 连接
mongosh
```

# 3. Docker安装

## 3.1 单节点快速启动

```bash
# 拉取镜像
docker pull mongo:7.0

# 运行容器（无认证，仅开发使用）
docker run -d \
  --name mongodb \
  -p 27017:27017 \
  -v mongo_data:/data/db \
  -v mongo_config:/data/configdb \
  mongo:7.0

# 运行容器（带认证）
docker run -d \
  --name mongodb \
  -p 27017:27017 \
  -v mongo_data:/data/db \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=admin123 \
  mongo:7.0
```

## 3.2 Docker Compose部署副本集

```yaml
# docker-compose.yml
version: '3.8'
services:
  mongo1:
    image: mongo:7.0
    container_name: mongo1
    ports:
      - "27017:27017"
    volumes:
      - mongo1_data:/data/db
      - mongo1_config:/data/configdb
    command: ["--replSet", "rs0", "--bind_ip_all"]
    networks:
      - mongo_net

  mongo2:
    image: mongo:7.0
    container_name: mongo2
    ports:
      - "27018:27017"
    volumes:
      - mongo2_data:/data/db
      - mongo2_config:/data/configdb
    command: ["--replSet", "rs0", "--bind_ip_all"]
    networks:
      - mongo_net

  mongo3:
    image: mongo:7.0
    container_name: mongo3
    ports:
      - "27019:27017"
    volumes:
      - mongo3_data:/data/db
      - mongo3_config:/data/configdb
    command: ["--replSet", "rs0", "--bind_ip_all"]
    networks:
      - mongo_net

volumes:
  mongo1_data:
  mongo1_config:
  mongo2_data:
  mongo2_config:
  mongo3_data:
  mongo3_config:

networks:
  mongo_net:
    driver: bridge
```

启动后初始化副本集：

```bash
# 进入mongo1容器
docker exec -it mongo1 mongosh

# 初始化副本集
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo1:27017", priority: 2 },
    { _id: 1, host: "mongo2:27017", priority: 1 },
    { _id: 2, host: "mongo3:27017", priority: 1 }
  ]
})
```

# 4. 核心配置文件详解

## 4.1 配置文件结构

MongoDB 3.2+使用YAML格式的配置文件，默认路径：

| 操作系统 | 路径 |
|---------|------|
| Linux | /etc/mongod.conf |
| macOS（Homebrew） | /usr/local/etc/mongod.conf |
| Windows | <安装目录>/bin/mongod.cfg |

## 4.2 完整配置示例

```yaml
# ========================================
# 系统日志配置
# ========================================
systemLog:
  verbosity: 0                          # 日志级别：0(默认)~5(最详细)
  path: /var/log/mongodb/mongod.log     # 日志文件路径
  logAppend: true                       # 日志追加而非覆盖
  logRotate: rename                     # 日志轮转方式：rename或reopen
  destination: file                     # file / syslog
  timeStampFormat: iso8601-utc          # 时间戳格式

# ========================================
# 存储引擎配置
# ========================================
storage:
  dbPath: /data/db                      # 数据文件存储路径
  journal:
    enabled: true                       # 开启日志（强烈建议开启）
    commitIntervalMs: 100               # journal日志提交间隔（ms）
  directoryPerDB: false                 # 每个数据库独立目录
  syncPeriodSecs: 60                    # 数据刷盘间隔（s）
  engine: wiredTiger                    # 存储引擎
  wiredTiger:
    engineConfig:
      cacheSizeGB: 1                    # WiredTiger缓存大小
      journalCompressor: snappy         # 日志压缩算法
      directoryForIndexes: false        # 索引单独目录
    collectionConfig:
      blockCompressor: snappy           # 数据压缩算法：snappy/zlib/zstd/none
      prefixCompression: true           # 键前缀压缩
    indexConfig:
      prefixCompression: true           # 索引前缀压缩

# ========================================
# 网络配置
# ========================================
net:
  port: 27017                           # 监听端口
  bindIp: 127.0.0.1                     # 绑定IP（0.0.0.0表示所有网卡）
  maxIncomingConnections: 65536         # 最大连接数
  wireObjectCheck: true                 # 检查文档大小
  ipv6: false                           # 启用IPv6
  unixDomainSocket:
    enabled: true                       # Unix套接字
    pathPrefix: /tmp
  tls:
    mode: disabled                      # TLS模式：disabled/allowTLS/preferTLS/requireTLS
    certificateKeyFile: /path/to/cert.pem
    CAFile: /path/to/ca.pem

# ========================================
# 安全配置
# ========================================
security:
  authorization: enabled                # 开启认证
  javascriptEnabled: true               # 启用服务端JavaScript执行
  keyFile: /path/to/keyfile             # 副本集认证密钥文件

# ========================================
# 进程管理
# ========================================
processManagement:
  fork: true                            # 后台运行（Linux）
  pidFilePath: /var/run/mongodb/mongod.pid
  timeZoneInfo: /usr/share/zoneinfo

# ========================================
# 操作分析（慢查询）
# ========================================
operationProfiling:
  mode: slowOp                          # off / slowOp / all
  slowOpThresholdMs: 100                # 慢查询阈值（ms）
  slowOpSampleRate: 1.0                 # 慢查询采样率（0.0~1.0）

# ========================================
# 副本集配置
# ========================================
replication:
  oplogSizeMB: 1024                     # oplog大小（MB）
  replSetName: rs0                      # 副本集名称
  enableMajorityReadConcern: true       # 启用majority读关注

# ========================================
# 分片配置
# ========================================
sharding:
  clusterRole: configsvr                # shardsvr / configsvr
  archiveMovedChunks: true              # 迁移chunk时保留文档

# ========================================
# 审计日志
# ========================================
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit.log
  filter: '{ atype: { $in: [ "authenticate", "dropDatabase" ] } }'

# ========================================
# SNMP监控（企业版）
# ========================================
snmp:
  disabled: true
```

# 5. WiredTiger存储引擎深入

## 5.1 为什么选择WiredTiger

从MongoDB 3.2开始，WiredTiger成为默认存储引擎，取代了旧的MMAPv1引擎。主要优势：

| 特性 | MMAPv1 | WiredTiger |
|------|--------|-----------|
| 并发控制 | 集合级锁 | 文档级锁（MVCC） |
| 压缩 | 不支持 | 支持snappy/zlib/zstd压缩 |
| 数据持久化 | 双写缓冲区 | Checkpoint + Journal |
| 内存使用 | 内存映射文件 | 独立缓存管理 |
| 磁盘空间 | 预分配+碎片 | 压缩+紧凑存储 |

## 5.2 MVCC并发控制机制

WiredTiger使用MVCC（多版本并发控制）实现文档级并发：

- 写操作不会阻塞读操作（读写分离）
- 多个写操作可以并发修改不同文档
- 每个操作看到的是一个一致的数据快照
- 通过事务ID和快照来管理可见性

核心数据结构：
```
┌─────────────────┐
│  WiredTiger      │
│  Cache (内存)    │  ← 热数据、索引
│  ┌─────────────┐ │
│  │ B-Tree       │ │  ← 索引和数据组织的核心结构
│  │ 内部节点     │ │
│  │ 叶子节点     │ │  ← 存放实际数据页
│  └─────────────┘ │
└─────────────────┘
         ↕ 页换入换出
┌─────────────────┐
│  磁盘文件        │
│  collection-*.wt │  ← 集合数据文件
│  index-*.wt      │  ← 索引文件
│  WiredTiger.wt   │  ← 元数据
│  WiredTiger.lock │  ← 锁文件
│  journal/        │  ← 预写日志
└─────────────────┘
```

## 5.3 Checkpoint与Journal

**Checkpoint（检查点）**：
- 每隔60秒（或日志写满2GB时）创建一个数据快照
- 将内存中所有已修改的数据页刷新到磁盘
- 检查点是数据库崩溃恢复的起点
- 使用copy-on-write技术，不阻塞读写

**Journal（预写日志 WAL）**：
- 在数据写入数据文件之前先写入journal日志
- 记录两次checkpoint之间的所有写操作
- 崩溃恢复时从最近checkpoint开始，重放journal中的操作
- 最多丢失100ms的数据（默认commitIntervalMs配置）

```
写操作流程：
  客户端写入 → 内存更新 → Journal日志 → (异步) → Checkpoint到磁盘
                                              ↑
                                     WriteConcern {j:true} 在此等待
```

## 5.4 数据压缩

MongoDB支持多种压缩算法来减少磁盘占用：

| 压缩算法 | 压缩比 | CPU消耗 | 适用场景 |
|---------|--------|--------|---------|
| snappy | 60-80% | 低 | 通用场景（默认） |
| zlib | 80-90% | 高 | 冷数据、存储优先 |
| zstd | 80-90% | 中 | zlib的现代替代（6.0+支持） |
| none | 0% | 无 | 纯内存、极致性能 |

# 6. 内存配置与优化

## 6.1 WiredTiger缓存

WiredTiger缓存大小是影响性能的最关键参数之一：

```yaml
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 1   # 默认：max(0.5 * (RAM - 1GB), 256MB)
```

计算建议：
- 缓存应能容纳**热数据 + 索引**的工作集
- 如果数据远小于内存，设为数据大小即可
- 如果数据远大于内存，设内存的50%~70%（为操作系统留足够内存）
- 避免缓存过大导致操作系统频繁换页

## 6.2 内存使用构成

```
总内存占用 = WiredTiger缓存 
           + 连接内存（每个连接 ~1MB）
           + 操作系统文件系统缓存
           + 聚合/排序等临时内存
```

# 7. MongoDB Shell（mongosh）

## 7.1 常用连接方式

```bash
# 本地默认连接
mongosh

# 指定端口
mongosh --port 27018

# 带认证连接
mongosh -u admin -p password --authenticationDatabase admin

# 连接字符串方式
mongosh "mongodb://admin:password@localhost:27017/admin"

# 连接副本集
mongosh "mongodb://mongo1:27017,mongo2:27017,mongo3:27017/?replicaSet=rs0"
```

## 7.2 常用Shell命令

```javascript
// 查看当前数据库
db.getName()

// 切换/创建数据库
use mydb

// 查看所有数据库
show dbs

// 查看当前库的所有集合
show collections

// 查看集合统计信息
db.collection.stats()

// 查看当前操作
db.currentOp()

// 终止某个操作
db.killOp(opid)

// 查看慢查询
db.system.profile.find().sort({ts: -1}).limit(5)

// 执行JavaScript脚本
load("/path/to/script.js")

// 导出查询结果为JSON
db.collection.find().toArray()
```

# 8. 版本选择建议

| 场景 | 推荐版本 | 理由 |
|------|---------|------|
| 生产新项目 | 7.0+ | 最新特性、长期支持 |
| 已有项目维护 | 6.0 | LTS稳定版本 |
| Spring Boot 3.x | 6.0+ | 驱动兼容性 |
| Java 17+项目 | 6.0+ | 驱动支持 |
| 时序数据 | 5.1+ | 原生时序集合 |

针对Java开发，推荐使用MongoDB 6.0或7.0配合MongoDB Java Driver 5.x（sync）或Spring Data MongoDB。
