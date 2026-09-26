
# 1. SSH 概述

SSH（Secure Shell）是一种加密网络协议，用于在不安全的网络上安全地访问远程系统。它替代了不安全的 telnet、rlogin、rsh 等协议。

**核心特性**：
- 加密通信（防止窃听）
- 身份验证（防止假冒）
- 完整性校验（防止篡改）
- 端口转发（隧道加密其他协议）

# 2. SSH 架构

```
客户端（ssh）               服务器（sshd）
┌─────────┐              ┌──────────────────┐
│  ssh     │ ── TCP 22 ─→ │  sshd            │
│  client  │ ← 加密通道 → │  authorized_keys │
│  ~/.ssh/ │              │  /etc/ssh/       │
└─────────┘              └──────────────────┘
```

**SSH 连接建立过程**：

```
1. TCP 三次握手
2. 服务器发送主机密钥，客户端验证
3. 密钥交换（Diffie-Hellman）建立共享密钥
4. 对称加密会话建立
5. 用户认证（密码/公钥）
6. 会话开始
```

# 3. SSH 客户端配置

## 3.1 基本连接

```bash
ssh user@hostname                  # 基本连接
ssh user@192.168.1.100             # 使用 IP
ssh -p 2222 user@hostname          # 指定端口
ssh -l user hostname               # 指定用户名（简写）
```

## 3.2 SSH 配置文件（~/.ssh/config）

通过配置文件简化连接：

```bash
# ~/.ssh/config
# 全局设置
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    Compression yes

# 特定主机配置
Host myserver
    HostName 192.168.1.100
    User alice
    Port 22
    IdentityFile ~/.ssh/id_ed25519

Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_github

Host *.internal.company.com
    User bob
    ProxyJump jump.internal.company.com

# 连接
ssh myserver                    # 等价于 ssh -p 22 alice@192.168.1.100
```

## 3.3 配置选项速查

| 选项 | 含义 |
|------|------|
| HostName | 目标主机地址 |
| User | 登录用户名 |
| Port | SSH 端口（默认 22）|
| IdentityFile | 私钥路径 |
| ProxyJump | 跳板机 |
| ForwardAgent | 转发 SSH agent |
| LocalForward | 本地端口转发 |
| RemoteForward | 远程端口转发 |
| ServerAliveInterval | 保活间隔（秒）|
| StrictHostKeyChecking | 是否严格验证主机密钥 |

# 4. 密钥认证

## 4.1 生成密钥对

```bash
# 生成 ed25519 密钥（推荐，更安全且更短）
ssh-keygen -t ed25519 -C "alice@example.com"

# 生成 RSA 密钥（兼容性更好）
ssh-keygen -t rsa -b 4096 -C "alice@example.com"

# 指定文件名和非交互式
ssh-keygen -t ed25519 -f ~/.ssh/deploy_key -N ""
```

## 4.2 复制公钥到服务器

```bash
# 自动复制（推荐）
ssh-copy-id user@hostname
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@hostname

# 手动复制
cat ~/.ssh/id_ed25519.pub | ssh user@hostname "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

## 4.3 authorized_keys 配置

```bash
# ~/.ssh/authorized_keys（每行一个公钥）
# 可以设置限制选项
from="192.168.1.*" ssh-ed25519 AAAAC3... alice@desktop
command="/usr/bin/rsync" ssh-rsa AAAAB3... backup@server
restrict,command="/usr/local/bin/backup.sh" ssh-ed25519 AAAAC3... backup-robot
```

**限制选项**：
| 选项 | 作用 |
|------|------|
| from="pattern" | 限制来源 IP |
| command="cmd" | 固定执行命令（不提供 Shell）|
| no-agent-forwarding | 禁止 agent 转发 |
| no-port-forwarding | 禁止端口转发 |
| no-pty | 不分配伪终端 |
| restrict | 限制所有转发功能 |

## 4.4 SSH Agent

SSH Agent 管理私钥，避免每次连接输入密码：

```bash
# 启动 agent（通常自动启动）
eval "$(ssh-agent -s)"

# 添加私钥
ssh-add ~/.ssh/id_ed25519
ssh-add -l                         # 列出已加载的密钥

# 转发 agent（在 /etc/ssh/ssh_config 或 config 中设置）
ssh -A user@hostname               # -A 转发 agent
```

# 5. SSH 服务器配置（sshd）

## 5.1 关键配置选项

`/etc/ssh/sshd_config`：

```bash
# 端口（建议修改非默认端口减少扫描）
Port 22

# 认证
PermitRootLogin prohibit-password   # 禁止 root 密码登录
PubkeyAuthentication yes            # 启用公钥认证
PasswordAuthentication yes          # 是否允许密码登录（建议关闭）
PermitEmptyPasswords no             # 禁止空密码

# 安全
MaxAuthTries 3                      # 最大认证尝试次数
MaxSessions 10                      # 最大并发会话数
ClientAliveInterval 300             # 客户端保活间隔
ClientAliveCountMax 2              # 保活失败次数

# 用户限制
AllowUsers alice bob                # 仅允许这些用户 SSH
DenyUsers root                      # 禁止用户登录
AllowGroups ssh-users               # 仅允许组内成员登录

# 转发
AllowTcpForwarding yes
X11Forwarding no                    # 建议关闭 X11 转发
```

## 5.2 安全推荐配置

```bash
# /etc/ssh/sshd_config.d/hardening.conf
Port 2222                           # 修改默认端口
PasswordAuthentication no           # 禁用密码认证
PubkeyAuthentication yes
PermitRootLogin no                  # 禁止 root 直接登录
MaxAuthTries 3
MaxSessions 5
AllowUsers alice bob
Protocol 2                          # 仅允许 SSHv2
LogLevel VERBOSE                    # 详细日志
```

应用配置后重启服务：
```bash
sudo systemctl restart sshd
```

## 5.3 检查 SSH 服务状态

```bash
sudo systemctl status sshd          # 服务状态
ss -tuln | grep :22                 # 确认端口监听
ssh -v user@hostname                # 客户端调试输出
ssh -vvv user@hostname              # 更详细的调试
```

# 6. 端口转发（SSH Tunneling）

## 6.1 本地端口转发（Local Forwarding）

将本地端口上的流量通过 SSH 隧道转发到远程目标：

```bash
# 访问远程内网的 Web 服务
ssh -L 8080:internal-web:80 user@gateway
# 本机 http://localhost:8080 → gateway → internal-web:80

# 通过跳板机访问 MySQL
ssh -L 3306:db-server:3306 user@bastion
# 本机连接 localhost:3306 相当于连接 db-server:3306
```

## 6.2 远程端口转发（Remote Forwarding）

将远程服务器的端口转发回本地：

```bash
# 将远程服务器的 8080 转发到本地的 80
ssh -R 8080:localhost:80 user@remote
# 访问 remote:8080 → 本机 localhost:80

# 穿透 NAT/防火墙展示本地服务
ssh -R 80:localhost:3000 user@public-server
```

## 6.3 动态端口转发（SOCKS 代理）

创建 SOCKS 代理，将流量通过 SSH 服务器路由：

```bash
ssh -D 1080 user@hostname
# 浏览器设置 SOCKS5 代理为 localhost:1080
# 所有流量通过 SSH 服务器转发
```

## 6.4 跳板机（ProxyJump）

通过跳板机访问内网服务器：

```bash
# 单跳
ssh -J user@jump-server user@target-server

# 多跳
ssh -J user@jump1,user@jump2 user@target

# 配置文件中定义
Host target
    HostName 10.0.1.100
    User alice
    ProxyJump bastion.company.com
```

# 7. 文件传输

## 7.1 scp

```bash
# 上传
scp file.txt user@hostname:/remote/path/
scp -P 2222 file.txt user@hostname:/path/  # 指定端口

# 下载
scp user@hostname:/remote/file.txt ./

# 递归传输目录
scp -r -P 2222 user@hostname:/remote/dir/ ./local/

# 在两台远程主机之间传输
scp user1@host1:/file user2@host2:/file
```

## 7.2 rsync

比 scp 更高效，支持增量传输：

```bash
# 基本用法
rsync -av source/ user@hostname:/dest/
# -a 归档模式（递归+保留属性），-v 详细输出

# 增量备份（只传输变更部分）
rsync -avz --progress /local/dir/ user@hostname:/backup/
# -z 压缩传输

# 删除目标端多出的文件
rsync -av --delete /source/ user@hostname:/dest/

# 通过 SSH（默认）
rsync -av -e "ssh -p 2222" /source/ user@hostname:/dest/

# 排除文件
rsync -av --exclude='*.log' --exclude='.git/' /source/ user@hostname:/dest/

# 限速（防止占满带宽）
rsync -av --bwlimit=1000 /source/ user@hostname:/dest/  # KB/s
```

## 7.3 sftp — 基于 SSH 的文件传输

```bash
# 交互式
sftp user@hostname
sftp> ls
sftp> get remote_file.txt
sftp> put local_file.txt
sftp> get -r remote_dir/   # 递归下载
sftp> exit

# 单命令
sftp user@hostname:remote_file.txt ./
```

# 8. SSH 安全加固

| 措施 | 说明 |
|------|------|
| 禁用密码认证 | 仅使用密钥认证 |
| 修改默认端口 | 减少扫描攻击 |
| 禁用 root 登录 | 使用普通用户 + sudo |
| 使用 ed25519 密钥 | 比 RSA 更安全高效 |
| 配置 AllowUsers | 限定允许登录的用户 |
| 使用 Fail2ban | 自动封禁暴力破解 IP |
| 设置登录通知 | 登录时发邮件或 webhook |
| 使用 SSH 证书 | 大型团队集中管理密钥 |
| 2FA 认证 | 结合 TOTP 或 U2F |

## 8.1 Fail2ban 保护 SSH

```bash
sudo apt install fail2ban
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local

# /etc/fail2ban/jail.local
[sshd]
enabled = true
port = ssh
maxretry = 5
bantime = 3600
findtime = 600
```
