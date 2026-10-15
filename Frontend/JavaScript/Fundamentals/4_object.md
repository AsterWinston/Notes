# 1. 对象
```javascript
正如我们在 数据类型 一章学到的，JavaScript 中有八种数据类型。有七种原始类型，因为它们的值只包含一种东西（字符串，数字或者其他）。

相反，对象则用来存储键值对和更复杂的实体。在 JavaScript 中，对象几乎渗透到了这门编程语言的方方面面。所以，在我们深入理解这门语言之前，必须先理解对象。

我们可以通过使用带有可选 属性列表 的花括号 {…} 来创建对象。一个属性就是一个键值对（“key: value”），其中键（key）是一个字符串（也叫做属性名），值（value）可以是任何值。

我们可以把对象想象成一个带有签名文件的文件柜。每一条数据都基于键（key）存储在文件中。这样我们就可以很容易根据文件名（也就是“键”）查找文件或添加/删除文件了。


我们可以用下面两种语法中的任一种来创建一个空的对象（“空柜子”）：

let user = new Object(); // “构造函数” 的语法
let user = {};  // “字面量” 的语法

通常，我们用花括号。这种方式我们叫做 字面量。
```
## 1.1 文本和属性
```
我们可以在创建对象的时候，立即将一些属性以键值对的形式放到 {...} 中。

let user = {     // 一个对象
  name: "John",  // 键 "name"，值 "John"
  age: 30        // 键 "age"，值 30
};
属性有键（或者也可以叫做“名字”或“标识符”），位于冒号 ":" 的前面，值在冒号的右边。

在 user 对象中，有两个属性：

第一个的键是 "name"，值是 "John"。
第二个的键是 "age"，值是 30。
生成的 user 对象可以被想象为一个放置着两个标记有 “name” 和 “age” 的文件的柜子。


我们可以随时添加、删除和读取文件。

可以使用点符号访问属性值：

// 读取文件的属性：
alert( user.name ); // John
alert( user.age ); // 30
属性的值可以是任意类型，让我们加个布尔类型：

user.isAdmin = true;

我们可以用 delete 操作符移除属性：

delete user.age;

我们也可以用多字词语来作为属性名，但必须给它们加上引号：

let user = {
  name: "John",
  age: 30,
  "likes birds": true  // 多词属性名必须加引号
};

列表中的最后一个属性应以逗号结尾：

let user = {
  name: "John",
  age: 30,
}
这叫做尾随（trailing）或悬挂（hanging）逗号。这样便于我们添加、删除和移动属性，因为所有的行都是相似的。
```
## 1.2 方括号
```
对于多词属性，点操作就不能用了：

// 这将提示有语法错误
user.likes birds = true
JavaScript 理解不了。它认为我们在处理 user.likes，然后在遇到意外的 birds 时给出了语法错误。

点符号要求 key 是有效的变量标识符。这意味着：不包含空格，不以数字开头，也不包含特殊字符（允许使用 $ 和 _）。

有另一种方法，就是使用方括号，可用于任何字符串：

let user = {};

// 设置
user["likes birds"] = true;

// 读取
alert(user["likes birds"]); // true

// 删除
delete user["likes birds"];
现在一切都可行了。请注意方括号中的字符串要放在引号中，单引号或双引号都可以。

方括号同样提供了一种可以通过任意表达式来获取属性名的方式 —— 与文本字符串不同 —— 例如下面的变量：

let key = "likes birds";

// 跟 user["likes birds"] = true; 一样
user[key] = true;
在这里，变量 key 可以是程序运行时计算得到的，也可以是根据用户的输入得到的。然后我们可以用它来访问属性。这给了我们很大的灵活性。

例如：

let user = {
  name: "John",
  age: 30
};

let key = prompt("What do you want to know about the user?", "name");

// 访问变量
alert( user[key] ); // John（如果输入 "name"）
点符号不能以类似的方式使用：

let user = {
  name: "John",
  age: 30
};

let key = "name";
alert( user.key ) // undefined
计算属性
当创建一个对象时，我们可以在对象字面量中使用方括号。这叫做 计算属性。

例如：

let fruit = prompt("Which fruit to buy?", "apple");

let bag = {
  [fruit]: 5, // 属性名是从 fruit 变量中得到的
};

alert( bag.apple ); // 5 如果 fruit="apple"
计算属性的含义很简单：[fruit] 含义是属性名应该从 fruit 变量中获取。

所以，如果一个用户输入 "apple"，bag 将变为 {apple: 5}。

本质上，这跟下面的语法效果相同：

let fruit = prompt("Which fruit to buy?", "apple");
let bag = {};

// 从 fruit 变量中获取值
bag[fruit] = 5;
……但是看起来更好。

我们可以在方括号中使用更复杂的表达式：

let fruit = 'apple';
let bag = {
  [fruit + 'Computers']: 5 // bag.appleComputers = 5
};
方括号比点符号更强大。它允许任何属性名和变量，但写起来也更加麻烦。

所以，大部分时间里，当属性名是已知且简单的时候，就使用点符号。如果我们需要一些更复杂的内容，那么就用方括号。
```
## 1.3 属性值简写
```
在实际开发中，我们通常用已存在的变量当做属性名。

例如：

function makeUser(name, age) {
  return {
    name: name,
    age: age,
    // ……其他的属性
  };
}

let user = makeUser("John", 30);
alert(user.name); // John
在上面的例子中，属性名跟变量名一样。这种通过变量生成属性的应用场景很常见，在这有一种特殊的 属性值缩写 方法，使属性名变得更短。

可以用 name 来代替 name:name 像下面那样：

function makeUser(name, age) {
  return {
    name, // 与 name: name 相同
    age,  // 与 age: age 相同
    // ...
  };
}
我们可以把属性名简写方式和正常方式混用：

let user = {
  name,  // 与 name:name 相同
  age: 30
};
```
## 1.4 属性名称限制
```
我们已经知道，变量名不能是编程语言的某个保留字，如 “for”、“let”、“return” 等……

但对象的属性名并不受此限制：

// 这些属性都没问题
let obj = {
  for: 1,
  let: 2,
  return: 3
};

alert( obj.for + obj.let + obj.return );  // 6
简而言之，属性命名没有限制。属性名可以是任何字符串或者 symbol（一种特殊的标志符类型，将在后面介绍）。

其他类型会被自动地转换为字符串。

例如，当数字 0 被用作对象的属性的键时，会被转换为字符串 "0"：

let obj = {
  0: "test" // 等同于 "0": "test"
};

// 都会输出相同的属性（数字 0 被转为字符串 "0"）
alert( obj["0"] ); // test
alert( obj[0] ); // test (相同的属性)
这里有个小陷阱：一个名为 __proto__ 的属性。我们不能将它设置为一个非对象的值：

let obj = {};
obj.__proto__ = 5; // 分配一个数字
alert(obj.__proto__); // [object Object] —— 值为对象，与预期结果不同
我们从代码中可以看出来，把它赋值为 5 的操作被忽略了。

我们将在 后续章节 中学习 __proto__ 的特殊性质，并给出了 解决此问题的方法。
```
## 1.5 属性存在测试，"in"操作符
```
相比于其他语言，JavaScript 的对象有一个需要注意的特性：能够被访问任何属性。即使属性不存在也不会报错！

读取不存在的属性只会得到 undefined。所以我们可以很容易地判断一个属性是否存在：

let user = {};

alert( user.noSuchProperty === undefined ); // true 意思是没有这个属性
这里还有一个特别的，检查属性是否存在的操作符 "in"。

语法是：

"key" in object
例如：

let user = { name: "John", age: 30 };

alert( "age" in user ); // true，user.age 存在
alert( "blabla" in user ); // false，user.blabla 不存在。
请注意，in 的左边必须是 属性名。通常是一个带引号的字符串。

如果我们省略引号，就意味着左边是一个变量，它应该包含要判断的实际属性名。例如：

let user = { age: 30 };

let key = "age";
alert( key in user ); // true，属性 "age" 存在
为何会有 in 运算符呢？与 undefined 进行比较来判断还不够吗？

确实，大部分情况下与 undefined 进行比较来判断就可以了。但有一个例外情况，这种比对方式会有问题，但 in 运算符的判断结果仍是对的。

那就是属性存在，但存储的值是 undefined 的时候：

let obj = {
  test: undefined
};

alert( obj.test ); // 显示 undefined，所以属性不存在？

alert( "test" in obj ); // true，属性存在！
在上面的代码中，属性 obj.test 事实上是存在的，所以 in 操作符检查通过。

这种情况很少发生，因为通常情况下不应该给对象赋值 undefined。我们通常会用 null 来表示未知的或者空的值。因此，in 运算符是代码中的特殊来宾。
```
## 1.6 "for...in"循环
```
为了遍历一个对象的所有键（key），可以使用一个特殊形式的循环：for..in。这跟我们在前面学到的 for(;;) 循环是完全不一样的东西。

语法：

for (key in object) {
  // 对此对象属性中的每个键执行的代码
}
例如，让我们列出 user 所有的属性：

let user = {
  name: "John",
  age: 30,
  isAdmin: true
};

for (let key in user) {
  // keys
  alert( key );  // name, age, isAdmin
  // 属性键的值
  alert( user[key] ); // John, 30, true
}
注意，所有的 “for” 结构体都允许我们在循环中定义变量，像这里的 let key。

同样，我们可以用其他属性名来替代 key。例如 "for(let prop in obj)" 也很常用。

像对象一样排序
对象有顺序吗？换句话说，如果我们遍历一个对象，我们获取属性的顺序是和属性添加时的顺序相同吗？这靠谱吗？

简短的回答是：“有特别的顺序”：整数属性会被进行排序，其他属性则按照创建的顺序显示。详情如下：

例如，让我们考虑一个带有电话号码的对象：

let codes = {
  "49": "Germany",
  "41": "Switzerland",
  "44": "Great Britain",
  // ..,
  "1": "USA"
};

for(let code in codes) {
  alert(code); // 1, 41, 44, 49
}
对象可用于面向用户的建议选项列表。如果我们的网站主要面向德国观众，那么我们可能希望 49 排在第一。

但如果我们执行代码，会看到完全不同的现象：

USA (1) 排在了最前面
然后是 Switzerland (41) 及其它。
因为这些电话号码是整数，所以它们以升序排列。所以我们看到的是 1, 41, 44, 49。

整数属性？那是什么？
这里的“整数属性”指的是一个可以在不做任何更改的情况下与一个整数进行相互转换的字符串。

所以，"49" 是一个整数属性名，因为我们把它转换成整数，再转换回来，它还是一样的。但是 “+49” 和 “1.2” 就不行了：

// Number(...) 显式转换为数字
// Math.trunc 是内建的去除小数部分的方法。
alert( String(Math.trunc(Number("49"))) ); // "49"，相同，整数属性
alert( String(Math.trunc(Number("+49"))) ); // "49"，不同于 "+49" ⇒ 不是整数属性
alert( String(Math.trunc(Number("1.2"))) ); // "1"，不同于 "1.2" ⇒ 不是整数属性
……此外，如果属性名不是整数，那它们就按照创建时的顺序来排序，例如：

let user = {
  name: "John",
  surname: "Smith"
};
user.age = 25; // 增加一个

// 非整数属性是按照创建的顺序来排列的
for (let prop in user) {
  alert( prop ); // name, surname, age
}
所以，为了解决电话号码的问题，我们可以使用非整数属性名来 欺骗 程序。只需要给每个键名加一个加号 "+" 前缀就行了。

像这样：

let codes = {
  "+49": "Germany",
  "+41": "Switzerland",
  "+44": "Great Britain",
  // ..,
  "+1": "USA"
};

for (let code in codes) {
  alert( +code ); // 49, 41, 44, 1
}
现在跟预想的一样了。
```


# 2. 对象引用和复制
```javascript
对象与原始类型的根本区别之一是，对象是“通过引用”存储和复制的，而原始类型：字符串、数字、布尔值等 —— 总是“作为一个整体”复制。

如果我们深入了解复制值时会发生什么，就很容易理解了。

让我们从原始类型开始，例如一个字符串。

这里我们将 message 复制到 phrase：

let message = "Hello!";
let phrase = message;
结果我们就有了两个独立的变量，每个都存储着字符串 "Hello!"。


显而易见的结果，对吧？

但是，对象不是这样的。

赋值了对象的变量存储的不是对象本身，而是该对象“在内存中的地址” —— 换句话说就是对该对象的“引用”。

让我们看一个这样的变量的例子：

let user = {
  name: "John"
};
这是它实际存储在内存中的方式：


该对象被存储在内存中的某个位置（在图片的右侧），而变量 user（在左侧）保存的是对其的“引用”。

我们可以将一个对象变量（例如 user）想象成一张写有对象的地址的纸。

当我们对对象执行操作时，例如获取一个属性 user.name，JavaScript 引擎会查看该地址中的内容，并在实际对象上执行操作。

现在，这就是为什么它很重要。

当一个对象变量被复制 —— 引用被复制，而该对象自身并没有被复制。

例如：

let user = { name: "John" };

let admin = user; // 复制引用
现在我们有了两个变量，它们保存的都是对同一个对象的引用：


正如你所看到的，这里仍然只有一个对象，但现在有两个引用它的变量。

我们可以通过其中任意一个变量来访问该对象并修改它的内容：

let user = { name: 'John' };

let admin = user;

admin.name = 'Pete'; // 通过 "admin" 引用来修改

alert(user.name); // 'Pete'，修改能通过 "user" 引用看到
这就像我们有一个带有两把钥匙的柜子，使用其中一把钥匙（admin）打开柜子并更改了里面的东西。那么，如果我们稍后用另一把钥匙（user），我们仍然可以打开同一个柜子并且可以访问更改的内容。
```
## 2.1 通过引用来比较
```
仅当两个对象为同一对象时，两者才相等。

例如，这里 a 和 b 两个变量都引用同一个对象，所以它们相等：

let a = {};
let b = a; // 复制引用

alert( a == b ); // true，都引用同一对象
alert( a === b ); // true
而这里两个独立的对象则并不相等，即使它们看起来很像（都为空）：

let a = {};
let b = {}; // 两个独立的对象

alert( a == b ); // false
对于类似 obj1 > obj2 的比较，或者跟一个原始类型值的比较 obj == 5，对象都会被转换为原始值。我们很快就会学到对象是如何转换的，但是说实话，很少需要进行这样的比较 —— 通常是在编程错误的时候才会出现这种情况。
```
## 2.2 克隆与合并，Object.assign
```
那么，拷贝一个对象变量会又创建一个对相同对象的引用。

但是，如果我们想要复制一个对象，那该怎么做呢？

我们可以创建一个新对象，通过遍历已有对象的属性，并在原始类型值的层面复制它们，以实现对已有对象结构的复制。

就像这样：

let user = {
  name: "John",
  age: 30
};

let clone = {}; // 新的空对象

// 将 user 中所有的属性拷贝到其中
for (let key in user) {
  clone[key] = user[key];
}

// 现在 clone 是带有相同内容的完全独立的对象
clone.name = "Pete"; // 改变了其中的数据

alert( user.name ); // 原来的对象中的 name 属性依然是 John
我们也可以使用 Object.assign 方法来达成同样的效果。

语法是：

Object.assign(dest, [src1, src2, src3...])
第一个参数 dest 是指目标对象。
更后面的参数 src1, ..., srcN（可按需传递多个参数）是源对象。
该方法将所有源对象的属性拷贝到目标对象 dest 中。换句话说，从第二个开始的所有参数的属性都被拷贝到第一个参数的对象中。
调用结果返回 dest。
例如，我们可以用它来合并多个对象：

let user = { name: "John" };

let permissions1 = { canView: true };
let permissions2 = { canEdit: true };

// 将 permissions1 和 permissions2 中的所有属性都拷贝到 user 中
Object.assign(user, permissions1, permissions2);

// 现在 user = { name: "John", canView: true, canEdit: true }
如果被拷贝的属性的属性名已经存在，那么它会被覆盖：

let user = { name: "John" };

Object.assign(user, { name: "Pete" });

alert(user.name); // 现在 user = { name: "Pete" }
我们也可以用 Object.assign 代替 for..in 循环来进行简单克隆：

let user = {
  name: "John",
  age: 30
};

let clone = Object.assign({}, user);
它将 user 中的所有属性拷贝到了一个空对象中，并返回这个新的对象。

还有其他克隆对象的方法，例如使用 spread 语法 clone = {...user}，在后面的章节中我们会讲到。
```
## 2.3 深层克隆
```
到现在为止，我们都假设 user 的所有属性均为原始类型。但属性可以是对其他对象的引用。

例如：

let user = {
  name: "John",
  sizes: {
    height: 182,
    width: 50
  }
};

alert( user.sizes.height ); // 182
现在这样拷贝 clone.sizes = user.sizes 已经不足够了，因为 user.sizes 是个对象，它会以引用形式被拷贝。因此 clone 和 user 会共用一个 sizes：

let user = {
  name: "John",
  sizes: {
    height: 182,
    width: 50
  }
};

let clone = Object.assign({}, user);

alert( user.sizes === clone.sizes ); // true，同一个对象

// user 和 clone 分享同一个 sizes
user.sizes.width++;       // 通过其中一个改变属性值
alert(clone.sizes.width); // 51，能从另外一个获取到变更后的结果
为了解决这个问题，并让 user 和 clone 成为两个真正独立的对象，我们应该使用一个拷贝循环来检查 user[key] 的每个值，如果它是一个对象，那么也复制它的结构。这就是所谓的“深拷贝”。

我们可以使用递归来实现它。或者为了不重复造轮子，采用现有的实现，例如 lodash 库的 _.cloneDeep(obj)。

使用 const 声明的对象也是可以被修改的
通过引用对对象进行存储的一个重要的副作用是声明为 const 的对象 可以 被修改。

例如：

const user = {
  name: "John"
};

user.name = "Pete"; // (*)

alert(user.name); // Pete
看起来 (*) 行的代码会触发一个错误，但实际并没有。user 的值是一个常量，它必须始终引用同一个对象，但该对象的属性可以被自由修改。

换句话说，只有当我们尝试将 user=... 作为一个整体进行赋值时，const user 才会报错。

也就是说，如果我们真的需要创建常量对象属性，也是可以的，但使用的是完全不同的方法。我们将在 属性标志和属性描述符 一章中学习它。
```


# 3. 垃圾回收
```javascript
对于开发者来说，JavaScript 的内存管理是自动的、无形的。我们创建的原始值、对象、函数……这一切都会占用内存。

当我们不再需要某个东西时会发生什么？JavaScript 引擎如何发现它并清理它？
```
## 3.1 可达性（Reachability）
```
JavaScript 中主要的内存管理概念是 可达性。

简而言之，“可达”值是那些以某种方式可访问或可用的值。它们被存储在内存中。

这里列出固有的可达值的基本集合，这些值明显不能被释放。

比方说：

当前执行的函数，它的局部变量和参数。
当前嵌套调用链上的其他函数、它们的局部变量和参数。
全局变量。
（还有一些其他的，内部实现）
这些值被称作 根（roots）。

如果一个值可以从根通过引用或者引用链进行访问，则认为该值是可达的。

比方说，如果全局变量中有一个对象，并且该对象有一个属性引用了另一个对象，则 该 对象被认为是可达的。而且它引用的内容也是可达的。下面是详细的例子。

在 JavaScript 引擎中有一个被称作 垃圾回收器 的东西在后台执行。它监控着所有对象的状态，并删除掉那些已经不可达的。
```
## 3.2 一个简单的例子
```
这里是一个最简单的例子：

// user 具有对这个对象的引用
let user = {
  name: "John"
};

这里的箭头描述了一个对象引用。全局变量 "user" 引用了对象 {name："John"}（为简洁起见，我们称它为 John）。John 的 "name" 属性存储一个原始值，所以它被写在对象内部。

如果 user 的值被重写了，这个引用就没了：

user = null;

现在 John 变成不可达的了。因为没有引用了，就不能访问到它了。垃圾回收器会认为它是垃圾数据并进行回收，然后释放内存。
```
## 3.3 两个引用
```
现在让我们想象下，我们把 user 的引用复制给 admin：

// user 具有对这个对象的引用
let user = {
  name: "John"
};

let admin = user;

现在如果执行刚刚的那个操作：

user = null;
……然后对象仍然可以被通过 admin 这个全局变量访问到，因此它必须被保留在内存中。如果我们又重写了 admin，对象就会被删除。
```
## 3.4 相互关联的对象
```
现在来看一个更复杂的例子。这是个家庭：

function marry(man, woman) {
  woman.husband = man;
  man.wife = woman;

  return {
    father: man,
    mother: woman
  }
}

let family = marry({
  name: "John"
}, {
  name: "Ann"
});
marry 函数通过让两个对象相互引用使它们“结婚”了，并返回了一个包含这两个对象的新对象。

由此产生的内存结构：


到目前为止，所有对象都是可达的。

现在让我们移除两个引用：

delete family.father;
delete family.mother.husband;

仅删除这两个引用中的一个是不够的，因为所有的对象仍然都是可达的。

但是，如果我们把这两个都删除，那么我们可以看到再也没有对 John 的引用了：


对外引用不重要，只有传入引用才可以使对象可达。所以，John 现在是不可达的，并且将被从内存中删除，同时 John 的所有数据也将变得不可达。

经过垃圾回收：
```
## 3.5 无法到达的岛屿
```
几个对象相互引用，但外部没有对其任意对象的引用，这些对象也可能是不可达的，并被从内存中删除。

源对象与上面相同。然后：

family = null;
内存内部状态将变成：


这个例子展示了可达性概念的重要性。

显而易见，John 和 Ann 仍然连着，都有传入的引用。但是，这样还不够。

前面说的 "family" 对象已经不再与根相连，没有了外部对其的引用，所以它变成了一座“孤岛”，并且将被从内存中删除。
```
## 3.6 内部算法
```
垃圾回收的基本算法被称为 “mark-and-sweep”。

定期执行以下“垃圾回收”步骤：

垃圾收集器找到所有的根，并“标记”（记住）它们。
然后它遍历并“标记”来自它们的所有引用。
然后它遍历标记的对象并标记 它们的 引用。所有被遍历到的对象都会被记住，以免将来再次遍历到同一个对象。
……如此操作，直到所有可达的（从根部）引用都被访问到。
没有被标记的对象都会被删除。
例如，使我们的对象有如下的结构：


我们可以清楚地看到右侧有一个“无法到达的岛屿”。现在我们来看看“标记和清除”垃圾收集器如何处理它。

第一步标记所有的根：


然后，我们跟随它们的引用标记它们所引用的对象：


……如果还有引用的话，继续标记：


现在，无法通过这个过程访问到的对象被认为是不可达的，并且会被删除。


我们还可以将这个过程想象成从根溢出一大桶油漆，它流经所有引用并标记所有可到达的对象。然后移除未标记的。

这是垃圾收集工作的概念。JavaScript 引擎做了许多优化，使垃圾回收运行速度更快，并且不会对代码执行引入任何延迟。

一些优化建议：

分代收集（Generational collection）—— 对象被分成两组：“新的”和“旧的”。在典型的代码中，许多对象的生命周期都很短：它们出现、完成它们的工作并很快死去，因此在这种情况下跟踪新对象并将其从内存中清除是有意义的。那些长期存活的对象会变得“老旧”，并且被检查的频次也会降低。
增量收集（Incremental collection）—— 如果有许多对象，并且我们试图一次遍历并标记整个对象集，则可能需要一些时间，并在执行过程中带来明显的延迟。因此，引擎将现有的整个对象集拆分为多个部分，然后将这些部分逐一清除。这样就会有很多小型的垃圾收集，而不是一个大型的。这需要它们之间有额外的标记来追踪变化，但是这样会带来许多微小的延迟而不是一个大的延迟。
闲时收集（Idle-time collection）—— 垃圾收集器只会在 CPU 空闲时尝试运行，以减少可能对代码执行的影响。
还有其他垃圾回收算法的优化和风格。尽管我想在这里描述它们，但我必须打住了，因为不同的引擎会有不同的调整和技巧。而且，更重要的是，随着引擎的发展，情况会发生变化，所以在没有真实需求的时候，“提前”学习这些内容是不值得的。当然，除非你纯粹是出于兴趣。我在下面给你提供了一些相关链接。
```

# 4. 对象方法，"this"
```javascript
通常创建对象来表示真实世界中的实体，如用户和订单等：

let user = {
  name: "John",
  age: 30
};
并且，在现实世界中，用户可以进行 操作：从购物车中挑选某物、登录和注销等。

在 JavaScript 中，行为（action）由属性中的函数来表示。
```
## 4.1 方法示例
```
刚开始，我们来教 user 说 hello：

let user = {
  name: "John",
  age: 30
};

user.sayHi = function() {
  alert("Hello!");
};

user.sayHi(); // Hello!
这里我们使用函数表达式创建了一个函数，并将其指定给对象的 user.sayHi 属性。

随后我们像这样 user.sayHi() 调用它。用户现在可以说话了！

作为对象属性的函数被称为 方法。

所以，在这我们得到了 user 对象的 sayHi 方法。

当然，我们也可以使用预先声明的函数作为方法，就像这样：

let user = {
  // ...
};

// 首先，声明函数
function sayHi() {
  alert("Hello!");
}

// 然后将其作为一个方法添加
user.sayHi = sayHi;

user.sayHi(); // Hello!
面向对象编程
当我们在代码中用对象表示实体时，就是所谓的 面向对象编程，简称为 “OOP”。

OOP 是一门大学问，本身就是一门有趣的科学。怎样选择合适的实体？如何组织它们之间的交互？这就是架构，有很多关于这方面的书，例如 E. Gamma、R. Helm、R. Johnson 和 J. Vissides 所著的《设计模式：可复用面向对象软件的基础》，G. Booch 所著的《面向对象分析与设计》等。

方法简写
在对象字面量中，有一种更短的（声明）方法的语法：

// 这些对象作用一样

user = {
  sayHi: function() {
    alert("Hello");
  }
};

// 方法简写看起来更好，对吧？
let user = {
  sayHi() { // 与 "sayHi: function(){...}" 一样
    alert("Hello");
  }
};
如上所示，我们可以省略 "function"，只写 sayHi()。

说实话，这种表示法还是有些不同。在对象继承方面有一些细微的差别（稍后将会介绍），但目前它们并不重要。在几乎所有的情况下，更短的语法是首选的。
```
## 4.2 方法中的"this"
```
通常，对象方法需要访问对象中存储的信息才能完成其工作。

例如，user.sayHi() 中的代码可能需要用到 user 的 name 属性。

为了访问该对象，方法中可以使用 this 关键字。

this 的值就是在点之前的这个对象，即调用该方法的对象。

举个例子：

let user = {
  name: "John",
  age: 30,

  sayHi() {
    // "this" 指的是“当前的对象”
    alert(this.name);
  }

};

user.sayHi(); // John
在这里 user.sayHi() 执行过程中，this 的值是 user。

技术上讲，也可以在不使用 this 的情况下，通过外部变量名来引用它：

let user = {
  name: "John",
  age: 30,

  sayHi() {
    alert(user.name); // "user" 替代 "this"
  }

};
……但这样的代码是不可靠的。如果我们决定将 user 复制给另一个变量，例如 admin = user，并赋另外的值给 user，那么它将访问到错误的对象。

下面这个示例证实了这一点：

let user = {
  name: "John",
  age: 30,

  sayHi() {
    alert( user.name ); // 导致错误
  }

};


let admin = user;
user = null; // 重写让其更明显

admin.sayHi(); // TypeError: Cannot read property 'name' of null
如果我们在 alert 中以 this.name 替换 user.name，那么代码就会正常运行。
```
## 4.3 "this"不受限制
```
在 JavaScript 中，this 关键字与其他大多数编程语言中的不同。JavaScript 中的 this 可以用于任何函数，即使它不是对象的方法。

下面这样的代码没有语法错误：

function sayHi() {
  alert( this.name );
}
this 的值是在代码运行时计算出来的，它取决于代码上下文。

例如，这里相同的函数被分配给两个不同的对象，在调用中有着不同的 “this” 值：

let user = { name: "John" };
let admin = { name: "Admin" };

function sayHi() {
  alert( this.name );
}

// 在两个对象中使用相同的函数
user.f = sayHi;
admin.f = sayHi;

// 这两个调用有不同的 this 值
// 函数内部的 "this" 是“点符号前面”的那个对象
user.f(); // John（this == user）
admin.f(); // Admin（this == admin）

admin['f'](); // Admin（使用点符号或方括号语法来访问这个方法，都没有关系。）
这个规则很简单：如果 obj.f() 被调用了，则 this 在 f 函数调用期间是 obj。所以在上面的例子中 this 先是 user，之后是 admin。

在没有对象的情况下调用：this == undefined
我们甚至可以在没有对象的情况下调用函数：

function sayHi() {
  alert(this);
}

sayHi(); // undefined
在这种情况下，严格模式下的 this 值为 undefined。如果我们尝试访问 this.name，将会报错。

在非严格模式的情况下，this 将会是 全局对象（浏览器中的 window，我们稍后会在 全局对象 一章中学习它）。这是一个历史行为，"use strict" 已经将其修复了。

通常这种调用是程序出错了。如果在一个函数内部有 this，那么通常意味着它是在对象上下文环境中被调用的。

解除 this 绑定的后果
如果你经常使用其他的编程语言，那么你可能已经习惯了“绑定 this”的概念，即在对象中定义的方法总是有指向该对象的 this。

在 JavaScript 中，this 是“自由”的，它的值是在调用时计算出来的，它的值并不取决于方法声明的位置，而是取决于在“点符号前”的是什么对象。

在运行时对 this 求值的这个概念既有优点也有缺点。一方面，函数可以被重用于不同的对象。另一方面，更大的灵活性造成了更大的出错的可能。

这里我们的立场并不是要评判编程语言的这个设计是好是坏。而是要了解怎样使用它，如何趋利避害。
```
## 4.4 箭头函数没有自己的"this"
```
箭头函数有些特别：它们没有自己的 this。如果我们在这样的函数中引用 this，this 值取决于外部“正常的”函数。

举个例子，这里的 arrow() 使用的 this 来自于外部的 user.sayHi() 方法：

let user = {
  firstName: "Ilya",
  sayHi() {
    let arrow = () => alert(this.firstName);
    arrow();
  }
};

user.sayHi(); // Ilya
这是箭头函数的一个特性，当我们并不想要一个独立的 this，反而想从外部上下文中获取时，它很有用。在后面的 深入理解箭头函数 一章中，我们将深入介绍箭头函数。
```


# 5. 构造器和操作符 "new"
```javascript
常规的 {...} 语法允许创建一个对象。但是我们经常需要创建很多类似的对象，例如多个用户或菜单项等。

这可以使用构造函数和 "new" 操作符来实现。
```
## 5.1 构造函数
```
构造函数在技术上是常规函数。不过有两个约定：

它们的命名以大写字母开头。
它们只能由 "new" 操作符来执行。
例如：

function User(name) {
  this.name = name;
  this.isAdmin = false;
}

let user = new User("Jack");

alert(user.name); // Jack
alert(user.isAdmin); // false
当一个函数被使用 new 操作符执行时，它按照以下步骤：

一个新的空对象被创建并分配给 this。
函数体执行。通常它会修改 this，为其添加新的属性。
返回 this 的值。
换句话说，new User(...) 做的就是类似的事情：

function User(name) {
  // this = {};（隐式创建）

  // 添加属性到 this
  this.name = name;
  this.isAdmin = false;

  // return this;（隐式返回）
}
所以 new User("Jack") 的结果和以下操作的结果相同：

let user = {
  name: "Jack",
  isAdmin: false
};
现在，如果我们想创建其他用户，我们可以调用 new User("Ann")，new User("Alice") 等。比每次都使用字面量创建要短得多，而且更易于阅读。

这是构造器的主要目的 —— 实现可重用的对象创建代码。

让我们再强调一遍 —— 从技术上讲，任何函数（除了箭头函数，它没有自己的 this）都可以用作构造器。即可以通过 new 来运行，它会执行上面的算法。“首字母大写”是一个共同的约定，以明确表示一个函数将被使用 new 来运行。

new function() { … }
如果我们有许多行用于创建单个复杂对象的代码，我们可以将它们封装在一个立即调用的构造函数中，像这样：

// 创建一个函数并立即使用 new 调用它
let user = new function() {
  this.name = "John";
  this.isAdmin = false;

  // ……用于用户创建的其他代码
  // 也许是复杂的逻辑和语句
  // 局部变量等
};
这个构造函数不能被再次调用，因为它不保存在任何地方，只是被创建和调用。因此，这个技巧旨在封装构建单个对象的代码，而无需将来重用。
```
## 5.2 构造器模式测试：new.target
```
进阶内容
本节涉及的语法内容很少使用，除非你想了解所有内容，否则你可以直接跳过该语法。

在一个函数内部，我们可以使用 new.target 属性来检查它是否被使用 new 进行调用了。

对于常规调用，它为 undefined，对于使用 new 的调用，则等于该函数：

function User() {
  alert(new.target);
}

// 不带 "new"：
User(); // undefined

// 带 "new"：
new User(); // function User { ... }
它可以被用在函数内部，来判断该函数是被通过 new 调用的“构造器模式”，还是没被通过 new 调用的“常规模式”。

我们也可以让 new 调用和常规调用做相同的工作，像这样：

function User(name) {
  if (!new.target) { // 如果你没有通过 new 运行我
    return new User(name); // ……我会给你添加 new
  }

  this.name = name;
}

let john = User("John"); // 将调用重定向到新用户
alert(john.name); // John
这种方法有时被用在库中以使语法更加灵活。这样人们在调用函数时，无论是否使用了 new，程序都能工作。

不过，到处都使用它并不是一件好事，因为省略了 new 使得很难观察到代码中正在发生什么。而通过 new 我们都可以知道这创建了一个新对象。
```
## 5.3 构造器的return
```
通常，构造器没有 return 语句。它们的任务是将所有必要的东西写入 this，并自动转换为结果。

但是，如果这有一个 return 语句，那么规则就简单了：

如果 return 返回的是一个对象，则返回这个对象，而不是 this。
如果 return 返回的是一个原始类型，则忽略。
换句话说，带有对象的 return 返回该对象，在所有其他情况下返回 this。

例如，这里 return 通过返回一个对象覆盖 this：

function BigUser() {

  this.name = "John";

  return { name: "Godzilla" };  // <-- 返回这个对象
}

alert( new BigUser().name );  // Godzilla，得到了那个对象
这里有一个 return 为空的例子（或者我们可以在它之后放置一个原始类型，没有什么影响）：

function SmallUser() {

  this.name = "John";

  return; // <-- 返回 this
}

alert( new SmallUser().name );  // John
通常构造器没有 return 语句。这里我们主要为了完整性而提及返回对象的特殊行为。

省略括号
顺便说一下，如果没有参数，我们可以省略 new 后的括号：

let user = new User; // <-- 没有参数
// 等同于
let user = new User();
这里省略括号不被认为是一种“好风格”，但是规范允许使用该语法。
```
## 5.4 构造器中的方法
```
使用构造函数来创建对象会带来很大的灵活性。构造函数可能有一些参数，这些参数定义了如何构造对象以及要放入什么。

当然，我们不仅可以将属性添加到 this 中，还可以添加方法。

例如，下面的 new User(name) 用给定的 name 和方法 sayHi 创建了一个对象：

function User(name) {
  this.name = name;

  this.sayHi = function() {
    alert( "My name is: " + this.name );
  };
}

let john = new User("John");

john.sayHi(); // My name is: John

/*
john = {
   name: "John",
   sayHi: function() { ... }
}
*/
类 是用于创建复杂对象的一个更高级的语法，我们稍后会讲到。
```

# 6. 可选链 "?."
```javascript
最近新增的特性
这是一个最近添加到 JavaScript 的特性。 旧式浏览器可能需要 polyfills.
可选链 ?. 是一种访问嵌套对象属性的安全的方式。即使中间的属性不存在，也不会出现错误。
```
## 6.1 "不存在的属性"的问题
```
如果你才刚开始读此教程并学习 JavaScript，那可能还没接触到这个问题，但它却相当常见。

举个例子，假设我们有很多个 user 对象，其中存储了我们的用户数据。

我们大多数用户的地址都存储在 user.address 中，街道地址存储在 user.address.street 中，但有些用户没有提供这些信息。

在这种情况下，当我们尝试获取 user.address.street，而该用户恰好没提供地址信息，我们则会收到一个错误：

let user = {}; // 一个没有 "address" 属性的 user 对象

alert(user.address.street); // Error!
这是预期的结果。JavaScript 的工作原理就是这样的。因为 user.address 为 undefined，尝试读取 user.address.street 会失败，并收到一个错误。

但是在很多实际场景中，我们更希望得到的是 undefined（表示没有 street 属性）而不是一个错误。

……还有另一个例子。在 Web 开发中，我们可以使用特殊的方法调用（例如 document.querySelector('.elem')）以对象的形式获取一个网页元素，如果没有这种对象，则返回 null。

// 如果 document.querySelector('.elem') 的结果为 null，则这里不存在这个元素
let html = document.querySelector('.elem').innerHTML; // 如果 document.querySelector('.elem') 的结果为 null，则会出现错误
同样，如果该元素不存在，则访问 null 的 .innerHTML 属性时会报错。在某些情况下，当元素的缺失是没问题的时候，我们希望避免出现这种错误，而是接受 html = null 作为结果。

我们如何实现这一点呢？

可能最先想到的方案是在访问该值的属性之前，使用 if 或条件运算符 ? 对该值进行检查，像这样：

let user = {};

alert(user.address ? user.address.street : undefined);
这样可以，这里就不会出现错误了……但是不够优雅。就像你所看到的，"user.address" 在代码中出现了两次。

我们看一个以相同方式获取 document.querySelector 的例子：

let html = document.querySelector('.elem') ? document.querySelector('.elem').innerHTML : null;
我们可以看到用于进行元素搜索的 document.querySelector('.elem') 在这里实际上被调用了两次。这样不优雅。

对于嵌套层次更深的属性，代码会变得更丑，因为需要更多的重复。

例如，让我们以相同的方式尝试获取 user.address.street.name。

let user = {}; // user 没有 address 属性

alert(user.address ? user.address.street ? user.address.street.name : null : null);
这样就太扯淡了，并且这可能导致写出来的代码很难让别人理解。

这里有一种更好的实现方式，就是使用 && 运算符：

let user = {}; // user 没有 address 属性

alert( user.address && user.address.street && user.address.street.name ); // undefined（不报错）
依次对整条路径上的属性使用与运算进行判断，以确保所有节点是存在的（如果不存在，则停止计算），但仍然不够优雅。

就像你所看到的，在代码中我们仍然重复写了好几遍对象属性名。例如在上面的代码中，user.address 被重复写了三遍。

这就是为什么可选链 ?. 被加入到了 JavaScript 这门编程语言中。那就是彻底地解决以上所有问题！
```
## 6.2 可选链
```
如果可选链 ?. 前面的值为 undefined 或者 null，它会停止运算并返回 undefined。

为了简明起见，在本文接下来的内容中，我们会说如果一个属性既不是 null 也不是 undefined，那么它就“存在”。

换句话说，例如 value?.prop：

如果 value 存在，则结果与 value.prop 相同，
否则（当 value 为 undefined/null 时）则返回 undefined。
下面这是一种使用 ?. 安全地访问 user.address.street 的方式：

let user = {}; // user 没有 address 属性

alert( user?.address?.street ); // undefined（不报错）
代码简洁明了，也不用重复写好几遍属性名。

这里是一个结合 document.querySelector 使用的示例：

let html = document.querySelector('.elem')?.innerHTML; // 如果没有符合的元素，则为 undefined
即使 对象 user 不存在，使用 user?.address 来读取地址也没问题：

let user = null;

alert( user?.address ); // undefined
alert( user?.address.street ); // undefined
请注意：?. 语法使其前面的值成为可选值，但不会对其后面的起作用。

例如，在 user?.address.street.name 中，?. 允许 user 为 null/undefined（在这种情况下会返回 undefined）也不会报错，但这仅对于 user。更深层次的属性是通过常规方式访问的。如果我们希望它们中的一些也是可选的，那么我们需要使用更多的 ?. 来替换 .。

不要过度使用可选链
我们应该只将 ?. 使用在一些东西可以不存在的地方。

例如，如果根据我们的代码逻辑，user 对象必须存在，但 address 是可选的，那么我们应该这样写 user.address?.street，而不是这样 user?.address?.street。

那么，如果 user 恰巧为 undefined，我们会看到一个编程错误并修复它。否则，如果我们滥用 ?.，会导致代码中的错误在不应该被消除的地方消除了，这会导致调试更加困难。

?. 前的变量必须已声明
如果未声明变量 user，那么 user?.anything 会触发一个错误：

// ReferenceError: user is not defined
user?.address;
?. 前的变量必须已声明（例如 let/const/var user 或作为一个函数参数）。可选链仅适用于已声明的变量。
```
## 6.3 短路效应
```
正如前面所说的，如果 ?. 左边部分不存在，就会立即停止运算（“短路效应”）。

因此，如果在 ?. 的右侧有任何进一步的函数调用或操作，它们均不会执行。

例如：

let user = null;
let x = 0;

user?.sayHi(x++); // 没有 "user"，因此代码执行没有到达 sayHi 调用和 x++

alert(x); // 0，值没有增加
```
## 6.4 其他变体：?.(), ?.[]
```
可选链 ?. 不是一个运算符，而是一个特殊的语法结构。它还可以与函数和方括号一起使用。

例如，将 ?.() 用于调用一个可能不存在的函数。

在下面这段代码中，有些用户具有 admin 方法，而有些没有：

let userAdmin = {
  admin() {
    alert("I am admin");
  }
};

let userGuest = {};

userAdmin.admin?.(); // I am admin

userGuest.admin?.(); // 啥都没发生（没有这样的方法）
在这两行代码中，我们首先使用点符号（userAdmin.admin）来获取 admin 属性，因为我们假定对象 userAdmin 存在，因此可以安全地读取它。

然后 ?.() 会检查它左边的部分：如果 admin 函数存在，那么就调用运行它（对于 userAdmin）。否则（对于 userGuest）运算停止，没有报错。

如果我们想使用方括号 [] 而不是点符号 . 来访问属性，语法 ?.[] 也可以使用。跟前面的例子类似，它允许从一个可能不存在的对象上安全地读取属性。

let key = "firstName";

let user1 = {
  firstName: "John"
};

let user2 = null;

alert( user1?.[key] ); // John
alert( user2?.[key] ); // undefined
此外，我们还可以将 ?. 跟 delete 一起使用：

delete user?.name; // 如果 user 存在，则删除 user.name
我们可以使用 ?. 来安全地读取或删除，但不能写入
可选链 ?. 不能用在赋值语句的左侧。

例如：

let user = null;

user?.name = "John"; // Error，不起作用
// 因为它在计算的是：undefined = "John"
```

# 7. symbol 类型
```javascript
根据规范，只有两种原始类型可以用作对象属性键：

字符串类型
symbol 类型
否则，如果使用另一种类型，例如数字，它会被自动转换为字符串。所以 obj[1] 与 obj["1"] 相同，而 obj[true] 与 obj["true"] 相同。

到目前为止，我们一直只使用字符串。

现在我们来看看 symbol 能给我们带来什么。
```
## 7.1 symbol
```
“symbol” 值表示唯一的标识符。

可以使用 Symbol() 来创建这种类型的值：

let id = Symbol();
创建时，我们可以给 symbol 一个描述（也称为 symbol 名），这在代码调试时非常有用：

// id 是描述为 "id" 的 symbol
let id = Symbol("id");
symbol 保证是唯一的。即使我们创建了许多具有相同描述的 symbol，它们的值也是不同。描述只是一个标签，不影响任何东西。

例如，这里有两个描述相同的 symbol —— 它们不相等：

let id1 = Symbol("id");
let id2 = Symbol("id");

alert(id1 == id2); // false
如果你熟悉 Ruby 或者其他有 “symbol” 的语言 —— 别被误导。JavaScript 的 symbol 是不同的。

所以，总而言之，symbol 是带有可选描述的“原始唯一值”。让我们看看我们可以在哪里使用它们。

symbol 不会被自动转换为字符串
JavaScript 中的大多数值都支持字符串的隐式转换。例如，我们可以 alert 任何值，都可以生效。symbol 比较特殊，它不会被自动转换。

例如，这个 alert 将会提示出错：

let id = Symbol("id");
alert(id); // 类型错误：无法将 symbol 值转换为字符串。
这是一种防止混乱的“语言保护”，因为字符串和 symbol 有本质上的不同，不应该意外地将它们转换成另一个。

如果我们真的想显示一个 symbol，我们需要在它上面调用 .toString()，如下所示：

let id = Symbol("id");
alert(id.toString()); // Symbol(id)，现在它有效了
或者获取 symbol.description 属性，只显示描述（description）：

let id = Symbol("id");
alert(id.description); // id
```
## 7.2 "隐藏"属性
```
symbol 允许我们创建对象的“隐藏”属性，代码的任何其他部分都不能意外访问或重写这些属性。

例如，如果我们使用的是属于第三方代码的 user 对象，我们想要给它们添加一些标识符。

我们可以给它们使用 symbol 键：

let user = { // 属于另一个代码
  name: "John"
};

let id = Symbol("id");

user[id] = 1;

alert( user[id] ); // 我们可以使用 symbol 作为键来访问数据
使用 Symbol("id") 作为键，比起用字符串 "id" 来有什么好处呢？

由于 user 对象属于另一个代码库，所以向它们添加字段是不安全的，因为我们可能会影响代码库中的其他预定义行为。但 symbol 属性不会被意外访问到。第三方代码不会知道新定义的 symbol，因此将 symbol 添加到 user 对象是安全的。

另外，假设另一个脚本希望在 user 中有自己的标识符，以实现自己的目的。

那么，该脚本可以创建自己的 Symbol("id")，像这样：

// ...
let id = Symbol("id");

user[id] = "Their id value";
我们的标识符和它们的标识符之间不会有冲突，因为 symbol 总是不同的，即使它们有相同的名字。

……但如果我们出于同样的目的，使用字符串 "id" 而不是用 symbol，那么 就会 出现冲突：

let user = { name: "John" };

// 我们的脚本使用了 "id" 属性。
user.id = "Our id value";

// ……另一个脚本也想将 "id" 用于它的目的……

user.id = "Their id value"
// 砰！无意中被另一个脚本重写了 id！
对象字面量中的 symbol
如果我们要在对象字面量 {...} 中使用 symbol，则需要使用方括号把它括起来。

就像这样：

let id = Symbol("id");

let user = {
  name: "John",
  [id]: 123 // 而不是 "id"：123
};
这是因为我们需要变量 id 的值作为键，而不是字符串 “id”。

symbol 在 for…in 中会被跳过
symbol 属性不参与 for..in 循环。

例如：

let id = Symbol("id");
let user = {
  name: "John",
  age: 30,
  [id]: 123
};

for (let key in user) alert(key); // name, age（没有 symbol）

// 使用 symbol 任务直接访问
alert("Direct: " + user[id]); // Direct: 123
Object.keys(user) 也会忽略它们。这是一般“隐藏符号属性”原则的一部分。如果另一个脚本或库遍历我们的对象，它不会意外地访问到符号属性。

相反，Object.assign 会同时复制字符串和 symbol 属性：

let id = Symbol("id");
let user = {
  [id]: 123
};

let clone = Object.assign({}, user);

alert( clone[id] ); // 123
这里并不矛盾，就是这样设计的。这里的想法是当我们克隆或者合并一个 object 时，通常希望 所有 属性被复制（包括像 id 这样的 symbol）。
```
## 7.3 全局symbol
```
正如我们所看到的，通常所有的 symbol 都是不同的，即使它们有相同的名字。但有时我们想要名字相同的 symbol 具有相同的实体。例如，应用程序的不同部分想要访问的 symbol "id" 指的是完全相同的属性。

为了实现这一点，这里有一个 全局 symbol 注册表。我们可以在其中创建 symbol 并在稍后访问它们，它可以确保每次访问相同名字的 symbol 时，返回的都是相同的 symbol。

要从注册表中读取（不存在则创建）symbol，请使用 Symbol.for(key)。

该调用会检查全局注册表，如果有一个描述为 key 的 symbol，则返回该 symbol，否则将创建一个新 symbol（Symbol(key)），并通过给定的 key 将其存储在注册表中。

例如：

// 从全局注册表中读取
let id = Symbol.for("id"); // 如果该 symbol 不存在，则创建它

// 再次读取（可能是在代码中的另一个位置）
let idAgain = Symbol.for("id");

// 相同的 symbol
alert( id === idAgain ); // true
注册表内的 symbol 被称为 全局 symbol。如果我们想要一个应用程序范围内的 symbol，可以在代码中随处访问 —— 这就是它们的用途。

这听起来像 Ruby
在一些编程语言中，例如 Ruby，每个名字都有一个 symbol。

正如我们所看到的，在 JavaScript 中，全局 symbol 也是这样的。

Symbol.keyFor
我们已经看到，对于全局 symbol，Symbol.for(key) 按名字返回一个 symbol。相反，通过全局 symbol 返回一个名字，我们可以使用 Symbol.keyFor(sym)：

例如：

// 通过 name 获取 symbol
let sym = Symbol.for("name");
let sym2 = Symbol.for("id");

// 通过 symbol 获取 name
alert( Symbol.keyFor(sym) ); // name
alert( Symbol.keyFor(sym2) ); // id
Symbol.keyFor 内部使用全局 symbol 注册表来查找 symbol 的键。所以它不适用于非全局 symbol。如果 symbol 不是全局的，它将无法找到它并返回 undefined。

但是，所有 symbol 都具有 description 属性。

例如：

let globalSymbol = Symbol.for("name");
let localSymbol = Symbol("name");

alert( Symbol.keyFor(globalSymbol) ); // name，全局 symbol
alert( Symbol.keyFor(localSymbol) ); // undefined，非全局

alert( localSymbol.description ); // name
```
## 7.4 系统symbol
```
JavaScript 内部有很多“系统” symbol，我们可以使用它们来微调对象的各个方面。

它们都被列在了 众所周知的 symbol 表的规范中：

Symbol.hasInstance
Symbol.isConcatSpreadable
Symbol.iterator
Symbol.toPrimitive
……等等。
例如，Symbol.toPrimitive 允许我们将对象描述为原始值转换。我们很快就会看到它的使用。

当我们研究相应的语言特征时，我们对其他的 symbol 也会慢慢熟悉起来。

```


# 8. 对象 —— 原始值转换
```javascript
当对象相加 obj1 + obj2，相减 obj1 - obj2，或者使用 alert(obj) 打印时会发生什么？

JavaScript 不允许自定义运算符对对象的处理方式。与其他一些编程语言（Ruby，C++）不同，我们无法实现特殊的对象处理方法来处理加法（或其他运算）。

在此类运算的情况下，对象会被自动转换为原始值，然后对这些原始值进行运算，并得到运算结果（也是一个原始值）。

这是一个重要的限制：因为 obj1 + obj2（或者其他数学运算）的结果不能是另一个对象！

例如，我们无法使用对象来表示向量或矩阵，把它们相加并期望得到一个“总和”向量作为结果。这样的想法是行不通的。

因此，由于我们从技术上无法实现此类运算，所以在实际项目中不存在对对象的数学运算。如果你发现有，除了极少数例外，通常是写错了。

本文将介绍对象是如何转换为原始值的，以及如何对其进行自定义。

我们有两个目的：

让我们在遇到类似的对对象进行数学运算的编程错误时，能够更加理解到底发生了什么。
也有例外，这些操作也可以是可行的。例如日期相减或比较（Date 对象）。我们稍后会遇到它们。
```
## 8.1 转换规则
```
在 类型转换 一章中，我们已经看到了数字、字符串和布尔转换的规则。但是我们没有讲对象的转换规则。现在我们已经掌握了方法（method）和 symbol 的相关知识，可以开始学习对象原始值转换了。

没有转换为布尔值。所有的对象在布尔上下文（context）中均为 true，就这么简单。只有字符串和数字转换。
数字转换发生在对象相减或应用数学函数时。例如，Date 对象（将在 日期和时间 一章中介绍）可以相减，date1 - date2 的结果是两个日期之间的差值。
至于字符串转换 —— 通常发生在我们像 alert(obj) 这样输出一个对象和类似的上下文中。
我们可以使用特殊的对象方法，自己实现字符串和数字的转换。

现在让我们一起探究技术细节，因为这是深入讨论该主题的唯一方式。
```
## 8.2 hint
```
JavaScript 是如何决定应用哪种转换的？

类型转换在各种情况下有三种变体。它们被称为 “hint”，在 规范 所述：

"string"
对象到字符串的转换，当我们对期望一个字符串的对象执行操作时，如 “alert”：

// 输出
alert(obj);

// 将对象作为属性键
anotherObj[obj] = 123;
"number"
对象到数字的转换，例如当我们进行数学运算时：

// 显式转换
let num = Number(obj);

// 数学运算（除了二元加法）
let n = +obj; // 一元加法
let delta = date1 - date2;

// 小于/大于的比较
let greater = user1 > user2;
大多数内建的数学函数也包括这种转换。

"default"
在少数情况下发生，当运算符“不确定”期望值的类型时。

例如，二元加法 + 可用于字符串（连接），也可以用于数字（相加）。因此，当二元加法得到对象类型的参数时，它将依据 "default" hint 来对其进行转换。

此外，如果对象被用于与字符串、数字或 symbol 进行 == 比较，这时到底应该进行哪种转换也不是很明确，因此使用 "default" hint。

// 二元加法使用默认 hint
let total = obj1 + obj2;

// obj == number 使用默认 hint
if (user == 1) { ... };
像 < 和 > 这样的小于/大于比较运算符，也可以同时用于字符串和数字。不过，它们使用 “number” hint，而不是 “default”。这是历史原因。

上面这些规则看起来比较复杂，但在实践中其实挺简单的。

除了一种情况（Date 对象，我们稍后会讲到）之外，所有内建对象都以和 "number" 相同的方式实现 "default" 转换。我们也可以这样做。

尽管如此，了解上述的 3 个 hint 还是很重要的，很快你就会明白为什么这样说。

为了进行转换，JavaScript 尝试查找并调用三个对象方法：

调用 obj[Symbol.toPrimitive](hint) —— 带有 symbol 键 Symbol.toPrimitive（系统 symbol）的方法，如果这个方法存在的话，
否则，如果 hint 是 "string" —— 尝试调用 obj.toString() 或 obj.valueOf()，无论哪个存在。
否则，如果 hint 是 "number" 或 "default" —— 尝试调用 obj.valueOf() 或 obj.toString()，无论哪个存在。
```
## 8.3 Symbol.toPrimitive
```
我们从第一个方法开始。有一个名为 Symbol.toPrimitive 的内建 symbol，它被用来给转换方法命名，像这样：

obj[Symbol.toPrimitive] = function(hint) {
  // 这里是将此对象转换为原始值的代码
  // 它必须返回一个原始值
  // hint = "string"、"number" 或 "default" 中的一个
}
如果 Symbol.toPrimitive 方法存在，则它会被用于所有 hint，无需更多其他方法。

例如，这里 user 对象实现了它：

let user = {
  name: "John",
  money: 1000,

  [Symbol.toPrimitive](hint) {
    alert(`hint: ${hint}`);
    return hint == "string" ? `{name: "${this.name}"}` : this.money;
  }
};

// 转换演示：
alert(user); // hint: string -> {name: "John"}
alert(+user); // hint: number -> 1000
alert(user + 500); // hint: default -> 1500
从代码中我们可以看到，根据转换的不同，user 变成一个自描述字符串或者一个金额。user[Symbol.toPrimitive] 方法处理了所有的转换情况。
```
## 8.4 toString/valueOf
```
如果没有 Symbol.toPrimitive，那么 JavaScript 将尝试寻找 toString 和 valueOf 方法：

对于 "string" hint：调用 toString 方法，如果它不存在，则调用 valueOf 方法（因此，对于字符串转换，优先调用 toString）。
对于其他 hint：调用 valueOf 方法，如果它不存在，则调用 toString 方法（因此，对于数学运算，优先调用 valueOf 方法）。
toString 和 valueOf 方法很早己有了。它们不是 symbol（那时候还没有 symbol 这个概念），而是“常规的”字符串命名的方法。它们提供了一种可选的“老派”的实现转换的方法。

这些方法必须返回一个原始值。如果 toString 或 valueOf 返回了一个对象，那么返回值会被忽略（和这里没有方法的时候相同）。

默认情况下，普通对象具有 toString 和 valueOf 方法：

toString 方法返回一个字符串 "[object Object]"。
valueOf 方法返回对象自身。
下面是一个示例：

let user = {name: "John"};

alert(user); // [object Object]
alert(user.valueOf() === user); // true
所以，如果我们尝试将一个对象当做字符串来使用，例如在 alert 中，那么在默认情况下我们会看到 [object Object]。

这里提到的默认的 valueOf 只是为了完整起见，以避免混淆。正如你看到的，它返回对象本身，因此被忽略。别问我为什么，这是历史原因。所以我们可以假设它根本就不存在。

让我们实现一下这些方法来自定义转换。

例如，这里的 user 执行和前面提到的那个 user 一样的操作，使用 toString 和 valueOf 的组合（而不是 Symbol.toPrimitive）：

let user = {
  name: "John",
  money: 1000,

  // 对于 hint="string"
  toString() {
    return `{name: "${this.name}"}`;
  },

  // 对于 hint="number" 或 "default"
  valueOf() {
    return this.money;
  }

};

alert(user); // toString -> {name: "John"}
alert(+user); // valueOf -> 1000
alert(user + 500); // valueOf -> 1500
我们可以看到，执行的动作和前面使用 Symbol.toPrimitive 的那个例子相同。

通常我们希望有一个“全能”的地方来处理所有原始转换。在这种情况下，我们可以只实现 toString，就像这样：

let user = {
  name: "John",

  toString() {
    return this.name;
  }
};

alert(user); // toString -> John
alert(user + 500); // toString -> John500
如果没有 Symbol.toPrimitive 和 valueOf，toString 将处理所有原始转换。

转换可以返回任何原始类型
关于所有原始转换方法，有一个重要的点需要知道，就是它们不一定会返回 “hint” 的原始值。

没有限制 toString() 是否返回字符串，或 Symbol.toPrimitive 方法是否为 "number" hint 返回数字。

唯一强制性的事情是：这些方法必须返回一个原始值，而不是对象。

历史原因
由于历史原因，如果 toString 或 valueOf 返回一个对象，则不会出现 error，但是这种值会被忽略（就像这种方法根本不存在）。这是因为在 JavaScript 语言发展初期，没有很好的 “error” 的概念。

相反，Symbol.toPrimitive 更严格，它 必须 返回一个原始值，否则就会出现 error。
```
## 8.5 进一步的转换
```
我们已经知道，许多运算符和函数执行类型转换，例如乘法 * 将操作数转换为数字。

如果我们将对象作为参数传递，则会出现两个运算阶段：

对象被转换为原始值（通过前面我们描述的规则）。
如果还需要进一步计算，则生成的原始值会被进一步转换。
例如：

let obj = {
  // toString 在没有其他方法的情况下处理所有转换
  toString() {
    return "2";
  }
};

alert(obj * 2); // 4，对象被转换为原始值字符串 "2"，之后它被乘法转换为数字 2。
乘法 obj * 2 首先将对象转换为原始值（字符串 “2”）。
之后 "2" * 2 变为 2 * 2（字符串被转换为数字）。
二元加法在同样的情况下会将其连接成字符串，因为它更愿意接受字符串：

let obj = {
  toString() {
    return "2";
  }
};

alert(obj + 2); // 22（"2" + 2）被转换为原始值字符串 => 级联
```