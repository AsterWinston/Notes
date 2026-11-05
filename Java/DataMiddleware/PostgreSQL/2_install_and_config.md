# 1. 安装与目录结构

PostgreSQL 装起来不难，难的是**装完之后知道每个文件在哪、该改哪一份配置**。Linux 发行版打包的 PG 往往把配置放在 `/etc/postgresql`，数据放在 `/var/lib/postgresql`，二进制又在 `/usr/lib/postgresql`；Docker 和源码安装则是另一种布局。这一篇先把"东西在哪"讲清楚，再谈配置、权限和远程连接。

## 1.1 四种安装方式

| 方式 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| 官方仓库 + 包管理器 | 升级方便、自动建服务 | 目录布局被发行版定制 | 生产服务器 |
| Docker 镜像 | 秒级启动、版本隔离 | 数据卷与网络需额外管理 | 开发、测试、CI |
| 源码编译 | 可裁剪、可打补丁 | 升级维护成本高 | 深度定制 |
| Windows 安装包 | 图形向导、自带 pgAdmin | 仅本地开发 | Windows 桌面 |

```bash
# Debian/Ubuntu：添加 PGDG 仓库并安装 17
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
sudo apt install -y postgresql-17
pg_lsclusters

# Docker：只映射到本机回环，避免暴露到公网
docker run -d --name pg-dev \
  -e POSTGRES_PASSWORD=postgres \
  -p 127.0.0.1:5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  postgres:17
```

RHEL 系安装 `postgresql17-server` 后执行 `postgresql-17-setup initdb` 即可，思路相同。生产环境优先用官方仓库或云托管，不建议用 `latest` 标签跑核心库。

## 1.2 数据目录与文件布局

PG 的所有数据都在一个目录里，称为 **PGDATA（数据目录）**。**一个数据目录就是一个实例**，备份、复制、启动参数都围绕它展开。

| 路径 | 内容 |
|------|------|
| `base/<dboid>/` | 各数据库的数据文件，按数据库 OID 分目录 |
| `global/` | 跨库共享的系统表，如 `pg_database`、`pg_authid` |
| `pg_wal/` | WAL 日志段，默认每段 16MB |
| `pg_xact/` | 事务提交状态（CLOG） |
| `pg_multixact/` | 多事务 ID 状态，用于行级锁 |
| `pg_tblspc/` | 表空间的符号链接 |
| `postgresql.conf` | 主配置文件 |
| `pg_hba.conf` | 客户端认证规则 |
| `pg_ident.conf` | 用户名映射 |
| `PG_VERSION` | 数据目录格式版本，**不可手改** |

```sql
-- 任何时候都能确认实例在用哪个目录、哪个配置
SHOW data_directory;
SHOW config_file;
SHOW hba_file;
```

Debian 系把配置与数据分开：`/etc/postgresql/17/main/postgresql.conf` 是实际读取的配置，`/var/lib/postgresql/17/main` 是 PGDATA，二进制在 `/usr/lib/postgresql/17/bin`。

## 1.3 初始化数据目录

包管理器安装会自动 `initdb`，手工部署时需要自己执行：

```bash
sudo useradd -m postgres
sudo mkdir -p /data/pgdata && sudo chown postgres:postgres /data/pgdata

# 指定超级用户、编码、区域与本地认证方式
sudo -u postgres initdb \
  -D /data/pgdata -U postgres \
  --encoding=UTF8 --locale=C.UTF-8 \
  --auth-host=scram-sha-256 --auth-local=peer
```

`--locale` 影响排序与大小写规则。`C.UTF-8` 行为稳定、不依赖系统 locale，适合容器；需要按中文拼音排序时可用 `zh_CN.UTF-8` 或 ICU collation。**初始化后不要随意修改 locale**，否则索引顺序可能与数据不一致。

# 2. 配置文件

PG 的配置分实例级和会话级：实例级写在 `postgresql.conf`，会话级用 `SET` 临时覆盖。理解优先级，是排查"改了没生效"的前提。

## 2.1 postgresql.conf 的关键参数

```ini
listen_addresses = '*'          # 生产按需开放，不要无条件 *
port = 5432
max_connections = 200           # 配合连接池，不是越大越好

shared_buffers = 4GB            # 通常为物理内存 25%
work_mem = 16MB                 # 每个排序/哈希节点，不是每连接
maintenance_work_mem = 512MB    # VACUUM、建索引等维护操作

wal_level = replica             # 逻辑复制改为 logical
max_wal_size = 4GB
min_wal_size = 1GB
checkpoint_timeout = 15min
full_page_writes = on           # 不建议关闭

log_min_duration_statement = 500ms
log_line_prefix = '%m [%p] %u@%d '
log_checkpoints = on
log_autovacuum_min_duration = 0

password_encryption = scram-sha-256
ssl = on
```

改完必须**重启**的参数：`shared_buffers`、`max_connections`、`wal_level`、`listen_addresses`、`port`、`archive_mode`。其余大部分可以 `reload`。

| 操作 | 命令 | 影响 |
|------|------|------|
| reload | `pg_ctl reload -D $PGDATA` 或 `SELECT pg_reload_conf()` | 重读配置，不断连接，不中断事务 |
| restart | `pg_ctl restart -D $PGDATA` | 断开连接，重新初始化共享内存 |
| 会话覆盖 | `SET work_mem = '64MB';` | 只影响当前会话 |
| 库级默认 | `ALTER DATABASE appdb SET work_mem = '32MB';` | 新会话生效 |

> **注意**：`ALTER SYSTEM SET` 写入 `postgresql.auto.conf`，优先级高于 `postgresql.conf`。改了主配置却不生效时，先检查这个文件。

## 2.2 pg_hba.conf 的匹配规则

`pg_hba.conf`（Host-Based Authentication）决定"谁、从哪、用什么方式"能连进来。每行五个字段：

```text
# TYPE   DATABASE   USER       ADDRESS         METHOD
local    all        postgres                   peer
host     all        all        127.0.0.1/32    scram-sha-256
host     all        all        10.0.0.0/8      scram-sha-256
hostssl  appdb      app_rw     0.0.0.0/0       scram-sha-256
```

| 字段 | 取值示例 | 说明 |
|------|---------|------|
| TYPE | `local` / `host` / `hostssl` | `local` 走 Unix socket |
| DATABASE | `all`、`appdb`、`appdb,reportdb` | 逗号分隔多个 |
| USER | `all`、`app_rw`、`+group` | `+` 表示角色组 |
| ADDRESS | CIDR 或 `all` | 仅 host 类型需要 |
| METHOD | `trust`、`peer`、`scram-sha-256`、`cert` | `trust` 禁止用于生产 |

**匹配顺序从上到下，第一条命中即生效**。"明明加了规则还是被拒"，通常是上面有一条更宽的 `reject` 或旧的 `md5` 规则先命中了。改完 reload 即可，不用重启。

# 3. 用户、角色与权限

PG 的权限模型以**角色（role）**为核心。**用户就是可以登录的角色**，两者在系统表里是同一类对象，区别只在有没有 `LOGIN` 属性。

## 3.1 创建角色

```sql
-- 应用账号：可登录、不可建库、不可建角色
CREATE ROLE app_rw LOGIN PASSWORD 'StrongPassw0rd!';
CREATE DATABASE appdb OWNER app_rw ENCODING 'UTF8';

\du
ALTER ROLE app_rw PASSWORD 'NewPassw0rd!';
```

| 属性 | 含义 |
|------|------|
| `LOGIN` | 允许建立连接 |
| `SUPERUSER` | 绕过所有权限检查 |
| `CREATEDB` / `CREATEROLE` | 允许创建数据库 / 角色 |
| `REPLICATION` | 允许建立复制连接 |
| `BYPASSRLS` | 绕过行级安全策略 |
| `CONNECTION LIMIT` | 该角色的并发连接上限 |

## 3.2 最小权限实践

不要用超级用户跑应用。一个够用的读写账号：

```sql
GRANT CONNECT ON DATABASE appdb TO app_rw;
GRANT USAGE ON SCHEMA public TO app_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_rw;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_rw;

-- 以后新建的表/序列自动继承权限
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_rw;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT USAGE, SELECT ON SEQUENCES TO app_rw;
```

> **PG 15 的重要变化**：`public` schema 不再默认允许所有用户 `CREATE`。升级后普通用户执行 `CREATE TABLE` 可能报 `permission denied for schema public`，需要显式 `GRANT CREATE ON SCHEMA public TO xxx;`，或者让每个应用使用自己的 schema。

权限是分层检查的：**先能连库，再能进 schema，最后才能操作对象**。排查时按这个顺序逐层确认：

```sql
SELECT datname FROM pg_database WHERE has_database_privilege(datname, 'CONNECT');
SELECT nspname FROM pg_namespace WHERE has_schema_privilege(nspname, 'USAGE');
SELECT grantee, privilege_type FROM information_schema.role_table_grants
WHERE table_name = 'orders';
```

# 4. 远程连接

远程连接失败几乎总是三个地方之一：**监听地址、认证规则、网络防火墙**。

## 4.1 三要素

```ini
# postgresql.conf：监听所有网卡或指定内网地址
listen_addresses = '*'
```

```text
# pg_hba.conf：允许应用网段用密码认证
host    appdb    app_rw    10.0.0.0/8    scram-sha-256
```

```bash
# 系统防火墙放行 5432
sudo firewall-cmd --permanent --add-port=5432/tcp
sudo firewall-cmd --reload
```

云数据库还要检查安全组。三者缺一，现象都是连接超时或被拒，但日志位置不同：防火墙层面通常没有 PG 日志，`pg_hba.conf` 拒绝会留下 `FATAL: no pg_hba.conf entry`。

## 4.2 连接命令

```bash
# -W 强制交互输入密码，避免密码进 shell 历史
psql -h 192.168.1.10 -p 5432 -U app_rw -d appdb -W

# URL 形式，便于放进环境变量或 JDBC 配置
psql "postgresql://app_rw:secret@192.168.1.10:5432/appdb?sslmode=require"

# 执行单条 SQL 后退出，适合脚本与健康检查
psql -h 127.0.0.1 -U app_rw -d appdb -c "SELECT now();"
```

```sql
\conninfo
```

> **JDBC 建议**：连接串显式写 `sslmode=require`（云厂商可能要求 `verify-full`），并设置 `connectTimeout`、`socketTimeout`、`ApplicationName`。`ApplicationName` 会出现在 `pg_stat_activity` 里，排查时能一眼看出是哪个服务。

# 5. 常见启动与连接错误

| 现象 | 常见原因 | 处理 |
|------|---------|------|
| `Connection refused` | 服务未启动、端口不对、监听地址未开放、防火墙 | `pg_ctlcluster 17 main status`，`ss -lntp \| grep 5432` |
| `no pg_hba.conf entry for host` | 缺规则或被前面的规则抢先匹配 | 补规则并 reload |
| `password authentication failed` | 密码错、认证方式不匹配、md5 与 scram 混用 | 重置密码，检查 `password_encryption` |
| `role "xxx" does not exist` | 角色不存在或连错实例 | `\du` 确认，检查 `-h/-p` |
| `database "xxx" does not exist` | 库名拼错或未创建 | `\l` 查看 |
| `too many connections` | 连接数打满、连接池配置不当 | 查 `pg_stat_activity`，调整池大小 |
| `the database system is starting up` | 正在崩溃恢复 | 等待并查看日志 |
| `database files are incompatible with server` | 高版本二进制启动低版本数据目录 | 换回对应版本或按官方步骤升级 |
| `could not bind IPv4 address` | 端口被占用或监听地址非法 | `ss -lntp` 找占用进程 |
| `PANIC: could not locate a valid checkpoint record` | WAL 缺失或损坏 | 从备份恢复，不要手改 WAL |

```bash
# 包管理器安装
sudo pg_ctlcluster 17 main start
sudo pg_ctlcluster 17 main status

# 源码/手工部署
pg_ctl -D /data/pgdata -l /data/pgdata/server.log start
pg_ctl -D /data/pgdata status
```

```sql
-- 启动失败时先看最近的日志
SELECT pg_read_file('log/postgresql-2026-11-06.log', 0, 20000);
```

# 6. 练习与自检

**练习一**：用 Docker 启动 PG 17，只允许 `10.0.0.0/8` 网段、只允许用户 `app_rw` 访问 `appdb`，并验证其他网段被拒绝。

**练习二**：创建只读账号 `report_ro`，要求能查询 `public` 下现有表和以后新建的表，但不能写入、不能建表。

**练习三**：故意把 `pg_hba.conf` 的认证方式写错，观察连接报错与日志，然后修复并 reload。

**自检清单**：

- 能说出 `postgresql.conf`、`pg_hba.conf`、`postgresql.auto.conf` 的分工与优先级；
- 能解释 `reload` 与 `restart` 分别影响哪些参数；
- 能按"监听地址 → 认证规则 → 防火墙"独立排查远程连接失败；
- 知道 PG 15 之后 `public` schema 的默认权限变化。

# 7. 小结

| 主题 | 核心结论 |
|------|---------|
| 安装 | 生产用官方仓库或托管，开发用 Docker |
| 目录 | 一个 PGDATA 就是一个实例，`base/`、`global/`、`pg_wal/`、`pg_xact/` 各司其职 |
| 配置 | 主配置 + auto.conf 覆盖，改完区分 reload 与 restart |
| 认证 | `pg_hba.conf` 从上到下首条命中，生产禁用 `trust` |
| 权限 | 角色即用户，按"连库 → 进 schema → 操作对象"逐层授权 |
| 远程 | 监听地址、认证规则、防火墙三者缺一不可 |

实例跑起来之后，下一步就是真正写 SQL。下一篇进入**SQL 基础与数据类型**，重点讲清 schema/search_path、常用类型的取舍，以及 NULL 的三值逻辑——这些是后面索引与事务讨论的地基。