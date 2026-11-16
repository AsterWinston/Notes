
---

用 Nginx 托管静态资源是它最本职的工作：前端构建产物、下载文件、SPA 站点，都由 Nginx 从磁盘读出直接送回客户端。静态服务的性能天花板由三件事决定——**读取路径是否零拷贝、能否被浏览器/CDN 长效缓存、传输能否压缩**。

# 1. 基础站点配置

一个最小可用的静态站点只需要四行核心指令：

```nginx
server {
    listen 80;
    server_name www.example.com;
    root /usr/share/nginx/html;       # 站点根目录
    index index.html index.htm;       # 默认首页文件
}
```

`index` 与 `root` 的关系：请求 `/` 时，Nginx 把 `root` 与 URI 拼接后按 `index` 列表逐个探测文件，命中哪个就内部重定向到哪个。

**`root` vs `alias`**：

| 指令 | 行为 | 示例 |
|------|------|------|
| `root` | 拼接路径：`root路径 + URI` | `root /www;` → `/www/images/logo.png` |
| `alias` | 替换路径：`alias路径替换location前缀` | `alias /www/;` → `/www/logo.png` |

```nginx
# root 示例
location /images/ {
    root /var/www;              # 请求 /images/logo.png → /var/www/images/logo.png
}

# alias 示例（注意路径末尾的 /）
location /static/ {
    alias /var/www/static/;     # 请求 /static/logo.png → /var/www/static/logo.png
}
```

> `alias` 替换的是 location 匹配的前缀部分，不是拼接。路径末尾的 `/` 需要保持一致。

**这是新手最高频的 404 来源**：`location /static/` 配 `alias /var/www/static`（少尾斜杠）会拼出 `/var/www/staticlogo.png` 这种不存在的路径。心智模型——`root` 是"在 URI 前面加目录"，`alias` 是"把 location 前缀换成目录"。**能用 root 就不用 alias**，只有在 URI 前缀与磁盘目录结构对不上时才用。

# 2. MIME 类型配置

响应头 `Content-Type` 由文件扩展名查 `mime.types` 映射表得出，错误的 MIME 会直接改变浏览器行为：`.js` 被识别成 `text/plain` 时 ES Module 拒绝执行，字体 MIME 不对时 Firefox 拒绝加载。
```nginx
http {
    include /etc/nginx/mime.types;      # 标准 MIME 映射表
    default_type application/octet-stream;  # 未匹配时的默认类型
}
```

常用 MIME 类型片段（`mime.types`）：

```nginx
types {
    text/html                    html htm shtml;
    text/css                     css;
    text/javascript              js;
    image/png                    png;
    image/jpeg                   jpeg jpg;
    image/gif                    gif;
    application/json             json;
    application/pdf              pdf;
    application/xml              xml;
    font/woff2                   woff2;
}
```

**排查 MIME 问题用 `nginx -T | grep -A2 woff2` 看实际加载的映射表**。新版 Nginx 已把 `.js` 定为 `text/javascript`（旧版 `application/javascript`，两者浏览器都接受）；`.mjs`、`.wasm` 等新扩展在老映射表里可能缺失，需自行在 `types{}` 补充。`default_type application/octet-stream` 是安全兜底——未识别扩展名一律按"下载文件"处理而非按 HTML 渲染，能规避一部分 XSS 风险。

# 3. 目录浏览（autoindex）

```nginx
location /downloads/ {
    alias /data/files/;
    autoindex on;                # 开启目录列表
    autoindex_exact_size off;    # 显示可读的文件大小（KB/MB/GB）
    autoindex_localtime on;      # 显示本地时间而非 UTC
    autoindex_format html;       # 格式：html | json | xml | jsonp
}
```

> **生产环境谨慎开启** autoindex，可能暴露目录结构。

请求命中**目录**且 index 不存在时 Nginx 现场生成列表页。风险有三：**目录/文件名本身泄密**（备份包被爬走）、**爬虫无障碍遍历**整站文件、与上传目录叠加时给攻击者导航。公网确需列目录，也应加 `auth_basic` 或 IP 白名单。

# 4. 文件访问控制

静态站点最常见的暴露面是"根目录混入不该公开的文件"（`.git`、`.env`、日志、SQL 备份），用 deny 规则兜底：

```nginx
# 禁止访问隐藏文件（.开头的文件）
location ~ /\. {
    deny all;
    access_log off;
    log_not_found off;
}

# 禁止访问指定文件类型
location ~* \.(sql|log|md|yml|yaml)$ {
    deny all;
}

# 只允许特定请求方法
location / {
    limit_except GET HEAD POST {
        deny all;
    }
}
```

注意 `location ~ /\.` 这条正则同时挡住了 `.git`、`.env`、`.htaccess`，是部署事故（把仓库整个 rsync 上线）最后的保险丝；`access_log off; log_not_found off;` 防止扫描器刷爆日志。

# 5. 静态资源缓存（Expires）

静态服务性能的半壁江山在缓存头。浏览器/CDN 是否回源，由 `Expires` 与 `Cache-Control` 决定：

```nginx
# 图片、字体等长期缓存
location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot)$ {
    expires 30d;                                 # 缓存 30 天
    add_header Cache-Control "public, immutable";
}

# HTML 不缓存（或短期缓存）
location ~* \.html$ {
    expires -1;                                  # 不缓存
    add_header Cache-Control "no-cache, must-revalidate";
}
```

| `expires` 值 | 说明 |
|--------------|------|
| `expires 1d;` | 1 天后过期 |
| `expires -1;` | 不缓存（等同于 `Cache-Control: no-cache`） |
| `expires off;` | 不添加 Expires/Cache-Control 头 |
| `expires max;` | 2037 年过期（等同于永久缓存） |

**分层策略是行业标配**：带内容 hash 的文件名（`main.3f8a2b.js`）配 `expires max; + immutable`，永不回源，改内容必然改文件名；HTML 入口配 `no-cache`（不是 `no-store`——no-cache 允许本地存副本但每次必须校验），保证发版即时生效。两者必须**成对出现**：入口缓存过长或资源文件名不带 hash，都会造成"发版后用户看到新旧混合页面"的经典事故。

> **注意**：`expires` 与 `add_header Cache-Control` 会互相覆盖——`expires` 本身也写 Cache-Control（`expires 30d` 生成 `max-age=2592000`），后写的 `add_header` 会顶掉它。另一条隐蔽规则：location 内一旦出现任何 `add_header`，**父级的 add_header 全部不再继承**，安全头要在本层重写。

# 6. sendfile 与 tcp_nopush

发送路径决定了单机静态吞吐上限。传统读文件发网络要经历"磁盘→内核页缓存→用户态缓冲→socket 缓冲→网卡"四次拷贝，Nginx 用 `sendfile()` 系统调用直接在内核态完成搬运：

```nginx
http {
    sendfile on;          # 零拷贝文件传输（绕过用户空间，直接内核态传输）
    tcp_nopush on;        # 配合 sendfile，将数据包装满后一次性发送
}
```

**sendfile 原理对比**：

```text
传统方式：
  磁盘 → 内核缓冲区 → 用户缓冲区 → 内核Socket缓冲区 → 网卡

sendfile：
  磁盘 → 内核缓冲区（直接）→ 内核Socket缓冲区 → 网卡
  省去了一次"内核 → 用户 → 内核"的拷贝
```

`tcp_nopush`（Linux 上对应 `TCP_CORK`）解决另一个问题：sendfile 之后响应头和文件数据可能分属多个小包发出，浪费带宽。开启后内核**攒满一个 MSS 再发**。与之相对的 `tcp_nodelay on`（默认开）在 keep-alive 连接上禁用 Nagle、小数据立即发出——两者不冲突，**nopush 管"发响应时攒包"，nodelay 管"收请求时不等待"**，典型组合 `sendfile on; tcp_nopush on; tcp_nodelay on;`。

> **设计考量**：sendfile 的收益在**大文件与高并发小文件**场景最明显；但响应体若需内容改写（sub_filter）、或旧版 OpenSSL 不支持 TLS 下的 sendfile，路径会回退用户态拷贝，配置照开不影响正确性。

# 7. gzip / brotli 压缩

文本资源（HTML/CSS/JS/JSON/SVG）压缩收益极大（常见 60-80% 体积下降），图片视频（本身已压缩）则不要开。内置 gzip 模块：

```nginx
http {
    gzip on;
    gzip_min_length 1k;                       # 小于 1KB 不值得压（可能反而变大）
    gzip_comp_level 5;                        # 1-9，5 是 CPU/压缩率平衡点
    gzip_types text/css application/javascript application/json image/svg+xml;
    gzip_vary on;                             # 输出 Vary: Accept-Encoding，CDN 缓存正确性关键
}
```

`gzip_types` **不含 text/html**——HTML 永远会被压缩，这是硬编码行为；也不要把 `application/octet-stream` 加进去。`gzip_vary on` 在有 CDN 时是必须的，否则 CDN 可能把 gzip 响应发给不支持的客户端。对构建产物更优的是**预压缩**：CI 生成 `.gz`/`.br`，Nginx 直接发送，零运行时 CPU：

```nginx
# 方式1：内置 gzip_static（编译需 --with-http_gzip_static_module）
gzip_static on;      # 有 main.js.gz 就直接发它，还能读 .gz 生成正确的 ETag/Last-Modified

# 方式2：brotli（第三方模块 ngx_brotli，或直接用 OpenResty 发行版）
brotli on;
brotli_comp_level 6;
brotli_types text/css application/javascript application/json image/svg+xml;
```

**gzip 与 brotli 的优先级由 `Accept-Encoding` 协商**：现代浏览器都声明 `br`，装了模块的 Nginx 优先回 brotli，老客户端回 gzip。brotli 压缩率高出 gzip 约 15-25%，但运行时 CPU 也更高——运行时开 `brotli_comp_level 4-6`，更高级别留给构建期预压缩。

# 8. 条件请求：ETag 与 Last-Modified

缓存过期（或 `no-cache`）不等于重新下载。浏览器会带上校验头发起**条件请求**，Nginx 判断文件未变则回 `304 Not Modified`（零 body），即"协商缓存"：

```text
首次请求                         过期后再请求
GET /app.js                      GET /app.js
← 200, Last-Modified: A          ← If-Modified-Since: A
← 200, ETag: "x"                 ← If-None-Match: "x"
                                   → 文件未变 → 304（无响应体）
```

静态文件的 `Last-Modified` 取自 mtime，`ETag` 默认由"Last-Modified + 文件大小"拼出，无需任何配置即自动生效；Nginx 收到带 `If-Modified-Since`/`If-None-Match` 的请求会先比对再决定 200/304，**ETag 优先级高于 Last-Modified**。多实例部署时 ETag 由 mtime+size 生成同样稳定，一般无需调整；NFS 场景则相反，mtime 精度不可靠，必须保留 ETag 校验。

> **注意**：动态接口经 `proxy_pass` 转发时校验头透传给后端，由应用自己决定 304；多级代理部署时务必确认中间层没有吞掉 `If-None-Match`，否则 304 优化全线失效，CDN 回源流量翻倍。

# 9. CORS 跨域头设置

静态资源域（如 `static.example.com`）与应用域不同源时，字体、ES Module、fetch/ajax 资源需要显式 CORS 头。无凭证的公共资源最简单：`add_header Access-Control-Allow-Origin "*" always;`。需要携带 Cookie（`credentials: include`）时**不能用 `*`**，必须回显具体来源并允许凭证，且要处理预检请求：

```nginx
location /api-assets/ {
    # 预检：OPTIONS 单独放行，if 在 location 顶部配合 return 是安全用法
    if ($request_method = OPTIONS) {
        add_header Access-Control-Allow-Origin "https://app.example.com" always;
        add_header Access-Control-Allow-Methods "GET, HEAD, OPTIONS" always;
        add_header Access-Control-Max-Age 86400 always;
        return 204;
    }
    add_header Access-Control-Allow-Origin "https://app.example.com" always;
    add_header Access-Control-Allow-Credentials "true" always;
}
```

> **注意**：`add_header` 默认只作用于 2xx/3xx 响应，404/403 时浏览器看不到 CORS 头会报"跨域错误"掩盖真实的"资源不存在"，排查极具迷惑性——**凡与 CORS 相关的 add_header 一律加 `always`**。多来源场景不要用 if 硬枚举，可用 `map $http_origin $cors_origin` 在白名单命中时回显 origin。

# 10. try_files 指令与 SPA 部署

`try_files` 按顺序探测候选路径，全部落空则走最后一项，是静态路由的"兜底逻辑"：

```nginx
location / {
    try_files $uri $uri/ /index.html;   # 找不到文件时返回 index.html（SPA 历史模式）
}
```

```nginx
# 静态找不到再转后端（命名 location 不会被 URI 直接匹配，不易成环）
location / {
    try_files $uri $uri/ @backend;
}

location @backend {
    proxy_pass http://127.0.0.1:8080;
}
```

```nginx
# =code 变体：候选全落空时直接返回 404
location /images/ {
    try_files $uri /default.png =404;
}
```

**SPA 刷新 404 的成因**：Vue/React history 路由下，`/user/profile` 只存在于前端路由表，磁盘上没有这个文件；刷新时请求打到服务器就 404。`try_files $uri $uri/ /index.html` 的语义是"真实文件优先（JS/CSS/图片照常命中），文件不存在返回 index.html 让前端路由接管"，且这是**内部重写**（URL 栏不变），不是重定向。

> **行业实践**：SPA 站点必须"入口不缓存、资源长缓存"配套——`/index.html` 显式配 `Cache-Control: no-cache`，否则 try_files 兜底返回的旧入口会把用户锁死在旧版本上。

# 11. 静态资源优化实践

综合前面各节，给出可直接投产的静态站点配置：

```nginx
server {
    listen 80;
    server_name static.example.com;

    root /data/static;

    # 防盗链
    location ~* \.(jpg|jpeg|png|gif|webp)$ {
        valid_referers none blocked example.com *.example.com;
        if ($invalid_referer) {
            return 403;
        }
    }

    # 不记录静态资源日志
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        access_log off;
        expires 30d;
        add_header Cache-Control "public, immutable";
        gzip_static on;
    }

    # SPA 回退
    location / {
        try_files $uri $uri/ /index.html;
    }

    location = /index.html {
        add_header Cache-Control "no-cache";
    }
}
```

| 手段 | 解决的问题 | 代价/注意点 |
|------|-----------|------------|
| sendfile + tcp_nopush | 发送路径拷贝开销 | 基本无脑开 |
| expires + immutable | 重复回源 | 依赖文件名带 hash 发版 |
| gzip_static / brotli 预压缩 | 传输体积 | 构建期产出 .gz/.br，占磁盘 |
| access_log off | 日志量（静态 QPS 常是 API 百倍） | 失去细粒度访问统计 |
| try_files 兜底 | SPA history 刷新 404 | 入口必须 no-cache |

# 12. 小结

- **路径解析**：`root` 拼接、`alias` 替换，尾斜杠是 404 高发点；能用 root 不用 alias。
- **发送性能**：`sendfile on + tcp_nopush on + tcp_nodelay on` 是标准组合，零拷贝 + 攒包发送。
- **缓存分层**：hash 文件名资源 `max + immutable`，HTML 入口 `no-cache`，配条件请求（ETag/Last-Modified → 304）把校验成本降到几十字节。
- **压缩**：运行时 gzip level 5 或构建期预压缩 `gzip_static`；brotli 优先靠模块 + 预压缩，`gzip_vary on` 保护 CDN。
- **SPA 部署**：`try_files $uri $uri/ /index.html` 解决刷新 404，前提是入口不缓存。
- **CORS**：需要凭证不能 `*`；所有 CORS 头加 `always`，预检用 `if + return 204`。

下一篇（6_reverse_proxy）从"发文件"转向"转发请求"：proxy_pass 的 URI 拼接规则与反向代理的完整参数族。
