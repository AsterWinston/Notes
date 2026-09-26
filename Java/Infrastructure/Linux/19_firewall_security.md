
# 1. Linux 防火墙体系

## 1.1 Netfilter 架构

Linux 内核的防火墙功能由 Netfilter 框架提供，它在内核协议栈中嵌入钩子点，允许注册回调函数处理网络包：

```
数据包流向：
                     ┌───────────┐
    ┌──→ PREROUTING ─→ ROUTE DECISION ─→ FORWARD ─→ POSTROUTING ──→ 出站
    │       (DNAT)          ↕                  ↕
入站 ←─────────────────────── INPUT ───→ 本地进程
                           (过滤)
```

**Netfilter 钩子点**：
| 钩子 | 触发时机 |
|------|----------|
| NF_INET_PRE_ROUTING | 所有入站包，路由决策前 |
| NF_INET_LOCAL_IN | 路由决策后，发往本机的包 |
| NF_INET_FORWARD | 路由决策后，需要转发的包 |
| NF_INET_LOCAL_OUT | 本机发出的包 |
| NF_INET_POST_ROUTING | 所有出站包，发送前 |

## 1.2 防火墙工具演进

```
iptables (传统，IPv4/IPv6 分开管理)
    ↓
ip6tables (单独管理 IPv6)
    ↓
nftables (现代统一框架，内核 3.13+)
    ↓
firewalld (基于 nftables 的动态防火墙，RHEL/CentOS)
ufw (基于 iptables/nftables 的简化封装，Ubuntu)
```

# 2. iptables

## 2.1 概念：表（Table）和链（Chain）

**四张表**：

| 表 | 用途 | 内置链 |
|----|------|--------|
| filter | 包过滤（默认） | INPUT, FORWARD, OUTPUT |
| nat | 地址转换 | PREROUTING, INPUT, OUTPUT, POSTROUTING |
| mangle | 修改包头部 | PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING |
| raw | 连接跟踪豁免 | PREROUTING, OUTPUT |

**常用链流向**：
```
入站包：PREROUTING (nat/raw/mangle) → INPUT (filter/mangle) → 本地进程
转发包：PREROUTING (nat/raw/mangle) → FORWARD (filter/mangle) → POSTROUTING (nat/mangle)
出站包：OUTPUT (raw/nat/mangle/filter) → POSTROUTING (nat/mangle)
```

## 2.2 基本规则管理

```bash
# 查看规则
iptables -L                        # 列出规则（默认 filter 表）
iptables -t nat -L -n              # 列出 NAT 规则
iptables -L INPUT -v               # 详细统计
iptables -S                        # 输出可保存的规则文本

# 添加规则
iptables -A INPUT -p tcp --dport 22 -j ACCEPT    # 追加到末尾
iptables -I INPUT 1 -p tcp --dport 443 -j ACCEPT # 插入到第1条

# 删除规则
iptables -D INPUT 3                # 删除第3条规则
iptables -D INPUT -p tcp --dport 22 -j ACCEPT    # 按内容删除

# 修改默认策略
iptables -P INPUT DROP             # 默认丢弃入站包
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT
```

## 2.3 规则匹配条件

```bash
# 协议匹配
-p tcp
-p udp
-p icmp

# 端口匹配
--dport 80                       # 目标端口
--sport 1024:65535               # 源端口范围

# IP 匹配
-s 192.168.1.0/24                # 源地址
-d 10.0.0.1                      # 目标地址
! -s 192.168.1.100               # 取反

# 接口匹配
-i eth0                          # 入站接口
-o eth1                          # 出站接口

# 状态匹配（基于连接跟踪）
-m state --state NEW,ESTABLISHED,RELATED
```

## 2.4 目标动作（Jump Targets）

| 动作 | 说明 |
|------|------|
| ACCEPT | 允许通过 |
| DROP | 丢弃（无响应）|
| REJECT | 拒绝（返回错误消息）|
| LOG | 记录日志（继续处理）|
| SNAT | 修改源地址（POSTROUTING）|
| DNAT | 修改目标地址（PREROUTING）|
| MASQUERADE | 动态 SNAT（用于 PPPoE/DHCP）|
| RETURN | 返回调用链 |

## 2.5 实用规则示例

```bash
# 允许已建立的连接
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# 开放 SSH（端口 22）
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# 开放 HTTP/HTTPS
iptables -A INPUT -p tcp -m multiport --dports 80,443 -j ACCEPT

# 允许回环接口
iptables -A INPUT -i lo -j ACCEPT

# 允许 ping
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# 限制 SSH 暴力破解（每分钟最多 4 次连接）
iptables -A INPUT -p tcp --dport 22 -m state --state NEW \
         -m recent --set --name SSH
iptables -A INPUT -p tcp --dport 22 -m state --state NEW \
         -m recent --update --seconds 60 --hitcount 4 --name SSH -j DROP

# 限制并发连接数
iptables -A INPUT -p tcp --dport 80 -m connlimit --connlimit-above 100 -j REJECT

# NAT：端口转发（将本机 8080 转发到内网 192.168.1.100:80）
iptables -t nat -A PREROUTING -p tcp --dport 8080 \
         -j DNAT --to-destination 192.168.1.100:80

# 日志记录被丢弃的包
iptables -A INPUT -j LOG --log-prefix "FW-DROP: " --log-level info
```

## 2.6 保存和恢复规则

```bash
# Debian/Ubuntu
sudo apt install iptables-persistent
sudo netfilter-persistent save
sudo netfilter-persistent reload

# 手动保存
iptables-save > /etc/iptables/rules.v4
ip6tables-save > /etc/iptables/rules.v6

# 恢复
iptables-restore < /etc/iptables/rules.v4
```

# 3. nftables（现代替代）

## 3.1 优势

- 统一 IPv4/IPv6 管理
- 更简洁的语法
- 更好的性能（不需要遍历所有规则）
- 支持集合和映射

## 3.2 基础用法

```bash
# 查看规则
nft list ruleset

# 创建表
nft add table inet filter

# 创建链
nft add chain inet filter input { type filter hook input priority 0 \; }
nft add chain inet filter forward { type filter hook forward priority 0 \; }
nft add chain inet filter output { type filter hook output priority 0 \; }

# 添加规则
nft add rule inet filter input iif lo accept
nft add rule inet filter input ct state established,related accept
nft add rule inet filter input tcp dport 22 accept
nft add rule inet filter input counter drop

# 端口集合
nft add set inet filter allowed_ports { type inet_service \; }
nft add element inet filter allowed_ports { 22, 80, 443 }
nft add rule inet filter input tcp dport @allowed_ports accept
```

# 4. UFW（Ubuntu 默认）

## 4.1 基本用法

```bash
# 启用/禁用
sudo ufw enable
sudo ufw disable
sudo ufw status verbose

# 默认策略
sudo ufw default deny incoming
sudo ufw default allow outgoing

# 允许端口
sudo ufw allow ssh                  # 允许 SSH（22）
sudo ufw allow 80/tcp               # 允许 HTTP
sudo ufw allow 443/tcp              # 允许 HTTPS
sudo ufw allow 3000:3100/tcp        # 端口范围

# 允许特定来源
sudo ufw allow from 192.168.1.0/24 to any port 22

# 拒绝
sudo ufw deny 23                    # 拒绝 telnet

# 删除规则
sudo ufw delete allow 80/tcp
sudo ufw status numbered
sudo ufw delete 3                    # 删除编号3的规则

# 日志
sudo ufw logging on
```

# 5. SELinux

## 5.1 概述

SELinux（Security Enhanced Linux）是 NSA 开发的强制访问控制（MAC）系统，比传统的 DAC（用户/组权限）更严格。

**DAC vs MAC**：
- DAC：用户拥有文件，可以随意授权他人
- MAC：系统策略控制所有访问，即使是 root 也受限制

## 5.2 工作模式

```bash
# 查看模式
getenforce
# Enforcing  |  Permissive  |  Disabled

# 切换模式（临时）
sudo setenforce 0                  # 切换到 Permissive
sudo setenforce 1                  # 切换到 Enforcing

# 配置永久模式
# /etc/selinux/config
SELINUX=enforcing
SELINUXTYPE=targeted
```

| 模式 | 说明 |
|------|------|
| Enforcing | 强制模式，违反策略的操作被阻止并记录日志 |
| Permissive | 仅记录日志，不阻止操作 |
| Disabled | 完全禁用 SELinux |

## 5.3 上下文与策略

```bash
# 查看文件 SELinux 上下文
ls -Z
# -rw-r--r--. root root system_u:object_r:httpd_sys_content_t:s0 index.html

# 修改文件上下文
sudo chcon -t httpd_sys_content_t /var/www/html/index.html

# 恢复默认上下文
sudo restorecon -Rv /var/www/html/

# 查看进程上下文
ps -eZ | grep httpd
```

# 6. AppArmor

Ubuntu 默认使用的 MAC 机制，比 SELinux 更易用：

```bash
# 查看状态
sudo aa-status

# 查看配置文件
ls /etc/apparmor.d/

# 设置模式
sudo aa-complain /usr/sbin/nginx   # 仅记录（不阻止）
sudo aa-enforce /usr/sbin/nginx    # 强制模式

# 查看日志
sudo journalctl | grep apparmor
```

# 7. fail2ban — 暴力破解防护

## 7.1 配置示例

```bash
# /etc/fail2ban/jail.local
[DEFAULT]
bantime = 3600                     # 封禁时间（秒）
findtime = 600                     # 检测时间窗口
maxretry = 5                       # 最大尝试次数
ignoreip = 127.0.0.1/8 192.168.1.0/24  # 白名单

[sshd]
enabled = true
port = ssh
logpath = %(sshd_log)s

[nginx-http-auth]
enabled = true
port = http,https
logpath = /var/log/nginx/error.log
```

## 7.2 管理

```bash
sudo fail2ban-client status                    # 总体状态
sudo fail2ban-client status sshd               # SSH 封禁状态
sudo fail2ban-client set sshd unbanip 192.168.1.100  # 解封 IP
```
