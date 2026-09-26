# 1. 知识点
- **OSI七层模型** — 物理层(比特流传输/网线光纤)、数据链路层(帧/交换机/MAC地址)、网络层(IP地址/路由/路由器)、传输层(TCP/UDP/端口)、会话层(建立管理会话)、表示层(数据的编码/加密/压缩)、应用层(HTTP/DNS/FTP/SMTP)
- **TCP/IP四层模型** — 网络接口层(对应OSI物理+数据链路)、网际层(对应网络层)、传输层(对应传输层)、应用层(对应会话+表示+应用)
- **HTTP 0.9** — 仅支持GET请求，只能传输HTML，无请求头／响应头，无状态码，响应完即断开TCP
- **HTTP 1.0** — 增加POST/HEAD方法，增加请求头和响应头(Content-Type等)、状态码，但默认短连接(Connection: keep-alive需手动开启)
- **HTTP 1.1** — 默认持久连接(Connection: keep-alive)，支持管线化(Pipelining，一个连接同时发出多个请求，但响应必须按序返回→队头阻塞)，增加PUT/DELETE/OPTIONS等方法，Host头(支持同一IP多虚拟主机)，断点续传(Range)，分块传输编码(Chunked)，缓存机制更完善
- **HTTP 2.0** — 二进制分帧层(将HTTP消息拆为帧)，多路复用(同TCP连接并发多请求/响应，解决HTTP/1.1队头阻塞)，头部压缩(HPACK)，服务器推送(Server Push可提前推送所需资源)，数据流优先级，但仍基于TCP所以TCP层面的队头阻塞依旧存在
- **HTTP 3.0** — 基于UDP的QUIC协议，多路复用无队头阻塞(QUIC各流独立)，0-RTT建连(缓存server config时)，更好的移动网络切换(连接ID迁移)，内置TLS 1.3加密
- **GET请求** — 获取资源，参数在URL后面(有长度限制约2K)，幂等安全，可被缓存/收藏/历史记录
- **POST请求** — 创建/提交数据，参数在请求体(request body)，无长度限制，不幂等，不缓存
- **PUT请求** — 更新/替换完整资源(客户端提供完整数据)，幂等
- **DELETE请求** — 删除资源，幂等(删除后再次DELETE结果一样)
- **PATCH请求** — 部分修改资源(非幂等)，如只更新某一个字段
- **HEAD请求** — 与GET类似但只返回响应头，用于检查资源是否存在/获取元信息
- **OPTIONS请求** — 查询服务器支持的HTTP方法，CORS预检请求(Preflight)使用
- **GET vs POST报文区别** — GET请求行: GET /url?params HTTP/1.1 无body；POST有Content-Type头+body传参数据，POST通常不会只传1次(非幂等)
- **幂等性** — 同一操作执行多次结果相同：GET/PUT/DELETE/HEAD幂等，POST非幂等，PATCH非幂等
- **HTTP状态码1xx** — 信息提示：100 Continue(大请求可先发头问服务器是否接收)，101 Switching Protocols(WebSocket升级)
- **HTTP状态码2xx** — 成功：200 OK(请求成功)，201 Created(资源创建成功通常在POST响应)，204 No Content(成功但无返回体)
- **HTTP状态码3xx** — 重定向：301 Moved Permanently(永久重定向，浏览器后续直接走新URl)，302 Found(临时重定向，浏览器仍用旧URL)，304 Not Modified(缓存未过期，不反回body)，307/308不改变请求方法和body
- **HTTP状态码4xx** — 客户端错误：400 Bad Request(请求语法错误)，401 Unauthorized(未认证)，403 Forbidden(已认证但无权限)，404 Not Found(资源不存在)，405 Method Not Allowed(请求方法不支持)，429 Too Many Requests(请求频率超限)
- **HTTP状态码5xx** — 服务端错误：500 Internal Server Error(服务器内部错误/代码Bug)，502 Bad Gateway(网关/代理收到无效响应)，503 Service Unavailable(服务暂时不可用/重启中/负载过高)，504 Gateway Timeout(网关超时)
- **HTTPS = HTTP + TLS/SSL** — 提供加密(对称)、认证(非对称+证书)、完整性(MAC)三个维度安全
- **对称加密** — 加解密用同一密钥，速度快，问题在于如何安全传输密钥
- **非对称加密** — 公钥加密私钥解密(加密传输)、私钥加密公钥解密(数字签名)，计算慢
- **TLS 1.2握手(2-RTT)** — ClientHello(随机数+支持的密码套件)→ServerHello(随机数+选定密码套件)+Certificate+ServerHelloDone→ClientKeyExchange(预主密钥用服务端公钥加密)+ChangeCipherSpec→双方由三随机数生成会话密钥→开始对称加密通信(2次RTT)
- **TLS 1.3握手(1-RTT)** — 简化套件，ClientHello带上猜测密钥参数→ServerHello+加密扩展→计算密钥，减少到1-RTT；且支持0-RTT(恢复连接时)
- **数字证书** — CA机构签发，包含域名+公钥+有效期+签名，浏览器内置CA根证书公钥验证证书链
- **证书链验证** — 用户证书→中间CA→根CA，逐级验证签名直到内置的根证书，检查域名是否匹配、是否在有效期内、是否被吊销(CRL/OCSP)
- **中间人攻击(MITM)** — 拦截通信伪装两端；HTTPS防中间人因为CA证书绑定公钥-身份，中间人无法伪造证书(除非私钥泄露或用户安装不可信证书)
- **URL输入到页面展示** — DNS解析域名→获得IP→TCP三次握手→(若HTTPS)TLS握手→发送HTTP请求→服务器处理返回响应→浏览器解析HTML构建DOM树→解析CSS构建CSSOM→合成渲染树→布局Layout→绘制Paint→显示页面; 各层的细节贯穿网络栈
- **DNS解析** — 将域名转换为IP，查询顺序：浏览器缓存→hosts文件→本地DNS→根域名服务器→顶级域名服务器→权威域名服务器；递归查询(LDNS帮全查)/迭代查询(LDNS返回下一步地址)
- **DNS记录类型** — A记录(域名→IPv4)、AAAA(域名→IPv6)、CNAME(域名→别名/域名的映射)、NS(指定域名由哪个DNS服务器解析)、MX(邮件交换记录)、TXT(域名附加文本信息，SPF/DMARC)
- **DNS负载均衡** — 同一个域名配置多个A记录，DNS轮询返回不同IP，简单但不支持权重且客户端可能缓存
- **CDN原理** — 智能DNS将用户请求导向地理/网络距离最近的CDN缓存节点；CNAME到CDN域名；节点命中缓存直接返回，未命中回源站拉取后缓存。
- **HTTPDNS** — 用HTTP请求代替传统DNS查询(绕过运营商LocalDNS劫持/解析慢/调度不准)，直接拿到IP
- **TCP三次握手** — SYN(syn=x)→SYN-ACK(syn=y,ack=x+1)→ACK(ack=y+1)。为什么是三次？1)防止过期连接请求到达服务端建立无效连接(场景: 第1次SYN延迟重发; 2次握手服务端直接建连浪费资源)；2)等双方确认收发能力(1次:服务端确认客户端发送能力; 2次:客户端确认服务端收发能力; 3次:服务端确认客户端接收能力)
- **半连接队列(SYN Queue)** — 服务端收到SYN后将该连接放入半连接队列，等待第三次握手，若满后新SYN丢弃(可能为SYN Flood)
- **SYN Flood攻击** — 伪造大量IP发送SYN，占满半连接队列 → 正常用户无法连接； 防御：SYN Cookie(不分配资源，cookie方式验证)、增加半连接队列、缩短超时
- **四次挥手** — FIN(seq=x)→ACK(ack=x+1)→FIN(seq=y)→ACK(ack=y+1)。为何是四次？TCP全双工，一方发送FIN仅表示本方无数据发了，仍可接收对方数据，对方也需发FIN确认关闭 → 两次FIN不可合并为三次
- **TIME_WAIT状态** — 主动关闭方在等2MSL(最长报文段生存时间，通常60s)：1)保证最后一个ACK能到达对端(若丢失对端重发FIN还能回应)；2)使本次连接内的报文都从网络消失，不影响新连接(MSL复用)
- **大量TIME_WAIT排查** — 大量短连接主动关闭 → TIME_WAIT过多；优化：1)服务端主动关闭(客户端少等待)；2)SO_REUSEADDR端口复用；3)连接池长连接减少新建关闭；4)tcp_tw_reuse(谨慎在小网开启)
- **TCP可靠传输机制** — 确认应答ACK + 超时重传(RTO动态计算基于RTT均值偏差) + 序列号 + 校验和
- **滑动窗口** — 发送方维持发送窗口(不等待ACK可连续发送多个段)，接收方通告窗口大小(rwnd)，流量控制：接收方窗口大小控制发送速度
- **拥塞控制** — 慢启动(cwnd从1指数增长到ssthresh)→拥塞避免(线性增长)→拥塞时ssthresh=cwnd/2,cwnd=1重新慢启动(Tahoe)或快恢复(cwnd=ssthresh,Reno)；快重传：收到3个重复ACK立即重传不需要等超时
- **TCP粘包/拆包** — 原因：TCP流式协议无边界，发送端缓冲区(Nagle)或接收端读取时机导致多个包粘在一起或一个包被分割。解决：固定长度、分隔符(如\r\n)、长度字段(header+body给出body长度)、自描述格式(Protobuf/JSON用分隔符)
- **UDP** — 无连接，不保证可靠/有序，面向报文(一次send就一个UDP包)，首部仅8字节(源端口+目标端口+长度+校验和)，快但不可靠；适用：音视频、DHCP、DNS(小查询)、QUIC(HTTP3)
- **Cookie** — 服务端通过Set-Cookie设置，客户端每次请求自动携带(同域)，大小限制4KB，属性：Domain(域)、Path(路径)、Expires/Max-Age(有效期)、HttpOnly(防XSS读取)、Secure(仅HTTPS)、SameSite(跨站限制Strict/Lax/None)
- **Session** — 服务端存储会话信息，通过SessionID(一般存Cookie或URL重写)关联客户端；问题：分布式环境下Session不共享
- **分布式Session方案** — 1)粘滞会话(负载均衡将同一IP固定定向同节点,单点故障丢失)；2)Session复制(多节点广播复制,冗余开销)；3)集中存储(Redis统一管理,主流方案)；4)JWT令牌无状态
- **JWT(JSON Web Token)** — 无状态认证, 包含Header+Payload+Signature三部分; Signature = HMAC(Header+B64(Payload), secret)，用于验证完整性和来源；Payload明文(不存敏感信息)；服务端不需保存Session
- **CDN预热** — 主动将资源提前推送到CDN各节点缓存中，避免初次访问回源慢
- **ARP协议** — Address Resolution Protocol, 通过IP获取MAC地址；广播ARP请求"谁是192.168.1.1告诉我MAC"，目标主机单播应答
- **RARP** — 反向地址转换，通过MAC获取IP(现在基本被DHCP取代)
- **NAT** — 网络地址转换，私有IP映射为公网IP出口，解决IPv4资源短缺；NAPT(端口复用)多个内网主机共用一个公网IP不同端口区分
- **ICMP** — 网际控制报文协议，ping(回送请求/应答)检测连通性，traceroute(利用TTL超时)追踪路由路径
- **IP协议核心** — 无连接、不可靠、尽力而为转发；TTL字段(每跳减1防无限循环)、分段与重组(MTU超限时分片)、首部校验和(仅校验首部)
- **WebSocket** — 全双工双向通信，通过HTTP Upgrade握手(返回101 Switching Protocols状态码)，之后数据帧传输，心跳保活(Ping/Pong帧)；对比HTTP长轮询：HTTP客户端poll仍半双工开销大，WebSocket真正双向推送
- **正向代理** — 客户端→代理→服务器；帮助客户端访问不可达资源(翻墙)，隐藏客户端真实IP，缓存上网加速
- **反向代理** — 客户端→(以为是)反向代理=服务器→实际后端；接收请求分发到后端服务器(Nginx)，隐藏服务端实现/负载均衡/缓存/SSL卸载/安全网关
- **Wireshark & tcpdump** — 抓包工具；tcpdump命令行抓包+过滤(host/port/protocol)导出pcap后用Wireshark GUI分析；Wireshark支持TCP流追踪(Follow TCP Stream)、统计、过滤器(ip.addr ==x等)
- **TCP和UDP的区别** — TCP面向连接/可靠/全双工流式/有序/首部20字节/适用可靠场景；UDP无连接/不可靠/面向报文/不保证顺序/首部8字节/快/适用实时视频等

# 2. 面试题

## 2.1 OSI七层模型和TCP/IP四层模型分别是什么？如何对应？
OSI七层：物理层(比特流)→数据链路层(帧/MAC)→网络层(IP/路由)→传输层(TCP/UDP/端口)→会话层→表示层(加密/编码)→应用层。TCP/IP四层：网络接口层(物理+链路)→网际层(网络层)→传输层(传输层)→应用层(会话+表示+应用)。OSI是理论参考模型更细，TCP/IP是工业事实标准，简化了上层合并为应用层。

## 2.2 HTTP各版本有什么演进？（1.0→1.1→2.0→3.0核心差异）
1.0: 默认短连接，一个请求一个TCP连接。1.1: 默认持久连接(keep-alive)，管线化但队头阻塞，增加Host头支持虚拟主机，断点续传。2.0: 二进制分帧，多路复用(同连接并发)，头部压缩HPACK，服务端推送，但TCP层面队头阻塞仍存在。3.0: 基于UDP QUIC协议，多路复用无队头阻塞(QPACK)，0-RTT建连，连接迁移。

## 2.3 HTTP的GET和POST有什么区别？
约定区别：GET参数放在URL中长度受限于浏览器约2K，数据可见；POST放请求体中长度不受限。语义区别：GET用于获取数据幂等安全；POST创建/提交数据非幂等。缓存：GET请求可缓存可收藏书签，POST不缓存。TCP包数：有的浏览器GET发1个包(请求头+数据一起)，POST发2个(先发请求头服务器回复100 Continue再发body)。实际POST也可带URL参数，GET也可有body但通常服务器忽略。

## 2.4 HTTP常见状态码有哪些？
2xx成功：200 OK，201 Created，204 No Content。3xx重定向：301永久(搜索引擎更新权重)、302临时(不建议用，浏览器可能改方法)、307/308严格不改变请求方法、304 Not Modified(缓存未过期)。4xx客户端错误：400 Bad Request，401 Unauthorized(未认证)，403 Forbidden(拒接访问)，404 Not Found，405 Method Not Allowed，429 Too Many Requests(限流)。5xx服务端错误：500 Internal Server Error，502 Bad Gateway(代理/网关收到上游无效响应)，503 Service Unavailable(服务临时不可用)，504 Gateway Timeout(代理等待超时)。

## 2.5 HTTPS的加密过程是怎样的？（TLS握手）
TLS 1.2握手(2-RTT)：1)ClientHello: 客户端随机数+支持的密码套件; 2)ServerHello: 服务端随机数+选定密码套件 + 证书; 3)客户端验证证书: CA链验证 → 取出服务端公钥; 4)ClientKeyExchange: 客户端生成预主密钥，用服务端公钥加密发送; 5)ChangeCipherSpec: 双方用客户端随机数+服务端随机数+预主密钥生成对称会话密钥，后续加密通信。TLS 1.3握手简化到1-RTT：ClientHello带上DH密钥参数猜测，可以直接在第二个往返就开始加密传输或0-RTT恢复。

## 2.6 浏览器输入URL后发生了什么？
1)DNS解析: 查找浏览器缓存→hosts→本地DNS→根DNS→顶级DNS→权威DNS→拿到IP; 2)建立TCP连接: 三次握手; 3)若HTTPS: TLS握手; 4)构建HTTP请求并发送(请求行+请求头+空行+body); 5)服务器收到请求后处理(Controller→Service→DAO→DB)，返回HTTP响应; 6)浏览器解析渲染: 解析HTML构建DOM树，CSS构建CSSOM，合成Render树，布局Layout(计算元素位置大小)，绘制Paint(光栅化显示); 7)遇到script根据同步/async/defer决定阻塞渲染行为。还可能涉及缓存策略(强缓存/协商缓存)、CDN寻址。

## 2.7 TCP为什么是三次握手而不是两次？
1)防止过期连接请求导致无效连接(客户端的旧SYN网络滞留，若两次握手服务器直接建立连接分配资源但客户端早已放弃，纯浪费资源)。2)三次握手确保双方收发能力均确认妥当：第一次服务端确认客户端有发送能力; 第二次客户端确认服务端有收发能力; 第三次服务端确认客户端有接收能力。两次的话服务端无法确认客户端是否收到自己的SYN响应。

## 2.8 TCP四次挥手为什么要等待2MSL？
主动关闭方收到对方FIN并应答ACK后进入TIME_WAIT需等2MSL(约60s)。原因：1)保证最后一个ACK能被对端收到(若丢失对方重传FIN，TIME_WAIT还能再次回应ACK)，否则被动关闭方卡在LAST_ACK; 2)让此次连接的报文全部在网络消失，避免和后续同端口新连接混淆。优化大量TIME_WAIT：短连接改长连接池/让被动关闭方发FIN(少等待)/tcp_tw_reuse。

## 2.9 TCP是如何保证可靠性的？（核心机制）
1)序列号与确认应答: 每个字节一个序列号，收到后回ACK确认; 2)超时重传: RTO基于RTT动态计算(绝对值偏差EWMA)，超时未收到ACK则重传; 3)校验和: 端到端校验确保数据无差错; 4)流量控制: 接收方通过rwnd滑动窗口控制发送速率; 5)拥塞控制: 慢启动→拥塞避免→快重传(3个重复ACK)/快恢复，防止网络拥塞恶化; 6)连接管理: 三次握手四次挥手确保连接状态正确。

## 2.10 TCP粘包/拆包问题以及如何解决？
原因：TCP流式协议无消息边界，应用层收发之间没有一一对应关系。发送端Nagle算法可能导致小包合并，接收端读取缓冲区大小不固定。解决方案：1)固定消息长度(不足补位); 2)消息分隔符(行分隔符\r\n，需转义避免数据内含分隔符); 3)消息头+消息体: 在头部4字节定长保存body长度，解析时先读头部获长度再读取对应长度body，这是最常用的方案(Netty LengthFieldBasedFrameDecoder); 4)使用高层协议如HTTP、WebSocket自带边界。

## 2.11 Cookie和Session的区别？
Cookie存在客户端浏览器，大小限制4KB，每次同域请求都自动携带(可设置HttpOnly/Secure/SameSite)，不安全(存敏感信息需加密)。Session存在服务端，客户端只存SessionID(通常通过Cookie或URL重写)，相对安全，但分布式横向扩展时需要Session共享方案(Nginx iphash/Redis集中存储/Sticky黏结)。Session过期则需重新登录，Cookie可设置持久化。

## 2.12 CDN的工作原理与应用场景？
CDN通过遍布各地区的缓存节点，把静态资源(图片CSS JS视频)缓存到离用户最近的节点。核心原理是智能DNS：用户DNS解析时，CDN的调度系统根据请求来源IP(地理位置/运营商)返回最优节点的IP地址，用户直接向该节点请求。节点有缓存直接返回，没有则回源站拉取并缓存(回源)。配合预热/刷新管理缓存。场景：网站加速、视频分发、直播、下载分发。

## 2.13 WebSocket与HTTP长轮询的区别？
HTTP长轮询：客户端发起请求，服务端hold住不立即返回直到有新数据，客户端收到后再次发起请求，仍是半双工基于请求-响应，每次都要带完整的HTTP头开销大。WebSocket：通过HTTP Upgrade握手建立TCP长连接后升级为ws协议，真正全双工双向推送，数据帧头部仅2-6字节开销极低，通过Ping/Pong帧保活检测断连。适用：实时聊天、游戏、股票行情、协同编辑等实时双向通信场景。

## 2.14 DNS解析过程是怎样的？
递归查: 浏览器DNS缓存→操作系统hosts→本地DNS服务器(递归)，若缓存无：根DNS服务器(返回负责该TLD的服务器地址)→顶级域名DNS服务器→权威域名服务器→返回IP。迭代查: 本地DNS服务器主动发多个请求。DNS主要记录: A(域名到IPv4)、CNAME(域名别名,CDN常用用户域名→CDN域名)、NS(DNS服务器指定)、MX(邮箱服务)。

## 2.15 HTTPDNS是什么？优势？
HTTPDNS是用HTTP请求(而不是传统的DNS查询)从DNS服务商获取域名对应的IP地址。优势：1)绕过运营商LocalDNS可能劫持、污染、故障; 2)更准确，支持根据来源IP做精确调度(返回最近节点IP); 3)降低DNS解析延迟(HTTP直接请求目标服务商)。通常配合客户端SDK缓存实现。

## 2.16 什么是数字证书和证书链？
数字证书由CA签发，包含网站域名、持有者公钥、有效期、签名(CA用自己私钥对上述信息摘要签名)。浏览器验证证书：1)用颁发CA的已知公钥验证签名是否匹配; 2)逐级追溯信任链到根CA(浏览器内置可信); 3)检查域名是否匹配; 4)检查是否在有效期内; 5)通过CRL/OCSP检查是否被吊销。任何一个环节失败浏览器显示不安全警告。

## 2.17 HTTP缓存机制: 强缓存与协商缓存？
强缓存：Cache-Control: max-age=xxxx(秒级别的存储时间)或Expires(绝对时间)，有效期内直接从缓存取不发送请求，返回200(from disk/memory cache)，优先级高于协商缓存。协商缓存：需要向服务器发请求验证缓存是否过期，服务器返回304 Not Modified(内容没变化需从缓存取)或200(重新发送内容)；通过Last-Modified/If-Modified-Since时间戳或ETag/If-None-Match指纹两种方式。强缓存有效时不请求直接取缓存，过期才走协商缓存。

## 2.18 TCP的拥塞控制算法简述？
拥塞窗口cwnd初始为1(MSS)，慢启动阶段每收到ACK指数倍增(cwnd*=2)。达到慢启动阈值ssthresh后进入拥塞避免阶段，每RTT发送窗口线性增长(cwnd+=1)。拥塞发生时：超时→ssthresh=cwnd/2, cwnd=1重新慢启动(Tahoe)。快重传：收到3个重复ACK→立即重传丢失报文段并进入快恢复：ssthresh=cwnd/2, cwnd=ssthresh+3, 之后在快恢复阶段收到重复ACK使其cwnd+=1，直到新ACK则退出→cwnd=ssthresh进入拥塞避免(Reno算法)。

## 2.19 限流算法中的令牌桶和漏桶区别？
漏桶：水(请求)进桶里以恒定速度流出→不管进来的速率如何，出去永远是恒定速率→强限制突发流量，桶满就丢弃。令牌桶：以恒定速度往桶里放令牌，请求来需要拿令牌才可处理→如果桶里积攒了令牌允许短时间突发流量(买峰值)同时平均速率仍然是恒定的。实际中令牌桶更适用于允许一定突发但同时限制平均速率的场景; 漏桶适合强制严格线性速率的场景。Guava RateLimiter就是令牌桶。

## 2.20 什么是TIME_WAIT状态？出现大量TIME_WAIT如何排查和处理？
TIME_WAIT是主动关闭方在四次挥手最后ACK发出后等待2MSL(60s)的状态。大量出现通常因为频繁短连接，尤其短HTTP连接或被压测的连接。可在服务端使用连接池(HTTP KeepAlive优化)，或调节系统参数(tcp_tw_reuse, tcp_tw_recycle慎用)，或让被动方关闭Nginx上设置proxy_connection_drop。SS/netstat -an|grep TIME_WAIT|wc -l检查数量。

## 2.21 什么是CDN？它的工作原理是什么？
CDN(Content Delivery Network，内容分发网络)通过遍布全球的缓存节点将静态资源分发到离用户最近的节点。**工作原理**：用户发起域名DNS解析请求 → CDN智能DNS调度系统根据请求来源IP(地理位置/运营商)返回最优边缘节点IP → 用户向该节点请求资源 → 节点缓存命中直接返回，未命中则回源站拉取并缓存后返回。配合**缓存预热**(主动推送资源到节点)和**缓存刷新**(更新过期资源)管理缓存。CNAME记录将用户域名指向CDN域名完成调度。适用于：网站静态资源加速、图片/视频/大文件下载加速、直播流分发、全站加速。优势：降低延迟、减轻源站压力、抗DDoS。
