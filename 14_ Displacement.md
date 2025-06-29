

纹理函数不仅可以用于修改表面属性（如颜色和透明度），还能改变表面的位置，这被称为*位移贴图(displacement mapping)*。严格来说，这是一种创建程序化*几何体*的方法，本应属于本书第二部分，但我们在此讨论它有两个原因：一是位移函数定义在表面上，与其他纹理方法密切相关；二是位移贴图有个近亲——*凹凸贴图(bump mapping)*，后者不实际移动表面，而是通过改变法线来模拟位移效果。凹凸贴图无疑是一种纹理方法，能为表面外观增色不少。许多现实世界表面的颜色相对均匀，其外观主要由局部朝向变化决定。还有些表面若没有凹凸贴图（或位移贴图，或两者结合）来补充其反射特性，效果会大打折扣。

![2506291340 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/06/29/1mg/2506291340-.png)
*一些具有强烈凹凸感的表面*

凹凸贴图由Jim Blinn在其1978年Siggraph论文《Simulation of wrinkled surfaces》[https://doi.org/10.1145/965139.507101]中首次提出，此后一直是计算机图形学中重要且流行的工具。位移贴图自计算机图形学诞生初期就用于几何体生成，1980年代的原始Renderman接口中*位移着色器*就是核心组成部分。实时GPU着色最初就有两个着色阶段：顶点着色器和片段着色器，其中顶点着色器不仅能执行对象和视图变换，还能以受限方式处理位移——只能移动现有顶点，无法生成新顶点。后来引入了更多阶段（几何着色器、细分着色器和细分控制着色器）来更好地处理程序化几何生成。随着着色阶段及其交互日益复杂，正如从笨拙的固定功能图形管线到可编程着色框架的转变，我们现在正见证从各类几何生成着色器向更通用的计算着色器的演进。计算着色器不仅限于图形处理，但也可用于此目的。本书第二部分将更详细讨论使用着色器生成程序化几何体，下文我们仅聚焦位移贴图和凹凸贴图。

这两种方法的物理和数学原理相同，且不依赖于位移贴图或凹凸贴图的生成方式。但事实证明，使用3D程序化函数能大幅简化所需计算（Mikkelsen 2008年论文《Simulation of Wrinkled Surfaces Revisited》http://image.diku.dk/projects/media/morten.mikkelsen.08.pdf指出）。该作者后来在已停刊的《Journal of Graphics Tools》发表文章[https://doi.org/10.1080/2151237X.2010.10390651]，实质上是下文内容的加长优化版。若无访问权限，不必支付出版商对该短文的高昂单篇费用，可直接使用作者提供的预印版[https://mmikk.github.io/papers3d/mm_sfgrad_bump.pdf]。

为阐明程序化方法如何简化凹凸贴图，我们先总结经典实现方式。

### 传统凹凸贴图（复杂版）

Jim Blinn最初提出的凹凸贴图针对参数化曲面定义在2D参数空间$(u,v)$中，使用虚拟位移函数$h(u,v)$。要将其应用于非参数化表面（如现代计算机图形学中普遍的多边形网格模型），需要某种外在参数化。幸运的是，现代图形应用大多使用纹理映射，2D纹理空间$(s,t)$正是这样一种外在参数化。位移可表示为单通道（灰度）2D纹理图像$h(s,t)$，表面虚拟位移后的位置$\bar{p}$沿法线方向$\hat{N}_0$与原位置$\bar{p}_0$相距$h(s,t)$：

$$
\bar{p} = \bar{p}_0 + h(s,t) \hat{N}_0
$$

重新计算凹凸表面的法线方向需要$h(s,t)$对$s$和$t$的偏导数：$\partial h / \partial s$和$\partial h / \partial t$。出于性能考虑，这些导数常被预计算并存储为纹理。

*法线贴图*作为凹凸贴图的现代变体，操作类似但直接将法线明确指定为3分量纹理$\hat{N}(s,t)$，完全取代插值后的网格法线。这种方式灵活性稍逊但渲染时计算量更小。

位移贴图实际改变表面位置，与凹凸贴图紧密相关，因为位移通常沿法线方向发生。位移后重新计算法线涉及与凹凸贴图相同的计算，需要位移函数的平面内偏导数。

当应用于与世界空间坐标轴对齐的平面时，法线重映射对凹凸贴图和法线贴图都很直接。但对任意表面，重新计算法线需要全面了解表面在3D空间中的朝向。除了普遍存在的法线方向，还需指定一个或两个额外向量来关联2D纹理坐标$(s,t)$与3D对象空间$(x,y,z)$。定义切平面的基向量$\hat{s}$和$\hat{t}$与表面法线$\hat{N}$正交，且平行于各纹理坐标的局部梯度$\nabla s$和$\nabla t$：

$$
\bar{s} = \nabla s = (\partial s / \partial x, \partial s / \partial y, \partial s / \partial z) 
$$

$$
\bar{t} = \nabla t = (\partial t / \partial x, \partial t / \partial y, \partial t / \partial z)
$$

单位向量$\hat{s}$和$\hat{t}$通过对$\bar{s}$和$\bar{t}$归一化得到。需注意这些向量通常不是通过偏导数计算的，而是作为网格模型的额外顶点属性明确指定。若2D纹理映射坐标方向基本正交，指定一个向量并通过与法线叉积计算另一个可能就足够。这两个附加向量常被称为*切线(tangent)*和*副法线(binormal)*，尽管二者实际上都是表面的切线。

有了单位长度的基向量$\hat{s}$和$\hat{t}$后，新的凹凸表面法线$\hat{N}$可通过修改原始表面法线$\hat{N}_0$计算：

$$
\bar{N} = \hat{N}_0 - (\partial h / \partial s) \hat{s} - (\partial h / \partial t) \hat{t}
$$

再将$\bar{N}$归一化为单位长度得到$\hat{N}$。

Blinn在1978年就指出，这只是近似方法，仅在表面局部曲率远小于凹凸函数局部曲率时有效。凹凸尺度应明显小于对象的几何细节。但即使近似不精确，视觉效果仍可能令人信服。

处理2D切空间及其到3D空间映射的复杂操作，一直是计算机图形学界许多困惑和挫败的源头。教程和示例含糊不清甚至错误，一些商业工具也未能正确执行相关变换和插值。

定义在3D空间中的程序化凹凸贴图函数消除了对显式2D切空间的需求，事情因此变得简单许多。

## 程序化风格凹凸贴图（简易版）

凹凸贴图是一种虚拟位移技术：我们仅重新计算法线，*仿佛*表面发生了位移。假设使用3D函数$h(x, y, z)$定义位移：

$$
\bar{p} = \bar{p}_0 + h(x, y, z) \hat{N}_0
$$

与传统方法的区别在于，这里不再有显式的2D纹理空间和切空间$(s, t)$，只有3D对象空间$(x, y, z)$，且不涉及表面参数化。要计算位移后表面$\bar{p}$的法线，仍需将位移函数$h(x, y, z)$的梯度投影到切平面，但这可通过基础线性代数轻松实现：通过标量积将梯度$\nabla h$投影到法线方向，然后减去梯度中平行于法线的部分（这是*格拉姆-施密特正交化*的核心步骤，用于从任意线性无关向量集构造正交坐标基）。相减后剩余的就是梯度中*正交*于法线的部分，即投影到表面局部切平面的梯度。不过现在这个向量已方便地用3D*对象空间*表示，可直接与同样定义在对象空间中的法线相加：

$$
\bar{g} = \nabla h = (\partial h / \partial x, \partial h / \partial y, \partial h / \partial z)
$$

$$
\bar{g}_{//} = (\bar{g} \cdot \hat{N}_0) \hat{N}_0 \quad \bar{g}_{\perp} = \bar{g} - \bar{g}_{//} \quad \bar{N} = \hat{N}_0 - \bar{g}_{\perp} \quad \hat{N} = \bar{N} / \| \bar{N} \|
$$

就这么简单。注意整个过程只需在*对象*空间$(x, y, z)$中进行向量运算。

对于程序化位移函数，梯度$\bar{g}$通常可以解析计算并相对容易地精确求值。若不可行，有限差分法总能以足够精度实现（尽管计算成本更高）。用有限差分法计算$\bar{g}$至少需要额外三次$h(x,y,z)$求值，而像噪声函数这类解析梯度计算往往成本低得多——单纯形噪声和基于散射的距离函数显然属于此类。经典Perlin噪声的解析微分稍麻烦，但仍比有限差分计算简单。

下面展示一个着色器对示例，顶点着色器执行位移贴图，片段着色器执行凹凸贴图，二者都使用能自计算解析梯度的**psrdnoise**噪声函数。注意两者重新算法线的方式完全相同。顶点着色器可以且被期望移动顶点，而片段着色器只能沿深度方向移动片段，因此仅修改法线。光照计算保持简单以展示原理，使用固定位置和强度的单一光源。我们仅加入高光成分来更清晰演示凹凸贴图效果。

顶点着色器简洁明了：
```
// Vertex shader with noise-based displacement mapping
in vec3 xyz; // Vertex attribute: position
in vec3 N0; // Vertex attribute: normal
uniform mat4 MV, P; // Modelview and projection matrices
out vec3 p; // Object space coords for 3-D texture mapping
out vec3 N; // Modified vertex normal
void main() {
	const float displaceamount = 0.1; // Object space units
	vec3 g;
	float d = psrdnoise(2.0*v, vec3(0.0), 0.0, g);
	g *= 2.0; // Inner derivative of noise texcoords
	p = xyz + displaceamount * d * N0; // Object space (before transforms)
	vec3 g_ = g - dot(g, N0) * N0;
	N = N0 - displaceamount * g_;
	N = normalize(N);
	gl_Position = P * MV * vec4(p, 1.0); // Transform to view space
}
```
The fragment shader is longer, because it needs to compute the lighting as well:

```
// Fragment shader with noise-based bump mapping
in vec3 p; // Interpolated object space coords for bump map
in vec3 N; // Interpolated displaced normal
uniform mat4 MV; // Modelview matrix, used to transform normals
out vec4 fragmentColor; // RGBA output
void main() {
	const float bumpamount = 0.02; // Object space units
	vec3 g;
	float bump = psrdnoise(8.0*p, vec3(0.0), 0.0, g);
	g *= 8.0; // Inner derivative of scaled p
	vec3 g_ = g - dot(g, N) * N;
	vec3 bumpedN = N - bumpamount * g_;
	bumpedN = normalize(mat3(MV) * bumpedN); // Transform normal
	// Very simple lighting
	vec3 diffusecolor = vec3(0.5 + 0.5*bump); // Color value
	diffusecolor.rb = 1.0 - diffusecolor.rb; // Put some color to the gray
	vec3 L = normalize(vec3(1.0, 1.0, 1.0));
	vec3 specularcolor = vec3(1.0); // White
	float lightA = 0.2; // “Ambient lighting”, the old kludge
	float phongD = max(0.0, dot(L, bumpedN)); // Diffuse reflection
	float lightD = mix(phongD, 1.0, lightA);
	float lightS = - normalize(reflect(L, bumpedN)).z; // View is from -z
	lightS = max(0.0, lightS);
	lightS = pow(lightS, 16.0); // Phong “specular lobe”
	vec3 color = lightD * diffusecolor + lightS * specularcolor;
	fragmentColor = vec4(color, 1.0);
}
```


*运行此着色器对在"球体"（近似光滑球体的高细分三角网格对象）上的渲染效果*
![2506292134 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/06/29/1mg/2506292134-.png)
*应用位移贴图和凹凸贴图的球体网格对象*


##  抗锯齿处理

对于凹凸贴图，缩小(minification)情况下的抗锯齿处理可能相当特殊。小于单个像素的凹凸需要被替换为其平均效果，但简单地消除那些会产生过小高光（导致锯齿）的凹凸是不够的。凹凸贴图在物理位移空间中执行（即使位移并未实际发生），但视觉锯齿出现在反射空间中。表面反射率取决于法线方向，但这是非线性且复杂的关联。一个粗糙但具有高光的表面，当其粗糙度尺度远小于单个像素时，*不会*呈现与平坦表面相同的反射特性。这正是为什么局部反射模型会有"高光"项，并带有可变的扩散范围——这项设计就是用来模拟"微观"尺度（即小于单个像素的尺度）的表面粗糙度。

为了在凹凸表面的高分辨率和低分辨率渲染间取得良好匹配，可以使用图像编辑软件对高分辨率图像进行平均和下采样，并努力使未加凹凸的低分辨率视图与之达到合理的视觉匹配。这需要在通过频率截断消除凹凸成分时，同步调整表面反射模型的"高光粗糙度"参数。具体如何调整取决于反射模型、表面高光强度以及凹凸贴图的统计特性。某些情况下，表面物理属性与高光参数间存在较明确关联。但对于大多数流行表面反射模型而言，模型参数与实际物理光学特性并无联系，只能凭经验调整以获得定性视觉匹配——达到"看起来还行"的效果。

![2506292320 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/06/29/1mg/2506292320-.png)
*凹凸贴图影响高光的角范围。**左图：**带凹凸的高光表面。**右图：**当为远距抗锯齿消除凹凸贴图时，高光应相应变化。上图使用与下图相同的高光参数，中图则增大了高光尺寸以更好匹配带凹凸的外观。*

当前趋势是即使在GPU着色器中也使用更优、更基于物理的反射模型，但要成为默认选择可能尚需时日。问题因计算机图形渲染传统处理高光的方式而加剧——仿佛高光强度与白色漫反射表面相当。现实中，高光强度可比漫反射高数个数量级。这可通过高动态范围(HDR)渲染正确处理，其中像素值既不局限于狭窄的[0,1]范围，也不粗糙量化为每通道8位，而是以浮点值存储。实时渲染中的HDR处理目前仍较混乱，但正在改进。多数离线渲染器现已正确处理。

除基于物理的反射模型和HDR渲染外，计算机图形学还需开始将光线和反射率都视为光谱分布。仅用三个通道(R,G,B)采样颜色对图像捕获和复现足够，但对模拟图像*形成*（光与物质的相互作用）而言，这种折衷往往过于粗糙。皮克斯早在1980年代就意识到这点，在Renderman SL中定义了带可选多通道内部表示的*color*数据类型。遗憾的是，所有实现仍仅使用传统RGB。在GPU着色器中，所有颜色现在都常规处理为*vec3*，全然不考虑这对除最终像素输出外的所有环节都存在根本性错误。不过这是题外话了。


## 凹凸贴图设计

当有真实案例可供详细研究或能准确把握物理结构时，凹凸贴图的设计可以相当直观。需注意的是，表面反射变化的很大部分可能源于结构而非反射率本身，因此除非颜色恒定且表面凹凸是唯一变化，否则最好将凹凸图案与颜色*共同*设计和评估。

第13章的龟裂图案就是合适案例——若想近看呈现剥落油漆效果，它确实需要凹凸贴图。为清晰起见，我们暂不考虑最终着色器，仅使用未受干扰的直线主裂纹简单图案。

![2506293104 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/06/29/1mg/2506293104-.png)
*演示用的简单裂纹图案（部分代码和平涂渲染效果）*

图案基础是返回随机散射最近点距离的cellular函数。该距离的平方适合作为薄片高度剖面。下图以一维形式展示：

![2506293118 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/06/29/1mg/2506293118-.png)
*最近点距离及其平方。点位于1.0、2.0、2.7和3.5处*

对2D GLSL函数，我们将使用最近点距离平方$F_x * F_x$，并缩放到较小高度，因为要模拟的表面高度变化较细微。还需计算凹凸函数的导数。作为距离函数，$F_x$的梯度模长恒为1，方向背离最近特征点。返回值$P_1$是指向该点的向量，因此梯度为-normalize(P1)。函数$F_x * F_x$的梯度模长为$2.0*F_x$且方向相同，可表示为$2.0*F_x*(-normalize(P1))$。注意到$F_x$等于length(P1)，梯度其实就是-2.0*P1（记得我们说过解析导数有时非常容易计算吗？）。

![2506293143 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/06/29/1mg/2506293143-.png)
*裂纹表面的简单凹凸函数及效果渲染。完整代码见https://www.shadertoy.com/view/43GcD1*

并非所有凹凸贴图都如此简单，但令人惊讶的是，粗糙近似往往就够用。凹凸贴图的视觉关键是其特征的整体强度、尺寸和形状，而非精确剖面。但对位移贴图而言，函数直接生成可见的几何细节，通常需更多努力才能准确。

凹凸贴图的正负号也很重要。凹坑与凸起看起来截然不同，尤其在动画和明确方向照明下。人类视觉系统非常擅长根据反光细节判断表面结构，符号错误会使凹凸贴图完全失真。例如上述代码符号错误会使薄片异常凸起：

![2506293219 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/06/29/1mg/2506293219-.png)
*符号反转会使凹凸贴图完全错误。静态图中效果不如交互视图明显，但仍可辨识*

强调"数学计算要注意符号"这个明显要点的原因是，这实在是个常见错误。甚至某些高质量商业软件也处理错凹凸贴图符号。这可能是测试中易忽略的bug——典型"偷懒"的凹凸贴图只是噪声，对符号不敏感。但程序化凹凸贴图远不止随机波动。当同时使用位移贴图和凹凸贴图并通过混合实现动态细节层次时，符号错误会成为真正问题。

另一个常见错误是忘记用纹理坐标缩放的内导数来缩放解析梯度，导致凹凸效果要么过弱要么过强。支持凹凸的着色器设置中，"bump amount"参数通常具有与虚拟位移高度无关的任意范围。其实不必如此。设计程序化凹凸贴图时，按位移贴图思路考虑有助于正确缩放，并使同一贴图同时适用于凹凸和位移。


## 更高级的平滑度处理

在使用凹凸贴图时，我们使表面反射强烈依赖于法线方向的局部修改。要实现无可见折痕的光滑凹凸效果，凹凸函数需要具备二阶连续性——即一阶和二阶导数都连续。许多常规用于程序化着色器的函数并不满足这一特性，例如经典Perlin噪声就不具备。但使用改进的五次插值函数的Perlin噪声则满足，这也正是改进版存在的理由。单纯形噪声同样符合要求。而**smoothstep**函数则存在问题，因为它采用三次插值函数$3t^2 - 2t^3$（$0 \leq t \leq 1$），在端点处斜率为零但二阶导数非零。正因如此，再加上人类视觉系统对对比度和边缘的敏感度，使用**smoothstep**的程序化凹凸贴图或位移贴图会在斜率起始和结束处产生微弱但可见的折痕。这些折痕在高光表面上最为明显，但即使漫反射表面也会出现，只是程度较弱、较不刺眼。

还记得用于解决Perlin噪声导数不连续的五次插值方案吗？我们可以定义一个采用该曲线过渡的混合函数。这个函数有时被戏称为**smootherstep**，这个名字其实相当贴切。这种"更平滑的smoothstep"在某些情况下非常有用，特别是在创建程序化凹凸贴图时。

![2506294220 ](https://blogimgbeg.oss-cn-shanghai.aliyuncs.com/2025/06/29/1mg/2506294220-.png)
*"smootherstep"混合函数（并非任何着色语言的内置函数）*

该函数在$a$和$b$点的一阶与二阶导数均为零。相比**smoothstep**，其过渡开始和结束更平缓，中间部分更陡峭。GLSL实现如下：

```glsl
float smootherstep(float a, float b, float x) {
    float t = clamp((x - a)/(b - a), 0.0, 1.0);
    return ((6.0 * t - 15.0) * t + 10.0) * t * t * t;
}
```
