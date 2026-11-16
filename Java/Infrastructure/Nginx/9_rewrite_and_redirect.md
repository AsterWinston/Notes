
---

rewrite 是 Nginx 的"流量整形"工具：补全跳转、隐藏真实路径、做伪静态、统一域名与协议形态。看似只有 `rewrite`、`return`、`if` 几条指令，但它们的**执行阶段与 location 匹配优先级交织在一起**，出错时表现为"死循环 500"或"规则明明写了就是不生效"。本篇按"指令家族与执行顺序 → location 匹配优先级 → rewrite/return 实战场景 → if 的争议 → error_page 与调试"展开。

# 1. rewrite 指令家族与执行顺序

rewrite 相关指令共有六条，分属不同阶段，**搞清楚"谁在什么时机生效"是全部问题的根源**：

```nginx
rewrite regex replacement [flag];   # 改写 URI（rewrite 阶段）
set $var value;                     # 自定义变量（rewrite 阶段，按书写顺序）
if (condition) { ... }              # 条件包裹，本身不是指令（rewrite 阶段产生隐式 location）
break;                              # 停止 rewrite 阶段，直接进入内容处理（find-content 后）
return code [text];                 # 直接产出响应，终止后续所有处理
uri / 内部指令（try_files 等）       # 内容处理阶段
```

| 指令 | 所属阶段 | 作用 | 典型场景 |
|------|---------|------|---------|
| `rewrite ... last` | rewrite | 改写 URI 并**重新走一遍 location 匹配** | 伪静态、目录补斜杠 |
| `rewrite ... break` | rewrite | 改写 URI 但**不再重匹配**，留在当前 location | 配合 proxy_pass/root 改路径 |
| `rewrite ... redirect` | rewrite | 改写并发 302 给浏览器 | 临时迁移 |
| `rewrite ... permanent` | rewrite | 改写并发 301 给浏览器 | SEO 规范 URL |
| `set` | rewrite | 写自定义变量 | 攒条件、给 map 补位 |
| `if` | rewrite | 条件判断容器 | UA/方法/文件存在性判断 |
| `return` | rewrite（最优先） | 直接结束并响应 | 强制 https、封路径 |

**执行顺序要点**：同一 location 内，`return` 先于 `rewrite` 生效（写了 return 后面的 rewrite 是死代码）；多条 `rewrite` 按书写顺序依次尝试；`last` 触发的重匹配会再次执行新 location 里的 rewrite 指令——**新旧 location 互相 rewrite 回对方就是 500 endless loop**，error.log 里 `use uninitialized ... cyclic rewrite` 说的就是它。

> **注意**：`last` 与 `break` 的选择取决于"改写后的 URI 希望被谁处理"。改写后要落到**另一个 location**（比如图片路径改写后交给专门的缓存 location）用 `last`；改写只是给**当前 location 的 root/proxy_pass 用**、不希望再匹配一轮用 `break`。用错方向的直接症状：用 `last` 写出循环，用 `break` 改写后仍被原 location 处理。

# 2. location 匹配规则

rewrite 的"目标 location 是谁"由匹配优先级决定，这是必须先立住的地基。

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

```text
1. 精确匹配 =     → 找到则停止
2. 前缀匹配 ^~    → 找到最长匹配，停止正则检查
3. 正则匹配 ~ ~*  → 按配置顺序，找到第一个匹配即停止
4. 普通前缀匹配   → 按最长匹配原则
```

正确的完整心智模型分五步：① 先找 `=` 精确匹配，命中直接结束；② 否则扫描所有前缀匹配（`^~` 与普通前缀一起比），**记住最长的那个**；③ 若最长前缀带 `^~`，直接用它、跳过正则；④ 否则按**配置文件书写顺序**逐条试正则 `~`/`~*`，命中第一条立即用它（正则之间没有"最长"概念，顺序即优先级）；⑤ 正则全不中，回到第②步记住的最长前缀 location。

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

**请求 `/static/logo.png` 命中谁？** `^~ /static/`——前缀阶段它比 `/` 长，`^~` 使图片正则不再参与。这正是 `^~` 的存在意义：**保护某个目录不被下方某条正则劫持**，典型的正则劫持事故是 `location ~ \.php$ { fastcgi_pass ...; }` 把 `avatar.png.php` 或上传目录里的双扩展名文件解析执行。安全上要求"上传目录内文件永不当代码执行"时，就该给上传目录加 `^~` 并在其中显式 deny php。

> **设计考量**：`= /favicon.ico`、`= /robots.txt` 这类高频固定路径用精确匹配可以跳过前缀树的其余比较与正则扫描，配合 `access_log off; log_not_found off;` 是老牌优化项；但收益在现代版本里已很小，更多是表达"我要独占这个 URI"的意图。

# 3. rewrite 实战场景

## 3.1 正则捕获与变量

rewrite 用 PCRE 正则，括号捕获经 `$1`–`$9` 引用，也可用命名捕获：

```nginx
# 将 /file/xxx 重写为 /download.php?file=xxx
location / {
    rewrite ^/file/(.*)$ /download.php?file=$1 last;
}

# 伪静态：/article/123.html → /article.php?id=123
location / {
    rewrite ^/article/(\d+)\.html$ /article.php?id=$1 last;
}
```

replacement 里**出现 `?` 会丢弃原查询串**，想保留要显式追加 `$is_args$args`；引用不存在的捕获组（如正则只有两组却写 `$3`）配置检查直接失败。`$uri` 是**解码后、去掉参数**的当前 URI，`$request_uri` 是**原始编码带参数**的 URI——rewrite 作用于前者，做整段跳转（https 升级、域名迁移）保留现场要用后者，混用会丢参数或把 `%20` 之类解码再编码出乱码。

## 3.2 强制 https 与域名规范化

```nginx
# 裸域 → www，同时统一到 https（老域名迁移同理）
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://www.example.com$request_uri;
}
```

`return 301` 恒定优于 `rewrite ... permanent`：省一次正则引擎，意图也更直白。**301 与 302 的选择要慎**：301 会被浏览器与搜索引擎**永久缓存/转移权重**，测试跳转请用 302，确认无误再改 301——301 一旦发错，用户浏览器里的错误缓存比配置修复的生效期长得多。

> **注意**：301 响应必须带上完整目标 URI（`$request_uri` 保留路径与参数），漏掉参数的 301 会把带查询串的入口页重定向到错误页面；HTTPS 域名迁移时优先一跳直达 `https://新域`，先跳 https 再跳新域的两跳方案会把跳转次数浪费在中间态上。

## 3.3 目录尾斜杠

```nginx
# 目录没有 / 时，自动添加 /（relative redirect）
location /blog {
    return 301 /blog/;
}
```

尾斜杠问题不止是风格：不带斜杠时相对路径的基准不同，页面里 `src="a.css"` 会解析成 `/a.css` 而不是 `/blog/a.css`，资源全 404。**目录必须 301 到带斜杠形态**；静态目录 Nginx 其实会自动处理（磁盘上是目录时 `server` 级 `absolute_redirect` 默认产出 301），显式写出来是为了动态路由或后端应用目录。反过来"去斜杠"（`rewrite ^(/.+)/$ $1 permanent;`）多见于 SEO 统一 URL 形态，注意别把 `/` 根路径自己圈进去。

# 4. return 指令

```nginx
# 语法
return code [text];
return code URL;

# 典型用法
return 301 https://www.example.com$request_uri;          # 301 重定向
return 302 https://www.example.com/private$request_uri;  # 302 临时重定向
return 403 "Access Denied";                              # 返回状态码 + 消息
return 204;                                              # 纯状态码
```

`return` 在 rewrite 阶段**最先执行**且立即终止请求处理，`return 301/302` 的 URL 可以是完整 URL（外部跳转），也可以是相对路径（补全成当前 scheme+host）。`return 200 "text"` 常用做轻量健康检查端点或给探针回固定 JSON，**不需要真的存在文件或后端**——比建个空 location 干净。对 `return code;` 非标准码（如 8xx）也能用，一些站点用 `return 444`（Nginx 特有）直接**不回响应断开连接**，对抗恶意扫描非常经济。

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

**"if is evil" 是 Nginx 官方文档（"If is E-V-I-L"附录）的原话**，根源在实现：`if` 会创建**隐式嵌套 location**，外层 location 的指令不会按直觉继承；而 `add_header`/`location` 级指令又有"子级出现即屏蔽父级同名继承"的规则，组合起来就出现"加了 if 之后某些 header 莫名消失、proxy_pass 行为漂移"。社区总结的**安全用法边界**是：`if` 只配 **`return`** 或只配 **`rewrite ... last`** 这两种"立即终止/重定向"指令，几乎不出错；`if` 块里放 `proxy_pass`、`add_header`、`fastcgi_pass` 等则高危。

**替代方案**按场景选：

| 场景 | if 写法（危险） | 替代写法 |
|------|----------------|---------|
| 多条件组合判断 | 嵌套 if | `set` 攒标志位 + 判断聚合变量 |
| 按 UA/来源分流变量 | if + set | **`map` 指令**（http 级，声明式，无坑） |
| 文件存在性分流 | `if (!-f $request_filename)` | `try_files $uri ... 兜底` |
| 屏蔽方法 | if + 正则 | `limit_except` |
| 按扩展名走不同策略 | if + 后缀 | 拆成正则 location |

`map` 是最值得记的替代品：

```nginx
# http 级声明：UA 含 mobile → $device=mobile，否则 desktop
map $http_user_agent $device {
    default       "desktop";
    ~*android|iphone "mobile";
}
```

map 在**变量被取用的瞬间惰性求值**，没有 location 副作用，把"条件逻辑"压回数据层面，location 内只用 `$device`，行为可预测。

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

`error_page` 的 `=` 号语义最易被忽略：不带 `=` 时**保留原状态码**（404 页面回 404，正确利于 SEO）；`error_page 404 = /404.html` 把状态改回 200（软 404，搜索引擎会把死页面当活页面收录，慎用）；`= @notfound` 则内部跳转到命名 location。`proxy_intercept_errors on`（默认 off）决定后端应用返回的 HTML 错误页是否被 Nginx 的 error_page 接管——统一错误页风格时要打开，但要确认业务错误响应（往往也带 4xx/5xx 码）不被误吞，通常配合后端"业务错误一律回 200 + JSON"的约定。

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

这里的 `break` 出现在 `if` 块内、不带 rewrite flag 语义，作用同上：**文件存在就终止 rewrite 阶段**（避免再执行下面的 rewrite），直接进入内容处理把静态文件返回；不存在才伪静态到 `index.php`。等价且更安全的现代写法是 `try_files $uri /index.php?$query_string;`——**一行、无 if、行为可预测**，这也是 WordPress/Laravel 的官方 Nginx 模板现状。

# 8. Rewrite 日志调试

```nginx
error_log /var/log/nginx/error.log debug;       # 开启 debug 级别日志
rewrite_log on;                                  # 专门记录 rewrite 过程（输出到 error_log）
```

`rewrite_log on` 把每条 rewrite 的匹配结果、标志、URI 变化记入 error.log（info 级），**排查"规则没生效/死循环"的第一开关**；debug 级输出量巨大，仅限复现问题时临时开。循环重写报 `cyclic rewrite` 时，检查两条规则是否把 URI 改来改去互为起点，`last` 换 `break` 或加条件短路是解法；rewrite 不生效则优先怀疑：`return` 挡在前面、location 根本没匹配到（回到第 2 节优先级流程核对）、以及正则锚点遗漏（`rewrite ^/foo` 会同时匹配 `/foobar`，收尾要 `$`）。

# 9. 小结

- **执行顺序**：return 最先终止 → rewrite/set/if 按书写顺序执行 → `last` 重匹配 location、`break` 停在原地；循环 rewrite 的根因是两条规则互相 last。
- **location 优先级**：`= 精确 > ^~ 最长前缀 > 正则按书写顺序 > 普通最长前缀`；`^~` 用来保护目录不被正则劫持。
- **变量选择**：rewrite 操作解码后的 `$uri`，整站跳转保留现场用 `$request_uri`；replacement 含 `?` 会丢原参数，需 `$is_args$args`。
- **if 争议**：官方承认 if is evil——只在 if 里写 `return`/`rewrite last`；其余场景用 `map`/`try_files`/`limit_except` 替代。
- **error_page**：`=` 控制状态码是否改写；软 404（=200）伤 SEO；后端错误页接管要 `proxy_intercept_errors on` 且防误吞业务错误。
- **调试**：`rewrite_log on` + 核对 location 命中，是最短排错路径。

下一篇（10_caching）进入代理缓存：proxy_cache 的键、TTL 分层与缓存穿透防护。
