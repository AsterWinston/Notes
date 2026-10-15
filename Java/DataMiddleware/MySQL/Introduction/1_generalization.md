# 1. MySQL概述

## 1.1 什么是MySQL

MySQL 是由瑞典 MySQL AB 公司开发的一款**开源的关系型数据库管理系统（RDBMS）**，目前属于 Oracle 公司。MySQL 将数据保存在不同的表中，而不是将所有数据放在一个大仓库内，这样就增加了速度并提高了灵活性。

MySQL 的核心特征：

- **开源免费**：社区版遵循 GPL 协议，可免费使用（企业版需付费）
- **跨平台**：支持 Linux、Windows、macOS 等多种操作系统
- **高性能**：优化的 SQL 查询算法，支持索引、缓存，单表可支撑千万级数据
- **高可靠性**：提供事务（ACID）、灾难恢复（备份/还原）等机制
- **高可扩展性**：支持主从复制、读写分离、分布式集群
- **多存储引擎**：插件式存储引擎架构，可按需选择 InnoDB、MyISAM、Memory 等

## 1.2 MySQL发展历史

| 版本 | 发布时间 | 核心改进 |
|------|---------|---------|
| 3.x | 2000 | 初代产品，基础 CRUD，MyISAM 引擎为主 |
| 4.x | 2003 | 支持联合查询、子查询、初步的 InnoDB |
| 5.0 | 2005 | 视图、存储过程、触发器、游标、事务全面支持 |
| 5.1 | 2008 | 分区表、事件调度器、行级复制 |
| 5.5 | 2010 | InnoDB 成为默认引擎，半同步复制 |
| 5.6 | 2013 | 全文索引（InnoDB）、GTID 复制、Memcached API |
| 5.7 | 2015 | JSON 数据类型、多源复制、在线 DDL 改进 |
| **8.0** | 2018 | **窗口函数、CTE、原子 DDL、降序索引、不可见索引、角色管理、双密码、克隆插件、直方图** |

> MySQL 8.0 是目前（截至 2026 年）的主流生产版本。MySQL 5.7 于 2023 年 10 月停止官方支持，生产环境应尽快迁移至 8.0 或更高版本。MySQL 版本号 6.x 和 7.x 被跳过，直接从 5.7 跳至 8.0，以表明这是一个包含重大架构变更的里程碑版本。

## 1.3 MySQL体系结构

MySQL 采用**分层架构**设计，从上到下分为四层：

### 1.3.1 连接层（Connection Layer）

负责客户端连接处理、身份验证、权限校验。主要组件包括：

- **连接池**：管理客户端连接的创建与销毁，复用 TCP 连接减少开销
- **线程处理**：每连接一线程（thread-per-connection）模型，8.0 支持线程池插件
- **认证模块**：支持 mysql_native_password、caching_sha2_password（8.0 默认）等认证插件

### 1.3.2 服务层（Service Layer）

MySQL 的核心功能层，完成大部分逻辑处理：

- **SQL 接口**：接收 SQL 语句，返回执行结果
- **解析器（Parser）**：语法分析、语义分析，生成解析树
- **优化器（Optimizer）**：生成最佳执行计划，选择索引、决定 JOIN 顺序等
- **缓存（Query Cache）**：MySQL 8.0 已移除查询缓存（QC 在高并发下性能退化严重，弊大于利）
- **执行器（Executor）**：调用存储引擎 API 执行查询，返回结果

### 1.3.3 存储引擎层（Storage Engine Layer）

插件式架构，负责数据的存储和提取。MySQL 通过定义统一的存储引擎 API 接口，使得不同引擎可以灵活切换。

| 引擎 | 特点 | 适用场景 |
|------|------|---------|
| InnoDB | 事务、行锁、MVCC、外键、崩溃恢复 | 通用 OLTP，默认引擎 |
| MyISAM | 表锁、全文索引、不支持事务 | 只读/日志型场景（已淘汰） |
| Memory | 内存存储、hash 索引、重启丢数据 | 临时表、缓存 |
| Archive | 高压缩比、仅支持 INSERT/SELECT | 归档日志 |
| CSV | CSV 文件存储 | 数据交换 |
| NDB | 分布式集群引擎 | 高可用电信级应用 |

### 1.3.4 文件系统层

负责数据在物理磁盘上的存储，包括：

- 数据文件（.ibd）
- 日志文件（redo log、undo log、binlog、error log、slow log、general log）
- 配置文件（my.cnf / my.ini）

## 1.4 SQL执行流程

一条 SELECT 语句在 MySQL 中的完整执行路径：

```
客户端 → 连接器（认证） → 解析器（语法树） → 优化器（执行计划）
→ 执行器（调用引擎 API） → 存储引擎（读写数据） → 返回结果
```

关键步骤说明：

- **解析器**：对 SQL 做词法分析和语法分析，生成"解析树"；同时检查表、字段、函数是否存在
- **预处理器**：进一步检查语义（如列是否存在、权限是否满足），生成新的解析树
- **优化器**：基于成本模型（CBO，Cost-Based Optimizer），估算不同执行计划的代价，选择最优方案。核心决策包括：索引选择、多表 JOIN 顺序、子查询重写等
- **执行器**：根据执行计划调用存储引擎接口，逐行获取数据，完成过滤、排序、分组等操作

## 1.5 关系型数据库核心概念

| 概念 | 说明 |
|------|------|
| 数据库（Database） | 数据的集合，包含若干表、视图、存储过程等对象 |
| 表（Table） | 按行列组织的二维数据结构，一行 = 一条记录，一列 = 一个字段 |
| 模式（Schema） | 在 MySQL 中，Schema 等同于 Database（与其他 DB 不同，如 Oracle 中 Schema 是用户下的对象集合） |
| 主键（Primary Key） | 唯一标识每一行的字段，非空 + 唯一 |
| 外键（Foreign Key） | 引用另一张表主键的字段，用于维护表间关系 |
| 索引（Index） | 加速数据查找的数据结构（B+Tree 为主），以额外写入开销换取查询性能 |
| 视图（View） | 虚拟表，基于 SELECT 语句的预定义查询，用于简化复杂查询或权限隔离 |
| 事务（Transaction） | 一组不可分割的 SQL 操作，要么全部成功，要么全部失败（ACID） |

## 1.6 MySQL安装

### 1.6.1 安装方式概览

| 方式 | 优点 | 缺点 | 适合场景 |
|------|------|------|---------|
| **二进制包（tar.gz/zip）** | 纯净、可控、多版本共存 | 需手动配置 | 生产环境 |
| **YUM/APT 包管理器** | 安装简单、自动处理依赖 | 版本可能滞后 | 测试环境、离线环境 |
| **Docker 容器** | 秒级启动、隔离性好 | 运维复杂度增加 | 开发环境、CI/CD |
| **源码编译** | 可定制编译参数 | 耗时长、容易出错 | 深度定制需求 |
| **Windows MSI** | 图形化安装向导 | 仅限 Windows | Windows 本地开发 |

### 1.6.2 Docker快速安装（开发推荐）

```bash
# 拉取官方镜像
docker pull mysql:8.0

# 运行容器
docker run -d \
  --name mysql-dev \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=root123 \
  -v mysql-data:/var/lib/mysql \
  mysql:8.0
```

### 1.6.3 Linux二进制包安装（生产推荐）

核心步骤：

1. 下载官方 Linux Generic 包
2. 创建 mysql 用户和组
3. 解压到 /usr/local/mysql
4. 初始化数据目录：`mysqld --initialize --user=mysql`
5. 配置 /etc/my.cnf
6. 启动 MySQL 服务：`systemctl start mysqld`
7. 获取初始 root 密码：`grep 'temporary password' /var/log/mysql/mysqld.log`
8. 修改 root 密码：`ALTER USER 'root'@'localhost' IDENTIFIED BY 'new_password';`

## 1.7 MySQL连接管理

### 1.7.1 连接方式

```bash
# TCP/IP 连接（默认 3306 端口）
mysql -h 192.168.1.100 -P 3306 -u root -p

# Unix Socket 连接（仅 Linux，速度比 TCP 快）
mysql -u root -p -S /var/lib/mysql/mysql.sock

# 执行单条 SQL 后退出（脚本场景）
mysql -u root -p -e "SHOW DATABASES;"
```

> **注意**：在生产环境中避免在命令行直接写密码 `mysql -u root -p密码`，因为系统进程列表可能泄露密码。推荐使用 `mysql_config_editor` 存储认证信息。

### 1.7.2 连接数管理

```sql
-- 查看最大连接数（默认 151，生产环境通常设置 500~1000）
SHOW VARIABLES LIKE 'max_connections';

-- 查看当前连接数
SHOW STATUS LIKE 'Threads_connected';

-- 查看历史最大并发连接数
SHOW STATUS LIKE 'Max_used_connections';
```

> **连接数上限受系统限制**：每个连接消耗内存（约 256KB），同时受 `open_files_limit`（打开文件数）约束。`max_connections` 不宜设置过大，否则可能导致 OOM。

### 1.7.3 连接超时参数

```sql
-- 非交互式连接超时（JDBC 等，默认 28800 秒 = 8 小时）
SHOW VARIABLES LIKE 'wait_timeout';

-- 交互式连接超时（mysql 客户端，默认 28800 秒）
SHOW VARIABLES LIKE 'interactive_timeout';
```

> JDBC 连接池（如 HikariCP）通常设置 `connectionTimeout=30000`（30s），远小于 MySQL 的 `wait_timeout=28800`。连接池会在空闲时发送心跳保活（`SELECT 1`），防止被 MySQL 断开。

## 1.8 字符集与排序规则

### 1.8.1 为什么字符集重要

字符集（Character Set）决定了数据在数据库中的编码方式。选错字符集会导致：
- **乱码**：无法正确显示中文、emoji 等字符
- **存储浪费**：UTF-8 对 ASCII 编码的英文用 1 字节，对中文用 3 字节；UTF-8MB4 对 emoji 用 4 字节
- **索引长度问题**：VARCHAR(255) 在 utf8mb4 下索引可能超出 767 字节限制

### 1.8.2 推荐配置

```sql
-- 查看当前字符集
SHOW VARIABLES LIKE 'character_set%';
SHOW VARIABLES LIKE 'collation%';

-- 推荐配置（8.0 默认已是 utf8mb4）
-- character_set_server = utf8mb4
-- collation_server = utf8mb4_0900_ai_ci

-- 建库时指定
CREATE DATABASE mydb CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
```

| 字符集 | 说明 |
|--------|------|
| utf8 | MySQL 的 utf8 是阉割版，最多 3 字节，**不支持 emoji** |
| **utf8mb4** | 真正的 UTF-8，最多 4 字节，**推荐使用**（从 5.5.3 开始支持） |
| latin1 | 西欧字符集（1 字节），不支持中文 |
| gbk / gb2312 | 中文字符集（2 字节），占用空间小，但不支持多语言 |

> **排序规则（Collation）命名解析**：如 `utf8mb4_0900_ai_ci`，`0900` = Unicode 9.0 标准，`ai` = accent insensitive（重音不敏感），`ci` = case insensitive（大小写不敏感）。`_cs` 表示大小写敏感，`_bin` 表示按二进制比较。

## 1.9 SQL语言分类

| 分类 | 全称 | 核心命令 | 说明 |
|------|------|---------|------|
| **DDL** | Data Definition Language | CREATE / ALTER / DROP / TRUNCATE / RENAME | 定义和修改数据库对象结构（表、索引、视图等） |
| **DML** | Data Manipulation Language | INSERT / UPDATE / DELETE | 对表中的数据进行增删改操作 |
| **DQL** | Data Query Language | SELECT | 查询表中的数据 |
| **DCL** | Data Control Language | GRANT / REVOKE | 控制数据库访问权限 |
| **TCL** | Transaction Control Language | COMMIT / ROLLBACK / SAVEPOINT | 控制事务的提交与回滚 |

> **补充**：MySQL 8.0 支持**原子 DDL**，即 DDL 语句（如 CREATE TABLE）在执行过程中如果系统崩溃，会自动回滚，不会留下不一致的中间状态（基于 InnoDB 的数据字典实现）。

## 1.10 数据模型

### 1.10.1 关系型数据库的数据模型

数据以**二维表（关系）**的形式组织，表与表之间通过**外键**建立关联。核心约束包括：

- **实体完整性**：主键非空且唯一
- **参照完整性**：外键值要么为 NULL，要么等于主表中存在的主键值
- **用户自定义完整性**：通过 CHECK、NOT NULL、UNIQUE 等约束业务规则

### 1.10.2 E-R模型（实体-关系模型）

用于数据库设计的可视化方法：

- **实体（Entity）**：用矩形表示，对应一张表（如"用户"、"订单"）
- **属性（Attribute）**：用椭圆表示，对应字段（如"用户名"、"金额"）
- **关系（Relationship）**：用菱形表示，表间关联（如"用户下单"）

| 关系类型 | 示例 |
|---------|------|
| 1:1（一对一） | 用户 ←→ 身份证 |
| 1:N（一对多） | 用户 ←→ 订单（一个用户多个订单） |
| M:N（多对多） | 学生 ←→ 课程（需要中间表"选课"） |

### 1.10.3 范式（Normalization）

范式是数据库设计的规范，用于减少数据冗余、避免更新异常：

| 范式 | 核心要求 | 反例 |
|------|---------|------|
| **1NF** | 列不可再分（原子性） | 一个字段存"北京,上海" |
| **2NF** | 在 1NF 基础上，所有非主键字段完全依赖于主键（消除部分依赖） | 订单表的主键是（订单ID, 商品ID），但商品名称只依赖商品ID |
| **3NF** | 在 2NF 基础上，消除传递依赖 | 学生表有"系主任"字段，但系主任依赖于"系名"而非学生ID |
| BCNF | 消除主键字段对非主键字段的部分依赖和传递依赖 | 很少用到 |

> **反范式化（Denormalization）**：在实际工程中，为了查询性能，有时会**故意违反范式**。例如在订单表中冗余存储"用户名"字段，避免每次都 JOIN 用户表。这是一种以空间换时间、以写入成本（维护冗余字段）换读取性能的通用优化策略。

## 1.11 SQL编写规范（推荐）

- **关键字大写**：`SELECT name FROM user WHERE id = 1;`
- **表名、字段名小写+下划线**：`user_order`、`create_time`
- **一条 SQL 占一行时，每个子句另起一行**，提高可读性
- **使用 COMMENT 注解**：表和字段加注释，方便后期维护
- **建表时必须指定存储引擎和字符集**
- **生产环境禁止 `SELECT *`**：明确指定需要的字段，避免不必要的数据传输和回表
- **生产 DDL/DML 操作前务必先用 EXPLAIN 检查**
