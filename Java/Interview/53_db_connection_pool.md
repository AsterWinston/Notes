# 1. 知识点
- **连接池核心作用**：复用数据库连接，避免频繁创建/销毁连接的三次握手TLS开销和数据库资源消耗
- **连接池解决的问题**：连接复用减少延迟、防止连接数暴增压垮数据库、统一管理连接生命周期、监控连接状态
- **HikariCP**：Spring Boot 2.0起默认连接池，极致性能，号称最快的Java连接池，核心设计ConcurrentBag、FastList优化
- **HikariCP ConcurrentBag**：无锁化并发连接管理容器，借用连接synchronized短临界区，归还连接无锁，使用ThreadLocal缓存连接
- **Druid**：阿里巴巴开源数据库连接池，功能全面，不仅连接池还有丰富的监控统计和SQL防御功能
- **Druid监控统计**：内置StatFilter统计SQL执行次数、耗时、并发、慢查询，提供可视化监控页面
- **Druid SQL防注入**：WallFilter实现SQL防火墙，拦截危险SQL（如DROP TABLE、注入攻击）并记录
- **Druid Filters拦截链**：可插拔Filter链，在连接获取/执行/关闭等环节插入自定义逻辑（监控/防御/日志/配置）
- **StatFilter**：统计SQL执行信息，包括执行次数、返回行数、更新行数、执行时间、错误次数、并发峰值
- **WallFilter**：防御SQL注入攻击，自动识别并拦截恶意SQL，可配置白名单/黑名单规则
- **Log4j2Filter**：使用Log4j2输出JDBC执行日志，便于追踪SQL执行详情
- **ConfigFilter**：支持从配置文件动态读取数据库密码等配置
- **慢SQL记录**：Druid内置通过StatFilter的慢SQL阈值配置记录执行时间过长的SQL到监控页面
- **连接泄漏检测**：检测长时间未归还的连接，核心参数removeAbandoned和logAbandoned
- **removeAbandoned**：是否自动回收泄露连接，生产慎用（可能中断正在执行的事务）
- **logAbandoned**：仅记录泄露但不回收，生产环境推荐的排查方式
- **responsibleTo**：标记连接持有人，便于定位泄露连接是谁未关闭
- **WebStatFilter**：Druid提供的Web请求监控Filter，统计URI请求与SQL关联
- **minIdle**：连接池最小空闲连接数，保持池中有这些就绪连接应对突发请求
- **maxActive**：连接池最大活跃连接数，防止连接数过多压垮数据库
- **maxWait**：获取连接最大等待时间（毫秒），超时抛出异常，避免请求无限等待
- **validationQuery**：连接有效性检测SQL（如SELECT 1），确保借出的连接可用
- **testWhileIdle**：空闲连接检测，归还连接池后定期用validationQuery检测，无效则清理掉
- **testOnBorrow**：每次借出前检测有效性，性能开销大，一般不开启
- **timeBetweenEvictionRunsMillis**：空闲连接回收线程的运行间隔，同时也是testWhileIdle的检测周期
- **keepAlive**：对空闲时间超过minEvictableIdleTime的连接发送心跳保持连接，防止被数据库超时断开
- **连接泄漏排查流程**：发现连接池达到maxActive无连接可用 → 开启logAbandonedDebug → 分析日志找到未归还的SQL和调用栈 → 定位代码修复（通常是未在finally中关闭连接）
- **Spring事务与连接绑定**：Spring事务管理器通过TransactionSynchronizationManager将数据库连接绑定到当前线程（ThreadLocal），同一事务内的多次DAO操作共享同一个连接
- **事务同一连接**：事务开始获取连接→绑定线程→DAO操作使用同一连接→事务结束解绑归还连接；保证事务原子性
- **事务隔离级别对连接的影响**：不同事务可能有不同的隔离级别，同一连接在事务中不能切换隔离级别
- **事务传播属性与连接**：PROPAGATION_REQUIRED时已有事务复用当前连接；REQUIRES_NEW时挂起当前连接获取新连接
- **多数据源**：一个应用连接多个数据库（多租户、读写分离、跨业务）
- **多数据源配置方式**：定义一个DataSource为主数据源（@Primary），另一个为次数据源，分别用SqlSessionFactoryBean或EntityManager绑定不同的包扫描路径
- **dynamic-datasource**：动态数据源切换框架，@DS注解方法级切换，支持一主多从、多主多从
- **@DS注解**：作用在方法或类上指定使用的数据源，支持DS（DynamicDataSource）路由
- **分布式事务连接管理**：Seata AT模式下各阶段自动管理连接的获取和释放，应用代码无需感知
- **同一事务多数据源**：必须包装在全局分布式事务中（Seata/XA），否则各数据源独立提交无法保证一致性
- **Tomcat JDBC Pool**：Tomcat内置连接池，性能比DBCP好但功能单一，Spring Boot早期默认
- **Commons DBCP2**：Apache Commons Database Connection Pool 2代，功能完善但有性能问题，已被HikariCP取代
- **Druid连接泄漏检测原理**：Connection对象包装为DruidPooledConnection，记录获取时间/线程栈，定时检查连接持有时长超过阈值则记录泄露信息

# 2. 面试题

## 2.1 数据库连接池的作用是什么？为什么需要它？
数据库连接池预先创建一定数量的数据库连接存放在池中，应用程序从池中借用连接，使用完毕归还。避免了每次请求都创建连接（TCP三次握手+数据库认证开销，单次约10-50ms）和销毁，同时限制最大连接数防止应用异常导致数据库连接数暴增。核心价值：连接复用减少延迟、资源管控保护数据库、统一监控管理。

## 2.2 HikariCP和Druid有什么区别？如何选择？
- HikariCP：极致轻量高性能（号称最快Java连接池），Spring Boot 2.0+默认，核心特性：ConcurrentBag无锁并发连接容器、FastList优化Statement关闭、精简代码（约130KB）。适合追求极致性能、不特别需要监控功能的场景。
- Druid：功能全面的连接池，不仅提供连接管理还有丰富的SQL监控统计、SQL防注入（WallFilter）、慢SQL记录、可视化监控页面、Web请求关联SQL等。适合需要细粒度监控和安全防护的生产级系统。

选择：如果只需要高性能连接池，用HikariCP；如果需要SQL监控、防注入、运维可视化等能力，用Druid。

## 2.3 Druid的核心功能Filter有哪些？各自作用是什么？
- StatFilter：SQL统计监控，记录SQL执行次数、耗时、返回行数、并发数、慢查询
- WallFilter：SQL防注入防火墙，自动检测并拦截恶意SQL（如WHERE 1=1、DROP TABLE、UNION SELECT等），支持黑白名单规则
- Log4j2Filter / Slf4jLogFilter：将JDBC执行日志输出到日志框架，便于追踪SQL执行详情
- ConfigFilter：从配置文件动态读取数据库密码等配置信息，避免明文写在代码中
- 自定义Filter：实现FilterEventAdapter，在连接获取、关闭、SQL执行前后等生命周期节点插入自定义逻辑

## 2.4 什么情况下会出现连接泄漏？如何排查？
连接泄漏是指从连接池获取的连接使用后未归还（close），导致池中连接耗尽，后续请求获取连接超时失败。常见原因：忘记在finally中关闭连接、异常处理不当（异常未进入close逻辑）、ORM框架自动管理但配置错误。

排查步骤：1）开启Druid的logAbandoned=true会输出泄露连接日志；2）启用logAbandonedDebug=true输出泄露连接的完整调用栈；3）查看日志中Connection.XXXX openTime找到创建时间和方法；4）通过responsibleTo配置标记连接持有者溯源；5）搜代码中未在finally中close的位置。

## 2.5 数据库连接池的核心参数有哪些？如何设置？
| 参数 | 说明 | 建议 |
|------|------|------|
| maxActive | 最大活跃连接数 | 根据数据库最大连接数和应用实例数反推，如DB max=200，4实例，则每实例约40-50 |
| minIdle | 最小空闲连接数 | 设置约maxActive的20-30%，保持就绪连接避免突发冷启动 |
| maxWait | 获取连接最大等待ms | 设置合理值（如3000）避免请求无限等待，超时快速失败 |
| validationQuery | 连接检测SQL | MySQL用SELECT 1；Oracle用SELECT 1 FROM DUAL |
| testWhileIdle | 空闲连接检测 | 建议true，防止获取到已被数据库端断开的连接 |
| timeBetweenEvictionRunsMillis | 检测周期 | 设置60秒左右，同时控制空闲回收和检测频率 |
| keepAlive | 心跳保活 | 建议true，防止连接被数据库wait_timeout断开 |
| removeAbandoned | 自动回收泄露连接 | 生产慎开启，可能误杀运行中的正经操作 |

一般不建议开启testOnBorrow（性能高开销），testOnReturn也不需要。

## 2.6 Spring事务中如何保证同一个事务使用同一个数据库连接？
Spring事务管理器（如DataSourceTransactionManager）通过TransactionSynchronizationManager使用ThreadLocal将数据库连接绑定到当前线程。事务开启时获取连接并绑定；事务内所有DAO方法调用获取连接时发现当前线程已绑定，直接返回同一个连接；事务提交/回滚时解绑并归还连接池。因此同一线程、同一事务中的多次数据库操作一定使用同一个连接。

## 2.7 多数据源如何配置？动态数据源切换怎么做？
配置多个数据源的方式：1）定义多个DataSource Bean，@ConfigurationProperties分别绑定不同的spring.datasource配置前缀；2）通过不同的@MapperScan或jdbcTemplate指定不同的DataSource；3）如果有主从之分，主数据源加@Primary。

动态数据源切换推荐使用dynamic-datasource-spring-boot-starter（如baomidou）：1）配置所有数据源，group分组；2）在Service方法上加@DS("db1")指定使用的数据源；3）支持方法级、类级、多层嵌套（就近原则）；4）支持读写分离（@DS("master")/@DS("slave")主从分离）。原理：内部维护DynamicRoutingDataSource持有多数据源Map，通过ThreadLocal存储当前线程的数据源key，获取连接时根据key路由。

## 2.8 分布式事务下如何管理多数据源连接？
在需要保证多个数据源一致性的分布式事务场景下，不能只依赖连接池和Spring本地事务，因为各自独立提交。需引入分布式事务框架（如Seata）：
1）Seata AT模式：通过全局事务协调各个数据源的事务。Seata代理DataSource为DataSourceProxy，自动管理各阶段的连接获取/提交/回滚。
2）XA强一致：使用JTA事务管理器（如Atomikos），调用XAResource接口实现两阶段提交（Prepare/Commit），连接由XAConnection管理。应用层和本地事务的编码方式一致，底层由事务管理器保证跨数据源一致性。

## 2.9 Tomcat JDBC Pool和Commons DBCP2对比？
二者都是较早的数据库连接池实现。Tomcat JDBC Pool比DBCP2性能更好，作为Tomcat内置连接池。DBCP2是Apache Commons的二代连接池，功能较完善但性能有瓶颈。HikariCP出来后二者都显得性能不足——HikariCP的吞吐量约为DBCP2的3倍、Tomcat JDBC的2倍。目前新项目基本选HikariCP或Druid。

## 2.10 如何防止数据库连接在空闲时被数据库端断开（MySQL wait_timeout）？
数据库端有wait_timeout参数（MySQL默认8小时），连接空闲超过该时间会被数据库主动断开。连接池可通过以下方式保持连接有效：1）testWhileIdle=true定时检测连接有效性（timeBetweenEvictionRunsMillis控制检测频率）；2）keepAlive=true对空闲连接发送心跳查询（validationQuery）；3）设置minEvictableIdleTime + keepAlive组合，对空闲时间过长的连接定期探活；4）确保timeBetweenEvictionRunsMillis + keepAlive的组合频率低于数据库wait_timeout。

## 2.11 Druid的WallFilter如何防御SQL注入？
WallFilter实现SQL防火墙，核心原理是SQL语法解析+黑白名单规则：1）将SQL解析为AST语法树，检查是否存在注入特征（如注释内的SELECT、UNION SELECT、堆叠查询;、特殊函数等）；2）根据配置的防火墙规则判断是否允许执行；3）支持黑名单规则（禁止特定SQL操作）和白名单规则（仅允许指定的SQL）；4）可记录所有被拦截的SQL便于审计。注意：WallFilter主要拦截注入攻击性质的SQL，不等于绝对安全，开发层仍需预编译（PreparedStatement）防注入。

## 2.12 如何配置和监控Druid的慢SQL？
配置方式：1）Druid数据源配置filter.stat，设置connectionProperties中的druid.stat.slowSqlMillis=2000（超过2秒记录为慢SQL）；2）配置druid.stat.logSlowSql=true输出到日志；3）配置Druid StatViewServlet + WebStatFilter开启内置监控页面（druid/index.html），可视化查看慢SQL列表、执行计划、耗时分布等统计。监控：内置页面显示SQL列表，按执行次数、总耗时、最大耗时排序，每条SQL可展开查看执行时间和参数。

## 2.13 Druid连接泄漏检测的removeAbandoned和logAbandoned有什么区别？
removeAbandoned=true：检测到泄露连接时自动关闭并回收该连接，清理泄露。看似省心但有风险——如果连接正在执行耗时较长的正常业务（如大报告批量INSERT），被误判为泄露并强制关闭会中断业务。logAbandoned=true：仅打印日志记录泄露信息，不回收连接，安全无害。生产环境推荐只用logAbandoned，泄露问题由日志驱动开发修复代码，不自动回收避免误杀正常长事务。
