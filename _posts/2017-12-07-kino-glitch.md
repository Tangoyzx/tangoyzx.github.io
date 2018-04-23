---
layout:     default
title:      "kino-glitch代码解析"
subtitle:   "Shader Code"
date:       2017-08-31 23:15:00
author:     "Tango"
tags:
    - UnityShader
---


（题外话：太懒啦，文章老是写一半，前一篇文章也是窝在电脑好久了，也没翻译完，算是翻译得第二多的文章了。还是偶尔达成一些小目标比较好……）

（所以）今天来解析一下kino glitch的shader效果。

## 源码在这：https://github.com/keijiro/KinoGlitch

先看AnalogGlitch。

c#代码不说了，基本就是生成参数