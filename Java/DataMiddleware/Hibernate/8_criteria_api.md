# 1. Criteria API 概述

## 1.1 什么是 Criteria API

Criteria API 是 JPA 规范（JPA 2.0+）定义的一套**类型安全的编程式查询构建 API**，允许开发者完全通过 Java 代码来构建数据库查询，而无需编写任何 JPQL 或 SQL 字符串。

**核心特征：**

- **类型安全（Type-Safe）**：查询的每个环节都受 Java 类型系统约束，属性名、返回值类型在编译期即可检查，避免运行时的字符串拼写错误。
- **编程式构建**：通过方法调用链构建查询，IDE 的代码补全、重构工具可完整覆盖查询逻辑。
- **动态查询友好**：可根据运行时条件灵活组装查询条件，无需手动拼接 JPQL 字符串，是动态查询场景的首选方案。

**JPA Criteria API vs Hibernate 旧 Criteria API：**

| 特性 | JPA Criteria API（`jakarta.persistence.criteria`） | Hibernate 旧 API（`org.hibernate.Criteria`） |
|------|----------------------------------------------------|----------------------------------------------|
| 状态 | JPA 标准，持续更新 | Hibernate 6 起正式移除（已被废弃多年） |
| 类型安全 | 原生支持（泛型 + 元模型） | 字符串属性名，无编译期检查 |
| 子查询 | 支持 | 通过 DetachedCriteria 间接支持 |
| 元模型支持 | 支持 Static Metamodel（JPA 规范） | 不支持 |
| JOIN FETCH | 原生支持 | 需额外设置 FetchMode |
| 面向对象 | 完全面向实体和属性路径 | 面向表字段和投影 |

> **注意**：本文档全部基于 JPA Criteria API（`jakarta.persistence.criteria.*` 或 `javax.persistence.criteria.*`），不再讨论已废弃的 `org.hibernate.Criteria`。旧 API 的迁移参考请见第 10 章。

## 1.2 适用场景

Criteria API 最适用于以下场景：

1. **动态构建复杂查询条件**：搜索页面中，用户可能选择任意组合的筛选条件（如可选填的时间范围、状态、关键词等），使用 Criteria API 可以按条件是否存在来动态组装 Predicate，而无需拼接 JPQL 字符串。

2. **避免字符串拼接 JPQL**：拼接 JPQL 字符串容易出错（多一个空格、缺少 AND 等），且难以维护。Criteria API 将这些逻辑表达为 Java 代码，语义更清晰。

```java
// 字符串拼接 JPQL — 容易出错
String jpql = "SELECT u FROM User u WHERE 1=1";
if (name != null) jpql += " AND u.name LIKE '%" + name + "%'";  // SQL注入风险
if (ageMin != null) jpql += " AND u.age >= " + ageMin;
// → 可读性差、维护困难、不安全

// Criteria API — 类型安全、语义清晰
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<User> query = cb.createQuery(User.class);
Root<User> root = query.from(User.class);
List<Predicate> predicates = new ArrayList<>();
if (name != null) predicates.add(cb.like(root.get("name"), "%" + name + "%"));
if (ageMin != null) predicates.add(cb.ge(root.get("age"), ageMin));
query.where(predicates.toArray(new Predicate[0]));
```

3. **类型安全要求高的场景**：金融、医疗等领域中，查询的正确性至关重要，Criteria API 的编译期检查可减少低级错误。

4. **与 Spring Data JPA Specification 协作**：Spring Data JPA 的 `JpaSpecificationExecutor` 底层正是基于 Criteria API，掌握 Criteria API 是使用 Specification 的前提（详见第 11 章）。

## 1.3 核心接口概览

Criteria API 的核心接口均位于 `jakarta.persistence.criteria` 包下：

| 接口 | 作用 | 获取方式 |
|------|------|----------|
| `CriteriaBuilder` | 查询构建器工厂，创建所有 Criteria 对象 | `EntityManager.getCriteriaBuilder()` |
| `CriteriaQuery<T>` | 顶级查询定义，定义 SELECT、WHERE、GROUP BY、HAVING、ORDER BY | `CriteriaBuilder.createQuery(Class<T>)` |
| `Root<T>` | 查询根实体，代表 FROM 子句的实体，同时是 `Path` 和 `From` | `CriteriaQuery.from(Class<T>)` |
| `Predicate` | 过滤条件（WHERE 子句），可组合为 AND/OR 逻辑树 | 通过 `CriteriaBuilder` 的 `equal`、`like` 等方法创建 |
| `Join<Z,X>` | 关联连接（JOIN），表示实体间的关联路径 | `Root.join(String)` / `Root.join(String, JoinType)` |
| `Subquery<T>` | 子查询，可用于 WHERE 子句中的 EXISTS、IN、ALL/ANY/SOME | `CriteriaQuery.subquery(Class<T>)` |
| `Path<X>` | 属性路径，可层层导航获取嵌套属性 | 通过 `Root.get(String)` 或元模型获取 |
| `Expression<T>` | 表达式，是 `Path` 的父接口，聚合函数等返回 `Expression` | 通过 `CriteriaBuilder` 的函数方法创建 |
| `CompoundSelection<X>` | SELECT 多字段选择，`multiselect()` 的返回值 | `CriteriaQuery.multiselect(...)` |
| `Tuple` | 键值对形式的查询结果，可混合不同类型 | `CriteriaBuilder.createTupleQuery()` 时使用 |
| `ParameterExpression<T>` | 命名参数 / 位置参数，用于创建可复用的参数化查询 | `CriteriaBuilder.parameter(Class<T>)` |
| `Order` | 排序规则 | `CriteriaBuilder.asc(Expression)` / `desc(...)` |

# 2. 基本查询构建

## 2.1 获取 CriteriaBuilder

`CriteriaBuilder` 是构建一切 Criteria 查询对象的入口工厂，通过 JPA 的 `EntityManager` 或 Hibernate 的 `Session` 获取：

```java
// 方法一：通过 JPA EntityManager（标准方式，推荐）
EntityManager em = entityManagerFactory.createEntityManager();
CriteriaBuilder cb = em.getCriteriaBuilder();

// 方法二：通过 Hibernate Session（Hibernate 特有）
Session session = entityManager.unwrap(Session.class);
CriteriaBuilder cb = session.getCriteriaBuilder();

// Spring 环境下通过 @PersistenceContext 注入
@PersistenceContext
private EntityManager em;

public List<User> findByCondition(String name) {
    CriteriaBuilder cb = em.getCriteriaBuilder();
    // ...
}
```

## 2.2 创建 CriteriaQuery

根据查询结果类型的不同，`CriteriaBuilder` 提供了三种创建查询的方式：

```java
CriteriaBuilder cb = em.getCriteriaBuilder();

// 方式一：查询完整实体
CriteriaQuery<User> query1 = cb.createQuery(User.class);
// → SELECT u FROM User u

// 方式二：查询多属性（返回 Object[]）
CriteriaQuery<Object[]> query2 = cb.createQuery(Object[].class);
// → SELECT u.name, u.age FROM User u
// 结果：每行是一个 Object[]，按 multiselect 顺序排列

// 方式三：Tuple 查询（返回类型安全的命名元组）
CriteriaQuery<Tuple> query3 = cb.createTupleQuery();
// → SELECT u.name AS username, u.age AS userAge FROM User u
// 结果：每行是一个 Tuple，可通过别名或索引获取值
// 类型安全：通过 TupleElement 泛型实现编译期检查
```

## 2.3 Root

`Root<T>` 代表查询的实体根，对应于 JPQL 中 `FROM Entity e` 的 `e`。它是查询的起点，所有属性访问和关联连接都从 Root 开始。

```java
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<User> query = cb.createQuery(User.class);

// from() 声明查询根，返回 Root<User>
Root<User> root = query.from(User.class);

// Root 继承自 From，From 继承自 Path
// 因此 Root 可以：
//   1. 作为 Path：获取属性 → root.get("name")
//   2. 作为 From：执行连接 → root.join("orders")
//   3. 作为 Selection：选择整个实体 → query.select(root)

// 别名（Alias）：Root 自身相当于 JPQL 的别名变量
// Root<User> root 在 Criteria 中就是"查询的起点"，无需显式声明别名
// 如果有多个 Root（不常见），可通过泛型类型区分
```

## 2.4 执行查询

构建完成的 `CriteriaQuery` 通过 `EntityManager.createQuery()` 转换为 `TypedQuery` 后执行：

```java
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<User> criteriaQuery = cb.createQuery(User.class);
Root<User> root = criteriaQuery.from(User.class);
criteriaQuery.select(root);

// 将 CriteriaQuery 转换为 TypedQuery
TypedQuery<User> typedQuery = em.createQuery(criteriaQuery);

// 执行查询
List<User> users = typedQuery.getResultList();              // 返回列表（无结果返回空列表）
User singleUser = typedQuery.getSingleResult();             // 返回唯一结果（无结果/多条抛异常）
User optionalUser = typedQuery.getSingleResultOrNull();     // Hibernate 6.1+：无结果返回 null

// 流式处理（Hibernate 6+）
try (Stream<User> stream = typedQuery.getResultStream()) {
    stream.filter(u -> u.getAge() > 18)
          .forEach(System.out::println);
}

// 设置查询提示（Hibernate 扩展）
typedQuery.setHint("org.hibernate.fetchSize", 50);
typedQuery.setHint("org.hibernate.readOnly", true);
```

## 2.5 完整基本示例

以下示例展示了一个完整的查询流程，涵盖筛选、排序和分页：

```java
public class UserCriteriaQueryExample {

    @PersistenceContext
    private EntityManager em;

    /**
     * 综合查询示例：按名称模糊匹配 + 年龄范围过滤 + 排序 + 分页
     */
    public List<User> searchUsers(String nameKeyword, Integer minAge, Integer maxAge,
                                   int page, int pageSize) {
        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<User> query = cb.createQuery(User.class);
        Root<User> root = query.from(User.class);

        // 构建条件
        List<Predicate> predicates = new ArrayList<>();

        // 模糊查询：name LIKE '%keyword%'
        if (nameKeyword != null && !nameKeyword.isBlank()) {
            predicates.add(cb.like(root.get("name"), "%" + nameKeyword + "%"));
        }

        // 年龄范围：age >= minAge
        if (minAge != null) {
            predicates.add(cb.ge(root.get("age"), minAge));
        }

        // 年龄范围：age <= maxAge
        if (maxAge != null) {
            predicates.add(cb.le(root.get("age"), maxAge));
        }

        // IN 查询：查询指定 ID 列表中的用户
        // predicates.add(root.get("id").in(Arrays.asList(1L, 2L, 3L)));

        // NOT IN 查询
        // predicates.add(cb.not(root.get("id").in(Arrays.asList(4L, 5L))));

        // 组合条件
        query.where(predicates.toArray(new Predicate[0]));

        // 排序：先按创建时间降序，相同则按姓名升序
        query.orderBy(cb.desc(root.get("createTime")), cb.asc(root.get("name")));

        // 转换为 TypedQuery 并执行分页
        TypedQuery<User> typedQuery = em.createQuery(query);
        typedQuery.setFirstResult((page - 1) * pageSize);  // 起始位置
        typedQuery.setMaxResults(pageSize);                  // 每页数量

        return typedQuery.getResultList();
    }

    /**
     * 查询所有用户
     */
    public List<User> findAll() {
        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<User> query = cb.createQuery(User.class);
        Root<User> root = query.from(User.class);
        query.select(root);
        return em.createQuery(query).getResultList();
    }
}
```

# 3. Predicate 构建详解

## 3.1 比较谓词

`CriteriaBuilder` 提供了一系列方法来创建各种比较条件，这些方法返回 `Predicate` 对象：

```java
CriteriaBuilder cb = em.getCriteriaBuilder();
Root<User> root = query.from(User.class);

// ---- 等值与不等值 ----
Predicate p1 = cb.equal(root.get("name"), "John");               // name = 'John'
Predicate p2 = cb.notEqual(root.get("status"), "DELETED");       // status <> 'DELETED'

// ---- 数值比较 ----
Predicate p3 = cb.greaterThan(root.get("age"), 18);              // age > 18
Predicate p4 = cb.greaterThanOrEqualTo(root.get("age"), 18);     // age >= 18
Predicate p5 = cb.lessThan(root.get("age"), 60);                 // age < 60
Predicate p6 = cb.lessThanOrEqualTo(root.get("age"), 60);        // age <= 60

// ---- 范围查询 ----
Predicate p7 = cb.between(root.get("age"), 18, 65);              // age BETWEEN 18 AND 65
// between 包含边界值，等价于 >= min AND <= max

// ---- 模糊匹配 ----
Predicate p8 = cb.like(root.get("name"), "%John%");              // name LIKE '%John%'
Predicate p9 = cb.like(root.get("name"), "J_hn");               // name LIKE 'J_hn'（_ 匹配单字符）
Predicate p10 = cb.notLike(root.get("email"), "%@spam.com");    // email NOT LIKE '%@spam.com'

// 自定义转义字符
Predicate p11 = cb.like(root.get("description"), "%100%%")
                   .value("%").escape('\\');                     // ESCAPE '\'

// ---- NULL 判断 ----
Predicate p12 = cb.isNull(root.get("deletedAt"));               // deletedAt IS NULL
Predicate p13 = cb.isNotNull(root.get("email"));                // email IS NOT NULL

// ---- IN 与 NOT IN ----
Predicate p14 = root.get("status").in("ACTIVE", "PENDING");     // status IN ('ACTIVE', 'PENDING')
Predicate p15 = root.get("id").in(Arrays.asList(1L, 2L, 3L));   // id IN (1, 2, 3)
Predicate p16 = cb.not(root.get("id").in(4L, 5L, 6L));          // id NOT IN (4, 5, 6)

// ---- 空字符串比较（Hibernate 扩展） ----
// cb.equal(root.get("name"), "") 产生 name = ''
```

## 3.2 逻辑运算

`CriteriaBuilder` 提供了 AND、OR、NOT 逻辑组合方法，以及用于动态查询构建的辅助方法：

```java
CriteriaBuilder cb = em.getCriteriaBuilder();

// ---- AND：所有条件同时满足 ----
Predicate andPredicate = cb.and(
    cb.equal(root.get("status"), "ACTIVE"),
    cb.greaterThan(root.get("age"), 18),
    cb.isNotNull(root.get("email"))
);
// → WHERE status = 'ACTIVE' AND age > 18 AND email IS NOT NULL

// ---- OR：任一条件满足 ----
Predicate orPredicate = cb.or(
    cb.equal(root.get("role"), "ADMIN"),
    cb.equal(root.get("role"), "MODERATOR")
);
// → WHERE role = 'ADMIN' OR role = 'MODERATOR'

// ---- NOT：条件取反 ----
Predicate notPredicate = cb.not(cb.equal(root.get("status"), "DELETED"));
// → WHERE NOT (status = 'DELETED')

// ---- 嵌套逻辑 (A AND B) OR (C AND D) ----
Predicate complex = cb.or(
    cb.and(cb.equal(root.get("role"), "ADMIN"), cb.greaterThan(root.get("age"), 25)),
    cb.and(cb.equal(root.get("role"), "USER"), cb.greaterThan(root.get("age"), 30))
);

// ---- conjunction()：恒真谓词（1=1），动态查询的完美起点 ----
Predicate conjunction = cb.conjunction();  // 恒真，始终返回 true
// 用途：作为动态 AND 拼接的初始值
List<Predicate> predicates = new ArrayList<>();
Predicate finalPredicate = cb.conjunction();
if (name != null) finalPredicate = cb.and(finalPredicate, cb.like(root.get("name"), "%" + name + "%"));
if (age != null) finalPredicate = cb.and(finalPredicate, cb.ge(root.get("age"), age));
query.where(finalPredicate);
// 如果所有条件都为空，等价于 WHERE 1=1，查询所有记录

// ---- disjunction()：恒假谓词（1=0） ----
Predicate disjunction = cb.disjunction();  // 恒假，始终返回 false
// 用途：作为动态 OR 拼接的初始值（初始状态：无任何匹配条件）
if (cond1) disjunction = cb.or(disjunction, pred1);
if (cond2) disjunction = cb.or(disjunction, pred2);
// 如果所有条件都为空，查询不返回任何行

// ---- 动态查询构建最佳模式 ----
public List<User> dynamicQuery(String name, Integer minAge, String email) {
    CriteriaBuilder cb = em.getCriteriaBuilder();
    CriteriaQuery<User> query = cb.createQuery(User.class);
    Root<User> root = query.from(User.class);

    Predicate predicate = cb.conjunction();  // 起点：恒真

    if (name != null && !name.isBlank()) {
        predicate = cb.and(predicate, cb.like(root.get("name"), "%" + name + "%"));
    }
    if (minAge != null) {
        predicate = cb.and(predicate, cb.greaterThanOrEqualTo(root.get("age"), minAge));
    }
    if (email != null && !email.isBlank()) {
        predicate = cb.and(predicate, cb.equal(root.get("email"), email));
    }

    query.where(predicate);
    query.orderBy(cb.desc(root.get("id")));

    return em.createQuery(query).getResultList();
}
```

## 3.3 集合谓词

针对 `@OneToMany`、`@ManyToMany` 等集合关联字段的谓词：

```java
// 假设 User 实体中有：
// @OneToMany(mappedBy = "user") private List<Order> orders;
// @ElementCollection private List<String> tags;

CriteriaBuilder cb = em.getCriteriaBuilder();
Root<User> root = query.from(User.class);

// ---- IS EMPTY：集合为空 ----
Predicate hasNoOrders = cb.isEmpty(root.get("orders"));
// → WHERE NOT EXISTS (SELECT 1 FROM orders WHERE user_id = user.id)
// 等价于：查询没有任何订单的用户

// ---- IS NOT EMPTY：集合不为空 ----
Predicate hasOrders = cb.isNotEmpty(root.get("orders"));
// → WHERE EXISTS (SELECT 1 FROM orders WHERE user_id = user.id)

// ---- MEMBER OF：元素是否属于集合 ----
Expression<String> tagExpr = cb.literal("VIP");
Predicate isVip = cb.isMember(tagExpr, root.get("tags"));
// → WHERE 'VIP' MEMBER OF tags

Predicate isNotVip = cb.isNotMember(tagExpr, root.get("tags"));
// → WHERE 'VIP' NOT MEMBER OF tags

// ---- SIZE：获取集合元素数量（可用于 WHERE/ORDER BY） ----
Expression<Integer> orderCount = cb.size(root.get("orders"));
Predicate moreThanThree = cb.greaterThan(orderCount, 3);
// → WHERE (SELECT COUNT(*) FROM orders WHERE user_id = user.id) > 3

// 按订单数量排序
query.orderBy(cb.desc(cb.size(root.get("orders"))));  // 订单最多的用户排在前面
```

## 3.4 条件表达式

`CriteriaBuilder` 支持 SQL 风格的 CASE WHEN / COALESCE / NULLIF 等条件表达式：

```java
CriteriaBuilder cb = em.getCriteriaBuilder();
Root<User> root = query.from(User.class);

// ---- SELECT CASE WHEN ... THEN ... ELSE ... END ----
// 示例：将年龄转为年龄段
Expression<String> ageCategory = cb.<String>selectCase()
    .when(cb.lessThan(root.get("age"), 18), "未成年")
    .when(cb.between(root.get("age"), 18, 35), "青年")
    .when(cb.between(root.get("age"), 36, 60), "中年")
    .otherwise("老年");

query.select(ageCategory);
// → SELECT CASE WHEN age < 18 THEN '未成年'
//               WHEN age BETWEEN 18 AND 35 THEN '青年'
//               WHEN age BETWEEN 36 AND 60 THEN '中年'
//               ELSE '老年' END FROM user

// CASE WHEN 也可用于 WHERE 子句的复杂条件
Predicate casePredicate = cb.equal(
    cb.selectCase()
      .when(cb.isNull(root.get("nickname")), root.get("name"))
      .otherwise(root.get("nickname")),
    "John"
);

// ---- COALESCE：返回第一个非 NULL 的值 ----
Expression<String> displayName = cb.coalesce(root.get("nickname"), root.get("name"));
// → COALESCE(nickname, name)
// 用于 ORDER BY 更安全：cb.coalesce(root.get("sortOrder"), 0)

// ---- NULLIF：两值相等则返回 NULL ----
Expression<Integer> nullIfZero = cb.nullif(root.get("loginCount"), 0);
// → NULLIF(loginCount, 0)  登录次数为 0 返回 NULL
```

# 4. 关联查询

## 4.1 Join 类型

通过 `Root.join()` 方法执行关联连接，返回 `Join` 对象后可继续导航属性：

```java
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Object[]> query = cb.createQuery(Object[].class);
Root<User> root = query.from(User.class);

// ---- INNER JOIN（默认） ----
Join<User, Address> address = root.join("address");
// → INNER JOIN address a ON u.address_id = a.id

// ---- LEFT JOIN（LEFT OUTER JOIN） ----
Join<User, Address> leftAddress = root.join("address", JoinType.LEFT);
// → LEFT JOIN address a ON u.address_id = a.id

// ---- RIGHT JOIN（RIGHT OUTER JOIN） ----
Join<User, Address> rightAddress = root.join("address", JoinType.RIGHT);
// → RIGHT JOIN address a ON u.address_id = a.id

// ---- Join 的 Path 能力：Join 对象可作为 Path 访问关联实体的属性 ----
Predicate cityPredicate = cb.equal(address.get("city"), "北京");
query.where(cityPredicate);

query.multiselect(root.get("name"), address.get("city"));

// ---- 链式 Join：多层关联 ----
// User → Department → Company
Join<User, Department> dept = root.join("department");           // User JOIN Department
Join<Department, Company> company = dept.join("company");        // Department JOIN Company
Predicate companyName = cb.equal(company.get("name"), "阿里巴巴");

// 链式简写形式（直接通过属性路径，返回的 Join 类型需要强转）
// Join<User, Company> company2 = root.join("department").join("company");

// ---- 多个 Join 并行 ----
Join<User, Address> addr = root.join("address", JoinType.LEFT);
Join<User, Profile> profile = root.join("profile", JoinType.LEFT);
query.multiselect(root.get("name"), addr.get("city"), profile.get("avatar"));
```

## 4.2 Fetch

`Fetch` 与 `Join` 的区别在于：`Fetch` 会将关联实体的数据也加载到当前查询结果中（即 SQL 的 JOIN FETCH），用于解决 N+1 查询问题。而 `Join` 仅用于过滤条件，不影响 SQL 的 SELECT 列。

```java
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<User> query = cb.createQuery(User.class);
Root<User> root = query.from(User.class);

// ---- INNER JOIN FETCH：立即加载关联实体（解决 N+1 查询） ----
root.fetch("orders");
// → SELECT u.*, o.* FROM user u
//   INNER JOIN orders o ON u.id = o.user_id

// ---- LEFT JOIN FETCH：即使关联为空也返回实体 ----
root.fetch("orders", JoinType.LEFT);
// → SELECT u.*, o.* FROM user u
//   LEFT JOIN orders o ON u.id = o.user_id

// ---- 同时使用多个 fetch ----
root.fetch("orders", JoinType.LEFT);
root.fetch("address", JoinType.LEFT);
// 注意：同时 Fetch 多个集合（如 List）会导致笛卡尔积问题

// ---- Fetch 链式调用 ----
Fetch<User, Department> deptFetch = root.fetch("department");
deptFetch.fetch("company");  // Fetch Department 后再 Fetch Company

// ---- 同时需要 Join（过滤）和 Fetch（加载） ----
// 如果既要用关联做过滤条件，又想立即加载，需要同时定义 Join 和 Fetch
Join<User, Order> orderJoin = root.join("orders", JoinType.LEFT);
root.fetch("orders", JoinType.LEFT);
// 注意：Hibernate 6+ 中，同一关联不能同时 join 和 fetch，
//       可考虑仅在 join 上过滤，通过 EntityGraph 控制加载
```

## 4.3 ON 条件

使用 `Join.on()` 方法可以为 JOIN 添加额外的连接条件（用法类似 SQL 的 `JOIN ... ON condition1 AND condition2`）：

```java
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Object[]> query = cb.createQuery(Object[].class);
Root<User> root = query.from(User.class);

// ---- LEFT JOIN with ON：只关联符合条件的记录 ----
Join<User, Order> activeOrders = root.join("orders", JoinType.LEFT);
activeOrders.on(cb.equal(activeOrders.get("status"), "ACTIVE"));
// → LEFT JOIN orders o ON u.id = o.user_id AND o.status = 'ACTIVE'

// ---- 多条件 ON ----
Join<User, Order> recentOrders = root.join("orders", JoinType.LEFT);
recentOrders.on(
    cb.and(
        cb.equal(recentOrders.get("status"), "PAID"),
        cb.greaterThan(recentOrders.get("amount"), new BigDecimal("100.00"))
    )
);
// → LEFT JOIN orders o ON u.id = o.user_id AND o.status = 'PAID' AND o.amount > 100

// ---- ON 与 WHERE 的区别 ----
// ON 条件：影响 JOIN 的匹配行为，LEFT JOIN 时，不满足 ON 条件的右表列为 NULL，但左表行保留
// WHERE 条件：过滤最终结果集，LEFT JOIN 后过滤可能将 LEFT JOIN 变为 INNER JOIN 的效果

// 示例：查询所有用户及他们已支付的订单
// 正确写法（ON）：
Join<User, Order> orders = root.join("orders", JoinType.LEFT);
orders.on(cb.equal(orders.get("status"), "PAID"));
// → 用户即使没有已支付订单，用户行也会出现（订单列为 NULL）

// 错误写法（WHERE）：
// query.where(cb.equal(orders.get("status"), "PAID"));
// → 没有已支付订单的用户会被过滤掉，等同于 INNER JOIN
```

# 5. SELECT 子句

## 5.1 单属性查询

当只需要查询实体的某个属性（而非整个实体）时，将 `CriteriaQuery` 的泛型指定为属性类型，并使用 `query.select()`：

```java
CriteriaBuilder cb = em.getCriteriaBuilder();

// 查询所有用户名
CriteriaQuery<String> query = cb.createQuery(String.class);
Root<User> root = query.from(User.class);
query.select(root.get("name"));
// → SELECT u.name FROM User u

List<String> names = em.createQuery(query).getResultList();

// 去重
query.select(root.get("name")).distinct(true);
// → SELECT DISTINCT u.name FROM User u
```

## 5.2 多属性查询

查询多个属性时，返回类型为 `Object[]`，使用 `multiselect()` 或 `cb.array()`：

```java
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Object[]> query = cb.createQuery(Object[].class);
Root<User> root = query.from(User.class);

// ---- 方式一：multiselect() ----
query.multiselect(root.get("name"), root.get("age"), root.get("email"));
// → SELECT u.name, u.age, u.email FROM User u

// 也可按顺序逐个 select：
// query.select(cb.array(root.get("name"), root.get("age"), root.get("email")));

List<Object[]> results = em.createQuery(query).getResultList();
for (Object[] row : results) {
    String name = (String) row[0];
    Integer age = (Integer) row[1];
    String email = (String) row[2];
    System.out.printf("Name: %s, Age: %d, Email: %s%n", name, age, email);
}

// ---- 结合 Join 的多属性查询 ----
Join<User, Department> dept = root.join("department", JoinType.LEFT);
query.multiselect(root.get("name"), dept.get("name"));
// → SELECT u.name, d.name FROM User u LEFT JOIN u.department d
```

## 5.3 构造器表达式

使用 `cb.construct()` 可以将查询结果直接映射为 DTO，避免手动从 `Object[]` 取值：

```java
// 定义 DTO 类
public class UserDTO {
    private String name;
    private Integer age;

    // 构造方法参数顺序必须与 cb.construct() 的参数顺序一致
    public UserDTO(String name, Integer age) {
        this.name = name;
        this.age = age;
    }
    // getters...
}

// 查询
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<UserDTO> query = cb.createQuery(UserDTO.class);
Root<User> root = query.from(User.class);

// construct() 调用 DTO 的构造方法
query.select(cb.construct(UserDTO.class, root.get("name"), root.get("age")));
// → SELECT u.name, u.age FROM User u

List<UserDTO> dtos = em.createQuery(query).getResultList();

// 结合聚合函数
query.select(cb.construct(
    UserStatsDTO.class,
    cb.count(root),
    cb.avg(root.get("age"))
));
// 注意事项：
// 1. DTO 必须有匹配参数类型和顺序的构造方法
// 2. DTO 必须是全限定类名或已导入
// 3. 构造方法参数的类型必须与 Expression 类型匹配
```

## 5.4 Tuple 查询

`Tuple` 提供了类型安全的多字段查询方式，支持按别名和索引取值：

```java
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Tuple> query = cb.createTupleQuery();
Root<User> root = query.from(User.class);

// ---- multiselect 带别名 ----
query.multiselect(
    root.get("name").alias("username"),       // 字段别名
    root.get("age").alias("userAge"),         // 字段别名
    root.get("email").alias("userEmail")
);
// → SELECT u.name AS username, u.age AS userAge, u.email AS userEmail FROM User u

// 也可使用 TupleElement 创建别名
// query.multiselect(
//     cb.tuple(root.get("name"), root.get("age"))
// );

List<Tuple> tuples = em.createQuery(query).getResultList();
for (Tuple tuple : tuples) {
    // 按别名获取（推荐：语义清晰）
    String name = tuple.get("username", String.class);
    Integer age = tuple.get("userAge", Integer.class);

    // 按索引获取（索引从 0 开始）
    String email = tuple.get(2, String.class);

    // 按 JPA 表达式对象获取
    // String name = tuple.get(root.get("name"));

    System.out.printf("Name: %s, Age: %d, Email: %s%n", name, age, email);
}

// 注意：类型安全体现在 tuple.get("alias", Type.class) 明确指定了预期类型
```

## 5.5 聚合查询

```java
CriteriaBuilder cb = em.getCriteriaBuilder();
Root<Order> root;

// ====== 聚合函数 ======

// COUNT：计数
CriteriaQuery<Long> countQuery = cb.createQuery(Long.class);
root = countQuery.from(Order.class);
countQuery.select(cb.count(root));                           // COUNT(*)
// countQuery.select(cb.countDistinct(root.get("userId")));  // COUNT(DISTINCT user_id)

Long totalOrders = em.createQuery(countQuery).getSingleResult();

// SUM：求和
CriteriaQuery<BigDecimal> sumQuery = cb.createQuery(BigDecimal.class);
root = sumQuery.from(Order.class);
sumQuery.select(cb.sum(root.get("amount")));         // SUM(amount)
// sumQuery.select(cb.sumAsLong(root.get("quantity")));
// sumQuery.select(cb.sumAsDouble(root.get("amount")));

BigDecimal totalAmount = em.createQuery(sumQuery).getSingleResult();

// AVG：平均值
CriteriaQuery<Double> avgQuery = cb.createQuery(Double.class);
root = avgQuery.from(Order.class);
avgQuery.select(cb.avg(root.get("amount")));                  // AVG(amount)

// MAX / MIN：最大值 / 最小值
CriteriaQuery<BigDecimal> maxQuery = cb.createQuery(BigDecimal.class);
root = maxQuery.from(Order.class);
maxQuery.select(cb.max(root.get("amount")));                // MAX(amount)
// maxQuery.select(cb.min(root.get("amount")));             // MIN(amount)

// GREATEST / LEAST：多值对比（行内比较）
// 查询多个数值列中的最大值
query.select(cb.greatest(root.get("discount1"), root.get("discount2"), root.get("discount3")));
// → GREATEST(discount1, discount2, discount3)

// ====== GROUP BY 与 HAVING ======

CriteriaQuery<Object[]> groupQuery = cb.createQuery(Object[].class);
root = groupQuery.from(Order.class);

// GROUP BY：按用户 ID 分组
groupQuery.groupBy(root.get("userId"));

// 分组查询每用户的订单总额和订单数
groupQuery.multiselect(
    root.get("userId"),
    cb.sum(root.get("amount")),
    cb.count(root)
);

// HAVING：过滤分组结果（只保留订单总额 > 1000 的用户）
groupQuery.having(
    cb.greaterThan(cb.sum(root.get("amount")), new BigDecimal("1000.00"))
);

List<Object[]> groupResults = em.createQuery(groupQuery).getResultList();

// ====== 完整聚合查询示例 ======
/**
 * 统计各状态订单的数量和总金额，只统计数量 > 5 的分组，
 * 按总金额降序排列
 */
public List<OrderStatsDTO> orderStatsByStatus() {
    CriteriaBuilder cb = em.getCriteriaBuilder();
    CriteriaQuery<OrderStatsDTO> query = cb.createQuery(OrderStatsDTO.class);
    Root<Order> root = query.from(Order.class);

    query.multiselect(
        root.get("status"),
        cb.count(root),
        cb.sum(root.get("amount"))
    );
    query.groupBy(root.get("status"));
    query.having(cb.greaterThan(cb.count(root), 5L));
    query.orderBy(cb.desc(cb.sum(root.get("amount"))));

    return em.createQuery(query).getResultList();
}
// 对应 JPQL：
// SELECT o.status, COUNT(o), SUM(o.amount)
// FROM Order o
// GROUP BY o.status
// HAVING COUNT(o) > 5
// ORDER BY SUM(o.amount) DESC
```

# 6. 排序

排序通过 `CriteriaBuilder.asc()` 和 `CriteriaBuilder.desc()` 创建 `Order` 对象，再由 `CriteriaQuery.orderBy()` 应用：

```java
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<User> query = cb.createQuery(User.class);
Root<User> root = query.from(User.class);

// ---- 单字段升序 ----
query.orderBy(cb.asc(root.get("age")));               // ORDER BY age ASC

// ---- 单字段降序 ----
query.orderBy(cb.desc(root.get("createTime")));       // ORDER BY create_time DESC

// ---- 多字段排序（先按 createTime 降序，再按 name 升序） ----
query.orderBy(
    cb.desc(root.get("createTime")),
    cb.asc(root.get("name"))
);
// → ORDER BY create_time DESC, name ASC

// ---- 通过 Join 关联字段排序 ----
Join<User, Department> dept = root.join("department", JoinType.LEFT);
query.orderBy(cb.asc(dept.get("name")));              // ORDER BY d.name ASC

// ---- 通过表达式排序 ----
query.orderBy(cb.desc(cb.size(root.get("orders"))));  // ORDER BY (订单数) DESC

// ---- NULL 排序处理 ----
// Hibernate 扩展：控制 NULL 值排序位置
// 默认行为：ASC 中 NULL 排在最前，DESC 中 NULL 排在最后
// 可通过 Hibernate 的 Order 扩展控制：
// cb.asc(root.get("name")).nullsFirst()
// cb.desc(root.get("name")).nullsLast()

// ---- 动态排序示例 ----
public List<User> getUsersByOrder(String sortBy, boolean ascending) {
    CriteriaBuilder cb = em.getCriteriaBuilder();
    CriteriaQuery<User> query = cb.createQuery(User.class);
    Root<User> root = query.from(User.class);

    if (ascending) {
        query.orderBy(cb.asc(root.get(sortBy)));
    } else {
        query.orderBy(cb.desc(root.get(sortBy)));
    }

    return em.createQuery(query).getResultList();
}
```

# 7. 子查询

## 7.1 Subquery 创建

子查询通过 `CriteriaQuery.subquery()` 方法创建，返回 `Subquery<T>` 对象：

```java
CriteriaBuilder cb = em.getCriteriaBuilder();

// 创建主查询
CriteriaQuery<User> mainQuery = cb.createQuery(User.class);
Root<User> root = mainQuery.from(User.class);

// 创建子查询（类型由子查询 SELECT 的返回值决定）
Subquery<Long> subquery = mainQuery.subquery(Long.class);
// 子查询也可指定其他类型：Integer.class, String.class, Double.class 等

Root<Order> subRoot = subquery.from(Order.class);
subquery.select(cb.count(subRoot));
// → (SELECT COUNT(o) FROM Order o WHERE ...)
```

## 7.2 关联子查询

关联子查询（Correlated Subquery）中，子查询需要引用外部主查询的 Root，通过 `subquery.correlate()` 实现：

```java
CriteriaBuilder cb = em.getCriteriaBuilder();

// 查询有订单的用户
CriteriaQuery<User> mainQuery = cb.createQuery(User.class);
Root<User> root = mainQuery.from(User.class);

Subquery<Long> subquery = mainQuery.subquery(Long.class);
Root<Order> subRoot = subquery.from(Order.class);

// correlate：将外部查询的 Root 传入子查询，建立关联
subquery.correlate(root);

// 子查询中引用外部 Root
subquery.select(cb.count(subRoot))
        .where(cb.equal(subRoot.get("user"), root));  // 关联条件
// → (SELECT COUNT(o) FROM Order o WHERE o.user = u)

mainQuery.where(cb.greaterThan(subquery, 0L));
// → WHERE (SELECT COUNT(o) FROM Order o WHERE o.user = u) > 0
```

## 7.3 子查询谓词

```java
CriteriaBuilder cb = em.getCriteriaBuilder();

// ====== EXISTS ======
CriteriaQuery<User> query = cb.createQuery(User.class);
Root<User> root = query.from(User.class);

Subquery<Order> existsSub = query.subquery(Order.class);
Root<Order> orderRoot = existsSub.from(Order.class);
existsSub.select(orderRoot)
         .where(cb.equal(orderRoot.get("user"), root));
// 关联子查询

query.where(cb.exists(existsSub));
// → WHERE EXISTS (SELECT 1 FROM Order o WHERE o.user = u)

// ====== NOT EXISTS ======
query.where(cb.not(cb.exists(existsSub)));
// → WHERE NOT EXISTS (SELECT 1 FROM Order o WHERE o.user = u)

// ====== IN 子查询 ======
Subquery<String> inSub = query.subquery(String.class);
Root<Role> roleRoot = inSub.from(Role.class);
inSub.select(roleRoot.get("name"))
     .where(cb.equal(roleRoot.get("type"), "ADMIN"));

query.where(root.get("role").in(inSub));
// → WHERE u.role IN (SELECT r.name FROM Role r WHERE r.type = 'ADMIN')

// ====== ALL / ANY / SOME ======
Subquery<BigDecimal> priceSub = query.subquery(BigDecimal.class);
Root<Product> productRoot = priceSub.from(Product.class);
priceSub.select(cb.max(productRoot.get("price")));

// ALL：大于子查询返回的每一个值
query.where(cb.greaterThan(root.get("amount"), cb.all(priceSub)));

// ANY / SOME（二者等价）：大于子查询返回的某个值
query.where(cb.greaterThan(root.get("amount"), cb.any(priceSub)));
// query.where(cb.greaterThan(root.get("amount"), cb.some(priceSub)));
```

## 7.4 完整示例

```java
/**
 * 查询有未支付订单的用户
 */
public List<User> findUsersWithUnpaidOrders() {
    CriteriaBuilder cb = em.getCriteriaBuilder();
    CriteriaQuery<User> query = cb.createQuery(User.class);
    Root<User> root = query.from(User.class);

    Subquery<Integer> subquery = query.subquery(Integer.class);
    Root<Order> subRoot = subquery.from(Order.class);

    subquery.select(cb.literal(1))
            .where(
                cb.and(
                    cb.equal(subRoot.get("user"), root),       // 关联条件
                    cb.equal(subRoot.get("status"), "UNPAID")  // 子查询条件
                )
            );

    query.where(cb.exists(subquery));
    return em.createQuery(query).getResultList();
}

/**
 * 查询订单金额大于所有已完成订单平均金额的订单
 */
public List<Order> findOrdersAboveAverage() {
    CriteriaBuilder cb = em.getCriteriaBuilder();
    CriteriaQuery<Order> query = cb.createQuery(Order.class);
    Root<Order> root = query.from(Order.class);

    // 子查询：计算所有已完成订单的平均金额
    Subquery<Double> avgSub = query.subquery(Double.class);
    Root<Order> avgRoot = avgSub.from(Order.class);
    avgSub.select(cb.avg(avgRoot.get("amount")))
          .where(cb.equal(avgRoot.get("status"), "COMPLETED"));

    // 主查询条件
    query.where(cb.greaterThan(root.get("amount"), avgSub));
    query.orderBy(cb.desc(root.get("amount")));

    return em.createQuery(query).getResultList();
}

/**
 * 查询角色属于 "ADMIN" 或 "SUPER_ADMIN" 的用户
 */
public List<User> findAdminUsers() {
    CriteriaBuilder cb = em.getCriteriaBuilder();
    CriteriaQuery<User> query = cb.createQuery(User.class);
    Root<User> root = query.from(User.class);

    Subquery<Long> roleSub = query.subquery(Long.class);
    Root<Role> roleRoot = roleSub.from(Role.class);
    roleSub.select(roleRoot.get("id"))
           .where(roleRoot.get("name").in("ADMIN", "SUPER_ADMIN"));

    query.where(root.get("roleId").in(roleSub));
    return em.createQuery(query).getResultList();
}
```

# 8. 函数与表达式

## 8.1 字符串函数

```java
CriteriaBuilder cb = em.getCriteriaBuilder();
Root<User> root = query.from(User.class);

// ---- CONCAT：字符串拼接 ----
Expression<String> fullName = cb.concat(
    cb.concat(root.get("firstName"), " "),
    root.get("lastName")
);
// → CONCAT(CONCAT(firstName, ' '), lastName)

// ---- SUBSTRING：子串截取 ----
Expression<String> shortName = cb.substring(root.get("name"), 1, 5);
// → SUBSTRING(name, 1, 5)

// ---- TRIM：去空格 ----
Expression<String> trimmed = cb.trim(root.get("name"));
// → TRIM(name)

// ---- LOWER / UPPER：大小写转换 ----
Expression<String> lowerName = cb.lower(root.get("name"));
Predicate caseInsensitive = cb.equal(cb.lower(root.get("name")), "john");
// → LOWER(name) = 'john'

// ---- LENGTH：字符串长度 ----
Expression<Integer> nameLength = cb.length(root.get("name"));
// → LENGTH(name)

// ---- LOCATE：子串位置（1-based） ----
Expression<Integer> position = cb.locate(root.get("name"), "John");
// → LOCATE('John', name)
// 带起始位置：cb.locate(root.get("name"), "John", 5)
```

## 8.2 数学函数

```java
CriteriaBuilder cb = em.getCriteriaBuilder();
Root<Product> root = query.from(Product.class);

// ---- ABS：绝对值 ----
Expression<Integer> absDiff = cb.abs(cb.diff(root.get("stock"), root.get("sold")));
// → ABS(stock - sold)

// ---- SQRT：平方根 ----
Expression<Double> sqrtValue = cb.sqrt(root.get("value"));
// → SQRT(value)

// ---- MOD：取模 ----
Expression<Integer> mod = cb.mod(root.get("totalCount"), 10);
// → MOD(totalCount, 10)

// ---- SUM / DIFF / PROD / QUOT / NEG ----
Expression<Integer> sum = cb.sum(root.get("a"), root.get("b"));    // a + b
Expression<Integer> diff = cb.diff(root.get("a"), root.get("b"));  // a - b
Expression<Integer> prod = cb.prod(root.get("a"), root.get("b"));  // a * b
Expression<Number> quot = cb.quot(root.get("a"), root.get("b"));   // a / b
Expression<Integer> neg = cb.neg(root.get("count"));               // -count
```

## 8.3 日期函数

```java
CriteriaBuilder cb = em.getCriteriaBuilder();

// ---- 当前日期/时间 ----
Expression<java.sql.Date> today = cb.currentDate();        // CURRENT_DATE
Expression<java.sql.Time> now = cb.currentTime();          // CURRENT_TIME
Expression<java.sql.Timestamp> timestamp = cb.currentTimestamp();  // CURRENT_TIMESTAMP

// ---- 使用示例：查询创建时间早于当前日期的记录 ----
Root<User> root = query.from(User.class);
query.where(cb.lessThan(root.get("createDate"), cb.currentDate()));

// 注意：Hibernate Criteria API 未直接提供 YEAR/MONTH/DAY 等提取函数，
// 需要通过 cb.function() 调用数据库原生函数（见 8.4 节）
```

## 8.4 自定义函数调用

当 JPA Criteria API 内置函数不足以满足需求时，可通过 `cb.function()` 调用数据库特定的函数：

```java
CriteriaBuilder cb = em.getCriteriaBuilder();
Root<User> root = query.from(User.class);

// ---- 调用数据库原生函数 ----
// cb.function(函数名, 返回值类型, 参数...)

// MySQL: DATE_FORMAT
Expression<String> formattedDate = cb.function(
    "DATE_FORMAT",
    String.class,
    root.get("createTime"),
    cb.literal("%Y-%m-%d")
);
// → DATE_FORMAT(create_time, '%Y-%m-%d')

// MySQL: GROUP_CONCAT
Expression<String> concatNames = cb.function(
    "GROUP_CONCAT",
    String.class,
    root.get("name")
);
// → GROUP_CONCAT(name)

// PostgreSQL / H2: TO_CHAR
Expression<String> dateToString = cb.function(
    "TO_CHAR",
    String.class,
    root.get("createTime"),
    cb.literal("YYYY-MM-DD")
);

// ---- 提取日期部分 ----
// MySQL: YEAR(), MONTH(), DAY()
Expression<Integer> year = cb.function("YEAR", Integer.class, root.get("createTime"));
Expression<Integer> month = cb.function("MONTH", Integer.class, root.get("createTime"));

// PostgreSQL: EXTRACT
Expression<Integer> yearPg = cb.function(
    "EXTRACT",
    Integer.class,
    cb.literal("YEAR"),
    root.get("createTime")
);

// ---- 自定义数据库函数（需在数据库注册） ----
// 如果数据库有自定义函数，如 fn_calculate_score(user_id, year)
Expression<Double> score = cb.function(
    "FN_CALCULATE_SCORE",
    Double.class,
    root.get("id"),
    cb.literal(2025)
);

// 注意：使用 cb.literal() 传入字面量值，避免被当作属性路径解析
```

# 9. 静态元模型（Static Metamodel）

## 9.1 概念

静态元模型（JPA Static Metamodel）是 JPA 规范定义的一种机制，为每个实体类生成对应的元模型类（canonical metamodel class）。元模型类将实体属性映射为类型安全的 `SingularAttribute`、`ListAttribute` 等对象，使得 Criteria API 查询中可以使用强类型属性引用替代字符串。

```
实体类：          元模型类：
User.java    →   User_.java
  id: Long         id: SingularAttribute<User, Long>
  name: String     name: SingularAttribute<User, String>
  age: Integer     age: SingularAttribute<User, Integer>
  orders: List     orders: ListAttribute<User, Order>
  address: Addr    address: SingularAttribute<User, Address>
```

有了元模型后，可以将易出错的字符串形式：

```java
root.get("name")       // 字符串 "name" 拼写错误无法在编译期发现
root.get("agee")       // 拼写错误 → 运行时才报错
```

替换为类型安全的形式：

```java
root.get(User_.name)   // 编译期检查，自动补全，重构安全
root.get(User_.age)
```

## 9.2 生成方式

使用 Hibernate 官方提供的注解处理器 `hibernate-jpamodelgen` 在编译期自动生成元模型类。

**Maven 配置：**

```xml
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-jpamodelgen</artifactId>
    <version>${hibernate.version}</version>
    <scope>provided</scope>
</dependency>

<!-- 如果使用 Maven Compiler Plugin，可能需要显式指定注解处理器 -->
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <annotationProcessorPaths>
                    <path>
                        <groupId>org.hibernate.orm</groupId>
                        <artifactId>hibernate-jpamodelgen</artifactId>
                        <version>${hibernate.version}</version>
                    </path>
                </annotationProcessorPaths>
            </configuration>
        </plugin>
    </plugins>
</build>
```

**Gradle 配置：**

```groovy
dependencies {
    annotationProcessor "org.hibernate.orm:hibernate-jpamodelgen:${hibernateVersion}"
    // 或者：
    // implementation "org.hibernate.orm:hibernate-jpamodelgen:${hibernateVersion}"
}
```

**生成的元模型类结构：**（以 User 实体为例）

```java
// 编译后自动生成于 target/generated-sources/annotations/ 下
@StaticMetamodel(User.class)
public abstract class User_ {
    public static volatile SingularAttribute<User, Long> id;
    public static volatile SingularAttribute<User, String> name;
    public static volatile SingularAttribute<User, String> email;
    public static volatile SingularAttribute<User, Integer> age;
    public static volatile SingularAttribute<User, LocalDateTime> createTime;
    public static volatile SingularAttribute<User, Address> address;
    public static volatile ListAttribute<User, Order> orders;
    public static final String ID = "id";
    public static final String NAME = "name";
    public static final String EMAIL = "email";
    public static final String AGE = "age";
    public static final String ORDERS = "orders";
}
```

## 9.3 使用方式

```java
// 字符串方式（原始写法）
root.get("name");
root.join("orders");

// 元模型方式（类型安全写法）
root.get(User_.name);
root.join(User_.orders);

// ---- 完整对比示例 ----
// 原始写法：字符串，无编译期检查
public List<User> findByNameOld(String name) {
    CriteriaBuilder cb = em.getCriteriaBuilder();
    CriteriaQuery<User> query = cb.createQuery(User.class);
    Root<User> root = query.from(User.class);
    query.where(cb.equal(root.get("name"), name));          // "name" 可能拼错
    query.orderBy(cb.desc(root.get("creatTime")));           // 拼写错误！无编译期提示
    return em.createQuery(query).getResultList();
}

// 元模型写法：编译期检查，IDE 自动补全
public List<User> findByNameNew(String name) {
    CriteriaBuilder cb = em.getCriteriaBuilder();
    CriteriaQuery<User> query = cb.createQuery(User.class);
    Root<User> root = query.from(User.class);
    query.where(cb.equal(root.get(User_.name), name));      // 编译期保证正确
    query.orderBy(cb.desc(root.get(User_.createTime)));     // IDE 可自动补全
    return em.createQuery(query).getResultList();
}

// ---- Join 的类型安全 ----
// 元模型进一步提供 Join 返回类型的泛型约束
Join<User, Order> orderJoin = root.join(User_.orders);
// Join 返回类型由元模型定义，确保后续属性访问类型正确
Predicate orderFilter = cb.equal(orderJoin.get(Order_.status), "PAID");

// ---- 嵌套属性路径 ----
// 字符串方式：root.get("address").get("city")
// 元模型方式：
Path<String> city = root.get(User_.address).get(Address_.city);
query.where(cb.equal(city, "北京"));
```

## 9.4 优缺点

| 优点 | 缺点 |
|------|------|
| 类型安全：编译期检查属性名和类型 | 增加编译复杂度：需要配置注解处理器 |
| 重构友好：重命名实体属性时元模型自动更新 | IDE 插件兼容性：某些 IDE 版本可能不支持 |
| IDE 支持好：代码自动补全、跳转定义 | 额外的 generated-sources 目录管理 |
| 减少运行时错误：消除属性名拼写错误 | 多模块项目需注意注解处理器的作用域 |
| 提高可读性：`User_.name` 比 `"name"` 更直观 | 元模型类与实体类需保持同包（默认同名包） |

> **建议**：新项目强烈推荐使用静态元模型，结合 IDE 的注解处理器支持可获得良好的开发体验。旧项目可逐步迁移，先从核心查询开始替换。

# 10. Hibernate 旧 Criteria API（参考）

## 10.1 概述

在 JPA 2.0 引入标准的 Criteria API 之前，Hibernate 3.x 就提供了自己的编程式查询 API——`org.hibernate.Criteria`。该 API 自 Hibernate 5.x 起标记为废弃（@Deprecated），在 Hibernate 6.x 中已正式移除。

**旧 API 核心类：**

```java
// 旧版（已废弃，Hibernate 6 中已移除）
import org.hibernate.Criteria;
import org.hibernate.criterion.Restrictions;
import org.hibernate.criterion.Projections;
import org.hibernate.criterion.Order;

// 创建查询
Criteria criteria = session.createCriteria(User.class);

// 添加条件
criteria.add(Restrictions.eq("name", "John"));        // 等值
criteria.add(Restrictions.like("name", "%John%"));    // 模糊
criteria.add(Restrictions.between("age", 18, 65));    // 范围
criteria.add(Restrictions.in("status", new String[]{"A", "B"}));

// 分页
criteria.setFirstResult(0);
criteria.setMaxResults(20);

// 排序
criteria.addOrder(Order.desc("createTime"));

// 投影
criteria.setProjection(Projections.rowCount());

// 结果
List<User> users = criteria.list();

// DetachedCriteria：可脱离 Session 构建的查询条件
DetachedCriteria detached = DetachedCriteria.forClass(User.class);
detached.add(Restrictions.eq("status", "ACTIVE"));
// 之后附加到 Session：detached.getExecutableCriteria(session).list();
```

**废弃原因：**

1. **类型不安全**：属性名都是字符串，无编译期检查，重构困难。
2. **不易维护**：`Restrictions` / `Projections` API 设计臃肿，方法过多。
3. **不支持 JPA 多态查询**：不支持 JOIN FETCH、关联子查询等高级特性。
4. **非标准**：是 Hibernate 专有 API，无法跨 JPA 实现移植。

**迁移指南：** 从旧 Criteria API 迁移到 JPA Criteria API 的基本对应关系：

| 旧 API | JPA Criteria API |
|--------|-----------------|
| `session.createCriteria(User.class)` | `cb.createQuery(User.class)` + `query.from(User.class)` |
| `criteria.add(Restrictions.eq("name", val))` | `cb.equal(root.get("name"), val)` |
| `criteria.add(Restrictions.like("name", val))` | `cb.like(root.get("name"), val)` |
| `criteria.addOrder(Order.desc("age"))` | `query.orderBy(cb.desc(root.get("age")))` |
| `criteria.setFirstResult(0)` | `typedQuery.setFirstResult(0)` |
| `criteria.setMaxResults(10)` | `typedQuery.setMaxResults(10)` |
| `criteria.list()` | `typedQuery.getResultList()` |
| `criteria.uniqueResult()` | `typedQuery.getSingleResult()` |
| `DetachedCriteria.forClass(User.class)` | 无直接等价物，使用 Specification 模式（见第 11 章） |

# 11. Spring Data JPA 中的 Criteria

## 11.1 JpaSpecificationExecutor

Spring Data JPA 提供了 `JpaSpecificationExecutor` 接口，它封装了 Criteria API 的复杂性，通过 `Specification<T>` 函数式接口实现动态查询的声明式构建。

```java
// 让 Repository 继承 JpaSpecificationExecutor
public interface UserRepository extends
        JpaRepository<User, Long>,
        JpaSpecificationExecutor<User> {   // 额外继承此接口
}

// Specification 接口定义
@FunctionalInterface
public interface Specification<T> {
    Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder cb);

    // 默认方法：组合
    default Specification<T> and(Specification<T> other) { ... }
    default Specification<T> or(Specification<T> other) { ... }
    static <T> Specification<T> where(Specification<T> spec) { ... }
}
```

`toPredicate` 的入参正是 Criteria API 的三个核心对象：`Root`、`CriteriaQuery`、`CriteriaBuilder`，因此 Spring Data JPA 的 Specification 本质上就是对 Criteria API 的封装。

## 11.2 Specification 组合

```java
// ---- 定义独立 Specification ----

public class UserSpecifications {

    /**
     * 按名称模糊查询
     */
    public static Specification<User> nameLike(String keyword) {
        return (root, query, cb) -> {
            if (keyword == null || keyword.isBlank()) {
                return cb.conjunction();  // 空条件返回恒真
            }
            return cb.like(root.get("name"), "%" + keyword + "%");
        };
    }

    /**
     * 按年龄区间过滤
     */
    public static Specification<User> ageBetween(Integer minAge, Integer maxAge) {
        return (root, query, cb) -> {
            Predicate predicate = cb.conjunction();
            if (minAge != null) {
                predicate = cb.and(predicate, cb.ge(root.get("age"), minAge));
            }
            if (maxAge != null) {
                predicate = cb.and(predicate, cb.le(root.get("age"), maxAge));
            }
            return predicate;
        };
    }

    /**
     * 按状态过滤
     */
    public static Specification<User> statusIn(List<String> statuses) {
        return (root, query, cb) -> {
            if (statuses == null || statuses.isEmpty()) {
                return cb.conjunction();
            }
            return root.get("status").in(statuses);
        };
    }

    /**
     * 按邮箱模糊查询
     */
    public static Specification<User> emailLike(String keyword) {
        return (root, query, cb) ->
            cb.like(root.get("email"), "%" + keyword + "%");
    }
}

// ---- 组合使用 ----
@Service
public class UserQueryService {

    @Autowired
    private UserRepository userRepository;

    public Page<User> searchUsers(String name, Integer minAge, Integer maxAge,
                                   List<String> statuses, Pageable pageable) {
        Specification<User> spec = Specification
            .where(UserSpecifications.nameLike(name))
            .and(UserSpecifications.ageBetween(minAge, maxAge))
            .and(UserSpecifications.statusIn(statuses));

        return userRepository.findAll(spec, pageable);
    }
}
```

## 11.3 完整示例

下面是一个完整的动态查询服务，展示如何使用 Specification 结合分页和排序：

```java
/**
 * 用户搜索 DTO
 */
@Data
public class UserSearchRequest {
    private String keyword;          // 姓名/邮箱模糊搜索
    private Integer minAge;          // 最小年龄
    private Integer maxAge;          // 最大年龄
    private String status;           // 精确状态
    private LocalDate startDate;     // 注册起始日期
    private LocalDate endDate;       // 注册结束日期
}

/**
 * 用户查询服务
 */
@Service
public class UserSearchService {

    @Autowired
    private UserRepository userRepository;

    public Page<UserDTO> search(UserSearchRequest req, int page, int size) {
        // 构建 Specification
        Specification<User> spec = buildSpecification(req);

        // 构建分页 + 排序
        PageRequest pageable = PageRequest.of(
            page - 1,                     // Spring Data 页码从 0 开始
            size,
            Sort.by(Sort.Direction.DESC, "createTime")  // 默认按创建时间降序
        );

        // 执行分页查询
        Page<User> userPage = userRepository.findAll(spec, pageable);

        // 转换为 DTO
        return userPage.map(user -> new UserDTO(
            user.getName(),
            user.getEmail(),
            user.getAge(),
            user.getCreateTime()
        ));
    }

    private Specification<User> buildSpecification(UserSearchRequest req) {
        return (root, query, cb) -> {
            List<Predicate> predicates = new ArrayList<>();

            // 关键词模糊搜索（同时匹配姓名和邮箱）
            if (req.getKeyword() != null && !req.getKeyword().isBlank()) {
                String pattern = "%" + req.getKeyword() + "%";
                predicates.add(
                    cb.or(
                        cb.like(root.get("name"), pattern),
                        cb.like(root.get("email"), pattern)
                    )
                );
            }

            // 年龄区间
            if (req.getMinAge() != null) {
                predicates.add(cb.ge(root.get("age"), req.getMinAge()));
            }
            if (req.getMaxAge() != null) {
                predicates.add(cb.le(root.get("age"), req.getMaxAge()));
            }

            // 精确状态
            if (req.getStatus() != null && !req.getStatus().isBlank()) {
                predicates.add(cb.equal(root.get("status"), req.getStatus()));
            }

            // 注册日期区间
            if (req.getStartDate() != null) {
                predicates.add(cb.greaterThanOrEqualTo(root.get("createDate"), req.getStartDate()));
            }
            if (req.getEndDate() != null) {
                predicates.add(cb.lessThanOrEqualTo(root.get("createDate"), req.getEndDate()));
            }

            return cb.and(predicates.toArray(new Predicate[0]));
        };
    }
}
```

# 12. 常见问题与最佳实践

## 12.1 动态查询模式

动态查询是 Criteria API 的核心应用场景。以下是经过生产验证的最佳实践：

```java
/**
 * 模式一：精确控制模式（推荐）
 * 使用 conjunction() 作为起点，逐条检查并添加条件
 */
public List<User> dynamicQuery(QueryParams params) {
    CriteriaBuilder cb = em.getCriteriaBuilder();
    CriteriaQuery<User> query = cb.createQuery(User.class);
    Root<User> root = query.from(User.class);

    Predicate predicate = cb.conjunction();  // 恒真起点

    if (params.getName() != null) {
        predicate = cb.and(predicate,
            cb.like(root.get("name"), "%" + params.getName() + "%"));
    }
    if (params.getMinAge() != null) {
        predicate = cb.and(predicate, cb.ge(root.get("age"), params.getMinAge()));
    }
    // ... 更多条件

    query.where(predicate);
    return em.createQuery(query).getResultList();
}

/**
 * 模式二：List + toArray 模式
 * 条件收集到 List，最后统一组合
 */
public List<User> dynamicQueryV2(QueryParams params) {
    CriteriaBuilder cb = em.getCriteriaBuilder();
    CriteriaQuery<User> query = cb.createQuery(User.class);
    Root<User> root = query.from(User.class);

    List<Predicate> predicates = new ArrayList<>();
    // 条件检查后直接 add
    if (params.getName() != null) {
        predicates.add(cb.like(root.get("name"), "%" + params.getName() + "%"));
    }
    // ...

    if (!predicates.isEmpty()) {
        query.where(cb.and(predicates.toArray(new Predicate[0])));
    }
    // 注意：predicates 为空时不设置 where，查询返回全部
    return em.createQuery(query).getResultList();
}

/**
 * 提取可复用的 Predicate 构建方法
 * 大型项目中推荐将常用的条件构建提取为工具方法
 */
public class PredicateUtils {
    public static Predicate likeIfNotEmpty(CriteriaBuilder cb, Path<String> path, String value) {
        if (value == null || value.isBlank()) return cb.conjunction();
        return cb.like(path, "%" + value + "%");
    }

    public static Predicate geIfNotNull(CriteriaBuilder cb, Path<Integer> path, Integer value) {
        if (value == null) return cb.conjunction();
        return cb.ge(path, value);
    }

    public static Predicate inIfNotEmpty(CriteriaBuilder cb, Path<?> path, List<?> values) {
        if (values == null || values.isEmpty()) return cb.conjunction();
        CriteriaBuilder.In<Object> in = cb.in(path);
        values.forEach(in::value);  // 或使用 path.in(values)
        return in;
    }
}
```

## 12.2 性能注意事项

1. **FETCH 应仅在确实需要时使用**：Join Fetch 虽然能解决 N+1 问题，但同时 Fetch 多个集合（如一对多）会导致笛卡尔积，数据集膨胀严重。对于报表类查询，优先使用 `Tuple` 或多属性查询获取必要字段。

2. **避免在循环中创建多个查询**：CriteriaBuilder、CriteriaQuery 的创建开销虽不大，但每次调用 `em.createQuery()` 都会编译生成 SQL。将可复用的查询逻辑提取为方法，避免在集合循环中重复构建。

3. **CriteriaQuery 不可变（不可重用）**：一个 `CriteriaQuery` 实例只能用于创建一次 `TypedQuery`。如果需要重复执行相同结构的查询（如不同参数），使用 `ParameterExpression`：

```java
// 创建带参数的查询模板
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<User> query = cb.createQuery(User.class);
Root<User> root = query.from(User.class);

ParameterExpression<String> nameParam = cb.parameter(String.class, "name");
query.where(cb.equal(root.get("name"), nameParam));

// 多次执行相同查询，仅绑定不同参数
TypedQuery<User> q1 = em.createQuery(query);
q1.setParameter("name", "Alice");
List<User> users1 = q1.getResultList();

TypedQuery<User> q2 = em.createQuery(query);
q2.setParameter("name", "Bob");
List<User> users2 = q2.getResultList();
```

4. **大数据量查询使用游标**：对于百万级数据处理，使用 `setFetchSize()` 控制每次读取的记录数，或使用 `StatelessSession` 避免一级缓存堆积。

5. **避免不必要的 JOIN**：只在确实需要过滤或获取关联数据时才添加 JOIN。不必要的 JOIN 会增加数据库负担。

## 12.3 Criteria vs JPQL 选择

| 维度 | Criteria API | JPQL |
|------|-------------|------|
| **可读性** | 复杂查询时代码膨胀，可读性下降 | 声明式语法，清晰直观，接近 SQL |
| **动态查询** | 天然支持，通过条件判断动态组装 | 需手动拼接字符串，易出错 |
| **编译期检查** | 类型安全，编译期发现错误 | 字符串形式，运行时才报错 |
| **开发效率** | 简单查询代码量大 | 简单查询编写快速 |
| **重构安全** | 实体属性改名后自动更新（需元模型） | 字符串不会自动更新 |
| **维护成本** | 复杂查询维护困难 | 查询逻辑一目了然 |

**选择建议：**

- **静态查询**（条件固定）→ **JPQL**：如 `SELECT u FROM User u WHERE u.status = 'ACTIVE'`，JPQL 更简洁清晰。
- **动态查询**（条件可变）→ **Criteria API**：如搜索页面的多条件组合查询。
- **简单动态 + 复杂逻辑** → **Specification + Spring Data JPA**：平衡了灵活性和可读性。
- **极复杂 SQL 查询** → 考虑使用原生 SQL（`@Query(nativeQuery = true)`）或 MyBatis。Criteria API 对于 5 表以上 JOIN + 多层子查询的复杂场景，代码可读性会急剧下降。

**综合示例——合理混合使用：**

```java
// 静态查询 → JPQL（Repository 方法）
@Query("SELECT u FROM User u WHERE u.status = 'ACTIVE'")
List<User> findActiveUsers();

// 动态查询 → Specification（Repository 继承 JpaSpecificationExecutor）
Page<User> searchUsers(UserSearchRequest req, Pageable pageable);

// 极复杂统计 → 原生 SQL 或视图
@Query(value = "SELECT ... FROM user u JOIN ... GROUP BY ... HAVING ...", nativeQuery = true)
List<Object[]> complexReport();
```
