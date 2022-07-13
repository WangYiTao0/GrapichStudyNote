# URP Custom Postprocess勉強ノート



## URP と Custom Postprocess

最近はURPでCustom Postprocessを勉強しました。URP現在はCustom Postprocessに対応していない状態です。色々調べていました。記録します。

## URP 自身はどうPostprocessするですか

Bloomを例として、流れを見よう。

![image-20220713173636151](https://vip2.loli.io/2022/07/13/Qq1v8RcnXdSA6pr.png)

![image-20220713181218584](https://vip2.loli.io/2022/07/13/5ducZ2ISAOPQ6oN.png)

```c#
    /// <summary>
    /// Renders the post-processing effect stack.
    /// </summary>
    public class PostProcessPass : ScriptableRenderPass
    {
        ...
            MaterialLibrary m_Materials;
        ...
            Bloom m_Bloom;
        ...
        
        //Init Data 
          public PostProcessPass(RenderPassEvent evt, PostProcessData data, Material blitMaterial)
          {
          	//Init Material
              m_Materials = new MaterialLibrary(data);
          }
        
        public void Cleanup() => m_Materials.Cleanup();
        //RenderTextureDesccriptorなど の設定
        public void Setup(...){...}
        //inherit 
        public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData renderingData){}
        //inherit 
        public override void OnCameraCleanup(CommandBuffer cmd){}
        //inherit ここで処理をする。
        public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
        {
            //Get Volume stack
             var stack = VolumeManager.instance.stack;
            //Get Bloom
             m_Bloom = stack.GetComponent<Bloom>();
            
             var cmd = CommandBufferPool.Get();
             using (new ProfilingScope(cmd, m_ProfilingRenderPostProcessing))
             {
                 //Call Render
                  Render(cmd, ref renderingData);
             }

             context.ExecuteCommandBuffer(cmd);
             CommandBufferPool.Release(cmd);
        }
        
        //Blit Image
        private new void Blit(CommandBuffer cmd, RenderTargetIdentifier source, RenderTargetIdentifier destination, Material material, int passIndex = 0)
        {}
        
        //
        void Render(CommandBuffer cmd, ref RenderingData renderingData)
           {
               	...
                // Reset uber keywords
                m_Materials.uber.shaderKeywords = null;

                // Bloom goes first
                bool bloomActive = m_Bloom.IsActive();
                if (bloomActive)
                {
                    using (new ProfilingScope(cmd, ProfilingSampler.Get(URPProfileId.Bloom)))
                        SetupBloom(cmd, GetSource(), m_Materials.uber);
                 ...
                }
           }
        
             void SetupBloom(CommandBuffer cmd, int source, Material uberMaterial)
             {
                            int iterations = Mathf.FloorToInt(Mathf.Log(maxSize, 2f) - 1);
            iterations -= m_Bloom.skipIterations.value;
            int mipCount = Mathf.Clamp(iterations, 1, k_MaxPyramidSize);

            // Pre-filtering parameters
            float clamp = m_Bloom.clamp.value;
            float threshold = Mathf.GammaToLinearSpace(m_Bloom.threshold.value);
            float thresholdKnee = threshold * 0.5f; 
// Pass Parameter to material
                 bloomMaterial.SetVector(ShaderConstants._Params, new Vector4(scatter, clamp, threshold, thresholdKnee));
                 ...
               Blit(cmd, lastDown, mipUp, bloomMaterial, 1);
               Blit(cmd, mipUp, mipDown, bloomMaterial, 2);

                 ...
                 
             }
        
        
        class MaterialLibrary
        {
			...
            public readonly Material bloom;
			...

            public MaterialLibrary(PostProcessData data)
            {
			...
                bloom = Load(data.shaders.bloomPS);
  			...
        }
    }
```

```c#
public class PostProcessData : ScriptableObject
{
     ...
        
        public sealed class ShaderResources
        {
            ...              				         [Reload("Shaders/PostProcessing/Bloom.shader")]
            public Shader bloomPS;
            ...
        }
    
         public ShaderResources shaders;
     ...
｝
```

```c#
using System;

namespace UnityEngine.Rendering.Universal
{
    //Must inherit VolumeComponent
    [Serializable, VolumeComponentMenu("Post-processing/Bloom")]
    public sealed class Bloom : VolumeComponent, IPostProcessComponent
    {
        public MinFloatParameter threshold = new MinFloatParameter(0.9f, 0f);        
        ...
        
        public bool IsActive() => intensity.value > 0f;

        public bool IsTileCompatible() => false;
    }
}

```

この流れで分かっていること

1. Postprocess Effect Classは　VolumeComponent　IPostProcessComponentを継承する

2. Postprocess Effect は VolumeManager.instance.stack にいる。

   ```c#
   //Get Volume stack
   var stack = VolumeManager.instance.stack;
   //Get Bloom
   m_Bloom = stack.GetComponent<Bloom>();
   ```

3. 事前マテリアルを作成して、Postprocess Effect ClassでShaderに必要なパラメータを作って、Renderの時、 パラメータをマテリアルをに渡す

   ```c#
   public sealed class Bloom : VolumeComponent, IPostProcessComponent
   {
       ...
       public MinFloatParameter threshold = new MinFloatParameter(0.9f, 0f);   
       ...
   }
   
   void SetupBloom(CommandBuffer cmd, int source, Material uberMaterial)
   {
        int iterations = Mathf.FloorToInt(Mathf.Log(maxSize, 2f) - 1);
        iterations -= m_Bloom.skipIterations.value;
        int mipCount = Mathf.Clamp(iterations, 1, k_MaxPyramidSize);
   
               // Pre-filtering parameters
               float clamp = m_Bloom.clamp.value;
               float threshold = Mathf.GammaToLinearSpace(m_Bloom.threshold.value);
               float thresholdKnee = threshold * 0.5f; 
   // Pass Parameter to material
                    bloomMaterial.SetVector(ShaderConstants._Params, new Vector4(scatter, clamp, threshold, thresholdKnee));
                }
   ```

## 自分でRadiusBlurを実現してみよう

1. まずは　RadiusBlurEffect　を　VolumeComponent, IPostProcessComponent　継承する。

```c#
    [Serializable, VolumeComponentMenu("Custom/RadiusEffect")]
    public class RadiusBlurEffect : VolumeComponent, IPostProcessComponent
    {
        public ClampedFloatParameter BlurRange = new ClampedFloatParameter(0, 0.0f, 0.6f);
        //public NoInterpColorParameter BlurColor = new NoInterpColorParameter(Color.white);
        public ClampedIntParameter SampleCount = new ClampedIntParameter(4, 4, 16);
        public ClampedFloatParameter BlurCenterX = new ClampedFloatParameter(0.5f, 0.0f, 1f);
        public ClampedFloatParameter BlurCenterY = new ClampedFloatParameter(0.5f, 0.0f, 1f);

        public Vector2 BlurCenter
        {
            get
            {
               return new Vector2(BlurCenterX.value,BlurCenterY.value);
            }
        } 

        public ClampedFloatParameter BlurPower = new ClampedFloatParameter(3, 0.0f, 10.0f);

        public bool IsActive() => BlurRange.value > 0;

        public bool IsTileCompatible() => true;
    }
```

ここでInspectorで確認できます。

![RadiusBlur01](https://vip2.loli.io/2022/07/13/qNkQxwD8fCsF1WP.gif)

2. Shader を書いて、マテリアルを作成する

   ```shader
   Shader "Custom/RadiusBlurShader" 
   {
   	Properties 
   	{
   		[HideInInspector]_MainTex ("Texture", 2D) = "white" {}
   	}
   	SubShader
   	{
   		Tags {"RenderType"="Opaque" "RenderPipeline" = "UniversalPipeline"}
   		
   		HLSLINCLUDE
   		#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
   		#pragma vertex vert
   		#pragma fragment frag
   		
   		struct a2v
   		{
   			float4 position : Position;
   			float2 uv : TEXCOORD;
   		};
   		struct v2f
   		{
   			float4 positionClip : SV_Position;
   			float2 uv : TEXCOORD0;
   		};
   		
   		TEXTURE2D(_MainTex);
           SAMPLER(sampler_MainTex);
           float4 _MainTex_TexelSize;
           float4 _MainTex_ST;
   		
   		float2 _BlurCenter;
   		int _SampleCount;
   		float _BlurRange;
   		float _BlurPower;
   		//float4 _BlurColor;
   		ENDHLSL
   		
   		Pass
   		{
   			Name "Radius Blur"
   			
   			HLSLPROGRAM
   			v2f vert(a2v i)
   	        {
   	            v2f o;
   	            o.positionClip = TransformObjectToHClip(i.position.xyz);
   	            o.uv = TRANSFORM_TEX(i.uv, _MainTex);
   	            return o;
   	        }
   
   			half4 frag(v2f i) : SV_TARGET
   			{
   				float2 dir =  i.uv - _BlurCenter;
   				float2 dir_normal = normalize(dir) * _BlurRange;
   				float4 col;
   				for (int j = 1; j<= _SampleCount * 0.5; j++)
   				{
   					col += SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, i.uv - dir_normal * 0.01 * j);
   				}
   				for (int k = 1; k<= _SampleCount * 0.5; k++)
   				{
   					col += SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, i.uv + dir_normal * 0.01 * k);
   				}
   				col /= _SampleCount;
   				float d = length(dir);
   				col = lerp(SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, i.uv),col
   					,saturate(_BlurPower * d * d));
   				return col;
   			}
   			ENDHLSL
   		}
   	}
   }
   ```

3. CustomPostProcessRenderer と　CustomPostProcessPass　を作成

   ```c#
       [Serializable]
   //単純にCustomPostProcessPassをRender Passに入れる
       public class CustomPostProcessRenderer:ScriptableRendererFeature
       {
           public FeatureSettings settings = new FeatureSettings();
           RenderTargetHandle renderTextureHandle;
           CustomPostProcessPass _customPostProcessPass;
           
           public override void Create()
           {
               _customPostProcessPass = new CustomPostProcessPass(settings.WhenToInsert);
           }
   
           public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
           {
               if (!settings.IsEnabled)
               {
                   // we can do nothing this frame if we want
                   return;
               }
               renderer.EnqueuePass(_customPostProcessPass);
           }
       }
       
       [System.Serializable]
       public class FeatureSettings
       {
           // we're free to put whatever we want here, public fields will be exposed in the inspector
           public bool IsEnabled = true;
           public RenderPassEvent WhenToInsert = RenderPassEvent.AfterRenderingTransparents;
           //public Material MaterialToBlit;
       }
   ```

   ```c#
   using UnityEngine;
   using UnityEngine.Rendering;
   using UnityEngine.Rendering.Universal;
   
   namespace Effect
   {
       public class CustomPostProcessPass : ScriptableRenderPass
       {
           RenderTargetIdentifier _source;
           RenderTargetIdentifier _destinationA;
           RenderTargetIdentifier _destinationB;
           RenderTargetIdentifier _latestDest;
           
           readonly int _temporaryRTIdA = Shader.PropertyToID("_TempRTA");
           readonly int _temporaryRTIdB = Shader.PropertyToID("_TempRTB");
           private CustomPostProcessingMaterials _materials;
           public CustomPostProcessPass(RenderPassEvent settingsWhenToInsert)
           {
               // Set the render pass event
               renderPassEvent = settingsWhenToInsert;
               // Here you get your materials from your custom class
               _materials = CustomPostProcessingMaterials.Instance;
               if (_materials == null)
               {
                   Debug.LogError("Custom Post Processing Materials instance is null");
                   return;
               }
           }
   
           public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData renderingData)
           {
               //Grab the camera target descriptor.
               //We will use this when creating a temporary render texture.
               RenderTextureDescriptor descriptor = renderingData.cameraData.cameraTargetDescriptor;
               descriptor.depthBufferBits = 0;
               
               var renderer = renderingData.cameraData.renderer;
               _source = renderer.cameraColorTarget;
               
               // Create a temporary render texture using the descriptor from above.
               cmd.GetTemporaryRT(_temporaryRTIdA , descriptor, FilterMode.Bilinear);
               _destinationA = new RenderTargetIdentifier(_temporaryRTIdA);
               cmd.GetTemporaryRT(_temporaryRTIdB , descriptor, FilterMode.Bilinear);
               _destinationB = new RenderTargetIdentifier(_temporaryRTIdB);
           }
   
           public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
           {
               // Skipping post processing rendering inside the scene View
               if(renderingData.cameraData.isSceneViewCamera)
                   return;
               
      
               CommandBuffer cmd = CommandBufferPool.Get("Custom Post Processing");
               cmd.Clear();
               
               // This holds all the current Volumes information
               // which we will need later
               var stack = VolumeManager.instance.stack;
               
               #region Local Methods
               // Swaps render destinations back and forth, so that
               // we can have multiple passes and similar with only a few textures
               void BlitTo(Material mat, int pass = 0)
               {
                   var first = _latestDest;
                   var last = first == _destinationA ? _destinationB : _destinationA;
                   Blit(cmd, first, last, mat, pass);
   
                   _latestDest = last;
               }
               #endregion
               // Starts with the camera source
               _latestDest = _source;
               
               //---Custom effect here---
               #region CustomEffect
               
               #region OverlayEffect
      
               var OverlayEffect = stack.GetComponent<OverlayEffect>();
               // Only process if the effect is active
               if (OverlayEffect.IsActive())
               {
                   var material = _materials.OverlayEffect;
                   material.SetFloat(ShaderString.Overlay_IntensityID, OverlayEffect.Intensity.value);
                   material.SetColor(ShaderString.Overlay_OverlayColorID, OverlayEffect.OverlayColor.value);
               
                   BlitTo(material);
               } 
   
               #endregion
               
               #region BoxBlurEffect
   
               var boxBlurEffect = stack.GetComponent<BoxBlurEffect>();
               
               if (boxBlurEffect.IsActive())
               {
                   var material = _materials.BoxBlurEffect;
                   material.SetFloat(ShaderString.BoxBlur_BlurSizeID, boxBlurEffect.BlurSize.value);
               
                   BlitTo(material);
               } 
   
               #endregion
   
               #region RadiusBlurEffect
   
               var radiusBlurEffect = stack.GetComponent<RadiusBlurEffect>();
               // Only process if the effect is active
               if (radiusBlurEffect.IsActive())
               {
                   var material = _materials.RadiusBlurEffect;
                   // P.s. optimize by caching the property ID somewhere else
                   material.SetFloat(ShaderString.RadiusBlur_BlurRangeID, radiusBlurEffect.BlurRange.value);
                   material.SetVector(ShaderString.RadiusBlur_BlurCenterID, radiusBlurEffect.BlurCenter);
                   material.SetInt(ShaderString.RadiusBlur_SampleCountID, radiusBlurEffect.SampleCount.value);
                   material.SetFloat(ShaderString.RadiusBlur_BlurPowerID, radiusBlurEffect.BlurPower.value);
                   BlitTo(material);
               } 
   
               #endregion
    
               #endregion
               
               // DONE! Now that we have processed all our custom effects, applies the final result to camera
               Blit(cmd, _latestDest, _source);
               
               context.ExecuteCommandBuffer(cmd);
               CommandBufferPool.Release(cmd);
          
           }
           public override void OnCameraCleanup(CommandBuffer cmd)
           {
               cmd.ReleaseTemporaryRT(_temporaryRTIdA);
               cmd.ReleaseTemporaryRT(_temporaryRTIdB);
           }
       }
   }
   ```

   

## Project Repo

https://github.com/WangYiTao0/CustomURPPostprocessing

## 参考資料

https://www.febucci.com/2022/05/custom-post-processing-in-urp/

