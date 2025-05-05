本章介绍如何在着色器中绘制基本几何形状。将这部分内容放在本书这么靠后的位置，可能看起来有点奇怪，因为在传统计算机图形学中，绘制线条、盒子和圆形等形状是基础内容。然而，在程序生成着色器中，你需要以一种颇为别扭的方式来完成，而在我们讲解完图案和抗锯齿之后，现在正是介绍这部分内容的最佳时机。其关键在于创建距离函数。 


> 译者注： 老师在这里推导过程比较少，我学习 shader就是从 sdf推导开始学起，因为我数学不好，所以我用画图的方式理解了很多 sdf的推导过程，详细可以查看 https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzkzNzY1MTM0OQ==&action=getalbum&album_id=3414230896834969605#wechat_redirect   这个系列最开始都是 SDF推导

## 圆圈和椭圆
回想一下第4章中关于圆的例子。圆心位于$(x_0, y_0)$的圆，到其圆心的距离为
$$d = \sqrt{(x - x_0)^2+(y - y_0)^2}$$
半径为\(R\)的圆，到其边缘的距离是
$$\vert\sqrt{(x - x_0)^2+(y - y_0)^2}-R\vert$$
但出于着色的目的，省略绝对值会得到一个更好的函数。适合着色器的函数
$$f(x,y)=\sqrt{(x - x_0)^2+(y - y_0)^2}-R$$
是一个有符号距离函数（常缩写为“SDF”），它既能告诉我们点$(x,y)$到轮廓上最近点的距离，又能表明我们是在轮廓内部$(f<0$还是外部$f>0$。恰好位于轮廓上的点$f = 0$，而且这个函数不仅有平滑的导数，其梯度的模值还是恒定的，这使得它非常适合与抗锯齿函数（aastep）配合进行阈值处理，以生成效果良好的抗锯齿结果。如果我们使用“距离”的传统定义，也就是带有绝对值的形式，那么在$f(x,y)=0$处导数就会不连续，使用自动求导来进行抗锯齿处理就会出现问题。如果你回顾一下第5章中介绍的一些关于网格线的技巧，就会注意到那些设计出来在通过抗锯齿函数（aastep）处理时表现良好的函数，实际上都是有符号距离函数。

大家可能会认为，通过将函数修改为
$$f(x,y)=\sqrt{\frac{(x - x_0)^2}{a^2}+\frac{(y - y_0)^2}{b^2}} - 1$$
就可以绘制出一个椭圆，其中\(a\)和\(b\)分别是椭圆的长半轴和短半轴。然而，这个函数并不是一个距离函数，因为它在不同方向上的缩放比例不同。它具备我们需要的一些属性：在轮廓上其值为\(0\)，在轮廓内部为负，在轮廓外部为正，并且在轮廓上及其附近都有局部平滑的导数。但是，各向异性缩放是个问题。用它来绘制椭圆的内部很容易，但如果我们想在椭圆的边缘绘制一条线，要让这条线具有均匀的宽度就很棘手了。不过也不是不可能，不是非常困难，但有点棘手。 我们可以稍微滥用一下自动求导，并使用第 6 章末尾提到的雅可比矩阵从屏幕空间转换到着色器空间，但要在着色器空间中获得均匀的宽度，我们还需要考虑视图空间中表面的法线。对于三维空间中任意取向的表面，这会变得很复杂。

对于椭圆，表达和计算其真实的欧几里得符号距离函数（SDF）是一个出人意料的难题，我们不会在此详细探讨，仅需注意，计算到椭圆（一种二次曲线）上最近点的距离，这个看似简单的问题，需要求解一个四次方程。存在一个封闭形式的解析解，但这涉及大量工作，并且在使用浮点运算时需要特别留意数值精度。如果你想了解所有细节，我们推荐Inigo Quilez在 https://iquilezles.org/articles/ellipsedist/ 上的详尽解释。 

![2505042547 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/05/05/1mg/2505042547-.png)
*Contour plots for the SDF of a circle (left), a stretched/squashed circle (center) and
the true SDF of an ellipse (right). Ellipse SDF and color scheme by Inigo Quilez.*

幸运的是，有一个通常效果还不错的简便方法，即通过将隐函数$f(x, y)$的值除以其梯度的长度来对其进行归一化。椭圆有一个简单的隐函数，其解析导数也相对简单。我们可以用$f(x, y)=\frac{x^{2}}{a^{2}}+\frac{y^{2}}{b^{2}} - 1$作为椭圆的隐函数，但加上一个平方根会使函数的非线性程度降低，并且能显著改进伪距离函数：
$$f(x, y)=\sqrt{\frac{x^{2}}{a^{2}}+\frac{y^{2}}{b^{2}} - 1}$$
$$\nabla f = (\frac{x}{a^{2}\sqrt{\frac{x^{2}}{a^{2}}+\frac{y^{2}}{b^{2}}}}, \frac{y}{b^{2}\sqrt{\frac{x^{2}}{a^{2}}+\frac{y^{2}}{b^{2}}}})$$

$$\|\nabla f\| = \sqrt{\frac{x^{2}}{a^{4}(\frac{x^{2}}{a^{2}}+\frac{y^{2}}{b^{2}})}+\frac{y^{2}}{b^{4}(\frac{x^{2}}{a^{2}}+\frac{y^{2}}{b^{2}})}}$$


> 译者注： 以下给出椭圆偏导数的求解过程, 但是有一个-1为什么没了，不清楚

求 $f(x,y)$ 对 $x$ 的偏导数 $\frac{\partial f}{\partial x}$：
	1. 令 $u = \frac{x^{2}}{a^{2}}+\frac{y^{2}}{b^{2}} - 1$，则 $f(x,y)=\sqrt{u}$。
    2. 根据复合函数求导法则，先对 $\sqrt{u}$ 关于 $u$ 求导，$(\sqrt{u})^\prime=\frac{1}{2\sqrt{u}}$。
    3. 再对 $u$ 关于 $x$ 求导，$\frac{\partial u}{\partial x}=\frac{2x}{a^{2}}$。
    4. 所以 $\frac{\partial f}{\partial x}=\frac{1}{2\sqrt{u}}\cdot\frac{2x}{a^{2}}=\frac{x}{a^{2}\sqrt{\frac{x^{2}}{a^{2}}+\frac{y^{2}}{b^{2}} - 1}}$，





> 译者注： 求解过程结束，进入正文


下面给出此公式的GLSL代码，之后是距离函数的演示等高线图。考虑到它比精确版本简单得多，其效果相当不错，并且在$f(x, y)=0$附近相当准确，这使得它非常适合用于二维椭圆的绘制。它不是一个真正的有向距离函数（SDF），但对于这个应用来说已经足够了。
```glsl
// 通过隐式方程值除以梯度长度来计算椭圆的“距离”。不是真正的有向距离函数，但对于二维绘图来说足够好了。
float SDFfakedellipse(vec2 p, vec2 ab) {
    vec2 h = p/ab; // 只是一个辅助中间值，用于计算f和g
    float f = sqrt(dot(h,h)); // 这是sqrt(x^2/a^2 + y^2/b^2)
    vec2 g = h / (ab*f); // f的梯度：(x/(a^2*f), y/(b^2*f))
    return (f - 1.0) / length(g); // (f - 1.0)==0 是我们的隐式椭圆
}
```
椭圆伪有向距离函数（SDF）的等高线图，在近距离范围内效果良好。

### 矩形
我们选择圆形作为有符号距离函数（SDF）的第一个例子，并非因为它是最重要的（可能并非如此），而是因为它特别简单。我们的下一个例子——矩形，可以说更为重要，因为矩形在现代计算机显示屏中随处可见。尽管矩形作为顶级几何图形很容易绘制，而且更为常见，在这种情况下，你只需绘制形状内部的像素，而无需关心外部的情况，但能够在着色器中将矩形表示为图案的一部分也是很有用的。 

与所有几何问题一样，选择合适的坐标系可以使解决方案特别简单。如果正方形或矩形的边与坐标轴对齐，而不是任意定向，那么到它们的距离最容易表示；如果形状以原点为中心，那就更容易了。处于其他位置和方向的矩形可以通过坐标变换来绘制。GPU旨在高效处理变换，甚至在片段着色器中，在像素级别执行此类操作也是完全合理的。为了使到形状的距离度量与形状的大小无关，在评估有符号距离场（SDF）之前进行的任何变换都应该是刚性的，即不包括任何缩放或扭曲。 
![2505044720 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/05/05/1mg/2505044720-.png)
*Expressing an axis-aligned rectangle efficiently in terms of |x| and |y|.*

对于一个中心在原点、在x和y方向上边长分别为2a和2b的轴对齐矩形，一个有点简单但可行的有符号距离函数（SDF）可以简单地表示为f (x, y)=max(|x|−a,|y|−b) 。

![2505044757 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/05/05/1mg/2505044757-.png)
*Contour plots for |x|−a (left), |y|−a (center) and max (|x|−a ,|y|−b) (right).*

上述函数或许并非矩形的“正确”有向距离函数（SDF）。它在沿矩形边缘定向的坐标系中使用了切比雪夫距离（“棋盘距离”），在许多情况下，这完全没问题。然而，有向距离函数的传统定义使用的是常规欧几里得距离。在矩形内部，边缘对齐坐标系中的切比雪夫距离恰好与欧几里得距离相同，但在矩形外部，我们需要采用不同的方法来计算到各角的距离。由于我们所选坐标系具有对称性，我们只需考虑一个角，即位于$(a, b)$的角，到该角的欧几里得距离为$\sqrt{(|x| - a)^2 + (|y| - b)^2}$

![2505044904 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/05/05/1mg/2505044904-.png)
*Different cases to consider for a proper Euclidean SDF for a rectangle.*

在GPU代码中使用条件语句并不是一个好主意。编译器有时可以优化掉它们，但我们还是友好一些，给它提供对GPU友好的代码。我们可以通过巧妙地使用min和max函数在不同的情况之间进行选择，由于有直接的硬件支持，这些函数的速度非常快。

下面这个GLSL函数的工作原理并非一目了然，但你应该可以通过纸笔运算来验证它确实有效。表达式 `min(max(d.x, d.y), 0.0)` 表示内部的负切比雪夫距离，在外部各处均钳制为零；而表达式 `length(max(d, vec2(0.0)))` 表示外部的正欧几里得距离，在内部各处均钳制为零。它们的和就是最终的有符号距离场（SDF）。确实有点复杂，但对于当前这一代GPU以及现有的GLSL编译器而言，这段特定代码的速度是最优的，而速度至关重要。 

```
// Signed distance function for an axis-aligned box, centered at the origin
// and with its edges at +/- ab. Based on code by Inigo Quilez.
float SDFbox(vec2 p, vec2 ab) {
	 vec2 d = abs(p) - ab;
	 float inside = min(max(d.x, d.y), 0.0); // Chebyshev distance inside
	 float outside = length(max(d, vec2(0.0))); // Euclidean distance outside
	 return inside + outside; // Either of these are zero, so they don’t interfere
}
```
此有符号距离函数（SDF）的等高线图如下所示，和 rounded box放在一起
外角外侧的等距线是四分之一圆，这也为我们提供了一种绘制圆角矩形的方法。通过将阈值设为某个正值r而不是零，我们可以得到一个角半径为r的圆角形状。这也会使矩形在各个方向上增大r，因此，为了保持其大小为2a×2b，我们可以在计算有符号距离函数（SDF）之前从a和b中减去r。如果我们随后也从结果中减去r，那么我们就得到了一个合适的 “圆角框” 有符号距离函数（SDF）：
```
// SDF for an axis-aligned box with rounded corners and edges at +/- ab
float SDFroundedbox(vec2 p, vec2 ab, float r) {
 return SDFbox(p, ab - r) - r;
}
```
![2505045147 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/05/05/1mg/2505045147-.png)
*Proper SDF functions for a rectangle (left) and for a rounded rectangle (right).*

鉴于我们最近提出“速度很重要”这一观点，圆角盒子函数调用另一个函数可能看起来有些奇怪。然而，这实际上并非真正的函数调用。目前，GLSL中的所有函数都是内联的，这实际上只是额外的两次减法运算，这是完成该任务所需的最少运算量。如果函数不进行内联，那么重复（复制粘贴）被调用函数中的代码会更好。在未来的GLSL版本以及未来的通用GPU着色中，内联可能会成为可选操作，但届时，任何优秀的编译器都会决定将像SDFbox这样简短且简单的函数进行内联。 

## 线


最早的计算机生成图像是线画图，并且线条在计算机图形学中仍然很常见。在参数形式中，一条线可以表示为：
$$\overline{p} = \overline{p}_0 + t\overline{v}$$

在二维空间中，它有五个参数：$\overline{p}_0$和$\overline{v}$各自的两个分量，以及“自由参数”$t$。然而，对于一条无限长的线来说，实际上只有两个自由度，因为$\overline{v}$的长度和$t$的缩放都是任意的（只要$\overline{v}$不全为零），并且点$\overline{p}_0$可以沿着$\overline{v}$移动而不改变线的位置。经典教科书上的“直线方程”$y = ax + b$因无法描述垂直线而声名狼藉，但它的两个参数$a$和$b$是相互独立的。

在隐式形式中（这是我们在着色时会使用的形式），一条线由下式表示：
$$f(\overline{p}) = \overline{N} \cdot \overline{p} + k = 0$$

向量$\overline{N}$与线垂直，$k$决定了线相对于原点的位置。这里有三个参数，但仍然只有两个自由度，因为$\overline{N}$的长度和$k$的缩放是相关的。

在计算机图形学中，我们所说的“线”通常是指从一个点$\overline{p}_1$到另一个点$\overline{p}_2$的线段。在二维空间中，这总共构成了四个独立参数。对于这种情况，参数形式很容易修改——我们只需为参数$t$确定一些范围，通常是$0 \leq t \leq 1$，然后将方程写成两个点之间的线性插值，这里的“线性”是其字面意义：
$$\overline{p} = \overline{p}_1 + t(\overline{p}_2 - \overline{p}_1) = (1 - t)\overline{p}_1 + t\overline{p}_2$$

对于隐式形式，当我们不仅想通过$f(\overline{p})$的符号来确定我们是在线上还是在它的某一侧，还想知道到线上最近点的距离时，事情就没那么简单了。为了计算到线段上最近点的距离，我们需要确保$|\overline{N}| = 1$以得到真实距离，并且我们还需要检查有限长线上的最近点是否在两个端点之间。如果它在该区间之外，那么到线段的距离就是到两个端点中最近一个的距离。

在旧版的Renderman SL中，有一个名为`ptlined`的函数，用于计算一个点到一条线段的距离。在OSL中，有一个有点晦涩的三参数版本的`distance`函数，它以三个点作为输入并完成相同的工作。在GLSL中，我们需要自己实现这个函数，但这并不困难。我们只需将点正交投影到无限长线上，算出该点处参数$t$的值，将该参数限制在$[0, 1]$范围内，然后计算从我们的点到线上该点的距离。使用GLSL中的内置函数`dot`、`clamp`和`length`，代码非常简洁：
```glsl
// 从点p到从p1到p2、宽度为w的线段的有符号距离
float SDFline( vec2 p, vec2 p1, vec2 p2, float w ) {
    vec2 p1p2 = p2 - p1;
    vec2 p1p = p - p1;
    float t = clamp( dot(p1p, p1p2) / dot(p1p2, p1p2), 0.0, 1.0 );
    return length(p1p – t*p1p2) – w; // 为SDF加上偏移距离w
}
```

请注意，在上述代码中，我们没有显式地对向量$\overline{p}_2 - \overline{p}_1$进行归一化处理。我们仍然需要进行一次除法运算，但通过不进行归一化，我们节省了一次开方运算：
$$\overline{p}_{12} = \overline{p}_2 - \overline{p}_1$$
$$\overline{p}_{1}p = \overline{p} - \overline{p}_1$$
$$\hat{p}_{12} = \frac{\overline{p}_{12}}{\|\overline{p}_{12}\|}$$
$$\overline{p}_t = \overline{p}_1 + (\overline{p}_{1}p \cdot \hat{p}_{12})\hat{p}_{12}$$
$$d = \|\overline{p} - \overline{p}_t\| = \overline{p} - \overline{p}_1 + (\overline{p}_{1}p \cdot \hat{p}_{12})\hat{p}_{12} = \overline{p}_{1}p + \frac{(\overline{p}_{1}p \cdot \overline{p}_{12})\overline{p}_{12}}{\|\overline{p}_{12}\|^2}$$
$t = \frac{\overline{p}_{1}p \cdot \overline{p}_{12}}{\|\overline{p}_{12}\|^2}$，并将其限制在$[0, 1]$范围内。

这种公式的另一个优点是，对$t$的限制总是在$[0, 1]$范围内，而不是$[0, |\overline{p}_{12}|]$。归一化通常是由硬件加速的，但在撰写本文时（2025年），上述代码是在GLSL中实现此功能的最快方法。速度并非一切，显式进行归一化处理也不会慢太多，但对于一个我们可能会经常使用的函数来说，速度快是有好处的。

用这个有向距离函数（SDF）绘制的线具有圆形端点，这可能并不总是你想要的效果。如果需要直线端点，可以绘制一个窄框来代替。


![2505044010 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/05/05/1mg/2505044010-.png)
*（左图）带圆形端点的线的有向距离函数（SDF）和（右图）带直线端点的框的有向距离函数（SDF）*

如何最好地旋转和缩放一个轴对齐的框，使其一端位于$p_1$，另一端位于$p_2$，这一点并不直观，所以这里给出一段尽可能高效的代码。它基于前面介绍的`SDFbox`函数。
```glsl
// 从点p到从p1到p2、宽度为w且端点为直线的线段的有向距离
float SDFboxline(vec2 p, vec2 p1, vec2 p2, float w) {
    float len = length(p2 - p1); // 线段长度
    vec2 v = (p2 - p1) / len; // 沿线段的单位向量
    vec2 plocal = p - (p1 + p2) * 0.5; // 移动到线段中点
    plocal = mat2(v.x, -v.y, v.y, v.x) * plocal; // 使坐标与线段方向一致
    vec2 d = abs(plocal) - vec2(len * 0.5, w); // 现在进行框的有向距离函数计算
    return length(max(d, 0.0)) + min(max(d.x, d.y), 0.0); // （外部 + 内部）
}
```

请注意，线条宽度w是在着色器空间中指定的，而非屏幕空间。绘制以像素单位指定宽度的线条需要使用自动求导。如果d是点p处有向距离函数（SDF）的计算结果，那么$length(vec2(dFdx(d), dFdy(d)))$就是p空间中一个像素的宽度。这里不适合使用其粗略近似值$fwidth(d)$，因为这会使线条宽度明显依赖于线条的方向。 

## 圆弧

圆形线段曾经是计算机图形学中必不可少的基本图形。它们仍然存在于PostScript、PDF和SVG等二维图形格式中，并且在CAD/CAM中被大量用于描述计算机控制的路由器、切割机和3D打印机的刀具路径。对于传统的二维图形，像贝塞尔线段这样的二次或三次参数曲线如今几乎已完全取代圆弧，成为绘制曲线的主要基本图形。贝塞尔线段更加灵活，在传统环境中绘制起来也不会复杂太多，但在着色器中使用它们的隐式形式要复杂得多。基于这个原因，我们来介绍圆形线段的有符号距离函数（SDF），这样我们就可以用使用直线和圆弧这种传统方法绘制相当通用的平滑曲线。 

圆弧段由一个中点、一个半径、一个起始角度和一个结束角度来描述，总共有五个自由度。通过指定半径以及起始点和结束点而不是角度，可以使描述更加方便，但这样我们就需要指定弧线是顺时针还是逆时针弯曲。这就是用于CAD/CAM的传统 “Gerber” 文件格式的做法。在着色器中，我们不指定刀具路径，这意味着弧线的方向无关紧要。我们可以假设所有弧线都是逆时针方向的，如果需要将弧线翻转到另一个方向，只需交换端点的顺序即可。

确定到圆弧最近点距离的算法是，首先判断我们是否处于圆弧的“间隙”中。在这种情况下，最近点是端点之一。如果不是，该距离与到具有相同圆心和半径的圆的距离相同。

![2505042847 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/05/05/1mg/2505042847-.png)
*Different cases to consider for a Euclidean SDF for a circle segment.*

极端情况下，我们可以选择最佳的坐标系，并为以原点为中心的圆弧制作一个有符号距离函数（SDF），该圆弧在±x方向上对称，且缺口始终朝向负y方向。Inigo Quilez为此编写了这段非常简洁的代码：
```
// SDF for a circular arc centered on the origin, with radius r and line width w.
// The input parameter “sc” is the sine and cosine of half of the gap angle.
float SDFarc( vec2 p, vec2 sc, float r, float w ) {
 p.x = abs(p.x);
 return ((sc.y*p.x > sc.x*p.y) ? length(p - sc*r) : abs(length(p) - r)) - w;
}
```

现在，尽管这是一段 “巧妙的单行代码”，但不可否认它是非常紧凑且快速的GLSL代码。然而，要让它在绘制不同位置和方向的弧线时发挥作用，还需要相当多的辅助代码，而这些额外的代码会让它显得没那么优雅，速度也会慢很多。平移和旋转看上去是简单的操作，但要确切弄清楚这些操作背后具体是什么并非是一件易事。上一节中的SDFboxline函数具备我们所需的某些某些变换，但由于上述函数仅指定了起始角、结束角和半径，要弄清楚弧线的起点和终点位置还需要额外的工作。这不难，但要更多代码。 

相反，让我们像在Gerber文件中那样绘制一条弧线，并指定三个点：圆弧段的起点、终点和圆心。这是一种合理的权衡：该函数比上面的最简代码更复杂，但使用起来会容易得多。三个二维点有六个自由度，这意味着我们的参数列表是超定的，我们需要注意将起点和终点放置在与圆心相同的距离处。Gerber文件也有同样的问题。然而，明确指定圆心而不仅仅是半径，是在着色器中节省计算量的关键。 

![2505043051 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/05/05/1mg/2505043051-.png)
*SDF function for two arcs with the same center, but start and end points swapped.*

```
// Compute the shortest distance from p to an arc with center c from p1 to p2.
// p1, p2 are in counter-clockwise order along the arc, so swapping the order
// of p1 and p2 in the function call will yield the arc’s complement.
// If p1 and p2 have different distances to c, the end of the arc will be cut off
// with a disconnected half-circle end cap at p2. Rescale v2 to avoid this.
// Author: Stefan Gustavson 2025. This code is placed in the public domain.
float DFarc(vec2 p, vec2 c, vec2 p1, vec2 p2) {
	 vec2 v1 = p1 - c;
	 vec2 v2 = p2 - c;
	 // v2 = normalize(v2)*length(v1); // Uncomment to use only the angle of p2
	 vec2 v = p - c;
	 // The signs of w.x, w.y are used to determine if we're in the gap
	 vec2 w = vec2(dot(v, -vec2(-v1.y, v1.x)), dot(v, vec2(-v2.y, v2.x)));
	 bool shortarc = (dot(v1, vec2(-v2.y, v2.x)) >= 0.0); // Arc angle <= pi
	 bvec2 wnegative = lessThan(vec2(0.0), w);
	 bool ingap = shortarc ? all(wnegative) : any(wnegative); // Is p in the gap?
	 return ingap ? min(length(p1-p), length(p2-p)) : abs(length(v) - length(v1));
}
// SDF for an arc line of width w with round ends
float SDFarc(vec2 p, vec2 c, vec2 p1, vec2 p2, float w) {
  return DFarc(p, c, p1, p2) - w;
}
```
这并非关于如何在GLSL着色器中绘制弧线的最终定论。指定一条弧线有数十种可能有用的方法，哪种方法最佳取决于具体情况。古老但仍常见的Gerber文件格式对顺时针和逆时针弧线使用不同的函数，但除此之外，其做法与我们上面所做的相同，即指定起点、中心点和终点。另一种方法是遵循PostScript和PDF的惯例，这种惯例影响了许多其他图形API，即在两条直线之间绘制一个圆角。这需要一个起点、一个终点、一个中点以及靠近中点处弧线的拐角半径，而且我们可能还需要绘制两条直线，以便将起点和终点与弧线连接起来。对于在着色器中进行此操作而言，这可能不是一个好策略，因为有相当多的计算内容未在参数中指定。弧线的中点在哪里？线段与弧线在何处连接？最好将此类计算排除在着色器之外，因为在着色器中，每帧的每个像素都必须执行这些计算，相反，将更多工作交给着色器程序员来完成，这样这些工作只需执行一次。 

## 多边形

多边形有多种类型。传统的计算机图形学几乎用三角形来表示所有物体，而三角形很容易通过到三条线性边界的最小距离来隐式表达。如果我们想要绘制一个三角形的轮廓，可以根据三条直线构建一个有符号距离函数：
```
// SDF for the outline of a triangle
float SDFlinetriangle(vec2 p, vec2 p0, vec2 p1, vec2 p2, float w) {
    float d0 = SDFline(p, p0, p1, w);
    float d1 = SDFline(p, p1, p2, w);
    float d2 = SDFline(p, p2, p0, w);
    return min(d0, min(d1, d2));
}
```

为了创建一个更通用的有向距离场（SDF），使其在三角形内部为负，外部为正，我们需要跟踪我们位于边的哪一侧，并相应地调整函数的符号。这涉及到一些行列式的运算，但并非高深莫测。在此不深入细节，我们直接给出一个GLSL函数，让代码来说明一切。下面的函数有详尽的注释：
```
// 计算向量 a 的点积的平方（即向量 a 与自身的点积），只是为了简化输入
float dot2(vec2 a) { 
    return dot(a, a); 
} 

// 计算向量 a 和向量 b 的行列式，用于判断环绕方向
float det(vec2 a, vec2 b) { 
    return a.x*b.y - a.y*b.x; 
} 

// 计算从点 p 到三角形（顶点为 p0、p1、p2）的有符号欧几里得距离
// 基于 Inigo Quilez 2014 年的代码，但为了清晰进行了大量编辑
// 在 MIT 许可证下发布：https://opensource.org/license/mit
float SDFtriangle( vec2 p, vec2 p0, vec2 p1, vec2 p2 ) {
    // 计算边 0（从 p0 到 p1）和从点 p0 到点 p 的向量 v0
    vec2 e0 = p1 - p0, v0 = p - p0; 
    // 将向量 v0 投影到边 e0 上，得到投影系数 t0
    float t0 = dot(v0, e0) / dot(e0, e0); 
    // 计算点 p 到边 e0 的平方距离 d0（通过投影后的向量差的点积的平方得到）
    float d0 = dot2(v0 - e0*clamp(t0, 0.0, 1.0)); 
    
    // 计算边 1（从 p1 到 p2）和从点 p1 到点 p 的向量 v1
    vec2 e1 = p2 - p1, v1 = p - p1; 
    // 将向量 v1 投影到边 e1 上，得到投影系数 t1
    float t1 = dot(v1, e1) / dot(e1, e1); 
    // 计算点 p 到边 e1 的平方距离 d1（通过投影后的向量差的点积的平方得到）
    float d1 = dot2(v1 - e1*clamp(t1, 0.0, 1.0)); 
    
    // 计算边 2（从 p2 到 p0）和从点 p2 到点 p 的向量 v2
    vec2 e2 = p0 - p2, v2 = p - p2; 
    // 将向量 v2 投影到边 e2 上，得到投影系数 t2
    float t2 = dot(v2, e2) / dot(e2, e2); 
    // 计算点 p 到边 e2 的平方距离 d2（通过投影后的向量差的点积的平方得到）
    float d2 = dot2(v2 - e2*clamp(t2, 0.0, 1.0)); 
    
    // w 的符号用于判断三角形是顺时针（CW）还是逆时针（CCW）环绕
    float w = det(e0, e2); 
    
    // w0、w1、w2 的符号用于判断点 p 位于对应边的哪一侧
    float w0 = det(v0, e0); 
    w0 = (w < 0.0)? -w0 : w0; 
    float w1 = det(v1, e1); 
    w1 = (w < 0.0)? -w1 : w1; 
    float w2 = det(v2, e2); 
    w2 = (w < 0.0)? -w2 : w2; 
    
    // 找出 d0、d1、d2 中的最小距离，并确定点 p 相对于最近边的位置
    vec2 d = min(min(vec2(d0, w0), vec2(d1, w1)), vec2(d2, w2)); 
    // 最后计算距离（先找出最小平方距离，最后再开方，这样可以节省两次开方运算）
    float dist = sqrt(d.x); 
    // 如果点 p 在三角形内部（对应边的符号为正），则返回负的距离；否则返回正的距离
    return d.y > 0.0? -dist : dist; 
}
```

![2505044739 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/05/05/1mg/2505044739-.png)
*Two SDF functions for a triangle: outline (left) and inside-outside area (right).*

在传统的二维和三维图形学中，多于三个角的多边形通常会被分割成三角形。这是个很棒的主意，因为它在不牺牲通用性的前提下简化了问题。精确确定那些最终不可见的内部边的位置所带来的代价，远远比不上将所有内容简化为仅处理三角形所带来的好处。然而，如果我们想通过创建有符号距离场来绘制二维多边形，将多边形分割成三角形根本行不通，因为内部边不应被视为到边界距离为零的点。此外，在着色器中，工作量与我们评估的线性边界数量成正比，因此我们应注意仅评估到外部边（即多边形轮廓）的距离。 

上述用于SDF三角形的方法可以扩展到任意多边形，只要它们是凸多边形。只需添加顶点，并对每条新增的边重复这些操作。具有任意数量顶点的多边形可以通过遍历数组来绘制。然而，请记住，着色器的工作量会随着你添加的每条边而增加。用数百个短线段来近似曲线轮廓，这在传统图形学中很常见，但对于隐式绘制来说却是个糟糕的主意。你甚至可能会遇到硬件限制，即向片段着色器发送数组中的数据量上限。目前GLSL的实现允许将几千个浮点值作为统一数据提供，但通常来说，接近这个限制或滥用纹理来为每个像素遍历大量数据都不是个好主意。 

对于非凸多边形，这个问题会变得更加复杂，但如果我们沿着轮廓依次评估所有边，我们就能追踪我们是在内部还是外部，并在循环结束时最终决定是否翻转最小距离的符号。我们这里不会给出相关代码，但你可以在 https://www.shadertoy.com/view/wdBXRW 上展示的Shadertoy示例中查看（Inigo Quilez）编写的一个巧妙函数。这个函数速度快、稳定性好且经过充分测试，但遗憾的是完全没有文档说明。

## 曲线

从直线、多边形、圆形和椭圆过渡到更一般的曲线，比如二次或三次贝塞尔曲线，这似乎是顺理成章的。然而，在着色器编程中，这些属于小众应用，尚未广泛使用。计算二次贝塞尔曲线的精确有符号距离函数（SDF）需要求解一个四次方程，类似于上面计算椭圆精确SDF的方法，而计算三次贝塞尔线段（或任何其他三次曲线）的SDF则需要求解一个五次方程，这只能通过数值方法来完成。三次曲线还有一个额外的问题，即它们可能会自相交，这就需要考虑曲线上哪个点离任意点最近的更多可能性。像我们对椭圆那样模拟一个SDF，也可以应用于其他曲线轮廓，这是一种可能有用的方法，不应轻易忽视。一个方便的数学定理表明，任何N次有理参数多项式曲线，包括B样条曲线和贝塞尔曲线，都可以表示为一个N次隐式多项式的零点。如何在两者之间转换超出了本次演示的范围，但如果你想搜索更多信息，一个关键词是（Bezout implicitization）。 


该领域具有开创性的参考文献是塞德伯格（Sederberg）、安德森（Anderson）和戈德曼（Goldman）撰写的两篇文章：《参数曲线和曲面的隐式表示》， Implicit representation of parametric curves and surfaces, Journal of Computer Vision, Graphics, and Image Processing, 1984 [https://doi.org/10.1016/0734-189X(84)90140-3], and Vector elimination: A technique for the implicitization, inversion, and intersection of planar parametric rational polynomial curves, Journal of Computer Aided Geometric Design, 1984 [https://doi.org/10.1016/0167-8396(84)90020-7]。后来，奇翁（Chionh）和塞德伯格发表了一篇后续文章：On the minors of the implicitization Bézout matrix for a rational plane curve, Journal of Computer Aided Geometric Design, 2001 [https://doi.org/10.1016/S0167-8396(00)00034-0]. (There be dragons, beware.) 


为了表明在着色器中这样做并非愚蠢之举，只是不常见而已，下面是一个着色器的输出，该着色器使用一些贝塞尔曲线的隐式形式来渲染字母“S”。着色器代码杂乱且极难读懂，它源自Renderman SL，但速度相当快，所以表明这一原理是可行的。 

![2505055342 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/05/05/1mg/2505055342-.png)
*The letter “S” displayed by implicit functions derived from a set of Béziér curves.
Source and a link to documentation is on https://www.shadertoy.com/view/ssScDR.*

我们选择在这里不详细介绍这一点，不是因为它不重要，而是因为它目前不是大多数应用程序的首选方法。一个很大的原因是，一组复杂形状的 SDF 是一个表现良好的连续函数，其梯度也大部分是连续的，并且具有单位斜率。因此，SDF 非常适合采样并存储为纹理数据，而距离场纹理确实在渲染复杂形状（如角色和装饰图案）方面越来越受欢迎。我们将在第 18 章 “Hybirds” 中回到这一点。


## 组合形
在之前的示例中，我们已经了解了如何将多个有符号距离场（SDF）合并为一个：只需从计算得到的到两个或更多子特征的距离中选取最小的距离。为了使这种方法有效，正确地相互缩放每个SDF是很重要的。创建一个SDF并不一定需要排序，通过一系列两两比较选出最小的那个就足够了。在GLSL中，min函数是硬件加速的，甚至比三元条件选择还要快。同样值得一提的是，在像GPU核心这样潜在的并行执行环境中，建议不要通过顺序搜索和更新来找到几个值中的最小值，而是通过两两比较的二叉树来实现。对于八个或更少的值，就需要执行的比较次数而言，这并没有改进，但即使是四个值，它也能更好地并行化。写成min(min(x, y), min(z, w))比min(x, min(y, min(z, w)))对硬件更 “友好”，尽管两者的操作数量相同。在第一种情况下，两个min操作可以并行执行，而在第二种情况下，它们都必须顺序执行。指望优化编译器发现这种模式并重新排列操作顺序，就目前的GLSL实现而言，充其量只是碰运气。 

最后要说明的是，由于硬件对 `min` 和 `max` 函数有直接支持，它们实际上比等效的选择语句更快：`xmin = min(x1, x2)` 目前比 `xmin = (x1 < x2)? x1 : x2` 更快。而且，我们再次强调，在进行 GLSL 编程时，只要有可能，通常应尽量避免使用 `ifelse` 语句。去掉这些语句虽然常常会降低代码的可读性，但能显著提高速度，而注释对提高可读性很有帮助。 

作为复合SDF生成形状的一个实用示例，下面是一组数字的渲染图，其中字形由弧线和直线组成。中间一行的字形是在几分钟内利用本章前面介绍的SDFcircle、SDFline和SDFarc拼凑而成的。底部一行中数字1到9的外观更为精致，设计它们花费的时间要长得多。（实际上是好几天。但我乐在其中。）

![2505055723 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/05/05/1mg/2505055723-.png)
*Text (numbers) displayed by abundant use of SDF functions.
Source code on https://www.shadertoy.com/view/fdSyRh*

该着色器会对每个渲染的字形的多个有符号距离场（SDF）评估执行多个最小值函数，并且每个数字的每个部分都要针对每个像素进行评估。这很浪费，但对于绘制少量字形是可行的。如果我们要渲染数百个字形，明智的做法是将渲染划分为多个区域，这样在每个点上只需要完整评估一个或几个字符。二叉空间划分（BSP）将是一种合适的方法。这是一种if - else类型的代码结构，而我们不建议在GLSL中使用这种结构。然而，这将是一种条件执行，相邻像素很可能执行代码的同一分支，在这种情况下，现代GPU实际上可以避免执行该区域内没有任何像素会用到的那部分着色器代码需要运行。这被称为对等感知条件执行。从硬件角度来看，它是一个单独的位，分布在核心集群中，用于检测所有核心何时就逻辑条件达成一致。这是GPU硬件中相当近期引入的一项功能，在某些情况下它可以大幅提升速度。它还使得某些情况下使用if - else语句不仅可行，而且非常值得推荐。不过要记住，加速效果取决于条件在空间上的一致性：并行处理的相邻像素对于if关键字后的表达式应该很可能产生相同的结果（真或假）。 

通过距离场绘制字符形状为额外的图形效果提供了很多选择。形状可以做得更细或更粗，边缘可以做得模糊、有噪点或发光，并且与以经典方式将边缘定义为清晰轮廓相比，对形状进行凹凸映射以使其看起来有浮雕或雕刻效果要容易得多。这是一个对这些字符形状和 “流动噪点” 进行试验的示例。

![2505055842 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/05/05/1mg/2505055842-.png)
*“Cloud computing”: text rendered by an SDF “font” and heavy use of noise.
Animated version with source code on https://www.shadertoy.com/view/sd2yWz.*
