# 1. 安装前的准备

Oracle 的安装比 MySQL、PostgreSQL 重得多，因为它同时要准备**操作系统用户、内核参数、目录权限、监听器、数据库实例**五件事。任何一步没做对，后面都会以奇怪的报错形式出现。初学阶段不必追求生产级部署，先用 Docker 跑通一条完整链路，比反复查安装文档更有效。

## 1.1 安装方式选择

| 方式 | 优点 | 缺点 | 适用 |
|------|------|------|------|
| **Docker 镜像** | 分钟级启动、环境隔离、可反复重来 | 不适合生产、镜像体积大 | 学习、开发、CI |
| **官方安装程序** | 完整、可定制、接近生产 | 依赖多、耗时长 | 测试与生产 |
| **静默安装** | 可脚本化、可重复 | 参数写错排查困难 | 批量部署 |
| **云数据库** | 免运维、自带备份 | 成本高、可控性低 | 中小企业 |

> **行业实践**：学习阶段优先用 Docker，把精力放在 SQL、PL/SQL、执行计划上；生产部署通常由 DBA 用静默安装配合标准化脚本完成，应用开发只需要拿到连接信息。

## 1.2 操作系统准备

以 Linux 为例，安装前要确认：内核参数（`shmmax`、`shmall`、`sem`、`file-max`）满足要求；创建 `oinstall`、`dba` 组和 `oracle` 用户；规划 `ORACLE_BASE`、`ORACLE_HOME` 与数据目录；补齐 `libaio`、`gcc`、`glibc` 等依赖包；学习环境可关 SELinux 与防火墙，生产环境应放行监听端口。

```bash
# 查看共享内存、信号量参数与 oracle 用户
ipcs -lm
sysctl kernel.shmmax kernel.shmall kernel.sem
id oracle
```

## 1.3 Docker 快速启动（开发推荐）

```bash
# 启动 Oracle 19c 容器，映射监听端口与数据卷
docker run -d --name oracle19c \
  -p 1521:1521 -p 5500:5500 \
  -e ORACLE_PDB=ORCLPDB1 -e ORACLE_PWD=Oracle123 \
  -v oracle-data:/opt/oracle/oradata \
  container-registry.oracle.com/database/enterprise:19.3.0.0

# 查看启动日志，等待 "DATABASE IS READY TO USE"
docker logs -f oracle19c
```

进入容器可用 `docker exec -it oracle19c sqlplus / as sysdba` 以 sysdba 身份连接。

> **注意**：免费镜像受 Oracle 许可约束，仅限开发与测试。生产使用需要正式授权与安装介质，不要拿容器镜像直接上线。

# 2. 实例、监听器与网络

Oracle 的连接链路比 MySQL 多一层：客户端先连**监听器（Listener）**，由监听器把连接转交给对应的数据库实例。理解这条链路，是排查连接问题的前提。

```text
客户端 (JDBC/sqlplus)
        │  1. TCP 连接 1521
        ▼
┌─────────────────────┐
│ 监听器 Listener     │  读取 listener.ora，只知道有哪些服务
└──────────┬──────────┘
           │ 2. 根据服务名转交连接
           ▼
┌─────────────────────┐
│ 数据库实例           │  PMON 向监听器注册服务
└─────────────────────┘
```

## 2.1 监听器的作用

监听器是独立进程 `tnslsnr`，默认监听 1521 端口。它不访问数据，只负责接收请求、校验服务名、把连接交给实例的服务器进程。实例启动后 PMON 主动注册服务；实例没起来时，监听器可提供静态注册兜底。

```bash
# 查看监听器状态、已注册服务、启动/停止
lsnrctl status
lsnrctl services
lsnrctl start
```

`lsnrctl status` 要重点看监听端口、`Services Summary` 里的服务名，以及实例状态是 `READY` 还是 `BLOCKED`。服务没注册上，客户端就会报 `ORA-12514`。

## 2.2 listener.ora 与 tnsnames.ora

两个文件默认在 `$ORACLE_HOME/network/admin` 下。`listener.ora` 定义服务端监听什么，`tnsnames.ora` 定义客户端怎么连。

```text
# listener.ora（服务端）
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = 0.0.0.0)(PORT = 1521))
    )
  )

# tnsnames.ora（客户端）
ORCL =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.1.100)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = orcl)
    )
  )
```

配置后用 `tnsping ORCL` 测试网络别名。注意 `tnsping` 只验证到监听器的连通性与服务名是否被监听，**不验证用户名密码**；它能通但 `sqlplus` 连不上，问题多半在认证或服务状态。

# 3. 服务名与 SID

这是 Oracle 初学者最容易混淆的一对概念。

- **SID**：实例的唯一标识，一台机器上某个实例的名字，长度通常不超过 8 个字符。
- **服务名**：数据库对外提供的逻辑名称，可以更长，也可以是多个，用于区分同一实例下的不同服务。

早期单实例里二者经常同名（都叫 `orcl`），所以很多人混用。12c 多租户之后，一个 CDB 下可以有多个 PDB，每个 PDB 有自己的服务名，此时 SID 只有一个，服务名有多个。

| 概念 | 层级 | 典型值 | 用途 |
|------|------|--------|------|
| SID | 实例 | `orcl` | 实例标识，本地环境变量 `ORACLE_SID` |
| 服务名 | 数据库对外 | `orcl`、`orclpdb1` | 客户端连接串中的 `SERVICE_NAME` |
| 全局数据库名 | 数据库 | `orcl.example.com` | 分布式环境唯一名 |
| 数据库名 | 数据库 | `orcl` | 控制文件中记录的库名 |

```sql
-- 查看实例名、数据库名、服务名
SELECT instance_name, host_name, status FROM v$instance;
SELECT name, db_unique_name FROM v$database;
SELECT sys_context('USERENV', 'SERVICE_NAME') AS service_name,
       sys_context('USERENV', 'DB_NAME') AS db_name
FROM dual;
```

> **设计考量**：服务名让"一个实例对外提供多个逻辑库"成为可能，配合多租户，PDB 可以独立连接、独立授权、独立备份。这是 Oracle 从 12c 起推动的架构方向。

# 4. JDBC 连接

Java 应用连接 Oracle 使用 `ojdbc` 驱动：19c 对应 `ojdbc8`（JDK 8+），21c 对应 `ojdbc11`（JDK 11+）。

```xml
<dependency>
    <groupId>com.oracle.database.jdbc</groupId>
    <artifactId>ojdbc8</artifactId>
    <version>19.22.0.0</version>
</dependency>
```

## 4.1 三种连接串写法

```java
// 方式一：SID 写法（老项目常见，不推荐用于新系统）
String url1 = "jdbc:oracle:thin:@192.168.1.100:1521:orcl";

// 方式二：服务名写法（推荐）
String url2 = "jdbc:oracle:thin:@//192.168.1.100:1521/orcl";

// 方式三：TNS 别名写法，需要客户端配置 tnsnames.ora
String url3 = "jdbc:oracle:thin:@ORCL";
```

| 写法 | 语法特征 | 说明 |
|------|---------|------|
| SID | `host:port:sid` | 只能连默认服务，多租户下容易连错 |
| 服务名 | `//host:port/service` | 推荐，可指定 PDB 服务名 |
| TNS | `@别名` | 依赖本地 `tnsnames.ora` |

> **行业实践**：统一用服务名写法，把连接串放进配置中心，不要硬编码。多租户环境里应用应连 PDB 的服务名，而不是 CDB 的 SID。

## 4.2 Java 连接示例

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;

public class OracleDemo {
    public static void main(String[] args) throws Exception {
        String url = "jdbc:oracle:thin:@//192.168.1.100:1521/orclpdb1";
        try (Connection conn = DriverManager.getConnection(url, "app_user", "App123456");
             PreparedStatement ps = conn.prepareStatement(
                     "SELECT id, name FROM demo_user WHERE id = ?")) {
            ps.setLong(1, 1L); // 绑定变量，复用执行计划
            try (ResultSet rs = ps.executeQuery()) {
                while (rs.next()) {
                    System.out.println(rs.getLong("id") + ":" + rs.getString("name"));
                }
            }
        }
    }
}
```

## 4.3 连接池配置要点

```yaml
spring:
  datasource:
    url: jdbc:oracle:thin:@//192.168.1.100:1521/orclpdb1
    username: app_user
    password: App123456
    hikari:
      maximum-pool-size: 20
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
      connection-test-query: SELECT 1 FROM DUAL
```

> **注意**：Oracle 的检测语句是 `SELECT 1 FROM DUAL`，不是 MySQL 的 `SELECT 1`。`max-lifetime` 应小于数据库侧连接空闲超时，避免拿到已被服务端断开的死连接。

# 5. 用户与权限

Oracle 的**用户与 schema 绑定**：创建用户就自动创建一个同名 schema，用户拥有的表都放在这个 schema 下。这和 MySQL"用户与库分离"完全不同。

```sql
-- 创建用户并指定默认表空间与配额
CREATE USER app_user IDENTIFIED BY "App123456"
  DEFAULT TABLESPACE users
  TEMPORARY TABLESPACE temp
  QUOTA 500M ON users;

-- 授予登录与开发权限
GRANT CREATE SESSION TO app_user;
GRANT CREATE TABLE, CREATE SEQUENCE, CREATE VIEW, CREATE PROCEDURE TO app_user;
```

| 角色 | 权限范围 | 适用 |
|------|---------|------|
| `CONNECT` | 登录、创建会话 | 所有普通用户 |
| `RESOURCE` | 建表、建序列、建存储过程 | 开发用户 |
| `DBA` | 几乎所有管理权限 | DBA 专属 |
| `SYSDBA` | 最高权限，可启动关闭实例 | 仅 DBA |

> **注意**：`GRANT DBA TO app_user` 是典型的安全事故，业务账号拿到 DBA 等于拿到整个数据库。生产环境应遵循最小权限，按需授予对象权限（`GRANT SELECT ON schema.table TO user`）。

# 6. 常见连接错误

| 错误码 | 含义 | 常见原因 | 排查 |
|--------|------|---------|------|
| `ORA-12541` | 无监听器 | 监听未启动、端口不对 | `lsnrctl status`，查防火墙 |
| `ORA-12514` | 监听器不认识该服务 | 服务名错、实例未注册 | `lsnrctl services`，确认 PDB 服务名 |
| `ORA-12505` | SID 未注册 | 用了 SID 写法但实例名不对 | 改服务名写法或查 `V$INSTANCE` |
| `ORA-01017` | 用户名/密码无效 | 密码错、大小写、账号锁定 | 重置密码，查 `DBA_USERS` |
| `ORA-28000` | 账号被锁 | 多次密码错误 | `ALTER USER ... ACCOUNT UNLOCK` |
| `ORA-12170` | 连接超时 | 网络不通、防火墙拦截 | `tnsping`、`telnet host 1521` |
| `ORA-00257` | 归档空间满 | 归档目录写满，无法写日志 | 清理归档，勿直接删在线日志 |

## 6.1 ORA-12514 的定位过程

```bash
# 1. 监听器是否活着
lsnrctl status
# 2. 服务是否注册
lsnrctl services
# 3. 从客户端测网络
tnsping //192.168.1.100:1521/orclpdb1
```

如果 `lsnrctl services` 里只有 `orcl`、没有 `orclpdb1`，说明 PDB 没打开或服务没注册。进入数据库确认并打开：

```sql
SELECT name, open_mode FROM v$pdbs;
ALTER PLUGGABLE DATABASE orclpdb1 OPEN;
```
## 6.2 ORA-01017 与密码大小写

Oracle 11g 之后密码默认**大小写敏感**，老脚本里的全大写密码迁移到 19c 后按原样输入会失败。连续输错 10 次（默认）会锁定账号。

```sql
-- 查看账号状态并解锁、重置密码
SELECT username, account_status, lock_date FROM dba_users WHERE username = 'APP_USER';
ALTER USER app_user ACCOUNT UNLOCK;
ALTER USER app_user IDENTIFIED BY "App123456";
```

## 6.3 字符集导致的乱码

连接正常但中文显示为问号，通常是**客户端与服务端字符集不一致**。Oracle 字符集由 `NLS_CHARACTERSET` 决定，建库后不能随意修改。

```sql
SELECT parameter, value FROM nls_database_parameters
WHERE parameter IN ('NLS_CHARACTERSET', 'NLS_NCHAR_CHARACTERSET');
SELECT parameter, value FROM nls_session_parameters
WHERE parameter = 'NLS_CHARACTERSET';
```

> **行业实践**：国内系统建库字符集一般选 `AL32UTF8`，不要沿用老库的 `ZHS16GBK`。JDBC 侧可通过 `-Doracle.jdbc.defaultNChar=true` 或连接串设置 `NLS_LANG` 统一编码。建库后修改字符集风险极高，属于 DBA 高危操作。

# 7. 练习

- **练习一**：用 Docker 启动 Oracle 19c，分别用 SID 和服务名写法连接，对比 `sys_context` 返回的 `SERVICE_NAME` 差异。
- **练习二**：创建普通用户 `app_user`，只授予 `CREATE SESSION` 与 `CREATE TABLE`，尝试访问其他 schema 的表并记录报错；再通过 `GRANT SELECT ON` 授予只读权限，验证只能查不能改。
- **练习三**：故意把 `tnsnames.ora` 里的服务名改错，观察 `tnsping` 与 `sqlplus` 的报错差异，说明为什么 `tnsping` 通过不代表能登录。

# 8. 小结

| 主题 | 核心结论 |
|------|---------|
| 安装 | Docker 适合学习，生产用官方介质与静默安装 |
| 连接链路 | 客户端 → 监听器 1521 → 实例，PMON 负责服务注册 |
| 服务名与 SID | SID 标识实例，服务名对外连接，多租户靠服务名区分 PDB |
| JDBC | 推荐 `//host:port/service` 写法，用 `PreparedStatement` 绑定变量 |
| 用户与权限 | 用户即 schema，最小权限，禁止业务账号拿 DBA |
| 常见错误 | 12514/12505 看服务注册，1017 看密码与锁定，257 看归档空间 |

连接打通之后，就可以真正写 SQL 了。下一篇进入**SQL 与数据类型**，重点讲清 schema 语义、`VARCHAR2` 的字节陷阱、`NUMBER` 与 `DATE` 的精度、序列的用法，以及没有 `LIMIT` 时怎么做分页。
