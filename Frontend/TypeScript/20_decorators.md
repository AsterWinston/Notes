装饰器是一种"用语法糖在类定义时刻改写类行为"的元编程特性，TypeScript 里同时存在 legacy（experimentalDecorators）与 TC39 标准两套体系。本篇逐个拆解五种装饰器的签名与执行顺序，用纯 TS 手写一个依赖注入容器和一套参数校验装饰器，让你彻底理解元数据编程的运作机制。

# 1. 装饰器是什么

## 1.1 语法与定位

```typescript
// 装饰器 = 写在 @ 后面的函数，在类/成员"定义时"被自动调用
function frozen(target: Function) {       // 类装饰器：参数是类构造函数
  Object.freeze(target);                  // 冻结类：禁止再加静态成员
  Object.freeze(target.prototype);        // 冻结原型：禁止增删方法
}

@frozen                                   // 类定义完成时自动调用 frozen(Point)
class Point {
  constructor(public x: number, public y: number) {}  // 参数属性简写
}

// new Point(1, 2) 正常；再给 Point.prototype 加方法则静默失败或报错
```

★ **关键认知**：装饰器不是运行时"监听"，而是**编译器在类定义处插入的函数调用**（legacy 方案还会做语法降级改写）。它的典型用途是横切逻辑——日志、校验、缓存、依赖注入——把"与主流程无关的重复代码"收拢到一个装饰器里。这与 Java 的**注解 + 反射**思路同源，但 Java 注解本身不改行为（需要框架扫描后处理），TS 装饰器是**直接执行的代码**，行为改写当场发生。

## 1.2 两套标准对比

| 维度 | legacy `experimentalDecorators` | TC39 标准装饰器 |
|------|-------------------------------|----------------|
| 开关 | tsconfig 设 `experimentalDecorators: true` | 无需开关（TS 5.0+，类语法原生支持） |
| 本质 | TS 私有提案，编译期语法改写 | 纯运行时函数，类定义时调用 |
| 签名 | 按目标类型各异（2~3 个参数） | 统一 `(value, context)` 两参 |
| 执行时机 | 成员装饰器 → 类装饰器，同目标倒序应用 | 元素装饰器 → 类装饰器，同目标倒序应用 |
| 元数据 | `emitDecoratorMetadata` 产出 design:* | 无内建元数据（用 `context` 或自管 WeakMap） |
| 返回值 | 类/方法可返回替换物（描述符/构造函数） | 返回值即替换后的 value（方法返回新函数） |
| 生态存量 | NestJS、Angular、inversify 等老框架 | 新代码与无框架场景 |

⚠ 两套装饰器**不兼容**：同一段装饰器代码不能同时满足两套签名。混开（项目里部分包用 legacy、部分用标准）是混乱之源，务必全项目统一。

# 2. 装饰器签名逐个击破

## 2.1 类装饰器与方法装饰器（legacy）

```typescript
// ===== 类装饰器：参数是被装饰的类（构造函数） =====
function singleton(target: Function) {          // target 即类本身
  let instance: unknown;                        // 缓存唯一实例
  const original = target;                      // 保存原构造函数
  // 返回值可完全替换类（此处演示"替换成单例工厂"的思路，略去实现细节）
}

// ===== 方法装饰器：三个参数 =====
function log(
  target: Object,                               // 原型对象（static 方法则是构造函数）
  propertyKey: string | symbol,                 // 方法名
  descriptor: PropertyDescriptor                // 属性描述符，descriptor.value 是原方法
) {
  const original = descriptor.value;            // 保存原方法
  descriptor.value = function (...args: unknown[]) {  // 替换为包装函数
    console.log(`call ${String(propertyKey)}`, args); // 调用前记日志
    const result = original.apply(this, args);        // 调用原方法（this 透传）
    console.log(`return`, result);                    // 调用后记日志
    return result;                                    // 透传返回值
  };
}

class Calculator {
  @log
  add(a: number, b: number): number {
    return a + b;                               // 被 log 包裹，调用即打印
  }
}
new Calculator().add(1, 2);                    // call add [1, 2] / return 3
```

★ **关键认知**：方法装饰器靠 **PropertyDescriptor** 工作——它拿到的不是方法本身而是"属性描述符"，把 `descriptor.value` 换掉就完成了拦截。这就是"装饰器 = 编译期的中间件"的含义：多个装饰器像洋葱一样层层包裹原方法。⚠ 用普通函数替换方法时注意 `this`：必须用 `function` 而非箭头函数接收调用方的 `this`，否则 `original.apply(this, ...)` 会丢失上下文。

## 2.2 存取器、属性与参数装饰器（legacy）

```typescript
// ===== 存取器装饰器（get/set）：签名同方法装饰器 =====
function editable(
  target: Object,
  propertyKey: string | symbol,
  descriptor: PropertyDescriptor
) {
  descriptor.set = function () {                // 直接替换 setter
    throw new Error(`${String(propertyKey)} 是只读的`);  // 禁止赋值
  };
}

// ===== 属性装饰器：只有两个参数，拿不到描述符 =====
function notNull(target: Object, propertyKey: string | symbol) {
  // 只能"登记"不能改写：把属性名记进校验清单
  const list: (string | symbol)[] =
    Reflect.getMetadata("notNull", target) ?? [];        // 读已登记清单
  Reflect.defineMetadata("notNull", [...list, propertyKey], target); // 写回
}

// ===== 参数装饰器：target + 方法名 + 参数下标 =====
function required(target: Object, propertyKey: string | symbol, index: number) {
  const map: Record<string, number[]> =
    Reflect.getMetadata("required", target, propertyKey) ?? {};  // 读已有记录
  map[propertyKey] = [...(map[propertyKey] ?? []), index];       // 记下第几参
  Reflect.defineMetadata("required", map, target, propertyKey);  // 写回元数据
}
```

★ **关键认知**：**属性装饰器与参数装饰器不能改写目标**——它们没有描述符参数，只能做"登记"（往元数据里写备注），真正的行为生效要靠方法装饰器在调用前读取这些登记（这正是第 5 节校验框架的骨架）。参数装饰器的 `index` 是参数在形参列表中的下标，多个参数装饰器对同一参数仍按倒序应用。

## 2.3 TC39 标准装饰器的统一签名

```typescript
// 标准装饰器：统一 (value, context) 两参，context.kind 说明装饰目标
function log(value: unknown, context: ClassMethodDecoratorContext) {
  if (context.kind === "method") {              // 用 kind 区分目标类型
    return function (this: unknown, ...args: unknown[]) {  // 返回替换函数
      console.log(`call ${String(context.name)}`);         // context.name 即方法名
      return (value as Function).apply(this, args);        // value 是原方法
    };
  }
}

class Greeter {
  @log                                          // 与 legacy 用法几乎一样
  greet(name: string): string {
    return `hello ${name}`;
  }
}
```

★ **关键认知**：标准装饰器把"五种签名"收敛成一种——`value` 是被装饰物本身（类/方法函数/字段值），`context` 是包含 `kind`、`name`、`static`、`private`、`addInitializer` 的上下文对象。方法装饰器返回的新函数直接替换原方法，类装饰器返回的新类直接替换原类。⚠ 标准装饰器**没有 `emitDecoratorMetadata`**，拿不到 `design:paramtypes`；需要元数据时要么用 `context.metadata`（提案内建、生态尚新），要么自管 `WeakMap` 登记。

# 3. 装饰器工厂与执行顺序

## 3.1 装饰器工厂：高阶函数返回装饰器

```typescript
// 工厂：接收配置参数，返回真正的装饰器（带参数的 @Xxx(...) 都是它）
function retry(times: number) {                 // 外层接收配置
  return function (                            // 内层才是装饰器
    target: Object,
    propertyKey: string | symbol,
    descriptor: PropertyDescriptor
  ) {
    const original = descriptor.value;          // 保存原方法
    descriptor.value = async function (...args: unknown[]) {
      for (let i = 0; i < times; i++) {         // 最多重试 times 次
        try {
          return await original.apply(this, args);   // 成功即返回
        } catch (e) {
          if (i === times - 1) throw e;         // 最后一次仍失败：抛出
        }
      }
    };
  };
}

class Api {
  @retry(3)                                     // 等价于 @retry(3)(target, key, desc)
  fetch(): string {
    return "ok";                                // 失败会自动重试至多 3 次
  }
}
```

★ **关键认知**：`@retry(3)` 分两步执行——**先调用工厂 `retry(3)` 得到装饰器函数，再用这个函数装饰目标**。理解这一点就能看懂报错：`@retry`（忘写括号）会把工厂函数本身当装饰器传入，签名对不上直接编译报错。Java 的注解传参（`@Retry(3)`）在运行时由框架读取，TS 装饰器的"参数"则在定义时就被闭包捕获。

## 3.2 执行顺序

```
 同一目标多个装饰器：表达式自上而下求值，应用自下而上（倒序包裹）

   源码：  @A @B class C {}          @C @D @E method() {}
   求值：  A → B                     C → D → E
   应用：  B 先包 C，A 再包 B         E 先包原方法，D 包 E，C 包 D

 跨目标顺序：参数装饰器 → 方法/属性/存取器装饰器 → 类装饰器

   ┌─────────────────────────────────────────────┐
   │ class 装饰器 @A          ← 最后执行（包住一切）│
   │ ┌─────────────────────────────────────────┐ │
   │ │ method 装饰器 @C @D @E   ← 成员间按声明序 │ │
   │ │ ┌─────────────────────────────────────┐ │ │
   │ │ │ 参数装饰器 @P @Q      ← 最先执行     │ │ │
   │ │ │        原方法 body                  │ │ │
   │ │ └─────────────────────────────────────┘ │ │
   │ └─────────────────────────────────────────┘ │
   └─────────────────────────────────────────────┘
```

★ **关键认知**：装饰器执行顺序与函数组合完全同构——**求值从上到下（先算完配置），包裹从下到上（离方法近的先生效）**。类装饰器最后执行，因为类必须先有完整的成员定义才能被整体处理。⚠ 想验证顺序最快的办法是写几个 `console.log` 的装饰器套在一个类上跑一遍，比死记规则可靠；这也是排查"装饰器没生效"类 bug 的第一招。

# 4. reflect-metadata 与设计时类型元数据

## 4.1 design:type 三件套

```typescript
import "reflect-metadata";                  // ⚠ 必须最先导入，扩展 Reflect

function typeInfo(target: Object, propertyKey: string | symbol) {
  const t = Reflect.getMetadata("design:type", target, propertyKey); // 设计时类型
  console.log(`${String(propertyKey)}:`, t?.name);                   // 打印构造名
}

class Article {
  @typeInfo title: string = "";            // 输出 "title: String"
  @typeInfo views: number = 0;             // 输出 "views: Number"
  @typeInfo tags: string[] = [];           // 输出 "tags: Array"
}
```

★ **关键认知**：开启 `emitDecoratorMetadata` 后，tsc 会在**有装饰器的成员**上额外发射两个元数据：`design:type`（属性/返回值类型）、`design:paramtypes`（构造器/方法参数类型数组）。这与第 1 篇的**类型擦除**看似矛盾——实则是"擦除为主、局部留痕"：只有你显式开启且成员挂了装饰器，编译器才把类型退化成运行时构造函数（`String`/`Number`/`Array`/自定义类）写进元数据。

⚠ 三个经典限制：① 只在 `experimentalDecorators` + `emitDecoratorMetadata` 双开时产出，TC39 标准装饰器没有这套机制；② `interface` 类型的参数退化成 `Object`（接口运行时不存在）；③ 泛型参数被擦成其约束类型或 `Object`——**元数据是设计时类型的"近似值"，不能当精确类型用**。

```typescript
// tsconfig.json 对应开关（legacy 方案）
// {
//   "compilerOptions": {
//     "experimentalDecorators": true,      // 启用 legacy 装饰器
//     "emitDecoratorMetadata": true        // 产出 design:* 元数据
//   }
// }
```

★ **关键认知**：`reflect-metadata` 是一个**运行时 polyfill**：它往全局 `Reflect` 上挂 `defineMetadata`/`getMetadata` 等方法（基于 WeakMap 实现元数据存储），tsc 只负责"写"元数据，"读"要靠这个库。Java 程序员会发现这套东西和 `java.lang.reflect` + 注解处理器惊人地相似——存储位置从 JVM 常量池换成了 WeakMap，"编译期登记、运行期读取"的思想如出一辙。

# 5. 综合实战：纯 TS 手写小型元编程框架

## 5.1 极简依赖注入容器

```typescript
import "reflect-metadata";

// 1. @Injectable：类装饰器工厂，把类登记为"可注入"
function Injectable(): ClassDecorator {
  return (target) => {
    Reflect.defineMetadata("injectable", true, target);  // 打标记
  };
}

// 2. 极简容器：按 design:paramtypes 递归装配构造器依赖
class Container {
  private registry = new Map<Function, unknown>();       // 类 → 单例缓存

  register(cls: Function): void {
    this.registry.set(cls, cls);                         // 登记类
  }

  resolve<T>(cls: new (...args: unknown[]) => T): T {
    const cached = this.registry.get(cls);               // 查单例缓存
    if (cached instanceof cls) return cached as T;

    const paramTypes: unknown[] =
      Reflect.getMetadata("design:paramtypes", cls) ?? [];  // 读参数类型元数据
    const deps = paramTypes.map((p) =>
      this.resolve(p as new (...args: unknown[]) => unknown) // 递归解析依赖
    );
    const instance = new cls(...deps);                   // 注入并实例化
    this.registry.set(cls, instance);                    // 缓存单例
    return instance;
  }
}

// 3. 业务类：构造器声明依赖，其余交给容器
@Injectable()
class Logger {
  write(msg: string): void {
    console.log(`[log] ${msg}`);                         // 简单打日志
  }
}

@Injectable()
class UserService {
  constructor(private logger: Logger) {}                 // 依赖 Logger
  greet(name: string): void {
    this.logger.write(`hello ${name}`);                  // 用注入的实例
  }
}

const container = new Container();
container.register(Logger);                              // 登记依赖
container.register(UserService);
container.resolve(UserService).greet("TS");              // [log] hello TS
```

★ **关键认知**：容器的全部魔力就是**读 `design:paramtypes` 然后递归 `new`**——依赖关系从"手写 `new UserService(new Logger())`"变成"构造器声明即装配清单"。这演示了元数据编程的本质：**编译器留下的类型痕迹 + 运行时反射 = 自动化装配**。⚠ 实战注意两点：接口做不了依赖 token（元数据里只有 `Object`），要注入接口类型必须用具名 token 或抽象类；循环依赖会让递归 `resolve` 无限展开，需要引入"延迟解析"（provider 工厂）打破环。

## 5.2 参数校验装饰器

```typescript
const RULES = "validation:rules";                // 元数据 key 常量

interface Rule {                                  // 规则结构：下标 + 检查 + 提示
  index: number;
  check: (v: unknown) => boolean;
  msg: string;
}

// @Min(n)：参数装饰器工厂，登记"数值下限"规则
function Min(min: number): ParameterDecorator {
  return (target, propertyKey, parameterIndex) => {
    const rules: Rule[] = Reflect.getMetadata(RULES, target, propertyKey) ?? [];
    rules.push({
      index: parameterIndex,
      check: (v) => typeof v === "number" && v >= min,   // 校验闭包
      msg: `参数 #${parameterIndex} 必须 >= ${min}`,
    });
    Reflect.defineMetadata(RULES, rules, target, propertyKey);  // 写回元数据
  };
}

// @Required：参数装饰器工厂，登记"非空"规则
function Required(): ParameterDecorator {
  return (target, propertyKey, parameterIndex) => {
    const rules: Rule[] = Reflect.getMetadata(RULES, target, propertyKey) ?? [];
    rules.push({
      index: parameterIndex,
      check: (v) => v !== undefined && v !== null,        // 非空检查
      msg: `参数 #${parameterIndex} 不能为空`,
    });
    Reflect.defineMetadata(RULES, rules, target, propertyKey);
  };
}

// @Validate：方法装饰器，调用前执行全部登记的规则
function Validate(target: Object, propertyKey: string | symbol, descriptor: PropertyDescriptor) {
  const original = descriptor.value;                     // 保存原方法
  descriptor.value = function (...args: unknown[]) {
    const rules: Rule[] = Reflect.getMetadata(RULES, target, propertyKey) ?? [];
    for (const rule of rules) {                          // 逐条执行规则
      if (!rule.check(args[rule.index])) {               // 不满足即拦截
        throw new Error(`${String(propertyKey)}: ${rule.msg}`);
      }
    }
    return original.apply(this, args);                   // 全部通过才执行
  };
}

// 使用：三个装饰器分工明确——参数装饰器"登记规则"，方法装饰器"执行规则"
class OrderService {
  @Validate
  place(@Min(1) quantity: number, @Required() customer: string): string {
    return `下单 ${quantity} 件，客户 ${customer}`;
  }
}

const svc = new OrderService();
console.log(svc.place(3, "Alice"));   // 下单 3 件，客户 Alice
svc.place(0, "Bob");                  // ✗ Error: place: 参数 #0 必须 >= 1
```

★ **关键认知**：这个例子完整复刻了"注解 + 校验框架"的经典架构：**参数装饰器只负责把规则写进元数据（声明），方法装饰器负责读取规则并执行（生效）**。声明与执行分离，正是 2.2 节所说"属性/参数装饰器只能登记"的自然结果。与 Java 的 Bean Validation（`@Min` 注解 + 框架反射扫描）相比，TS 版少了注解处理器这层，规则闭包直接活在元数据里，调试时可以直接断点进 `check`。⚠ 校验闭包捕获的是装饰器求值时的配置值，若规则需要动态配置（如多语言错误消息），应把配置对象整体存入元数据而不是拼死字符串。

# 6. 两套标准怎么选

| 场景 | 建议 |
|------|------|
| 新项目、无框架绑定 | TC39 标准装饰器（语言原生，无编译开关） |
| 维护依赖注入/老框架存量代码 | legacy `experimentalDecorators`（生态要求） |
| 需要 design:* 构造器元数据 | 只能 legacy + `emitDecoratorMetadata` |
| 只想加日志/缓存等横切逻辑 | 两者皆可，优先标准装饰器 |

★ **关键认知**：决策的分水岭是**要不要构造器参数元数据**：标准装饰器更"纯"（就是普通函数，可独立测试、可 tree-shake），legacy 胜在能拿到 `design:paramtypes` 做自动装配。⚠ 另有一个常被忽略的坑：`useDefineForClassFields`（target ES2022+ 默认开启）会改变类字段的初始化语义，与 legacy 装饰器组合时可能出现字段覆盖顺序问题，混用老代码时留意 tsconfig 里这两位的搭配。
