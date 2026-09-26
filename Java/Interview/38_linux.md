# 1. 知识点
- **top命令** — 实时进程监控，PID/USER/PR/NI/VIRT/RES/SHR/S/%CPU/%MEM/TIME+/COMMAND; 交互操作: P按CPU排序、M按内存排序、1展开CPU核心、c显示完整命令、k发信号
- **uptime** — 系统运行时间和平均负载(load average): 1分钟/5分钟/15分钟三个数字，负载值表示运行+等待CPU+等待IO的任务平均值；CPU核数乘以0.7参考警戒值
- **load average** — 三个数值含义: 1分钟(瞬间)、5分钟(中期)、15分钟(长期)。若1min > 15min → 负载上升；若1min < 15min → 负载下降；持续>CPU核数 → 过饱和系统CPU在排队
- **CPU高判断** — top里%CPU(us用户/sy内核/ni nice/wa等待IO/hi硬中断/si软中断/st); 高us → 用户态繁忙(可查应用线程)；高sy → 系统调用繁忙/上下文切换过多；高wa → CPU在等IO，不一定是CPU瓶颈可能是磁盘/网络慢
- **free** — 内存统计: total/used/free/shared/buff/cache/available。available = free + 可回收的buffer/cache ≈ 剩余可用内存。不要只看free字段
- **free -h** — 人类可读的GB/MB显示内存大小
- **buffer vs cache** — buffer(块设备I/O缓冲区元数据)、cache(文件系统页缓存page cache读缓存文件)；Linux也统计page cache和slab内存回收
- **vmstat** — 虚拟内存统计; vmstat 1每秒更新: r(运行队列) b(等待IO阻塞) swapd(in swapped内存) free buff cache; si(从swap换入) so(换出swap) io(bi从块设备读块/s  bo块写); cs(每秒上下文切换) us sy id wa
- **dmesg** — 内核日志，dmesg | tail -20 看最后20行OOM killer/panic等；dmesg -T查看人类时间戳
- **df -h** — 查看挂载磁盘各分区使用率和挂载点
- **du -sh*** — 统计当前目录下文件/目录大小(h human-readable、s summarize该目录总大小)
- **iostat** — 磁盘IO监控；iostat -x 1 看%util(使用率饱和度)、await(平均I/O响应时间 含排队+服务)、svctm(服务时间)、r/s w/s rkB/s wkB/s；util接近100%表示磁盘瓶颈
- **netstat** — 网络连接和各种协议统计: netstat -anp(grep查看端口归属)、netstat -s(协议统计)、netstat -i(网络接口)。基本被ss替代
- **ss** — 更快的网络连接统计(替代netstat)；ss -tlnp(TCP监听-t含listen-l、numeric-n、process-p)；ss -s查看连接总数；ss -antp | grep ESTAB | wc -l ESTAB连接数
- **tcpdump** — 抓包工具: tcpdump -i eth0 host x.x.x.x and port 80 -w xxx.pcap(-w保存成Wireshark可读); 支持表达式过滤
- **ping** — 测试连通性，基于ICMP echo; ping -c 4(发4个结束) -i 间隔; 关注延迟和丢包率
- **telnet** — 测试端口服务连通性: telnet 某IP 端口; 连接成功说明端口可达 (新替代: nc -zv ip port)
- **curl** — 发送HTTP请求测试接口: curl -X POST -H "Content-Type: application/json" -d '{"key":"val"}' url; 常用 -v (verbose打印请求/响应头) -o (输出到文件) -k (忽略SSL验证)
- **iftop** — 实时网卡流量监控，需要root权限
- **ps aux** — 列出所有用户的进程: %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND。ps -ef看进程父子关系 PPID
- **pstree** — 进程树显示父子关系: pstree -p <pid> 显示某进程及其子进程的树结构
- **lsof** — 列出打开的文件/端口/进程: lsof -i :端口查监听此端口的进程; lsof -p pid查进程打开的所有文件和描述符; lsof +D directory查某目录被哪些进程打开
- **cat** — 查看文件内容(cat file全文打印)
- **more** — 分页查看，空格翻页，q退出
- **less** — 比more强大的分页阅读(支持搜索：/pattern向下搜索 ?pattern向上搜索、n跳下一个匹配、上下滚动)
- **tail** — 查看文件尾部; tail -n100 file(最后100行); tail -f file实时追踪文件追加内容; tail -F file同名但有重试(就算日志rotation也能重新打开新文件)
- **head** — 查看文件头部; head -n20 file(前20行)
- **grep** — 文本搜索: grep "pattern" file; -r递归目录; -i忽略大小写; -v反向匹配排除; -c统计行数; -n显示行号; -A/B/C数字 显示行前后文上下文
- **awk** — 文本处理: '{print $1, $3}'按列打印(默认空格分隔); -F指定分隔符; END{print NR, sum}累加; 支持条件过滤(variable判断)、内置变量NF列数、NR行数; 适合日志分析统计
- **sed** — 流编辑器: 's/old/new/g'替换; '/pattern/d'删除匹配行; '2,5d'删2-5行; '/pattern/a\new line'在匹配行后插入一行
- **wc** — 计数: -l行数、-w单词数、-c字节数
- **sort** — 排序; -n按数值排序(默认字典序待字母会造成9<10但字典序不同); -r逆序; -k2按第二列排序; -t, CSV分隔; sort -t ' ' -k2 -n 按第二列空格分隔数字排序
- **uniq** — 去重(要求相邻重复才有效应配合sort一起用): sort file | uniq -c 统计重复次数; uniq -d只输出重复的行
- **find** — 搜索文件: find /dir -name "*.log" -mtime -7(7天内修改); -type f/d; -size +100M(大于100M文件); -exec command {} \; 对匹配文件执行命令
- **chmod** — 改变权限: chmod 755 file (rwx r-x r-x, 4读/2写/1执行组成7/5/5); 字母模式: chmod u+x 加执行
- **chown** — 改变所有者: chown user:group file; chown -R递归目录
- **ln** — 创建链接: ln -s source target (软链接)、ln source target(硬链接)
- **Linux权限rwx** — 读4/写2/执行1加上属主u/属组g/其他o三位构成0777→rwxrwxrwx形式的权限。目录的x(执行)是进入目录的权限
- **特殊权限位** — SUID(4位: 文件执行时以属主权限运行如/usr/bin/passwd); SGID(2位: 文件执行以属组权限, 目录中新建文件继承组); Sticky Bit(1位: 目录只有文件拥有者和root能删除,如/tmp)
- **线上CPU飙升排查流程** — 1)top找到高CPU的进程PID; 2)top -Hp PID找到高CPU的线程PID(或ps -mp PID -o THREAD,tid,time); 3)printf '%x\n' TID 转十六进制nid; 4)jstack PID | grep nid -A30查看线程堆栈，定位代码行号(/具体定位类和方法)
- **内存泄漏排查** — 1)jmap -dump:format=b,file=heap.hprof PID导出堆内存Dump; 2)用MAT /JProfiler /jhat分析：看Histogram或Dominator Tree里最大对象(Retained Heap) 和 GC Root引用链; 3)多次间隔dump比较大小的差异(什么是持续增长)
- **Full GC频繁排查** — 1)jstat -gc PID 1000 10 (每秒一次共10次) 看YGC/YGCT/FGC/FGCT/GCT是否连续FGC; 2)free -h 先确认物理内存够不够(不够 JVM被迫FC); 3)查看JVM启动参数(-Xmx/-Xms/CMS/G1堆大小够不够、是否调优)；4)监控GC日志(-Xloggc...) visualvm / GCeasy分析; 5)jmap dump heap 分析大对象和可能导致Full GC的大版本内存分配/泄漏
- **慢接口排查** — 1)Arthas trace arthas: trace com.xxx.Controller method 查看请求链路每一层方法耗时(定位耗时环节); 2)SkyWalking / Zipkin / Pinpoint 调用链追踪看各服务间的耗时分布; 3)看代码逻辑和SQL可能全表扫描/索引缺失; 4)内存/GC暂停对接口抖动也可看jvm gc日志排查(STW)
- **磁盘快满排查** — 1)df -h列出各分区使用率; 2)du -sh /* | sort -rh 按根目录逐级找大目录(du --max-depth=1 -h /); 3)lsof | grep deleted 找出已删除但进程还持有文件描述符占用空间的大文件(删除后未释放); 4)定期清理策略: 如日志rotate保留特定天数的压缩.gz、core dump、tmp文件; 安排echo "''" > file清理(重定向空)
- **常用Shell一行命令** — 统计access.log中每个IP访问次数: awk '{print $1}' access.log | sort | uniq -c | sort -rn | head; 查找包含error的行并统计: grep 'ERROR' app.log | wc -l; 指定所有.log文件批量改名为.txt: find . -name "*.log" -exec bash -c 'mv "$0" "${0%.log}.txt"' {} \;
- **定时清理日志脚本** — 比如每天删除7天以前的日志: find /data/logs -name "*.log" -mtime +7 -exec rm -f {} \; 写入crontab 0 2 * * *
- **服务状态监控脚本** — 用curl检测HTTP健康接口: check_url="http://localhost:8080/health"; if ! curl -f -s -o /dev/null "$check_url"; then echo "Service down at $(date)"; systemctl restart service; fi
- **awk高级用法** — awk -F, '{sum += $3} END {print "Total:", sum}' data.csv 按列求和; awk '/ERROR/{count++} END{print count}' 统计; awk '{a[$1]++} END{for(i in a) print i, a[i]}' 分组统计
- **grep高阶** — grep -r "pattern" /dir(递归目录); grep -v "#" nginx.conf | grep -v "^$" (显示有效配置去掉注释和空行); grep -A 2 -B 1 "keyword" file(显示上下文); grep -E \<regex\>... 应用扩展正则(但不全支持Perl RE建议用ERE); grep -c统计总匹配行; grep -o只输出匹配部分
- **sed常见操作** — 替换: sed -i 's/old/new/g' file; 删除: sed -i '/pattern/d' file; 在第3行后插入: sed '3a\new line text'; 在第2行前i: sed '2i\insert'; 取行: sed -n '5,10p' file
- **systemctl** — systemctl start|stop|restart|status|enable|disable 服务名; systemctl daemon-reload重载service脚本; 查看所有服务: systemctl list-units --type=service; 查看开机启动: systemctl list-unit-files | grep enabled
- **service** — 传统SysV init管理: service 服务名 start|stop|restart|status
- **crontab** — 格式: * * * * * command 分别代表分(0-59) 时(0-23) 日(1-31) 月(1-12) 星期(0-7 0和7=周日); crontab -e编辑、-l查看; 指令最好用绝对路径。每5分钟执行: */5 * * * * command
- **visudo** — 用visudo安全编辑/etc/sudoers文件进行权限管理; 格式: user host=(run_as) NOPASSWD: command(免密)
- **SSH免密** — ssh-keygen生成密钥对，ssh-copy-id user@host将公钥拷贝到目标机器~/.ssh/authorized_keys即可免密登录
- **scp** — 远程拷贝文件: scp user@host:/remote/path /local/path; scp -r递归目录
- **rsync** — 文件同步工具: -avz增量内容对比更高效同步(常用同步文件、备份)

# 2. 面试题

## 2.1 top命令输出的load average三个数字代表什么含义？
load average显示1分钟、5分钟、15分钟的系统平均负载，代表处于running+uninterruptible sleep状态进程的平均数。1分钟高5/15低说明当前瞬时负载高但正在处理；15分钟持续高且1分钟也高说明长期满负荷；1分钟低但15分钟高说明负载在消减。单核系统>1表示过载，多核要除以核心数。对比实际CPU按核计算可以得出结论。

## 2.2 线上CPU 100%怎么排查？
1)top找到占用CPU最高的Java进程PID; 2)top -Hp PID找到该进程下CPU高的线程TIDs; 3)printf '%x\n' TID转十六进制为nid; 4)jstack PID | grep nid -A 30查看对应线程在做什么; 5)通常找到的是应用代码里的死循环(own code)、垃圾收集频繁(GC线程反而高)、或者JIT编译期瞬间高/死锁。若线程状态RUNNABLE且堆栈对应自己的代码说明应用自己的逻辑(排查耗CPU操作)。如果是GC线程高说明内存分配紧张。

## 2.3 线上内存泄漏怎么排查？
1)jstat -gc PID查看内存使用持续上涨不回收; 2)jmap -dump:format=b,file=heap.dump PID 在内存还没溢出前dump或设置-XX:+HeapDumpOnOutOfMemoryError内存溢出自动dump; 3)下载dump用MAT工具分析：打开Histogram/Leak Suspects->看Dominator Tree中占用Retained heap大的对象; 4)查看对它们的GC Root引用链确定是谁hold住了引用（如一直add到集合未清理、监听器未反转、ThreadLocal泄漏）。多份dump隔半小时对比找增长对象。5)通过代码分析修复。

## 2.4 Full GC过于频繁怎么办？
1)jstat -gc PID 1000多次打印：看Full GC次数是否在持续增长(可能每分钟数次)。2)free -h检查物理内存是否不足导致频繁换页(wap区)系统本身内存被压→swap。3)查看JVM参数: 初始-Xms和最大-Xmx差距是否过大会不断地申请分配。4)堆分配太小 老年代容易打满触发Full GC(增大-Xmx或优化内存分配)。5)若因大对象直接进入老年代(通过PretenureSizeThreshold参数)可调整；/如果是代码泄露先定位大对象避免它占用。6)考虑GC器选择 G1/Parallel/CMS按长短场景优化，查看gc日志解决分配失败(Allocation Failure)导致的Full GC。

## 2.5 如何排查某个接口响应很慢？
1)Arthas: trace className methodName追链路查看每一级方法（Controller→Service→DAO→外部调用等）的耗时明细，精准找耗时瓶颈。2)APM平台 SkyWalking/Pinpoint看完整的分布式服务调用链、各服务耗时/SQL、外部调用。3)看最近发布的代码是否引入慢逻辑或N+1查询。4)数据库端慢SQL→ explain 查看执行计划，索引缺失/全表扫描。5)网络排查：外部调用接口超时、DNS解析慢。6)资源瓶颈：GC暂停时间排查 jstat/jmap/日志。

## 2.6 磁盘空间满了怎么排查？
1)df -h查看各分区使用率找出已满分区; 2)du -h --max-depth=1 / | sort -rh 从根出发逐级找大目录大文件，逐层深挖定位到哪一个日志/核心dump/临时数据/上传文件太大; 3)lsof | grep deleted → 显示已删除（文件名标记为deleted）文件描述符但进程仍持有句柄未释放它们占用的磁盘→定期重启或echo > /proc/<pid>/fd/fd号清空; 4)日志rotate配置文件不生效或压缩周期太长，配合定时清理策略(crontab).

## 2.7 TCP连接中TIME_WAIT过多怎么办？
先看数量: ss -ant | grep TIME_WAIT | wc -l。如果数量上千持续：1)让应用使用HTTP长连接(keep-alive连接池)减少短连接创建/关闭频繁; 2)反向代理Nginx加proxy_set_header Connection "" 且与上游走长连接; 3)让被动关闭方(如服务端)主动发FIN (客户端减少TIME_WAIT); 4)Linux内核参数优化: net.ipv4.tcp_tw_reuse = 1 (允许TIME_WAIT端口给新连接复用，仅客户端有效), nets.ipv4.tcp_timestamps=1 (配合开启)。

## 2.8 ss命令和netstat的区别？常用场景？
两者都列网络连接。netstat会读大量/proc下文件效率较低（socket信息递归/slab等），连接量大时很慢。ss直接查询内核socket快的多。常用：ss -tlnp查TCP监听端口并显示进程；ss -antp按连接状态/进程查所有(TCP)；-t/-u按TCP/UDP过滤；-s查看统计信息。基本现代系统都应优先用ss替代netstat。

## 2.9 如何找到占用某端口的进程？
方式一: lsof -i :8080 直接打印出进程名+PID(最简洁)。方式二: ss -tlnp | grep :8080(找到PID)再ps PID查看。方式三: netstat -tlnp | grep :8080。三者都可以。知道PID后续可以看进程所属的服务、jstack等。

## 2.10 如何找到某个进程打开的文件和网络连接？
lsof -p PID 列出打开的文件描述符(包括日志、jar文件、socket句柄)极有用。定位Java进程用了哪些本地端口: lsof -p PID | grep TCP。或者通过ls -l /proc/PID/fd看所有文件描述符。

## 2.11 grep常见的高级用法有哪些？
grep -r "keyword" /path 递归; grep -i 忽略大小写; grep -v "排除" 显示不匹配; grep "key" -A 3 -B 2 显示前后3/2行上下文; grep -c "pattern" file计数; grep -E "regex1|regex2" 扩展正则多条件; grep -o print(仅显示匹配本身); 用来过滤空行和注释: grep -v "^#" nginx.conf | grep -v "^$"。

## 2.12 awk如何按列求和和分组统计？
按列求和(以空格分隔): awk '{sum+=$3} END{print sum}' file; 第3列相加最后打印总和。按第一列分组统计出现次数: awk '{count[$1]++} END{for(i in count) print i, count[i]}' file | sort。还可以搭配条件统计，结合NR行列、NF字段数和格式化输出完成复杂日志分析。例如统计接口耗时分布: awk '{time[$4]++} END{for(t in time) print t, time[t]}' logs。

## 2.13 sed的常见替换和删除操作？
替换全文件: sed -i 's/old_word/new_word/g' file(原地修改)。批量删除所有注释行(#开头)并在原地生效: sed -i '/^#/d' file。在后面添加：sed '/pattern/a\new line content' file。取指定范围行: sed -n '50,100p' file显示50-100行。MAC由于-i后加扩展名需写 sed -i '' 's/...'。

## 2.14 如何统计access.log中访问量前十的IP？
cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -10。第一步 awk提取IP(一般第1列)，第二步对所有IP排序使相同相邻，第三步uniq -c聚合并计数，第四步排序按出现次数降序，最后head取前10。也可以gunzip -c * | awk... 分析压缩历史。

## 2.15 如何监控Java GC情况？
jstat -gc PID 1000每1秒一次打印(frequently 排查)。关注: YGC/YGCT/FGC/FGCT/GCT等指标。如果FGC频繁=STW影响响应时间; 配合GC日志参数-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:gc.log，将日志导入 GCeasy / GCViewer 分析GC趋势。Arthas dashboard也能实时看GC。prometheus + jmx_exporter 适合中长期可视化。

## 2.16 怎么批量杀死含特定关键词的进程？
ps aux | grep "关键词" | grep -v grep | awk '{print $2}' | xargs kill -9。先列出匹配进程(过滤掉grep自身行)，提取PID，再用xargs kill。注意非常危险：确保关键词精准否则误杀其他进程；也可以先打印出后再手动kill: ps -ef | grep "xxx" | grep -v grep。

## 2.17 tcpdump如何抓取某个端口和主机的包？
tcpdump -i eth0 host 192.168.1.100 and port 8080 -w capture.pcap。host是对端或本机IP、port是端口、-w保存为pcap文件Wireshark打开分析。加选项 -A以ASCII显示包内容(查看HTTP明文)，-X以十六进制+ASCII。不加-w直接打印到终端包摘要。-s0抓完整包不截断。进阶过滤器 src/dst host, tcp/udp 协议。

## 2.18 crontab的格式和每5分钟执行怎么写？
格式分 分 时 日 月 星期: */5 * * * * /path/to/script.sh 表示每5分钟执行一次。更精确规则也可写为 0,5,10,15...55 * * * *。生产环境建议加日志 >redirect 2>&1，脚本含绝对路径和必要的环境变量。

## 2.19 什么是SUID、SGID、Sticky Bit？
SUID(4xxx)针对可执行文件，设置后执行者以文件所有者权限运行(如/usr/bin/passwd 拥有者是root普通用户改密码时需要修改/etc/shadow只有root可写)。SGID(2xxx)针对文件:执行时拥有组权限;针对目录:该目录下新建文件的组继承目录组(方便共享协作)。Sticky Bit(1xxx)只对目录有效，标定此目录内任何人创建的文件只有文件所有者或root能删除，如/tmp。ls -l 中 SUID显示rwsrwxrwx里首个x表示为s; SGID写在组权限那组。
