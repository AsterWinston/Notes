
# 1. 环境要求

| 组件 | 版本要求 | 说明 |
|------|---------|------|
| JDK | 11+（推荐 17） | Mycat2 需要 Java 11 以上 |
| MySQL | 5.7+ / 8.0+ | 后端数据库 |
| 内存 | 最小 2GB，推荐 4GB+ | — |
| 操作系统 | Linux / macOS / Windows | 推荐 Linux |

# 2. 下载与安装

## 2.1 直接下载

```bash
# 下载 Mycat2 安装包
wget http://dl.mycat.org.cn/2.0/1.22-release/mycat2-install-template-1.22.zip
unzip mycat2-install-template-1.22.zip

# 下载核心 JAR（核心包与安装包分离）
wget http://dl.mycat.org.cn/2.0/1.22-release/mycat2-1.22-release-jar-with-dependencies.jar
cp mycat2-1.22-release-jar-with-dependencies.jar mycat/lib/

# 目录结构
cd mycat/
tree -L 1
# ├── bin/          # 启动脚本
# ├── conf/         # 配置文件
# ├── lib/          # JAR 依赖
# └── logs/         # 日志文件
```

## 2.2 Docker 部署

```bash
docker run -d \
  --name mycat2 \
  -p 8066:8066 \
  -p 9066:9066 \
  -v /path/to/mycat/conf:/opt/mycat/conf \
  -v /path/to/mycat/logs:/opt/mycat/logs \
  mycat2/mycat2:latest
```

# 3. 后端 MySQL 准备

```sql
-- 在每个后端 MySQL 上创建用户
CREATE USER 'mycat'@'%' IDENTIFIED BY 'mycat123';
GRANT ALL PRIVILEGES ON *.* TO 'mycat'@'%';
FLUSH PRIVILEGES;

-- 创建测试库和表
CREATE DATABASE test_db;
USE test_db;
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    name VARCHAR(50),
    age INT
);
```

# 4. 配置文件

## 4.1 conf/datasources.json — 数据源

```json
{
  "prototype": {
    "name": "prototypeDs",
    "url": "jdbc:mysql://127.0.0.1:3306/test_db?useUnicode=true&serverTimezone=UTC",
    "user": "mycat",
    "password": "mycat123",
    "type": "JDBC",
    "minCon": 2,
    "maxCon": 10,
    "maxRetryCount": 3,
    "idleTimeout": 60000,
    "initSQLs": ["SET NAMES utf8mb4"]
  },
  "db1": {
    "name": "ds1",
    "url": "jdbc:mysql://127.0.0.1:3306/test_db?useUnicode=true&serverTimezone=UTC",
    "user": "mycat",
    "password": "mycat123",
    "type": "JDBC",
    "minCon": 2,
    "maxCon": 10
  }
}
```

## 4.2 conf/schemas.json — 逻辑 Schema

```json
{
  "schemas": {
    "mycat_test": {
      "tables": {
        "users": {
          "type": "sharding",
          "dataNodes": "db1.users",
          "sharding": {
            "function": {
              "clazz": "io.mycat.router.function.PartitionByMod",
              "properties": {
                "columnName": "id",
                "count": 2
              }
            }
          }
        }
      }
    }
  }
}
```

## 4.3 conf/users.json — 用户配置

```json
{
  "users": {
    "root": {
      "password": "123456",
      "schemas": ["mycat_test"]
    }
  }
}
```

# 5. 启动与验证

```bash
# 1. 启动 Mycat2
cd mycat/bin
./mycat start

# 2. 查看日志确认启动成功
tail -f ../logs/wrapper.log
# 出现 "Mycat2 is ready" 表示启动成功

# 3. 连接 Mycat2（如连接普通 MySQL）
mysql -h 127.0.0.1 -P 8066 -u root -p123456

# 4. 测试查询
USE mycat_test;
SHOW TABLES;
INSERT INTO users(id, name, age) VALUES (1, 'Alice', 25);
INSERT INTO users(id, name, age) VALUES (2, 'Bob', 30);
SELECT * FROM users;
SELECT * FROM users WHERE id = 1;
```

# 6. 管理端口

Mycat2 提供两个端口：

| 端口 | 用途 | 连接方式 |
|------|------|---------|
| **8066** | 数据端口（业务连接） | `mysql -h x.x.x.x -P 8066` |
| **9066** | 管理端口（运维管理） | `mysql -h x.x.x.x -P 9066` |

```sql
-- 管理端口常用命令
/*!mycat:showSchemas*/       -- 查看所有逻辑 Schema
/*!mycat:showDataNodes*/     -- 查看所有数据节点
/*!mycat:showReactors*/      -- 查看 Reactor 线程状态
/*!mycat:showHeartbeatStatus*/ -- 查看心跳状态
```

# 7. 配置热加载

```bash
# 方式一：通过管理端口
mysql -h 127.0.0.1 -P 9066 -u root -p123456 -e "/*!mycat:reloadConfig*/"

# 方式二：通过 HTTP API（如果开启）
curl -X PUT http://localhost:8080/mycat/reloadConfig
```

# 8. 常用配置项速查

```json
// 全局配置 — conf/mycat.yml
{
  "server": {
    "port": 8066,
    "managerPort": 9066,
    "maxCon": 10000,
    "reactorNumber": 4,
    "bufferPoolPageSize": 2097152,
    "bufferPoolChunkSize": 4096,
    "processorBufferPoolType": "direct"
  }
}
```

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| `port` | 数据端口 | 8066 |
| `managerPort` | 管理端口 | 9066 |
| `maxCon` | 最大前端连接数 | 10000 |
| `reactorNumber` | Reactor 线程数 | CPU 核数 × 2 |
| `bufferPoolPageSize` | Buffer Pool 页大小 | 2MB |
