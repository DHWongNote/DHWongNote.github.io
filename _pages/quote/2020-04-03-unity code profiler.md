---
title: 《unity中定位某段逻辑的执行效率》
author: DH Wang
date: 1000-01-01
category: TA
layout: post
---
　


# 利用profiler查看执行效率
　　通常我们在unity中处理一段逻辑时会通过profiler来查看它的执行效率，但是往往只能通过看到相对集中的模块，不能精确到一个函数或者一段很小的逻辑。

![](https://dhwblog-1301640854.cos.ap-chongqing.myqcloud.com/picture/20/4/3/0.jpg) 

但是如果想要在update()中查看自定义函数的耗时时间的话那就需要用到性能采样接口Profiler.BeginSample来查看某个函数的执行效率。

![](https://dhwblog-1301640854.cos.ap-chongqing.myqcloud.com/picture/20/4/3/1.jpg) 

当然这样还有个问题就是倘若我的逻辑只执行一帧的话那么又怎么找到呢，因为profiler是针对每一帧来查看的，总不能挨个挨个地查看吧（不嫌麻烦的话），这时候需要我们用到另一个方法来帮助完成EditorApplication.isPaused，这个方法可以暂停运行编辑器来调试，所以说unity很方便，提供了很多的工具来帮助我们完成各种各样的功能需求开发。

![](https://dhwblog-1301640854.cos.ap-chongqing.myqcloud.com/picture/20/4/3/2.jpg) 

因为是在对每一帧的效率计算所以需要定位最后一帧然后再继续运行，这样在下一帧便可通过搜索栏定位到自己的函数了。

![](https://dhwblog-1301640854.cos.ap-chongqing.myqcloud.com/picture/20/4/3/3.jpg) 
 
这样通过Profiler提供的性能采样接口，在profiler中更精确地定位和查看某段逻辑的信息就会变得很方便啦。

[阅读原文](https://www.cnblogs.com/WangDHong/p/12627178.html "博客") 