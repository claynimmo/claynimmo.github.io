---
layout: default
title: Portfolio |  Worldspace
---

[Home](../../../../index.md) / [Smaller Projects](../index.md) /

# Worldspace Shader
This shader was written to tile textures on world scaling rather than local UVs, preventing stretching, making it easier to fit different size and shape components with the same material together. The original hlsl code is given, but was converted to shader graph to natively support URP lighting

<img src="/assets/files/shaders/images/worldtile.png" alt="Aurora shader">

<a href="/assets/files/shaders/WorldTiling.shadergraph" download>
    Download WorldTiling.shadergraph
</a>

```hlsl
Shader "Legacy Shaders/diffuse-worldspace-smooth" {

    Properties
    {
        _Color ("Main Color", Color) = (1,1,1,1)
        _MainTex ("Base (RGB)", 2D) = "white" {}
        _Scale ("Texture Scale", Float) = 1.0
        _Smoothness ("Smoothness", Range(0,1)) = 0.5
    }

    SubShader
    {
        Tags { "RenderType"="Opaque" "Queue"="Geometry"}
        LOD 200

        Pass
        {
            Name "ForwardLit"
            Tags { "LightMode"="UniversalForward" "Queue"="Geometry"}
            ZWrite On

            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/SurfaceData.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/RealtimeLights.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/RealtimeLights.hlsl"

            #pragma multi_compile _ _ADDITIONAL_LIGHTS
            #pragma multi_compile _ _FORWARD_PLUS
            #pragma multi_compile _ LIGHTMAP_ON
            #pragma multi_compile _ DIRLIGHTMAP_COMBINED
            #pragma multi_compile _ LIGHTMAP_SHADOW_MIXING
            #pragma multi_compile _ SHADOWS_SHADOWMASK

            struct Attributes
            {
                float4 positionOS : POSITION;
                float3 normalOS   : NORMAL;
                float2 lightmapUV : TEXCOORD1;

            };

            struct Varyings
            {
                float4 positionHCS : SV_POSITION;
                float3 worldPos    : TEXCOORD0;
                float3 worldNormal : TEXCOORD1;
                float2 lightmapUV  : TEXCOORD2;

            };

            sampler2D _MainTex;
            float4 _MainTex_ST;
            float4 _Color;
            float _Scale;
            float _Smoothness;
            
            Varyings vert (Attributes IN)
            {
                Varyings OUT;
                float3 worldPos = TransformObjectToWorld(IN.positionOS);
                OUT.positionHCS = TransformWorldToHClip(worldPos);
                OUT.worldPos = worldPos;

                OUT.worldNormal = TransformObjectToWorldNormal(IN.normalOS);
                OUT.lightmapUV = IN.lightmapUV;

                return OUT;
            }

            float3 Lighting(float3 albedo, float3 normalWS, Light light)
            {
                float NdotL = dot(normalWS, normalize(light.direction));
                NdotL = saturate((NdotL + 1) * 0.5);
                return albedo * NdotL * light.color * light.distanceAttenuation * light.shadowAttenuation;
            }

            float3 LightLoop(float3 albedo, InputData inputData)
            {
                float3 lighting = 0;

                Light mainLight = GetMainLight();
                lighting += Lighting(albedo, inputData.normalWS, mainLight);

                #if defined(_ADDITIONAL_LIGHTS)
                uint pixelLightCount = GetAdditionalLightsCount();
                LIGHT_LOOP_BEGIN(pixelLightCount)
                    Light additionalLight = GetAdditionalLight(lightIndex, inputData.positionWS, half4(1,1,1,1));
                    lighting += LightingFunction(albedo, inputData.normalWS, additionalLight);
                LIGHT_LOOP_END
                #endif

                return lighting;
            }

            half4 frag (Varyings IN) : SV_Target
            {
                float2 UV;
                float3 n = normalize(IN.worldNormal);
                float3 p = IN.worldPos;

                if (abs(n.x) > 0.5)
                    UV = p.yz;
                else if (abs(n.z) > 0.5)
                    UV = p.xy;
                else
                    UV = p.xz;

                float2 scaledUV = frac(UV * _Scale);
                half4 texColor = tex2D(_MainTex, scaledUV);
                half3 albedo = texColor.rgb * _Color.rgb;

                
                InputData inputData = (InputData)0;
                inputData.positionWS = IN.worldPos;
                inputData.normalWS = normalize(IN.worldNormal);
                inputData.viewDirectionWS = normalize(_WorldSpaceCameraPos - IN.worldPos);
                inputData.shadowCoord = TransformWorldToShadowCoord(IN.worldPos);

                inputData.fogCoord = 0;
                inputData.vertexLighting = VertexLighting(IN.worldPos, inputData.normalWS);

                inputData.normalizedScreenSpaceUV = IN.positionHCS.xy / IN.positionHCS.w;
                inputData.shadowMask = float4(1, 1, 1, 1);
                float3 bakedDirectional = SampleLightmap(IN.lightmapUV, inputData.normalWS);
                float3 ambientSH = SampleSH(inputData.normalWS);
                inputData.bakedGI = lerp(bakedDirectional, ambientSH, 0.7);inputData.positionCS = IN.positionHCS;

                float3 indirect = inputData.bakedGI + inputData.vertexLighting;
                float3 lighting = LightLoop(albedo, inputData) + albedo * indirect;

                half4 finalColor = half4(lighting, 1);
                return finalColor;

            }
            ENDHLSL
        }

        Pass
        {
            Name "DepthOnly"
            Tags { "LightMode" = "DepthOnly" }

            ZWrite On
            ColorMask 0

            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/SurfaceData.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/RealtimeLights.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/RealtimeLights.hlsl"

            struct Attributes
            {
                float4 positionOS : POSITION;
            };

            struct Varyings
            {
                float4 positionHCS : SV_POSITION;
            };

            Varyings vert(Attributes IN)
            {
                Varyings OUT;
                OUT.positionHCS = mul(UNITY_MATRIX_MVP, IN.positionOS);
                return OUT;
            }

            float4 frag(Varyings IN) : SV_Target
            {
                return 0;
            }
            ENDHLSL
        }

        Pass
        {
            Name "ShadowCaster"
            Tags { "LightMode" = "ShadowCaster" }

            ZWrite On
            ColorMask 0

            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #pragma multi_compile_instancing
            #pragma multi_compile _ _CASTING_PUNCTUAL_LIGHT_SHADOW

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
           

            struct Attributes
            {
                float4 positionOS : POSITION;
                float3 normalOS   : NORMAL;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };

            struct Varyings
            {
                float4 positionHCS : SV_POSITION;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };

            Varyings vert(Attributes IN)
            {
                Varyings OUT;
                UNITY_SETUP_INSTANCE_ID(IN);
                UNITY_TRANSFER_INSTANCE_ID(IN, OUT);

                float3 positionWS = TransformObjectToWorld(IN.positionOS.xyz);
                float3 normalWS = TransformObjectToWorldNormal(IN.normalOS);

                OUT.positionHCS = TransformWorldToHClip(positionWS);
                return OUT;
            }

            float4 frag(Varyings IN) : SV_Target
            {
                UNITY_SETUP_INSTANCE_ID(IN);
                return 0;
            }
            ENDHLSL
        }
    }

    FallBack "Hidden/InternalErrorShader"
}
```