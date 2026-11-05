# 1. PL/SQL 是什么

**PL/SQL（Procedural Language/SQL）** 是 Oracle 在 SQL 之上扩展的过程化语言，把变量、条件、循环、异常、游标这些编程能力带进了数据库。它的价值是**减少应用与数据库之间的往返**：一段需要多次查询和计算的逻辑，写成 PL/SQL 后在数据库内部一次调用完成。

但这也是一把双刃剑。把大量业务塞进存储过程，会带来版本管理困难、调试不便、数据库 CPU 集中、迁移成本高等问题。PL/SQL 适合**数据密集型批处理、复杂校验、权限内的封装**，而不是把所有业务都下沉。SQL 擅长集合操作，PL/SQL 擅长流程控制与封装，但逐行处理大量数据时通常比集合 SQL 慢一个数量级。

> **行业实践**：能用一条 SQL 解决的，不要写游标循环。PL/SQL 的优势来自减少往返，一旦变成逐行处理百万数据，反而更慢。

# 2. 块结构

PL/SQL 的基本单位是**块（Block）**，由声明、执行、异常三部分组成，只有执行部分（`BEGIN ... END`）是必需的。

```sql
DECLARE
    -- 声明部分：变量、常量、游标、自定义类型
    v_name  VARCHAR2(50 CHAR) := 'Oracle';
    v_count NUMBER := 0;
BEGIN
    -- 执行部分：SQL 与流程控制
    SELECT COUNT(*) INTO v_count FROM user_tables;
    DBMS_OUTPUT.PUT_LINE('表数量：' || v_count);
EXCEPTION
    -- 异常部分：错误处理
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('发生错误：' || SQLERRM);
END;
/
```

几个语法细节必须记住：每条语句以**分号**结尾，整个块用 `/` 单独一行结束；`DBMS_OUTPUT.PUT_LINE` 需要先 `SET SERVEROUTPUT ON`；赋值用 `:=`，`=` 只用于比较；字符串拼接用 `||`，不是 `+`。

**匿名块**没有名字，声明后直接执行，适合一次性脚本和测试；**命名块**是存储过程、函数、包、触发器，编译后存进数据库，可反复调用。

# 3. 变量与类型

```sql
DECLARE
    v_id       NUMBER(19) := 1;                    -- 带初值
    v_name     VARCHAR2(50 CHAR) NOT NULL := 'x';  -- 非空必须初始化
    c_tax_rate CONSTANT NUMBER(5, 4) := 0.13;      -- 常量
BEGIN
    v_name := 'Oracle';
    DBMS_OUTPUT.PUT_LINE(v_name || ' 税率 ' || c_tax_rate);
END;
/
```

手写变量类型最怕表结构变更后不同步。`%TYPE` 取某列的类型，`%ROWTYPE` 取整行的类型，表结构改了变量自动跟随。

```sql
DECLARE
    v_amount orders.amount%TYPE;   -- 与 orders.amount 同类型
    v_order  orders%ROWTYPE;       -- 与 orders 整行同结构
BEGIN
    SELECT * INTO v_order FROM orders WHERE id = 1;
    DBMS_OUTPUT.PUT_LINE(v_order.id || ':' || v_order.amount);
END;
/
```

> **设计考量**：`%TYPE` 是 PL/SQL 里重要的健壮性手段。比如 `NUMBER(18, 2)` 后来扩成 `NUMBER(20, 2)`，用 `%TYPE` 的过程不需要改，硬编码类型的地方就可能溢出。

需要自定义结构时用 `RECORD`，需要按键索引的集合时用关联数组（`TABLE OF ... INDEX BY PLS_INTEGER`），用法与普通变量类似，这里不再展开。

# 4. 控制结构

```sql
DECLARE
    v_score NUMBER := 85;
BEGIN
    IF v_score >= 90 THEN
        DBMS_OUTPUT.PUT_LINE('优秀');
    ELSIF v_score >= 60 THEN
        DBMS_OUTPUT.PUT_LINE('及格');
    ELSE
        DBMS_OUTPUT.PUT_LINE('不及格');
    END IF;

    FOR i IN 1..3 LOOP                     -- 简单循环
        DBMS_OUTPUT.PUT_LINE('第 ' || i || ' 次');
    END LOOP;
END;
/
```

> **注意**：`FOR i IN 1..3` 的循环变量是隐式声明的，不要在 `DECLARE` 里再声明同名变量，否则会报错或产生遮蔽。循环变量在循环内只读。

# 5. 游标

游标用于**逐行处理结果集**，分**隐式游标**（Oracle 自动管理）和**显式游标**（开发者声明）。

```sql
DECLARE
    CURSOR c_orders IS
        SELECT id, amount FROM orders WHERE status = 'NEW';
    v_id     orders.id%TYPE;
    v_amount orders.amount%TYPE;
BEGIN
    OPEN c_orders;                              -- 1. 打开，执行查询
    LOOP
        FETCH c_orders INTO v_id, v_amount;     -- 2. 取一行
        EXIT WHEN c_orders%NOTFOUND;            -- 3. 无数据则退出
        DBMS_OUTPUT.PUT_LINE(v_id || ':' || v_amount);
    END LOOP;
    CLOSE c_orders;                             -- 4. 关闭，释放资源
END;
/
```

游标属性：`%FOUND` 最近一次是否取到行，`%NOTFOUND` 是否没取到，`%ROWCOUNT` 已取行数，`%ISOPEN` 是否已打开。

显式游标的 OPEN/FETCH/CLOSE 样板容易漏写 `CLOSE`。**游标 FOR 循环**会自动打开、逐行取、自动关闭，是日常最常用的写法：

```sql
BEGIN
    FOR r IN (SELECT id, amount FROM orders WHERE status = 'NEW') LOOP
        DBMS_OUTPUT.PUT_LINE(r.id || ':' || r.amount);
    END LOOP;
END;
/
```

带参数的游标让查询条件可复用，写法是在游标名后加参数列表，例如 `CURSOR c_orders(p_status VARCHAR2) IS SELECT ... WHERE status = p_status`，循环时用 `FOR r IN c_orders('NEW')` 传入。

> **行业实践**：能不用游标就不用。如果只是"把一张表处理后写入另一张表"，优先用 `INSERT INTO ... SELECT` 或 `MERGE`，一条 SQL 完成，避免百万次上下文切换。游标适合"每行需要复杂逻辑或外部调用"的场景。

# 6. 存储过程与函数

```sql
CREATE OR REPLACE PROCEDURE sp_create_order(
    p_user_id  IN  NUMBER,
    p_amount   IN  NUMBER,
    p_order_id OUT NUMBER
) AS
BEGIN
    IF p_amount IS NULL OR p_amount <= 0 THEN
        RAISE_APPLICATION_ERROR(-20001, '金额必须大于 0');
    END IF;
    SELECT seq_orders.NEXTVAL INTO p_order_id FROM dual;
    INSERT INTO orders (id, user_id, amount, status, created_at)
    VALUES (p_order_id, p_user_id, p_amount, 'NEW', SYSDATE);
END;
/
```

调用时在匿名块中声明变量接收 `OUT` 参数，例如 `sp_create_order(1, 99.00, v_id)`，随后即可读取 `v_id` 并提交。

函数与过程的区别是**函数必须返回值**，且通常应尽量**只读**，否则在 SQL 中调用时可能违反读一致性。

```sql
CREATE OR REPLACE FUNCTION fn_order_level(p_amount NUMBER)
RETURN VARCHAR2 IS
BEGIN
    RETURN CASE
        WHEN p_amount >= 1000 THEN 'VIP'
        WHEN p_amount >= 100  THEN 'NORMAL'
        ELSE 'LOW'
    END;
END;
/

SELECT id, amount, fn_order_level(amount) AS level FROM orders;
```

参数模式有三种：`IN` 是输入、过程内只读；`OUT` 是输出、过程内可写；`IN OUT` 既可输入也可输出。

> **注意**：`OUT` 参数在过程内初始为 NULL，不要假设它带着调用方的旧值。需要"读入再改写"就用 `IN OUT`。

# 7. 包

**包（Package）** 把相关的过程、函数、变量、游标组织在一起，分为**规范（spec）**和**包体（body）**。规范是对外接口，包体是实现，支持重载、封装私有逻辑、减少依赖编译。

```sql
-- 1. 包规范：只声明对外可见的接口
CREATE OR REPLACE PACKAGE pkg_order AS
    g_tax_rate CONSTANT NUMBER := 0.13;
    PROCEDURE create_order(p_user_id IN NUMBER, p_amount IN NUMBER);
    FUNCTION  calc_tax(p_amount IN NUMBER) RETURN NUMBER;
END pkg_order;
/

-- 2. 包体：实现，可包含私有过程和函数
CREATE OR REPLACE PACKAGE BODY pkg_order AS

    PROCEDURE create_order(p_user_id IN NUMBER, p_amount IN NUMBER) IS
    BEGIN
        INSERT INTO orders (id, user_id, amount, status)
        VALUES (seq_orders.NEXTVAL, p_user_id, p_amount, 'NEW');
    END create_order;

    FUNCTION calc_tax(p_amount IN NUMBER) RETURN NUMBER IS
    BEGIN
        RETURN ROUND(p_amount * g_tax_rate, 2);
    END calc_tax;

END pkg_order;
/

SELECT pkg_order.calc_tax(100) FROM dual;
```

> **设计考量**：包的规范一旦被其他对象依赖，改规范会导致依赖对象失效（`INVALID`），而改包体通常不影响调用方。所以**把稳定接口放规范、易变实现放包体**，是减少编译连锁反应的关键。

# 8. 触发器

触发器在表的 `INSERT`、`UPDATE`、`DELETE` 前后自动执行，分**行级（FOR EACH ROW）**和**语句级**。

```sql
-- 行级触发器：自动维护 updated_at
CREATE OR REPLACE TRIGGER trg_orders_updated
BEFORE UPDATE ON orders
FOR EACH ROW
BEGIN
    :NEW.updated_at := SYSDATE;
END;
/
```

`INSERT` 和 `UPDATE` 触发器里可以用 `:NEW` 引用新值，`UPDATE` 和 `DELETE` 里可以用 `:OLD` 引用旧值，语句级触发器没有这两个伪记录。触发器常用于自动填充时间戳、审计日志、跨表校验和同步冗余表，但共同的风险是**逻辑隐蔽、难以测试、级联触发难追踪**。

> **警告**：触发器里的逻辑对应用是"隐式"的，容易造成"我只插了一行，为什么另一张表也变了"的困惑。生产环境应严格控制触发器数量，能用应用层或物化视图解决的尽量不用。触发器内的 DML 参与当前事务，异常时一起回滚。

# 9. 异常处理

```sql
DECLARE
    v_name orders.status%TYPE;
BEGIN
    SELECT status INTO v_name FROM orders WHERE id = -1;
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('没有找到数据');
    WHEN TOO_MANY_ROWS THEN
        DBMS_OUTPUT.PUT_LINE('返回了多行');
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('其他错误：' || SQLCODE || ' - ' || SQLERRM);
END;
/
```

自定义异常用 `RAISE` 主动抛出，业务错误用 `RAISE_APPLICATION_ERROR` 抛出错误号（范围 -20000 到 -20999），方便应用捕获：

```sql
-- 自定义异常：声明后 RAISE，在 WHEN 中捕获
DECLARE
    e_amount_invalid EXCEPTION;
BEGIN
    RAISE e_amount_invalid;
EXCEPTION
    WHEN e_amount_invalid THEN DBMS_OUTPUT.PUT_LINE('金额非法');
END;
/
-- 业务错误：抛出 -20000 到 -20999 的错误号，供应用捕获
RAISE_APPLICATION_ERROR(-20002, '库存不能为负');
```

> **注意**：`WHEN OTHERS THEN NULL` 是最危险的写法。它把错误吞掉，事务可能已经不一致，调用方却以为成功。正确做法是记录日志后 `RAISE` 重新抛出，或至少把 `SQLERRM` 写进日志表。

# 10. 性能注意

**第一，避免逐行提交。** 在循环里 `COMMIT` 会产生大量日志同步等待，且破坏事务原子性。应整批提交或由调用方控制事务边界。

**第二，避免在循环里执行 SQL。** 每次 `SELECT`/`INSERT` 都是上下文切换，能批量就批量：

```sql
-- 错误示范：逐行插入
FOR r IN (SELECT * FROM src_table) LOOP
    INSERT INTO dst_table VALUES r.id, r.name;
END LOOP;
-- 正确做法：一条集合 SQL
INSERT INTO dst_table (id, name) SELECT id, name FROM src_table;
```

**第三，注意大集合参数的复制开销。** `OUT`/`IN OUT` 的大集合默认按值复制，可用 `NOCOPY` 提示按引用传递（编译器可忽略该提示）。

**第四，注意包状态。** 包的会话级变量在会话内一直存在，连接池复用时可能"串状态"。需要无状态时用 `PRAGMA SERIALLY_REUSABLE` 或在调用开始时显式初始化。

**第五，关注编译与依赖。** 表结构变更会让依赖的存储过程变 `INVALID`，首次调用时自动重编译，可能带来瞬时延迟。大批量 DDL 后应主动 `UTL_RECOMP` 或 `ALTER ... COMPILE`。

# 11. 练习

- **练习一**：写匿名块，用游标 FOR 循环遍历 `orders`，打印金额大于 1000 的订单并计数；再改写成一条 `SELECT COUNT(*)`，比较百万行下的耗时差异。
- **练习二**：创建包 `pkg_account`，规范声明 `deposit`、`withdraw`，包体用私有函数校验余额，取款超余额时抛 `RAISE_APPLICATION_ERROR`，并写调用示例验证异常能被捕获。
- **练习三**：写 `BEFORE UPDATE` 行级触发器自动更新 `updated_at`，验证更新后 `updated_at` 变化、`created_at` 不变，并说明这类逻辑放触发器里的利弊。

# 12. 小结

| 主题 | 核心结论 |
|------|---------|
| 块结构 | `DECLARE`/`BEGIN`/`EXCEPTION` 三段，只有执行段必需 |
| 变量 | `:=` 赋值，`%TYPE`/`%ROWTYPE` 绑定表结构 |
| 游标 | 显式游标四步，优先用游标 FOR 循环自动管理 |
| 过程与函数 | 过程无返回值，函数必须 `RETURN`，参数分 `IN`/`OUT`/`IN OUT` |
| 包 | 规范定接口、包体放实现，改包体不影响调用方 |
| 触发器 | 隐式执行、难排查，生产环境要克制 |
| 异常 | 自定义异常 + `RAISE_APPLICATION_ERROR`，禁止 `WHEN OTHERS THEN NULL` |
| 性能 | 拒绝逐行提交和在循环里跑 SQL，优先集合操作 |

PL/SQL 让我们能在数据库内完成复杂逻辑，但真正决定系统正确性的，是**事务边界与锁**。下一篇进入事务与锁，讲清 Oracle 的隔离级别实现、行锁与表锁、死锁排查，以及 `AUTONOMOUS_TRANSACTION` 这个既强大又危险的特性。
