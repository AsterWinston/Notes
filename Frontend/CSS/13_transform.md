`transform` 让元素"变形"——移动、旋转、缩放、倾斜,而且**不触发回流、GPU 加速**,是动画性能的基石。本篇讲清 transform 的四大基本函数、它与合成层的关系、变形矩阵与累乘顺序、transform-origin 变形原点、will-change 的正确姿势,以及 3D 变换的入门。

# 1. 四大变换函数

## 1.1 translate / rotate / scale / skew

```css
.box {
  transform: translate(50px, 20px);   /* 移动:x 向右 50,y 向下 20 */
  transform: translateX(50px);        /* 单方向 */
  transform: rotate(45deg);           /* 顺时针旋转 45° */
  transform: scale(1.5);              /* 放大 1.5 倍(scale(1.5, 2) 双向) */
  transform: skew(10deg);             /* 倾斜 */
}
```

★ **关键认知**：四大函数——**translate 移动、rotate 旋转、scale 缩放、skew 倾斜**。⚠ 关键差异:**transform 不占布局**(别的元素不因它让位)——`translateX` 移动后,原位置仍被占;这点与 `top/left/margin` 等会改变布局的属性截然不同。

## 1.2 尺寸单位与自身百分比

```css
.box {
  /* translate 的百分比相对"自身尺寸"(不是父元素) */
  transform: translate(-50%, -50%);   /* 左移自身一半、上移自身一半 */
}
```

★ **关键认知**：`translate` 的**百分比相对自身尺寸**——`translate(-50%,-50%)` 是"把自身的一半移回",正是第 6、11 篇"绝对居中"里见到的写法。⚠ 这与 `width/left/margin` 百分比相对**父元素**完全不同,是 transform 独有的语义。

## 1.3 一切皆是矩阵

```text
每个函数都是一个 4x4 矩阵,transform 列表 = 矩阵连乘:
translate → [1 0 0 tx]   scale → [sx 0 0 0]   rotate → [cos -sin 0 0]
浏览器把整条连乘"折叠"成一个最终矩阵,再一次性作用到元素上。
matrix(a, b, c, d, e, f) 就是 2D 矩阵的原始写法,DevTools 里能看到折叠结果。
```

★ **关键认知**：transform 的所有函数最终都会被**连乘成一个矩阵**应用——这解释了为什么 rotate 之后的 translate 会"斜着走":连乘时,后面的 translate 已经在旋转后的坐标系里。⚠ 矩阵不可交换(`A×B ≠ B×A`),这就是下面"顺序敏感"的数学根源;另外**scale 会让 border/阴影/文字一起缩**——想要"放大内容不放大边",该用 `zoom` 或改 font-size,而非 scale。

# 2. transform-origin 变形原点

## 2.1 默认原点与改变

```css
.box {
  transform: rotate(45deg);
  transform-origin: center;       /* 默认:绕中心转 */
  transform-origin: left top;     /* 绕左上角转 */
  transform-origin: 50px 50px;    /* 具体坐标 */
}
```

★ **关键认知**：`transform-origin` 定**变形的锚点**——rotate 绕哪转、scale 从哪缩放,默认 `center`(中心)。⚠ 很多"旋转效果不对"是因为没注意原点:绕中心转和绕左上转,结果完全不同;时钟指针要绕端点转,就得改原点。

## 2.2 原点参与的其实是三步矩阵

```css
/* 语义上,绕 (ox, oy) 的变换 = 先平移原点回 0,变换,再平移回去:
   T(ox,oy) · M · T(-ox,-oy)  —— 所以 origin 会影响 rotate/scale/skew 的结果 */
.door {
  transform-origin: left center;      /* 门轴在左 */
  transform: perspective(800px) rotateY(-60deg);  /* 开门效果 */
}
```

★ **关键认知**：原点改变对 **translate 无效**(纯位移与锚点无关),但对 rotate/scale/skew 全部有效——因为实现上原点是"平移到锚点→变换→平移回去"三步的一部分。⚠ 3D 变换里 origin 的第三维 `transform-origin: center center 200px` 还能把锚点推出平面;另外 **origin 百分比相对自身盒子**,和 translate 一样是"自己人"。

# 3. 多重变换与顺序

## 3.1 顺序重要

```css
/* 顺序不同,结果不同(变换是从右到左"应用矩阵") */
transform: translateX(100px) rotate(45deg);  /* 先转再移 */
transform: rotate(45deg) translateX(100px);  /* 先移再转,轴也转了 */
```

★ **关键认知**：多个变换**顺序敏感**——`rotate(45deg) translateX(100px)` 里的 translate 方向**会跟着旋转后的坐标轴走**。⚠ 这是 transform 最细的坑:需要"移过去再转"和"转过去再移"结果不同,写错顺序会导致移动方向出乎意料。

## 3.2 累乘顺序的记法

```text
阅读顺序: 从右向左"依次施加到坐标系"
等价视角: 从左向右"坐标系自身被逐步变换"

transform: A B C  →  实际矩阵 = A × B × C
先施加 C 还是先施加 A? 点 p 的变换是 p' = A·B·C·p,
C 离点最近,先作用于点;但 B 作用在"A、C 变换过的坐标系"里。
```

★ **关键认知**：记两套视角就不会晕——**从右往左读是"对点的操作顺序"**,**从左往右读是"对坐标系的操作顺序"**;`rotate() translate()` 里 translate 之所以"斜着走",是因为它在已被 rotate 扭转过的坐标系中执行。⚠ 实战建议:**把 translate 放最前面**(先定位后姿态),除非你明确要"沿旋转后的方向走"(开门、钟摆)。

## 3.3 与动画配合

```css
.box {
  /* transform 是可动画的黄金属性,配合 transition/animation */
  transition: transform 0.3s;
}
.box:hover { transform: translateY(-4px) scale(1.05); }
```

★ **关键认知**：`transform` 是**动画性能最好的属性之一**——只触发布局层合成,GPU 加速。⚠ "卡片悬停上浮"的经典效果,就是 hover 时 `translateY(-4px)` + transition,只动 transform 保证流畅。

# 4. transform 与合成层、布局的分工

## 4.1 合成层机制

```text
渲染流水线: Style → Layout → Paint → Composite
                 │          │        └ transform/opacity 变化只走到这里(GPU)
                 │          └ width/top/margin 变化要回到这里(回流,贵)
                 └ color/background 变化到这里(重绘,中)
元素被"提升"成独立合成层后,transform 动画 = 移动一张 GPU 纹理,几乎零成本。
```

★ **关键认知**：`transform` 动画便宜是因为它**只改合成阶段的矩阵**,布局与绘制结果被当成现成纹理复用。⚠ 提升成合成层也要**代价**:每层一张 GPU 位图,`transform: translateZ(0)`/`will-change` 满屏乱加会把显存打爆、滚动反而掉帧——这正是第 21 篇 will-change 说的"滥用爆内存";另外 transform 会**创建包含块与层叠上下文**(见第 6 篇),fixed 子元素会被它"捕获",这是"加了 transform 之后 fixed 弹窗错位"的元凶。

## 4.2 与 position 的分工

| 需求 | 用什么 | 原因 |
|------|--------|------|
| 永久性地挪位置 | `position: relative` + top/left | 影响最终布局语义 |
| 动画/交互位移 | `transform` | 不触发回流,可 GPU 合成 |
| 居中且尺寸未知 | absolute + translate(-50%,-50%) | 百分比相对自身的独有组合 |
| 不改变占位、其他元素不让位 | transform | relative/top 也不让位,但会动回流 |

★ **关键认知**：分工原则——**静态定位交给 position/布局,动态位移交给 transform**。⚠ 用 `top/left` 做动画是经典错误:每帧触发回流;而用 `translate` 做静态排版又会让"布局上的位置"和"视觉位置"不一致,调试、碰撞检测、`getBoundingClientRect` 都会被骗(它返回**变换后**的盒子)。

# 5. 3D 变换入门

## 5.1 perspective 与 rotateX/Y

```css
.container { perspective: 400px; }   /* 父提供透视(近大远小) */
.card {
  transform: rotateY(30deg);         /* 绕 Y 轴转,产生 3D 感 */
  /* rotateX 绕横轴、rotateZ 绕纵轴(=rotate) */
}
```

★ **关键认知**：3D 变换需要 `perspective`(透视,父容器提供)配合 `rotateX/rotateY`,产生近大远小的立体感。⚠ 没有 perspective,3D 旋转会看起来像"被压扁"而非"转过去";`transform-style: preserve-3d` 让子元素共享 3D 空间(做翻转卡片的必需品)。另外**父 perspective 与自身 `perspective()` 函数不等价**:前者所有子元素共用一个消失点,后者每个元素独立透视——多卡片同时 3D 时两种观感明显不同。

## 5.2 卡片翻转示例

```css
.flip-card { perspective: 1000px; }
.flip-inner {
  transform-style: preserve-3d;
  transition: transform 0.6s;
}
.flip-card:hover .flip-inner { transform: rotateY(180deg); }
.flip-face { backface-visibility: hidden; }
.flip-back { transform: rotateY(180deg); }
```

★ **关键认知**：**卡片翻转** = 父 `perspective` + 内层 `preserve-3d` + hover 时 `rotateY(180deg)`。⚠ 正面/背面各自 `backface-visibility: hidden`(背面朝上时隐藏)是完整翻转效果的必要细节;还要注意 `preserve-3d` **会被 overflow/opacity/filter 打断**——给翻转容器加 `overflow: hidden` 的瞬间 3D 就塌成平面,这是"翻面后背面直接盖正面"的常见原因。

# 6. 本篇小结与下一步

★ **关键认知**：transform 四函数(translate/rotate/scale/skew)本质是**连乘成一个矩阵**——矩阵不可交换所以顺序敏感(translate 在旋转后的坐标系里斜着走)、transform-origin 只影响非位移变换。它与 `top/left` 的分工是"静态归布局、动态归合成":只走 Composite 阶段、GPU 加速,但会创建层叠上下文/捕获 fixed、滥用 will-change 升层会爆显存。3D 要 perspective + preserve-3d + backface-visibility。

下一步第 14 篇:CSS 变量与函数——自定义属性、calc/min/max/clamp,让样式"可配置、会计算"。
