---
layout:     default
title:      "Tiling within subUV or pseudo-volume textures"
subtitle:   " "
date:       2017-12-13 11:54:00
author:     "Tango"
tags:
    - 图形学翻译
---

[原文在此](http://shaderbits.com/blog/tiling-within-subuv-or-volume-textures "请点")

这篇文章是建立raymarched volumetric效果系列文章的第一篇。subUV纹理和volume纹理有一些相似的地方。假如你想不使用真正硬件3d纹理的情况下来创建一个subUV或者volume纹理的话，在格子的边沿会出现视觉问题。这是很重要的，因为UE4还没有3d纹理格式，尽管以后可能会有。尽管有这样的支持了，对subUV动画使用3d纹理可能并不是最理想的方案，因为你会失去硬件mip的支持。还好我们可以通过少数几个材质指令解决这个问题。

例如只做一个平铺的焦散纹理动画。

SubUV和"pseudo volume txtures" 都有平铺问题，因为他们的的框彼此相邻。这意味着即使每个框本身是无缝的，你始终会看到边沿问题，因为纹素会开始与其他框的错误值进行混合。这里有个利用subUV动画做的焦散渲染的例子，每个框都是一个无缝平铺纹理。

纹理如下：

![](http://storage.googleapis.com/wzukusers/user-22455410/images/57f8326b7740bDzH04F1/SubUV.jpg)

假如你用SubUV函数来显示的话，你会看到角落有边沿问题。

![](http://storage.googleapis.com/wzukusers/user-22455410/images/57f832516c637Bh4wl3L/Tiling_Artifact.gif)

为了找到个解决方案，我想起了个在游戏Monster Truck Madness中的老式技巧。我那时候在Monster Truck Madness 2做了一些修改，所以我需要解决这个问题。

![](http://storage.googleapis.com/wzukusers/user-22455410/images/57f80168df660zdResz1/Mtm.jpg)

在早期GPU里没有太多的原生硬件特性。那个游戏使用地形系统，每个地形系统只能分配一张纹理。也没有任何混合，所以一个地形就是通过大量小而唯一的纹理组成的。一个弯位通常是个6×6的纹理。每个纹理都是64×64的分辨率。开发者希望他们的地形是自然并且有细节的——不管在什么时候。他们层次上通常包含一些自定义的石头切片，它们在很多纹理批次里面很难重用。下面是一个90度弯的切片例子：

![](http://storage.googleapis.com/wzukusers/user-22455410/images/57f850cf1c3f13yNwhWJ/MTM2ED.jpg)

实际上，由于这些巨大的独一的纹理被切分到小的组合，这会导致一个纹理过滤的问题。假如你把这些切片纹理放到一个正常的64×64格子中，这意味着无缝纹理之间会出现过滤的接缝。这些纹理的边缘纹素不应该与附近的过滤。

开发者为此创建了一个解决方案。它们把纹理之间的纹素重复了两次填充在中间，这令到每个纹理有效分辨率从64变成了60.我放大了monster truck madness 2的两个纹理来展现。每一个纹理都是64×64但是只有蓝框里的60×60的区域是唯一的。蓝框外的两边界纹素来自于隔壁：

![](http://storage.googleapis.com/wzukusers/user-22455410/images/57f846b21bbacmjmaMBb/mtm2_tex.gif)

在UE4中重现这个技术是容易和cheap的。

## 方法与限制

最基本的想法是对每个subUV框填充边框边缘像素。点的像素数量取决于无缝mipmap。例如说你每边填充1个边缘纹素（总共2个），就可以修复第一层mipmap。Mip 1以上的依然有缝隙问题。例如说你每边填充1个边缘纹素（总共4个），就可以修复到mipmap 2。mipmap 3需要每边填充四个纹素或者总共8个边缘纹素。这是因为每个mip的分辨率都是一般。为了支持4 mip则需要每边填充8个纹素。5的话则需要16个，如此类推。

基于试验，修正前三个mip是个不错的妥协。超过这个距离的话接缝太明显，而支持更高的mip又牺牲太多分辨率。

## 数学
### 把填充写入纹理

创建填充纹素的数学非常直截了当，我们通过下面公式缩放UV：

UVScale = (EdgePadTexels + TexelsPerFrame) / TexelsPerFrame 

像这样从原点缩放会把所有额外纹素放在右下角，但是我们希望每边各放一半。为了达到这个目的我们需要偏移额外大小的一半：

UVOffset = -0.5 * EdgePadTexels * (1 / TexelsPerFrame ) * TilingPerFrame

参数TilingPerFrame是为了以后再volume纹理上使用而预留的。对于subUV纹理而言他永远都是1，可以无视。

在材质节点中，这是对单框来说可以编码为subUV动画的样子。注意在这个例子里面，TextureRes和Frames是用于仅有单帧的现有纹理创建的新subUV纹理。

![](http://storage.googleapis.com/wzukusers/user-22455410/images/57f8585aaefa2hGS11mn/graph.gif)

当看到这些subUV纹理的帧以后，会发现每帧都缩小了一定并且出现的一些额外的重复。

![](http://storage.googleapis.com/wzukusers/user-22455410/images/57f8480f8016dgFyf4yw/padpixels.gif)

## 读取带有填充的纹理

读取有边缘填充纹理的方式就是你加密方式的反过来：

UVScale = TexelsPerFrame / (TexelsPerFrame + EdgePadTexels)

同样需要添加一半大小的偏移：

UVOffset = 0.5 * EdgePadTexels * (1 / TexelsPerFrame )

使用材质节点SubUV_Function，看起来像这样

![](http://storage.googleapis.com/wzukusers/user-22455410/images/57f85f8de9382Zn4MXaC/SubUVGraph.gif)

注意一个新的输入添加到了SubUV_Function节点：Seamless UVs。这必须是个UV输入，并且匹配一般的UV输入的平铺系数。注意缩放在frac前处理，就是那个选择了Tiling Per UV的节点。

剩下的步骤就是使用我们添加到函数里面的Seamless UVs输入。选择Texture Sample节点并且设置MipValueMode为Derivative。这在hlsl中与SampleGrad一致，你可以指定ddx和ddy来使用。添加ddx和ddy节点并且把它们连接到纹理采样中。

![](http://storage.googleapis.com/wzukusers/user-22455410/images/57f86071778ba9OLXMgo/ddxddy2.gif)

现在我们可以创建并采样无缝平铺SubUV材质了。我们可以看subUV帧之间的角落，已经是无缝的了：

![](http://storage.googleapis.com/wzukusers/user-22455410/images/57f849e975ee9DUtCd12/Tiling_Artifact_Gone.gif)