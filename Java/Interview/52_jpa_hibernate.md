# 1. 知识点
- **JPA定义**：Java Persistence API，Java持久化规范接口（javax.persistence / jakarta.persistence），定义ORM标准和注解，2.0/2.1/2.2/3.0版本，不提供具体实现
- **Hibernate**：最流行的JPA实现，在JPA规范外还提供额外功能（HQL、Criteria、缓存策略等），Spring Boot默认集成
- **ORM思想**：对象关系映射，将Java对象与数据库表建立映射关系，通过操作对象来操作数据库，屏蔽SQL差异
- **EntityManager**：JPA核心接口，管理实体的生命周期（持久化操作、查询、事务等），类似Hibernate的Session
- **EntityManagerFactory**：EntityManager的工厂，重量级对象，应用级别单例，负责创建EntityManager
- **PersistenceUnit**：持久化单元，配置一组Entity类及数据源信息
- **@Entity注解**：标注Java类为JPA实体，类名默认映射数据库表名，需有无参构造方法（JPA规范要求）
- **@Table注解**：配置实体对应的表名（默认类名转换）、schema、catalog、索引定义（@Index）、唯一约束等
- **@Id**：标注主键字段，每个@Entity必须有一个
- **@GeneratedValue**：主键生成策略，配合strategy参数选择四种策略
- **IDENTITY策略**：自增主键，依赖数据库自增列（MySQL AUTO_INCREMENT），INSERT后立即获取ID（需再次查询）
- **SEQUENCE策略**：序列生成，依赖数据库SEQUENCE对象（Oracle/PostgreSQL），预分配序列号，性能好
- **TABLE策略**：模拟序列的表，用一张表维护主键值，所有数据库可跨，性能最差
- **AUTO策略**：JPA自动选择（Hibernate按Dialect自行选择，MySQL选IDENTITY，Oracle选SEQUENCE），JPA默认
- **@Column**：配置列名、长度、可空、唯一、精度等细节，不配置则使用字段名自动映射
- **@Enumerated**：枚举映射，EnumType.ORDINAL存序数（危险，枚举顺序变更会错乱），EnumType.STRING存字符串（推荐）
- **@Transient**：标记字段不持久化，排除在ORM映射之外
- **@Temporal**：日期类型精确度配置，TemporalType.DATE/TIME/TIMESTAMP
- **@OneToOne**：一对一关联，可任意侧维护外键
- **@OneToMany**：一对多关联，默认LAZY加载，多的一方存外键或通过中间表
- **@ManyToOne**：多对一关联，多的一方设置外键指向一的一方，默认EAGER加载
- **@ManyToMany**：多对多关联，通过中间表关联，两端都默认LAZY，中间表由JPA自动管理
- **mappedBy**：双向关联中被维护方设置，声明外键由对方维护，值为对方实体中关联本实体的字段名
- **外键维护方**：关联关系中拥有外键列的一方，对维护方设置关联会自动更新外键值；被维护方（mappedBy方）设置关联不生效
- **级联CascadeType**：ALL（所有操作级联）、PERSIST（级联保存）、MERGE（级联合并）、REMOVE（级联删除）、REFRESH（级联刷新）、DETACH（级联分离）
- **FetchType.LAZY**：懒加载，访问关联属性时才发SQL查询，返回的是Hibernate代理对象
- **FetchType.EAGER**：立即加载，查询主实体同时查关联实体，JOIN或额外SELECT，可能引发N+1问题
- **LazyInitializationException**：懒加载在Session关闭后（如Controller返回JSON序列化）访问未加载的关联属性时抛出，因为代理对象需要Session
- **Open Session in View**：Spring Boot默认开启，在一次HTTP请求期间保持Session打开，View层仍可懒加载，但可能导致数据库连接占用时间长；高并发建议关闭
- **N+1查询问题**：查询N条主实体后，遍历访问关联属性时每条额外发1条SQL，共N+1条；用@NamedEntityGraph/@EntityGraph或FETCH JOIN JPQL解决
- **@NamedEntityGraph/@EntityGraph**：定义和指定加载时的关联属性抓取策略，避免N+1问题
- **FETCH JOIN**：JPQL中LEFT JOIN FETCH在一条SQL中关联加载关联数据，达到和EAGER类似效果但按需使用
- **Hibernate一级缓存**：Session级别缓存，默认开启，同一Session内重复查询同一ID直接返回缓存对象，事务提交时脏检查和自动更新，事务结束缓存清空
- **Hibernate二级缓存**：SessionFactory级别，需手动配置（如Ehcache、Redis），跨Session共享缓存，只缓存实体（不缓存关联/集合查询结果），适用读多写少的数据
- **Hibernate查询缓存**：对查询语句（JPQL/Criteria）和参数缓存结果ID列表，结合二级缓存获取实体，需手动开启
- **持久化对象状态-瞬态Transient**：new出来还未persist的对象，无ID，与Session无关
- **持久化对象状态-持久化Persistent**：已persist/load的对象，有ID，关联Session，属性变更在flush时自动同步
- **持久化对象状态-游离Detached**：Session关闭后的对象，有ID但不关联Session，需merge重新关联
- **持久化对象状态-删除Removed**：调用了remove方法标记待删除，flush时生成DELETE语句
- **JPQL**：Java Persistence Query Language，面向对象查询（操作用实体属性和对象而非表和列），独立于SQL方言
- **HQL**：Hibernate Query Language，Hibernate对JPQL的增强版，功能更多
- **位置参数**：占位符?1，按位置传参
- **命名参数**：占位符:paramName，按名称传参，可读性好
- **New构造DTO**：JPQL中SELECT new com.xxx.dto.UserDTO(u.id, u.name) FROM User u，直接将查询结果映射为DTO对象
- **Spring Data JPA**：Spring对JPA的进一步封装，通过Repository接口自动生成CRUD实现，减少样板代码
- **JpaRepository**：Spring Data JPA核心接口，继承PagingAndSortingRepository再继承CrudRepository，提供CRUD、分页、排序方法
- **JpaSpecificationExecutor**：支持动态条件查询Specification接口，配合入参CriteriaBuilder构建动态SQL
- **PagingAndSortingRepository**：提供findAll(Sort)和findAll(Pageable)分页排序方法
- **方法命名查询**：Spring Data JPA根据方法名自动生成SQL，如findByNameAndAge(String name, Integer age)生成WHERE name=? AND age=?
- **方法命名关键词**：Between、Like、In、OrderBy、GreaterThan、LessThan、Before、After、IsNull、NotNull、Top/First取前N条
- **@Query注解**：自定义JPQL/HQL查询，value定义查询语句，nativeQuery=true表示原生SQL
- **@Modifying**：配合@Query执行UPDATE/DELETE操作，必须配合@Transactional使用
- **@Transactional**：声明式事务，应用于Service层方法；JPA中修改/删除必须开启事务
- **分页查询Pageable**：方法参数Pageable pageable，可传PageRequest.of(page, size, Sort)，返回Page对象含内容、总页数、总元素数
- **Specification动态查询**：实现toPredicate方法，使用CriteriaBuilder构建WHERE条件，可动态拼接多个Predicate
- **JpaSpecificationExecutor**：继承后获得findAll(Specification, Pageable)等方法
- **CriteriaBuilder多表关联**：Root.join获取关联实体路径，在关联实体属性上做条件查询
- **审计Auditing**：自动记录创建时间/修改时间/创建人/修改人，@EnableJpaAuditing开启
- **@CreatedDate/@LastModifiedDate**：自动填充创建/修改时间，配合@EnableJpaAuditing生效
- **@CreatedBy/@LastModifiedBy**：自动填充操作人，需配置AuditorAware实现获取当前用户逻辑
- **@EntityListeners(AuditingEntityListener.class)**：实体类添加审计监听器
- **批量操作优化-saveAll**：一次保存多条记录，但仍逐条INSERT/UPDATE，减少事务和一级缓存开销
- **分批flush清除一级缓存**：大量插入时每N条flush+clear一次，防止一级缓存膨胀OOM
- **jdbcTemplate批量**：极大数据量写直接使用Spring JDBC batchUpdate，完全不经过JPA持久化上下文
- **SQL日志打印**：spring.jpa.show-sql=true + spring.jpa.properties.hibernate.format_sql=true + logging.level.org.hibernate.SQL=DEBUG
- **慢查询监控**：通过数据源代理（如log4jdbc或Druid慢SQL过滤器）记录/监控慢查询
- **禁用Open Session In View**：spring.jpa.open-in-view=false，高并发应用推荐，避免连接长期占用
- **合理使用FetchType.LAZY**：多对一、一对多关联默认或显式设为LAZY，按需用@EntityGraph或FETCH JOIN加载

# 2. 面试题

## 2.1 JPA和Hibernate有什么区别？
JPA是Java持久化规范（接口定义），定义了一套ORM标准注解和API（如EntityManager、@Entity等），不提供具体实现。Hibernate是JPA最流行的实现之一，除了完全兼容JPA规范外，还提供额外功能如HQL、二级缓存管理、Criteria查询扩展等。Spring Data JPA又在JPA之上进一步封装，通过Repository接口自动生成CRUD实现。三者关系：Spring Data JPA → JPA规范 → Hibernate实现。

## 2.2 什么是ORM？它的核心思想是什么？
ORM（Object-Relational Mapping）对象关系映射，核心思想是将Java对象与数据库表建立映射关系，通过操作对象来间接操作数据库。数据中的表对应Java的实体类，列对应字段，记录对应对象实例。ORM屏蔽了底层数据库SQL差异，开发者可用面向对象思维操作数据，不用手写大部分SQL。

## 2.3 Hibernate中一级缓存和二级缓存的区别是什么？
| 维度 | 一级缓存 | 二级缓存 |
|------|---------|---------|
| 级别 | Session级别 | SessionFactory级别 |
| 开启 | 默认开启，不可关闭 | 需手动配置（如Ehcache/Redis） |
| 作用域 | 同一Session内 | 跨Session、跨应用 |
| 生命周期 | 随事务结束清空 | 手动管理或直到缓存过期 |
| 存储内容 | 实体对象 | 实体对象数据 |
| 使用场景 | 天然存在 | 读多写少的字典/配置/基础数据 |

一级缓存自动管理，二级缓存需评估收益成本后配置。

## 2.4 什么是N+1查询问题？如何解决？
查询N条主实体后，在遍历访问关联属性时，Hibernate会为每条主实体再发一条SQL查询关联数据，总共产生1+N条SQL。例如：查询10条Order后遍历打印每个Order的OrderItem，会额外产生10条SQL。

解决方案：
1. @EntityGraph/@NamedEntityGraph 指定关联加载策略
2. JPQL中使用JOIN FETCH（LEFT JOIN FETCH o.items）
3. 使用原生SQL或@Query一次性JOIN加载
4. 适当批量加载（@BatchSize）
5. 业务需要时可设置FetchType.EAGER（但全局EAGER可能带来更大问题）

## 2.5 LazyInitializationException是什么？为什么发生？怎么解决？
当使用FetchType.LAZY懒加载的关联属性在Session/EntityManager关闭后被访问时，由于代理对象需要数据库连接来加载数据但连接已关闭，抛出LazyInitializationException。常见场景：Service层返回Entity到Controller，Controller在JSON序列化时访问懒加载属性。

解决方案：1）关闭懒加载改用EAGER（不推荐全局）；2）使用@EntityGraph或JOIN FETCH预加载；3）返回DTO而非Entity（推荐，精准控制数据）；4）Spring Boot默认Open Session in View已解决（但高并发慎用）；5）在事务方法内准备好所有需要的数据。

## 2.6 JPA实体对象的四种状态分别是什么？
- Transient（瞬态）：new出来未调用persist，无ID，与Session无关联，类似临时对象
- Persistent（持久化）：已persist/merge/load，有ID，关联Session，属性变更自动在flush时同步到数据库（脏检查）
- Detached（游离）：Session关闭后，对象有ID但不关联Session，需merge重新关联后更新
- Removed（删除）：调用了remove方法标记删除，flush时生成DELETE语句

理解状态有助于避免在Detached状态进行关联操作导致异常。

## 2.7 JPA的四种主键生成策略分别是什么？适用场景？
- IDENTITY：依赖数据库自增列（MySQL AUTO_INCREMENT），简单但对批量插入性能差（JDBC批量优化失效，因每次INSERT后需select last_insert_id获取ID）
- SEQUENCE：依赖数据库序列，Hibernate可预分配序列号减少数据库访问（allocationSize），适合Oracle/PostgreSQL
- TABLE：用一张表模拟序列，数据维护成本高、性能差，所有数据库都可用但一般不用
- AUTO：由Hibernate依据Dialect自动选择（MySQL选IDENTITY，Oracle选SEQUENCE），最常用策略

## 2.8 @OneToMany和@ManyToOne有什么区别？mappedBy的作用是什么？
@OneToMany：一方关联多方，通常一方配置mappedBy表明该关联由多方维护；@ManyToOne：多对一关联，多的一方持有外键列，默认EAGER加载。

mappedBy用于双向关联中标识"被维护方"，值是对端实体中持有本实体引用的字段名。维护方（不设mappedBy的一方）操作关联属性会更新外键，被维护方操作关联属性不影响外键。示例：Department中有@OneToMany(mappedBy="department")，Employee中有@ManyToOne @JoinColumn(name="dept_id") private Department department，则外键由Employee维护。

## 2.9 JPA的级联操作CascadeType有哪些？使用时需要注意什么？
PERSIST（级联保存）、MERGE（级联合并）、REMOVE（级联删除，不同于orphanRemoval）、REFRESH（级联刷新更新）、DETACH（级联分离）、ALL（包含以上全部）。

注意事项：1）REMOVE危险，删除父实体时所有子实体一并删除，不小心使用时数据丢失风险大；2）谨慎使用ALL，按需选择细粒度级联；3）级联与orphanRemoval区别：CascadeType.REMOVE删除父时才删除子，orphanRemoval是移出集合就删除（孤儿删除）；4）双向关系需两端都设置级联，否则一方失效。

## 2.10 Spring Data JPA的方法命名查询支持哪些关键词？
支持的关键词包括：And/Or、Between(start, end)、LessThan/GreaterThan、Like、In、OrderBy、IsNull/IsNotNull、True/False、Before/After（日期比较）、Top/First（取前N条）、Distinct、IgnoreCase。方法名格式：findBy/getBy/queryBy/readBy + 属性名 + 条件关键词[+其他条件] + [OrderBy + 属性 + Asc/Desc]。例如：findByNameLikeAndAgeBetweenOrderByCreateTimeDesc(String name, Integer minAge, Integer maxAge)。

## 2.11 @Query注解什么时候使用？@Modifying的作用是什么？
@Query用于方法命名规则无法表达的复杂查询：JPQL复杂关联查询、需要NEW构造DTO的SELECT、原生SQL查询（nativeQuery=true）。@Modifying声明该@Query是UPDATE/DELETE操作（非SELECT），必须配合@Transactional使用，因为JPA要求修改操作必须在事务中执行。注意@Modifying后clearAutomatically=true可自动清除一级缓存避免脏数据。

## 2.12 Specification动态查询的使用场景和原理是什么？
适用于多条件组合且条件可选（高级搜索、报表查询）。原理：实现Specification<T>接口的toPredicate方法，接收Root（实体根路径）、CriteriaQuery、CriteriaBuilder三个参数，用CriteriaBuilder构建等于/不等于/like/范围等条件，返回Predicate。配合JpaSpecificationExecutor的findAll(Specification, Pageable)方法。多个条件可动态拼接cb.and()/cb.or()。

## 2.13 如何优化JPA/Hibernate的批量操作性能？
1. saveAll代替逐条save，减少事务开销。
2. 大批量插入时每N条执行flush()+clear()防止一级缓存膨胀OOM。
3. 配置spring.jpa.properties.hibernate.jdbc.batch_size=50开启JDBC批量。
4. 配置spring.jpa.properties.hibernate.order_inserts=true和order_updates=true对批量INSERT/UPDATE排序以使相邻记录在相同数据页。
5. 极大量数据直接用jdbcTemplate.batchUpdate绕过JPA上下冗余。
6. 批量查询使用IN而非循环单查。
7. 开启show-sql定位N+1问题。

## 2.14 什么是Open Session in View？应该开启还是关闭？
Spring Boot默认开启spring.jpa.open-in-view=true，在HTTP请求进入时打开Session，响应返回前Session保持open，使得Controller/View层仍然可以懒加载关联属性，避免LazyInitializationException。缺点：数据库连接从请求开始一直占用到请求结束（包括业务逻辑计算、序列化等），高并发下连接池容易耗尽。

建议：开发阶段可开启（方便调试）；高并发生产环境必须关闭，改为Service层返回DTO或在事务内通过@EntityGraph/FETCH JOIN预加载所需数据。关闭后可减少数据库连接占用时间50%以上。

## 2.15 如何实现JPA审计自动记录创建和更新时间、操作人？
1. 在配置类加@EnableJpaAuditing注解
2. 实体类加@EntityListeners(AuditingEntityListener.class)
3. 创建时间字段加@CreatedDate，修改时间字段加@LastModifiedDate
4. 创建人加@CreatedBy，修改人加@LastModifiedBy
5. 实现AuditorAware<String>接口，通过SecurityContextHolder或ThreadLocal获取当前用户
6. 将AuditorAware实现注册为Bean

时间通过@CreatedDate/@LastModifiedDate自动填充，操作人通过AuditorAware回调获取。
