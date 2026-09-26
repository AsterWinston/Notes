
---

# 1. rewrite 指令

```nginx
rewrite regex replacement [flag];
```

| flag | 说明 |
|------|------|
| `last` | 停止当前 rewrite，用新 URI 重新开始 location 匹配（类似内部重定向） |
| `break` | 停止 rewrite，在当前 location 内继续处理 |
| `redirect` | 302 临时重定向（用于替换后的 URI 不以 `http://` 或 `https://` 开头但需外部跳转） |
| `permanent` | 301 永久重定向 |

---

# 2. location 匹配规则

## 2.1 匹配语法

| 语法 | 优先级 | 说明 |
|------|--------|------|
| `= /uri` | **最高** | 精确匹配 |
| `^~ /uri` | 高 | 前缀匹配，匹配后不再检查正则 |
| `~ regex` | 中 | 区分大小写的正则匹配 |
| `~* regex` | 中 | 不区分大小写的正则匹配 |
| `/uri` | 低 | 普通前缀匹配 |
| `/` | 最低 | 通用匹配 |

## 2.2 匹配优先级流程

```
1. 精确匹配 =     → 找到则停止
2. 前缀匹配 ^~    → 找到最长匹配，停止正则检查
3. 正则匹配 ~ ~*  → 按配置顺序，找到第一个匹配即停止
4. 普通前缀匹配   → 按最长匹配原则
```

## 2.3 示例

```nginx
location = /login {
    # 只处理 /login 请求
}

location ^~ /static/ {
    # 处理 /static/ 开头的请求，不再检查正则
}

location ~* \.(gif|jpg|jpeg|png)$ {
    # 正则匹配图片（不区分大小写）
}

location /api/ {
    # 处理 /api/ 开头的请求
}

location / {
    # 兜底匹配
}
```

---

# 3. return 指令

```nginx
# 语法
return code [text];
return code URL;

# 典型用法
return 301 https://www.example.com$request_uri;          # 301 重定向
return 302 https://www.example.com$request_uri;          # 302 临时重定向
return 403 "Access Denied";                              # 返回状态码 + 消息
return 204;                                               # 纯状态码
```

---

# 4. 常见 Rewrite 场景

## 4.1 HTTP → HTTPS 强制跳转

```nginx
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://www.example.com$request_uri;
}
```

## 4.2 无 www 跳转到有 www

```nginx
server {
    listen 80;
    server_name example.com;
    return 301 $scheme://www.example.com$request_uri;
}
```

## 4.3 URL 重写（隐藏后端真实路径）

```nginx
location / {
    # 将 /file/xxx 重写为 /download.php?file=xxx
    rewrite ^/file/(.*)$ /download.php?file=$1 last;
}
```

## 4.4 伪静态 / SEO 友好 URL

```nginx
# 将 /article/123.html 重写为 /article.php?id=123
location / {
    rewrite ^/article/(\d+)\.html$ /article.php?id=$1 last;
}
```

## 4.5 目录最后是否带斜杠

```nginx
# 目录没有 / 时，自动添加 /（relative redirect）
location /blog {
    return 301 /blog/;
}
```

## 4.6 旧域名永久迁移

```nginx
server {
    listen 80;
    server_name old.example.com;
    return 301 https://new.example.com$request_uri;
}
```

---

# 5. if 指令（谨慎使用）

```nginx
if ($http_user_agent ~* "bot") {
    return 403;
}

if ($request_method !~ ^(GET|POST|HEAD)$) {
    return 405;
}

if ($invalid_referer) {
    return 403;
}
```

> ⚠️ `if` 指令在 location 中可能产生意外行为（Nginx 的 if 不是真正的编程语言 if）。优先使用 `try_files`、`return`、`rewrite` 等替代。

---

# 6. error_page 自定义错误页面

```nginx
# 简单方式
error_page 404 /404.html;
error_page 500 502 503 504 /50x.html;

# 重定向到后端处理
error_page 404 = @notfound;
location @notfound {
    proxy_pass http://backend/error/404;
}

# 更改响应码（将 404 转为 302 重定向）
error_page 404 =302 /custom-404.html;
```

---

# 7. break 指令

```nginx
location / {
    if (-f $request_filename) {
        break;       # 跳出 rewrite 模块处理
    }
    if (!-f $request_filename) {
        rewrite ^/(.*)$ /index.php?$1 last;
    }
}
```

---

# 8. Rewrite 日志调试

```nginx
error_log /var/log/nginx/error.log debug;       # 开启 debug 级别日志
rewrite_log on;                                  # 专门记录 rewrite 过程（输出到 error_log）
```
