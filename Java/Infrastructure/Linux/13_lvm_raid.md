
# 1. LVM 概述

## 1.1 什么是 LVM

LVM（Logical Volume Manager，逻辑卷管理器）是 Linux 内核提供的逻辑卷管理机制，在物理磁盘和文件系统之间增加一层抽象，使存储空间的管理更加灵活。

**传统分区 vs LVM**：
```
传统分区：
[磁盘] → [分区] → [文件系统]（大小固定，调整困难）

LVM：
[磁盘/分区] → [PV] → [VG] → [LV] → [文件系统]
（灵活调整、快照、在线扩展）
```

## 1.2 核心概念

```
PV（Physical Volume，物理卷）
  └─ 物理磁盘、分区或 RAID 阵列

VG（Volume Group，卷组）
  └─ 多个 PV 组成的存储池

LV（Logical Volume，逻辑卷）
  └─ 从 VG 中划分的逻辑块设备
```

## 1.3 LVM 优势

| 特性 | 说明 |
|------|------|
| 在线扩展 | 无需卸载即可扩大 LV |
| 在线缩小 | 部分文件系统支持在线缩小（需特定 FS）|
| 快照 | 创建时间点一致的备份 |
| 跨磁盘 | 一个 LV 可横跨多个物理磁盘 |
| 迁移 | 在线迁移 PV 数据 |

# 2. LVM 操作

## 2.1 准备物理卷（PV）

```bash
# 创建 PV
sudo pvcreate /dev/sdb1
sudo pvcreate /dev/sdb2
sudo pvcreate /dev/sdc

# 查看 PV 信息
sudo pvs                     # 简要信息
sudo pvdisplay /dev/sdb1     # 详细信息
```

## 2.2 创建卷组（VG）

```bash
# 创建 VG
sudo vgcreate vg_data /dev/sdb1 /dev/sdb2

# 向 VG 添加更多 PV
sudo vgextend vg_data /dev/sdc

# 从 VG 移除 PV
sudo vgreduce vg_data /dev/sdb2

# 查看 VG 信息
sudo vgs
sudo vgdisplay vg_data
```

## 2.3 创建逻辑卷（LV）

```bash
# 创建 LV
sudo lvcreate -n lv_home -L 100G vg_data
sudo lvcreate -n lv_logs -L 50G vg_data
sudo lvcreate -n lv_remaining -l 100%FREE vg_data  # 使用所有剩余空间

# LV 设备路径
# /dev/vg_data/lv_home
# /dev/mapper/vg_data-lv_home

# 查看 LV
sudo lvs
sudo lvdisplay /dev/vg_data/lv_home
```

## 2.4 创建文件系统并挂载

```bash
# 在 LV 上创建文件系统
sudo mkfs.ext4 /dev/vg_data/lv_home
sudo mkfs.xfs /dev/vg_data/lv_logs

# 挂载
sudo mkdir -p /home
sudo mount /dev/vg_data/lv_home /home

# 写入 /etc/fstab，使用 UUID
blkid /dev/vg_data/lv_home
# UUID=...  /home  ext4  defaults  0  2
```

## 2.5 扩展逻辑卷

**ext4 文件系统扩展**：
```bash
# 1. 扩展 LV
sudo lvextend -L +20G /dev/vg_data/lv_home    # 增加 20G
sudo lvextend -L 150G /dev/vg_data/lv_home    # 扩展到 150G
sudo lvextend -l +100%FREE /dev/vg_data/lv_home  # 使用所有剩余空间

# 2. 扩展文件系统
sudo resize2fs /dev/vg_data/lv_home            # ext4 在线扩展
```

**XFS 文件系统扩展**：
```bash
# 1. 扩展 LV（同上）
sudo lvextend -L +20G /dev/vg_data/lv_logs

# 2. 扩展 XFS（必须使用 xfs_growfs）
sudo xfs_growfs /var/log                       # XFS 使用挂载点而非设备
```

## 2.6 缩小逻辑卷（ext4）

```bash
# 1. 卸载文件系统
sudo umount /home

# 2. 检查文件系统
sudo e2fsck -f /dev/vg_data/lv_home

# 3. 缩小文件系统（先缩小 FS，再缩小 LV）
sudo resize2fs /dev/vg_data/lv_home 100G

# 4. 缩小 LV
sudo lvreduce -L 100G /dev/vg_data/lv_home

# 5. 重新挂载
sudo mount /home
```

**注意**：XFS 不支持缩小！

## 2.7 LVM 快照

```bash
# 创建快照（需要 VG 中有空闲空间）
sudo lvcreate -n lv_home_snapshot -L 10G -s /dev/vg_data/lv_home

# 挂载快照（只读备份）
sudo mkdir /mnt/snapshot
sudo mount -o ro /dev/vg_data/lv_home_snapshot /mnt/snapshot

# 从快照恢复
sudo umount /home
sudo lvconvert --merge /dev/vg_data/lv_home_snapshot

# 删除快照
sudo lvremove /dev/vg_data/lv_home_snapshot
```

**快照原理**：COW（Copy-on-Write）—— 快照创建后，原 LV 的修改在写入前将原始数据复制到快照区，因此快照保留了创建时刻的数据。

## 2.8 移动 PV 数据

```bash
# 将 PV 数据迁移到其他 PV
sudo pvmove /dev/sdb1              # 将 sdb1 上的数据移到 VG 中的其他 PV
sudo pvmove /dev/sdb1 /dev/sdc     # 指定目标 PV

# 从 VG 中移除 PV
sudo vgreduce vg_data /dev/sdb1
```

# 3. RAID 概述

## 3.1 什么是 RAID

RAID（Redundant Array of Independent Disks）将多块磁盘组合成一个逻辑单元，以提升性能或提供冗余。

## 3.2 RAID 级别

| 级别 | 最少磁盘 | 容量利用率 | 读性能 | 写性能 | 容错 |
|------|----------|-----------|--------|--------|------|
| RAID 0 | 2 | 100% | 高（并行） | 高（并行） | 无（任何盘损全毁）|
| RAID 1 | 2 | 50% | 高（镜像读取） | 中（双写） | N-1 块盘 |
| RAID 5 | 3 | (N-1)/N | 高 | 中（奇偶校验） | 1 块盘 |
| RAID 6 | 4 | (N-2)/N | 高 | 低（双奇偶校验） | 2 块盘 |
| RAID 10 | 4（偶数） | 50% | 高 | 中 | 每个镜像组 1 块 |

## 3.3 软 RAID vs 硬 RAID

| 特性 | 软 RAID（mdadm） | 硬 RAID |
|------|------------------|---------|
| 实现位置 | 操作系统（内核） | 专用 RAID 卡 |
| 性能 | 消耗 CPU | 不消耗 CPU |
| 成本 | 免费 | 需要硬件 |
| 配置 | 灵活 | 取决于卡型号 |
| 缓存 | 系统内存 | RAID 卡缓存（带电池保护）|

# 4. 软 RAID 操作（mdadm）

## 4.1 创建 RAID

```bash
# 安装 mdadm
sudo apt install mdadm

# RAID 0（条带化）
sudo mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sdb /dev/sdc

# RAID 1（镜像）
sudo mdadm --create /dev/md1 --level=1 --raid-devices=2 /dev/sdb /dev/sdc

# RAID 5（条带+奇偶校验）
sudo mdadm --create /dev/md0 --level=5 --raid-devices=3 /dev/sdb /dev/sdc /dev/sdd

# RAID 10
sudo mdadm --create /dev/md0 --level=10 --raid-devices=4 /dev/sdb /dev/sdc /dev/sdd /dev/sde

# 创建文件系统
sudo mkfs.ext4 /dev/md0
```

## 4.2 查看 RAID 状态

```bash
# 详细信息
cat /proc/mdstat
# Personalities : [raid1] [raid6] [raid5] [raid4]
# md0 : active raid1 sdb[0] sdc[1]
#       976762584 blocks super 1.2 [2/2] [UU]

sudo mdadm --detail /dev/md0
```

## 4.3 模拟故障与恢复

```bash
# 模拟磁盘故障
sudo mdadm /dev/md0 --fail /dev/sdb
cat /proc/mdstat
# md0 : active raid1 sdc[1] sdb[2](F)

# 移除故障盘
sudo mdadm /dev/md0 --remove /dev/sdb

# 添加新盘
sudo mdadm /dev/md0 --add /dev/sdd

# 等待重建完成
cat /proc/mdstat
# md0 : active raid1 sdd[2] sdc[1]
#       [2/2] [UU]
```

## 4.4 RAID 配置持久化

```bash
# 保存 RAID 配置
sudo mdadm --detail --scan >> /etc/mdadm/mdadm.conf

# 更新 initramfs
sudo update-initramfs -u
```

# 5. LVM + RAID 结合

在实践中，LVM 和 RAID 常结合使用——先用 mdadm 做 RAID 提供冗余，再在 RAID 设备上创建 LVM 提供灵活性：

```bash
# 1. 创建 RAID
sudo mdadm --create /dev/md0 --level=10 --raid-devices=4 /dev/sd[bcde]

# 2. 在 RAID 上创建 LVM
sudo pvcreate /dev/md0
sudo vgcreate vg_production /dev/md0
sudo lvcreate -n lv_apps -L 500G vg_production
sudo lvcreate -n lv_data -l 100%FREE vg_production

# 3. 文件系统
sudo mkfs.ext4 /dev/vg_production/lv_apps
sudo mkfs.xfs /dev/vg_production/lv_data
```

# 6. 选择建议

| 场景 | 推荐方案 |
|------|----------|
| 系统盘（需要高可靠性）| RAID 1 或 RAID 1 上建 LVM |
| 数据库（高性能+可靠性）| RAID 10 + LVM 快照备份 |
| 大容量存储 | RAID 5/6 |
| 缓存/临时数据 | RAID 0 或不做 RAID |
| 云环境（无法用软 RAID）| LVM 条带化或直接使用云盘 |
