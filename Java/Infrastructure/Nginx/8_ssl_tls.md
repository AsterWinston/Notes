
---

HTTPS 是 Nginx 最常被托付的基础设施职责之一：**把 TLS 终结在入口，证书集中管理，后端只跑明文 HTTP**。但 TLS 的配置细节与排错复杂度远超"挂个证书"的直觉——证书链顺序不对安卓就验不过、套件没收紧会被扫描器打低分、Java 客户端握手失败往往是曲线或协议版本问题。本篇口径为 Nginx 1.24/1.26 + OpenSSL 1.1.1/3.x。

# 1. HTTPS 基本配置

一个可用的 HTTPS server 块最少需要监听 443、证书、私钥三要素：

```nginx
server {
    listen 443 ssl;
    server_name www.example.com;

    ssl_certificate     /etc/nginx/ssl/example.com.pem;      # 证书文件（公钥）
    ssl_certificate_key /etc/nginx/ssl/example.com.key;      # 私钥文件

    # 可选：中间证书链
    # ssl_trusted_certificate /etc/nginx/ssl/chain.pem;
}
```

`listen 443 ssl` 里的 `ssl` 参数是给端口启用 TLS 的开关。**证书与域名必须匹配（CN 或 SAN）**，现代客户端只看 SAN 不看 CN，自签测试证书时尤其容易漏配 SAN 导致 hostname 校验失败。私钥文件权限须 `chmod 600`，带口令的私钥用 `ssl_password_file` 指定口令文件。

# 2. 证书相关指令

| 指令 | 说明 |
|------|------|
| `ssl_certificate` | 服务器证书（PEM 格式） |
| `ssl_certificate_key` | 私钥文件 |
| `ssl_trusted_certificate` | 中间证书 / CA 根证书链 |
| `ssl_password_file` | 私钥密码文件 |
| `ssl_protocols` | 启用的 SSL/TLS 协议版本 |
| `ssl_ciphers` | 加密套件 |
| `ssl_prefer_server_ciphers` | 优先使用服务端加密套件 |
| `ssl_session_cache` | SSL 会话缓存 |
| `ssl_session_timeout` | SSL 会话超时时间 |

## 2.1 证书链与 fullchain 坑

CA 签发的通常不是单张证书，而是一条**链**：`服务器证书 → 中间 CA → 根 CA`。浏览器内置了根 CA，但**不一定有中间 CA**，需要服务器在握手时把中间证书一起下发。漏发中间证书时，桌面 Chrome 可能因缓存了中间证书"看起来正常"，而安卓、部分 curl、Java 客户端直接报 "unable to find valid certification path"——**这是最典型的"我电脑能开手机打不开"故障**。正确做法是合并成一个 fullchain 文件喂给 `ssl_certificate`：

```bash
# 服务器证书在前，中间证书在后，顺序拼成一个文件
cat example.com.crt intermediate.crt > fullchain.pem
```

```nginx
ssl_certificate     /etc/nginx/ssl/fullchain.pem;   # 含服务器证书 + 中间证书
ssl_certificate_key /etc/nginx/ssl/example.com.key;
ssl_trusted_certificate /etc/nginx/ssl/fullchain.pem;  # 供 OCSP stapling 校验链用
```

> **注意**：链文件里证书**顺序必须是服务器证书在最前**（根证书可以不包含，客户端本地有）。`ssl_trusted_certificate` 用于**校验对方/OCSP 应答**的信任链，`ssl_certificate` 才是下发给客户端的证书；为简化部署常让两者都指向 fullchain。Let's Encrypt 生成的就是 `fullchain.pem`，直接用它，不要用单独的 `cert.pem`。

# 3. 协议与加密套件基线

协议层要禁用一切已知不安全的旧版本。SSLv3、TLS 1.0、TLS 1.1 均已被标准组织宣布废弃（POODLE、BEAST 等攻击 + 弱 PRF），PCI-DSS 早已强制下线：

```nginx
server {
    listen 443 ssl;
    server_name www.example.com;

    ssl_certificate     /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/example.com.key;

    # ── 协议与加密套件 ─────────────────
    ssl_protocols TLSv1.2 TLSv1.3;            # 禁用 TLSv1.0 / TLSv1.1（不安全）
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers on;
}
```

| 项目 | 现代基线（2024+） | 淘汰项 | 说明 |
|------|-------------------|--------|------|
| 协议 | TLS 1.2 + TLS 1.3 | SSLv3 / TLS 1.0 / TLS 1.1 | TLS 1.3 握手 1-RTT，优先启用 |
| 密钥交换 | ECDHE（前向保密） | RSA 密钥交换 | 必须支持 PFS，历史流量抗解密 |
| 对称加密 | AES-GCM、CHACHA20-POLY1305 | 3DES、RC4、AES-CBC | GCM 是 AEAD，自带完整性 |
| 签名 | RSA-2048+ / ECDSA | SHA-1 签名证书 | ECDSA 握手更快、CPU 更省 |
| 曲线 | X25519、secp384r1 | secp160 等弱曲线 | ssl_ecdh_curve 指定 |

`ssl_prefer_server_ciphers on` 表示握手以服务端套件顺序为准；**TLS 1.3 的套件是固定的**，`ssl_ciphers` 只对 TLS 1.2 及以下生效。

> **设计考量**：是否要留 TLS 1.0/1.1 完全取决于最老客户端。若确有 IE11/老安卓必须兼容，宁可让它连不上，也不要拉低主站安全基线；必要时单独划 server 端口承接遗留流量。

## 3.1 DH 参数与前向保密

使用 `DHE-*` 套件需显式生成 DH 参数（`openssl dhparam -out /etc/nginx/ssl/dhparam.pem 2048` 后配 `ssl_dhparam`），否则默认 1024 位有 Logjam 风险。**只保留 ECDHE 套件则可以不配**——现在常见做法是不在 `ssl_ciphers` 里放 `DHE-*`，全走 ECDHE，省掉这份握手 CPU。

# 4. HTTP 强制跳转 HTTPS

对外通常同时监听 80 与 443，把 80 的流量 301 到 443：

```nginx
# 方案1：单独 server 块做 301 重定向
server {
    listen 80;
    server_name www.example.com;
    return 301 https://$host$request_uri;
}

# 方案2：同 server 块内判断
server {
    listen 80;
    listen 443 ssl http2;
    server_name www.example.com;

    if ($scheme = http) {
        return 301 https://$host$request_uri;
    }
}
```

> 推荐**方案1**（两个独立的 server 块），更清晰且性能更好。

`return 301` 比 `rewrite` 更适合做协议跳转：不涉及正则、不触发 location 重匹配。**ACME 验证**常需要 80 端口的 `/.well-known/acme-challenge/` 放行明文，此时在 80 的 server 里单独为该路径配 location，其余才 301，别把验证路径也跳走了。

# 5. HSTS（HTTP Strict Transport Security）

强制跳转只防"第一次明文访问"，挡不住 SSL Strip 中间人降级攻击。HSTS 让浏览器在 `max-age` 内**只要访问该域名就自动走 https、根本不发明文请求**：

```nginx
server {
    listen 443 ssl http2;
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
}
```

| 参数 | 含义 |
|------|------|
| `max-age` | HSTS 生效时长（秒），推荐 63072000（2年） |
| `includeSubDomains` | 子域名同样强制 HTTPS |
| `preload` | 允许加入浏览器 HSTS Preload 列表 |

**HSTS 是单向门，一旦下发难以撤回**：浏览器在 max-age 内会拒绝该域名所有 http 访问，哪怕之后撤了证书、某个子域没配 https，用户也会看到无法绕过的错误。

> **警告**：加 `includeSubDomains` 与 `preload` 前，务必确认**所有子域都能走 HTTPS**。首次上线建议 `max-age` 从短（如 6 个月，且不带 preload）开始观察，稳定后再拉长并申请 preload 列表（preload 要求：有效证书、禁明文自动跳转、includeSubDomains、max-age ≥ 18 周）。

# 6. SSL 性能优化

TLS 握手的代价是**非对称运算 + 多次往返**，优化方向就是少握手、少往返：

```nginx
# 开启 SSL Session 缓存
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 1d;

# 开启 OCSP Stapling（减少客户端验证时间）
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 valid=300s;
resolver_timeout 5s;
```

**session 复用**有两条路。`ssl_session_cache shared:SSL:10m` 建共享内存缓存 **session ID**（10MB ≈ 4 万会话，`ssl_session_timeout 1d`），同一客户端重连命中缓存可跳过完整握手（TLS 1.2 从 2-RTT 降到 1-RTT）。多实例下缓存不共享，打到另一台就 miss；`ssl_session_tickets` 又有"ticket key 复用破坏前向保密"的争议，更安全写法是 `ssl_session_tickets off;` 并让多台机器共享 cache，或接受 TLS 1.3 的 PSK 复用。

> **注意**：TLS 1.3 的会话恢复（PSK/0-RTT）与 1.2 的 session cache 是两套机制，1.3 默认更快；`ssl_session_cache shared:` 的缓存名全局唯一，多个 server 复用同一份即可，别重复声明不同 name。

**OCSP Stapling** 解决证书吊销状态查询的延迟：客户端本要自己去 CA 的 OCSP 服务器问"这证书吊销没"，开启 stapling 后由 **Nginx 定期代查并把签名的 OCSP 应答"钉"在握手里发给客户端**，省一次外部往返。`ssl_stapling_verify on` 要求 `ssl_trusted_certificate` 指向 fullchain，`resolver` 解析 OCSP 服务器地址。

# 7. HTTP/2 搭配

HTTPS 之上应叠加 HTTP/2，享受多路复用与头部压缩：

```nginx
listen 443 ssl http2;    # Nginx 1.25.1 之前：http2 作为 listen 参数

listen 443 ssl;          # 1.25.1+ 推荐：独立指令，1.26 口径用这种写法
http2 on;
```

HTTP/2 必须建立在 TLS 上（浏览器普遍不接受 h2c 明文）。1.25.1 把 `listen ... http2` 参数拆成独立指令 `http2 on;`，因为旧参数曾导致一系列行为混乱。

# 8. 多域名 / 泛域名证书与 SNI

多域名靠证书的 **SAN（Subject Alternative Name）** 字段列出所有名字，一张证书覆盖多个域名：

```nginx
server {
    listen 443 ssl;
    server_name www.example.com api.example.com;   # 证书含多个 SAN
    ssl_certificate     /etc/nginx/ssl/multi.pem;
    ssl_certificate_key /etc/nginx/ssl/multi.key;
}

server {
    listen 443 ssl;
    server_name ~^(www|api|cdn)\.example\.com$;    # 泛域名证书
    ssl_certificate     /etc/nginx/ssl/wildcard.pem;
    ssl_certificate_key /etc/nginx/ssl/wildcard.key;
}
```

**SNI（Server Name Indication）** 让一个 443 端口按客户端握手时报的域名在不同 server 间选择对应证书，各 server 挂各自的 `ssl_certificate` 即可。

> **注意**：**客户端不发 SNI（老安卓、扫描器、IP 直连）时只能命中默认 server 的证书**，这是极少数"个别设备证书报错"的根因；兼容性敏感场景可配一张覆盖常用名的默认证书兜底。

# 9. 双向认证 mTLS（略讲）

普通 HTTPS 只验证服务端证书；**mTLS（mutual TLS）额外要求客户端出示证书、服务端验证**，用于零信任内网、开放 API 白名单、服务间调用：

```nginx
server {
    listen 443 ssl;
    ssl_certificate     /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/example.com.key;

    ssl_client_certificate /etc/nginx/ssl/client-ca.pem;   # 验证客户端证书的 CA
    ssl_verify_client on;                                  # on=强制要求, optional=可选
}
```

`ssl_verify_client on` 时没带合法客户端证书的握手直接失败；`optional` 则允许不带证书、由应用按 `$ssl_client_verify`/`$ssl_client_s_dn` 自行判断。mTLS 的复杂度在**客户端证书的签发、分发与吊销**，多配合私有 CA + 短周期轮换。

# 10. Let's Encrypt 免费证书 + 自动续签

```bash
# 安装 certbot
yum install -y certbot python3-certbot-nginx

# 获取证书（自动配置 Nginx）
certbot --nginx -d example.com -d www.example.com

# 仅获取证书（手动配置）
certbot certonly --nginx -d example.com

# 自动续签（添加到 crontab）
0 3 * * * certbot renew --quiet --post-hook "nginx -s reload"
```

Let's Encrypt 证书有效期仅 **90 天**，自动续签是刚需：crontab/systemd timer 每天跑两次 `certbot renew`（它自己判断 < 30 天才续），`--post-hook "nginx -s reload"` 保证续到新证书后重新加载。

> **注意**：`certbot renew` 只在真正续期时才触发 `--deploy-hook`（比 post-hook 更精确，仅在证书更新成功时执行 reload）。若 Nginx 与 certbot webroot 不同机，要用 `--webroot -w` 指定验证目录并确保 80 端口 `/.well-known/acme-challenge/` 未被 301 到 https、未被 deny 规则挡掉——ACME http-01 验证失败九成栽在这。

# 11. 自签证书（测试环境）

```bash
# 生成自签证书（仅限本地/内网联调，生产严禁）
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/nginx/ssl/selfsigned.key \
    -out /etc/nginx/ssl/selfsigned.crt \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=Test/CN=localhost"
```

# 12. SSL 测试工具与 Java 排错

```bash
# 在线检测：Qualys SSL Labs（https://www.ssllabs.com/ssltest/）
openssl s_client -connect example.com:443 -tls1_2
nmap --script ssl-enum-ciphers -p 443 example.com
```

`openssl s_client` 是最直接的排错入口：`-servername www.example.com` 模拟 SNI、`-tls1_3` 指定版本、看证书链是否完整、`Verify return code` 是否 0。**与 Java 客户端握手失败的排错**常见根因：

| 现象 | 根因 | 处置 |
|------|------|------|
| `unable to find valid certification path` | Nginx 未下发中间证书 / 服务端证书链不完整 | 用 fullchain.pem；`openssl s_client -showcerts` 验证链 |
| 同样报错但浏览器正常 | 证书在浏览器受信但不在 JDK `cacerts`（自签/私有CA） | 导入 JDK truststore 或用 `-Djavax.net.ssl.trustStore` |
| `no cipher suites in common` | JDK 版本老不支持 TLS 1.3 / Nginx 只开 1.3 | JDK 升级到 8u261+，或 Nginx 保留 TLS 1.2 |
| `handshake_failure`（RSA 证书） | JDK 8 早期不支持 ECDSA 或某曲线 | 服务端优先 RSA 证书或升级 JDK |
| 连接直接 reset，无 TLS 错误 | SNI 缺失命中默认 server 错证书 / IP 直连 | 客户端按域名访问，服务端配默认证书兜底 |

> **行业实践**：Java 侧握手问题先分"**信任问题**"与"**算法/版本问题**"。信任类（找不到有效认证路径）八成是证书链不完整或 JDK truststore 缺根，用 `openssl s_client -showcerts` 数链上几张证书再和浏览器对比；算法类（no common cipher / handshake_failure）多半是 JDK 与 Nginx 的 `ssl_protocols`/`ssl_ciphers` 交集为空。最省心的组合是服务端保留 TLS 1.2 + JDK 8u261 以上。

# 13. 小结

- **证书链**：用 fullchain（服务器证书 + 中间证书）；漏中间证书是"手机打不开/Java 报错而电脑正常"的头号元凶。
- **协议基线**：只留 TLS 1.2/1.3，禁用 1.0/1.1 与 3DES/RC4；优先 ECDHE 保证前向保密。
- **性能**：`ssl_session_cache` 复用会话、OCSP Stapling 省往返、`ssl_session_tickets off` 换 PFS 安全；HTTP/2 用 `http2 on;`。
- **安全**：80 用 `return 301` 跳 https；HSTS 防降级但下发即单向门（先短 max-age、慎用 includeSubDomains/preload）。
- **多域名**：SAN 多域名证书 + SNI 单端口多证书；无 SNI 客户端命中默认证书。
- **续签**：Let's Encrypt 90 天，certbot 定时 renew + reload；ACME 验证路径别被跳转/deny 挡。
- **Java 排错**：先分"信任问题 vs 算法/版本问题"，`openssl s_client -showcerts` 数链、对比协议/套件交集。

下一篇（9_rewrite_and_redirect）讲 URL 重写：rewrite 指令家族、location 匹配优先级与 if 的争议。
