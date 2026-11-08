# 1. 环境要求

| 组件 | 版本要求 | 说明 |
|------|---------|------|
| JDK | 1.6.x：8；2.x：11+（推荐 17） | 版本混用是最常见的启动失败原因 |
| MySQL | 5.7+ / 8.0+ | 后端数据库 |
| 内存 | 最小 2GB，推荐 4GB+ | 大结果集归并依赖堆外缓冲 |
| 操作系统 | Linux / macOS / Windows | 推荐 Linux |

**Mycat 1.6 对 JDK 的要求经常被踩错**：1.6.x 基于旧版 wrapper 脚本，官方二进制在 JDK 8 上验证，直接换 JDK 17 会因反射限制与 `java.endorsed.dirs` 参数报错起不来；反过来 2.x 要求 JDK 11+。**注意**：同一台机器上业务应用与 Mycat 共存时，务必用 `JAVA_HOME` 单独指定 Mycat 的 JDK，`conf/wrapper.conf` 里的 `java.command` 路径也要同步改，否则 `./mycat start` 报 "Unsupported major.minor version"。

后端 MySQL 侧的前置要求：为 Mycat 建独立账号并授权、`max_connections` 预留足够余量（Mycat 每个 Processor 对每个 dataHost 各持有一个连接池）、MySQL 8.0 实例需要确认认证插件（见第 9 节排坑）。

# 2. 下载与安装

## 2.1 直接下载（Mycat2）

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

## 2.3 Mycat 1.6.x 安装（存量主流版本）

**生产上更多见的是 1.6.7.x 的二进制包**，解压即用，没有 2.x 那种"安装包与核心 JAR 分离"的步骤：

```bash
# 解压 1.6.7.6（社区事实上的最终版本）
tar -zxf Mycat-server-1.6.7.6-release-20230xxx-linux.tar.gz -C /usr/local/
ln -s /usr/local/mycat-1.6.7.6 /usr/local/mycat

# conf 目录是三个 XML 而不是 JSON：
# ├── bin/mycat        # start/stop/restart/status/console
# ├── conf/server.xml  # 系统配置：端口/线程/用户
# ├── conf/schema.xml  # 逻辑库拓扑：schema/dataNode/dataHost
# ├── conf/rule.xml    # 分片规则：tableRule/function
# ├── conf/sequence_conf.properties   # 全局序列参数
# └── logs/

# 指定 JDK 后启动
export JAVA_HOME=/usr/local/jdk1.8.0_361
/usr/local/mycat/bin/mycat start
/usr/local/mycat/bin/mycat status   # MyCAT Server startup successfully.
```

> **行业实践**：1.6 的官方下载站点近年时有不可用，公司内网一般把二进制包放入私有制品库，Docker 镜像也建议自建；社区 fork（如 mycat2 衍生发行版）鱼龙混杂，选包前先看 last commit 时间与 issue 响应。

# 3. 后端 MySQL 准备

```sql
-- 在每个后端 MySQL 上创建用户
CREATE USER 'mycat'@'%' IDENTIFIED BY 'mycat123';
GRANT ALL PRIVILEGES ON *.* TO 'mycat'@'%';
FLUSH PRIVILEGES;

-- 创建测试库和表（两个分片实例上都要执行）
CREATE DATABASE test_db;
USE test_db;
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    name VARCHAR(50),
    age INT
);
```

**授权粒度是有讲究的**：开发环境图省事给 `ALL PRIVILEGES ON *.*`，生产上应最小化——能读写业务库即可；若配置了主从延迟检测（第 4 篇的 `slaveThreshold`），Mycat 账号还必须带 `REPLICATION CLIENT` 权限，否则执行 `SHOW SLAVE STATUS` 被拒、延迟永远按 0 处理，故障 Slave 摘不掉。MySQL 8.0 还要处理认证插件：

```sql
-- MySQL 8.0 默认 caching_sha2_password，Mycat 1.6 低版本不支持
-- 正确做法：为 Mycat 账号改用旧插件
ALTER USER 'mycat'@'%' IDENTIFIED WITH mysql_native_password BY 'mycat123';
```

**物理库表要预先创建**：Mycat 不会帮你建库建表（1.6 连 `CREATE TABLE` 都只透传给单节点），所有 `test_db` 与 `users/orders` 表结构必须在每个分片上先建好，这是与"真数据库"最大的使用差异。

# 4. 配置文件

Mycat2 采用 JSON 配置体系（`conf/` 下四个 json），Mycat 1.6 采用三张 XML，两套体系不可混用。

## 4.1 Mycat2 的 JSON 配置

```json
// conf/datasources.json —— prototype 是模板，其余数据源继承其默认值
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

```json
// conf/schemas.json —— 逻辑 Schema + 分片规则
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

```json
// conf/users.json —— 用户配置
{
  "users": {
    "root": {
      "password": "123456",
      "schemas": ["mycat_test"]
    }
  }
}
```

## 4.2 Mycat 1.6 最小分片配置（两库取模）

1.6 的"最小可用"由三张 XML 组成。目标：逻辑库 `TESTDB`，表 `orders` 按 `user_id` 取模分到 `dn1/dn2` 两个物理库。**注意分片函数的 count 必须等于 dataNode 个数**，这是新手第一大坑。

conf/schema.xml：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
  <schema name="TESTDB" checkSQLschema="false" sqlMaxTimeout="100">
    <table name="orders" primaryKey="id" autoIncrement="true"
           dataNode="dn1,dn2" rule="orders-mod"/>
  </schema>
  <dataNode name="dn1" dataHost="host1" database="test_db"/>
  <dataNode name="dn2" dataHost="host2" database="test_db"/>
  <dataHost name="host1" maxCon="1000" minCon="10" balance="0"
            writeType="0" dbType="mysql" dbDriver="native" switchType="1"
            slaveThreshold="100">
    <heartbeat>select 1</heartbeat>
    <writeHost host="w1" url="127.0.0.1:3306" user="mycat" password="mycat123"/>
  </dataHost>
  <dataHost name="host2" maxCon="1000" minCon="10" balance="0"
            writeType="0" dbType="mysql" dbDriver="native" switchType="1"
            slaveThreshold="100">
    <heartbeat>select 1</heartbeat>
    <writeHost host="w2" url="127.0.0.1:3307" user="mycat" password="mycat123"/>
  </dataHost>
</mycat:schema>
```

conf/rule.xml：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mycat:rule SYSTEM "rule.dtd">
<mycat:rule xmlns:mycat="http://io.mycat/">
  <tableRule name="orders-mod">
    <rule>
      <columns>user_id</columns>
      <algorithm>mod-2</algorithm>
    </rule>
  </tableRule>
  <function name="mod-2" class="io.mycat.route.function.PartitionByMod">
    <property name="count">2</property>
  </function>
</mycat:rule>
```

conf/server.xml 只需确认默认端口与用户：`serverPort=8066`、`managerPort=9066`，`<user name="mycat_user">` 的 `<property name="schemas">TESTDB</property>` 指定可访问逻辑库。

> **设计考量**：分片算法 count 与 dataNode 数量的一致性 Mycat 不做启动期校验——配错时进程照常起来，直到第一条带分片键的 INSERT 才抛 "dataNode not found"。上线前用一条 `INSERT` + `EXPLAIN` 冒烟测试把这类错误提前暴露，比任何 review 都有效。

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

**验证分片是否真的生效，关键看物理库**：登进后端两个 MySQL 分别 `SELECT * FROM users`，id=1 与 id=2 应该分别落在不同实例上；如果两边都能查到同一行，说明路由没生效。

```sql
-- Mycat 1.6 侧的等价验证
mysql -h127.0.0.1 -P8066 -umycat_user -pMycat@2024
USE TESTDB;
EXPLAIN SELECT * FROM orders WHERE user_id = 3;
-- +----+--------------------------+
-- | NODE | DATA                   |
-- +----+--------------------------+
-- | dn2  | SELECT * FROM orders WHERE user_id = 3 |
-- +----+--------------------------+
-- 3 % 2 = 1 → 路由到 dn2，符合预期
```

`EXPLAIN` 是 Mycat 自带的命令而非 MySQL 的，返回"这条 SQL 会发到哪些节点"，**它是排查路由问题最常用的工具**，优先级高于看日志。

# 6. 管理端口与运维命令

Mycat 提供两个端口，1.6 与 2.x 一致：

| 端口 | 用途 | 连接方式 | 账号来源 |
|------|------|---------|---------|
| **8066** | 数据端口（业务连接） | `mysql -h x.x.x.x -P 8066` | server.xml `<user>` / users.json |
| **9066** | 管理端口（运维管理） | `mysql -h x.x.x.x -P 9066` | server.xml `<manager>` |

```sql
-- 1.6 管理端口常用命令（注意 @@ 前缀）
SHOW @@DATAHOST;        -- 各 dataHost 主从状态、活跃连接数
SHOW @@DATANODE;        -- dataNode 与所属 dataHost 映射
SHOW @@SCHEMA;          -- 逻辑库与表数量
SHOW @@SQL;             -- 当前正在执行的 SQL（排障首选）
SHOW @@PROCESSOR;       -- Processor 线程与 buffer pool 状态
SHOW @@THREAD_STATUS;   -- 线程分布：BUSY_REACTION/CON_READ 等
SHOW @@CACHE;           -- 全表路由缓存命中率
SHOW @@HEARTBEAT;       -- 心跳明细
RELOAD @@CONFIG;        -- 重载 schema.xml + rule.xml（不断连接）
RELOAD @@CONFIG ALL;    -- 连同 server.xml 一起重载（部分参数仍需重启）

-- Mycat2 管理端口命令风格不同，走注释形式
/*!mycat:showSchemas*/
/*!mycat:showDataNodes*/
/*!mycat:showReactors*/
/*!mycat:reloadConfig*/
```

> **行业实践**：`SHOW @@SQL` 是 Mycat 1.6 排障第一现场——线上偶发"卡住"时，登 9066 执行它，能看到哪条 SQL 卡在哪个节点、执行了多少毫秒，比翻 logs 快一个数量级。生产脚本里普遍有每分钟采集 `SHOW @@PROCESSOR`、`SHOW @@DATAHOST` 写入监控的定时任务，Mycat 自身没有暴露 metrics 端点。

# 7. 配置热加载

```bash
# 1.6：通过管理端口热加载（不重启、不断开已有连接）
mysql -h127.0.0.1 -P9066 -umycat -pMycat@2024 -e "RELOAD @@CONFIG"

# 校验是否加载成功
mysql -h127.0.0.1 -P9066 -umycat -pMycat@2024 -e "SHOW @@SCHEMA"

# Mycat2：通过管理端口
mysql -h 127.0.0.1 -P 9066 -u root -p123456 -e "/*!mycat:reloadConfig*/"

# Mycat2：通过 HTTP API（如果开启）
curl -X PUT http://localhost:8080/mycat/reloadConfig
```

**reload 的能力边界要记牢**：schema.xml 加新表、改分片规则、加 dataNode 可以热生效；server.xml 的 `processors`、buffer pool 大小必须重启；分片规则改了之后，**已存在的历史数据不会自动搬迁**，只是新写入按新规则路由——这是"扩容配置改完数据不对"事故的根源（详见第 5 篇扩容流程）。

# 8. 排坑清单

| 现象 | 原因 | 解决 |
|------|------|------|
| 启动即退出，wrapper.log 报 `unsupported major.minor` | JDK 版本与 Mycat 版本不匹配 | 1.6 用 JDK 8，2.x 用 JDK 11+ |
| 连接 8066 报 `Unknown character set index` | 客户端默认 utf8mb4 与旧版 Mycat 握手不兼容 | 升级到 1.6.7.4+，或连接串指定 `characterEncoding=UTF-8` |
| 认证失败但账号密码正确 | MySQL 8.0 默认 `caching_sha2_password` | 后端账号改 `mysql_native_password` |
| 启动正常，一执行 SQL 报 `no dataHost` / 找不到节点 | rule.xml `count` 与 schema.xml dataNode 数不一致 | 对齐两者数量 |
| 中文乱码 | 前端字符集与 dataHost `initSQL`/后端库字符集不一致 | 统一 utf8mb4，schema 加 `initSQL=SET NAMES utf8mb4` |
| 查询结果偶尔为空但物理库有数据 | 读写分离读到了延迟 Slave | 强制走主库或降低 balance（见第 4 篇） |
| 大结果集查询后 Mycat OOM 被 kill | 归并把全部结果拉到中间件侧堆外内存 | 分页、限制返回量、调大 `usePageMemory` 相关参数 |
| 事务中的 SELECT 查不到刚 INSERT 的数据 | 写走 master、读走 slave 的延迟 | 同事务内强制读主（Mycat 默认事务内读走主） |
| reload 后配置没生效 | 只 reload 了部分文件，或多台 Mycat 未逐台 reload | `RELOAD @@CONFIG ALL`；配置纳入统一发布 |

> **注意**：Mycat 1.6 默认把大结果集直接驻留内存，压测时最容易复现的不是 SQL 慢而是 Mycat 进程 OOM。上线前必须在预发环境跑一遍**最大预期结果集**的查询（比如不带 WHERE 的全表 `SELECT`），观察 wrapper 日志里的内存水位。

# 9. 小结

**Mycat 快速上手的主线只有一条**：准备后端 MySQL（建库建表建账号，8.0 注意认证插件）→ 写配置（1.6 三张 XML 或 2.x 四个 JSON，重点是 schema.xml 的拓扑与 rule.xml 的分片函数对齐）→ 启动并用 `EXPLAIN` 验证路由落点 → 用 9066 管理端口的 `SHOW @@SQL`/`SHOW @@DATAHOST`/`RELOAD @@CONFIG` 做日常运维。踩坑集中在三处：JDK 版本、`count` 与 dataNode 数量不一致、reload 不搬迁历史数据。

