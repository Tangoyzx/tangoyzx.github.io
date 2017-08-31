---
layout:     default
title:      "ReconstructViewPos"
subtitle:   "重新构造视图空间坐标的方法"
date:       2017-08-31 23:15:00
author:     "Tango"
tags:
    - UnityShader
---

```shaderlab
// Reconstruct view-space position from UV and depth.
// p11_22 = (unity_CameraProjection._11, unity_CameraProjection._22)
// p13_31 = (unity_CameraProjection._13, unity_CameraProjection._23)
float3 ReconstructViewPos(float2 uv, float depth, float2 p11_22, float2 p13_31)
{
    return float3((uv * 2.0 - 1.0 - p13_31) / p11_22 * CheckPerspective(depth), depth);
}
```

做很多屏幕特效时候会需要根据屏幕深度重新获得一次该像素渲染点在屏幕内的坐标，
如SSAO、SSR之类。

我们先从投影矩阵开始

![](http://tangoyzx.github.io/images/posts/post_1.jpg)

其他位置都是正常的投影矩阵推导，
x1(\_13)和y1(\_23)这两个位置的数值用得比较少，
从数值上可以看到改变它们会修改整个画面的偏移，
所以它们是用来偏移整个相机的位置的。

好了我们现在回到这个方法，
uv * 2 - 1明显是把屏幕uv从0~1映射回-1~1，
方便后面的向量运算。
减去p13\_31（这个命名一直很莫名其妙，不是应该p13\_23才对吗？）是由于上面所说把摄像机中心偏移回来
p11_22，本来摄像机空间就是通过tan和aspect映射到-1~1之间，现在除以它们相当于获得一个位置，
也就是z=1平面上当前视线碰撞的点，
然后把这个方向乘以深度就能计算到该像素渲染的摄像机空间的坐标了！
