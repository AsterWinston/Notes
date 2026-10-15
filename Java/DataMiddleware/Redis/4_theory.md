# 1. 课程介绍

- 本章聚焦 Redis 内部数据结构、网络模型、通信协议与内存回收。
- 深入源码级别理解 Redis 如何做到高性能、低内存，涵盖从数据结构选型到网络 I/O 模型的完整链路。
- 最终目标：能根据业务场景选择合适的数据结构编码，理解配置参数背后的原理，具备排查 Redis 性能问题的理论基础。

# 2. Redis数据结构

## 2.1 动态字符串

- SDS（Simple Dynamic String）。
- 记录 length 和 free，避免反复 `strlen`。
- 支持预分配与惰性释放。

### 2.1.1 SDS 结构布局

```
+--------+--------+--------+--------+-------------------+
| flags  | len    | alloc  | buf[]  | (未使用空间)       |
+--------+--------+--------+--------+-------------------+
  1B       1~8B     1~8B    变长        alloc - len
```

- `flags`：低 3 bit 存储类型（sdshdr5/8/16/32/64），决定 len/alloc 字段宽度。
- `len`：已使用的字节数（不含 `\0`）。
- `alloc`：已分配的总字节数（不含 `\0` 和 header）。
- `buf[]`：实际字符数据，末尾必定有 `\0` 兼容 C 字符串函数。

### 2.1.2 SDS 与 C 字符串对比

| 特性 | C 字符串 | SDS |
|------|----------|-----|
| 获取长度 | O(N)，反复 `strlen` | O(1)，读 `len` 字段 |
| 缓冲区溢出 | 需手动检查，易溢出 | 自动扩容，防溢出 |
| 内存分配 | 每次修改都需分配 | 预分配 + 惰性释放 |
| 二进制安全 | 遇 `\0` 截断 | 通过 `len` 判断长度，安全 |
| 函数兼容 | C 标准库 | 部分兼容 C 字符串函数 |

### 2.1.3 空间预分配策略

- 当新 len < 1MB：分配 `2 * newLen` 空间，即扩容后预留 100% 冗余。
- 当新 len >= 1MB：分配 `newLen + 1MB`，即额外预留 1MB，避免大字符串翻倍浪费。
- 预分配减少 `malloc/realloc` 调用次数，在频繁 append 场景（如日志追加、计数器）效果显著。

### 2.1.4 惰性空间释放

- 字符串缩短时，SDS 不会立即 `realloc` 缩小内存，只更新 `len` 并修改 `free`（内部记录的未使用空间）。
- 后续 append 操作可直接复用空闲空间，避免重新分配。
- 若确实需要释放，可调用 `sdsRemoveFreeSpace()` 手动收缩。

### 2.1.5 二进制安全

- C 字符串以 `\0` 作为结束标记，因此无法存储包含 `\0` 的二进制数据（如图片、序列化对象）。
- SDS 以 `len` 字段标记数据长度，不以 `\0` 为终止条件，可安全存储任意二进制数据。
- 示例：存储 HTTP body 中的图片二进制流、Protobuf/MessagePack 序列化后的字节数组均无问题。

## 2.2 intset

- 整数集合，底层连续数组。
- 自动升级（int16 -> int32 -> int64）。
- 用于小集合的 Set。

### 2.2.1 IntSet 结构布局

```
+----------+----------+-------------------+
| encoding | length   | contents[]        |
+----------+----------+-------------------+
  2~8B       2~8B       length * sizeof(element)
```

- `encoding`：取值 `INTSET_ENC_INT16(2)` / `INTSET_ENC_INT32(4)` / `INTSET_ENC_INT64(8)`，表示每个元素占用的字节数。
- `length`：集合中元素个数。
- `contents[]`：按编码宽度存储的有序整数数组，无重复，升序排列。

### 2.2.2 升级（Upgrade）过程详解

当新插入的整数值超出当前编码范围时触发升级：

1. **检测**：判断新值是否能用当前 encoding 表示（如 int16 范围 -32768~32767）。
2. **扩容**：根据新 encoding 计算所需总空间，`realloc` 重新分配 `contents`。
3. **倒序迁移**：从最后一个元素开始，向前逐个将旧元素按新编码宽度写入新位置。倒序避免覆盖未迁移数据。
4. **插入新值**：将触发升级的新元素写入合适位置（保持有序）。
5. **更新 encoding**：设置为新编码值。

示例：原 int16 集合 `[1, 3, 5]`，插入 65536（超出 int16）时升级为 int32。

### 2.2.3 为什么 IntSet 不支持降级

- 降级场景极少见：通常元素被删除后集合规模自然缩小，不会从 int64 回到 int16。
- 实现复杂度：降级需要扫描全集合判断最大值，收益低。
- 若确实需要节省内存，建议业务层重新创建集合插入元素。

### 2.2.4 查找算法

- `contents[]` 有序排列，采用**二分查找**定位元素。
- `intsetSearch()` 函数：`while (min <= max) { mid = (min+max)/2; ... }`。
- 时间复杂度 O(log N)。

### 2.2.5 Set 底层编码切换规则

- 满足以下两条件使用 `intset`：
  1. 所有元素均为整数。
  2. 元素个数 <= `set-max-intset-entries`（默认 512）。
- 任意条件不满足时，转换为 `hashtable`（Dict）。

## 2.3 Dict

- Hash 表，数组 + 链表/结构体。
- 采用渐进式 rehash。

### 2.3.1 Dict 结构层级

```
dict
  ├── dictType (类型方法表，含 hashFunction、keyCompare 等)
  ├── privdata (私有数据)
  ├── dictht ht[2] (两个 Hash 表，一个主用，一个 rehash 用)
  │     ├── table (dictEntry** 数组，即 buckets)
  │     ├── size (bucket 数量，恒为 2^n)
  │     ├── sizemask (size - 1，用于取模)
  │     └── used (已存储元素数量)
  └── rehashidx (当前 rehash 进度，-1 表示未进行)
```

- `dictEntry`：包含 key 指针、union value（可存 ptr/uint64_t/int64_t/double）、next 指针（拉链法）。

### 2.3.2 哈希算法

- Redis 使用 **SipHash**（Redis 5.0+），代替早期的 MurmurHash2。
- SipHash 优点：抗 hash-flooding 攻击（HashDoS），避免碰撞退化为 O(N)。
- 对 key 取哈希后 `hash & sizemask` 即得到 bucket 索引。

### 2.3.3 哈希冲突解决：分离链接法（Separate Chaining）

- 同一 bucket 下多个 `dictEntry` 通过 `next` 指针串成单向链表。
- **头插法**：新 entry 总是插入链表头部（O(1)），无需遍历尾节点。

## 2.4 Dict的渐进式rehash

- 同时维护旧表与新表。
- 每次读写迁移一部分，避免阻塞。

### 2.4.1 Rehash 触发条件

Rehash 由 `_dictExpandIfNeeded()` 检查触发：

- **扩展（扩容）** - 满足任一即触发：
  - 负载因子 >= 1 且当前无子进程 `bgsave`/`bgrewriteaof`（`dict_can_resize = 1`）。
  - 负载因子 > 5，**强制**扩容（即使有子进程），避免 bucket 链过长影响性能。
- **收缩**：负载因子 < 0.1 时，缩容为 `used` 的 2 次幂。

负载因子 = `ht[0].used / ht[0].size`。

### 2.4.2 渐进式 Rehash 完整流程

```
初始状态:
  ht[0].table = [...buckets...]   (旧表, size=4, used=3)
  ht[1].table = NULL
  rehashidx = -1

Step 1 - 触发扩容:
  分配 ht[1].table (size=8, used=0)
  rehashidx = 0

Step 2 - 渐进迁移 (每次操作触发):
  遍历 ht[0].table[rehashidx] 的链表
  将每个 entry 重新哈希 → 迁移到 ht[1]
  每迁移完一个 bucket，rehashidx++

Step 3 - 完成:
  当 rehashidx == ht[0].size
  释放 ht[0].table
  ht[1] 替换为 ht[0]
  rehashidx = -1

ASCII 示意:
  访问请求: set k v
     ├── 先迁移 1 个 bucket (rehashidx++)
     ├── 然后再执行 set 命令 (查 ht[0] 和 ht[1] 两个表)
     └── 返回结果
```

### 2.4.3 单步 Rehash 步数

- 每次对 Dict 的增/删/改/查操作中，调用 `_dictRehashStep(d)` 迁移 **1 个 bucket**（即一个槽位及其整个链表）。
- 每个 bucket 可能包含多个 entry（链表），一次迁移整条链，避免一条链被拆分。
- 定时任务（`serverCron`）中，每次执行 1ms 的批量 rehash，迁移多个 bucket。

### 2.4.4 rehashidx 字段

- `-1`：未进行 rehash。
- `>= 0`：当前正在迁移的 bucket 索引（ht[0].table 的下标）。

### 2.4.5 Rehash 期间的读写行为

- **查询**：先在 `ht[0]` 查找，未找到再到 `ht[1]` 查找。
- **新增**：直接写入 `ht[1]`，避免向旧表写入新数据。
- **删除**：先在 `ht[0]` 查找并删除，再在 `ht[1]` 查找并删除。
- **更新**：先在 `ht[0]` 查，找到则删除 + 往 `ht[1]` 新增；未找到则在 `ht[1]` 查并更新。

## 2.5 ZipList

- 压缩列表，连续内存存储。
- 适合小数据量的 Hash/List/ZSet。

### 2.5.1 ZipList 整体结构（字节级布局）

```
+--------+--------+--------+--------+--------+-----+--------+--------+
| zlbytes| zltail | zllen  | entry1 | entry2 | ... | entryN | zlend  |
+--------+--------+--------+--------+--------+-----+--------+--------+
   4B       4B       2B      变长     变长            变长      1B(0xFF)
```

- `zlbytes`（4B）：整个 ziplist 占用的字节数（含自身）。
- `zltail`（4B）：最后一个 entry 的偏移量，用于 O(1) 双向遍历。
- `zllen`（2B）：entry 数量。若 > 65535，需遍历才知真实数量。
- `zlend`（1B）：固定为 `0xFF`，结束标记。

### 2.5.2 Entry 编码格式

每个 entry 包含：`prevlen` + `encoding` + `entry-data`

**prevlen（前一个 entry 的长度）**：
- 前驱 < 254 字节：1 字节直接存长度。
- 前驱 >= 254 字节：1 字节 `0xFE` + 4 字节存长度（共 5 字节）。

**encoding + entry-data**：

| encoding 首字节 | 含义 | 示例 |
|----------------|------|------|
| `00pppppp` | 长度 ≤ 63 的短字符串 | "hello" |
| `01pppppp` `qqqqqqqq` | 长度 ≤ 16383 的中等字符串 | 较长文本 |
| `10000000` `xxxxxxxx` | 长度 ≤ 4294967295 的长字符串（5 字节 header） | 超长文本 |
| `11000000` | int16_t（2 字节） | 127、-1 |
| `11010000` | int32_t（4 字节） | 32768 |
| `11100000` | int64_t（8 字节） | 9999999999 |
| `11110000` | 24 位有符号整数（3 字节） | 8388607 |
| `11111110` | 8 位有符号整数（1 字节） | -1~127 |
| `1111xxxx` | 0~12 的小整数（4 位直接存于 encoding 中） | 0～12，不占额外空间 |

### 2.5.3 为什么 ZipList 节省内存

- 连续内存分配，无指针开销（相比于链表/SkipList 中的大量指针）。
- 小整数直接编码在 `encoding` 字段中（如 0~12 不占额外空间）。
- 没有对齐填充，紧凑排列。

### 2.5.4 典型使用阈值

- `hash-max-ziplist-entries`：默认 512。
- `zset-max-ziplist-entries`：默认 128。
- `list-max-ziplist-size`：定义单个 quicklist 节点中 ziplist 的最大占用字节数。
- 超过阈值或元素长度超过 `*-max-ziplist-value`（默认 64 字节），转换为其他编码。

## 2.6 ZipList的连锁更新问题

- 元素长度变化导致后续节点元数据变化。
- 可能造成多次拷贝，已由 ListPack 替代。

### 2.6.1 连锁更新执行过程示例

```
初始状态 (每个 entry 长度均为 253B，所以 prevlen 占 1B):
  [zlbytes][zltail][zllen][E1: prevlen=1B, 253B data][E2: same][E3: same][zlend]

Step 1: 在 E1 之前插入一个长度 > 254B 的新条目 E0
  E1 的 prevlen 需要从 1B → 5B (因为 E0 > 254B)
  E1 总长度从 254B → 258B (> 254B)

Step 2: E1 长度由 254B → 258B 后,
  E2 的 prevlen 也需要从 1B → 5B (因为 E1 现在 > 254B)
  E2 总长度从 254B → 258B (> 254B)

Step 3: 同理, E3 的 prevlen 也需 1B → 5B...

... 连锁反应，每个后续 entry 都需要 prevlen 扩容并连带自身总长度变化，
    触发一连串内存重分配和数据拷贝
```

### 2.6.2 问题严重性

- 最坏情况下，一个插入操作导致 O(N²) 的内存重分配 + 数据拷贝（每个 entry 都要重新分配）。
- N 为几百时影响可控；N 达数万时造成明显阻塞。
- 该 Bug 在 ZipList 设计层面无法彻底修复，因此 Redis 作者开发了 **ListPack** 替代。

### 2.6.3 ListPack 解决方案（关键区别）

- ListPack 的 entry 不存储 `prevlen`，而是存储 `entry-tot-len`（当前 entry 自身总长度）。
- 反向遍历时，通过读取末尾的 `entry-tot-len` 跳转到前一个 entry，消除了对前驱条目长度的依赖。
- 因此 ListPack 不存在连锁更新问题。

## 2.7 ListPack（Redis 5.0 引入，7.0 全面替代 ZipList）

### 2.7.1 ListPack 结构布局

```
+--------+--------+--------+-----+--------+--------+
| lpbytes| lplen  | entry1 | ... | entryN | lpend  |
+--------+--------+--------+-----+--------+--------+
   4B       2B      变长             变长      1B(0xFF)
```

- `lpbytes`（4B）：listpack 总字节数（含自身）。
- `lplen`（2B）：元素个数，>65535 时需遍历。
- `lpend`（1B）：固定 `0xFF`。

### 2.7.2 ListPack Entry 编码

```
+------------+----------+-------------------+
| encoding   | data     | backlen(entry总长) |
+------------+----------+-------------------+
  变长(1~?)    变长        1~5B (位于 entry 末尾)
```

- `backlen` 记录**本 entry 的总长度**，允许反向遍历。
- `backlen` 自身 1~5 字节，最高 bit 作为连续标记：`0xxxxxxxx`=结束，`1xxxxxxxx`=继续读下个字节。
- **关键差异**：不依赖前一个 entry 的长度 → 从根本上消除了连锁更新。

### 2.7.3 ListPack 与 ZipList 对比

| 特性 | ZipList | ListPack |
|------|---------|----------|
| 反向遍历 | prevlen 存前驱长度 | backlen 存自身长度 |
| 连锁更新 | 存在 O(N²) 风险 | 消除，无依赖 |
| 存储开销 | 无 backlen | 多出 backlen（1~5B） |
| Redis 版本 | 旧版默认 | 5.0 引入，7.0 成为 Stream/Set 等内部实现 |

### 2.7.4 Redis 7.0+ 中 ListPack 替代范围

- Hash 小数据编码：`listpack` 替代 `ziplist`。
- ZSet 小数据编码：`listpack` 替代 `ziplist`。
- Set 小数据编码：`listpack` 替代 `ziplist`（Redis 7.2+）。
- Stream 内部消息存储：`listpack` 替代 `radix_tree + listpack`。

## 2.8 QuickList

- 链表 + ZipList 组合。
- 兼顾插入效率与内存使用。

### 2.8.1 QuickList 结构

```
quicklist
  ├── head → quicklistNode → quicklistNode → ... → tail
  ├── count (总元素数)
  ├── len (节点数)
  ├── fill (每个 ziplist 最大占用)
  └── compress (压缩深度)

quicklistNode
  ├── prev / next (双向链表指针)
  ├── zl (指向内部的 ziplist / listpack)
  ├── sz (ziplist 字节数)
  ├── count (节点内元素数)
  ├── encoding (RAW=1 或 LZF=2)
  ├── container (PLAIN=1 或 PACKED=2)
  └── recompress (lazy 压缩标记)
```

### 2.8.2 fill 参数详解

- `list-max-ziplist-size`（fill 参数）：
  - **正值**（如 5, 10）：表示每个 ziplist/listpack 节点最多容纳的元素个数。
  - **负值**：按字节限制，具体含义：
    - `-1`：每个节点最大 4KB
    - `-2`：每个节点最大 8KB（默认）
    - `-3`：每个节点最大 16KB
    - `-4`：每个节点最大 32KB
    - `-5`：每个节点最大 64KB

### 2.8.3 compress 参数

- `list-compress-depth`：
  - `0`：不压缩（默认）。
  - `1`：仅压缩两端各 1 个节点（头和尾）。
  - `N`：压缩两端各 N 个节点，中间节点不压缩。
  - 原理：列表大部分访问集中在头尾（队列/栈操作），两端 LZF 解压缩开销小，中间节点一般不访问。
- LZF 是无损压缩算法，对文本类数据压缩率高，但 CPU 开销增加。

### 2.8.4 不同场景的配置建议

- **消息队列**（大量头尾操作）：`list-compress-depth 1` 开启两端压缩。
- **随机访问列表**：关闭压缩或 `list-compress-depth 0`。
- **小对象列表**：`list-max-ziplist-size -2`（8KB），平衡内存与性能。
- **大对象列表**：`list-max-ziplist-size -1`（4KB），让每个节点较少元素以提升插入效率。

## 2.9 SkipList

- 用于 ZSet。
- 多层索引，范围查询性能好。

### 2.9.1 SkipList 结构示意（ASCII Art）

```
Level 3:  HEAD ──────────────────────> 30 ──────────────> NULL
Level 2:  HEAD ──────────> 10 ───────> 30 ───────> 40 ──> NULL
Level 1:  HEAD ──> 5 ────> 10 ──> 15 ─> 30 ──> 35 ─> 40 ─> 45 ──> NULL
Level 0:  HEAD -> 5 -> 7 -> 10 -> 15 -> 30 -> 35 -> 40 -> 45 -> 50 -> NULL
          (底层完整双向链表)

zskiplist { header, tail, length, level(max) }
zskiplistNode { sds ele, double score, zskiplistNode *backward,
                zskiplistLevel { zskiplistNode *forward; unsigned long span; } level[] }
```

### 2.9.2 插入算法

1. 从最高层开始，逐层找到每一层的插入位置（记录前驱节点和 span）。
2. 用随机算法确定新节点的层数：默认为 `ZSKIPLIST_P = 0.25`，即每次上升一层的概率为 25%。
3. 在各层插入新节点，更新 forward 指针和 backward 指针。
4. 更新受影响节点的 span（跨度值，用于计算 rank）。

### 2.9.3 为什么 Redis 用 SkipList 而非平衡树

- **实现简单**：相比 AVL/红黑树，SkipList 插入删除仅需修改相邻节点指针，代码量小。
- **范围查询天然高效**：找到起点后沿着 level[0] 前后遍历即可，O(log N) 定位 + O(M) 遍历。
- **无再平衡**：不涉及复杂的旋转操作。

### 2.9.4 ZSet 的双结构：Dict + SkipList

```
ZSet 同时维护:
  ┌── Dict:  {member → score}     用于 O(1) 单点查询/更新 score
  └── SkipList: {score, member}   用于 O(log N) 范围查询/排序/rank

两者通过 member 共享同一份 SDS 字符串（指针共享），不额外占用内存。
```

### 2.9.5 复杂度一览

| 操作 | 时间复杂度 |
|------|-----------|
| 查找（按 member）| O(1) via Dict |
| 查找（按 score）| O(log N) |
| 范围查询（ZRANGE）| O(log N + M) |
| 插入/删除 | O(log N) |
| 计算 rank | O(log N) |

## 2.10 RedisObject

- 统一对象结构：类型 + 编码 + LRU/引用计数 + 指针。
- 同一类型可有不同编码（如 String: int/raw）。

### 2.10.1 RedisObject 结构体全字段解析

```c
typedef struct redisObject {
    unsigned type:4;       // 数据类型 (4 bits)
    unsigned encoding:4;   // 底层编码 (4 bits)
    unsigned lru:LRU_BITS; // LRU 时间 / LFU 计数器 (24 bits, 默认)
    int refcount;          // 引用计数 (4 bytes)
    void *ptr;             // 指向实际数据的指针 (8 bytes)
} robj;                    // 总共 16 bytes (64-bit 系统)
```

### 2.10.2 type 字段取值

| type 值 | 宏 | 对应命令 |
|---------|----|----------|
| 0 | OBJ_STRING | SET, GET |
| 1 | OBJ_LIST | LPUSH, RPUSH |
| 2 | OBJ_SET | SADD |
| 3 | OBJ_ZSET | ZADD |
| 4 | OBJ_HASH | HSET |
| 5 | OBJ_MODULE | MODULE LOAD |
| 6 | OBJ_STREAM | XADD |

### 2.10.3 encoding 字段取值（按类型）

| 类型 | 编码值 | 宏 | 使用场景 |
|------|--------|----|----------|
| String | 0 | OBJ_ENCODING_RAW | raw SDS，长度 > 44 |
| String | 1 | OBJ_ENCODING_INT | 整数值嵌入 ptr |
| String | 2 | OBJ_ENCODING_EMBSTR | 嵌入式 SDS，长度 ≤ 44 |
| List | 1 | OBJ_ENCODING_QUICKLIST | 默认 |
| Set | 2 | OBJ_ENCODING_INTSET | 纯小整数 |
| Set | 0 | OBJ_ENCODING_HT | 较大或非整数 |
| ZSet | 1 | OBJ_ENCODING_ZIPLIST/LISTPACK | 小数据 |
| ZSet | 2 | OBJ_ENCODING_SKIPLIST | 大数据 |
| Hash | 1 | OBJ_ENCODING_ZIPLIST/LISTPACK | 小数据 |
| Hash | 0 | OBJ_ENCODING_HT | 大数据 |

### 2.10.4 LRU/LFU 字段

- `LRU_BITS = 24`（3 字节），存储时间戳（秒级）或 LFU 计数（高 16 bit 存分钟的 timestamp，低 8 bit 存频率 counter）。
- 用于 `maxmemory-policy` 的内存淘汰决策。

### 2.10.5 refcount 共享对象

- Redis 服务启动时创建 0~9999 的整数共享对象。
- 当存储的整数值在 0~9999 范围时，`robj->ptr` 直接指向共享对象，`refcount` 引用计数 ++。
- `OBJECT REFCOUNT key` 可查看该 key 的引用计数（共享对象 > 1，非共享对象 = 1）。

### 2.10.6 OBJECT 命令揭示内部编码

```bash
> SET key1 123
> OBJECT ENCODING key1
"int"

> SET key2 "hello"
> OBJECT ENCODING key2
"embstr"

> SET key3 "a very long string that exceeds 44 bytes threshold..."
> OBJECT ENCODING key3
"raw"
```

## 2.11 五种数据类型

### 2.11.1 String

- 编码：int、embstr、raw。
- 适合计数、缓存、分布式锁。

**三种编码图解**

```
【int 编码】(整数值在 0~9999 时共享对象, 否则嵌入 ptr):
  redisObject.type=STRING, encoding=INT
  ptr → (整数值直接嵌在 ptr 中, 非指针)

【embstr 编码】(长度 ≤ 44 字节, 一次分配内存):
  +--------------------+
  | redisObject (16B)  |  ← 与 SDS 连续分配
  | SDS header + buf   |  ← 最大 44B
  +--------------------+

【raw 编码】(长度 > 44 字节, 两次分配内存):
  redisObject.ptr → [SDS header + buf]  (独立 heap 分配)
```

**embstr vs raw 分界线（44 字节）**

- Redis 使用 jemalloc 分配器，64 字节是一个常见分配单元。
- `redisObject` 占 16 字节，SDS header（sdshdr8）占 `3+1` ≈ 4 字节，剩余 `64 - 16 - 4 = 44` 字节可为字符串内容。
- 44 字节以内一次 `malloc`，以上需两次 `malloc`。少一次内存分配 = 少一次系统调用 = 更快。

**编码转换规则**

- `int` 编码：只有 `set` 整数值时才用。一旦 append 字符串或值不再是纯整数 → 转为 `raw`。
- `embstr` 编码：**只读不写**。任何修改操作（`APPEND`, `SETRANGE` 等）直接转 `raw`，无法回退为 `embstr`。
- `raw` 编码：可任意修改。

### 2.11.2 List

- 旧版使用 QuickList。
- 适合队列、时间线。

**QuickList 结构说明**

- Redis 3.2 后，List 统一使用 **QuickList** 编码（替代早期 `linkedlist + ziplist` 方案）。
- QuickList 本质是：双向链表，每个链表节点内嵌一个 ziplist/listpack。
- 关键配置：
  - `list-max-ziplist-size`：控制单个 quicklistNode 内 ziplist 大小。默认 `-2`（8KB）。
  - `list-compress-depth`：控制压缩深度。默认 `0`（不压缩）。
- 典型场景：
  - **队列**（LPUSH + RPOP 或 RPUSH + LPOP）：fill 用负值（按字节），compress-depth 设为 1，两端解压缩。
  - **栈**（LPUSH + LPOP）：同上。
  - **随机索引**（LINDEX）：fill 设小值（如 `-1` = 4KB），方便定位，关闭压缩。

### 2.11.3 Set

- `intset` 或 `hashtable`。
- 适合去重、交并差。

**编码转换规则**

```
新 Set, all integers && count <= set-max-intset-entries
  → intset

后续添加非整数元素 OR count > set-max-intset-entries
  → hashtable (dict, val=NULL)
```

- `set-max-intset-entries`：默认 512。
- intset → hashtable 转换是**不可逆**的，即使后续删除元素使集合变小也不会切回 intset。

**集合运算的内部实现**

- `SINTER`（交集）：遍历最小集合，检查每个元素是否存在于其他集合。O(N*M)。
- `SUNION`（并集）：将所有集合元素复制到新集合。
- `SDIFF`（差集）：遍历第一个集合，检查元素是否不存在于其他集合。

### 2.11.4 ZSet

- `ziplist` 或 `skiplist`。
- 适合排行榜、延迟队列。

**双结构详解**

```
ZSet 对象:
  redisObject.type = ZSET

  ┌─ 小数据 (ziplist/listpack) ──────────────────────────┐
  │ encoding = ZIPLIST/LISTPACK                           │
  │ 数据在 ziplist 中交替存储: [member1, score1, member2,  │
  │ score2, ...] 按 score 升序排列                        │
  │ O(N) 查找，但 N 很小时无感知                           │
  └──────────────────────────────────────────────────────┘

  ┌─ 大数据 (skiplist + dict) ────────────────────────────┐
  │ encoding = SKIPLIST                                   │
  │ zset: { dict, zsl }                                   │
  │ dict: {member → score}        O(1) 单点查分           │
  │ zsl (skiplist): {score,member}→ O(log N) 范围查/排序  │
  │ 两者共享 member 指针，内存高效                         │
  └──────────────────────────────────────────────────────┘
```

**配置参数**

- `zset-max-ziplist-entries`：默认 **128**。member 数量 ≤ 128 时使用 ziplist/listpack。
- `zset-max-ziplist-value`：默认 **64** 字节。每个 member 的长度 ≤ 64 时才能用。

二者**必须同时满足**才使用 ziplist/listpack 编码。

**转换规则**

- ziplist/listpack → skiplist：任一配置条件不满足时触发转换。
- 该转换为**不可逆**。

### 2.11.5 Hash

- `ziplist/listpack` 或 `hashtable`。
- 适合对象存储。

**编码转换规则**

```
新 Hash, field-count ≤ hash-max-ziplist-entries
      && each-field-value ≤ hash-max-ziplist-value
  → ziplist/listpack (field, value 交替存储)

否则
  → hashtable (dict)
```

- `hash-max-ziplist-entries`：默认 **512**。
- `hash-max-ziplist-value`：默认 **64** 字节。
- Redis 7.0+ 中 `listpack` 替代 `ziplist` 作为 Hash 的小数据编码。
- 转换为**不可逆**。

**Hash 使用场景与编码权衡**

| 场景 | 推荐编码 | 理由 |
|------|----------|------|
| 固定少量字段（如用户属性 user:1001） | ziplist/listpack | 内存少，字段数 << 512 |
| 动态增长/字段较多（如购物车，用户可能频繁变动） | 考虑默认 | 超阈值自然切换 |
| JSON 替代（多个键值对） | ziplist/listpack | 相比 String 存 JSON，可部分读写，节省反序列化开销 |
| 大量小 Hash（如百万用户属性） | 调大 `hash-max-ziplist-entries` | 内存节省显著，但 O(N) 查找需注意 |

# 3. Redis网络模型

## 3.1 用户空间与内核空间

- 用户态无法直接操作硬件。
- IO 需陷入内核态，涉及上下文切换。

### 3.1.1 为什么区分用户空间与内核空间

- 操作系统为保护系统安全，将虚拟地址空间划分为**用户空间**和**内核空间**。
- 用户程序运行在用户态（Ring 3），无法直接访问内核内存、磁盘、网卡等硬件。
- IO 操作必须通过**系统调用**陷入内核态（Ring 0），内核代理执行后再返回用户态。

### 3.1.2 上下文切换的代价

- 每次系统调用涉及（部分上下文切换）：
  - 保存/恢复 CPU 寄存器（约几条~几十条指令）。
  - 切换页表（TLB 刷新，CPU cache miss）。
- 高并发场景下频繁的系统调用 → CPU 大部分时间浪费在切换而非处理业务。

### 3.1.3 DMA（直接内存访问）

- 传统 IO：CPU 负责将磁盘数据拷贝到内核 buffer，再拷贝到用户 buffer，CPU 全程参与。
- DMA：磁盘控制器直接将数据写入内核缓冲区，**CPU 不参与**数据搬移，仅发起指令和接收中断。
- 网卡同样通过 DMA 将网络数据写入内核的 socket buffer。

### 3.1.4 Page Cache / Socket Buffer

- 内核维护**页高速缓存（Page Cache）** 缓存文件数据。
- Socket 在内核层有**发送缓冲区**和**接收缓冲区**。
- 应用程序 `read()` 从 socket receive buffer 拷贝到用户空间；`write()` 从用户空间拷贝到 socket send buffer。

## 3.2 阻塞IO

- 线程一直阻塞等待数据。
- 简单但性能差。

### 3.2.1 阻塞 IO 流程（伪代码 + 时序）

```
【服务端 - 单线程模型】
socket = socket(AF_INET, SOCK_STREAM)
bind(socket, addr)
listen(socket)

while (1) {
    // Step 1: accept 阻塞等待连接
    client = accept(socket)          // 阻塞, 直到有连接到达

    // Step 2: read 阻塞等待数据
    n = read(client, buf, size)      // 阻塞, 直到 client 发送数据

    // Step 3: 处理数据
    process(buf)

    // Step 4: write 响应
    write(client, response)
    close(client)
}
```

```
时序图 (单连接处理):
  Client          Kernel           Server Process
    |               |                    |
    |--connect----->|                    |--accept(阻塞)-->
    |               |--通知-------------->|
    |               |                    |--read(阻塞)--->
    |--data-------->|                    |
    |               |--数据就绪---------->|--返回读取到的数据
    |               |                    |--process()---->
    |               |<--write响应--------|--write-------->
    |<--response----|                    |
    |               |                    |--accept(阻塞)-->(等待下一个连接)
```

**问题**：每次只能处理一个连接，处理期间其他客户端全部等待。

## 3.3 非阻塞IO

- 轮询方式检查数据。
- CPU 消耗大。

### 3.3.1 非阻塞 IO 流程（伪代码）

```
【服务端 - 轮询模型】
socket = socket(...)
fcntl(socket, F_SETFL, O_NONBLOCK)    // 设置为非阻塞
bind(socket, addr)
listen(socket)
clients = []

while (1) {
    // 非阻塞 accept
    client = accept(socket)
    if (client != -1) {
        fcntl(client, F_SETFL, O_NONBLOCK)
        clients.append(client)
    }

    // 轮询所有已连接客户端
    for client in clients:
        n = read(client, buf, size)
        if (n > 0) {
            process(buf)
            write(client, response)
        } else if (n == -1 && errno == EAGAIN) {
            // 数据未就绪, 继续下一个
            continue
        } else {
            // 连接关闭
            close(client)
            clients.remove(client)
        }
}
```

```
时序图:
  Server Process                           Kernel
       |                                     |
       |--read(fd) → EAGAIN (未就绪)----------->|
       |--read(fd) → EAGAIN (未就绪)----------->|
       |--read(fd) → EAGAIN (未就绪)----------->|
       |         ... CPU 空转轮询 ...          |
       |--read(fd) → 数据就绪! ---------------->|
       |--process(), write() ----------------->|
```

**问题**：`read()` 返回 EAGAIN 时无事可做，CPU 在用户态空转，浪费大量 CPU 时间。连接数越多，每次轮询的 fd 数越多，效率越低。

## 3.4 IO多路复用

- 单线程监听多个 socket。
- 事件就绪后再读写。

### IO 多路复用流程（核心思想）

```
while (1) {
    // 1. 向内核注册多个 fd
    // 2. 内核监控这些 fd, 当任一 fd 数据就绪时返回
    ready_fds = multiplexer_wait(registered_fds)   // 阻塞等待

    // 3. 只遍历已就绪的 fd, 逐个非阻塞读写
    for fd in ready_fds:
        n = read(fd, buf, size)   // 非阻塞, 数据已就绪
        process(buf)
        write(fd, response)
}
```

```
时序图:
  Server Process         Kernel(多路复用器)
       |                        |
       |--注册 fd1,fd2,fd3------>|
       |                        |
       |--multiplexer_wait(阻塞)-->|--监控 fd1,fd2,fd3
       |                        |  fd2 数据到达
       |<---返回 fd2 就绪--------|
       |                        |
       |--read(fd2) 数据-------->|
       |--process()             |
       |--write(fd2) 响应------->|
       |                        |
       |--multiplexer_wait(阻塞)-->|--继续监控
```

**核心优势**：一次系统调用等待多个 fd，只处理就绪的 fd，不用轮询所有。

### 3.4.1 IO多路复用-select

- 通过 fd 集合轮询。
- 最大 fd 数限制。

**select 伪代码**

```c
fd_set readfds;
FD_ZERO(&readfds);

while (1) {
    FD_SET(listen_sock, &readfds);
    for (int i = 0; i < client_count; i++)
        FD_SET(client_fds[i], &readfds);

    // select 返回就绪 fd 总数，内核修改 readfds 仅保留就绪 fd
    int ready = select(max_fd + 1, &readfds, NULL, NULL, NULL);

    if (FD_ISSET(listen_sock, &readfds)) {
        int client = accept(listen_sock, ...);
        // 加入 client_fds
    }

    for (int i = 0; i < client_count; i++) {
        if (FD_ISSET(client_fds[i], &readfds)) {
            n = read(client_fds[i], buf, size);
            process(buf);
            write(client_fds[i], response);
        }
    }
}
```

**select 缺点**

- fd_set 使用位图，默认最大值 **1024**（`FD_SETSIZE`）。
- 每次调用都需将 fd_set **从用户态拷贝到内核态**。
- 内核需**遍历**全部 fd（O(N)）找到就绪的。
- 返回后，用户态仍然需要**遍历全部 fd** 确定哪些就绪。

### 3.4.2 IO多路复用-poll

- 无 fd 数限制。
- 仍然线性遍历。

**poll 伪代码**

```c
struct pollfd fds[MAX_FDS];
fds[0].fd = listen_sock;
fds[0].events = POLLIN;

int nfds = 1;

while (1) {
    int ready = poll(fds, nfds, -1);  // -1: 无限阻塞

    if (fds[0].revents & POLLIN) {
        int client = accept(listen_sock, ...);
        fds[nfds].fd = client;
        fds[nfds].events = POLLIN;
        nfds++;
    }

    for (int i = 1; i < nfds; i++) {
        if (fds[i].revents & POLLIN) {
            n = read(fds[i].fd, buf, size);
            process(buf);
            write(fds[i].fd, response);
        }
    }
}
```

**poll 缺点**

- 无 fd 数量上限（`nfds` 用 `unsigned long`）。
- 但依然是**线性扫描**全部 `pollfd`，O(N) 复杂度。
- `pollfd` 数组每次仍需**完整拷贝**到内核空间。

### 3.4.3 IO多路复用-epoll

- 事件驱动，回调就绪 fd。
- 支持 ET/LT 模式。

**epoll 伪代码**

```c
int epfd = epoll_create(1);  // 创建 epoll 实例

struct epoll_event ev;
ev.events = EPOLLIN;
ev.data.fd = listen_sock;
epoll_ctl(epfd, EPOLL_CTL_ADD, listen_sock, &ev);  // 注册监听 fd

struct epoll_event events[MAX_EVENTS];  // 就绪事件数组

while (1) {
    // 只返回就绪的 fd, nready 为就绪数量
    int nready = epoll_wait(epfd, events, MAX_EVENTS, -1);

    for (int i = 0; i < nready; i++) {
        if (events[i].data.fd == listen_sock) {
            int client = accept(listen_sock, ...);
            ev.events = EPOLLIN | EPOLLET;  // ET 模式
            ev.data.fd = client;
            epoll_ctl(epfd, EPOLL_CTL_ADD, client, &ev);
        } else {
            n = read(events[i].data.fd, buf, size);
            process(buf);
            write(events[i].data.fd, response);
        }
    }
}
```

**select vs poll vs epoll 对比**

| 特性 | select | poll | epoll |
|------|--------|------|-------|
| 最大 fd 数 | 1024 (FD_SETSIZE) | 无上限 | 无上限（受系统限制） |
| 内核数据结构 | fd_set 位图 | pollfd 数组 | 红黑树（注册）+ 就绪链表 |
| 注册与监听分离 | 否，每次传所有 fd | 否，每次传所有 fd | 是，epoll_ctl 注册一次 |
| 拷贝开销 | 每次拷贝全量 fd_set | 每次拷贝全量 pollfd | 仅在注册时拷贝（epoll_ctl） |
| 内核查找就绪 fd | O(N) 遍历 | O(N) 遍历 | O(1) 回调 + 就绪链表 |
| 返回后用户遍历 | O(N) 全量 | O(N) 全量 | O(M) 仅遍历就绪的 |
| 触发模式 | LT | LT | LT + ET |
| 使用场景 | 小并发 | 中小并发 | 高并发（> 1000） |

## 3.5 epoll的ET和LT模式

- LT：水平触发，容易编程。
- ET：边缘触发，性能高但需一次读完。

### 3.5.1 LT（Level Triggered，水平触发）

- **触发条件**：只要 socket 接收缓冲区有数据（可读）或发送缓冲区未满（可写），每次 `epoll_wait` 都会通知。
- **示例**：
  ```
  fd 缓冲区收到 2KB 数据
    → epoll_wait 返回 fd 通知
    → 应用 read 了 1KB (还剩 1KB)
    → 下次 epoll_wait 再次返回 fd 通知 (因为数据未读完)
    → 应用 read 剩余 1KB
    → 下次 epoll_wait 不再通知 (缓冲区已空)
  ```
- **优点**：编程简单，不要求一次性读完，漏读不丢事件。
- **缺点**：可能产生多余的通知调用，稍低效。

### 3.5.2 ET（Edge Triggered，边缘触发）

- **触发条件**：只在状态**变化**的时刻通知一次（如缓冲区从空→非空，或从满→非满）。
- **示例**：
  ```
  fd 缓冲区收到 2KB 数据 (状态变化！)
    → epoll_wait 返回 fd 通知 (仅此一次)
    → 应用 read 了 1KB (还剩 1KB)
    → 下次 epoll_wait 不返回 fd (状态未再变化)
    → 再下次 epoll_wait 仍不返回 → 剩余 1KB 数据永远不会被读到
  ```
- **正确做法**：ET 模式下，读到 `read()` 返回 `EAGAIN` 为止。
  ```c
  while (1) {
      n = read(fd, buf, size);
      if (n > 0) process(buf);
      else if (n == -1 && errno == EAGAIN) break;  // 读完
      else break;  // 出错或连接关闭
  }
  ```
- **优点**：减少系统调用次数，事件仅通知一次。
- **缺点**：编程复杂度高，必须非阻塞 IO + 循环读到 EAGAIN。

### 3.5.3 ET vs LT 选择

- Redis 默认使用 **LT 模式**，代码简单、不易出错。
- Nginx 使用 **ET 模式**，追求极致的连接处理效率。

## 3.6 基于epoll的服务端流程

- `epoll_create` 创建实例。
- `epoll_ctl` 注册 socket。
- `epoll_wait` 等待就绪事件。
- 读写处理并继续监听。

### 3.6.1 完整 C 风格伪代码

```c
#include <sys/socket.h>
#include <sys/epoll.h>
#include <netinet/in.h>
#include <fcntl.h>
#include <unistd.h>

#define MAX_EVENTS 1024

int main() {
    // 1. 创建 socket
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);

    // 2. 设置非阻塞
    int flags = fcntl(listen_fd, F_GETFL, 0);
    fcntl(listen_fd, F_SETFL, flags | O_NONBLOCK);

    // 3. 绑定地址
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(6379);          // Redis 默认端口
    addr.sin_addr.s_addr = INADDR_ANY;
    bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr));

    // 4. 监听
    listen(listen_fd, SOMAXCONN);         // SOMAXCONN = 128 (默认)

    // 5. 创建 epoll 实例
    int epfd = epoll_create(1);

    // 6. 注册 listen_fd 到 epoll (LT 模式)
    struct epoll_event ev, events[MAX_EVENTS];
    ev.events = EPOLLIN;                  // 可读事件
    ev.data.fd = listen_fd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev);

    // 7. 事件循环
    while (1) {
        // 阻塞等待就绪事件, timeout = -1 表示无限等待
        int nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);

        for (int i = 0; i < nfds; i++) {
            if (events[i].data.fd == listen_fd) {
                // 新连接到达
                struct sockaddr_in client_addr;
                socklen_t client_len = sizeof(client_addr);
                int client_fd = accept(listen_fd,
                    (struct sockaddr*)&client_addr, &client_len);

                // 设置客户端 socket 为非阻塞
                fcntl(client_fd, F_SETFL, O_NONBLOCK);

                // 注册到 epoll
                ev.events = EPOLLIN;          // 监听可读
                ev.data.fd = client_fd;
                epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &ev);
            } else {
                // 已有客户端数据到达
                int client_fd = events[i].data.fd;
                char buf[4096];

                int n = read(client_fd, buf, sizeof(buf));
                if (n <= 0) {
                    // 连接关闭或错误
                    epoll_ctl(epfd, EPOLL_CTL_DEL, client_fd, NULL);
                    close(client_fd);
                } else {
                    // 处理请求 (Redis 核心: 命令解析 + 执行)
                    char *response = process_command(buf, n);

                    // 回写响应 (实际 Redis 中通过 epoll EPOLLOUT 事件异步写)
                    write(client_fd, response, strlen(response));
                }
            }
        }
    }
    close(epfd);
    close(listen_fd);
    return 0;
}
```

## 3.7 信号驱动IO及异步IO

- 信号驱动：就绪时通知应用。
- 异步 IO：内核完成后回调，无需应用阻塞。

### 3.7.1 信号驱动 IO（Signal-Driven IO）

- 应用调用 `sigaction()` 注册 SIGIO 信号处理函数。
- 内核数据就绪时，发送 SIGIO 信号给进程。
- 信号处理函数中执行 `read()` 操作。
- **本质仍为同步 IO**：`read` 调用仍需应用主动执行。

### 3.7.2 异步 IO（AIO / io_uring）

- 应用调用 `aio_read()`，传入缓冲区 + 回调。
- 内核完成所有操作（等待数据 + 拷贝数据到用户空间缓冲区）后通知应用。
- **真正的异步 IO**：应用无需执行 `read()` 操作。

### 3.7.3 IO 模型对比总结

| 模型 | 发起 | 等待数据 | 拷贝数据 | 是否阻塞应用 |
|------|------|----------|----------|------------|
| 阻塞 IO | read | 阻塞 | 阻塞 | 全程阻塞 |
| 非阻塞 IO | read | 轮询 | read 拷贝 | 轮询不阻塞 |
| IO 多路复用 | select/poll/epoll | select 阻塞 | read 拷贝 | select 阻塞, read 不阻塞 |
| 信号驱动 IO | sigaction | 不阻塞 | read 拷贝 | 无阻塞 |
| 异步 IO | aio_read | 不阻塞 | 不阻塞 | 完全无阻塞 |

## 3.8 Redis是单线程吗，为什么用单线程

- 核心命令处理是单线程。
- 依赖 IO 多路复用减少阻塞。
- 避免多线程锁带来的开销。

### 3.8.1 Redis 事件循环详解

```c
// Redis aeEventLoop 核心结构 (ae.h)
typedef struct aeEventLoop {
    int maxfd;                          // 当前最大 fd
    long long timeEventNextId;
    aeFileEvent *events;                // 注册的 file events
    aeFiredEvent *fired;                // 就绪事件
    aeTimeEvent *timeEventHead;         // 时间事件链表头 (如 serverCron)
    int stop;
    void *apidata;                      // epoll/kqueue/select 数据
    aeBeforeSleepProc *beforesleep;     // 每次 event loop 迭代前调用
    aeBeforeSleepProc *aftersleep;
} aeEventLoop;

typedef struct aeFileEvent {
    int mask;               // AE_READABLE | AE_WRITABLE
    aeFileProc *rfileProc;  // 读回调
    aeFileProc *wfileProc;  // 写回调
    void *clientData;
} aeFileEvent;

typedef struct aeTimeEvent {
    long long id;
    long when_sec, when_ms;    // 到期时间
    aeTimeProc *timeProc;      // 回调(如 serverCron)
    aeEventFinalizerProc *finalizerProc;
    void *clientData;
    struct aeTimeEvent *next;  // 链表
} aeTimeEvent;
```

### 3.8.2 Redis 事件循环主流程

```
main() → initServer() → aeMain()
  └── aeProcessEvents() 循环
        │
        ├── 1. 计算等待超时 (最近 time event 的到期时间)
        │      如果有 time event → 计算差值
        │      如果没有 → 无限等待
        │
        ├── 2. beforesleep() 钩子 (6.0+ 多线程 IO 在此执行)
        │
        ├── 3. aeApiPoll() → epoll_wait() 等待 fd 事件
        │
        ├── 4. aftersleep() 钩子
        │
        ├── 5. 遍历就绪 fd, 执行 rfileProc/wfileProc
        │      (readQueryFromClient → processInputBuffer → processCommand)
        │
        └── 6. processTimeEvents()
               (serverCron: 过期键清理、rehash、统计、AOF 重写触发...)
```

### 3.8.3 File Events vs Time Events

| 类型 | 用途 | 示例 |
|------|------|------|
| File Events | 网络 IO 事件 | 客户端连接、数据读写、集群通信 |
| Time Events | 定时任务 | serverCron (每秒 10~100 次)、过期键清理、超时连接关闭、rehash 推进 |

### 3.8.4 为什么单线程这么快

1. **纯内存操作**：绝大部分命令在内存中完成，无需磁盘 IO（持久化异步进行）。
2. **高效数据结构**：SDS、Dict、SkipList、QuickList 等针对内存效率优化。
3. **IO 多路复用**：epoll 单线程处理数万连接，无上下文切换。
4. **无锁竞争**：单线程执行命令，无需加锁/解锁，无并发控制开销。
5. **避免上下文切换**：不需要线程调度，CPU cache 命中率高。

### 3.8.5 单线程的局限性

| 瓶颈类型 | 示例命令 | 影响 |
|----------|----------|------|
| CPU 密集型 | `KEYS *`、`SORT big_set`、`SINTERSTORE` | 阻塞其他所有请求 |
| 大 Key 操作 | `DEL big_hash`（数百万元素） | 单线程 O(N) 删除，阻塞 |
| 范围查询 | `LRANGE big_list 0 -1` | 大数据量回传阻塞网络和内存 |
| RDB/AOF fork | `bgsave` 时 fork 子进程 | 大内存时 fork 有短暂阻塞 |

### 3.8.6 解决方案

- `KEYS` → `SCAN` 游标分批。
- 大 Key 删除 → `UNLINK`（异步删除，Redis 4.0+）或 `HSCAN` + `HDEL` 分批。
- 慢查询 → 通过 `SLOWLOG` 监控，`redis-benchmark` 压测。

## 3.9 Redis单线程及多线程网络模型变更

- 6.0 引入多线程网络 IO。
- 命令执行仍为单线程，提升网络吞吐。

### 3.9.1 Redis 6.0+ 多线程 IO 模型

```
                    ┌──────────────────────────┐
                    │    单线程命令执行引擎       │
                    │  (processCommand)         │
                    └──────────▲───────────────┘
                               │ 解析好的命令
                               │
         ┌─────────────────────┴─────────────────────┐
         │           IO 线程池 (多线程)                │
         │  Thread 0  Thread 1  Thread 2  ... Thread N │
         │  read/write read/write read/write          │
         └──────────────▲─────────────────────────────┘
                        │
                   epoll_wait()
                        │
              ┌─────────┴─────────┐
              │ socket1  socket2  │  ... (数千连接)
              └───────────────────┘

处理流程:
  1. 主线程 epoll_wait 等待可读事件
  2. 就绪 fd 分配给 IO 线程池，多线程并行 read 并解析 RESP 协议
  3. 所有 IO 线程完成后，主线程串行执行命令 (单线程)
  4. 执行完成后，响应数据分配给 IO 线程池并行 write 回客户端
```

### 3.9.2 哪些部分是多线程 vs 单线程

| 阶段 | 单/多线程 | 说明 |
|------|-----------|------|
| `epoll_wait` | 主线程 | 事件分发 |
| 网络读取 + RESP 解析 | **多线程**（并行） | io-threads 个线程 |
| 命令执行 | **单线程**（串行） | 保持原子性和简单性 |
| 响应回写 | **多线程**（并行） | io-threads 个线程 |

### 3.9.3 配置参数

```conf
# 默认关闭多线程 IO
io-threads 4                          # IO 线程数, 建议 ≤ CPU 核心数, 最少 1
io-threads-do-reads yes               # 是否在读取阶段也使用多线程 (Redis 6.0 默认 yes)
```

- 若 `io-threads 1`（默认），等于单线程模式。
- 一般建议设为 **4**，在 4 核以上机器可获得明显提升。

### 3.9.4 性能提升数据

- 在 8 核 CPU、万兆网卡环境下，32 字节 SET/GET 基准测试：
  - 单线程：~10 万 QPS
  - 4 个 IO 线程：~40 万 QPS（约 **4x** 提升）
- 提升主要在高并发（>1000 连接）场景显著，低并发场景开启多线程反而有线程调度开销。

### 3.9.5 何时启用多线程 IO

| 场景 | 建议 |
|------|------|
| 单机 QPS < 5 万 | 保持单线程（io-threads 1） |
| 单机 QPS 5~10 万 | io-threads 2~4 |
| CPU 核心数 ≥ 4 + 万兆网卡 | io-threads 4 |
| 云服务器（vCPU 虚拟化） | 谨慎调高，线程调度开销可能抵消收益 |
| 高延迟连接（如跨地域） | 开启，减少客户端等待时间 |

# 4. Redis通信协议

## 4.1 RESP协议

- 文本协议，易解析。
- 结构：`+`、`-`、`:`、`$`、`*`。

### 4.1.1 RESP2 完整规范

RESP 有 5 种数据类型：

| 类型 | 前缀 | 格式 | 说明 |
|------|------|------|------|
| **Simple String** | `+` | `+OK\r\n` | 简单字符串，无二进制安全 |
| **Error** | `-` | `-ERR unknown command\r\n` | 错误信息 |
| **Integer** | `:` | `:1000\r\n` | 64 位整数 |
| **Bulk String** | `$` | `$5\r\nhello\r\n` | 二进制安全字符串（带长度前缀） |
| **Array** | `*` | `*3\r\n$3\r\nSET\r\n...\r\n` | 数组（元素可为任意 RESP 类型） |

### 4.1.2 Null 值

- **Null Bulk String**：`$-1\r\n`（长度为 -1 表示 null，如 GET 不存在的 key）。
- **Null Array**：`*-1\r\n`（长度为 -1 表示 null，如 BLPOP 超时）。

### 4.1.3 常见命令 RESP 示例

**SET key value**：
```
请求:
*3\r\n
$3\r\n
SET\r\n
$3\r\n
key\r\n
$5\r\n
value\r\n

响应:
+OK\r\n
```

**GET key（存在）**：
```
请求:
*2\r\n
$3\r\n
GET\r\n
$3\r\n
key\r\n

响应:
$5\r\n
value\r\n
```

**GET key（不存在）**：
```
响应:
$-1\r\n
```

**INCR counter**：
```
请求:
*2\r\n
$4\r\n
INCR\r\n
$7\r\n
counter\r\n

响应:
:42\r\n
```

**HMSET user name tom age 20**：
```
请求:
*6\r\n
$5\r\n
HMSET\r\n
$4\r\n
user\r\n
$4\r\n
name\r\n
$3\r\n
tom\r\n
$3\r\n
age\r\n
$2\r\n
20\r\n

响应:
+OK\r\n
```

### 4.1.4 RESP3（Redis 6.0+）

RESP3 兼容 RESP2，新增类型解决 RESP2 的局限：

| 新类型 | 前缀 | 说明 |
|--------|------|------|
| **Null** | `_` | `_\r\n`，独立 null 类型 |
| **Boolean** | `#` | `#t\r\n` 或 `#f\r\n` |
| **Double** | `,` | `,3.14\r\n`，浮点数 |
| **Big Number** | `(` | `(12345678901234567890\r\n`，big integer |
| **Map** | `%` | `%2\r\n+key\r\n+val\r\n...\r\n`，键值对 |
| **Set** | `~` | `~3\r\n+mem1\r\n+mem2\r\n+mem3\r\n`，集合 |
| **Verbatim String** | `=` | `=3\r\ntxt\r\n...\r\n`，带编码信息的字符串 |
| **Push** | `>` | `>2\r\n...\r\n`，服务端推送（PubSub、监控） |

优点：结构化响应更清晰（HELLO 命令中客户端可协商使用 RESP3），服务端推送支持更优雅。

示例：
```
*2
$4
PING
$4
PONG
```

## 4.2 基于Socket的自定义Redis客户端

- TCP 连接到 6379。
- 按 RESP 拼接命令并发送。
- 解析返回结果。

### 4.2.1 完整 Java Socket Redis 客户端

```java
import java.io.*;
import java.net.Socket;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;

public class SimpleRedisClient implements Closeable {
    private Socket socket;
    private OutputStream out;
    private InputStream in;

    public SimpleRedisClient(String host, int port) throws IOException {
        this.socket = new Socket(host, port);
        this.out = socket.getOutputStream();
        this.in = socket.getInputStream();
    }

    /**
     * 发送任意 Redis 命令
     * @param args 命令 + 参数, 如 sendCommand("SET", "key", "value")
     * @return 解析后的响应对象
     */
    public Object sendCommand(String... args) throws IOException {
        // 构造 RESP 协议命令
        StringBuilder sb = new StringBuilder();
        sb.append("*").append(args.length).append("\r\n");
        for (String arg : args) {
            byte[] bytes = arg.getBytes(StandardCharsets.UTF_8);
            sb.append("$").append(bytes.length).append("\r\n");
            sb.append(arg).append("\r\n");
        }
        out.write(sb.toString().getBytes(StandardCharsets.UTF_8));
        out.flush();

        // 解析返回
        return readResponse();
    }

    /**
     * 读取一个 RESP 类型的响应
     */
    private Object readResponse() throws IOException {
        int firstByte = in.read();
        if (firstByte == -1) return null;

        switch (firstByte) {
            case '+': return readLine();                        // Simple String
            case '-': throw new RuntimeException(readLine());   // Error
            case ':': return Long.parseLong(readLine());        // Integer
            case '$': return readBulkString();                  // Bulk String
            case '*': return readArray();                       // Array
            default:  throw new IOException("Unknown RESP prefix: " + (char) firstByte);
        }
    }

    private String readLine() throws IOException {
        StringBuilder sb = new StringBuilder();
        int c;
        while ((c = in.read()) != '\r') {
            sb.append((char) c);
        }
        in.read(); // 跳过 \n
        return sb.toString();
    }

    private String readBulkString() throws IOException {
        long len = Long.parseLong(readLine());
        if (len == -1) return null;  // Null Bulk String

        byte[] buf = new byte[(int) len + 2]; // +2 for \r\n
        int totalRead = 0;
        while (totalRead < buf.length) {
            int n = in.read(buf, totalRead, buf.length - totalRead);
            if (n == -1) break;
            totalRead += n;
        }
        return new String(buf, 0, (int) len, StandardCharsets.UTF_8);
    }

    private List<Object> readArray() throws IOException {
        long len = Long.parseLong(readLine());
        if (len == -1) return null;  // Null Array

        List<Object> result = new ArrayList<>((int) len);
        for (int i = 0; i < len; i++) {
            result.add(readResponse());
        }
        return result;
    }

    @Override
    public void close() throws IOException {
        if (socket != null) socket.close();
    }

    // 使用示例
    public static void main(String[] args) throws IOException {
        try (SimpleRedisClient client = new SimpleRedisClient("127.0.0.1", 6379)) {
            Object result1 = client.sendCommand("SET", "hello", "world");
            System.out.println("SET: " + result1);                                 // OK

            Object result2 = client.sendCommand("GET", "hello");
            System.out.println("GET: " + result2);                                 // world

            Object result3 = client.sendCommand("INCR", "counter");
            System.out.println("INCR: " + result3);                                // 1 (or N)

            Object result4 = client.sendCommand("HMGET", "user:1", "name", "age");
            System.out.println("HMGET: " + result4);                               // [tom, 20]
        }
    }
}
```

### 4.2.2 命令管道（Pipeline）

```java
// 管道发送: 批量发送命令，不等待单个响应，最后统一读取
public List<Object> pipeline(String[]... commands) throws IOException {
    // 构造所有命令并一次性发送
    StringBuilder sb = new StringBuilder();
    for (String[] cmd : commands) {
        sb.append("*").append(cmd.length).append("\r\n");
        for (String arg : cmd) {
            byte[] bytes = arg.getBytes(StandardCharsets.UTF_8);
            sb.append("$").append(bytes.length).append("\r\n");
            sb.append(arg).append("\r\n");
        }
    }
    out.write(sb.toString().getBytes(StandardCharsets.UTF_8));
    out.flush();

    // 按顺序读取所有响应
    List<Object> results = new ArrayList<>();
    for (int i = 0; i < commands.length; i++) {
        results.add(readResponse());
    }
    return results;
}
```

### 4.2.3 连接池概念

```java
// 连接池核心要素
public class RedisConnectionPool {
    private BlockingQueue<SimpleRedisClient> idlePool;  // 空闲连接
    private int maxTotal;   // 最大连接数
    private int maxIdle;    // 最大空闲连接数
    private String host;
    private int port;

    // borrow: 从池中获取连接 (优先空闲，不足时创建，超出 maxTotal 则等待/报错)
    // return: 归还连接到池中 (超过 maxIdle 则关闭)
    // validate: 定期检测连接可用性 (PING)
    // 常见实现: JedisPool, Lettuce (基于 Netty, 单连接多路复用)
}
```

# 5. Redis内存回收

## 5.1 过期key处理

- 定期删除 + 惰性删除。
- 定期扫描部分 key 释放空间。

### 5.1.1 过期键存储方式

- Redis 内部维护一个独立的 **Expires Dict**（过期字典），存储 `key → 过期时间戳(毫秒)`。
- `EXPIRE key 60`：在 expires dict 中插入 key, 过期时间戳 = 当前时间 + 60秒。
- `PERSIST key`：从 expires dict 中移除 key。
- `TTL key`：查询 expires dict 中对应时间戳与当前时间的差值。

### 5.1.2 惰性删除（Lazy Expiration）

- **触发点**：
  - 每次 `GET/SET/HGET/INCR` 等读写命令执行前，先调用 `expireIfNeeded()` 检查 key 是否过期。
  - 在 dict rehash 过程中访问 key 时也会检查。
  - 主从复制中，从节点不主动过期，依赖主节点 `DEL` 同步。
- **逻辑**：
  ```c
  int expireIfNeeded(redisDb *db, robj *key) {
      if (!keyIsExpired(db, key)) return 0;   // 未过期, 继续
      if (server.masterhost != NULL) return 1; // 从节点只返回过期标识, 不删除
      deleteKey(db, key);                      // 主节点删除
      server.stat_expiredkeys++;
      return 1;
  }
  ```

### 5.1.3 定期删除（Active Expiration）

- **执行位置**：`serverCron()` → `activeExpireCycle()`。
- **算法**：
  1. 每个遍历周期随机抽取 **20** 个带过期时间的 key。
  2. 删除其中已过期的 key。
  3. 如果已过期比例 > 25%，说明过期 key 密度大，**继续循环**抽取+删除。
  4. 设定时间上限（如 CPU_TIME=25ms），避免长时间阻塞主线程。
  5. 16 个数据库（db0~db15）轮流清理，一轮只能清部分 db。
- **复杂度限制**：`ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP = 20`，`ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC = 25`。

### 5.1.4 RDB/AOF 与过期 Key

| 场景 | 行为 |
|------|------|
| RDB 保存（SAVE） | 不保存已过期 key |
| RDB 加载（主节点） | 忽略已过期 key |
| RDB 加载（从节点） | 不过滤（主节点后续会同步 DEL） |
| AOF 写入 | 过期 key 被惰性删除后，追加一条 `DEL key` 到 AOF |
| AOF 重写 | 忽略已过期 key |

## 5.2 内存淘汰策略

- `noeviction`：禁止写入。
- `allkeys-lru`、`volatile-lru`。
- `allkeys-lfu`、`volatile-lfu`。
- `allkeys-random`、`volatile-ttl`。

### 5.2.1 全部 8 种淘汰策略

| 策略 | 淘汰范围 | 选择逻辑 | 适用场景 |
|------|----------|----------|----------|
| **noeviction** | 不淘汰 | 写请求直接报错 | 仅当缓存, 严格数据一致 |
| **allkeys-lru** | 全部 key | 近似 LRU, 淘汰最久未使用 | 通用缓存, 幂律分布 |
| **volatile-lru** | 有 TTL 的 key | 近似 LRU, 淘汰最久未使用 | 持久 key + 过期 key 混合 |
| **allkeys-random** | 全部 key | 随机淘汰 | 访问模式均匀分布 |
| **volatile-random** | 有 TTL 的 key | 随机淘汰 | 同上 |
| **volatile-ttl** | 有 TTL 的 key | 淘汰剩余 TTL 最短的 key | 期望优先淘汰即将过期的 |
| **allkeys-lfu** | 全部 key | 近似 LFU, 淘汰最不频繁使用的 | 热点数据场景 (Redis 4.0+) |
| **volatile-lfu** | 有 TTL 的 key | 近似 LFU, 淘汰最不频繁使用的 | 热点 + 持久混合 (Redis 4.0+) |

### 5.2.2 近似 LRU 算法

- Redis 的 LRU **不是精确的**（不维护双向链表按访问时间排序）。
- **近似实现**：
  1. 使用 `redisObject.lru` 字段（24 位）存储最后访问时间戳（秒级，`LRU_CLOCK()`）。
  2. 触发淘汰时，随机采样 `maxmemory-samples` 个 key（默认 **5**）。
  3. 从中选出 `lru` 值最小的（最久未访问的）key 淘汰。
  4. 重复采样-淘汰直到内存低于 maxmemory。
- **参数**：`maxmemory-samples 5`，值越大越接近精确 LRU，但 CPU 开销也越大。

### 5.2.3 近似 LFU 算法（Redis 4.0+）

LFU 用 `lru` 字段的 24 位分成两部分：
```
+-----------------------+------------------------+
| 16 bits: last_dec_time | 8 bits: logarithmic ctr|
+-----------------------+------------------------+
  分钟级时间戳              Morris Counter 频次计数
```

**计数器的增加**：
- 不是每访问一次就 +1，而是**概率性递增**。
- 计算公式：`p = 1 / (old_value * lfu_log_factor + 1)`
- 设定 `lfu-log-factor`（默认 **10**），值越大计数器增长越慢 → 区分冷热数据的粒度更细。

**计数器的衰减**：
- 键的计数器随时间衰减，确保不再被访问的键逐渐变为淘汰候选。
- 衰减公式：`衰减值 = (now - last_dec_time) / lfu_decay_time`
- `lfu-decay-time`：默认 **1**（每分钟衰减 1），设为 0 表示不衰减。

### 5.2.4 LRU vs LFU 选择

| 对比维度 | LRU | LFU |
|----------|-----|-----|
| 淘汰依据 | "多久没被访问" | "访问频次多低" |
| 适合场景 | 突发性访问少、访问模式较均匀 | 有明显冷热数据区分（如热点新闻） |
| 缺陷 | 某 key 仅在 1 小时前被访问一次, 可能误保留 | 新 key 频次低, 易被误淘汰 |
| 推荐策略 | allkeys-lru | allkeys-lfu (4.0+) |

### 5.2.5 淘汰执行流程

```
当内存使用 > maxmemory 时:
  1. 判断是否有需要淘汰的 key (noeviction → 报错)
  2. 计算需要释放的内存大小 (mem_tofree)
  3. 循环:
     a. 采样 maxmemory-samples 个 key 放入池
     b. 从池中选出空闲时间最大的 (LRU) 或频次最低的 (LFU)
     c. 删除该 key 并统计释放的字节数
     d. 若释放足够, 退出
  4. 执行写操作
```

## 5.3 内存碎片

### 5.3.1 什么是内存碎片

- Redis 频繁分配和释放不同大小的内存块，导致物理内存中出现不连续的小空闲空间。
- 这些空闲块无法被大块分配请求利用，形成外部碎片。
- 后果：实际已用内存 1GB，但操作系统分配了 2GB（碎片占 1GB）。

### 5.3.2 碎片率诊断

```bash
> INFO memory
used_memory:1048576000           # Redis 实际使用 (约 1GB)
used_memory_rss:2097152000       # OS 分配 (约 2GB)
mem_fragmentation_ratio:2.0      # 碎片率 = rss / used

# 健康值: 1.0 ~ 1.5
# 轻度碎片: 1.5 ~ 2.0
# 严重碎片: > 2.0
```

- 碎片率 < 1：表示部分内存被 swap 到磁盘（`used_memory_rss < used_memory`），性能会大幅下降。

### 5.3.3 碎片产生的原因

- 频繁更新不同大小的 value（如 `SET key "small"` 后 `SET key "a much longer value"`）。
- 大量 key 过期后被删除，留下的空间不连续。
- `jemalloc` 分配器的页（page）管理方式也会产生内部碎片。

### 5.3.4 主动碎片整理（Redis 4.0+）

```conf
activedefrag yes                                    # 开启主动整理 (默认 no)
active-defrag-ignore-bytes 100mb                    # 碎片量超过此值才开始整理
active-defrag-threshold-lower 10                    # 碎片率 > 1.1 时开始整理
active-defrag-threshold-upper 100                   # 碎片率 > 2 时加大整理力度
active-defrag-cycle-min 1                            # CPU 时间占比至少 1%
active-defrag-cycle-max 25                           # CPU 时间占比最多 25%
active-defrag-max-scan-fields 1000                  # 每次扫描最多处理的 set/hash/zset 字段数
```

### 5.3.5 何时开启/关闭

| 条件 | 建议 |
|------|------|
| 碎片率 > 1.5 持续 | 开启 `activedefrag yes` |
| 碎片率 < 1.1 | 无需开启，开销大于收益 |
| CPU 已经是瓶颈 | 关闭，整理消耗 CPU（5%~25%） |
| 内存足够但碎片高 | 开启，释放内存 |

## 5.4 jemalloc 内存分配器

### 5.4.1 jemalloc 概述

- jemalloc 是 FreeBSD 项目开发的通用内存分配器，Redis 默认使用（编译时可通过 `make MALLOC=libc` 切换）。
- 替代 glibc 的 `malloc`，在多线程环境下有更好的性能和更少的碎片。
- Facebook、Firefox 等也在使用 jemalloc。

### 5.4.2 核心设计

- **多级分配尺度**：Small（<14KB）、Large（<4MB）、Huge（≥4MB）。
- **线程缓存（tcache）**：每个线程有独立的分配缓存，减少锁竞争。
- **大小类（size class）**：预定义分配粒度（如 8B, 16B, 32B, 48B, 64B, ...），相近大小请求共享同一 size class，减少碎片。
- **内存通过 mmap 按 chunk（4MB）管理**。

### 5.4.3 jemalloc 与 Redis 的配合

- Redis 单线程命令执行 + 可选的 IO 多线程，jemalloc 的 tcache 对主线程仍有效。
- jemalloc 对 Redis 常见的小对象（SDS、dictEntry、ziplist entry）分配有优化的 size class。
- `MEMORY STATS` 和 `MEMORY MALLOC-STATS` 命令可查看 jemalloc 内部统计。

### 5.4.4 glibc malloc vs jemalloc

| 对比维度 | glibc malloc | jemalloc |
|----------|-------------|----------|
| 碎片率 | 较高，尤其高并发 | 低，专为减少碎片设计 |
| 多线程支持 | 锁竞争明显 | tcache 减少竞争 |
| 内存统计 | 基础 | 精细，可通过 INFO memory 查看 |
| Redis 默认 | 否 | 是 |

# 6. Redis事务

## 6.1 MULTI/EXEC/DISCARD 内部实现

```
MULTI
  →  client.flags |= CLIENT_MULTI  (标记客户端进入事务模式)
  →  后续命令不立即执行, 而是加入 client.multiState.commands 队列

命令入队 (SET key value):
  →  queueMultiCommand(cmd) → 加入 commands 数组

EXEC:
  →  遍历 commands 数组, 逐个调用 call(c, cmd)
  →  所有命令串联执行 (单线程, 中间不被其他客户端打断)
  →  返回所有命令的 RESP 结果数组

DISCARD:
  →  清空 commands 队列
  →  client.flags &= ~CLIENT_MULTI
  →  返回 +OK
```

### 6.1.1 WATCH 乐观锁实现

```
WATCH key1 key2:
  →  client 的 watched_keys 列表中加入 (db, key1), (db, key2)
  →  被监视的 key 在 redisDb.watched_keys 字典中注册

在 EXEC 执行前:
  →  检查 watched_keys 中的 key 是否被其他客户端修改过 (CAS)
  →  如果所有 key 均未修改 → 执行 commands 队列
  →  如果任一 key 被修改 → 返回 (nil), 不执行任何命令
```

- WATCH 本质上是一种**乐观锁**（Optimistic Locking / CAS）。
- 不需要显式 `UNWATCH`：EXEC/DISCARD 后自动取消所有 WATCH。

### 6.1.2 为什么 Redis 事务不支持回滚

- Redis 认为命令错误是**编程错误**（在开发阶段应被发现和避免），不是运行时错误。
- 回滚的实现会增加复杂性，违背 Redis 简洁高效的设计哲学。
- 实际上 `MULTI/EXEC` 中的语法错误会在入队时被检测（返回 `-ERR`），运行时类型错误（如对 List 执行 `HGET`）不会回滚已执行的命令。

### 6.1.3 Redis 事务的 ACID 分析

| 属性 | 支持情况 | 说明 |
|------|----------|------|
| **Atomicity（原子性）** | 部分支持 | EXEC 中命令串行执行，中间不被插入；入队阶段语法错误会拒绝执行。但运行时错误不会回滚已执行的命令。 |
| **Consistency（一致性）** | 基本支持 | 单线程 + 命令级别的校验，不破坏数据结构的内部一致性。 |
| **Isolation（隔离性）** | 支持 | 单线程执行命令，天然串行化隔离。 |
| **Durability（持久性）** | 取决于配置 | 若 `appendfsync no` 则在宕机时可能丢失数据；`appendfsync always` 可保证持久性但性能差。 |

# 7. Redis Lua脚本

## 7.1 Lua 脚本执行流程

```
客户端:
  EVAL "return redis.call('SET', KEYS[1], ARGV[1])" 1 key1 value1

服务端:
  1. 解析 EVAL 命令, 提取 script_body, numkeys, keys[], argv[]
  2. 调用 luaL_loadbuffer() 加载脚本字节码
  3. 创建 Lua 协程 (lua_newthread)
  4. 注入 redis.call / redis.pcall / KEYS / ARGV 等 API
  5. lua_resume / lua_pcall 执行脚本
  6. 将 Lua 返回值转为 RESP 格式返回客户端
```

### 7.1.1 SCRIPT LOAD + EVALSHA (避免重复传输脚本)

```bash
> SCRIPT LOAD "return redis.call('GET', KEYS[1])"
"a1b2c3d4e5f6..."               # SHA1 摘要

> EVALSHA a1b2c3d4e5f6... 1 key1
"value1"
```

- 原理：服务端维护一个 `server.lua_scripts` 字典（SHA1 → 脚本源码 + 字节码）。
- EVALSHA 如果 SHA1 不存在（如重启后），返回 `NOSCRIPT` 错误，客户端应回退到 `EVAL`。

### 7.1.2 lua-time-limit 配置

```conf
lua-time-limit 5000    # 脚本最大执行时间 (毫秒), 默认 5000ms
```

- Lua 脚本运行期间会阻塞主线程（单线程执行）。
- 超过 `lua-time-limit` 后，Redis **不强制终止**脚本（Lua 不支持强制中断），但会记录慢日志并允许 `SCRIPT KILL` 杀死只读脚本。
- 如果脚本已执行写操作，`SCRIPT KILL` 无效，只能 `SHUTDOWN NOSAVE` —— **写 Lua 脚本必须确保执行时间可控**。

### 7.1.3 脚本缓存与 SCRIPT FLUSH

- `SCRIPT FLUSH`：清空所有已缓存的脚本。主从同步环境中会复制到从节点。
- `SCRIPT EXISTS sha1 sha2 ...`：检查脚本是否已缓存。
- `SCRIPT DEBUG YES|SYNC|NO`：进入/退出调试模式。

### 7.1.4 为什么 Lua 脚本是原子的

- 脚本执行期间，Redis **不会处理任何其他命令**（单线程被脚本独占）。
- 相当于 `MULTI/EXEC` 的加强版：脚本内多个命令同属一个原子操作。
- 这既是优点（简单可靠），也是风险（长脚本阻塞一切）。

### 7.1.5 脚本复制（Redis 3.2+）

- **Redis 3.2 之前**：`EVAL` 语句直接复制到 AOF 和从节点（`scripts replication`），主从环境必须保证脚本完全一致。
- **Redis 3.2+**：默认使用 `effects replication`（`lua-replicate-commands yes`），将脚本产生的**写入命令**复制到 AOF/从节点。
  - 优点：主从脚本无需完全相同，减少带宽传输。
  - 缺点：脚本中带随机性或时间依赖时，复制的不确定性需通过 `redis.replicate_commands()` 控制。

# 8. Redis Stream内部实现

## 8.1 Rax（Radix Tree）概述

- Stream 的底层索引采用 **Rax（压缩基数树）**。
- Rax 是 Radix Tree（基数树）的压缩版，节点可以包含多个字符前缀，减少树的深度和节点数。
- 与普通 Radix Tree 不同，Rax 是**压缩的且支持值存储**的结构。

### 8.1.1 Stream 宏观结构

```
stream {
    rax *rax;               // 基数树索引: (stream_id + key) → streamID
    rax *cgroups;           // 消费者组索引 (Radix Tree)
    unsigned long long length;
    streamID last_id;       // 最后生成的 ID
    streamID first_id;      // 最早未被删除的 ID
    // ... statistics fields
}
```

### 8.1.2 Stream 消息存储

- 消息按 `stream_id`（ms 时间戳 + sequence）的顺序存入 **listpack** 的 Radix Tree 节点。
- 每个 Rax 节点包含一个 **listpack**，存储连续的若干条消息。
- 新消息追加到 listpack 尾部。

### 8.1.3 消费者组内部实现

```
consumerGroup {
    streamID last_id;             // 最后投递的消息 ID
    rax *pel;                     // PEL（Pending Entries List, Rax 字典)
    rax *consumers;               // 消费者列表 (Rax 字典)
}

consumer {
    streamID *pel;                // 该消费者的 PEL 条目
    mstime_t seen_time;           // 上次活跃时间
    // ...
}
```

### 8.1.4 PEL（Pending Entries List）

- 当消息被投递给消费者但**尚未 ACK**，记录在 `consumerGroup.pel` 和 `consumer.pel` 中。
- `XPENDING` 命令可查看未确认的消息列表。
- 支持消息**重新投递**（Claim）：`XCLAIM` 可将超时未 ACK 的消息转移给其他消费者。
- PEL 通过 Rax 存储，key 为 stream_id，value 包含消费者名、投递次数、投递时间等。

### 8.1.5 Stream 与传统消息队列对比

| 特性 | Redis Stream | Kafka | RabbitMQ |
|------|-------------|-------|----------|
| 消息存储 | 内存 (+ RDB/AOF) | 磁盘 | 磁盘 |
| ACK 机制 | 支持 (XACK) | 支持 | 支持 |
| 消费者组 | 支持 | 支持 | 支持 |
| 消息回溯 | 支持 (按 ID 重读) | 支持 | 有限 |
| 持久化 | RDB/AOF | 同步刷盘 | 磁盘持久化 |
| 适用于 | 轻量级、低延迟场景 | 海量日志/事件 | 企业集成 |

# 9. Redis模块系统

## 9.1 Redis Module API 概述

- Redis 4.0 引入模块系统，允许开发者通过 C 扩展 Redis 的功能（新命令、新数据类型）。
- 模块以 `.so` 动态库形式加载：`MODULE LOAD /path/to/module.so`。
- 模块注册回调获取生命周期通知：`RedisModule_OnLoad()` / `RedisModule_OnUnload()`。
- 模块可创建新的数据类型，实现自定义序列化/反序列化（RDB、AOF 支持）、复制等。

### 9.1.1 生态模块一览

| 模块 | 核心数据结构 | 核心能力 | 典型用例 |
|------|-------------|----------|----------|
| **RedisBloom** | Bloom Filter、Cuckoo Filter、Count-Min Sketch | 概率型去重/存在判断 | 黑名单、爬虫去重、推荐去已读 |
| **RedisJSON** | 树结构 (JSON 文档) | JSONPath 查询、部分更新 | 文档存储、API 响应缓存、配置管理 |
| **RediSearch** | 倒排索引 + 分词 | 全文搜索、模糊匹配、聚合 | 搜索引擎、日志检索、自动补全 |
| **RedisGraph** | 邻接表 + 矩阵 | Cypher 图查询 | 社交关系、推荐、知识图谱 |
| **RedisTimeSeries** | 压缩块 (chunk) + 双 delta 编码 | 时序数据写入、聚合、降采样 | IoT 监控、APM 指标、股价数据 |
| **RedisGears** | 触发器 + 函数流 | 事件驱动计算、流式处理 | 实时 ETL、事件处理 |
| **RedisAI** | Tensor 封装 | 模型推理 | 在线预测、实时推荐 |

### 9.1.2 各模块核心实现简析

**RedisBloom**：
- Bloom Filter：使用 `SETBIT` 位数组 + 多个 hash 函数。`BF.ADD` / `BF.EXISTS`。
- Cuckoo Filter：支持删除的布隆过滤器变体。
- Count-Min Sketch：近似频率估计，用于 `CMS.INCRBY`。

**RedisJSON**：
- 存储：JSON 文档以内存树存储，每个节点为 Key（JSON field）+ Value + Type。
- 操作：`JSON.SET key $ '{"a":1}'`、`JSON.GET key $.a`、`JSON.ARRAPPEND`、`JSON.NUMINCRBY`。
- JSONPath 支持原子部分更新，无需全量 get/set。

**RediSearch**：
- 索引结构：倒排索引（term → doc_id list）存储在内存中。
- 中文分词：依赖外部分词器（jieba、Friso 等）。
- 搜索：`FT.SEARCH idx "@title:redis"`，支持前缀、模糊、geo、tag、数值范围。
- 聚合管道：类似 Elasticsearch 的 Aggregation。

**RedisGraph**：
- 图存储：节点表 + 边表 + 邻接矩阵，基于 `GraphBLAS` 库。
- 查询：子集 Cypher (`MATCH (a)-[r]->(b) RETURN a, b`)。
- 场景：社交推荐（朋友的朋友）、反欺诈（关联分析）。

**RedisTimeSeries**：
- 规则压缩：固定大小 chunk（默认 256 samples/chunk），采用双 delta 编码压缩时间戳。
- 降采样：`TS.CREATERULE` 自动聚合原始数据到粗粒度 key。
- 聚合：`TS.RANGE key - + AGGREGATION avg 60000` 获取每分钟均值。

### 9.1.3 何时使用模块

- 需要高级数据结构但不想引入其他存储系统时。
- 希望利用 Redis 的持久化、复制、集群能力扩展功能。
- 非超高写入吞吐场景（模块会消耗额外 CPU/内存）。
- 对于重量级场景（PB 级搜索、大规模图计算），建议使用专用组件（Elasticsearch、Neo4j 等）。

# 10. 总结

- 深入理解 Redis 内部数据结构（SDS、IntSet、Dict、ZipList、ListPack、QuickList、SkipList）是高性能 Redis 使用的前提。
- 掌握网络模型（epoll、单线程模型、6.0+ 多线程 IO）有助于诊断性能瓶颈。
- 熟悉 RESP 协议才能实现安全的自定义客户端或理解代理实现。
- 合理配置内存过期策略与淘汰策略，是生产环境 Redis 稳定性的保障。
- 事务与 Lua 脚本提供了原子操作的机制，各有适用场景和局限。
- Stream 与模块系统拓展了 Redis 的能力边界，使其从缓存进化到消息队列、搜索引擎、时序数据库等。

---

> 参考：Redis 源码 (redis/src/)，Redis 官方文档 (redis.io)，《Redis设计与实现》(黄健宏)
