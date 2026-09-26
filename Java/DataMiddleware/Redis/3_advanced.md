# 1.分布式缓存

## 1.1Redis持久化

### 1.1.1RDB演示

- 触发方式：`save`/`bgsave`，或配置 `save 900 1`。
- 生成文件：`dump.rdb`（默认在 `dir` 配置目录）。
- 特点：快照式、恢复快，但可能丢失最后一次快照后的数据。

常用配置（触发bgsave）：
```conf
save 900 1
save 300 10
save 60 10000
```

#### bgsave 内部流程详解

`bgsave` 命令执行后，Redis 内部完成以下步骤：

1. **主进程检查**：检查是否已有子进程在执行 RDB/AOF rewrite，若有则直接返回错误（同一时刻只允许一个子进程做持久化）。
2. **fork 子进程**：Redis 主进程调用 OS 的 `fork()` 系统调用创建一个子进程。fork 期间主进程会短暂阻塞，阻塞时间取决于内存大小（通常 10GB 内存约 100ms）。
3. **信息记录**：主进程记录子进程 PID，并更新 `info persistence` 中的 `rdb_bgsave_in_progress` 状态。
4. **子进程写文件**：子进程利用 fork 时共享的父进程内存页表，遍历所有键值对写入临时 RDB 文件。
5. **COW 机制**：写入期间，主进程修改某个 key 时，OS 会通过 COW 将对应内存页复制一份，子进程始终看到 fork 时的数据快照。
6. **原子替换**：子进程写完后，通过 `rename()` 系统调用将临时文件原子替换旧的 `dump.rdb`。
7. **通知主进程**：子进程退出，主进程通过 `wait3()` 回收子进程，更新 `rdb_last_bgsave_status` 为 `ok`。

查看 bgsave 状态：
```bash
redis-cli INFO persistence | grep rdb
```

#### RDB 配置参数完整表

| 参数 | 默认值 | 说明 |
|---|---|---|
| `save 900 1` | - | 900 秒内至少 1 次修改则触发 bgsave |
| `save 300 10` | - | 300 秒内至少 10 次修改则触发 bgsave |
| `save 60 10000` | - | 60 秒内至少 10000 次修改则触发 bgsave |
| `save ""` | - | 禁用自动 RDB（设置为空字符串） |
| `dbfilename dump.rdb` | `dump.rdb` | RDB 文件名 |
| `dir ./` | `./` | RDB 文件存储目录 |
| `rdbcompression yes` | `yes` | 是否使用 LZF 压缩 RDB 文件 |
| `rdbchecksum yes` | `yes` | 是否使用 CRC64 校验 RDB 文件（末尾 8 字节） |
| `stop-writes-on-bgsave-error yes` | `yes` | bgsave 失败时是否拒绝写入（生产建议 yes） |
| `rdb-del-sync-files no` | `no` | 主从全量同步后是否删除主节点上的 RDB 文件 |

#### RDB COW 内存使用分析

`fork()` 使用的是 Copy On Write 机制：

- **fork 前**：主进程占用 M GB 内存。
- **fork 后**：子进程与主进程共享同一份物理内存页表。
- **写入时触发 COW**：主进程每修改一个内存页（通常 4KB），OS 复制一份新页给主进程，子进程继续使用旧页。
- **极端情况**：如果 bgsave 期间主进程修改了大量 key（例如写入 50% 的数据），内存占用可能达到 M x 1.5 ~ M x 2。
- **内存额外开销估算公式**：额外内存 ~ 写入量 x 3%（取决于页修改比例），保守预留 fork 时内存的 1.5 倍。
- **避免 COW 爆炸**：
  - 使用 Redis 4.0+ 混合持久化减少 RDB 频率。
  - 关闭 THP（Transparent Huge Pages）：`echo never > /sys/kernel/mm/transparent_hugepage/enabled`。
  - 设置 `vm.overcommit_memory = 1`：`sysctl vm.overcommit_memory=1`。

#### RDB 文件恢复步骤

```bash
# 1. 确认 RDB 文件位置
redis-cli CONFIG GET dir
redis-cli CONFIG GET dbfilename

# 2. 停止 Redis
redis-cli shutdown nosave

# 3. 备份旧 RDB
cp dump.rdb dump.rdb.bak.$(date +%Y%m%d_%H%M%S)

# 4. 将备份的 RDB 文件复制到 dir 目录
cp /backup/dump.rdb /var/lib/redis/dump.rdb

# 5. 检查文件权限，确保 Redis 用户有读写权限

# 6. 启动 Redis
redis-server /etc/redis/redis.conf

# 7. 验证恢复
redis-cli DBSIZE
redis-cli INFO persistence | grep rdb_last_load_time

# 8. 检查日志无 Short read 或 CRC error
tail -f /var/log/redis/redis.log
```

#### RDB 故障排查：bgsave 失败怎么办

```bash
# 1. 查看错误信息
redis-cli INFO persistence | grep rdb_last_bgsave_status
redis-cli LASTSAVE

# 2. 常见原因与解决：
# - 磁盘空间不足：df -h 检查磁盘，清理或扩容
# - 目录权限不足：确保 dir 目录 Redis 进程可写
# - fork 失败（内存不足）：dmesg | grep -i oom，增加系统内存或降低 maxmemory
# - stop-writes-on-bgsave-error yes 导致拒绝写入：先用 CONFIG SET 临时恢复，再排查持久化

# 3. 手动触发排查
redis-cli BGSAVE
```

---

### 1.1.2RDB的fork原理

- `bgsave` 会 fork 子进程写 RDB。
- 基于 Copy On Write（COW），主进程继续处理请求。
- 数据量大时 fork 可能导致短暂阻塞。

#### 深入理解 fork 阻塞

- `fork()` 调用会复制父进程的页表（Page Table），页表大小 = 内存大小 / 4KB x 页表项大小。
- 10GB 内存的 Redis，页表约 2.5GB / 4KB = 约 640K 个页表项，fork 时间约 50~150ms。
- 设置 `linux-overcommit-memory = 1` 确保 fork 不会因内存不足被杀。
- **排查 fork 耗时**：`redis-cli INFO stats | grep latest_fork_usec`（单位微秒）。

---

### 1.1.3AOF演示

- 开启 AOF：`appendonly yes`。
- 同步策略：`appendfsync everysec`（推荐）。
- 发生 AOF 膨胀时可 `bgrewriteaof` 重写。

常用配置：
```conf
appendonly yes
appendfsync everysec
```

#### AOF Rewrite 流程详解

AOF 重写是把内存中的数据等效转换为写入命令，生成一个新的 AOF 文件，替代旧的大文件。

1. **触发重写条件**：手动 `BGREWRITEAOF` 或自动 `auto-aof-rewrite-percentage 100` 且 `auto-aof-rewrite-min-size 64mb`。
2. **fork 子进程**：主进程 fork 一个子进程。
3. **子进程重写**：遍历当前内存数据，按 RESP 协议格式写入临时 AOF 文件（文件名 `temp-rewriteaof-bg-<pid>.aof`）。例如一个 `SET key value` 在内存中只占一行 AOF 指令，但旧的 AOF 文件可能有 100 次对该 key 的修改记录。
4. **主进程累积增量**：重写期间主进程的新写入同时写入旧的 AOF 缓冲区和 AOF 重写缓冲区。
5. **子进程通知完成**：子进程完成后，向父进程发信号。
6. **主进程合并增量**：主进程将 AOF 重写缓冲区的内容追加到临时 AOF 文件末尾。
7. **原子替换**：主进程用 `rename()` 将临时文件替换为正式的 AOF 文件。

整个重写过程不阻塞主进程的读写请求。

#### AOF 配置参数完整表

| 参数 | 默认值 | 说明 |
|---|---|---|
| `appendonly no` | `no` | 是否开启 AOF |
| `appendfilename "appendonly.aof"` | `appendonly.aof` | AOF 文件名 |
| `appenddirname "appendonlydir"` | `appendonlydir` | AOF 文件目录（Redis 7.0+ 多文件 AOF） |
| `appendfsync everysec` | `everysec` | 同步策略：`always`/`everysec`/`no` |
| `no-appendfsync-on-rewrite no` | `no` | AOF rewrite 期间是否暂停 fsync |
| `auto-aof-rewrite-percentage 100` | `100` | AOF 增长率达到多少触发 rewrite |
| `auto-aof-rewrite-min-size 64mb` | `64mb` | AOF 文件最小多大才触发 rewrite |
| `aof-load-truncated yes` | `yes` | 启动时 AOF 末尾截断是否允许加载 |
| `aof-use-rdb-preamble yes` | `yes` | 是否开启 RDB+AOF 混合持久化（Redis 4.0+） |
| `aof-timestamp-enabled no` | `no` | AOF 是否记录时间戳（Redis 7.0+） |

#### AOF fsync 三种模式对比

| 模式 | 性能 | 数据安全 | 工作原理 |
|---|---|---|---|
| `always` | 低（每个命令都 fsync） | 最高，最多丢失一个命令 | 每条命令写入后立即 fsync |
| `everysec` | 中（推荐） | 最多丢失 1 秒数据 | 每秒由后台线程 fsync 一次 |
| `no` | 高 | 最差，完全依赖 OS 调度 | Redis 不主动 fsync，由 OS 决定何时刷盘 |

性能实测参考（单机，SSD 磁盘）：
- `always`：QPS 约 2~3 万。
- `everysec`：QPS 约 10 万+（接近无持久化性能）。
- `no`：QPS 与无持久化相当，但可能丢失 30 秒的数据。

生产环境推荐 `appendfsync everysec`。

#### AOF 文件大小估算

- 估算公式：AOF 大小 ≈ 平均命令长度 x 写入次数 x (1 + rewrite percentage 阈值)
- 每秒 1000 次 SET（平均 50 字节），每小时 AOF 增长约 180MB
- 开启 rewrite 后，最终 AOF 文件约等于 RDB 大小的 1.1~1.3 倍

#### AOF 损坏修复步骤

```bash
# 1. 备份当前 AOF
cp appendonly.aof appendonly.aof.bak

# 2. 使用 redis-check-aof 工具修复
redis-check-aof --fix appendonly.aof

# 3. 检查修复结果
diff appendonly.aof appendonly.aof.bak | tail -20

# 4. 如果 --fix 修复失败，尝试 redis-check-rdb 从混合持久化的 RDB 头恢复
redis-check-rdb appendonly.aof

# 5. 重新启动 Redis，观察日志
```

---

### 1.1.4RDB和AOF的对比

- RDB：恢复快、文件小，可能丢数据。
- AOF：数据更安全，恢复较慢，文件可能更大。
- 生产常用：RDB + AOF 混合。

#### 详细对比表

| 维度 | RDB | AOF |
|---|---|---|
| 持久化方式 | 内存快照 | 追加写命令 |
| 文件体积 | 小（压缩后的二进制） | 大（原始命令文本，rewrite 后缩小） |
| 数据恢复速度 | 快（直接加载二进制） | 慢（逐条回放命令） |
| 数据安全性 | 可能丢失上次快照之后的数据 | 最多丢失 1 秒数据（everysec） |
| 对性能影响 | fork + COW，瞬间微小阻塞 | 持续的磁盘 I/O 开销 |
| 适用场景 | 允许少量数据丢失的对读取性能敏感场景 | 对数据安全性要求高、可接受一定 I/O 开销 |
| 文件可读性 | 不可读（二进制） | 可读（Redis 命令文本） |

---

### 1.1.5RDB+AOF混合持久化 (Redis 4.0+)

混合持久化结合了两者优势：
- **AOF 文件结构**：前半部分是 RDB 格式的当前数据快照，后半部分是 RDB 之后追加的 AOF 命令。
- **优点**：恢复速度接近 RDB，数据安全性接近 AOF（最多丢 1 秒数据），AOF rewrite 后文件体积变小（RDB 部分是压缩的）。

配置：
```conf
aof-use-rdb-preamble yes	
```

| 场景 | 纯 RDB | 纯 AOF | 混合持久化 |
|---|---|---|---|
| 重启恢复速度 | 快 | 慢 | 快 |
| 数据丢失风险 | 最多丢数分钟 | 最多丢 1 秒 | 最多丢 1 秒 |
| 文件体积 | 小 | 大 | 中 |
| 推荐场景 | 备份/灾备 | 高安全要求 | **生产推荐** |

---

### 1.1.6持久化选型决策树

```
需要持久化吗？
+-- 否 -> 不开启任何持久化（纯缓存场景，数据可接受全部丢失）
+-- 是 -> 对数据丢失容忍度？
          +-- 允许丢失数分钟数据 -> 仅 RDB
          |     +-- 适用于：缓存预热数据、非关键统计数据
          +-- 允许丢失 <= 1 秒 -> RDB + AOF 混合持久化（推荐）
          |     +-- 适用于：大多数业务场景
          +-- 不允许任何丢失 -> RDB + AOF (appendfsync always)
                +-- 适用于：金融结算、订单系统核心数据
                      注意：性能大幅下降，需评估能否接受

特殊情况：
- 纯只读场景（如只做缓存，数据全从 DB 加载） -> 关闭持久化
- 极高写入吞吐（>10万 ops/s） -> 仅 RDB 或关闭 AOF，从节点做持久化
- 内存非常大的实例（>32GB） -> 注意 fork 阻塞风险，建议混合持久化 + 降低 rewrite 频率
```

---

## 1.2Redis主从

### 1.2.1主从集群结构

- 一主多从。
- 主节点写入，从节点只读复制。
- 读写分离提高读性能。

### 1.2.2搭建主从集群

- 从节点执行：`replicaof <master-ip> <port>`。
- 主节点可设置密码：`requirepass`，从节点使用 `masterauth`。

#### Docker-Compose 完整搭建（1 主 2 从）

```yaml
version: '3.8'
services:
  redis-master:
    image: redis:7.2
    container_name: redis-master
    command: redis-server --port 6379 --requirepass redis123
    ports:
      - "6379:6379"
    volumes:
      - ./data/master:/data
    networks:
      - redis-net

  redis-slave-1:
    image: redis:7.2
    container_name: redis-slave-1
    command: redis-server --port 6379 --replicaof redis-master 6379 --masterauth redis123 --requirepass redis123
    ports:
      - "6380:6379"
    volumes:
      - ./data/slave1:/data
    depends_on:
      - redis-master
    networks:
      - redis-net

  redis-slave-2:
    image: redis:7.2
    container_name: redis-slave-2
    command: redis-server --port 6379 --replicaof redis-master 6379 --masterauth redis123 --requirepass redis123
    ports:
      - "6381:6379"
    volumes:
      - ./data/slave2:/data
    depends_on:
      - redis-master
    networks:
      - redis-net

networks:
  redis-net:
    driver: bridge
```

#### 验证主从搭建

```bash
# 1. 启动
docker-compose up -d

# 2. 查看主节点信息
redis-cli -a redis123 -p 6379 INFO replication
# 预期：role:master, connected_slaves:2, slave0:state=online

# 3. 查看从节点信息
redis-cli -a redis123 -p 6380 INFO replication
# 预期：role:slave, master_link_status:up

# 4. 主从复制测试：在主节点写入
redis-cli -a redis123 -p 6379 SET test:key "hello-master"

# 5. 在从节点读取
redis-cli -a redis123 -p 6380 GET test:key
# 输出: "hello-master"

# 6. 验证读写分离（从节点默认只读）
redis-cli -a redis123 -p 6380 SET test:key2 "from-slave"
# 报错: (error) READONLY You can't write against a read only replica.
```

#### 读写分离测试（Spring Boot Lettuce）

```java
@Configuration
public class RedisReplicaConfig {

    @Bean
    public LettuceConnectionFactory redisConnectionFactory() {
        LettuceClientConfiguration clientConfig = LettuceClientConfiguration.builder()
                .readFrom(ReadFrom.REPLICA_PREFERRED)
                .build();

        RedisStaticMasterReplicaConfiguration serverConfig =
                new RedisStaticMasterReplicaConfiguration("127.0.0.1", 6379);
        serverConfig.addNode("127.0.0.1", 6380);
        serverConfig.addNode("127.0.0.1", 6381);
        serverConfig.setPassword("redis123");

        return new LettuceConnectionFactory(serverConfig, clientConfig);
    }
}
```

---

### 1.2.3主从的全量同步原理

- 从节点发送 `psync`。
- 主节点 `bgsave` 生成 RDB 并发送给从节点。
- 同步过程中主节点会记录增量命令到复制缓冲区。

#### 全量同步详细流程（replid + offset 机制）

每个 Redis 实例有两个关键标识：
- **replid（Replication ID）**：一个 40 字节的伪随机字符串，标识当前数据集。主节点每次重启或成为新主时会生成新的 replid。
- **replid2**：用于部分重同步的辅助复制 ID。
- **offset**：复制偏移量，主节点每收到一个写命令就增加 N 字节（命令长度），从节点也维护自己的 offset。

**第一阶段：建立连接与协商**
1. 从节点执行 `REPLICAOF <master-ip> <port>`，与主节点建立 TCP 连接。
2. 从节点发送 `PING`，等待主节点返回 `PONG`。
3. 如果设置了 `masterauth`，从节点发送 `AUTH <password>` 进行认证。
4. 从节点发送 `REPLCONF listening-port <port>` 告知自己的监听端口。
5. 从节点发送 `PSYNC ? -1` 发起全量同步请求（`?` 表示询问主节点 replid，`-1` 表示需要全量同步）。

**第二阶段：快照生成与传输**
6. 主节点收到 `PSYNC ? -1` 后，返回 `FULLRESYNC <replid> <offset>`。
7. 主节点 `BGSAVE` 生成 RDB 快照。
8. bgsave 期间，主节点将所有写命令累积到复制缓冲区。
9. RDB 生成完成后，主节点将 RDB 文件通过网络发送给从节点。
10. 从节点清空自己的旧数据，加载接收到的 RDB。

**第三阶段：增量同步**
11. 从节点加载完 RDB 后，发送 `REPLCONF ACK <offset>` 确认。
12. 主节点将复制缓冲区中累积的增量命令发送给从节点。
13. 从节点逐条执行，直到 offset 追上主节点。

#### 复制缓冲区与复制积压缓冲区

- **replication buffer（复制缓冲区）**：每个从节点独立分配，用于存储 RDB 生成期间的写命令，大小由 `client-output-buffer-limit replica 256mb 64mb 60` 决定。
- **repl-backlog（复制积压缓冲区）**：环形缓冲区，主节点只有一个，所有从节点共享，用于支持部分重同步。大小由 `repl-backlog-size` 决定（默认 1MB），生产建议至少 32mb ~ 128mb。

```conf
repl-backlog-size 128mb
repl-backlog-ttl 3600
```

#### repl-backlog size 计算指南

估算公式：
```
backlog_size = 主节点 QPS x 平均命令大小 x 允许断线时间(秒)
```

示例：主节点 QPS 5000，平均命令大小 100 字节，允许断开 60 秒：
```
backlog_size = 5000 x 100 x 60 = 30,000,000 字节 ~ 30MB
```
建议设置 `repl-backlog-size 64mb` 留有余量。

---

### 1.2.4增量同步原理

- 依赖 `replid` 与 `offset`。
- 网络抖动后从节点携带 offset，主节点发送 backlog 中的增量命令。

#### 增量同步（部分重同步）详细流程

1. 从节点与主节点网络断开后重新连接。
2. 从节点发送 `PSYNC <replid> <offset>` 请求增量同步。
3. 主节点校验：replid 是否匹配，offset 是否还在 repl-backlog 缓冲区内。
4. 如果满足条件，主节点返回 `CONTINUE`，并将 backlog 中 offset 之后的命令发送给从节点。
5. 如果 backlog 中已经找不到该 offset（说明断线太长，backlog 被覆盖了），则只能进行全量同步。

#### 常见复制问题与解决方案

**问题 1：复制超时**
- 现象：`master_link_status:down`，`repl_backlog_active:0`
- 解决：
```conf
repl-timeout 60
client-output-buffer-limit replica 512mb 256mb 120
```

**问题 2：repl-backlog 太小导致频繁全量同步**
- 现象：从节点日志频繁出现 `Full resync`
- 解决：增大 `repl-backlog-size`，如设置为 `128mb`

**问题 3：主节点重启后 replid 变化导致全量同步**
- 主节点重启后 replid 会改变，所有从节点需要全量同步。
- 减轻方法：使用哨兵/集群自动故障转移，而不是直接重启主节点。

---

#### 主从复制监控

**INFO replication 输出字段详解**

主节点关键字段：

| 字段 | 含义 | 正常值 |
|---|---|---|
| `role` | 角色 | `master` |
| `connected_slaves` | 已连接从节点数 | 等于预期从节点数 |
| `master_replid` | 当前复制 ID | 随机字符串 |
| `master_repl_offset` | 当前复制偏移量 | 持续增长 |
| `repl_backlog_active` | backlog 是否活跃 | `1` |
| `repl_backlog_size` | backlog 大小（字节） | 配置值 |
| `repl_backlog_histlen` | backlog 当前有效数据长度 | - |
| `slave0:ip,port,state,offset,lag` | 从节点信息 | `state=online`, `lag` < 3 |

从节点关键字段：

| 字段 | 含义 | 正常值 |
|---|---|---|
| `role` | 角色 | `slave` |
| `master_host` | 主节点 IP | - |
| `master_link_status` | 复制链路状态 | `up` |
| `master_last_io_seconds_ago` | 距离上次通信秒数 | < 10 |
| `master_sync_in_progress` | 是否正在全量同步 | `0` |
| `slave_repl_offset` | 从节点复制偏移量 | 接近 master offset |
| `slave_priority` | 哨兵选举优先级 | 配置值 |
| `slave_read_only` | 从节点只读 | `1` |

**需要重点监控的指标**：
- **复制延迟（lag）**：offset 差值越小越好（< 3 秒）
- **复制状态**：`master_link_status` 必须为 `up`
- **全量同步次数**：检查日志中 `Full resync` 频率

监控命令示例：
```bash
redis-cli INFO replication | grep master_repl_offset
redis-cli -p 6380 INFO replication | grep master_repl_offset
# 计算差值即可得到复制延迟

watch -n 1 'redis-cli INFO replication | grep -E "role|connected_slaves|master_repl_offset|lag"'
```

---

## 1.3Redis哨兵

### 1.3.1哨兵的作用和工作原理

- 监控主从节点健康。
- 自动故障转移，选举新主。
- 通知客户端新的主节点地址。

#### 哨兵组件核心功能拆解

1. **监控（Monitoring）**：每隔 1 秒，每个哨兵对所有主从节点发送 PING，检测是否可达。
2. **通知（Notification）**：节点状态变更时，哨兵通过 API 或脚本通知运维人员。
3. **自动故障转移（Automatic Failover）**：主节点不可达时，多个哨兵达成一致后自动执行故障转移。
4. **配置提供者（Configuration Provider）**：客户端连接哨兵获取当前主节点地址。

---

### 1.3.2搭建哨兵集群

哨兵配置示例：
```conf
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 10000
```

- 建议 3 个哨兵节点以上。

#### Docker-Compose 完整搭建（1 主 2 从 + 3 哨兵）

```yaml
services:
  redis-master:
    image: redis:7.2
    container_name: redis-master
    restart: always
    command: >
      redis-server
      --port 6379
      --bind 0.0.0.0
      --protected-mode no
      --masterauth 123456
      --requirepass 123456
    ports:
      - "6379:6379"
    volumes:
      - ./docker/redis-master-slave-sentinel/redis-master/data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "123456", "ping"]
      interval: 5s
      timeout: 3s
      retries: 3
    networks:
      - redis-net

  redis-slave-1:
    image: redis:7.2
    container_name: redis-slave-1
    restart: always
    command: >
      redis-server
      --port 6379
      --bind 0.0.0.0
      --protected-mode no
      --replicaof redis-master 6379
      --masterauth 123456
      --requirepass 123456
    ports:
      - "6380:6379"
    volumes:
      - ./docker/redis-master-slave-sentinel/redis-slave-1/data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "123456", "ping"]
      interval: 5s
      timeout: 3s
      retries: 3
    depends_on:
      redis-master:
        condition: service_healthy
    networks:
      - redis-net

  redis-slave-2:
    image: redis:7.2
    container_name: redis-slave-2
    restart: always
    command: >
      redis-server
      --port 6379
      --bind 0.0.0.0
      --protected-mode no
      --replicaof redis-master 6379
      --masterauth 123456
      --requirepass 123456
    ports:
      - "6381:6379"
    volumes:
      - ./docker/redis-master-slave-sentinel/redis-slave-2/data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "123456", "ping"]
      interval: 5s
      timeout: 3s
      retries: 3
    depends_on:
      redis-master:
        condition: service_healthy
    networks:
      - redis-net

  sentinel-1:
    image: redis:7.2
    container_name: sentinel-1
    restart: always
    command: redis-sentinel /etc/redis/sentinel.conf
    ports:
      - "26379:26379"
    volumes:
      - ./docker/redis-master-slave-sentinel/sentinel-1/config:/etc/redis
      - ./docker/redis-master-slave-sentinel/sentinel-1/data:/data
    depends_on:
      redis-master:
        condition: service_healthy
      redis-slave-1:
        condition: service_started
      redis-slave-2:
        condition: service_started
    networks:
      - redis-net

  sentinel-2:
    image: redis:7.2
    container_name: sentinel-2
    restart: always
    command: redis-sentinel /etc/redis/sentinel.conf
    ports:
      - "26380:26379"
    volumes:
      - ./docker/redis-master-slave-sentinel/sentinel-2/config:/etc/redis
      - ./docker/redis-master-slave-sentinel/sentinel-2/data:/data
    depends_on:
      redis-master:
        condition: service_healthy
      redis-slave-1:
        condition: service_started
      redis-slave-2:
        condition: service_started
    networks:
      - redis-net

  sentinel-3:
    image: redis:7.2
    container_name: sentinel-3
    restart: always
    command: redis-sentinel /etc/redis/sentinel.conf
    ports:
      - "26381:26379"
    volumes:
      - ./docker/redis-master-slave-sentinel/sentinel-3/config:/etc/redis
      - ./docker/redis-master-slave-sentinel/sentinel-3/data:/data
    depends_on:
      redis-master:
        condition: service_healthy
      redis-slave-1:
        condition: service_started
      redis-slave-2:
        condition: service_started
    networks:
      - redis-net

networks:
  redis-net:
    driver: bridge
```

**sentinel.conf 完整示例**：

```conf
port 26379
dir /data

# 使用容器名监控 master，依赖 Docker 内置 DNS 解析
sentinel monitor cluster-master redis-master 6379 2

# 启用主机名解析和公告（用于slave发现）
sentinel resolve-hostnames yes
sentinel announce-hostnames yes

sentinel auth-pass cluster-master 123456

sentinel down-after-milliseconds cluster-master 3000
sentinel failover-timeout cluster-master 60000
sentinel parallel-syncs cluster-master 1
```

#### sentinel.conf 参数完整说明表

| 参数 | 说明 | 建议值 |
|---|---|---|
| `sentinel monitor <name> <ip> <port> <quorum>` | 监控的主节点配置 | quorum 建议为 N/2+1 |
| `sentinel down-after-milliseconds` | 主观下线判定时间 | 5000（网络好可设 3000） |
| `sentinel failover-timeout` | 故障转移超时 | 30000 ~ 60000 |
| `sentinel parallel-syncs` | 故障转移后并行同步从节点数 | 1 |
| `sentinel auth-pass` | 主节点认证密码 | - |
| `sentinel notification-script` | 故障转移通知脚本 | 可选 |
| `sentinel client-reconfig-script` | 客户端重新配置通知脚本 | 可选 |

#### 测试故障转移和节点恢复

使用docker的pause/unpause命令而不是stop/start，否则会导致域名解析问题

```
docker pause <container_id> // 模拟节点故障
doker unpause <container_id> // 模拟节点恢复
docker kill <container_id>
```
#### 详细故障转移选举流程

**第一阶段：主观下线（SDOWN）**

1. 每个哨兵每秒向主节点发送 PING。
2. 如果主节点在 down-after-milliseconds 时间内无有效回复，该哨兵将主节点标记为主观下线（SDOWN）。

**第二阶段：客观下线（ODOWN）**
3. 将主节点标记为 SDOWN 的哨兵，向其他哨兵询问是否也认为该主节点下线。
4. 如果收到至少 quorum 个哨兵的确认（包括自己），则将该主节点标记为客观下线（ODOWN）。

**第三阶段：哨兵 Leader 选举**
5. 第一个标记主节点为 ODOWN 的哨兵向其他哨兵发送投票请求。
6. 每个哨兵在一个纪元（epoch）中只能投票一次，先到先得。
7. 获得票数 > N/2 且 >= quorum 的哨兵成为 Leader。
8. 如果在 failover-timeout 内未选出 Leader，则重新选举。

**第四阶段：故障转移执行**
9. Leader 哨兵选择新主节点，规则如下：
   - 排除不可用的从节点
   - 排除 slave-priority 为 0 的从节点
   - 优先选择 slave-priority 最小的从节点
   - 优先级相同时选择复制偏移量最大的（数据最新）
   - 偏移量也相同时选择 runid 字典序最小的
10. Leader 哨兵向选中从节点发送 `SLAVEOF NO ONE`，提升为主节点。
11. 哨兵向其他从节点发送 `SLAVEOF <new-master-ip> <new-master-port>`。
12. 哨兵持续监控旧主节点，旧主恢复后降级为从节点。

#### sentinel quorum vs odown 参数辨析

- **quorum**：客观下线所需的最少哨兵确认数。
- quorum 值建议：3 个哨兵 -> quorum = 2，5 个哨兵 -> quorum = 3
- quorum 不能小于等于 N/2，否则可能脑裂。
- **生产建议 quorum >= N/2 + 1**。

#### TILT 模式说明

- **TILT 模式**：哨兵检测到系统时钟发生异常跳变时进入的特殊保护模式。
- 触发条件：两次取时间的差值 < 0 或 > 2 秒。
- TILT 模式下行为：暂停所有操作（不发送 PING、不执行故障转移），持续 30 秒后自动退出。
- 排查：`redis-cli -p 26379 INFO sentinel | grep tilt`

---

#### 哨兵通知与脚本

**通知脚本配置**：

```conf
sentinel notification-script mymaster /etc/redis/scripts/notify.sh
```

`notify.sh` 示例：
```bash
#!/bin/bash
EVENT_TYPE=$1
EVENT_DESC=$2
echo "[$(date)] Event: $EVENT_TYPE, Desc: $EVENT_DESC" >> /var/log/redis/sentinel_events.log

if [ "$EVENT_TYPE" = "+switch-master" ]; then
    curl -X POST "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx" \
        -H 'Content-Type: application/json' \
        -d '{"msgtype":"text","text":{"content":"[Redis告警] 主节点切换: '"$EVENT_DESC"'"}}'
fi
```

**client-reconfig-script**：

```conf
sentinel client-reconfig-script mymaster /etc/redis/scripts/reconfig.sh
```

```bash
#!/bin/bash
# 参数: <master-name> <old-ip> <old-port> <new-ip> <new-port>
echo "[$(date)] Master: $3:$4 -> $5:$6" >> /var/log/redis/master_switch.log
```

#### Spring Boot 哨兵事件监听

```java
@Component
public class RedisSwitchListener {

    @Resource
    private RedisConnectionFactory redisConnectionFactory;

    @PostConstruct
    public void registerListener() {
        if (redisConnectionFactory instanceof LettuceConnectionFactory factory) {
            EventBus eventBus = factory.getClientResources().eventBus();
            eventBus.get().subscribe(event -> {
                if (event instanceof ConnectedEvent) {
                    log.info("[Redis Sentinel] Reconnected: {}", event);
                } else if (event instanceof DisconnectedEvent) {
                    log.warn("[Redis Sentinel] Disconnected: {}", event);
                }
            });
        }
    }
}
```

---

### 1.3.3RedisTemplate连接哨兵

Spring Boot 配置：
```yaml
spring:
  redis:
    sentinel:
      master: mymaster
      nodes: 127.0.0.1:26379,127.0.0.1:26380,127.0.0.1:26381
```

#### Sentinel 模式完整配置

```yaml
spring:
  redis:
    sentinel:
      master: mymaster
      nodes:
        - 127.0.0.1:26379
        - 127.0.0.1:26380
        - 127.0.0.1:26381
    password: redis123
    lettuce:
      pool:
        max-active: 20
        max-idle: 10
        min-idle: 5
        max-wait: 3000ms
      shutdown-timeout: 100ms
    timeout: 3000ms
```

---

## 1.4Redis分片集群

### 1.4.1搭建分片集群

- Redis Cluster 采用 16384 个槽位。
- 通过 `redis-cli --cluster create` 创建集群。

#### Docker-Compose 完整搭建（6 节点：3 主 3 从）

```yaml
services:

  redis-master:
    image: redis:7.2
    container_name: redis-master
    restart: always
    hostname: redis-master
    command: redis-server /usr/local/etc/redis/redis.conf
    volumes:
      - ./conf/redis-master.conf:/usr/local/etc/redis/redis.conf
      - ./data/redis-master:/data
    ports:
      - "7001:7001"
      - "17001:17001"
    networks:
      - redis-net

  redis-master-2:
    image: redis:7.2
    container_name: redis-master-2
    restart: always
    hostname: redis-master-2
    command: redis-server /usr/local/etc/redis/redis.conf
    volumes:
      - ./conf/redis-master-2.conf:/usr/local/etc/redis/redis.conf
      - ./data/redis-master-2:/data
    ports:
      - "7002:7002"
      - "17002:17002"
    networks:
      - redis-net

  redis-master-3:
    image: redis:7.2
    container_name: redis-master-3
    restart: always
    hostname: redis-master-3
    command: redis-server /usr/local/etc/redis/redis.conf
    volumes:
      - ./conf/redis-master-3.conf:/usr/local/etc/redis/redis.conf
      - ./data/redis-master-3:/data
    ports:
      - "7003:7003"
      - "17003:17003"
    networks:
      - redis-net

  redis-slave-1:
    image: redis:7.2
    container_name: redis-slave-1
    restart: always
    hostname: redis-slave-1
    command: redis-server /usr/local/etc/redis/redis.conf
    volumes:
      - ./conf/redis-slave-1.conf:/usr/local/etc/redis/redis.conf
      - ./data/redis-slave-1:/data
    ports:
      - "7004:7004"
      - "17004:17004"
    networks:
      - redis-net

  redis-slave-2:
    image: redis:7.2
    container_name: redis-slave-2
    restart: always
    hostname: redis-slave-2
    command: redis-server /usr/local/etc/redis/redis.conf
    volumes:
      - ./conf/redis-slave-2.conf:/usr/local/etc/redis/redis.conf
      - ./data/redis-slave-2:/data
    ports:
      - "7005:7005"
      - "17005:17005"
    networks:
      - redis-net

  redis-slave-3:
    image: redis:7.2
    container_name: redis-slave-3
    restart: always
    hostname: redis-slave-3
    command: redis-server /usr/local/etc/redis/redis.conf
    volumes:
      - ./conf/redis-slave-3.conf:/usr/local/etc/redis/redis.conf
      - ./data/redis-slave-3:/data
    ports:
      - "7006:7006"
      - "17006:17006"
    networks:
      - redis-net

networks:
  redis-net:
    name: redis-net
    driver: bridge
```

#### 创建集群命令

```bash
# 启动所有节点
docker-compose up -d

# 创建集群（3 主 3 从，replicas 1 表示每个主节点有 1 个从节点）
docker exec -it redis-master redis-cli \
-a 123456 \
--cluster create \
redis-master:7001 \
redis-master-2:7002 \
redis-master-3:7003 \
redis-slave-1:7004 \
redis-slave-2:7005 \
redis-slave-3:7006 \
--cluster-replicas 1
```

#### 集群健康验证

```bash
# 查看集群状态
redis-cli -p 7001 CLUSTER INFO
# 预期: cluster_state:ok, cluster_slots_assigned:16384

# 查看集群节点
redis-cli -p 7001 CLUSTER NODES

# 查看槽位分配
redis-cli -p 7001 CLUSTER SLOTS

# 数据验证 - 跨槽位写入
redis-cli -c -p 7001 SET user:1:name "Alice"
redis-cli -c -p 7001 SET order:100 "data"

# 查看 key 所在槽位
redis-cli -p 7001 CLUSTER KEYSLOT user:1:name
```

---

### 1.4.2散列插槽

- `key` 通过 CRC16 映射到槽。
- 节点负责部分槽位。

#### Hash Tag 机制

Hash Tag 允许将多个 key 分配到同一个槽位：

```bash
# 这些 key 会被分配到同一个槽位
user:{1001}:name    -> hash("1001")
user:{1001}:age     -> hash("1001")
user:{1001}:email   -> hash("1001")
order:{1001}:items  -> hash("1001")
```

**注意事项**：
- `{}` 内必须包含实际字符，`user:{}:name` 会对 `{}` 取 hash，等于没有 hash tag。
- 过多使用同一个 hash tag 会导致数据倾斜（热点问题）。
- Hash tag 仅在有跨 key 操作需求时使用，普通场景不建议。

**Hash Tag Java 示例**：
```java
String userKeyPrefix = "user:{" + userId + "}:";
redisTemplate.opsForValue().set(userKeyPrefix + "name", name);
redisTemplate.opsForValue().set(userKeyPrefix + "age", age);
// 现在可以在同一节点上使用 Lua 脚本操作这些 key
String luaScript =
    "local name = redis.call('GET', KEYS[1]) " +
    "local age = redis.call('GET', KEYS[2]) " +
    "return {name, age}";
redisTemplate.execute(
    new DefaultRedisScript<>(luaScript, List.class),
    Arrays.asList(userKeyPrefix + "name", userKeyPrefix + "age")
);
```

#### Resharding 步骤详解（槽位迁移）

```bash
# 1. 查看当前集群状态
redis-cli --cluster check 127.0.0.1:7001

# 2. 开始重新分片（交互式）
redis-cli --cluster reshard 127.0.0.1:7001
# 交互式输入: 迁移槽位数、目标节点ID、源节点ID

# 3. 非交互式命令
redis-cli --cluster reshard 127.0.0.1:7001 \
  --cluster-from <source-node-id> \
  --cluster-to <target-node-id> \
  --cluster-slots 500 \
  --cluster-yes

# 4. 迁移完成后验证
redis-cli --cluster check 127.0.0.1:7001
```

#### 槽位迁移性能注意事项

- 槽位迁移过程中，源节点和目标节点均不阻塞读写。
- 迁移过程：IMPORTING -> MIGRATING -> SETSLOT NODE。
- 若槽位有大量 key（百万级），迁移可能需要数分钟到半小时。
- 建议业务低峰期进行，单次迁移槽位不超过 500 个，分散多次执行。
- 使用 `--cluster-timeout 60000` 加大超时时间。

---

### 1.4.3集群伸缩

- `--cluster add-node` 添加节点。
- `--cluster reshard` 迁移槽位。

#### 集群扩容完整步骤

```bash
# 1. 启动新节点
# 2. 将新节点加入集群
redis-cli --cluster add-node <new-node-ip>:7007 <existing-node-ip>:7001

# 3. 验证
redis-cli -p 7001 CLUSTER NODES

# 4. 给新节点分配槽位
redis-cli --cluster reshard <existing-node-ip>:7001 \
  --cluster-from <source-node-id> \
  --cluster-to <new-node-id> \
  --cluster-slots 1000 \
  --cluster-yes

# 5. 为新主节点添加从节点
redis-cli --cluster add-node <slave-ip>:7008 <master-ip>:7007 \
  --cluster-slave --cluster-master-id <new-master-node-id>
```

#### 集群缩容完整步骤

```bash
# 1. 移除从节点
redis-cli --cluster del-node <existing-node-ip>:7001 <slave-node-id>

# 2. 将待下线主节点的槽位迁走
redis-cli --cluster reshard <existing-node-ip>:7001 \
  --cluster-from <node-to-remove-id> \
  --cluster-to <remaining-node-id> \
  --cluster-slots <slot-count> \
  --cluster-yes

# 3. 移除已清空槽位的主节点
redis-cli --cluster del-node <existing-node-ip>:7001 <node-to-remove-id>
```

#### 集群节点替换流程（主从替换）

```bash
# 1. 如果是主节点，先做手动故障转移
redis-cli -p <slave-port> CLUSTER FAILOVER

# 2. 确认故障转移完成
redis-cli -p 7001 CLUSTER NODES

# 3. 移除旧节点
redis-cli --cluster del-node 127.0.0.1:7001 <old-node-id>

# 4. 添加新节点作为从节点
redis-cli --cluster add-node <new-node-ip>:<port> 127.0.0.1:7001 \
  --cluster-slave --cluster-master-id <target-master-id>
```

---

### 1.4.4故障转移

- 主节点宕机，自动切换为从节点。
- 客户端通过重定向 `MOVED` 获取新地址。

#### 自动故障转移流程

1. 每个 Redis Cluster 节点通过 Gossip 协议互相通信。
2. 节点 A 超过 `cluster-node-timeout` 未收到节点 B 的 PONG 回复，将节点 B 标记为 PFAIL（可能故障）。
3. 当足够多的节点（至少半数主节点）都认为节点 B 可能故障时，将 PFAIL 升级为 FAIL。
4. 节点 B 的从节点发起故障转移选举。
5. 获得多数票的从节点成为新主节点，接管故障主节点的所有槽位。

#### 手动故障转移

```bash
# 将当前连接的从节点提升为主节点
redis-cli -p 7004 CLUSTER FAILOVER

# 强制故障转移（即使主节点正常）
redis-cli -p 7004 CLUSTER FAILOVER FORCE

# 接管式故障转移（不等待数据同步）
redis-cli -p 7004 CLUSTER FAILOVER TAKEOVER
```

#### Gossip 协议详解

Redis Cluster 节点之间使用 Gossip 协议交换信息：

- **PING 消息**：节点向其他节点发送自身和已知的部分节点信息。
- **PONG 消息**：对 PING 的回复。
- **MEET 消息**：通知新节点加入集群。
- **FAIL 消息**：广播通知某个节点故障。
- **PUBLISH 消息**：跨节点广播。

#### Cluster BUS 端口

每个 Redis Cluster 节点除了客户端通信端口（如 7001），还有一个集群总线端口，默认为 `端口 + 10000`（如 17001）。

- **用途**：节点间 Gossip 协议通信、故障检测、配置更新。
- **注意**：防火墙需要同时开放客户端端口和 BUS 端口；Docker 中必须同时映射两个端口。

---

### 1.4.5RedisTemplate访问分片集群

Spring Boot 配置：
```yaml
spring:
  redis:
    cluster:
      nodes:
        - 127.0.0.1:7001
        - 127.0.0.1:7002
        - 127.0.0.1:7003
```

#### Cluster 模式完整配置

```yaml
spring:
  redis:
    cluster:
      nodes:
        - 127.0.0.1:7001
        - 127.0.0.1:7002
        - 127.0.0.1:7003
        - 127.0.0.1:7004
        - 127.0.0.1:7005
        - 127.0.0.1:7006
      max-redirects: 3
    password: redis123
    lettuce:
      pool:
        max-active: 20
        max-idle: 10
        min-idle: 5
        max-wait: 3000ms
      cluster:
        refresh:
          adaptive: true
          period: 30s
    timeout: 3000ms
```

---

#### 集群运维

#### 集群扩容缩容最佳实践

1. **扩容时机**：内存使用率 > 70%，CPU 使用率 > 60%，网卡接近带宽上限。
2. **扩容步骤**：先扩从节点，再扩主节点并迁移槽位。
3. **缩容步骤**：先迁走槽位，再移除节点。
4. **迁移策略**：每次迁移不超过 10% 的槽位，分多次执行。
5. **回滚预案**：保留被移除节点的数据文件至少 24 小时。
6. **监控重点**：迁移期间监控 cluster_slots_ok、cluster_slots_pfail 以及节点延迟。

#### 集群常见问题排查

| 问题 | 排查命令 | 常见原因 |
|---|---|---|
| `CLUSTERDOWN` | `CLUSTER INFO` | 有槽位未分配，检查主节点是否宕机 |
| `MOVED` 错误 | 使用 `-c` 参数连接 | 客户端连的节点不负责该 key 的槽位 |
| `ASK` 重定向 | 等待迁移完成 | 槽位正在迁移中 |
| 分片不均 | `CLUSTER SLOTS` | 使用 `rebalance` 命令重新分配 |
| 节点无法加入 | 检查 BUS 端口 | 确保防火墙开放了端口+10000 |
| `FAIL` 状态节点 | `CLUSTER NODES` | 节点宕机超时，CLUSTER FORGET 清除 |

```bash
# 快速排查命令
redis-cli -p 7001 CLUSTER INFO
redis-cli -p 7001 CLUSTER NODES | grep -E "fail|disconnected"
redis-cli -p 7001 CLUSTER FORGET <node-id>
```

---

#### 1.4.6 Codis 与 Redis Cluster 对比

| 维度 | Redis Cluster | Codis |
|---|---|---|
| 官方支持 | Redis 官方原生 | 豌豆荚开源，非官方 |
| 架构 | 无中心 (P2P Gossip) | 有中心 (proxy + zookeeper/etcd) |
| 槽位数量 | 16384 固定 | 1024 可自定义 |
| 节点通信 | Gossip 协议，节点间直连 | 通过 Proxy 中转 |
| 扩缩容 | redis-cli 迁移 | Dashboard 自动迁 slot |
| 事务支持 | 同 slot 多 key 支持 | 单 key 或同组支持 |
| Lua 脚本 | 支持（同 slot） | 有限支持 |
| 客户端兼容性 | 需 Cluster 协议客户端 | 支持所有 Redis 客户端 |
| 运维复杂度 | 较复杂 | 较简单（Dashboard） |
| 性能 | 直连，无 Proxy 损耗 | Proxy 增加一层网络开销 |
| 社区活跃度 | 非常活跃 | 较不活跃 |

**选型建议**：新项目选 Redis Cluster，遗留系统迁移可选 Codis。

---

# 2.多级缓存

## 2.1什么是多级缓存

- JVM 进程缓存 + Redis + DB。
- 本地缓存命中快，Redis 兜底，DB 最后。

多级缓存的完整链路：
```
请求 > Nginx 本地缓存(lua_shared_dict) > Redis 缓存 > JVM 进程缓存(Caffeine) > 数据库
```

每一层都拦截掉大部分请求，流量逐层递减，最终只有极少数请求到达数据库。

## 2.2JVM进程缓存

### 2.2.1导入Demo数据

- 准备 shop 数据与坐标数据。

### 2.2.2导入Demo工程

- Maven 工程包含 Caffeine + Redis + Nginx/OpenResty。

### 2.2.3初识Caffeine

- 本地高性能缓存，支持 LRU/LFU。

```java
Cache<Long, Shop> cache = Caffeine.newBuilder()
		.maximumSize(10_000)
		.expireAfterWrite(Duration.ofMinutes(10))
		.build();
```

#### Caffeine 完整配置详解

```java
Cache<Long, Shop> cache = Caffeine.newBuilder()
        // === 容量控制 ===
        .maximumSize(10_000)             // 最大缓存条目数（W-TinyLFU 淘汰）
        .maximumWeight(100_000)          // 最大权重（需配合 weigher）
        .weigher((Long key, Shop value) -> value.getSize())

        // === 过期策略 ===
        .expireAfterWrite(Duration.ofMinutes(10))   // 写入后过期
        .expireAfterAccess(Duration.ofMinutes(5))   // 访问后过期
        .expireAfter(new Expiry<Long, Shop>() {      // 自定义过期
            public long expireAfterCreate(Long k, Shop v, long t) {
                return TimeUnit.MINUTES.toNanos(v.getTtlMinutes());
            }
            public long expireAfterUpdate(Long k, Shop v, long t, long d) {
                return d;
            }
            public long expireAfterRead(Long k, Shop v, long t, long d) {
                return d;
            }
        })

        // === 刷新策略 ===
        .refreshAfterWrite(Duration.ofMinutes(1))

        // === 统计 ===
        .recordStats()

        // === 监听器 ===
        .removalListener((Long key, Shop value, RemovalCause cause) -> {
            log.info("Key {} removed, reason: {}", key, cause);
        })

        .build();
```

#### Caffeine 刷新与 Loading Cache

```java
LoadingCache<Long, Shop> cache = Caffeine.newBuilder()
        .maximumSize(10_000)
        .refreshAfterWrite(Duration.ofMinutes(1))
        .expireAfterWrite(Duration.ofMinutes(10))
        .build(key -> shopService.queryFromDb(key));

Shop shop = cache.get(shopId);
cache.refresh(shopId);
Map<Long, Shop> shops = cache.getAll(Arrays.asList(1L, 2L, 3L));
```

#### Caffeine vs Guava Cache vs Ehcache 对比

| 特性 | Caffeine | Guava Cache | Ehcache 3 |
|---|---|---|---|
| 淘汰算法 | W-TinyLFU（高命中率） | LRU | 多种算法 |
| 命中率 | 非常高 | 较高 | 较高 |
| 内存效率 | 高 | 中 | 中（支持 off-heap） |
| 异步缓存 | 原生 CompletableFuture | 不支持 | 支持 |
| 持久化 | 不支持 | 不支持 | 支持 |
| Spring Boot 集成 | 手动集成 | 手动集成 | 自动配置 |
| 维护状态 | 活跃 | 维护模式 | 活跃 |

**推荐**：大多数场景选 Caffeine，命中率最高。Ehcache 适合需要 off-heap 或磁盘持久化的场景。

#### Caffeine Stats 统计监控

```java
Cache<Long, Shop> cache = Caffeine.newBuilder()
        .maximumSize(10_000)
        .recordStats()
        .build();

ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
scheduler.scheduleAtFixedRate(() -> {
    CacheStats stats = cache.stats();
    System.out.printf(
        "[Caffeine Stats] hitRate=%.2f%%, missCount=%d, evictionCount=%d, " +
        "averageLoadPenalty=%.2fms%n",
        stats.hitRate() * 100, stats.missCount(),
        stats.evictionCount(), stats.averageLoadPenalty() / 1_000_000.0);
}, 30, 30, TimeUnit.SECONDS);
```

关键指标：
- `hitRate`：命中率，建议 > 90%
- `evictionCount`：淘汰次数，频繁说明 maxSize 过小
- `averageLoadPenalty`：平均加载耗时

---

### 2.2.4实现进程缓存

- 查询流程：先 Caffeine，未命中再查 Redis/DB。

```java
Shop shop = cache.getIfPresent(id);
if (shop == null) { shop = queryRedisOrDb(id); cache.put(id, shop); }
```

#### 完整的三层查询实现

```java
@Service
public class ShopService {

    private static final String CACHE_PREFIX = "cache:shop:";

    @Autowired private Cache<Long, Shop> caffeineCache;
    @Autowired private StringRedisTemplate redisTemplate;
    @Autowired private ShopMapper shopMapper;
    private final ObjectMapper mapper = new ObjectMapper();

    public Shop queryShop(Long shopId) {
        // L1: Caffeine 本地缓存
        Shop shop = caffeineCache.getIfPresent(shopId);
        if (shop != null) return shop;

        // L2: Redis 分布式缓存
        String redisKey = CACHE_PREFIX + shopId;
        String json = redisTemplate.opsForValue().get(redisKey);
        if (json != null && !json.isEmpty()) {
            try {
                shop = mapper.readValue(json, Shop.class);
                caffeineCache.put(shopId, shop);
                return shop;
            } catch (JsonProcessingException ignored) {}
        }

        // L3: 数据库
        shop = shopMapper.selectById(shopId);
        if (shop != null) {
            caffeineCache.put(shopId, shop);
            try {
                int ttl = 600 + ThreadLocalRandom.current().nextInt(60);
                redisTemplate.opsForValue()
                    .set(redisKey, mapper.writeValueAsString(shop), Duration.ofSeconds(ttl));
            } catch (JsonProcessingException ignored) {}
        }
        return shop;
    }
}
```

---

## 2.3Lua语法

### 2.3.1初始Lua

- 变量定义：`local a = 1`。
- `redis.call` 调用 Redis 命令。

### 2.3.2变量和循环

```lua
for i = 1, 3 do
		local v = redis.call('get', KEYS[i])
end
```

### 2.3.3函数和条件控制

```lua
if tonumber(ARGV[1]) > 0 then
		return 1
else
		return 0
end
```

#### Lua 数据类型详解

```lua
-- 1. nil
local a = nil

-- 2. boolean
local flag = true

-- 3. number（Lua 只有双精度浮点）
local count = 100

-- 4. string
local name = "hello"
local multi = [[
多行字符串
line1
line2
]]

-- 5. table（数组和字典合一）
local arr = {10, 20, 30}
local dict = {name = "tom", age = 20}
local mixed = {1, 2, name = "test"}

-- 6. function（一等公民）
local fn = function(x) return x * 2 end
```

#### Lua Tables 深入

```lua
-- 数组遍历
local fruits = {"apple", "banana", "cherry"}
for i = 1, #fruits do print(i, fruits[i]) end

-- ipairs：只遍历连续整数 key
for i, v in ipairs(fruits) do print(i, v) end

-- pairs：遍历所有键值对
local user = {name = "alice", age = 25}
for k, v in pairs(user) do print(k, v) end

-- 表操作
table.insert(fruits, "orange")
table.insert(fruits, 2, "grape")
table.remove(fruits, 1)
table.sort(fruits)

-- 获取长度（# 只对连续整数 key 有效）
local len = #fruits
```

#### Lua 字符串函数

```lua
local s = "hello world"
string.len(s)           -- 11
string.upper(s)         -- "HELLO WORLD"
string.lower(s)         -- "hello world"
string.sub(s, 1, 5)     -- "hello"
string.find(s, "world") -- 7, 11
string.gsub(s, "world", "redis")  -- "hello redis"
string.format("key:%d:name", 100) -- "key:100:name"
local full = "hello" .. " " .. "redis"
local num = tonumber("123")
```

#### cjson 使用（Redis Lua 内置）

```lua
local data = {id = 100, name = "phone", price = 2999}
local json = cjson.encode(data)
-- {"id":100,"name":"phone","price":2999}

local obj = cjson.decode('{"id":100,"name":"phone"}')

-- 处理 null
local decoded = cjson.decode('{"key":null}')
if decoded.key == cjson.null then print("key is null") end

cjson.encode({})          -- "[]"
cjson.encode(cjson.empty_array)  -- "[]"
```

#### 常见 Redis Lua 脚本模式

**1. CAS（Compare and Swap）原子更新**

```lua
local current = redis.call('GET', KEYS[1])
if current == ARGV[1] then
    redis.call('SET', KEYS[1], ARGV[2])
    return 1
else
    return 0
end
```

**2. 批量检查后再操作**

```lua
for i, key in ipairs(KEYS) do
    if redis.call('EXISTS', key) == 0 then
        return 0
    end
end
for i, key in ipairs(KEYS) do
    redis.call('SET', key, ARGV[i])
end
return 1
```

**3. 分布式锁释放（原子性检查）**

```lua
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
else
    return 0
end
```

**4. 限速器（滑动窗口）**

```lua
local now = tonumber(redis.call('TIME')[1])
local window = tonumber(ARGV[1])
local maxRequests = tonumber(ARGV[2])

redis.call('ZREMRANGEBYSCORE', KEYS[1], 0, now - window)
local current = redis.call('ZCARD', KEYS[1])
if current < maxRequests then
    redis.call('ZADD', KEYS[1], now, now .. '-' .. math.random())
    redis.call('EXPIRE', KEYS[1], window + 1)
    return 1
else
    return 0
end
```

#### Lua 脚本调试技巧

```bash
# 使用 redis-cli --ldb 进入调试模式
redis-cli --ldb --eval script.lua key1 key2 , arg1 arg2
# 调试命令: s(step), p(print), b(break), c(continue), list, help

# 在脚本中输出日志
redis.log(redis.LOG_NOTICE, "debug: key=" .. KEYS[1])

# pcall 捕获错误
local ok, result = pcall(function()
    return redis.call('SET', KEYS[1], ARGV[1])
end)
if not ok then
    redis.log(redis.LOG_WARNING, "SET failed: " .. tostring(result))
    return -1
end
```

---

## 2.4多级缓存

### 2.4.1安装OpenResty

- OpenResty = Nginx + Lua。
- 常用目录：`conf/nginx.conf`。

#### OpenResty 完整安装指南

**CentOS**：
```bash
yum install -y pcre-devel openssl-devel gcc curl
wget https://openresty.org/package/centos/openresty.repo -O /etc/yum.repos.d/openresty.repo
yum install -y openresty
/usr/local/openresty/bin/openresty -v
```

**Docker**：
```bash
docker pull openresty/openresty:jammy
docker run -d --name openresty -p 80:80 -p 8081:8081 \
  -v $(pwd)/nginx.conf:/usr/local/openresty/nginx/conf/nginx.conf \
  -v $(pwd)/lua:/usr/local/openresty/nginx/lua \
  openresty/openresty:jammy
```

**Ubuntu**：
```bash
wget -qO - https://openresty.org/package/pubkey.gpg | apt-key add -
echo "deb http://openresty.org/package/ubuntu $(lsb_release -sc) main" \
    | tee /etc/apt/sources.list.d/openresty.list
apt update && apt install -y openresty
```

---

### 2.4.2OpenResty快速入门

```nginx
server {
		listen 8081;
		location / {
				content_by_lua_block {
						ngx.say("hello openresty")
				}
		}
}
```

---

### 2.4.3OpenResty获取请求参数

```lua
local args = ngx.req.get_uri_args()
local id = args["id"]
```

#### OpenResty 请求处理完整 API

```lua
-- URI 参数: GET /shop?id=123
local args = ngx.req.get_uri_args()
local id = args["id"]

-- POST 表单参数
ngx.req.read_body()
local post_args = ngx.req.get_post_args()

-- HTTP 头
local headers = ngx.req.get_headers()
local user_agent = headers["User-Agent"]

-- 请求方法
local method = ngx.req.get_method()

-- 设置响应
ngx.status = 200
ngx.header["Content-Type"] = "application/json; charset=utf-8"
ngx.header["X-Cache-Status"] = "HIT"
ngx.say('{"code":200,"data":{}}')
ngx.exit(200)
```

---

### 2.4.4封装Http请求工具

```lua
local http = require("resty.http")
local client = http.new()
local res = client:request_uri("http://127.0.0.1:8080/shop/" .. id)
```

#### resty.http 完整使用

```lua
local http = require("resty.http")
local httpc = http.new()
httpc:set_timeouts(1000, 3000, 3000)  -- connect, send, read ms

-- GET 请求
local res, err = httpc:request_uri("http://127.0.0.1:8080/api/shop/" .. id, {
    method = "GET",
    headers = { ["Content-Type"] = "application/json" },
    keepalive_timeout = 60000,
    keepalive_pool = 10,
})

if not res then
    ngx.log(ngx.ERR, "HTTP request failed: ", err)
    ngx.exit(500)
end

local body = res:read_body()

-- POST 请求
local post_res, post_err = httpc:request_uri("http://127.0.0.1:8080/api/update", {
    method = "POST",
    body = '{"name":"test"}',
    headers = { ["Content-Type"] = "application/json" },
})

httpc:set_keepalive()
```

---

### 2.4.5向tomcat发送http请求

- OpenResty 作为边缘层，查询本地缓存或 Redis。
- 未命中则回源 Tomcat。

---

### 2.4.6根据商品id对tomcat集群负载均衡

- 通过 `hash` 或一致性算法选择后端。

```lua
local index = id % #backend_list + 1
local backend = backend_list[index]
```

#### 完整的 Nginx 多级缓存配置文件

```nginx
# nginx.conf - OpenResty 多级缓存
worker_processes auto;
error_log logs/error.log info;

events { worker_connections 10240; }

http {
    # === L1: 本地缓存共享字典 ===
    lua_shared_dict shop_cache 100m;
    lua_shared_dict dict_cache 50m;

    upstream backend_tomcat {
        server 127.0.0.1:8080 weight=1;
        server 127.0.0.1:8081 weight=1;
        keepalive 32;
    }

    init_worker_by_lua_block {
        local redis = require("resty.redis")
        local red = redis:new()
        red:set_timeout(1000)
        local ok, err = red:connect("127.0.0.1", 6379)
        if ok then
            red:auth("redis123")
            red:set_keepalive(10000, 100)
        end
    }

    server {
        listen 80;

        location /api/shop {
            default_type application/json;
            rewrite_by_lua_block {
                local args = ngx.req.get_uri_args()
                local id = args["id"]
                if not id then ngx.exit(400) end

                local shop_cache = ngx.shared.shop_cache
                local cache_key = "shop:" .. id

                -- L1: 本地共享字典
                local cached = shop_cache:get(cache_key)
                if cached then
                    ngx.header["X-Cache-Level"] = "L1"
                    ngx.say(cached)
                    ngx.exit(200)
                end

                -- 防击穿锁
                local lock_key = "lock:shop:" .. id
                local locked = shop_cache:add(lock_key, "1", 10)

                -- L2: Redis
                local redis = require("resty.redis")
                local red = redis:new()
                red:set_timeout(1000)
                local ok, err = red:connect("redis-master", 6379)
                if ok then
                    red:auth("redis123")
                    local val, rerr = red:get(cache_key)
                    red:set_keepalive(10000, 100)
                    if val and val ~= ngx.null then
                        shop_cache:set(cache_key, val, 600)
                        if locked then shop_cache:delete(lock_key) end
                        ngx.header["X-Cache-Level"] = "L2"
                        ngx.say(val)
                        ngx.exit(200)
                    end
                end

                if not locked then
                    ngx.header["Retry-After"] = "1"
                    ngx.exit(503)
                end

                -- L3: 回源 Tomcat
                local http = require("resty.http")
                local httpc = http.new()
                local res, herr = httpc:request_uri(
                    "http://backend_tomcat/api/shop?id=" .. id,
                    { method = "GET", keepalive_timeout = 60000 }
                )
                if not res or res.status ~= 200 then
                    shop_cache:delete(lock_key)
                    ngx.exit(502)
                end

                local body = res:read_body()
                httpc:set_keepalive()
                shop_cache:set(cache_key, body, 600)
                shop_cache:delete(lock_key)
                ngx.header["X-Cache-Level"] = "L3"
                ngx.say(body)
            }
        }

        # 缓存失效接口
        location /api/cache/invalidate {
            default_type application/json;
            content_by_lua_block {
                local args = ngx.req.get_uri_args()
                ngx.shared.shop_cache:delete(args["key"])
                ngx.say('{"code":200}')
            }
        }
    }
}
```

---

### 2.4.7Redis缓存预热

- 启动时提前写入热点数据。
- 避免上线瞬时击穿。

#### 预热脚本示例

```lua
-- warmup.lua (在 init_worker_by_lua_file 中调用)
local redis = require("resty.redis")
local hot_ids = {1, 2, 3, 10, 20, 50, 100, 200, 500, 1000}

local red = redis:new()
red:set_timeout(3000)
local ok = red:connect("127.0.0.1", 6379)
if not ok then return end
red:auth("redis123")

local cached = 0
for _, id in ipairs(hot_ids) do
    local val = red:get("cache:shop:" .. id)
    if val and val ~= ngx.null and val ~= false then
        ngx.shared.shop_cache:set("shop:" .. id, val, 600)
        cached = cached + 1
    end
end
red:set_keepalive(10000, 100)
ngx.log(ngx.INFO, "Warmup: loaded ", cached, " items from Redis")
```

---

### 2.4.8查询Redis

```lua
local redis = require("resty.redis")
local r = redis:new()
r:connect("127.0.0.1", 6379)
local val = r:get(key)
```

#### lua-resty-redis 完整操作 API

```lua
local redis = require("resty.redis")
local red = redis:new()
red:set_timeout(1000)

local ok, err = red:connect("127.0.0.1", 6379)
if not ok then return end
red:auth("redis123")
red:select(0)

-- String
red:set("key", "value")
red:setex("key", 3600, "value")
local val = red:get("key")

-- Hash
red:hset("hash_key", "field", "val")
red:hget("hash_key", "field")

-- Pipeline
red:init_pipeline()
red:get("key1")
red:get("key2")
red:get("key3")
local results = red:commit_pipeline()

-- 归还连接池（重要！）
red:set_keepalive(10000, 100)
```

---

### 2.4.9nginx本地缓存

- 使用 `lua_shared_dict` 作为 OpenResty 本地缓存。

```nginx
lua_shared_dict shop_cache 50m;
```

#### lua_shared_dict 完整操作

```lua
local cache = ngx.shared.shop_cache

cache:add("key", "value", 3600)      -- SET NX
cache:set("key", "new_value", 3600)  -- SET
cache:replace("key", "updated", 3600) -- SET XX
local val = cache:get("key")
cache:delete("key")
local new_val = cache:incr("counter", 1)
cache:flush_all()
local cap = cache:capacity()
local free = cache:free_space()
```

注意事项：
- 所有 worker 进程共享，无需额外同步。
- 数据存储在共享内存，重启丢失。
- 淘汰策略：LRU。

---

## 2.5缓存同步

### 2.5.1数据同步策略

- 订阅 binlog 变化，及时更新缓存。
- 避免业务层遗漏更新。

#### 三种缓存同步策略对比

| 策略 | 实现方式 | 优点 | 缺点 |
|---|---|---|---|
| **设置有效期** | EXPIRE key 3600 | 简单，无需改造 | 到期前不一致 |
| **双写** | 更新 DB + 删除缓存 | 实时 | 耦合，易遗漏 |
| **订阅 binlog** | Canal/Debezium 监听 | 解耦、实时、完整 | 引入额外组件 |

推荐：订阅 binlog 是最佳实践。

---

### 2.5.2安装Canal

- Canal 监听 MySQL binlog。
- 配置 `canal.instance.master.address`。

#### Canal + Docker 完整搭建

```yaml
version: '3.8'
services:
  canal-server:
    image: canal/canal-server:v1.1.7
    container_name: canal-server
    ports:
      - "11111:11111"
    environment:
      - CANAL_DESTINATIONS=example
      - CANAL_INSTANCE_MASTER_ADDRESS=192.168.1.100:3306
      - CANAL_INSTANCE_DBUSER=canal
      - CANAL_INSTANCE_DBPASSWD=canal
      - CANAL_INSTANCE_FILTER_REGEX=test_db\\..*
    networks:
      - canal-net

networks:
  canal-net:
    driver: bridge
```

**MySQL 开启 binlog**：

```ini
[mysqld]
server-id = 1
log-bin = mysql-bin
binlog-format = ROW
binlog-row-image = FULL
expire_logs_days = 7
```

```sql
CREATE USER 'canal'@'%' IDENTIFIED BY 'canal';
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
FLUSH PRIVILEGES;
```

---

### 2.5.3监听canal实现缓存同步

- 监听到 update/delete 时，删除缓存。
- 支持消息队列或直接回调。

#### Canal Java 客户端完整代码

```java
@Component
public class CanalCacheSync {

    @Autowired private StringRedisTemplate redisTemplate;

    private CanalConnector connector;
    private final ExecutorService executor = Executors.newSingleThreadExecutor();
    private volatile boolean running = true;
    private static final String REDIS_KEY_PREFIX = "cache:shop:";

    @PostConstruct
    public void init() {
        connector = CanalConnectors.newSingleConnector(
            new InetSocketAddress("127.0.0.1", 11111), "example", "", "");

        executor.submit(() -> {
            connector.connect();
            connector.subscribe("test_db\\..*");
            connector.rollback();

            while (running) {
                try {
                    Message message = connector.getWithoutAck(1000);
                    long batchId = message.getId();
                    if (batchId == -1 || message.getEntries().isEmpty()) {
                        Thread.sleep(100);
                        continue;
                    }
                    handleEntries(message.getEntries());
                    connector.ack(batchId);
                } catch (Exception e) {
                    e.printStackTrace();
                    connector.rollback();
                }
            }
        });
    }

    private void handleEntries(List<Entry> entries) {
        for (Entry entry : entries) {
            if (entry.getEntryType() == EntryType.TRANSACTIONBEGIN
                || entry.getEntryType() == EntryType.TRANSACTIONEND) continue;

            RowChange rowChange;
            try {
                rowChange = RowChange.parseFrom(entry.getStoreValue());
            } catch (Exception e) {
                throw new RuntimeException("Parse error: " + entry, e);
            }

            EventType eventType = rowChange.getEventType();
            String tableName = entry.getHeader().getTableName();

            for (RowData rowData : rowChange.getRowDatasList()) {
                handleRow(tableName, eventType, rowData);
            }
        }
    }

    private void handleRow(String tableName, EventType eventType, RowData rowData) {
        if (!"shop".equals(tableName)) return;

        if (eventType == EventType.DELETE || eventType == EventType.UPDATE) {
            String id = getColumnValue(rowData.getBeforeColumnsList(), "id");
            if (id != null) {
                redisTemplate.delete(REDIS_KEY_PREFIX + id);
                // 通知 Nginx 清除本地缓存
                redisTemplate.convertAndSend("cache:invalidate", "shop:" + id);
            }
        }
    }

    private String getColumnValue(List<Column> columns, String columnName) {
        for (Column col : columns) {
            if (columnName.equals(col.getName())) return col.getValue();
        }
        return null;
    }

    @PreDestroy
    public void destroy() {
        running = false;
        executor.shutdown();
        if (connector != null) connector.disconnect();
    }
}
```

#### 缓存更新通知机制（Redis Pub/Sub 通知 Nginx）

应用层通过 Redis Pub/Sub 通知 Nginx 清除本地缓存：

```nginx
# Nginx 缓存失效接口
location /invalidate {
    content_by_lua_block {
        local args = ngx.req.get_uri_args()
        ngx.shared.shop_cache:delete(args["key"])
        ngx.say('{"status":"ok"}')
    }
}
```

#### 替代方案：Debezium 对比

| 特性 | Canal | Debezium |
|---|---|---|
| 开发者 | 阿里巴巴 | Red Hat |
| 架构 | Server + Client (Java) | Kafka Connect 框架 |
| 数据库支持 | 主要为 MySQL | MySQL, PostgreSQL, MongoDB 等 |
| 输出格式 | 自定义 protobuf | Kafka 消息 (Avro/JSON) |
| 扩展性 | 较有限 | 极强（Connect 生态） |
| 运维部署 | 简单（单进程） | 较重（需 Kafka + Connect） |
| 适用场景 | 中小规模、仅 MySQL | 大规模、多数据源 |

---

## 2.6 多级缓存完整实战项目

### 2.6.1 项目结构

```
shop-cache-demo/
|-- src/main/java/com/example/shop/
|   |-- ShopApplication.java
|   |-- controller/ShopController.java
|   |-- service/ShopService.java
|   |-- cache/CaffeineConfig.java
|   |-- cache/CacheSyncService.java
|   |-- mapper/ShopMapper.java
|   +-- entity/Shop.java
|-- openresty/
|   |-- nginx.conf
|   +-- lua/
|       |-- shop_access.lua
|       +-- warmup.lua
+-- docker-compose.yml
```

### 2.6.2 核心配置

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/shop_db
    username: root
    password: root
  redis:
    cluster:
      nodes:
        - 127.0.0.1:7001
        - 127.0.0.1:7002
        - 127.0.0.1:7003
        - 127.0.0.1:7004
        - 127.0.0.1:7005
        - 127.0.0.1:7006
      max-redirects: 3
    lettuce:
      pool:
        max-active: 20
        max-idle: 10
        min-idle: 5

shop:
  cache:
    caffeine:
      max-size: 10000
      expire-after-write-minutes: 10
      refresh-after-write-minutes: 1
```

### 2.6.3 性能对比测试

同环境压测（JMeter，1000 并发，100 万请求）：

| 方案 | 平均响应 | P99 | QPS | DB 连接数 |
|---|---|---|---|---|
| 无缓存（直连 DB） | 180ms | 520ms | 5,200 | 200 |
| 仅 Redis | 8ms | 35ms | 42,000 | 15 |
| Redis + Caffeine | 3ms | 12ms | 55,000 | 5 |
| OpenResty + Redis + Caffeine | 1ms | 5ms | 82,000 | 3 |

**多级缓存 QPS 提升 15 倍+，响应时间降至 0.5%**。

---

## 2.6课程总结

- 多级缓存提升读性能，降低 DB 压力。
- OpenResty 适合作为边缘层缓存与路由。


# 3.Redis最佳实践

## 3.1课程介绍

- 关注 Key 设计、数据结构、批处理、性能配置。

## 3.2键值设计

### 3.2.1如何设计优雅的Key

- `业务:对象:主键:字段`。
- 统一前缀，便于管理。
- 尽量短、可读、可扫描。

**Key 命名规范示例**：

| 场景 | Key 示例 | 说明 |
|---|---|---|
| 用户信息 | `user:info:10001` | 用户基础信息 |
| 用户会话 | `user:session:a1b2c3d4` | 登录 session |
| 商品缓存 | `cache:shop:sku:1001` | 商品 SKU 缓存 |
| 计数器 | `counter:like:article:100` | 文章点赞数 |
| 排行榜 | `rank:score:daily:20240101` | 每日积分排行 |
| 分布式锁 | `lock:order:create:10086` | 订单创建锁 |
| 消息队列 | `queue:order:delay` | 延迟订单队列 |

---

### 3.2.2BigKey问题

- 大 key 影响内存与网络传输。
- 避免超大 Hash/List/ZSet。

处理方式：
- 拆分 key 或分页存储。
- 压缩或使用更合适的数据结构。

#### BigKey 检测命令

```bash
# 扫描大 key（建议在从节点执行）
redis-cli --bigkeys -h 127.0.0.1 -p 6379 -a redis123

# 检查单个 key 内存占用
redis-cli MEMORY USAGE user:sessions
# (integer) 4521984 -> 约 4.3MB

# 批量扫描
redis-cli --scan --pattern '*' | while read key; do
    size=$(redis-cli MEMORY USAGE "$key")
    if [ "$size" -gt 102400 ]; then
        echo "BigKey: $key => $size bytes"
    fi
done
```

#### BigKey 的危害

- **内存不均**：Cluster 模式下大 key 导致槽位数据倾斜。
- **阻塞风险**：删除大 key（DEL）是 O(N) 阻塞操作。
- **网络带宽**：GET 大 key 传输耗时长。
- **主从同步**：全量同步时可能导致从节点超时。
- **过期问题**：大 key 过期也触发阻塞（4.0+ lazyfree 缓解）。

#### BigKey 拆分策略

**策略 1：Hash 分片拆分**
```java
// 原方案：一个巨大 Hash
// HSET user:all-sessions token1 data1 token2 data2 ...

// 拆分为 100 个小 Hash 桶
public void addSession(String token, String data, String userId) {
    int bucket = Math.abs(userId.hashCode()) % 100;
    String key = "user:sessions:" + bucket;
    redisTemplate.opsForHash().put(key, token, data);
}
```

**策略 2：List 按日期拆分**
```java
public void pushToQueue(String item) {
    String date = LocalDate.now().format(DateTimeFormatter.BASIC_ISO_DATE);
    String key = "queue:pending:" + date;
    redisTemplate.opsForList().rightPush(key, item);
    redisTemplate.expire(key, Duration.ofDays(7));
}
```

**策略 3：安全删除大 key**
```java
// 使用 UNLINK 异步删除（Redis 4.0+）
redisTemplate.unlink("big_key");

// 分批删除 Hash
Cursor<Map.Entry<Object, Object>> cursor =
    redisTemplate.opsForHash().scan(key, ScanOptions.NONE);
while (cursor.hasNext()) {
    redisTemplate.opsForHash().delete(key, cursor.next().getKey());
}
```

```bash
# Shell 批量异步删除
redis-cli --scan --pattern 'user:sessions:*' | xargs -L 1 redis-cli UNLINK
```

#### Hot Key 检测与解决方案

**检测方法**：
- 客户端收集：Jedis/Lettuce 连接层统计 key 访问次数。
- Redis 自带：`redis-cli --hotkeys`（需 LFU 淘汰策略）。
- Proxy 层统计：Codis/Twemproxy 层。

**解决方案**：

| 方案 | 实现 | 优点 | 缺点 |
|---|---|---|---|
| 本地缓存 | Caffeine 缓存热点 key | 零网络开销 | 一致性需处理 |
| Key 拆分 | `hotkey:1` ~ `hotkey:N` 随机读 | 分散到多槽位 | 更新需同步 |
| 读写分离 | 增加从节点 | 无需改代码 | 复制延迟 |
| 前缀拆分 | `prefix:random:hotkey` | 分散请求 | 查询复杂度增加 |

```java
// Key 拆分 + 随机读取
public String getHotValue(String key) {
    int shardCount = 10;
    int shard = ThreadLocalRandom.current().nextInt(shardCount);
    String val = redisTemplate.opsForValue().get(key + ":" + shard);
    return val != null ? val : redisTemplate.opsForValue().get(key + ":0");
}
```

---

### 3.2.3选择合适的数据结构

- 计数：String。
- 对象：Hash。
- 排行榜：ZSet。
- 关注关系：Set。

#### 数据结构选型决策表

| 需求 | 推荐结构 | 示例 | 复杂度 |
|---|---|---|---|
| 简单 KV 缓存 | String | `SET shop:1 '{"name":"phone"}'` | O(1) |
| 对象存储 | Hash | `HSET user:1 name "alice"` | O(1) |
| 去重集合 | Set | `SADD tags:article:1 "redis"` | O(1) |
| 排行榜 | Sorted Set | `ZADD rank 100 player1` | O(log N) |
| 消息队列 | List | `LPUSH timeline:1 "event1"` | O(1) |
| 位图（签到） | Bitmap | `SETBIT sign:20240101 1001 1` | O(1) |
| 地理位置 | Geo | `GEOADD cities 116.4 39.9 beijing` | O(log N) |
| 概率统计 | HyperLogLog | `PFADD uv:page1 user1` | O(1) |
| 流数据 | Stream | `XADD mystream * f v` | O(1) |

**内存效率对比**（100 万条数据）：

| 结构 | 内存占用 | 说明 |
|---|---|---|
| String (JSON) | ~200MB | 每个 key 独立 |
| Hash (ziplist) | ~40MB | ziplist 压缩 |
| Hash (listpack) | ~35MB | Redis 7.0 默认 |

---

## 3.3批处理优化

### 3.3.1pipline和mset

- Pipeline 减少 RTT。
- `MSET/MGET` 批量操作。

#### Pipeline vs 逐条操作性能对比

```java
// 逐条操作（1000 次 RTT）
public void sequentialSet(List<KeyValue> kvList) {
    long start = System.currentTimeMillis();
    for (KeyValue kv : kvList) {
        redisTemplate.opsForValue().set(kv.key, kv.value);
    }
    // 典型: ~8000ms (本地), ~30000ms (跨机房)
}

// Pipeline 批量操作（1 次 RTT）
public void pipelineSet(List<KeyValue> kvList) {
    long start = System.currentTimeMillis();
    redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
        for (KeyValue kv : kvList) {
            connection.stringCommands().set(
                kv.key.getBytes(), kv.value.getBytes());
        }
        return null;
    });
    // 典型: ~50ms (本地), ~150ms (跨机房)
}
```

**性能对比数据**（1000 次 SET）：

| 方式 | 本地回环 | 同机房 | 跨机房(10ms RTT) |
|---|---|---|---|
| 逐条 | ~8000ms | ~10000ms | ~20000ms |
| Pipeline batch=100 | ~80ms | ~120ms | ~220ms |
| Pipeline batch=1000 | ~50ms | ~60ms | ~150ms |
| MSET (1000 key) | ~10ms | ~15ms | ~30ms |

#### Cluster Hash Tag 批量方案

```java
// 使用 Hash Tag 确保同用户 key 在同一槽位
String userId = "1001";
String nameKey = "user:{" + userId + "}:name";
String ageKey  = "user:{" + userId + "}:age";
String cityKey = "user:{" + userId + "}:city";

redisTemplate.opsForValue().multiSet(Map.of(
    nameKey, "Alice",
    ageKey, "25",
    cityKey, "Beijing"
));

// Lua 脚本原子操作
String script = "redis.call('MSET', KEYS[1],ARGV[1], KEYS[2],ARGV[2], KEYS[3],ARGV[3])";
redisTemplate.execute(
    new DefaultRedisScript<>(script, String.class),
    Arrays.asList(nameKey, ageKey, cityKey),
    "Alice", "25", "Beijing"
);
```

#### Lua 脚本批量处理

```lua
-- 批量设置并设置过期时间
local count = 0
for i = 1, #KEYS do
    local value = ARGV[i * 2 - 1]
    local ttl = tonumber(ARGV[i * 2])
    redis.call('SET', KEYS[i], value)
    if ttl > 0 then
        redis.call('EXPIRE', KEYS[i], ttl)
    end
    count = count + 1
end
return count
```

---

### 3.3.2集群模式下的批处理问题

- 不同槽位无法批量。
- 使用 hash tag：`user:{1}:name`。

#### 跨槽位批处理解决方案汇总

| 方案 | 原理 | 适用场景 |
|---|---|---|
| Hash Tag | `{}` 强制同槽位 | 逻辑关联的 key |
| 串行 Pipeline | 客户端按节点分组发 Pipeline | 跨槽位批量 |
| Lua + Hash Tag | 脚本操作同槽位多 key | 需要原子性 |
| JedisCluster 自动分组 | 框架自动处理 | 使用默认行为 |

---

## 3.4服务端优化

### 3.4.1持久化配置

- 大量写入场景可降低频率或关闭 AOF。
- 读多写少场景可开启 RDB。

#### 持久化配置场景推荐

| 场景 | RDB | AOF | 混合 | 备注 |
|---|---|---|---|---|
| 纯缓存（可丢失） | 关闭 | 关闭 | - | 不持久化 |
| 会话存储 | 开启 | 可选 | 可选 | save 300 1 |
| 计数器/统计 | 开启 | 关闭 | - | 可接受丢 5 分钟 |
| 核心业务 | 开启 | everysec | 推荐 | 混合持久化 |
| 高写入业务 | 降低 | no-appendfsync-on-rewrite yes | 推荐 | 从节点做持久化 |

---

### 3.4.2慢查询问题

- `slowlog-log-slower-than` 配置慢查询阈值。
- `SLOWLOG GET` 排查热点命令。

#### Slowlog 完整配置与分析

```bash
redis-cli CONFIG SET slowlog-log-slower-than 10000
redis-cli CONFIG SET slowlog-max-len 128
redis-cli CONFIG REWRITE

redis-cli SLOWLOG GET 10
# 示例输出:
# 1) (integer) 12345     # ID
# 2) (integer) 1700...   # 时间戳
# 3) (integer) 25300     # 耗时（微秒）
# 4) 1) "KEYS"           # 命令
#    2) "prefix:*"

redis-cli SLOWLOG LEN
redis-cli SLOWLOG RESET
```

**常见慢查询原因与解决**：

| 慢命令 | 原因 | 解决方案 |
|---|---|---|
| `KEYS *` | O(N) 全表扫描 | 改用 `SCAN` |
| `HGETALL big_hash` | 大 key 传输慢 | 改用 `HSCAN` 分批 |
| `ZREVRANGE key 0 -1` | 返回所有元素 | 分页 `0 99` |
| `DEL big_key` | O(N) 阻塞删除 | 使用 `UNLINK` |
| `SUNION` 大集合 | O(N) 复杂度 | 应用层合并 |

```conf
slowlog-log-slower-than 10000
slowlog-max-len 1024
latency-monitor-threshold 100
```

---

### 3.4.3命令及安全配置

- 关闭危险命令：`rename-command FLUSHALL ""`。
- 设置访问密码与 IP 白名单。

#### rename-command 安全加固

```conf
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command SHUTDOWN ""
rename-command DEBUG ""

# 保留给管理员但重命名
rename-command CONFIG "CONFIG_93462ac9"
rename-command KEYS    "KEYS_93462ac9"
```

（Redis 6.0+ 推荐使用 ACL 实现精细控制，详见第 4 章）

---

### 3.4.4内存安全和配置

- `maxmemory` + 淘汰策略。
- 监控 `used_memory` 与 key 数量。

#### maxmemory-policy 选型指南

| 淘汰策略 | 行为 | 适用场景 |
|---|---|---|
| `noeviction` | 拒绝写入 | 不想丢数据（谨慎） |
| `allkeys-lru` | 淘汰最近最少使用 | **通用缓存（推荐）** |
| `allkeys-lfu` | 淘汰最不频繁使用 | 热点场景 |
| `volatile-lru` | 有过期 key 中淘汰 LRU | 混合持久化 |
| `volatile-lfu` | 有过期 key 中淘汰 LFU | 缓存+持久化 |
| `volatile-ttl` | 淘汰最近过期 | TTL 优先清理 |
| `allkeys-random` | 随机淘汰 | 极少用 |

推荐：纯缓存用 `allkeys-lru`，混合持久化用 `volatile-lru`。

#### 完整性能调优 Checklist

```conf
# === 内存 ===
maxmemory 8gb
maxmemory-policy allkeys-lru
maxmemory-samples 10

# === 网络 ===
tcp-backlog 511
timeout 300
tcp-keepalive 60

# === 客户端 ===
maxclients 10000
client-output-buffer-limit replica 512mb 256mb 60

# === 持久化 ===
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes

# === AOF ===
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes

# === 慢查询 ===
slowlog-log-slower-than 10000
slowlog-max-len 1024
latency-monitor-threshold 100

# === 碎片整理 ===
activedefrag yes
active-defrag-ignore-bytes 100mb
active-defrag-threshold-lower 10
active-defrag-threshold-upper 100

# === lazy free ===
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes
replica-lazy-flush yes
lazyfree-lazy-user-del yes

# === 主从复制 ===
repl-backlog-size 128mb
repl-backlog-ttl 3600
repl-diskless-sync yes

# === IO 线程 (Redis 6.0+) ===
io-threads 4
io-threads-do-reads yes
```

---

## 3.5Redis优化-集群还是主从

- 读多写少、容量不大：主从即可。
- 数据量大、写入高：集群分片。

#### 容量规划指南

| 指标 | 主从 | 集群 |
|---|---|---|
| 数据量 | < 20GB | > 20GB |
| QPS（读） | < 10万 | > 10万 |
| QPS（写） | < 3万 | > 3万 |
| 可用性 | 哨兵自动切换 | 内置故障转移 |
| 运维复杂度 | 低 | 中 |
| 客户端兼容 | 所有客户端 | 需 Cluster 协议 |
| 扩展性 | 只扩读（加从节点） | 读写都可扩 |

**案例**：
- 某电商：100GB 商品缓存，QPS 15万 -> 6 节点集群（3 主 3 从）
- 某社交：5GB 用户 session，QPS 2万 -> 主从 + 哨兵
- 某金融：30GB 交易数据，QPS 8万 -> 集群 + 读写分离

---

# 4. Redis安全

## 4.1 ACL (Redis 6.0+)

Redis 6.0 引入了访问控制列表（ACL），支持多用户和权限精细管理。

#### ACL 用户创建与管理

```bash
# 创建只读用户
redis-cli ACL SETUSER readonly on >password123 ~* +@read -@write -@dangerous

# 创建管理员用户
redis-cli ACL SETUSER admin on >admin123 ~* +@all

# 创建受限用户（只能访问特定前缀 key）
redis-cli ACL SETUSER app_user on >app123 ~cache:* +@all ~*

# 查看所有用户
redis-cli ACL LIST

# 查看当前用户权限
redis-cli ACL WHOAMI

# 删除用户
redis-cli ACL DELUSER readonly

# 保存 ACL 到文件
redis-cli ACL SAVE
```

#### ACL 用户/角色体系

```
ACL 命令格式: ACL SETUSER <username> [rule [rule ...]]

规则类型:
  on/off           - 启用/禁用用户
  >password        - 设置密码
  ~<pattern>       - key 匹配模式（~* 表示所有 key）
  +<command>       - 允许命令（+@all 表示所有命令）
  -<command>       - 禁止命令（-@dangerous 表示危险命令组）
  +@<category>     - 允许命令组（+@read, +@write, +@admin 等）
  nopass           - 无密码登录
  resetpass        - 重置密码
```

#### ACL 配置文件

```conf
# users.acl
user default on >default123 ~* +@all

user readonly on >read123 ~* +@read -@write -@dangerous

user app_user on >app123 ~cache:* ~session:* +@all -FLUSHALL -FLUSHDB -SHUTDOWN -CONFIG -DEBUG

user admin on >admin123 ~* +@all +@admin
```

在 `redis.conf` 中启用：
```conf
aclfile /etc/redis/users.acl
```

#### 命令分类

| 类别 | 说明 |
|---|---|
| `@keyspace` | 对 key 类型操作的命令 |
| `@read` | 读取命令 |
| `@write` | 写入命令 |
| `@set` | 集合操作命令 |
| `@sortedset` | 有序集合操作 |
| `@list` | 列表操作 |
| `@hash` | 哈希操作 |
| `@string` | 字符串操作 |
| `@bitmap` | 位图操作 |
| `@hyperloglog` | 超对数操作 |
| `@geo` | 地理位置操作 |
| `@stream` | 流操作 |
| `@admin` | 管理命令 |
| `@dangerous` | 危险命令（FLUSHALL, KEYS 等） |
| `@slow` | 可能慢的命令 |
| `@fast` | 快速命令 |
| `@all` | 所有命令 |

```bash
# 查看命令分类
redis-cli ACL CAT
redis-cli ACL CAT dangerous
```

---

## 4.2 TLS/SSL 配置

Redis 6.0+ 支持 TLS 加密通信。

```conf
# redis.conf
tls-port 6380
port 0                                    # 关闭非 TLS 端口

tls-cert-file /etc/redis/tls/redis.crt
tls-key-file /etc/redis/tls/redis.key
tls-ca-cert-file /etc/redis/tls/ca.crt    # 双向认证

tls-auth-clients yes                      # 要求客户端提供证书
tls-auth-clients optional                 # 可选证书

tls-protocols "TLSv1.2 TLSv1.3"
tls-ciphers "HIGH:!aNULL:!eNULL:!MD5"
tls-prefer-server-ciphers yes
```

客户端连接：
```bash
redis-cli --tls --cert client.crt --key client.key --cacert ca.crt -p 6380
```

Spring Boot TLS 连接：
```java
@Bean
public LettuceConnectionFactory redisConnectionFactory() {
    RedisStandaloneConfiguration config = new RedisStandaloneConfiguration();
    config.setPort(6380);

    LettuceClientConfiguration clientConfig = LettuceClientConfiguration.builder()
        .useSsl()
        .build();

    return new LettuceConnectionFactory(config, clientConfig);
}
```

---

## 4.3 网络安全加固

```conf
# redis.conf
bind 127.0.0.1 192.168.1.100     # 只监听指定 IP，不监听 0.0.0.0
protected-mode yes                # 启用保护模式（无密码时只允许本地连接）
requirepass <strong-password>     # 设置密码
```

**防火墙规则**（iptables 示例）：
```bash
# 只允许内网访问 Redis
iptables -A INPUT -p tcp --dport 6379 -s 10.0.0.0/8 -j ACCEPT
iptables -A INPUT -p tcp --dport 6379 -j DROP
```

**云环境安全组**：
- 只开放 6379 端口给应用服务器 IP。
- Cluster BUS 端口（16379-16384）只开放给集群节点间。
- 哨兵端口（26379）只开放给哨兵节点和应用服务器。

---

# 5. Redis监控运维

## 5.1 INFO 命令关键指标详解

```bash
redis-cli INFO
```

#### Server 区块

| 指标 | 含义 |
|---|---|
| `redis_version` | Redis 版本 |
| `uptime_in_seconds` | 运行时长 |
| `uptime_in_days` | 运行天数 |

#### Clients 区块

| 指标 | 含义 | 告警阈值 |
|---|---|---|
| `connected_clients` | 当前连接数 | > maxclients * 0.8 |
| `blocked_clients` | 阻塞等待的客户端 | > 0 需关注 |

#### Memory 区块

| 指标 | 含义 | 告警阈值 |
|---|---|---|
| `used_memory` | 当前内存使用（字节） | > maxmemory * 0.8 |
| `used_memory_rss` | 操作系统视角的内存 | > used_memory * 1.5 |
| `mem_fragmentation_ratio` | 内存碎片率 | > 1.5 |
| `maxmemory` | 最大内存配置 | - |

#### Stats 区块

| 指标 | 含义 |
|---|---|
| `total_connections_received` | 累计连接数 |
| `total_commands_processed` | 累计命令数 |
| `instantaneous_ops_per_sec` | 当前 QPS |
| `keyspace_hits` | 命中次数 |
| `keyspace_misses` | 未命中次数 |
| `expired_keys` | 已过期 key 数 |
| `evicted_keys` | 被淘汰 key 数 |
| `latest_fork_usec` | 最近 fork 耗时（微秒） |

#### Replication 区块

| 指标 | 含义 |
|---|---|
| `role` | 角色（master/slave） |
| `connected_slaves` | 已连接从节点数 |
| `master_repl_offset` | 主节点复制偏移量 |
| `slave_repl_offset` | 从节点偏移量 |
| `master_link_status` | 复制链路状态 |

#### CPU 区块

| 指标 | 含义 |
|---|---|
| `used_cpu_sys` | 系统 CPU 时间 |
| `used_cpu_user` | 用户态 CPU 时间 |

---

## 5.2 Prometheus + Redis Exporter + Grafana

#### Docker Compose 部署

```yaml
version: '3.8'
services:
  redis-exporter:
    image: oliver006/redis_exporter:v1.55.0
    container_name: redis-exporter
    ports:
      - "9121:9121"
    command:
      - '--redis.addr=redis://redis:6379'
      - '--redis.password=redis123'
    networks:
      - monitor-net

  prometheus:
    image: prom/prometheus:v2.47.0
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    networks:
      - monitor-net

  grafana:
    image: grafana/grafana:10.1.0
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana
    networks:
      - monitor-net

networks:
  monitor-net:
    driver: bridge

volumes:
  grafana-data:
```

#### prometheus.yml

```yaml
scrape_configs:
  - job_name: 'redis_exporter'
    static_configs:
      - targets: ['redis-exporter:9121']
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        regex: '(.*):\d+'
        replacement: 'redis-${1}'
```

#### 导入 Grafana Dashboard

- 推荐 Dashboard ID：763（Redis Dashboard for Prometheus Redis Exporter）
- 进入 Grafana > Import > 输入 763 > 选择 Prometheus 数据源 > Import

---

## 5.3 关键告警规则

```yaml
# prometheus-alert-rules.yml
groups:
  - name: redis_alerts
    rules:
      - alert: RedisDown
        expr: redis_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Redis instance {{ $labels.instance }} down"

      - alert: RedisMemoryHigh
        expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis memory usage > 80%"

      - alert: RedisConnectionSpike
        expr: redis_connected_clients > 1000
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Redis connected clients > 1000"

      - alert: RedisReplicationLag
        expr: redis_master_repl_offset - redis_slave_repl_offset > 100000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis replication lag > 100KB"

      - alert: RedisEvictedKeys
        expr: rate(redis_evicted_keys_total[5m]) > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis evicting keys (maxmemory reached)"

      - alert: RedisRejectedConnections
        expr: rate(redis_rejected_connections_total[1m]) > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Redis rejecting connections"
```

---

## 5.4 redis-cli 运维工具

```bash
# 实时统计（每秒刷新）
redis-cli --stat

# 输出:------- data ------ --------------------- load -------------------- - child -
#      keys       mem      clients blocked requests            connections
#      1000       12.34M   5       0       5000 (+5000/s)      10

# 延迟直方图
redis-cli --latency
redis-cli --latency-history     # 持续记录
redis-cli --latency-dist        # 延迟分布

# 内在线延迟监控
redis-cli LATENCY LATEST
redis-cli LATENCY HISTORY command
redis-cli LATENCY DOCTOR
redis-cli LATENCY RESET

# 内存分析
redis-cli --bigkeys
redis-cli MEMORY STATS
redis-cli MEMORY DOCTOR
redis-cli MEMORY USAGE key
```

---

## 5.5 备份策略

#### crontab + bgsave 自动备份

```bash
# 添加 crontab 任务
# 每天凌晨 2 点执行 bgsave，并在 3 点复制到远程备份

# /etc/cron.d/redis-backup
0 2 * * * redis redis-cli -a password BGSAVE
0 3 * * * root rsync -avz /var/lib/redis/dump.rdb backup-server:/backup/redis/$(date +\%Y\%m\%d).rdb
0 4 * * * root find /backup/redis/ -name "*.rdb" -mtime +7 -delete
```

#### 备份脚本

```bash
#!/bin/bash
# redis_backup.sh

BACKUP_DIR="/backup/redis"
REDIS_DATA_DIR="/var/lib/redis"
REDIS_PASSWORD="redis123"
REMOTE_SERVER="backup@192.168.1.200:/backup/redis/"
DATE=$(date +%Y%m%d_%H%M%S)

# 1. 触发 bgsave
redis-cli -a "$REDIS_PASSWORD" BGSAVE

# 2. 等待 bgsave 完成
while true; do
    STATUS=$(redis-cli -a "$REDIS_PASSWORD" INFO persistence | grep rdb_bgsave_in_progress | cut -d: -f2)
    if [ "$STATUS" = "0" ]; then break; fi
    sleep 5
done

# 3. 本地备份
cp "$REDIS_DATA_DIR/dump.rdb" "$BACKUP_DIR/dump_$DATE.rdb"

# 4. 上传到远程
scp "$BACKUP_DIR/dump_$DATE.rdb" "$REMOTE_SERVER"

# 5. 清理 7 天前的备份
find "$BACKUP_DIR" -name "dump_*.rdb" -mtime +7 -delete

echo "[$DATE] Backup completed"
```

---

# 6. 性能压测

## 6.1 redis-benchmark 完整指南

```bash
# 基本语法
redis-benchmark [options]

# 完整参数示例：100 并发，100000 请求，只测 SET/GET
redis-benchmark -h 127.0.0.1 -p 6379 -a redis123 \
  -c 100 -n 100000 -d 1024 -t set,get \
  -P 16 --csv

# 参数说明：
# -h <host>         Redis 服务器地址
# -p <port>         端口
# -a <password>     密码
# -c <clients>      并发客户端数（默认 50）
# -n <requests>     总请求数（默认 100000）
# -d <size>         SET/GET 的数据大小（字节，默认 3）
# -t <tests>        测试类型（逗号分隔：set,get,incr,lpush,lpop,sadd,hset,等等）
# -P <numreq>       Pipeline 模式，每次发送 numreq 条命令
# -r <keyspacelen>  使用随机 key（指定 key 空间大小）
# --csv             CSV 格式输出
# --latency         显示延迟百分比
# --cluster         集群模式
```

#### 关键参数详解

| 参数 | 说明 | 典型值 |
|---|---|---|
| `-c` | 并发客户端数 | 50, 100, 500, 1000 |
| `-n` | 总请求数 | 100000, 1000000 |
| `-d` | 数据大小（字节） | 3, 256, 1024, 10240 |
| `-t` | 测试命令 | set,get,lpush,lpop,sadd,hset,zadd |
| `-P` | Pipeline 批量 | 1, 16, 50, 100 |
| `-r` | 随机 key（避免热点） | 100000, 1000000 |
| `-q` | 安静模式（只显示 QPS） | - |
| `--threads` | 多线程（Redis 6.0+） | 2, 4 |
| `--cluster` | 集群模式 | - |

#### 典型压测用例

```bash
# 1. 基础 SET 性能
redis-benchmark -h 127.0.0.1 -p 6379 -c 100 -n 1000000 -t set -q

# 2. SET/GET 混合测试（1024 字节数据）
redis-benchmark -h 127.0.0.1 -p 6379 -c 200 -n 500000 -d 1024 -t set,get -q

# 3. Pipeline 模式（每次 50 条）
redis-benchmark -h 127.0.0.1 -p 6379 -c 50 -n 500000 -P 50 -t set -q

# 4. 随机 key（100 万 key 空间），避免单 key 热点
redis-benchmark -h 127.0.0.1 -p 6379 -c 100 -n 1000000 -r 1000000 -t get -q

# 5. 集群模式压测
redis-benchmark -h 127.0.0.1 -p 7001 --cluster -c 200 -n 1000000 -t set,get -q

# 6. CSV 格式输出（便于分析）
redis-benchmark -h 127.0.0.1 -p 6379 -c 100 -n 100000 -t set,get,lpush,lpop --csv
```

#### 解读压测结果

```
====== SET ======
  100000 requests completed in 0.72 seconds
  100 parallel clients
  3 bytes payload
  keep alive: 1

Summary:
  throughput summary: 138888.89 requests per second
  latency summary (msec):
      avg       min       p50       p95       p99       max
     0.712     0.104     0.679     1.111     3.199     5.439
```

- **throughput**：QPS，越高越好。
- **avg latency**：平均延迟，P95/P99 更重要。
- **P99 latency**：99% 的请求在这个延迟内完成。P99 越低越好。

## 6.2 memtier_benchmark

memtier_benchmark 是 Redis Labs 开发的压测工具，比 redis-benchmark 更强大：

```bash
# 安装
# Ubuntu: apt install memtier-benchmark
# CentOS: yum install memtier_benchmark

# 基本用法
memtier_benchmark -s 127.0.0.1 -p 6379 -a redis123 \
  --clients=100 --threads=4 \
  --ratio=1:10 \           # SET:GET 比例 1:10
  --data-size=256 \         # 数据大小 256 字节
  --key-pattern=R:R \       # 随机 key + 随机 value
  --test-time=60 \          # 测试 60 秒
  --pipeline=8              # Pipeline 批量 8

# 参数说明：
# --clients          并发客户端数
# --threads          测试线程数
# --ratio            SET:GET 比例
# --data-size        值大小
# --key-pattern      P:P（固定）, R:R（随机）, S:S（顺序）
# --key-maximum      key 空间最大值
# --test-time        测试时长（秒）
# --pipeline         Pipeline 批量
# --hide-histogram   隐藏延迟直方图
```

**与 redis-benchmark 对比**：

| 特性 | redis-benchmark | memtier_benchmark |
|---|---|---|
| 内置 | 是（Redis 自带） | 需单独安装 |
| 多线程 | Redis 6.0+ 支持 | 原生多线程 |
| 读写混合 | 不支持混合 | 支持 SET:GET 比例 |
| 延迟直方图 | 基本支持 | 详细直方图 |
| 集群模式 | 支持 | 不支持 |
| 自定义命令 | 有限 | 不支持 |

---

# 7. 数据迁移

## 7.1 redis-shake

redis-shake 是阿里巴巴开源的 Redis 数据迁移工具，支持全量 + 增量同步。

```bash
# 下载
wget https://github.com/alibaba/RedisShake/releases/latest/download/redis-shake.tar.gz
tar -xzf redis-shake.tar.gz

# 配置文件 shake.toml
# 从单机迁移到集群
[source]
address = "192.168.1.100:6379"
password = "password123"
type = "standalone"

[target]
address = "192.168.1.200:7001"
password = "password123"
type = "cluster"

[advanced]
dir = "data"
ncpu = 4
psync = true   # 使用 PSYNC 进行增量同步
```

```bash
# 运行
./redis-shake shake.toml

# 查看进度
tail -f logs/redis-shake.log
```

**redis-shake 功能特性**：
- 全量数据迁移（RDB）
- 增量数据同步（PSYNC）
- 支持跨版本迁移（3.x -> 7.x）
- 支持单机到集群、集群到集群
- 数据过滤和转换

## 7.2 MIGRATE 命令（单 Key 迁移）

```bash
# 将 key 从当前节点迁移到目标节点
redis-cli MIGRATE 192.168.1.200 6379 "user:1001" 0 5000
# MIGRATE <host> <port> <key> <db> <timeout_ms>

# 迁移多个 key
redis-cli MIGRATE 192.168.1.200 6379 "" 0 5000 KEYS user:1001 user:1002 order:200

# COPY 模式（不删除源端 key）
redis-cli MIGRATE 192.168.1.200 6379 "user:1001" 0 5000 COPY

# REPLACE 模式（覆盖目标端已有 key）
redis-cli MIGRATE 192.168.1.200 6379 "user:1001" 0 5000 REPLACE
```

#### 批量迁移脚本

```bash
#!/bin/bash
# 批量迁移匹配模式的所有 key
PATTERN="user:*"
TARGET_HOST="192.168.1.200"
TARGET_PORT="6379"
TIMEOUT="5000"

redis-cli --scan --pattern "$PATTERN" | while read key; do
    redis-cli MIGRATE "$TARGET_HOST" "$TARGET_PORT" "$key" 0 "$TIMEOUT"
    echo "Migrated: $key"
done
```

## 7.3 Redis Dump + Restore 流程

```bash
# 源端：导出 RDB
redis-cli -h source-host -p 6379 -a password BGSAVE
# 等待完成
redis-cli -h source-host -p 6379 -a password INFO persistence | grep rdb_bgsave_in_progress

# 传输 RDB 文件
scp /var/lib/redis/dump.rdb target-host:/var/lib/redis/dump.rdb

# 目标端：停止 Redis，放置 RDB，重启
ssh target-host "systemctl stop redis"
ssh target-host "chown redis:redis /var/lib/redis/dump.rdb"
ssh target-host "systemctl start redis"

# 验证
redis-cli -h target-host -p 6379 DBSIZE
```

## 7.4 迁移检查清单与回滚计划

#### 迁移前检查

```
[x] 源端和目标端 Redis 版本兼容性确认
[x] 目标端内存 >= 源端 used_memory
[x] 目标端 maxmemory-policy 确认
[x] 网络带宽评估（RDB 传输时间）
[x] 目标端持久化配置
[x] 业务方通知（停机窗口或灰度）
[x] 回滚方案确认
[x] 监控和告警已就绪
```

#### 回滚计划

```
[x] 迁移前备份源端 RDB（cp dump.rdb backup.rdb）
[x] 记录源端配置（redis-cli CONFIG GET '*' > config_backup.txt）
[x] 切换流量到目标端
[x] 验证目标端数据完整性
[x] 如发现问题：切回源端流量
[x] 排除问题后重新迁移
```

---

# 8. Redis on Kubernetes

## 8.1 Helm Chart 部署

```bash
# 添加 Bitnami Helm 仓库
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# 部署 Redis 主从
helm install redis bitnami/redis \
  --set architecture=replication \
  --set auth.password=redis123 \
  --set master.persistence.enabled=true \
  --set master.persistence.size=8Gi \
  --set replica.replicaCount=2 \
  --set replica.persistence.enabled=true \
  --set replica.persistence.size=8Gi \
  --namespace redis --create-namespace

# 部署 Redis Cluster
helm install redis-cluster bitnami/redis-cluster \
  --set global.redis.password=redis123 \
  --set cluster.nodes=6 \
  --set cluster.replicas=1 \
  --set persistence.size=10Gi \
  --namespace redis --create-namespace
```

#### 自定义 values.yaml

```yaml
architecture: replication

auth:
  password: redis123

master:
  persistence:
    enabled: true
    size: 8Gi
    storageClass: "fast-ssd"
  resources:
    requests:
      memory: "2Gi"
      cpu: "500m"
    limits:
      memory: "4Gi"
      cpu: "2000m"

replica:
  replicaCount: 2
  persistence:
    enabled: true
    size: 8Gi
    storageClass: "fast-ssd"

metrics:
  enabled: true
  serviceMonitor:
    enabled: true  # Prometheus Operator
```

## 8.2 Redis Operator 概述

Redis Operator（如 Ot-Consulting Redis Operator）提供了 K8s 原生的 Redis 管理能力：

```yaml
apiVersion: redis.redis.opstreelabs.in/v1beta2
kind: Redis
metadata:
  name: redis-cluster
spec:
  kubernetesConfig:
    image: quay.io/opstree/redis:v7.0.5
    imagePullPolicy: IfNotPresent
  redisExporter:
    enabled: true
    image: quay.io/opstree/redis-exporter:v1.44.0
  storage:
    volumeClaimTemplate:
      spec:
        storageClassName: fast-ssd
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
  redisConfig:
    additionalRedisConfig: redis-additional-config
```

## 8.3 PVC 和持久化配置

#### StorageClass（SSD 推荐）

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/gce-pd   # 替换为你的云厂商
parameters:
  type: pd-ssd
reclaimPolicy: Retain               # 保留 PVC，防止误删
volumeBindingMode: WaitForFirstConsumer
```

#### 持久化注意事项

1. RDB 和 AOF 数据必须持久化到 PVC。
2. Redis Cluster 的 nodes.conf 也需要持久化。
3. Pod 重启时不要清空数据目录。
4. 使用 StatefulSet 而非 Deployment 部署 Redis（保持稳定的网络标识）。
5. PVC reclaimPolicy 设为 Retain。

## 8.4 K8s 常见问题

| 问题 | 原因 | 解决方案 |
|---|---|---|
| Pod 频繁重启 | OOMKilled | 增大 resources.limits.memory |
| 主从切换后连接失败 | IP 变化 | 使用 Headless Service 代替 IP |
| 数据丢失 | PVC 被删除 | reclaimPolicy: Retain |
| Cluster 节点失联 | DNS 解析失败 | 确保 StatefulSet 网络策略正确 |
| 内存不足 | maxmemory 设置过大 | 设置合理 maxmemory，预留 30% 给系统 |

#### Redis on K8s 最佳实践（注意）

- use StatefulSet, not Deployment
- use Anti-Affinity 保证节点分布在不同物理机
- use Init Container 检查 Redis 就绪状态
- use PodDisruptionBudget 防止并发删除
- use Readiness Probe 确保服务可用后再接入流量
- 关闭 THP：使用 initContainer 执行 `echo never > /sys/kernel/mm/transparent_hugepage/enabled`
- 设置 `vm.overcommit_memory=1`：同 initContainer

