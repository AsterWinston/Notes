# 1. 扩展机制

PostgreSQL 的功能边界不是由核心代码写死的。**扩展（extension）** 允许把一组 SQL 对象、类型、函数、操作符和索引访问方法打包安装到数据库里，用一条 `CREATE EXTENSION` 启用。统计、加密、空间、外部数据源、向量检索，很多能力都是通过这套机制接进来的。

## 1.1 一个扩展由什么组成

| 组成 | 作用 |
|------|------|
| 控制文件 `<name>.control` | 元数据：默认版本、依赖、是否可重定位、是否需要预加载 |
| SQL 脚本 `<name>--<version>.sql` | 创建类型、函数、操作符、视图等对象 |
| 共享库 `<name>.so` | C 实现的函数，放在 `$libdir` 下 |

```ini
# pg_stat_statements.control 的关键字段
comment = 'track planning and execution statistics of all SQL statements'
default_version = '1.11'
module_pathname = '$libdir/pg_stat_statements'
relocatable = true
```

`relocatable = true` 表示可以安装到任意 schema；`trusted = true` 表示数据库属主（而非超级用户）也能安装。

## 1.2 常用命令

```sql
-- 查看可用与已安装的扩展
SELECT name, default_version, installed_version FROM pg_available_extensions ORDER BY name;
SELECT extname, extversion, extnamespace::regnamespace FROM pg_extension;

CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
ALTER EXTENSION pg_stat_statements UPDATE;

-- CASCADE 会连带删除依赖对象，慎用
DROP EXTENSION pg_stat_statements;
```

> **注意**：`CREATE EXTENSION` 是**按数据库**生效的。在 `appdb` 装了 PostGIS，不代表 `reportdb` 也能用。备份恢复时 `pg_dump` 只导出 `CREATE EXTENSION` 语句，**不导出扩展的二进制文件**，目标实例必须预先安装对应软件包。

## 1.3 扩展放在哪里

```bash
pg_config --sharedir      # 控制文件与 SQL 脚本，通常是 /usr/share/postgresql/17
pg_config --pkglibdir     # 共享库，通常是 /usr/lib/postgresql/17/lib

sudo apt install postgresql-contrib                    # Debian/Ubuntu
sudo dnf install postgresql17-contrib                 # RHEL 系
```

# 2. pg_stat_statements

这是生产环境**必装**的扩展。它按归一化 SQL 聚合执行次数、耗时、行数、缓冲区访问，是定位慢查询的第一入口。

## 2.1 配置

```ini
shared_preload_libraries = 'pg_stat_statements'   # 必须预加载，修改后重启
pg_stat_statements.max = 10000
pg_stat_statements.track = top                    # top 只跟踪顶层，all 含嵌套
```

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

## 2.2 关键字段与查询

| 字段 | 含义 |
|------|------|
| `queryid` / `query` | 归一化语句的哈希标识 / SQL 文本（常量被替换为 `$1`） |
| `calls` | 执行次数 |
| `total_exec_time` / `mean_exec_time` | 总耗时 / 平均耗时（PG 13+） |
| `rows` | 返回或影响的总行数 |
| `shared_blks_hit/read` | 共享缓冲区命中 / 读盘块数 |

```sql
SELECT queryid, calls,
       round(total_exec_time::numeric, 1) AS total_ms,
       round(mean_exec_time::numeric, 1) AS mean_ms,
       rows, shared_blks_read, left(query, 120) AS sql
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

> **注意**：`query` 是归一化文本，`IN (1,2,3)` 与 `IN (4,5,6)` 会合并。需要看具体参数时，结合 `auto_explain` 或日志。PG 13 把 `total_time` 改名为 `total_exec_time`，老监控脚本要同步修改。

# 3. PostGIS

PostGIS 把 PostgreSQL 变成空间数据库，支持几何类型、坐标系、空间索引和上千个空间函数。LBS、轨迹、地理围栏、行政区划计算基本都靠它。

## 3.1 geometry 与 geography

| 类型 | 坐标含义 | 计算方式 | 适用 |
|------|---------|---------|------|
| `geometry` | 平面坐标，需指定 SRID | 平面欧氏距离，速度快 | 投影坐标系、地图展示 |
| `geography` | 经纬度（WGS84） | 球面距离，结果以米为单位 | 跨城市、跨国距离计算 |

```sql
CREATE EXTENSION IF NOT EXISTS postgis;

CREATE TABLE shop (
    id   bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name text NOT NULL,
    geom geometry(Point, 4326)
);

INSERT INTO shop (name, geom)
VALUES ('旗舰店', ST_SetSRID(ST_MakePoint(121.4737, 31.2304), 4326));

CREATE INDEX idx_shop_geom ON shop USING gist (geom);
```

## 3.2 常用空间查询

```sql
-- 半径 1 公里内的门店：geography 直接按米计算
SELECT id, name, ST_Distance(geom::geography, ST_SetSRID(ST_MakePoint(121.47, 31.23), 4326)::geography) AS meters
FROM shop
WHERE ST_DWithin(geom::geography, ST_SetSRID(ST_MakePoint(121.47, 31.23), 4326)::geography, 1000)
ORDER BY meters;
```

`ST_DWithin` 能利用 GiST 索引，是"附近的人/店"场景的标准写法；`ST_Distance < 1000` 往往会退化为全表计算，应避免。

> **注意**：不同 SRID 的几何直接比较会报错，需要 `ST_Transform`；`ST_Transform` 作用在索引列上会导致索引失效。空间索引不是万能的，过滤条件要尽量写成可下推的形式。

# 4. pgcrypto

`pgcrypto` 提供摘要、HMAC、密码哈希、随机数和对称/公钥加密函数。

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- 摘要、HMAC、bcrypt 哈希
SELECT encode(digest('hello', 'sha256'), 'hex');
SELECT encode(hmac('order=1001&amount=99', 'secret-key', 'sha256'), 'hex');
SELECT crypt('user-password', gen_salt('bf', 12));

-- 校验时把用户输入与库中哈希再算一次
SELECT crypt('user-password', stored_hash) = stored_hash FROM app_user WHERE username = 'alice';
```

列级加密可以防"拖库直接读明文"：

```sql
SELECT pgp_sym_encrypt('13800138000', current_setting('app.crypto_key'));
SELECT pgp_sym_decrypt(phone_enc, current_setting('app.crypto_key')) FROM app_user;
```

> **最佳实践**：密码哈希优先在应用层用 Argon2/bcrypt 完成。数据库层做哈希的意义是"明文密码不进入 SQL 日志"，但连接未加密时明文仍会暴露。列级加密的难点是**密钥管理**，密钥放在数据库配置里等于把锁和钥匙放在一起，生产应使用 KMS 或密钥托管服务。

# 5. UUID 相关扩展

## 5.1 uuid-ossp 与内置函数

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
SELECT uuid_generate_v4();   -- 随机 UUID，不泄露时间与机器信息
SELECT uuid_generate_v1();   -- 基于时间与 MAC，可能泄露隐私，不推荐
```

PG 13 起核心内置了 `gen_random_uuid()`，生成 v4 UUID，**不再需要 uuid-ossp**：

```sql
CREATE TABLE api_token (
    id   uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    name text NOT NULL
);
```

## 5.2 UUIDv7

PG 18 引入 `uuidv7()`：前 48 位是毫秒时间戳，天然按时间有序。相比 v4 的完全随机，v7 作为主键能显著减少 B-tree 页分裂，写入更友好。更早的版本如果既想要全局唯一又想要有序，可以用雪花 ID（`bigint`）或应用层生成 UUIDv7 后写入。

# 6. FDW 外部数据包装器

**FDW（Foreign Data Wrapper）** 让 PG 把外部数据源当作本地表查询。核心对象包括 `FOREIGN DATA WRAPPER`（驱动）、`SERVER`（数据源地址）、`USER MAPPING`（认证映射）和 `FOREIGN TABLE`（表结构）。

## 6.1 postgres_fdw

```sql
CREATE EXTENSION IF NOT EXISTS postgres_fdw;

CREATE SERVER remote_pg FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (host '10.0.0.12', port '5432', dbname 'appdb');

CREATE USER MAPPING FOR app_rw SERVER remote_pg
OPTIONS (user 'remote_ro', password 'RemotePass!');

-- 批量导入远程 schema 中的两张表
IMPORT FOREIGN SCHEMA public LIMIT TO (orders, customers)
FROM SERVER remote_pg INTO public;
```

```sql
-- 查询与本地表无异，可以 JOIN
SELECT r.order_no, c.name
FROM remote_orders r
JOIN customers c ON c.id = r.customer_id
WHERE r.created_at >= now() - interval '1 day';
```

## 6.2 下推与 file_fdw

`postgres_fdw` 会尽量把 `WHERE`、`JOIN`、聚合、排序下推到远程执行。用 `EXPLAIN (VERBOSE)` 可以看到 `Remote SQL`。如果大量行被拉回本地再过滤，说明下推失败，常见原因是用了远程不存在的函数、混合了本地表表达式，或统计信息不足。可以让 FDW 使用远程统计：

```sql
ALTER SERVER remote_pg OPTIONS (use_remote_estimate 'true');
```

```sql
CREATE EXTENSION IF NOT EXISTS file_fdw;
CREATE SERVER file_server FOREIGN DATA WRAPPER file_fdw;

CREATE FOREIGN TABLE access_log (
    log_time timestamptz, client_ip inet, path text, status int
)
SERVER file_server
OPTIONS (filename '/data/logs/access.csv', format 'csv', header 'true');
```

`file_fdw` 读取的是**数据库服务器本地文件**，只有超级用户能创建，或显式授权。不要把它暴露给普通业务账号。

## 6.3 FDW 的边界

- 不适合高频、低延迟的跨库事务，每次查询都有网络往返。
- 远程表没有本地索引，过滤与连接尽量下推。
- 大结果集传输会占用本地内存与网络，必要时先在远程聚合。

> **行业实践**：FDW 适合"低频数据集成、报表汇总、冷数据归档查询"。核心交易链路跨库访问应通过服务接口或消息同步，不要依赖 FDW 做分布式事务。

# 7. 其他值得了解的扩展

| 扩展 | 用途 | 备注 |
|------|------|------|
| `pg_trgm` | 三元组模糊匹配，支持 `LIKE '%x%'` 索引 | 短文本搜索、拼写容错 |
| `citext` | 大小写不敏感的文本类型 | 邮箱、用户名，排序规则仍有坑 |
| `hstore` | 键值对类型 | JSONB 之前的历史方案 |
| `pg_partman` | 分区表自动创建与维护 | 时间分区表必备 |
| `pg_repack` | 在线重建表、消除膨胀 | 替代 `VACUUM FULL` |
| `pg_cron` | 在数据库内调度定时任务 | 清理、汇总、归档 |
| `pgvector` | 向量类型与相似度索引 | RAG、语义检索 |

# 8. 安装与升级风险

- **版本兼容**：扩展的 SQL 脚本与共享库必须匹配 PostgreSQL 主版本。用 PG 17 加载为 PG 16 编译的 `.so`，轻则 `undefined symbol`，重则进程崩溃。
- **主版本升级流程**：先确认扩展有对应版本；`pg_upgrade` 前检查 `pg_available_extensions`；升级后对每个扩展执行 `ALTER EXTENSION ... UPDATE`；重启并检查日志。
- **预加载参数**：`pg_stat_statements`、`auto_explain`、`pg_cron` 需要在 `shared_preload_libraries` 中声明，**必须重启实例**。
- **权限**：默认只有超级用户能 `CREATE EXTENSION`；`trusted = true` 的扩展允许数据库属主安装。
- **备份恢复**：`pg_dump` 导出 `CREATE EXTENSION`，但不导出二进制。恢复前必须先安装扩展软件包，否则后续依赖对象全部无法创建。

```sql
-- 查看当前预加载了哪些库
SHOW shared_preload_libraries;

-- 查看有哪些对象依赖某个扩展
SELECT classid::regclass, objid, deptype
FROM pg_depend
WHERE refobjid = (SELECT oid FROM pg_extension WHERE extname = 'postgis');
```

# 9. 常见错误与排错

| 现象 | 根因 | 处理 |
|------|------|------|
| `could not open extension control file` | 扩展软件包未安装 | 安装对应 contrib/第三方包 |
| `extension "xxx" is not available` | 包已装但版本不匹配 | 查 `pg_available_extensions`，装对应版本 |
| `undefined symbol` | 共享库与 PG 主版本不匹配 | 重新编译或安装对应版本 |
| `permission denied to create extension` | 非超级用户且扩展非 trusted | 让超级用户安装，或用 trusted 扩展 |
| 修改 `shared_preload_libraries` 不生效 | 只 reload 未重启 | 重启实例 |

# 10. 练习与自检

**练习一**：安装 `pg_stat_statements`，执行若干条 SQL 后，分别按总耗时、平均耗时、读盘块数排出前三名，并解释三者优化方向的区别。

**练习二**：用 `postgres_fdw` 把本地实例中的一张表映射为外部表，执行带 `WHERE` 的查询，用 `EXPLAIN (VERBOSE)` 确认条件是否下推。

**练习三**：用 `pgcrypto` 实现用户密码的 bcrypt 哈希与校验，并说明为什么不应该把密钥和密文放在同一个数据库里。

**自检清单**：

- 能说出扩展的三类文件（control、SQL、so）及各自作用；
- 知道 `CREATE EXTENSION` 是按数据库生效，备份恢复需要预装软件包；
- 能配置 `pg_stat_statements` 并读懂 `total_exec_time`、`shared_blks_read`；
- 能区分 `geometry` 与 `geography`，知道 `ST_DWithin` 为什么能走索引；
- 能判断 FDW 的下推是否成功，以及它的适用边界。

# 11. 小结

| 主题 | 核心结论 |
|------|---------|
| 扩展机制 | control + SQL 脚本 + 共享库，`CREATE EXTENSION` 按库安装 |
| pg_stat_statements | 生产必装，按归一化 SQL 聚合耗时与 I/O |
| PostGIS | `geometry` 算平面、`geography` 算球面，GiST 索引是性能关键 |
| pgcrypto | 摘要、HMAC、密码哈希、列加密，密钥管理才是核心 |
| UUID | 内置 `gen_random_uuid()`，PG 18 起可用有序的 `uuidv7()` |
| FDW | 外部数据源映射为本地表，关注下推与网络开销 |

扩展让 PG 的能力可以按需生长，但也引入了版本、权限、备份层面的额外复杂度。下一篇讨论一个更贴近生产的问题：**复制与高可用**。当单机不再是可接受的故障域时，如何用流复制、逻辑复制和 Patroni 把数据库做成一个能自动切换的集群。