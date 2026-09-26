
# 1. Linux 用户模型

Linux 是多用户操作系统，每个进程都关联一个用户 ID（UID）和组 ID（GID），用于权限控制和资源隔离。

**用户分类**：
| 类型 | UID 范围 | 说明 |
|------|----------|------|
| root（超级用户） | 0 | 拥有系统全部权限 |
| 系统用户 | 1-999（Debian）/ 1-199（RHEL） | 运行系统服务（sshd、www-data 等），通常无登录 Shell |
| 普通用户 | 1000+（Debian）/ 1000+（RHEL） | 普通用户账户，受权限限制 |

查看当前用户信息：
```bash
whoami                # 当前用户名
id                    # UID、GID、所属组
id -u                 # 仅显示 UID
id -g                 # 仅显示 GID
id -G                 # 显示所有附加组 GID
groups                # 显示所属组名称
```

# 2. 用户账户文件

## 2.1 /etc/passwd

每行对应一个用户账户，7 个字段以冒号分隔：

```
alice:x:1000:1000:Alice,,,:/home/alice:/bin/bash
^     ^ ^    ^   ^          ^          ^
|     | |    |   |          |          登录 Shell 路径
|     | |    |   |          用户家目录
|     | |    |   GECOS 注释信息（全名、电话等，逗号分隔）
|     | |    GID
|     | UID
|     密码占位符（x 表示密码存储在 /etc/shadow）
用户名
```

**注意**：`/etc/passwd` 对所有人可读（`-rw-r--r--`），所以密码不能直接存在这里。

## 2.2 /etc/shadow

存储加密密码和密码策略，仅 root 可读：

```
alice:$y$j9T$...:19876:0:99999:7:30:90:
^     ^          ^      ^ ^     ^ ^  ^
|     |          |      | |     | |  └ 账户过期天数
|     |          |      | |     | └ 警告过期后的宽限天数
|     |          |      | |     └ 密码过期前警告天数（默认7）
|     |          |      | └ 密码最长有效期（默认99999）
|     |          |      └ 密码最短使用天数（0=可随时更改）
|     |          └ 最近一次修改密码的时间（1970-01-01 起天数）
|     └ 加密密码（$type$salt$hash）
└ 用户名
```

**加密方式**：
- `$y$`：yescrypt（新版本 Debian/Ubuntu）
- `$6$`：SHA-512（较常见）
- `$5$`：SHA-256
- `$1$`：MD5（已不推荐）
- `$2b$`：bcrypt

## 2.3 /etc/group

```
staff:x:1000:alice,bob
^     ^ ^    ^
|     | |    └ 附加组成员（逗号分隔，不含主要组）
|     | └ GID
|     └ 组密码（通常为空，x 表示使用 gshadow）
└ 组名
```

## 2.4 /etc/gshadow

组的密码文件，存储组管理员和组密码（很少使用）。

# 3. 用户管理命令

## 3.1 useradd — 创建用户

```bash
# 基本创建
sudo useradd alice                           # 创建用户（默认行为）

# 常用选项
sudo useradd -m alice                        # 创建家目录 /home/alice
sudo useradd -m -d /custom/home alice        # 指定家目录路径
sudo useradd -m -s /bin/bash alice           # 指定登录 Shell
sudo useradd -m -u 1500 alice               # 指定 UID
sudo useradd -m -g staff alice              # 指定主要组（默认创建同名组）
sudo useradd -m -G sudo,docker alice        # 指定附加组
sudo useradd -m -c "Alice Wang" alice       # 添加注释

# 创建系统用户
sudo useradd -r -s /usr/sbin/nologin myapp   # 无登录能力的系统用户
```

**useradd 默认值**定义在 `/etc/default/useradd` 和 `/etc/login.defs`：
```bash
useradd -D            # 查看/修改默认值
# GROUP=100
# HOME=/home
# INACTIVE=-1
# EXPIRE=
# SHELL=/bin/bash
# SKEL=/etc/skel
```

**SKEL 机制**：创建用户时，`/etc/skel/` 目录下的所有文件自动复制到用户家目录：
```bash
ls -la /etc/skel/
# .bashrc  .profile  .bash_logout  ...
```

## 3.2 passwd — 设置密码

```bash
sudo passwd alice                       # 设置/修改 alice 的密码
sudo passwd -l alice                    # 锁定用户（在密码前加 !）
sudo passwd -u alice                    # 解锁用户
sudo passwd -d alice                    # 删除密码（空密码登录）
sudo passwd -S alice                    # 查看密码状态
```

## 3.3 usermod — 修改用户属性

```bash
sudo usermod -s /bin/zsh alice          # 修改登录 Shell
sudo usermod -l newname alice           # 修改用户名
sudo usermod -d /new/home -m alice      # 修改家目录并移动文件
sudo usermod -aG docker alice           # 追加到组（-a 必须配合 -G 使用）
sudo usermod -G "" alice                # 移除所有附加组
sudo usermod -e 2025-12-31 alice        # 设置账户过期日期
sudo usermod -L alice                   # 锁定账户（在 shadow 密码前加 !）
sudo usermod -U alice                   # 解锁账户
```

## 3.4 userdel — 删除用户

```bash
sudo userdel alice                      # 删除用户（保留家目录和邮件）
sudo userdel -r alice                   # 删除用户及家目录、邮件
sudo userdel -f alice                   # 强制删除（即使正在登录）
```

## 3.5 chage — 密码过期策略

```bash
sudo chage -l alice                     # 列出当前策略
sudo chage -M 90 alice                  # 密码最长使用90天
sudo chage -m 7 alice                   # 密码最短使用7天
sudo chage -W 14 alice                  # 过期前14天警告
sudo chage -I 30 alice                  # 过期后30天锁定账户
sudo chage -E 2025-12-31 alice          # 账户过期日期

# 强制用户下次登录时修改密码
sudo chage -d 0 alice
```

# 4. 组管理命令

## 4.1 groupadd / groupdel / groupmod

```bash
sudo groupadd developers                # 创建组
sudo groupadd -g 2000 developers        # 指定 GID
sudo groupdel developers                # 删除组（删除前需移除所有主要用户）

sudo groupmod -n newname oldname        # 重命名组
sudo groupmod -g 2500 developers        # 修改 GID
```

## 4.2 gpasswd — 组管理

```bash
sudo gpasswd -a alice developers        # 添加用户到组（同 usermod -aG）
sudo gpasswd -d alice developers        # 从组中移除用户
sudo gpasswd -A alice developers        # 设置组管理员
sudo gpasswd developers                 # 设置组密码
```

## 4.3 groups 和 newgrp

```bash
groups alice                            # 查看用户所属组
newgrp developers                       # 切换当前会话的主要组
sg developers "command"                 # 以指定组执行命令
```

# 5. 常用管理操作

## 5.1 列出所有用户和组

```bash
# 列出所有用户
cut -d: -f1 /etc/passwd

# 列出所有组
cut -d: -f1 /etc/group

# 列出可登录的用户（有 Shell）
grep -v "/usr/sbin/nologin\|/bin/false" /etc/passwd

# 列出当前登录用户
who                     # 当前登录用户
w                       # 更详细的信息
last                    # 登录历史
lastlog                 # 所有用户的最后登录时间
```

## 5.2 切换用户

```bash
su                      # 切换到 root（但保留当前环境变量）
su -                    # 切换到 root（完全切换，加载目标用户环境）
su - alice              # 切换到 alice
su -c "command" root    # 以 root 身份执行单条命令（需输入 root 密码）

sudo -i                 # 以 root 身份交互式登录
sudo -u alice command   # 以 alice 身份执行命令
sudo -l                 # 查看当前用户有哪些 sudo 权限
```

## 5.3 用户创建完整示例

```bash
#!/bin/bash
# 创建新开发人员的完整流程

USERNAME="devuser"
GROUP="developers"

# 1. 创建组（如果不存在）
sudo groupadd -f "$GROUP"

# 2. 创建用户
sudo useradd -m -s /bin/bash -g "$GROUP" -G docker,sudo "$USERNAME"

# 3. 设置密码并强制下次修改
sudo passwd "$USERNAME"
sudo chage -d 0 "$USERNAME"

# 4. 设置密码策略
sudo chage -M 90 -m 1 -W 7 "$USERNAME"

# 5. 设置 SSH 目录
sudo mkdir -p /home/$USERNAME/.ssh
sudo chmod 700 /home/$USERNAME/.ssh

# 6. 验证
id "$USERNAME"
```

# 6. 特殊用户与安全

## 6.1 nobody 用户

- UID 通常为 65534 或 99
- 用于运行无特权的进程（如 Nginx worker 进程）
- 权限最小化原则的典型应用

## 6.2 锁定与禁用用户

```bash
# 完全禁用（禁止登录 + 锁定）
sudo usermod -s /sbin/nologin alice      # 更改 Shell 为 nologin
sudo usermod -L alice                    # 锁定密码

# 临时禁用
sudo passwd -l alice                     # 锁定密码（可恢复）

# 查看锁定状态
sudo passwd -S alice
```

## 6.3 用户会话管理

```bash
# 踢出用户（终止其所有进程）
sudo pkill -u alice
sudo pkill -9 -u alice                  # 强制终止

# 查看用户进程
pgrep -u alice -l
ps -u alice
```
