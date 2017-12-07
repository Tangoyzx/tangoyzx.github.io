---
layout:     default
title:      "理解微面BRDF中的masking-shadowing函数（未全译完）"
subtitle:   "重新构造视图空间坐标的方法"
date:       2017-08-31 23:15:00
author:     "Tango"
tags:
    - UnityShader
---


#1、介绍

微面理论最初是用来在光学物理学领域开发研究统计表面散射的。在图形社区中，我们使用它来派生物理的BRDF函数，这被广泛用于事实和产品渲染。今天，微面理论是计算机图形学的基本背景话题。例如，最近两年，SIGGRAPH上的基于物理渲染课程开始介绍微面理论，目标是提供由底层物理派生的主要观感。如艺术方向的灵活性以及技术方向的考虑同样通过课程来讨论。基于微面的BRDF因它不同参数的组合提供了很大范围的可能性，所以是一个连续开发的领域。所以每个参数的正确选择一般来说不会很明显，并且在我们的经验中通常是混乱的来源。


*这个文档的内容* 这文档的内容是对基于微面BRDF函数中的Masking-Shadowing函数提供新的看法和回答长久以来对其选择方法的问题的答案。这些问题会在2.5，3.6和4.4回答。实践者可能希望直接跳到这些章节。此文剩余部分是帮堵着寻求发展他们对微面理论的直觉与理解。

*这文档不包括的内容* 我们不会介绍新的BRDF模型；我们只会讨论常用的模型。我们不会推荐读者去使用哪个模型而不用哪个明星；我们目标是提供这些模型的背景知识，帮助理解他们怎么来的，他们做了什么，和我们可以期待他们干什么。我们不会体积他们的在特定渲染技术上的实现或者使用，因为他们已经在计算机图形社区广泛地使用了；我们集中在理解他们物理属性上。

*关于微面模型的“基于物理”的含义* 一个物理模型是一个系统或者物理现象的简化的表示，可以被分析、解释以及对其行为作出预测。
在微面理论中，被研究的模型是，在宏观上，一个平滑的几何平面，它的表面在微观上是粗糙的以及由微面组成的。这篇文章用于解释与预测几何平面上的散射，也就是光在相交的几何平面上如何在其他方向反射和散射的。

一个有意义的微面模型是由法线分布（对微面朝向的统计模型），微面轮廓（微面如何在表面上组织的模型）。由此为表面模型派生出来的微面BRDF等式恰好被称为“基于物理”因为他们基于一个微表面模型。相互的，假如一个微表面模型没办法派生BRDF等式，那他就不是一个“基于物理”的微面模型。遮蔽与阴影函数是微面BRDF的一部分。他们给出了一个位面是否能在出射方向（遮蔽）或者入射方向（阴影）可见。对BRDF来说，微面的遮蔽和阴影函数只可以在由微表面模型派生的时候被称为“基于物理”。

在这篇文章内，我们会解释微表面模型如何正确地描述，以及基于物理的遮蔽和阴影函数是如何派生的。我们同样会展示如何推导相关的基于物理的BRDF。

但是，需要注意的是微面模型是简单的*模型*。它们总是基于微表面光学行为的一些假设上建立的，例如，只有几何光学，完美的镜面或者漫反射，无多重散射等。所以，要记住叫做“基于物理”并不代表它们可以精确测量一个真实物理表面。在这些假设是错误的情况下，使用经验模型甚至会比使用数学严谨的“基于物理”模型在测量数据上更加准确。

*想法与组织*本文档中提供的想法主要由三个前人的工作提供：

*The Smith*遮蔽函数是图形学文章中最著名的一个。但是，比较少人知道的是，在他文章的结尾Smith指出他的公式具有保证可视投影区域守恒的属性，是一个正确遮蔽函数需要的属性。

Ashikhimin同样观察到可视投影区域是一个从几何表面到位表面守恒的一个数值。他们使用这个知识派生了一个正确的遮蔽参数公式，它可以保证正确的归一化以及能量守恒。通过做这个这个，他们可以在不意识到Smith遮蔽函数下重塑它。事实上，他们的遮蔽项是用积分的形式提出来的，而他们也没提供闭合形式。取而代之的是，他们预计算并把它放在look-up table中。

Ross 提出了海洋反射率的研究。他们使用高斯粗糙平面（Beckmann分布）对海面建模并且结合Smith的遮蔽和阴影函数计算一个归一化的BRDF。在这些为分钟，他们观察到在高斯平面上，BRDF的归一化系数和Smith遮蔽函数有类似的取消表达式。他们注意到这个属性方便于计算，但是他们没有提供一个物理原因去解释这个出现的原因。

在本文档中，我们提出了一个统一的微面框架，所有前面的结果（从遮蔽函数到整个BRDF）都可以直接从可视投影区域的守恒派生出来。

第二节中，我们会介绍微面统计数量和派生可视投影区域的守恒公式，通过正确的遮蔽函数来满足。

第三节中，我们介绍了*可视法线分布*以及展示一般BRDF模型怎样从这个分布中衍生出来。微面BRDF需要阴影的原因是他们只对微表面的第一次散射事件进行了模拟。一般微面BRDF不会对多重散射建模，也不会因此归一化，也就是说，它们和不刚好为1（即使当他们对一个完美反射的表面建模的时候）。从这种观察开始，我们提出了一个归一化测试——我们称为Weak White Furnace Test——可以用于校验一般基于微面的BRDF是否设计良好，尽管他们只对首次散射事件进行建模。

第四节中，我们会通过Smith和V-cavity微面轮廓来实例化由上面章节衍生的公式，并且对比它们各自BRDF的属性。我们衍生的共识没有提供新的结果，它们的优势在于强调结果是精确的而不是近似的，并且显示在任意表面上遮蔽是怎么与可视投影区域概念联系起来的。我们同样会回顾其他常用的遮蔽函数，一些没有从微表面模型衍生而来的或者是那些不准确或者不基于物理的。

第五节中，我们第一次展示遮蔽函数的弹性不变性属性。我们展示了如何使用它简单地推导出各向异性法线分布的遮蔽函数。这简化了前面几个结果的各向异性概括。

第六节中，我们为了阴影讨论Smith遮蔽函数的属性，并且回顾数个不同类型的遮蔽-阴影模型的相关性。

最终在第七节，我们讨论现有微面框架的一些限制，我们提出了根据我们调查获得的间接，为未来的工作提出了可能性。


# 2. 遮蔽函数的衍生

这一节，我们会展示微表面投影区域是如何对基于物理遮蔽函数引入约束，Ashikhmin。我们从定义投影区域的概念开始，并且展示为什么他对辐射测量是很重要的。然后，我们定义微面理论的统计框架。预估投影空间给出一个新的用于约束遮蔽函数的微面等式。这个约束，与微表面轮廓相关，引向基于物理遮蔽函数的推导。

## 2.1. 测量表面上的辐射

辐射是一个区域从一个立体角出来的能量强度。它单位是瓦特每球面度每平米。出射辐射对于给定平面M在特定方向w0，是对平面中心点pm的每个patch通过出射角度w0射出辐射的积分，并且通过出射方向的投影区域决定权重。

每个表面点投影在出射方向的区域是一个基于视线的权重系数，投影区域的积分，是投影面积分数的归一化系数。注意归一化系数让整个表达式是单位辐射；没有他，结果会在分母中失去区域单位。

在下面的章节我们可以看到，按照微面理论，这些微面同样根据投影区域计算权重，以及遮蔽函数（或者说几何强度系数）是一个需要能量守恒的归一化项。

## 2.2. 微面统计
我们考虑一个表面的平面区域，我们成为“几何平面”G，按照惯例面积是1平方米。微面模型假定真正的表面是微面集合形式的偏移，我们成为“微表面”M。准确来说：假如wg是几何表面G的法线，那么M就是微面点沿着wg投影到G的几何。对于微表面M每个点pm拥有法线wm(pm)，也就是说，wm:M-Ω是从微表面的点到那点在表面法线向量的函数。我们用 (xm,ym,zm)表示这个向量的三个坐标。

微面理论是一个微表面散射属性的统计模型。所以，对这个研究而言写*统计*比写*空间*公式更加方便。在微面理论，统计数据是在法线空间中被定义，也就是球形空间Ω。

*法线分布* 为了关联微表面的积分和球面积分——即，从空间转换到统计积分——我们需要一个可以在切换空间的时候测量改变区域的工具。*法线分布*提供了这个。它的单位是平方米每立体角，以及定义如下：

Dirac delta分布的单位是1/sr，是它参数的倒数。想象一个区域Ω'单位圆上。现有微表面的子集M'包含所有M中的法线范围wm(pm)属于Ω'的点，也就是说


法线分布，在单位圆上任意区域，


*空间与统计等式* 作为D定义的结果，假如f(wm)是微表面法线的任意函数，那么f的空间积分可以被统计积分代替

左边的是空间积分，右边的是统计积分。这个属性用于图3(a)，f是点乘

*统计函数* 假如说g(pm)是在微表面上定义的空间函数，我们可以定义关联的统计函数g(wm)

这个统计函数可以以下面的形式用于统计积分：

这个属性用于图3(c)，当g是我们在2.3节介绍的遮蔽函数G1。

## 2.3. 微面投影

*(a) 微表面沿着几何法线w0投影的区域* 为表面区域沿着几何法线投影的区域是几何表面的区域（图3(a))，为了方便面积记为1平米。于是，法线分布投影到几何上的值是归一化的。

*(b) 几何表面沿着出射方向w0的投影区域* geometric surface area是1平米，并且他投影到出射方向（图3(b))的面积就是区域面积乘以入射角的余弦。

*(c)可视微表面沿着出射方向wo的投影区域* 我们现在展示geometric suface沿着出射方向的投影区域同样也是*可视*microsurface的投影区域(图3(c))。这是每个可视microfacet的投影区域总和。法线为wm的microfacet有几何投影系数clamp(dot(wo, wm))。注意这里我们使用clamped的点乘因为背向的microfacet是不可见的。同样，被microsurface挡住的microfacets对投影区域没有任何贡献，所以必须从总和中移除。这是通过乘以spatial masking function G1(w0, pm)达到的，这个函数拥有二进制值：假如点pm被遮蔽了则值为0，否则即为可视的话值为1。于是有：

*statistical masking function G1(w0, wm)*范围是[0,1]，给出沿着出射方向w0可视并且microfacets法线为wm的分数。

统计学等式是：


## 2.4. 遮蔽函数的约束
图3强调了微面理论的一个基本属性：等式13中的可视microsurface投影区域正好是等式10里geometric surface的投影区域。这个条件给统计遮蔽函数带来了一个约束，可以写成下列等式

基于物理的遮蔽函数G1应该永远满足这个约束。但是这个约束并不能完全地确定G1，因为对于固定的出射方向w0，遮蔽函数是二维的——G1(w0, wm)被每个法线定义着——有无数的G1函数满足着这个等式。为了选择出适合的方案，我们引入第二个约束：我们选择一个*microsurface profile*

一种直观方式是认为法线分布就像是直方图，只描述microsurface上每种法线的*比例*。而并不提供法线是如何组织的信息，但是，为了这点，我们需要microsurface profile。此外，图4说明，一个正确的profile可以对BRDF结果的形状造成强大的影响。

一旦选择了microsurface profile，masking function就被完全确定了，并且可以推导出确切的形式。在第4节会回顾一下通过Smith和V-cavity microsurface profile获得的G1的确切形式。

## 2.5. 总结
一个masking function常见的问题是*“在不同的masking functions(或者geometric attenuation因子)中，我应该选择哪个？它们都是基于物理的吗？”*

在本节，我们已经展示了：

### 在任意方向，可视microsurface的投影区域等于geometric surface的投影区域。
### masking function收到这个等式的约束。更正式地说，基于物理的masking function都会满足等式14.
### masking function并不是完全由这个约束决定。
### masking function在选择microsurface profile后就可以完全确定
### microsurface profile影响着BRDF的形状


# 3. Microfacet-Based BRDFs

在这一节中，我们会定义了可视法线分布(3.1)，我们会展示microfacet model一般是怎么从这个分布中建立的(3.2)，以及在特殊的specular(3.3)和diffuse(3.4) microfacets中建立。我们展示了masking function是可视法线分布的归一化系数，并且我们会讨论由此分布建立的BRDF的能量守恒问题(3.5)。


## 3.1. 可视法线分布

在本节，在microfacet范例中可以使用等式(1):


L(w0, M)代表着microsurface的出射辐射，L(w0, wm)是法线为wm的microfacet的出射辐射，系数(1/cosθ)在这里是geometric surface的投影区域用于归一化积分。我们可以看到microsurface的出射辐射就是每个microfacet的出射辐射乘以可视法线分布的权重的总和，如图5所示。法线分布根据每种法线的投影区域（clamped后的余弦）和masking function进行权重调整：

可视法线分布函数是归一化这点很重要，因为我们要用它作为平均辐射的权重函数：

以及在2.1节和图1中解释过的一样，平均辐射只有在权重函数是归一化过的情况下是可行的。上一条等式是明确的因为在等式1的分母用于保证正确归一化的的积分现在由masking function G1提供。实际上，通过使用等式14的结果，我们可以替换掉等式16中的余弦，这就可以保证法线分布是归一化的：


等式15和17的平均出射辐射可以用等式1一样的形式来表达，强调正确的归一化：



## 3.2 BRDF的构造
我们现在可以通过上面的可视法线分布来构造BRDF。每个microfacet的辐射L(wo, wm)可以用每个位面关联的micro-BRDF来表达，在Ω域中积分辐射L(wi)：



micro-BRDF是出射辐射与入射辐射的比例：


下一步，我们将等式17与入射辐照度以及通过等式21替换成dL(wo, wm)：


其中L(wi)dwi可以提到积分外，因为它与wm无关。而macro-BRDF被下面等式定义：


我们有下面的结论：

一个重要的观察是这个这个等式只是基于以下情况进行建模：光线在离开表面附近之前被第一次反弹后马上反射（如图6b)。但是，BRDF模型必须替代描述在所有微观散射情况下射线是如何在离开表面以后是如何分布的。这分布在离开表面附近前后是不一样的，因为有一部分反射的光线再次碰到microsurface并且在离开前反射到其他地方（图6d）。由于这里得出的BRDF模型仅仅是表面上的第一个反弹，包含多次反弹的射线（图6c）必须从模型中移除，这是通过引入shadowing function解决。实践中我们用masking-shadowing函数G2代替masking函数G1：


接下来，我们会实现该方程的特定情况，则microfacet分别是完美的镜面反射（3.3）和完美的Lambertian漫反射（3.4）。


## 3.3. 使用specular microfacets来构建BRDF

镜子一般的microfacets的micro-BRDF 是：


其中？？？是Jacobian of the reflection transformation，F是Fresnel项。通过使用方程27中的PM和方程16中的Dwo(wm)放到方程26中，我们得到：


delta function允许我们使用在wm=wh对被积函数评估来代替积分，并且wo·wh=wh·wi这个条件可以把等式简化为：


我们就得到了为人熟知的specular microfacet-based BRDF方程。

## 3.4. 使用diffuse microfacets来构建BRDF

diffuse microfacet的micro-BRDF是常量：

在方程26中，通过使用方程30的PM和方程16的Dwo(wm)替换后我们得到：

这个方程没有解析解。Oren和Nayar提出一个满足这个函数的解析，其中D是spherical Gaussian——避免与Beckmann分布产生混乱——以及G2是V-cavity masking-shadowing function。

## 3.5. BRDF归一化测试

*The White Furnace Test* 双向散射分布函数(BSDF)是上半球的双向反射分布函数p和下半球的双向投射分布函数(BTDF)t的总和：



加入我们有一个完全不吸收任何辐射的表面，那么在散射期间光线的辐射将会被完美保留。所以有一个重要的属性需要通过microfacet-based scattering模型来确定，就是当表面的吸收为0的时候，散射光线的分布是完美的归一化的：


假如Fresnel项固定为1，则光线永远不会透射（它们永远不会穿透表面），所以BTDF评估为0，而散射模型完全由BRDF来定义。在这个例子里面，所有光线都被反射，没有能量损失，而且它们的分布是归一化的。这是通过*White Furnace Test*方程建模：


直观来说，这反映了一个事实，从出射方向离开的光线(图6a)可能被散射过一或者多次最终离开了表面(图6d)。一般analytical BRDF并不会对microsurface上的多重散射进行建模；反射多次的光线通过shadowing function从BRDF中移除了，如图6c和3.2节中描述的那样。这就是为什么一般BRDF模型无法积分到1即使是在“完美反射”的microsurface上，并且不满足the White Furnace Test方程。

*The Weak White Furnace Test* The White Furnace Test不能用于验证一般的BRDF模型，因为它们只整合了第一次散射的情况。但是，我们可以设计另外一个限制性较少而且肯定能满足一般的microfacet-based BRDF模型的测试.我们可以验证第一次反射后就离开表面的光线的分布是归一化的(图6b)。这可以通过把masking-shadowing替换成单独的masking。移除Fresnel和shadowing 以后，方程29的BRDF变成了：


把*White Furnace Test*代入方程34以后，|wg·wi|项被消除了，那么*Weak White Furnace Test*公式为：

这个条件只有在有满足等式14的近似masking function G1的时候满足。在附录C中，我们提供了MATLAB代码来计算Beckmann和GGX分布以及他们关联的方程36的值。
我们可以定义相同的测试给diffuse microfacet的BRDF， 通过把G2(wo, wi, wh)=G1(wo, wh)代入等式31并且对不同方向进行积分：



## 3.6. 总结
一个关于BRDF归一化的常见的问题是：“Microfacet-based BRDF和不等于1。他们不是应该是完美的归一化么？”

在这节，我们通过延伸下面几个想法来回答这个问题：

### 这个BRDF是通过可视法线分布来构建的
### 可视法线分布必须归一化来保证BRDF是能量守恒的
### 可视法线的归一化系数就是masking function
### Microfacet-based BRDF应该是归一化的，也就是说，对于无吸收无透射的材质积分必须正好是1
### 在microfacet-based BRDF的shadowing function用于把microsurface中的首次散射的情况从多重散射情况中区分出来。
### Shadowing抛弃(设为0)散射次数多于1的情况，在缺少一个对多重散射情况建模的参数情况下令到BRDF人为地变为非归一化。
### 只有masking function没有Fresnel和shadowing的BRDF标准格式是归一化的。基于物理的masking function会满足方程36和37，这就是我们说的“Weak White Furnace Test”

注意Weak White Furnace Test，它不包含shadowing，是一个保证masking function是物理可信的简单方法。需要注意的是这并不代表一般的BRDF模型应该抛弃shadowing。Shadowing是用于把单次反射的能量从多次反射的能量中分离开来，这在一般地BRDF模型中不应被抛弃。


# 4. 一般基于物理和非基于物理的Masking Function

在2、3节我们派生了masking function的几个常见的结果——方程14、18、36——对microsurface的类型没有任何假设。在这一节，我们回顾一下Smith(4.1)和V-cavity(4.2)的microsurface profile，得出他们各自masking function的闭合形式，并且讨论他们的属性。我们同样会回顾其他一般的masking function，那些没有关联的microsurface profile，所以并不是基于物理的(4.3)。

## 4.1. The Smith Microsurface Profile

*Normal/Masking Independence* The Smith microsurface profile假定microsurface并不是自相关的，也就是说microsurface上一个点的高度(或者法线)与邻近一个点的高度(或者法线)是没有关系的，哪怕是最近的点。这意味着是一组随机的microfacet而不是连续的表面——如图7右显示的，其中microsurface的高度和法线都是独立的随机变量。
这种模型的结果就是通过masking function给的概率G1(wo, wm)与非背面(wo·wm>0)的法线方向无关。有个直觉是法线wm是一个microfacet的*局部*属性