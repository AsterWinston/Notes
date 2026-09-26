
# 1. 目录操作

## 1.1 基本目录命令

```bash
pwd                     # 打印当前工作目录（Print Working Directory）
ls [选项] [路径]        # 列出目录内容
cd [目录]               # 切换目录（Change Directory）
mkdir [选项] 目录名      # 创建目录（Make Directory）
rmdir 目录名             # 删除空目录（Remove Directory）
tree [路径]             # 以树状图显示目录结构
```

## 1.2 ls 详解

```bash
ls -l           # 长格式显示（权限、链接数、所有者、大小、时间、名称）
ls -a           # 显示所有文件（包括以 . 开头的隐藏文件）
ls -la          # 组合使用
ls -lh          # 以人类可读格式显示大小（KB/MB/GB）
ls -lt          # 按修改时间排序（最新在前）
ls -ltr         # 按修改时间逆序（最旧在前）
ls -lS          # 按文件大小排序
ls -R           # 递归列出子目录
ls -d */        # 仅列出目录
```

## 1.3 mkdir 详解

```bash
mkdir dir                     # 创建单个目录
mkdir -p parent/child         # 递归创建（父目录不存在时自动创建）
mkdir -m 755 secure_dir       # 创建时指定权限模式
mkdir -p a/b/c/d/e            # 创建深层嵌套目录
```

# 2. 文件操作

## 2.1 创建文件

```bash
touch file.txt                    # 创建空文件或更新文件时间戳
echo "content" > file.txt         # 创建并写入内容（覆盖）
echo "content" >> file.txt        # 追加内容
cat > file.txt << EOF             # 多行输入创建文件
> line1
> line2
> EOF
```

## 2.2 复制（cp）

```bash
cp source.txt dest.txt                  # 复制文件
cp -i source.txt dest.txt               # 交互式（覆盖前提示）
cp -r sourcedir/ destdir/               # 递归复制目录
cp -a sourcedir/ destdir/               # 归档复制（保留权限、时间戳、符号链接）
cp -u source.txt dest.txt               # 仅当源文件更新时复制（update）
cp -v source.txt dest.txt               # 显示复制过程（verbose）
cp file{1,2,3}.txt dest/               # 花括号展开同时复制多个文件
```

## 2.3 移动与重命名（mv）

```bash
mv source.txt dest.txt          # 重命名文件
mv file.txt dir/                # 移动到目录
mv dir1/ dir2/                  # 重命名或移动目录
mv -i file.txt dest.txt         # 交互式（覆盖前提示）
mv -u source.txt dest.txt       # 仅当源文件更新时移动
mv -v file.txt dest.txt         # 显示过程
```

**注意**：Linux 没有专门的重命名命令，`mv` 兼具移动和重命名功能。对于批量重命名，可使用 `rename` 命令：

```bash
# 将所有 .txt 后缀改为 .md
rename 's/\.txt$/.md/' *.txt

# Debian/Ubuntu 版本语法
rename .txt .md *.txt
```

## 2.4 删除（rm）

```bash
rm file.txt                 # 删除文件（直接删除，不经过回收站）
rm -i file.txt              # 交互式确认
rm -f file.txt              # 强制删除（不提示，忽略不存在的文件）
rm -r dir/                  # 递归删除目录及内容
rm -rf dir/                 # 强制递归删除（危险操作！）
rm -v file.txt              # 显示删除过程
```

**安全警示**：
- `rm -rf /` 或 `rm -rf .` 是极其危险的操作
- 建议在重要操作前使用 `ls` 确认路径
- 使用别名（alias）增加保护：
  ```bash
  alias rm='rm -i'
  alias rm='rm -I'   # 删除超过3个文件或递归删除时才提示
  ```

# 3. 查看文件与目录信息

## 3.1 文件详细信息解读（ls -l）

```
-rw-r--r--  1  alice  staff  1024  Jan 15 10:30  file.txt
^----------  ^  ^      ^      ^     ^            ^
|            |  |      |      |     |           文件名
|            |  |      |      |     最后修改时间
|            |  |      |      文件大小（字节）
|            |  |      组名（group）
|            |      所有者（owner）
|            |  硬链接数
权限位（10个字符）
```

权限位详解：
```
-rw-r--r--
^ ^^^ ^^^ ^^^
| |   |   |
| |   |   其他人的权限（other）: r--
| |   所属组的权限（group）: r--
| 文件所有者的权限（user）: rw-
文件类型：- 普通文件 | d 目录 | l 符号链接 | b 块设备 | c 字符设备 | s 套接字 | p 管道
```

## 3.2 file 命令 — 确定文件类型

```bash
file file.txt           # 输出：ASCII text
file script.sh          # 输出：Bourne-Again shell script
file /bin/ls            # 输出：ELF 64-bit LSB executable
file photo.jpg          # 输出：JPEG image data
file unknown.bin        # 输出：data（未知格式）
```

`file` 通过检查文件**魔数（magic number）** 而非扩展名来判断文件类型。

## 3.3 stat 命令 — 详细元数据

```bash
stat file.txt
#   File: file.txt
#   Size: 1024       Blocks: 8          IO Block: 4096   regular file
# Device: 801h/2049d Inode: 1234567     Links: 1
# Access: (0644/-rw-r--r--)  Uid: ( 1000/   alice)   Gid: ( 1000/   staff)
# Access: 2024-01-15 10:00:00.000000000 +0800
# Modify: 2024-01-15 10:30:00.000000000 +0800
# Change: 2024-01-15 10:30:00.000000000 +0800
```

**三个时间戳**：
| 字段 | 含义 | 触发条件 |
|------|------|----------|
| Access (atime) | 最后访问时间 | cat/less/read |
| Modify (mtime) | 最后修改时间（内容） | echo/vim 写入 |
| Change (ctime) | 最后状态改变时间 | chmod/chown/mv |

# 4. 查找文件

## 4.1 find — 强大但复杂

```bash
# 基本查找
find /path -name "file.txt"              # 按名称精确查找
find /path -name "*.log"                 # 通配符查找
find /path -iname "file.txt"             # 忽略大小写

# 按类型
find /path -type f                       # 只找普通文件
find /path -type d                       # 只找目录
find /path -type l                       # 只找符号链接

# 按大小
find /path -size +100M                   # 大于 100MB
find /path -size -10k                    # 小于 10KB
find /path -size 1G                      # 正好 1GB

# 按时间
find /path -mtime -7                     # 7天内修改过的文件
find /path -mtime +30                    # 超过30天未修改
find /path -atime -1                     # 1天内访问过的文件
find /path -mmin -60                     # 60分钟内修改过

# 按权限
find /path -perm 644                     # 权限精确匹配
find /path -perm -4000                   # 设置 SUID 的文件
find /path -perm /u+s                    # 同上，另一种写法

# 组合条件
find /path -name "*.log" -size +100M     # AND（默认）
find /path -name "*.log" -o -name "*.tmp" # OR
find /path -not -name "*.txt"            # NOT

# 对结果执行操作
find /path -name "*.tmp" -delete                 # 删除找到的文件
find /path -name "*.log" -exec rm {} \;          # 对每个文件执行命令
find /path -name "*.log" -exec rm {} +           # 批量执行（一次调用）
find /path -name "*.jpg" -exec cp {} /backup/ \; # 复制到备份目录

# 高级用法
find . -type f -name "*.php" | xargs grep "password"   # 配合 xargs
find . -type f -newer reference.txt                      # 比某文件更新的文件
```

## 4.2 locate — 快速查找

基于预构建的数据库，速度远快于 find，但结果可能不是实时的：

```bash
locate file.txt                   # 根据数据库查找
locate -i file.txt                # 忽略大小写
locate -c "*.conf"                # 只统计数量

# 更新数据库
sudo updatedb
```

**原理**：`updatedb` 定期扫描文件系统建立索引数据库（通常由 cron 每日执行），`locate` 直接查询数据库。

## 4.3 which 和 whereis — 定位程序

```bash
which python          # 找出命令的完整路径（从 PATH 中查找）
whereis ls            # 查找二进制、源码、man 页
type ls               # 查看命令类型（别名、内置命令、外部命令）
```

# 5. 链接文件

## 5.1 硬链接（Hard Link）

```bash
ln original.txt hardlink.txt       # 创建硬链接
```

**特性**：
- 与原文件共享同一个 inode（使用相同的磁盘数据块）
- 硬链接没有独立于原文件的概念——删除原文件不影响链接
- 仅删除所有链接后，数据块才被释放
- **不能跨文件系统**（inode 表是文件系统本地的）
- **不能链接目录**

```bash
ls -li                     # -i 显示 inode 号
# 1234567 -rw-r--r-- 2 alice staff 1024 Jan 15 10:30 original.txt
# 1234567 -rw-r--r-- 2 alice staff 1024 Jan 15 10:30 hardlink.txt
```

## 5.2 软链接/符号链接（Symbolic Link）

```bash
ln -s original.txt symlink.txt     # 创建软链接
```

**特性**：
- 是一个指向原文件路径的特殊文件（有自己的 inode）
- 原文件删除后，软链接变成**断裂（broken）** 状态
- 可以**跨文件系统**
- 可以**链接目录**
- 类似于 Windows 的快捷方式但更底层

```bash
ls -li
# 1234567 -rw-r--r-- 1 alice staff 1024 Jan 15 10:30 original.txt
# 1234568 lrwxrwxrwx 1 alice staff   12 Jan 15 10:31 symlink.txt -> original.txt
```

## 5.3 对比总结

| 特性 | 硬链接 | 软链接 |
|------|--------|--------|
| inode | 与原文件相同 | 独立的 inode |
| 删除原文件 | 链接仍然有效 | 链接断裂 |
| 跨文件系统 | 不支持 | 支持 |
| 链接目录 | 不支持 | 支持 |
| 文件类型 | 普通文件 | 符号链接类型（l） |
| 大小 | 同原文件 | 仅存路径字符串 |
| 链接计数影响 | 增加链接计数 | 不影响 |
