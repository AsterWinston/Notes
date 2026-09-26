
# 1. 网络基础概念回顾

## 1.1 OSI 模型与 TCP/IP 模型

| OSI 七层 | TCP/IP 四层 | Linux 对应 |
|----------|-------------|-----------|
| 应用层 | 应用层 | 应用程序（HTTP、DNS、SSH）|
| 表示层 | ↓ | — |
| 会话层 | ↓ | — |
| 传输层 | 传输层 | TCP/UDP（内核协议栈）|
| 网络层 | 网络层 | IP、ICMP（内核）|
| 数据链路层 | 网络接口层 | 网卡驱动（内核）|
| 物理层 | ↓ | 硬件 |

## 1.2 IP 地址与子网

```bash
# IPv4 地址分类
A类：1.0.0.0 - 126.255.255.255（/8，大型网络）
B类：128.0.0.0 - 191.255.255.255（/16，中型网络）
C类：192.0.0.0 - 223.255.255.255（/24，小型网络）

# 保留地址
127.0.0.0/8    # 回环地址（localhost）
10.0.0.0/8     # 私有地址 A 类
172.16.0.0/12  # 私有地址 B 类
192.168.0.0/16 # 私有地址 C 类
169.254.0.0/16 # 链路本地地址（DHCP 失败时自动分配）
```

# 2. 网卡与设备管理

## 2.1 查看网卡信息

```bash
# 查看所有网络设备
ip link show
ip link show eth0

# 传统命令
ifconfig -a
ifconfig eth0

# 查看网卡详细信息
ethtool eth0                # 速度、双工模式、链路状态
ethtool -S eth0             # 网卡统计计数器
lspci | grep -i ethernet    # 查看网卡硬件
```

## 2.2 网卡命名

现代 Linux 使用基于固件的可预测命名规则（systemd 的 udev）：

```bash
eno1      # 内嵌网卡（onboard）
ens1      # 热插拔 PCIe 插槽
enp2s0    # PCIe 总线位置
enx78e7  # MAC 地址结尾
eth0      # 传统命名（若 biosdevname=0）
```

**命名前缀**：
| 前缀 | 含义 |
|------|------|
| en | 以太网 |
| wl | 无线局域网 |
| ww | WWAN（蜂窝网络）|

## 2.3 网卡配置（Netplan）

Ubuntu 18.04+ 使用 Netplan 作为默认网络配置工具：

```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: true
    eth1:
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

```bash
sudo netplan apply          # 应用配置
sudo netplan try            # 测试配置（超时自动回滚）
netplan get                 # 查看当前配置
```

## 2.4 NetworkManager

通常用于桌面环境或需要动态网络管理的场景：

```bash
# 基本用法
nmcli device status                      # 查看设备状态
nmcli connection show                    # 查看连接配置
nmcli connection up eth0                 # 激活连接
nmcli dev wifi connect "MyWiFi" password "xxx"  # 连接 Wi-Fi

# 命令行配置静态 IP
nmcli con mod eth0 ipv4.addresses 192.168.1.100/24
nmcli con mod eth0 ipv4.gateway 192.168.1.1
nmcli con mod eth0 ipv4.dns "8.8.8.8 8.8.4.4"
nmcli con mod eth0 ipv4.method manual
nmcli con up eth0
```

## 2.5 传统 /etc/network/interfaces 配置

```bash
# Debian 传统配置方式
auto eth0
iface eth0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 8.8.4.4
```

# 3. 网络诊断与排查

## 3.1 ping — 基本连通性测试

```bash
ping -c 4 8.8.8.8                 # 发送 4 个包
ping -i 0.5 google.com            # 每 0.5 秒发一次
ping -s 1472 google.com           # 指定包大小（MTU 测试）
ping -M do -s 1472 google.com     # 禁止分片（测试 PMTUD）
```

## 3.2 ip — 现代网络配置命令

`ip` 命令是 `ifconfig`、`route`、`arp` 等传统命令的统一替代：

```bash
# IP 地址管理
ip addr show                         # 显示所有接口 IP
ip addr add 192.168.1.100/24 dev eth0  # 添加 IP
ip addr del 192.168.1.100/24 dev eth0  # 删除 IP

# 路由管理
ip route show                        # 显示路由表
ip route add default via 192.168.1.1  # 添加默认网关
ip route add 10.0.0.0/8 via 192.168.1.1  # 添加静态路由
ip route del 10.0.0.0/8              # 删除路由

# 链路管理
ip link set eth0 up                  # 启用网卡
ip link set eth0 down                # 禁用网卡
ip link set eth0 mtu 9000            # 设置 MTU
```

## 3.3 ss — 套接字统计

ss 是 netstat 的现代替代，性能更好：

```bash
ss -tuln                            # 列出所有监听端口
# -t TCP, -u UDP, -l 监听, -n 数字显示
ss -tup                             # TCP 连接及进程信息
ss -ant state established           # 已建立的连接
ss -o state fin-wait-1              # 特定状态的连接

# 详细输出
ss -tulnpe                          # 含进程和用户信息
```

## 3.4 netstat（传统命令）

```bash
netstat -tuln                       # 查看监听端口
netstat -anp | grep 80              # 查看指定端口
netstat -rn                         # 查看路由表
netstat -i                          # 查看网络接口统计
netstat -s                          # 协议统计信息
```

## 3.5 流量与连通性

```bash
# traceroute — 路由追踪
traceroute google.com               # 追踪到目标的路由路径
traceroute -n google.com            # 不解析主机名
mtr google.com                      # traceroute + ping 组合

# 端口连通性测试
nc -zv 192.168.1.1 22               # 测试 TCP 端口
nc -zvu 192.168.1.1 53              # 测试 UDP 端口
telnet 192.168.1.1 80               # 测试端口（传统方式）

# 查看 ARP 表
ip neigh
arp -n
```

# 4. 网络性能

## 4.1 带宽测试

```bash
# iperf3 — 带宽测试
# 服务端
iperf3 -s

# 客户端
iperf3 -c 192.168.1.100
iperf3 -c 192.168.1.100 -P 4 -t 30  # 4 并发，30 秒
```

## 4.2 MTU 与性能

```bash
# 测试路径 MTU
ping -M do -s 1472 google.com        # 1472 + 28(ICMP+IP) = 1500

# Jumbo Frames（9000 MTU）在数据中心使用
ip link set eth0 mtu 9000
```

## 4.3 网络性能优化参数

```bash
# 查看和调整网络内核参数
sysctl net.core.rmem_max
sysctl net.core.wmem_max

# 增加 TCP 缓冲区
sysctl -w net.ipv4.tcp_rmem="4096 87380 16777216"
sysctl -w net.ipv4.tcp_wmem="4096 65536 16777216"

# 启用 TCP BBR 拥塞控制（需要内核 4.9+）
sysctl -w net.core.default_qdisc=fq
sysctl -w net.ipv4.tcp_congestion_control=bbr
```

# 5. 高级网络功能

## 5.1 网卡绑定（Bonding）

将多块物理网卡绑定为一块逻辑网卡，提供冗余和带宽聚合：

```bash
# 创建 bond 接口
ip link add bond0 type bond mode 802.3ad
ip link set eth0 master bond0
ip link set eth1 master bond0
ip addr add 192.168.1.100/24 dev bond0
ip link set bond0 up
```

| 模式 | 名称 | 说明 |
|------|------|------|
| 0 | balance-rr | 轮询（负载均衡 + 容错）|
| 1 | active-backup | 主备模式（容错）|
| 2 | balance-xor | XOR 哈希（负载均衡）|
| 3 | broadcast | 广播（容错）|
| 4 | 802.3ad | 动态链路聚合（LACP）|
| 5 | balance-tlb | 自适应传输负载均衡 |
| 6 | balance-alb | 自适应负载均衡 |

## 5.2 网桥（Bridge）

用于虚拟机/容器网络和软件交换机：

```bash
# 创建网桥
ip link add br0 type bridge
ip link set eth0 master br0
ip addr add 192.168.1.100/24 dev br0
ip link set br0 up
```

## 5.3 网络命名空间（Network Namespace）

实现容器网络隔离的核心技术：

```bash
# 创建网络命名空间
ip netns add ns1
ip netns list

# 在命名空间中执行命令
ip netns exec ns1 ip addr
ip netns exec ns1 ping 8.8.8.8

# 创建 veth 对连接两个命名空间
ip link add veth0 type veth peer name veth1
ip link set veth0 netns ns1
ip link set veth1 netns ns2
```
