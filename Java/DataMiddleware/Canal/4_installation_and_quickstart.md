
# 1. 环境要求

## 1.1 软件依赖

| 组件 | 版本要求 | 说明 |
|------|---------|------|
| JDK | 1.8+ | 推荐 JDK 11 或 17 |
| MySQL | 5.6+ / 8.0+ | 支持 MySQL 5.5 到 8.3 |
| ZooKeeper | 3.4+ | HA 模式必需，单机模式可选 |
| Maven | 3.6+ | 源码编译时需要（可选） |
| 操作系统 | Linux/macOS/Windows | 推荐 Linux 生产部署 |

## 1.2 硬件建议

| 规模 | CPU | 内存 | 磁盘 | 网络 |
|------|-----|------|------|------|
| 开发测试 | 2核 | 2GB | 20GB | 100Mbps |
| 小规模生产（<500 TPS） | 4核 | 4GB | 50GB SSD | 1Gbps |
| 中等规模（500-2000 TPS） | 8核 | 8GB | 100GB SSD | 1Gbps |
| 大规模（>2000 TPS） | 16核 | 16GB | 200GB SSD | 10Gbps |

# 2. MySQL 端配置

## 2.1 修改 my.cnf

```ini
[mysqld]
# ========== Canal 必需配置 ==========

# Server ID（必须配置，且与从库不同）
server-id = 1

# 开启 binlog
log-bin = mysql-bin

# 设置 binlog 格式为 ROW
binlog_format = ROW

# 设置 binlog_row_image 为 FULL（推荐）
binlog_row_image = FULL

# ========== 推荐配置 ==========

# 每 1 次事务提交后刷盘
sync_binlog = 1

# binlog 过期时间（7天）
binlog_expire_logs_seconds = 604800

# 最大 binlog 文件大小
max_binlog_size = 512M

# binlog 缓存大小
binlog_cache_size = 32768

# 启用 GTID（推荐，便于 HA）
gtid_mode = ON
enforce_gtid_consistency = ON
```

## 2.2 配置完成后重启 MySQL

```bash
# Linux 重启 MySQL
sudo systemctl restart mysql

# macOS 重启 MySQL
brew services restart mysql

# Windows 重启 MySQL
net stop mysql
net start mysql
```

## 2.3 验证 binlog 配置

```sql
-- 登录 MySQL
mysql -u root -p

-- 检查 log_bin 是否开启
SHOW VARIABLES LIKE 'log_bin';
-- +---------------+-------+
-- | Variable_name | Value |
-- +---------------+-------+
-- | log_bin       | ON    |
-- +---------------+-------+

-- 检查 binlog 格式
SHOW VARIABLES LIKE 'binlog_format';
-- +---------------+-------+
-- | Variable_name | Value |
-- +---------------+-------+
-- | binlog_format | ROW   |
-- +---------------+-------+

-- 检查 binlog_row_image
SHOW VARIABLES LIKE 'binlog_row_image';
-- +------------------+-------+
-- | Variable_name    | Value |
-- +------------------+-------+
-- | binlog_row_image | FULL  |
-- +------------------+-------+

-- 查看当前 binlog 文件和位置
SHOW MASTER STATUS;
-- +------------------+----------+--------------+------------------+
-- | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
-- +------------------+----------+--------------+------------------+
-- | mysql-bin.000001 |      154 |              |                  |
-- +------------------+----------+--------------+------------------+
```

## 2.4 创建 Canal 用户

```sql
-- 创建 Canal 专用账号
CREATE USER 'canal'@'%' IDENTIFIED BY 'canal_password';

-- 授予必要权限
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';

-- 刷新权限表
FLUSH PRIVILEGES;

-- 验证用户和权限
SELECT user, host FROM mysql.user WHERE user = 'canal';
SHOW GRANTS FOR 'canal'@'%';
-- +---------------------------------------------------------------------+
-- | Grants for canal@%                                                  |
-- +---------------------------------------------------------------------+
-- | GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO ...   |
-- +---------------------------------------------------------------------+
```

## 2.5 开启 MySQL 远程连接（如需要）

```sql
-- 允许 canal 用户从任何主机连接
ALTER USER 'canal'@'%' IDENTIFIED WITH mysql_native_password BY 'canal_password';
FLUSH PRIVILEGES;
```

# 3. 下载 Canal

## 3.1 下载预编译包（推荐）

```bash
# 从 GitHub Releases 下载最新版本
# 最新稳定版：v1.1.7
wget https://github.com/alibaba/canal/releases/download/canal-1.1.7/canal.deployer-1.1.7.tar.gz

# 或者使用国内镜像（更快）
wget https://ghproxy.com/https://github.com/alibaba/canal/releases/download/canal-1.1.7/canal.deployer-1.1.7.tar.gz
```

## 3.2 源码编译（可选）

```bash
# 克隆源码
git clone https://github.com/alibaba/canal.git
cd canal

# 切换到指定版本
git checkout canal-1.1.7

# Maven 编译打包
mvn clean install -DskipTests

# 编译后的包位于
# target/canal.deployer-1.1.7.tar.gz
```

## 3.3 Canal 各模块下载说明

| 模块 | 下载包名 | 说明 |
|------|---------|------|
| Canal Server | `canal.deployer-{version}.tar.gz` | 服务端核心（必需） |
| Canal Admin | `canal.admin-{version}.tar.gz` | Web 管理平台（独立部署） |
| Canal Adapter | `canal.adapter-{version}.tar.gz` | 数据适配器（独立部署） |

# 4. 安装 Canal Server

## 4.1 解压部署包

```bash
# 创建安装目录
sudo mkdir -p /opt/canal
cd /opt/canal

# 解压
sudo tar -zxvf canal.deployer-1.1.7.tar.gz -C /opt/canal

# 目录结构
tree -L 2 /opt/canal
# /opt/canal
# ├── bin
# │   ├── restart.sh
# │   ├── startup.bat    # Windows 启动脚本
# │   ├── startup.sh     # Linux 启动脚本
# │   └── stop.sh
# ├── conf
# │   ├── canal.properties          # Canal Server 全局配置
# │   ├── canal_local.properties    # 本地模式配置
# │   ├── logback.xml               # 日志配置
# │   └── example/                  # 默认实例配置
# │       └── instance.properties
# ├── lib/                          # 依赖 jar 包
# ├── logs/                         # 日志目录
# └── plugin/                       # 插件目录
```

## 4.2 配置 Canal Server

**编辑 `conf/canal.properties`**：

```properties
#################################################
#########      common argument      #############
#################################################

# Canal 注册到 ZooKeeper 的集群名（单机可不改）
canal.register.ip =

# Canal Server 的 TCP 端口
canal.port = 11111

# Canal 实例列表（逗号分隔）
canal.destinations = example

# 是否启用 TSDB（表结构快照数据库）
canal.instance.tsdb.enable = true
canal.instance.tsdb.dir = ${canal.file.data.dir:../conf}/${canal.instance.destination:}
canal.instance.tsdb.url = jdbc:h2:${canal.instance.tsdb.dir}/h2;CACHE_SIZE=1000;MODE=MYSQL;
canal.instance.tsdb.dbUsername = canal
canal.instance.tsdb.dbPassword = canal

# 全局实例管理模式
canal.instance.global.mode = spring
canal.instance.global.lazy = false

# ZooKeeper 配置（单机模式可注释掉）
# canal.zkServers = 127.0.0.1:2181

# 服务模式（tcp / kafka / rocketMQ / rabbitMQ）
canal.serverMode = tcp

# MQ 相关配置（使用 tcp 模式时不需要）
# canal.mq.servers = 127.0.0.1:9092
# canal.mq.topic = example
# kafka.acks = all
# kafka.compression.type = none
```

**编辑 `conf/example/instance.properties`**：

```properties
#################################################
#########    mysql serverId     #################
#################################################

# Canal 模拟的 MySQL Slave ID（不能与任何 MySQL Server 的 server-id 冲突）
canal.instance.mysql.slaveId = 1234

#################################################
#########    mysql instance     #################
#################################################

# MySQL Master 地址
canal.instance.master.address = 127.0.0.1:3306

# MySQL Master 初始 binlog 文件和位置（留空从最新位置开始）
canal.instance.master.journal.name =
canal.instance.master.position =
canal.instance.master.timestamp =

# GTID 模式（启用 GTID 时生效）
canal.instance.gtidon = false

#################################################
#########    username/password   #################
#################################################

# MySQL 连接账号
canal.instance.dbUsername = canal
canal.instance.dbPassword = canal_password

# 连接编码
canal.instance.connectionCharset = UTF-8

# 是否启用 druid 解密密码
canal.instance.enableDruid = false

#################################################
#########    table filter regex  #################
#################################################

# 表过滤规则（正则表达式）
# 示例：.*\\..* 表示订阅所有库的所有表
# 示例：test\\..* 表示订阅 test 库的所有表
# 示例：test\\.user 表示只订阅 test 库的 user 表
canal.instance.filter.regex = .*\\..*

# 表过滤黑名单
canal.instance.filter.black.regex =

# 表字段过滤（不常用）
canal.instance.filter.field = test1.t_product:id|subject

#################################################
#########      ring buffer        #################
#################################################

# RingBuffer 大小配置
canal.instance.rds.accesskey =
canal.instance.rds.secretkey =
canal.instance.rds.instanceId =
```

## 4.3 启动 Canal Server

```bash
# 启动
cd /opt/canal
./bin/startup.sh

# 查看日志
tail -f logs/canal/canal.log

# 查看 example 实例日志
tail -f logs/example/example.log

# 预期看到以下日志：
# the canal server is running now ......
# start successful....
```

**Windows 启动**：

```bat
cd C:\canal
bin\startup.bat
```

## 4.4 验证启动

```bash
# 检查进程
ps -ef | grep canal
# canal    12345     1  0 10:00 ?    00:00:05 java -server -Xms2048m ...

# 检查端口
netstat -tlnp | grep 11111
# tcp6  0  0  :::11111  :::*  LISTEN  12345/java

# 查看 Canal 日志确认启动成功
grep "start successful" logs/canal/canal.log
```

# 5. Docker 部署

## 5.1 Docker Compose 单机部署

```yaml
# docker-compose.yml
version: '3.8'

services:
  canal-server:
    image: canal/canal-server:v1.1.7
    container_name: canal-server
    restart: unless-stopped
    network_mode: host  # 或使用 bridge 模式
    environment:
      # MySQL 连接信息
      - canal.instance.master.address=192.168.1.100:3306
      - canal.instance.dbUsername=canal
      - canal.instance.dbPassword=canal_password
      # 过滤规则
      - canal.instance.filter.regex=.*\\..*
      # Canal Server 配置
      - canal.admin.manager=127.0.0.1:8089  # 关联 Admin（可选）
    volumes:
      - ./logs:/home/admin/canal-server/logs
```

```bash
# 启动
docker-compose up -d

# 查看日志
docker-compose logs -f canal-server
```

## 5.2 Docker 挂载配置文件

```bash
# 从容器中提取默认配置
docker run --rm canal/canal-server:v1.1.7 cat /home/admin/canal-server/conf/canal.properties > canal.properties
docker run --rm canal/canal-server:v1.1.7 cat /home/admin/canal-server/conf/example/instance.properties > instance.properties

# 按需修改配置文件后挂载启动
docker run -d \
  --name canal-server \
  --network host \
  -v $(pwd)/canal.properties:/home/admin/canal-server/conf/canal.properties \
  -v $(pwd)/instance.properties:/home/admin/canal-server/conf/example/instance.properties \
  canal/canal-server:v1.1.7
```

# 6. 安装 Canal Admin（可选）

Canal Admin 提供了 Web 管理界面，可以对多台 Canal Server 进行统一管理。

## 6.1 下载并解压

```bash
# 下载 Canal Admin
wget https://github.com/alibaba/canal/releases/download/canal-1.1.7/canal.admin-1.1.7.tar.gz

# 解压
sudo mkdir -p /opt/canal-admin
sudo tar -zxvf canal.admin-1.1.7.tar.gz -C /opt/canal-admin

# 目录结构
# /opt/canal-admin
# ├── bin
# │   ├── restart.sh
# │   ├── startup.sh
# │   └── stop.sh
# ├── conf
# │   ├── application.yml       # Admin 服务配置
# │   ├── canal_manager.sql     # 初始化 SQL
# │   ├── canal-template.properties
# │   └── public/
# └── lib/
```

## 6.2 初始化数据库

```sql
-- 创建 Canal Admin 数据库
CREATE DATABASE IF NOT EXISTS canal_manager DEFAULT CHARSET utf8mb4;

-- 导入初始化 SQL
-- mysql -u root -p canal_manager < /opt/canal-admin/conf/canal_manager.sql
```

## 6.3 配置并启动

```yaml
# conf/application.yml
server:
  port: 8089
spring:
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: GMT+8
spring.datasource:
  address: 127.0.0.1:3306
  database: canal_manager
  username: root
  password: root_password
  driver-class-name: com.mysql.cj.jdbc.Driver
  url: jdbc:mysql://${spring.datasource.address}/${spring.datasource.database}?useUnicode=true&characterEncoding=UTF-8&useSSL=false
```

```bash
# 启动
cd /opt/canal-admin
./bin/startup.sh

# 访问管理界面
# http://localhost:8089
# 默认账号：admin / 123456
```

# 7. 安装 Canal Adapter（可选）

## 7.1 下载并解压

```bash
# 下载 Canal Adapter
wget https://github.com/alibaba/canal/releases/download/canal-1.1.7/canal.adapter-1.1.7.tar.gz

# 解压
sudo mkdir -p /opt/canal-adapter
sudo tar -zxvf canal.adapter-1.1.7.tar.gz -C /opt/canal-adapter
```

## 7.2 配置并启动

```yaml
# conf/application.yml
server:
  port: 8081
spring:
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: GMT+8

canal.conf:
  mode: tcp  # tcp / kafka / rocketMQ
  canalServerHost: 127.0.0.1:11111
  srcDataSources:
    defaultDS:
      url: jdbc:mysql://127.0.0.1:3306/source_db?useUnicode=true
      username: canal
      password: canal_password
  canalAdapters:
    - instance: example
      groups:
        - groupId: g1
          outerAdapters:
            - name: logger  # 打印到日志
            - name: rdb     # 同步到关系型数据库
              properties:
                jdbc.driverClassName: com.mysql.cj.jdbc.Driver
                jdbc.url: jdbc:mysql://127.0.0.1:3306/target_db?useUnicode=true
                jdbc.username: root
                jdbc.password: root_password
```

```bash
# 启动 Canal Adapter
cd /opt/canal-adapter
./bin/startup.sh
```

# 8. 快速验证

## 8.1 使用 Canal Client 示例验证

创建 Maven 项目，引入依赖：

```xml
<dependency>
    <groupId>com.alibaba.otter</groupId>
    <artifactId>canal.client</artifactId>
    <version>1.1.7</version>
</dependency>
```

编写测试客户端：

```java
import com.alibaba.otter.canal.client.CanalConnector;
import com.alibaba.otter.canal.client.CanalConnectors;
import com.alibaba.otter.canal.protocol.CanalEntry.*;
import com.alibaba.otter.canal.protocol.Message;

import java.net.InetSocketAddress;
import java.util.List;

public class SimpleCanalClientExample {

    public static void main(String[] args) {
        // 创建连接器（单机模式）
        CanalConnector connector = CanalConnectors.newSingleConnector(
                new InetSocketAddress("127.0.0.1", 11111),
                "example",    // destination（实例名）
                "",           // username
                ""            // password
        );

        int batchSize = 1000;
        int emptyCount = 0;

        try {
            connector.connect();
            connector.subscribe(".*\\..*");   // 订阅所有库的所有表
            connector.rollback();              // 回滚到上次 ACK 的位置

            while (true) {
                // 获取指定数量的数据
                Message message = connector.getWithoutAck(batchSize);
                long batchId = message.getId();
                int size = message.getEntries().size();

                if (batchId == -1 || size == 0) {
                    emptyCount++;
                    if (emptyCount > 10) {
                        System.out.println("No data, sleeping...");
                        Thread.sleep(1000);
                    }
                    continue;
                }

                emptyCount = 0;
                printEntries(message.getEntries());
                connector.ack(batchId); // 确认消费成功
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            connector.disconnect();
        }
    }

    private static void printEntries(List<Entry> entries) {
        for (Entry entry : entries) {
            if (entry.getEntryType() == EntryType.TRANSACTIONBEGIN
                    || entry.getEntryType() == EntryType.TRANSACTIONEND) {
                continue;
            }

            if (entry.getEntryType() == EntryType.ROWDATA) {
                try {
                    RowChange rowChange = RowChange.parseFrom(entry.getStoreValue());
                    EventType eventType = rowChange.getEventType();
                    Header header = entry.getHeader();

                    System.out.printf("============> binlog[%s:%s], db[%s], table[%s], eventType[%s]%n",
                            header.getLogfileName(), header.getLogfileOffset(),
                            header.getSchemaName(), header.getTableName(), eventType);

                    for (RowData rowData : rowChange.getRowDatasList()) {
                        if (eventType == EventType.DELETE) {
                            printColumns(rowData.getBeforeColumnsList());
                        } else if (eventType == EventType.INSERT) {
                            printColumns(rowData.getAfterColumnsList());
                        } else if (eventType == EventType.UPDATE) {
                            System.out.println("  Before:");
                            printColumns(rowData.getBeforeColumnsList());
                            System.out.println("  After:");
                            printColumns(rowData.getAfterColumnsList());
                        }
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
    }

    private static void printColumns(List<Column> columns) {
        for (Column column : columns) {
            System.out.printf("    %s(%s) = %s, updated = %s%n",
                    column.getName(), column.getMysqlType(),
                    column.getValue(), column.getUpdated());
        }
    }
}
```

## 8.2 执行数据变更测试

```sql
-- 创建测试数据库和表
CREATE DATABASE IF NOT EXISTS test;
USE test;

CREATE TABLE IF NOT EXISTS users (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50),
    email VARCHAR(100),
    status TINYINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 执行 INSERT
INSERT INTO users (name, email, status) VALUES ('Alice', 'alice@example.com', 1);

-- 执行 UPDATE
UPDATE users SET status = 0 WHERE name = 'Alice';

-- 执行 DELETE
DELETE FROM users WHERE name = 'Alice';
```

## 8.3 预期输出

运行 SimpleCanalClientExample 后，可以看到：

```
============> binlog[mysql-bin.000001:154], db[test], table[users], eventType[INSERT]
    id(bigint) = 1, updated = true
    name(varchar) = Alice, updated = true
    email(varchar) = alice@example.com, updated = true
    status(tinyint) = 1, updated = true
    created_at(timestamp) = 2024-01-01 10:00:00, updated = true

============> binlog[mysql-bin.000001:456], db[test], table[users], eventType[UPDATE]
  Before:
    id(bigint) = 1, updated = false
    name(varchar) = Alice, updated = false
    email(varchar) = alice@example.com, updated = false
    status(tinyint) = 1, updated = true
  After:
    id(bigint) = 1, updated = false
    name(varchar) = Alice, updated = false
    email(varchar) = alice@example.com, updated = false
    status(tinyint) = 0, updated = true

============> binlog[mysql-bin.000001:789], db[test], table[users], eventType[DELETE]
    id(bigint) = 1, updated = false
    name(varchar) = Alice, updated = false
    email(varchar) = alice@example.com, updated = false
    status(tinyint) = 0, updated = false
    created_at(timestamp) = 2024-01-01 10:00:00, updated = false
```

# 9. 常见安装问题

## 9.1 Canal 连接 MySQL 失败

**错误日志**：
```
ERROR c.a.o.c.p.d.rds.RdsBinlogEventParserProxy - connect failed
java.io.IOException: connect /127.0.0.1:3306 failure
```

**解决方案**：
```bash
# 1. 检查 MySQL 是否允许远程连接
mysql -h 127.0.0.1 -u canal -p

# 2. 检查 MySQL 绑定的 IP
SHOW VARIABLES LIKE 'bind_address';

# 3. 如果 bind_address = 127.0.0.1，Canal 在其他机器上时需改为 0.0.0.0
```

## 9.2 Binlog Dump 权限不足

**错误日志**：
```
ERROR c.a.o.c.p.d.rds.RdsBinlogEventParserProxy - dump error
java.io.IOException: Access denied; you need (at least one of) the SUPER, REPLICATION CLIENT privilege(s)
```

**解决方案**：
```sql
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
FLUSH PRIVILEGES;
```

## 9.3 Server ID 冲突

**错误日志**：
```
ERROR c.a.o.c.p.d.rds.RdsBinlogEventParserProxy - dump error
A slave with the same server_uuid/server_id as this slave has connected to the master
```

**解决方案**：
```properties
# instance.properties 中修改 slaveId
canal.instance.mysql.slaveId = 5678  # 确保与所有 MySQL Server 的 server-id 不同
```

## 9.4 Binlog 格式不是 ROW

**错误日志**：
```
ERROR c.a.o.c.p.d.rds.RdsBinlogEventParserProxy - dump error
Cannot replicate because the master has BINLOG_FORMAT = STATEMENT
```

**解决方案**：
```sql
-- 在线修改（立即生效，但重启后失效）
SET GLOBAL binlog_format = 'ROW';
SET GLOBAL binlog_row_image = 'FULL';

-- 永久修改需要写入 my.cnf 并重启
```

## 9.5 端口被占用

```bash
# 检查 11111 端口占用的进程
lsof -i :11111
# 或
netstat -tlnp | grep 11111

# 修改 canal.port 配置项可更换端口
```

# 10. 停止 Canal

```bash
# 正常停止
cd /opt/canal
./bin/stop.sh

# 强制停止
kill -9 $(ps -ef | grep CanalLauncher | grep -v grep | awk '{print $2}')
```
