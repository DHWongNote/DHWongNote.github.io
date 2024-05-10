---
title: 【Unity Tip】URP
author: DH Wang
date:  1000-01-01
category: GAME
layout: post
---

`PreDepthPass`

```
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class PreDepthFeature : ScriptableRendererFeature
{  
    private PreDepthPass m_PredepthPass = null;
      
    public override void Create()
    {
        // Create the pass...
        if (m_PredepthPass == null)
        {
            m_PredepthPass = new PreDepthPass();
        }  
    }

    /// <inheritdoc/>
    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    { 
        m_PredepthPass.Setup( ); 
        renderer.EnqueuePass(m_PredepthPass); 
    }

    public class PreDepthPass : ScriptableRenderPass
    {  
        public void Setup( )
        {
            renderPassEvent = RenderPassEvent.AfterRenderingPrePasses + 1;

            ConfigureInput(ScriptableRenderPassInput.Depth); 
        }
         
        public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
        { 
        } 
    }
}



```


`过滤LayerMask,以及只获取深度`
 
```
namespace RainyControl
{

	public class RainEffectFeature : ScriptableRendererFeature
	{

		public class RainEffectPass : ScriptableRenderPass
		{  
			const string m_ProfilerTag = "RainyEffectsFeaturePass_CopyDepthWithFilter";


			List<ShaderTagId> m_ShaderTagIdList = new List<ShaderTagId>();
			private int renderingLayerMask;


			FilteringSettings m_TransFilteringSettings;
			ProfilingSampler m_ProfilingSampler; 
			RenderStateBlock m_RenderStateBlock;

			CastMasktype castMasktype;
			public RainEffectPass(int filterlayer, string[] shaderTags,CastMasktype type)
			{ 
				renderPassEvent = RenderPassEvent.AfterRendering; 
				m_ProfilingSampler = new ProfilingSampler(m_ProfilerTag);
				castMasktype = type;
				if (shaderTags != null && shaderTags.Length > 0)
				{
					foreach (var passName in shaderTags)
						m_ShaderTagIdList.Add(new ShaderTagId(passName));
				}
				else
				{
					m_ShaderTagIdList.Add(new ShaderTagId("SRPDefaultUnlit"));
					m_ShaderTagIdList.Add(new ShaderTagId("UniversalForward"));
					m_ShaderTagIdList.Add(new ShaderTagId("UniversalForwardOnly")); 
				}

				renderingLayerMask = filterlayer; 
				 
				m_TransFilteringSettings = new FilteringSettings(RenderQueueRange.transparent, -1, ((uint)1) << renderingLayerMask);
				m_RenderStateBlock = new RenderStateBlock(RenderStateMask.Nothing);
				m_RenderStateBlock.mask |= RenderStateMask.Depth;
				m_RenderStateBlock.depthState = new DepthState(true, CompareFunction.Less);
			}
			 

            public override void Configure(CommandBuffer cmd, RenderTextureDescriptor cameraTextureDescriptor)
            {
                cmd.GetTemporaryRT(Uniforms.CastDepthTex, 1024, 1024, 32, FilterMode.Bilinear, RenderTextureFormat.Depth);
				 
                //set the RT as Render Target
                ConfigureTarget(Uniforms.CastDepthTex, Uniforms.CastDepthTex); 
                //clear the RT
                ConfigureClear(ClearFlag.All, Color.black);
            }
			public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
			{ 
				CommandBuffer cmd = CommandBufferPool.Get(m_ProfilerTag);  
				this.renderPassEvent = renderPassEvent;
				using (new ProfilingScope(cmd, m_ProfilingSampler))
				{
					//Renderering Layer Mask Draw----------------------------------------------------------------------------------------------------------------------------
					FilteringSettings m_FilteringSettingsnew = new FilteringSettings(RenderQueueRange.opaque, -1, ((uint)1) << renderingLayerMask);
					var drawingOpaqueSettings = CreateDrawingSettings(m_ShaderTagIdList, ref renderingData, renderingData.cameraData.defaultOpaqueSortFlags);
					//Transparent----------------------------------------------------------------------------------------------------------------------------
					DrawingSettings drawingTransparentSettings = CreateDrawingSettings(m_ShaderTagIdList, ref renderingData, SortingCriteria.CommonTransparent);
					switch (castMasktype)
					{
						case CastMasktype.Opaque:
							context.DrawRenderers(renderingData.cullResults, ref drawingOpaqueSettings, ref m_FilteringSettingsnew);
							break;
						case CastMasktype.Transparent:
							context.ExecuteCommandBuffer(cmd);
							cmd.Clear(); 
							context.DrawRenderers(renderingData.cullResults, ref drawingTransparentSettings, ref m_TransFilteringSettings, ref m_RenderStateBlock);
							break;
						default:
							context.DrawRenderers(renderingData.cullResults, ref drawingOpaqueSettings, ref m_FilteringSettingsnew);
							context.ExecuteCommandBuffer(cmd);
							cmd.Clear();
							context.DrawRenderers(renderingData.cullResults, ref drawingTransparentSettings, ref m_TransFilteringSettings, ref m_RenderStateBlock);
							break;
					}
				} 
				context.ExecuteCommandBuffer(cmd);
				CommandBufferPool.Release(cmd);  
			}
					 
			public override void FrameCleanup(CommandBuffer cmd)
			{
				cmd.ReleaseTemporaryRT(Uniforms.CastDepthTex);

				if (cmd == null)
					throw new ArgumentNullException("cmd"); 
			}
		}



		public RenderingLayerMask RenderingLayermask;//我想要指定的RenderingLayerMask  
		public string[] CustomShadertags;//Pass name  

		public CastMasktype castMasktype;
		RainEffectPass rainPass;


		 
		private static class Uniforms
		{ 
			internal static readonly int CastDepthTex = Shader.PropertyToID("_CastDepthTex");
		}
		public override void Create()
		{
			rainPass = new RainEffectPass((int)RenderingLayermask, CustomShadertags, castMasktype);
		}


		public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
		{
			if (RainControl.buildcastRT)
			{ 
				renderer.EnqueuePass(rainPass);
			}
		}
		public enum CastMasktype
		{
			Both,
			Opaque,
			Transparent
		}
		public enum RenderingLayerMask
		{ 
			LightLayerDefault,
			LightLayer1,
			LightLayer2, 
            ...
		}
	}

}
```


`SceneCamera`

```
        if (!renderingData.cameraData.isSceneViewCamera)
        {
            renderer.EnqueuePass(copyDepthPass);
        }

```

# RenderToDepthTexture.cs
```
	using System.Collections.Generic;
	using UnityEngine;
	using UnityEngine.Rendering;
	using UnityEngine.Rendering.Universal;

	/* 
	- Renders DepthOnly pass to _CameraDepthTexture, filtered by LayerMask
	- If the depth texture is generated via a Depth Prepass, URP uses the Opaque Layer Mask at the top of the Forward/Universal Renderer asset
	to determine which objects should be rendered to the depth texture. This feature can be used to render objects on *other layers* into the depth texture as well.
	- The Frame Debugger window can be used to check if a project is using a Depth Prepass vs Copy Depth pass, though unsure if that is the same for final build.
	- Also see :
	https://github.com/Unity-Technologies/Graphics/blob/d48795e93b2e62d936ca6fca61364922f8c91285/com.unity.render-pipelines.universal/Runtime/UniversalRenderer.cs#L436
	https://github.com/Unity-Technologies/Graphics/blob/d48795e93b2e62d936ca6fca61364922f8c91285/com.unity.render-pipelines.universal/Runtime/UniversalRenderer.cs#L1174
	Example : 
	- This RenderObjects example from the URP docs allows you to see the character through walls :
	https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@11.0/manual/renderer-features/how-to-custom-effect-render-objects.html
	- But the character will no longer appear in the depth texture if using a Depth Prepass (but still appears if using Copy Depth, just to be confusing! woo~)
	- Any shaders that rely on the depth texture (aka Scene Depth node) now won't include the character. e.g. Water Foam
	- Using this feature can fix that, by allowing us to add the Character layer to the depth texture
	- Note : Probably shouldn't be used with SSAO Depth Normals mode since that makes depth texture generation use DepthNormals instead of DepthOnly passes 
	and might be weird to have objects appear in the depth texture but not the normals texture.
	*/
	public class RenderToDepthTexture : ScriptableRendererFeature {

		public LayerMask layerMask;

		public RenderPassEvent _event = RenderPassEvent.BeforeRenderingSkybox;
		/* 
		- Set to at least AfterRenderingPrePasses if depth texture is generated via a Depth Prepass,
			or it will just be overriden
		- Set to at least BeforeRenderingSkybox if depth texture is generated via a Copy Depth pass,
			Anything before this is already included in the texture! (Though not for Scene View as that always uses a prepass)
		*/

		class RenderToDepthTexturePass : ScriptableRenderPass {

			private ProfilingSampler m_ProfilingSampler;
			private FilteringSettings m_FilteringSettings;
			private List<ShaderTagId> m_ShaderTagIdList = new List<ShaderTagId>();

			private RenderTargetHandle depthTex;

			public RenderToDepthTexturePass(LayerMask layerMask) {
				m_ProfilingSampler = new ProfilingSampler("RenderToDepthTexture");
				m_FilteringSettings = new FilteringSettings(RenderQueueRange.opaque, layerMask);
				m_ShaderTagIdList.Add(new ShaderTagId("DepthOnly")); // Only render DepthOnly pass
				depthTex.Init("_CameraDepthTexture");
			}

			public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData renderingData) {
				ConfigureTarget(depthTex.Identifier());
			}

			public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData) {
				SortingCriteria sortingCriteria = renderingData.cameraData.defaultOpaqueSortFlags;
				DrawingSettings drawingSettings = CreateDrawingSettings(m_ShaderTagIdList, ref renderingData, sortingCriteria);

				CommandBuffer cmd = CommandBufferPool.Get();
				using (new ProfilingScope(cmd, m_ProfilingSampler)) {
					context.DrawRenderers(renderingData.cullResults, ref drawingSettings, ref m_FilteringSettings);
				}

				context.ExecuteCommandBuffer(cmd);
				CommandBufferPool.Release(cmd);
			}

			public override void OnCameraCleanup(CommandBuffer cmd) { }
		}

		RenderToDepthTexturePass m_ScriptablePass;

		public override void Create() {
			m_ScriptablePass = new RenderToDepthTexturePass(layerMask);
			m_ScriptablePass.renderPassEvent = _event;
		}

		public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData) {
			if (!renderingData.cameraData.requiresDepthTexture) return;
			renderer.EnqueuePass(m_ScriptablePass);
		}
	}
```

# Renders Unlit Opaque objects into CameraNormalsTexture for SSAO Depth Normals mode
```
	/*
	Render Unlit Opaque objects into CameraNormalsTexture for SSAO Depth Normals mode. 
	If you need Alpha clipping or vertex displacement, you'll need to use Depth mode,
	or edit the generated shader code to add in the DepthNormals pass instead.
	This is meant as an alternative if you don't need those.
	For depthNormalsMat, I'm using a blank PBR/Lit SG with passIndex set to 4 (corresponds to it's DepthNormals pass).
	*/
	using System.Collections.Generic;
	using UnityEngine;
	using UnityEngine.Rendering;
	using UnityEngine.Rendering.Universal;

	public class UnlitDepthNormalsFeature : ScriptableRendererFeature {
		class UnlitDepthNormalsPass : ScriptableRenderPass {
			
			private Material depthNormalsMat;
			private int passIndex;

			private ProfilingSampler m_ProfilingSampler;
			private FilteringSettings m_FilteringSettings;
			private List<ShaderTagId> m_ShaderTagIdList = new List<ShaderTagId>();

			private RenderTargetHandle depthTex;
			private RenderTargetHandle normalsTex;
			
			public UnlitDepthNormalsPass(Material depthNormalsMat, int passIndex, LayerMask layerMask) {
				this.depthNormalsMat = depthNormalsMat;
				this.passIndex = passIndex;
				m_ProfilingSampler = new ProfilingSampler("UnlitDepthNormals");
				m_FilteringSettings = new FilteringSettings(RenderQueueRange.opaque, layerMask);

				m_ShaderTagIdList.Add(new ShaderTagId("SRPDefaultUnlit"));

				depthTex.Init("_CameraDepthTexture");
				normalsTex.Init("_CameraNormalsTexture");
			}

			public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData renderingData) {
				ConfigureTarget(normalsTex.Identifier(), depthTex.Identifier());
			}

			public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData) {
				SortingCriteria sortingCriteria = renderingData.cameraData.defaultOpaqueSortFlags;

				DrawingSettings drawingSettings = CreateDrawingSettings(m_ShaderTagIdList, ref renderingData, sortingCriteria);
				drawingSettings.overrideMaterial = depthNormalsMat;
				drawingSettings.overrideMaterialPassIndex = passIndex;

				CommandBuffer cmd = CommandBufferPool.Get();
				using (new ProfilingScope(cmd, m_ProfilingSampler)) {
					context.DrawRenderers(renderingData.cullResults, ref drawingSettings, ref m_FilteringSettings);
				}

				context.ExecuteCommandBuffer(cmd);
				CommandBufferPool.Release(cmd);
			}
			
			public override void OnCameraCleanup(CommandBuffer cmd) {

			}
		}

		public Material depthNormalsMat;
		public int passIndex;
		public LayerMask layerMask;

		UnlitDepthNormalsPass m_ScriptablePass;
		
		public override void Create() {
			m_ScriptablePass = new UnlitDepthNormalsPass(depthNormalsMat, passIndex, layerMask);
			
			m_ScriptablePass.renderPassEvent = RenderPassEvent.AfterRenderingPrePasses;
		}

		public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData) {
			if (depthNormalsMat == null) return;
			renderer.EnqueuePass(m_ScriptablePass);
		}
	}

	/*
	In Unlit Shader Graph, can then use a Custom Function (String mode) to sample the SSAO texture :
	#ifdef SHADERGRAPH_PREVIEW
	DirectAO = 1;
	#else
	AmbientOcclusionFactor aoFactor = GetScreenSpaceAmbientOcclusion(ScreenPos);
	DirectAO = aoFactor.directAmbientOcclusion;
	#endif
	Unsure exactly how to combine it with the shader, but a Multiply might do.
	There's also aoFactor.indirectAmbientOcclusion if you want to support that,
	See the UniversalFragmentPBR method in URP's Shader Library Lighting.hlsl.
	*/
```

# Renderer2DBlitTest Trick简单Bilt
```
	// Possible workaround for the 2D Renderers not allowing you to assign Renderer Features yet.
	// Enqueues a ScriptableRenderPass without relying on a Renderer Feature
	// Used with https://github.com/Cyanilux/URP_BlitRenderFeature/blob/master/Blit.cs
	public class Renderer2DBlitTest : MonoBehaviour {
		
		public Blit.BlitSettings settings;
		private Blit.BlitPass blitPass;

		private void OnEnable(){
			blitPass = new Blit.BlitPass(settings.Event, settings, "BlitTest");

			RenderPipelineManager.beginCameraRendering += Test;
		}
		
		private void OnDisable(){
			RenderPipelineManager.beginCameraRendering -= Test;
		}

		private void Test(ScriptableRenderContext context, Camera camera) {
			blitPass.Setup("_CameraColorTexture", "_CameraColorTexture");
			
			camera.GetUniversalAdditionalCameraData().scriptableRenderer.EnqueuePass(blitPass);
		}
	}
```

#
