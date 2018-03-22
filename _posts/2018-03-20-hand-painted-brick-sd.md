---
layout:     default
title:      "Hand-Painted Birck with Substance Designer"
subtitle:   " \"码农认知substance designer与手绘风格\""
date:       2018-03-20 22:43:00
author:     "Tango"
tags:
    - 美术
---

# 1 概述

作为一个码农想转TA，所以开始学习各种DCC，打算以tunic为模板做个lowpoly的游戏。结果用blender建完低模后玩上了substance designer一发不可收拾，简直是我这种不会画画的救星。

但是我想象中又想做手绘风格的东西，例如

![UnityStylizedAssets](https://forum.unity.com/proxy.php?image=http%3A%2F%2Fi.imgur.com%2FvtcFbaX.jpg&hash=28091cbcacc5d6c95222a6fece41392f)

Substance Designer做的大多是真实的石头木头这类东西，还好翻了下youtube有个[教风格化砖块的教程](https://www.youtube.com/watch?v=e-_5idWnhio&t=4107s)，把其中Substance Designer的技巧和作者对手绘风的一些见解记录在这里。

# 2 高度图制作

## 2.1 基本砖块

这个其实有很多做法，这个作者的做法主要是为了后面做每个砖块的MASK方便的感觉。

1、先用Tile Random的Square来做无缝的不同亮度的色块，用Level调整一下

2、根据步骤1来做Edge Detect生成砖块

3、步骤2得到的图经过Bevel节点就可以得到对应的Height Map和Normal Map。

4、由步骤1所生成的色块可以通过Histogram Shift + Histogram Scan来选中做Mask的砖块。

![基本砖块](http://tangoyzx.github.io/images/posts/hand-painted-1-1.jpg)


## 2.2 小洞

作者说不管是木头还是石头材质上都会有些小洞，所以下一步就是把这些小洞加上去。

作者使用Shape做了一个Paraboloid作为Pattern给Splatter Circular然后加随机大小随机位置随机半径什么的。

我觉得没什么太大必要，改成了直接使用Tile Random使用Paraboloid加各种随机代替。

所以

1、用Tile Random做一个随机大小随机位置的Paraboloid出来

2、Slope Blur修改一下（我看不出区别……）

3、加个Histogram Scane可以方便控制洞的具体大小

4、增加一个Directional Warp去给小洞修改一下形状，不要都是死板的单位圆

5、对步骤4的图缩小并叠加回去增加细节

6、使用Substract的方式Blend到HeightMap上

![小洞](http://tangoyzx.github.io/images/posts/hand-painted-1-2.jpg)

## 2.3 裂缝

作者好像是用了一个别人的资源来生成不规则满铺色块？

我自己在别的地方看到的不规则满铺色块的具体生成方法

1、用Tile Generator生成随机明暗随机位置的Disc

2、步骤1的图同时连到Distance的Mask和Source，把Maximum Distance调高，选择Only Source，这样能得到一张不规则的平铺色块图了。

3、Edge Detect抠出边缘

4、Slope Blur + Directional Warp调整一下

5、通过2.1步骤1的砖块色块去Directional Warp，这样可以避免裂缝在砖块之间连续

6、通过Perlin + Mosaic Grayscale获取一些随机亮斑

7、步骤6的图同样通过2.1步骤1的砖块色块去Directional Warp

8、用Levels修正步骤7的图

9、用Divide来Blend步骤5和步骤8的图，通过步骤8的Levels可以控制裂缝消失和粗细情况

10、把最终裂缝通过Multiply的方式Blend到Height Map上

11、步骤10以后可以通过Slope Blur和Warp做一下整体Height Map的调整

![裂缝](http://tangoyzx.github.io/images/posts/hand-painted-1-3.jpg)

## 2.4 大片凹凸

1、用2.1步骤1、2同样的方法生成不规则平铺色块图

2、Blur + Slope Blur + Warp调整一下

3、通过2.1步骤1的砖块色块Directional Warp切断砖块间的连续

4、再用一个噪声图添加一下细节，作者用的是Grout的噪声图，但是好像没有保持一致的必要

5、通过Multiply的方式Blend到Height Map上

![大片凹凸](http://tangoyzx.github.io/images/posts/hand-painted-1-4.jpg)

## 2.5 Grout

看了下翻译叫灌浆，总觉得怪怪的就不翻译了……

1、噪声 + Histogram Range + Blur +Mosaic Grayscale生成有一点点亮斑形状的噪声图。

2、Blur以后和一些高频噪声Blend来增加一些细节

3、通过Level控制想要的Grout的形状

4、通过Divide的方式把Grount图放在Background上，Blend到Height Map上。

![Grout](http://tangoyzx.github.io/images/posts/hand-painted-1-5.jpg)

![粗糙度](http://tangoyzx.github.io/images/posts/hand-painted-1-10.jpg)


# 3 基于法线的Mask

先写这个是因为后面颜色部分很多都用上了这堆Mask，


## 3.1 亮部

1、对Normal进行通道分离，分离RGBA通道，并获取G通道。

2、通过Levels获取亮部，因为这代表着法线的y大于0，则比较朝上的，所以可以用来做亮的区域。

## 3.2 暗部

1、用亮部的步骤1分离出来的G通道，用Levels获取暗部，同理这代表法线的y小于0

## 3.3 泛高光区域（不知道怎么叫好，自己乱起的名字）

1、用Curvature Smooth生成砖块凸出边缘较大范围的高光区域

2、用Curvature生成砖块凸出边比较尖锐的高光区域

3、步骤1、2经过Blur以后混合，再与一个噪声混合一下效果。这样就能得到一个边缘很亮，然后渐变到平面的泛高光遮罩

## 3.4 角落高光区域（同样自创名字）

1、同3.3步骤1，用不同的Levels参数把高光范围限制在角落附近。

![法线遮罩](http://tangoyzx.github.io/images/posts/hand-painted-1-6.jpg)

# 4 颜色制作

啊，个人觉得这块才是最有手绘感的地方（这是句废话），

之前自己看着U3D那个手绘风格assets没什么想法，

像作者自己说单纯用Gradient Map做出来也是没有手绘的感觉，

这里按着作者的做法做出来还挺有感觉的

## 4.1 基本颜色

1、选几个相近的颜色做基本基调

2、通过Noise + Blur + Mosaic可以得出一些小块黑白色块的图

3、把步骤1的颜色以步骤2的图为Opacity来Blend的话就会得到充满着小色块的图，按作者的说法有点像用PS不断地添加细节出来的图片。

4、之前一样用Directional Map打破砖块间的连续

5、通过基本砖块步骤4的砖块MASK，调亮部分砖块和调暗部分砖块

## 4.2 添加光照

对，贴图添加光照！

虽说作为PBR贴图本身理论上贴图上不应该有类似烘焙过之后的光照颜色，

但是按作者的话说，由于本来就是手绘风，而且把光照做上去感觉还可以，而且还能继续走PBR流程，

看完效果我也挺同意的

1、使用3.1做Opacity让一个亮色与基本颜色混合

2、使用3.2做Opacity让一个暗色与上一个混合

3、使用3.3做Opacity让另一个亮色与上一个混合

4、使用3.4做Opacity让另另一个亮色与上一个混合

做完这几步基本颜色已经很有手绘风的感觉了

![基本颜色+添加光照](http://tangoyzx.github.io/images/posts/hand-painted-1-7.jpg)

## 4.3 Grout颜色

这步我觉得好像对效果的影响不算很大，可能是我审美不达标的原因……

1、使用4.1的基本颜色

2、类似4.2的步骤1、2一样加上亮部和暗部的颜色，用什么颜色自己考虑了……

3、用2.5步骤4的贴图做Opacity来Blend步骤2的图和4.2步骤4的图。这样能把步骤2的颜色塞到凸起的裂缝里。

4、生成一个AO，取反后作为Opacity给某个颜色Blend到颜色上。生成裂缝比较深坑里面的颜色。

5、生成另一个AO（不同参数），取反后作为Opacity给某个颜色Blend到颜色上。生成裂缝比较浅里面的颜色。

![Grout颜色](http://tangoyzx.github.io/images/posts/hand-painted-1-8.jpg)


# 5 粗糙度

这个比较简单，最终是在大片凹凸这个区域希望高的地方光滑，低的地方粗糙，视频里也是速度过的，这里简单说说

1、大块凹凸步骤4 Blend之前的图取反。

2、步骤1与2.1步骤1的图混合。

3、用步骤2的图通过Substract的方式Blend 3.3的图。其实因为这些区域都定义成了高光，所以不希望他们会粗糙。完成了普遍的粗糙。

4、用4.1步骤2与2.5步骤3的结果混合，做出Grout的粗糙基本图

5、同样Substract 3.3的图，原因一样。

6、通过Levels调整2.5步骤3获取比较高的Grout区域，并且用来作为Opacity把步骤3、5的图Blend在一起。

![粗糙度](http://tangoyzx.github.io/images/posts/hand-painted-1-9.jpg)


# 6 结语

写到后面有点乱，假如有什么不对的希望各位指出。