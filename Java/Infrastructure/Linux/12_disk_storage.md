
# 1. 磁盘基础知识

## 1.1 磁盘类型

| 类型 | 介质 | 特性 | 用途 |
|------|------|------|------|
| HDD | 磁碟片 | 大容量、低价格、随机读写慢 | 数据存储、冷数据 |
| SSD | NAND Flash | 读写快、抗震、寿命有限 | 系统盘、热数据 |
| NVMe | PCIe SSD | 极快速度、低延迟 | 高性能场景 |
| 虚拟磁盘 | 内存/文件 | 云环境、虚拟机 | 弹性存储 |

## 1.2 接口与协议

```
SATA      → AHCI 协议     → 传统 HDD/SSD
SAS       → SCSI 协议     → 企业级磁盘
NVMe      → PCIe 协议     → 高性能 SSD
```

## 1.3 磁盘命名规则（Linux）

```
传统命名：
/dev/sda      # 第一块 SCSI/SATA 磁盘
/dev/sdb      # 第二块
/dev/sda1     # 第一块磁盘的第一个分区
/dev/sda2     # 第一块磁盘的第二个分区

NVMe 命名：
/dev/nvme0n1  # 第一个 NVMe 控制器下的第一个命名空间
/dev/nvme0n1p1  # 第一个分区

虚拟磁盘：
/dev/vda      # 虚拟化环境中的磁盘（KVM/VirtIO）
/dev/xvda     # Xen 环境
/dev/mmcblk0  # eMMC/SD 卡
```

# 2. 磁盘查看与识别

## 2.1 列出磁盘信息

```bash
lsblk                         # 树状列出所有块设备
lsblk -f                      # 显示文件系统类型和 UUID
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,FSTYPE,UUID  # 自定义输出

fdisk -l                      # 列出分区表
fdisk -l /dev/sda             # 查看指定磁盘

parted -l                     # 查看分区信息（支持 GPT）
blkid                         # 查看块设备属性（UUID、文件系统类型）
```

## 2.2 lsblk 输出解读

```bash
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda           8:0    0 238.5G  0  disk
├─sda1        8:1    0   512M  0  part /boot/efi
├─sda2        8:2    0   488M  0  part /boot
└─sda3        8:3    0 237.5G  0  part
  ├─vg-root 253:0    0   100G  0  lvm  /
  ├─vg-swap 253:1    0     4G  0  lvm  [SWAP]
  └─vg-home 253:2    0 133.5G  0  lvm  /home
```

## 2.3 磁盘性能测试

```bash
# 顺序读写测试
sudo hdparm -t /dev/sda       # 读取缓存性能
sudo hdparm -T /dev/sda       # 读取缓存性能（非直接磁盘）

# dd 测速
dd if=/dev/zero of=./test bs=1M count=1024 conv=fdatasync  # 写入速度
dd if=./test of=/dev/null bs=1M count=1024                  # 读取速度

# 更专业的工具
sudo apt install fio
fio --name=test --ioengine=libaio --rw=randread --bs=4k --numjobs=4 --size=1G
```

# 3. 分区管理

## 3.1 MBR vs GPT

| 特性 | MBR | GPT |
|------|-----|-----|
| 最大磁盘容量 | 2TB | 9.4ZB（2^64 扇区 × 512B）|
| 最大分区数 | 4 主分区（或 3 主+扩展） | 128 默认（可更多）|
| 引导方式 | Legacy BIOS | UEFI |
| 冗余 | 无 | GPT 头和分区表在磁盘尾部有备份 |

## 3.2 fdisk（MBR 和 GPT）

```bash
# 交互式分区
sudo fdisk /dev/sdb
# 常用交互命令：
# n - 新建分区
# d - 删除分区
# p - 打印分区表
# w - 写入并退出
# q - 不保存退出
```

**fdisk 创建分区的典型步骤**：
```
fdisk /dev/sdb
  → g    (创建 GPT 分区表)
  → n    (新建分区)
  → 默认分区号 1
  → 默认起始扇区
  → +100G  (100GB 大小)
  → n    (新建第二个分区)
  → +50G
  → w    (写入)
```

## 3.3 parted — 更强大的分区工具

```bash
# 使用 parted 创建分区
sudo parted /dev/sdb mklabel gpt                          # 创建 GPT 分区表
sudo parted /dev/sdb mkpart primary ext4 1MiB 100GiB      # 创建分区
sudo parted /dev/sdb mkpart primary ext4 100GiB 150GiB    # 第二个分区

# 查看分区表
sudo parted /dev/sdb print

# 删除分区
sudo parted /dev/sdb rm 1
```

# 4. 创建文件系统

## 4.1 常用文件系统类型

```bash
# 创建 ext4 文件系统（Linux 默认）
sudo mkfs.ext4 /dev/sdb1
sudo mkfs.ext4 -L "mydata" /dev/sdb1    # 指定卷标

# 创建 XFS
sudo mkfs.xfs /dev/sdb2
sudo mkfs.xfs -L "mydata" /dev/sdb2

# 创建 Btrfs
sudo mkfs.btrfs -L "mydata" /dev/sdb1

# 创建交换分区
sudo mkswap /dev/sdb3
sudo swapon /dev/sdb3
```

## 4.2 mkfs 常用选项

| 选项 | ext4 示例 | 说明 |
|------|-----------|------|
| -L | `-L mylabel` | 卷标 |
| -b | `-b 4096` | 块大小 |
| -i | `-i 16384` | 每多少字节分配一个 inode |
| -m | `-m 0.5` | 保留块百分比（默认 5%）|
| -O | `-O ^has_journal` | 启用/禁用特定功能 |

## 4.3 查看文件系统信息

```bash
# 查看文件系统详细信息
dumpe2fs /dev/sdb1            # ext4 文件系统详情
xfs_info /dev/sdb2            # XFS 信息

# 检查文件系统
sudo fsck /dev/sdb1           # 文件系统一致性检查
sudo e2fsck -f /dev/sdb1      # 强制检查 ext4
sudo xfs_repair /dev/sdb2     # XFS 修复
```

**注意**：检查前必须先卸载文件系统。

# 5. 挂载与卸载

## 5.1 mount 命令

```bash
# 基本挂载
sudo mount /dev/sdb1 /mnt/data

# 指定文件系统类型
sudo mount -t ext4 /dev/sdb1 /mnt/data

# 挂载选项
sudo mount -o rw,noexec,nosuid,nodev /dev/sdb1 /mnt/data

# 挂载 ISO 镜像
sudo mount -o loop image.iso /mnt/iso

# 绑定挂载（使一个目录出现在另一个挂载点）
sudo mount --bind /some/dir /mnt/target
```

## 5.2 常用挂载选项

| 选项 | 作用 |
|------|------|
| `rw` / `ro` | 读写/只读 |
| `noexec` | 不允许在该文件系统上执行程序 |
| `nosuid` | 忽略 SUID/SGID 位 |
| `nodev` | 不解析设备文件 |
| `noatime` | 不更新访问时间（提升性能）|
| `relatime` | 仅在 mtime 更新时更新 atime（默认）|
| `sync` | 同步 I/O（数据立即写入磁盘）|
| `async` | 异步 I/O（默认）|
| `defaults` | rw, suid, dev, exec, auto, nouser, async |

## 5.3 umount

```bash
sudo umount /mnt/data              # 卸载（通过挂载点）
sudo umount /dev/sdb1              # 卸载（通过设备）

# 如果设备忙
lsof /mnt/data                     # 查看哪些进程在使用
fuser -km /mnt/data                # 终止使用该挂载点的所有进程
sudo umount -l /mnt/data           # 延迟卸载（lazy unmount）
```

## 5.4 /etc/fstab — 自动挂载配置

```bash
# /etc/fstab 格式
# <file system>  <mount point>  <type>  <options>  <dump>  <pass>

# 使用设备路径（不推荐，设备名会变）
/dev/sdb1       /mnt/data      ext4    defaults    0       2

# 使用 UUID（推荐）
UUID=abc12345-...  /mnt/data   ext4    defaults    0       2

# 使用卷标（LABEL）
LABEL=mydata    /mnt/data      ext4    defaults    0       2

# 挂载后执行 mount -a 立即生效
sudo mount -a
```

**字段说明**：

| 字段 | 含义 |
|------|------|
| file system | 设备、UUID 或 LABEL |
| mount point | 挂载点目录 |
| type | 文件系统类型 |
| options | 挂载选项（逗号分隔）|
| dump | 是否备份（0=不备份，1=备份）|
| pass | fsck 检查顺序（0=不检查，1=根文件系统，2=其他）|

# 6. 查看磁盘使用

## 6.1 df — 磁盘空间概况

```bash
df -h                         # 人类可读格式
df -T                         # 显示文件系统类型
df -i                         # 显示 inode 使用情况
df -h /home                   # 查看特定目录所在分区的使用
```

## 6.2 du — 目录空间详情

```bash
du -sh /home/alice            # 统计目录总大小
du -h --max-depth=1 /home     # 只看一级子目录
du -a /path | sort -rh        # 按大小排序所有文件
du -ch *.log                  # 统计多个文件总大小

# 查找大于 100MB 的文件
find / -type f -size +100M -exec du -sh {} \; | sort -rh
```

# 7. I/O 调度

## 7.1 I/O 调度器

Linux 内核 I/O 调度器决定磁盘请求的顺序：

```bash
# 查看当前调度器
cat /sys/block/sda/queue/scheduler
# [mq-deadline] kyber bfq none

# 修改调度器
echo bfq > /sys/block/sda/queue/scheduler
```

| 调度器 | 特点 | 适用场景 |
|--------|------|----------|
| mq-deadline | 保证每个请求的最大延迟 | 通用场景 |
| BFQ | 公平排队，考虑进程优先级 | 桌面/交互式系统 |
| Kyber | 基于延迟的自适应调度 | 快速设备（NVMe）|
| none | 不调度（由设备自行管理） | NVMe 等高性能设备 |

## 7.2 I/O 监控

```bash
# iostat
iostat -x 1                  # 扩展统计，每秒刷新
# %util 表示磁盘繁忙程度（但非压测时可能不准）
# await 表示平均 I/O 等待时间（ms）
# r/s, w/s 表示每秒读写次数

# iotop（类似 top 的 I/O 监控）
sudo iotop
sudo iotop -o                # 只显示有 I/O 的进程

# pidstat I/O 统计
pidstat -d 1                 # 查看每个进程的 I/O
```
