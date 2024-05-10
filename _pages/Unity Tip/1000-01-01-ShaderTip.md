---
title: 【Unity Tip】ShaderTip
author: DH Wang
date:  1000-01-01
category: GAME
layout: post
---
    

##  . Functions 

UnityObjectToClipPos(v.vertex) |    将模型空间下的顶点转换到齐次裁剪空间    |
UnityObjectToWorldNormal(v.normal) |    将模型空间下的法线转换到世界空间(已归一化)  |
UnityObjectToWorldDir (v.tangent)  |    将模型空间下的切线转换到世界空间(已归一化)  |
UnityWorldSpaceLightDir (i.worldPos)   |    世界空间下顶点到灯光方向的向量(未归一化)    |
UnityWorldSpaceViewDir (i.worldPos)    |    世界空间下顶点到视线方向的向量(未归一化)    |
UNITY_INITIALIZE_OUTPUT(type,name) |    用0初始化结构   |
TRANSFORM_TEX(i.uv,_MainTex)   |    对UV进行Tiling与Offset变换  |
float3 UnityWorldSpaceLightDir( float3 worldPos )  |    返回顶点到灯光的向量    |
ComputeScreenPos(float4 pos)   |    pos为裁剪空间下的坐标位置，返回的是某个投影点下的屏幕坐标位置 由于这个函数返回的坐标值并未除以齐次坐标，所以如果直接使用函数的返回值的话，需要使用：tex2Dproj(_ScreenTexture, uv.xyw); 也可以自己处理其次坐标,使用：tex2D(_ScreenTexture, uv.xy / uv.w)  |
Luminance(float rgb)   |    去色,内部公式为：dot(rgb,fixed3(0.22,0.707,0.071)). |

float3 TransformObjectToWorld (float3 positionOS)   |                      从本地空间变换到世界空间 |
float3 TransformWorldToObject (float3 positionWS)   |                      从世界空间变换到本地空间 |
float3 TransformWorldToView(float3 positionWS)  |                      从世界空间变换到视图空间"    |
float4 TransformObjectToHClip(float3 positionOS)    |                      将模型空间下的顶点变换到齐次裁剪空间 |
float4 TransformWorldToHClip(float3 positionWS) |                      将世界空间下的顶点变换到齐次裁剪空间 |
float4 TransformWViewToHClip (float3 positionVS)    |                      将视图空间下的顶点变换到齐次裁剪空间 |
float3 TransformObjectToWorldNormal (float3 normalOS)   |                      将法线从本地空间变换到世界空间(已归一化) |
 
##  . Variables
Transformations 

UNITY_MATRIX_MVP    |   模型空间>>投影空间   |
UNITY_MATRIX_MV |   模型空间>>观察空间   |
UNITY_MATRIX_V  |   视图空间 |
UNITY_MATRIX_P  |   投影空间 |
UNITY_MATRIX_VP |   视图空间>投影空间    |
unity_ObjectToWorld |   本地空间>>世界空间   |
unity_WorldToObject |   世界空间>本地空间    |
 



Camera and Screen

_WorldSpaceCameraPos    |   主相机的世界坐标，类型：float3  |
UnityWorldSpaceViewDir(i.worldPos)  |   世界空间下的相机方向(顶点到主相机)，类型：float3    |
_ScreenParams   |   屏幕的相关参数，单位为像素。\nx表示屏幕的宽度\ny表示屏幕的高度\nz表示1+1/屏幕宽度\nw表示1+1/屏幕高度    |
 


Time

_Time   |   时间，主要用于在Shader做动画,类型：float4\nx = t/20\ny = t\nz = t*2\nw = t*3    |
_SinTime    |   t是时间的正弦值，返回值(-1~1): \nx = t/8\ny = t/4\nz = t/2\nw = t   |
_CosTime    |   t是时间的余弦值，返回值(-1~1):\nx = t/8\ny = t/4\nz = t/2\nw = t    |
unity_DeltaTime |   dt是时间增量,smoothDt是平滑时间\nx = dt\ny = 1/dt\nz = smoothDt\nz = 1/smoothDt |


Lighting(ForwardBase & ForwardAdd)

_LightColor0    |      主平行灯的颜色值,rgb = 颜色x亮度; a = 亮度   |
_WorldSpaceLightPos0    |      平行灯:\t(xyz=位置,z=0))\n其它类型灯:\t(xyz=位置,z=1)    |
unity_WorldToLight  |      从世界空间转换到灯光空间下，等同于旧版的_LightMatrix0.   |


Fog and Ambient

unity_AmbientSky    |   环境光（Gradient）中的Sky Color |
unity_AmbientEquator    |   环境光（Gradient）中的Equator Color |
unity_AmbientGround |   环境光（Gradient）中的Ground Color  |
UNITY_LIGHTMODEL_AMBIENT    |   环境光(Color)中的颜色，等同于环境光（Gradient）中的Sky Color.   |

##  . Math

abs (x) |   取绝对值，即正值返回正值，负值返回的还是正值\nx值可以为向量 |
acos (x)    |   反余切函数,输入参数范围为[-1, 1],返回[0, π] 区间的角度值    |
any (x) |   如果x=0，或者x中的所有分量为0，则返回0;否则返回1.   |
ceil (x)    |   对x进行向上取整，即x=0.1返回1，x=1.5返回2，x=-0.3返回0  |
clamp(x,a,b)    |   如果 x 值小于 a，则返回 a;如果 x 值大于 b,返回 ;否则，返回 x.   |
clip (x)    |   如果x<0则裁剪掉此片断   |
cross (a,b) |   返回两个三维向量a与b的叉积,结果相当于a.yzx * b.zxy - a.zxy * b.yzx; |
distance (a,b)  |   返回a,b间的距离.    |
exp(x)  |   计算e的x次方，e = 2.71828182845904523536    |
exp2 (x)    |   计算2的x次方    |
floor (x)   |   对x值进行向下取整(去尾求整)\n比如:floor (0.6) = 0.0,floor (-0.6) = -1.0 |
fmod (x,y)  |   返回x/y的余数。如果y为0，结果不可预料,注意！如果x为负值，返回的结果也是负值！   |
frac (x)    |   返回x的小数部分 |
length (v)  |   返回一个向量的模，即 sqrt(dot(v,v)) |
lerp (A,B,alpha)    |   线性插值.\n如果alpha=0，则返回A;\n如果alpha=1，则返回B;\n否则返回A与B的混合值;内部执行:A + alpha*(B-A)  |
max (a,b)   |   比较两个标量或等长向量元素，返回最大值  |
min (a,b)   |   比较两个标量或等长向量元素，返回最小值  |
mul (M,V)   |   表示矩阵M与向量V进行点乘，结果就是对向量V进行M矩阵变换后的值    |
pow (x,y)   |   返回x的y次方    |
reflect(I, N)   |   根据入射光方向向量 I ，和顶点法向量 N ，计算反射光方向向量。其中 I 和 N 必须被归一化，需要非常注意的是，这个 I 是指向顶点的；函数只对三元向量有效。 |
refract(I,N,eta)    |   计算折射向量， I 为入射光线， N 为法向量， eta 为折射系数；其中 I 和 N 必须被归一化，如果 I 和 N 之间的夹角太大，则返回（ 0 ， 0 ， 0 ），也就是没有折射光线； I 是指向顶点的；函数只对三元向量有效。   |
round (x)   |   返回x四舍五入的值   |
rsqrt (x)   |   返回x的平方根倒数,注意x不能为0.相当于 pow(x, -0.5)  |
saturate (x)    |   如果x<0返回0,如果x>1返回1,否则返回x.    |
sqrt (x)    |   返回x的平方根.  |
step (a,b)  |   如果a<b返回1,否则返回0. |
sign (x)    |   如果x=0返回0,如果x>0返回1,如果x<0返回-1.    |
smoothstep (min,max,x)  |   如果 x 比min 小，返回 0\n如果 x 比max 大，返回 1\n如果 x 处于范围 [min，max]中，则返回 0 和 1 之间的值(按值在min和max间的比例).\n如果只想要线性过渡，并不需要平滑的话，可以采用saturate((x - min)/(max - min))  |
 
 
 
##  . Other

CGPROGRAM/ENDCG |   cg代码的开始与结束. |
CGINCLUDE/ENDCG |   通常用于定义多段vert/frag函数，然后这段CG代码会插入到所有Pass的CG中，根据当前Pass的设置来选择加载.  |
HLSLPROGRAM/ENDHLSL |   HLSL代码的开始与结束.   |
HLSLINCLUDE/ENDHLSL |   通常用于定义多段vert/frag函数，然后这段CG代码会插入到所有Pass的CG中，根据当前Pass的设置来选择加载.  |
CBUFFER_START(UnityPerMaterial)/CBUFFER_END |   将材质属性面板中的变量定义在这个常量缓冲区中，用于支持SRP Batcher.  |
Category{}  |   定义一组所有SubShader共享的命令，位于SubShader外面。\n  |
LOD |   Shader LOD，可利用脚本来控制LOD级别，通常用于不同配置显示不同的SubShader。  |
Fallback \"name\"   |   备胎，当Shader中没有任何SubShader可执行时，则执行FallBack。默认值为Off,表示没有备胎。\n示例:FallBack \"Diffuse\"    |
CustomEditor \"name\"   |   自定义材质面板，name为自定义的脚本名称。可利用此功能对材质面板进行个性化自定义。    |
Name \"MyPassName\" |   给当前Pass指定名称，以便利用UsePass进行调用。   |
UsePass \"Shader/NAME\" |   调用其它Shader中的Pass，注意Pass的名称要全部大写！Shader的路径也要写全，以便能找到具体是哪个Shader的哪个Pass。另外加了UsePass后，也要注意相应的Properties要自行添加。   |
GrabPass    |   GrabPass{} 抓取当前屏幕存储到_GrabTexture中，每个有此命令的Shader都会每帧执行。\nGrabPass { \"TextureName\" } 抓取当前屏幕存储到自定义的TextureName中，每帧中只有第一个拥有此命令的Shader执行一次。\nGrabPass也支持Name与Tags。 |


 
##  . Pipline

应用程序阶段（ Application Stage）
几何阶段（ Geometry Stage）
光栅化阶段（Rasterizer Stage
 
0.Application Stage |   此阶段一般由CPU将需要在屏幕上绘制的几何体、摄像机位置、光照纹理等输出到管线的几何阶段。 |
1.模型和视图变换（Model and View Transform）    |   模型和视图变换阶段分为模型变换和视图变换.\n模型变换的目的是将模型从本地空间变换到世界空间当中，而视图变换的目的是将摄像机放置于坐标原点（以使裁剪和投影操作更简单高效），将模型从世界空间变换到相机空间，以便后续步骤的操作。   |
2.顶点着色（Vertex Shading）    |   顶点着色阶段的目的在于确定模型上顶点处的光照效果,其输出结果（颜色、向量、纹理坐标等）会被发送到光栅化阶段以进行插值操作。   |
3.几何、曲面细分着色器  |   【可选项】分为几何着色器(Geometry Shader)和曲面细分着色器(Tessellation Shader)，主要是对顶点进行增加与删除修改等操作.   |
4.投影（Projection）    |   投影阶段分为正交投影与透视投影.\n投影其实就是矩阵变换，最终会使坐标位于归一化的设备坐标中，之所以叫投影就是因为最终Z轴坐标将被舍弃，也就是说此阶段主要的目的是将模型从三维空间投射到了二维的空间中的过程（但是坐标仍然是三维的，只是显示上看是二维的）。    |
5.裁剪（Clipping）  |   裁剪根据图元在视体的位置分为三种情况：\n1.当图元完全位于视体内部，那么它可以直接进行下一个阶段。\n2.当图元完全位于视体外部，则不会进入下一个阶段，直接丢弃。\n3.当图元部分位于视体内部，则需要对位于视体内的图元进行裁剪处理。\n最终的目的就是对部分位于视体内部的图元进行裁剪操作，以使处于视体外部不需要渲染的图元进行裁剪丢弃。  |
6.屏幕映射（Screen Mapping）    |   屏幕映射阶段的主要目的，是将之前步骤得到的坐标映射到对应的屏幕坐标系上。    |
7.三角形设定（Triangle Setup）  |   此阶段主要是将从几何阶段得到的一个个顶点通过计算来得到一个个三角形网格。    |
8.三角形遍历（Triangle Traversal）  |   此阶段将进行逐像素遍历检查操作，以检查出该像素是否被上一步得到的三角形所覆盖，这个查找过程被称为三角形遍历. |
9.像素着色(Pixel Shading)   |   对应于ShaderLab中的frag函数,主要目的是定义像素的最终输出颜色.   |
10.混合（Merging）  |   主要任务是合成当前储存于缓冲器中的由之前的像素着色阶段产生的片段颜色。此阶段还负责可见性问题（深度测试、模版测试等）的处理. |



##  . Macros

SHADER_API_D3D11    |   Direct3D 11 |
SHADER_API_GLCORE   |   桌面OpenGL核心(GL3/4)   |
SHADER_API_GLES |   OpenGl ES 2.0   |
SHADER_API_GLES3    |   OpenGl ES 3.0/3.1   |
SHADER_API_METAL    |   IOS/Mac Metal   |
SHADER_API_VULKAN   |   Vulkan  |
SHADER_API_D3D11_9X |   IOS/Mac Metal   |
SHADER_API_PS4  |   PS4平台,SHADER_API_PSSL同时也会被定义   |
SHADER_API_XBOXONE  |   Xbox One    |
SHADER_API_MOBILE   |   所有移动平台(GLES/GLES3/METAL)");   |
#if SHADER_TARGET < 30  |   对应于#pragma target的值,2.0就是20,3.0就是30    |
#if UNITY_VERSION >= 500    |   Unity版本号判断，500表示5.0.0   |
UNITY_NO_SCREENSPACE_SHADOWS    |   定义移动平台不进行Cascaded ScreenSpace Shadow.  |
UNITY_UI_CLIP_RECT  |   当父级物体有Rect Mask 2D组件时激活" |
UNITY_SHOULD_SAMPLE_SH  |   是否进行计算SH（光照探针与顶点着色）    |
UNITY_SAMPLE_FULL_SH_PER_PIXEL  |   光照贴图uv和来自SHL2的环境颜色在顶点和像素内插器中共享,在启用静态lightmap和LIGHTPROBE_SH时，在像素着色器中执行完整的SH计算。    |
HANDLE_SHADOWS_BLENDING_IN_GI   |   当同时定义了SHADOWS_SCREEN与LIGHTMAP_ON时开启.  |
UNITY_SHADOW_COORDS(N)  |   定义一个float4类型的变量_ShadowCoord,语义为第N个TEXCOORD.   |
V2F_SHADOW_CASTER   |   用于\"LightMode\" = \"ShadowCaster\"中,相当于定义了float4 pos:SV_POSITION.  |




##  . Properties

_Slider (\"Slider\", Range(0, 1)) = 0   |   类型:数值滑动条\n本身还是Float类型，只是通过Range(min,max)来控制滑动条的最小值与最大值  |
_Color(\"Color\", Color) = (1,1,1,1)    |   类型:颜色属性\nCg/HLSL:float4/half4/fixed4  |
_Vector (\"Vector\", Vector) = (0,0,0,0)    |   类型:四维向量\n在Properties中无法定义二维或者三维向量，只能定义四维向量 |
_MainTex (\"Texture\", 2D) = \"white\" {}   |   类型:2D纹理\nCg/HLSL:sampler2D/sampler2D_half/sampler2D_float\n默认值有white、black、gray、bump以及空，空就是white  |
_MainTex3D (\"Texture\", 3D) = \"white\" {} |   类型:3D纹理\nCg/HLSL:sampler3D/sampler3D_half/sampler3D_float   |
_MainCube (\"Texture\", Cube) = \"white\" {}    |   类型:立方体纹理\nCg/HLSL:samplerCUBE/samplerCUBE_half/samplerCUBE_float |
[Header(xxx)]   |   用于在材质面板中当前属性的上方显示标题xxx，注意只支持英文、数字、空格以及下划线 |
[HideInInspector]   |   在材质面板中隐藏此条属性，在不希望暴露某条属性时可以快速将其隐藏    |
[Space(n)]  |   使材质面板属性之前有间隔，n为间隔的数值大小 |
[HDR]   |   标记为属性为高动态范围  |
[PowerSlider(3)]    |   滑条曲率,必须加在range属性前面，用于控制滑动的数值比例  |
[IntRange]  |   必须使用在Range属性之上，以使在材质面板中滑动时只能生成整数 |
[Toggle]    |   开关,加在数值类型前,可使材质面板中的数值变成开关，0是关，1是开  |
[Enum(Off, 0, On, 1)]   |   数值枚举,可直接在cg中使用此关键字来替代数字.    |
[KeywordEnum (Enum0, Enum1, Enum2, Enum3, Enum4, Enum5, Enum6, Enum7, Enum8)]   |   关键字枚举,可最多定义8个,需要#pragma multi_compile _ENUM_ENUM0 _ENUM_ENUM1 ...来依次声明变体关键字. |
[Enum (UnityEngine.Rendering.CullMode)] |   内置枚举,可在Enum()内直接调用Unity内部的枚举.   |
[NoScaleOffset] |   只能加在纹理属性前，使其隐藏材质面板中的Tiling和Offset  |
[Normal]    |   只能加在纹理属性前，标记此纹理是用来接收法线贴图的，当用户指定了非法线的纹理时会在材质面板上进行警告提示    |
[Gamma] |   Float和Vector属性默认情况下不会进行颜色空间转换，可以通过添加[Gamma]来指明此属性为sRGB值    |
[PerRendererData]   |   标记当前属性将以材质属性块的形式来自于每个渲染器数据    |

 
##  . Semantics

float4 vertex : POSITION;   |   顶点的本地坐标  |
uint vid : SV_VertexID; |   顶点的索引ID    |
float3 normal : NORMAL; |   顶点的法线信息  |
float4 tangent : TANGENT;   |   顶点的切线信息  |
float4 texcoord : TEXCOORD0;    |   顶点的UV1信息   |
float4 texcoord1 : TEXCOORD1;   |   顶点的UV2信息   |
float4 texcoord2 : TEXCOORD2;   |   顶点的UV3信息   |
float4 texcoord3 : TEXCOORD3;   |   顶点的UV4信息   |
fixed4 color : COLOR;   |   顶点的顶点色信息    |
float4 pos:SV_POSITION; |   顶点的齐次裁剪空间下的坐标  |
TEXCOORD0~N |   例如TEXCOORD0、TEXCOORD1、TEXCOORD2...等等，主要用于高精度数据  |
COLOR0~N    |   例如COLOR0、COLOR1、COLOR2...等等，主要用于低精度数据   |
float face:VFACE    |   如果渲染表面朝向摄像机，则Face节点输出正值1，如果远离摄像机，则输出负值-1   |
UNITY_VPOS_TYPE screenPos : VPOS    |   1.当前片断在屏幕上的位置(单位是像素,可除以_ScreenParams.xy来做归一化    |
uint vid : SV_VertexID  |   顶点着色器可以接收具有“顶点编号”作为无符号整数的变量,当需要从纹理或ComputeBuffers中获取额外的顶点数据时比较有用，此语义仅支持#pragma target 3.5及以上   |
注意事项    |   1.OpenGL ES2.0支持最多8个\n2.OpenGL ES3.0支持最多16个   |
fixed4 color : SV_Target;   |   默认RenderTarget,也是默认输出的屏幕上的颜色\n同时支持SV_Target1、SV_Target2...等等  |
fixed depth : SV_Depth; |   通过在片断着色器中输出SV_DEPTH语义可以更改像素的深度值,注意此功能相对会消耗性能，在没有特别需求的情况下尽量不要用   |


 
##  . Tags

Tags { \"TagName1\" = \"Value1\" \"TagName2\" = \"Value2\" }            |           Tag的语法结构，通过Tags{}来表示需要添加的标识,大括号内可以添加多组Tag（所以才叫Tags嘛）,名称（TagName）和值（Value）是成对成对出现的，并且全部用字符串表示。    |
\"RenderPipeline\" = \"UniversalPipeline\"          |           渲染管线标记，对应的管线C#代码UniversalRenderPipeline.cs中的Shader.globalRenderPipeline = UniversalPipeline,LightweightPipeline,只有带有UniversalPipeline或LightweightPipeline的Tag的SubShader才会生效. |
Queue           |           渲染队列直接影响性能中的重复绘制，合理的队列可极大的提升渲染效率。\n渲染队列数<=2500的对象都被认为是不透明的物体（从前往后渲染），>2500的被认为是半透明物体（从后往前渲染）。\n\"Queue\" = \"Geometry+1\" 可通过在值后加数字的方式来改变队列。  |
\"Queue\" = \"Background\"          |           值为1000，此队列的对象最先进行渲染。    |
\"Queue\" = \"Geometry\"            |           默认值，值为2000，通常用于不透明对象，比如场景中的物件与角色等。    |
\"Queue\" = \"AlphaTest\"           |           值为2450，要么完全透明要么完全不透明，多用于利用贴图来实现边缘透明的效果，也就是美术常说的透贴。    |
\"Queue\" = \"Transparent\"         |           值为3000，常用于半透明对象，渲染时从后往前进行渲染，建议需要混合的对象放入此队列。  |
\"Queue\" = \"Overlay\"         |           值为4000,此渲染队列用于叠加效果。最后渲染的东西应该放在这里（例如镜头光晕等）。 |
RenderType          |           用来区别这个Shader要渲染的对象是属于什么类别的，你可以想像成是把各种不同的物体按需要的类型来进行分类一样。\n当然你也可以根据需要改成自定义的名称，这样并不会影响到Shader的效果。\n此Tag多用于摄像机的替换材质功能(Camera.SetReplacementShader)。    |
\"RenderType\" = \"Opaque\"         |           大多数不透明着色器。    |
\"RenderType\" = \"Transparent\"            |           大多数半透明着色器，比如粒子、特效、字体等。    |
\"RenderType\" = \"TransparentCutout\"          |           透贴着色器，多用于植被等。  |
\"RenderType\" = \"Background\"         |           多用于天空盒着色器。    |
\"RenderType\" = \"Overlay\"            |           GUI、光晕着色器等。 |
\"RenderType\" = \"TreeOpaque\"         |           Terrain地形中的树干。   |
\"RenderType\" = \"TreeTransparentCutout\"          |           Terrain地形中的树叶。   |
\"RenderType\" = \"TreeBillboard\"          |           Terrain地形中的永对面树。   |
\"RenderType\" = \"Grass\"          |           Terrain地形中的草。 |
\"RenderType\" = \"GrassBillboard\"         |           Terrain地形中的永对面草。   |
DisableBatching         |           在利用Shader在模型的顶点本地坐标下做一些位移动画，而当此模型有批处理时会出现效果不正确的情况，这是因为批处理会将所有模型转换为世界坐标空间，因此“本地坐标空间”将丢失。  |
\"DisableBatching\" = \"True\"          |           禁用批处理。    |
\"DisableBatching\" = \"False\"         |           不禁用批处理。  |
\"DisableBatching\" = \"LODFading\"         |           仅当LOD激活时禁用批处理。   |
ForceNoShadowCasting            |           是否强制关闭投射阴影。  |
\"ForceNoShadowCasting\" = \"True\"         |           强制关闭阴影投射。  |
\"ForceNoShadowCasting\" = \"False\"            |           不关闭阴影投射。    |
IgnoreProjector         |           是否忽略Projector投影器的影响。 |
\"IgnoreProjector\" = \"True\"          |           不受投影器影响。    |
\"IgnoreProjector\" = \"False\"         |           受投影器影响。  |
CanUseSpriteAtlas           |           是否可用于打包图集的精灵。  |
\"CanUseSpriteAtlas\" = \"True\"            |           支持精灵打包图集。  |
\"CanUseSpriteAtlas\" = \"False\"           |           不支持精灵打包图集。    |
PreviewType         |           定义材质面板中的预览的模型显示,默认不写或者不为Plane与Skybox时则为圆球。    |
\"PreviewType\" = \"Plane\"         |           平面。  |
\"PreviewType\" = \"Skybox\"            |           天空盒。    |
PerformanceChecks           |           是否对shader在当前平台进行性能检测，并在材质面板进行警告提示    |
\"PerformanceChecks\" = \"True\"            |           开启性能检测提示    |
\"PerformanceChecks\" = \"False\"           |           关闭性能检测提示    |

ForwardBase             |               用于前向渲染路径，支持环境光、主像素光、球谐光照与烘焙光照。        |
ForwardAdd              |               用于前向渲染路径，支持额外的逐像素光照，每盏灯一个Pass。        |
Deferred                |               用于延迟渲染。      |
ShadowCaster                |               深度渲染与Shadowmap。       |
MotionVectors               |               运动矢量。      |
PrepassBase             |               旧版延迟渲染，法线与高光处理。      |
PrepassFinal                |               旧版延迟渲染，最终颜色。        |
Vertex              |               旧版顶点渲染。      |
VertexLMRGBM                |               旧版顶点渲染，支持烘焙光照。        |
VertexLM                |               旧版顶点渲染，支持烘焙光照，解码为双LDR。       |
Always              |               永远渲染。"     |
\"LightMode\" = \"UniversalForward\"                |               用于前向渲染路径，所有的灯光都在这一个pass中执行，包括GI、自发光、雾效.(在不需要光照的pass中，可以不写LightMode)        |
\"LightMode\" = \"ShadowCaster\"                |               用于生成阴影贴图(灯光视角下的深度信息)      |
\"LightMode\" = \"DepthOnly\"               |               用于生成相机下的深度信息        |
\"LightMode\" = \"Meta\"                |               仅在光照烘焙时才会使用此Pass,用于间接光的反弹.      |
\"LightMode\" = \"Universal2D\"             |               用于URP使用2D渲染器时绘制物体的Pass，不受光照影响.      |