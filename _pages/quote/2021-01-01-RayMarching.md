---
title: 《RayMarching》
author: DH Wang
date: 1000-01-01
category: TA
layout: post
---
 
射线行进是一种相当新的用于渲染实时场景的技术。这项技术特别有趣，因为它完全是在屏幕空间着色器中计算的。换句话说，没有网格数据提供给渲染器，场景绘制在覆盖相机视野的单个四边形上。场景中的对象由一个解析方程定义，该方程描述了场景中任何对象的表面和一个点之间的最短距离射线行进距离场).事实证明，只有这些信息，你才能构成一些惊人的复杂和美丽的场景。此外，因为您没有使用多边形网格(而是使用数学方程)，所以有可能定义非常平滑的表面，这与传统的渲染器不同。


### Code

``` 
    // Raymarch along given ray
    // ro: ray origin
    // rd: ray direction
    fixed4 raymarch(float3 ro, float3 rd) {
        fixed4 ret = fixed4(0,0,0,0);

        const int maxstep = 64;
        float t = 0; // current distance traveled along ray
        for (int i = 0; i < maxstep; ++i) {

            float3 p = ro + rd * t; // World space position of sample
            float d = map(p);       // Sample of distance field (see map())

            // If the sample <= 0, we have hit something (see map()).
            if (d < 0.001) {
                // Simply return a gray color if we have hit an object
                // We will deal with lighting later.
                ret = fixed4(0.5, 0.5, 0.5, 1);
                break;
            }

            // If the sample > 0, we haven't hit anything yet so we should march forward
            // We step forward by distance d, because d is the minimum distance possible to intersect
            // an object (see map()).
            t += d;
        }

        return ret;
    } 
```


### U

1、通常步进长度会采样随机步进，为避免出现格子块状的现象

