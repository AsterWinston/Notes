# 1. 继承映射概述

面向对象编程的核心特性之一——继承，允许子类复用父类的属性和行为，形成 "is-a" 的层级关系。然而，关系数据库并不原生支持继承概念，表与表之间通过外键建立关联，不存在真正的父子继承关系。这种面向对象继承与关系数据库建模之间的"阻抗不匹配"（Impedance Mismatch），正是 ORM 框架需要解决的核心问题之一。

JPA（Java Persistence API）提供了四种处理继承关系的策略，分别对应不同的表结构设计思想：

| 策略 | 枚举值 | 核心思想 |
|------|--------|----------|
| 单表策略 | `InheritanceType.SINGLE_TABLE` | 所有子类共用一张表，通过鉴别器列区分类型 |
| 连接表策略 | `InheritanceType.JOINED` | 父类一张表，每个子类额外一张表，通过 JOIN 关联 |
| 每类一表策略 | `InheritanceType.TABLE_PER_CLASS` | 每个具体类一张独立的表，包含所有继承字段 |
| 超类映射 | `@MappedSuperclass` | 非继承策略，仅代码复用，父类不映射为表 |

**选型决策框架**：

1. 首先确认：是否需要以父类类型进行多态查询和多态关联？
   - 需要 → 使用 `SINGLE_TABLE`、`JOINED` 或 `TABLE_PER_CLASS`
   - 不需要 → 使用 `@MappedSuperclass`（纯代码复用）

2. 再评估子类字段差异程度：
   - 子类字段差异小（2-3 个特有字段）→ `SINGLE_TABLE`
   - 子类字段差异大（10+ 特有字段）→ `JOINED`

3. 评估查询模式：
   - 频繁多态查询 → `SINGLE_TABLE`（性能最优）
   - 几乎不多态查询 → `TABLE_PER_CLASS`（具体类查询最快）
   - 数据完整性优先 → `JOINED`（满足第三范式）

4. 考虑数据库限制：
   - 不支持 JOIN 操作 → `SINGLE_TABLE` 或 `TABLE_PER_CLASS`
   - 遗留数据库已有表结构 → 按现有表结构选择对应策略

---

# 2. SINGLE_TABLE（单表策略）

## 2.1 策略说明

单表策略是 JPA 的**默认继承策略**。使用 `@Inheritance(strategy = InheritanceType.SINGLE_TABLE)` 显式声明后，整个继承体系中所有子类的数据都存储在同一张数据库表中。

核心注解：

- **`@Inheritance(strategy = InheritanceType.SINGLE_TABLE)`**：标注在父类上，声明使用单表策略。
- **`@DiscriminatorColumn`**：定义鉴别器列，用于区分每一行数据属于哪个子类。默认列名为 `DTYPE`，类型为 `VARCHAR(31)`。
- **`@DiscriminatorValue`**：标注在每个子类上，定义该子类在鉴别器列中的值。如果不指定，Hibernate 默认使用实体类名（不包含包路径）。

```java
// 父类定义
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "vehicle_type", discriminatorType = DiscriminatorType.STRING, length = 20)
public abstract class Vehicle {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String manufacturer;
    private Double price;
}

// 子类 Car
@Entity
@DiscriminatorValue("CAR")
public class Car extends Vehicle {
    private Integer seatCount;
    private String fuelType;   // PETROL, DIESEL, ELECTRIC
}

// 子类 Truck
@Entity
@DiscriminatorValue("TRUCK")
public class Truck extends Vehicle {
    private Double loadCapacity;  // 吨
    private Integer axleCount;    // 轴数
}
```

> **注意**：`@DiscriminatorValue` 如果省略，Hibernate 默认使用类名（如 `"Car"`, `"Truck"`）。如果自行指定继承策略映射到遗留数据库，可以通过 `@DiscriminatorFormula` 或自定义鉴别器值来适配。

## 2.2 表结构

单表策略下，整个继承层次只生成**一张表**，包含所有子类的全部字段。不属于某个子类的字段，该行的值为 `NULL`。

**MySQL DDL**：

```sql
CREATE TABLE vehicle (
    id             BIGINT       NOT NULL AUTO_INCREMENT,
    vehicle_type   VARCHAR(20)  NOT NULL,
    manufacturer   VARCHAR(255),
    price          DOUBLE,
    seat_count     INTEGER,       -- Car 特有字段
    fuel_type      VARCHAR(255),  -- Car 特有字段
    load_capacity  DOUBLE,        -- Truck 特有字段
    axle_count     INTEGER,       -- Truck 特有字段
    PRIMARY KEY (id)
);
```

**PostgreSQL DDL**：

```sql
CREATE TABLE vehicle (
    id             BIGSERIAL    NOT NULL,
    vehicle_type   VARCHAR(20)  NOT NULL,
    manufacturer   VARCHAR(255),
    price          DOUBLE PRECISION,
    seat_count     INTEGER,       -- Car 特有字段，插入 Truck 时为 NULL
    fuel_type      VARCHAR(255),
    load_capacity  DOUBLE PRECISION,
    axle_count     INTEGER,
    PRIMARY KEY (id)
);
```

单表策略下，表结构的核心特征：

- 子类特有字段必须允许 `NULL`（数据库层面无法强制 NOT NULL，因为插入其他子类时这些列为 NULL）。
- 无法在数据库层面添加针对子类的 `CHECK` 约束来保证数据完整性。
- 鉴别器列通常添加 `NOT NULL` 约束，确保每行数据有明确的类型标识。

## 2.3 完整示例

### 注解配置

```java
import javax.persistence.*;

@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "vehicle_type", discriminatorType = DiscriminatorType.STRING)
@Table(name = "vehicle")
public abstract class Vehicle {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String manufacturer;

    private Double price;

    public Vehicle() {}

    public Vehicle(String manufacturer, Double price) {
        this.manufacturer = manufacturer;
        this.price = price;
    }

    // getters and setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getManufacturer() { return manufacturer; }
    public void setManufacturer(String manufacturer) { this.manufacturer = manufacturer; }
    public Double getPrice() { return price; }
    public void setPrice(Double price) { this.price = price; }
}
```

```java
import javax.persistence.*;

@Entity
@DiscriminatorValue("CAR")
public class Car extends Vehicle {

    private Integer seatCount;

    @Column(name = "fuel_type")
    private String fuelType;

    public Car() {}

    public Car(String manufacturer, Double price, Integer seatCount, String fuelType) {
        super(manufacturer, price);
        this.seatCount = seatCount;
        this.fuelType = fuelType;
    }

    public Integer getSeatCount() { return seatCount; }
    public void setSeatCount(Integer seatCount) { this.seatCount = seatCount; }
    public String getFuelType() { return fuelType; }
    public void setFuelType(String fuelType) { this.fuelType = fuelType; }
}
```

```java
import javax.persistence.*;

@Entity
@DiscriminatorValue("TRUCK")
public class Truck extends Vehicle {

    @Column(name = "load_capacity")
    private Double loadCapacity;

    private Integer axleCount;

    public Truck() {}

    public Truck(String manufacturer, Double price, Double loadCapacity, Integer axleCount) {
        super(manufacturer, price);
        this.loadCapacity = loadCapacity;
        this.axleCount = axleCount;
    }

    public Double getLoadCapacity() { return loadCapacity; }
    public void setLoadCapacity(Double loadCapacity) { this.loadCapacity = loadCapacity; }
    public Integer getAxleCount() { return axleCount; }
    public void setAxleCount(Integer axleCount) { this.axleCount = axleCount; }
}
```

### 插入操作生成的 SQL

```java
// 测试代码
Vehicle car = new Car("Tesla", 29999.0, 5, "ELECTRIC");
Vehicle truck = new Truck("Volvo", 89999.0, 18.5, 3);

entityManager.persist(car);
entityManager.persist(truck);
```

**Hibernate 生成的 SQL**（单表策略下插入操作只需一条 SQL，插入到同一张表）：

```sql
-- 插入 Car
INSERT INTO vehicle (manufacturer, price, seat_count, fuel_type, vehicle_type)
VALUES ('Tesla', 29999.0, 5, 'ELECTRIC', 'CAR');

-- 插入 Truck
INSERT INTO vehicle (manufacturer, price, load_capacity, axle_count, vehicle_type)
VALUES ('Volvo', 89999.0, 18.5, 3, 'TRUCK');
```

> 注意：INSERT 语句仅包含**当前子类实际拥有**的字段，其他子类的字段不会出现在当前 INSERT 语句中（即使它们在表中以 NULL 值存在）。

**表中的实际数据**：

| id | vehicle_type | manufacturer | price | seat_count | fuel_type | load_capacity | axle_count |
|----|-------------|-------------|-------|------------|-----------|---------------|------------|
| 1  | CAR         | Tesla       | 29999 | 5          | ELECTRIC  | NULL          | NULL       |
| 2  | TRUCK       | Volvo       | 89999 | NULL       | NULL      | 18.5          | 3          |

### 查询操作生成的 SQL

```java
// 按 ID 查询具体子类
Car car = entityManager.find(Car.class, 1L);

// 按 ID 查询父类（多态查询）
Vehicle vehicle = entityManager.find(Vehicle.class, 2L);
```

```sql
-- 查询具体子类 Car：自动添加 DTYPE 过滤
SELECT c.id, c.manufacturer, c.price, c.seat_count, c.fuel_type
FROM vehicle c
WHERE c.id = 1 AND c.vehicle_type = 'CAR';

-- 多态查询父类 Vehicle：无需 DTYPE 过滤，因为要返回所有类型
SELECT v.id, v.manufacturer, v.price, v.seat_count, v.fuel_type,
       v.load_capacity, v.axle_count, v.vehicle_type
FROM vehicle v
WHERE v.id = 2;
```

## 2.4 多态查询

多态查询是单表策略的核心优势——查询父类实例时，Hibernate 自动扫描所有子类。

### 查询父类 → 自动包含所有子类

```java
// JPQL 查询所有 Vehicle（多态查询）
List<Vehicle> vehicles = entityManager.createQuery(
    "SELECT v FROM Vehicle v", Vehicle.class
).getResultList();
```

生成的 SQL（单表策略下，多态查询无需 JOIN，只需扫描一张表）：

```sql
SELECT v.id, v.vehicle_type, v.manufacturer, v.price,
       v.seat_count, v.fuel_type, v.load_capacity, v.axle_count
FROM vehicle v;
```

Hibernate 读取 `vehicle_type` 列，根据 `@DiscriminatorValue` 将每行数据映射到对应的子类实例：

- `vehicle_type = 'CAR'` → 实例化为 `Car` 对象
- `vehicle_type = 'TRUCK'` → 实例化为 `Truck` 对象

### 查询具体子类 → 自动添加 DTYPE 过滤

```java
// 查询所有 Car 实例
List<Car> cars = entityManager.createQuery(
    "SELECT c FROM Car c", Car.class
).getResultList();
```

```sql
SELECT c.id, c.manufacturer, c.price, c.seat_count, c.fuel_type
FROM vehicle c
WHERE c.vehicle_type = 'CAR';
```

### JPQL 中的 TYPE() 表达式

JPQL 提供 `TYPE()` 函数，用于在多态查询中基于类型进行过滤：

```java
// 查询所有是 Car 类型的 Vehicle（效果等同于 Car.class 查询）
List<Vehicle> cars = entityManager.createQuery(
    "SELECT v FROM Vehicle v WHERE TYPE(v) = Car", Vehicle.class
).getResultList();

// 查询所有 Car 或 Truck 类型的 Vehicle
List<Vehicle> vehicles = entityManager.createQuery(
    "SELECT v FROM Vehicle v WHERE TYPE(v) IN (Car, Truck)", Vehicle.class
).getResultList();
```

生成的 SQL：

```sql
-- TYPE(v) = Car
SELECT ... FROM vehicle v WHERE v.vehicle_type = 'CAR';

-- TYPE(v) IN (Car, Truck)
SELECT ... FROM vehicle v WHERE v.vehicle_type IN ('CAR', 'TRUCK');
```

## 2.5 特殊注解

### @DiscriminatorFormula

`@DiscriminatorFormula` 允许使用 SQL 表达式来动态计算鉴别器值，而非直接存储固定字符串。适用于鉴别逻辑复杂的场景，例如根据某列的值范围来区分类型。

```java
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorFormula(
    "CASE " +
    "  WHEN seat_count IS NOT NULL THEN 'CAR' " +
    "  WHEN load_capacity IS NOT NULL THEN 'TRUCK' " +
    "  ELSE 'UNKNOWN' " +
    "END"
)
public abstract class Vehicle {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    // ...
}
```

使用 `@DiscriminatorFormula` 后，表中不需要额外的鉴别器列（但 Hibernate 生成的 SQL 中 WHERE 子句会使用该公式表达式）。优点是可以兼容遗留数据库；缺点是不直观、调试困难，且某些数据库对 CASE 表达式有限制。

> **注意**：`@DiscriminatorFormula` 与 `@DiscriminatorColumn` 是互斥的，二者不能同时使用。

### @DiscriminatorOptions

`@DiscriminatorOptions` 用于微调鉴别器列的行为，有两个属性：

**force = true**：强制在**所有查询中**都包含鉴别器列的过滤条件。即使查询的是父类（多态查询），也会添加 DTYPE 的 IN 条件。

```java
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "vehicle_type")
@DiscriminatorOptions(force = true)
public abstract class Vehicle { }
```

效果对比：

```sql
-- force = false（默认）：多态查询不添加 DTYPE 过滤
SELECT v.* FROM vehicle v WHERE v.price > 10000;

-- force = true：多态查询强制添加 DTYPE IN 条件
SELECT v.* FROM vehicle v
WHERE v.price > 10000 AND v.vehicle_type IN ('CAR', 'TRUCK');
```

适用场景：当表中混入了不属于当前继承层次的数据时（同一张表被多个继承体系共享），需要 `force = true` 来排除无关行。

**insert = false**：插入时不写入鉴别器列的值。此时鉴别器列的值完全依赖于 `@DiscriminatorFormula` 或数据库默认值。

```java
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "vehicle_type")
@DiscriminatorOptions(insert = false)
public abstract class Vehicle { }
```

插入时生成的 SQL 将**不包含** `vehicle_type` 列：

```sql
-- insert = false 时
INSERT INTO vehicle (manufacturer, price, seat_count, fuel_type)
VALUES ('Tesla', 29999.0, 5, 'ELECTRIC');
-- vehicle_type 列的值由数据库默认值或 @DiscriminatorFormula 决定
```

## 2.6 优缺点

### 优点

| 优点 | 说明 |
|------|------|
| **单表查询最快** | 无论是多态查询还是具体子类查询，都只需扫描一张表，无需 JOIN |
| **无需 JOIN** | 所有数据在同一张表中，避免多表关联的性能开销 |
| **简单直观** | 表结构简单，只需一张表，易于理解和维护 |
| **插入高效** | 插入操作只需一条 INSERT 语句，不需要跨表操作 |

### 缺点

| 缺点 | 说明 |
|------|------|
| **大量 NULL 列** | 子类不相关的字段在表中全为 NULL，浪费存储空间（尤其在列数很多时） |
| **违反数据库范式** | 非公共字段依赖于具体类型而非主键，违反第三范式（传递依赖） |
| **字段必须 NULLABLE** | 子类特有字段在数据库层面无法添加 NOT NULL 约束，数据完整性依赖应用层校验 |
| **宽表问题** | 子类越多、字段越多，表越宽，可能超过数据库单表列数限制（如 MySQL 最多 4096 列） |
| **索引效率下降** | 大量 NULL 值可能影响索引性能（特别是复合索引中包含子类特有字段时） |

### 适用场景

- 子类之间字段差异较小（2~5 个特有字段）
- 需要频繁进行多态查询
- 查询性能是首要考量
- 不需要在数据库层面对子类特有字段做 NOT NULL 约束
- 团队希望快速开发、表结构简单

---

# 3. JOINED（连接表策略）

## 3.1 策略说明

连接表策略是**最符合关系数据库范式设计**的继承映射策略。每个类（包括抽象父类）都映射为一张独立的表，子类表通过外键与父类表关联。

核心思想：父类表存储公共属性，子类表仅存储该子类特有的属性，所有子类表与父类表共享同一个主键值。

核心注解：

- **`@Inheritance(strategy = InheritanceType.JOINED)`**：标注在父类上。
- **`@DiscriminatorColumn`**：虽然是可选的，但 Hibernate 仍然默认生成 `DTYPE` 鉴别器列（在父类表中），用于标识行属于哪个子类。
- **`@PrimaryKeyJoinColumn`**：标注在子类上，用于自定义子类表与父类表关联的外键列名。默认子类表主键列名与父类表主键列名相同。

## 3.2 表结构

连接表策略生成的表结构：

- **父类表** `vehicle`：存储所有公共字段 + 鉴别器列 + 主键
- **子类表** `car`：主键（同时是引用 `vehicle` 的外键）+ Car 特有字段
- **子类表** `truck`：主键（同时是引用 `vehicle` 的外键）+ Truck 特有字段

**MySQL DDL**：

```sql
-- 父类表
CREATE TABLE vehicle (
    id             BIGINT       NOT NULL AUTO_INCREMENT,
    vehicle_type   VARCHAR(20)  NOT NULL,
    manufacturer   VARCHAR(255),
    price          DOUBLE,
    PRIMARY KEY (id)
);

-- 子类表 Car
CREATE TABLE car (
    id          BIGINT       NOT NULL,
    seat_count  INTEGER,
    fuel_type   VARCHAR(255),
    PRIMARY KEY (id),
    CONSTRAINT fk_car_vehicle FOREIGN KEY (id) REFERENCES vehicle(id)
);

-- 子类表 Truck
CREATE TABLE truck (
    id             BIGINT       NOT NULL,
    load_capacity  DOUBLE,
    axle_count     INTEGER,
    PRIMARY KEY (id),
    CONSTRAINT fk_truck_vehicle FOREIGN KEY (id) REFERENCES vehicle(id)
);
```

**PostgreSQL DDL**：

```sql
CREATE TABLE vehicle (
    id             BIGSERIAL    NOT NULL,
    vehicle_type   VARCHAR(20)  NOT NULL,
    manufacturer   VARCHAR(255),
    price          DOUBLE PRECISION,
    PRIMARY KEY (id)
);

CREATE TABLE car (
    id          BIGINT         NOT NULL,
    seat_count  INTEGER,
    fuel_type   VARCHAR(255),
    PRIMARY KEY (id),
    CONSTRAINT fk_car_vehicle FOREIGN KEY (id) REFERENCES vehicle(id)
);

CREATE TABLE truck (
    id             BIGINT         NOT NULL,
    load_capacity  DOUBLE PRECISION,
    axle_count     INTEGER,
    PRIMARY KEY (id),
    CONSTRAINT fk_truck_vehicle FOREIGN KEY (id) REFERENCES vehicle(id)
);
```

核心设计原则：
- 子类表的主键**同时是外键**，指向父类表主键
- 子类表不生成独立的 ID，而是复用父类表分配的 ID
- 子类特有字段可以有 `NOT NULL` 约束，满足数据库范式

## 3.3 完整示例

### 注解配置

```java
import javax.persistence.*;

@Entity
@Inheritance(strategy = InheritanceType.JOINED)
@DiscriminatorColumn(name = "vehicle_type", discriminatorType = DiscriminatorType.STRING)
@Table(name = "vehicle")
public abstract class Vehicle {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String manufacturer;

    private Double price;

    public Vehicle() {}

    public Vehicle(String manufacturer, Double price) {
        this.manufacturer = manufacturer;
        this.price = price;
    }

    // getters and setters（同单表策略，省略）
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getManufacturer() { return manufacturer; }
    public void setManufacturer(String manufacturer) { this.manufacturer = manufacturer; }
    public Double getPrice() { return price; }
    public void setPrice(Double price) { this.price = price; }
}
```

```java
import javax.persistence.*;

@Entity
@DiscriminatorValue("CAR")
@Table(name = "car")
@PrimaryKeyJoinColumn(name = "id")
public class Car extends Vehicle {

    @Column(nullable = false)
    private Integer seatCount;

    private String fuelType;

    public Car() {}

    public Car(String manufacturer, Double price, Integer seatCount, String fuelType) {
        super(manufacturer, price);
        this.seatCount = seatCount;
        this.fuelType = fuelType;
    }

    public Integer getSeatCount() { return seatCount; }
    public void setSeatCount(Integer seatCount) { this.seatCount = seatCount; }
    public String getFuelType() { return fuelType; }
    public void setFuelType(String fuelType) { this.fuelType = fuelType; }
}
```

```java
import javax.persistence.*;

@Entity
@DiscriminatorValue("TRUCK")
@Table(name = "truck")
@PrimaryKeyJoinColumn(name = "id")
public class Truck extends Vehicle {

    @Column(nullable = false)
    private Double loadCapacity;

    private Integer axleCount;

    public Truck() {}

    public Truck(String manufacturer, Double price, Double loadCapacity, Integer axleCount) {
        super(manufacturer, price);
        this.loadCapacity = loadCapacity;
        this.axleCount = axleCount;
    }

    public Double getLoadCapacity() { return loadCapacity; }
    public void setLoadCapacity(Double loadCapacity) { this.loadCapacity = loadCapacity; }
    public Integer getAxleCount() { return axleCount; }
    public void setAxleCount(Integer axleCount) { this.axleCount = axleCount; }
}
```

### 插入操作生成的 SQL

```java
Vehicle car = new Car("Tesla", 29999.0, 5, "ELECTRIC");
Vehicle truck = new Truck("Volvo", 89999.0, 18.5, 3);

entityManager.persist(car);
entityManager.persist(truck);
```

**Hibernate 生成的 SQL**（连接表策略下每次插入需要两条 INSERT 语句）：

```sql
-- ===== 插入 Car =====
-- 步骤 1：插入父类表
INSERT INTO vehicle (manufacturer, price, vehicle_type)
VALUES ('Tesla', 29999.0, 'CAR');

-- 步骤 2：使用父类生成的 ID 插入子类表
INSERT INTO car (id, seat_count, fuel_type)
VALUES (1, 5, 'ELECTRIC');
-- 注意：id = 1 来自 vehicle 表的 AUTO_INCREMENT

-- ===== 插入 Truck =====
-- 步骤 1：插入父类表
INSERT INTO vehicle (manufacturer, price, vehicle_type)
VALUES ('Volvo', 89999.0, 'TRUCK');

-- 步骤 2：使用父类生成的 ID 插入子类表
INSERT INTO truck (id, load_capacity, axle_count)
VALUES (2, 18.5, 3);
```

**表中实际数据**：

`vehicle` 表：

| id | vehicle_type | manufacturer | price |
|----|-------------|-------------|-------|
| 1  | CAR         | Tesla       | 29999 |
| 2  | TRUCK       | Volvo       | 89999 |

`car` 表：

| id（FK→vehicle） | seat_count | fuel_type |
|-------------------|------------|-----------|
| 1                 | 5          | ELECTRIC  |

`truck` 表：

| id（FK→vehicle） | load_capacity | axle_count |
|-------------------|---------------|------------|
| 2                 | 18.5          | 3          |

### 查询操作

**查询具体子类**（Car）：

```java
Car car = entityManager.find(Car.class, 1L);
```

```sql
-- Hibernate 生成 LEFT OUTER JOIN 关联父类表
SELECT c.id AS id,
       v.manufacturer AS manufacturer,
       v.price AS price,
       c.seat_count AS seat_count,
       c.fuel_type AS fuel_type,
       v.vehicle_type AS vehicle_type
FROM car c
LEFT OUTER JOIN vehicle v ON c.id = v.id
WHERE c.id = 1;
```

**多态查询**（查询所有 Vehicle）：

```java
List<Vehicle> vehicles = entityManager.createQuery(
    "SELECT v FROM Vehicle v", Vehicle.class
).getResultList();
```

```sql
-- 多态查询需要 LEFT OUTER JOIN 所有子类表
SELECT v.id, v.vehicle_type, v.manufacturer, v.price,
       c.seat_count, c.fuel_type,
       t.load_capacity, t.axle_count,
       CASE
           WHEN c.id IS NOT NULL THEN 1
           WHEN t.id IS NOT NULL THEN 2
           WHEN v.id IS NOT NULL THEN 0
       END AS clazz
FROM vehicle v
LEFT OUTER JOIN car c ON v.id = c.id
LEFT OUTER JOIN truck t ON v.id = t.id;
```

Hibernate 通过检查哪个子类表的 `id` 不为 NULL 来判断具体的实体类型。子类越多，JOIN 的表越多，查询性能下降越明显。

## 3.4 @PrimaryKeyJoinColumn

`@PrimaryKeyJoinColumn` 用于指定子类表中与父类表关联的外键列名。默认情况下，子类表主键列名与父类表主键列名相同。

```java
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
@Table(name = "vehicle")
public abstract class Vehicle {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "vehicle_id")
    private Long id;
    // ...
}

@Entity
@DiscriminatorValue("CAR")
@Table(name = "car")
@PrimaryKeyJoinColumn(name = "vehicle_id")
public class Car extends Vehicle {
    // vehicle_id 既是 car 表的主键，也是引用 vehicle 表的外键
    // ...
}
```

对应的 DDL：

```sql
CREATE TABLE vehicle (
    vehicle_id   BIGINT NOT NULL AUTO_INCREMENT,
    -- ...
    PRIMARY KEY (vehicle_id)
);

CREATE TABLE car (
    vehicle_id  BIGINT NOT NULL,
    seat_count  INTEGER,
    fuel_type   VARCHAR(255),
    PRIMARY KEY (vehicle_id),
    CONSTRAINT fk_car_vehicle FOREIGN KEY (vehicle_id) REFERENCES vehicle(vehicle_id)
);
```

也可以使用多个 `@PrimaryKeyJoinColumn`（复合主键场景）：

```java
@Entity
@Table(name = "car")
@PrimaryKeyJoinColumns({
    @PrimaryKeyJoinColumn(name = "vehicle_id", referencedColumnName = "vehicle_id"),
    @PrimaryKeyJoinColumn(name = "brand_code", referencedColumnName = "brand_code")
})
public class Car extends Vehicle { }
```

## 3.5 优缺点

### 优点

| 优点 | 说明 |
|------|------|
| **满足数据库范式** | 每张表结构清晰，无冗余字段，完全符合第三范式 |
| **无 NULL 列** | 子类特有字段可添加 NOT NULL 约束，数据库层面保证数据完整性 |
| **字段定义精确** | 每张子类表只包含属于自己的字段，不会出现"臃肿宽表" |
| **表结构易扩展** | 新增子类只需新建一张表，不影响已有的父子类表结构 |
| **适合列数极多的场景** | 避免单表列数超限的问题 |

### 缺点

| 缺点 | 说明 |
|------|------|
| **查询需 JOIN** | 查询任何子类都需要 JOIN 父类表，影响查询性能（尤其是深层继承） |
| **插入需多表操作** | 每次插入子类需要先插入父类表再插入子类表，两次 INSERT 在事务中执行 |
| **子类越多 JOIN 越多** | 多态查询需要 LEFT OUTER JOIN 所有子类表，N 个子类产生 N 个 JOIN |
| **深层继承性能差** | 三层继承（孙类）需要 JOIN 三张表才能读取完整数据 |
| **外键约束开销** | 外键约束在插入/删除时有一定性能开销 |

### 适用场景

- 子类之间字段差异大（10 个以上特有字段）
- 数据完整性和规范化要求高，必须满足第三范式
- 主要在具体子类层面进行查询，多态查询不频繁
- 继承层次较浅（不超过 2 层）
- DBA 对数据库表设计有严格的范式要求

---

# 4. TABLE_PER_CLASS（每类一表策略）

## 4.1 策略说明

每类一表策略为**每个具体实体类**（非抽象类）创建一张独立的表，每张表包含该实体类的**所有字段**（包括从父类继承的公共字段和自身特有字段）。父类如果是抽象类，则不会生成对应的表。

核心注解：

- **`@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)`**：标注在父类上。
- 该策略**不需要** `@DiscriminatorColumn` 和 `@DiscriminatorValue`，因为每张表天然对应一个具体类，无需鉴别器。

```java
@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class Vehicle {
    @Id
    @GeneratedValue(strategy = GenerationType.TABLE)  // 不能用 IDENTITY!
    private Long id;
    private String manufacturer;
    private Double price;
    // ...
}
```

## 4.2 表结构

每类一表策略下，每个具体子类生成一张包含**所有字段**的独立表。父类如果是 `abstract`，不生成表。

**MySQL DDL**：

```sql
-- 注意：父类 Vehicle 是 abstract，不生成表

-- Car 表（包含 Vehicle 的所有字段 + Car 特有字段）
CREATE TABLE car (
    id             BIGINT       NOT NULL,
    manufacturer   VARCHAR(255),
    price          DOUBLE,
    seat_count     INTEGER,
    fuel_type      VARCHAR(255),
    PRIMARY KEY (id)
);

-- Truck 表（包含 Vehicle 的所有字段 + Truck 特有字段）
CREATE TABLE truck (
    id             BIGINT       NOT NULL,
    manufacturer   VARCHAR(255),
    price          DOUBLE,
    load_capacity  DOUBLE,
    axle_count     INTEGER,
    PRIMARY KEY (id)
);
```

**PostgreSQL DDL**：

```sql
CREATE TABLE car (
    id             BIGINT         NOT NULL,
    manufacturer   VARCHAR(255),
    price          DOUBLE PRECISION,
    seat_count     INTEGER,
    fuel_type      VARCHAR(255),
    PRIMARY KEY (id)
);

CREATE TABLE truck (
    id             BIGINT         NOT NULL,
    manufacturer   VARCHAR(255),
    price          DOUBLE PRECISION,
    load_capacity  DOUBLE PRECISION,
    axle_count     INTEGER,
    PRIMARY KEY (id)
);
```

核心特征：
- 无鉴别器列
- 子类特有字段可以有 NOT NULL 约束（因为整行都属于该子类）
- 公共字段（如 `manufacturer`, `price`）在每张子类表中重复定义
- 如果父类不是抽象类，也会生成父类表（但极少使用）

## 4.3 完整示例

### 注解配置

```java
import javax.persistence.*;

@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class Vehicle {

    @Id
    @GeneratedValue(strategy = GenerationType.TABLE, generator = "vehicle_gen")
    @TableGenerator(
        name = "vehicle_gen",
        table = "hibernate_sequences",
        pkColumnName = "sequence_name",
        pkColumnValue = "vehicle",
        valueColumnName = "next_val",
        allocationSize = 10
    )
    private Long id;

    @Column(nullable = false)
    private String manufacturer;

    private Double price;

    public Vehicle() {}

    public Vehicle(String manufacturer, Double price) {
        this.manufacturer = manufacturer;
        this.price = price;
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getManufacturer() { return manufacturer; }
    public void setManufacturer(String manufacturer) { this.manufacturer = manufacturer; }
    public Double getPrice() { return price; }
    public void setPrice(Double price) { this.price = price; }
}
```

```java
import javax.persistence.*;

@Entity
@Table(name = "car")
public class Car extends Vehicle {

    private Integer seatCount;
    private String fuelType;

    public Car() {}

    public Car(String manufacturer, Double price, Integer seatCount, String fuelType) {
        super(manufacturer, price);
        this.seatCount = seatCount;
        this.fuelType = fuelType;
    }

    public Integer getSeatCount() { return seatCount; }
    public void setSeatCount(Integer seatCount) { this.seatCount = seatCount; }
    public String getFuelType() { return fuelType; }
    public void setFuelType(String fuelType) { this.fuelType = fuelType; }
}
```

```java
import javax.persistence.*;

@Entity
@Table(name = "truck")
public class Truck extends Vehicle {

    private Double loadCapacity;
    private Integer axleCount;

    public Truck() {}

    public Truck(String manufacturer, Double price, Double loadCapacity, Integer axleCount) {
        super(manufacturer, price);
        this.loadCapacity = loadCapacity;
        this.axleCount = axleCount;
    }

    public Double getLoadCapacity() { return loadCapacity; }
    public void setLoadCapacity(Double loadCapacity) { this.loadCapacity = loadCapacity; }
    public Integer getAxleCount() { return axleCount; }
    public void setAxleCount(Integer axleCount) { this.axleCount = axleCount; }
}
```

### 插入操作生成的 SQL

```java
Vehicle car = new Car("Tesla", 29999.0, 5, "ELECTRIC");
Vehicle truck = new Truck("Volvo", 89999.0, 18.5, 3);

entityManager.persist(car);
entityManager.persist(truck);
```

```sql
-- TABLE 生成器首先查询并更新序列值
SELECT next_val FROM hibernate_sequences WHERE sequence_name = 'vehicle' FOR UPDATE;
UPDATE hibernate_sequences SET next_val = next_val + 10 WHERE sequence_name = 'vehicle';

-- 插入 Car → 只操作 car 表
INSERT INTO car (id, manufacturer, price, seat_count, fuel_type)
VALUES (1, 'Tesla', 29999.0, 5, 'ELECTRIC');

-- 插入 Truck → 只操作 truck 表
INSERT INTO truck (id, manufacturer, price, load_capacity, axle_count)
VALUES (2, 'Volvo', 89999.0, 18.5, 3);
```

### 查询操作

**查询具体类（Car）→ 单表查询，效率最高**：

```java
Car car = entityManager.find(Car.class, 1L);
```

```sql
-- 只查 car 表，无 JOIN
SELECT c.id, c.manufacturer, c.price, c.seat_count, c.fuel_type
FROM car c
WHERE c.id = 1;
```

**多态查询（查询所有 Vehicle）→ UNION ALL**：

```java
List<Vehicle> vehicles = entityManager.createQuery(
    "SELECT v FROM Vehicle v", Vehicle.class
).getResultList();
```

```sql
-- 多态查询使用 UNION ALL 合并所有子类表
SELECT v.id, v.manufacturer, v.price,
       v.seat_count AS seat_count, v.fuel_type AS fuel_type,
       NULL AS load_capacity, NULL AS axle_count,
       1 AS clazz
FROM car v

UNION ALL

SELECT v.id, v.manufacturer, v.price,
       NULL AS seat_count, NULL AS fuel_type,
       v.load_capacity AS load_capacity, v.axle_count AS axle_count,
       2 AS clazz
FROM truck v;
```

多态查询时，Hibernate 需要扫描**所有子类表**并 UNION ALL，子类越多性能越差。`clazz` 列用于 Hibernate 内部判断每行数据应映射到哪个实体类。

## 4.4 主键策略限制

TABLE_PER_CLASS 策略对主键生成策略有严格限制：**不能使用 `GenerationType.IDENTITY`**。

原因解释：

```sql
-- 假设使用 IDENTITY
-- Car 表自增：id 从 1 开始 → 1, 2, 3, ...
-- Truck 表自增：id 也从 1 开始 → 1, 2, 3, ...

-- 多态查询时 UNION：
SELECT id FROM car
UNION ALL
SELECT id FROM truck;
-- 结果可能：1, 1, 2, 2, 3, 3 → ID 冲突！
```

Hibernate 通过 ID 值来唯一标识实体对象，UNION ALL 中出现重复 ID 会导致 Hibernate 一级缓存混乱、对象覆盖等问题。

**允许的生成策略**：

| 策略 | 是否可用 | 说明 |
|------|---------|------|
| `GenerationType.TABLE` | 可用 | 推荐：使用专门的序列表，全局唯一 ID |
| `GenerationType.SEQUENCE` | 可用 | 各子类共享同一数据库 SEQUENCE |
| `GenerationType.IDENTITY` | **不可用** | 每表独立自增，UNION 时 ID 冲突 |
| `GenerationType.AUTO` | 依赖实现 | Hibernate 会自动选择 TABLE 或 SEQUENCE |
| `GenerationType.UUID` | 可用 | UUID 天然全局唯一 |

**推荐的 TABLE 生成器配置**：

```java
@Id
@GeneratedValue(strategy = GenerationType.TABLE, generator = "vehicle_gen")
@TableGenerator(
    name = "vehicle_gen",
    table = "hibernate_sequences",
    pkColumnName = "sequence_name",
    pkColumnValue = "vehicle",
    valueColumnName = "next_val",
    initialValue = 1,
    allocationSize = 50   // 优化：减少序列表查询次数
)
private Long id;
```

`allocationSize = 50` 表示 Hibernate 一次从序列表中获取 50 个 ID 值缓存在内存中，减少对序列表的访问频率，大幅提升插入性能。

**PostgreSQL SEQUENCE 策略**（共享 SEQUENCE）：

```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "vehicle_seq")
@SequenceGenerator(
    name = "vehicle_seq",
    sequenceName = "vehicle_id_seq",
    initialValue = 1,
    allocationSize = 50
)
private Long id;
```

## 4.5 优缺点

### 优点

| 优点 | 说明 |
|------|------|
| **具体类查询极快** | 查询具体子类只扫描一张表，无 JOIN，无 DTYPE 过滤 |
| **无 NULL 列** | 每张表只包含自己的字段，可添加 NOT NULL 约束 |
| **表结构独立** | 每个子类表完全独立，可独立优化（索引、分区等） |
| **插入高效** | 插入只需一条 INSERT 语句，无跨表操作 |

### 缺点

| 缺点 | 说明 |
|------|------|
| **多态查询性能差** | 多态查询需 UNION ALL 所有子类表，子类越多越慢 |
| **数据重复** | 公共字段在每张子类表中重复定义，存储空间浪费 |
| **主键策略受限** | 不能使用 IDENTITY，必须使用 TABLE 或 SEQUENCE |
| **父类不能直接查询** | 父类若为抽象类没有对应表，无法 `SELECT * FROM vehicle` |
| **修改公共字段成本高** | 增加父类公共字段需要 ALTER 所有子类表 |
| **无法做跨类外键** | 其他表无法通过外键引用"所有 Vehicle"，只能引用具体子类表 |

### 适用场景

- 几乎不需要多态查询（绝大多数操作都是对具体子类的查询）
- 子类之间结构差异巨大（字段无交集，各自独立）
- 每个子类查询性能要求极高
- 继承层次浅，子类数量少（2~3 个）
- 数据量极大且子类查询模式完全不同（如不同子类数据分布在不同数据库实例）

---

# 5. @MappedSuperclass

## 5.1 策略说明

`@MappedSuperclass` **不是 JPA 继承映射策略**，而是一种**代码复用机制**。它将公共字段和映射信息定义在父类中，子类通过继承复用这些定义，但每个子类各自映射到独立的表。

与继承映射策略的根本区别：

| 特征 | @MappedSuperclass | @Entity 继承 |
|------|-------------------|-------------|
| 父类是实体吗？ | 否（不能标注 @Entity） | 是 |
| 父类有对应的表吗？ | 否（不映射为表） | 是 |
| 能查询父类吗？ | 否（`em.find(Base.class, id)` 不允许） | 是 |
| 能关联到父类吗？ | 否（`@ManyToOne` 目标不能是 @MappedSuperclass） | 是 |
| 本质 | 纯 Java 代码复用 | 对象/关系继承映射 |

```java
@MappedSuperclass
public abstract class BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @PrePersist
    protected void onCreate() {
        this.createdAt = LocalDateTime.now();
        this.updatedAt = LocalDateTime.now();
    }

    @PreUpdate
    protected void onUpdate() {
        this.updatedAt = LocalDateTime.now();
    }

    // getters and setters
}
```

## 5.2 完整示例

```java
// ===== 基类（不标 @Entity）=====
@MappedSuperclass
public abstract class BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @Version
    private Integer version;  // 乐观锁版本号

    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
        updatedAt = LocalDateTime.now();
    }

    @PreUpdate
    protected void onUpdate() {
        updatedAt = LocalDateTime.now();
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
    public Integer getVersion() { return version; }
}
```

```java
// ===== 子类 User =====
@Entity
@Table(name = "users")
public class User extends BaseEntity {

    @Column(nullable = false, unique = true, length = 50)
    private String username;

    @Column(nullable = false)
    private String password;

    private String email;

    public User() {}

    public User(String username, String password, String email) {
        this.username = username;
        this.password = password;
        this.email = email;
    }

    // getters and setters
}
```

```java
// ===== 子类 Order =====
@Entity
@Table(name = "orders")
public class Order extends BaseEntity {

    @Column(nullable = false)
    private String orderNumber;

    private BigDecimal totalAmount;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    public Order() {}

    // getters and setters
}
```

**生成的表结构**——各自独立：

```sql
-- users 表（拥有自己的所有字段，包括从 BaseEntity 继承的）
CREATE TABLE users (
    id          BIGINT       NOT NULL AUTO_INCREMENT,
    created_at  DATETIME     NOT NULL,
    updated_at  DATETIME,
    version     INTEGER,
    username    VARCHAR(50)  NOT NULL UNIQUE,
    password    VARCHAR(255) NOT NULL,
    email       VARCHAR(255),
    PRIMARY KEY (id)
);

-- orders 表（完全独立，也拥有从 BaseEntity 继承的字段）
CREATE TABLE orders (
    id           BIGINT         NOT NULL AUTO_INCREMENT,
    created_at   DATETIME       NOT NULL,
    updated_at   DATETIME,
    version      INTEGER,
    order_number VARCHAR(255)   NOT NULL,
    total_amount DECIMAL(19,2),
    status       VARCHAR(255),
    PRIMARY KEY (id)
);
```

**不能以 BaseEntity 作为关联目标**：

```java
// 编译通过，但运行时可能抛异常或行为不符合预期
// 因为 BaseEntity 不是 JPA 实体，JPA 无法建立正确的映射关系

// 错误示范：
// @ManyToOne
// private BaseEntity creator;  // 不能关联到 @MappedSuperclass
```

正确做法是定义接口或使用具体实体类型：

```java
// 方案 1：使用接口抽象
public interface HasTimestamps {
    LocalDateTime getCreatedAt();
    LocalDateTime getUpdatedAt();
}

@MappedSuperclass
public abstract class BaseEntity implements HasTimestamps { ... }

// 方案 2：关联到具体实体
@ManyToOne
private User creator;  // 具体实体类型
```

## 5.3 与 @Embeddable 的对比

`@MappedSuperclass` 和 `@Embeddable` 都是代码复用机制，但适用场景截然不同：

| 维度 | @MappedSuperclass | @Embeddable |
|------|-------------------|-------------|
| 关系类型 | "is-a"（继承） | "has-a"（组合） |
| 复用方式 | 子类 extends 父类 | 实体类 contains 组件 |
| 存在独立生命周期？ | 是（作为实体存在） | 否（依附于宿主实体） |
| 可独立查询？ | 是（子类独立查询） | 否（只能通过宿主实体访问） |
| 映射到表 | 子类各自独立表 | 嵌入到宿主实体的表（字段前缀/flat） |
| 典型场景 | 公共审计字段、公共 ID 策略 | Address、AuditInfo、Money（值对象） |

**@Embeddable 示例**：

```java
@Embeddable
public class Address {
    private String street;
    private String city;
    private String zipCode;

    public Address() {}
    public Address(String street, String city, String zipCode) {
        this.street = street;
        this.city = city;
        this.zipCode = zipCode;
    }
    // getters and setters
}

@Entity
public class Company {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "street", column = @Column(name = "office_street")),
        @AttributeOverride(name = "city", column = @Column(name = "office_city")),
        @AttributeOverride(name = "zipCode", column = @Column(name = "office_zip"))
    })
    private Address officeAddress;

    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "street", column = @Column(name = "warehouse_street")),
        @AttributeOverride(name = "city", column = @Column(name = "warehouse_city")),
        @AttributeOverride(name = "zipCode", column = @Column(name = "warehouse_zip"))
    })
    private Address warehouseAddress;
}
```

生成的 DDL（Address 的字段被 "压平" 到 company 表中）：

```sql
CREATE TABLE company (
    id               BIGINT NOT NULL AUTO_INCREMENT,
    name             VARCHAR(255),
    office_street    VARCHAR(255),
    office_city      VARCHAR(255),
    office_zip       VARCHAR(255),
    warehouse_street VARCHAR(255),
    warehouse_city   VARCHAR(255),
    warehouse_zip    VARCHAR(255),
    PRIMARY KEY (id)
);
```

**场景选择建议**：

- 需要共享审计字段、乐观锁、ID 策略 → `@MappedSuperclass`
- 需要在实体之间复用一组相关的值（地址、金额、坐标）→ `@Embeddable`
- "这是一个..."的关系 → `@MappedSuperclass`
- "它拥有一个..."的关系 → `@Embeddable`

## 5.4 @AttributeOverride / @AttributeOverrides

`@AttributeOverride` 允许子类覆盖从 `@MappedSuperclass` 父类继承的列映射，包括列名、类型、长度、是否可空等。

```java
@MappedSuperclass
public abstract class BaseEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
}

@Entity
@Table(name = "users")
@AttributeOverride(name = "id", column = @Column(name = "user_id"))
@AttributeOverride(name = "createdAt", column = @Column(name = "reg_time", nullable = false))
public class User extends BaseEntity {
    // User 表中的列为：user_id, reg_time, updated_at
    // ...
}

@Entity
@Table(name = "orders")
@AttributeOverride(name = "id", column = @Column(name = "order_id"))
@AttributeOverride(name = "updatedAt", column = @Column(name = "modify_time"))
public class Order extends BaseEntity {
    // Order 表中的列为：order_id, created_at, modify_time
    // ...
}
```

**使用 @AttributeOverrides 覆盖多个字段**：

```java
@Entity
@Table(name = "products")
@AttributeOverrides({
    @AttributeOverride(name = "id",
        column = @Column(name = "product_id")),
    @AttributeOverride(name = "createdAt",
        column = @Column(name = "listing_date", nullable = false, length = 3)),
    @AttributeOverride(name = "updatedAt",
        column = @Column(name = "last_modified"))
})
public class Product extends BaseEntity { ... }
```

**覆盖关联映射**（也可以覆盖 `@ManyToOne` 等关联注解）：

```java
@MappedSuperclass
public abstract class AuditableEntity {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "created_by")
    private User createdBy;
}

@Entity
@Table(name = "invoices")
@AssociationOverride(name = "createdBy",
    joinColumns = @JoinColumn(name = "invoice_creator_id"))
public class Invoice extends AuditableEntity { ... }
```

> **注意**：`@AttributeOverride` 也适用于 `@Entity` 继承（如 JOINED 策略中子类覆盖父类的列映射），但最常见的用途是在 `@MappedSuperclass` 场景中。

---

# 6. 混合继承策略

## 6.1 多层继承与混合策略

在多层继承体系中，不同层级可以混合使用不同的继承策略。例如：顶层使用 SINGLE_TABLE，中间层使用 JOINED。

```java
// ===== 第一层：使用 SINGLE_TABLE =====
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "payment_type", discriminatorType = DiscriminatorType.STRING)
@Table(name = "payment")
public abstract class Payment {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private BigDecimal amount;
    private LocalDateTime paymentDate;

    // getters, setters...
}

@Entity
@DiscriminatorValue("CARD")
public class CardPayment extends Payment {
    private String cardNumber;
    private String cardType;  // VISA, MASTERCARD
}

// ===== 第二层：CardPayment 下再使用 JOINED =====
@Entity
@DiscriminatorValue("CREDIT")
@Inheritance(strategy = InheritanceType.JOINED)
@Table(name = "credit_card_payment")
public class CreditCardPayment extends CardPayment {
    private Integer installmentMonths;  // 分期月数
}

@Entity
@DiscriminatorValue("DEBIT")
@Table(name = "debit_card_payment")
public class DebitCardPayment extends CardPayment {
    private String bankName;
}
```

**表结构分析**：

在这种混合策略下：

- `payment` 表（SINGLE_TABLE）：包含 `Payment`、`CardPayment`、`CreditCardPayment`、`DebitCardPayment` 所有字段（大量 NULL 列）
- `credit_card_payment` 表（JOINED 子类表）：仅包含 `installment_months` 字段，主键引用 `payment.id`
- `debit_card_payment` 表（JOINED 子类表）：仅包含 `bank_name` 字段，主键引用 `payment.id`

插入 `CreditCardPayment` 时的 SQL：

```sql
-- 1. 插入父类表（SINGLE_TABLE 的 payment 表）
INSERT INTO payment (amount, payment_date, payment_type,
                     card_number, card_type, installment_months)
VALUES (1000.00, '2024-01-15', 'CREDIT',
        '4111-1111-1111-1111', 'VISA', 12);

-- 2. 插入 JOINED 子类表
INSERT INTO credit_card_payment (id, installment_months)
VALUES (1, 12);
-- 注意：installment_months 在 payment 表和 credit_card_payment 表中都出现了（冗余）
```

> **警告**：混合策略在实际项目中极少使用，因为它会使表结构非常复杂，并可能导致 Hibernate 的元模型处理出现意外行为。大多数情况下，选择一个统一的策略是最佳实践。

通常在表格中实际展示时，可以为列添加对应的说明标注。

## 6.2 @Inheritance 的继承传递

`@Inheritance` 注解的传递规则：

- **只在直接超类上定义** `@Inheritance`：JPA 规定继承策略由继承层次中第一个标注 `@Entity` 且标注 `@Inheritance` 的类决定。
- **子类继承超类的继承策略**：如果子类没有显式标注 `@Inheritance`，将自动继承父类的策略。
- **避免在子类上重新声明**：在子类上重新标注 `@Inheritance` 可能导致不明确的行为，除非在混合策略场景下有意为之。

```java
// 正确：只在根实体上定义
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "type")
public abstract class Animal { ... }

@Entity
@DiscriminatorValue("MAMMAL")
public class Mammal extends Animal { ... }

@Entity
@DiscriminatorValue("DOG")
public class Dog extends Mammal { ... }
// Dog 自动使用 SINGLE_TABLE 策略

// 错误：在子类上重复定义 @Inheritance
@Entity
@Inheritance(strategy = InheritanceType.JOINED) // 不要这样做！
@DiscriminatorValue("MAMMAL")
public class Mammal extends Animal { ... }
// 这会导致 Hibernate 行为不确定，可能在 Animal 层使用 SINGLE_TABLE 又在 Mammal 层使用 JOINED
```

---

# 7. 继承策略对比与选型

## 7.1 综合对比表

| 维度 | SINGLE_TABLE | JOINED | TABLE_PER_CLASS |
|------|-------------|--------|-----------------|
| **查询性能（多态）** | ★★★★★ 单表全扫描 | ★★★ 多表 JOIN | ★ 多表 UNION ALL |
| **查询性能（具体类）** | ★★★★★ 单表 + DTYPE 过滤 | ★★★★ 两表 JOIN | ★★★★★ 单表扫描 |
| **插入性能** | ★★★★★ 单条 INSERT | ★★★ 两条 INSERT | ★★★★★ 单条 INSERT |
| **更新性能** | ★★★★★ 单表 UPDATE | ★★★★ 单表 UPDATE | ★★★★★ 单表 UPDATE |
| **数据完整性** | ★★★ 子类字段必须 NULLABLE | ★★★★★ 满足 3NF | ★★★★ 满足 3NF |
| **存储空间效率** | ★★★★★ 单表，无冗余 | ★★★★ 主键冗余 | ★★★ 公共列冗余 |
| **表数量** | ★★★★★ 仅 1 张 | ★★★ 1+N 张 | ★★ N 张 |
| **可维护性** | ★★★ 宽表难以维护 | ★★★★★ 每张表结构清晰 | ★★★ 公共列分散 |
| **可扩展性** | ★★★ 新增子类加列 | ★★★★★ 新增子类建新表 | ★★★★★ 新增子类建新表 |
| **范式合规** | 违反 3NF | 符合 3NF | 符合 3NF |
| **NOT NULL 约束** | 子类字段不可用 | 完全可用 | 完全可用 |
| **外键引用父类** | 可直接引用 | 可直接引用 | 不可（只能引用具体子类）|
| **主键策略限制** | 无限制 | 无限制 | 不能用 IDENTITY |
| **适用子类字段差异** | 小（2~5 个字段） | 大（10+ 字段） | 极大（字段几乎无交集） |

## 7.2 选型建议

### 决策流程图

```
                           ┌──────────────────────────────┐
                           │ 需要以父类类型进行多态查询？   │
                           └──────────────┬───────────────┘
                              是 ↓                    ↓ 否
                   ┌──────────────────────┐    ┌─────────────────────┐
                   │ 子类字段差异有多大？   │    │ 使用 @MappedSuperclass│
                   └──────┬───────────────┘    │ 纯代码复用，非继承映射 │
                    小 ↓            ↓ 大       └─────────────────────┘
           ┌──────────────┐  ┌──────────────┐
           │ SINGLE_TABLE │  │ 数据完整性要求│
           │ 性能最优      │  │ 是否严格？   │
           └──────────────┘  └──┬───────┬───┘
                            是 ↓       ↓ 否
                     ┌──────────┐  ┌──────────────────┐
                     │ JOINED   │  │ 多态查询是否频繁？ │
                     │ 符合范式  │  └──┬───────────┬───┘
                     └──────────┘  频繁 ↓           ↓ 极少
                              ┌──────────────┐  ┌──────────────────┐
                              │ SINGLE_TABLE │  │ TABLE_PER_CLASS  │
                              │ 或 JOINED    │  │ 具体类查询最快    │
                              └──────────────┘  └──────────────────┘
```

### 具体建议

**1. 大多数场景推荐 SINGLE_TABLE（性能最佳）**

- 子类数量少（2~5 个），子类特有字段不多（2~5 个）
- 需要频繁做多态查询和多态关联
- 数据库层面不需要强制 NOT NULL 约束（应用层保证即可）
- 这是 Spring Data JPA / Hibernate 的默认策略，生态最成熟

**2. 子类字段差异很大时选 JOINED**

- 子类有大量特有字段（10+），单表会极其臃肿
- 数据完整性要求严格，必须数据库层面 NOT NULL
- 主要在具体子类层面操作，多态查询较少
- 需要对不同子类做不同的数据库优化（如子类表独立分区）
- 团队有 DBA 参与，需要满足数据库范式规范

**3. 多态查询极少时可选 TABLE_PER_CLASS**

- 几乎不需要 `SELECT * FROM Vehicle` 这种多态查询
- 每个子类的查询模式、数据量、优化策略完全不同
- 主键可以使用 SEQUENCE 或 TABLE 策略
- 子类数量固定且不会频繁新增

**4. 代码复用选 @MappedSuperclass（不是继承映射）**

- 不需要以父类类型进行查询和关联
- 只是共享公共字段定义（如审计字段、ID 策略）
- 每个实体完全独立，不存在 "is-a" 的逻辑关系

**5. 遗留数据库 → 按实际表结构选策略**

- 现有表结构是单表多类型 → `SINGLE_TABLE`，用 `@DiscriminatorFormula` 适配
- 现有表结构是父表+子表 → `JOINED`，用 `@PrimaryKeyJoinColumn` 适配列名
- 现有表结构是每类一张表 → `TABLE_PER_CLASS`

## 7.3 Hibernate 实现细节

### 继承策略对应的 Persister 实现

Hibernate 内部为每种继承策略提供了不同的 `EntityPersister` 实现，负责实体的持久化操作：

| 策略 | Persister 实现类 | 职责 |
|------|-----------------|------|
| SINGLE_TABLE | `SingleTableEntityPersister` | 处理单表映射，负责鉴别器列的读写、子类字段的 NULL 处理 |
| JOINED | `JoinedSubclassEntityPersister` | 处理多表映射，负责父类表与子类表的 JOIN 查询和多表插入 |
| TABLE_PER_CLASS | `UnionSubclassEntityPersister` | 处理多表独立映射，负责 UNION ALL 查询和分表插入 |

### 各策略的加载方式

**SINGLE_TABLE 加载**：

```
Session.get(Vehicle.class, id)
  → SingleTableEntityPersister.load(id)
    → SELECT * FROM vehicle WHERE id = ?  -- 单表直接查询
    → 读取 DTYPE 列 → 确定具体子类 → 实例化对应 Class
```

**JOINED 加载**：

```
Session.get(Car.class, id)
  → JoinedSubclassEntityPersister.load(id)
    → SELECT v.*, c.*
      FROM vehicle v
      LEFT OUTER JOIN car c ON v.id = c.id
      WHERE v.id = ?           -- 至少一个 JOIN
```

**JOINED 多态加载**：

```
Session.get(Vehicle.class, id)  -- 父类多态加载
  → SELECT v.*, c.*, t.*
    FROM vehicle v
    LEFT OUTER JOIN car c ON v.id = c.id
    LEFT OUTER JOIN truck t ON v.id = t.id
    WHERE v.id = ?
  → 检查哪个子类表的 id 不为 NULL → 实例化对应的 Class
```

**TABLE_PER_CLASS 加载**：

```
Session.get(Car.class, id)
  → UnionSubclassEntityPersister.load(id)
    → SELECT * FROM car WHERE id = ?  -- 单表直接查询

Session.get(Vehicle.class, id)  -- 父类多态加载
  → UnionSubclassEntityPersister.load(id)
    → SELECT * FROM car WHERE id = ?
      UNION ALL
      SELECT * FROM truck WHERE id = ?
    -- Hibernate 逐个子类表查询，直到找到匹配的记录
```

### 鉴别器列的默认命名

Hibernate 鉴别器列的默认规则：

| 属性 | 默认值 | 说明 |
|------|--------|------|
| 列名 | `DTYPE` | 如不指定 `@DiscriminatorColumn(name=...)`，则使用此名称 |
| 列类型 | `DiscriminatorType.STRING` | 对应 DDL 类型 VARCHAR(31) |
| 长度 | 31 | VARCHAR(31)，足以容纳完整类名 |
| 值 | 实体类名 | 如 `@DiscriminatorValue` 未指定，使用简单类名（如 `"Car"`） |

Hibernate 6.x 变更（如果使用）：

- Hibernate 6 中鉴别器列的长度由 `@DiscriminatorColumn(length = ...)` 指定，默认值与 Hibernate 5 相同
- Hibernate 6 改进了鉴别器列的优化，在不需要时不生成额外的 SELECT

### 查询时的鉴别器处理

```java
// Hibernate 查询引擎处理步骤
// Step 1: 解析 JPQL "FROM Vehicle"
// Step 2: 检查 @Inheritance 策略
// Step 3: 如果是 SINGLE_TABLE 或 JOINED
//         → 在 WHERE 子句中自动添加 DTYPE 条件（针对具体子类查询）
// Step 4: 如果是 TABLE_PER_CLASS
//         → 生成 UNION ALL 查询
```

---

# 8. 继承查询技巧

## 8.1 TYPE() 表达式

JPQL 的 `TYPE()` 表达式用于在多态查询中基于实体类型进行过滤。它接受一个实体别名作为参数，返回该实体的实际类型。

```java
// 基本用法：筛选特定类型的实体
List<Vehicle> cars = entityManager.createQuery(
    "SELECT v FROM Vehicle v WHERE TYPE(v) = Car",
    Vehicle.class
).getResultList();

// IN 子句：筛选多种类型
List<Vehicle> vehicles = entityManager.createQuery(
    "SELECT v FROM Vehicle v WHERE TYPE(v) IN (Car, Truck)",
    Vehicle.class
).getResultList();

// 与 AND 组合多个条件
List<Vehicle> cheapCars = entityManager.createQuery(
    "SELECT v FROM Vehicle v WHERE TYPE(v) = Car AND v.price < 20000",
    Vehicle.class
).getResultList();
```

**不同策略生成的 SQL**：

| JPQL | SINGLE_TABLE | JOINED | TABLE_PER_CLASS |
|------|-------------|--------|-----------------|
| `TYPE(v) = Car` | `WHERE v.vehicle_type = 'CAR'` | `WHERE v.vehicle_type = 'CAR'` | `(SELECT ... FROM car WHERE ...)` |
| `TYPE(v) IN (Car, Truck)` | `WHERE v.vehicle_type IN ('CAR', 'TRUCK')` | `WHERE v.vehicle_type IN ('CAR', 'TRUCK')` | `UNION ALL` |

**注意事项**：

- 不能在 `SELECT` 子句中使用 `TYPE()`（如 `SELECT TYPE(v) FROM Vehicle v`），这需要注册自定义函数
- `TYPE()` 只能用于比较，不能获取实际的类型值

## 8.2 TREAT() 表达式（JPA 2.1+）

`TREAT()` 是 JPA 2.1 引入的**向下转型**（Downcasting）操作符，允许在多态查询中访问子类特有的属性。

```java
// 向下转型：在 Vehicle 查询中直接访问 Car 特有属性
List<Vehicle> fiveSeatCars = entityManager.createQuery(
    "SELECT v FROM Vehicle v WHERE TREAT(v AS Car).seatCount = 5",
    Vehicle.class
).getResultList();

// 在 SELECT 子句中使用 TREAT
List<Object[]> results = entityManager.createQuery(
    "SELECT v.manufacturer, TREAT(v AS Car).fuelType, TREAT(v AS Truck).loadCapacity FROM Vehicle v",
    Object[].class
).getResultList();

// 结合 JOIN 使用
List<Driver> drivers = entityManager.createQuery(
    "SELECT d FROM Driver d JOIN d.vehicle v WHERE TREAT(v AS Car).fuelType = 'ELECTRIC'",
    Driver.class
).getResultList();
```

**实际 SQL 行为**：

SINGLE_TABLE 策略下：

```sql
-- TREAT(v AS Car).seatCount = 5
SELECT v.* FROM vehicle v
WHERE v.vehicle_type = 'CAR' AND v.seat_count = 5;
```

JOINED 策略下：

```sql
-- TREAT(v AS Car).seatCount = 5
SELECT v.*, c.*
FROM vehicle v
LEFT OUTER JOIN car c ON v.id = c.id
WHERE v.vehicle_type = 'CAR' AND c.seat_count = 5;
```

TABLE_PER_CLASS 策略下：

```sql
-- TREAT(v AS Car).seatCount = 5
-- 实际上只查询 car 表，因为向下转型排除了其他子类
SELECT c.* FROM car c WHERE c.seat_count = 5;
```

**TREAT() 的限制**：

- 向下转型只能访问实际匹配的子类属性；如果实体不是 Car 类型，相关属性为 NULL
- 并非所有 JPA 提供者都完全支持 TREAT()，Hibernate 5.2+ 支持较好
- 在某些复杂关联场景中，TREAT() 可能导致意外的 SQL 生成

## 8.3 汇总查询与多态

### 按子类分组统计

```java
// 方案 1：使用 TYPE() 分组（推荐）
List<Object[]> stats = entityManager.createQuery(
    "SELECT TYPE(v), COUNT(v), AVG(v.price) " +
    "FROM Vehicle v " +
    "GROUP BY TYPE(v)",
    Object[].class
).getResultList();

for (Object[] row : stats) {
    System.out.println("Type: " + row[0] + ", Count: " + row[1] + ", Avg Price: " + row[2]);
}
```

SINGLE_TABLE 策略生成的 SQL：

```sql
SELECT v.vehicle_type, COUNT(v.id), AVG(v.price)
FROM vehicle v
GROUP BY v.vehicle_type;
```

JOINED 策略生成的 SQL：

```sql
SELECT v.vehicle_type, COUNT(v.id), AVG(v.price)
FROM vehicle v
GROUP BY v.vehicle_type;
-- JOINED 策略下按 DTYPE 分组即可，因为 vehicle 表已包含所有公共字段
```

TABLE_PER_CLASS 策略生成的 SQL：

```sql
-- 因为 TYPE() 无法直接转换为 SQL GROUP BY，Hibernate 使用 UNION ALL + 子查询
SELECT 'Car' AS type, COUNT(*), AVG(price) FROM car
UNION ALL
SELECT 'Truck' AS type, COUNT(*), AVG(price) FROM truck;
```

### 多态关联的查询行为

当实体与其他实体存在多态关联时，查询行为取决于关联目标使用的继承策略：

```java
@Entity
public class Driver {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "vehicle_id")
    private Vehicle vehicle;  // 多态关联：可以是 Car 或 Truck

    // getters and setters
}
```

```java
// 查询 Driver 及其 Vehicle 信息
List<Driver> drivers = entityManager.createQuery(
    "SELECT d FROM Driver d JOIN FETCH d.vehicle",
    Driver.class
).getResultList();
```

SINGLE_TABLE 策略下：

```sql
SELECT d.*, v.*
FROM driver d
INNER JOIN vehicle v ON d.vehicle_id = v.id;
-- 简单 INNER JOIN，Hibernate 根据 vehicle_type 列自动实例化 Car 或 Truck
```

JOINED 策略下：

```sql
SELECT d.*, v.*, c.*, t.*
FROM driver d
INNER JOIN vehicle v ON d.vehicle_id = v.id
LEFT OUTER JOIN car c ON v.id = c.id
LEFT OUTER JOIN truck t ON v.id = t.id;
-- 需要 LEFT OUTER JOIN 所有子类表来获取完整属性
```

TABLE_PER_CLASS 策略下：

```sql
-- Hibernate 必须对每个子类表分别查询
-- 因为外键引用的是"逻辑上的 Vehicle"，实际 ID 可能存在于 car 表或 truck 表中
-- 这通常是 TABLE_PER_CLASS 多态关联的最大痛点
```

> **性能提示**：如果使用 TABLE_PER_CLASS 策略且有大量多态关联查询，考虑切换到 SINGLE_TABLE 或 JOINED 策略。TABLE_PER_CLASS 在多态关联场景下性能最差。

---

# 附录：常见问题与陷阱

### Q1: 为什么 @MappedSuperclass 不能作为 @OneToMany 的目标？

因为 `@MappedSuperclass` 不是实体，没有对应的数据库表，也没有持久化标识。JPA 无法为其创建外键约束。应使用具体实体类型或接口。

### Q2: SINGLE_TABLE 中鉴别器列可以使用 INT 类型吗？

可以。将 `discriminatorType` 设置为 `DiscriminatorType.INTEGER`，对应的 `@DiscriminatorValue` 使用数值字符串：

```java
@DiscriminatorColumn(name = "vehicle_type", discriminatorType = DiscriminatorType.INTEGER)
// ...
@Entity
@DiscriminatorValue("1")
public class Car extends Vehicle { }

@Entity
@DiscriminatorValue("2")
public class Truck extends Vehicle { }
```

### Q3: 如何禁止父类实例化？

将父类声明为 `abstract`。这样 `new Vehicle()` 不合法，只能实例化具体子类。同时父类仍然可以使用 `@Entity` + `@Inheritance`。

### Q4: JOINED 策略可以避免多态查询时的全外连接吗？

不能完全避免，但可以通过以下方式优化：
- 使用 `@Polymorphism(type = PolymorphismType.EXPLICIT)` 限制多态范围（见 Hibernate 文档）
- 尽量查询具体子类而非父类
- 使用 DTO 投影只查询需要的字段，避免加载整个实体图

### Q5: TABLE_PER_CLASS 中可以使用 UUID 作主键吗？

可以，UUID 天然全局唯一，是 TABLE_PER_CLASS 策略的最佳主键类型之一：

```java
@Id
@GeneratedValue(strategy = GenerationType.UUID)
@Column(columnDefinition = "CHAR(36)")
private UUID id;
```

### Q6: Hibernate 6 在此有什么改进吗？

Hibernate 6（6.0+）在继承映射方面的重要改进：
- 改进了鉴别器列的自动检测和处理
- 对 TABLE_PER_CLASS 的 UNION 查询进行了优化（子查询消除）
- 更好地支持 Jakarta Persistence 3.0+ 规范
- 增强了 `TREAT()` 运算符的 SQL 生成
