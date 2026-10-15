# 1. 鼠标事件
```
在本章中，我们将详细介绍鼠标事件及其属性。

请注意：此类事件不仅可能来自于“鼠标设备”，还可能来自于对此类操作进行了模拟以实现兼容性的其他设备，例如手机和平板电脑。
```
## 1.1 鼠标事件类型
```javascript
我们已经见过了其中一些事件：

mousedown/mouseup
在元素上点击/释放鼠标按钮。
mouseover/mouseout
鼠标指针从一个元素上移入/移出。
mousemove
鼠标在元素上的每个移动都会触发此事件。
click
如果使用的是鼠标左键，则在同一个元素上的 mousedown 及 mouseup 相继触发后，触发该事件。
dblclick
在短时间内双击同一元素后触发。如今已经很少使用了。
contextmenu
在鼠标右键被按下时触发。还有其他打开上下文菜单的方式，例如使用特殊的键盘按键，在这种情况下它也会被触发，因此它并不完全是鼠标事件。
……还有其他几种事件，我们稍后会学习它们。
```
## 1.2 事件顺序
```javascript
从上面的列表中我们可以看到，一个用户操作可能会触发多个事件。

例如，点击鼠标左键，在鼠标左键被按下时，会首先触发 mousedown，然后当鼠标左键被释放时，会触发 mouseup 和 click。

在单个动作触发多个事件时，事件的顺序是固定的。也就是说，会遵循 mousedown → mouseup → click 的顺序调用处理程序。

点击（译注：即单击）下面的按钮，你会看到事件。并尝试双击它。

在测试台下面记录了所有的鼠标事件，如果它们之间的延迟时间超过 1 秒，那么它们会被水平分割线分开。

我们还可以看出 button 属性允许检测鼠标按键，演示示例如下。

```
## 1.3 鼠标按钮
```javascript
与点击相关的事件始终具有 button 属性，该属性允许获取确切的鼠标按钮。

通常我们不在 click 和 contextmenu 事件中使用这一属性，因为前者只在单击鼠标左键时触发，后者只在单击鼠标右键时触发。

不过，在 mousedown 和 mouseup 事件中则可能需要用到 event.button，因为这两个事件在任何按键上都会触发，所以我们可以使用 button 属性来区分是左键单击还是右键单击。

event.button 的所有可能值如下：

鼠标按键状态	event.button
左键 (主要按键)	0
中键 (辅助按键)	1
右键 (次要按键)	2
X1 键 (后退按键)	3
X2 键 (前进按键)	4
大多数鼠标设备只有左键和右键，对应的值就是 0 和 2。触屏设备中的点按操作也会触发类似的事件。

另外，还有一个 event.buttons 属性，其中以整数的形式存储着当前所有按下的鼠标按键，每个按键一个比特位。在实际开发中，很少会用到这个属性，如果有需要的话，你可以在 MDN 中找到更多细节。

过时的 event.which
一些老代码可能会使用 event.which 属性来获得按下的按键。这是一个古老的非标准的方式，具有以下可能值：

event.which == 1 —— 鼠标左键，
event.which == 2 —— 鼠标中键，
event.which == 3 —— 鼠标右键。
现在，event.which 已经被弃用了，不应再使用它。
```
## 1.4 组合键：shift, alt, ctrl, meta
```javascript
所有的鼠标事件都包含有关按下的组合键的信息。

事件属性：

shiftKey：Shift
altKey：Alt（或对于 Mac 是 Opt）
ctrlKey：Ctrl
metaKey：对于 Mac 是 Cmd
如果在事件期间按下了相应的键，则它们为 true。

比如，下面这个按钮仅在 Alt+Shift+click 时才有效：

<button id="button">Alt+Shift+Click on me!</button>

<script>
  button.onclick = function(event) {
    if (event.altKey && event.shiftKey) {
      alert('Hooray!');
    }
  };
</script>

注意：在 Mac 上我们通常使用 Cmd 代替 Ctrl
在 Windows 和 Linux 上有 Alt，Shift 和 Ctrl。在 Mac 上还有：Cmd，它对应于属性 metaKey。

在大多数情况下，当在 Windows/Linux 上使用 Ctrl 时，在 Mac 是使用 Cmd。

也就说：当 Windows 用户按下 Ctrl+Enter 或 Ctrl+A 时，Mac 用户会按下 Cmd+Enter 或 Cmd+A，以此类推。

因此，如果我们想支持 Ctrl+click，那么对于 Mac 应该使用 Cmd+click。对于 Mac 用户而言，这更舒适。

即使我们想强制 Mac 用户使用 Ctrl+click —— 这非常困难。问题是：在 MacOS 上左键单击和 Ctrl 一起使用会被解释为 右键单击，并且会生成 contextmenu 事件，而不是像 Windows/Linux 中的 click 事件。

因此，如果我们想让所有操作系统的用户都感到舒适，那么我们应该将 ctrlKey 与 metaKey 一起进行检查。

对于 JS 代码，这意味着我们应该检查 if (event.ctrlKey || event.metaKey)。

还有移动设备
键盘组合是工作流的一个补充。这样，如果访客使用键盘操作 —— 它们就会起作用。

但是，如果访客的设备没有键盘 —— 那么这里应该有另一种不使用键盘也能做到这一点的方式。
```
## 1.5 坐标：clientX/Y, pageX/Y
```javascript
所有的鼠标事件都提供了两种形式的坐标：

相对于窗口的坐标：clientX 和 clientY。
相对于文档的坐标：pageX 和 pageY。
我们已经在 坐标 中解释过它们之间的区别。

简而言之，相对于文档的坐标 pageX/Y 以文档的左上角为参照物，并且同一位置的坐标不随页面的滚动而改变。相对于窗口的坐标 clientX/Y 以当前窗口的左上角为参照物，并且同一位置的坐标会随着页面的滚动而改变。

例如，如果我们有一个大小为 500x500 的窗口，并且鼠标在左上角，那么 clientX 和 clientY 均为 0，无论页面如何滚动。

如果鼠标位于中间，那么 clientX 和 clientY 均为 250。这与它在文档中的位置无关。在这方面，它们类似于 position:fixed。

将鼠标移动到输入字段上，可以看到 clientX/clientY（此示例位于 iframe 中，因此坐标是相对于 iframe 的）：

<input onmousemove="this.value=event.clientX+':'+event.clientY" value="Mouse over me">


```
## 1.6 防止在鼠标按下时的选择
```javascript
双击鼠标会有副作用，在某些界面中可能会出现干扰：它会选择文本。

比如，双击下面的文本，除了我们的处理程序外，还会选择文本：

<span ondblclick="alert('dblclick')">Double-click me</span>

如果按下鼠标左键，并在不松开的情况下移动鼠标，这也常常会造成不必要的选择。

有多种防止选择的方法，你可以在 选择（Selection）和范围（Range） 一章中详细阅读。

在这种情况下，最合理的方式是防止浏览器对 mousedown 进行操作。这样能够阻止刚刚提到的两种选择：

Before...
<b ondblclick="alert('Click!')" onmousedown="return false">
  Double-click me
</b>
...After

现在，在双击时，粗体元素不会被选中，并且在粗体元素上按下鼠标左键也不会开始选择。

请注意：其中的文本仍然是可选择的。但是，选择不应该开始于该文本自身，而应该在该文本之前或之后开始。通常，这对用户来说挺好的。

防止复制
如果我们想禁用选择以保护我们页面的内容不被复制粘贴，那么我们可以使用另一个事件：oncopy。

<div oncopy="alert('Copying forbidden!');return false">
  Dear user,
  The copying is forbidden for you.
  If you know JS or HTML, then you can get everything from the page source though.
</div>

如果你试图在 <div> 中复制一段文本，这是行不通的，因为默认行为 oncopy 被阻止了。

当然，用户可以访问页面的 HTML 源码，并且可以从那里获取内容，但并不是每个人都知道如何做到这一点。
```
## 1.7 总结
```javascript
鼠标事件有以下属性：

按钮：button。

组合键（如果被按下则为 true）：altKey，ctrlKey，shiftKey 和 metaKey（Mac）。

如果你想处理 Ctrl，那么不要忘记 Mac 用户，他们通常使用的是 Cmd，所以最好检查 if (e.metaKey || e.ctrlKey)。
窗口相对坐标：clientX/clientY。

文档相对坐标：pageX/pageY。

mousedown 的默认浏览器操作是文本选择，如果它对界面不利，则应避免它。

在下一章中，我们将看到有关鼠标指针移动后的事件，以及如何跟踪其下元素变化的更多详细信息。
```


# 2. 移动鼠标：mouseover/out, mouseenter/leave
```
我们将深入研究鼠标在元素之间移动时发生的事件。
```
## 2.1 事件mouseover/mouseout, relatedTarget
```javascript
当鼠标指针移到某个元素上时，mouseover 事件就会发生，而当鼠标离开该元素时，mouseout 事件就会发生。


这些事件很特别，因为它们具有 relatedTarget 属性。此属性是对 target 的补充。当鼠标从一个元素离开并去往另一个元素时，其中一个元素就变成了 target，另一个就变成了 relatedTarget。

对于 mouseover：

event.target —— 是鼠标移过的那个元素。
event.relatedTarget —— 是鼠标来自的那个元素（relatedTarget → target）。
mouseout 则与之相反：

event.target —— 是鼠标离开的元素。
event.relatedTarget —— 是鼠标移动到的，当前指针位置下的元素（target → relatedTarget）。
在下面这个示例中，每张脸及其功能都是单独的元素。当你移动鼠标时，你可以在文本区域中看到鼠标事件。

每个事件都具有关于 target 和 relatedTarget 的信息：

结果script.jsstyle.cssindex.html

relatedTarget 可以为 null
relatedTarget 属性可以为 null。

这是正常现象，仅仅是意味着鼠标不是来自另一个元素，而是来自窗口之外。或者它离开了窗口。

当我们在代码中使用 event.relatedTarget 时，我们应该牢记这种可能性。如果我们访问 event.relatedTarget.tagName，那么就会出现错误。
```
## 2.2 跳过元素
```javascript
当鼠标移动时，就会触发 mousemove 事件。但这并不意味着每个像素都会导致一个事件。

浏览器会一直检查鼠标的位置。如果发现了变化，就会触发事件。

这意味着，如果访问者非常快地移动鼠标，那么某些 DOM 元素就可能被跳过：


如果鼠标从上图所示的 #FROM 快速移动到 #TO 元素，则中间的 <div>（或其中的一些）元素可能会被跳过。mouseout 事件可能会在 #FROM 上被触发，然后立即在 #TO 上触发 mouseover。

这对性能很有好处，因为可能有很多中间元素。我们并不真的想要处理每一个移入和离开的过程。

另一方面，我们应该记住，鼠标指针并不会“访问”所有元素。它可以“跳过”一些元素。

特别是，鼠标指针可能会从窗口外跳到页面的中间。在这种情况下，relatedTarget 为 null，因为它是从石头缝里蹦出来的（nowhere）：


你可以在下面的测试台中“实时”查看。

它的 HTML 有两个嵌套的元素：<div id="child"> 在 <div id="parent"> 内部。如果将鼠标快速移动到它们上，则可能只有 <div id="child"> 或者只有 <div id="parent"> 触发事件，或者根本没有事件触发。

还可以将鼠标指针移动到 <div id="child"> 中，然后将其快速向下移动过其父级元素。如果移动速度足够快，则父元素就会被忽略。鼠标会越过父元素而不会引起其注意。

结果script.jsstyle.cssindex.html

如果 mouseover 被触发了，则必须有 mouseout
在鼠标快速移动的情况下，中间元素可能会被忽略，但是我们可以肯定一件事：如果鼠标指针“正式地”进入了一个元素（生成了 mouseover 事件），那么一旦它离开，我们就会得到 mouseout。
```
## 2.3 当移动到一个子元素时mouseout
```javascript
mouseout 的一个重要功能 —— 当鼠标指针从元素移动到其后代时触发，例如在下面的这个 HTML 中，从 #parent 到 #child：

<div id="parent">
  <div id="child">...</div>
</div>
如果我们在 #parent 上，然后将鼠标指针更深入地移入 #child，在 #parent 上我们会得到 mouseout！


这听起来很奇怪，但很容易解释。

根据浏览器的逻辑，鼠标指针随时可能位于单个元素上 —— 嵌套最多的那个元素（z-index 最大的那个）。

因此，如果它转到另一个元素（甚至是一个后代），那么它将离开前一个元素。

请注意事件处理的另一个重要的细节。

后代的 mouseover 事件会冒泡。因此，如果 #parent 具有 mouseover 处理程序，它将被触发：


你可以在下面这个示例中很清晰地看到这一点：<div id="child"> 位于 <div id="parent"> 内部。#parent 元素上有 mouseover/out 的处理程序，这些处理程序用于输出事件详细信息。

如果你将鼠标从 #parent 移动到 #child，那么你会看到在 #parent 上有两个事件:

mouseout [target: parent]（离开 parent），然后
mouseover [target: child]（来到 child，冒泡）。
结果script.jsstyle.cssindex.html

如上例所示，当鼠标指针从 #parent 元素移动到 #child 时，会在父元素上触发两个处理程序：mouseout 和 mouseover：

parent.onmouseout = function(event) {
  /* event.target: parent element */
};
parent.onmouseover = function(event) {
  /* event.target: child element (bubbled) */
};
如果我们不检查处理程序中的 event.target，那么似乎鼠标指针离开了 #parent 元素，然后立即回到了它上面。

但是事实并非如此！鼠标指针仍然位于父元素上，它只是更深入地移入了子元素。

如果离开父元素时有一些行为（action），例如一个动画在 parent.onmouseout 中运行，当鼠标指针深入 #parent 时，我们并不希望发生这种行为。

为了避免它，我们可以在处理程序中检查 relatedTarget，如果鼠标指针仍在元素内，则忽略此类事件。

另外，我们可以使用其他事件：mouseenter 和 mouseleave，它们没有此类问题，接下来我们就对其进行详细介绍。
```
## 2.4 事件mouseenter和mouseleave
```javascript
事件 mouseenter/mouseleave 类似于 mouseover/mouseout。它们在鼠标指针进入/离开元素时触发。

但是有两个重要的区别：

元素内部与后代之间的转换不会产生影响。
事件 mouseenter/mouseleave 不会冒泡。
这些事件非常简单。

当鼠标指针进入一个元素时 —— 会触发 mouseenter。而鼠标指针在元素或其后代中的确切位置无关紧要。

当鼠标指针离开该元素时，事件 mouseleave 才会触发。

这个例子和上面的例子相似，但是现在最顶部的元素有 mouseenter/mouseleave 而不是 mouseover/mouseout。

正如你所看到的，唯一生成的事件是与将鼠标指针移入或移出顶部元素有关的事件。当鼠标指针进入 child 并返回时，什么也没发生。在后代之间的移动会被忽略。

结果script.jsstyle.cssindex.html

```
## 2.5 事件委托
```javascript
事件 mouseenter/leave 非常简单且易用。但它们不会冒泡。因此，我们不能使用它们来进行事件委托。

假设我们要处理表格的单元格的鼠标进入/离开。并且这里有数百个单元格。

通常的解决方案是 —— 在 <table> 中设置处理程序，并在那里处理事件。但 mouseenter/leave 不会冒泡。因此，如果类似的事件发生在 <td> 上，那么只有 <td> 上的处理程序才能捕获到它。

<table> 上的 mouseenter/leave 的处理程序仅在鼠标指针进入/离开整个表格时才会触发。无法获取有关其内部移动的任何信息。

因此，让我们使用 mouseover/mouseout。

让我们从高亮显示鼠标指针下的元素的简单处理程序开始：

// 高亮显示鼠标指针下的元素
table.onmouseover = function(event) {
  let target = event.target;
  target.style.background = 'pink';
};

table.onmouseout = function(event) {
  let target = event.target;
  target.style.background = '';
};
现在它们已经激活了。当鼠标在下面这个表格的各个元素上移动时，当前位于鼠标指针下的元素会被高亮显示：

结果script.jsstyle.cssindex.html

在我们的例子中，我们想要处理表格的单元格 <td> 之间的移动：进入一个单元格并离开它。我们对其他移动并不感兴趣，例如在单元格内部或在所有单元格的外部。让我们把这些过滤掉。

我们可以这样做：

在变量中记住当前被高亮显示的 <td>，让我们称它为 currentElem。
mouseover —— 如果我们仍然在当前的 <td> 中，则忽略该事件。
mouseout —— 如果没有离开当前的 <td>，则忽略。
这是说明所有可能情况的代码示例：

// 现在位于鼠标下方的 <td>（如果有）
let currentElem = null;

table.onmouseover = function(event) {
  // 在进入一个新的元素前，鼠标总是会先离开前一个元素
  // 如果设置了 currentElem，那么我们就没有鼠标所悬停在的前一个 <td>，
  // 忽略此事件
  if (currentElem) return;

  let target = event.target.closest('td');

  // 我们移动到的不是一个 <td> —— 忽略
  if (!target) return;

  // 现在移动到了 <td> 上，但在处于了我们表格的外部（可能因为是嵌套的表格）
  // 忽略
  if (!table.contains(target)) return;

  // 给力！我们进入了一个新的 <td>
  currentElem = target;
  onEnter(currentElem);
};


table.onmouseout = function(event) {
  // 如果我们现在处于所有 <td> 的外部，则忽略此事件
  // 这可能是一个表格内的移动，但是在 <td> 外，
  // 例如从一个 <tr> 到另一个 <tr>
  if (!currentElem) return;

  // 我们将要离开这个元素 —— 去哪儿？可能是去一个后代？
  let relatedTarget = event.relatedTarget;

  while (relatedTarget) {
    // 到父链上并检查 —— 我们是否还在 currentElem 内
    // 然后发现，这只是一个内部移动 —— 忽略它
    if (relatedTarget == currentElem) return;

    relatedTarget = relatedTarget.parentNode;
  }

  // 我们离开了 <td>。真的。
  onLeave(currentElem);
  currentElem = null;
};

// 任何处理进入/离开一个元素的函数
function onEnter(elem) {
  elem.style.background = 'pink';

  // 在文本区域显示它
  text.value += `over -> ${currentElem.tagName}.${currentElem.className}\n`;
  text.scrollTop = 1e6;
}

function onLeave(elem) {
  elem.style.background = '';

  // 在文本区域显示它
  text.value += `out <- ${elem.tagName}.${elem.className}\n`;
  text.scrollTop = 1e6;
}
再次，重要的功能是：

它使用事件委托来处理表格中任何 <td> 的进入/离开。因此，它依赖于 mouseover/out 而不是 mouseenter/leave，mouseenter/leave 不会冒泡，因此也不允许事件委托。
额外的事件，例如在 <td> 的后代之间移动都会被过滤掉，因此 onEnter/Leave 仅在鼠标指针进入/离开 <td> 整体时才会运行。
这是带有所有详细信息的完整示例：

结果script.jsstyle.cssindex.html

尝试将鼠标指针移入和移出表格单元格及其内部。快还是慢都没关系。与前面的示例不同，只有 <td> 被作为一个整体高亮显示。
```
## 2.6 总结
```javascript
我们讲了 mouseover，mouseout，mousemove，mouseenter 和 mouseleave 事件。

以下这些内容要注意：

快速移动鼠标可能会跳过中间元素。
mouseover/out 和 mouseenter/leave 事件还有一个附加属性：relatedTarget。这就是我们来自/到的元素，是对 target 的补充。
即使我们从父元素转到子元素时，也会触发 mouseover/out 事件。浏览器假定鼠标一次只会位于一个元素上 —— 最深的那个。

mouseenter/leave 事件在这方面不同：它们仅在鼠标进入和离开元素时才触发。并且它们不会冒泡。
```

# 3. 鼠标拖放事件
```
拖放（Drag’n’Drop）是一个很赞的界面解决方案。取某件东西并将其拖放是执行许多东西的一种简单明了的方式，从复制和移动文档（如在文件管理器中）到订购（将物品放入购物车）。

在现代 HTML 标准中有一个 关于拖放的部分，其中包含了例如 dragstart 和 dragend 等特殊事件。

这些事件使我们能够支持特殊类型的拖放，例如处理从 OS 文件管理器中拖动文件，并将其拖放到浏览器窗口中。之后，JavaScript 便可以访问此类文件中的内容。

但是，原生的拖放事件也有其局限性。例如，我们无法阻止从特定区域的拖动。并且，我们无法将拖动变成“水平”或“竖直”的。还有很多其他使用它们无法完成的拖放任务。并且，移动设备对此类事件的支持非常有限。

因此，在这里我们将看到，如何使用鼠标事件来实现拖放。
```
## 3.1 拖放算法
```javascript
基础的拖放算法如下所示：

在 mousedown 上 —— 根据需要准备要移动的元素（也许创建一个它的副本，向其中添加一个类或其他任何东西）。
然后在 mousemove 上，通过更改 position:absolute 情况下的 left/top 来移动它。
在 mouseup 上 —— 执行与完成的拖放相关的所有行为。
这些都是基础内容。稍后，我们将看到如何实现其他功能，例如当我们将一个东西拖动到一个元素上方时，高亮显示该元素。

下面是拖放一个球的实现代码：

ball.onmousedown = function(event) {
  // (1) 准备移动：确保 absolute，并通过设置 z-index 以确保球在顶部
  ball.style.position = 'absolute';
  ball.style.zIndex = 1000;

  // 将其从当前父元素中直接移动到 body 中
  // 以使其定位是相对于 body 的
  document.body.append(ball);

  // 现在球的中心在 (pageX, pageY) 坐标上
  function moveAt(pageX, pageY) {
    ball.style.left = pageX - ball.offsetWidth / 2 + 'px';
    ball.style.top = pageY - ball.offsetHeight / 2 + 'px';
  }

  // 将我们绝对定位的球移到指针下方
  moveAt(event.pageX, event.pageY);

  function onMouseMove(event) {
    moveAt(event.pageX, event.pageY);
  }

  // (2) 在 mousemove 事件上移动球
  document.addEventListener('mousemove', onMouseMove);

  // (3) 放下球，并移除不需要的处理程序
  ball.onmouseup = function() {
    document.removeEventListener('mousemove', onMouseMove);
    ball.onmouseup = null;
  };

};
如果我们运行这段代码，我们会发现一些奇怪的事情。在拖放的一开始，球“分叉”了：我们开始拖动它的“克隆”。

这是一个正在运行中的示例：


尝试使用鼠标进行拖放，你会看到这种奇怪的行为。

这是因为浏览器有自己的对图片和一些其他元素的拖放处理。它会在我们进行拖放操作时自动运行，并与我们的拖放处理产生了冲突。

禁用它：

ball.ondragstart = function() {
  return false;
};
现在一切都正常了。

这是一个正在运行中的示例：


另一个重要的方面是 —— 我们在 document 上跟踪 mousemove，而不是在 ball 上。乍一看，鼠标似乎总是在球的上方，我们可以将 mousemove 放在球上。

但正如我们所记得的那样，mousemove 会经常被触发，但不会针对每个像素都如此。因此，在快速移动鼠标后，鼠标指针可能会从球上跳转至文档中间的某个位置（甚至跳转至窗口外）。

因此，我们应该监听 document 以捕获它。
```
## 3.2 修正定位
```javascript
在上述示例中，球在移动时，球的中心始终位于鼠标指针下方：

ball.style.left = pageX - ball.offsetWidth / 2 + 'px';
ball.style.top = pageY - ball.offsetHeight / 2 + 'px';
不错，但这存在副作用。要启动拖放，我们可以在球上的任意位置 mousedown。但是，如果从球的边缘“抓住”球，那么球会突然“跳转”以使球的中心位于鼠标指针下方。

如果我们能够保持元素相对于鼠标指针的初始偏移，那就更好了。

例如，我们按住球的边缘处开始拖动，那么在拖动时，鼠标指针应该保持在一开始所按住的边缘位置上。


让我们更新一下我们的算法：

当访问者按下按钮（mousedown）时 —— 我们可以在变量 shiftX/shiftY 中记住鼠标指针到球左上角的距离。我们应该在拖动时保持这个距离。

我们可以通过坐标相减来获取这个偏移：

// onmousedown
let shiftX = event.clientX - ball.getBoundingClientRect().left;
let shiftY = event.clientY - ball.getBoundingClientRect().top;
然后，在拖动球时，我们将鼠标指针相对于球的这个偏移也考虑在内，像这样：

// onmousemove
// 球具有 position: absolute
ball.style.left = event.pageX - shiftX + 'px';
ball.style.top = event.pageY - shiftY + 'px';
能够更好地进行定位的最终代码：

ball.onmousedown = function(event) {

  let shiftX = event.clientX - ball.getBoundingClientRect().left;
  let shiftY = event.clientY - ball.getBoundingClientRect().top;

  ball.style.position = 'absolute';
  ball.style.zIndex = 1000;
  document.body.append(ball);

  moveAt(event.pageX, event.pageY);

  // 移动现在位于坐标 (pageX, pageY) 上的球
  // 将初始的偏移考虑在内
  function moveAt(pageX, pageY) {
    ball.style.left = pageX - shiftX + 'px';
    ball.style.top = pageY - shiftY + 'px';
  }

  function onMouseMove(event) {
    moveAt(event.pageX, event.pageY);
  }

  // 在 mousemove 事件上移动球
  document.addEventListener('mousemove', onMouseMove);

  // 放下球，并移除不需要的处理程序
  ball.onmouseup = function() {
    document.removeEventListener('mousemove', onMouseMove);
    ball.onmouseup = null;
  };

};

ball.ondragstart = function() {
  return false;
};
In action (inside <iframe>):


如果我们按住球的右下角来进行拖动，这种差异会尤其明显。在前面的示例中，球会在鼠标指针下“跳转”一下。现在，更新后的代码可以让我们从当前位置流畅地跟随鼠标。
```
## 3.3 潜在的放置目标
```javascript
在前面的示例中，球可以被放置（drop）到“任何地方”。在实际中，我们通常是将一个元素放到另一个元素上。例如，将一个“文件”放置到一个“文件夹”或者其他地方。

抽象地讲，我们取一个 “draggable” 的元素，并将其放在 “droppable” 的元素上。

我们需要知道：

在拖放结束时，所拖动的元素要放在哪里 —— 执行相应的行为
并且，最好知道我们所拖动到的 “droppable” 的元素的位置，并高亮显示 “droppable” 的元素。
这个解决方案很有意思，只是有点麻烦，所以我们在这儿对此进行介绍。

第一个想法是什么？可能是将 onmouseover/mouseup 处理程序放在潜在的 “droppable” 的元素中？

但这行不通。

问题在于，当我们拖动时，可拖动元素一直是位于其他元素上的。而鼠标事件只发生在顶部元素上，而不是发生在那些下面的元素。

例如，下面有两个 <div> 元素，红色的在蓝色的上面（完全覆盖）。这里，在蓝色的 <div> 中没有办法来捕获事件，因为红色的 <div> 在它上面：

<style>
  div {
    width: 50px;
    height: 50px;
    position: absolute;
    top: 0;
  }
</style>
<div style="background:blue" onmouseover="alert('never works')"></div>
<div style="background:red" onmouseover="alert('over red!')"></div>

与可拖动的元素相同。球始终位于其他元素之上，因此事件会发生在球上。无论我们在较低的元素上设置什么处理程序，它们都不会起作用。

这就是一开始的那个想法，将处理程序放在潜在的 “droppable” 的元素，在实际操作中不起作用的原因。它们不会运行。

那么，该怎么办？

有一个叫做 document.elementFromPoint(clientX, clientY) 的方法。它会返回在给定的窗口相对坐标处的嵌套的最深的元素（如果给定的坐标在窗口外，则返回 null）。如果同一坐标上有多个重叠的元素，则返回最上面的元素。

我们可以在我们的任何鼠标事件处理程序中使用它，以检测鼠标指针下的潜在的 “droppable” 的元素，就像这样：

// 在一个鼠标事件处理程序中
ball.hidden = true; // (*) 隐藏我们拖动的元素

let elemBelow = document.elementFromPoint(event.clientX, event.clientY);
// elemBelow 是球下方的元素，可能是 droppable 的元素

ball.hidden = false;
请注意：我们需要在调用 (*) 之前隐藏球。否则，我们通常会在这些坐标上有一个球，因为它是在鼠标指针下的最顶部的元素：elemBelow=ball。

我们可以使用该代码来检查我们正在“飞过”的元素是什么。并在放置（drop）时，对放置进行处理。

基于 onMouseMove 扩展的代码，用于查找 “droppable” 的元素：

// 我们当前正在飞过的潜在的 droppable 的元素
let currentDroppable = null;

function onMouseMove(event) {
  moveAt(event.pageX, event.pageY);

  ball.hidden = true;
  let elemBelow = document.elementFromPoint(event.clientX, event.clientY);
  ball.hidden = false;

  // mousemove 事件可能会在窗口外被触发（当球被拖出屏幕时）
  // 如果 clientX/clientY 在窗口外，那么 elementfromPoint 会返回 null
  if (!elemBelow) return;

  // 潜在的 droppable 的元素被使用 "droppable" 类进行标记（也可以是其他逻辑）
  let droppableBelow = elemBelow.closest('.droppable');

  if (currentDroppable != droppableBelow) {
    // 我们正在飞入或飞出...
    // 注意：它们两个的值都可能为 null
    //   currentDroppable=null —— 如果我们在此事件之前，鼠标指针不是在一个 droppable 的元素上（例如空白处）
    //   droppableBelow=null —— 如果现在，在当前事件中，我们的鼠标指针不是在一个 droppable 的元素上

    if (currentDroppable) {
      // 处理“飞出” droppable 的元素时的处理逻辑（移除高亮）
      leaveDroppable(currentDroppable);
    }
    currentDroppable = droppableBelow;
    if (currentDroppable) {
      // 处理“飞入” droppable 的元素时的逻辑
      enterDroppable(currentDroppable);
    }
  }
}
在下面这个示例中，当球被拖到球门上时，球门会被高亮显示。

结果style.cssindex.html

现在，我们在整个处理过程中，在当前变量 currentDroppable 中都存储了当前的“放置目标”，可以用它来进行高亮显示或者其他操作。

```
## 3.4 总结
```javascript
我们考虑了一种基础的拖放算法。

关键部分：

事件流：ball.mousedown → document.mousemove → ball.mouseup（不要忘记取消原生 ondragstart）。
在拖动开始时 —— 记住鼠标指针相对于元素的初始偏移（shift）：shiftX/shiftY，并在拖动过程中保持它不变。
使用 document.elementFromPoint 检测鼠标指针下的 “droppable” 的元素。
我们可以在此基础上做很多事情。

在 mouseup 上，我们可以智能地完成放置（drop）：更改数据，移动元素。
我们可以高亮我们正在“飞过”的元素。
我们可以将拖动限制在特定的区域或者方向。
我们可以对 mousedown/up 使用事件委托。一个大范围的用于检查 event.target 的事件处理程序可以管理数百个元素的拖放。
等。
有一些在此基础上已经将体系结构构建好的框架：DragZone，Droppable，Draggable 及其他 class。它们中的大多数做的都是与上述类似的事情，所以现在你应该很容易理解它们了。或者自己动手实现。正如你所看到的，其实挺简单的，有时候比基于第三方解决方案进行改写还容易。
```

# 4. 指针事件
```
指针事件（Pointer Events）是一种用于处理来自各种输入设备（例如鼠标、触控笔和触摸屏等）的输入信息的现代化解决方案。
```
## 4.1 一段简史
```javascript
让我们先做一个简短的概览，以便你对指针事件及其在其它事件类型中所处位置有个粗略认识。

很早以前，只存在鼠标事件。

后来，触屏设备开始普及，尤其是手机和平板电脑。为了使现有的脚本仍能正常工作，它们生成（现在仍生成）鼠标事件。例如，轻触屏幕就会生成 mousedown 事件。因此，触摸设备可以很好地与网页配合使用。

但是，触摸设备比鼠标具有更多的功能。例如，我们可以同时触控多点（多点触控）。然而，鼠标事件并没有相关属性来处理这种多点触控。

因此，引入了触摸事件，例如 touchstart、touchend 和 touchmove，它们具有特定于触摸的属性（这里不再赘述这些特性，因为指针事件更加完善）。

不过这还是不够完美。因为很多其他输入设备（如触控笔）都有自己的特性。而且同时维护两份分别处理鼠标事件和触摸事件的代码，显得有些笨重了。

为了解决这些问题，人们引入了全新的规范「指针事件」。它为各种指针输入设备提供了一套统一的事件。

目前，各大主流浏览器已经支持了 Pointer Events Level 2 标准，版本更新的 Pointer Events Level 3 已经发布，并且大多数情况下与 Pointer Events Level 2 兼容。

因此，除非你写的代码需要兼容旧版本的浏览器，例如 IE 10 或 Safari 12 或更低的版本，否则无需继续使用鼠标事件或触摸事件 —— 我们可以使用指针事件。

这样，你的代码就可以在触摸设备和鼠标设备上都能正常工作了。

话虽如此，指针事件仍然有一些重要的奇怪特性，你应当对它们有所了解以正确使用指针事件，并避免一些意料之外的错误。我们将在本文中对它们进行介绍。
```
## 4.2 指针事件类型
```javascript
指针事件的命名方式和鼠标事件类似：

指针事件	类似的鼠标事件
pointerdown	mousedown
pointerup	mouseup
pointermove	mousemove
pointerover	mouseover
pointerout	mouseout
pointerenter	mouseenter
pointerleave	mouseleave
pointercancel	-
gotpointercapture	-
lostpointercapture	-
不难发现，每一个 mouse<event> 都有与之相对应的 pointer<event>。同时还有 3 个额外的事件没有相应的 mouse...，我们会在稍后详细解释它们。

在代码中用 pointer<event> 替换 mouse<event>
我们可以把代码中的 mouse<event> 都替换成 pointer<event>，程序仍然正常兼容鼠标设备。

替换之后，程序对触屏设备的支持会“魔法般”地提升。但是，我们可能需要在 CSS 中的某些地方添加 touch-action: none。我们会在下文的 pointercancel 一节中描述这里面的细节。
```
## 4.3 指针事件属性
```javascript
指针事件具备和鼠标事件完全相同的属性，包括 clientX/Y 和 target 等，以及一些其他属性：

pointerId —— 触发当前事件的指针唯一标识符。

浏览器生成的。使我们能够处理多指针的情况，例如带有触控笔和多点触控功能的触摸屏（下文会有相关示例）。

pointerType —— 指针的设备类型。必须为字符串，可以是：“mouse”、“pen” 或 “touch”。

我们可以使用这个属性来针对不同类型的指针输入做出不同响应。

isPrimary —— 当指针为首要指针（多点触控时按下的第一根手指）时为 true。

有些指针设备会测量接触面积和点按压力（例如一根手指压在触屏上），对于这种情况可以使用以下属性：

width —— 指针（例如手指）接触设备的区域的宽度。对于不支持的设备（如鼠标），这个值总是 1。
height —— 指针（例如手指）接触设备的区域的长度。对于不支持的设备，这个值总是 1。
pressure —— 触摸压力，是一个介于 0 到 1 之间的浮点数。对于不支持压力检测的设备，这个值总是 0.5（按下时）或 0。
tangentialPressure —— 归一化后的切向压力（tangential pressure）。
tiltX, tiltY, twist —— 针对触摸笔的几个属性，用于描述笔和屏幕表面的相对位置。
大多数设备都不支持这些属性，因此它们很少被使用。如果你需要使用它们，可以在 规范文档 中查看更多有关它们的详细信息。
```
## 4.4 多点触控
```javascript
多点触控（用户在手机或平板上同时点击若干个位置，或执行特殊手势）是鼠标事件完全不支持的功能之一。

指针事件使我们能够通过 pointerId 和 isPrimary 属性的帮助，能够处理多点触控。

当用户用一根手指触摸触摸屏的某个位置，然后将另一根手指放在该触摸屏的其他位置时，会发生以下情况：

第一个手指触摸：
pointerdown 事件触发，isPrimary=true，并且被指派了一个 pointerId。
第二个和后续的更多个手指触摸（假设第一个手指仍在触摸）：
pointerdown 事件触发，isPrimary=false，并且每一个触摸都被指派了不同的 pointerId。
请注意：pointerId 不是分配给整个设备的，而是分配给每一个触摸的。如果 5 根手指同时触摸屏幕，我们会得到 5 个 pointerdown 事件和相应的坐标以及 5 个不同的 pointerId。

和第一个触摸相关联的事件总有 isPrimary=true。

利用 pointerId，我们可以追踪多根正在触摸屏幕的手指。当用户移动或抬起某根手指时，我们会得到和 pointerdown 事件具有相同 pointerId 的 pointermove 或 pointerup 事件。

这是一个记录 pointerdown 和 pointerup 事件的演示:


请注意：你使用的必须是一个多点触控设备（如平板或手机）才能在 pointerId/isPrimary 中看到区别。对于使用鼠标这样的单点触控设备，所有指针事件都会具有相同的 pointerId 和 isPrimary=true 属性。
```
## 4.5 事件：pointercancel
```javascript
pointercancel 事件将会在一个正处于活跃状态的指针交互由于某些原因被中断时触发。也就是在这个事件之后，该指针就不会继续触发更多事件了。

导致指针中断的可能原因如下：

指针设备硬件在物理层面上被禁用。
设备方向旋转（例如给平板转了个方向）。
浏览器打算自行处理这一交互，比如将其看作是一个专门的鼠标手势或缩放操作等。
我们会用一个实际例子来阐释 pointercancel 的影响。

例如，我们想要实现一个像 鼠标拖放事件 中开头提到的那样的一个对球的拖放操作。

用户的操作流和对应的事件如下：

用户按住了一张图片，开始拖拽
pointerdown 事件触发
用户开始移动指针（从而拖动图片）
pointermove 事件触发，可能触发多次
然后意料之外的情况发生了！浏览器有自己原生的图片拖放操作，接管了之前的拖放过程，于是触发了 pointercancel 事件。
现在拖放图片的操作由浏览器自行实现。用户甚至可能会把图片拖出浏览器，放进他们的邮件程序或文件管理器。
我们不会再得到 pointermove 事件了。
这里的问题就在于浏览器”劫持“了这一个互动操作：在“拖放”过程开始时触发了 pointercancel 事件，并且不再有 pointermove 事件会被生成。

这里是拖放示例的演示，并且在拖放过程中，指针事件（只包含 up/down、move 和 cancel）的触发会被记录在 textarea 中：


我们想要实现自己的拖放操作，所以让我们来看看如何告诉浏览器不要接管拖放操作。

阻止浏览器的默认行为来防止 pointercancel 触发。

我们需要做两件事：

阻止原生的拖放操作发生：
正如我们在 鼠标拖放事件 中描述的那样，我们可以通过设置 ball.ondragstart = () => false 来实现这一需求。
这种方式也适用于鼠标事件。
对于触屏设备，还有其他和触摸相关的浏览器行为（除了拖放）。为了避免它们所引发的问题：
我们可以通过在 CSS 中设置 #ball { touch-action: none } 来阻止它们。
之后我们的代码便可以在触屏设备中正常工作了。
经过上述操作，事件将会按照我们预期的方式触发，浏览器不会劫持拖放过程，也不会触发 pointercancel 事件。

这个演示增加了以下几行：


可以看到，pointercancel 事件不再被触发。

现在我们就可以添加让球的位置移动的代码了，并且我们的代码对鼠标和触控设备都有效。
```
## 4.6 指针捕获
```javascript
指针捕获（Pointer capturing）是针对指针事件的一个特性。

这个想法很简单，但是乍一看可能感觉很奇怪，因为在其他任何事件类型中都没有这种东西。

主要的方法是：

elem.setPointerCapture(pointerId) —— 将给定的 pointerId 绑定到 elem。在调用之后，所有具有相同 pointerId 的指针事件都将 elem 作为目标（就像事件发生在 elem 上一样），无论这些 elem 在文档中的实际位置是什么。
换句话说，elem.setPointerCapture(pointerId) 将所有具有给定 pointerId 的后续事件重新定位到 elem。

绑定会在以下情况下被移除：

当 pointerup 或 pointercancel 事件出现时，绑定会被自动地移除。
当 elem 被从文档中移除后，绑定会被自动地移除。
当 elem.releasePointerCapture(pointerId) 被调用，绑定会被移除。
那么，它有什么用？我们一起来看一个实际的例子吧。

指针捕获可以被用于简化拖放类的交互。

让我们来回忆一下在 鼠标拖放事件 中提到的，如何实现一个自定义滑动条。

我们可以创建一个带有条形图的、并且内部有一个“滑块”（thumb）的滑动条元素（slider）：

<div class="slider">
  <div class="thumb"></div>
</div>
添加样式后的效果如下：


用指针事件替换鼠标事件后的实现逻辑：

用户按下滑动条的滑块 thumb —— pointerdown 事件被触发。
然后用户移动指针 —— pointermove 事件被触发，我们让移动事件只作用在 thumb 上。
……在指针的移动过程中，指针可能会离开滑动条的 thumb 元素，移动到 thumb 之上或之下的位置。而 thumb 应该严格在水平方向上移动，并与指针保持对齐。
在基于鼠标事件实现的方案中，要跟踪指针的所有移动，包括指针移动到 thumb 之上或之下的位置时，我们必须在整个文档 document 上分配 mousemove 事件处理程序。

不过，这并不是一个没有副作用的解决方案。其中的一个问题就是，指针在文档周围的移动可能会引起副作用，在其他元素上触发事件处理程序（例如 mouseover）并调用其他元素上与滑动条不相关的功能，这不是我们预期的效果。

这就是 setPointerCapture 适用的场景。

我们可以在 pointerdown 事件的处理程序中调用 thumb.setPointerCapture(event.pointerId)，
这样接下来在 pointerup/cancel 之前发生的所有指针事件都会被重定向到 thumb 上。
当 pointerup 发生时（拖动完成），绑定会被自动移除，我们不需要关心它。
因此，即使用户在整个文档上移动指针，事件处理程序也将仅在 thumb 上被调用。尽管如此，事件对象的坐标属性，例如 clientX/clientY 仍将是正确的 —— 捕获仅影响 target/currentTarget。

主要代码如下：

thumb.onpointerdown = function(event) {
  // 把所有指针事件（pointerup 之前发生的）重定向到 thumb
  thumb.setPointerCapture(event.pointerId);

  // 开始跟踪指针的移动
  thumb.onpointermove = function(event) {
    // 移动滑动条：在 thumb 上监听即可，因为所有指针事件都被重定向到了 thumb
    let newLeft = event.clientX - slider.getBoundingClientRect().left;
    thumb.style.left = newLeft + 'px';
  };

  // 当结束(pointerup)时取消对指针移动的跟踪
  thumb.onpointerup = function(event) {
    thumb.onpointermove = null;
    thumb.onpointerup = null;
    // ...这里还可以处理“拖动结束”相关的逻辑
  };
};

// 注意：无需调用 thumb.releasePointerCapture，
// 它会在 pointerup 时被自动调用
完整示例：


在这个 demo 中还有一个元素，当它的 onmouseover 处理程序被触发时会显示当前的时间。

请注意：当你拖动滑块的时候，鼠标可能会悬停在这个元素上，它的 onmouseover 处理程序不会被触发。

借助于 setPointerCapture，现在拖动滑块不会再产生副作用了。

言而总之，指针捕获为我们带来了两个好处：

代码变得更加简洁，我们不再需要在整个 document 上添加/移除处理程序。绑定会被自动释放。
如果文档中有其他指针事件处理程序，则在用户拖动滑动条时，它们不会因指针的移动被意外地触发。
指针捕获事件
完整起见，这里还需要提及一个知识点。

还有两个与指针捕获相关的事件：

gotpointercapture 会在一个元素使用 setPointerCapture 来启用捕获后触发。
lostpointercapture 会在捕获被释放后触发：其触发可能是由于 releasePointerCapture 的显式调用，或是 pointerup/pointercancel 事件触发后的自动调用。
```
## 4.7 总结
```javascript
指针事件允许我们通过一份代码，同时处理鼠标、触摸和触控笔事件。

指针事件是鼠标事件的拓展。我们可以在事件名称中用 pointer 替换 mouse 来让我们的代码既能继续支持鼠标，也能更好地支持其他类型的设备。

对于浏览器可能会决定进行劫持并自行处理的拖放和复杂的触控交互 —— 请记住取消事件的默认操作，并在 CSS 中为涉及到的元素设置 touch-action: none。

指针事件还额外具备以下能力：

基于 pointerId 和 isPrimary 的多点触控支持。
针对特定设备的属性，例如 pressure 和 width/height 等。
指针捕获：我们可以把 pointerup/pointercancel 之前的所有指针事件重定向到一个特定的元素。
目前，指针事件已经被各大主流浏览器支持，尤其是如果不需要支持 IE10 和 Safari 12 以下的版本，我们可以放心地使用它们。不过即便是针对这些老式浏览器，也可以通过 polyfill 来让它们支持指针事件。


```

# 5. 键盘：keydown和keyup
```
在我们开始学习键盘的相关内容之前，请注意，在现代设备上，还有其他“输入内容”的方法。例如，人们使用语音识别（尤其是在移动端设备上）或用鼠标复制/粘贴。

因此，如果我们想要跟踪 <input> 字段中的所有输入，那么键盘事件是不够的。无论如何，还需要一个名为 input 的事件来跟踪 <input> 字段中的更改。对于这样的任务来说，这可能是一个更好的选择。稍后我们将在 事件：change，input，cut，copy，paste 一章中介绍它们。

当我们想要处理键盘行为时，应该使用键盘事件（虚拟键盘也算）。例如，对方向键 Up 和 Down 或热键（包括按键的组合）作出反应。
```
## 5.1 测试台
```javascript
为了更好地理解键盘事件，你可以使用下面这个测试台。

在文本区域中尝试使用不同的组合键。

结果script.jsstyle.cssindex.html

```
## 5.2 Keydown和Keyup
```javascript
当一个按键被按下时，会触发 keydown 事件，而当按键被释放时，会触发 keyup 事件。

event.code 和 event.key
事件对象的 key 属性允许获取字符，而事件对象的 code 属性则允许获取“物理按键代码”。

例如，同一个按键 Z，可以与或不与 Shift 一起按下。我们会得到两个不同的字符：小写的 z 和大写的 Z。

event.key 正是这个字符，并且它将是不同的。但是，event.code 是相同的：

Key	event.key	event.code
Z	z（小写）	KeyZ
Shift+Z	Z（大写）	KeyZ
如果用户使用不同的语言，那么切换到另一种语言将产生完全不同的字符，而不是 "Z"。它将成为 event.key 的值，而 event.code 则始终都是一样的："KeyZ"。

“KeyZ” 和其他按键代码
每个按键的代码都取决于该按键在键盘上的位置。UI 事件代码规范 中描述了按键代码。

例如：

字符键的代码为 "Key<letter>"："KeyA"，"KeyB" 等。
数字键的代码为："Digit<number>"："Digit0"，"Digit1" 等。
特殊按键的代码为按键的名字："Enter"，"Backspace"，"Tab" 等。
有几种广泛应用的键盘布局，该规范给出了每种布局的按键代码。

有关更多按键代码，请参见 规范的字母数字部分，或者只需在上面的 测试台 中按下一个按键。

大小写敏感："KeyZ"，不是 "keyZ"
这是显而易见的，但人们仍会搞错。

请规避错误类型：它是 KeyZ，而不是 keyZ。像 event.code=="keyZ" 这样的检查不起作用："Key" 的首字母必须大写。

如果按键没有给出任何字符呢？例如，Shift 或 F1 或其他。对于这些按键，它们的 event.key 与 event.code 大致相同：

Key	event.key	event.code
F1	F1	F1
Backspace	Backspace	Backspace
Shift	Shift	ShiftRight 或 ShiftLeft
请注意，event.code 准确地标明了哪个键被按下了。例如，大多数键盘有两个 Shift 键，一个在左边，一个在右边。event.code 会准确地告诉我们按下了哪个键，而 event.key 对按键的“含义”负责：它是什么（一个 “Shift”）。

假设，我们要处理一个热键：Ctrl+Z（或 Mac 上的 Cmd+Z）。大多数文本编辑器将“撤销”行为挂在其上。我们可以在 keydown 上设置一个监听器，并检查哪个键被按下了。

这里有个难题：在这样的监听器中，我们应该检查 event.key 的值还是 event.code 的值？

一方面，event.key 的值是一个字符，它随语言而改变。如果访问者在 OS 中使用多种语言，并在它们之间进行切换，那么相同的按键将给出不同的字符。因此检查 event.code 会更好，因为它总是相同的。

像这样：

document.addEventListener('keydown', function(event) {
  if (event.code == 'KeyZ' && (event.ctrlKey || event.metaKey)) {
    alert('Undo!')
  }
});
另一方面，event.code 有一个问题。对于不同的键盘布局，相同的按键可能会具有不同的字符。

例如，下面是美式布局（“QWERTY”）和德式布局（“QWERTZ”）—— 来自 Wikipedia：



对于同一个按键，美式布局为 “Z”，而德式布局为 “Y”（字母被替换了）。

从字面上看，对于使用德式布局键盘的人来说，但他们按下 Y 时，event.code 将等于 KeyZ。

如果我们在代码中检查 event.code == 'KeyZ'，那么对于使用德式布局键盘的人来说，当他们按下 Y 时，这个测试就通过了。

听起来确实很怪，但事实确实如此。规范 中明确提到了这种行为。

因此，event.code 可能由于意外的键盘布局而与错误的字符进行了匹配。不同键盘布局中的相同字母可能会映射到不同的物理键，从而导致了它们有不同的代码。幸运的是，这种情况只发生在几个代码上，例如 keyA，keyQ，keyZ（我们已经看到了），而对于诸如 Shift 这样的特殊按键没有发生这种情况。你可以在 规范 中找到该列表。

为了可靠地跟踪与受键盘布局影响的字符，使用 event.key 可能是一个更好的方式。

另一方面，event.code 的好处是，绑定到物理键位置的 event.code 会始终保持不变。因此，即使在切换了语言的情况下，依赖于它的热键也能正常工作。

我们想要处理与布局有关的按键？那么 event.key 是我们必选的方式。

或者我们希望一个热键即使在切换了语言后，仍能正常使用？那么 event.code 可能会更好。
```
## 5.3 自动重复
```javascript
如果按下一个键足够长的时间，它就会开始“自动重复”：keydown 会被一次又一次地触发，然后当按键被释放时，我们最终会得到 keyup。因此，有很多 keydown 却只有一个 keyup 是很正常的。

对于由自动重复触发的事件，event 对象的 event.repeat 属性被设置为 true。
```
## 5.4 默认行为
```javascript
默认行为各不相同，因为键盘可能会触发很多可能的东西。

例如：

出现在屏幕上的一个字符（最明显的结果）。
一个字符被删除（Delete 键）。
滚动页面（PageDown 键）。
浏览器打开“保存页面”对话框（Ctrl+S）
……等。
阻止对 keydown 的默认行为可以取消大多数的行为，但基于 OS 的特殊按键除外。例如，在 Windows 中，Alt+F4 会关闭当前浏览器窗口。并且无法通过在 JavaScript 中阻止默认行为来阻止它。

例如，下面的这个 <input> 期望输入的内容为一个电话号码，因此它不会接受除数字，+，() 和 - 以外的按键：

<script>
function checkPhoneKey(key) {
  return (key >= '0' && key <= '9') || ['+','(',')','-'].includes(key);
}
</script>
<input onkeydown="return checkPhoneKey(event.key)" placeholder="请输入手机号" type="tel">

这里 onkeydown 的处理程序使用 checkPhoneKey 来检查被按下的按键。如果它是有效的（0..9 或 +-() 之一），那么将返回 true，否则返回 false。

我们都知道，像上面那样，从事件处理程序返回 false 会阻止事件的默认行为，所以如果按下的按键未通过按键检查，那么 <input> 中什么都不会出现（从事件处理程序返回 true 不会对任何行为产生影响，只有返回 false 会产生对应的影响）。

请注意，像 Backspace、Left、Right 这样的特殊按键在 <input> 中无效。这是严格过滤器 checkPhoneKey 的副作用。这些按键会使 checkPhoneKey 返回 false。

让我们将过滤条件放宽一点，允许 Left、Right、Delete 和 Backspace 按键：

<script>
function checkPhoneKey(key) {
  return (key >= '0' && key <= '9') ||
    ['+','(',')','-','ArrowLeft','ArrowRight','Delete','Backspace'].includes(key);
}
</script>
<input onkeydown="return checkPhoneKey(event.key)" placeholder="Phone, please" type="tel">

现在方向键和删除键都能正常使用了。

……即使我们对按键进行了过滤，但仍然可以使用鼠标右键单击 + 粘贴来输入任何内容。移动端设备提供了其他输入内容的方式。因此，这个过滤器并不是 100% 可靠。

另一种方式是跟踪 oninput 事件 —— 在任何修改后都会触发此事件。这样我们就可以检查新的 input.value，并在其无效时修改它/高亮显示 <input>。或者我们可以同时使用这两个事件处理程序。
```
## 5.5 历史遗留
```javascript
过去有一个 keypress 事件，还有事件对象的 keyCode、charCode 和 which 属性。

大多数浏览器对它们都存在兼容性问题，以致使该规范的开发者不得不弃用它们并创建新的现代的事件（本文上面所讲的这些事件），除此之外别无选择。旧的代码仍然有效，因为浏览器还在支持它们，但现在完全没必要再使用它们。
```
## 5.6 移动端键盘
```javascript
当使用虚拟/移动端键盘时，更正式一点的名字叫做 IME（Input-Method Editor），W3C 标准规定 KeyboardEvent 的 e.keyCode 应该为 229，并且 e.key 应该为 "Unidentified"。

当按下某些按键（例如箭头或退格键）时，虽然其中一些键盘可能仍然使用正确的值来表示 e.key、e.code、e.keyCode…，但并不能保证所有情况下都能对应正确的值。所以你的键盘逻辑可能并不能保证适用于移动设备。
```
## 5.7 总结
```javascript
按一个按键总是会产生一个键盘事件，无论是符号键，还是例如 Shift 或 Ctrl 等特殊按键。唯一的例外是有时会出现在笔记本电脑的键盘上的 Fn 键。它没有键盘事件，因为它通常是被在比 OS 更低的级别上实现的。

键盘事件：

keydown —— 在按下键时（如果长按按键，则将自动重复），
keyup —— 释放按键时。
键盘事件的主要属性：

code —— “按键代码”（"KeyA"，"ArrowLeft" 等），特定于键盘上按键的物理位置。
key —— 字符（"A"，"a" 等），对于非字符（non-character）的按键，通常具有与 code 相同的值。
过去，键盘事件有时会被用于跟踪表单字段中的用户输入。这并不可靠，因为输入可能来自各种来源。我们有 input 和 change 事件来处理任何输入（稍后我们会在 事件：change，input，cut，copy，paste 一章中进行介绍）。它们在任何类型的输入（包括复制粘贴或语音识别）后触发。

当我们真的想要键盘时，我们应该使用键盘事件。例如，对热键或特殊键作出反应。
```

# 6. 滚动
```
scroll 事件允许对页面或元素滚动作出反应。我们可以在这里做一些有用的事情。

例如：

根据用户在文档中的位置显示/隐藏其他控件或信息。
当用户向下滚动到页面末端时加载更多数据。
这是一个显示当前滚动的小函数：

window.addEventListener('scroll', function() {
  document.getElementById('showScroll').innerHTML = window.pageYOffset + 'px';
});
在运行中：

Current scroll = scroll the window

scroll 事件在 window 和可滚动元素上都可以运行。
```
## 6.1 防止滚动
```javascript
我们如何使某些东西变成不可滚动？

我们不能通过在 onscroll 监听器中使用 event.preventDefault() 来阻止滚动，因为它会在滚动发生 之后 才触发。

但是我们可以在导致滚动的事件上，例如在 pageUp 和 pageDown 的 keydown 事件上，使用 event.preventDefault() 来阻止滚动。

如果我们向这些事件中添加事件处理程序，并向其中添加 event.preventDefault()，那么滚动就不会开始。

启动滚动的方式有很多，使用 CSS 的 overflow 属性更加可靠。

有几个练习题，你可以解决或者浏览以下几个任务来看一下 onscroll 的应用。
```