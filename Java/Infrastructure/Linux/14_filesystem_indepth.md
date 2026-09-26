
# 1. 虚拟文件系统（VFS）

## 1.1 VFS 架构

VFS（Virtual File System）是 Linux 内核中的抽象层，使得用户空间的程序可以用统一的系统调用接口操作不同类型的文件系统：

```
User Space (read/write/open/close)
        ↓ 系统调用
VFS Layer (vfs_read / vfs_write / vfs_open)
        ↓ 具体文件系统操作
ext4 / xfs / btrfs / tmpfs / ...
        ↓
Block Layer (通用块层)
        ↓
Device Driver
        ↓
Hardware
```

## 1.2 VFS 核心对象

| 对象 | 结构体 | 说明 |
|------|--------|------|
| super_block | struct super_block | 已挂载文件系统的实例信息 |
| inode | struct inode | 文件的元数据（不包含文件名）|
| dentry | struct dentry | 目录项，连接文件名与 inode |
| file | struct file | 已打开文件的上下文 |

**inode 和 dentry 的关系**：

```
用户看到：/home/alice/file.txt
                ↓
dentry 链："/" → "home" → "alice" → "file.txt"
                ↓                    ↓
         inode A (目录)         inode B (文件)
```

## 1.3 dentry 缓存（dcache）

- 缓存最近使用的 dentry，避免频繁从磁盘读取目录结构
- `/proc/meminfo` 中的 `Slab` 包含 dcache 占用
- 通过 `slabtop` 查看 dentry 缓存使用

# 2. inode 深度解析

## 2.1 inode 结构

inode（Index Node）是 VFS 中描述文件元数据的核心结构，包含：

```c
struct inode {
    umode_t          i_mode;       // 文件类型和权限
    uid_t            i_uid;        // 所有者 UID
    gid_t            i_gid;        // 所属组 GID
    loff_t           i_size;       // 文件大小（字节）
    struct timespec  i_atime;      // 最后访问时间
    struct timespec  i_mtime;      // 最后修改时间
    struct timespec  i_ctime;      // 最后状态改变时间
    unsigned long    i_nlink;      // 硬链接数
    unsigned int     i_blkbits;    // 块大小（bit 位数）
    blkcnt_t         i_blocks;     // 文件占用的块数（512B 为单位）
    umode_t          i_flags;      // 文件属性标志
    // 数据块指针数组
    // ext4 使用 15 个指针：12 个直接 + 1 个间接 + 1 个二级间接 + 1 个三级间接
};
```

## 2.2 ext4 数据块寻址

```
inode 数据块指针（15 个条目，每个 4 字节）：
┌────────────┐
│ block[0]   │ ──→ 直接数据块（12KB）
│ block[1]   │ ──→ 直接数据块
│ ...        │
│ block[11]  │ ──→ 直接数据块
├────────────┤
│ block[12]  │ ──→ 间接块（指向 1024 个块 = 4MB）
├────────────┤
│ block[13]  │ ──→ 二级间接块（1024² = 4GB）
├────────────┤
│ block[14]  │ ──→ 三级间接块（1024³ = 4TB）
└────────────┘
```

（块大小 4096 字节，每个指针 4 字节，一个块可存放 1024 个指针）

**查看 inode 信息**：
```bash
stat file.txt
df -i /dev/sda1               # 查看分区 inode 使用情况
ls -li file.txt               # -i 显示 inode 号
```

**inode 耗尽问题**：
- 即使磁盘还有空闲空间，inode 耗尽后也无法创建新文件
- 大文件系统通常有足够的 inode（每 16KB 空间一个 inode）

# 3. 主流文件系统对比

## 3.1 ext4

Linux 默认文件系统，ext3 的演进版本：

**特性**：
- 最大文件：16TB（4K 块时）
- 最大文件系统：1EB
- 支持 extents（范围映射，取代间接块映射）
- 延迟分配（延迟写入减少碎片）
- 在线 defragmentation（碎片整理）
- 日志校验和
- 支持纳秒时间戳

**适用场景**：通用的默认选择，适合大多数用途。

**操作**：
```bash
# 创建 ext4
mkfs.ext4 /dev/sdb1
mkfs.ext4 -O ^has_journal /dev/sdb1  # 创建不带日志的 ext4

# 查看 ext4 信息
dumpe2fs /dev/sdb1 | grep -E "Block count|Inode count|Block size"

# 调整 reserved blocks
tune2fs -m 0.5 /dev/sdb1       # 保留块从 5% 降到 0.5%

# 强制磁盘检查
fsck.ext4 -f /dev/sdb1

# 碎片整理
e4defrag /path/to/file
```

## 3.2 XFS

SGI 开发的高性能 64 位日志文件系统，RHEL 7+ 默认：

**特性**：
- 最大文件：8EB
- 最大文件系统：8EB
- 分配组（Allocation Groups）—— 并行分配和 I/O
- 延迟分配 + 预分配
- 支持在线扩展（不可缩小）
- 元数据校验和（v5 特性）
- **不支持缩小和快照**

**适用场景**：大文件、高性能计算、数据库、视频编辑。

**操作**：
```bash
# 创建 XFS
mkfs.xfs /dev/sdb1
mkfs.xfs -f -m reflink=1 -c size=1g /dev/sdb1  # 启用 reflink，日志 1G

# 查看 XFS 信息
xfs_info /dev/sdb1

# 扩展（只能增大）
xfs_growfs /mount/point       # 使用挂载点

# 检查修复（先卸载）
xfs_repair /dev/sdb1
```

## 3.3 Btrfs（B-tree FS）

COW（Copy-on-Write）文件系统，号称 Linux 的 ZFS：

**特性**：
- 写时复制（COW）
- 快照（子卷快照，瞬间完成）
- 压缩（zlib、lzo、zstd）
- 数据校验和（检测静默数据损坏）
- RAID 支持（0/1/5/6/10，内置）
- 在线扩展/缩小
- 支持 reflink 和 dedup

**适用场景**：需要快照、压缩、数据完整性保护的场景。

**操作**：
```bash
# 创建 Btrfs
mkfs.btrfs -L "mydata" /dev/sdb1 /dev/sdc1  # 多设备

# 创建子卷
btrfs subvolume create /mnt/data/@home
btrfs subvolume create /mnt/data/@logs

# 创建快照
btrfs subvolume snapshot /mnt/data/@home /mnt/data/@home_snap

# 启用压缩
mount -o compress=zstd /dev/sdb1 /mnt/data
```

## 3.4 其他文件系统

| 文件系统 | 特点 | 适用场景 |
|----------|------|----------|
| tmpfs | 基于内存/交换的临时 FS | /tmp、/dev/shm |
| ZFS | OpenZFS，强大但许可问题 | 需要高级数据管理 |
| F2FS | Flash-Friendly，针对闪存优化 | SD 卡、SSD |
| OverlayFS | 联合挂载，多层叠加 | Docker 容器层 |
| SquashFS | 只读压缩文件系统 | Live CD、嵌入式 |

## 3.5 性能对比速览

| 指标 | ext4 | XFS | Btrfs |
|------|------|-----|-------|
| 大文件读写 | 好 | 优秀 | 好 |
| 小文件读写 | 优秀 | 好 | 好 |
| CPU 开销 | 低 | 中 | 中高 |
| 碎片敏感度 | 中 | 低 | 低 |
| 元数据性能 | 好 | 好 | 中 |

# 4. 日志机制

## 4.1 日志的作用

文件系统操作（如写文件）通常涉及多个步骤，系统崩溃可能导致不一致。日志（Journal）通过预写日志来解决：

```
写操作步骤：
1. 更新 inode（记录新大小）
2. 更新数据块
3. 更新目录条目

系统崩溃时：可能只完成 1 和 2，inode 指向错误数据 → 不一致！

日志模式：
1. 先将操作写入日志
2. 再执行实际操作
3. 崩溃后回放日志恢复一致性
```

## 4.2 日志模式

```bash
# ext4 日志模式
mount -o data=ordered /dev/sdb1 /mnt    # 默认：元数据日志 + 数据有序
mount -o data=journal /dev/sdb1 /mnt    # 完全日志：数据和元数据都记录
mount -o data=writeback /dev/sdb1 /mnt  # 仅元数据日志（最快，但可能数据错乱）
```

| 模式 | 一致性保证 | 性能影响 | 风险 |
|------|-----------|----------|------|
| ordered | 元数据一致 | 中 | 无 |
| journal | 数据和元数据一致 | 最大（双写）| 无 |
| writeback | 元数据一致 | 最小 | 崩溃后文件内容可能错误 |

**XFS 没有日志模式选择**——它始终以类似 ordered 的方式运行。

# 5. 挂载选项优化

针对不同场景的挂载优化：

```bash
# 桌面/文件服务器
defaults,noatime,errors=remount-ro

# 数据库服务器（降低 fsync 延迟）
defaults,noatime,nobarrier

# SSD 优化
defaults,noatime,discard,nodiratime

# 邮件/NFS 服务器
defaults,noatime,data=writeback,barrier=0

# USB 闪存盘
defaults,noatime,sync  # 或 async 权衡速度与安全
```
