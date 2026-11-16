静止的界面已经完整,但"动起来"才更自然——悬停反馈、元素入场、加载动画,都靠 `transition` 与 `animation`。本篇讲清两者的区别、transition 的触发与写法、keyframes 关键帧、以及动画性能的底线(哪些属性能高效动画)。

# 1. transition 过渡

## 1.1 什么是过渡

```css
.btn {
  background: #42b883;
  transition: background 0.3s;   /* 背景色在 0.3s 内平滑变化 */
}
.btn:hover { background: #357a5c; }
```

★ **关键认知**：`transition` 让属性值**变化时平滑过渡**——从"瞬间跳变"变成"渐变过去"。它需要一个**触发**(如 :hover、类切换、值改变),只做"从 A 到 B 的补间"。⚠ transition **只能单向单段**("从旧值到新值过渡一下")、且**要有状态变化才发生**,不是无限循环。

## 1.2 transition 四要素

```css
.box {
  transition-property: all;       /* 哪些属性过渡(all/具体属性) */
  transition-duration: 0.3s;      /* 时长 */
  transition-timing-function: ease; /* 缓动函数 */
  transition-delay: 0s;           /* 延迟 */

  /* 简写:属性 时长 缓动 延迟(时长、缓动必须) */
  transition: all 0.3s ease 0s;
}
```

★ **关键认知**：transition 四要素——**属性、时长、缓动(`ease` 等)、延迟**;简写时 **`时长` 与 `缓动` 是必须**,属性(`all`)默认。⚠ 缓动函数决定"变化节奏"——`ease-in`(慢进快出)、`ease-out`(快进慢出)、`linear`(匀速),选对缓动能大幅提升"手感"。

# 2. animation 动画

## 2.1 keyframes + animation

```css
.box {
  animation: fade-in 1s ease;    /* 引用关键帧 + 时长 + 缓动 */
}

/* @keyframes 定义动画的"关键帧"序列 */
@keyframes fade-in {
  from { opacity: 0; }           /* 起点 */
  to   { opacity: 1; }           /* 终点 */
}
```

★ **关键认知**：`animation` 引用 `@keyframes`(关键帧),能做 **from→to 或多段(0%→50%→100%)** 的动画;与 transition 不同,**animation 可自动开始、可循环、可多段**。⚠ keyframes 里的百分比是"动画进度的位置",`from`=`0%`、`to`=`100%`。

## 2.2 animation 完整属性

```css
.box {
  animation-name: slide;         /* 关键帧名 */
  animation-duration: 2s;        /* 时长 */
  animation-timing-function: ease;
  animation-delay: 0s;
  animation-iteration-count: infinite;  /* 循环次数(infinite 无限) */
  animation-direction: alternate;       /* 往返 */
  animation-fill-mode: forwards;        /* 结束停留 */
  /* 简写 */
  animation: slide 2s ease infinite alternate;
}
```

★ **关键认知**：`animation` 常用扩展——`iteration-count`(循环)、`direction: alternate`(往返)、`fill-mode: forwards`(**结束后停留在最后一帧**,否则会跳回原样)。⚠ `fill-mode: forwards` 是"动画结束别弹回去"的关键,入场动画常用。

# 3. transition vs animation

| 维度 | transition | animation |
|------|-----------|-----------|
| 触发 | 需要状态变化 | 可自动开始 |
| 段数 | 一段(A→B) | 多段(0%→50%→100%) |
| 循环 | 无 | 可 |
| 复杂度 | 简单 | 更强 |
| 典型 | hover 反馈、颜色过渡 | 入场、加载、循环动画 |

★ **关键认知**：**简单交互反馈(悬停变色、展开)用 transition;复杂/自动/循环动画(入场、加载 spinner)用 animation**。⚠ 一个常见选择困难是"元素出现时想动一下"——那要用 animation(它是自动的),而不是 transition(它等状态变化触发)。

# 4. 动画性能

## 4.1 该动画哪些属性

```css
/* ✓ 高效:只引发合成(composite),不重排重绘 */
.box { animation: move 1s; }
@keyframes move {
  to { transform: translateX(100px); }   /* transform */
}
/* ✓ opacity 也高效 */

/* ⚠ 低效:布局类属性触发回流(重排) */
/* @keyframes bad { to { width: 200px; top: 100px; } } */
```

★ **关键认知**：动画**只该用 `transform` 和 `opacity`**——它们只触发布局层的**合成(composite)**,由 GPU 加速,不掉帧。⚠ 动画 `width/height/top/left/margin` 会触发**回流(重排)**,开销巨大、易掉帧——这是"动画凭什么卡"的答案。

## 4.2 will-change 预告

```css
.box { will-change: transform; }  /* 预告"我要动 transform",提前优化 */
```

★ **关键认知**：`will-change` 预告浏览器"这个属性马上要变",让它提前准备(如升层)。⚠ 但 will-change **别滥用**(每个都预告反而占内存),只在确知要动画的元素上用;完整的性能手段见第 21 篇。

# 5. 常用动画套路

## 5.1 淡入 + 上移入场

```css
@keyframes fade-up {
  from { opacity: 0; transform: translateY(20px); }
  to   { opacity: 1; transform: translateY(0); }
}
.enter { animation: fade-up 0.4s ease-out; }
```

★ **关键认知**：**"淡入 + 上移"**是使用频率最高的入场动画——透明度从 0 到 1、位置从下移到原位,只用 opacity + transform(高效)。⚠ 这类动画要"轻"——0.3~0.5s 足够,太长反而拖沓。

## 5.2 加载 spinner

```css
.spinner {
  width: 40px; height: 40px;
  border: 4px solid #eee;
  border-top-color: #42b883;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}
@keyframes spin { to { transform: rotate(360deg); } }
```

★ **关键认知**：**旋转加载圈** = 圆形边框 + 一边上色 + `rotate` 无限循环,只用 transform(高效)。⚠ 这是"animation 无限循环 + 高效动画"的标准示范,也是 CSS 动画最实用的案例之一。

# 6. 本篇小结与下一步

★ **关键认知**：transition 做状态过渡(需触发、单段),animation 做关键帧动画(自动、多段、可循环)。性能铁律:**动画只用 transform + opacity**。反馈用 transition,入场/循环用 animation。

下一步第 13 篇:transform 变形——translate/rotate/scale 与 transform-origin,让元素"移动、旋转、缩放"。