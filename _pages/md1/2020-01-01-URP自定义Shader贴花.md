---
title: 《URP自定义Shader贴花》
author: DH Wang
date: 1000-01-01
category: TA
layout: post
---


# Title


![](https://dhwblog-1301640854.cos.ap-chongqing.myqcloud.com/picture/img/22/3/8/decal.png)

URP中自带的贴花工具仅支持ShaderGraph连线方式的自定义Shader，现通过模板测试和屏幕空间贴花的方式实现一款自定义Shader的贴花工具。在延迟管线下作贴花很容易因为大部分数据都是可以直接从Gbuffer中获取，但是前向中需要进一步转换收集自己需要的一些数据如法线等。



说到底贴花只是一个在物体表面上多渲染一个片而已，只不过这个片的UV是需要进行转化的。关键代码如下：

hlsl:
``` 
void InitDecalData(float3 positionOS,out float4 screenPos ,out float4 viewRayOS ,out float3 cameraPosOS)
{ 
    VertexPositionInputs vertexPositionInput = GetVertexPositionInputs(positionOS);
    screenPos = ComputeScreenPos(vertexPositionInput.positionCS);
    float3 viewRay = vertexPositionInput.positionVS;
    viewRayOS.w = viewRay.z;//store the division value to varying o.viewRayOS.w
    viewRay *= -1;
    float4x4 ViewToObjectMatrix = mul(UNITY_MATRIX_I_M, UNITY_MATRIX_I_V);
    viewRayOS.xyz = mul((float3x3)ViewToObjectMatrix, viewRay);
    cameraPosOS = mul(ViewToObjectMatrix, float4(0,0,0,1)).xyz; // hard code 0 or 1 can enable many compiler optimization
}
 
float2 GetDecalSpaceUV(float4 screenPos , float4 viewRayOS , float3 cameraPosOS)
{
    viewRayOS.xyz /= viewRayOS.w; 
    float2 screenSpaceUV = screenPos.xy / screenPos.w;
    float sceneRawDepth =SAMPLE_TEXTURE2D(_CameraDepthTexture, sampler_CameraDepthTexture,screenSpaceUV).r; 
    float3 decalSpaceScenePos;
    #if _SupportOrthographicCamera      
    if(unity_OrthoParams.w)
    {
    #if defined(UNITY_REVERSED_Z)
    sceneRawDepth = 1-sceneRawDepth;
    #endif
    float sceneDepthVS = lerp(_ProjectionParams.y, _ProjectionParams.z, sceneRawDepth);
    float2 viewRayEndPosVS_xy = float2(unity_OrthoParams.xy * (screenPos.xy - 0.5) * 2 /* to clip space */);  // Ortho near/far plane xy pos 
    float4 vposOrtho = float4(viewRayEndPosVS_xy, -sceneDepthVS, 1);                                            // Constructing a view space pos
    float3 wposOrtho = mul(UNITY_MATRIX_I_V, vposOrtho).xyz;                                                 // Trans. view space to world space
    decalSpaceScenePos = mul(GetWorldToObjectMatrix(), float4(wposOrtho, 1)).xyz;
    }
    else
    {
    #endif
    float sceneDepthVS = LinearEyeDepth(sceneRawDepth,_ZBufferParams);
    decalSpaceScenePos = cameraPosOS + viewRayOS.xyz * sceneDepthVS;
    #if _SupportOrthographicCamera
    }
    #endif
    float2 decalSpaceUV = decalSpaceScenePos.xy + 0.5;
    float shouldClip = 0;
    #if _ProjectionAngleDiscardEnable
    float3 decalSpaceHardNormal = normalize(cross(ddx(decalSpaceScenePos), ddy(decalSpaceScenePos)));//reconstruct scene hard normal using scene pos ddx&ddy
    shouldClip = decalSpaceHardNormal.z > _ProjectionAngleDiscardThreshold ? 0 : 1;
    #endif

    clip(0.5 - abs(decalSpaceScenePos) - shouldClip); 
    return decalSpaceUV.xy;
}

// 需要计算Ray和屏幕空间
float3 GetWPos(float3 ray, float2 screenuv)
{ 
    ray = ray * (_ProjectionParams.z / ray.z);

    float z = SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, sampler_CameraDepthTexture, screenuv); 
      
    float3 vpos = ComputeViewSpacePosition(ray,screenuv);
    float3 wpos = mul(unity_CameraToWorld, float4(vpos, 1)).xyz;
    return wpos; 
}

 
```
shader:
```   
    Stencil
    {
        Ref 5
        Comp Equal
    }  
    ZWrite off
    Blend[_SrcBlend][_DstBlend]

        
    v2f vert(appdata input)
    {
        v2f o; 
        InitDecalData(input.positionOS,o.screenPos,o.viewRayOS,o.cameraPosOS); 
        InitOrientation(o.orientation,o.orientationX,o.orientationZ,input.positionOS); 
            
        VertexPositionInputs vertexInput = GetVertexPositionInputs(input.positionOS.xyz);
        half3 viewDirWS = GetCameraPositionWS() - vertexInput.positionWS;

        o.positionCS =vertexInput.positionCS;   
        o.viewDir = viewDirWS; 
        return o;
    }

    half4 frag(v2f i) : SV_Target
    { 
        half3 worldNormal =0;
        half3 viewDir = normalize(i.viewDir);
        half3 ray =half3(i.orientation.w,i.orientationX.w,i.orientationZ.w);
        float2 uv =GetDecalSpaceUV(i.screenPos , i.viewRayOS , i.cameraPosOS);
        float2 screenuv = i.positionCS.xy*(_ScreenParams.zw-1);
        float3 WorldPos = GetWPos(ray,screenuv);  
            
        half4 BaseColor = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex,uv)*_Color;  
    } 

```