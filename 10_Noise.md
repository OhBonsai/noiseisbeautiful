在上一章的结尾，我们得出了一个如下所示的模式：

![2505052136 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/05/05/1mg/2505052136-.png)
*Frequency synthesis using a sum of 50 random sinusoidal terms in one octave.*

在本章中，我们将介绍一种可直接生成类似图案的算法：
![2505052144 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/05/05/1mg/2505052144-.png)
*A classic 2-D “Perlin Noise” function, from https://github.com/stegu/webgl-noise.*

该算法和模式都被称为柏林噪声（Perlin Noise），以其发明者肯尼斯（“肯”）·柏林（Kenneth ("Ken") Perlin）的名字命名，他于1985年在一篇题为《“An
Image Synthesizer” [https://doi.org/10.1145/325165.325247]. 发布了这个算法

## 为什么需要噪声

为什么我们想要这种模式呢？它看起来并不是特别有趣。嗯，确实，它看起来不怎么样，但它本就不该如此。这是一个基础函数，可以作为构建各种各样模式的基石。它是一个带宽为一个倍频程的带限函数，这意味着它具有一定尺寸范围内的特征，缺少更大或更小的细节。分形图案的频率合成可以通过叠加不同比例的噪声来实现。我们无需叠加数百个正弦波，只需叠加几个噪声函数。我们失去了对每个单独项的相位、方向和幅度的精细控制，但在许多应用中，并不需要这种程度的控制。当柏林噪声以 “FBM” 风格用于伪分形叠加时，最终结果在视觉上与使用单个正弦波进行合成的结果难以区分。 

![2505052424 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/05/05/1mg/2505052424-.png)
*Fake “FBM” fractal sums. Left: five terms of Perlin noise. Right: 500 sine waves.*

事实上，柏林算法生成的 “伪造” 带状限制图案，在生成图案方面表现得更好。它的振幅受到限制，所以不会像 “真正的” 频谱合成那样，偶尔出现强烈的局部最小值或最大值。它本身看起来没那么随机，但也不需要那么随机 —— 它看起来足够随机了。柏林噪声函数，或其变体，一直是计算机图形行业中独一无二的主力工具，几乎在任何计算机图形制作中都被大量使用。实际上，肯·柏林（Ken Perlin）因其在技术上的成就，获得了奥斯卡金像奖（一座奥斯卡小金人）。 

在后续我们展示了一系列通过各种方式操控噪声生成的图案，以及生成这些图案的GLSL代码。此处的目的是展示原理，而非呈现照片般的真实感，但请注意，逼真渲染广泛使用噪声来生成污垢、磨损、瑕疵和粗糙度——简而言之，就是让图像看起来可信。 

![2505051750 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/05/05/1mg/2505051750-.jpg)

*Example shaders with Perlin noise. The code uses functions that we have not yet
defined, and most of it is insufficiently commented. Consider this a “teaser”.*

## 做噪声


<div style="text-align:center; background-color:#d6e977; padding:10px;color:black;font-weight:bolder">
“It is a rare mind indeed that can render the hitherto nonexistent
blindingly obvious. The cry ‘I could have thought of that!’
is a very popular and misleading one, for the fact is they didn’t.”
<br/>
</div>
<div style="text-align:center; background-color:#d6e977; padding:10px;color:black;font-weight:light">From “Dirk Gently’s Holistic Detective Agency” by Douglas Adams</div>

和许多伟大的想法一样，噪声算法在向你解释过后，看似简单，实则不然。我们想要模拟一个带限信号，所以让我们看看一个八度内的正弦波之和是什么样子。
![2505052846 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/05/05/1mg/2505052846-.png)
*1-D signal with a bandwidth of one octave (sum of 100 random-phase sine waves).*

### 1-D 噪声
柏林噪声通过一种相当简单粗暴但却行之有效的近似方法来模拟这种普遍现象：它（1）使函数图像在固定间隔过x轴，（2）在x轴 即0 点处为曲线赋予伪随机的正斜率或负斜率，（3）将函数平滑地插值到所有中间点。为了简化算法，零交叉点设置在整数点上。当两个连续的零交叉点斜率符号相反时，它们之间就会产生一个“blob”。当它们的斜率符号相同时，它们之间就会额外产生一个零交叉点和两个“blob”。这基本上就是我们想要的那种函数。 

![2505053007 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/05/05/1mg/2505053007-.png)

对于两个整数i和i + 1之间的部分，1985年柏林（Perlin）提出的经典插值方案是使用以下公式。请记住，该函数在区间的任一端点处的值都设为零。

首先，我们在端点处赋予伪随机斜率$g_i$和$g_{i + 1}$。然后我们进行外推，将这些斜率扩展为$n_i$和$n_{i + 1}$的值，就好像曲线会从任意一个端点处一直笔直地延伸下去一样。将当前点表示为$x = i + t$，其中$0\leq t\lt1$，我们可以写出：
$$n_i = tg_i$$
$$n_{i + 1} = (t - 1)g_{i + 1}$$
请注意，$n_{i + 1}$的因子$(t - 1)$是负的，因为我们是从$i + 1$到$x$沿负方向进行外推的。然后，我们通过一个在$t = 0$和$t = 1$处斜率为零的三次（三阶）插值多项式将这两个值混合在一起：
$$f = (3t^2 - 2t^3) = (3 - 2t)t^2$$
$$noise(i + t) = (1 - f)n_i + fn_{i + 1}$$
还记得第4章中的平滑过渡函数（smoothstep函数）吗？这就是同一条曲线。

![2505053018 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/05/05/1mg/2505053018-.png)
*The principle behind Perlin noise in one dimension.*



计算我们为了算出$noise(x)$的值所需要执行的算术运算的总数时，我们发现我们只进行了一次向下取整运算来计算$i = \lfloor x\rfloor$，八次乘法运算和几次减法运算。在这个总数的基础上，我们还需要加上为两个端点选择伪随机斜率所需的运算。端点。如果我们使用一个二次模N置换多项式（见第8章）作为哈希函数，那么对于每个端点需要进行四次乘法、一次加法和两次向下取整运算。然后，我们需要将范围[0, N - 1]内的一个随机整数映射到比如说[-1, 1]范围内的一个斜率，这至少还需要一次乘法和一次减法，所以梯度选择实际上比插值运算的工作量更大。 

这里的重点不是计算操作的确切次数，而是要指出可以用非常合理的操作次数来计算噪声——比计算哪怕是一次对正弦函数的较好近似所需的操作次数还要少。这是噪声函数的一个很大优势：它很容易计算！ 

“柏林噪声”（Perlin noise）的一个同义词是“梯度噪声”（gradient noise），这是个不错的名字，因为它说明了算法的一些特性，而不仅仅是命名其作者。许多用心但消息不准确的作者对“柏林噪声”究竟是什么意思存在很多误解（它的意思就是我们刚刚告诉你的，没有别的意思），所以从现在起，我们将尽量使用“梯度噪声”这个术语。（尽量，但也可能做不到。） 

即使是一维柏林噪声（抱歉，梯度噪声）也是有用的，因为它可以应用于动画。位置、旋转以及许多其他需要以伪随机方式进行动画处理的参数，往往得益于具有准周期性：既不过于规则，也不过于随机，并且变化平滑，没有顿挫或跳跃。一维梯度噪声很好地满足了这一要求，而且一维噪声为无数动画师在制作次要动画时节省了大量枯燥的关键帧制作时间。然而，为了使其在图案生成中发挥更大作用，我们需要将该算法扩展到二维。 

### 2维噪音
对于二维噪声，我们遵循一般思路，为 (x, y) 平面上整数点构成的正方形网格分配伪随机梯度，并将函数值设为零。梯度有大小和方向，这两者我们都可以改变。然而，为了保持图案中的局部对比度，我们选择将梯度的大小保持在合理的较大值，从而避免出现大面积值接近零的平坦区域。现在，插值在二维空间中进行，我们选择一次在一个维度上进行。顺序是任意的，但我们先沿 x 方向进行，然后沿 y 方向。 

在方程中对所有内容都使用通用索引会妨碍解释，所以我们只看(x, y)网格中的一个单位正方形，并给四个角取局部名称：
$$
\overline{p}_{00}=(x_0,y_0)=(\text{floor}(x),\text{floor}(y))
$$
$$
\overline{p}_{10}=(x_0 + 1,y_0)\\
\overline{p}_{01}=(x_0,y_0 + 1)\\
\overline{p}_{11}=(x_0 + 1,y_0 + 1)
$$

然后，我们需要为每个角点伪随机地分配二维向量作为梯度。我们将这些向量命名为$\overline{g}_{00}$、$\overline{g}_{10}$、$\overline{g}_{01}$、$\overline{g}_{11}$。不同的作者提出了不同的选择这些梯度的策略。柏林噪声的原始版本建议使用范围在[-1,1]内的随机分量来生成一个二维向量，但这会产生一个噪声场，其局部对比度变化强烈得令人不适。这引入了低频成分，使得整个函数的频带限制不足，呈现出斑驳的外观。另一种变体是从一小部分方向均匀分布且幅度变化很小的向量中随机选择。一种特别简单且效果良好的方法是选择长度相同、仅方向不同的梯度。无论如何选择梯度，后续的外推和插值阶段与一维情况类似，但增加了复杂性，因为插值现在是二维向量$(s, t)$，并且线性外推应沿梯度方向进行。这通过对四个角点中的每一个进行一次标量积来实现。 
$$
\begin{align*}
s &= x - \lfloor x \rfloor = \text{fract}(x)\\
t &= y - \lfloor y \rfloor = \text{fract}(y)\\
\overline{v}_{00} &= (s, t)\\
\overline{v}_{10} &= (s - 1, t)\\
\overline{v}_{01} &= (s, t - 1)\\
\overline{v}_{11} &= (s - 1, t - 1)\\
n_{00} &= \overline{g}_{00} \cdot \overline{v}_{10}\\
n_{10} &= \overline{g}_{10} \cdot \overline{v}_{10}\\
n_{01} &= \overline{g}_{00} \cdot \overline{v}_{00}\\
n_{11} &= \overline{g}_{00} \cdot \overline{v}_{00}\\
f_x &= (3s^2 - 2s^3) = (3 - 2s)s^2\\
f_y &= (3t^2 - 2t^3) = (3 - 2t)t^2\\
n_0 &= (1 - f_x)n_{00} + f_x n_{10}\\
n_1 &= (1 - f_x)n_{01} + f_x n_{11}\\
\text{noise}(x, y) &= (1 - f_y)n_0 + f_y n_1
\end{align*}
$$


![2505053029 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/05/05/1mg/2505053029-.png)
*Gradient (Perlin) noise in 2-D: extrapolations from four corners to ¯p=( x , y).*

显然，这比一维情况的工作量更大，但与直接频率合成相比，仍然微不足道。乘法运算是这里最繁琐的操作，上述算法中大约有20次标量乘法。这还不包括选择四个伪随机二维梯度所需的运算，而这会使总工作量增加大约相同的量。生成一个二维噪声值可能需要进行40到50次乘法运算。

与计算50个正弦函数相比，工作量要少得多。计算50个正弦函数时，每个函数不仅需要进行乘法运算，还需要为每个函数生成并添加一个伪随机相位，而且自变量各不相同。即便我们依靠硬件加速来计算正弦函数，梯度噪声算法仍具有巨大的速度优势。

### 幻梦：硬件加速的噪声

如果我们至少有一些用于生成噪波的硬件支持，比如像OSL中的cellnoise那样优秀的内置哈希函数，那么噪波生成就会轻而易举。这里有个奇怪的细节，GLSL规范包含了几个旨在生成梯度噪波的函数，根据你想要生成的噪波分量数量，分别命名为noise1到noise4，并且每个函数都定义了以1维到4维向量作为参数。然而，只有一家GPU制造商3DLabs选择实现这些函数。它们的速度从来都不是特别快，而且在GLSL推出后不久，3DLabs在2006年就停止了GPU设计。目前所有的GPU制造商都将这些函数列为“未实现”。GLSL语法允许调用这些函数，但令人失望的是，它们都返回零。 

在GLSL早期，人们曾认真努力就实现哪个版本的噪声函数达成共识，但讨论陷入了犹豫不决的境地。此外，所考虑算法的专利声明问题实际上阻止了为在GLSL中实现内置噪声函数所做的进一步努力。

然而，没必要为此落泪，因为如今GPU的性能足以让我们实现自己选择的噪波算法，并在着色器中大量使用。对着色器实现进行细致的控制，不仅能使着色器模式具有可移植性，还为着色器程序员提供了选择使用何种算法的空间，他们可以对算法进行调整，在速度和质量之间取得平衡。 

### 示例实现

下面给出了一个二维梯度噪声的GLSL实现，虽有些年头，但依然非常实用。代码量不多。其中一些为了追求速度，写得几乎难以读懂，但总体结构正是上述公式中呈现的算法，你应该能够理解。 
```
// GLSL classic 2D gradient noise (“Perlin noise”)
// Copyright (c) 2011, 2024 Stefan Gustavson
// with thanks to Ian McEwan for several details.
// Distributed under the permissive MIT license:
// https://github.com/stegu/webgl-noise
// Please give credit, and please keep this header.

// Different permutation polynomials for x and y to avoid artifacts
// 为 x 和 y 使用不同的置换多项式以避免瑕疵
vec4 hash(vec4 ix, vec4 iy) {
    // 计算 51x^2 + 2x 并对 289 取模
    vec4 h = mod((ix*51.0 + 2.0)*ix + iy, 289.0); 
    // 计算 34x^2 + 10x 并对 289 取模
    return mod((h*34.0 + 10.0)*h, 289.0); 
}

// A fifth degree interpolating function to replace the cubic interpolation
// 一个五阶插值函数，用于替代三阶插值
vec2 fade(vec2 t) {
    // 计算 ((6.0*t - 15.0)*t + 10.0)*t*t*t ，据说比 (3.0 - 2.0*t)*t*t 更好
    return ((6.0*t-15.0)*t+10.0)*t*t*t; 
}

// Classic 2-D “Perlin noise”
// 经典二维“柏林噪声”
float cnoise(vec2 P) {
    // 将输入向量 P 的 xy 分量重复两次，然后对其进行向下取整，并分别加上 0, 0, 1, 1 ，得到四个角点的整数坐标
    vec4 Pi = floor(P.xyxy) + vec4(0.0, 0.0, 1.0, 1.0); 
    // 将输入向量 P 的 xy 分量重复两次，然后取其小数部分，并分别减去 0, 0, 1, 1 ，得到四个角点的小数偏移量
    vec4 Pf = fract(P.xyxy) - vec4(0.0, 0.0, 1.0, 1.0); 
    // 对整数坐标取模 289，以避免在哈希函数中出现截断
    Pi = mod(Pi, 289.0); 
    // 提取整数坐标的 x 分量和重复的 x 分量，形成新的向量
    vec4 ix = Pi.xzxz; 
    // 提取整数坐标的 y 分量和重复的 y 分量，形成新的向量
    vec4 iy = Pi.yyww; 
    // 提取小数偏移量的 x 分量和重复的 x 分量，形成新的向量
    vec4 fx = Pf.xzxz; 
    // 提取小数偏移量的 y 分量和重复的 y 分量，形成新的向量
    vec4 fy = Pf.yyww; 

    // 为四个角点生成哈希值
    vec4 i = hash(ix, iy); 

    // 沿着菱形形状生成梯度点
    vec4 gx = fract(i / 41.0) * 2.0 - 1.0 ; 
    // 计算梯度点 y 分量的绝对值并减去 0.5
    vec4 gy = abs(gx) - 0.5 ; 
    // 对梯度点 x 分量加上 0.5 后向下取整
    vec4 tx = floor(gx + 0.5); 
    // 得到梯度点 x 分量的小数部分
    gx = gx - tx; 

    // 重新排列分量以创建四个梯度向量
    vec2 g00 = vec2(gx.x, gy.x); 
    vec2 g10 = vec2(gx.y, gy.y); 
    vec2 g01 = vec2(gx.z, gy.z); 
    vec2 g11 = vec2(gx.w, gy.w); 

    // 计算缩放因子，使梯度向量长度相等
    vec4 norm = inversesqrt(vec4(dot(g00, g00), dot(g10, g10),
                                dot(g01, g01), dot(g11, g11))); 

    // 计算每个角点的外推贡献值
    float n00 = norm.x * dot(g00, vec2(fx.x, fy.x)); 
    float n10 = norm.y * dot(g10, vec2(fx.y, fy.y)); 
    float n01 = norm.z * dot(g01, vec2(fx.z, fy.z)); 
    float n11 = norm.w * dot(g11, vec2(fx.w, fy.w)); 

    // 计算五阶插值因子
    vec2 fade_xy = fade(Pf.xy); 
    // 首先沿着 x 方向进行插值
    vec2 n_x = mix(vec2(n00, n01), vec2(n10, n11), fade_xy.x); 
    // 然后沿着 y 方向进行插值
    float n = mix(n_x.x, n_x.y, fade_xy.y); 

    // 经验因子，将输出缩放到 [-1, 1] 区间
    return 1.58 * n; 
}
```

上述实现是2012年发布代码的现代化版本[Journal of Graphics Tools, https://doi.org/10.1080/2151237X.2012.649621].。当时普遍存在的一些针对不太智能的编译器的“手把手”式内容已被移除，但代码在很大程度上仍与2012年相同，它能正常运行，而且速度很快。二十年来，GLSL一直是一个非常稳定的开发平台。该语言增加了许多有用的内容，但旧代码仍然可以正常运行。 

请注意，这里的插值并非使用前文给出公式中的三次多项式，而是采用了五次插值，这在仅增加少量开销的情况下就能带来显著的提升。下一章将详细介绍这一点——我们只是不想在示例中给出糟糕的代码。你可以随意将上述噪声函数用于自己的实验。头部提到的 “MIT 许可证” 是一种非常宽松的许可证，其基本含义是 “可免费用于任何目的，但请注明出处”。只要不删除头部信息，就没问题。 

### 3维噪声

噪声算法也可以扩展到三维空间。三维函数可用于超纹理，描述诸如烟雾、云层或雾气的局部密度，或者像海绵状多孔材料的结构，但它最常见的应用是，对于形状复杂的物体，或者确实是从具有三维结构的材料（如木材或大理石）中雕刻出来的物体，无需进行显式的二维表面映射。我们无需为表面上的每个点分配二维坐标，只需使用表面点的三维空间坐标，然后让三维程序纹理F(x, y, z)来决定每个点的外观。三维纹理可以为形状复杂的物体的表面映射节省大量工作。三维纹理还可以避免纹理接缝、不均匀缩放和挤压问题。对于三维纹理而言，三维噪声是一种特别通用的函数。 
![2505055400 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/05/05/1mg/2505055400-.png)
*Left: 2-D texture mapping of a sphere, showing stretching, pinching and a
texture seam. Right: 3-D mapping, showing none of those problems.*

从一维到二维的扩展相当顺利，而向三维的扩展也非常简单直接：
![2505055432 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/05/05/1mg/2505055432-.png)

插值仍然是一次在一个维度上进行，先沿x方向进行四次，接着沿y方向进行两次，最后沿z方向进行一次，以计算最终的噪声值。
![2505050044 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/05/05/1mg/2505050044-.png)
*Successive interpolations in 3-D by the classic Perlin noise algorithm: first along x
(orange circles), then along y (green circles) and finally along z (black circle).*

所有计算都以与之前相同的方式进行，但现在计算量要大得多。一个立方体有$2^3=8$个角。 八个点积使用具有3个分量的梯度向量来执行，每个分量也都需要生成。三维噪声的复杂度仍然不算大问题——上述方程指定了大约45次标量乘法——但我们可能希望进一步扩展它。创建动画三维噪声的一种直接方法是使用四维噪声函数，并使第四个坐标依赖于时间：noise(x, y, z, t)。这并不总是个好主意，但这是一种实现方式。然而，对于我们添加的每个维度，经典噪声算法所需的工作量都会增加一倍以上。这并不理想，并且对于生成更高维度的噪声以及动画噪声，还有更好的策略。下一章将对此进行更多介绍。
