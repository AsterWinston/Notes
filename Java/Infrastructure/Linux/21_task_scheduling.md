
# 1. 任务调度概述

Linux 提供多种任务调度机制，适用于不同的场景：

| 机制 | 精度 | 重复性 | 系统/用户 | 适用场景 |
|------|------|--------|-----------|----------|
| cron | 分钟级 | 周期性 | 系统级和用户级 | 定时备份、日志轮替 |
| anacron | 天级 | 周期性 | 系统级 | 非 7×24 运行的机器 |
| at | 分钟级 | 单次 | 用户级 | 一次性计划任务 |
| systemd timer | 秒级 | 周期性/单次 | 系统级 | 与 systemd 集成 |
| sleep/batch | 秒级 | 自定义 | 脚本内 | 脚本内部延时 |

# 2. cron

## 2.1 cron 工作原理

cron 由 crond 守护进程（PID 1 的子进程）负责调度，每分钟检查一次任务列表，匹配当前时间的任务并执行。

**cron 的执行环境**：
- 使用用户自己的 Shell（/etc/passwd 中指定的 Shell）
- $HOME 设置为用户家目录
- 仅设置少量环境变量（LOGNAME、HOME、SHELL）
- **PATH 通常只有 /usr/bin:/bin**，建议在脚本中使用绝对路径

## 2.2 crontab 格式

```bash
# 每一行代表一个定时任务
# ┌───────── 分钟 (0-59)
# │ ┌───────── 小时 (0-23)
# │ │ ┌───────── 日 (1-31)
# │ │ │ ┌───────── 月 (1-12)
# │ │ │ │ ┌───────── 星期 (0-7, 0和7都表示周日)
# │ │ │ │ │
# * * * * * command-to-execute
```

**特殊符号**：

| 符号 | 含义 | 示例 |
|------|------|------|
| `*` | 所有可能的值 | `* * * * *` 每分钟 |
| `,` | 多个值 | `1,15 * * * *` 每小时的第1和第15分钟 |
| `-` | 范围 | `9-17 * * * *` 9点到17点 |
| `/` | 步长 | `*/5 * * * *` 每5分钟 |
| `@reboot` | 开机时 | `@reboot /path/to/script` |

**特殊字符串**：

| 字符串 | 等价于 |
|--------|--------|
| `@reboot` | 启动时 |
| `@yearly` / `@annually` | `0 0 1 1 *` |
| `@monthly` | `0 0 1 * *` |
| `@weekly` | `0 0 * * 0` |
| `@daily` / `@midnight` | `0 0 * * *` |
| `@hourly` | `0 * * * *` |

## 2.3 crontab 命令

```bash
crontab -l                        # 列出当前用户的任务
crontab -e                        # 编辑当前用户的任务（默认 vi）
crontab -r                        # 删除当前用户的所有任务
crontab -u alice -l               # 查看 alice 的任务（root 可用）
crontab -u alice -e               # 编辑 alice 的任务

# 从文件导入
crontab mycron.txt

# 备份 crontab
crontab -l > mycron.backup
```

## 2.4 常见示例

```bash
# 每小时的 30 分执行
30 * * * * /usr/bin/backup.sh

# 每天凌晨 2:30
30 2 * * * /usr/bin/backup.sh

# 工作日（周一到周五）的 9 点
0 9 * * 1-5 /usr/bin/report.sh

# 每 10 分钟
*/10 * * * * /usr/bin/check_status.sh

# 每月第一天午夜
0 0 1 * * /usr/bin/monthly_cleanup.sh

# 每年 1 月 1 日
0 0 1 1 * /usr/bin/new_year_archive.sh

# 每分钟，但仅在 9-17 点之间（工作日）
* 9-17 * * 1-5 /usr/bin/business_hours_task.sh

# 每个季度（1月、4月、7月、10月的第一天）
0 0 1 1,4,7,10 * /usr/bin/quarterly_report.sh

# 重启后执行
@reboot /usr/bin/start_my_service.sh

# 命令输出重定向（避免邮件）
30 2 * * * /usr/bin/backup.sh > /dev/null 2>&1

# 多条命令用 && 或 ; 分隔
30 2 * * * cd /backup && ./run.sh && echo "done" >> /var/log/cron.log
```

## 2.5 环境变量问题

cron 在执行脚本时的环境变量非常有限，常见解决方案：

```bash
# 在 crontab 中设置环境变量
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
MAILTO=admin@example.com

30 2 * * * /usr/bin/backup.sh

# 或在脚本开头加载环境
#!/bin/bash
source /etc/profile
source ~/.bashrc
```

## 2.6 系统级 crontab

除了用户的 crontab，系统级 crontab 位于 `/etc/crontab`（第6个字段指定执行用户）：

```bash
# /etc/crontab
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# 分钟 小时 日 月 星期 用户 命令
25 6  *   *   *   root  /usr/sbin/update-locatedb
47 6  *   *   7   root  /usr/sbin/updatedb
```

## 2.7 cron 目录

Debian/Ubuntu 提供按频率存放脚本的目录（自动执行，无需 crontab -e）：

```bash
/etc/cron.hourly/        # 每小时执行（每分钟检查）
/etc/cron.daily/         # 每天执行（凌晨）
/etc/cron.weekly/        # 每周执行（周日凌晨）
/etc/cron.monthly/       # 每月执行（1号凌晨）
/etc/cron.d/             # 自定义频率（格式同 /etc/crontab）
```

执行时间由 `/etc/crontab` 中的 `run-parts` 控制（通常为 6:25 左右）。

## 2.8 cron 权限控制

```bash
# 允许/禁止用户使用 cron
/etc/cron.allow           # 白名单（存在则仅这些用户可用）
/etc/cron.deny            # 黑名单（默认所有用户可用）
```

# 3. anacron

## 3.1 适用场景

anacron 用于非 7×24 运行的机器，如果任务在计划时间未运行（关机状态），开机后自动补执行。

```bash
# /etc/anacrontab
# 周期(天)  延迟(分钟)  任务标识  命令
1           5           cron.daily    run-parts /etc/cron.daily
7           10          cron.weekly   run-parts /etc/cron.weekly
30          15          cron.monthly  run-parts /etc/cron.monthly
```

**cron vs anacron**：
- cron：精确到分钟，假设机器一直运行
- anacron：精确到天，会补执行错过的任务

# 4. at — 一次性任务

## 4.1 基本用法

```bash
# 指定时间执行
echo "backup.sh" | at 02:30
echo "shutdown -h now" | at 23:00

# 交互模式
at 14:30
at> /usr/bin/backup.sh
at> <Ctrl+D>

# 灵活的时间格式
at now + 30 minutes              # 30 分钟后
at now + 2 hours                 # 2 小时后
at now + 1 day                   # 1 天后
at 10:00 tomorrow                # 明天 10 点
at 10:00 + 7 days                # 7 天后 10 点
at teatime                       # 下午 4 点
at midnight                      # 午夜
at 3:00 2025-06-01               # 指定日期
```

## 4.2 at 管理

```bash
atq                               # 显示排队任务（Queue）
atrm 5                            # 删除编号为 5 的任务
at -c 5                           # 查看编号 5 的任务内容

# 权限控制
/etc/at.allow
/etc/at.deny
```

# 5. systemd timer（现代替代）

## 5.1 优势

- 与 systemd service 集成，功能完善
- 支持秒级精度
- 支持随机延迟、单调时钟
- 支持定时器和日历事件
- 完整的日志和依赖管理

## 5.2 Timer Unit 示例

**myapp.service**：
```ini
[Unit]
Description=My Application Task

[Service]
Type=oneshot
ExecStart=/usr/local/bin/myapp-task
User=myapp
```

**myapp.timer**：
```ini
[Unit]
Description=Run myapp-task every 30 minutes

[Timer]
# 日历事件（类似 cron 格式）
OnCalendar=*:0/30

# 或使用相对时间
# OnUnitActiveSec=30m

# 首次执行延迟
OnBootSec=5min

# 随机延迟（防止大量任务同时执行）
RandomizedDelaySec=60

# 持久化——如果机器关机错过任务，开机后补执行
Persistent=true

# 单位
# sec, s, min, m, hour, h, day, d, week, month, year

[Install]
WantedBy=timers.target
```

**启用 timer**：
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now myapp.timer   # 启用并立即开始计时
sudo systemctl status myapp.timer
systemctl list-timers                      # 查看所有 timer
```

## 5.3 OnCalendar 格式

```bash
# 标准格式：DayOfWeek Year-Month-Day Hour:Minute:Second
OnCalendar=Mon..Fri *-*-* 09:00:00     # 工作日 9 点
OnCalendar=*-*-* 02:30:00              # 每天凌晨 2:30
OnCalendar=*-01-01 00:00:00            # 每年 1 月 1 日

# 缩写
OnCalendar=hourly                       # 每小时
OnCalendar=daily                        # 每天
OnCalendar=weekly                       # 每周一
OnCalendar=monthly                      # 每月 1 号
OnCalendar=yearly                       # 每年 1 月 1 号

# 与 cron 对应的 systemd timer
# cron: */5 * * * *
OnCalendar=*:0/5

# cron: 30 2 * * *
OnCalendar=*-*-* 02:30:00

# cron: 0 9 * * 1-5
OnCalendar=Mon..Fri *-*-* 09:00:00
```

## 5.4 各方案对比

| 维度 | cron | anacron | systemd timer |
|------|------|---------|---------------|
| 精度 | 分钟 | 天 | 秒 |
| 依赖管理 | 无 | 无 | 完善 |
| 资源控制 | 无 | 无 | cgroups |
| 日志集成 | 需手动 | 需手动 | journalctl |
| 错过补执行 | 否 | 是 | 可配置 |
| 随机延迟 | 否 | 否 | 支持 |
| 配置复杂度 | 低 | 低 | 中 |

**选择建议**：
- 简单周期性任务 → cron
- 非 24 小时运行的机器 → anacron
- 需要精细控制的系统服务 → systemd timer
- 一次性任务 → at
