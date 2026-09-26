
# 1. DNS 基础

## 1.1 域名层级结构

DNS（Domain Name System）是一个分布式的命名系统，将域名映射为 IP 地址。

```
根域（.）
├── 顶级域（TLD）：.com .org .net .cn .jp ...
│   ├── 二级域：google.com baidu.com
│   │   └── 子域：www.google.com mail.google.com
```

**FQDN（完全限定域名）**：`hostname.domain.tld.`（末尾的点表示根域）

## 1.2 DNS 查询流程

```
用户在浏览器输入 www.example.com

1. 浏览器检查本地 DNS 缓存
2. 检查 /etc/hosts
3. 查询系统配置的 DNS 解析器（通常是本地缓存 DNS 服务）
4. DNS 解析器：
   a. 查询根 DNS 服务器 → 获取 .com TLD 服务器地址
   b. 查询 .com TLD 服务器 → 获取 example.com 权威 NS 地址
   c. 查询 example.com 权威 NS → 获取 www.example.com 的 A 记录
5. 返回 IP 地址（逐级缓存）
```

## 1.3 DNS 记录类型

| 类型 | 说明 | 示例 |
|------|------|------|
| A | IPv4 地址映射 | www.example.com → 93.184.216.34 |
| AAAA | IPv6 地址映射 | www.example.com → 2606:2800:220:1:248:1893:25c8:1946 |
| CNAME | 别名 | www.example.com → example.com |
| MX | 邮件交换 | example.com → mail.example.com（优先级 10）|
| NS | 权威名称服务器 | example.com → ns1.example.com |
| TXT | 文本记录 | SPF、DKIM、DMARC 验证 |
| SOA | 权威起始记录 | DNS 区域的权威信息 |
| PTR | 反向解析（IP→域名）| 93.184.216.34 → www.example.com |

# 2. 本地名称解析

## 2.1 /etc/hosts

本地静态主机名到 IP 的映射，优先级高于 DNS：

```bash
# /etc/hosts
127.0.0.1       localhost
127.0.1.1       myhostname
192.168.1.100   dev-server.local
::1             localhost ip6-localhost ip6-loopback

# 可以临时覆盖 DNS 解析（常用于开发和测试）
127.0.0.1       api.example.com
```

## 2.2 /etc/resolv.conf

DNS 解析器配置：

```bash
# /etc/resolv.conf
nameserver 8.8.8.8            # DNS 服务器地址（可配置多个）
nameserver 8.8.4.4
search example.com local       # 短主机名自动补全的搜索域
options timeout:2 attempts:3   # 超时 2 秒，尝试 3 次
options rotate                 # 轮询使用 nameserver
```

**注意**：在现代 Linux 中，`/etc/resolv.conf` 通常由 systemd-resolved 或 NetworkManager 管理，不应手动编辑。

## 2.3 /etc/nsswitch.conf

名称服务交换机（Name Service Switch）配置文件，定义查找顺序：

```bash
# /etc/nsswitch.conf
hosts:          files dns          # 先查 /etc/hosts，再查 DNS
networks:       files
# 可用的数据源：files、dns、mdns、systemd、myhostname
```

# 3. DNS 查询工具

## 3.1 dig — 最强大的 DNS 查询工具

```bash
# 基本查询
dig google.com                      # 查询 A 记录（默认）
dig google.com A                    # 显式查询 A 记录
dig google.com AAAA                 # 查询 AAAA 记录
dig google.com MX                   # 查询 MX 记录
dig google.com NS                   # 查询 NS 记录
dig google.com TXT                  # 查询 TXT 记录
dig google.com ANY                  # 查询所有记录（已被多数 DNS 禁用于放大攻击）

# 简洁模式
dig google.com +short               # 只返回 IP 地址
dig google.com +noall +answer       # 只显示回答部分

# 指定 DNS 服务器
dig @8.8.8.8 google.com             # 使用 Google DNS 查询
dig @1.1.1.1 google.com             # 使用 Cloudflare DNS

# 追踪 DNS 路径
dig +trace google.com               # 从根服务器逐级追踪

# 反向查询
dig -x 93.184.216.34                # IP 到域名的反向查询

# 批量查询
dig -f domains.txt +short           # 从文件读取域名

# 查看完整响应
dig google.com
# ;; ANSWER SECTION:
# google.com.        299     IN      A       142.250.80.46
# 299 = TTL（缓存时间，单位秒）
```

## 3.2 nslookup — 传统查询工具

```bash
nslookup google.com                 # 基本查询
nslookup google.com 8.8.8.8        # 指定 DNS 服务器
nslookup -type=mx google.com       # 查询 MX 记录
nslookup -type=ns google.com       # 查询 NS 记录

# 交互模式
nslookup
> server 8.8.8.8
> set type=MX
> google.com
> exit
```

## 3.3 host — 简洁查询

```bash
host google.com                     # 简洁查询
host -a google.com                  # 所有记录（相当于 ANY）
host -t MX google.com               # 查询 MX 记录
host 93.184.216.34                  # 反向查询
```

## 3.4 resolvectl — systemd-resolved 查询

```bash
resolvectl status                   # 查看系统 DNS 状态
resolvectl query google.com         # 查询 DNS
resolvectl query google.com 2>/dev/null | grep -oP '\d+\.\d+\.\d+\.\d+'
resolvectl dns eth0                 # 查看接口的 DNS 服务器
resolvectl statistics               # DNS 缓存统计
```

# 4. DNS 缓存

## 4.1 本地 DNS 缓存

系统级 DNS 缓存服务：

| 服务 | 特点 | 启用 |
|------|------|------|
| systemd-resolved | systemd 内置 | 默认多数发行版 |
| dnsmasq | 轻量级 DNS 转发器 | 可手动安装 |
| nscd | 较老的名缓存服务 | 逐渐被取代 |
| Unbound | 递归验证式解析器 | 安全要求高的场景 |

## 4.2 清除 DNS 缓存

```bash
# systemd-resolved
sudo resolvectl flush-caches

# dnsmasq
sudo systemctl restart dnsmasq

# nscd
sudo /etc/init.d/nscd restart

# 应用层缓存
# Chrome: chrome://net-internals/#dns
# Firefox: 隐私与安全 → 清除历史
```

# 5. 高级 DNS 概念

## 5.1 EDNS（扩展 DNS）

- 支持大于 512 字节的 UDP DNS 响应
- 支持 DNSSEC、ECS（客户端子网）等扩展

## 5.2 DNSSEC

DNS 安全扩展（DNS Security Extensions），通过数字签名验证 DNS 响应的真实性：

```bash
# 验证 DNSSEC
dig google.com +dnssec                # 查看 DNSSEC 签名
delv google.com                        # DNS 验证工具（DNSSEC 感知）
```

## 5.3 DNS over HTTPS/TLS

加密 DNS 查询以保护隐私：

```bash
# systemd-resolved 配置 DoT
resolvectl dns eth0 1.1.1.1
resolvectl tls eth0 yes

# 或使用 /etc/systemd/resolved.conf
[Resolve]
DNS=1.1.1.1#cloudflare-dns.com
DNSOverTLS=yes
```

## 5.4 mDNS（多播 DNS）

局域网内 .local 域名的零配置名称解析：

```bash
# 使用 Avahi（Linux mDNS 实现）
sudo apt install avahi-daemon

# 查询 .local 主机名
ping mylaptop.local

# mDNS 通过 UDP 5353 端口通信
```

# 6. 常见问题排查

```bash
# 检查 DNS 解析是否正常
dig google.com +short
nslookup google.com

# 检查 DNS 服务器是否能到达
nc -z -v 8.8.8.8 53

# 检查 /etc/hosts 是否有冲突
grep -v "^#" /etc/hosts | grep -v "^$"

# 检查系统 DNS 配置
resolvectl status

# DNS 解析耗时测试
time dig google.com +short

# 对比不同 DNS 服务器
dig @8.8.8.8 google.com +short
dig @1.1.1.1 google.com +short
```
