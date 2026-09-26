
# 1. systemd 概述

## 1.1 什么是 systemd

systemd 是 Linux 系统的 init 系统和服务管理器，PID=1，负责启动和管理系统服务。它取代了传统的 SysV init 和 Upstart。

**设计目标**：
- 并行启动服务（大幅缩短开机时间）
- 按需启动（Socket/D-Bus 激活）
- 依赖管理（明确定义启动顺序）
- 统一的日志管理（journald）
- 资源控制和隔离（cgroups）

**命名来源**：systemd = System D（daemon 的 D）

## 1.2 核心组件

| 组件 | 作用 |
|------|------|
| systemd | PID 1，系统和服务管理器 |
| systemctl | systemd 的控制命令行工具 |
| journald | 日志守护进程 |
| udev | 设备管理器 |
| logind | 用户登录管理 |
| timedated | 时间同步管理 |
| resolved | DNS 解析器 |
| networkd | 网络配置守护进程 |

# 2. systemctl 命令

## 2.1 服务生命周期管理

```bash
# 启动/停止/重启
sudo systemctl start nginx.service        # 启动服务
sudo systemctl stop nginx.service         # 停止服务
sudo systemctl restart nginx.service      # 重启服务
sudo systemctl reload nginx.service       # 重新加载配置（不中断服务）

# 启用/禁用开机自启
sudo systemctl enable nginx.service       # 启用开机自启
sudo systemctl disable nginx.service      # 禁用开机自启
sudo systemctl enable --now nginx.service # 启用并立即启动
sudo systemctl disable --now nginx.service # 禁用并立即停止

# 状态查询
systemctl status nginx.service            # 详细状态
systemctl is-active nginx.service         # 是否活跃
systemctl is-enabled nginx.service        # 是否开机自启
systemctl is-failed nginx.service         # 是否失败
```

## 2.2 服务状态解读

```bash
systemctl status nginx.service
# ● nginx.service - A high performance web server and a reverse proxy server
#      Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
#      Active: active (running) since Mon 2025-05-12 10:30:00 CST; 2h 15min ago
#     Process: 1234 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
#     Process: 1235 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
#    Main PID: 1236 (nginx)
#       Tasks: 3 (limit: 2345)
#      Memory: 5.2M
#         CPU: 50ms
#      CGroup: /system.slice/nginx.service
#              ├─1236 nginx: master process /usr/sbin/nginx
#              └─1237 nginx: worker process
```

## 2.3 查看和管理所有服务

```bash
# 列出所有服务
systemctl list-units --type=service                  # 激活的服务
systemctl list-units --type=service --all            # 所有服务
systemctl list-units --type=service --state=running  # 运行中的
systemctl list-units --type=service --state=failed   # 失败的

# 查找服务
systemctl list-units | grep nginx

# 查看服务依赖
systemctl list-dependencies nginx.service
systemctl list-dependencies --reverse nginx.service  # 反向依赖
```

# 3. systemd Unit 文件

## 3.1 Unit 类型

| 类型 | 扩展名 | 用途 |
|------|--------|------|
| Service | .service | 系统服务（最常见）|
| Socket | .socket | 套接字监听（按需激活服务）|
| Timer | .timer | 定时任务（替代 cron）|
| Target | .target | 同步点，组合多个 unit |
| Mount | .mount | 文件系统挂载 |
| Path | .path | 路径监控（文件变更时激活服务）|
| Device | .device | 设备管理 |
| Slice | .slice | cgroups 资源分组 |

## 3.2 Unit 文件位置

```bash
/usr/lib/systemd/system/           # 发行版提供的 unit（包管理器安装）
/etc/systemd/system/               # 用户自定义（优先级最高）
/lib/systemd/system/               # 传统位置
/etc/systemd/user/                 # 用户级 unit
```

## 3.3 Unit 文件结构

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Custom Application
Documentation=https://example.com/docs
After=network.target postgresql.service
Wants=redis.service
Requires=postgresql.service

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
Environment="NODE_ENV=production"
EnvironmentFile=/etc/myapp/env
ExecStart=/usr/local/bin/myapp --config /etc/myapp/config.yaml
ExecReload=/bin/kill -HUP $MAINPID
ExecStop=/bin/kill -TERM $MAINPID
Restart=on-failure
RestartSec=10
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

## 3.4 Unit 文件指令详解

**[Unit] 部分**：

| 指令 | 含义 |
|------|------|
| Description | 服务描述 |
| Documentation | 文档 URI |
| After | 在哪些 unit 之后启动（不表示依赖）|
| Before | 在哪些 unit 之前启动 |
| Requires | 强依赖——被依赖的 unit 失败，本 unit 也失败 |
| Wants | 弱依赖——被依赖的 unit 失败不影响本 unit |
| BindsTo | 当被依赖的 unit 退出时，本 unit 也退出 |

**[Service] 部分**：

| 指令 | 含义 |
|------|------|
| Type | 服务类型 |
| User/Group | 运行服务的用户和组 |
| ExecStart | 启动命令（绝对路径）|
| ExecStartPre | 启动前执行的命令 |
| ExecStartPost | 启动后执行的命令 |
| ExecReload | 重载命令 |
| ExecStop | 停止命令 |
| Restart | 重启策略 |
| RestartSec | 重启间隔 |
| Environment | 环境变量 |
| EnvironmentFile | 环境变量文件 |
| WorkingDirectory | 工作目录 |
| LimitNOFILE | 文件描述符限制 |
| TimeoutStartSec | 启动超时 |

**Type 类型**：

| Type | 说明 |
|------|------|
| simple | ExecStart 启动后即认为服务就绪（默认）|
| forking | 父进程 fork 子进程后退出（传统 daemon）|
| oneshot | 执行一次就退出（需 RemainAfterExit=yes）|
| dbus | 在 D-Bus 总线注册后认为就绪 |
| notify | 通过 sd_notify() 发送就绪通知 |
| idle | 等所有任务完成再执行 |

**[Install] 部分**：

| 指令 | 含义 |
|------|------|
| WantedBy | 被加入哪个 target 的 Wants |
| RequiredBy | 被加入哪个 target 的 Requires |
| Alias | 别名 |
| Also | 启用本 unit 时同时启用的其他 unit |

## 3.5 管理自定义服务

```bash
# 创建并启用自定义服务
sudo vim /etc/systemd/system/myapp.service
sudo systemctl daemon-reload          # 重新加载 unit 文件
sudo systemctl enable myapp.service   # 启用开机自启
sudo systemctl start myapp.service    # 启动
sudo systemctl status myapp.service   # 验证状态

# 查看服务日志
journalctl -u myapp.service
```

# 4. Target（运行级别）

systemd target 相当于 SysV init 的运行级别：

| Target | SysV 级别 | 说明 |
|--------|-----------|------|
| poweroff.target | 0 | 关机 |
| rescue.target | 1 | 单用户救援模式 |
| multi-user.target | 2,3,4 | 多用户非图形模式 |
| graphical.target | 5 | 多用户图形模式 |
| reboot.target | 6 | 重启 |
| emergency.target | — | 紧急 Shell（最小环境）|

```bash
# 查看和切换 target
systemctl get-default                     # 查看默认 target
sudo systemctl set-default multi-user.target  # 设置默认命令行模式
sudo systemctl isolate graphical.target   # 切换到图形界面

# target vs 运行级别
systemctl list-units --type=target        # 列出所有 target
runlevel                                  # 查看当前和之前的运行级别
```

# 5. journalctl — 日志管理

## 5.1 基本使用

```bash
journalctl                               # 查看所有日志
journalctl -u nginx.service               # 查看特定服务日志
journalctl -k                             # 仅内核日志
journalctl -f                             # 实时跟踪（类似 tail -f）
journalctl -n 50                          # 显示最后 50 条

# 按时间过滤
journalctl --since "2025-05-12 10:00:00" --until "2025-05-12 11:00:00"
journalctl --since "1 hour ago"
journalctl --since today
journalctl --since yesterday

# 按优先级过滤
journalctl -p err                         # 只显示错误及以上
journalctl -p err -u nginx.service        # 组合过滤
# 优先级: emerg alert crit err warning notice info debug
```

## 5.2 日志维护

```bash
# 查看日志占用
journalctl --disk-usage

# 清理日志
sudo journalctl --vacuum-size=200M        # 限制总大小
sudo journalctl --vacuum-time=30d         # 保留最近 30 天
sudo journalctl --rotate                  # 轮替日志文件

# 日志持久化（默认 /run/log/journal 是临时存储）
# 创建 /var/log/journal 目录即可持久化
sudo mkdir -p /var/log/journal
sudo systemctl restart systemd-journald
```

# 6. 资源控制（cgroups）

systemd 使用 cgroups 管理服务资源：

```ini
[Service]
# CPU
CPUAccounting=yes
CPUQuota=50%                      # 限制 CPU 使用率 50%
CPUShares=512

# 内存
MemoryAccounting=yes
MemoryMax=512M                    # 内存硬限制
MemoryHigh=400M                   # 内存软限制

# 磁盘 I/O
IOAccounting=yes
IOReadBandwidthMax=/dev/sda 100M
IOWriteBandwidthMax=/dev/sda 50M

# 进程数量
TasksMax=500

# 限制文件大小
LimitFSIZE=1G
```

```bash
# 查看当前资源使用
systemd-cgls                      # cgroups 树状结构
systemd-cgtop                     # 类似 top，按 cgroup 显示
```

# 7. 疑难排查

```bash
# 排查启动失败的服务
systemctl list-units --state=failed

# 查看错误详情
systemctl status <unit>
journalctl -xe                      # -x 添加解释，-e 跳到末尾
journalctl -u <unit> --since "5 min ago"

# 分析启动时间
systemd-analyze                     # 整体启动时间
systemd-analyze blame               # 各服务启动耗时（从高到低）
systemd-analyze critical-chain      # 关键启动链

# 生成启动时间图表
systemd-analyze plot > boot.svg

# 依赖分析
systemd-analyze dot nginx.service | dot -Tsvg > nginx-deps.svg
```
