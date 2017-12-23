---
layout:     default
title:      "CommandBuffer Decal Shadow"
subtitle:   "脑洞阴影方案"
date:       2017-12-23 11:34:00
author:     "Tango"
tags:
    - UnityShader
---

# 1 概述

## 1.1 [Unity移动端动态阴影总结](https://zhuanlan.zhihu.com/p/26662359 "去知乎")

之前在这个git里实现过一个Planar Shadow，核心是额外渲染一次模型，并在此次把顶点投影在某个xz平面上。按照算法其实很容易想到这个实现的缺点：阴影无法渲染在其他地方，如墙壁、楼梯、凹凸的地面等。一般会控制渲染顺序让阴影覆盖在地形上避免地形穿插问题，墙壁这种就没办法解决了。

解决这个核心问题往后有两个方案：

### Projector

使用Projector时，先在Projector处渲染一次需要生成阴影的物体（阴影图），然后重绘接受阴影的Mesh，计算出每个像素在阴影图中的颜色并且附加到屏幕上。缺点在于要重绘接受阴影的Mesh，假如是地形这类超大的Mesh的话消耗太大。

### ShadowMap

这个就是比较通用的方法了，一般是对全场景使用，当然效果是最好的，ShadowMap需要用各种方式来解决精度和大小问题。

刚好看到了CommandBuffer官方例子里面有个贴花例子，想着可以使用结合ShadowMap做出一个类似Projector阴影的效果而又不需要渲染目标Mesh。

## 1.2 基于CommandBuffer的贴花效果

官方例子是一个基于延迟渲染的贴花，核心原理如下（DiffuseOnly）：
1、正常渲染，在BeforeLighting处加入贴花渲染流程。
2、对于每一个贴花，渲染一个Cube。
3、对于Cube渲染的每一个像素，根据屏幕坐标获取该像素点的深度，根据屏幕坐标和深度反推出世界空间的坐标。
4、然后把世界空间的坐标使用Cube的WorldToLocal变换到物体空间坐标，然后根据xy对贴花的贴图进行采样。
5、把采样的结果写到像素的漫反射贴图中。
6、正常进行后续渲染流程。

假如是带Normal的贴花也会对法线采样并且写到GBuffer的法线贴图中。

这个流程是对于延迟渲染的，但是步骤1主要是为了获得深度图，这在Forward流程也可以获得。步骤5对于Forward也是不必要的，直接输出到屏幕就可以了。

把贴花改成阴影主要是要修改第4步，把取样的贴图改成从Cube的-Z往Z方向拍摄的深度图，然后对比深度判断是否在阴影之中，是的话把半透明黑色叠加在对应的位置，就可以有类似阴影的效果。

接下来开始一步步实现。

# 2 实现

## 2.1 渲染Cube中的深度图

### 2.1.1 计算最小包围盒

为了让生成的ShadowMap尽可能的小，我们需要计算需要生成阴影物体在Cube物品空间中的最小包围盒。

我们可以获取每一个生成阴影的Renderer的AABB，把它们转换到Cube空间下。最笨的做法就是把八个顶点都转换到Cube空间，但其实不用。

假定变换矩阵为m，当前控件的AABB的八个顶点分别为center±extents。八个顶点转换后的x坐标如下：

target.x = m00 * (center.x ± extents.x) + m01 * (center.y ± extents.y) + m02 * (center.z ± extents.z)

假设我们最终要求x的最大值，相当于求三项的最大值的和。

但看第一项m00 * (center.x ± extents.x)，由于extents相当于size的一半，永远非负，所以这一项的最大值只与m00的符号有关：

#### 假如m00为正数，那么此项最大值为m00 * (center.x + extents.x)

#### 假如m00为负数，那么此项最大值为m00 * (center.x - extents.x)

综上所述，m00的最大值为m00 * (center.x + sign(m00) * extents.x)。

同理与其他两项，可以通过一次多项式运算得出x最大值。最小值和yz的最大最小值也可以同理算出。

这样计算后可得出物品空间的AABB（如下图左），有个问题是由于我们大多阴影都是投影在地面，而物品空间的包围盒一般都没办法覆盖需要投影的地面（如下图右）。

![](http://tangoyzx.github.io/images/posts/post_3.jpg)

（上图中，黑圆为需要生成阴影的物体，蓝框为物体空间的AABB， 褐色线为地面）

AABB的计算就到这里了。这个做法还有一个问题，就是每个Renderer的Bounds都是已经处理到世界空间的AABB，所以最后算出来的物品空间的AABB会有问题（如下图）。

![](http://tangoyzx.github.io/images/posts/post_4.gif)

### 2.1.2 相机与投影矩阵

有了最小包围盒，我们就可以用一个正交相机拍摄包围盒里面的所有东西。首先，正交相机的ViewMatrix就相当于绘制Cube的WorldToLocalMatrix，唯一有点不同就是Unity的Shader里面，Z轴是反的，所以同样也需要对WorldToLocalMatrix的Z轴取反。

正交投影矩阵可以通过AABB来设置。

### 2.1.3 渲染深度图

摄像机都准备好了，可以使用CommandBuffer在Opaque后渲染对应的深度图。

## 2.2 渲染贴花阴影

### 2.2.1 渲染对应的Cube

由于我们使用的是系统自带的Cube，坐标范围是(-0.5~0.5)，要经过缩放变换才能吻合AABB所代表的矩形，当然也可以选择使用自定义的吻合AABB的矩形，但是使用自带的Cube有自己的好处，后面会详述为什么。

### 2.2.2 获取屏幕深度图

众所周知的，Unity在forward rendering path中需要获取深度图的话，首先要把Camera的DepthTextureMode设置正确，然后在Shader中使用_CameraDepthTexture获取。

问题在于_CameraDepthTexture的具体实现方式是在渲染前把所有物体通过另外一个只写深度的Shader渲染一边到一张深度图中，在后面的渲染都使用这张深度图。

为了避免这一次深度的渲染，在某大佬推广中学习到，通过设置Camera的TargetBuffer可以直接获取到相机这次渲染的深度。但在后续地使用过程中法线，无法使用target buffer中的buffer所属的RT（纯黑），所以只能改成每帧交替设置Buffer来使用，这样令到每次获取的深度图都是上一帧的深度图。

虽然每帧节省了一次深度的渲染，但是带来的缺点：双RenderTexture以及延迟获得深度，这两点太致命，特别对于倒推求摄像机空间的坐标而言，后者会导致求的坐标不正确令到整个效果错误。

所以最终还是屈服在_CameraDepthTexture中。

### 2.2.3 倒推摄像机空间的坐标

通过摄像机空间的射线与深度图获得坐标，很多效果都有涉及，这就不赘述了。

### 2.2.4 倒推深度图的UV

假如使用的是默认Cube+缩放变换的组合，这时候可以通过WorldToObject直接获得物品空间的(-0.5, 0.5)的坐标，加上0.5就是深度图的UV了，非常方便。同时深度也可以通过z轴获取，对比以后就可以叠加上阴影的颜色了。当然你也可以做一些Soft的操作。

## 2.3 完成的效果

![](http://tangoyzx.github.io/images/posts/post_5.gif)
