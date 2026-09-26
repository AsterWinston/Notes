# 1. 参数化测试（Parameterized）

JUnit 4 的 `Parameterized` Runner 提供基础参数化能力。

## 1.1 基本写法

```java
@RunWith(Parameterized.class)
public class CalculatorParamTest {

    @Parameters
    public static Collection<Object[]> data() {
        return Arrays.asList(new Object[][]{
            {1, 2, 3},
            {2, 3, 5}
        });
    }

    @Parameter(0)
    public int a;

    @Parameter(1)
    public int b;

    @Parameter(2)
    public int expected;

    @Test
    public void add_should_work() {
        assertEquals(expected, new Calculator().add(a, b));
    }
}
```

## 1.2 优缺点

- **优点**：避免重复代码，覆盖多组输入。
- **缺点**：仅支持一个 Runner，扩展受限；数据可读性一般。

# 2. Theories

Theories 用于基于“性质”进行验证，可组合数据点。

## 2.1 示例

```java
@RunWith(Theories.class)
public class MathTheoryTest {

    @DataPoint public static int zero = 0;
    @DataPoint public static int one = 1;

    @Theory
    public void add_should_be_commutative(int a, int b) {
        assertEquals(a + b, b + a);
    }
}
```

## 2.2 适用场景

- 数学性质验证
- 规则型业务逻辑

## 2.3 局限

- 数据点数量膨胀导致组合爆炸
- 失败案例定位成本高

# 3. 实践建议

- 小规模数据使用 Parameterized 即可。
- 对大量数据用外部数据源或迁移到 JUnit 5 参数化体系。
