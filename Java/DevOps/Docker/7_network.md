
# 1. 网络问题的常见本质

容器网络排障最有效的方式是先把问题分类：

1. **容器到外网不通**：NAT、DNS、宿主机路由/防火墙、代理。
2. **容器到容器不通**：是否在同一网络、DNS 解析、端口是否监听、网络隔离。
3. **外部到容器不通**：端口映射、宿主机防火墙、安全组、应用监听地址。

排查时先判断属于哪一类，再逐跳检查：容器内 `curl localhost:端口`（应用是否监听）→ 同网络另一容器 `curl 服务名:端口`（服务发现是否通）→ 宿主机 `curl 宿主机IP:映射端口`（端口映射是否生效）→ 外部客户端（防火墙是否放行）。本篇覆盖 Docker 单机网络（bridge/host/none/macvlan），overlay 偏编排层（Swarm/K8s），最后说明 Docker 网络与 Kubernetes CNI 的关系。

# 2. 网络模式总览

Docker daemon 内部通过 libnetwork 提供网络抽象，一个"网络"由三部分组成：**驱动**（bridge/overlay/host/none/macvlan，负责在容器 netns 里接线）、**IPAM**（分配 IP 与网关）、**沙箱**（每个容器一份，本质是一个 Network Namespace）。容器加入网络时，Docker 创建一对 veth，一端留在网桥，另一端塞进容器 netns。理解这个结构后，下面的对比表就是不同"接线方式"的取舍：

| 模式 | 说明 | 外部访问 | 容器间通信 | 典型场景 |
|------|------|----------|------------|----------|
| **bridge** | 默认网桥 + NAT | 需要端口映射 `-p` | 同网络可互通 | 单机多容器、Compose 默认 |
| **host** | 共享宿主机网络栈 | 直接用宿主机端口 | 与宿主机等同 | 极致性能、特殊网络需求 |
| **none** | 无网络（仅 lo） | 不可 | 不可 | 强隔离/纯离线任务 |
| **macvlan** | 容器像"独立主机"在二层网络 | 直接分配 LAN IP | 同 VLAN 内直连 | 需要真实 IP、对 NAT 敏感 |
| **overlay** | 跨主机虚拟网络 | 可 | 可（跨主机） | Swarm/K8s（概念到此为止） |

> **行业实践**：生产单机部署普遍用"自定义 bridge + 反向代理（Nginx/Traefik）终结入口流量"，而不是让每个业务容器各自 `-p` 暴露端口。入口收敛到代理后，防火墙只需要开 80/443，服务端口变更也不再影响外部。

# 3. bridge 网络：默认与自定义的区别

## 3.1 为什么推荐自定义 bridge

在自定义 bridge 网络下：

1. Docker 提供内置 DNS，容器可以通过**容器名**互相访问。
2. 不同网络之间默认隔离，边界更清晰。

```bash
docker network create my-net
docker run -d --name db --network my-net mysql:8
docker run -d --name app --network my-net my-app:1.0

# app 内可以直接访问 db
docker exec -it app sh
# ping db
```

内置 DNS 的实现是：容器 `/etc/resolv.conf` 指向 `127.0.0.11`，这是 Docker 在每个 netns 里起的嵌入式 DNS 服务器，先查本网络内的容器名与别名，解析不了再转发给上游 DNS。

**设计考量**：

- **服务发现按网络划分作用域**：只有在同一自定义网络里的容器才能按名字互相解析，跨网络默认连 IP 都不通。这让"哪些服务允许互访"变成网络拓扑问题，而不是防火墙规则问题。
- **一个容器可以加入多个网络**：`docker network connect` 可运行时追加，典型用法是网关容器同时挂"对外网络"与"内部网络"，业务容器只挂内部网络。
- **IPAM 可控**：自定义网络可指定子网与网关，避免和机房网段冲突：`docker network create --subnet 172.28.0.0/16 --gateway 172.28.0.1 prod-net`。另有 `--internal` 网络，只允许容器间通信、禁止访问外网。

## 3.2 默认 bridge 的常见问题

默认 bridge 也能用，但工程体验明显更差：

1. **没有基于名称的 DNS**：挂在 `docker0` 上的容器只能按 IP 互访，容器重建后 IP 变化，硬编码 IP 必然失效。旧版靠 `--link` 注入 hosts，该机制已被废弃。
2. **所有默认 bridge 容器彼此互通**：缺少网络边界，多项目混用一台机器时隔离形同虚设。
3. **网段固定为 172.17.0.0/16**：可能与机房内网、VPN 网段冲突。

> **注意**：Compose v2 会为项目自动创建默认工程网络，`docker compose` 拉起的服务天然有服务名 DNS。手写 `docker run` 时如果没有显式 `--network`，容器就落在默认 bridge 上，此时按名字访问会失败——这是"本地 Compose 能跑、裸 docker run 互访不通"的经典根因。

# 4. 端口映射：外部访问容器的关键

端口映射把"宿主机端口"转发到"容器端口"。

```bash
docker run -p 8080:8080 my-app:1.0
docker run -p 127.0.0.1:8080:8080 my-app:1.0   # 只绑回环，配合本机反代
docker run -p 8000-8010:8000-8010 my-app:1.0   # 批量端口
docker port <container>
```

`-p` 默认绑定 `0.0.0.0`，即宿主机所有网卡都能访问；如果只想让本机 Nginx 访问后端，绑 `127.0.0.1` 是成本最低的收敛手段。

常见坑：

1. 应用只监听 `127.0.0.1`，导致容器外访问不到。容器内服务应监听 `0.0.0.0`——Spring Boot 的 `server.address`、Nginx 的 `listen` 都要检查。
2. 映射端口被宿主机其它进程占用，`docker run` 直接报 `address already in use`。
3. 宿主机防火墙/安全组拦截。
4. **hairpin 问题**（旧版环境）：容器通过 `-p` 映射的宿主机地址访问邻居服务时，回包路径需要内核支持 hairpin NAT，现在通常不再需要手工干预。

# 5. 端口映射的底层：iptables 与 NAT

`-p 8080:8080` 并不是 Docker 起了一个用户态转发进程，而是**在内核 netfilter 里插入 DNAT 规则**。理解这条链路，网络排障就从玄学变成查表：

```text
外部请求: client -> 宿主机 192.168.1.10:8080

┌────────────────────────── 宿主机 ──────────────────────────┐
│ PREROUTING(nat/DOCKER 链): DNAT 8080/tcp -> 172.17.0.3:8080│
│                 ▼                                          │
│ FORWARD 过滤链: 放行(DOCKER 链 / --icc)                      │
│                 ▼ docker0 网桥 ── veth pair ──┐             │
│  ┌──────────────┐                             │             │
│  │ 容器 netns    │◄────────────────────────────┘             │
│  │ 172.17.0.3   │ 回程报文由 conntrack 还原后返回 client      │
│  └──────────────┘                                           │
│ 容器出网: POSTROUTING MASQUERADE 源地址改写为宿主机 IP         │
└─────────────────────────────────────────────────────────────┘
```

验证方式：`sudo iptables -t nat -L DOCKER -n -v`，典型输出 `DNAT tcp -- 0.0.0.0/0 tcp dpt:8080 to:172.17.0.3:8080`。

Docker 24+ 还引入了 **DOCKER-USER 链**：Docker 自管的链（DOCKER、DOCKER-CT 等）会在 daemon 重启时重建，写在里面的自定义规则会丢；自定义 FORWARD 过滤应放进 `DOCKER-USER`，这条链 Docker 不会动。

> **注意**：Docker 默认会写 iptables，与 firewalld/ufw 混用时可能出现"防火墙没放行但端口却通了"或"docker 重启后自建规则丢失"的错觉。原则是：**放行策略写在 DOCKER-USER 或上游安全组，不要和 Docker 抢 nat 表**。另外 daemon 级的 `--icc=false` 会把同网桥容器互访收紧为"只通显式别名"，安全基线高的环境会打开它。

# 6. DNS：容器能 ping IP 但不能解析域名

Docker 会给容器生成 `/etc/resolv.conf`，并通过内置 DNS/转发完成解析。排查入口：`docker exec my-app cat /etc/resolv.conf`，再用 `getent hosts my-db`（比 nslookup 更接近 glibc 的实际解析路径）。

能 ping 通 IP 但域名解析失败，基本锁定在 DNS 层，按顺序检查：容器 resolv.conf 指向（127.0.0.11 是否正常转发）、daemon 上游 DNS、镜像是否缺 `libnss_*`（alpine 的 musl 解析行为与 glibc 不同）。

自定义 DNS（容器级）：`docker run --dns=8.8.8.8 --dns=114.114.114.114 --dns-search=corp.example.com my-app:1.0`。

全局 DNS（daemon 级别，`/etc/docker/daemon.json`）：

```json
{
  "dns": ["8.8.8.8", "114.114.114.114"],
  "dns-search": ["corp.example.com"],
  "dns-options": ["ndots:3", "timeout:2", "attempts:2"]
}
```

**ndots**：resolv.conf 里 `options ndots:5`（很多镜像默认值）意味着点数少于 5 的域名会先拼接 search 域再直查，search 域多时会造成解析延迟抖动；遇到"偶发解析慢"应先检查 search 与 ndots 的组合。另外自定义网络支持**网络别名**：`docker network connect --alias db-slave my-net mysql`。

# 7. host 网络：性能与风险

```bash
docker run --network host nginx
```

特点：

1. 无 NAT，性能更接近宿主机原生。
2. 端口直接占用宿主机端口，容易冲突。
3. 网络隔离边界变弱，不建议作为默认。

host 模式下容器共享宿主机 netns，`docker port` 没有意义，`-p` 会被忽略——两个同样监听 8080 的服务在 bridge 模式下互不影响，在 host 模式下第二个会直接起不来。另外 **Linux 之外没有真正的 host 网络**：Docker Desktop 跑在虚拟机里，`--network host` 指的是 VM 的网络栈，跨平台开发时不要依赖它。

# 8. macvlan：让容器拥有局域网 IP

macvlan 让容器在二层网络像一台独立机器，适合需要直接被局域网访问、或对 NAT 敏感的场景（如做源 IP 审计、组播、协议探测的遗留系统）。

```bash
docker network create -d macvlan \
  --subnet=192.168.1.0/24 --gateway=192.168.1.1 \
  --ip-range=192.168.1.128/25 -o parent=eth0 lan-net

docker run -d --network lan-net --ip=192.168.1.130 my-app:1.0
```

注意：

1. 需要网络侧允许额外 MAC/IP。**公有云 VPC 默认会丢弃"来源 MAC 不是本机弹性网卡"的报文**，macvlan 在云上需要先开启厂商的混杂模式支持，否则容器看似有 IP 却完全不通。
2. 宿主机与 macvlan 容器互访通常需要额外配置（很多人会踩这个坑）：宿主机与容器同网段通信时，内核会在父接口和 macvlan 子接口之间二选一，常常发不出去。标准解法是让宿主机也挂一个 macvlan 子接口接入同一网络：

```bash
ip link add macvlan-host link eth0 type macvlan mode bridge
ip addr add 192.168.1.192/32 dev macvlan-host
ip link set macvlan-host up
ip route add 192.168.1.130/32 dev macvlan-host   # 指向容器 IP
```

# 9. overlay 与 Kubernetes CNI

Swarm 的 overlay 网络用 VXLAN 做跨主机隧道：服务名解析返回 VIP（虚拟服务 IP），VIP 经 IPVS 负载均衡到各节点的真实任务。只需记住：**容器名/服务名 DNS 只在所属网络内有效**，与 bridge 的作用域规则一致。Docker 与 K8s 的网络模型则是两套体系：

| 维度 | Docker CNM（libnetwork） | Kubernetes CNI |
|------|--------------------------|----------------|
| 抽象 | Network/Sandbox/Endpoint，由 dockerd 内置管理 | 只做"接线"接口，由 CNI 插件（bridge/flannel/Calico/Cilium）实现 |
| 服务发现 | 内置 DNS（127.0.0.11），按网络作用域 | CoreDNS + Service 环境变量/Headless DNS 记录 |
| 负载均衡 | Swarm 的 VIP/IPVS | kube-proxy 的 iptables/IPVS/nftables 模式 |
| 端口发布 | `-p` DNAT（nat 表 DOCKER 链） | NodePort/LoadBalancer，portmap 插件做 DNAT |
| 跨主机 | overlay（VXLAN） | 插件决定：VXLAN（flannel）、BGP（Calico）、eBPF（Cilium） |

关键演进事实：**Kubernetes 从 1.24 起移除 dockershim**，kubelet 通过 CRI（containerd/CRI-O）直接管理 Pod，Pod 内容器共享同一个 netns（由 pause 容器持有），网络接线完全交给 CNI 插件。也就是说 K8s 里"每个 Pod 一个 IP、Pod 间直通、无 NAT"的扁平网络，是 CNI 插件实现的，而不是 Docker 网络。

> **行业实践**：Docker 网段（172.17.0.0/16）与机房网段冲突时，不要改 docker0（影响存量容器），而是新建自定义 bridge 指定不冲突的 `--subnet`，并在 daemon.json 里用 `default-address-pools` 约束后续自动创建的网段。

# 10. 常用命令与快速排障清单

```bash
docker network ls
docker network inspect my-net        # 排障第一入口: 本网段所有容器的 IP、别名与接入端点
docker network create|rm my-net
docker network connect|disconnect my-net my-app
bridge link show master docker0      # 宿主机视角看网桥与 veth
sudo iptables -t nat -L DOCKER -n -v # 查 DNAT 规则
```

## 10.1 外部访问不到容器

检查点：`-p` 是否映射正确；容器内服务是否监听 `0.0.0.0`（容器里 `ss -lntp` 确认）；宿主机端口是否被占用/防火墙拦截；`iptables -t nat -L DOCKER` 里 DNAT 规则是否存在。

## 10.2 容器访问外网失败

检查点：DNS 是否可用（`getent hosts example.com`，再用 `curl --resolve` 绕过 DNS 验证连通性）；宿主机能否出网（是否需要代理）；容器是否误挂在 `--internal` 网络（internal 天生无外网出口，是特性不是故障）。

## 10.3 容器间访问失败

检查点：是否同一自定义网络（`docker network inspect` 双方都要出现）；服务名是否正确，跨网络 DNS 作用域不覆盖；对端端口是否监听、应用是否就绪；daemon 是否开了 `--icc=false` 或 DOCKER-USER 链是否拦截 FORWARD。

# 11. 面试要点

| 问题 | 答题关键词 |
|------|------------|
| bridge 网络是什么 | 网桥 + NAT，端口映射对外暴露 |
| 为什么推荐自定义 bridge | 内置 DNS、容器名互访、隔离更清晰 |
| host 网络的优缺点 | 性能好、无 NAT；隔离弱、端口冲突 |
| 端口映射底层原理 | iptables DNAT（DOCKER 链）+ MASQUERADE + FORWARD 放行 |
| 容器外访问不到常见原因 | 端口映射/监听地址/防火墙 |
| DNS 常见故障现象 | 能 ping IP 但域名解析失败；127.0.0.11 嵌入式 DNS |

# 12. 小结

Docker 网络的核心心智模型是：**网络 = 一个 netns 接线方案 + 一段 IPAM 地址池 + 一个作用域边界**。bridge 模式用 veth 接进网桥、用 iptables DNAT 对外发布端口、用嵌入式 DNS 做按网络作用域的服务发现；host/macvlan/overlay 对应性能、二层可达、跨主机三种取舍。排障按"应用监听 → 同网 DNS → 宿主机映射 → 外部防火墙"逐跳收敛。默认 bridge 缺 DNS 与隔离，工程上一律用自定义 bridge；K8s 不用 Docker 的 CNM，Pod 网络由 CNI 插件负责，但 veth/conntrack/DNAT 这些底层机制在两个体系中是同一套东西。
