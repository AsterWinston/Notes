Java 从 JDK 9 起进入了"**每半年一个版本**"的快速迭代节奏，语言层面的新语法（`var`、Switch 表达式、文本块、`Record`、密封类、模式匹配）密集地在这几个版本里落地。这篇按**版本顺序**把 JDK 8 之后值得关注的语言与 API 特性串一遍，重点是每条特性"**解决什么问题、边界在哪、有什么坑**"，最后单开一节讲 JDK 21 正式落地的**虚拟线程**——它可能是近年来对服务端编程影响最大的一项改动。`Record` 的完整展开见第 26 篇，`Optional` 见第 28 篇，`Stream` 见第 13 篇。

# 1. JDK 版本与 LTS 一览

## 1.1 版本时间线

从 JDK 9（2017 年 9 月）开始，Java 改为**每 6 个月发一个版本**（每年 3 月与 9 月），版本号也换成"功能版本 + 年份"的形式（如 JDK 21 对应 `21.0.x`）。绝大多数版本是**非 LTS 的过渡版本**，只维护到下一个版本发布；只有 **LTS（Long Term Support，长期支持）** 版本会获得多年补丁更新，是企业唯一会真正落地的目标。

| 版本 | 发布时间 | 是否 LTS | 最值得记住的特性 |
|------|---------|---------|----------------|
| JDK 8 | 2014-03 | **LTS** | Lambda、Stream、`Optional`、新日期时间 API、接口默认方法 |
| JDK 9 | 2017-09 | 否 | 模块系统 JPMS、`List.of`/`Set.of`/`Map.of`、接口私有方法、`ProcessHandle`、`Optional.stream` |
| JDK 10 | 2018-03 | 否 | **`var` 局部变量类型推断**、`List.copyOf`、`Optional.orElseThrow()` 无参版 |
| JDK 11 | 2018-09 | **LTS** | `HttpClient` 正式、`String` 增强（`isBlank`/`strip`/`lines`/`repeat`）、`Files.readString`/`writeString`、Lambda 参数可用 `var`、ZGC 实验性引入 |
| JDK 12 | 2019-03 | 否 | Switch 表达式（预览）、`Collectors.teeing` |
| JDK 13 | 2019-09 | 否 | 文本块（预览）、`String.formatted` 的前身 |
| JDK 14 | 2020-03 | 否 | **Switch 表达式正式**、`instanceof` 模式匹配（预览）、`Record`（预览）、更有用的 NPE 信息 |
| JDK 15 | 2020-09 | 否 | **文本块正式**、密封类（预览） |
| JDK 16 | 2021-03 | 否 | **`instanceof` 模式匹配正式**、**`Record` 正式**、`Stream.toList()` |
| JDK 17 | 2021-09 | **LTS** | **密封类正式**、强封装 JDK 内部、移除实验性 AOT 与 SecurityManager（废弃） |
| JDK 18~20 | 2022-03 ~ 2023-03 | 否 | 虚拟线程（预览）、Record 模式（预览）、switch 模式匹配（预览） |
| JDK 21 | 2023-09 | **LTS** | **虚拟线程正式**、**switch 模式匹配正式**、**Record 模式正式**、`SequencedCollection`、分代 ZGC |
| JDK 22~24 | 2024-03 ~ 2025-03 | 否 | 未命名变量 `_`、模式匹配的持续完善、结构化并发（预览） |
| JDK 25 | 2025-09 | **LTS** | 紧凑源文件与实例 `main` 方法、模块导入声明、作用域值（Scoped Values）正式、灵活构造器体 |

## 1.2 LTS 策略与选型建议

**LTS 的节奏**：JDK 8 → 11 → 17 间隔 3 年，从 JDK 17 起缩短为**每 2 年一个 LTS**（17 → 21 → 25）。企业通常只跟随 LTS 升级，跳过中间的所有非 LTS 版本。

| 现状 | 建议 |
|------|------|
| 还在 JDK 8 | 尽快升到 **JDK 17 或 21**；8 已停止免费商用更新，且缺少 `var`、文本块、`Record` 等大幅提效的语法 |
| 正在 JDK 11 | 升 **JDK 21**，收益最大的是虚拟线程 + 模式匹配 |
| 已用 JDK 17 | 升 **JDK 21** 主要是为了虚拟线程与 `SequencedCollection`；升 **JDK 25** 可得到更完整的语言收尾 |
| 新项目 | 直接选 **最新的 LTS（JDK 25）**，或至少 JDK 21 |

> **行业实践**：升级前务必确认依赖生态（Spring Boot、Lombok、字节码增强框架、各类 agent）是否已支持目标版本。**Lombok 和 ASM 相关的库通常滞后**，这是实际升级时最常见的拦路虎。

# 2. var — 局部变量类型推断（JDK 10）

`var` 让编译器根据右侧表达式**自动推断**左侧变量的类型，省掉重复的类型书写。注意它是**编译期的语法糖**，字节码里变量的类型依然是那个具体类型，**不影响运行期性能**。

## 2.1 基本用法

```java
public class VarDemo {
    public static void main(String[] args) {
        // 编译器根据右侧推断：String
        var str = "Hello";
        // 推断为 ArrayList<String>，右侧的泛型参数依然要写
        var list = new ArrayList<String>();
        // 推断为 Path
        var path = Path.of("/tmp");

        System.out.println(str.getClass().getSimpleName());     // String
        System.out.println(list.getClass().getSimpleName());    // ArrayList
        System.out.println(path.getClass().getSimpleName());    // UnixPath 或 WindowsPath

        // 在 for 循环中同样可用（包括增强 for 和 try-with-resources）
        for (var i = 0; i < 3; i++) System.out.print(i + " ");  // 0 1 2
        System.out.println();

        var map = new HashMap<String, List<Integer>>();          // 类型名很长的经典受益场景
        map.put("a", List.of(1, 2, 3));
        for (var entry : map.entrySet()) {
            System.out.println(entry.getKey() + " -> " + entry.getValue());   // a -> [1, 2, 3]
        }
    }
}
```

## 2.2 适用限制

`var` 的使用范围被严格限制，下面这些**全部编译不通过**：

| 限制 | 反例 | 原因 |
|------|------|------|
| 只能用于**局部变量** | `private var name;` | 字段类型是类签名的一部分，必须显式 |
| 不能用于**方法参数** | `void f(var x)` | 参数类型同样是签名的一部分 |
| 不能用于**方法返回值** | `var f() { ... }` | 同上，会破坏 API 的稳定性 |
| 必须**声明时初始化** | `var x;` | 没有右侧表达式就无从推断 |
| 不能初始化为 **`null`** | `var x = null;` | `null` 没有类型信息，无法推断 |
| 不能用于**数组初始化简写** | `var a = {1, 2, 3};` | 必须写成 `var a = new int[]{1, 2, 3};` |
| 不能用于 **Lambda 表达式** | `var f = x -> x + 1;` | Lambda 需要目标类型才能确定函数式接口 |
| **Lambda 参数**加类型才能用 | `(var a, var b) -> a + b` | 单个参数用 `var` 不合法，**必须每个参数都写 `var` 并加括号** |
| 匿名类**拿不到额外方法** | `var obj = new Object() { void f() {} };` | `var` 推断为匿名类，但官方规定**不能通过 `var` 变量调用匿名类新增的方法** |

```java
// Lambda 参数的 var：必须所有参数都写 var，且必须有括号
list.forEach((var s) -> System.out.println(s));          // ✓ 单个参数也要括号
// list.forEach(var s -> System.out.println(s));         // ✗ 编译错误：少了括号
// BiFunction<Integer, Integer, Integer> f = (var a, int b) -> a + b;  // ✗ 不能混用
```

## 2.3 适用场景与注意事项

**推荐用 `var` 的场景**：

```java
// 1. 类型名冗长、右侧已经一目了然
var map = new HashMap<String, List<Map<String, Integer>>>();
var users = userRepository.findAllByStatus(Status.ACTIVE);

// 2. try-with-resources：类型对读者并不重要
try (var reader = new BufferedReader(new FileReader("file.txt"))) {
    System.out.println(reader.readLine());
}

// 3. 增强 for：泛型类型又长又啰嗦
for (var entry : configMap.entrySet()) { /* ... */ }
```

**不建议用 `var` 的场景**：

```java
// ✗ 右侧是方法调用，光看这行看不出 getResult() 返回什么
var result = service.getResult();
// ✓ 显式写出类型，读者不用跳转源码
ProcessResult result = service.getResult();

// ✗ 数值字面量：推断成 int 还是 long？容易踩坑
var count = 0;              // 推断为 int
var big = 10_000_000_000L;  // 有 L 后缀才是 long
```

> **注意**：`var` **不是关键字**，而是一个"**保留的类型名**"。所以你仍然可以把变量、方法命名为 `var`（`var var = 1;` 是合法的，只是极其不建议）。另外，**`var` 与 `final` 可以组合**：`final var list = List.of(1, 2);` 会推断成 `final List<Integer>`。

# 3. Switch 表达式（JDK 12 预览 → JDK 14 正式）

传统 `switch` 有两个老毛病：**忘记 `break` 会穿透（fall-through）**，以及**它是一个语句而不是表达式，没法直接赋值**。Switch 表达式一次解决这两点。

## 3.1 箭头语法：无穿透、可直接返回

```java
public class SwitchDemo {
    public static void main(String[] args) {
        int day = 6;

        // 传统写法：要声明临时变量，要自己写 break，忘了就穿透
        String oldType;
        switch (day) {
            case 1:
            case 2:
            case 3:
            case 4:
            case 5:
                oldType = "工作日";
                break;                  // 忘写这行，case 6/7 也会执行到
            case 6:
            case 7:
                oldType = "周末";
                break;
            default:
                oldType = "未知";
        }

        // Switch 表达式：多个 case 用逗号合并，箭头右侧就是结果值
        String newType = switch (day) {
            case 1, 2, 3, 4, 5 -> "工作日";
            case 6, 7 -> "周末";
            default -> "未知";
        };

        System.out.println(oldType);    // 周末
        System.out.println(newType);    // 周末
    }
}
```

箭头形式里**每个分支互不干扰，不可能发生穿透**——因为 `->` 右边的部分本身就是一个完整的结果，没有"掉到下一个 case"的概念。

## 3.2 块语句与 yield

分支里要做多件事时用块语句，块里用 **`yield`** 返回值（**不是 `return`**，`return` 会把整个方法返回掉）：

```java
String type = switch (day) {
    case 1, 2, 3, 4, 5 -> {
        System.out.println("今天是工作日");   // 块里可以先做别的事
        yield "工作日";                      // 用 yield 交出这个分支的值
    }
    case 6, 7 -> "周末";
    default -> "未知";
};
```

`yield` 与 `return` 的区别必须记牢：

| 关键字 | 作用范围 |
|--------|---------|
| `yield` | **只结束当前 switch 分支**，把值交回给 switch 表达式 |
| `return` | **结束整个方法**，switch 表达式本身得不到结果 |

## 3.3 传统冒号语法也支持 yield

老的冒号（`:`）语法在 Switch 表达式里依然可用，只是所有分支都必须用 `yield` 交出值，**不能再靠 `break` 携带值**：

```java
String type = switch (day) {
    case 1: case 2: case 3: case 4: case 5:
        yield "工作日";
    case 6: case 7:
        yield "周末";
    default:
        yield "未知";
};
```

> **注意**：冒号形式在 Switch 表达式中**不允许穿透**（缺 `yield` 会编译报错），这一点和传统 `switch` 语句不同，别再按老习惯去"省略 break"。

## 3.4 穷尽性检查

**Switch 表达式必须有值**，所以编译器会做**穷尽性检查（exhaustiveness）**——所有可能的情况都必须被覆盖，否则编译不通过。有三种方式满足它：

| 被 switch 的类型 | 满足穷尽性的方式 |
|-----------------|----------------|
| 枚举 | 列出**全部**枚举常量，或写一个 `default` |
| `sealed` 密封类型 | 覆盖**全部** `permits` 的子类，此时**不需要 `default`**（见第 7、8 节） |
| 其他（`int`、`String`、普通类） | **必须**写 `default` |

```java
enum Status { ACTIVE, INACTIVE, DELETED }

// ✓ 枚举列全，不需要 default；将来新增枚举常量时这里会编译报错，等于多了一道保护
String label = switch (status) {
    case ACTIVE   -> "启用";
    case INACTIVE -> "停用";
    case DELETED  -> "已删除";
};

// ✗ 漏了 DELETED，编译报错：the switch expression does not cover all possible input values
// String bad = switch (status) {
//     case ACTIVE -> "启用";
//     case INACTIVE -> "停用";
// };
```

> **最佳实践**：**枚举场景下宁可列全也不要写 `default`**。因为一旦有人给枚举加了新常量，写全的 switch 会**编译失败**从而提醒你处理；而写了 `default` 的 switch 会**静默地走进默认分支**，把新情况悄悄吞掉——这正是很多"新增枚举值后行为异常"线上事故的根源。

# 4. Text Blocks — 文本块（JDK 13 预览 → JDK 15 正式）

文本块用三个双引号 `"""` 包裹多行字符串，解决"SQL、JSON、HTML 拼字符串时到处是 `\n` 和 `\"`"的痛点。

## 4.1 基本语法与自动缩进处理

```java
public class TextBlockDemo {
    public static void main(String[] args) {
        String html = """
                <html>
                    <body>
                        <p>Hello, World!</p>
                    </body>
                </html>
                """;
        System.out.println(html);
    }
}
```

输出的每一行**都没有保留源码里的那段左缩进**。规则是：

| 规则 | 说明 |
|------|------|
| 起始定界符 | `"""` 后**必须换行**，否则编译错误 |
| 公共缩进 | 编译器自动去除所有行（含结束定界符那一行）的**最小公共缩进** |
| 结束定界符 | `"""` 的位置**参与缩进计算**，想保留缩进就把它往左移 |
| 换行符 | 每行末尾自动补 `\n`（Windows 上编译出来的也是 `\n`，不是 `\r\n`） |

```java
// 结束定界符右移 → 计算出的公共缩进更小 → 输出保留更多缩进
String indented = """
        line1
        line2
            """;
System.out.println(indented);     // "line1\nline2\n"（缩进被算作公共部分去掉了）

// 结束定界符对齐到最左 → 整体缩进被保留
String preserved = """
            line1
            line2
""";
System.out.println(preserved);    // "    line1\n    line2\n"
```

## 4.2 三个转义技巧

**① `\` 行尾续行：把多行拼成一行**

```java
// 长 SQL 拆成多行写，实际结果是一整行，不产生多余换行和空格
String sql = """
        SELECT id, name, age \
        FROM users \
        WHERE age > 18 \
        ORDER BY id
        """;
System.out.println(sql);
// SELECT id, name, age FROM users WHERE age > 18 ORDER BY id
```

**② `\s` 保留尾部空格**

编译器和很多编辑器的格式化工具都会**自动删掉行尾空格**，导致文本块里的对齐（比如表格、ASCII 图）失效。写一个 `\s` 可以强制保留它前面那个空格：

```java
String table = """
        name    \s
        Tom     \s
        """;
System.out.println(table.replace("\n", "\\n"));   // "name    \nTom     \n"
// 每行末尾的空格被 \s 保住了，不会被编辑器或编译器吃掉
```

**③ `\` 转义特殊字符**

文本块只是"不用转义引号"，`\n`、`\t`、`\\` 这些依然是转义序列；而三个连续引号 `"""` 在块内**必须至少转义一个**，否则会被当成结束定界符。

```java
String s = """
        Tab\t制表
        反斜杠\\就是反斜杠
        这里有三个引号: \""" 结束了吗？没有
        """;
```

## 4.3 内部引号无需转义

写 JSON、HTML 时最爽的一点——**双引号不用转义**（除非连续出现三个）：

```java
String json = """
        {
            "name": "Tom",
            "age": 25,
            "tags": ["java", "jdk"]
        }
        """;
System.out.println(json);
```

## 4.4 变量内插：用 formatted 或 String.format

文本块本身**不做变量插值**（Java 没有 Kotlin 那样的 `$name`，直到 JDK 21 引入模板表达式预览、后来又被撤回设计）。要在文本块里填变量，标准做法是 `formatted`：

```java
public class FormatDemo {
    public static void main(String[] args) {
        String name = "Tom";
        int age = 25;

        // String.formatted(Object...) 是 JDK 15+ 的实例方法，等价于 String.format(this, args)
        String report = """
                ========== 用户信息 ==========
                姓名: %s
                年龄: %d
                是否成年: %s
                ==============================
                """.formatted(name, age, age >= 18 ? "是" : "否");

        System.out.println(report);
    }
}
```

```text
========== 用户信息 ==========
姓名: Tom
年龄: 25
是否成年: 是
==============================
```

> **最佳实践**：文本块最适合**写死的多行常量**——SQL、JSON 模板、HTML 片段、正则表达式、单元测试里的期望输出。一旦需要在里面拼大量动态变量，说明它已经不是"常量"了，应该改用模板引擎（如 Freemarker、Thymeleaf）或至少用 `formatted` 而非 `+` 拼接。

# 5. instanceof 模式匹配（JDK 14 预览 → JDK 16 正式）

传统 `instanceof` 判断之后必须再强转一次，类型和变量名写两遍，还容易改一处忘一处：

```java
// 传统写法：判断一次、强转一次
if (obj instanceof String) {
    String s = (String) obj;          // 这行强转是纯噪音，出错了还是运行时 ClassCastException
    System.out.println(s.length());
}

// 模式匹配写法：判断通过后直接得到绑定变量
if (obj instanceof String s) {
    System.out.println(s.length());   // s 已经声明好了，直接用
}
```

## 5.1 作用域规则

绑定变量 `s` 的**作用域是"编译器能证明匹配成功"的范围**：

```java
// ✓ && 右侧可用：走到这里说明左边的 instanceof 已经为 true
if (obj instanceof String s && s.length() > 5) {
    System.out.println("长字符串: " + s);
}

// ✗ || 右侧不可用：短路求值时可能压根没做本次判断
// if (obj instanceof String s || s.length() > 5) { }   // 编译错误

// ✓ 提前 return 的"护卫语句"写法：判断为假就返回，后面 s 全程可用
if (!(obj instanceof String s)) {
    return;
}
System.out.println(s);   // 此处 s 可用

// ✓ 用 isPresent 风格的三元表达式也可以
String result = obj instanceof String s ? s.toUpperCase() : "";
```

> **注意**：模式匹配改变的只是**写法**，没有改变 `instanceof` 的语义——它依然**不包含 `null` 检查的便利**：`null instanceof String` 永远是 `false`，所以模式匹配里传入 `null` 会安全地走 `else` 分支，这一点比直接强转更安全。

# 6. Record（JDK 14 预览 → JDK 16 正式）

`Record` 是"不可变数据载体类"的语法糖，编译器自动生成全参构造器、getter、`equals`、`hashCode`、`toString`。

```java
public record Point(int x, int y) {}

// 使用
Point p = new Point(1, 2);
System.out.println(p);              // Point[x=1, y=2]（自动的 toString）
System.out.println(p.x() + "," + p.y());   // 1,2（访问器名就是组件名，没有 get 前缀）
System.out.println(new Point(1, 2).equals(p));   // true（自动的 equals）
```

> **注意**：`Record` 的**完整讲解（紧凑构造器、静态字段与实例字段的区别、`Record` 与 `record` 关键字的限制、与枚举 / Lombok `@Data` 的对比）在第 26 篇**，本节只保留它在新特性脉络里的位置。这里只需记住一条对下一节至关重要的性质：**`Record` 的组件是可以通过模式匹配直接"解构"出来的**。

# 7. Sealed Classes — 密封类（JDK 15 预览 → JDK 17 正式）

**密封类用于精确控制"谁能继承我"**——在此之前 Java 只有"完全开放"（任何类都能继承）和"完全封闭"（`final`，谁都不能继承）两个极端，密封类补上了中间态。

## 7.1 语法与三种子类修饰符

```java
// 用 permits 明确列出允许继承的子类（子类必须与父类在同一模块/包内）
public sealed class Shape permits Circle, Rectangle, Triangle { }

public final class Circle extends Shape { }              // final：到此为止，不能再被继承
public final class Rectangle extends Shape { }           // final
public non-sealed class Triangle extends Shape { }       // non-sealed：重新打开，可被任意类继承
```

| 子类修饰符 | 含义 |
|-----------|------|
| `final` | 子类**不能再被继承**，继承链到此结束 |
| `sealed` | 子类**继续受控**，必须自己再写 `permits` 列出下一层 |
| `non-sealed` | **打破密封**，该子类可以被任意类继承 |

**注意：被 `permits` 列出的子类必须显式声明这三种修饰符之一**，不能什么都不写，否则编译报错。

## 7.2 密封接口与更典型的用法

密封接口是更常见的形态——因为它可以和 `Record` 组合，做出"代数数据类型"式的建模：

```java
public sealed interface JsonValue permits JsonNumber, JsonString, JsonObject { }

public record JsonNumber(double value) implements JsonValue { }   // 叶子节点用 Record
public record JsonString(String value) implements JsonValue { }
public final class JsonObject implements JsonValue { }
```

## 7.3 与模式匹配配合：穷尽性

密封类真正的价值在**配合 switch 模式匹配**时才完全体现：**编译器知道所有子类，因此能检查 switch 是否穷尽**（见 8.4），从而**不需要 `default`**。这带来一个巨大的好处——将来新增一个子类，所有漏处理的 switch 都会**编译报错**，而不是在运行期静默走入 `default`。

```java
String describe(Shape shape) {
    return switch (shape) {                       // Shape 是 sealed，列出全部子类即穷尽
        case Circle c    -> "圆形，半径: " + c.radius();
        case Rectangle r -> "矩形，宽高: " + r.width() + "x" + r.height();
        case Triangle t  -> "三角形";              // 不需要 default
    };
}
```

> **行业实践**：密封类的典型落地场景是**领域模型的建模**（支付方式、订单状态、事件类型）和**表达式树 / AST**。它的副作用是"**封闭了扩展点**"——第三方无法再实现你的接口。所以库作者要谨慎使用：**面向外部扩展的 API 不要密封，面向内部收敛的类型非常适合密封**。

# 8. Record 模式匹配与 Switch 模式匹配（JDK 17 预览 → JDK 21 正式）

## 8.1 Record 模式：直接解构

`Record` 的组件可以**在模式里直接拆出来**，省掉一次访问器调用：

```java
record Point(int x, int y) { }

Object obj = new Point(3, 4);

// 传统写法：先判断类型，再用访问器逐个取值
if (obj instanceof Point p) {
    System.out.println(p.x() + ", " + p.y());
}

// Record 模式（JDK 21+）：在模式里直接声明组件的绑定变量
if (obj instanceof Point(int x, int y)) {
    System.out.println(x + ", " + y);       // 3, 4
}
```

## 8.2 嵌套 Record 模式

模式可以**递归嵌套**，一次就把深层结构拆开——这是处理树形数据最优雅的写法：

```java
record Point(int x, int y) { }
record Rectangle(Point topLeft, Point bottomRight) { }

Object obj = new Rectangle(new Point(0, 0), new Point(10, 20));

if (obj instanceof Rectangle(Point(int x1, int y1), Point(int x2, int y2))) {
    System.out.println("矩形: (" + x1 + "," + y1 + ") - (" + x2 + "," + y2 + ")");
    // 矩形: (0,0) - (10,20)
}
```

## 8.3 switch 模式匹配

`switch` 从"只能按常量匹配"升级为"**可以按类型和结构匹配**"：

```java
public class SwitchPatternDemo {
    sealed interface Shape permits Circle, Rect { }
    record Circle(double radius) implements Shape { }
    record Rect(double w, double h) implements Shape { }

    public static void main(String[] args) {
        Object[] objs = { null, "hello", 42, new int[]{1, 2, 3}, new Circle(2.0), new Rect(3.0, 4.0) };

        for (Object obj : objs) {
            System.out.println(describe(obj));
        }
    }

    static String describe(Object obj) {
        return switch (obj) {
            case null               -> "null";                     // JDK 21 起 case null 合法
            case String s           -> "字符串: " + s;
            case Integer i         -> "整数: " + i;
            case int[] arr         -> "整型数组，长度: " + arr.length;
            case Circle(double r)  -> "圆，半径: " + r;             // Record 模式解构
            case Rect(double w, double h) -> "矩形，面积: " + (w * h);
            default                 -> "未知类型: " + obj.getClass().getSimpleName();
        };
    }
}
```

```text
null
字符串: hello
整数: 42
整型数组，长度: 3
圆，半径: 2.0
矩形，面积: 12.0
```

> **注意**：JDK 21 之前 `switch` 遇 `null` 直接抛 `NullPointerException`，现在可以用 `case null` 显式接住。如果不写 `case null`，`null` 依然会抛 NPE。

## 8.4 守卫条件 when（JDK 21）

`case` 后面可以再加一个 `when` 守卫，让模式匹配带上额外条件：

```java
static String classify(Object obj) {
    return switch (obj) {
        case String s when s.length() > 10 -> "长字符串: " + s;
        case String s                      -> "短字符串: " + s;   // 注意顺序：更具体的放前面
        case Integer i when i < 0          -> "负整数: " + i;
        case Integer i                      -> "非负整数: " + i;
        default                             -> "其他";
    };
}
```

**顺序很重要**：`switch` 从上往下匹配，**更具体的 `when` 条件必须写在更宽泛的模式之前**，否则后面的分支永远不可达——编译器对这种情况会报错（"this case label is dominated by a preceding case label"）。

## 8.5 穷尽性要求

| 被匹配的类型 | 穷尽性怎么满足 |
|-------------|--------------|
| `sealed` 类型 | 覆盖**全部** `permits` 子类即可，**无需 `default`** |
| 枚举 | 覆盖全部常量，或写 `default` |
| 其他任意类型 | **必须**写 `default`（否则编译报错） |

> **最佳实践**：**能用密封类型就用密封类型**，因为"没有 `default` 的穷尽 switch"能得到编译器最强的保护——**新增子类时编译失败**，逼着你把所有分支处理完。这是 Java 在"可扩展性"和"编译期安全"之间找到的一个非常好的平衡点，也是模式匹配这套特性真正的价值所在。

# 9. 不可变集合的便捷创建（JDK 9+）

## 9.1 List.of / Set.of / Map.of / Map.ofEntries

```java
import java.util.*;

List<String> list = List.of("a", "b", "c");
Set<Integer> set = Set.of(1, 2, 3);
Map<String, Integer> map = Map.of("a", 1, "b", 2, "c", 3);   // 键值对交替传入

// Map.of 最多 10 个键值对（因为它是重载而非可变参数），超过就用 Map.ofEntries
Map<String, Integer> big = Map.ofEntries(
        Map.entry("a", 1),
        Map.entry("b", 2),
        Map.entry("c", 3),
        Map.entry("d", 4),
        Map.entry("e", 5),
        Map.entry("f", 6),
        Map.entry("g", 7),
        Map.entry("h", 8),
        Map.entry("i", 9),
        Map.entry("j", 10),
        Map.entry("k", 11)          // 超过 10 个也没问题
);
```

## 9.2 copyOf：不可变副本（JDK 10+）

```java
List<String> mutable = new ArrayList<>(List.of("a", "b"));

List<String> copy = List.copyOf(mutable);      // JDK 10+，返回不可变副本
mutable.add("c");                              // 改原集合
System.out.println(copy);                      // [a, b] —— 副本不受影响

Set<String> setCopy = Set.copyOf(mutable);     // Set.copyOf 会去重
Map<String, Integer> mapCopy = Map.copyOf(map);
```

> **注意**：`List.copyOf(collection)` 如果传入的**已经是一个不可变集合，会直接返回它本身**（不复制），这是一个值得知道的小优化。

## 9.3 三个必须记住的坑

| 坑 | 表现 |
|----|------|
| **不接受 `null` 元素 / 键 / 值** | `List.of("a", null)` 立即抛 `NullPointerException`；`Map.of("k", null)` 同样抛 |
| **完全不可变** | 任何修改（`add`/`set`/`remove`/`put`）都抛 `UnsupportedOperationException`，包括 `sort`、`replaceAll` |
| **`Set.of` 不接受重复元素** | `Set.of(1, 1)` 抛 `IllegalArgumentException: duplicate element`（注意**不是**静默去重） |

```java
try {
    List.of("a", null);
} catch (NullPointerException e) {
    System.out.println("List.of 不接受 null 元素");     // 会打印
}

try {
    Set.of(1, 1);
} catch (IllegalArgumentException e) {
    System.out.println("Set.of 不接受重复元素");        // 会打印
}

List<String> immutable = List.of("a", "b");
try {
    immutable.add("c");
} catch (UnsupportedOperationException e) {
    System.out.println("不可变集合不能修改");            // 会打印
}

// 需要可变就立刻包一层
List<String> mutable = new ArrayList<>(List.of("a", "b"));
mutable.add("c");
System.out.println(mutable);                           // [a, b, c]
```

> **陷阱**：`Arrays.asList(...)` **不是** `List.of(...)` 的替代品——`Arrays.asList` 返回的列表**允许 `set` 但不允许结构性修改（`add`/`remove`）**，且**允许 `null`**，语义完全不同：
>
> | 对比 | `Arrays.asList` | `List.of` |
> |------|----------------|-----------|
> | `add` / `remove` | 抛 `UnsupportedOperationException` | 抛 `UnsupportedOperationException` |
> | `set` | **允许**（会写回原数组） | 抛 `UnsupportedOperationException` |
> | `null` 元素 | **允许** | **抛 `NullPointerException`** |
> | 与源数组的关系 | **共享**，改一处两边都变 | 完全独立 |
>
> `Arrays.asList` 返回的是**数组的视图**，只做了一层包装，所以 `set` 会直接改到原数组上，这是很多隐蔽 bug 的来源。

# 10. 其他实用新特性

## 10.1 String 增强（JDK 11+）

| 方法 | 说明 |
|------|------|
| `isBlank()` | 是否为空字符串或**仅含空白**（比 `isEmpty()` 更实用，能识别全空格串） |
| `strip()` / `stripLeading()` / `stripTrailing()` | 去除首尾 / 前导 / 尾部空白，**按 `Character.isWhitespace` 判断，支持 Unicode 空白** |
| `lines()` | 按行切分，返回 `Stream<String>` |
| `repeat(int n)` | 重复拼接 `n` 次 |
| `indent(int n)` | 给每一行增加 / 减少缩进，并保证结尾有换行 |
| `formatted(Object...)` | JDK 15+，等价于 `String.format(this, args)`，配合文本块最好用 |

```java
System.out.println("  ".isBlank());              // true
System.out.println("abc".isBlank());             // false

// strip 与 trim 的区别：trim 只处理 <= U+0020 的字符，strip 用 Unicode 空白判定
String s = " Hello ";                  // U+2003 是 EM SPACE（Unicode 空白）
System.out.println(s.trim().length());           // 7，trim 认不出它
System.out.println(s.strip().length());          // 5，strip 正确去掉了

"a\nb\nc".lines().forEach(System.out::println);  // a b c 逐行打印
System.out.println("ha".repeat(3));              // hahaha
System.out.println("ab".indent(4));              // "    ab\n"
```

> **注意**：`trim()` 的判定标准是"字符编码 `<= U+0020`"，而 `strip()` 用的是 `Character.isWhitespace()`。**新代码一律用 `strip()`**，尤其在处理从外部系统读来的、含有全角空格或不换行空格的文本时，`trim()` 会漏掉它们。

## 10.2 集合与 Stream 增强

| 特性 | 版本 | 说明 |
|------|------|------|
| `List.of` / `Set.of` / `Map.of` / `Map.ofEntries` | JDK 9 | 见第 9 节 |
| `List.copyOf` / `Set.copyOf` / `Map.copyOf` | JDK 10 | 不可变副本，见 9.2 |
| `Collection.toArray(IntFunction)` | JDK 11 | `list.toArray(String[]::new)`，替代 `toArray(new String[0])` |
| `Stream.toList()` | **JDK 16** | `stream.toList()` 取代 `.collect(Collectors.toList())`，**返回不可变 List** |
| `Collectors.teeing` | JDK 12 | 一次遍历同时做两种归约，再把两个结果合并 |
| `SequencedCollection` | **JDK 21** | 给 `List`/`Deque` 等统一了"有序集合"接口：`getFirst()`/`getLast()`/`addFirst()`/`reversed()` |

```java
// Stream.toList()（JDK 16+）：更短，而且返回不可变列表
List<String> upper = List.of("a", "b").stream().map(String::toUpperCase).toList();
System.out.println(upper);                    // [A, B]
// upper.add("C");                            // 抛 UnsupportedOperationException

// Collectors.teeing（JDK 12+）：一次遍历同时算平均分与总人数
record Student(String name, int score) { }
List<Student> students = List.of(new Student("A", 90), new Student("B", 80));

String summary = students.stream().collect(Collectors.teeing(
        Collectors.averagingInt(Student::score),   // 下游 1：平均值
        Collectors.counting(),                     // 下游 2：计数
        (avg, count) -> "平均分 " + avg + "，共 " + count + " 人"));
System.out.println(summary);                  // 平均分 85.0，共 2 人

// SequencedCollection（JDK 21+）：统一的"首尾"访问
SequencedCollection<Integer> seq = new ArrayList<>(List.of(1, 2, 3));
System.out.println(seq.getFirst() + " / " + seq.getLast());   // 1 / 3
System.out.println(seq.reversed());                            // [3, 2, 1]（只是视图）
```

## 10.3 接口私有方法（JDK 9+）

JDK 8 有了 `default` 方法，但多个 `default` 方法之间无法抽取公共代码。JDK 9 补上了**接口私有方法**：

```java
public interface OrderService {
    default void createOrder() {
        validate();                 // default 方法可以调用私有方法
        System.out.println("创建订单");
    }

    default void updateOrder() {
        validate();                 // 复用了同一段逻辑
        System.out.println("更新订单");
    }

    // 私有实例方法：只能在接口内部被 default 方法调用，实现类看不到
    private void validate() {
        System.out.println("参数校验");
    }

    // 私有静态方法：供静态方法或 default 方法使用
    private static void log(String msg) {
        System.out.println("[log] " + msg);
    }
}
```

这对于**给接口新增方法而不破坏已有实现类**很有价值——把实现细节藏在接口内部，不需要额外抽一个抽象类。

## 10.4 Process API 增强（JDK 9+）

以前 Java 想拿到进程的 PID、父子关系、启动时间非常困难，`ProcessHandle` 一次补齐：

```java
// 当前进程
ProcessHandle current = ProcessHandle.current();
System.out.println("当前 PID: " + current.pid());

// 进程信息（返回 Optional，因为很多属性系统可能不提供）
ProcessHandle.Info info = current.info();
System.out.println("启动命令: " + info.command().orElse("未知"));
System.out.println("启动参数: " + info.arguments().map(Arrays::toString).orElse("无"));

// 遍历所有进程
ProcessHandle.allProcesses()
        .filter(p -> p.info().command().isPresent())
        .limit(5)
        .forEach(p -> System.out.println(p.pid() + " → " + p.info().command().orElse("")));

// 杀掉一个进程（destroy 是优雅终止，destroyForcibly 是强杀）
// ProcessHandle.of(12345L).ifPresent(ProcessHandle::destroy);
```

> **注意**：`ProcessHandle.Info` 的绝大多数方法返回的都是 `Optional`（`command()`、`arguments()`、`startInstant()` 等），因为**权限不足或平台不支持时这些信息可能拿不到**——这正好是第 28 篇讲的 `Optional` 的典型应用场景。

## 10.5 HttpClient（JDK 9 孵化 → JDK 11 正式）

`java.net.http.HttpClient` 取代了又老又难用的 `HttpURLConnection`，原生支持 **HTTP/2、WebSocket、异步请求**：

```java
import java.net.URI;
import java.net.http.*;
import java.time.Duration;

public class HttpDemo {
    public static void main(String[] args) throws Exception {
        // HttpClient 是线程安全的、可复用的，应该作为单例长期持有，不要每次请求都新建
        HttpClient client = HttpClient.newBuilder()
                .version(HttpClient.Version.HTTP_2)          // 优先 HTTP/2
                .connectTimeout(Duration.ofSeconds(5))
                .followRedirects(HttpClient.Redirect.NORMAL)
                .build();

        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create("https://httpbin.org/get"))
                .timeout(Duration.ofSeconds(10))
                .header("Accept", "application/json")
                .GET()
                .build();

        // 同步：调用线程阻塞等待完整响应
        HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
        System.out.println("状态码: " + response.statusCode());     // 200
        System.out.println("响应体长度: " + response.body().length());

        // 异步：返回 CompletableFuture，不阻塞
        client.sendAsync(request, HttpResponse.BodyHandlers.ofString())
                .thenApply(HttpResponse::body)
                .thenAccept(body -> System.out.println("异步收到 " + body.length() + " 字符"))
                .join();                                             // 演示用；生产中不要轻易 join
    }
}
```

> **最佳实践**：`HttpClient` **每次 `newBuilder().build()` 都会创建独立的连接池与线程池**，必须复用同一个实例。这与"`new` 一个对象就能用"的直觉相反，是使用它时最容易犯的性能错误。

## 10.6 Helpful NullPointerException（JDK 14+）

JDK 14 起，NPE 的消息会**明确指出是哪个变量、哪个方法调用**触发了空指针，默认开启（由 `-XX:+ShowCodeDetailsInExceptionMessages` 控制）：

```java
// 老版本输出：只说 NPE 发生在这一行，不知道该怪谁
// Exception in thread "main" java.lang.NullPointerException
//     at Demo.main(Demo.java:5)

// JDK 14+ 输出：
// Exception in thread "main" java.lang.NullPointerException:
//     Cannot invoke "String.length()" because "s" is null
//     at Demo.main(Demo.java:5)
```

```java
public class NpeDemo {
    public static void main(String[] args) {
        String s = null;
        System.out.println(s.length());   // Cannot invoke "String.length()" because "s" is null
    }
}
```

> **注意**：这条信息是**在 JVM 现场重建**的（字节码里并没有原始变量名），会带来极小的性能开销。JDK 14/15 需要显式开启 `-XX:+ShowCodeDetailsInExceptionMessages`，**JDK 15 起默认开启**。它对线上排错的价值极高，非常值得升级。

# 11. 虚拟线程 Virtual Threads（JDK 21 正式）

## 11.1 它解决什么问题

**平台线程（Platform Thread）很贵**。Java 的线程是"1:1 映射到操作系统线程"的——每创建一个 Java 线程就要创建一个内核线程：

| 平台线程的成本 | 具体表现 |
|---------------|---------|
| 栈内存 | 默认每个线程预留约 **1MB** 栈空间，1000 个线程就是约 1GB |
| 创建 / 销毁 | 涉及内核态系统调用，比创建普通对象贵几个数量级 |
| 上下文切换 | 由操作系统调度，每次切换都要保存 / 恢复寄存器与栈，开销在微秒级 |
| 数量上限 | 实践中几千到上万个线程，系统就开始明显劣化 |

这直接导致了第 27 篇讲的那个结论：**传统的"一请求一线程"模型撑不住高并发**——不是因为 CPU 算不过来，而是因为线程本身太贵。

更关键的是，**大部分线程的时间都花在"阻塞等待"上**：等数据库返回、等下游 RPC、等磁盘 IO。线程被阻塞时**占着那 1MB 栈却什么都不干**，这是纯粹的浪费。

**虚拟线程的思路**：把线程的实现从"内核线程"换成"**由 JVM 管理的轻量级线程**"。多个虚拟线程**复用少量平台线程**（称为载体线程，carrier thread）。当虚拟线程遇到阻塞式 IO 操作时，**JVM 会把它从载体线程上"卸载"下来**（`unmount`），让载体线程去跑别的虚拟线程；等 IO 就绪后再把它"挂载"回去继续执行。

这和 NIO 的异步回调是**两种截然不同的解题思路**：

| 思路 | 做法 | 代价 |
|------|------|------|
| NIO / 响应式 | 阻塞操作改成异步回调，一个线程轮询多连接 | **业务代码被拆成回调链**，难以阅读、调试、维护 |
| **虚拟线程** | **代码照旧写同步阻塞**，由 JVM 负责在阻塞时切换 | 几乎为零——**代码不用改，写法回到最直观的样子** |

## 11.2 三种创建方式

```java
import java.time.Duration;
import java.util.concurrent.*;
import java.util.stream.IntStream;

public class VirtualThreadDemo {
    public static void main(String[] args) throws Exception {
        // 方式一：Thread.ofVirtual()，最直接
        Thread vt = Thread.ofVirtual().name("vt-1").start(() -> {
            System.out.println("我是虚拟线程: " + Thread.currentThread());
            // 我是虚拟线程: VirtualThread[#21,vt-1]/runnable@ForkJoinPool-1-worker-1
        });
        vt.join();

        // 方式二（推荐）：每个任务一个虚拟线程的执行器
        // 注意：这里不需要、也不应该用 newFixedThreadPool 那种池化思路
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            Future<Integer> f = executor.submit(() -> 1 + 1);
            System.out.println("结果: " + f.get());     // 结果: 2
        }

        // 方式三：批量提交，观察虚拟线程可以轻松开到十万级
        long start = System.currentTimeMillis();
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            IntStream.range(0, 100_000).forEach(i -> executor.submit(() -> {
                Thread.sleep(Duration.ofMillis(10));    // 模拟 IO 等待
                return i;
            }));
        }   // try-with-resources 退出时会等待全部任务完成
        System.out.println("10 万个任务耗时: " + (System.currentTimeMillis() - start) + " ms");
    }
}
```

> **最佳实践**：**虚拟线程不要池化**。池化平台线程是为了避免昂贵的创建成本，而虚拟线程创建成本极低（一个虚拟线程只占几百字节），**池化反而会破坏它"用完即弃"的设计**，把请求和虚拟线程数量绑死，重新引入排队问题。官方的建议很直白：**用 `Executors.newVirtualThreadPerTaskExecutor()`，每个任务开一个新虚拟线程。**

## 11.3 与平台线程的对比

| 维度 | 平台线程（Platform Thread） | 虚拟线程（Virtual Thread） |
|------|---------------------------|--------------------------|
| 实现 | 1:1 映射到操作系统线程 | N:1 由 JVM 调度，挂载到少量载体线程上 |
| 栈内存 | 约 1MB（预留） | 几百字节起，**按需增长**，存在堆里 |
| 创建成本 | 高（内核系统调用） | **极低**（普通对象创建） |
| 数量上限 | 数千 ~ 上万 | **数十万 ~ 百万级** |
| 阻塞时的行为 | 线程被占用，什么都不干 | **自动卸载**，载体线程转去执行别的虚拟线程 |
| 调度者 | 操作系统 | **JVM**（默认 `ForkJoinPool`） |
| 是否可池化 | 是，**应该池化** | **不应该池化** |
| 线程优先级 / `ThreadGroup` | 支持 | 不支持（无意义） |
| 引入版本 | JDK 1.0 | JDK 19 预览，**JDK 21 正式** |

## 11.4 适用场景

```text
  IO 密集型（数据库、RPC、文件、网络）          CPU 密集型（加解密、图像处理、复杂计算）
  ────────────────────────────────────        ────────────────────────────────────
  线程大部分时间在"等"                          线程一直在占着 CPU 算
  虚拟线程能把"等"的时间交还给载体线程            虚拟线程的卸载机制用不上（没有阻塞发生）
  收益：极大，吞吐量可提升数倍到数十倍        收益：几乎没有（甚至更慢）
```

| 场景 | 是否适合虚拟线程 | 说明 |
|------|----------------|------|
| Web 服务的请求处理（每请求等待 DB / RPC） | **非常适合** | 这是虚拟线程的主战场，吞吐量提升明显 |
| 批量的外部 API 调用 / 爬虫 | **非常适合** | 过去要靠线程池 + 队列限流，现在直接每任务一线程 |
| 消息消费（含 IO 等待） | 适合 | 注意下游连接池也要相应放开 |
| 大数计算、加解密、图像处理 | **不适合** | 纯 CPU 消耗，虚拟线程没有任何优势 |
| 长时间持有 `synchronized` 锁的代码 | **谨慎** | 见下面的陷阱 |

> **陷阱**：**在 `synchronized` 块内阻塞会导致"钉住（pinning）"**——虚拟线程无法从载体线程卸载，这时它不仅不能发挥优势，还可能把整个载体线程池拖死。JDK 21 起，`ReentrantLock`、`Semaphore`、`Socket` IO 等都能正确卸载，**但 `synchronized` 里的阻塞调用不能**（JDK 24 起才有针对性的改进）。所以**在虚拟线程上跑的老代码，要把 `synchronized` 换成 `ReentrantLock`**；用 `-Djdk.tracePinnedThreads=full` 可以打印钉住的调用栈来定位。

> **注意**：还有两个连带影响。一是 `ThreadLocal` 在虚拟线程下**每个虚拟线程一份副本**，十万个虚拟线程就有十万份，非常吃内存——官方因此推出了**作用域值（Scoped Values）**作为替代；二是**下游资源池必须相应放开**，线程不再排队了，压力会直接转移到**数据库连接池、HTTP 连接池**上，如果连接池还是 10 个连接，虚拟线程再多也只能干等。

# 12. 小结

| 主题 | 一句话要点 |
|------|-----------|
| LTS 策略 | 每 6 个月一个版本，每 2 年一个 LTS（17 / 21 / 25）；生产只跟 LTS，新项目直接上最新 LTS |
| `var` | 只能用于局部变量，必须初始化、不能为 `null`；**类型不明显时不要用**，`var` 不是关键字 |
| Switch 表达式 | 箭头语法无穿透；块里用 `yield`（不是 `return`）交出值；**枚举场景宁可列全也不要写 `default`** |
| 文本块 | 自动去公共缩进（结束定界符的位置参与计算）；`\` 续行、`\s` 保留尾部空格、`formatted` 做变量内插 |
| 模式匹配 | `instanceof` 模式把"判断 + 强转"合成一步；绑定变量在 `&&` 右侧可用、`||` 右侧不可用 |
| 密封类 | `sealed` / `permits` + 子类的 `final` / `sealed` / `non-sealed`；价值在**配合模式匹配的穷尽性检查** |
| switch 模式匹配 | 支持类型模式、`case null`、`when` 守卫、Record 解构；**密封类型可省 `default`，新增子类会编译报错** |
| 不可变集合 | `List.of` / `Set.of` / `Map.of` / `Map.ofEntries` / `copyOf`；**不接受 `null`**，`Set.of` 遇重复元素直接抛异常 |
| 其他实用特性 | `String.strip`/`lines`/`repeat`、`Stream.toList`、`Collectors.teeing`、`HttpClient`、`ProcessHandle`、`SequencedCollection`、有用的 NPE 信息 |
| 虚拟线程 | N:1 映射，阻塞时自动卸载；**不要池化**；**IO 密集型收益巨大，CPU 密集型无收益**；远离 `synchronized` 阻塞（钉住） |

本篇是 JavaSE 语法与核心 API 部分的收尾。往前看，第 26 篇展开 `Record` 的完整细节，第 27 篇讲 NIO 的通道与多路复用，第 28 篇讲 `Optional` 的用法与反模式，第 13 篇是 `Stream` 与方法引用的基础；再往后，Java 的进阶内容将进入 JVM、并发底层与框架源码的领域。
