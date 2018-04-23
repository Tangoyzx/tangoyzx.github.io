---
layout:     default
title:      "3ds Max 操作记录"
subtitle:   "常用操作记录"
date:       2018-03-22 20:54:00
author:     "Tango"
tags:
    - 美术
---


选择相关：
    线模式下：
        Ring：选择不相接的一圈线段
        Loop：选择相接的一圈线段
        双击线段：应该是Loop？

修改相关：
    ctrl + backspace：不断开的情况下删除点或者线
    线模式下：
        Connect：选择Ring以后连接每个线段的中点
        Chamfer：把每条线段往左右两侧推生成中间的面
        Bridge：使用两条不相邻的边连成面
    点模式下：
        Make Planar：右边的xyz可以让选中的顶点在对应轴上对齐
        Weld：像Blend的顶点Remove Doubles，可以调阈值

    多边形模式下：
        Cap：生成面
        

Target Weld：顶点焊接

Editable Poly情况下：
Preseve UV可以让顶点移动的同时更新它应该变成的位置 