
# 1. 安全加固原则

## 1.1 最小权限原则

- 每个用户/进程只拥有完成其任务所必需的最小权限
- 不存在不必要的 SUID/SGID 文件
- 服务使用专用系统用户运行

## 1.2 纵深防御

```
网络层：防火墙（iptables/nftables）
主机层：SELinux/AppArmor、文件权限
应用层：运行非 root 用户、沙箱
审计层：日志审计、入侵检测
```

## 1.3 安全基线与标准

- **CIS Benchmarks**：Center for Internet Security 发布的加固指南
- **NIST 800-53**：美国国家标准与技术研究院安全控制
- **等保 2.0**：中国信息安全等级保护标准

# 2. 账户安全

## 2.1 用户账户管理

```bash
# 禁用不必要的系统账户
# 将 Shell 设为 /usr/sbin/nologin
sudo usermod -s /usr/sbin/nologin <username>

# 锁定不需要的账户
sudo passwd -l <username>

# 删除无主用户
sudo userdel -r <username>

# 审计所有可登录用户
grep -v "/usr/sbin/nologin\|/bin/false" /etc/passwd
```

## 2.2 密码策略

```bash
# /etc/login.defs
PASS_MAX_DAYS   90           # 密码最长使用 90 天
PASS_MIN_DAYS   1            # 密码最短使用 1 天
PASS_WARN_AGE   14           # 过期前 14 天警告

# 密码复杂度（安装 libpam-pwquality）
sudo apt install libpam-pwquality

# /etc/pam.d/common-password
password requisite pam_pwquality.so retry=3 minlen=12 \
    dcredit=-1 ucredit=-1 ocredit=-1 lcredit=-1
# minlen=12: 最小 12 位
# dcredit=-1: 至少 1 个数字
# ucredit=-1: 至少 1 个大写字母
# ocredit=-1: 至少 1 个特殊字符
# lcredit=-1: 至少 1 个小写字母
```

## 2.3 sudo 安全配置

```bash
# /etc/sudoers（使用 visudo 编辑）

# 最小权限原则：限制用户执行特定命令
alice ALL=(root) /usr/bin/systemctl restart nginx
bob ALL=(root) /usr/bin/apt

# 禁止使用 sudo 执行特定危险命令
alice ALL=(root) !/usr/bin/passwd root

# 禁用 root sudo（先配置好 sudo 用户）
# 需要将用户加入 sudo 组
# 然后允许 sudo 组 sudo 但禁止 root
# 或在 /etc/ssh/sshd_config 设置 PermitRootLogin no

# 记录所有 sudo 操作
Defaults logfile=/var/log/sudo.log
Defaults log_input, log_output    # 记录输入输出（安全要求高的环境）
```

## 2.4 SSH 安全

```bash
# /etc/ssh/sshd_config 安全配置
Port 2222                          # 修改默认端口（减少扫描）
Protocol 2                         # 仅使用 SSHv2
PermitRootLogin no                 # 禁止 root 直接登录
PasswordAuthentication no          # 禁用密码认证
PubkeyAuthentication yes           # 仅密钥认证
MaxAuthTries 3                     # 最多 3 次尝试
MaxSessions 2                      # 最多 2 个会话
LoginGraceTime 30                  # 登录超时 30 秒
AllowUsers alice bob              # 限定 SSH 用户
PermitEmptyPasswords no            # 禁止空密码
ClientAliveInterval 300            # 客户端保活 300 秒
ClientAliveCountMax 2              # 保活失败 2 次断开
```

# 3. 文件系统安全

## 3.1 文件权限检查

```bash
# 查找 SUID/SGID 文件（定期审计）
find / -perm -4000 -type f 2>/dev/null
find / -perm -2000 -type f 2>/dev/null

# 查找全局可写文件
find / -perm -o+w -type f 2>/dev/null | grep -v "/proc\|/sys\|/dev"

# 查找无主文件
find / -nouser -o -nogroup 2>/dev/null

# 检查和修复（移除不必要的 SUID）
chmod u-s /usr/bin/<binary>
```

## 3.2 关键目录权限

```bash
# 确保关键目录权限正确
chmod 755 /etc                    # 配置文件目录
chmod 750 /etc/ssh                # SSH 配置
chmod 600 /etc/ssh/sshd_config    # SSH 服务器配置
chmod 700 /root                   # root 家目录
chmod 700 /home/*/.ssh            # 用户 SSH 目录
chmod 600 /home/*/.ssh/authorized_keys  # 公钥认证文件
chmod 644 /etc/passwd             # 密码文件只读
chmod 640 /etc/shadow             # shadow 文件
chmod 644 /etc/group
chmod 640 /etc/gshadow
```

## 3.3 文件属性保护

```bash
# 防止关键配置文件被修改
sudo chattr +i /etc/passwd
sudo chattr +i /etc/shadow
sudo chattr +i /etc/group
sudo chattr +i /etc/sudoers

# 日志文件仅追加
sudo chattr +a /var/log/syslog
sudo chattr +a /var/log/auth.log
```

# 4. PAM 认证

## 4.1 PAM 架构

PAM（Pluggable Authentication Modules）提供可插拔的认证模块体系：

```
应用程序（login, sshd, sudo...）
    ↓
PAM 配置（/etc/pam.d/）
    ↓
PAM 模块（pam_unix, pam_limits, pam_google_authenticator...）
```

## 4.2 配置示例

```bash
# /etc/pam.d/sshd（SSH 认证流程）
auth       required     pam_securetty.so      # 检查 securetty
auth       requisite    pam_nologin.so        # 检查 nologin
auth       required     pam_unix.so           # Unix 密码验证
account    required     pam_unix.so           # 账户状态
password   required     pam_unix.so           # 密码修改
session    required     pam_unix.so           # 会话管理

# 添加登录失败锁定
# 在 /etc/pam.d/common-auth 添加
auth required pam_tally2.so deny=5 unlock_time=300
# 5 次失败后锁定 300 秒

# 限制资源（/etc/security/limits.conf）
alice   hard    nproc   100
@staff  hard    nofile  4096
```

## 4.3 双因素认证

```bash
# 安装 Google Authenticator PAM 模块
sudo apt install libpam-google-authenticator

# 为用户配置
google-authenticator

# 在 /etc/pam.d/sshd 添加
auth required pam_google_authenticator.so

# /etc/ssh/sshd_config 启用
ChallengeResponseAuthentication yes
AuthenticationMethods publickey,keyboard-interactive
```

# 5. 审计与监控

## 5.1 auditd

Linux 内核审计系统，监控系统调用和文件访问：

```bash
# 安装
sudo apt install auditd

# 添加审计规则
# /etc/audit/rules.d/audit.rules

# 监控关键文件变更
-w /etc/passwd -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/sudoers -p wa -k sudoers
-w /etc/ssh/sshd_config -p wa -k sshd

# 监控系统调用（由 uname 触发）
-a always,exit -S uname -k system_info

# 监控执行
-a exit,always -F arch=b64 -S execve -k process_execution

# 查看审计日志
sudo ausearch -k identity             # 按关键字搜索
sudo aureport --summary               # 审计摘要
sudo aureport -l                      # 登录报告

# 审计日志
sudo tail -f /var/log/audit/audit.log
```

## 5.2 入侵检测

```bash
# AIDE（文件完整性检查）
sudo apt install aide
sudo aideinit                          # 初始化数据库
sudo aide --check                      # 检查文件变更

# rkhunter（Rootkit 检测）
sudo apt install rkhunter
sudo rkhunter --check                  # 检查 Rootkit

# Lynis（安全审计）
sudo apt install lynis
sudo lynis audit system                # 系统安全审计
```

# 6. 网络层安全

## 6.1 系统参数加固

```bash
# /etc/sysctl.d/99-security.conf

# IP 转发（如果不是路由器，关闭）
net.ipv4.ip_forward = 0

# 反向路径过滤（防止 IP 欺骗）
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# 忽略 ICMP 重定向
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv6.conf.all.accept_redirects = 0

# 忽略 ICMP 请求
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1

# TCP SYN Cookie（防止 SYN 洪水）
net.ipv4.tcp_syncookies = 1

# 减少 TIME_WAIT 连接风险
net.ipv4.tcp_tw_reuse = 1

# 禁用源路由
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0

# 日志可疑包
net.ipv4.conf.all.log_martians = 1
```

## 6.2 防火墙配置

```bash
# 最小权限防火墙策略
# 默认丢弃入站，允许出站
sudo ufw default deny incoming
sudo ufw default allow outgoing

# 只开放必要端口
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https

# 限制 SSH 来源 IP
sudo ufw allow from 192.168.1.0/24 to any port 22

# 启用
sudo ufw enable
```

# 7. 数据加密

## 7.1 LUKS 磁盘加密

```bash
# 加密分区
sudo cryptsetup luksFormat /dev/sdb1
sudo cryptsetup open /dev/sdb1 secret
sudo mkfs.ext4 /dev/mapper/secret
sudo mount /dev/mapper/secret /mnt/secret

# 配置自动挂载（需密钥文件或 passphrase）
# /etc/crypttab
secret /dev/sdb1 none luks

# 查看 LUKS 状态
sudo cryptsetup status secret
```

## 7.2 GPG 文件加密

```bash
# 对称加密
gpg -c file.txt                       # 加密（输入密码）
gpg -d file.txt.gpg                   # 解密

# 公钥加密
gpg --gen-key                         # 生成密钥对
gpg --recipient alice@example.com -e file.txt  # 加密
gpg -d file.txt.gpg                   # 解密
```

## 7.3 SSL/TLS 证书

```bash
# 自签名证书（测试环境）
sudo apt install openssl
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout private.key -out certificate.crt

# 查看证书
openssl x509 -in certificate.crt -text -noout
```

# 8. 容器安全

```bash
# 不要以 root 运行容器
docker run --user 1000:1000 myapp

# 限制容器权限
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp

# 只读根文件系统
docker run --read-only --tmpfs /tmp myapp

# 使用安全扫描
docker scan myapp:latest
trivy image myapp:latest
```
