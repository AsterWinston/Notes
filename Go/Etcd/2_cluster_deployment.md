etcd 的部署难点不在"把进程跑起来"，而在"让三个节点对同一份集群身份达成一致"。端口写错、advertise 地址用了 127.0.0.1、旧 data-dir 没清、token 不一致，都会让集群停在"互相找不到"的状态。本篇从单机最小启动开始，逐步过渡到三节点集群，并把常见启动错误逐一拆开。

# 1. 部署前必须搞清的三件事

## 1.1 两个端口：client 与 peer

etcd 每个成员监听两类流量：

| 端口 | 名称 | 谁连它 | 用途 |
|------|------|--------|------|
| 2379 | client URL | 业务客户端、etcdctl、kube-apiserver | 读写 key、Watch、Lease |
| 2380 | peer URL | 其他 etcd 成员 | Raft 日志复制、心跳、选举 |

★ **关键认知**：client 端口和 peer 端口是两套完全独立的监听。只开放 2379 能让客户端连上，但集群之间无法复制日志；只开放 2380 则客户端无法读写。防火墙、安全组、Kubernetes NetworkPolicy 都要同时放行。

## 1.2 成员身份：三组 URL 别混

每个节点有四组容易混淆的地址：

| 参数 | 含义 | 常见错误 |
|------|------|----------|
| `--listen-client-urls` | 本机在哪个地址上监听客户端请求 | 写成公网 IP 导致本机起不来 |
| `--advertise-client-urls` | 告诉客户端"来这个地址找我" | 写成 127.0.0.1，其他机器连不上 |
| `--listen-peer-urls` | 本机在哪个地址上监听 peer 请求 | 只监听 127.0.0.1，集群跨机不通 |
| `--initial-advertise-peer-urls` | 告诉其他成员"来这个地址复制日志" | 与 `initial-cluster` 里的地址不一致 |

`--listen-*` 是"绑定的网卡地址"，`--advertise-*` 是"对外公布的地址"。单机可以都用 `127.0.0.1`；多机部署时 advertise 必须是其他机器能访问到的真实 IP 或域名。

## 1.3 数据目录与版本

- 每个节点的 `--data-dir` 必须独立，不能多个进程共享；首次启动会生成 `member/`，包含 WAL、快照和成员元数据。
- 已经初始化过的目录再用 `--initial-cluster-state new` 启动，etcd 会检测到已有集群身份并忽略引导参数，但更稳妥的做法是保持原配置、不要删数据。
- 集群内所有节点的 etcd 版本应保持一致或至少在同一小版本线上；跨大版本升级要按官方步骤逐个进行。

# 2. 单机模式

## 2.1 最小启动命令

先创建数据目录：

```bash
sudo mkdir -p /var/lib/etcd
sudo chown -R etcd:etcd /var/lib/etcd
```

用命令行直接启动：

```bash
etcd \
  --name etcd-single \
  --data-dir /var/lib/etcd \
  --listen-client-urls http://127.0.0.1:2379 \
  --advertise-client-urls http://127.0.0.1:2379 \
  --listen-peer-urls http://127.0.0.1:2380 \
  --initial-advertise-peer-urls http://127.0.0.1:2380 \
  --initial-cluster etcd-single=http://127.0.0.1:2380 \
  --initial-cluster-state new \
  --initial-cluster-token etcd-single-token
```

参数逐个解释：

- `--name`：成员名，集群内唯一，重启时必须保持一致。
- `--data-dir`：WAL 与快照的存放位置，必须可写。
- `--initial-cluster`：引导时的成员列表，单机只有自己。
- `--initial-cluster-state new`：表示这是新建集群；加入已有集群时用 `existing`。
- `--initial-cluster-token`：集群身份的一部分，防止不同集群的节点误连。

★ **关键认知**：单机模式也走完整的 Raft 流程，只是多数派就是自己。它适合开发和测试，**不能提供任何容错**：进程挂了或磁盘坏了，数据就不可用。

## 2.2 用配置文件启动

生产上参数多，建议写成 YAML：

```yaml
# /etc/etcd/etcd.yaml
name: etcd-single
data-dir: /var/lib/etcd
listen-client-urls: http://127.0.0.1:2379
advertise-client-urls: http://127.0.0.1:2379
listen-peer-urls: http://127.0.0.1:2380
initial-advertise-peer-urls: http://127.0.0.1:2380
initial-cluster: etcd-single=http://127.0.0.1:2380
initial-cluster-state: new
initial-cluster-token: etcd-single-token
quota-backend-bytes: 2147483648
auto-compaction-mode: periodic
auto-compaction-retention: "1h"
```

用 `etcd --config-file=/etc/etcd/etcd.yaml` 启动。字段名与命令行参数基本一一对应，省去了长命令换行和转义的问题。

## 2.3 验证单机

另开一个终端：

```bash
export ETCDCTL_API=3   # 3.5 默认就是 v3，老版本需要显式设置
etcdctl --endpoints=http://127.0.0.1:2379 endpoint health
etcdctl --endpoints=http://127.0.0.1:2379 put /demo/hello world
etcdctl --endpoints=http://127.0.0.1:2379 get /demo/hello
```

健康检查会返回 `is healthy: successfully committed proposal`，随后 put 返回 `OK`、get 返回 `world`。如果 `put` 超时，先看 etcd 日志里有没有 `no leader`、`bind` 或磁盘错误，不要盲目重试。

# 3. 三节点集群

## 3.1 节点规划

假设三台机器：

| 成员名 | IP | client URL | peer URL |
|--------|----|------------|----------|
| etcd-1 | 10.0.0.1 | http://10.0.0.1:2379 | http://10.0.0.1:2380 |
| etcd-2 | 10.0.0.2 | http://10.0.0.2:2379 | http://10.0.0.2:2380 |
| etcd-3 | 10.0.0.3 | http://10.0.0.3:2379 | http://10.0.0.3:2380 |

三台机器的时钟必须同步（NTP/chrony），否则 TLS 证书校验和日志时间会出现问题。磁盘优先 SSD，etcd 对 fsync 延迟非常敏感。

## 3.2 每个节点的启动参数

节点 1 的完整启动命令：

```bash
etcd \
  --name etcd-1 \
  --data-dir /var/lib/etcd \
  --listen-client-urls http://0.0.0.0:2379 \
  --advertise-client-urls http://10.0.0.1:2379 \
  --listen-peer-urls http://0.0.0.0:2380 \
  --initial-advertise-peer-urls http://10.0.0.1:2380 \
  --initial-cluster etcd-1=http://10.0.0.1:2380,etcd-2=http://10.0.0.2:2380,etcd-3=http://10.0.0.3:2380 \
  --initial-cluster-state new \
  --initial-cluster-token etcd-prod-cluster \
  --quota-backend-bytes 8589934592
```

节点 2 和节点 3 只改三个地方：

| 参数 | etcd-2 | etcd-3 |
|------|--------|--------|
| `--name` | etcd-2 | etcd-3 |
| `--advertise-client-urls` | http://10.0.0.2:2379 | http://10.0.0.3:2379 |
| `--initial-advertise-peer-urls` | http://10.0.0.2:2380 | http://10.0.0.3:2380 |

`--initial-cluster` 和 `--initial-cluster-token` 三个节点必须**完全一致**。`--listen-*` 用 `0.0.0.0` 表示监听所有网卡，`--advertise-*` 用具体 IP 表示对外公布地址。

## 3.3 初始化顺序

依次启动 etcd-1、etcd-2、etcd-3：第一个节点会打印 `waiting for other members`，第二个启动后达到多数派并开始选主，第三个加入后开始复制日志。启动先后没有严格要求，但不要让某个节点用不同的 token 或不同的 `initial-cluster` 启动，否则它会认为自己在另一个集群里。

```bash
etcdctl --endpoints=http://10.0.0.1:2379,http://10.0.0.2:2379,http://10.0.0.3:2379 \
  member list -w table
```

期望看到三行，`IS LEARNER` 为 false，状态正常。

## 3.4 加入已有集群用 existing

新增第四个节点时，不能在旧节点上改 `--initial-cluster` 后重启，而要用成员管理命令：

```bash
# 在原集群上登记新成员，命令会打印出新节点需要的 initial-cluster
etcdctl --endpoints=http://10.0.0.1:2379 member add etcd-4 \
  --peer-urls=http://10.0.0.4:2380
```

然后在新节点上用打印出来的 `ETCD_INITIAL_CLUSTER` 启动，把状态设为 `existing`，并只改 `--name`、`--advertise-client-urls`、`--initial-advertise-peer-urls` 三处。不要照抄旧节点的完整命令后直接启动，否则新节点会以错误身份加入。

★ **关键认知**：`member add` 之后，集群的 quorum 会立刻按新规模计算。3 节点加第 4 个节点时，多数派从 2 变成 3；如果新节点迟迟不启动，集群可能暂时失去写能力。因此加节点要尽快完成，生产上也可以先加 learner 再提升。

# 4. 健康检查与日常验证

## 4.1 etcdctl 三件套

```bash
ENDPOINTS=http://10.0.0.1:2379,http://10.0.0.2:2379,http://10.0.0.3:2379
# health 会发起真实读请求；status 看版本、DB 大小、Leader；member 看成员
etcdctl --endpoints=$ENDPOINTS endpoint health --cluster
etcdctl --endpoints=$ENDPOINTS endpoint status --cluster -w table
etcdctl --endpoints=$ENDPOINTS member list -w table
```

`endpoint health` 会发起一次真实的线性一致读，因此它比端口探测更能说明"集群是否能完成多数派确认"。如果某个端点返回 `context deadline exceeded`，但端口是通的，通常是该成员没有 Leader 或磁盘卡住。

## 4.2 HTTP 健康接口与日志

`curl -s http://10.0.0.1:2379/health` 返回 `{"health":"true","reason":""}`，但它只反映本成员能否服务，不检查整个集群的 quorum，不能当作集群健康的唯一依据。日常用 `journalctl -u etcd -f` 关注 `leader changed`、`failed to send out heartbeat`、`slow fdatasync`、`database space exceeded` 这几类日志：前者看选主，后三者分别对应网络、磁盘和容量问题。

# 5. 常见部署错误

| 现象 | 常见原因 | 处理方式 |
|------|----------|----------|
| `bind: address already in use` | 2379/2380 被旧进程占用 | `ss -lntp` 找到进程并停止，或换端口 |
| `data-dir ... is not empty` | 目录里已有成员数据，却被当成新集群 | 不要删库重来；确认 name/token 是否一致 |
| 启动后一直 `no leader` | 多数派未启动、peer 端口不通、token 不一致 | 检查三节点 peer 连通性与参数 |
| 客户端 `context deadline exceeded` | advertise 地址不可达、防火墙、TLS 未配置 | 从客户端 `curl`/`openssl s_client` 验证 |
| `cluster ID mismatch` | 旧 data-dir 与新集群混用 | 换新数据目录，或恢复原集群配置 |
| `member ... already exists` | 重复执行 member add | 先 `member list` 确认，必要时 remove 后重加 |
| `x509: certificate signed by unknown authority` | CA 或证书链不对 | 三节点使用同一 CA，客户端带 `--cacert` |
| 写入突然全部失败 | 超过 quota，etcd 进入只读 | compact + defrag，或调大配额并清理数据 |
| 选主频繁 | 磁盘慢、网络抖动、时钟漂移 | 换 SSD、检查网络、启用 NTP |

## 5.1 data-dir 残留是最常见的坑

一个新集群启动时，如果 `/var/lib/etcd` 里还留着旧集群的 `member/`，etcd 会读取旧的集群 ID 和成员信息，可能出现：

```text
requested cluster ID 8e9e05c52164694d does not match current cluster ID 4c1c9f2a...
```

正确做法是停机、备份、整体移走旧目录，再创建空目录，而不是只删 `member/` 或 `wal/`：

```bash
sudo systemctl stop etcd
sudo mv /var/lib/etcd /var/lib/etcd.bak.$(date +%Y%m%d%H%M)
sudo mkdir -p /var/lib/etcd && sudo chown -R etcd:etcd /var/lib/etcd
```

⚠ **关键认知**：不要为了"重新初始化"随手 `rm -rf` 数据目录。生产环境里那可能是唯一一份还能恢复的数据。先停机、先备份、再移走，确认新集群正常后再清理。

## 5.2 advertise 地址写错

一个典型故障：节点监听在 `0.0.0.0:2380`，但 `--initial-advertise-peer-urls` 写成了 `http://127.0.0.1:2380`。本机看起来正常，其他节点却尝试连接自己的 127.0.0.1，日志里会出现 `peer ... is unreachable`。排查时依次确认：从其他节点 `curl http://10.0.0.1:2380/version` 是否通；本机 `ss -lntp | grep 2380` 是否监听在正确网卡；`member list` 里的 peer URL 是否与 `initial-cluster` 一致。

## 5.3 token 与集群身份

`--initial-cluster-token` 不是安全凭证，而是"集群身份证"。生产上建议用有意义的名字，例如 `--initial-cluster-token prod-k8s-etcd-2026`。

★ **关键认知**：集群身份由 cluster ID、member ID、token、initial-cluster 共同决定。任何一个不一致，节点就可能拒绝加入或反复重启。排错时先把这四个值打印出来对齐，比盲目看日志快得多。

# 6. 生产部署建议

## 6.1 开启 TLS

```bash
etcd \
  --name etcd-1 \
  --data-dir /var/lib/etcd \
  --listen-client-urls https://10.0.0.1:2379 \
  --advertise-client-urls https://10.0.0.1:2379 \
  --listen-peer-urls https://10.0.0.1:2380 \
  --initial-advertise-peer-urls https://10.0.0.1:2380 \
  --cert-file=/etc/etcd/pki/server.crt \
  --key-file=/etc/etcd/pki/server.key \
  --trusted-ca-file=/etc/etcd/pki/ca.crt \
  --client-cert-auth \
  --peer-cert-file=/etc/etcd/pki/peer.crt \
  --peer-key-file=/etc/etcd/pki/peer.key \
  --peer-trusted-ca-file=/etc/etcd/pki/ca.crt \
  --peer-client-cert-auth \
  --initial-cluster etcd-1=https://10.0.0.1:2380,etcd-2=https://10.0.0.2:2380,etcd-3=https://10.0.0.3:2380 \
  --initial-cluster-state new \
  --initial-cluster-token prod-k8s-etcd-2026
```

客户端连接时同样要带 `--cacert`、`--cert`、`--key`。证书的 SAN 必须包含所有节点 IP 和 DNS 名，否则跨节点复制时会报 `x509: certificate is valid for ..., not ...`。

## 6.2 systemd 托管

```ini
[Unit]
Description=etcd
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd --config-file=/etc/etcd/etcd.yaml
Restart=always
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

`Type=notify` 让 systemd 等 etcd 真正就绪后再标记启动成功；`LimitNOFILE` 不够时，大量 Watch 连接会报 `too many open files`。

# 7. 自检题

1. 三节点集群里，`--advertise-client-urls` 写成 `127.0.0.1` 会发生什么？本机 `etcdctl` 能连上吗？其他机器呢？
2. 一个新节点启动后日志一直出现 `no leader`，请列出至少四种可能原因，并给出验证命令。
3. 为什么 `member add` 之后要尽快启动新节点？如果新节点一直不启动，原集群的写能力会受什么影响？

# 8. 小结

| 主题 | 要点 |
|------|------|
| 端口 | 2379 给客户端，2380 给 peer，两者都要放行 |
| 地址 | listen 是绑定地址，advertise 是对外公布地址 |
| 单机 | 参数简单，但没有容错，只适合开发测试 |
| 集群 | `initial-cluster` 与 token 三节点必须一致 |
| 加节点 | 先 `member add`，新节点用 `existing` 启动 |
| 健康检查 | `endpoint health` 会发起真实读请求，比端口探测可靠 |
| 排错 | 先对齐 name、token、cluster ID、advertise 地址 |
| 生产 | TLS、SSD、NTP、systemd、独立 data-dir |

下一篇进入 v3 API，讲清楚 Go 客户端如何连接、Put/Get/Delete 的选项、前缀与分页、revision 的用法、线性一致读与串行化读的取舍，以及错误处理与重试的正确姿势。
