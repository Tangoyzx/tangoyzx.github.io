---
layout:     default
title:      "Hand-Painted Birck with Substance Designer"
subtitle:   " \"码农认知substance designer与手绘风格\""
date:       2018-03-20 22:43:00
author:     "Tango"
tags:
    - 美术
---

# 概述

作为一个码农想转TA，所以开始学习各种DCC，打算以tunic为模板做个lowpoly的游戏。结果用blender建完低模后玩上了substance designer一发不可收拾，简直是我这种不会画画的救星。

但是我想象中又想做手绘风格的东西，例如

![UnityStylizedAssets](https://forum.unity.com/proxy.php?image=http%3A%2F%2Fi.imgur.com%2FvtcFbaX.jpg&hash=28091cbcacc5d6c95222a6fece41392f)

Substance Designer做的大多是真实的石头木头这类东西，还好翻了下youtube有个[教风格化砖块的教程](https://www.youtube.com/watch?v=e-_5idWnhio&t=4107s)，把其中Substance Designer的技巧和作者对手绘风的一些见解记录在这里。

# 制作

## 基本砖块

这个其实有很多做法，这个作者的做法主要是为了后面做每个砖块的MASK方便的感觉。

因为是先用Tile Random的Square来做无缝的不同亮度的色块，然后再根据这个来做Edge Detect生成砖块，而前者可以通过Histogram Shift + Histogram Scan来选中做Mask的砖块。

还有个点就是用了Bevel Node来生成Height和Normal，感觉是方便调整一点。

看了几个简单的教程的话基本都是只对Height Map做操作，Normal Map中途是不会生成的，基本只有最后一步才会做一次Height Map到Normal Map的转换。

![基本砖块](砖块图片URL)


## 小洞

作者说不管是木头还是石头材质上都会有些小洞，所以下一步就是把这些小洞加上去。

作者使用Shape做了一个Paraboloid作为Pattern给Splatter Circular然后加随机大小随机位置随机半径什么的。

我觉得没什么太大必要，改成了直接使用Tile Random使用Paraboloid加各种随机代替。

然后Slope Blur + Histogram Scan，为了方便控制最后洞的大小。

说到Slope Blur还挺常用的，看了描述大概就是按照Slop Map来进行模糊。

这里其实我完全看不出来Slope Blur了以后有什么区别……