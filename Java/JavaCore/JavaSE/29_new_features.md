# 1. var — 局部变量类型推断（JDK10）

```
var 允许在局部变量声明时省略显式类型，由编译器根据右侧表达式自动推断。

语法：
  var str = "Hello";          // 推断为 String
  var list = new ArrayList<String>();  // 推断为 ArrayList<String>
  var path = Path.of("/tmp");  // 推断为 Path

限制：
  1. var 只能用于局部变量，不能用于字段、方法参数、方法返回值
  2. var 必须在声明时初始化（编译器需要推断类型）
  3. var 不能初始化为 null（无法推断类型）
  4. var 不能用于 Lambda 表达式（Lambda 需要目标类型）

适用场景：
  1. 类型名很长时：var map = new HashMap<String, List<Integer>>();
  2. 变量名已经足够表达含义时：var user = new User();
  3. try-with-resources：var reader = new BufferedReader(new FileReader("file.txt"));

注意事项：
  - 不要滥用 var，当类型不明显时（如 var result = someMethod()）会影响可读性
  - var 不是关键字，是一个"保留类型名"（可以用 var 作为变量名）
```

# 2. Switch 表达式（JDK12 预览 → JDK14 正式）

```
Switch 表达式允许 switch 语句直接返回值，语法更简洁，避免了 case 穿透。

## 2.1 箭头语法（->），无穿透，可返回

  传统写法：
    String type;
    switch (day) {
        case 1:
        case 2:
        case 3:
        case 4:
        case 5:
            type = "工作日";
            break;
        case 6:
        case 7:
            type = "周末";
            break;
        default:
            type = "未知";
    }

  Switch 表达式写法：
    String type = switch (day) {
        case 1, 2, 3, 4, 5 -> "工作日";        // 多个 case 用逗号合并
        case 6, 7 -> "周末";
        default -> "未知";
    };

## 2.2 块语句使用 yield 返回值

    String type = switch (day) {
        case 1, 2, 3, 4, 5 -> {
            System.out.println("今天是工作日");
            yield "工作日";  // 使用 yield 返回值（不是 return）
        }
        case 6, 7 -> "周末";
        default -> "未知";
    };

## 2.3 传统冒号(:)语法也支持 yield

    String type = switch (day) {
        case 1: case 2: case 3: case 4: case 5:
            yield "工作日";
        case 6: case 7:
            yield "周末";
        default:
            yield "未知";
    };

  注意：传统冒号写法配合 yield，不支持穿透（fall-through），
       但可以与 break 共存（但混用意义不大）。
```

# 3. Text Blocks — 文本块（JDK13 预览 → JDK15 正式）

```
文本块使用三引号 """...""" 包裹多行字符串，简化了多行文本和含特殊字符的字符串。

语法：
  String html = """
              <html>
                  <body>
                      <p>Hello, World!</p>
                  </body>
              </html>
              """;

特性：
  3.1 自动缩进处理
      编译器自动去除行首的公共缩进（以最小缩进行为基准）。
  
  3.2 行尾自动换行
      每行结尾默认加 \n，除非使用 \ 抑制换行符。
      String sql = """
                   SELECT id, name, age \
                   FROM users \
                   WHERE age > 18
                   """;
      // 结果为单行：SELECT id, name, age FROM users WHERE age > 18

  3.3 保留尾部空格
      使用 \s（一个空格后跟 \s 表示强制保留尾部空格）。
      String line = """
                    hello    \s
                    world
                    """;

  3.4 内部引号无需转义
      String json = """
                    {
                        "name": "Tom",
                        "age": 25
                    }
                    """;

  3.5 String.formatted() 方法（JDK15+）
      String template = """
                        Name: %s
                        Age: %d
                        """.formatted("Tom", 25);
```

# 4. instanceof 模式匹配（JDK14 预览 → JDK16 正式）

```
简化 instanceof 判断 + 类型转换的模式。

传统写法：
  if (obj instanceof String) {
      String s = (String) obj;
      System.out.println(s.length());
  }

模式匹配写法：
  if (obj instanceof String s) {
      System.out.println(s.length());  // 直接使用 s，无需强转
  }

进阶使用：
  // 可以结合条件
  if (obj instanceof String s && s.length() > 5) {
      System.out.println("长字符串: " + s);
  }

  // 取反
  if (!(obj instanceof String s)) {
      return;  // 此时 s 不在作用域内
  }
  System.out.println(s);  // 此处 s 可用

作用域：
  s 的作用域为 instanceof 结果为 true 的代码块范围。
  在 && 运算符右侧可直接使用 s（因为前面已确认类型）。
  在 || 运算符右侧不能使用 s（因为可能未匹配）。
```

# 5. Record（JDK14 预览 → JDK16 正式）

```
详见 26_record.md。

Record 是简化不可变数据类的语法糖。
  public record Point(int x, int y) {}
  自动生成构造器、getter、equals、hashCode、toString。
```

# 6. Sealed Classes — 密封类（JDK15 预览 → JDK17 正式）

```
密封类限制哪些类可以继承它们，增强了类型安全性（特别是配合 switch 模式匹配）。

语法：
  // 使用 permits 指定允许的子类
  public sealed class Shape permits Circle, Rectangle, Triangle {}

  public final class Circle extends Shape {}
  public final class Rectangle extends Shape {}
  public non-sealed class Triangle extends Shape {}  // non-sealed 表示可被任意子类继承

三种子类修饰符：
  - final：子类不能再被继承
  - sealed：子类只能在限定范围内被继承（需再次 permits）
  - non-sealed：子类可被任意类继承（打破密封限制）

密封接口：
  public sealed interface JsonValue permits JsonNumber, JsonString, JsonObject {}
  
  public record JsonNumber(double value) implements JsonValue {}
  public record JsonString(String value) implements JsonValue {}
  public final class JsonObject implements JsonValue {}

结合 switch 模式匹配（JDK21 正式）：
  // Record Pattern + switch
  String describe(Shape shape) {
      return switch (shape) {
          case Circle c -> "圆形，半径: " + c.radius();
          case Rectangle r -> "矩形，宽高: " + r.width() + "x" + r.height();
          case Triangle t -> "三角形";
      };  // sealed 确保覆盖了所有子类，无需 default
  }
```

# 7. Record 模式匹配与 Switch 模式匹配（JDK17 预览 → JDK21 正式）

```
7.1 Record 模式：解构 Record，直接获取其组件值

  record Point(int x, int y) {}

  // 传统写法
  if (obj instanceof Point p) {
      System.out.println(p.x() + ", " + p.y());
  }

  // Record 模式（JDK21+）
  if (obj instanceof Point(int x, int y)) {
      System.out.println(x + ", " + y);  // 直接使用 x, y
  }

7.2 嵌套 Record 模式

  record Rectangle(Point topLeft, Point bottomRight) {}

  if (obj instanceof Rectangle(Point(int x1, int y1), Point(int x2, int y2))) {
      System.out.println("矩形: (" + x1 + "," + y1 + ") - (" + x2 + "," + y2 + ")");
  }

7.3 switch 模式匹配（JDK21 正式）

  String toString(Object obj) {
      return switch (obj) {
          case null -> "null";
          case String s -> "字符串: " + s;
          case Integer i -> "整数: " + i;
          case int[] arr -> "整型数组，长度: " + arr.length;
          case Point(int x, int y) -> "点: " + x + "," + y;
          default -> "未知类型";
      };
  }

  当 switch 的表达式类型是 sealed 类型且覆盖了所有子类时，default 可以省略。
```

# 8. 不可变集合的便捷创建（JDK9+）

```
List.of、Set.of、Map.of 系列方法简化了不可变集合的创建。

8.1 List.of
  List<String> list = List.of("a", "b", "c");
  // 不可变：list.add("d") → UnsupportedOperationException
  // 不允许 null 元素：List.of("a", null) → NullPointerException

8.2 Set.of
  Set<Integer> set = Set.of(1, 2, 3);
  // 不允许重复元素：Set.of(1, 1) → IllegalArgumentException

8.3 Map.of（最多 10 个键值对）
  Map<String, Integer> map = Map.of("a", 1, "b", 2, "c", 3);
  // 键值对交替传入：key1, value1, key2, value2, ...

8.4 Map.ofEntries（无限制）
  Map<String, Integer> map = Map.ofEntries(
      Map.entry("a", 1),
      Map.entry("b", 2),
      Map.entry("c", 3)
  );

8.5 集合的 copyOf（JDK10+）
  List<String> original = Arrays.asList("a", "b");
  List<String> copy = List.copyOf(original);  // 不可变副本
```

# 9. 其他实用新特性

```
9.1 String 增强（JDK11+）
  isBlank()       // 是否为空字符串或仅包含空白
  strip()         // 去除首尾空白（比 trim 更强，支持 Unicode 空白）
  stripLeading()  // 去除前导空白
  stripTrailing() // 去除尾部空白
  lines()         // 按行分割，返回 Stream<String>
  repeat(int)     // 重复拼接
  indent(int)     // 调整每行缩进
  formatted(Object...) // JDK15+，格式化字符串

  示例：
    String multiline = "a\nb\nc";
    multiline.lines().forEach(System.out::println);  // 逐行打印
    "ha".repeat(3);  // "hahaha"

9.2 集合增强
  List.of/Set.of/Map.of (JDK9+) // 见第 8 节
  toArray(IntFunction) (JDK11)
    String[] arr = list.toArray(String[]::new);
  toList() 方法 (JDK16+)
    List<String> list = stream.toList();  // 返回不可变 List
    替代 .collect(Collectors.toList())，更简洁且不可变

9.3 接口私有方法（JDK9+）
  public interface MyInterface {
      // 默认方法可以调用私有方法
      default void methodA() {
          commonLogicA();
      }
      default void methodB() {
          commonLogicA();
          commonLogicB();
      }
      private void commonLogicA() {
          // 抽取的公共逻辑
      }
      private static void commonLogicB() {
          // 静态私有方法
      }
  }

9.4 Process API 增强（JDK9+）
  ProcessHandle.allProcesses()    // 获取所有进程
  ProcessHandle.current()         // 获取当前进程
  ProcessHandle.Info.info()       // 获取进程信息
  long pid = ProcessHandle.current().pid();  // 获取 PID

9.5 HttpClient（JDK9 孵化 → JDK11 正式）
  java.net.http.HttpClient 取代了老旧的 HttpURLConnection。
  支持 HTTP/2 和 WebSocket，异步编程。

  HttpClient client = HttpClient.newHttpClient();
  HttpRequest request = HttpRequest.newBuilder()
      .uri(URI.create("https://api.example.com/data"))
      .GET()
      .build();

  // 同步
  HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
  
  // 异步
  client.sendAsync(request, HttpResponse.BodyHandlers.ofString())
      .thenApply(HttpResponse::body)
      .thenAccept(System.out::println);

9.6 Helpful NullPointerException（JDK14+）
  更精确的空指针异常信息，明确指出是哪个变量为 null。
  
  开启（JDK14+ 默认开启）：
    -XX:+ShowCodeDetailsInExceptionMessages

  示例输出：
    Exception in thread "main" java.lang.NullPointerException:
        Cannot invoke "String.length()" because "s" is null

9.7 增强 for 循环中不可知的副作用优化（JDK16+）
  Stream.toList() 返回不可变的 List，比 Collectors.toList() 更佳。
```
