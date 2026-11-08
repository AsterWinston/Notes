
# 1. 先明确威胁模型：你要防什么

容器不是虚拟机，它共享宿主机内核。Docker 安全问题通常集中在：

1. **镜像供应链**：镜像里混入恶意代码、漏洞依赖、后门。
2. **运行时逃逸**：容器内进程利用内核漏洞/配置不当拿到宿主机权限。
3. **权限过大**：`--privileged`、挂载 docker.sock、过多 capabilities。
4. **敏感信息泄露**：镜像层、环境变量、日志、卷。
5. **网络暴露面**：不必要的端口暴露、弱口令服务直接暴露公网。

工程结论：Docker 安全不是"某个参数"，而是一套从构建到运行的边界治理。理解这句话的前提是理解容器与 VM 的隔离强度差异：VM 的边界是硬件虚拟化，攻击面收敛到一个 hypervisor；容器的边界是内核的 namespace/cgroup/seccomp 实现，**内核特性越多（模块、系统调用、/proc 可写项），逃逸面越大**。历史上真实发生过的逃逸几乎都不是"0day 打穿内核"，而是配置错误：特权容器、docker.sock、hostPath 挂载、过宽的 capabilities。

```text
攻击面地图(由外到内):

  镜像层 ──▶ 构建链 ──▶ Registry ──▶ 运行时 ──▶ 内核
  (依赖投毒) (CI密钥)   (未授权访问) (配置错误)  (逃逸漏洞)
     │                                      │
     └──── 供应链攻击 ◀────── 信任链 ──── 逃逸 = 宿主机 root
```

# 2. 镜像安全：最小化与可追溯

## 2.1 最小化镜像与攻击面

1. 使用多阶段构建，把构建工具链留在 builder stage。
2. 选择更小的运行时基础镜像（distroless/alpine/精简版 JRE 等，结合你的兼容性要求）。
3. 用 `.dockerignore` 排除 `.git`、本地缓存、密钥文件等。

攻击面与镜像内容成正比：镜像里每多一个 setuid 二进制、包管理器、shell，就多一条可利用路径。**distroless 镜像没有 shell**，攻击者即使注入命令也没有解释器可用。

## 2.2 漏洞扫描与依赖治理

建议把扫描放入 CI，主流工具是 **Trivy**（覆盖 OS 包、jar 依赖、密钥泄露、配置错误、镜像违规五类）：

```bash
# 扫描镜像: OS 包 + Java 依赖(jar/pom 解析), 高危退出非零阻断 CI
trivy image --severity HIGH,CRITICAL \
  --exit-code 1 --ignore-unfixed \
  harbor.example.com/pay/order-service:1.2.3-8f3c2a1

# 扫描 Dockerfile 与 filesystem(构建前预判)
trivy config ./Dockerfile
trivy fs --scanners vuln,secret ./
```

**--ignore-unfixed 是实践里最重要的开关**：上游存在"有公告但无补丁版本"的漏洞，不忽略它们 CI 会被迫红灯却无事可做。Java 项目要注意：Trivy 解析 jar 依赖靠 pom/gradle 元数据与 uber-jar 内嵌信息，Spring Boot fat jar 的 BOOT-INF/lib 可以扫到，shaded 混淆后的产物可能漏报。

提示：扫描不等于修复；修复路径通常是升级基础镜像/依赖版本、或重建打了补丁的基础镜像。

## 2.3 tag、digest 与签名

1. tag 可能被覆盖。
2. digest 是不可变标识。
3. 签名（cosign）证明"这个 digest 确实来自你的构建流水线"。

生产关键环境建议以 digest 锁定部署版本（避免 tag 漂移），再叠加准入层的验签（见 `11_registry.md` 第 6 节）。供应链安全的行业标准是 **SLSA 分级**（构建来源证明）+ **SBOM**（物料清单）：CI 用 `buildx --attest type=provenance` 与 `syft` 生成并挂到 registry，事故时可立刻回答"哪些服务用了这个被投毒的版本"。

# 3. 运行时权限：最小权限原则

## 3.1 不要用 `--privileged`

`--privileged` 近似把宿主机能力开给容器：解除 capabilities 收敛、关闭 seccomp/apparmor 限制、放开所有设备访问（`/dev` 整盘可见）。挂载宿主根 + privileged 的容器，一条命令就能改宿主机 crontab 获得 root——各类"容器逃逸"POC 的标准起手式。除非非常清楚代价，一律禁用，并用 CI/准入规则硬性拦截。

## 3.2 capabilities：只加需要的，能删就删

Linux 把 root 的超级权限拆成约 40 个 capability，Docker 默认给一组（CAP_NET_BIND_SERVICE、CAP_CHOWN、CAP_KILL、CAP_SETUID/SETGID……），其余 drop。

策略：

1. 优先 `--cap-drop=ALL`，再按需 `--cap-add`。
2. 避免给 `SYS_ADMIN`（这是著名的"准特权"能力，挂载文件系统、操作大量内核接口）。

```bash
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE my-app:1.0
```

常用能力速查：

| Capability | 作用 | 常见需求方 |
|------------|------|-----------|
| NET_BIND_SERVICE | 绑定 <1024 特权端口 | 直接监听 80/443 的进程 |
| NET_ADMIN | 改路由/防火墙/网卡 | 网络工具 |
| SYS_PTRACE | 调试他人进程 | arthas/jstack 排障 |
| DAC_OVERRIDE | 绕过文件权限检查 | 挂载目录属主不匹配(优先改属主) |

## 3.3 三层内核过滤器对照：capability / seccomp / AppArmor

| 机制 | 拦截层 | 配置单位 | 默认强度 | 典型用途 |
|------|--------|----------|----------|----------|
| **capabilities** | "能做什么类操作"（粗粒度权限位） | 每个 cap 开关 | Docker 默认 drop 大半 | 去特权化运行 |
| **seccomp** | "能调用哪些系统调用"（按 syscall 号） | 白名单/黑名单档 | `default` 档: 屏蔽约 40 个高危调用(mount/reboot/kexec/…) | 挡逃逸面最大的入口 |
| **AppArmor/SELinux** | "能按什么路径/规则访问资源" | 安全配置文件(profile) | Docker 带内置 `docker-default` profile | 限制 /proc 写、禁 ptrace 跨进程、约束文件路径 |
| **no-new-privileges** | 禁止 setuid/suid 提权路径 | 一个布尔开关 | 默认关, 建议常开 | 堵住"子进程借 setuid 二进制提权" |

三层是**递进收窄**关系：capability 决定权限类别，seccomp 决定系统调用可达性，LSM（AppArmor/SELinux）决定资源访问规则。生产基线：保持 default seccomp 与 docker-default AppArmor，追加 `--security-opt no-new-privileges=true`，特殊需求（如 APM agent 要 ptrace）用 `SYS_PTRACE` 单独开洞而不是整体放开。

```bash
docker run --security-opt no-new-privileges=true \
           --cap-drop=ALL --cap-add=SYS_PTRACE \
           my-app:1.0
```

## 3.4 以非 root 用户运行 + 文件系统只读

建议：

1. Dockerfile 用 `USER` 指定非 root（且给固定 UID，方便 K8s `runAsUser` 与挂载目录属主对齐）。
2. 宿主机上挂载目录的权限要配合调整。

```bash
docker run --read-only --tmpfs /tmp:rw,noexec,nosuid,size=64m my-app:1.0
```

价值：

1. 降低被写入后门文件的概率（webshell、恶意 so）。
2. 强制把"需要写"的目录显式挂载出来（更清晰的边界）。

> **注意**：`--read-only` 对 Java 应用的常见绊脚石是 `/tmp`（JVM 解压 native 库、`java.io.tmpdir`）与日志目录；解法是显式 tmpfs 或 volume，而不是放弃只读。K8s 侧对应 `securityContext.readOnlyRootFilesystem: true` + emptyDir 挂 /tmp。

# 4. rootless 与 user namespace：釜底抽薪的隔离

**Rootless Docker** 把整个 daemon 跑在普通用户下，容器内的 root 映射到宿主机的**非特权用户**（经 `newuidmap/newgidmap` 分配的子 UID 段），此时即使容器 root + 内核漏洞组合逃逸，拿到的也只是那个低权用户——这是目前单机最有效的纵深防御。

```bash
# 安装 rootless 支持并启动
dockerd-rootless-setuptool.sh install
systemctl --user enable --now docker
export DOCKER_HOST=unix:///run/user/1000/docker.sock
```

代价与限制：默认无法绑定 <1024 端口（改 `net.ipv4.ip_unprivileged_port_start=80` 或走高位端口+反代）、网络需 slirp4netns 性能略降、cgroup 资源限制依赖 systemd 委托。**user namespace** 是它的底层机制：dockerd 配 `"userns-remap": "default"` 后，容器 UID 0 = 宿主机 100000，挂载目录的读写权由属主映射表决定。

**rootful 与 rootless 差异对照**：

| 维度 | rootful dockerd | rootless |
|------|-----------------|----------|
| 逃逸后获得 | 宿主机 root | 普通用户 |
| 绑定低位端口 | 可 | 默认不可 |
| 适用 | 通用服务器 | 开发机、多租户共享机 |

> **行业实践**：共享开发机/堡垒机上的容器一律 rootless；生产服务器仍以 rootful 为主（网络与存储性能、运维成熟度更好），用 cap 收敛 + seccomp + 非 root 用户 + 准入控制补偿，K8s 环境则用 Pod Security Admission 的 `restricted` 档统一基线。

# 5. 最危险的配置之一：挂载 docker.sock 与 hostPath

```bash
-v /var/run/docker.sock:/var/run/docker.sock
```

风险：

1. 拿到 docker.sock 基本等价拿到宿主机 root（可以启动特权容器、挂载宿主机根目录等）。

等价的高危挂载还有 K8s 侧的 `hostPath: /`、`/proc`、`/sys`、`/dev`，以及 `--pid=host --net=host --ipc=host` 组合。**替代方案按优先级**：一是构建委托，CI 里用 kaniko/buildah/BuildKit 远程构建器，业务容器不接触 docker API；二是 socket 代理，用 tecnativa/docker-socket-proxy 只放行只读端点；三是远程 daemon，连另一台受控构建机的 API。

# 6. 网络暴露面：默认最小开放

建议：

1. 容器间互访用内部网络，不要把所有服务都 `-p` 暴露到宿主机。
2. 对外只暴露入口（网关/Nginx/API），内部 DB/Redis 不映射端口。
3. 避免把 Docker API（2375/2376）、数据库端口直接暴露公网。

> **警告**：裸暴露 2375 TCP 端口是真实世界高频入侵路径——扫描器全公网扫无 TLS 的 docker API，拿到后可直接起 privileged 容器挂宿主根写 SSH key。**TLS daemon**（`--tlsverify` + ca/cert/key）或只监听 127.0.0.1 是底线，远程管理宁可走 SSH 隧道。K8s 侧 kubelet 10250、apiserver 同理，靠网络策略与内网隔离兜底。

# 7. 密钥与敏感信息

常见泄露路径：

1. 写进镜像层（Dockerfile `ENV`、COPY 了密钥文件）——**镜像层是不可删除的历史**，`RUN rm` 只是把文件藏进下层，`docker history`/导出层即可复原。
2. 写进 compose 文件并提交到 Git。
3. 打到日志里。

BuildKit 提供了机制级解法——**secret mount 只在使用它的那条 RUN 里可见，不产生镜像层**：

```dockerfile
# syntax=docker/dockerfile:1.6
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /src
# settings.xml 与 token 都不会进入任何镜像层
RUN --mount=type=secret,id=mvn_settings,target=/root/.m2/settings.xml \
    --mount=type=secret,id=mvn_token \
    mvn -s /root/.m2/settings.xml \
        -Drepo.token="$(cat /run/secrets/mvn_token)" \
        package -DskipTests
```

```bash
docker build --secret id=mvn_settings,src=settings.xml \
             --secret id=mvn_token,src=$CI_TOKEN_FILE .
```

实践建议：

1. 本地开发可用 `.env`，但不要提交；`trivy fs --scanners secret` 与 gitleaks 在提交钩子上把关。
2. 生产使用更规范的密钥管理（Vault/云平台 KMS/K8s Secret+外部加密），运行时注入而非构建期注入。
3. 泄露后按"撤销→轮换→溯源访问日志"处理，而不是删镜像层了事（删不掉的）。

# 8. 容器逃逸面清单与纵深防御

把常见逃逸路径按"配置错误→内核漏洞"排序，配置类占绝大多数：

| 逃逸面 | 原理 | 缓解 |
|--------|------|------|
| privileged 容器 | 全设备+全能力 | 禁用；准入拦截 |
| docker.sock 挂载 | 控制 daemon 起特权容器 | 委托构建/socket 代理 |
| hostPath /、/proc | 直接写宿主敏感文件 | PSP/PSA、只读挂载、路径白名单 |
| 可写 /proc/sys | `core_pattern`、modprobe 等劫持宿主行为 | docker-default AppArmor/SELinux 正是限制它 |
| 多余 CAP_SYS_ADMIN | 自行 mount proc/cgroup 逃逸 | `--cap-drop=ALL` 基线 |
| 共享 host PID/IPC | 注入/ptrace 宿主进程 | 禁 `--pid=host` 等共享 |
| CVE 内核漏洞 | dirty pipe、Dirty COW 类 | 及时升级内核+seccomp default |
| runc/containerd CVE | 运行时代码漏洞 | 锁版本、订阅安全通告 |

> **注意**：seccomp `unconfined` 与 `--privileged` 的组合会让上表所有"配置类缓解"失效；反过来，即使内核打上全部补丁，一次错误的 `privileged` 挂载也能瞬间沦陷。**先治理配置，再谈漏洞**。

# 9. 审计与基线（落地清单）

上线前检查（建议做成自动化）：

1. 镜像来源可信（私有仓库/签名/白名单）。
2. 镜像最小化、无构建工具残留。
3. 非 root 用户运行（`docker inspect` 查 `Config.User` 非 root/空）。
4. 禁止 privileged、禁止 docker.sock、禁止 hostPath 根挂载。
5. capabilities 最小化（cap-drop=ALL 起步）。
6. 根文件系统只读（可行则启用）。
7. seccomp/AppArmor 未关闭，`no-new-privileges` 开启。
8. 不暴露不必要端口；daemon API 仅 socket 或 TLS。
9. 密钥扫描通过（trivy secret/gitleaks），运行时注入而非镜像内置。
10. 资源限制已设置（见 `12_resource_limits.md`）。

K8s 环境用 **Pod Security Standards（privileged/baseline/restricted 三档）+ PSA 准入** 把第 4-8 条变成 namespace 级强制；单机环境用 OPA Gatekeeper、或写一个 `docker inspect` 巡检脚本定时跑。

# 10. 面试要点

| 问题 | 答题关键词 |
|------|------------|
| 容器安全的核心差异 | 共享内核，容器不是 VM，逃逸面=内核+配置 |
| 最危险的挂载是什么 | docker.sock（≈宿主机 root）、hostPath / |
| 如何做最小权限 | 非 root、cap-drop/添加最小能力、read-only、no-new-privileges |
| capability/seccomp/AppArmor 区别 | 权限位/系统调用白名单/资源路径规则，三层递进 |
| rootless 与 userns | daemon 跑普通用户, 容器 root 映射宿主非特权 UID |
| 如何做供应链安全 | 镜像扫描(trivy)、最小化、tag/digest、cosign 签名、SBOM |
| 密钥为什么不能 COPY 进镜像 | 层历史不可删, docker history 可复原; 用 BuildKit secret |

# 11. 小结

Docker 安全的心智模型一句话：**容器的隔离强度不取决于镜像里写了什么，而取决于你给了它哪些内核接口**。配置层面记住"四个默认开、一个永远关"——默认 seccomp、默认 AppArmor、`no-new-privileges`、`--read-only`（可行时）四个开，`--privileged` 永远关；docker.sock 与 hostPath 是最高频的真实逃逸路径，比任何 CVE 都危险，CI/准入必须硬拦。供应链侧形成"构建最小化 → Trivy 扫描 → cosign 签名 → digest 部署 → 准入验签"的闭环。rootless 用架构换安全，适合开发机与多租户共享机，生产服务器用"rootful + 严格基线"更务实。把第 9 节清单自动化之后，剩下的才是漏洞管理问题。下一篇（`15_cicd.md`）把上述纪律全部固化进流水线。
