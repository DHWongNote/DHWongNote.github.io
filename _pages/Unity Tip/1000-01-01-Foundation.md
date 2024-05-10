---
title: 【Foundation】
author: DH Wang
date:  1000-01-01
category: GAME
layout: post
---
    
# URP


在ComputeShader中首先定义一个可随机写入的纹理，来作为最终输出目标:
```hlsl
RWTexture2D<float4> _Result;
```

在c#中我们需要创建临时RenderTexture来为其做绑定:

```csharp

//定义_ReflectionTex
private int _reflectionTexID = Shader.PropertyToID("_ReflectionTex");

public void Render(CommandBuffer cmd, ref RenderingData renderingData,ref PlanarDescriptor planarDescriptor){
    var reflectionTexDes = renderingData.cameraData.cameraTargetDescriptor;
    reflectionTexDes.enableRandomWrite = true; //开启随机像素写入
    cmd.GetTemporaryRT(_reflectionTexID,reflectionTexDes); //申请临时RT

    //......

    //将临时纹理绑定到ComputeShader的_Result变量上
    cmd.SetComputeTextureParam(_computeShader,kernal,"_Result",_reflectionTexID);
}

//完成渲染后，要正确释放申请的临时RT
public void ReleaseTemporary(CommandBuffer cmd){
    cmd.ReleaseTemporaryRT(_reflectionTexID);
}

```

## CBuffer
Constant Buffer 是GPU中的一处常量缓冲区。 Unity Shader中使用CBUFFER_START和CBUFFER_END来定义缓冲区变量。

当前Unity内部使用的缓冲区有
```
UnityPerCamera
UnityLighting
UnityShadows
UnityPerDraw
UnityPerFrame
UnityPerMaterial
UnityPerObject
``` 
这些缓冲区是根据各自数据的刷新频率来定义的。 例如UnityPerCamera中的数据，仅在渲染的Camera发生变化时刷新。里面存的即是与Camera相关的数据。UnityPerMaterial则在材质球发生变化的时候刷新。
 


# SSAO 
SSAO的基本流程如下:

- 获得屏幕的深度图和法线图
- 利用uv和深度信息，可以重构每个像素在viewspace中的坐标positionVS,这个坐标表示我们要计算的被遮蔽点。
- 利用屏幕法线图，我们同样可以重构每个像素在viewspace中的法线normalVS
- 针对positionVS和normalVS，在法线朝向的半球，按照给定的半径(sampleRadius)，生成若干采样点(samplePositionVS)
- 将samplePositionVS重新投影到屏幕，计算出其对应的sampleUV
- 用sampleUV去深度图采样,得到sampleDepth.
- 利用sampleUV和sampleDepth,我们可以重构出一个hitPositionVS. 这个位置代表了最终计算出来的遮蔽点.
- 有了被遮点(positionVS)和遮蔽点(hitPositionVS)后,我们可以通过公式计算出一个遮蔽系数，这个系数就代表的遮蔽点对被遮点的阴影贡献度。

# SSR/SSPR
- 在屏幕空间，根据深度图和UV信息，可以重建每个像素的世界坐标PositionWS。
- 这时候给定一个平面，就可以计算出PositionWS对应的镜像世界坐标PositionMWS
- 将PositionMWS重新投影到屏幕，得到UV1
- 这样我们就可以知道，UV1处将反射UV处的像素


# HDR (High-Dynamic Range 高动态光照渲染)
与之相对的是LDR(Low-Dynamic Range 低动态光照渲染)。 由于硬件的限制，在过去我们保存一张图片的时候，RGBA每个通道只给予了8bit，也就是说每个颜色分量的取值范围为0-255。其所能表达亮度范围也只有0-255。

但是现实中，亮度范围变化是巨大的。 如何将巨大的亮度变化映射到有限的图片数据中，即为HDR渲染做的事。







# ToneMapping
- [Tone mapping进化论](https://zhuanlan.zhihu.com/p/21983679)
ToneMapping是一种非线性映射曲线，用以将高范围(超过1）的光照数值映射到(0-1)数值区间。
既保证细节不丢失，也可以不使照片失真。
实现思路就是首先要根据当前的场景推算出场景的平均亮度，再根据这个平均亮度选取一个合适的亮度域，再将整个场景映射到这个亮度域得到正确的结果。其中最重要的几个参数： 
Middle grey：整个场景的平均灰度，关系到场景所应处在亮度域。 
Key：场景的Key将决定整个场景的亮度倾向，倾向偏亮亦或是偏暗。 

## 常用算法（四种）
`Reinhard`:
该算法根据经验得来，其基本思想简单粗暴，亮的变暗，暗的变亮，但是缺点就是把所有的颜色都往中间压缩，整体画面会显得灰暗。

            ```
            float3 ReinhardToneMapping(float3 color, float adapted_lum) 
            {
                const float MIDDLE_GREY = 1;
                color *= MIDDLE_GREY / adapted_lum;
                return color / (1.0f + color);
            }
            ```
`CryEngine 2`:
将亮的部分压平,其中adapted_lum是曝光度，曝光度越高整体越亮，比Reinhard的方法更鲜艳。
            ```
            float3 CEToneMapping(float3 color, float adapted_lum) 
            {
                return 1 - exp(-adapted_lum * color);
            }
            ```

`Filmic Tone Mapping`:
游戏《神秘海域》（Uncharted）的公司公开了其用艺术家人肉tone mapping后再进行曲线拟合的算法，这样做可以最大程度在渲染结果上逼近电影级真实，于是称为Filmic tone mapping算法
           
            ```
            float3 F(float3 x)
            {
                const float A = 0.22f;
                const float B = 0.30f;
                const float C = 0.10f;
                const float D = 0.20f;
                const float E = 0.01f;
                const float F = 0.30f;
            
                return ((x * (A * x + C * B) + D * E) / (x * (A * x + B) + D * F)) - E / F;
            }

            float3 UnchartedToneMapping(float3 color,float adapted_lum)
            {
                const float WHITE = 11.2f;
                return F(1.6f * adapted_lum * color) / F(WHITE);
            }
            ```

`ACES`:
美国电影艺术与科学学会后来经过研究，又发明了一个编码系统，Academy Color Encoding System (ACES)，严格来讲，ACES为的是解决所有设备之间的颜色空间转换问题，但我们可以用其中一小部分即可，也就是HDR映射为LDR的算法，可以称为ACES ToneMapping。
            
            ```
            float3 ACESToneMapping(float3 color, float adapted_lum)
            {
            const float A = 2.51f;
            const float B = 0.03f;
            const float C = 2.43f;
            const float D = 0.59f;
            const float E = 0.14f;
            color *= adapted_lum;
            return (color * (A * color + B)) / (color * (C * color + D) + E);
            }
            ```


# Gamma、Linear、sRGB 和Unity Color Space
[Gamma、Linear、sRGB 和Unity Color Space](https://zhuanlan.zhihu.com/p/66558476)
 
`Unity色彩空间:`选择Gamma时，Unity不会在后台将图片进行转换，输入的即使是经过矫正的图片，Unity也不会处理。选择Linear时，Unity会利用GPU对图片进行采样，剔除矫正，转换到线性空间，然后才将此图片传递给Shader处理，处理后的图片会再加上Gamma矫正，再显示到屏幕上。
 

`Gammma:`其中的_Metallic这一项就带了[Gamma]前缀，表示在Lienar Space下Unity要将其认为在sRGB空间，进行Gamma Correction Removed。
扩展：为什么官方源代码中_Metallic项需要加[Gamma]？这和底层的光照计算中考虑能量守恒的部分有关，Metallic代表了物体的“金属度”，如果值越大则反射(高光)越强，漫反射会越弱。在实际的计算中，这个强弱的计算和Color Space有关，所以需要加上[Gamma]项。
```
[Gamma]_Metallic("Metallic",Range(0,1))=0
```





`SRGB:` 所有需要人眼参与被创作出来的纹理，都应是sRGB（如美术画出来的图）。所有通过计算机计算出来的纹理（如噪声，Mask，LightMap）都应是Linear。这很好解释，人眼看东西才需要考虑显示特性和校正的问题。而对计算机来说不需要，在计算机看来只是普通数据，自然直接选择Linear是最好的。
如果贴图代表着我们看到的颜色，则它该被阐释为sRGB，AO和Base Color贴图在创建的时候就已经在sRGB空间了,所以需要勾选SRGB。
如果贴图代表数据则需要精确计算,金属度贴图(Metallic)和粗糙度贴图(Roughness)大多都会保证在线性空间中。

示例1：如果在editor中用Texture2D.getPixels去取值，同一张图是否勾选srgb输出的结果应该都是一样的。【图片记录的只是一个数值信息，读取的时候是0.5就是0.5，只是因为后续不同的处理方法导致了它的值不同】

示例2：法线默认线性数据，如果需要将其他颜色贴图压缩到法线贴图多出富余的其他通道，则需要根据贴图之前的格式考虑使用的时候是否需要伽马校正。







# LUT 滤镜校色
[3DLUT的unity实现](https://zhuanlan.zhihu.com/p/43241990)







# 浮点纹理
如果要支持HDR，那么在渲染阶段，必须以高精度方式保存颜色，而不是传统的8bit。 然后再最后输出阶段，通过ToneMapping映射到屏幕。

opengl贴图格式类型

在以上链接中的internalformat表格中,GL_RGBA16F、GL_RGBA32F都是浮点纹理格式。例如GL_RGBA32F每个通道都是用32bit的浮点数保存，因此一个像素将占据32*4=128bit。 内存占用是传统RGBA8纹理的四倍，对硬件的要求很高。






# HSV to RGBA
色调、饱和度、明度
色调取值范围为0°～360°，标准色间隔60，转化需要分阶段性读取
 


# PBR
双向反射分布函数(BRDF)
Irradiance指某一微小平面所接受到的光线亮度
radiance衡量的是一条传播光线所具有的亮度(不受传播方向影响而改变)

完整的PBR光照着色可以拆分为三个部分实现:

- 1、直接光照(Direct Light),包括各种平行光、点光源等等
- 2、环境间接光镜面反射(Indirect Light Specular)
- 3、环境间接光漫反射(Indirect Light Diffuse)


## 直接光照
```
I=max(0,dot(n,l))*Li
L=f(l,v)*I
``` 
- Li为光源颜色(即辐射亮度)
- I为着色点的收到的辐照度(irradiance)
- f(l,v)我们称为BRDF，即双向反射分布函数。它代表了着色点出射方向辐射亮度与入射方向辐照度的比值。
- L为出射方向(即视线方向)的辐射亮度，代表了最终的颜色.


最常用的总BRDF公式为Cook-Torrance版本 
```
f(l,v)=kd*fd + ks*fs
```
- fd为漫反射BRDF函数        `fd是有很多实现版本的，这里我们使用Lambert版本: fd=DiffuseColor/3.14`
- fs为镜面/高光反射BRDF函数 `Cook-Torrance给出的高光部分的BRDF: fs=D *G *F / 4*dot(n,l)dot(n,v)`
- - D为法线分布函数
- - G为几何遮蔽函数 `后面可和分母除去相同项变为V`
- - F为菲涅尔反射函数

- ks为镜面反射系数
- kd为漫反射系数
## 间接光镜面反射
UE 近似推导

```
L= A*（F0*scale + bias）
```
- 其中F0为垂直材质表面观察时，材质的反射率。不同的材质，这个值是不一样的。通常来说，金属反射率很高，非金属则很低。    `F0 = metalness*Aldebo +(1-metalness)*0.04 `
- 其中A项只与入射光线相关，scale与bias只与视线相关，于是我们可以分别针对A、scale、bias进行预计算。
- 对A项预计算得到具有Mip结构的CubeMap，我们称为IBLSpecularCubeMap，使用视线在Normal下的反射向量进行索引。
- 对scale和bias项，我们将其预计算到同一张贴图的r和g分量中，称作BRDFLUT（网上下载），以(dot(n,v),roughness)进行索引

## 间接漫反射
NITY3D ShadeSH9 属于Irradiance environment maps

漫反射的BRDF函数与view和light都无关，是一个固定常数 1/3.14 ，因此生成预计算贴图也就很简单了。Unity中同样提供了这个预计算功能:

假设我们已有这么一个预计算好的Indirect Diffuse Irradiance Map了。那么只要按法线方向采样，就能得到环境光照的漫反射项。
```
float3 indirectDiff = texCUBElod(_IBLSpec,float4(normalWS,_IBLSpecMaxMip / 3)) * pbrDesc.albedo * INV_PI;
```



## D项的各种公式 
//D项法线微表面分布函数 
//高光

```
float D_Function(float NdotH, float roughness) //GGX
{

    float a2 = roughness * roughness;

    float NdotH2 = NdotH * NdotH;

    float nom = a2; //分子

    float denom = NdotH2 * (a2 - 1) + 1; //分母

    denom = denom * denom * PI;

    return nom / denom;

}

float BeckmannNormalDistribution(float NdotH, float roughness) 
{

    float roughnessSqr = roughness * roughness;

    float NdotHSqr = NdotH * NdotH;

    return max(0.000001, (1.0 / (3.1415926535 * roughnessSqr * NdotHSqr * NdotHSqr))

    * exp((NdotHSqr - 1) / (roughnessSqr * NdotHSqr)));

}


float GaussianNormalDistribution(float NdotH, float roughness)
{

    float roughnessSqr = roughness * roughness;

    float thetaH = acos(NdotH);

    return exp(-thetaH * thetaH / roughnessSqr);

}



float BlinnPhongNormalDistribution(float NdotH, float specularpower, float speculargloss)
{
    float Distribution = pow(NdotH, speculargloss) * specularpower;

    Distribution *= (2 + specularpower) / (2 * 3.1415926535);

    return Distribution;

}


float PhongNormalDistribution(float RdotV, float specularpower, float speculargloss)
{ 
    float Distribution = pow(RdotV, speculargloss) * specularpower;

    Distribution *= (2 + specularpower) / (2 * 3.1415926535);

    return Distribution;

}
 

float TrowbridgeReitzAnisotropicNormalDistribution(float anisotropic, float NdotH, float HdotX, float HdotY,float smoothness)
{

    float aspect = sqrt(1.0h - anisotropic * 0.9h);

    float X = max(.001, sqr(1.0 - smoothness) / aspect) * 5;

    float Y = max(.001, sqr(1.0 - smoothness) * aspect) * 5;
    return 1.0 / (PI * X * Y * sqr(sqr(HdotX / X) + sqr(HdotY / Y) + NdotH * NdotH));

}
 
float WardAnisotropicNormalDistribution(float anisotropic, float NdotL,float NdotV, float NdotH, float HdotX, float HdotY, float smoothness)
{ 
    float aspect = sqrt(1.0h - anisotropic * 0.9h);

    float X = max(.001, sqr(1.0 - smoothness) / aspect) * 5;

    float Y = max(.001, sqr(1.0 - smoothness) * aspect) * 5;

    float exponent = -(sqr(HdotX / X) + sqr(HdotY / Y)) / sqr(NdotH);

    float Distribution = 1.0 / (4.0 * PI * X * Y * sqrt(NdotL * NdotV));

    Distribution *= exp(exponent);

    return Distribution;

}
```

测试高光项
```
//in frag: 
    float D = D_Function(NdotH, roughness); //GGX 
    if (_NDFModel == 1) //Beckmann
    {
        D = BeckmannNormalDistribution(NdotH, roughness);
    }

    else if (_NDFModel == 2) //Phong
    {
        D = PhongNormalDistribution(RdotV, smoothness, _SpecularRange);
    }
    else if (_NDFModel == 3) //Blin-Phong
    {
        D = BlinnPhongNormalDistribution(NdotH, smoothness, _SpecularRange);
    }

    else if (_NDFModel == 4) //Gaussian
    {
        D = GaussianNormalDistribution(NdotH, roughness);
    }
    else if (_NDFModel == 5) //TrowbridgeReitzAnisotropic
    {
        D = TrowbridgeReitzAnisotropicNormalDistribution(_Anisotropic, NdotH, dot(H, i.tangentWS),
        dot(H, i.BtangentWS), smoothness);
    }
    else if (_NDFModel == 6) //WardAnisotropic
    {
        D = WardAnisotropicNormalDistribution(_Anisotropic, NdotL, NdotV, NdotH, dot(H, i.tangentWS),
        dot(H, i.BtangentWS), smoothness);
    }
    if (_OutPutD) 
    { 
        return real4(final_color * D, 1.0); 
    }
```