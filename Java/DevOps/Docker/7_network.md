
# 1. 网络问题的常见本质

容器网络排障最有效的方式是先把问题分类：

1. **容器到外网不通**：NAT、DNS、宿主机路由/防火墙、代理。
2. **容器到容器不通**：是否在同一网络、DNS 解析、端口是否监听、网络隔离。
3. **外部到容器不通**：端口映射、宿主机防火墙、安全组、应用监听地址。

这章重点覆盖 Docker 单机网络（bridge/host/none/macvlan）。overlay 更偏编排层（Swarm/K8s），在这里以概念为主。

# 2. 网络模式总览

| 模式 | 说明 | 外部访问 | 容器间通信 | 典型场景 |
|------|------|----------|------------|----------|
| **bridge** | 默认网桥 + NAT | 需要端口映射 `-p` | 同网络可互通 | 单机多容器、Compose 默认 |
| **host** | 共享宿主机网络栈 | 直接用宿主机端口 | 与宿主机等同 | 极致性能、特殊网络需求 |
| **none** | 无网络（仅 lo） | 不可 | 不可 | 强隔离/纯离线任务 |
| **macvlan** | 容器像“独立主机”在二层网络 | 直接分配 LAN IP | 同 VLAN 内直连 | 需要真实 IP、对 NAT 敏感 |
| **overlay** | 跨主机虚拟网络 | 可 | 可（跨主机） | Swarm/K8s（概念到此为止） |

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

## 3.2 默认 bridge 的常见问题

默认 bridge 也能用，但“服务发现”和隔离体验通常更差；工程上更推荐显式创建自定义网络。

# 4. 端口映射：外部访问容器的关键

端口映射把“宿主机端口”转发到“容器端口”。

```bash
docker run -p 8080:8080 my-app:1.0
docker port ccontainere
```

常见坑：

1. 应用只监听 `127.0.0.1`，导致容器外访问不到。容器内服务应监听 `0.0.0.0`。
2. 映射端口被宿主机其它进程占用。
3. 宿主机防火墙/安全组拦截。

# 5. DNS：容器能 ping IP 但不能解析域名

Docker 会给容器生成 `/etc/resolv.conf`，并通过内置 DNS/转发完成解析。

排查入口：

```bash
docker exec my-app cat /etc/resolv.conf
docker exec -it my-app sh
# nslookup example.com
```

自定义 DNS（容器级别）：

```bash
docker run --dns=8.8.8.8 --dns=114.114.114.114 my-app:1.0
```

全局 DNS（daemon 级别）：

```json
{
  "dns": ["8.8.8.8", "114.114.114.114"]
}
```

# 6. host 网络：性能与风险

```bash
docker run --network host nginx
```

特点：

1. 无 NAT，性能更接近宿主机原生。
2. 端口直接占用宿主机端口，容易冲突。
3. 网络隔离边界变弱，不建议作为默认。

# 7. macvlan：让容器拥有局域网 IP

macvlan 让容器在二层网络像一台独立机器，适合需要直接被局域网访问、或对 NAT 敏感的场景。

注意：

1. 需要网络侧允许额外 MAC/IP。
2. 宿主机与 macvlan 容器互通常需要额外配置（很多人会踩这个坑）。

# 8. 常用网络命令

```bash
docker network ls
docker network inspect my-net
docker network create my-net
docker network rm my-net

docker network connect my-net my-app
docker network disconnect my-net my-app
```

# 9. 快速排障清单（按现象）

## 9.1 外部访问不到容器

```bash
docker ps
docker port my-app
docker logs my-app
docker exec -it my-app sh
```

检查点：

1. `-p` 是否映射正确。
2. 容器内服务是否监听 `0.0.0.0`。
3. 宿主机端口是否被占用/被防火墙拦截。

## 9.2 容器访问外网失败

检查点：

1. DNS 是否可用（先解析再连通）。
2. 宿主机是否能访问外网（以及是否需要代理）。
3. 是否被企业网络策略限制。

## 9.3 容器间访问失败

检查点：

1. 是否在同一个自定义网络。
2. 服务名是否正确（Compose 下通常用 service name）。
3. 对端容器端口是否监听、应用是否就绪。

# 10. 面试要点

| 问题 | 答题关键词 |
|------|------------|
| bridge 网络是什么 | 网桥 + NAT，端口映射对外暴露 |
| 为什么推荐自定义 bridge | 内置 DNS、容器名互访、隔离更清晰 |
| host 网络的优缺点 | 性能好、无 NAT；隔离弱、端口冲突 |
| 容器外访问不到常见原因 | 端口映射/监听地址/防火墙 |
| DNS 常见故障现象 | 能 ping IP 但域名解析失败 |
