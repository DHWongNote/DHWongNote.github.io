---
title: 《ComputeShader》
author: DH Wang
date: 1000-01-01
category: TA
layout: post
---


## 注意事项
1、变量名定义开头不能以数字

threadGroupsXYZ代表线程组的数量。

GPU一次Dispatch会调用64（AMD成为wavefront）或32（NVIDIA称为warp）个线程（这实际上是一种SIMD技术），所以，numthreads的乘积最好是这个值的整数倍。但是Mali不需要这种优化[8]。此外，Metal可以通过api获取这个值[7]。

避免回读：回读操作在渲染管线中使用的比较少，而在CS中可能会被用到，所以重点提一下[20]。

其main(参数：线程id)函数，实际上是单个线程的执行代码

DIspatch一次就会执行一次CS,同时单个线程组分配的线程数也只会执行一次

##  线程参数 

是否支持：SystemInfo.supportsComputeShaders

UnityAPI:OpenGL ES 3.1 (for (Android, iOS, tvOS platforms) only guarantees support for 4 compute buffers at a time. Actual implementations typically support more, but in general if developing for OpenGL ES, you should consider grouping related data in structs rather than having each data item in its own buffer.

ComputeBuffer:In regular graphics shaders the compute buffer support requires minimum shader model 4.5.

## 结构
https://docs.unity3d.com/Manual/class-ComputeShader.html
8
https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/sm5-attributes-numthreads

> Dispatch          =>[5,3,2]   =>  指定分配 线程组数（C#指定）
> > Thread Group    =>[10,8,3]  =>  单个线程组分配 线程数(CS指定)
> > > Thread        =>[7,5,0]   =>  单个线程 组内ID(假定)
> > >
> > > > 单个线程数据:kernel
> > > > > SV_GroupThreadID:     （uint3）   
> > > > > SV_GroupID:           （uint3）   
> > > > > SV_DispatchThreadID:  （uint3）   
> > > > > SV_GroupIndex:        （uint ）  
 
已知

    SV_GroupID = (2,1,0)
    [numthreads = (10,8,3)]
    SV_GroupThreadID = (7,5,0)

则

    SV_DispatchThreadID = ([(2,1,0)*(10,8,3)] + (7,5,0))=(27,13,0)//当前线程的DIspatchThreadID = 单个线程组的总线程数*线程组ID + 当前组内线程ID 
    SV_GroupIndex = 0*10*8 + 5*10 + 7 = 57  //当前z为0代表第一层，同时位于5行7列，即每一行10个一列8个（从线程组指定看出），顺序为先算横竖坐标 再算有一块纵坐标。

常用的就是SV_DispatchThreadID，因为每一个线程是独一无二的

 
## 示例
### 自定义结构赋值
传值buffer有限制，
```
//自定义数据结构
    struct ParticleData {
        public Vector3 pos;
        public Color color;
    } 

//设置ComputeBuffer:（定义100个变量）
   ComputeBuffer mParticleDataBuffer; 
   mParticleDataBuffer = new ComputeBuffer(100,(3+4)*4);//一个float4字节 ->Vector+Color
   mParticleDataBuffer.SetData(new ParticleData[100]);

//分配线程组:（因为数据在GPU中是并行处理，所以下面的线程组数只影响执行快慢）
    computeShader.Dispatch(kernelId, 100, 1, 1);
    computeShader.Dispatch(kernelId, 10, 10, 1);
    computeShader.Dispatch(kernelId, 1, 1, 1);

//执行kernel函数:（单个线程的计算过程如下，通过获取自身组内ID来依次计算传入buffer数据）
    [numthreads(100, 100, 1)]//总的粒子数只有100，但现在单个线程组分配的线程数太多，但也不影响
    [numthreads(100, 1, 1)]//刚好够用,单行或单列100个线程来执行数据计算
    [numthreads(1, 100, 1)]//刚好够用,单行或单列100个线程来执行数据计算
    
    [numthreads(1, 1, 100)]//会报错the final dimension specified (100) for numthreads must be less than or equal to 64 at kernel 
    [numthreads(1, 1, 64)]//会报错the final dimension specified (100) for numthreads must be less than or equal to 64 at kernel 

    [numthreads(10, 10, 1)]//刚好够用，正方形分配，
    void UpdateParticle(uint3 gid : SV_GroupID, uint index : SV_GroupIndex, uint3 gtid : SV_GroupThreadID)
    {
        ParticleBuffer[index].pos = index;
        ParticleBuffer[index].color = float4(gtid,1);
    }

//脚本和Shader中可读取:（该buffer中pos数据由0递增，color数据按行列分布，若线程组z为1则b通道始终为1因为是在当前线程组的第一片中） 
    C#:material.SetBuffer("_particleDataBuffer", mParticleDataBuffer);
    Shader:StructuredBuffer<particleData> _particleDataBuffer;


```


### 纹理赋值

如果我们为每个线程组分配了8x8个线程。假如我们要进行512x512次采样，那么就会有(512x512)/(8x8)=64x64个线程组。

传入CS中的贴图不能只写，也不能只读，即Rendertexture


```
RWTexture2D<float4> xResult;
RWTexture2D<float4> yResult;
RWTexture2D<float4> zResult;

[numthreads(4,8,2)]
void CSMain (uint3 id : SV_DispatchThreadID)
{
    //转为float进行计算
    float x = id.x;
    float y = id.y;
    float z = id.z;
    //根据id的xyz计算颜色值
    float xcol = x/512;
    float ycol = y/512; 
    float zcol = z/2; 
    //采样xyz的颜色值
    xResult[id.xy] = float4(xcol,xcol,xcol,1);
    yResult[id.xy] = float4(ycol,ycol,ycol,1); 
    zResult[id.xy] = float4(zcol,zcol,zcol,1); 
} 
```
xResult：横向渐变图

yResult：纵向渐变图

zResult：随机变换噪点图，CS线程SV_Group是乱序执行的，因为我们每次得到的id.z都不一样，所以zTex的黑灰色块一直变换。

## 。。。