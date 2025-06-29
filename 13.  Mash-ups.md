
至此我们已聚焦于基础函数——程序化着色器的构建模块。现在我们将运用这些工具，以特定视觉效果为目标创建更复杂的图案。所有示例都遵循自上而下的设计流程：从核心特征开始，逐步完善细节。每个案例都会组合使用前文介绍的工具。我们并非宣称这些着色器是最佳实现方案，重点在于用结构清晰且效率尚可的代码呈现目标图案。

## 星形图案
以Pixar早期短片《Luxo Jr.》中小球上的五角星为例。通过判断当前点与五条旋转直线的位置关系，可高效绘制该星形。

![2506244519 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/06/29/1mg/2506244519-.png)
*左：** 圆形内的五角星 **右：** 点与五条边界线关系的可视化（黑色到白色表示点位于0/1/2/3条线外侧）*

这段Shader代码是绘制上图直接且有效的方法。

```glsl
float assign(float a, float x){
    float w = 0.7*width(x-a);
    return smoothstep(w, w, x-a);
}

vec3 stappattern(vec2 w){
    // 这些是可以在编译时计算的常量
    const float twoP1 = 6.283185307;
    const float c15 = cos(btwoP45.0);
    const float s15 = sin(btwoP45.0);
    const maxi for maxi(c15, s15, s15, c15);
    vec2 uvrot = uvr;
    float outside = 0.0; // 该点“位于”多少条线之外
    for(float i = 0.0; i < 5.0; i++) {
    outside += assign(0.2, uvrot); // 如果在外侧则增加
    uvrot = R5*uvrot; // 最后一次迭代中无需此操作
    }
    vec3 col = vec3(0.0.0.3.0.8); // 蓝色背景
    float circle = assign(0.9, length(uv)); 
    col = mix(vec3(1.0.1.0.0.0), col, circle); // 黄色圆圈
    float star = clamp(outside-1.0, 0.0, 1.0);
    return mix(vec3(1.0.0.0.0), col, star); // 红色星星
}
```

注意星星区域的特征是：要么位于所有线性边界内（中心五边形），要么仅位于其中一个边界外（星星的尖端）。我们还通过使用**assign**计算每个线性边界使Shader抗锯齿化，并注意通过使用**clamp**而非**step**来保留最终的混合量相关中间值。

观察代码可以发现，最后的矩阵乘法是无用的，因为五次旋转会回到原点，除了舍入误差。编译器可能会展开这个固定长度的短循环，并消除最后的矩阵乘法，因为其结果从未被使用。即使没有发生这种情况，五次操作中一次不必要的操作也不是大问题，因此上述代码作为解决方案是完全可接受的。然而，它可以被显著加速。

对于星星部分，我们计算二维位置向量的旋转，但仅使用y分量进行图案生成。通过手动展开短循环并预计算一些额外的常量（**const float**表达式在编译时求值），我们可以减少计算星星图案所需乘法次数的一半以上，这可能值得付出努力。副作用是我们避免了增量旋转。这些增量旋转在数值上是不稳定的，因为每一步都会引入累积的舍入误差。在此特定情况下，舍入误差非常小且对应用程序完全无关紧要，但正确实现并无害处。

以下代码以显著减少的工作量生成完全相同的图案：

```glsl
vec3 stappattern(vec2 w){
    const float twoP1 = 6.283185307;
    const float c15 = cos(btwoP45.0);
    const float s15 = sin(btwoP45.0);
    const float c25 = cos(2-0*ww(P45.0);
    const float s25 = sin(2-0*ww(P45.0);
    const float c25 = c25; // $cos(3-0*ww(P45.0))$
    const float s35 = -s25; // $sin(3-0*ww(P45.0))$
    const float c45 = c15; // $cos(4-0*ww(P45.0))$
    const float s45 = -s15; // $sin(4-0*ww(P45.0))$
    float outside = assign(0.2, uv); // 白色
    outside += assign(0.2, dot(vec2/c15, c15), uv); // 白色
    outside += assign(0.2, dot(vec2/c25, c25), uv); // 白色
    outside += assign(0.2, dot(vec2/c35, c35), uv); // 黑色
    outside += assign(0.2, dot(vec2/c45, c45), uv); // 黑色
    void col = vec3(0.0.3.0.8); // 蓝色背景
    float circle = assign(0.9, length(uv)); 
    col = mix(vec3(1.0.1.0.0.0), col, circle); // 黄色圆圈
    float star = clamp(outside-1.0, 0.0, 1.0);
    return mix(vec3(1.0.0.0.0), col, star); // 红色星星
}
```

现在，如果我们观察**dot**表达式中的分量乘法，可以通过重用一些中间结果将乘法次数再次减半。然而，这可能完全不会加速执行，因为**dot**函数在硬件上被加速，可以一次性完成乘加操作。为了消除少量乘法（具体为四次）而付出更多努力，可能只会降低代码的可读性。

当然，根据具体情况，即使我们已经为加速付出的少量编码努力也可能不值得。未优化的Shader已经非常简单，对现代GPU来说完全不会造成负担。

## 大理石瓷砖

我们展示了各种类型的平铺，但并未真正将其用于创建图案。“平铺”一词的日常字面意思是用某种材料的瓷砖覆盖表面，比如

![2506244558 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/06/29/1mg/2506244558-.png)
**Ken Perlin风格的大理石纹路**

几个世纪以来一直深受建筑师和雕塑家喜爱的材料。完全可以在Shader中很好地复制其外观，但这需要大量努力。与其追求完美，不如保持简单，仅使用合理的大理石状噪声图案。通过精心设计的灰度映射对多个噪声分量进行分形求和，可以创建类似的效果。我们在第10章开头展示过这个例子：

![2506244612 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/06/29/1mg/2506244612-.png)
*一个合理的大理石状图案，向Ken Perlin致敬。*

单独展示时，这个图案并不十分逼真，但当我们将其分割成小块时，希望它能足够好。如果不够好，我们随时可以回头改进。Shader编程是一个迭代的过程。

为了从中制作“瓷砖”，我们需要在瓷砖边界处创建图案的不连续性。一个简单的方法是在整数边界处突然改变纹理坐标的伪随机偏移。这是*cellnoise*函数的完美任务。它不是GLSL的内置函数，但可以轻松实现一个足够快的版本，例如：
```
// Generate hash values in the range [0,1], constant across each integer cell
float cellnoise(vec2 p) {
	p = mod(floor(p), 289.0);
	float hash = mod((p.x*34.0 + 10.0)*p.x + p.y, 289.0);
	return mod((hash*34.0 + 10.0)*hash, 289.0) / 289.0;
}
```

为每个 tile增加相应随机的 offset, 我们得到了独特瓷砖效果

![2506244632 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/06/29/1mg/2506244632-.png)
*添加个体差异后的瓷砖*



所有纹理仍然朝同一方向，看起来有些人工。我们可以为每个瓷砖添加单独的旋转，不仅使用x分量作为最终的sin输入，而是使用与伪随机方向向量的标量积。我们可以使用相同的*cellnoise*哈希值来创建方向向量，因为人眼不会注意到坐标偏移和坐标旋转之间的相关性。同时，我们使用相同的哈希值来创建一些比例变化。这些变化共同作用，有效打破了图案的规律性，并创造了合理的大理石瓷砖效果。基础图案可以更好，但如果我们不追求与真实大理石的视觉匹配，这已经足够实用。如果想改进，平铺和瓷砖之间的变化方式不依赖于具体图案——它们只是纹理坐标的操作。因此，我们可以用任何其他程序化图案替换当前略显粗糙的仿大理石效果。最后用深灰色网格线遮盖接缝，形成完整的大理石瓷砖效果。

![2506294910 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/06/29/1mg/2506294910-.png)
*Adding more individual variation to the tiles.*


这看起来已经足够好。虽然不是照片级真实，但绝对具有瓷砖大理石表面的视觉特征。让我们在瓷砖之间添加深灰色缝隙，并认为任务完成。通过在图案上混合窄网格线来创建缝隙，是隐藏瓷砖接缝的便捷方法。尖锐的不连续性难以抗锯齿处理，因为这需要我们同时评估边缘附近两个瓷砖的图案以混合它们，以及角落附近四个瓷砖的图案。当然，这是可能的，也不太难，但需要大量额外代码。我们选择简单的方法，直接在接缝上叠加网格。网格线可以使用第5章的**aagridlines**函数生成，这样除了极远的视角外，抗锯齿问题就解决了。

![2506244702 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/06/29/1mg/2506244702-.png)
*我们最终对大理石瓷砖的半真半假的模仿。*


## 斑点、污渍与飞溅效果


液体飞溅形成的斑点和污渍能创造出有趣的图案，在着色器中重现这种效果既具教育意义又充满乐趣。真实的污渍形状差异极大，取决于液体撞击表面时的速度、液体量、密度和粘度等物理特性，当然还有表面的物理特性。

![2506295710 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/06/29/1mg/2506295710-.png)
*一些现实中的墨水污渍（或者，如果你愿意，血迹飞溅）。*

这里还有很强的随机性元素。如果要渲染多个这样的污渍，每个都应该独一无二。传统的基于图像的纹理方法使用少量固定图案，并通过旋转和缩放来掩盖表面上只有几种不同形状被复制的事实。虽然这种方法可以做得相当不错，但这正是程序化生成的理想任务。我们可以让每个斑点都不同，还可以为着色器的最终用户提供参数来控制斑点的许多特征：大小、混乱程度、流动性、粘稠度等等。程序化图案甚至可以动画化，展示液滴撞击时的飞溅效果，并按我们喜欢的方式让液体流动、沉淀或蒸发。这里我们不会尝试如此通用化，而是专注于重现【这是一张图片】中左上和右侧可见的特定类型斑点的静态图像——那些看起来像油漆飞溅或暴力游戏中血迹的斑点。（是的，我玩这种游戏。偶尔。出于教育目的。咳咳。）我们还将专注于一个斑点。可以通过散射函数放置更多斑点，但我们不会这样做。在虚拟环境中动态放置斑点通常通过在半透明平面对象（贴花）上放置每个斑点来实现，因此一个能生成单个斑点的着色器很可能非常有用。

我们可以从一个圆开始，通过在阈值化前向半径添加噪声来扭曲其轮廓：

| 代码 | 效果 |
|------|------|
| `float r = length(u);`<br>`float n = noise(12.0*vw);`<br>`n += 0.5 * noise(24.0*vw);`<br>`float spot = aastep(0.3, t + 0.1*n);` | ![2506290548 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/06/29/1mg/2506290548-.png)*一个被噪声扭曲的圆。* **最右侧：** *添加了不同量的噪声。* |

这是个好的开始，但还不是我们想要的效果。添加更多噪声会使斑点分裂并在边缘产生大量飞溅，但我们想要的是一个基本连通的区域，边缘有较长的径向条纹。一个好方法是在极坐标$(r,\phi)$而非笛卡尔坐标$(u,v)$中评估我们的**噪声**。通过这种方式，我们可以缩放$r$将噪声拉伸成更长的径向条纹，从而更符合我们想要的方式扭曲圆的轮廓。

为了避免噪声图案中出现难看的间断接缝，应确保将$\phi$缩放为$N/2\pi$（其中$N$为整数），并使用允许我们在$\phi$方向指定周期$N$的噪声函数。为了与抗锯齿良好配合，还应"重映射"这种"极坐标噪声"，使其在原点处为零，否则自动导数在那里会完全失控。

| 代码 | 效果 |
|------|------|
| `float phi = atan(nox, uv.y)/6.2831852;`<br>`float r = length(u);`<br>`float n1 = gnose(vec2(r*50, phi*32.0), vec2(0.0, 32.0)); // phi方向周期性`<br>`float n2 = 0.5*gnose(vec2(r*10.0, phi*64.0), vec2(0.0, 64.0)); // 双倍频率`<br>`float n = n1 + n2;`<br>`n *= smoothstep(0.1, 0.15, r); // 掩盖原点`<br>`float spot = aastep(0.3, r + 0.1*n);` |![2506290611 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/06/29/1mg/2506290611-.png) *更好的飞溅效果尝试，带有更长的径向条纹。* |

这仍然不够完美。我们想在边缘更远处添加一些不连续的飞溅，并希望边缘向外而非向内扭曲。实现这两个目标的简单方法是对第二项噪声值取平方。这既能使结果为正，又能使其值分布偏向低端。如果增加其权重，将偶尔产生强烈扭曲，在远处形成飞溅，但总体上扭曲更温和，且没有强烈的向内扭曲。

最后，我们向函数添加"种子"值，使每个污渍都独一无二。最简单的方法是向噪声坐标添加偏移。

| 代码 | 效果 |
|------|------|
| `float phi = atan(nox, uv.y)/6.2831852;`<br>`float r = length(u);`<br>`float seed = 117.0; // 改变外观`<br>`float n1 = gnose(vec2(r*5.0+seed, phi*32.0), vec2(0.0, 32.0)); // phi方向周期性`<br>`float n2 = n1 + 0.5*gnose(vec2(r*10.0+seed, phi*64.0), vec2(0.0, 64.0));`<br>`float n = n1 + 2.0*n2*n2; // n2是"向外"的`<br>`n *= smoothstep(0.1, 0.15, r); // 掩盖原点`<br>`float spot = aastep(0.3, r + 0.1*n);` | ![2506290647 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/06/29/1mg/2506290647-.png)*一个相当不错的飞溅污渍。* **底部：** *不同种子值的效果。* |

注意，我们已经定制了噪声图案$n$来扭曲这个特定形状。改变形状（例如改变圆的半径）会使圆轮廓在噪声图案上移动，从而改变噪声扭曲它的方式。虽然这不一定是坏事，但会使着色器难以控制。这个图案有些*不可预测*：改变任何参数都可能产生意外和不想要的结果。这不是混沌的，因为参数的微小变化会导致结果的微小变化，但不明显哪个参数控制图案的哪个方面，而且大多数参数会同时影响多个视觉特征。要使这段着色器代码更通用，需要提供更简单的方法来制作不同大小和扭曲程度的飞溅。它还将受益于命名参数，分别控制点的大小、噪声的基础频率和飞溅量。

图案仍然存在缺陷。污渍的基本形状是一个完美的圆，我们应该使用一些低频噪声来改变这一点。然而，考虑到一开始设定的限制，我们的工作已经完成。图案不完美，代码结构也不完美，但找出它到底哪里不对以及如何修复需要大量工作。此外，添加什么取决于我们想如何使用着色器。就本次演示而言，这是一个合适的停止点。如果你愿意，可以自行进一步实验。



在结束这个例子之前，让我们也借此机会做一件在程序化着色器开发中很常见的事：*从错误中汲取灵感*。我们第一次尝试的效果不适合这个应用，但适合模拟纸上印刷的半色调网点的一般外观。以这种风格在网格中重复的点类似于印刷品的显微图像。

![2506290726 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/06/29/1mg/2506290726-.png)
*印刷半色调网点近距离视图的合格模仿*

我们可以全力以赴模拟四色印刷、纸张的不均匀纹理和网点的不同透明度，但我们不会这么做。好吧，实际上我们会，但要等到第18章。

## 龟裂与剥落效果

龟裂(Craguelure)是一个专业术语，指表层材料因缓慢收缩或基层膨胀时形成的典型裂纹图案。表层受力后会产生特定方式的破裂：

![2506291123 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/06/29/1mg/2506291123-.png)
*开裂的油漆、开裂的覆膜箔片、干裂的泥土、龟裂的干燥土壤。*


注意大多数案例都存在两级裂纹：主裂纹首先形成，随着过程持续，主裂纹之间的区域也会受力并产生更细小的次级裂纹。

该图案类似于第12章（第169页列表）中用**cellular**函数生成的Voronoi图案。我们将使用该实现，但通过额外技巧从适度抖动中提取更多变化。不同于在方形区域内均匀分布特征点，我们会将所有点推到方形边界上。实现方式是将函数中的以下代码：

```glsl
float hashxy = permute289(hashx + Pi.y + iy);
float ox = fract(hashxy / 7.0) - 0.5;
float oy = mod(floor(hashxy / 7.0), 7.0) / 7.0 - 0.5;
float dx = Pf.x + ix + jitter*ox;
float dy = Pf.y + iy + jitter*oy;
float d = dx*dx + dy*dy;
```

替换为：

```glsl
float hashxy = permute289(hashx + Pi.y + iy);
float psi = hashxy*0.07482; // 缩放哈希值（并进一步打乱）
float ox = cos(psi); // 现代GPU计算sin/cos通常很快
float oy = sin(psi);
float sqc = jitter * 0.5 / max(abs(ox), abs(oy));
ox = ox * sqc; // "将圆形变为方形"
oy = oy * sqc; // （将点推到方形边界）
float dx = Pf.x + ix + ox;
float dy = Pf.y + iy + oy;
float d = dx*dx + dy*dy;
```

虽然可以不使用**sin**和**cos**在方形上生成点，但现代GPU通常能快速计算这些函数。此外，**cellular**函数本身就不算特别快，优化这点对整体影响不大。

让我们从用修改后的函数生成"Voronoi线"图案开始。为了更贴近现实（称其为"真实感"有些牵强），我们添加了坐标扭曲。本可以使用常规噪声实现，但我们选择完全使用**cellular**函数。这使Voronoi线呈现更多带尖角的锯齿状，我们主观认为这正是本例所需。

### 龟裂效果着色器实现

以下是龟裂(craquelure)效果的着色器实现过程，展示了从基础到进阶的演化步骤：

<table>
<tr>
<th style="width:50%">代码实现</th>
<th style="width:50%">效果展示</th>
</tr>
<tr>
<td>
<pre><code>vec2 Fa, Fb, P1a, P1b;
float ID1a, ID1b;
// 主裂纹的锯齿效果
Fa = cellular(uv*32.0, 0.7, P1a, ID1a);
// 主裂纹：用Fa.xy进行扭曲
Fb = cellular(uv*16.0 + Fa*0.15, 0.7, P1b, ID1b);
// 主裂纹图案
craquelure = aastep(0.1, Fb.y-Fb.x);</code></pre>
</td>
<td>
<img src="2506291216-.png"/>
<em>龟裂效果的初次尝试</em>
</td>
</tr>
</table>

仅用三行代码就能实现不错的效果，但裂纹宽度几乎完全一致。在第12章我们花了很大精力制作等宽的Voronoi线，但这里我们需要更多裂纹宽度的变化。通过添加基于cellular函数的扰动来实现：

<table>
<tr>
<th style="width:50%">代码实现</th>
<th style="width:50%">效果展示</th>
</tr>
<tr>
<td>
<pre><code>vec2 Fa, Fb, Fc, P1a, P1b, P1c;
float ID1a, ID1b, ID1c;
Fa = cellular(uv*32.0, 0.7, P1a, ID1a);
Fb = cellular(uv*16.0 + Fa*0.15, 0.7, P1b, ID1b);
// 主裂纹宽度的变化
Fc = cellular(uv*24.0, 0.7, P1c, ID1c);
craquelure = aastep(0.1, Fb.y-Fb.x
+ (Fc.x-0.35)*0.18);</code></pre>
</td>
<td>
<img src="2506292428-.png"/>
<em>更具变化的龟裂图案</em>
</td>
</tr>
</table>

这里选择的频率和振幅参数是为了突出我们添加的效果。实际使用时，可能需要使变化更加微妙。一个"专业"的着色器（设计用于艺术创作以生成各种严重程度的龟裂图案）应该使用参数代替某些常量，为非程序员提供修改图案的方式。我们在这些示例中有意忽略了这一点。这是着色器编程的重要方面，但需要大量思考才能做好。在下一个示例中，我们将花更多时间选择和命名相关着色器参数，供非程序员在创意工作中使用。 
同时cellular函数性能消耗有点大，至少我们需要做一些小小的限制，不要调用那么多次
![2506292618 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/06/29/1mg/2506292618-.png)
*第三次迭代： 增加二级裂缝*

图案看起来仍很人工化，原因是次级裂纹图案过于规则。特别是次级裂纹会跨越主裂纹延伸，这与现实不符。次级裂纹在主裂纹之后形成，且主裂纹两侧的次级裂纹图案是独立发展的。cellular函数返回的"单元ID"（0到288之间的整型哈希值ID1），我们可以利用它为相邻主裂纹单元创建不同的纹理坐标偏移。同时，也用该ID在不同单元间改变次级裂纹图案的缩放比例。只需修改一行代码即可实现：

![2506292703 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/06/29/1mg/2506292703-.png)
*第四次迭代：改进的次级裂纹图案变化*


这样就完成了吗？按照我们自己制定的"可接受"标准，这个图案确实达标了。但让我们再添加一个效果——剥落(flaking)。当龟裂程度比上述任何示例都更严重时，裂纹间的薄片就会脱落。当然剥落可能同时发生在主次裂纹之间，但为保持简单，我们仅随机去除部分主单元。根据其ID决定去除哪些，并用"裂纹"颜色绘制它们。只需在末尾添加两行代码：

```glsl
...  
// 可变比例的剥落薄片（0.1表示10%）  
float missing = step(0.1, ID1b / 288.0); // ID1b是模289值  
craquelure = min(craquelure, missing);
```

最终我们得到了视觉效果良好的图案，具有足够的变化来合理模拟真实效果：

![2506292802 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/06/29/1mg/2506292802-.png)  
*最终的龟裂图案*

该图案目前仍是黑白效果（尽管有适当的抗锯齿），通常不会单独使用。相反，它将作为多层纹理的一个组成部分，其他层决定裂纹层的颜色（可能使用基于图像的纹理）以及剥落后露出的底层。为获得更好的照片真实感，凹凸贴图也很有用。cellular函数的返回值包含足够信息来实现这点，为我们提供了丰富的创意变化空间。在第14章中，我们将演示如何为此类图案制作凹凸贴图。


## 木材纹理着色器

作为最后一个示例，我们选择创建一个程序化纹理来模拟木板的外观。用于木材生产的树木通常较为笔直，具有由年轮构成的3D结构，每年在树皮下新增一层。标准板材和梁木沿着树木的垂直方向切割，板材表面的图案很大程度上取决于它在原木中的切割位置。我们选择为整个原木实现3D函数，让着色器使用者通过缩放和平移3D纹理坐标来决定这些细节。

与所有着色器开发一样，这个过程应该从熟悉我们要复制的图案开始，决定代码需要渲染哪些变化，并了解产生这种图案的过程。我们仔细观察了许多现实中和网上的木板、梁木和原木来创建这个示例。

![2506293045 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/06/29/1mg/2506293045-.png)
*一些真实的木材图像，用于灵感和参考*

除了视觉研究外，我们还阅读了一些关于木材结构的百科全书级解释。正式的科学文章有些过于详细和具体，但有时这些参考资料也是可选的。由于有一个完整的科学领域研究木材作为建筑材料，我们学习了足够的术语来命名变量和编写注释。这比我们自己为所有东西发明名称要好得多。

着色器预期会被非编写者用于内容创作，我们无法确切知道他们想要什么。虽然可以做出有根据的猜测，但提供至少一定的灵活性是非常可取的。通过命名参数可以实现对外观的合理控制，这些参数可以在不重新编程和重新编译的情况下设置。离线渲染的软件着色通常提供一种机制来命名这些参数，设置它们的类型，指定默认值，提供交互式滑块或对话框来设置它们，并记录每个参数的作用。在GLSL中，这可以通过应用程序设置的**uniform**变量来实现。

不同于之前几个示例详细展示每个开发步骤，我们只展示最终版本的代码。下面迭代过程中的图像是模拟的，通过将最终着色器中的某些参数设为零来禁用部分变化而渲染得到。

![2506293107 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/06/29/1mg/2506293107-.png)
*3D木材着色器开发的四个阶段。**左上**：只有年轮。**右上**：用噪声位移的年轮。**左下**：添加噪声使年轮间的厚度变化。**右下**：靠近中心颜色较深的心材。*

实际开发确实按照所示顺序进行，但中间步骤很粗糙。颜色是黑白的，抗锯齿不足，噪声过于混乱。直到着色器完成后，我们才得到这种精美的外观。然后，在花时间挑选出看起来不错的参数值后，我们反向操作创建了这些"中间结果"。

"从大到小"通常是个好策略：先处理最突出的视觉方面，然后逐步添加细节，直到着色器达到我们想要的效果。

对于最终图像，我们将添加一个小细节使图案不那么卡通化：模糊纹理以在春材和夏材之间创建更宽的过渡区域。真实木材的生长并非如此——从春材到夏材的过渡是渐进的，但次年春材的开始经过休眠期后是突变的。然而，我们选择不处理这种不对称性。即使有这个细节，这个木材着色器也不会特别真实，而且很少有人会注意到我们在这个特定细节上的简化。

你可能会注意到，虽然图案有抗锯齿，但物体轮廓没有。平滑的轮廓对高质量图形很重要，但这将是渲染器的责任，而非表面着色器。我们这里关注的是表面图案，下图渲染时没有任何边缘抗锯齿。

![2506293134 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/06/29/1mg/2506293134-.png)
*3D木材着色器。实时版本见https://www.shadertoy.com/view/4X3cRS。*

最后，我们以视觉上最不有趣的部分——着色器代码列表来结束这个展示。注意代码中有许多命名常量。在实际生产用着色器中，这些应该是可以由应用程序设置的uniform变量。有许多具有长描述性名称的变量，这是好事。变量名的简洁性有时有利于可读性，但在这里不是。为了使着色器更有用，我们应该在注释或单独文档中更详细地记录变量，并为每个参数指定推荐范围。文档仍然很重要，即使对着色器程序员也是如此。着色器编程具有创造性和趣味性，但它不是无法之地。（好吧，有时是，但不应该是。）
```glsl
// A wood pattern shader. Stefan Gustavson 2024-12-27. Public domain code.
vec3 wood(vec3 p) {
	const vec3 spring_heart = vec3(0.5, 0.2, 0.0); // Spring heartwood color
	const vec3 summer_heart = vec3(0.3, 0.1, 0.0); // Summer heartwood color
	const vec3 spring_sap = vec3(1.0, 1.0, 0.5); // Spring sapwood color
	const vec3 summer_sap = vec3(0.7, 0.5, 0.1); // Summer sapwood color
	const float grain_size = 0.05; // Grain thickness in texture space
	const float grain_blur = 0.1; // Grain blur (relative to grain size)
	const float grain_variation = 0.2; // Variation in thickness between years
	const float spring_summer_ratio = 0.7; // Ratio of spring to summer wood
	const float heart_size = 10.0; // Heartwood radius (years)
	const float heart_transition = 5.0; // Heart/sap transition (years)
	const vec3 origin = vec3(0.6, 0.2, 0.0); // Log center in texture space
	const float curl_Rsize = 0.5; // Curl radial size in texture space
	const float curl_zsize = 2.0; // Curl longitudinal size in texture space
	const float curl_strength = 1.0; // Curl strength
	vec3 ptex = (p - origin); // Texture coordinates for the annual rings
	float R = length(ptex.xy) / grain_size; // Cylindrical coordinates
	float z = ptex.z / grain_size;
	// Texture coordinates for the noise displacing the rings
	vec3 pcurl = ptex / vec3(curl_Rsize, curl_Rsize, curl_zsize);
	float noise_mask = smoothstep(1.0, 4.0, R); // No noise near R=0
	R += noise_mask * curl_strength * noise(pcurl);
	// Variation in ring width (slicing 2-D noise to get 1-D noise)
	R += noise_mask * grain_variation * noise(vec2(R*0.5, 0.25));
	float Rtri = 2.0*abs(fract(R) – 0.5); // “Triangle wave” for nice AA
	// Deliberately blur the grain somewhat in close-ups
	float step_width = max(2.0*fwidth(R), grain_blur);
	float grain = smoothstep(spring_summer_ratio - step_width,
	spring_summer_ratio + step_width, Rtri);
	// Blend colors from heartwood (center) to sapwood (further out)
	float heart_start = heart_size - 0.5 * heart_transition;
	float heart_end = heart_size + 0.5 * heart_transition;
	vec3 spring_color = mix(spring_heart, spring_sap,
	smoothstep(heart_start, heart_end, R));
	vec3 summer_color = mix(summer_heart, summer_sap,
	smoothstep(heart_start, heart_end, R));
	return mix(spring_color, summer_color, grain); // Blend using grain pattern
}
```

## 足够好

我们可以无止境地做示例，因为这很有趣，但这五个案例研究必须足够代表，否则这本书永远无法完成。

希望这些示例已经向您展示了如何用目前学到的工具解决一些更实际的问题。着色器编程是一个极具创造性的过程，创建特定图案总有多种方法，您的代码会在开发过程中不断演化和变化。不必过分追求最佳解决方案。在许多情况下，"追求卓越"是黄金法则。但在着色器编程中，过分关注细节可能反而会影响整体目标。尽量避免发布明显笨拙、低效、难以理解且容易出错的代码，但在实验阶段不要害怕尝试"丑陋的代码"甚至"愚蠢的代码"。即使最伟大的艺术作品，最初也只是粗糙的草图。

---

<div style="text-align:center; font-size:1.2em; margin:20px 0;">
<span style="display:block; font-weight:bold;">"Only the best is good enough"</span>
<span style="display:block; font-style:italic; margin-top:8px;">
Ole Kirk Kristiansen，乐高创始人。在许多情况下成立，但<b>并非</b>总是如此。
</span>
</div>

<div style="text-align: center">
原始引文实际上是<b>"Det bedste er ikke for godt"</b>，丹麦语意为<b>"最好的也不会太过分"</b>。在我看来，这个说法更具普适性，尽管在截止日期前追求完美可能耗时过长。
</div>
