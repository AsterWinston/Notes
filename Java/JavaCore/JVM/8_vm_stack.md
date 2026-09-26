
# 1. 什么是虚拟机栈

- 每个线程创建时分配的内存区域，线程私有。
- 每个方法对应一个栈帧（Stack Frame）入栈/出栈。
- 存储：局部变量表、操作数栈、动态连接、方法返回地址。

```
线程的栈内存：

栈顶 ┌──────────────────┐ ← 当前执行的方法
     │    栈帧 (bar)     │
     ├──────────────────┤
     │    栈帧 (foo)     │
     ├──────────────────┤
     │    栈帧 (main)    │
栈底 └──────────────────┘
```

# 2. 栈帧的内部结构

```
┌──────────────────────────┐
│       栈帧 (Stack Frame)  │
├──────────────────────────┤
│   局部变量表 (Local Vars) │  ← 方法参数 + 局部变量
├──────────────────────────┤
│   操作数栈 (Operand Stack)│  ← 计算中间结果
├──────────────────────────┤
│   动态连接 (Dynamic Link) │  ← 指向运行时常量池中方法引用
├──────────────────────────┤
│   方法返回地址 (Return)   │  ← 调用者 PC 寄存器值
├──────────────────────────┤
│   附加信息                │  ← 调试信息等
└──────────────────────────┘
```

# 3. 局部变量表（Local Variables）

- 以**变量槽**（Slot）为最小单位（每个 Slot 32 位）。
- 存储方法参数和方法体内的局部变量。
- 非静态方法第 0 位 Slot 存放 `this`。
- long/double 占用 2 个 Slot，其余类型 1 个 Slot。

```java
public int add(int a, int b) {  // this(0), a(1), b(2)
    int sum = a + b;             // sum(3)
    return sum;
}
```

局部变量表布局：
```
Slot 0: this     (引用)
Slot 1: a        (int)
Slot 2: b        (int)
Slot 3: sum      (int)
```

**Slot 复用：**
```java
public void test() {
    {
        int a = 1;       // Slot 1 (0是this)
    }                    // a 的 Scope 结束
    int b = 2;           // Slot 1 被复用
}
```

- Slot 复用是 GC 中的一个重要因素——变量作用域结束后 Slot 被复用，原对象引用被覆盖，对象可被 GC。
- 反例：大对象在方法中未被及时回收，因为 Slot 还被引用着。

```java
// 不好的写法
public void bad() {
    byte[] bigData = new byte[100 * 1024 * 1024];  // 100MB
    // bigData 的 Slot 没有被复用，一直在占用
    // ...
}

// 更好的写法
public void better() {
    {
        byte[] bigData = new byte[100 * 1024 * 1024];
    }  // 离开作用域，但 Slot 可能还未被复用
    int dummy = 0;  // 强制复用 Slot，bigData 可被 GC
    // ...
}
```

# 4. 操作数栈（Operand Stack）

- 字节码指令的执行依赖操作数栈。
- 先进后出，最大深度在编译期确定。

示例：`int c = a + b;` 的字节码：
```
0: iload_1      // 将 a (Slot 1) 压入操作数栈       栈: [a]
1: iload_2      // 将 b (Slot 2) 压入操作数栈       栈: [a, b]
2: iadd         // 弹出 2 个 int，计算，压回结果     栈: [a+b]
3: istore_3     // 将栈顶结果存入 Slot 3 (c)        栈: []
```

操作数栈的另一个重要作用：准备方法调用参数。
```java
obj.doSomething(x, y);
// 字节码逻辑：
aload_0          // obj 压栈
iload_1          // x 压栈
iload_2          // y 压栈
invokevirtual    // 弹出 obj, x, y，调用方法
```

# 5. 动态连接（Dynamic Linking）

- 每个栈帧都包含一个指向运行时常量池中该栈帧所属方法的引用。
- 方法调用指令在运行时需要将符号引用解析为直接引用。
- 部分符号引用在类加载阶段就转为直接引用（静态解析），部分在运行时转换（动态连接）。

```
栈帧 ────→ 运行时常量池
             ├── #5 = Methodref  com.example.Foo.add:(II)I
             │                        ↓
             │                   编译后指向方法入口
             └── ...
```

# 6. 方法返回地址

- 方法返回时，恢复到调用者的位置继续执行。
- 返回地址存储调用者的 PC 寄存器值。
- 返回方式：
  - 正常返回：`return` 指令（ireturn, lreturn, areturn, return 等）
  - 异常返回：遇到未处理的异常，通过异常处理器表查找处理

```
main() 调用 foo():
  PC = 0x0010  → 调用 foo()
  PC = 0x0011  ← foo() 返回后继续（存储在 foo 栈帧的返回地址中）
```

# 7. 方法调用

JVM 支持 5 种方法调用指令：

| 指令 | 说明 | 解析阶段 |
|------|------|----------|
| `invokestatic` | 调用静态方法 | 静态解析（编译期） |
| `invokespecial` | 调用构造器、私有方法、父类方法 | 静态解析 |
| `invokevirtual` | 调用虚方法（动态分派） | 动态连接（运行时） |
| `invokeinterface` | 调用接口方法 | 动态连接 |
| `invokedynamic` | 动态类型语言支持 | 运行时由引导方法确定 |

**静态解析 vs 动态分派：**
```java
Father obj = new Child();
obj.staticMethod();    // invokestatic  → 解析为 Father.staticMethod()
obj.privateMethod();   // invokespecial  → 解析为 Father.privateMethod()
obj.virtualMethod();   // invokevirtual  → 运行时看实际类型，调用 Child.virtualMethod()
```

# 8. 分派（Dispatch）

## 8.1 静态分派（重载 Overload）

```java
public class StaticDispatch {
    public void say(Human h)  { System.out.println("human"); }
    public void say(Man m)    { System.out.println("man"); }
    public void say(Woman w)  { System.out.println("woman"); }
    
    public static void main(String[] args) {
        Human man = new Man();     // 静态类型 Human，实际类型 Man
        Human woman = new Woman(); // 静态类型 Human，实际类型 Woman
        
        StaticDispatch sd = new StaticDispatch();
        sd.say(man);     // 输出 "human"  ← 根据静态类型选择
        sd.say(woman);   // 输出 "human"  ← 根据静态类型选择
        sd.say((Man) man); // 输出 "man" ← 强转后静态类型为 Man
    }
}
```

重载方法版本在**编译期**根据参数的**静态类型**确定。

## 8.2 动态分派（重写 Override）

```java
class Human {
    public void say() { System.out.println("human say"); }
}
class Man extends Human {
    @Override
    public void say() { System.out.println("man say"); }
}

Human obj = new Man();
obj.say();  // 输出 "man say" ← 根据实际类型决定
```

`invokevirtual` 指令的运行时解析过程：
1. 找到操作数栈顶元素所指对象的实际类型 C
2. 在 C 中查找与描述符和简单名称都相符的方法
3. 找到则返回直接引用，否则从下往上在 C 的父类中查找
4. 始终未找到则抛出 `AbstractMethodError`

# 9. invokedynamic

JDK 7 引入，为动态类型语言（如 Groovy、JRuby）在 JVM 上的实现提供支持。
Lambda 表达式也通过 `invokedynamic` 和 `LambdaMetafactory` 实现。

```java
List<String> list = Arrays.asList("a", "b");
list.forEach(s -> System.out.println(s));
// Lambda 被编译成 invokedynamic 调用
// 运行时由 LambdaMetafactory 生成一个内部类/方法句柄
```

# 10. 栈溢出问题

```java
// StackOverflowError 示例
public class StackOverflow {
    private int count = 0;
    
    public void recursive() {
        count++;
        recursive();
    }
    
    public static void main(String[] args) {
        try {
            new StackOverflow().recursive();
        } catch (StackOverflowError e) {
            System.out.println("Stack depth: " + depth);
        }
    }
}

// 默认栈深约 10000-20000 层（与栈大小相关）
```

栈空间参数：
```bash
-Xss128k      # 设置每线程栈大小为 128KB
-Xss1m        # 默认约 1MB
```

**线程数造成的 OOM：**
- 每个线程分配一定栈内存，线程数多 → 总栈内存大 → 可能导致 OOM。
- 如果遇到 `unable to create new native thread`，在物理内存足够时可适当减小 `-Xss`。