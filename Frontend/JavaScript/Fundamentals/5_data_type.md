# 1. 原始类型的方法
```javascript
JavaScript 允许我们像使用对象一样使用原始类型（字符串，数字等）。JavaScript 还提供了这样的调用方法。我们很快就会学习它们，但是首先我们将了解它的工作原理，毕竟原始类型不是对象（在这里我们会分析地更加清楚）。

我们来看看原始类型和对象之间的关键区别。

一个原始值：

是原始类型中的一种值。
在 JavaScript 中有 7 种原始类型：string，number，bigint，boolean，symbol，null 和 undefined。
一个对象：

能够存储多个值作为属性。
可以使用大括号 {} 创建对象，例如：{name: "John", age: 30}。JavaScript 中还有其他种类的对象，例如函数就是对象。
关于对象的最好的事儿之一是，我们可以把一个函数作为对象的属性存储到对象中。

let john = {
  name: "John",
  sayHi: function() {
    alert("Hi buddy!");
  }
};

john.sayHi(); // Hi buddy!
所以我们在这里创建了一个包含 sayHi 方法的对象 john。

许多内建对象已经存在，例如那些处理日期、错误、HTML 元素等的内建对象。它们具有不同的属性和方法。

但是，这些特性（feature）都是有成本的！

对象比原始类型“更重”。它们需要额外的资源来支持运作。
```
## 1.1 当作对象的原始类型
```
以下是 JavaScript 创建者面临的悖论：

人们可能想对诸如字符串或数字之类的原始类型执行很多操作。最好使用方法来访问它们。
原始类型必须尽可能的简单轻量。
而解决方案看起来多少有点尴尬，如下：

原始类型仍然是原始的。与预期相同，提供单个值
JavaScript 允许访问字符串，数字，布尔值和 symbol 的方法和属性。
为了使它们起作用，创建了提供额外功能的特殊“对象包装器”，使用后即被销毁。
“对象包装器”对于每种原始类型都是不同的，它们被称为 String、Number、Boolean、Symbol 和 BigInt。因此，它们提供了不同的方法。

例如，字符串方法 str.toUpperCase() 返回一个大写化处理的字符串。

用法演示如下：

let str = "Hello";

alert( str.toUpperCase() ); // HELLO
很简单，对吧？以下是 str.toUpperCase() 中实际发生的情况：

字符串 str 是一个原始值。因此，在访问其属性时，会创建一个包含字符串字面值的特殊对象，并且具有可用的方法，例如 toUpperCase()。
该方法运行并返回一个新的字符串（由 alert 显示）。
特殊对象被销毁，只留下原始值 str。
所以原始类型可以提供方法，但它们依然是轻量级的。

JavaScript 引擎高度优化了这个过程。它甚至可能跳过创建额外的对象。但是它仍然必须遵守规范，并且表现得好像它创建了一样。

数字有其自己的方法，例如，toFixed(n) 将数字舍入到给定的精度：

let n = 1.23456;

alert( n.toFixed(2) ); // 1.23
我们将在后面 数字类型 和 字符串 章节中看到更多具体的方法。

构造器 String/Number/Boolean 仅供内部使用
像 Java 这样的一些语言允许我们使用 new Number(1) 或 new Boolean(false) 等语法，明确地为原始类型创建“对象包装器”。

在 JavaScript 中，由于历史原因，这也是可以的，但极其 不推荐。因为这样会出问题。

例如：

alert( typeof 0 ); // "number"

alert( typeof new Number(0) ); // "object"!
对象在 if 中始终为真，所以此处的 alert 将显示：

let zero = new Number(0);

if (zero) { // zero 为 true，因为它是一个对象
  alert( "zero is truthy?!?" );
}
另一方面，调用不带 new（关键字）的 String/Number/Boolean 函数是可以的且有效的。它们将一个值转换为相应的类型：转成字符串、数字或布尔值（原始类型）。

例如，下面完全是有效的：

let num = Number("123"); // 将字符串转成数字
null/undefined 没有任何方法
特殊的原始类型 null 和 undefined 是例外。它们没有对应的“对象包装器”，也没有提供任何方法。从某种意义上说，它们是“最原始的”。

尝试访问这种值的属性会导致错误：

alert(null.test); // error
```

# 2. 数字类型
```javascript
在现代 JavaScript 中，数字（number）有两种类型：

JavaScript 中的常规数字以 64 位的格式 IEEE-754 存储，也被称为“双精度浮点数”。这是我们大多数时候所使用的数字，我们将在本章中学习它们。

BigInt 用于表示任意长度的整数。有时会需要它们，因为正如我们在前面的章节 数据类型 中提到的，常规整数不能安全地超过 (253-1) 或小于 -(253-1)。由于仅在少数特殊领域才会用到 BigInt，因此我们在特殊的章节 BigInt 中对其进行了介绍。

所以，在这里我们将讨论常规数字类型。现在让我们开始学习吧。
```
## 2.1 编写数字的更多方法
```
假如我们需要表示 10 亿。显然，我们可以这样写：

let billion = 1000000000;
我们也可以使用下划线 _ 作为分隔符：

let billion = 1_000_000_000;
这里的下划线 _ 扮演了“语法糖”的角色，使得数字具有更强的可读性。JavaScript 引擎会直接忽略数字之间的 _，所以 上面两个例子其实是一样的。

但在现实生活中，我们通常会尽量避免写带一长串零的数。因为我们比较懒……我们会尝试将 10 亿写成 "1bn"，或将 73 亿写成 "7.3bn"。对于大多数大的数字来说都是如此。

在 JavaScript 中，我们可以通过在数字后面附加字母 "e" 并指定零的个数来缩短数字：

let billion = 1e9;  // 10 亿，字面意思：数字 1 后面跟 9 个 0

alert( 7.3e9 );  // 73 亿（与 7300000000 和 7_300_000_000 相同）
换句话说，e 把数字乘以 1 后面跟着给定数量的 0 的数字。

1e3 === 1 * 1000; // e3 表示 *1000
1.23e6 === 1.23 * 1000000; // e6 表示 *1000000
现在让我们写一些非常小的数字。例如，1 微秒（百万分之一秒）：

let mcs = 0.000001;
就像以前一样，可以使用 "e" 来完成。如果我们想避免显式地写零，我们可以这样写：

let mcs = 1e-6; // 1 的左边有 6 个 0
如果我们数一下 0.000001 中的 0 的个数，是 6 个。所以自然是 1e-6。

换句话说，e 后面的负数表示除以 1 后面跟着给定数量的 0 的数字：

// -3 除以 1 后面跟着 3 个 0 的数字
1e-3 === 1 / 1000; // 0.001

// -6 除以 1 后面跟着 6 个 0 的数字
1.23e-6 === 1.23 / 1000000; // 0.00000123

// 一个更大一点的数字的示例
1234e-2 === 1234 / 100; // 12.34，小数点移动两次
十六进制，二进制和八进制数字
十六进制 数字在 JavaScript 中被广泛用于表示颜色，编码字符以及其他许多东西。所以自然地，有一种较短的写方法：0x，然后是数字。

例如：

alert( 0xff ); // 255
alert( 0xFF ); // 255（一样，大小写没影响）
二进制和八进制数字系统很少使用，但也支持使用 0b 和 0o 前缀：

let a = 0b11111111; // 二进制形式的 255
let b = 0o377; // 八进制形式的 255

alert( a == b ); // true，两边是相同的数字，都是 255
只有这三种进制支持这种写法。对于其他进制，我们应该使用函数 parseInt（我们将在本章后面看到）。
```
## 2.2 toString(base)
```
方法 num.toString(base) 返回在给定 base 进制数字系统中 num 的字符串表示形式。

举个例子：

let num = 255;

alert( num.toString(16) );  // ff
alert( num.toString(2) );   // 11111111
base 的范围可以从 2 到 36。默认情况下是 10。

常见的用例如下：

base=16 用于十六进制颜色，字符编码等，数字可以是 0..9 或 A..F。

base=2 主要用于调试按位操作，数字可以是 0 或 1。

base=36 是最大进制，数字可以是 0..9 或 A..Z。所有拉丁字母都被用于了表示数字。对于 36 进制来说，一个有趣且有用的例子是，当我们需要将一个较长的数字标识符转换成较短的时候，例如做一个短的 URL。可以简单地使用基数为 36 的数字系统表示：

alert( 123456..toString(36) ); // 2n9c
使用两个点来调用一个方法
请注意 123456..toString(36) 中的两个点不是打错了。如果我们想直接在一个数字上调用一个方法，比如上面例子中的 toString，那么我们需要在它后面放置两个点 ..。

如果我们放置一个点：123456.toString(36)，那么就会出现一个 error，因为 JavaScript 语法隐含了第一个点之后的部分为小数部分。如果我们再放一个点，那么 JavaScript 就知道小数部分为空，现在使用该方法。

也可以写成 (123456).toString(36)。
```
## 2.3 舍入
```
舍入（rounding）是使用数字时最常用的操作之一。

这里有几个对数字进行舍入的内建函数：

Math.floor
向下舍入：3.1 变成 3，-1.1 变成 -2。
Math.ceil
向上舍入：3.1 变成 4，-1.1 变成 -1。
Math.round
向最近的整数舍入：3.1 变成 3，3.6 变成 4，中间值 3.5 变成 4。
Math.trunc（IE 浏览器不支持这个方法）
移除小数点后的所有内容而没有舍入：3.1 变成 3，-1.1 变成 -1。
这个是总结它们之间差异的表格：

Math.floor	Math.ceil	Math.round	Math.trunc
3.1	3	4	3	3
3.6	3	4	4	3
-1.1	-2	-1	-1	-1
-1.6	-2	-1	-2	-1
这些函数涵盖了处理数字小数部分的所有可能方法。但是，如果我们想将数字舍入到小数点后 n 位，该怎么办？

例如，我们有 1.2345，并且想把它舍入到小数点后两位，仅得到 1.23。

有两种方式可以实现这个需求：

乘除法

例如，要将数字舍入到小数点后两位，我们可以将数字乘以 100，调用舍入函数，然后再将其除回。

let num = 1.23456;

alert( Math.round(num * 100) / 100 ); // 1.23456 -> 123.456 -> 123 -> 1.23
函数 toFixed(n) 将数字舍入到小数点后 n 位，并以字符串形式返回结果。

let num = 12.34;
alert( num.toFixed(1) ); // "12.3"
这会向上或向下舍入到最接近的值，类似于 Math.round：

let num = 12.36;
alert( num.toFixed(1) ); // "12.4"
请注意 toFixed 的结果是一个字符串。如果小数部分比所需要的短，则在结尾添加零：

let num = 12.34;
alert( num.toFixed(5) ); // "12.34000"，在结尾添加了 0，以达到小数点后五位
我们可以使用一元加号或 Number() 调用，将其转换为数字，例如 + num.toFixed(5)。
```
## 2.4 不精确的计算
```
在内部，数字是以 64 位格式 IEEE-754 表示的，所以正好有 64 位可以存储一个数字：其中 52 位被用于存储这些数字，其中 11 位用于存储小数点的位置，而 1 位用于符号。

如果一个数字真的很大，则可能会溢出 64 位存储，变成一个特殊的数值 Infinity：

alert( 1e500 ); // Infinity
这可能不那么明显，但经常会发生的是，精度的损失。

考虑下这个（falsy!）相等性测试：

alert( 0.1 + 0.2 == 0.3 ); // false
没错，如果我们检查 0.1 和 0.2 的总和是否为 0.3，我们会得到 false。

奇了怪了！如果不是 0.3，那能是啥？

alert( 0.1 + 0.2 ); // 0.30000000000000004
我擦！想象一下，你创建了一个电子购物网站，如果访问者将价格为 ¥ 0.10 和 ¥ 0.20 的商品放入了他的购物车。订单总额将是 ¥ 0.30000000000000004。这会让任何人感到惊讶。

但为什么会这样呢？

一个数字以其二进制的形式存储在内存中，一个 1 和 0 的序列。但是在十进制数字系统中看起来很简单的 0.1，0.2 这样的小数，实际上在二进制形式中是无限循环小数。

什么是 0.1？0.1 就是 1 除以 10，1/10，即十分之一。在十进制数字系统中，这样的数字表示起来很容易。将其与三分之一进行比较：1/3。三分之一变成了无限循环小数 0.33333(3)。

在十进制数字系统中，可以保证以 10 的整数次幂作为除数能够正常工作，但是以 3 作为除数则不能。也是同样的原因，在二进制数字系统中，可以保证以 2 的整数次幂作为除数时能够正常工作，但 1/10 就变成了一个无限循环的二进制小数。

使用二进制数字系统无法 精确 存储 0.1 或 0.2，就像没有办法将三分之一存储为十进制小数一样。

IEEE-754 数字格式通过将数字舍入到最接近的可能数字来解决此问题。这些舍入规则通常不允许我们看到“极小的精度损失”，但是它确实存在。

我们可以看到：

alert( 0.1.toFixed(20) ); // 0.10000000000000000555
当我们对两个数字进行求和时，它们的“精度损失”会叠加起来。

这就是为什么 0.1 + 0.2 不等于 0.3。

不仅仅是 JavaScript
许多其他编程语言也存在同样的问题。

PHP，Java，C，Perl，Ruby 给出的也是完全相同的结果，因为它们基于的是相同的数字格式。

我们能解决这个问题吗？当然，最可靠的方法是借助方法 toFixed(n) 对结果进行舍入：

let sum = 0.1 + 0.2;
alert( sum.toFixed(2) ); // "0.30"
请注意，toFixed 总是返回一个字符串。它确保小数点后有 2 位数字。如果我们有一个电子购物网站，并需要显示 ¥ 0.30，这实际上很方便。对于其他情况，我们可以使用一元加号将其强制转换为一个数字：

let sum = 0.1 + 0.2;
alert( +sum.toFixed(2) ); // 0.3
我们可以将数字临时乘以 100（或更大的数字），将其转换为整数，进行数学运算，然后再除回。当我们使用整数进行数学运算时，误差会有所减少，但仍然可以在除法中得到：

alert( (0.1 * 10 + 0.2 * 10) / 10 ); // 0.3
alert( (0.28 * 100 + 0.14 * 100) / 100); // 0.4200000000000001
因此，乘/除法可以减少误差，但不能完全消除误差。

有时候我们可以尝试完全避免小数。例如，我们正在创建一个电子购物网站，那么我们可以用角而不是元来存储价格。但是，如果我们要打 30% 的折扣呢？实际上，完全避免小数处理几乎是不可能的。只需要在必要时剪掉其“尾巴”来对其进行舍入即可。

有趣的事儿
尝试运行下面这段代码：

// Hello！我是一个会自我增加的数字！
alert( 9999999999999999 ); // 显示 10000000000000000
出现了同样的问题：精度损失。有 64 位来表示该数字，其中 52 位可用于存储数字，但这还不够。所以最不重要的数字就消失了。

JavaScript 不会在此类事件中触发 error。它会尽最大努力使数字符合所需的格式，但不幸的是，这种格式不够大到满足需求。

两个零
数字内部表示的另一个有趣结果是存在两个零：0 和 -0。

这是因为在存储时，使用一位来存储符号，因此对于包括零在内的任何数字，可以设置这一位或者不设置。

在大多数情况下，这种区别并不明显，因为运算符将它们视为相同的值。
```
## 2.5 测试：isFinite和isNaN
```
还记得这两个特殊的数值吗？

Infinity（和 -Infinity）是一个特殊的数值，比任何数值都大（小）。
NaN 代表一个 error。
它们属于 number 类型，但不是“普通”数字，因此，这里有用于检查它们的特殊函数：

isNaN(value) 将其参数转换为数字，然后测试它是否为 NaN：

alert( isNaN(NaN) ); // true
alert( isNaN("str") ); // true
但是我们需要这个函数吗？我们不能只使用 === NaN 比较吗？很不幸，这不行。值 “NaN” 是独一无二的，它不等于任何东西，包括它自身：

alert( NaN === NaN ); // false
isFinite(value) 将其参数转换为数字，如果是常规数字而不是 NaN/Infinity/-Infinity，则返回 true：

alert( isFinite("15") ); // true
alert( isFinite("str") ); // false，因为是一个特殊的值：NaN
alert( isFinite(Infinity) ); // false，因为是一个特殊的值：Infinity
有时 isFinite 被用于验证字符串值是否为常规数字：

let num = +prompt("Enter a number", '');

// 结果会是 true，除非你输入的是 Infinity、-Infinity 或不是数字
alert( isFinite(num) );
请注意，在所有数字函数中，包括 isFinite，空字符串或仅有空格的字符串均被视为 0。

与 Object.is 进行比较
有一个特殊的内建方法 Object.is，它类似于 === 一样对值进行比较，但它对于两种边缘情况更可靠：

它适用于 NaN：Object.is(NaN, NaN) === true，这是件好事。
值 0 和 -0 是不同的：Object.is(0, -0) === false，从技术上讲这是对的，因为在内部，数字的符号位可能会不同，即使其他所有位均为零。
在所有其他情况下，Object.is(a, b) 与 a === b 相同。

这种比较方式经常被用在 JavaScript 规范中。当内部算法需要比较两个值是否完全相同时，它使用 Object.is（内部称为 SameValue）。
```
## 2.6 parseInt和parseFloat
```
使用加号 + 或 Number() 的数字转换是严格的。如果一个值不完全是一个数字，就会失败：

alert( +"100px" ); // NaN
唯一的例外是字符串开头或结尾的空格，因为它们会被忽略。

但在现实生活中，我们经常会有带有单位的值，例如 CSS 中的 "100px" 或 "12pt"。并且，在很多国家，货币符号是紧随金额之后的，所以我们有 "19€"，并希望从中提取出一个数值。

这就是 parseInt 和 parseFloat 的作用。

它们可以从字符串中“读取”数字，直到无法读取为止。如果发生 error，则返回收集到的数字。函数 parseInt 返回一个整数，而 parseFloat 返回一个浮点数：

alert( parseInt('100px') ); // 100
alert( parseFloat('12.5em') ); // 12.5

alert( parseInt('12.3') ); // 12，只有整数部分被返回了
alert( parseFloat('12.3.4') ); // 12.3，在第二个点出停止了读取
某些情况下，parseInt/parseFloat 会返回 NaN。当没有数字可读时会发生这种情况：

alert( parseInt('a123') ); // NaN，第一个符号停止了读取
parseInt(str, radix) 的第二个参数
parseInt() 函数具有可选的第二个参数。它指定了数字系统的基数，因此 parseInt 还可以解析十六进制数字、二进制数字等的字符串：

alert( parseInt('0xff', 16) ); // 255
alert( parseInt('ff', 16) ); // 255，没有 0x 仍然有效

alert( parseInt('2n9c', 36) ); // 123456
```
## 2.7 其他数学函数
```
JavaScript 有一个内建的 Math 对象，它包含了一个小型的数学函数和常量库。

几个例子：

Math.random()
返回一个从 0 到 1 的随机数（不包括 1）。

alert( Math.random() ); // 0.1234567894322
alert( Math.random() ); // 0.5435252343232
alert( Math.random() ); // ... (任何随机数)
Math.max(a, b, c...) 和 Math.min(a, b, c...)
从任意数量的参数中返回最大值和最小值。

alert( Math.max(3, 5, -10, 0, 1) ); // 5
alert( Math.min(1, 2) ); // 1
Math.pow(n, power)
返回 n 的给定（power）次幂。

alert( Math.pow(2, 10) ); // 2 的 10 次幂 = 1024
Math 对象中还有更多函数和常量，包括三角函数，你可以在 Math 对象文档 中找到这些内容。
```

# 3. 字符串
```javascript
在 JavaScript 中，文本数据被以字符串形式存储，单个字符没有单独的类型。

字符串的内部格式始终是 UTF-16，它不依赖于页面编码。
```
## 3.1 引号
```
让我们回忆一下引号的种类。

字符串可以包含在单引号、双引号或反引号中：

let single = 'single-quoted';
let double = "double-quoted";

let backticks = `backticks`;
单引号和双引号基本相同。但是，反引号允许我们通过 ${…} 将任何表达式嵌入到字符串中：

function sum(a, b) {
  return a + b;
}

alert(`1 + 2 = ${sum(1, 2)}.`); // 1 + 2 = 3.
使用反引号的另一个优点是它们允许字符串跨行：

let guestList = `Guests:
 * John
 * Pete
 * Mary
`;

alert(guestList); // 客人清单，多行
看起来很自然，不是吗？但是单引号和双引号可不能这样做。

如果我们使用单引号或双引号来实现字符串跨行的话，则会出现错误：

let guestList = "Guests: // Error: Unexpected token ILLEGAL
  * John";
单引号和双引号来自语言创建的古老时代，当时没有考虑到多行字符串的需要。反引号出现较晚，因此更通用。

反引号还允许我们在第一个反引号之前指定一个“模版函数”。语法是：func`string`。函数 func 被自动调用，接收字符串和嵌入式表达式，并处理它们。你可以在 docs 中阅读更多关于它们的信息。这叫做 “tagged templates”。此功能可以更轻松地将字符串包装到自定义模版或其他函数中，但这很少使用。
```
## 3.2 特殊字符
```
我们仍然可以通过使用“换行符（newline character）”，以支持使用单引号和双引号来创建跨行字符串。换行符写作 \n，用来表示换行：

let guestList = "Guests:\n * John\n * Pete\n * Mary";

alert(guestList); // 一个多行的客人列表
例如，这两行描述的是一样的，只是书写方式不同：

let str1 = "Hello\nWorld"; // 使用“换行符”创建的两行字符串

// 使用反引号和普通的换行创建的两行字符串
let str2 = `Hello
World`;

alert(str1 == str2); // true
还有其他不常见的“特殊”字符。

这是完整列表：

字符	描述
\n	换行
\r	在 Windows 文本文件中，两个字符 \r\n 的组合代表一个换行。而在非 Windows 操作系统上，它就是 \n。这是历史原因造成的，大多数的 Windows 软件也理解 \n。
\', \"	引号
\\	反斜线
\t	制表符
\b, \f, \v	退格，换页，垂直标签 —— 为了兼容性，现在已经不使用了。
\xXX	具有给定十六进制 Unicode XX 的 Unicode 字符，例如：'\x7A' 和 'z' 相同。
\uXXXX	以 UTF-16 编码的十六进制代码 XXXX 的 Unicode 字符，例如 \u00A9 —— 是版权符号 © 的 Unicode。它必须正好是 4 个十六进制数字。
\u{X…XXXXXX}（1 到 6 个十六进制字符）	具有给定 UTF-32 编码的 Unicode 符号。一些罕见的字符用两个 Unicode 符号编码，占用 4 个字节。这样我们就可以插入长代码了。
Unicode 示例：

alert( "\u00A9" ); // ©
alert( "\u{20331}" ); // 佫，罕见的中国象形文字（长 Unicode）
alert( "\u{1F60D}" ); // 😍，笑脸符号（另一个长 Unicode）
所有的特殊字符都以反斜杠字符 \ 开始。它也被称为“转义字符”。

如果我们想要在字符串中插入一个引号，我们也会使用它。

例如：

alert( 'I\'m the Walrus!' ); // I'm the Walrus!
正如你所看到的，我们必须在内部引号前加上反斜杠 \'，否则它将表示字符串结束。

当然，只有与外部闭合引号相同的引号才需要转义。因此，作为一个更优雅的解决方案，我们可以改用双引号或者反引号：

alert( `I'm the Walrus!` ); // I'm the Walrus!
注意反斜杠 \ 在 JavaScript 中用于正确读取字符串，然后消失。内存中的字符串没有 \。你从上述示例中的 alert 可以清楚地看到这一点。

但是如果我们需要在字符串中显示一个实际的反斜杠 \ 应该怎么做？

我们可以这样做，只需要将其书写两次 \\：

alert( `The backslash: \\` ); // The backslash: \
```
## 3.3 字符串长度
```
length 属性表示字符串长度：

alert( `My\n`.length ); // 3
注意 \n 是一个单独的“特殊”字符，所以长度确实是 3。

length 是一个属性
掌握其他编程语言的人，有时会错误地调用 str.length() 而不是 str.length。这是行不通的。

请注意 str.length 是一个数字属性，而不是函数。后面不需要添加括号。
```
## 3.4 访问字符
```
要获取在 pos 位置的一个字符，可以使用方括号 [pos] 或者调用 str.charAt(pos) 方法。第一个字符从零位置开始：

let str = `Hello`;

// 第一个字符
alert( str[0] ); // H
alert( str.charAt(0) ); // H

// 最后一个字符
alert( str[str.length - 1] ); // o
方括号是获取字符的一种现代化方法，而 charAt 是历史原因才存在的。

它们之间的唯一区别是，如果没有找到字符，[] 返回 undefined，而 charAt 返回一个空字符串：

let str = `Hello`;

alert( str[1000] ); // undefined
alert( str.charAt(1000) ); // ''（空字符串）
我们也可以使用 for..of 遍历字符：

for (let char of "Hello") {
  alert(char); // H,e,l,l,o（char 变为 "H"，然后是 "e"，然后是 "l" 等）
}
```
## 3.5 字符串是不可变的
```
在 JavaScript 中，字符串不可更改。改变字符是不可能的。

我们证明一下为什么不可能：

let str = 'Hi';

str[0] = 'h'; // error
alert( str[0] ); // 无法运行
通常的解决方法是创建一个新的字符串，并将其分配给 str 而不是以前的字符串。

例如：

let str = 'Hi';

str = 'h' + str[1];  // 替换字符串

alert( str ); // hi
在接下来的章节，我们将看到更多相关示例。
```
## 3.6 改变大小写
```
toLowerCase() 和 toUpperCase() 方法可以改变大小写：

alert( 'Interface'.toUpperCase() ); // INTERFACE
alert( 'Interface'.toLowerCase() ); // interface
或者我们想要使一个字符变成小写：

alert( 'Interface'[0].toLowerCase() ); // 'i'
```
## 3.7 查找子字符串
```
在字符串中查找子字符串有很多种方法。

str.indexOf
第一个方法是 str.indexOf(substr, pos)。

它从给定位置 pos 开始，在 str 中查找 substr，如果没有找到，则返回 -1，否则返回匹配成功的位置。

例如：

let str = 'Widget with id';

alert( str.indexOf('Widget') ); // 0，因为 'Widget' 一开始就被找到
alert( str.indexOf('widget') ); // -1，没有找到，检索是大小写敏感的

alert( str.indexOf("id") ); // 1，"id" 在位置 1 处（……idget 和 id）
可选的第二个参数允许我们从一个给定的位置开始检索。

例如，"id" 第一次出现的位置是 1。查询下一个存在位置时，我们从 2 开始检索：

let str = 'Widget with id';

alert( str.indexOf('id', 2) ) // 12
如果我们对所有存在位置都感兴趣，可以在一个循环中使用 indexOf。每一次新的调用都发生在上一匹配位置之后：

let str = 'As sly as a fox, as strong as an ox';

let target = 'as'; // 这是我们要查找的目标

let pos = 0;
while (true) {
  let foundPos = str.indexOf(target, pos);
  if (foundPos == -1) break;

  alert( `Found at ${foundPos}` );
  pos = foundPos + 1; // 继续从下一个位置查找
}
相同的算法可以简写：

let str = "As sly as a fox, as strong as an ox";
let target = "as";

let pos = -1;
while ((pos = str.indexOf(target, pos + 1)) != -1) {
  alert( pos );
}
str.lastIndexOf(substr, pos)
还有一个类似的方法 str.lastIndexOf(substr, position)，它从字符串的末尾开始搜索到开头。

它会以相反的顺序列出这些事件。

在 if 测试中 indexOf 有一点不方便。我们不能像这样把它放在 if 中：

let str = "Widget with id";

if (str.indexOf("Widget")) {
    alert("We found it"); // 不工作！
}
上述示例中的 alert 不会显示，因为 str.indexOf("Widget") 返回 0（意思是它在起始位置就查找到了匹配项）。是的，但是 if 认为 0 表示 false。

因此我们应该检查 -1，像这样：

let str = "Widget with id";

if (str.indexOf("Widget") != -1) {
    alert("We found it"); // 现在工作了！
}
按位（bitwise）NOT 技巧
这里使用的一个老技巧是 bitwise NOT ~ 运算符。它将数字转换为 32-bit 整数（如果存在小数部分，则删除小数部分），然后对其二进制表示形式中的所有位均取反。

实际上，这意味着一件很简单的事儿：对于 32-bit 整数，~n 等于 -(n+1)。

例如：

alert( ~2 ); // -3，和 -(2+1) 相同
alert( ~1 ); // -2，和 -(1+1) 相同
alert( ~0 ); // -1，和 -(0+1) 相同
alert( ~-1 ); // 0，和 -(-1+1) 相同
正如我们看到这样，只有当 n == -1 时，~n 才为零（适用于任何 32-bit 带符号的整数 n）。

因此，仅当 indexOf 的结果不是 -1 时，检查 if ( ~str.indexOf("...") ) 才为真。换句话说，当有匹配时。

人们用它来简写 indexOf 检查：

let str = "Widget";

if (~str.indexOf("Widget")) {
  alert( 'Found it!' ); // 正常运行
}
通常不建议以非显而易见的方式使用语言特性，但这种特殊技巧在旧代码中仍被广泛使用，所以我们应该理解它。

只要记住：if (~str.indexOf(...)) 读作 “if found”。

确切地说，由于 ~ 运算符将大数字截断为 32 位，因此存在给出 0 的其他数字，最小的数字是 ~4294967295=0。这使得这种检查只有在字符串没有那么长的情况下才是正确的。

现在我们只会在旧的代码中看到这个技巧，因为现代 JavaScript 提供了 .includes 方法（见下文）。

includes，startsWith，endsWith
更现代的方法 str.includes(substr, pos) 根据 str 中是否包含 substr 来返回 true/false。

如果我们需要检测匹配，但不需要它的位置，那么这是正确的选择：

alert( "Widget with id".includes("Widget") ); // true

alert( "Hello".includes("Bye") ); // false
str.includes 的第二个可选参数是开始搜索的起始位置：

alert( "Widget".includes("id") ); // true
alert( "Widget".includes("id", 3) ); // false, 从位置 3 开始没有 "id"
方法 str.startsWith 和 str.endsWith 的功能与其名称所表示的意思相同：

alert( "Widget".startsWith("Wid") ); // true，"Widget" 以 "Wid" 开始
alert( "Widget".endsWith("get") ); // true，"Widget" 以 "get" 结束
```
## 3.8 获取子字符串
```
JavaScript 中有三种获取字符串的方法：substring、substr 和 slice。

str.slice(start [, end])
返回字符串从 start 到（但不包括）end 的部分。

例如：

let str = "stringify";
alert( str.slice(0, 5) ); // 'strin'，从 0 到 5 的子字符串（不包括 5）
alert( str.slice(0, 1) ); // 's'，从 0 到 1，但不包括 1，所以只有在 0 处的字符
如果没有第二个参数，slice 会一直运行到字符串末尾：

let str = "stringify";
alert( str.slice(2) ); // 从第二个位置直到结束
start/end 也有可能是负值。它们的意思是起始位置从字符串结尾计算：

let str = "stringify";

// 从右边的第四个位置开始，在右边的第一个位置结束
alert( str.slice(-4, -1) ); // 'gif'
str.substring(start [, end])
返回字符串从 start 到（但不包括）end 的部分。

这与 slice 几乎相同，但它允许 start 大于 end。

例如：

let str = "stringify";

// 这些对于 substring 是相同的
alert( str.substring(2, 6) ); // "ring"
alert( str.substring(6, 2) ); // "ring"

// ……但对 slice 是不同的：
alert( str.slice(2, 6) ); // "ring"（一样）
alert( str.slice(6, 2) ); // ""（空字符串）
不支持负参数（不像 slice），它们被视为 0。

str.substr(start [, length])
返回字符串从 start 开始的给定 length 的部分。

与以前的方法相比，这个允许我们指定 length 而不是结束位置：

let str = "stringify";
alert( str.substr(2, 4) ); // 'ring'，从位置 2 开始，获取 4 个字符
第一个参数可能是负数，从结尾算起：

let str = "stringify";
alert( str.substr(-4, 2) ); // 'gi'，从第 4 位获取 2 个字符
我们回顾一下这些方法，以免混淆：

方法	选择方式……	负值参数
slice(start, end)	从 start 到 end（不含 end）	允许
substring(start, end)	从 start 到 end（不含 end）	负值被视为 0
substr(start, length)	从 start 开始获取长为 length 的字符串	允许 start 为负数
使用哪一个？
它们都可用于获取子字符串。正式一点来讲，substr 有一个小缺点：它不是在 JavaScript 核心规范中描述的，而是在附录 B 中。附录 B 的内容主要是描述因历史原因而遗留下来的仅浏览器特性。因此，理论上非浏览器环境可能无法支持 substr，但实际上它在别的地方也都能用。

相较于其他两个变体，slice 稍微灵活一些，它允许以负值作为参数并且写法更简短。因此仅仅记住这三种方法中的 slice 就足够了。
```
## 3.9 比较字符串
```
正如我们从 值的比较 一章中了解到的，字符串按字母顺序逐字比较。

不过，也有一些奇怪的地方。

小写字母总是大于大写字母：

alert( 'a' > 'Z' ); // true
带变音符号的字母存在“乱序”的情况：

alert( 'Österreich' > 'Zealand' ); // true
如果我们对这些国家名进行排序，可能会导致奇怪的结果。通常，人们会期望 Zealand 在名单中的 Österreich 之后出现。

为了明白发生了什么，我们回顾一下在 JavaScript 中字符串的内部表示。

所有的字符串都使用 UTF-16 编码。即：每个字符都有对应的数字代码。有特殊的方法可以获取代码表示的字符，以及字符对应的代码。

str.codePointAt(pos)
返回在 pos 位置的字符代码 :

// 不同的字母有不同的代码
alert( "z".codePointAt(0) ); // 122
alert( "Z".codePointAt(0) ); // 90
String.fromCodePoint(code)
通过数字 code 创建字符

alert( String.fromCodePoint(90) ); // Z
我们还可以用 \u 后跟十六进制代码，通过这些代码添加 Unicode 字符：

// 在十六进制系统中 90 为 5a
alert( '\u005a' ); // Z
现在我们看一下代码为 65..220 的字符（拉丁字母和一些额外的字符），方法是创建一个字符串：

let str = '';

for (let i = 65; i <= 220; i++) {
  str += String.fromCodePoint(i);
}
alert( str );
// ABCDEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrstuvwxyz{|}~
// ¡¢£¤¥¦§¨©ª«¬­®¯°±²³´µ¶·¸¹º»¼½¾¿ÀÁÂÃÄÅÆÇÈÉÊËÌÍÎÏÐÑÒÓÔÕÖ×ØÙÚÛÜ
看到没？先是大写字符，然后是一些特殊字符，然后是小写字符，而 Ö 几乎是最后输出。

现在很明显为什么 a > Z。

字符通过数字代码进行比较。越大的代码意味着字符越大。a（97）的代码大于 Z（90）的代码。

所有小写字母追随在大写字母之后，因为它们的代码更大。
一些像 Ö 的字母与主要字母表不同。这里，它的代码比任何从 a 到 z 的代码都要大。
正确的比较
执行字符串比较的“正确”算法比看起来更复杂，因为不同语言的字母都不相同。

因此浏览器需要知道要比较的语言。

幸运的是，所有现代浏览器（IE10- 需要额外的库 Intl.JS) 都支持国际化标准 ECMA-402。

它提供了一种特殊的方法来比较不同语言的字符串，遵循它们的规则。

调用 str.localeCompare(str2) 会根据语言规则返回一个整数，这个整数能指示字符串 str 在排序顺序中排在字符串 str2 前面、后面、还是相同：

如果 str 排在 str2 前面，则返回负数。
如果 str 排在 str2 后面，则返回正数。
如果它们在相同位置，则返回 0。
例如：

alert( 'Österreich'.localeCompare('Zealand') ); // -1
这个方法实际上在 文档 中指定了两个额外的参数，这两个参数允许它指定语言（默认语言从环境中获取，字符顺序视语言不同而不同）并设置诸如区分大小写，或应该将 "a" 和 "á" 作相同处理等附加的规则。
```
## 3.10 内部，Unicode
```
进阶内容
这部分会深入字符串内部。如果你计划处理 emoji、罕见的数学或象形文字或其他罕见的符号，这些知识会对你有用。

如果你不打算支持它们，你可以跳过这一部分。

代理对
所有常用的字符都是一个 2 字节的代码。大多数欧洲语言，数字甚至大多数象形文字中的字母都有 2 字节的表示形式。

但 2 字节只允许 65536 个组合，这对于表示每个可能的符号是不够的。所以稀有的符号被称为“代理对”的一对 2 字节的符号编码。

这些符号的长度是 2：

alert( '𝒳'.length ); // 2，大写数学符号 X
alert( '😂'.length ); // 2，笑哭表情
alert( '𩷶'.length ); // 2，罕见的中国象形文字
注意，代理对在 JavaScript 被创建时并不存在，因此无法被编程语言正确处理！

我们实际上在上面的每个字符串中都有一个符号，但 length 显示长度为 2。

String.fromCodePoint 和 str.codePointAt 是几种处理代理对的少数方法。它们最近才出现在编程语言中。在它们之前，只有 String.fromCharCode 和 str.charCodeAt。这些方法实际上与 fromCodePoint/codePointAt 相同，但是不适用于代理对。

获取符号可能会非常麻烦，因为代理对被认为是两个字符：

alert( '𝒳'[0] ); // 奇怪的符号……
alert( '𝒳'[1] ); // ……代理对的一块
请注意，代理对的各部分没有任何意义。因此，上述示例中的 alert 显示的实际上是垃圾信息。

技术角度来说，代理对也是可以通过它们的代码检测到的：如果一个字符的代码在 0xd800..0xdbff 范围内，那么它是代理对的第一部分。下一个字符（第二部分）必须在 0xdc00..0xdfff 范围中。这些范围是按照标准专门为代理对保留的。

在上述示例中：

// charCodeAt 不理解代理对，所以它给出了代理对的代码

alert( '𝒳'.charCodeAt(0).toString(16) ); // d835，在 0xd800 和 0xdbff 之间
alert( '𝒳'.charCodeAt(1).toString(16) ); // dcb3, 在 0xdc00 和 0xdfff 之间
本章节后面的 Iterable object（可迭代对象） 章节中，你可以找到更多处理代理对的方法。可能有专门的库，这里没有什么足够好的建议了。

变音符号与规范化
在许多语言中，都有一些由基本字符组成的符号，在其上方/下方有一个标记。

例如，字母 a 可以是 àáâäãåā 的基本字符。最常见的“复合”字符在 UTF-16 表中都有自己的代码。但不是全部，因为可能的组合太多。

为了支持任意组合，UTF-16 允许我们使用多个 Unicode 字符：基本字符紧跟“装饰”它的一个或多个“标记”字符。

例如，如果我们 S 后跟有特殊的 “dot above” 字符（代码 \u0307），则显示 Ṡ。

alert( 'S\u0307' ); // Ṡ
如果我们需要在字母上方（或下方）添加额外的标记 —— 没问题，只需要添加必要的标记字符即可。

例如，如果我们追加一个字符 “dot below”（代码 \u0323），那么我们将得到“S 上面和下面都有点”的字符：Ṩ。

例如：

alert( 'S\u0307\u0323' ); // Ṩ
这在提供良好灵活性的同时，也存在一个有趣的问题：两个视觉上看起来相同的字符，可以用不同的 Unicode 组合表示。

例如：

let s1 = 'S\u0307\u0323'; // Ṩ，S + 上点 + 下点
let s2 = 'S\u0323\u0307'; // Ṩ，S + 下点 + 上点

alert( `s1: ${s1}, s2: ${s2}` );

alert( s1 == s2 ); // false，尽管字符看起来相同（?!）
为了解决这个问题，有一个 “Unicode 规范化”算法，它将每个字符串都转化成单个“通用”格式。

它由 str.normalize() 实现。

alert( "S\u0307\u0323".normalize() == "S\u0323\u0307".normalize() ); // true
有趣的是，在实际情况下，normalize() 实际上将一个由 3 个字符组成的序列合并为一个：\u1e68（S 有两个点）。

alert( "S\u0307\u0323".normalize().length ); // 1

alert( "S\u0307\u0323".normalize() == "\u1e68" ); // true
事实上，情况并非总是如此，因为符号 Ṩ 是“常用”的，所以 UTF-16 创建者把它包含在主表中并给它了对应的代码。

如果你想了解更多关于规范化规则和变体的信息 —— 它们在 Unicode 标准附录中有详细描述：Unicode 规范化形式，但对于大多数实际目的来说，本文的内容就已经足够了。
```

# 4. 数组
```javascript
对象允许存储键值集合，这很好。

但很多时候我们发现还需要 有序集合，里面的元素都是按顺序排列的。例如，我们可能需要存储一些列表，比如用户、商品以及 HTML 元素等。

这里使用对象就不是很方便了，因为对象不能提供能够管理元素顺序的方法。我们不能在已有的元素“之间”插入一个新的属性。这种场景下对象就不太适用了。

这时一个特殊的数据结构数组（Array）就派上用场了，它能存储有序的集合。
```
## 4.1 声明
```
创建一个空数组有两种语法：

let arr = new Array();
let arr = [];
绝大多数情况下使用的都是第二种语法。我们可以在方括号中添加初始元素：

let fruits = ["Apple", "Orange", "Plum"];
数组元素从 0 开始编号。

我们可以通过方括号中的数字获取元素：

let fruits = ["Apple", "Orange", "Plum"];

alert( fruits[0] ); // Apple
alert( fruits[1] ); // Orange
alert( fruits[2] ); // Plum
可以替换元素：

fruits[2] = 'Pear'; // 现在变成了 ["Apple", "Orange", "Pear"]
……或者向数组新加一个元素：

fruits[3] = 'Lemon'; // 现在变成 ["Apple", "Orange", "Pear", "Lemon"]
length 属性的值是数组中元素的总个数：

let fruits = ["Apple", "Orange", "Plum"];

alert( fruits.length ); // 3
也可以用 alert 来显示整个数组。

let fruits = ["Apple", "Orange", "Plum"];

alert( fruits ); // Apple,Orange,Plum
数组可以存储任何类型的元素。

例如:

// 混合值
let arr = [ 'Apple', { name: 'John' }, true, function() { alert('hello'); } ];

// 获取索引为 1 的对象然后显示它的 name
alert( arr[1].name ); // John

// 获取索引为 3 的函数并执行
arr[3](); // hello
以逗号结尾
数组就像对象一样，可以以逗号结尾：

let fruits = [
  "Apple",
  "Orange",
  "Plum",
];
因为每一行都是相似的，所以这种以“逗号结尾”的方式使得插入/移除项变得更加简单。
```
## 4.2 使用"at"获取最后一个元素
```
最近新增的特性
这是一个最近添加到 JavaScript 的特性。 旧式浏览器可能需要 polyfills.
假设我们想要数组的最后一个元素。

一些编程语言允许我们使用负数索引来实现这一点，例如 fruits[-1]。

但在 JavaScript 中这行不通。结果将是 undefined，因为方括号中的索引是被按照其字面意思处理的。

我们可以显式地计算最后一个元素的索引，然后访问它：fruits[fruits.length - 1]。

let fruits = ["Apple", "Orange", "Plum"];

alert( fruits[fruits.length-1] ); // Plum
有点麻烦，不是吗？我们需要写两次变量名。

幸运的是，这里有一个更简短的语法 fruits.at(-1)：

let fruits = ["Apple", "Orange", "Plum"];

// 与 fruits[fruits.length-1] 相同
alert( fruits.at(-1) ); // Plum
换句话说，arr.at(i)：

如果 i >= 0，则与 arr[i] 完全相同。
对于 i 为负数的情况，它则从数组的尾部向前数。
```
## 4.3 pop/push, shift/unshift方法
```
队列（queue）是最常见的使用数组的方法之一。在计算机科学中，这表示支持两个操作的一个有序元素的集合：

push 在末端添加一个元素.
shift 取出队列首端的一个元素，整个队列往前移，这样原先排第二的元素现在排在了第一。

这两种操作数组都支持。

队列的应用在实践中经常会碰到。例如需要在屏幕上显示消息队列。

数组还有另一个用例，就是数据结构 栈。

它支持两种操作：

push 在末端添加一个元素.
pop 从末端取出一个元素.
所以新元素的添加和取出都是从“末端”开始的。

栈通常被被形容成一叠卡片：要么在最上面添加卡片，要么从最上面拿走卡片：


对于栈来说，最后放进去的内容是最先接收的，也叫做 LIFO（Last-In-First-Out），即后进先出法则。而与队列相对应的叫做 FIFO（First-In-First-Out），即先进先出。

JavaScript 中的数组既可以用作队列，也可以用作栈。它们允许你从首端/末端来添加/删除元素。

这在计算机科学中，允许这样的操作的数据结构被称为 双端队列（deque）。

作用于数组末端的方法：

pop
取出并返回数组的最后一个元素：

let fruits = ["Apple", "Orange", "Pear"];

alert( fruits.pop() ); // 移除 "Pear" 然后 alert 显示出来

alert( fruits ); // Apple, Orange
fruits.pop() 和 fruits.at(-1) 都返回数组的最后一个元素，但 fruits.pop() 同时也删除了数组的最后一个元素，进而修改了原数组。

push
在数组末端添加元素：

let fruits = ["Apple", "Orange"];

fruits.push("Pear");

alert( fruits ); // Apple, Orange, Pear
调用 fruits.push(...) 与 fruits[fruits.length] = ... 是一样的。

作用于数组首端的方法：

shift
取出数组的第一个元素并返回它：

let fruits = ["Apple", "Orange", "Pear"];

alert( fruits.shift() ); // 移除 Apple 然后 alert 显示出来

alert( fruits ); // Orange, Pear
unshift
在数组的首端添加元素：

let fruits = ["Orange", "Pear"];

fruits.unshift('Apple');

alert( fruits ); // Apple, Orange, Pear
push 和 unshift 方法都可以一次添加多个元素：

let fruits = ["Apple"];

fruits.push("Orange", "Peach");
fruits.unshift("Pineapple", "Lemon");

// ["Pineapple", "Lemon", "Apple", "Orange", "Peach"]
alert( fruits );
```
## 4.4 内部
```
数组是一种特殊的对象。使用方括号来访问属性 arr[0] 实际上是来自于对象的语法。它其实与 obj[key] 相同，其中 arr 是对象，而数字用作键（key）。

它们扩展了对象，提供了特殊的方法来处理有序的数据集合以及 length 属性。但从本质上讲，它仍然是一个对象。

记住，在 JavaScript 中只有 8 种基本的数据类型（详见 数据类型 一章）。数组是一个对象，因此其行为也像一个对象。

例如，它是通过引用来复制的：

let fruits = ["Banana"]

let arr = fruits; // 通过引用复制 (两个变量引用的是相同的数组)

alert( arr === fruits ); // true

arr.push("Pear"); // 通过引用修改数组

alert( fruits ); // Banana, Pear — 现在有 2 项了
……但是数组真正特殊的是它们的内部实现。JavaScript 引擎尝试把这些元素一个接一个地存储在连续的内存区域，就像本章的插图显示的一样，而且还有一些其它的优化，以使数组运行得非常快。

但是，如果我们不像“有序集合”那样使用数组，而是像常规对象那样使用数组，这些就都不生效了。

例如，从技术上讲，我们可以这样做:

let fruits = []; // 创建一个数组

fruits[99999] = 5; // 分配索引远大于数组长度的属性

fruits.age = 25; // 创建一个具有任意名称的属性
这是可以的，因为数组是基于对象的。我们可以给它们添加任何属性。

但是 Javascript 引擎会发现，我们在像使用常规对象一样使用数组，那么针对数组的优化就不再适用了，然后对应的优化就会被关闭，这些优化所带来的优势也就荡然无存了。

数组误用的几种方式:

添加一个非数字的属性，比如 arr.test = 5。
制造空洞，比如：添加 arr[0]，然后添加 arr[1000] (它们中间什么都没有)。
以倒序填充数组，比如 arr[1000]，arr[999] 等等。
请将数组视为作用于 有序数据 的特殊结构。它们为此提供了特殊的方法。数组在 JavaScript 引擎内部是经过特殊调整的，使得更好地作用于连续的有序数据，所以请以正确的方式使用数组。如果你需要任意键值，那很有可能实际上你需要的是常规对象 {}。
```
## 4.5 性能
```
push/pop 方法运行的比较快，而 shift/unshift 比较慢。


为什么作用于数组的末端会比首端快呢？让我们看看在执行期间都发生了什么：

fruits.shift(); // 从首端取出一个元素
只获取并移除索引 0 对应的元素是不够的。其它元素也需要被重新编号。

shift 操作必须做三件事:

移除索引为 0 的元素。
把所有的元素向左移动，把索引 1 改成 0，2 改成 1 以此类推，对其重新编号。
更新 length 属性。

数组里的元素越多，移动它们就要花越多的时间，也就意味着越多的内存操作。

unshift 也是一样：为了在数组的首端添加元素，我们首先需要将现有的元素向右移动，增加它们的索引值。

那 push/pop 是什么样的呢？它们不需要移动任何东西。如果从末端移除一个元素，pop 方法只需要清理索引值并缩短 length 就可以了。

pop 操作的行为：

fruits.pop(); // 从末端取走一个元素

pop 方法不需要移动任何东西，因为其它元素都保留了各自的索引。这就是为什么 pop 会特别快。

push 方法也是一样的。
```
## 4.6 循环
```
遍历数组最古老的方式就是 for 循环：

let arr = ["Apple", "Orange", "Pear"];

for (let i = 0; i < arr.length; i++) {
  alert( arr[i] );
}
但对于数组来说还有另一种循环方式，for..of：

let fruits = ["Apple", "Orange", "Plum"];

// 遍历数组元素
for (let fruit of fruits) {
  alert( fruit );
}
for..of 不能获取当前元素的索引，只是获取元素值，但大多数情况是够用的。而且这样写更短。

技术上来讲，因为数组也是对象，所以使用 for..in 也是可以的：

let arr = ["Apple", "Orange", "Pear"];

for (let key in arr) {
  alert( arr[key] ); // Apple, Orange, Pear
}
但这其实是一个很不好的想法。会有一些潜在问题存在：

for..in 循环会遍历 所有属性，不仅仅是这些数字属性。

在浏览器和其它环境中有一种称为“类数组”的对象，它们 看似是数组。也就是说，它们有 length 和索引属性，但是也可能有其它的非数字的属性和方法，这通常是我们不需要的。for..in 循环会把它们都列出来。所以如果我们需要处理类数组对象，这些“额外”的属性就会存在问题。

for..in 循环适用于普通对象，并且做了对应的优化。但是不适用于数组，因此速度要慢 10-100 倍。当然即使是这样也依然非常快。只有在遇到瓶颈时可能会有问题。但是我们仍然应该了解这其中的不同。

通常来说，我们不应该用 for..in 来处理数组。
```
## 4.7 关于"length"
```
当我们修改数组的时候，length 属性会自动更新。准确来说，它实际上不是数组里元素的个数，而是最大的数字索引值加一。

例如，一个数组只有一个元素，但是这个元素的索引值很大，那么这个数组的 length 也会很大：

let fruits = [];
fruits[123] = "Apple";

alert( fruits.length ); // 124
要知道的是我们通常不会这样使用数组。

length 属性的另一个有意思的点是它是可写的。

如果我们手动增加它，则不会发生任何有趣的事儿。但是如果我们减少它，数组就会被截断。该过程是不可逆的，下面是例子：

let arr = [1, 2, 3, 4, 5];

arr.length = 2; // 截断到只剩 2 个元素
alert( arr ); // [1, 2]

arr.length = 5; // 又把 length 加回来
alert( arr[3] ); // undefined：被截断的那些数值并没有回来
所以，清空数组最简单的方法就是：arr.length = 0;。
```
## 4.8 new Array()
```
这是创建数组的另一种语法：

let arr = new Array("Apple", "Pear", "etc");
它很少被使用，因为方括号 [] 更短更简洁。而且，这种语法还有一个棘手的特性。

如果使用单个参数（即数字）调用 new Array，那么它会创建一个 指定了长度，却没有任何项 的数组。

让我们看看如何搬起石头砸自己的脚:

let arr = new Array(2); // 会创建一个 [2] 的数组吗？

alert( arr[0] ); // undefined！没有元素。

alert( arr.length ); // length 2
为了避免这种意外情况，我们通常使用方括号，除非我们真的知道自己在做什么。
```
## 4.9 多维数组
```
数组里的项也可以是数组。我们可以将其用于多维数组，例如存储矩阵：

let matrix = [
  [1, 2, 3],
  [4, 5, 6],
  [7, 8, 9]
];

alert( matrix[1][1] ); // 最中间的那个数
```
## 4.10 toString
```
数组有自己的 toString 方法的实现，会返回以逗号隔开的元素列表。

例如：

let arr = [1, 2, 3];

alert( arr ); // 1,2,3
alert( String(arr) === '1,2,3' ); // true
此外，我们试试运行一下这个：

alert( [] + 1 ); // "1"
alert( [1] + 1 ); // "11"
alert( [1,2] + 1 ); // "1,21"
数组没有 Symbol.toPrimitive，也没有 valueOf，它们只能执行 toString 进行转换，所以这里 [] 就变成了一个空字符串，[1] 变成了 "1"，[1,2] 变成了 "1,2"。

当 "+" 运算符把一些项加到字符串后面时，加号后面的项也会被转换成字符串，所以下一步就会是这样：

alert( "" + 1 ); // "1"
alert( "1" + 1 ); // "11"
alert( "1,2" + 1 ); // "1,21"
```
## 4.11 不要使用==比较数组
```
JavaScript 中的数组与其它一些编程语言的不同，不应该使用 == 运算符比较 JavaScript 中的数组。

该运算符不会对数组进行特殊处理，它会像处理任意对象那样处理数组。

让我们回顾一下规则：

仅当两个对象引用的是同一个对象时，它们才相等 ==。
如果 == 左右两个参数之中有一个参数是对象，另一个参数是原始类型，那么该对象将会被转换为原始类型，转换规则如 对象 —— 原始值转换 一章所述。
……null 和 undefined 相等 ==，且各自不等于任何其他的值。
严格比较 === 更简单，因为它不会进行类型转换。

所以，如果我们使用 == 来比较数组，除非我们比较的是两个引用同一数组的变量，否则它们永远不相等。

例如：

alert( [] == [] ); // false
alert( [0] == [0] ); // false
从技术上讲，这些数组是不同的对象。所以它们不相等。== 运算符不会进行逐项比较。

与原始类型的比较也可能会产生看似很奇怪的结果：

alert( 0 == [] ); // true

alert('0' == [] ); // false
在这里的两个例子中，我们将原始类型和数组对象进行比较。因此，数组 [] 被转换为原始类型以进行比较，被转换成了一个空字符串 ''。

然后，接下来的比较就是原始类型之间的比较，如 类型转换 一章所述：

// 在 [] 被转换为 '' 后
alert( 0 == '' ); // true，因为 '' 被转换成了数字 0

alert('0' == '' ); // false，没有进一步的类型转换，是不同的字符串
那么，我们应该如何对数组进行比较呢？

很简单，不要使用 == 运算符。而是，可以在循环中或者使用下一章中我们将介绍的迭代方法逐项地比较它们。
```

# 5. 数组方法
```javascript
数组提供的方法有很多。为了方便起见，在本章中，我们将按组讲解。
```
## 5.1 添加/移除数组元素
```
我们已经学了从数组的首端或尾端添加和删除元素的方法：

arr.push(...items) —— 从尾端添加元素，
arr.pop() —— 从尾端提取元素，
arr.shift() —— 从首端提取元素，
arr.unshift(...items) —— 从首端添加元素。
这里还有其他几种方法。

splice
如何从数组中删除元素？

数组是对象，所以我们可以尝试使用 delete：

let arr = ["I", "go", "home"];

delete arr[1]; // remove "go"

alert( arr[1] ); // undefined

// now arr = ["I",  , "home"];
alert( arr.length ); // 3
元素被删除了，但数组仍然有 3 个元素，我们可以看到 arr.length == 3。

这很正常，因为 delete obj.key 是通过 key 来移除对应的值。对于对象来说是可以的。但是对于数组来说，我们通常希望剩下的元素能够移动并占据被释放的位置。我们希望得到一个更短的数组。

所以应该使用特殊的方法。

arr.splice 方法可以说是处理数组的瑞士军刀。它可以做所有事情：添加，删除和插入元素。

语法是：

arr.splice(start[, deleteCount, elem1, ..., elemN])
它从索引 start 开始修改 arr：删除 deleteCount 个元素并在当前位置插入 elem1, ..., elemN。最后返回被删除的元素所组成的数组。

通过例子我们可以很容易地掌握这个方法。

让我们从删除开始：

let arr = ["I", "study", "JavaScript"];

arr.splice(1, 1); // 从索引 1 开始删除 1 个元素

alert( arr ); // ["I", "JavaScript"]
简单，对吧？从索引 1 开始删除 1 个元素。（译注：当只填写了 splice 的 start 参数时，将删除从索引 start 开始的所有数组项）

在下一个例子中，我们删除了 3 个元素，并用另外两个元素替换它们：

let arr = ["I", "study", "JavaScript", "right", "now"];

// 删除数组的前三项，并使用其他内容代替它们
arr.splice(0, 3, "Let's", "dance");

alert( arr ) // 现在 ["Let's", "dance", "right", "now"]
在这里我们可以看到 splice 返回了被删除的元素所组成的数组：

let arr = ["I", "study", "JavaScript", "right", "now"];

// 删除前两个元素
let removed = arr.splice(0, 2);

alert( removed ); // "I", "study" <-- 被从数组中删除了的元素
我们可以将 deleteCount 设置为 0，splice 方法就能够插入元素而不用删除任何元素：

let arr = ["I", "study", "JavaScript"];

// 从索引 2 开始
// 删除 0 个元素
// 然后插入 "complex" 和 "language"
arr.splice(2, 0, "complex", "language");

alert( arr ); // "I", "study", "complex", "language", "JavaScript"
允许负向索引
在这里和其他数组方法中，负向索引都是被允许的。它们从数组末尾计算位置，如下所示：

let arr = [1, 2, 5];

// 从索引 -1（尾端前一位）
// 删除 0 个元素，
// 然后插入 3 和 4
arr.splice(-1, 0, 3, 4);

alert( arr ); // 1,2,3,4,5
slice
arr.slice 方法比 arr.splice 简单得多。

语法是：

arr.slice([start], [end])
它会返回一个新数组，将所有从索引 start 到 end（不包括 end）的数组项复制到一个新的数组。start 和 end 都可以是负数，在这种情况下，从末尾计算索引。

它和字符串的 str.slice 方法有点像，就是把子字符串替换成子数组。

例如：

let arr = ["t", "e", "s", "t"];

alert( arr.slice(1, 3) ); // e,s（复制从位置 1 到位置 3 的元素）

alert( arr.slice(-2) ); // s,t（复制从位置 -2 到尾端的元素）
我们也可以不带参数地调用它：arr.slice() 会创建一个 arr 的副本。其通常用于获取副本，以进行不影响原始数组的进一步转换。

concat
arr.concat 创建一个新数组，其中包含来自于其他数组和其他项的值。

语法：

arr.concat(arg1, arg2...)
它接受任意数量的参数 —— 数组或值都可以。

结果是一个包含来自于 arr，然后是 arg1，arg2 的元素的新数组。

如果参数 argN 是一个数组，那么其中的所有元素都会被复制。否则，将复制参数本身。

例如：

let arr = [1, 2];

// 从 arr 和 [3,4] 创建一个新数组
alert( arr.concat([3, 4]) ); // 1,2,3,4

// 从 arr、[3,4] 和 [5,6] 创建一个新数组
alert( arr.concat([3, 4], [5, 6]) ); // 1,2,3,4,5,6

// 从 arr、[3,4]、5 和 6 创建一个新数组
alert( arr.concat([3, 4], 5, 6) ); // 1,2,3,4,5,6
通常，它只复制数组中的元素。其他对象，即使它们看起来像数组一样，但仍然会被作为一个整体添加：

let arr = [1, 2];

let arrayLike = {
  0: "something",
  length: 1
};

alert( arr.concat(arrayLike) ); // 1,2,[object Object]
……但是，如果类数组对象具有 Symbol.isConcatSpreadable 属性，那么它就会被 concat 当作一个数组来处理：此对象中的元素将被添加：

let arr = [1, 2];

let arrayLike = {
  0: "something",
  1: "else",
  [Symbol.isConcatSpreadable]: true,
  length: 2
};

alert( arr.concat(arrayLike) ); // 1,2,something,else
```
## 5.2 遍历：forEach
```
arr.forEach 方法允许为数组的每个元素都运行一个函数。

语法：

arr.forEach(function(item, index, array) {
  // ... do something with item
});
例如，下面这个程序显示了数组的每个元素：

// 对每个元素调用 alert
["Bilbo", "Gandalf", "Nazgul"].forEach(alert);
而这段代码更详细地介绍了它们在目标数组中的位置：

["Bilbo", "Gandalf", "Nazgul"].forEach((item, index, array) => {
  alert(`${item} is at index ${index} in ${array}`);
});
该函数的结果（如果它有返回）会被抛弃和忽略。
```
## 5.3 在数组中搜索
```
现在，让我们介绍在数组中进行搜索的方法。

indexOf/lastIndexOf 和 includes
arr.indexOf 和 arr.includes 方法语法相似，并且作用基本上也与字符串的方法相同，只不过这里是对数组元素而不是字符进行操作：

arr.indexOf(item, from) —— 从索引 from 开始搜索 item，如果找到则返回索引，否则返回 -1。
arr.includes(item, from) —— 从索引 from 开始搜索 item，如果找到则返回 true（译注：如果没找到，则返回 false）。
通常使用这些方法时只会传入一个参数：传入 item 开始搜索。默认情况下，搜索是从头开始的。

例如：

let arr = [1, 0, false];

alert( arr.indexOf(0) ); // 1
alert( arr.indexOf(false) ); // 2
alert( arr.indexOf(null) ); // -1

alert( arr.includes(1) ); // true
请注意，indexOf 和 includes 使用严格相等 === 进行比较。所以，如果我们搜索 false，它会准确找到 false 而不是数字 0。

如果我们想检查数组中是否包含元素 item，并且不需要知道其确切的索引，那么 arr.includes 是首选。

方法 arr.lastIndexOf 与 indexOf 相同，但从右向左查找。

let fruits = ['Apple', 'Orange', 'Apple'];

alert( fruits.indexOf('Apple') ); // 0（第一个 Apple）
alert( fruits.lastIndexOf('Apple') ); // 2（最后一个 Apple）
方法 includes 可以正确的处理 NaN
方法 includes 的一个次要但值得注意的特性是，它可以正确处理 NaN，这与 indexOf 不同：

const arr = [NaN];
alert( arr.indexOf(NaN) ); // -1（错，应该为 0）
alert( arr.includes(NaN) );// true（正确）
这是因为 includes 是在比较晚的时候才被添加到 JavaScript 中的，并且在内部使用了更新了的比较算法。

find 和 findIndex/findLastIndex
想象一下，我们有一个对象数组。我们如何找到具有特定条件的对象？

这时可以用 arr.find 方法。

语法如下：

let result = arr.find(function(item, index, array) {
  // 如果返回 true，则返回 item 并停止迭代
  // 对于假值（falsy）的情况，则返回 undefined
});
依次对数组中的每个元素调用该函数：

item 是元素。
index 是它的索引。
array 是数组本身。
如果它返回 true，则搜索停止，并返回 item。如果没有搜索到，则返回 undefined。

例如，我们有一个存储用户的数组，每个用户都有 id 和 name 字段。让我们找到 id == 1 的那个用户：

let users = [
  {id: 1, name: "John"},
  {id: 2, name: "Pete"},
  {id: 3, name: "Mary"}
];

let user = users.find(item => item.id == 1);

alert(user.name); // John
在现实生活中，对象数组是很常见的，所以 find 方法非常有用。

注意在这个例子中，我们传给了 find 一个单参数函数 item => item.id == 1。这很典型，并且 find 方法的其他参数很少使用。

arr.findIndex 方法（与 arr.find）具有相同的语法，但它返回找到的元素的索引，而不是元素本身。如果没找到，则返回 -1。

arr.findLastIndex 方法类似于 findIndex，但从右向左搜索，类似于 lastIndexOf。

这是一个例子：

let users = [
  {id: 1, name: "John"},
  {id: 2, name: "Pete"},
  {id: 3, name: "Mary"},
  {id: 4, name: "John"}
];

// 寻找第一个 John 的索引
alert(users.findIndex(user => user.name == 'John')); // 0

// 寻找最后一个 John 的索引
alert(users.findLastIndex(user => user.name == 'John')); // 3
filter
find 方法搜索的是使函数返回 true 的第一个（单个）元素。

如果需要匹配的有很多，我们可以使用 arr.filter(fn)。

语法与 find 大致相同，但是 filter 返回的是所有匹配元素组成的数组：

let results = arr.filter(function(item, index, array) {
  // 如果 true item 被 push 到 results，迭代继续
  // 如果什么都没找到，则返回空数组
});
例如：

let users = [
  {id: 1, name: "John"},
  {id: 2, name: "Pete"},
  {id: 3, name: "Mary"}
];

// 返回前两个用户的数组
let someUsers = users.filter(item => item.id < 3);

alert(someUsers.length); // 2
```
## 5.4 转换数组
```
让我们继续学习进行数组转换和重新排序的方法。

map
arr.map 方法是最有用和经常使用的方法之一。

它对数组的每个元素都调用函数，并返回结果数组。

语法：

let result = arr.map(function(item, index, array) {
  // 返回新值而不是当前元素
})
例如，在这里我们将每个元素转换为它的字符串长度：

let lengths = ["Bilbo", "Gandalf", "Nazgul"].map(item => item.length);
alert(lengths); // 5,7,6
sort(fn)
arr.sort 方法对数组进行 原位（in-place） 排序，更改元素的顺序。(译注：原位是指在此数组内，而非生成一个新数组。)

它还返回排序后的数组，但是返回值通常会被忽略，因为修改了 arr 本身。

语法：

let arr = [ 1, 2, 15 ];

// 该方法重新排列 arr 的内容
arr.sort();

alert( arr );  // 1, 15, 2
你有没有注意到结果有什么奇怪的地方？

顺序变成了 1, 15, 2。不对，但为什么呢？

这些元素默认情况下被按字符串进行排序。

从字面上看，所有元素都被转换为字符串，然后进行比较。对于字符串，按照词典顺序进行排序，实际上应该是 "2" > "15"。

要使用我们自己的排序顺序，我们需要提供一个函数作为 arr.sort() 的参数。

该函数应该比较两个任意值并返回：

function compare(a, b) {
  if (a > b) return 1; // 如果第一个值比第二个值大
  if (a == b) return 0; // 如果两个值相等
  if (a < b) return -1; // 如果第一个值比第二个值小
}
例如，按数字进行排序：

function compareNumeric(a, b) {
  if (a > b) return 1;
  if (a == b) return 0;
  if (a < b) return -1;
}

let arr = [ 1, 2, 15 ];

arr.sort(compareNumeric);

alert(arr);  // 1, 2, 15
现在结果符合预期了。

我们思考一下这儿发生了什么。arr 可以是由任何内容组成的数组，对吗？它可能包含数字、字符串、对象或其他任何内容。我们有一组 一些元素。要对其进行排序，我们需要一个 排序函数 来确认如何比较这些元素。默认是按字符串进行排序的。

arr.sort(fn) 方法实现了通用的排序算法。我们不需要关心它的内部工作原理（大多数情况下都是经过 快速排序 或 Timsort 算法优化的）。它将遍历数组，使用提供的函数比较其元素并对其重新排序，我们所需要的就是提供执行比较的函数 fn。

顺便说一句，如果我们想知道要比较哪些元素 —— 那么什么都不会阻止 alert 它们：

[1, -2, 15, 2, 0, 8].sort(function(a, b) {
  alert( a + " <> " + b );
  return a - b;
});
该算法可以在此过程中，将一个元素与多个其他元素进行比较，但是它会尝试进行尽可能少的比较。

比较函数可以返回任何数字
实际上，比较函数只需要返回一个正数表示“大于”，一个负数表示“小于”。

通过这个原理我们可以编写更短的函数：

let arr = [ 1, 2, 15 ];

arr.sort(function(a, b) { return a - b; });

alert(arr);  // 1, 2, 15
箭头函数最好
你还记得 箭头函数 吗？这里使用箭头函数会更加简洁：

arr.sort( (a, b) => a - b );
这与上面更长的版本完全相同。

使用 localeCompare for strings
你记得 字符串比较 算法吗？默认情况下，它通过字母的代码比较字母。

对于许多字母，最好使用 str.localeCompare 方法正确地对字母进行排序，例如 Ö。

例如，让我们用德语对几个国家/地区进行排序：

let countries = ['Österreich', 'Andorra', 'Vietnam'];

alert( countries.sort( (a, b) => a > b ? 1 : -1) ); // Andorra, Vietnam, Österreich（错的）

alert( countries.sort( (a, b) => a.localeCompare(b) ) ); // Andorra,Österreich,Vietnam（对的！）
reverse
arr.reverse 方法用于颠倒 arr 中元素的顺序。

例如：

let arr = [1, 2, 3, 4, 5];
arr.reverse();

alert( arr ); // 5,4,3,2,1
它也会返回颠倒后的数组 arr。

split 和 join
举一个现实生活场景的例子。我们正在编写一个消息应用程序，并且该人员输入以逗号分隔的接收者列表：John, Pete, Mary。但对我们来说，名字数组比单个字符串舒适得多。怎么做才能获得这样的数组呢？

str.split(delim) 方法可以做到。它通过给定的分隔符 delim 将字符串分割成一个数组。

在下面的例子中，我们用“逗号后跟着一个空格”作为分隔符：

let names = 'Bilbo, Gandalf, Nazgul';

let arr = names.split(', ');

for (let name of arr) {
  alert( `A message to ${name}.` ); // A message to Bilbo（和其他名字）
}
split 方法有一个可选的第二个数字参数 —— 对数组长度的限制。如果提供了，那么额外的元素会被忽略。但实际上它很少使用：

let arr = 'Bilbo, Gandalf, Nazgul, Saruman'.split(', ', 2);

alert(arr); // Bilbo, Gandalf
拆分为字母
调用带有空参数 s 的 split(s)，会将字符串拆分为字母数组：

let str = "test";

alert( str.split('') ); // t,e,s,t
arr.join(glue) 与 split 相反。它会在它们之间创建一串由 glue 粘合的 arr 项。

例如：

let arr = ['Bilbo', 'Gandalf', 'Nazgul'];

let str = arr.join(';'); // 使用分号 ; 将数组粘合成字符串

alert( str ); // Bilbo;Gandalf;Nazgul
reduce/reduceRight
当我们需要遍历一个数组时 —— 我们可以使用 forEach，for 或 for..of。

当我们需要遍历并返回每个元素的数据时 —— 我们可以使用 map。

arr.reduce 方法和 arr.reduceRight 方法和上面的种类差不多，但稍微复杂一点。它们用于根据数组计算单个值。

语法是：

let value = arr.reduce(function(accumulator, item, index, array) {
  // ...
}, [initial]);
该函数一个接一个地应用于所有数组元素，并将其结果“搬运（carry on）”到下一个调用。

参数：

accumulator —— 是上一个函数调用的结果，第一次等于 initial（如果提供了 initial 的话）。
item —— 当前的数组元素。
index —— 当前索引。
arr —— 数组本身。
应用函数时，上一个函数调用的结果将作为第一个参数传递给下一个函数。

因此，第一个参数本质上是累加器，用于存储所有先前执行的组合结果。最后，它成为 reduce 的结果。

听起来复杂吗？

掌握这个知识点的最简单的方法就是通过示例。

在这里，我们通过一行代码得到一个数组的总和：

let arr = [1, 2, 3, 4, 5];

let result = arr.reduce((sum, current) => sum + current, 0);

alert(result); // 15
传递给 reduce 的函数仅使用了 2 个参数，通常这就足够了。

让我们看看细节，到底发生了什么。

在第一次运行时，sum 的值为初始值 initial（reduce 的最后一个参数），等于 0，current 是第一个数组元素，等于 1。所以函数运行的结果是 1。
在第二次运行时，sum = 1，我们将第二个数组元素（2）与其相加并返回。
在第三次运行中，sum = 3，我们继续把下一个元素与其相加，以此类推……
计算流程：


或者以表格的形式表示，每一行代表的是对下一个数组元素的函数调用：

sum	current	result
第 1 次调用	0	1	1
第 2 次调用	1	2	3
第 3 次调用	3	3	6
第 4 次调用	6	4	10
第 5 次调用	10	5	15
在这里，我们可以清楚地看到上一个调用的结果如何成为下一个调用的第一个参数。

我们也可以省略初始值：

let arr = [1, 2, 3, 4, 5];

// 删除 reduce 的初始值（没有 0）
let result = arr.reduce((sum, current) => sum + current);

alert( result ); // 15
结果是一样的。这是因为如果没有初始值，那么 reduce 会将数组的第一个元素作为初始值，并从第二个元素开始迭代。

计算表与上面相同，只是去掉第一行。

但是这种使用需要非常小心。如果数组为空，那么在没有初始值的情况下调用 reduce 会导致错误。

例如：

let arr = [];

// Error: Reduce of empty array with no initial value
// 如果初始值存在，则 reduce 将为空 arr 返回它（即这个初始值）。
arr.reduce((sum, current) => sum + current);
所以建议始终指定初始值。

arr.reduceRight 和 arr.reduce 方法的功能一样，只是遍历为从右到左。
```
## 5.5 Array.isArray
```
数组是基于对象的，不构成单独的语言类型。

所以 typeof 不能帮助从数组中区分出普通对象：

alert(typeof {}); // object
alert(typeof []); // object（相同）
……但是数组经常被使用，因此有一种特殊的方法用于判断：Array.isArray(value)。如果 value 是一个数组，则返回 true；否则返回 false。

alert(Array.isArray({})); // false

alert(Array.isArray([])); // true
```
## 5.6 大多数方法都支持"thisArg"
```
几乎所有调用函数的数组方法 —— 比如 find，filter，map，除了 sort 是一个特例，都接受一个可选的附加参数 thisArg。

上面的部分中没有解释该参数，因为该参数很少使用。但是为了完整性，我们需要讲讲它。

以下是这些方法的完整语法：

arr.find(func, thisArg);
arr.filter(func, thisArg);
arr.map(func, thisArg);
// ...
// thisArg 是可选的最后一个参数
thisArg 参数的值在 func 中变为 this。

例如，在这里我们使用 army 对象方法作为过滤器，thisArg 用于传递上下文（passes the context）：

let army = {
  minAge: 18,
  maxAge: 27,
  canJoin(user) {
    return user.age >= this.minAge && user.age < this.maxAge;
  }
};

let users = [
  {age: 16},
  {age: 20},
  {age: 23},
  {age: 30}
];

// 找到 army.canJoin 返回 true 的 user
let soldiers = users.filter(army.canJoin, army);

alert(soldiers.length); // 2
alert(soldiers[0].age); // 20
alert(soldiers[1].age); // 23
如果在上面的示例中我们使用了 users.filter(army.canJoin)，那么 army.canJoin 将被作为独立函数调用，并且这时 this=undefined，从而会导致即时错误。

可以用 users.filter(user => army.canJoin(user)) 替换对 users.filter(army.canJoin, army) 的调用。前者的使用频率更高，因为对于大多数人来说，它更容易理解。
```

# 6. Iterable object（可迭代对象）
```javascript
可迭代（Iterable） 对象是数组的泛化。这个概念是说任何对象都可以被定制为可在 for..of 循环中使用的对象。

数组是可迭代的。但不仅仅是数组。很多其他内建对象也都是可迭代的。例如字符串也是可迭代的。

如果从严格意义上讲，对象不是数组，而是表示某物的集合（列表，集合），for..of 是一个能够遍历它的很好的语法，因此，让我们来看看如何使其发挥作用。
```
## 6.1 Symbol.iterator
```
通过自己创建一个对象，我们就可以轻松地掌握可迭代的概念。

例如，我们有一个对象，它并不是数组，但是看上去很适合使用 for..of 循环。

比如一个 range 对象，它代表了一个数字区间：

let range = {
  from: 1,
  to: 5
};

// 我们希望 for..of 这样运行：
// for(let num of range) ... num=1,2,3,4,5
为了让 range 对象可迭代（也就让 for..of 可以运行）我们需要为对象添加一个名为 Symbol.iterator 的方法（一个专门用于使对象可迭代的内建 symbol）。

当 for..of 循环启动时，它会调用这个方法（如果没找到，就会报错）。这个方法必须返回一个 迭代器（iterator） —— 一个有 next 方法的对象。
从此开始，for..of 仅适用于这个被返回的对象。
当 for..of 循环希望取得下一个数值，它就调用这个对象的 next() 方法。
next() 方法返回的结果的格式必须是 {done: Boolean, value: any}，当 done=true 时，表示循环结束，否则 value 是下一个值。
这是带有注释的 range 的完整实现：

let range = {
  from: 1,
  to: 5
};

// 1. for..of 调用首先会调用这个：
range[Symbol.iterator] = function() {

  // ……它返回迭代器对象（iterator object）：
  // 2. 接下来，for..of 仅与下面的迭代器对象一起工作，要求它提供下一个值
  return {
    current: this.from,
    last: this.to,

    // 3. next() 在 for..of 的每一轮循环迭代中被调用
    next() {
      // 4. 它将会返回 {done:.., value :...} 格式的对象
      if (this.current <= this.last) {
        return { done: false, value: this.current++ };
      } else {
        return { done: true };
      }
    }
  };
};

// 现在它可以运行了！
for (let num of range) {
  alert(num); // 1, 然后是 2, 3, 4, 5
}
请注意可迭代对象的核心功能：关注点分离。

range 自身没有 next() 方法。
相反，是通过调用 range[Symbol.iterator]() 创建了另一个对象，即所谓的“迭代器”对象，并且它的 next 会为迭代生成值。
因此，迭代器对象和与其进行迭代的对象是分开的。

从技术上说，我们可以将它们合并，并使用 range 自身作为迭代器来简化代码。

就像这样：

let range = {
  from: 1,
  to: 5,

  [Symbol.iterator]() {
    this.current = this.from;
    return this;
  },

  next() {
    if (this.current <= this.to) {
      return { done: false, value: this.current++ };
    } else {
      return { done: true };
    }
  }
};

for (let num of range) {
  alert(num); // 1, 然后是 2, 3, 4, 5
}
现在 range[Symbol.iterator]() 返回的是 range 对象自身：它包括了必需的 next() 方法，并通过 this.current 记忆了当前的迭代进程。这样更短，对吗？是的。有时这样也可以。

但缺点是，现在不可能同时在对象上运行两个 for..of 循环了：它们将共享迭代状态，因为只有一个迭代器，即对象本身。但是两个并行的 for..of 是很罕见的，即使在异步情况下。

无穷迭代器（iterator）
无穷迭代器也是可能的。例如，将 range 设置为 range.to = Infinity，这时 range 则成为了无穷迭代器。或者我们可以创建一个可迭代对象，它生成一个无穷伪随机数序列。也是可能的。

next 没有什么限制，它可以返回越来越多的值，这是正常的。

当然，迭代这种对象的 for..of 循环将不会停止。但是我们可以通过使用 break 来停止它。
```
## 6.2 字符串是可迭代的
```
数组和字符串是使用最广泛的内建可迭代对象。

对于一个字符串，for..of 遍历它的每个字符：

for (let char of "test") {
  // 触发 4 次，每个字符一次
  alert( char ); // t, then e, then s, then t
}
对于代理对（surrogate pairs），它也能正常工作！（译注：这里的代理对也就指的是 UTF-16 的扩展字符）

let str = '𝒳😂';
for (let char of str) {
    alert( char ); // 𝒳，然后是 😂
}
```
## 6.3 显式调用可迭代器
```
为了更深层地了解底层知识，让我们来看看如何显式地使用迭代器。

我们将会采用与 for..of 完全相同的方式遍历字符串，但使用的是直接调用。这段代码创建了一个字符串迭代器，并“手动”从中获取值。

let str = "Hello";

// 和 for..of 做相同的事
// for (let char of str) alert(char);

let iterator = str[Symbol.iterator]();

while (true) {
  let result = iterator.next();
  if (result.done) break;
  alert(result.value); // 一个接一个地输出字符
}
很少需要我们这样做，但是比 for..of 给了我们更多的控制权。例如，我们可以拆分迭代过程：迭代一部分，然后停止，做一些其他处理，然后再恢复迭代。
```
## 6.4 可迭代（iterable）和类数组（array-like）
```
这两个官方术语看起来差不多，但其实大不相同。请确保你能够充分理解它们的含义，以免造成混淆。

Iterable 如上所述，是实现了 Symbol.iterator 方法的对象。
Array-like 是有索引和 length 属性的对象，所以它们看起来很像数组。
当我们将 JavaScript 用于编写在浏览器或任何其他环境中的实际任务时，我们可能会遇到可迭代对象或类数组对象，或两者兼有。

例如，字符串即是可迭代的（for..of 对它们有效），又是类数组的（它们有数值索引和 length 属性）。

但是一个可迭代对象也许不是类数组对象。反之亦然，类数组对象可能不可迭代。

例如，上面例子中的 range 是可迭代的，但并非类数组对象，因为它没有索引属性，也没有 length 属性。

下面这个对象则是类数组的，但是不可迭代：

let arrayLike = { // 有索引和 length 属性 => 类数组对象
  0: "Hello",
  1: "World",
  length: 2
};

// Error (no Symbol.iterator)
for (let item of arrayLike) {}
可迭代对象和类数组对象通常都 不是数组，它们没有 push 和 pop 等方法。如果我们有一个这样的对象，并想像数组那样操作它，那就非常不方便。例如，我们想使用数组方法操作 range，应该如何实现呢？
```
## 6.5 Array.from
```
有一个全局方法 Array.from 可以接受一个可迭代或类数组的值，并从中获取一个“真正的”数组。然后我们就可以对其调用数组方法了。

例如：

let arrayLike = {
  0: "Hello",
  1: "World",
  length: 2
};

let arr = Array.from(arrayLike); // (*)
alert(arr.pop()); // World（pop 方法有效）
在 (*) 行的 Array.from 方法接受对象，检查它是一个可迭代对象或类数组对象，然后创建一个新数组，并将该对象的所有元素复制到这个新数组。

如果是可迭代对象，也是同样：

// 假设 range 来自上文的例子中
let arr = Array.from(range);
alert(arr); // 1,2,3,4,5 （数组的 toString 转化方法生效）
Array.from 的完整语法允许我们提供一个可选的“映射（mapping）”函数：

Array.from(obj[, mapFn, thisArg])
可选的第二个参数 mapFn 可以是一个函数，该函数会在对象中的元素被添加到数组前，被应用于每个元素，此外 thisArg 允许我们为该函数设置 this。

例如：

// 假设 range 来自上文例子中

// 求每个数的平方
let arr = Array.from(range, num => num * num);

alert(arr); // 1,4,9,16,25
现在我们用 Array.from 将一个字符串转换为单个字符的数组：

let str = '𝒳😂';

// 将 str 拆分为字符数组
let chars = Array.from(str);

alert(chars[0]); // 𝒳
alert(chars[1]); // 😂
alert(chars.length); // 2
与 str.split 方法不同，它依赖于字符串的可迭代特性。因此，就像 for..of 一样，可以正确地处理代理对（surrogate pair）。（译注：代理对也就是 UTF-16 扩展字符。）

技术上来讲，它和下面这段代码做的是相同的事：

let str = '𝒳😂';

let chars = []; // Array.from 内部执行相同的循环
for (let char of str) {
  chars.push(char);
}

alert(chars);
……但 Array.from 精简很多。

我们甚至可以基于 Array.from 创建代理感知（surrogate-aware）的slice 方法（译注：也就是能够处理 UTF-16 扩展字符的 slice 方法）：

function slice(str, start, end) {
  return Array.from(str).slice(start, end).join('');
}

let str = '𝒳😂𩷶';

alert( slice(str, 1, 3) ); // 😂𩷶

// 原生方法不支持识别代理对（译注：UTF-16 扩展字符）
alert( str.slice(1, 3) ); // 乱码（两个不同 UTF-16 扩展字符碎片拼接的结果）
```

# 7. Map and Set（映射和集合）
```javascript
学到现在，我们已经了解了以下复杂的数据结构：

对象，存储带有键的数据的集合。
数组，存储有序集合。
但这还不足以应对现实情况。这就是为什么存在 Map 和 Set。
```
## 7.1 Map
```
Map 是一个带键的数据项的集合，就像一个 Object 一样。 但是它们最大的差别是 Map 允许任何类型的键（key）。

它的方法和属性如下：

new Map() —— 创建 map。
map.set(key, value) —— 根据键存储值。
map.get(key) —— 根据键来返回值，如果 map 中不存在对应的 key，则返回 undefined。
map.has(key) —— 如果 key 存在则返回 true，否则返回 false。
map.delete(key) —— 删除指定键的值。
map.clear() —— 清空 map。
map.size —— 返回当前元素个数。
举个例子：

let map = new Map();

map.set('1', 'str1');   // 字符串键
map.set(1, 'num1');     // 数字键
map.set(true, 'bool1'); // 布尔值键

// 还记得普通的 Object 吗? 它会将键转化为字符串
// Map 则会保留键的类型，所以下面这两个结果不同：
alert( map.get(1)   ); // 'num1'
alert( map.get('1') ); // 'str1'

alert( map.size ); // 3
如我们所见，与对象不同，键不会被转换成字符串。键可以是任何类型。

map[key] 不是使用 Map 的正确方式
虽然 map[key] 也有效，例如我们可以设置 map[key] = 2，这样会将 map 视为 JavaScript 的 plain object，因此它暗含了所有相应的限制（仅支持 string/symbol 键等）。

所以我们应该使用 map 方法：set 和 get 等。

Map 还可以使用对象作为键。

例如：

let john = { name: "John" };

// 存储每个用户的来访次数
let visitsCountMap = new Map();

// john 是 Map 中的键
visitsCountMap.set(john, 123);

alert( visitsCountMap.get(john) ); // 123
使用对象作为键是 Map 最值得注意和重要的功能之一。在 Object 中，我们则无法使用对象作为键。在 Object 中使用字符串作为键是可以的，但我们无法使用另一个 Object 作为 Object 中的键。

我们来尝试一下：

let john = { name: "John" };
let ben = { name: "Ben" };

let visitsCountObj = {}; // 尝试使用对象

visitsCountObj[ben] = 234; // 尝试将对象 ben 用作键
visitsCountObj[john] = 123; // 尝试将对象 john 用作键，但我们会发现使用对象 ben 作为键存下的值会被替换掉

// 变成这样了！
alert( visitsCountObj["[object Object]"] ); // 123
因为 visitsCountObj 是一个对象，它会将所有 Object 键例如上面的 john 和 ben 转换为字符串 "[object Object]"。这显然不是我们想要的结果。

Map 是怎么比较键的？
Map 使用 SameValueZero 算法来比较键是否相等。它和严格等于 === 差不多，但区别是 NaN 被看成是等于 NaN。所以 NaN 也可以被用作键。

这个算法不能被改变或者自定义。

链式调用
每一次 map.set 调用都会返回 map 本身，所以我们可以进行“链式”调用：

map.set('1', 'str1')
  .set(1, 'num1')
  .set(true, 'bool1');
```
## 7.2 Map迭代
```
如果要在 map 里使用循环，可以使用以下三个方法：

map.keys() —— 遍历并返回一个包含所有键的可迭代对象，
map.values() —— 遍历并返回一个包含所有值的可迭代对象，
map.entries() —— 遍历并返回一个包含所有实体 [key, value] 的可迭代对象，for..of 在默认情况下使用的就是这个。
例如：

let recipeMap = new Map([
  ['cucumber', 500],
  ['tomatoes', 350],
  ['onion',    50]
]);

// 遍历所有的键（vegetables）
for (let vegetable of recipeMap.keys()) {
  alert(vegetable); // cucumber, tomatoes, onion
}

// 遍历所有的值（amounts）
for (let amount of recipeMap.values()) {
  alert(amount); // 500, 350, 50
}

// 遍历所有的实体 [key, value]
for (let entry of recipeMap) { // 与 recipeMap.entries() 相同
  alert(entry); // cucumber,500 (and so on)
}
使用插入顺序
迭代的顺序与插入值的顺序相同。与普通的 Object 不同，Map 保留了此顺序。

除此之外，Map 有内建的 forEach 方法，与 Array 类似：

// 对每个键值对 (key, value) 运行 forEach 函数
recipeMap.forEach( (value, key, map) => {
  alert(`${key}: ${value}`); // cucumber: 500 etc
});
```
## 7.3 Object.entries：从对象创建Map
```
当创建一个 Map 后，我们可以传入一个带有键值对的数组（或其它可迭代对象）来进行初始化，如下所示：

// 键值对 [key, value] 数组
let map = new Map([
  ['1',  'str1'],
  [1,    'num1'],
  [true, 'bool1']
]);

alert( map.get('1') ); // str1
如果我们想从一个已有的普通对象（plain object）来创建一个 Map，那么我们可以使用内建方法 Object.entries(obj)，该方法返回对象的键/值对数组，该数组格式完全按照 Map 所需的格式。

所以可以像下面这样从一个对象创建一个 Map：

let obj = {
  name: "John",
  age: 30
};

let map = new Map(Object.entries(obj));

alert( map.get('name') ); // John
这里，Object.entries 返回键/值对数组：[ ["name","John"], ["age", 30] ]。这就是 Map 所需要的格式。
```
## 7.4 Object.fromEntries：从Map创建对象
```
我们刚刚已经学习了如何使用 Object.entries(obj) 从普通对象（plain object）创建 Map。

Object.fromEntries 方法的作用是相反的：给定一个具有 [key, value] 键值对的数组，它会根据给定数组创建一个对象：

let prices = Object.fromEntries([
  ['banana', 1],
  ['orange', 2],
  ['meat', 4]
]);

// 现在 prices = { banana: 1, orange: 2, meat: 4 }

alert(prices.orange); // 2
我们可以使用 Object.fromEntries 从 Map 得到一个普通对象（plain object）。

例如，我们在 Map 中存储了一些数据，但是我们需要把这些数据传给需要普通对象（plain object）的第三方代码。

我们来开始：

let map = new Map();
map.set('banana', 1);
map.set('orange', 2);
map.set('meat', 4);

let obj = Object.fromEntries(map.entries()); // 创建一个普通对象（plain object）(*)

// 完成了！
// obj = { banana: 1, orange: 2, meat: 4 }

alert(obj.orange); // 2
调用 map.entries() 将返回一个可迭代的键/值对，这刚好是 Object.fromEntries 所需要的格式。

我们可以把带 (*) 这一行写得更短：

let obj = Object.fromEntries(map); // 省掉 .entries()
上面的代码作用也是一样的，因为 Object.fromEntries 期望得到一个可迭代对象作为参数，而不一定是数组。并且 map 的标准迭代会返回跟 map.entries() 一样的键/值对。因此，我们可以获得一个普通对象（plain object），其键/值对与 map 相同。
```
## 7.5 Set
```
Set 是一个特殊的类型集合 —— “值的集合”（没有键），它的每一个值只能出现一次。

它的主要方法如下：

new Set(iterable) —— 创建一个 set，如果提供了一个 iterable 对象（通常是数组），将会从数组里面复制值到 set 中。
set.add(value) —— 添加一个值，返回 set 本身
set.delete(value) —— 删除值，如果 value 在这个方法调用的时候存在则返回 true ，否则返回 false。
set.has(value) —— 如果 value 在 set 中，返回 true，否则返回 false。
set.clear() —— 清空 set。
set.size —— 返回元素个数。
它的主要特点是，重复使用同一个值调用 set.add(value) 并不会发生什么改变。这就是 Set 里面的每一个值只出现一次的原因。

例如，我们有客人来访，我们想记住他们每一个人。但是已经来访过的客人再次来访，不应造成重复记录。每个访客必须只被“计数”一次。

Set 可以帮助我们解决这个问题：

let set = new Set();

let john = { name: "John" };
let pete = { name: "Pete" };
let mary = { name: "Mary" };

// visits，一些访客来访好几次
set.add(john);
set.add(pete);
set.add(mary);
set.add(john);
set.add(mary);

// set 只保留不重复的值
alert( set.size ); // 3

for (let user of set) {
  alert(user.name); // John（然后 Pete 和 Mary）
}
Set 的替代方法可以是一个用户数组，用 arr.find 在每次插入值时检查是否重复。但是这样性能会很差，因为这个方法会遍历整个数组来检查每个元素。Set 内部对唯一性检查进行了更好的优化。
```
## 7.6 Set迭代（iteration）
```
我们可以使用 for..of 或 forEach 来遍历 Set：

let set = new Set(["oranges", "apples", "bananas"]);

for (let value of set) alert(value);

// 与 forEach 相同：
set.forEach((value, valueAgain, set) => {
  alert(value);
});
注意一件有趣的事儿。forEach 的回调函数有三个参数：一个 value，然后是 同一个值 valueAgain，最后是目标对象。没错，同一个值在参数里出现了两次。

forEach 的回调函数有三个参数，是为了与 Map 兼容。当然，这看起来确实有些奇怪。但是这对在特定情况下轻松地用 Set 代替 Map 很有帮助，反之亦然。

Map 中用于迭代的方法在 Set 中也同样支持：

set.keys() —— 遍历并返回一个包含所有值的可迭代对象，
set.values() —— 与 set.keys() 作用相同，这是为了兼容 Map，
set.entries() —— 遍历并返回一个包含所有的实体 [value, value] 的可迭代对象，它的存在也是为了兼容 Map。
```

# 8. WeakMap and WeakSet（弱映射和弱集合）
```javascript
我们从前面的 垃圾回收 章节中知道，JavaScript 引擎在值“可达”和可能被使用时会将其保持在内存中。

例如：

let john = { name: "John" };

// 该对象能被访问，john 是它的引用

// 覆盖引用
john = null;

// 该对象将会被从内存中清除
通常，当对象、数组之类的数据结构在内存中时，它们的子元素，如对象的属性、数组的元素都被认为是可达的。

例如，如果把一个对象放入到数组中，那么只要这个数组存在，那么这个对象也就存在，即使没有其他对该对象的引用。

就像这样:

let john = { name: "John" };

let array = [ john ];

john = null; // 覆盖引用

// 前面由 john 所引用的那个对象被存储在了 array 中
// 所以它不会被垃圾回收机制回收
// 我们可以通过 array[0] 获取到它
类似的，如果我们使用对象作为常规 Map 的键，那么当 Map 存在时，该对象也将存在。它会占用内存，并且不会被（垃圾回收机制）回收。

例如：

let john = { name: "John" };

let map = new Map();
map.set(john, "...");

john = null; // 覆盖引用

// john 被存储在了 map 中，
// 我们可以使用 map.keys() 来获取它
WeakMap 在这方面有着根本上的不同。它不会阻止垃圾回收机制对作为键的对象（key object）的回收。

让我们通过例子来看看这指的到底是什么。
```
## 8.1 WeakMap
```
WeakMap 和 Map 的第一个不同点就是，WeakMap 的键必须是对象，不能是原始值：

let weakMap = new WeakMap();

let obj = {};

weakMap.set(obj, "ok"); // 正常工作（以对象作为键）

// 不能使用字符串作为键
weakMap.set("test", "Whoops"); // Error，因为 "test" 不是一个对象
现在，如果我们在 weakMap 中使用一个对象作为键，并且没有其他对这个对象的引用 —— 该对象将会被从内存（和map）中自动清除。

let john = { name: "John" };

let weakMap = new WeakMap();
weakMap.set(john, "...");

john = null; // 覆盖引用

// john 被从内存中删除了！
与上面常规的 Map 的例子相比，现在如果 john 仅仅是作为 WeakMap 的键而存在 —— 它将会被从 map（和内存）中自动删除。

WeakMap 不支持迭代以及 keys()，values() 和 entries() 方法。所以没有办法获取 WeakMap 的所有键或值。

WeakMap 只有以下的方法：

weakMap.get(key)
weakMap.set(key, value)
weakMap.delete(key)
weakMap.has(key)
为什么会有这种限制呢？这是技术的原因。如果一个对象丢失了其它所有引用（就像上面示例中的 john），那么它就会被垃圾回收机制自动回收。但是在从技术的角度并不能准确知道 何时会被回收。

这些都是由 JavaScript 引擎决定的。JavaScript 引擎可能会选择立即执行内存清理，如果现在正在发生很多删除操作，那么 JavaScript 引擎可能就会选择等一等，稍后再进行内存清理。因此，从技术上讲，WeakMap 的当前元素的数量是未知的。JavaScript 引擎可能清理了其中的垃圾，可能没清理，也可能清理了一部分。因此，暂不支持访问 WeakMap 的所有键/值的方法。

那么，在哪里我们会需要这样的数据结构呢？
```
## 8.2 使用案例：额外的数据
```
WeakMap 的主要应用场景是 额外数据的存储。

假如我们正在处理一个“属于”另一个代码的一个对象，也可能是第三方库，并想存储一些与之相关的数据，那么这些数据就应该与这个对象共存亡 —— 这时候 WeakMap 正是我们所需要的利器。

我们将这些数据放到 WeakMap 中，并使用该对象作为这些数据的键，那么当该对象被垃圾回收机制回收后，这些数据也会被自动清除。

weakMap.set(john, "secret documents");
// 如果 john 消失，secret documents 将会被自动清除
让我们来看一个例子。

例如，我们有用于处理用户访问计数的代码。收集到的信息被存储在 map 中：一个用户对象作为键，其访问次数为值。当一个用户离开时（该用户对象将被垃圾回收机制回收），这时我们就不再需要他的访问次数了。

下面是一个使用 Map 的计数函数的例子：

// 📁 visitsCount.js
let visitsCountMap = new Map(); // map: user => visits count

// 递增用户来访次数
function countUser(user) {
  let count = visitsCountMap.get(user) || 0;
  visitsCountMap.set(user, count + 1);
}
下面是其他部分的代码，可能是使用它的其它代码：

// 📁 main.js
let john = { name: "John" };

countUser(john); // count his visits

// 不久之后，john 离开了
john = null;
现在，john 这个对象应该被垃圾回收，但它仍在内存中，因为它是 visitsCountMap 中的一个键。

当我们移除用户时，我们需要清理 visitsCountMap，否则它将在内存中无限增大。在复杂的架构中，这种清理会成为一项繁重的任务。

我们可以通过使用 WeakMap 来避免这样的问题：

// 📁 visitsCount.js
let visitsCountMap = new WeakMap(); // weakmap: user => visits count

// 递增用户来访次数
function countUser(user) {
  let count = visitsCountMap.get(user) || 0;
  visitsCountMap.set(user, count + 1);
}
现在我们不需要去清理 visitsCountMap 了。当 john 对象变成不可达时，即便它是 WeakMap 里的一个键，它也会连同它作为 WeakMap 里的键所对应的信息一同被从内存中删除。
```
## 8.3 使用案例：缓存
```
另外一个常见的例子是缓存。我们可以存储（“缓存”）函数的结果，以便将来对同一个对象的调用可以重用这个结果。

为了实现这一点，我们可以使用 Map（非最佳方案）：

// 📁 cache.js
let cache = new Map();

// 计算并记住结果
function process(obj) {
  if (!cache.has(obj)) {
    let result = /* calculations of the result for */ obj;

    cache.set(obj, result);
  }

  return cache.get(obj);
}

// 现在我们在其它文件中使用 process()

// 📁 main.js
let obj = {/* 假设我们有个对象 */};

let result1 = process(obj); // 计算完成

// ……稍后，来自代码的另外一个地方……
let result2 = process(obj); // 取自缓存的被记忆的结果

// ……稍后，我们不再需要这个对象时：
obj = null;

alert(cache.size); // 1（啊！该对象依然在 cache 中，并占据着内存！）
对于多次调用同一个对象，它只需在第一次调用时计算出结果，之后的调用可以直接从 cache 中获取。这样做的缺点是，当我们不再需要这个对象的时候需要清理 cache。

如果我们用 WeakMap 替代 Map，便不会存在这个问题。当对象被垃圾回收时，对应缓存的结果也会被自动从内存中清除。

// 📁 cache.js
let cache = new WeakMap();

// 计算并记结果
function process(obj) {
  if (!cache.has(obj)) {
    let result = /* calculate the result for */ obj;

    cache.set(obj, result);
  }

  return cache.get(obj);
}

// 📁 main.js
let obj = {/* some object */};

let result1 = process(obj);
let result2 = process(obj);

// ……稍后，我们不再需要这个对象时：
obj = null;

// 无法获取 cache.size，因为它是一个 WeakMap，
// 要么是 0，或即将变为 0
// 当 obj 被垃圾回收，缓存的数据也会被清除
```
## 8.4 WeakSet
```
WeakSet 的表现类似：

与 Set 类似，但是我们只能向 WeakSet 添加对象（而不能是原始值）。
对象只有在其它某个（些）地方能被访问的时候，才能留在 WeakSet 中。
跟 Set 一样，WeakSet 支持 add，has 和 delete 方法，但不支持 size 和 keys()，并且不可迭代。
变“弱（weak）”的同时，它也可以作为额外的存储空间。但并非针对任意数据，而是针对“是/否”的事实。WeakSet 的元素可能代表着有关该对象的某些信息。

例如，我们可以将用户添加到 WeakSet 中，以追踪访问过我们网站的用户：

let visitedSet = new WeakSet();

let john = { name: "John" };
let pete = { name: "Pete" };
let mary = { name: "Mary" };

visitedSet.add(john); // John 访问了我们
visitedSet.add(pete); // 然后是 Pete
visitedSet.add(john); // John 再次访问

// visitedSet 现在有两个用户了

// 检查 John 是否来访过？
alert(visitedSet.has(john)); // true

// 检查 Mary 是否来访过？
alert(visitedSet.has(mary)); // false

john = null;

// visitedSet 将被自动清理(即自动清除其中已失效的值 john)
WeakMap 和 WeakSet 最明显的局限性就是不能迭代，并且无法获取所有当前内容。那样可能会造成不便，但是并不会阻止 WeakMap/WeakSet 完成其主要工作 —— 为在其它地方存储/管理的对象数据提供“额外”存储。
```

# 9. Object.keys，values，entries
```javascript
对各个数据结构的学习至此告一段落，下面让我们讨论一下如何迭代它们。

在前面的章节中，我们认识了 map.keys()，map.values() 和 map.entries() 方法。

这些方法是通用的，有一个共同的约定来将它们用于各种数据结构。如果我们创建一个我们自己的数据结构，我们也应该实现这些方法。

它们支持：

Map
Set
Array
普通对象也支持类似的方法，但是语法上有一些不同。
```
## 9.1 Object.keys, values, entries
```
对于普通对象，下列这些方法是可用的：

Object.keys(obj) —— 返回一个包含该对象所有的键的数组。
Object.values(obj) —— 返回一个包含该对象所有的值的数组。
Object.entries(obj) —— 返回一个包含该对象所有 [key, value] 键值对的数组。
……但是请注意区别（比如说跟 map 的区别）：

Map	Object
调用语法	map.keys()	Object.keys(obj)，而不是 obj.keys()
返回值	可迭代对象	“真正的”数组
第一个区别是，对于对象我们使用的调用语法是 Object.keys(obj)，而不是 obj.keys()。

为什么会这样？主要原因是灵活性。请记住，在 JavaScript 中，对象是所有复杂结构的基础。因此，我们可能有一个自己创建的对象，比如 data，并实现了它自己的 data.values() 方法。同时，我们依然可以对它调用 Object.values(data) 方法。

第二个区别是 Object.* 方法返回的是“真正的”数组对象，而不只是一个可迭代对象。这主要是历史原因。

举个例子：

let user = {
  name: "John",
  age: 30
};
Object.keys(user) = ["name", "age"]
Object.values(user) = ["John", 30]
Object.entries(user) = [ ["name","John"], ["age",30] ]
这里有一个使用 Object.values 来遍历属性值的例子：

let user = {
  name: "John",
  age: 30
};

// 遍历所有的值
for (let value of Object.values(user)) {
  alert(value); // John, then 30
}
Object.keys/values/entries 会忽略 symbol 属性
就像 for..in 循环一样，这些方法会忽略使用 Symbol(...) 作为键的属性。

通常这很方便。但是，如果我们也想要 Symbol 类型的键，那么这儿有一个单独的方法 Object.getOwnPropertySymbols，它会返回一个只包含 Symbol 类型的键的数组。另外，还有一种方法 Reflect.ownKeys(obj)，它会返回 所有 键。
```
## 9.2 转换对象
```
对象缺少数组存在的许多方法，例如 map 和 filter 等。

如果我们想应用它们，那么我们可以使用 Object.entries，然后使用 Object.fromEntries：

使用 Object.entries(obj) 从 obj 获取由键/值对组成的数组。
对该数组使用数组方法，例如 map，对这些键/值对进行转换。
对结果数组使用 Object.fromEntries(array) 方法，将结果转回成对象。
例如，我们有一个带有价格的对象，并想将它们加倍：

let prices = {
  banana: 1,
  orange: 2,
  meat: 4,
};

let doublePrices = Object.fromEntries(
  // 将价格转换为数组，将每个键/值对映射为另一对
  // 然后通过 fromEntries 再将结果转换为对象
  Object.entries(prices).map(entry => [entry[0], entry[1] * 2])
);

alert(doublePrices.meat); // 8
乍一看，可能感觉有点困难，但是使用一两次之后就很容易理解了。我们可以通过这种方式建立强大的转换链。
```

# 10. 解构赋值
```javascript
JavaScript 中最常用的两种数据结构是 Object 和 Array。

对象是一种根据键存储数据的实体。
数组是一种直接存储数据的有序列表。
但是，当我们把它们传递给函数时，函数可能不需要整个对象/数组，而只需要其中一部分。

解构赋值 是一种特殊的语法，它使我们可以将数组或对象“拆包”至一系列变量中。有时这样做更方便。

解构操作对那些具有很多参数和默认值等的函数也很奏效。下面有一些例子。
```
## 10.1 数组解构
```
这是一个将数组解构到变量中的例子：

// 我们有一个存放了名字和姓氏的数组
let arr = ["John", "Smith"]

// 解构赋值
// 设置 firstName = arr[0]
// 以及 surname = arr[1]
let [firstName, surname] = arr;

alert(firstName); // John
alert(surname);  // Smith
我们可以使用这些变量而非原来的数组项了。

当与 split 函数（或其他返回值为数组的函数）结合使用时，看起来更优雅：

let [firstName, surname] = "John Smith".split(' ');
alert(firstName); // John
alert(surname);  // Smith
正如我们所看到的，语法很简单。但是有几个需要注意的细节。让我们通过更多的例子来加深理解。

“解构”并不意味着“破坏”
这种语法被叫做“解构赋值”，是因为它“拆开”了数组或对象，将其中的各元素复制给一些变量。原来的数组或对象自身没有被修改。

换句话说，解构赋值只是写起来简洁一点。以下两种写法是等价的：

// let [firstName, surname] = arr;
let firstName = arr[0];
let surname = arr[1];
忽略使用逗号的元素
可以通过添加额外的逗号来丢弃数组中不想要的元素：

// 不需要第二个元素
let [firstName, , title] = ["Julius", "Caesar", "Consul", "of the Roman Republic"];

alert( title ); // Consul
在上面的代码中，数组的第二个元素被跳过了，第三个元素被赋值给了 title 变量。数组中剩下的元素也都被跳过了（因为在这没有对应给它们的变量）。

等号右侧可以是任何可迭代对象
……实际上，我们可以将其与任何可迭代对象一起使用，而不仅限于数组：

let [a, b, c] = "abc"; // ["a", "b", "c"]
let [one, two, three] = new Set([1, 2, 3]);
这种情况下解构赋值是通过迭代右侧的值来完成工作的。这是一种用于对在 = 右侧的值上调用 for..of 并进行赋值的操作的语法糖。

赋值给等号左侧的任何内容
我们可以在等号左侧使用任何“可以被赋值的”东西。

例如，一个对象的属性：

let user = {};
[user.name, user.surname] = "John Smith".split(' ');

alert(user.name); // John
alert(user.surname); // Smith
与 .entries() 方法进行循环操作
在前面的章节中我们已经见过了 Object.entries(obj) 方法。

我们可以将 .entries() 方法与解构语法一同使用，来遍历一个对象的“键—值”对：

let user = {
  name: "John",
  age: 30
};

// 使用循环遍历键—值对
for (let [key, value] of Object.entries(user)) {
  alert(`${key}:${value}`); // name:John, then age:30
}
用于 Map 的类似代码更简单，因为 Map 是可迭代的：

let user = new Map();
user.set("name", "John");
user.set("age", "30");

// Map 是以 [key, value] 对的形式进行迭代的，非常便于解构
for (let [key, value] of user) {
  alert(`${key}:${value}`); // name:John, then age:30
}
交换变量值的技巧
使用解构赋值来交换两个变量的值是一个著名的技巧：

let guest = "Jane";
let admin = "Pete";

// 让我们来交换变量的值：使得 guest = Pete，admin = Jane
[guest, admin] = [admin, guest];

alert(`${guest} ${admin}`); // Pete Jane（成功交换！）
这里我们创建了一个由两个变量组成的临时数组，并且立即以颠倒的顺序对其进行了解构赋值。

我们也可以用这种方式交换两个以上的变量。

其余的 ‘…’
通常，如果数组比左边的列表长，那么“其余”的数组项会被省略。

例如，这里只取了两项，其余的就被忽略了：

let [name1, name2] = ["Julius", "Caesar", "Consul", "of the Roman Republic"];

alert(name1); // Julius
alert(name2); // Caesar
// 其余数组项未被分配到任何地方
如果我们还想收集其余的数组项 —— 我们可以使用三个点 "..." 来再加一个参数以获取其余数组项：

let [name1, name2, ...rest] = ["Julius", "Caesar", "Consul", "of the Roman Republic"];

// rest 是包含从第三项开始的其余数组项的数组
alert(rest[0]); // Consul
alert(rest[1]); // of the Roman Republic
alert(rest.length); // 2
rest 的值就是数组中剩下的元素组成的数组。

不一定要使用变量名 rest，我们也可以使用任何其他的变量名。只要确保它前面有三个点，并且在解构赋值的最后一个参数位置上就行了：

let [name1, name2, ...titles] = ["Julius", "Caesar", "Consul", "of the Roman Republic"];
// 现在 titles = ["Consul", "of the Roman Republic"]
默认值
如果数组比左边的变量列表短，这里不会出现报错。缺少对应值的变量都会被赋 undefined：

let [firstName, surname] = [];

alert(firstName); // undefined
alert(surname); // undefined
如果我们想要一个“默认”值给未赋值的变量，我们可以使用 = 来提供：

// 默认值
let [name = "Guest", surname = "Anonymous"] = ["Julius"];

alert(name);    // Julius（来自数组的值）
alert(surname); // Anonymous（默认值被使用了）
默认值可以是更加复杂的表达式，甚至可以是函数调用。不过，这些表达式或函数只会在这个变量未被赋值的时候才会被计算。

举个例子，我们使用了 prompt 函数来提供两个默认值：

// 只会提示输入姓氏
let [name = prompt('name?'), surname = prompt('surname?')] = ["Julius"];

alert(name);    // Julius（来自数组）
alert(surname); // 你输入的值
请注意：prompt 将仅针对缺失值（surname）运行。
```
## 10.2 对象解构
```
解构赋值同样适用于对象。

基本语法是：

let {var1, var2} = {var1:…, var2:…}
在等号右侧是一个已经存在的对象，我们想把它拆分到变量中。等号左侧包含了对象相应属性的一个类对象“模式（pattern）”。在最简单的情况下，等号左侧的就是 {...} 中的变量名列表。

如下所示：

let options = {
  title: "Menu",
  width: 100,
  height: 200
};

let {title, width, height} = options;

alert(title);  // Menu
alert(width);  // 100
alert(height); // 200
属性 options.title、options.width 和 options.height 值被赋给了对应的变量。

变量的顺序并不重要，下面这个代码也是等价的：

// 改变 let {...} 中元素的顺序
let {height, width, title} = { title: "Menu", height: 200, width: 100 }
等号左侧的模式（pattern）可以更加复杂，指定属性和变量之间的映射关系。

如果我们想把一个属性赋值给另一个名字的变量，比如把 options.width 属性赋值给名为 w 的变量，那么我们可以使用冒号来设置变量名称：

let options = {
  title: "Menu",
  width: 100,
  height: 200
};

// { sourceProperty: targetVariable }
let {width: w, height: h, title} = options;

// width -> w
// height -> h
// title -> title

alert(title);  // Menu
alert(w);      // 100
alert(h);      // 200
冒号的语法是“从对象中什么属性的值：赋值给哪个变量”。上面的例子中，属性 width 被赋值给了 w，属性 height 被赋值给了 h，属性 title 被赋值给了同名变量。

对于可能缺失的属性，我们可以使用 "=" 设置默认值，如下所示：

let options = {
  title: "Menu"
};

let {width = 100, height = 200, title} = options;

alert(title);  // Menu
alert(width);  // 100
alert(height); // 200
就像数组或函数参数一样，默认值可以是任意表达式甚至可以是函数调用。它们只会在未提供对应的值时才会被计算/调用。

在下面的代码中，prompt 提示输入 width 值，但不会提示输入 title 值：

let options = {
  title: "Menu"
};

let {width = prompt("width?"), title = prompt("title?")} = options;

alert(title);  // Menu
alert(width);  // (prompt 的返回值)
我们还可以将冒号和等号结合起来：

let options = {
  title: "Menu"
};

let {width: w = 100, height: h = 200, title} = options;

alert(title);  // Menu
alert(w);      // 100
alert(h);      // 200
如果我们有一个具有很多属性的复杂对象，那么我们可以只提取所需的内容：

let options = {
  title: "Menu",
  width: 100,
  height: 200
};

// 仅提取 title 作为变量
let { title } = options;

alert(title); // Menu
剩余模式（pattern）“…”
如果对象拥有的属性数量比我们提供的变量数量还多，该怎么办？我们可以只取其中的某一些属性，然后把“剩余的”赋值到其他地方吗？

我们可以使用剩余模式（pattern），与数组类似。一些较旧的浏览器不支持此功能（例如 IE，可以使用 Babel 对其进行 polyfill），但可以在现代浏览器中使用。

看起来就像这样：

let options = {
  title: "Menu",
  height: 200,
  width: 100
};

// title = 名为 title 的属性
// rest = 存有剩余属性的对象
let {title, ...rest} = options;

// 现在 title="Menu", rest={height: 200, width: 100}
alert(rest.height);  // 200
alert(rest.width);   // 100
不使用 let 时的陷阱
在上面的示例中，变量都是在赋值中通过正确方式声明的：let {…} = {…}。当然，我们也可以使用已有的变量，而不用 let，但这里有一个陷阱。

以下代码无法正常运行：

let title, width, height;

// 这一行发生了错误
{title, width, height} = {title: "Menu", width: 200, height: 100};
问题在于 JavaScript 把主代码流（即不在其他表达式中）的 {...} 当做一个代码块。这样的代码块可以用于对语句分组，如下所示：

{
  // 一个代码块
  let message = "Hello";
  // ...
  alert( message );
}
因此，这里 JavaScript 假定我们有一个代码块，这就是报错的原因。我们需要解构它。

为了告诉 JavaScript 这不是一个代码块，我们可以把整个赋值表达式用括号 (...) 包起来：

let title, width, height;

// 现在就可以了
({title, width, height} = {title: "Menu", width: 200, height: 100});

alert( title ); // Menu
```
## 10.3 嵌套解构
```
如果一个对象或数组嵌套了其他的对象和数组，我们可以在等号左侧使用更复杂的模式（pattern）来提取更深层的数据。

在下面的代码中，options 的属性 size 是另一个对象，属性 items 是另一个数组。赋值语句中等号左侧的模式（pattern）具有相同的结构以从中提取值：

let options = {
  size: {
    width: 100,
    height: 200
  },
  items: ["Cake", "Donut"],
  extra: true
};

// 为了清晰起见，解构赋值语句被写成多行的形式
let {
  size: { // 把 size 赋值到这里
    width,
    height
  },
  items: [item1, item2], // 把 items 赋值到这里
  title = "Menu" // 在对象中不存在（使用默认值）
} = options;

alert(title);  // Menu
alert(width);  // 100
alert(height); // 200
alert(item1);  // Cake
alert(item2);  // Donut
对象 options 的所有属性，除了 extra 属性在等号左侧不存在，都被赋值给了对应的变量：


最终，我们得到了 width、height、item1、item2 和具有默认值的 title 变量。

注意，size 和 items 没有对应的变量，因为我们取的是它们的内容。
```
## 10.4 智能函数参数
```
有时，一个函数有很多参数，其中大部分的参数都是可选的。对用户界面来说更是如此。想象一个创建菜单的函数。它可能具有宽度参数，高度参数，标题参数和项目列表等。

下面是实现这种函数的一个很不好的写法：

function showMenu(title = "Untitled", width = 200, height = 100, items = []) {
  // ...
}
在实际开发中，记忆如此多的参数的位置是一个很大的负担。通常集成开发环境（IDE）会尽力帮助我们，特别是当代码有良好的文档注释的时候，但是…… 另一个问题就是，在大部分的参数只需采用默认值的情况下，调用这个函数时会需要写大量的 undefined。

像这样：

// 在采用默认值就可以的位置设置 undefined
showMenu("My Menu", undefined, undefined, ["Item1", "Item2"])
这太难看了。而且，当我们处理更多参数的时候可读性会变得很差。

解构赋值可以解决这些问题。

我们可以用一个对象来传递所有参数，而函数负责把这个对象解构成各个参数：

// 我们传递一个对象给函数
let options = {
  title: "My menu",
  items: ["Item1", "Item2"]
};

// ……然后函数马上把对象解构成变量
function showMenu({title = "Untitled", width = 200, height = 100, items = []}) {
  // title, items – 提取于 options，
  // width, height – 使用默认值
  alert( `${title} ${width} ${height}` ); // My Menu 200 100
  alert( items ); // Item1, Item2
}

showMenu(options);
我们也可以使用带有嵌套对象和冒号映射的更加复杂的解构：

let options = {
  title: "My menu",
  items: ["Item1", "Item2"]
};

function showMenu({
  title = "Untitled",
  width: w = 100,  // width goes to w
  height: h = 200, // height goes to h
  items: [item1, item2] // items first element goes to item1, second to item2
}) {
  alert( `${title} ${w} ${h}` ); // My Menu 100 200
  alert( item1 ); // Item1
  alert( item2 ); // Item2
}

showMenu(options);
完整语法和解构赋值是一样的：

function({
  incomingProperty: varName = defaultValue
  ...
})
对于参数对象，属性 incomingProperty 对应的变量是 varName，默认值是 defaultValue。

请注意，这种解构假定了 showMenu() 函数确实存在参数。如果我们想让所有的参数都使用默认值，那我们应该传递一个空对象：

showMenu({}); // 不错，所有值都取默认值

showMenu(); // 这样会导致错误
我们可以通过指定空对象 {} 为整个参数对象的默认值来解决这个问题：

function showMenu({ title = "Menu", width = 100, height = 200 } = {}) {
  alert( `${title} ${width} ${height}` );
}

showMenu(); // Menu 100 200
在上面的代码中，整个参数对象的默认是 {}，因此总会有内容可以用来解构。
```

# 11. 日期和时间
```javascript
让我们来学习一个新的内建对象：日期（Date）。该对象存储日期和时间，并提供了日期/时间的管理方法。

我们可以使用它来存储创建/修改时间，测量时间，或者仅用来打印当前时间。
```
## 11.1 创建
```
调用 new Date() 来创建一个新的 Date 对象。在调用时可以带有一些参数，如下所示：

new Date()
不带参数 —— 创建一个表示当前日期和时间的 Date 对象：

let now = new Date();
alert( now ); // 显示当前的日期/时间
new Date(milliseconds)
创建一个 Date 对象，其时间等于 1970 年 1 月 1 日 UTC+0 之后经过的毫秒数（1/1000 秒）。

// 0 表示 01.01.1970 UTC+0
let Jan01_1970 = new Date(0);
alert( Jan01_1970 );

// 现在增加 24 小时，得到 02.01.1970 UTC+0
let Jan02_1970 = new Date(24 * 3600 * 1000);
alert( Jan02_1970 );
传入的整数参数代表的是自 1970-01-01 00:00:00 以来经过的毫秒数，该整数被称为 时间戳。

这是一种日期的轻量级数字表示形式。我们通常使用 new Date(timestamp) 通过时间戳来创建日期，并可以使用 date.getTime() 将现有的 Date 对象转化为时间戳（下文会讲到）。

在 01.01.1970 之前的日期带有负的时间戳，例如：

// 31 Dec 1969
let Dec31_1969 = new Date(-24 * 3600 * 1000);
alert( Dec31_1969 );
new Date(datestring)
如果只有一个参数，并且是字符串，那么它会被自动解析。该算法与 Date.parse 所使用的算法相同，将在下文中进行介绍。

let date = new Date("2017-01-26");
alert(date);
// 未指定具体时间，所以假定时间为格林尼治标准时间（GMT）的午夜零点
// 并根据运行代码时的用户的时区进行调整
// 因此，结果可能是
// Thu Jan 26 2017 11:00:00 GMT+1100 (Australian Eastern Daylight Time)
// 或
// Wed Jan 25 2017 16:00:00 GMT-0800 (Pacific Standard Time)
new Date(year, month, date, hours, minutes, seconds, ms)
使用当前时区中的给定组件创建日期。只有前两个参数是必须的。

year 应该是四位数。为了兼容性，也接受 2 位数，并将其视为 19xx，例如 98 与 1998 相同，但强烈建议始终使用 4 位数。
month 计数从 0（一月）开始，到 11（十二月）结束。
date 是当月的具体某一天，如果缺失，则为默认值 1。
如果 hours/minutes/seconds/ms 缺失，则均为默认值 0。
例如：

new Date(2011, 0, 1, 0, 0, 0, 0); // 1 Jan 2011, 00:00:00
new Date(2011, 0, 1); // 同样，时分秒等均为默认值 0
时间度量最大精确到 1 毫秒（1/1000 秒）：

let date = new Date(2011, 0, 1, 2, 3, 4, 567);
alert( date ); // 1.01.2011, 02:03:04.567
```
## 11.2 访问日期组件
```
从 Date 对象中访问年、月等信息有多种方式：

getFullYear()
获取年份（4 位数）
getMonth()
获取月份，从 0 到 11。
getDate()
获取当月的具体日期，从 1 到 31，这个方法名称可能看起来有些令人疑惑。
getHours()，getMinutes()，getSeconds()，getMilliseconds()
获取相应的时间组件。
不是 getYear()，而是 getFullYear()
很多 JavaScript 引擎都实现了一个非标准化的方法 getYear()。不推荐使用这个方法。它有时候可能会返回 2 位的年份信息。永远不要使用它。要获取年份就使用 getFullYear()。

另外，我们还可以获取一周中的第几天：

getDay()
获取一周中的第几天，从 0（星期日）到 6（星期六）。第一天始终是星期日，在某些国家可能不是这样的习惯，但是这不能被改变。
以上的所有方法返回的组件都是基于当地时区的。

当然，也有与当地时区的 UTC 对应项，它们会返回基于 UTC+0 时区的日、月、年等：getUTCFullYear()，getUTCMonth()，getUTCDay()。只需要在 "get" 之后插入 "UTC" 即可。

如果你当地时区相对于 UTC 有偏移，那么下面代码会显示不同的小时数：

//  当前日期
let date = new Date();

// 当地时区的小时数
alert( date.getHours() );

// 在 UTC+0 时区的小时数（非夏令时的伦敦时间）
alert( date.getUTCHours() );
除了上述给定的方法，还有两个没有 UTC 变体的特殊方法：

getTime()
返回日期的时间戳 —— 从 1970-1-1 00:00:00 UTC+0 开始到现在所经过的毫秒数。

getTimezoneOffset()
返回 UTC 与本地时区之间的时差，以分钟为单位：

// 如果你在时区 UTC-1，输出 60
// 如果你在时区 UTC+3，输出 -180
alert( new Date().getTimezoneOffset() );
```
## 11.3 设置日期组件
```
下列方法可以设置日期/时间组件：

setFullYear(year, [month], [date])
setMonth(month, [date])
setDate(date)
setHours(hour, [min], [sec], [ms])
setMinutes(min, [sec], [ms])
setSeconds(sec, [ms])
setMilliseconds(ms)
setTime(milliseconds)（使用自 1970-01-01 00:00:00 UTC+0 以来的毫秒数来设置整个日期）
以上方法除了 setTime() 都有 UTC 变体，例如：setUTCHours()。

我们可以看到，有些方法可以一次性设置多个组件，比如 setHours。未提及的组件不会被修改。

举个例子：

let today = new Date();

today.setHours(0);
alert(today); // 日期依然是今天，但是小时数被改为了 0

today.setHours(0, 0, 0, 0);
alert(today); // 日期依然是今天，时间为 00:00:00。
```
## 11.4 自动校准
```
自动校准 是 Date 对象的一个非常方便的特性。我们可以设置超范围的数值，它会自动校准。

举个例子：

let date = new Date(2013, 0, 32); // 32 Jan 2013 ?!?
alert(date); // ……是 1st Feb 2013!
超出范围的日期组件将会被自动分配。

假设我们要在日期 “28 Feb 2016” 上加 2 天。结果可能是 “2 Mar” 或 “1 Mar”，因为存在闰年。但是我们不需要考虑这些，只需要直接加 2 天，剩下的 Date 对象会帮我们处理：

let date = new Date(2016, 1, 28);
date.setDate(date.getDate() + 2);

alert( date ); // 1 Mar 2016
这个特性经常被用来获取给定时间段后的日期。例如，我们想获取“现在 70 秒后”的日期：

let date = new Date();
date.setSeconds(date.getSeconds() + 70);

alert( date ); // 显示正确的日期信息
我们还可以设置 0 甚至可以设置负值。例如：

let date = new Date(2016, 0, 2); // 2016 年 1 月 2 日

date.setDate(1); // 设置为当月的第一天
alert( date );

date.setDate(0); // 天数最小可以设置为 1，所以这里设置的是上一月的最后一天
alert( date ); // 31 Dec 2015
```
## 11.5 日期转化为数字，日期差值
```
当 Date 对象被转化为数字时，得到的是对应的时间戳，与使用 date.getTime() 的结果相同：

let date = new Date();
alert(+date); // 以毫秒为单位的数值，与使用 date.getTime() 的结果相同
有一个重要的副作用：日期可以相减，相减的结果是以毫秒为单位时间差。

这个作用可以用于时间测量：

let start = new Date(); // 开始测量时间

// do the job
for (let i = 0; i < 100000; i++) {
  let doSomething = i * i * i;
}

let end = new Date(); // 结束测量时间

alert( `The loop took ${end - start} ms` );
```
## 11.6 Date.now()
```
如果我们仅仅想要测量时间间隔，我们不需要 Date 对象。

有一个特殊的方法 Date.now()，它会返回当前的时间戳。

它相当于 new Date().getTime()，但它不会创建中间的 Date 对象。因此它更快，而且不会对垃圾回收造成额外的压力。

这种方法很多时候因为方便，又或是因性能方面的考虑而被采用，例如使用 JavaScript 编写游戏或其他的特殊应用场景。

因此这样做可能会更好：

let start = Date.now(); // 从 1 Jan 1970 至今的时间戳

// do the job
for (let i = 0; i < 100000; i++) {
  let doSomething = i * i * i;
}

let end = Date.now(); // 完成

alert( `The loop took ${end - start} ms` ); // 相减的是时间戳，而不是日期
```
## 11.7 基准测试
```
在对一个很耗 CPU 性能的函数进行可靠的基准测试（Benchmarking）时，我们需要谨慎一点。

例如，我们想判断以下两个计算日期差值的函数：哪个更快？

这种性能测量通常称为“基准测试（benchmark）”。

// 我们有 date1 和 date2，哪个函数会更快地返回两者的时间差？
function diffSubtract(date1, date2) {
  return date2 - date1;
}

// or
function diffGetTime(date1, date2) {
  return date2.getTime() - date1.getTime();
}
这两个函数做的事情完全相同，但是其中一个函数使用显式的 date.getTime() 来获取毫秒形式的日期，另一个则依赖于“日期 — 数字”的转换。它们的结果是一样的。

那么，哪个更快呢？

首先想到的方法可能是连续运行两者很多次，并计算所消耗的时间之差。就这个例子而言，函数过于简单，所以我们必须执行至少 100000 次。

让我们开始测量：

function diffSubtract(date1, date2) {
  return date2 - date1;
}

function diffGetTime(date1, date2) {
  return date2.getTime() - date1.getTime();
}

function bench(f) {
  let date1 = new Date(0);
  let date2 = new Date();

  let start = Date.now();
  for (let i = 0; i < 100000; i++) f(date1, date2);
  return Date.now() - start;
}

alert( 'Time of diffSubtract: ' + bench(diffSubtract) + 'ms' );
alert( 'Time of diffGetTime: ' + bench(diffGetTime) + 'ms' );
看起来使用 getTime() 这种方式快得多，这是因为它没有进行类型转换，对引擎优化来说更加简单。

我们得到了结论，但是这并不是一个很好的度量的例子。

想象一下当运行 bench(diffSubtract) 的同时，CPU 还在并行处理其他事务，并且这也会占用资源。然而，运行 bench(diffGetTime) 的时候，并行处理的事务完成了。

对于现代多进程操作系统来说，这是一个非常常见的场景。

比起第二个函数，第一个函数所能使用的 CPU 资源更少。这可能导致错误的结论。

为了得到更加可靠的度量，整个度量测试包应该重新运行多次。

例如，像下面的代码这样：

function diffSubtract(date1, date2) {
  return date2 - date1;
}

function diffGetTime(date1, date2) {
  return date2.getTime() - date1.getTime();
}

function bench(f) {
  let date1 = new Date(0);
  let date2 = new Date();

  let start = Date.now();
  for (let i = 0; i < 100000; i++) f(date1, date2);
  return Date.now() - start;
}

let time1 = 0;
let time2 = 0;

// 交替运行 bench(diffSubtract) 和 bench(diffGetTime) 各 10 次
for (let i = 0; i < 10; i++) {
  time1 += bench(diffSubtract);
  time2 += bench(diffGetTime);
}

alert( 'Total time for diffSubtract: ' + time1 );
alert( 'Total time for diffGetTime: ' + time2 );
现代的 JavaScript 引擎的先进优化策略只对执行很多次的 “hot code” 有效（对于执行很少次数的代码没有必要优化）。因此，在上面的例子中，第一次执行的优化程度不高。我们可能需要增加一个预热步骤：

// 在主循环中增加预热环节
bench(diffSubtract);
bench(diffGetTime);

// 开始度量
for (let i = 0; i < 10; i++) {
  time1 += bench(diffSubtract);
  time2 += bench(diffGetTime);
}
进行微型基准测试时要小心
现代的 JavaScript 引擎执行了很多优化。与正常编写的代码相比，它们可能会改变“人为编写的专用于测试的代码”的执行流程，特别是在我们对很小的代码片段进行基准测试时，例如某个运算符或内建函数的工作方式。因此，为了深入理解性能问题，请学习 JavaScript 引擎的工作原理。在那之后，你或许再也不需要进行微型基准测试了。

http://mrale.ph 提供了很多 V8 引擎相关的文章。
```
## 11.8 对字符串调用
```
Date.parse(str) 方法可以从一个字符串中读取日期。

字符串的格式应该为：YYYY-MM-DDTHH:mm:ss.sssZ，其中：

YYYY-MM-DD —— 日期：年-月-日。
字符 "T" 是一个分隔符。
HH:mm:ss.sss —— 时间：小时，分钟，秒，毫秒。
可选字符 'Z' 为 +-hh:mm 格式的时区。单个字符 Z 代表 UTC+0 时区。
简短形式也是可以的，比如 YYYY-MM-DD 或 YYYY-MM，甚至可以是 YYYY。

Date.parse(str) 调用会解析给定格式的字符串，并返回时间戳（自 1970-01-01 00:00:00 起所经过的毫秒数）。如果给定字符串的格式不正确，则返回 NaN。

举个例子：

let ms = Date.parse('2012-01-26T13:51:50.417-07:00');

alert(ms); // 1327611110417  (时间戳)
我们可以通过时间戳来立即创建一个 new Date 对象：

let date = new Date( Date.parse('2012-01-26T13:51:50.417-07:00') );

alert(date);
```

# 12. JSON 方法，toJSON
```javascript
假设我们有一个复杂的对象，我们希望将其转换为字符串，以通过网络发送，或者只是为了在日志中输出它。

当然，这样的字符串应该包含所有重要的属性。

我们可以像这样实现转换：

let user = {
  name: "John",
  age: 30,

  toString() {
    return `{name: "${this.name}", age: ${this.age}}`;
  }
};

alert(user); // {name: "John", age: 30}
……但在开发过程中，会新增一些属性，旧的属性会被重命名和删除。每次更新这种 toString 都会非常痛苦。我们可以尝试遍历其中的属性，但是如果对象很复杂，并且在属性中嵌套了对象呢？我们也需要对它们进行转换。

幸运的是，不需要编写代码来处理所有这些问题。这项任务已经解决了。
```
## 12.1 JSON.stringify
```
JSON（JavaScript Object Notation）是表示值和对象的通用格式。在 RFC 4627 标准中有对其的描述。最初它是为 JavaScript 而创建的，但许多其他编程语言也有用于处理它的库。因此，当客户端使用 JavaScript 而服务器端是使用 Ruby/PHP/Java 等语言编写的时，使用 JSON 可以很容易地进行数据交换。

JavaScript 提供了如下方法：

JSON.stringify 将对象转换为 JSON。
JSON.parse 将 JSON 转换回对象。
例如，在这里我们 JSON.stringify 一个 student 对象：

let student = {
  name: 'John',
  age: 30,
  isAdmin: false,
  courses: ['html', 'css', 'js'],
  spouse: null
};

let json = JSON.stringify(student);

alert(typeof json); // we've got a string!

alert(json);
/* JSON 编码的对象：
{
  "name": "John",
  "age": 30,
  "isAdmin": false,
  "courses": ["html", "css", "js"],
  "spouse": null
}
*/
方法 JSON.stringify(student) 接收对象并将其转换为字符串。

得到的 json 字符串是一个被称为 JSON 编码（JSON-encoded） 或 序列化（serialized） 或 字符串化（stringified） 或 编组化（marshalled） 的对象。我们现在已经准备好通过有线发送它或将其放入普通数据存储。

请注意，JSON 编码的对象与对象字面量有几个重要的区别：

字符串使用双引号。JSON 中没有单引号或反引号。所以 'John' 被转换为 "John"。
对象属性名称也是双引号的。这是强制性的。所以 age:30 被转换成 "age":30。
JSON.stringify 也可以应用于原始（primitive）数据类型。

JSON 支持以下数据类型：

Objects { ... }
Arrays [ ... ]
Primitives：
strings，
numbers，
boolean values true/false，
null。
例如：

// 数字在 JSON 还是数字
alert( JSON.stringify(1) ) // 1

// 字符串在 JSON 中还是字符串，只是被双引号扩起来
alert( JSON.stringify('test') ) // "test"

alert( JSON.stringify(true) ); // true

alert( JSON.stringify([1, 2, 3]) ); // [1,2,3]
JSON 是语言无关的纯数据规范，因此一些特定于 JavaScript 的对象属性会被 JSON.stringify 跳过。

即：

函数属性（方法）。
Symbol 类型的键和值。
存储 undefined 的属性。
let user = {
  sayHi() { // 被忽略
    alert("Hello");
  },
  [Symbol("id")]: 123, // 被忽略
  something: undefined // 被忽略
};

alert( JSON.stringify(user) ); // {}（空对象）
通常这很好。如果这不是我们想要的方式，那么我们很快就会看到如何自定义转换方式。

最棒的是支持嵌套对象转换，并且可以自动对其进行转换。

例如：

let meetup = {
  title: "Conference",
  room: {
    number: 23,
    participants: ["john", "ann"]
  }
};

alert( JSON.stringify(meetup) );
/* 整个结构都被字符串化了
{
  "title":"Conference",
  "room":{"number":23,"participants":["john","ann"]},
}
*/
重要的限制：不得有循环引用。

例如：

let room = {
  number: 23
};

let meetup = {
  title: "Conference",
  participants: ["john", "ann"]
};

meetup.place = room;       // meetup 引用了 room
room.occupiedBy = meetup; // room 引用了 meetup

JSON.stringify(meetup); // Error: Converting circular structure to JSON
在这里，转换失败了，因为循环引用：room.occupiedBy 引用了 meetup，meetup.place 引用了 room：


```
## 12.2 排除和转换：replacer
```
JSON.stringify 的完整语法是：

let json = JSON.stringify(value[, replacer, space])
value
要编码的值。
replacer
要编码的属性数组或映射函数 function(key, value)。
space
用于格式化的空格数量。
大部分情况，JSON.stringify 仅与第一个参数一起使用。但是，如果我们需要微调替换过程，比如过滤掉循环引用，我们可以使用 JSON.stringify 的第二个参数。

如果我们传递一个属性数组给它，那么只有这些属性会被编码。

例如：

let room = {
  number: 23
};

let meetup = {
  title: "Conference",
  participants: [{name: "John"}, {name: "Alice"}],
  place: room // meetup 引用了 room
};

room.occupiedBy = meetup; // room 引用了 meetup

alert( JSON.stringify(meetup, ['title', 'participants']) );
// {"title":"Conference","participants":[{},{}]}
这里我们可能过于严格了。属性列表应用于了整个对象结构。所以 participants 是空的，因为 name 不在列表中。

让我们包含除了会导致循环引用的 room.occupiedBy 之外的所有属性：

let room = {
  number: 23
};

let meetup = {
  title: "Conference",
  participants: [{name: "John"}, {name: "Alice"}],
  place: room // meetup 引用了 room
};

room.occupiedBy = meetup; // room 引用了 meetup

alert( JSON.stringify(meetup, ['title', 'participants', 'place', 'name', 'number']) );
/*
{
  "title":"Conference",
  "participants":[{"name":"John"},{"name":"Alice"}],
  "place":{"number":23}
}
*/
现在，除 occupiedBy 以外的所有内容都被序列化了。但是属性的列表太长了。

幸运的是，我们可以使用一个函数代替数组作为 replacer。

该函数会为每个 (key,value) 对调用并返回“已替换”的值，该值将替换原有的值。如果值被跳过了，则为 undefined。

在我们的例子中，我们可以为 occupiedBy 以外的所有内容按原样返回 value。对于 occupiedBy，下面的代码返回 undefined：

let room = {
  number: 23
};

let meetup = {
  title: "Conference",
  participants: [{name: "John"}, {name: "Alice"}],
  place: room // meetup 引用了 room
};

room.occupiedBy = meetup; // room 引用了 meetup

alert( JSON.stringify(meetup, function replacer(key, value) {
  alert(`${key}: ${value}`);
  return (key == 'occupiedBy') ? undefined : value;
}));

/* key:value pairs that come to replacer:
:             [object Object]
title:        Conference
participants: [object Object],[object Object]
0:            [object Object]
name:         John
1:            [object Object]
name:         Alice
place:        [object Object]
number:       23
occupiedBy: [object Object]
*/
请注意 replacer 函数会获取每个键/值对，包括嵌套对象和数组项。它被递归地应用。replacer 中的 this 的值是包含当前属性的对象。

第一个调用很特别。它是使用特殊的“包装对象”制作的：{"": meetup}。换句话说，第一个 (key, value) 对的键是空的，并且该值是整个目标对象。这就是上面的示例中第一行是 ":[object Object]" 的原因。

这个理念是为了给 replacer 提供尽可能多的功能：如果有必要，它有机会分析并替换/跳过整个对象。
```
## 12.3 格式化：space
```
JSON.stringify(value, replacer, spaces) 的第三个参数是用于优化格式的空格数量。

以前，所有字符串化的对象都没有缩进和额外的空格。如果我们想通过网络发送一个对象，那就没什么问题。space 参数专门用于调整出更美观的输出。

这里的 space = 2 告诉 JavaScript 在多行中显示嵌套的对象，对象内部缩进 2 个空格：

let user = {
  name: "John",
  age: 25,
  roles: {
    isAdmin: false,
    isEditor: true
  }
};

alert(JSON.stringify(user, null, 2));
/* 两个空格的缩进：
{
  "name": "John",
  "age": 25,
  "roles": {
    "isAdmin": false,
    "isEditor": true
  }
}
*/

/* 对于 JSON.stringify(user, null, 4) 的结果会有更多缩进：
{
    "name": "John",
    "age": 25,
    "roles": {
        "isAdmin": false,
        "isEditor": true
    }
}
*/
第三个参数也可以是字符串。在这种情况下，字符串用于缩进，而不是空格的数量。

spaces 参数仅用于日志记录和美化输出。
```
## 12.4 自定义"toJSON"
```
像 toString 进行字符串转换，对象也可以提供 toJSON 方法来进行 JSON 转换。如果可用，JSON.stringify 会自动调用它。

例如：

let room = {
  number: 23
};

let meetup = {
  title: "Conference",
  date: new Date(Date.UTC(2017, 0, 1)),
  room
};

alert( JSON.stringify(meetup) );
/*
  {
    "title":"Conference",
    "date":"2017-01-01T00:00:00.000Z",  // (1)
    "room": {"number":23}               // (2)
  }
*/
在这儿我们可以看到 date (1) 变成了一个字符串。这是因为所有日期都有一个内建的 toJSON 方法来返回这种类型的字符串。

现在让我们为对象 room 添加一个自定义的 toJSON：

let room = {
  number: 23,
  toJSON() {
    return this.number;
  }
};

let meetup = {
  title: "Conference",
  room
};

alert( JSON.stringify(room) ); // 23

alert( JSON.stringify(meetup) );
/*
  {
    "title":"Conference",
    "room": 23
  }
*/
正如我们所看到的，toJSON 既可以用于直接调用 JSON.stringify(room) 也可以用于当 room 嵌套在另一个编码对象中时。
```
## 12.5 JSON.parse
```
要解码 JSON 字符串，我们需要另一个方法 JSON.parse。

语法：

let value = JSON.parse(str, [reviver]);
str
要解析的 JSON 字符串。
reviver
可选的函数 function(key,value)，该函数将为每个 (key, value) 对调用，并可以对值进行转换。
例如：

// 字符串化数组
let numbers = "[0, 1, 2, 3]";

numbers = JSON.parse(numbers);

alert( numbers[1] ); // 1
对于嵌套对象：

let userData = '{ "name": "John", "age": 35, "isAdmin": false, "friends": [0,1,2,3] }';

let user = JSON.parse(userData);

alert( user.friends[1] ); // 1
JSON 可能会非常复杂，对象和数组可以包含其他对象和数组。但是它们必须遵循相同的 JSON 格式。

以下是手写 JSON 时的典型错误（有时我们必须出于调试目的编写它）：

let json = `{
  name: "John",                     // 错误：属性名没有双引号
  "surname": 'Smith',               // 错误：值使用的是单引号（必须使用双引号）
  'isAdmin': false                  // 错误：键使用的是单引号（必须使用双引号）
  "birthday": new Date(2000, 2, 3), // 错误：不允许使用 "new"，只能是裸值
  "friends": [0,1,2,3]              // 这个没问题
}`;
此外，JSON 不支持注释。向 JSON 添加注释无效。

还有另一种名为 JSON5 的格式，它允许未加引号的键，也允许注释等。但这是一个独立的库，不在语言的规范中。

标准 JSON 格式之所以如此严格，并不是因为它的制定者们偷懒，而是为了能够简单，可靠且快速地实现解析算法。
```
## 12.6 使用reviver
```
想象一下，我们从服务器上获得了一个字符串化的 meetup 对象。

它看起来像这样：

// title: (meetup title), date: (meetup date)
let str = '{"title":"Conference","date":"2017-11-30T12:00:00.000Z"}';
……现在我们需要对它进行 反序列（deserialize），把它转换回 JavaScript 对象。

让我们通过调用 JSON.parse 来完成：

let str = '{"title":"Conference","date":"2017-11-30T12:00:00.000Z"}';

let meetup = JSON.parse(str);

alert( meetup.date.getDate() ); // Error!
啊！报错了！

meetup.date 的值是一个字符串，而不是 Date 对象。JSON.parse 怎么知道应该将字符串转换为 Date 呢？

让我们将 reviver 函数传递给 JSON.parse 作为第二个参数，该函数按照“原样”返回所有值，但是 date 会变成 Date：

let str = '{"title":"Conference","date":"2017-11-30T12:00:00.000Z"}';

let meetup = JSON.parse(str, function(key, value) {
  if (key == 'date') return new Date(value);
  return value;
});

alert( meetup.date.getDate() ); // 现在正常运行了！
顺便说一下，这也适用于嵌套对象：

let schedule = `{
  "meetups": [
    {"title":"Conference","date":"2017-11-30T12:00:00.000Z"},
    {"title":"Birthday","date":"2017-04-18T12:00:00.000Z"}
  ]
}`;

schedule = JSON.parse(schedule, function(key, value) {
  if (key == 'date') return new Date(value);
  return value;
});

alert( schedule.meetups[1].date.getDate() ); // 正常运行了！
```