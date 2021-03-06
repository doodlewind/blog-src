categories: Note

tags:

- Font

date: 2019-02-01

toc: true

title: 文字渲染那些事（二）文字的形状是怎么表示的？
---

在上期文章里，我们介绍了字体文件的结构，也就是一系列的二进制表。在这一期里，我们关注的是这件事情：计算机中的数据是怎么表达文字形状的呢？让我们从点和线的概念开始吧：

<!--more-->

## 从点到线
我们已经知道，TTF 及其之后的字体标准，都支持对文字的平滑缩放。文字放大后那光滑的曲线固然是非常优美的，但它是怎样在计算机中表达的呢？对这一问题最基础的科普，只需要九年义务教育的知识就足够理解了。

在初中数学里我们就知道，过两点即可确定一条直线。那么曲线呢？我们可以从最易于计算的二次曲线开始。步骤是这样的：

1. 给定一条二次曲线，在它上面任取 *p0* 和 *p2* 两个点。
2. 画出曲线在 *p0* 与 *p2* 处的切线，把它们的交点记为 *p1*。
3. 基于 *p0 p1 p2* 这三个点，即可准确地表达这条二次曲线。

看着描述有些复杂？一图胜千言：

![curve-1](/images/text-in-depth/curve-1.png)

这和数学老师以前教的有什么不一样呢？最大的不同在于，和数学题里常见「给曲线上的三个点，求出曲线表达式」的套路不同，我们选择的 *p1* 是在曲线之外（所谓 off-curve）的。这时曲线套的公式并不是 `y= ax^2 + bx + c` 的形式，而是给定一个新参数 *t*，求出点坐标 *p(t)* 的表达式——这就是所谓的二次贝塞尔曲线了。这种表达形式能消除掉一些你在做证明题时需要考虑的特殊情况，因此特别适合优（偷）雅（懒）的工程实现。

举一反三地说，再复杂的曲线，都可以通过多段二次曲线的连接来形成。这时候，让曲线「看起来平滑」的关键，在于两条曲线交点处**一阶导数的连续性**。看图说话，大概就是这样：

![curve-2](/images/text-in-depth/curve-2.png)

是不是有些 Photoshop 里钢笔工具的感觉？不难发现，保证曲线连接处平滑的关键，就在于保证 *p1 p2 p3* 处在同一条直线上。如果整条曲线都满足这个条件，那么 *p2* 其实是多余的，只需要下面的形式就足够了：

![curve-3](/images/text-in-depth/curve-3.png)

像这样，我们只依靠两个 off-curve 的控制点，就能把两段二次曲线平滑地连接起来——这个场景下，我们能够推导出一种能支持两个控制点的曲线：三次贝塞尔曲线。那么，难道我们还需要四次、五次……的曲线吗？不必了！平面字体中各种精巧的形状，总可以用若干段最高三次的贝塞尔曲线拟合得到——**我们能把很长的一段任意曲线近似成许多小段的贝塞尔曲线，它的性质能保证每两小段曲线在连接处的切线斜率都一致，进而让整条曲线都保持平滑**。这种通过近似来化整为零的工程思路，是不是和浮点数「存在舍入误差但足够精确」的原理有些接近呢？


## 线与轮廓
上面对贝塞尔曲线概念的介绍中，包含了一个重要的细节：**并非所有的点都在曲线上**。绘制曲线时，用到的点其实包含了 on-curve 和 off-curve 两种。有了这些点的配合，我们就能组合出复杂的形状了：

![contour-1](/images/text-in-depth/contour-1.png)

上图中使用贝塞尔曲线绘制这个字母 C 的时候，on-curve 的点是实心的，而 off-curve 的点则是空心的。像这样的闭合曲线，我们称之为一个**轮廓**（Contour）。但许多字体的字形可不是一个轮廓就能描述的，像这个：

![contour-2](/images/text-in-depth/contour-2.png)

这个字母 B 就包含了三个形状，每个形状都对应一个轮廓。TTF 字体的实际应用中，每个字形可以有零到多个轮廓。什么，你说哪有文字的轮廓数量是零？有一种威力强大的字符，叫做空格 :)

在 TTF 标准中，有专门的二进制字段来指定一个字形具备多少个轮廓。很有趣的一点是，这个数值可以小于零。轮廓数量是负数又是什么情况呢？这就是所谓的复合字形了。比如拉丁语系中各种带注音符号的文字，其字形实际上可以由其它的字形组合而成，从而减少数据的冗余。这大致对应于下面这样的情形：

![compound-glyph](/images/text-in-depth/compound-glyph.png)

这就是曲线轮廓（contour）与字形（glyph）之间的基本关系了。那么，轮廓上各点的数据又用怎样的方式定位的呢？这就要引出坐标系的概念了。 


## 坐标平面
在 TTF 中，每个字形都是在独立的主网格（Master Grid）上给其中的点定位的。不过，网格的坐标轴不是无限的，每个点的 x y 坐标只能取 -16384 到 +16384 之间的整数（大大超出了你的屏幕分辨率了吧）。对应的这一块方形平面长宽约定为 1em，而每个网格对应一个 font unit。控制点定位在网格上，勾勒出曲线，像这样：

![upm](/images/text-in-depth/upm.png)

点必须在网格上指定，而这个网格的取值范围则是可变的：网格大小可以是 2 的整数次幂，这样一来网格放大一级，就能减少每个点所占用的一位存储空间了。结合 font unit 与 1em 的关系，我们就能得到 units per em (UPM) 的概念，这个值越大则字体越精细。比如我们上一期作为示例使用的 LiberationSans 字体，就使用了 2048 的 UPM 值。

网格同样有自己的坐标原点。但这时，坐标原点的选取可不是随便在字形上找个角落，而和文字的排版方式有关系。比如对于罗马字体，x 轴选取在基线（baseline）位置，而 y 轴既可以选择放置在字形的视觉中心，也可以选择放在左侧，与字形带了少量留白的 left side bearing 重合，直观地看大概是这样：

![grid-1](/images/text-in-depth/grid-1.png)

对于汉字，一个很有趣的地方在于基线位置的变动——记得小时候的草稿纸吗？练英语的稿纸会印上四条线，其中加粗的第三条就是基线。而中文的基线则放置在文字底部，类似于日记本的横线上，像这样:

![grid-2](/images/text-in-depth/grid-2.png)

不过，不同字体坐标轴的选取也并不算是特别重要，相信大家更感兴趣的至少应该是下面这两件事：

* 如何将网格上的点绘制出来？
* 字体乱七八糟的各种 Metrics 度量大概都是啥？

对于前者，在理解了这篇文章的基础上，有一个很好的方式可以让你直观地实现简易的「字体光栅化器」：只要读取出点的坐标，使用现成可控的渲染工具（如 Web 上的 canvas 或 SVG）把这些 TTF 标准下的坐标转换为路径，不就曲线救国地实现了文字的光栅化吗？在不考虑 hinting 等进阶情况的前提下，这个操作并不算复杂。这部分内容还是蛮有趣的，希望可以有机会往下写出来和大家分享 XD

本文的主要内容来源是 Apple 的 [Digitizing Letterform Designs](https://developer.apple.com/fonts/TrueType-Reference-Manual/RM01/Chap1.html) 文档，感兴趣的同学如果移步果家与软家的字体系列，相信能有更多的收获~
