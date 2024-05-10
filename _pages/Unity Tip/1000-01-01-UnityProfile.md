---
title: 【Unity Tip】Profile
author: DH Wang
date:  1000-01-01
category: GAME
layout: post
---

# GpuInstance 

自定义需指定宏，官方介绍
```
//--------------------------------------
// GPU Instancing
#pragma multi_compile_instancing
#pragma instancing_options procedural:setup
```
其中#pragma instancing_options procedural:setup表示每次实例渲染的时候，都会执行以下setup这个函数。

在Builtin渲染管线，也即文档中所说的是，通过unity_InstanceID这个变量。但实测在URP中不起作用。经过搜索发现，可以在顶点函数输入结构中增加如下字段，来获取

```
instanceID:uint instanceID : SV_InstanceID; 
```

 

# Debug DrawCall
## Q:
请问一下性能指标里面的SetPassCalls和Draw Calls、Batches这些有什区别，具体指什么呢？网上有人说是一样的东西，但是我发现有时候值不一样。
## A:
SetPass Call指的是切换渲染状态（render state）的次数，比如你的shader中如果有多个pass，或者是场景中有不同的material，都会造成渲染状态切换。

Drawcall的话，以gles为例，就是调用draw的实际次数，例如drawarray、drawelement，调用一次都会增加。

Batch则是会在第一次调用draw行为的时候加1，如果之后渲染状态没有改变，则batch的数量不再增加，但是一次batch内可能会有多次drawcall调用，只是渲染状态没有改变。