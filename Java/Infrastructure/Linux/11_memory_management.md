
# 1. 物理内存 vs 虚拟内存

## 1.1 虚拟内存的原理

Linux 为每个进程提供独立的虚拟地址空间，通过 MMU（内存管理单元）将虚拟地址映射到物理地址：

```
进程 A 虚拟地址空间             进程 B 虚拟地址空间
┌──────────────────┐          ┌──────────────────┐
│ 0x0000 0000~     │          │ 0x0000 0000~     │
│   0xFFFF FFFF    │          │   0xFFFF FFFF    │
│ (32位: 4GB)      │    MMU   │ (32位: 4GB)      │
│ 或              │ ───────→ │ 或              │
│ 0x0000 0000 0000~│          │ 0x0000 0000 0000~│
│   0x7FFF FFFF FFFF│          │   0x7FFF FFFF FFFF│
│ (64位: 128TB)   │          │ (64位: 128TB)   │
└──────────────────┘          └──────────────────┘
            ↕                         ↕
            └──────────┬──────────────┘
                       ↓
            ┌──────────────────────┐
            │    物理内存（4GB）    │
            │    ┌────┬────┬────┐  │
            │    │页框│页框│页框│  │
            │    └────┴────┴────┘  │
            └──────────────────────┘
```

**虚拟内存的优势**：
1. 进程隔离：一个进程不能访问另一个进程的内存
2. 简化内存管理：进程看到连续地址空间
3. 高效使用物理内存：共享库、页共享、页面换出
4. 允许运行大于物理内存的程序

## 1.2 分页机制

物理内存被划分为固定大小的**页框（Page Frame）**，虚拟内存被划分为相同大小的**页面（Page）**。默认页面大小为 4KB（x86_64）。

**页表**：记录虚拟页面到物理页框的映射关系。

```
虚拟地址：
┌────────────┬──────────────┐
│  页号       │   页内偏移    │
│  (VPN)     │   (Offset)   │
└──────┬─────┴──────┬───────┘
       │            │
       ↓            ↓
   页表查询 ─→ 物理页框号
       │            │
       └──────┬─────┘
              ↓
      物理地址
```

## 1.3 多级页表（x86_64）

64 位系统使用 4 级页表（PGD → PUD → PMD → PTE），每一级索引在虚拟地址中占 9 位（共 48 位地址空间，12 位用于页内偏移）：

```
63          48 47      39 38      30 29      21 20      12 11         0
┌─────────────┬──────────┬──────────┬──────────┬──────────┬────────────┐
│  保留/符号位 │  PGD索引  │  PUD索引  │  PMD索引  │  PTE索引  │  页内偏移  │
│   (sign)    │   (9位)   │   (9位)   │   (9位)   │   (9位)   │   (12位)   │
└─────────────┴──────────┴──────────┴──────────┴──────────┴────────────┘
```

**大页面（Huge Pages）**：
- 2MB 大页（x86_64）：减少 TLB Miss 次数
- 1GB 大页（需要硬件支持）
- 透明大页（Transparent Huge Pages, THP）：内核自动尝试使用大页

# 2. 内存分配机制

## 2.1 伙伴系统（Buddy System）

内核管理物理内存的核心算法，用于分配和释放连续物理页：

**原理**：
- 将空闲页面按 2^n 大小分组（1、2、4、8、...、1024 页）
- 分配时从满足需求的最小块中分配
- 释放时检查相邻块是否空闲，合并为更大的块

```bash
# 查看伙伴系统信息
cat /proc/buddyinfo
```

## 2.2 slab 分配器

用于管理内核对象（如进程描述符、inode 等）的分配，避免频繁的伙伴系统操作：

```
slab 分配器
├── 进程描述符缓存 (task_struct)
├── inode 缓存
├── dentry 缓存
└── 其他内核对象
```

```bash
# 查看 slab 分配器状态
cat /proc/slabinfo
slabtop                   # 实时查看 slab 使用情况
```

## 2.3 用户空间内存分配

```bash
malloc() / calloc() / realloc()    # C 标准库（glibc 使用 ptmalloc）
    ↓
brk() / sbrk()                     # 堆扩展（小内存）
    ↓
mmap()                             # 内存映射（大内存，默认 > 128KB）
```

**内存分配图**：
```
进程虚拟地址空间：
┌──────────────────────┐ 高地址
│        内核空间        │ 0xFFFF...（仅内核可访问）
├──────────────────────┤
│       栈（Stack）      │ 向下增长
├──────────────────────┤
│          ↓            │
│       空闲区域         │
│          ↑            │
├──────────────────────┤
│        堆（Heap）      │ 向上增长（brk）
├──────────────────────┤
│    数据段（.data/.bss） │
├──────────────────────┤
│    代码段（.text）      │ 低地址
└──────────────────────┘
```

# 3. Swap 与内存回收

## 3.1 Swap 机制

当物理内存不足时，内核将不常用的内存页面换出到磁盘的 Swap 分区或文件：

```
物理内存已满
    ↓
页面置换算法（LRU 近似）选择被换出的页面
    ↓
将页面写入 Swap
    ↓
释放物理页面给需要者
    ↓
访问已换出页面 → 缺页异常 → 从 Swap 换入
```

## 3.2 Swap 管理

```bash
# 查看 Swap 使用
swapon --show
cat /proc/swaps
free -h

# 创建 Swap 文件
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# 持久化配置（/etc/fstab）
/swapfile none swap sw 0 0

# 调整 Swap 倾向性（swappiness）
# 范围 0-100，默认 60
# 值越低越倾向于使用物理内存，值越高越倾向于使用 Swap
cat /proc/sys/vm/swappiness
sysctl vm.swappiness=10
```

## 3.3 OOM Killer（内存不足杀手）

当系统内存严重不足时，OOM Killer 选择并杀死一个进程以释放内存：

```bash
# 查看 OOM Killer 日志
dmesg | grep -i "killed process"
journalctl -k | grep -i "oom"

# 查看进程的 OOM 评分
cat /proc/1234/oom_score       # OOM 分数（越高越容易被杀）
cat /proc/1234/oom_score_adj   # OOM 分数调整值（-1000 表示禁用）

# 调整进程的 OOM 保护
# 防止重要进程被 OOM Killer 杀掉
sudo echo -1000 > /proc/1234/oom_score_adj
```

# 4. 内存监控

## 4.1 free

```bash
free -h
#               total        used        free      shared  buff/cache   available
# Mem:           7.7G        3.1G        2.0G        156M        2.6G        4.2G
# Swap:          2.0G        200M        1.8G
```

**关键指标解读**：
| 字段 | 含义 |
|------|------|
| total | 物理内存总量 |
| used | 实际已使用的内存 |
| free | 完全空闲的内存 |
| shared | tmpfs 等共享内存 |
| buff/cache | 缓冲区+页面缓存（可以回收）|
| available | 新进程可以使用的内存（free + 可回收的 cache）|

**重要**：`buff/cache` 是正常的缓存行为，Linux 会尽量利用空闲内存做缓存。`available` 才是真正能反映内存压力的指标。

## 4.2 vmstat

```bash
vmstat 1              # 每秒刷新一次
# procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
#  r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
#  1  0 200000 2100000  50000 2400000  0    0     5     3   20   30  5  2 93  0  0
```

**字段解读**：
| 字段 | 含义 |
|------|------|
| r | 运行队列中的进程数 |
| b | 不可中断睡眠进程数 |
| swpd | 使用的 Swap 总量 |
| free | 空闲内存 |
| buff | 缓冲区 |
| cache | 页面缓存 |
| si | 从 Swap 换入（每秒）|
| so | 换出到 Swap（每秒）|
| bi | 块设备读（每秒）|
| bo | 块设备写（每秒）|

**内存压力指标**：
- `si`/`so` 持续大于 0 → 内存不足，正在大量使用 Swap
- `free` 持续减少且 `cache` 也在减少 → 内存紧张

## 4.3 /proc/meminfo

最详细的内存信息：

```bash
cat /proc/meminfo
# MemTotal:        8048760 kB
# MemFree:         2147584 kB
# MemAvailable:    4426752 kB
# Buffers:          124928 kB
# Cached:          2359296 kB
# SwapCached:        32768 kB
# Active:          3276800 kB
# Inactive:        2097152 kB
# Active(anon):    1835008 kB
# Inactive(anon):   524288 kB
# Active(file):    1441792 kB
# Inactive(file):  1572864 kB
# Unevictable:       16384 kB
# Mlocked:               0 kB
# SwapTotal:       2097152 kB
# SwapFree:        1896448 kB
# ...
```

**Dirty 页面**：
```bash
cat /proc/meminfo | grep -E "Dirty|Writeback"
# Dirty:             4096 kB      # 已修改但尚未写回磁盘
# Writeback:            0 kB      # 正在写回磁盘
```

# 5. 内存优化

## 5.1 缓存与缓冲区

**Page Cache（页面缓存）**：
- 缓存文件内容，加速文件读写
- `cat /proc/meminfo` 中的 `Cached` 字段
- 内核有 pdflush 线程在后台将脏页写回磁盘

**Buffer Cache（缓冲区缓存）**：
- 缓存块设备元数据
- `cat /proc/meminfo` 中的 `Buffers` 字段

**手动清理缓存**（仅用于测试，不建议在生产环境使用）：
```bash
# 清理 Page Cache
echo 1 > /proc/sys/vm/drop_caches

# 清理 dentries 和 inode
echo 2 > /proc/sys/vm/drop_caches

# 清理所有
echo 3 > /proc/sys/vm/drop_caches
```

## 5.2 内存相关 sysctl 参数

```bash
# 查看所有内存参数
sysctl -a | grep vm.

# 关键参数
vm.swappiness = 60         # Swap 使用倾向（0-100）
vm.vfs_cache_pressure = 100  # 回收 dentry/inode 的倾向（值越大越积极回收）
vm.dirty_ratio = 20        # 脏页占内存百分比上限（超过后同步写）
vm.dirty_background_ratio = 10  # 脏页百分比后台开始写回
vm.min_free_kbytes = 65536 # 保留空闲内存下限
vm.overcommit_memory = 0   # 内存过载策略（0=启发式，1=总是允许，2=禁止超过限制）
vm.overcommit_ratio = 50   # overcommit=2 时允许的百分比
```
