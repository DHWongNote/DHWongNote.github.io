---
title: 《PBD》
author: DH Wang
date: 1000-01-01
category: TA
layout: post
---

# 基于Position Based Dynamics的布料模拟
 
目前实现的功能有:

- 外力
  - 重力
  - 风力
- 内部约束
  - 距离约束
  - 弯曲约束
- 碰撞约束
  - 与Sphere、Box、Capsule的碰撞
- 固定约束
  - 可以将布料固定到任意位置
- 对场景物体施加反作用力

暂未实现的功能

- 布料自碰撞
- 布料破坏
- GPU模拟版本
- 连续碰撞检测
  



# References

[[1] Matthias Müller, Position Based Dynamics, 2006](https://matthias-research.github.io/pages/publications/posBasedDyn.pdf)
