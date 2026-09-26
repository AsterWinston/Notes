
# 1. Linux 日志体系

## 1.1 日志架构演进

```
传统（Syslog）:
Application → syslogd/rsyslogd → /var/log/*.log

现代（journald）:
Application → systemd-journald → /var/log/journal/ (结构化二进制)
                    ↕
              rsyslogd → /var/log/*.log (向前兼容)
```

## 1.2 主要日志组件

| 组件 | 作用 | 特点 |
|------|------|------|
| rsyslogd | 传统系统日志守护进程 | 规则灵活，支持远程转发 |
| journald | systemd 日志系统 | 结构化、二进制、自动管理 |
| logrotate | 日志轮替压缩 | 避免日志无限增长 |

# 2. rsyslog

## 2.1 基本架构

```
输入模块（inputs）
    ↓
处理管道（rules）
    ↓
输出模块（outputs）
```

## 2.2 配置文件

主配置文件：`/etc/rsyslog.conf`
扩展配置：`/etc/rsyslog.d/*.conf`

**选择器（Selector）语法**：
```
facility.priority          action
设施.优先级                操作（文件/远程/用户）
```

## 2.3 设施（Facility）和优先级（Priority）

**设施**：

| 设施 | 关键字 | 说明 |
|------|--------|------|
| 0 | kern | 内核消息 |
| 1 | user | 用户级消息 |
| 2 | mail | 邮件系统 |
| 3 | daemon | 系统守护进程 |
| 4 | auth | 认证/安全消息 |
| 5 | syslog | syslog 自身 |
| 7 | news | 新闻组 |
| 8 | uucp | UUCP |
| 16-23 | local0-local7 | 本地自定义 |

**优先级**（从高到低）：

| 等级 | 关键字 | 说明 |
|------|--------|------|
| 0 | emerg | 系统不可用 |
| 1 | alert | 需立即处理 |
| 2 | crit | 严重错误 |
| 3 | err | 错误 |
| 4 | warning | 警告 |
| 5 | notice | 正常但重要 |
| 6 | info | 信息 |
| 7 | debug | 调试 |

## 2.4 配置示例

```bash
# /etc/rsyslog.conf

# 内核日志
kern.*                          /var/log/kern.log

# 系统日志
*.info;auth,authpriv.none       /var/log/syslog

# 认证日志
auth,authpriv.*                 /var/log/auth.log

# 邮件日志
mail.*                          /var/log/mail.log
mail.err                        /var/log/mail.err

# 计划任务日志
cron.*                          /var/log/cron.log

# 调试日志
*.=debug                        /var/log/debug

# 将所有紧急消息通知所有登录用户
*.emerg                         *

# 远程日志转发
# 发送到远程日志服务器
*.* @192.168.1.100:514
*.* @@logserver.example.com:514    # TCP 方式
```

## 2.5 测试 rsyslog 配置

```bash
# 测试配置
sudo rsyslogd -N1

# 手动生成测试日志
logger "This is a test message"
logger -p local0.err "Custom facility test"
logger -t myapp "Application log message"

# 查看结果
tail /var/log/syslog
```

# 3. journald

## 3.1 特性

- **结构化日志**：每个日志条目包含时间、PID、UID、GID、优先级等字段
- **二进制格式**：高效存储，支持元数据索引
- **自动管理**：按大小/时间自动清理
- **内核日志集成**：自动捕获 dmesg 输出
- **转发兼容**：可转发到 rsyslogd 实现传统路径

## 3.2 配置

```bash
# /etc/systemd/journald.conf
[Journal]
Storage=auto                    # auto(持久化到磁盘)/persistent/volatile(内存)
Compress=yes                    # 启用压缩
Seal=yes                        # 启用完整性校验(FSS)

SystemMaxUse=4G                 # 日志最大占用
SystemMaxFileSize=200M          # 单个日志文件大小
MaxRetentionSec=30day           # 保留时间

ForwardToSyslog=yes             # 转发到 rsyslog
ForwardToWall=yes               # 紧急消息通知所有用户
```

## 3.3 journalctl 使用

```bash
# 基本查看
journalctl                       # 所有日志
journalctl -f                    # 实时跟踪
journalctl -n 50                 # 最后 50 条
journalctl -r                    # 逆序（最新在前）

# 按 unit 过滤
journalctl -u nginx.service
journalctl -u nginx.service -u sshd.service  # 多个 unit
journalctl -u nginx.service -f               # 实时跟踪某个服务

# 按时间过滤
journalctl --since "2025-05-12 10:00:00"
journalctl --until "2025-05-12 12:00:00"
journalctl --since "1 hour ago"
journalctl --since "yesterday" --until "today"

# 按优先级
journalctl -p err                  # 仅错误及以上
journalctl -p warning              # 警告及以上

# 按字段过滤
journalctl _PID=1234               # 按 PID
journalctl _UID=1000               # 按用户
journalctl _SYSTEMD_UNIT=nginx.service

# 输出格式
journalctl -o verbose              # 显示所有字段
journalctl -o json                 # JSON 格式
journalctl -o json-pretty          # 格式化 JSON

# 查看内核日志
journalctl -k                      # 等价于 dmesg
journalctl -k -p err               # 内核错误
```

## 3.4 journald vs 传统日志

| 特性 | journald | rsyslog |
|------|----------|--------|
| 格式 | 二进制，结构化 | 纯文本 |
| 存储 | 自动管理 | 需 logrotate |
| 性能 | 高（批量写入）| 中 |
| 索引 | 内置字段索引 | 无 |
| 远程转发 | 需配合 rsyslog | 原生支持 |
| 兼容性 | 新系统 | 所有系统 |

**生产环境推荐**：journald + rsyslog 并存，journald 做本地结构化存储，rsyslog 做远程转发和传统文件输出。

# 4. logrotate

## 4.1 工作原理

logrotate 定期检查日志文件，按策略进行轮替（rotate）、压缩、删除旧日志。

通常由 cron 每日执行 `/etc/cron.daily/logrotate`。

## 4.2 配置文件

主配置：`/etc/logrotate.conf`
应用配置：`/etc/logrotate.d/*`

```bash
# /etc/logrotate.conf 全局默认配置
weekly                            # 每周轮替
rotate 4                          # 保留 4 个旧日志
create                            # 轮替后创建新日志文件
dateext                           # 日期后缀（如 .log-20250512）
compress                          # 压缩旧日志
include /etc/logrotate.d          # 包含应用配置

# /var/log/wtmp {
#     monthly
#     create 0664 root utmp
#     rotate 1
# }
```

## 4.3 应用配置示例

```bash
# /etc/logrotate.d/nginx
/var/log/nginx/*.log {
    daily                          # 每天轮替
    missingok                      # 日志不存在不报错
    rotate 14                      # 保留 14 天
    compress                       # 压缩
    delaycompress                  # 压缩延后一次（避免刚轮替就压缩）
    notifempty                     # 空文件不轮替
    dateext                        # 使用日期后缀
    sharedscripts                  # 所有日志共享 postrotate 脚本
    postrotate
        [ -f /var/run/nginx.pid ] && kill -USR1 `cat /var/run/nginx.pid`
    endscript
}

# /etc/logrotate.d/myapp
/var/log/myapp/*.log {
    size 100M                      # 日志超过 100M 轮替
    rotate 5                       # 保留 5 个旧日志
    compress
    maxage 30                      # 超过 30 天的日志删除
    missingok
    copytruncate                   # 复制截断（进程不支持 reopen 时用）
}

# /etc/logrotate.d/rsyslog
/var/log/syslog
/var/log/auth.log
/var/log/kern.log
/var/log/mail.log
/var/log/cron.log
/var/log/debug
{
    rotate 7
    daily
    missingok
    notifempty
    delaycompress
    compress
    postrotate
        /usr/lib/rsyslog/rsyslog-rotate
    endscript
}
```

## 4.4 手动执行与测试

```bash
# 强制执行 logrotate
sudo logrotate -f /etc/logrotate.conf

# 调试模式（不实际执行）
sudo logrotate -d /etc/logrotate.d/nginx

# 查看状态
cat /var/lib/logrotate/status
```

## 4.5 常用指令

| 指令 | 作用 |
|------|------|
| daily/weekly/monthly | 轮替频率 |
| rotate N | 保留旧日志数量 |
| size SIZE | 按大小触发轮替（如 100M, 1G）|
| compress | 启用 gzip 压缩 |
| delaycompress | 延迟一次再压缩 |
| copytruncate | 复制文件后截断原文件 |
| create [mode] [owner] [group] | 创建新日志文件 |
| dateext | 使用日期后缀（.log-1 → .log-20250512）|
| missingok | 文件缺失不报错 |
| notifempty | 空文件不轮替 |
| sharedscripts | 所有日志匹配后执行一次脚本 |
| postrotate/endscript | 轮替后执行脚本 |
| prerotate/endscript | 轮替前执行脚本 |

# 5. 常见日志文件

| 日志文件 | 内容 |
|----------|------|
| /var/log/syslog | 系统综合日志 |
| /var/log/auth.log | 认证日志（登录、sudo）|
| /var/log/kern.log | 内核日志 |
| /var/log/mail.log | 邮件服务器日志 |
| /var/log/cron.log | cron 任务日志 |
| /var/log/nginx/access.log | Web 访问日志 |
| /var/log/nginx/error.log | Web 错误日志 |
| /var/log/mysql/error.log | MySQL 错误日志 |
| /var/log/apache2/access.log | Apache 访问日志 |
| /var/log/dpkg.log | 包管理日志 |
| /var/log/apt/history.log | apt 操作日志 |
| /var/log/boot.log | 启动日志 |

# 6. 集中式日志方案

## 6.1 rsyslog 远程转发

```bash
# 日志服务器（接收端）配置
# /etc/rsyslog.conf
module(load="imtcp")
module(load="imudp")
input(type="imtcp" port="514")
input(type="imudp" port="514")

# 按主机名分类存储
$template RemoteLogs,"/var/log/remote/%HOSTNAME%/%PROGRAMNAME%.log"
*.* ?RemoteLogs

# 客户端（发送端）配置
# /etc/rsyslog.d/remote.conf
*.* @192.168.1.100:514       # UDP
*.* @@192.168.1.100:514      # TCP（更可靠）
```

## 6.2 ELK 栈

```
Filebeat/Logstash → Elasticsearch → Kibana
        ↑
   日志文件
```
