# 1.页面生命周期：DOMContentLoaded, load, beforeunload, unload
```
HTML 页面的生命周期包含三个重要事件：

DOMContentLoaded —— 浏览器已完全加载 HTML，并构建了 DOM 树，但像 <img> 和样式表之类的外部资源可能尚未加载完成。
load —— 浏览器不仅加载完成了 HTML，还加载完成了所有外部资源：图片，样式等。
beforeunload/unload —— 当用户正在离开页面时。
每个事件都是有用的：

DOMContentLoaded 事件 —— DOM 已经就绪，因此处理程序可以查找 DOM 节点，并初始化接口。
load 事件 —— 外部资源已加载完成，样式已被应用，图片大小也已知了。
beforeunload 事件 —— 用户正在离开：我们可以检查用户是否保存了更改，并询问他是否真的要离开。
unload 事件 —— 用户几乎已经离开了，但是我们仍然可以启动一些操作，例如发送统计数据。
我们探索一下这些事件的细节。
```
## 1.1DOMContentLoaded
```javascript
DOMContentLoaded 事件发生在 document 对象上。

我们必须使用 addEventListener 来捕获它：

document.addEventListener("DOMContentLoaded", ready);
// 不是 "document.onDOMContentLoaded = ..."
例如：

<script>
  function ready() {
    alert('DOM is ready');

    // 图片目前尚未加载完成（除非已经被缓存），所以图片的大小为 0x0
    alert(`Image size: ${img.offsetWidth}x${img.offsetHeight}`);
  }

  document.addEventListener("DOMContentLoaded", ready);
</script>

<img id="img" src="https://en.js.cx/clipart/train.gif?speed=1&cache=0">
在示例中，DOMContentLoaded 处理程序在文档加载完成后触发，所以它可以查看所有元素，包括它下面的 <img> 元素。

但是，它不会等待图片加载。因此，alert 显示其大小为零。

乍一看，DOMContentLoaded 事件非常简单。DOM 树准备就绪 —— 这是它的触发条件。它并没有什么特别之处。
DOMContentLoaded 和脚本
当浏览器处理一个 HTML 文档，并在文档中遇到 <script> 标签时，就会在继续构建 DOM 之前运行它。这是一种防范措施，因为脚本可能想要修改 DOM，甚至对其执行 document.write 操作，所以 DOMContentLoaded 必须等待脚本执行结束。

因此，DOMContentLoaded 肯定在下面的这些脚本执行结束之后发生：

<script>
  document.addEventListener("DOMContentLoaded", () => {
    alert("DOM ready!");
  });
</script>

<script src="https://cdnjs.cloudflare.com/ajax/libs/lodash.js/4.3.0/lodash.js"></script>

<script>
  alert("Library loaded, inline script executed");
</script>
在上面这个例子中，我们首先会看到 “Library loaded…”，然后才会看到 “DOM ready!”（所有脚本都已经执行结束）。

不会阻塞 DOMContentLoaded 的脚本
此规则有两个例外：

具有 async 特性（attribute）的脚本不会阻塞 DOMContentLoaded，稍后 我们会讲到。
使用 document.createElement('script') 动态生成并添加到网页的脚本也不会阻塞 DOMContentLoaded。
DOMContentLoaded 和样式
外部样式表不会影响 DOM，因此 DOMContentLoaded 不会等待它们。

但这里有一个陷阱。如果在样式后面有一个脚本，那么该脚本必须等待样式表加载完成：

<link type="text/css" rel="stylesheet" href="style.css">
<script>
  // 在样式表加载完成之前，脚本都不会执行
  alert(getComputedStyle(document.body).marginTop);
</script>
原因是，脚本可能想要获取元素的坐标和其他与样式相关的属性，如上例所示。因此，它必须等待样式加载完成。

当 DOMContentLoaded 等待脚本时，它现在也在等待脚本前面的样式。

浏览器内建的自动填充
Firefox，Chrome 和 Opera 都会在 DOMContentLoaded 中自动填充表单。

例如，如果页面有一个带有登录名和密码的表单，并且浏览器记住了这些值，那么在 DOMContentLoaded 上，浏览器会尝试自动填充它们（如果得到了用户允许）。

因此，如果 DOMContentLoaded 被需要加载很长时间的脚本延迟触发，那么自动填充也会等待。你可能在某些网站上看到过（如果你使用浏览器自动填充）—— 登录名/密码字段不会立即自动填充，而是在页面被完全加载前会延迟填充。这实际上是 DOMContentLoaded 事件之前的延迟。
```
## 1.2window.onload
```javascript
当整个页面，包括样式、图片和其他资源被加载完成时，会触发 window 对象上的 load 事件。可以通过 onload 属性获取此事件。

下面的这个示例正确显示了图片大小，因为 window.onload 会等待所有图片加载完毕：

<script>
  window.onload = function() { // 也可以用 window.addEventListener('load', (event) => {
    alert('Page loaded');

    // 此时图片已经加载完成
    alert(`Image size: ${img.offsetWidth}x${img.offsetHeight}`);
  };
</script>

<img id="img" src="https://en.js.cx/clipart/train.gif?speed=1&cache=0">
window.onunload
当访问者离开页面时，window 对象上的 unload 事件就会被触发。我们可以在那里做一些不涉及延迟的操作，例如关闭相关的弹出窗口。

有一个值得注意的特殊情况是发送分析数据。

假设我们收集有关页面使用情况的数据：鼠标点击，滚动，被查看的页面区域等。

自然地，当用户要离开的时候，我们希望通过 unload 事件将数据保存到我们的服务器上。

有一个特殊的 navigator.sendBeacon(url, data) 方法可以满足这种需求，详见规范 https://w3c.github.io/beacon/。

它在后台发送数据，转换到另外一个页面不会有延迟：浏览器离开页面，但仍然在执行 sendBeacon。

使用方式如下：

let analyticsData = { /* 带有收集的数据的对象 */ };

window.addEventListener("unload", function() {
  navigator.sendBeacon("/analytics", JSON.stringify(analyticsData));
});
请求以 POST 方式发送。
我们不仅能发送字符串，还能发送表单以及其他格式的数据，在 Fetch 一章有详细讲解，但通常它是一个字符串化的对象。
数据大小限制在 64kb。
当 sendBeacon 请求完成时，浏览器可能已经离开了文档，所以就无法获取服务器响应（对于分析数据来说通常为空）。

还有一个 keep-alive 标志，该标志用于在 fetch 方法中为通用的网络请求执行此类“离开页面后”的请求。你可以在 Fetch API 一章中找到更多相关信息。

如果我们要取消跳转到另一页面的操作，在这里做不到。但是我们可以使用另一个事件 —— onbeforeunload。
```
## 1.3window.onunload
```javascript
当访问者离开页面时，window 对象上的 unload 事件就会被触发。我们可以在那里做一些不涉及延迟的操作，例如关闭相关的弹出窗口。

有一个值得注意的特殊情况是发送分析数据。

假设我们收集有关页面使用情况的数据：鼠标点击，滚动，被查看的页面区域等。

自然地，当用户要离开的时候，我们希望通过 unload 事件将数据保存到我们的服务器上。

有一个特殊的 navigator.sendBeacon(url, data) 方法可以满足这种需求，详见规范 https://w3c.github.io/beacon/。

它在后台发送数据，转换到另外一个页面不会有延迟：浏览器离开页面，但仍然在执行 sendBeacon。

使用方式如下：

let analyticsData = { /* 带有收集的数据的对象 */ };

window.addEventListener("unload", function() {
  navigator.sendBeacon("/analytics", JSON.stringify(analyticsData));
});
请求以 POST 方式发送。
我们不仅能发送字符串，还能发送表单以及其他格式的数据，在 Fetch 一章有详细讲解，但通常它是一个字符串化的对象。
数据大小限制在 64kb。
当 sendBeacon 请求完成时，浏览器可能已经离开了文档，所以就无法获取服务器响应（对于分析数据来说通常为空）。

还有一个 keep-alive 标志，该标志用于在 fetch 方法中为通用的网络请求执行此类“离开页面后”的请求。你可以在 Fetch API 一章中找到更多相关信息。

如果我们要取消跳转到另一页面的操作，在这里做不到。但是我们可以使用另一个事件 —— onbeforeunload。
```
## 1.4window.onbeforeunload
```javascript
如果访问者触发了离开页面的导航（navigation）或试图关闭窗口，beforeunload 处理程序将要求进行更多确认。

如果我们要取消事件，浏览器会询问用户是否确定。

你可以通过运行下面这段代码，然后重新加载页面来进行尝试：

window.onbeforeunload = function() {
  return false;
};
由于历史原因，返回非空字符串也被视为取消事件。在以前，浏览器曾经将其显示为消息，但是根据 现代规范 所述，它们不应该这样。

这里有个例子：

window.onbeforeunload = function() {
  return "有未保存的值。确认要离开吗？";
};
它的行为已经改变了，因为有些站长通过显示误导性和恶意信息滥用了此事件处理程序。所以，目前一些旧的浏览器可能仍将其显示为消息，但除此之外 —— 无法自定义显示给用户的消息。

event.preventDefault() 在 beforeunload 处理程序中不起作用
这听起来可能很奇怪，但大多数浏览器都会忽略 event.preventDefault()。

这意味着，以下代码可能不起作用：

window.addEventListener("beforeunload", (event) => {
  // 不起作用，所以这个事件处理程序没做任何事儿
  event.preventDefault();
});
相反，在这样的处理程序中，应该将 event.returnValue 设置为一个字符串，以获得类似于上面代码的结果：

window.addEventListener("beforeunload", (event) => {
  // 起作用，与在 window.onbeforeunload 中 return 值的效果是一样的
  event.returnValue = "有未保存的值。确认要离开吗？";
});
```
## 1.5readyState
```javascript
如果我们在文档加载完成之后设置 DOMContentLoaded 事件处理程序，会发生什么？

很自然地，它永远不会运行。

在某些情况下，我们不确定文档是否已经准备就绪。我们希望我们的函数在 DOM 加载完成时执行，无论现在还是以后。

document.readyState 属性可以为我们提供当前加载状态的信息。

它有 3 个可能值：

loading —— 文档正在被加载。
interactive —— 文档被全部读取。
complete —— 文档被全部读取，并且所有资源（例如图片等）都已加载完成。
所以，我们可以检查 document.readyState 并设置一个处理程序，或在代码准备就绪时立即执行它。

像这样：

function work() { /*...*/ }

if (document.readyState == 'loading') {
  // 仍在加载，等待事件
  document.addEventListener('DOMContentLoaded', work);
} else {
  // DOM 已就绪！
  work();
}
还有一个 readystatechange 事件，会在状态发生改变时触发，因此我们可以打印所有这些状态，就像这样：

// 当前状态
console.log(document.readyState);

// 状态改变时打印它
document.addEventListener('readystatechange', () => console.log(document.readyState));
readystatechange 事件是跟踪文档加载状态的另一种机制，它很早就存在了。现在则很少被使用。

但是为了完整起见，让我们看看完整的事件流。

这是一个带有 <iframe>，<img> 和记录事件的处理程序的文档：

<script>
  log('initial readyState:' + document.readyState);

  document.addEventListener('readystatechange', () => log('readyState:' + document.readyState));
  document.addEventListener('DOMContentLoaded', () => log('DOMContentLoaded'));

  window.onload = () => log('window onload');
</script>

<iframe src="iframe.html" onload="log('iframe onload')"></iframe>

<img src="http://en.js.cx/clipart/train.gif" id="img">
<script>
  img.onload = () => log('img onload');
</script>
此示例运行 在 sandbox 中。

典型输出：

[1] initial readyState:loading
[2] readyState:interactive
[2] DOMContentLoaded
[3] iframe onload
[4] img onload
[4] readyState:complete
[4] window onload
方括号中的数字表示发生这种情况的大致时间。标有相同数字的事件几乎是同时发生的（± 几毫秒）。

在 DOMContentLoaded 之前，document.readyState 会立即变成 interactive。它们俩的意义实际上是相同的。
当所有资源（iframe 和 img）都加载完成后，document.readyState 变成 complete。这里我们可以发现，它与 img.onload（img 是最后一个资源）和 window.onload 几乎同时发生。转换到 complete 状态的意义与 window.onload 相同。区别在于 window.onload 始终在所有其他 load 处理程序之后运行。
```
## 1.6总结
```javascript
页面生命周期事件：

当 DOM 准备就绪时，document 上的 DOMContentLoaded 事件就会被触发。在这个阶段，我们可以将 JavaScript 应用于元素。
诸如 <script>...</script> 或 <script src="..."></script> 之类的脚本会阻塞 DOMContentLoaded，浏览器将等待它们执行结束。
图片和其他资源仍然可以继续被加载。
当页面和所有资源都加载完成时，window 上的 load 事件就会被触发。我们很少使用它，因为通常无需等待那么长时间。
当用户想要离开页面时，window 上的 beforeunload 事件就会被触发。如果我们取消这个事件，浏览器就会询问我们是否真的要离开（例如，我们有未保存的更改）。
当用户最终离开时，window 上的 unload 事件就会被触发。在处理程序中，我们只能执行不涉及延迟或询问用户的简单操作。正是由于这个限制，它很少被使用。我们可以使用 navigator.sendBeacon 来发送网络请求。
document.readyState 是文档的当前状态，可以在 readystatechange 事件中跟踪状态更改：
loading —— 文档正在被加载。
interactive —— 文档已被解析完成，与 DOMContentLoaded 几乎同时发生，但是在 DOMContentLoaded 之前发生。
complete —— 文档和资源均已加载完成，与 window.onload 几乎同时发生，但是在 window.onload 之前发生。
```

# 2.脚本：async, defer
```
现代的网站中，脚本往往比 HTML 更“重”：它们的大小通常更大，处理时间也更长。

当浏览器加载 HTML 时遇到 <script>...</script> 标签，浏览器就不能继续构建 DOM。它必须立刻执行此脚本。对于外部脚本 <script src="..."></script> 也是一样的：浏览器必须等脚本下载完，并执行结束，之后才能继续处理剩余的页面。

这会导致两个重要的问题：

脚本不能访问到位于它们下面的 DOM 元素，因此，脚本无法给它们添加处理程序等。
如果页面顶部有一个笨重的脚本，它会“阻塞页面”。在该脚本下载并执行结束前，用户都不能看到页面内容：
<p>...content before script...</p>

<script src="https://javascript.info/article/script-async-defer/long.js?speed=1"></script>

<!-- This isn't visible until the script loads -->
<p>...content after script...</p>
这里有一些解决办法。例如，我们可以把脚本放在页面底部。此时，它可以访问到它上面的元素，并且不会阻塞页面显示内容：

<body>
  ...all content is above the script...

  <script src="https://javascript.info/article/script-async-defer/long.js?speed=1"></script>
</body>
但是这种解决方案远非完美。例如，浏览器只有在下载了完整的 HTML 文档之后才会注意到该脚本（并且可以开始下载它）。对于长的 HTML 文档来说，这样可能会造成明显的延迟。

这对于使用高速连接的人来说，这不值一提，他们不会感受到这种延迟。但是这个世界上仍然有很多地区的人们所使用的网络速度很慢，并且使用的是远非完美的移动互联网连接。

幸运的是，这里有两个 <script> 特性（attribute）可以为我们解决这个问题：defer 和 async。
```
## 2.1defer
```javascript
defer 特性告诉浏览器不要等待脚本。相反，浏览器将继续处理 HTML，构建 DOM。脚本会“在后台”下载，然后等 DOM 构建完成后，脚本才会执行。

这是与上面那个相同的示例，但是带有 defer 特性：

<p>...content before script...</p>

<script defer src="https://javascript.info/article/script-async-defer/long.js?speed=1"></script>

<!-- 立即可见 -->
<p>...content after script...</p>
换句话说：

具有 defer 特性的脚本不会阻塞页面。
具有 defer 特性的脚本总是要等到 DOM 解析完毕，但在 DOMContentLoaded 事件之前执行。
下面这个示例演示了上面所说的第二句话：

<p>...content before scripts...</p>

<script>
  document.addEventListener('DOMContentLoaded', () => alert("DOM ready after defer!"));
</script>

<script defer src="https://javascript.info/article/script-async-defer/long.js?speed=1"></script>

<p>...content after scripts...</p>
页面内容立即显示。
DOMContentLoaded 事件处理程序等待具有 defer 特性的脚本执行完成。它仅在脚本下载且执行结束后才会被触发。
具有 defer 特性的脚本保持其相对顺序，就像常规脚本一样。

假设，我们有两个具有 defer 特性的脚本：long.js 在前，small.js 在后。

<script defer src="https://javascript.info/article/script-async-defer/long.js"></script>
<script defer src="https://javascript.info/article/script-async-defer/small.js"></script>
浏览器扫描页面寻找脚本，然后并行下载它们，以提高性能。因此，在上面的示例中，两个脚本是并行下载的。small.js 可能会先下载完成。

……但是，defer 特性除了告诉浏览器“不要阻塞页面”之外，还可以确保脚本执行的相对顺序。因此，即使 small.js 先加载完成，它也需要等到 long.js 执行结束才会被执行。

当我们需要先加载 JavaScript 库，然后再加载依赖于它的脚本时，这可能会很有用。

defer 特性仅适用于外部脚本
如果 <script> 脚本没有 src，则会忽略 defer 特性。
```
## 2.2async
```javascript
async 特性与 defer 有些类似。它也能够让脚本不阻塞页面。但是，在行为上二者有着重要的区别。

async 特性意味着脚本是完全独立的：

浏览器不会因 async 脚本而阻塞（与 defer 类似）。
其他脚本不会等待 async 脚本加载完成，同样，async 脚本也不会等待其他脚本。
DOMContentLoaded 和异步脚本不会彼此等待：
DOMContentLoaded 可能会发生在异步脚本之前（如果异步脚本在页面完成后才加载完成）
DOMContentLoaded 也可能发生在异步脚本之后（如果异步脚本很短，或者是从 HTTP 缓存中加载的）
换句话说，async 脚本会在后台加载，并在加载就绪时运行。DOM 和其他脚本不会等待它们，它们也不会等待其它的东西。async 脚本就是一个会在加载完成时执行的完全独立的脚本。就这么简单，现在明白了吧？

下面是一个类似于我们在讲 defer 时所看到的例子：long.js 和 small.js 两个脚本，只是现在 defer 变成了 async。

它们不会等待对方。先加载完成的（可能是 small.js）—— 先执行：

<p>...content before scripts...</p>

<script>
  document.addEventListener('DOMContentLoaded', () => alert("DOM ready!"));
</script>

<script async src="https://javascript.info/article/script-async-defer/long.js"></script>
<script async src="https://javascript.info/article/script-async-defer/small.js"></script>

<p>...content after scripts...</p>
页面内容立刻显示出来：加载写有 async 的脚本不会阻塞页面渲染。
DOMContentLoaded 可能在 async 之前或之后触发，不能保证谁先谁后。
较小的脚本 small.js 排在第二位，但可能会比 long.js 这个长脚本先加载完成，所以 small.js 会先执行。虽然，可能是 long.js 先加载完成，如果它被缓存了的话，那么它就会先执行。换句话说，异步脚本以“加载优先”的顺序执行。
当我们将独立的第三方脚本集成到页面时，此时采用异步加载方式是非常棒的：计数器，广告等，因为它们不依赖于我们的脚本，我们的脚本也不应该等待它们：

<!-- Google Analytics 脚本通常是这样嵌入页面的 -->
<script async src="https://google-analytics.com/analytics.js"></script>
async 特性仅适用于外部脚本
就像 defer 一样，如果 <script> 标签没有 src 特性（attribute），那么 async 特性会被忽略。
```
## 2.3总结
```javascript
此外，还有一种向页面添加脚本的重要的方式。

我们可以使用 JavaScript 动态地创建一个脚本，并将其附加（append）到文档（document）中：

let script = document.createElement('script');
script.src = "/article/script-async-defer/long.js";
document.body.append(script); // (*)
当脚本被附加到文档 (*) 时，脚本就会立即开始加载。

默认情况下，动态脚本的行为是“异步”的。

也就是说：

它们不会等待任何东西，也没有什么东西会等它们。
先加载完成的脚本先执行（“加载优先”顺序）。
如果我们显式地设置了 script.async=false，则可以改变这个规则。然后脚本将按照脚本在文档中的顺序执行，就像 defer 那样。

在下面这个例子中，loadScript(src) 函数添加了一个脚本，并将 async 设置为了 false。

因此，long.js 总是会先执行（因为它是先被添加到文档的）：

function loadScript(src) {
  let script = document.createElement('script');
  script.src = src;
  script.async = false;
  document.body.append(script);
}

// long.js 先执行，因为代码中设置了 async=false
loadScript("/article/script-async-defer/long.js");
loadScript("/article/script-async-defer/small.js");
如果没有 script.async=false，脚本则将以默认规则执行，即加载优先顺序（small.js 大概会先执行）。

同样，和 defer 一样，如果我们要加载一个库和一个依赖于它的脚本，那么顺序就很重要。
```

# 3.资源加载：onload, onerror
```
浏览器允许我们跟踪外部资源的加载 —— 脚本，iframe，图片等。

这里有两个事件：

onload —— 成功加载，
onerror —— 出现 error。
```
## 3.1加载脚本
```javascript
假设我们需要加载第三方脚本，并调用其中的函数。

我们可以像这样动态加载它：

let script = document.createElement('script');
script.src = "my.js";

document.head.append(script);
……但如何运行在该脚本中声明的函数？我们需要等到该脚本加载完成，之后才能调用它。

请注意：
对于我们自己的脚本，可以使用 JavaScript module，但是它们并未被广泛应用于第三方库。

script.onload
我们的得力助手是 load 事件。它会在脚本加载并执行完成时触发。

例如：

let script = document.createElement('script');

// 可以从任意域（domain），加载任意脚本
script.src = "https://cdnjs.cloudflare.com/ajax/libs/lodash.js/4.3.0/lodash.js"
document.head.append(script);

script.onload = function() {
  // 该脚本创建了一个变量 "_"
  alert( _.VERSION ); // 显示库的版本
};
因此，在 onload 中我们可以使用脚本中的变量，运行函数等。

……如果加载失败怎么办？例如，这里没有这样的脚本（error 404）或者服务器宕机（不可用）。

script.onerror
发生在脚本加载期间的 error 会被 error 事件跟踪到。

例如，我们请求一个不存在的脚本：

let script = document.createElement('script');
script.src = "https://example.com/404.js"; // 没有这个脚本
document.head.append(script);

script.onerror = function() {
  alert("Error loading " + this.src); // Error loading https://example.com/404.js
};
请注意，在这里我们无法获取更多 HTTP error 的详细信息。我们不知道 error 是 404 还是 500 或者其他情况。只知道是加载失败了。

重要：
onload/onerror 事件仅跟踪加载本身。

在脚本处理和执行期间可能发生的 error 超出了这些事件跟踪的范围。也就是说：如果脚本成功加载，则即使脚本中有编程 error，也会触发 onload 事件。如果要跟踪脚本 error，可以使用 window.onerror 全局处理程序。
```
## 3.2其他资源
```javascript
load 和 error 事件也适用于其他资源，基本上（basically）适用于具有外部 src 的任何资源。

例如：

let img = document.createElement('img');
img.src = "https://js.cx/clipart/train.gif"; // (*)

img.onload = function() {
  alert(`Image loaded, size ${img.width}x${img.height}`);
};

img.onerror = function() {
  alert("Error occurred while loading image");
};
但是有一些注意事项：

大多数资源在被添加到文档中后，便开始加载。但是 <img> 是个例外。它要等到获得 src (*) 后才开始加载。
对于 <iframe> 来说，iframe 加载完成时会触发 iframe.onload 事件，无论是成功加载还是出现 error。
这是出于历史原因。
```
## 3.3跨域策略
```javascript
这里有一条规则：来自一个网站的脚本无法访问其他网站的内容。例如，位于 https://facebook.com 的脚本无法读取位于 https://gmail.com 的用户邮箱。

或者，更确切地说，一个源（域/端口/协议三者）无法获取另一个源（origin）的内容。因此，即使我们有一个子域，或者仅仅是另一个端口，这都是不同的源，彼此无法相互访问。

这个规则还影响其他域的资源。

如果我们使用的是来自其他域的脚本，并且该脚本中存在 error，那么我们无法获取 error 的详细信息。

例如，让我们使用一个脚本 error.js，该脚本只包含一个（错误）函数调用：

// 📁 error.js
noSuchFunction();
现在从它所在的同一个网站加载它：

<script>
window.onerror = function(message, url, line, col, errorObj) {
  alert(`${message}\n${url}, ${line}:${col}`);
};
</script>
<script src="/article/onload-onerror/crossorigin/error.js"></script>
我们可以看到一个很好的 error 报告，就像这样：

Uncaught ReferenceError: noSuchFunction is not defined
https://javascript.info/article/onload-onerror/crossorigin/error.js, 1:1
现在，让我们从另一个域中加载相同的脚本：

<script>
window.onerror = function(message, url, line, col, errorObj) {
  alert(`${message}\n${url}, ${line}:${col}`);
};
</script>
<script src="https://cors.javascript.info/article/onload-onerror/crossorigin/error.js"></script>
此报告与上面那个示例中的不同，就像这样：

Script error.
, 0:0
error 的详细信息可能因浏览器而异，但是原理是相同的：有关脚本内部的任何信息（包括 error 堆栈跟踪）都被隐藏了。正是因为它来自于另一个域。

为什么我们需要 error 的详细信息？

因为有很多服务（我们也可以构建自己的服务）使用 window.onerror 监听全局 error，保存 error 并提供访问和分析 error 的接口。这很好，因为我们可以看到由用户触发的实际中的 error。但是，如果一个脚本来自于另一个源（origin），那么正如我们刚刚看到的那样，其中没有太多有关 error 的信息。

对其他类型的资源也执行类似的跨源策略（CORS）。

要允许跨源访问，<script> 标签需要具有 crossorigin 特性（attribute），并且远程服务器必须提供特殊的 header。

这里有三个级别的跨源访问：

无 crossorigin 特性 —— 禁止访问。
crossorigin="anonymous" —— 如果服务器的响应带有包含 * 或我们的源（origin）的 header Access-Control-Allow-Origin，则允许访问。浏览器不会将授权信息和 cookie 发送到远程服务器。
crossorigin="use-credentials" —— 如果服务器发送回带有我们的源的 header Access-Control-Allow-Origin 和 Access-Control-Allow-Credentials: true，则允许访问。浏览器会将授权信息和 cookie 发送到远程服务器。
请注意：
你可以在 Fetch：跨源请求 一章中了解有关跨源访问的更多信息。这一章描述了用于网络请求的 fetch 方法，但策略是完全相同的。

诸如 “cookie” 之类的内容超出了本章的范围，但你可以在 Cookie，document.cookie 一章学习它们。

在我们的示例中没有任何跨源特性（attribute）。因此，跨源访问被禁止。让我们来添加它吧。

我们可以在 "anonymous"（不会发送 cookie，需要一个服务器端的 header）和 "use-credentials"（会发送 cookie，需要两个服务器端的 header）之间进行选择。

如果我们不关心 cookie，那么可以选择 "anonymous"：

<script>
window.onerror = function(message, url, line, col, errorObj) {
  alert(`${message}\n${url}, ${line}:${col}`);
};
</script>
<script crossorigin="anonymous" src="https://cors.javascript.info/article/onload-onerror/crossorigin/error.js"></script>
现在，假设服务器提供了 Access-Control-Allow-Origin header，一切都正常。我们有了完整的 error 报告。
```
## 3.4总结
```javascript
图片 <img>，外部样式，脚本和其他资源都提供了 load 和 error 事件以跟踪它们的加载：

load 在成功加载时被触发。
error 在加载失败时被触发。
唯一的例外是 <iframe>：出于历史原因，不管加载成功还是失败，即使页面没有被找到，它都会触发 load 事件。

readystatechange 事件也适用于资源，但很少被使用，因为 load/error 事件更简单。
```