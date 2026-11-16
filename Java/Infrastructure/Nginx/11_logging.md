
---

# 1. 日志类型

| 日志 | 路径（yum 安装） | 说明 |
|------|-------------------|------|
| `access_log` | `/var/log/nginx/access.log` | 访问日志，记录每次请求 |
| `error_log` | `/var/log/nginx/error.log` | 错误日志，记录运行错误与诊断信息 |

---

# 2. access_log 配置

## 2.1 基本语法

```nginx
access_log path [format [buffer=size] [gzip[=level]] [flush=time] [if=condition]];
access_log off;    # 关闭访问日志
```

## 2.2 多级作用域

```nginx
http {
    # http 级别——全局默认
    log_format main '...';
    access_log /var/log/nginx/access.log main;

    server {
        # server 级别——覆盖 http 级别
        access_log /var/log/nginx/www.access.log main;
    }

    location /static/ {
        # location 级别——关闭静态资源日志
        access_log off;
    }
}
```

---

# 3. log_format 自定义格式

```nginx
http {
    # ── 标准格式 ─────────────────
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    # ── JSON 格式（方便日志采集与分析）───
    log_format json escape=json '{'
        '"timestamp":"$time_iso8601",'
        '"remote_addr":"$remote_addr",'
        '"host":"$host",'
        '"method":"$request_method",'
        '"uri":"$uri",'
        '"query":"$query_string",'
        '"status":$status,'
        '"body_bytes":$body_bytes_sent,'
        '"request_time":$request_time,'
        '"upstream_addr":"$upstream_addr",'
        '"upstream_status":"$upstream_status",'
        '"upstream_response_time":"$upstream_response_time",'
        '"http_user_agent":"$http_user_agent",'
        '"http_referer":"$http_referer",'
        '"http_x_forwarded_for":"$http_x_forwarded_for"'
    '}';

    access_log /var/log/nginx/access.log json;
}
```

> `escape=json`：对变量中的双引号、换行等特殊字符进行 JSON 转义。

---

# 4. error_log 配置

```nginx
error_log /var/log/nginx/error.log warn;
```

| 日志级别（从低到高） | 说明 |
|----------------------|------|
| `debug` | 调试信息（编译时需带 `--with-debug`） |
| `info` | 一般信息 |
| `notice` | 需要关注的信息 |
| `warn` | 警告（推荐生产环境使用） |
| `error` | 非致命错误 |
| `crit` | 严重错误 |
| `alert` | 必须立即处理 |
| `emerg` | 系统不可用 |

---

# 5. 日志切割（Log Rotation）

## 5.1 使用 logrotate（推荐）

```bash
# 创建配置文件 /etc/logrotate.d/nginx
cat > /etc/logrotate.d/nginx << 'EOF'
/var/log/nginx/*.log {
    daily                      # 每天切割
    missingok                  # 日志文件不存在时跳过
    rotate 30                  # 保留 30 份
    compress                   # 压缩旧日志
    delaycompress              # 延迟一天压缩
    notifempty                 # 空文件不切割
    create 640 nginx adm       # 创建新文件权限
    sharedscripts              # 所有日志切割完再执行脚本
    postrotate
        # 通知 Nginx 重新打开日志文件
        if [ -f /var/run/nginx.pid ]; then
            kill -USR1 $(cat /var/run/nginx.pid)
        fi
    endscript
}
EOF
```

## 5.2 手动切割脚本

```bash
#!/bin/bash
LOG_DIR="/var/log/nginx"
DATE=$(date +%Y%m%d)

# 重命名旧日志
mv $LOG_DIR/access.log $LOG_DIR/access.log.$DATE

# 通知 Nginx 重新打开日志
kill -USR1 $(cat /var/run/nginx.pid)

# 压缩旧日志
gzip $LOG_DIR/access.log.$DATE
```

---

# 6. 条件日志

```nginx
# 不记录状态码 2xx 和 3xx 的请求（只记录错误和重定向）
map $status $loggable {
    ~^[23]  0;
    default 1;
}

access_log /var/log/nginx/access.log main if=$loggable;
```

```nginx
# 不记录健康检查请求
map $http_user_agent $ignore_healthcheck {
    default 0;
    "~*healthcheck" 1;
}
access_log /var/log/nginx/access.log main if=$ignore_healthcheck;
```

---

# 7. 日志缓冲

```nginx
access_log /var/log/nginx/access.log main buffer=64k flush=30s;
access_log /var/log/nginx/access.log main buffer=64k gzip=6 flush=1m;
```

- `buffer`：缓冲区大小，数据攒满后批量写入磁盘
- `gzip`：压缩写入（减少 I/O）
- `flush`：最长延迟写入时间

---

# 8. 日志分析工具

| 工具 | 说明 |
|------|------|
| `tail -f` | 实时查看日志 `tail -f /var/log/nginx/access.log` |
| `awk` | 统计分析（PV、UV、响应时间等） |
| **GoAccess** | 实时 Web 日志分析（终端/CGI 界面） |
| **ELK / EFK** | Elasticsearch + Logstash/Fluentd + Kibana，大规模日志分析 |
| **Loki + Grafana** | 轻量级日志聚合方案 |

```bash
# 使用 awk 统计访问最多的 10 个 IP
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10

# 统计状态码分布
awk '{print $9}' access.log | sort | uniq -c | sort -rn
```
