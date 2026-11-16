
# 1. 基本权限模型

Linux 权限模型是基于用户身份的 **DAC（自主访问控制）** 系统，每个文件和目录都有三组权限位，对应三类用户：

| 类别 | 符号 | 说明 |
|------|------|------|
| User（所有者） | u | 文件所有者 |
| Group（组） | g | 文件所属组的用户 |
| Other（其他人） | o | 其他所有用户 |

每种身份有三种权限：

| 权限 | 文件含义 | 目录含义 | 数值 |
|------|---------|----------|------|
| r（读取） | 查看文件内容 | 列出目录内容（ls） | 4 |
| w（写入） | 修改文件内容 | 创建/删除/重命名目录中的文件 | 2 |
| x（执行） | 执行文件（如脚本/程序） | 进入目录（cd） | 1 |

## 1.1 权限表示格式

```bash
-rw-r--r--  1 alice staff  1024 Jan 15 10:30 file.txt
^ ^^^ ^^^ ^^^
| |   |   |
| |   |   └ others 权限：r--
| |   └ group 权限：r--
| └ user 权限：rw-
└ 文件类型：- 普通文件 / d 目录 / l 符号链接
```

```bash
drwxr-x---  2 alice staff  4096 Jan 15 10:30 documents/
```

**目录的关键：x 权限决定是否能进入目录**。如果目录只有 r 没有 x，可以列出内容但无法 cd 进入和访问文件。

## 1.2 数值表示法（八进制）

```bash
rwx = 4+2+1 = 7
rw- = 4+2+0 = 6
r-- = 4+0+0 = 4
-wx = 0+2+1 = 3
--- = 0+0+0 = 0

示例：
-rw-r--r-- = 644
-rwxr-xr-x = 755
-rw------- = 600
drwxr-x--- = 750
```

# 2. 权限管理命令

## 2.1 chmod — 修改权限

**符号法**：
```bash
chmod u+x file.txt              # 为所有者添加执行权限
chmod g-w file.txt              # 移除组的写入权限
chmod o=r file.txt              # 其他人的权限设为只读
chmod u+rwx,g+rx,o-rwx file    # 组合设置
chmod a+x script.sh             # 所有用户（ugo）加上执行权限
chmod -R g+rwx dir/             # 递归设置目录及内容
```

**数值法**：
```bash
chmod 755 script.sh             # rwxr-xr-x
chmod 644 file.txt              # rw-r--r--
chmod 600 private.key           # rw-------
chmod 700 private_dir/          # rwx------
chmod 777 file.txt              # 危险！任何人可读写执行
```

**常见权限组合**：

| 权限 | 八进制 | 用途 |
|------|--------|------|
| -rw------- | 600 | 私密文件（如 SSH 私钥） |
| -rw-r--r-- | 644 | 普通文件（默认） |
| -rw-rw-r-- | 664 | 团队协作文件 |
| -rwx------ | 700 | 私有脚本 |
| -rwxr-xr-x | 755 | 可执行程序/脚本 |
| -rwxrwxrwx | 777 | 危险！所有人可写 |
| drwx------ | 700 | 私有目录 |
| drwxr-x--- | 750 | 团队项目目录 |
| drwxr-xr-x | 755 | 普通目录（默认） |

## 2.2 chown — 修改所有者

```bash
sudo chown alice file.txt               # 修改所有者为 alice
sudo chown alice:staff file.txt         # 同时修改所有者和组
sudo chown :staff file.txt              # 只修改组
sudo chown -R alice:staff dir/          # 递归修改目录及内容
sudo chown --from=root alice file.txt   # 仅当当前所有者为 root 才修改
```

## 2.3 chgrp — 修改组（仅修改组时可用）

```bash
chgrp staff file.txt                    # 修改组为 staff
chgrp -R staff dir/                     # 递归修改
chgrp --reference=ref.txt target.txt    # 参考 ref.txt 的组设置
```

## 2.4 umask — 默认权限掩码

umask 决定新建文件/目录的默认权限：

```bash
umask                   # 查看当前 umask 值（通常 0022 或 0002）
umask 0022              # 设置 umask（临时）
```

**计算规则**：
- 文件：基础 666 - umask（但不赋予执行权限）
- 目录：基础 777 - umask

| umask | 文件默认权限 | 目录默认权限 |
|-------|-------------|-------------|
| 022 | 644 (rw-r--r--) | 755 (rwxr-xr-x) |
| 002 | 664 (rw-rw-r--) | 775 (rwxrwxr-x) |
| 007 | 660 (rw-rw----) | 770 (rwxrwx---) |
| 077 | 600 (rw-------) | 700 (rwx------) |

**全局设置**：写入 `/etc/profile`、`~/.bashrc` 或 `/etc/login.defs`。

# 3. 特殊权限

## 3.1 SUID（Set User ID）— 4000

允许执行者以**文件所有者**的权限运行该程序，而不是以执行者自己的权限。

```bash
# 设置 SUID
chmod u+s program              # 符号法
chmod 4755 program             # 数值法（4 在最高位）

# 示例
ls -l /usr/bin/passwd
# -rwsr-xr-x  1 root root   ... /usr/bin/passwd
#   ^
#   s 表示 SUID 已设置（原本是 x，覆盖为 s）
```

**原理**：当 passwd 执行时，普通用户临时获得 root 权限来修改 `/etc/shadow`。

**安全风险**：SUID 的 shell 脚本或可被利用的程序是常见提权攻击向量。应定期检查系统 SUID 文件：

```bash
find / -perm -4000 -type f 2>/dev/null
```

## 3.2 SGID（Set Group ID）— 2000

- **文件**：以文件所属组权限执行
- **目录**：在该目录下新建的文件和子目录继承目录的组

```bash
# 设置 SGID
chmod g+s dir/                 # 符号法
chmod 2755 dir/                # 数值法（2 在最高位）

# 应用场景：共享团队目录
sudo mkdir /projects/team
sudo chown :developers /projects/team
sudo chmod 2775 /projects/team
# 所有在此目录创建的文件的组自动设为 developers
```

## 3.3 Sticky Bit — 1000

主要作用于目录，限制只有**文件所有者**才能删除自己的文件，即使用户对目录有 w 权限：

```bash
# 设置 Sticky Bit
chmod o+t /tmp/                # 符号法
chmod 1777 /tmp/               # 数值法（1 在最高位）

ls -ld /tmp/
# drwxrwxrwt   ... /tmp/
#          ^
#          t 表示 Sticky Bit 已设置

# 典型目录
ls -ld /tmp/ /var/tmp/
```

**效果**：任何人都可以在 `/tmp` 创建文件，但只有文件所有者或 root 才能删除。

## 3.4 特殊权限组合总结

| 权限 | 符号 | 数值 | 文件效果 | 目录效果 |
|------|------|------|----------|----------|
| SUID | `s`（user） | 4xxx | 以所有者权限执行 | 无意义 |
| SGID | `s`（group） | 2xxx | 以组权限执行 | 新文件继承组 |
| Sticky | `t`（other） | 1xxx | 无意义 | 仅所有者能删除 |

大写字母（`S`、`T`）表示设置了特殊权限但没有对应的执行权限（几乎从不使用）。

# 4. ACL（访问控制列表）

当基本权限的三类用户不够用时，ACL 提供更精细的权限控制。

## 4.1 查看 ACL

```bash
getfacl file.txt
# file: file.txt
# owner: alice
# group: staff
# user::rw-
# user:bob:rw-
# group::r--
# mask::rw-
# other::r--
```

## 4.2 设置 ACL

```bash
# 为特定用户设置权限
setfacl -m u:bob:rw file.txt           # 给 bob 读写权限
setfacl -m u:bob:- file.txt            # 移除 bob 的所有权限

# 为特定组设置权限
setfacl -m g:developers:rx dir/

# 设置默认 ACL（目录：新文件自动继承）
setfacl -m d:u:bob:rx dir/

# 递归设置
setfacl -R -m u:bob:rx dir/

# 复制 ACL
getfacl file1 | setfacl --set-file=- file2
```

## 4.3 移除 ACL

```bash
setfacl -x u:bob file.txt              # 移除用户 bob 的 ACL 条目
setfacl -b file.txt                    # 移除所有 ACL 条目（重置为基本权限）
```

**ACL 的影响**：设置 ACL 后，`ls -l` 权限位末尾显示 `+`：
```bash
-rw-rw----+ 1 alice staff  1024 Jan 15 10:30 file.txt
#          ^
#          + 表示该文件有额外的 ACL 条目
```

# 5. 文件属性（chattr）

Linux 文件系统提供额外属性，比权限更底层，即使是 root 也可能受限：

```bash
chattr +i file.txt              # 设为不可变（immutable）
    # 任何人都不能修改、删除、重命名，即使是 root
    # 需先 chattr -i 移除属性才能操作

chattr +a logfile.log           # 仅允许追加（append only）
    # 适用于日志文件，只能追加不能删除或覆盖

chattr +e file                  # extent 格式（ext4 默认）
```

常用 chattr 选项：

| 属性 | 作用 |
|------|------|
| `+i` | 不可变（Immutable）|
| `+a` | 仅追加（Append only）|
| `+u` | 可恢复（undeletable）|
| `+s` | 安全删除（覆盖后再释放）|

查看属性：
```bash
lsattr file.txt
# ----i--------e-- file.txt
#     ^
#     i 表示设置了 immutable
```

# 6. 权限故障排查

## 6.1 常见问题

| 症状 | 可能原因 |
|------|----------|
| Permission denied | 无权操作 |
| Operation not permitted | 即使 root 也无法操作（chattr +i）|
| 无法进入目录 | 缺少目录 x 权限 |
| 无法删除文件 | 缺少目录 w 权限或 Sticky Bit 限制 |
| sudo: command not found | 命令不在 sudo 的 secure_path 中 |

## 6.2 诊断命令

```bash
# 测试实际权限
namei -l /path/to/file              # 分析路径上每层的权限
ls -la /path/to/                    # 查看目录权限
lsattr /path/to/                    # 检查文件属性

# 调试权限问题
strace -e access ls /path/          # 跟踪权限检查
sudo -u alice test -r file.txt && echo "readable"  # 模拟用户测试
```
