`transform` 让元素"变形"——移动、旋转、缩放、倾斜,而且**不触发回流、GPU 加速**,是动画性能的基石。本篇讲清 transform 的四大基本函数、transform-origin 变形原点、多重变换的顺序陷阱,以及 3D 变换的入门。

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

# 3. 多重变换与顺序

## 3.1 顺序重要

```css
/* 顺序不同,结果不同(变换是从右到左"应用矩阵") */
transform: translateX(100px) rotate(45deg);  /* 先转再移 */
transform: rotate(45deg) translateX(100px);  /* 先移再转,轴也转了 */
```

★ **关键认知**：多个变换**顺序敏感**——`rotate(45deg) translateX(100px)` 里的 translate 方向**会跟着旋转后的坐标轴走**。⚠ 这是 transform 最细的坑:需要"移过去再转"和"转过去再移"结果不同,写错顺序会导致移动方向出乎意料。

## 3.2 与动画配合

```css
.box {
  /* transform 是可动画的黄金属性,配合 transition/animation */
  transition: transform 0.3s;
}
.box:hover { transform: translateY(-4px) scale(1.05); }
```

★ **关键认知**：`transform` 是**动画性能最好的属性之一**——只触发布局层合成,GPU 加速。⚠ "卡片悬停上浮"的经典效果,就是 hover 时 `translateY(-4px)` + transition,只动 transform 保证流畅。

# 4. 3D 变换入门

## 4.1 perspective 与 rotateX/Y

```css
.container { perspective: 400px; }   /* 父提供透视(近大远小) */
.card {
  transform: rotateY(30deg);         /* 绕 Y 轴转,产生 3D 感 */
  /* rotateX 绕横轴、rotateZ 绕纵轴(=rotate) */
}
```

★ **关键认知**：3D 变换需要 `perspective`(透视,父容器提供)配合 `rotateX/rotateY`,产生近大远小的立体感。⚠ 没有 perspective,3D 旋转会看起来像"被压扁"而非"转过去";`transform-style: preserve-3d` 让子元素共享 3D 空间(做翻转卡片的必需品)。

## 4.2 卡片翻转示例

```css
.flip-card { perspective: 1000px; }
.flip-inner {
  transform-style: preserve-3d;
  transition: transform 0.6s;
}
.flip-card:hover .flip-inner { transform: rotateY(180deg); }
```

★ **关键认知**：**卡片翻转** = 父 `perspective` + 内层 `preserve-3d` + hover 时 `rotateY(180deg)`。⚠ 正面/背面各自 `backface-visibility: hidden`(背面朝上时隐藏)是完整翻转效果的必要细节。

# 5. 本篇小结与下一步

★ **关键认知**：transform 四函数(translate/rotate/scale/skew)做纯"变形"、不碰布局、GPU 加速。要点:百分比相对自身、transform-origin 定原点、多变换顺序敏感、3D 要 perspective。它是动画性能与居中、翻转等特效的基石。

下一步第 14 篇:CSS 变量与函数——自定义属性、calc/min/max/clamp,让样式"可配置、会计算"。