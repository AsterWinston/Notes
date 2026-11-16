单播是一对一，广播和组播是一对多。TCP 只支持单播，因为它要为每对通信维护连接状态；UDP 天然支持把同一个数据报发给多台主机。这让局域网设备发现、集群心跳、直播分发、服务注册这些场景有了不同于"逐个连接"的实现方式。但广播和组播也是最容易被网络环境坑到的部分：家宽的交换机可能过滤广播，云厂商的 VPC 默认不转发组播，笔记本插着 Wi-Fi 和有线两张网卡时，包从哪张网卡出去决定了它能不能被对端看见。

# 1. 广播与组播的定位

## 1.1 一对一的局限

假设要让局域网内 100 台设备知道自己的存在。用 TCP 或 UDP 单播，就得维护 100 个目标地址，逐个发送，发送方要保存清单，新设备上线还得被发现。广播和组播把"发给谁"交给网络层处理，发送方只发一份，由网络负责复制或分发。

```text
单播：发送方 ──► 设备 1
              ──► 设备 2
              ──► 设备 3      发送 N 份，维护 N 个地址

广播/组播：发送方 ──► 网络 ──┬──► 设备 1
                            ├──► 设备 2
                            └──► 设备 3      发送 1 份，网络负责分发
```

## 1.2 广播

广播把数据报发给**同一个广播域**内的所有主机。IPv4 的广播地址由子网掩码决定，比如 `192.168.1.0/24` 的广播地址是 `192.168.1.255`。广播不会跨路由器转发（除非路由器显式配置了定向广播转发，这通常被关闭）。

**广播的问题**：它打扰广播域内的所有主机，包括不关心这个服务的设备。1000 台机器的局域网里，一个每秒一次的广播发现包会带来 1000 次无谓的中断和 CPU 唤醒。这是组播要解决的问题。

## 1.3 组播

组播把数据报只发给**加入了特定组**的主机。组播地址是一个逻辑标识，主机通过 IGMP 告诉路由器"我要加入这个组"，网络只在这个组有成员的分支上复制数据。

| 维度 | 广播 | 组播 |
|------|------|------|
| 目标 | 广播域内所有主机 | 加入组的主机 |
| 地址范围 | 子网广播地址 | 224.0.0.0 到 239.255.255.255 |
| 跨路由器 | 默认不转发 | 需要 IGMP + 组播路由 |
| 对无关主机的打扰 | 有 | 无 |
| 网卡过滤 | 全部上交内核 | 网卡按组播地址过滤 |
| 适用 | 极小局域网、DHCP 发现 | 服务发现、直播、集群 |

**设计考量**：广播适合"还不知道谁在关心"的初始发现阶段（比如 DHCP Discover），组播适合"已知有一群订阅者"的持续分发。很多服务发现协议会先用广播或固定组播地址做第一次探测，再转为单播。

# 2. 广播

## 2.1 广播地址

IPv4 的广播分三种：

| 类型 | 地址 | 范围 |
|------|------|------|
| 受限广播 | `255.255.255.255` | 只在本广播域，不跨路由器 |
| 定向广播 | `192.168.1.255` | 指定子网的广播地址 |
| 全子网广播 | 由掩码计算 | 同上 |

`255.255.255.255` 不需要知道本机 IP 和掩码，常用于 DHCP 这类"还没配好地址"的场景。定向广播需要知道子网，但更容易被路由器正确处理。

## 2.2 本地广播与定向广播

发送广播需要打开 `SO_BROADCAST` 选项，否则内核会拒绝：

```java
import java.net.DatagramPacket;
import java.net.DatagramSocket;
import java.net.InetAddress;
import java.nio.charset.StandardCharsets;

public class BroadcastSender {
    public static void main(String[] args) throws Exception {
        try (DatagramSocket socket = new DatagramSocket()) {
            // 不开启这个选项，发送广播会抛 SocketException
            socket.setBroadcast(true);

            byte[] data = "DISCOVER".getBytes(StandardCharsets.UTF_8);
            InetAddress broadcast = InetAddress.getByName("255.255.255.255");
            DatagramPacket packet = new DatagramPacket(data, data.length, broadcast, 9000);
            socket.send(packet);
            System.out.println("广播已发送");
        }
    }
}
```

接收广播不需要特殊设置，只要绑定对应端口即可：

```java
try (DatagramSocket socket = new DatagramSocket(9000)) {
    byte[] buf = new byte[1024];
    while (true) {
        DatagramPacket p = new DatagramPacket(buf, buf.length);
        socket.receive(p);
        String msg = new String(p.getData(), p.getOffset(), p.getLength(),
                               StandardCharsets.UTF_8);
        System.out.println("来自 " + p.getAddress() + ": " + msg);
    }
}
```

**注意**：发送方和接收方不能绑在同一个端口上运行在同一台机器——`new DatagramSocket(9000)` 会占用端口，发送方再绑定 9000 就会失败。通常发送方用临时端口，接收方固定端口。

## 2.3 Java 发送与接收广播

一个完整的广播回声示例需要两个进程：接收方固定绑定端口，发送方打开 `setBroadcast` 并指定广播地址。

```text
┌──────────────┐  255.255.255.255:9000   ┌──────────────┐
│ 广播发送方     │ ─────────────────────► │ 接收方 A      │
│ 临时端口       │ ─────────────────────► │ 接收方 B      │
└──────────────┘                         └──────────────┘
```

**注意**：`255.255.255.255` 在很多系统上只从**主网卡**发出。多网卡机器（VPN、Docker、虚拟机网卡）上，广播可能走错接口。需要精确控制时，用定向广播地址，或者用 `MulticastSocket` 的 `setNetworkInterface` 指定网卡。

# 3. 组播与 IGMP

## 3.1 组播地址

IPv4 组播地址范围是 `224.0.0.0` 到 `239.255.255.255`（D 类地址）。其中有一段是保留的：

| 地址 | 用途 | 是否可路由 |
|------|------|-----------|
| `224.0.0.1` | 所有主机 | 否，仅本地链路 |
| `224.0.0.2` | 所有路由器 | 否 |
| `224.0.0.9` | RIPv2 | 否 |
| `224.0.1.0` 到 `238.255.255.255` | 可路由组播 | 是，需要组播路由 |
| `239.0.0.0/8` | 管理范围地址 | 由管理员划分 |

自己选组播地址时，避开 `224.0.0.x`，通常用 `239.x.x.x` 这类管理范围地址，减少和公共协议冲突。

## 3.2 IGMP 的作用

IGMP（Internet Group Management Protocol，互联网组管理协议）是主机和本地路由器之间的协议。主机想接收某个组播组时，发 IGMP 报告说"我要加入 239.1.1.1"；路由器记录这个组在哪个接口下有成员，只把组播流量转发到有成员的分支。

```text
        ┌─────────────┐
        │  组播源      │
        └──────┬──────┘
               │ 组播流量
        ┌──────▼──────┐
        │  路由器      │  记录：接口 A 有 239.1.1.1 成员
        └──┬───────┬──┘
           │       │
     接口 A │       │ 接口 B（无成员，不转发）
           ▼       ▼
      ┌────────┐  ┌────────┐
      │ 主机 1  │  │ 主机 2  │
      │ 已加入   │  │ 未加入   │
      └────────┘  └────────┘
```

IGMP 有三个版本：v1 只支持加入、离开靠超时；v2 增加了显式离开；v3 支持指定源过滤（只接收来自特定源的组播）。交换机上还有 **IGMP Snooping**，让交换机监听 IGMP 报文，只把组播转发到有成员的端口，避免泛洪。

## 3.3 组播的转发路径

组播能否跨子网，取决于沿途路由器是否运行组播路由协议（PIM、DVMRP 等）。这也是组播在公网上几乎不可用的原因：绝大多数 ISP 不转发组播，企业网和云环境也常常默认关闭。

# 4. MulticastSocket

## 4.1 加入与退出组

`MulticastSocket` 继承自 `DatagramSocket`，额外提供了加入组、设置 TTL、选择网卡等能力。

```java
import java.net.InetAddress;
import java.net.InetSocketAddress;
import java.net.MulticastSocket;
import java.net.NetworkInterface;
import java.nio.charset.StandardCharsets;

public class MulticastReceiver {
    public static void main(String[] args) throws Exception {
        InetAddress group = InetAddress.getByName("239.1.1.1");
        // 绑定端口；构造参数用 null 表示稍后 bind
        try (MulticastSocket socket = new MulticastSocket(9000)) {
            // 指定从哪张网卡加入组，多网卡环境必须显式指定
            NetworkInterface nic = NetworkInterface.getByInetAddress(
                    InetAddress.getByName("192.168.1.10"));
            socket.joinGroup(new InetSocketAddress(group, 9000), nic);

            byte[] buf = new byte[2048];
            DatagramPacket packet = new DatagramPacket(buf, buf.length);
            while (true) {
                socket.receive(packet);
                String msg = new String(packet.getData(), packet.getOffset(),
                                        packet.getLength(), StandardCharsets.UTF_8);
                System.out.println("组播消息: " + msg);
            }
        }
    }
}
```

**注意**：`joinGroup(InetAddress)` 这种单参数写法依赖系统选择默认网卡，在有多张网卡（Wi-Fi、有线、Docker、VPN）时经常选错。Java 提供了带 `NetworkInterface` 参数的重载，生产代码应始终显式指定。

## 4.2 发送组播

```java
import java.net.DatagramPacket;
import java.net.InetAddress;
import java.net.MulticastSocket;
import java.net.NetworkInterface;
import java.nio.charset.StandardCharsets;

public class MulticastSender {
    public static void main(String[] args) throws Exception {
        InetAddress group = InetAddress.getByName("239.1.1.1");
        try (MulticastSocket socket = new MulticastSocket()) {
            // 限制组播只在本局域网内传播
            socket.setTimeToLive(1);

            byte[] data = "HELLO GROUP".getBytes(StandardCharsets.UTF_8);
            DatagramPacket packet = new DatagramPacket(data, data.length, group, 9000);
            socket.send(packet);
        }
    }
}
```

发送端不一定要加入组，它只需要向组播地址发送。但如果它也想收到自己发的消息（回环），则需要加入组或开启回环。

## 4.3 TTL 与回环

| 选项 | 方法 | 含义 |
|------|------|------|
| TTL | `setTimeToLive(int)` | 组播能跨多少个路由器，1 表示只在本子网 |
| 回环 | `setLoopbackMode(true)` | 关闭回环后，本机发送的组播不会回到本机 |
| 网卡 | `setNetworkInterface(nic)` | 指定发送和加入组使用的网卡 |

```java
// 只在本局域网传播，默认值就是 1
socket.setTimeToLive(1);
// 允许本机收到自己发的组播（默认允许）
socket.setLoopbackMode(false);
// 指定发送网卡，避免从错误的接口出去
socket.setNetworkInterface(NetworkInterface.getByName("eth0"));
```

**注意**：`setLoopbackMode(true)` 的语义是"**禁用**回环"，参数为 `true` 反而关闭。这个反直觉的命名经常导致测试时"本机收不到自己发的包"。

# 5. 局域网发现实战

## 5.1 协议设计

一个最小的服务发现协议可以这样设计：

```text
┌──────────┬──────────┬──────────────┐
│ 魔数 4B   │ 类型 1B   │ 服务名 + 端口  │
│ UDPDISC  │ 1=查询    │ UTF-8 字符串   │
│          │ 2=响应    │              │
└──────────┴──────────┴──────────────┘
```

客户端向组播地址 `239.1.1.1:9000` 发查询，服务端收到后单播回自己的 IP 和端口。这样既利用了组播的一对多，又避免响应也走组播造成风暴。

```text
客户端                                    服务端（多个）
  │ ── 组播 QUERY ──────────────────────► │
  │                                       │ 收到查询
  │ ◄── 单播 RESPONSE(ip, port) ───────── │
  │ ◄── 单播 RESPONSE(ip, port) ───────── │
```

## 5.2 服务端

```java
import java.net.*;
import java.nio.charset.StandardCharsets;

public class DiscoveryServer {
    public static void main(String[] args) throws Exception {
        InetAddress group = InetAddress.getByName("239.1.1.1");
        try (MulticastSocket socket = new MulticastSocket(9000)) {
            NetworkInterface nic = NetworkInterface.getByInetAddress(
                    InetAddress.getLocalHost());
            socket.joinGroup(new InetSocketAddress(group, 9000), nic);

            byte[] buf = new byte[1024];
            while (true) {
                DatagramPacket p = new DatagramPacket(buf, buf.length);
                socket.receive(p);
                String req = new String(p.getData(), p.getOffset(), p.getLength(),
                                        StandardCharsets.UTF_8);
                if (!"QUERY".equals(req)) continue;

                // 单播回应，带上自己的服务端口
                String resp = "RESPONSE:" + 8080;
                byte[] data = resp.getBytes(StandardCharsets.UTF_8);
                socket.send(new DatagramPacket(data, data.length,
                        p.getAddress(), p.getPort()));
            }
        }
    }
}
```

## 5.3 客户端

```java
import java.net.*;
import java.nio.charset.StandardCharsets;

public class DiscoveryClient {
    public static void main(String[] args) throws Exception {
        InetAddress group = InetAddress.getByName("239.1.1.1");
        try (MulticastSocket socket = new MulticastSocket()) {
            socket.setTimeToLive(1);
            socket.setSoTimeout(1000);
            // 关闭回环，避免收到自己发的查询
            socket.setLoopbackMode(true);

            byte[] data = "QUERY".getBytes(StandardCharsets.UTF_8);
            socket.send(new DatagramPacket(data, data.length, group, 9000));

            byte[] buf = new byte[1024];
            long deadline = System.currentTimeMillis() + 2000;
            while (System.currentTimeMillis() < deadline) {
                DatagramPacket p = new DatagramPacket(buf, buf.length);
                try {
                    socket.receive(p);
                } catch (SocketTimeoutException e) {
                    break;
                }
                String resp = new String(p.getData(), p.getOffset(), p.getLength(),
                                         StandardCharsets.UTF_8);
                System.out.println("发现服务 " + p.getAddress() + " -> " + resp);
            }
        }
    }
}
```

## 5.4 多网卡问题

多网卡是组播最常见的故障源。笔记本同时连着 Wi-Fi 和有线、开了 Docker 或 VPN 时，系统可能从 `docker0` 或 `tun0` 发出组播，对端自然收不到。

排查步骤：

1. 用 `NetworkInterface.getNetworkInterfaces()` 列出所有网卡，找到有 `UP` 和 `MULTICAST` 标志的那个。
2. 发送和加入组时都显式 `setNetworkInterface(nic)`。
3. 在接收端用 `tcpdump -i <网卡> udp port 9000` 确认包是否真的到了该网卡。

```java
// 打印所有支持组播的网卡，用于定位
var interfaces = NetworkInterface.getNetworkInterfaces();
while (interfaces.hasMoreElements()) {
    NetworkInterface ni = interfaces.nextElement();
    if (ni.isUp() && ni.supportsMulticast()) {
        System.out.println(ni.getName() + " -> " + ni.getInterfaceAddresses());
    }
}
```

> **行业实践**：生产环境的服务发现很少直接用裸组播，而是用 mDNS（也是组播，但带 DNS 报文格式）、Consul、etcd 或 Kubernetes 的 Service。裸组播的问题在于跨网段不可用、云环境不支持、多网卡难配置，只适合单交换机的小型局域网。

# 6. TTL 与网络限制

## 6.1 TTL 的含义

组播的 TTL 和 IP 首部的 TTL 不是一回事。IP TTL 是"还能经过多少个路由器"，每跳减 1；组播 TTL 是发送方设置的"允许传播的范围"，路由器根据阈值决定是否转发。

| TTL | 范围 | 典型用途 |
|-----|------|----------|
| 0 | 仅本机 | 本机进程间测试 |
| 1 | 本子网 | 局域网发现（默认） |
| 2 到 31 | 同一园区 | 小范围企业网 |
| 32 到 127 | 同一区域 | 跨楼宇 |
| 128 到 255 | 全球 | 需要公网组播路由，几乎不可用 |

```java
socket.setTimeToLive(1);    // 只在本局域网
```

**注意**：TTL 设大了不会"更可靠"，只会让包传播到不该去的地方。局域网发现就应该用 1。

## 6.2 常见阻断

| 环境 | 现象 | 原因 |
|------|------|------|
| 家用路由器 | 跨网段收不到 | 不转发组播，且常过滤广播 |
| 企业交换机 | 同网段部分主机收不到 | 未开 IGMP Snooping，或 VLAN 隔离 |
| 云服务器 VPC | 完全不工作 | 虚拟网络不支持广播/组播 |
| 容器网络 | 只在同一 bridge 内可见 | Docker 默认不转发组播 |
| 跨公网 | 不可用 | ISP 不转发组播 |

## 6.3 云环境

AWS、阿里云、腾讯云的 VPC 默认不支持广播和组播。部分云厂商提供"组播域"或"广播功能"作为独立产品，需要显式开通，且通常限制在同一 VPC 内。在云上做服务发现，正确方向是使用云厂商提供的注册中心或自建 etcd/Consul，而不是指望组播。

> **行业实践**：Kubernetes 的 Service 和 CoreDNS 本质上是用中心化的方式替代了组播发现。这不是技术退步，而是因为组播在虚拟化网络里成本太高、可观测性太差。设计跨环境可用的发现协议时，应把组播作为"可选加速路径"，而不是唯一路径。

# 7. 常见错误

| 错误 | 后果 | 正确做法 |
|------|------|----------|
| 发送广播未 `setBroadcast(true)` | 抛 `SocketException` | 显式开启 |
| 组播未指定网卡 | 从错误接口发出，对端收不到 | 显式 `setNetworkInterface` |
| 误以为 `setLoopbackMode(true)` 是开启回环 | 本机收不到自己发的包 | 记住参数是"禁用" |
| TTL 设得过大 | 包扩散到无关网络 | 局域网用 1 |
| 在云 VPC 上用广播/组播 | 完全不工作 | 改用注册中心 |
| 接收方和发送方绑同一端口 | 端口冲突 | 发送方用临时端口 |
| 组播地址选 `224.0.0.x` | 与保留协议冲突 | 用 `239.x.x.x` |
| 忘记处理 IGMP 离开 | 组成员关系残留 | 退出时 `leaveGroup` |

# 8. 练习与自检

**练习一**：在本机启动两个接收进程，分别绑定 9000 和 9001，向 `255.255.255.255:9000` 发广播，观察哪个进程收到。再把目标改成 `239.1.1.1`，让两个进程分别加入组，验证组播只投递给加入者。

**练习二**：用 `tcpdump -i any -n udp port 9000` 抓包，分别在 TTL 为 1 和 2 时发送组播，观察 IP 首部 TTL 字段的变化，并解释"组播 TTL"和"IP TTL"的区别。

**练习三**：给发现协议加上"服务名过滤"：查询报文里带服务名，服务端只有匹配时才回应。思考如果局域网内有 100 个服务，全量响应会不会造成风暴，应该如何分片或随机延迟。

自检问题：`255.255.255.255` 和 `192.168.1.255` 有什么区别？IGMP 解决什么问题？为什么多网卡环境下组播经常"看起来发出去了但对方收不到"？

# 9. 小结

| 主题 | 记住这一句 |
|------|-----------|
| 广播 | 发给广播域内所有主机，不跨路由器，会打扰无关设备 |
| 组播 | 只发给加入组的主机，靠 IGMP 管理成员关系 |
| 地址 | 组播是 224.0.0.0 到 239.255.255.255，避开 224.0.0.x |
| MulticastSocket | 加入组、设 TTL、选网卡，多网卡必须显式指定 |
| TTL | 控制组播传播范围，局域网发现用 1 |
| 网络限制 | 云 VPC、容器、跨公网基本不支持，需改用注册中心 |
| 发现协议 | 组播查询 + 单播响应，避免响应风暴 |

广播和组播解决的是"局域网内一对多"，但绝大多数真实通信要跨公网，而公网中间隔着 NAT。UDP 在 NAT 下的映射生命周期短、方向受限，打洞和保活成了必须处理的问题。下一篇进入 NAT 穿越，讲清锥形与对称型、STUN/TURN 和 UDP 打洞的完整流程。