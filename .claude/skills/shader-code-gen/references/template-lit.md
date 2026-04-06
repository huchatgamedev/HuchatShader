# Lit Shader テンプレート

学習Phase 2で使用するライティング対応シェーダー。
Lambert拡散反射 + Blinn-Phong鏡面反射を手書きで実装する。

## 基本テンプレート

```hlsl
// ============================================================
// [学習メモ] このShaderで学べること:
// - URPでのライティング取得（メインライト + アディショナルライト）
// - Lambert拡散反射モデル（N dot L）
// - Blinn-Phong鏡面反射モデル（N dot H）
// - 法線のワールド空間変換
// - 影の受け取り（Shadow Coord）
// ============================================================
Shader "Custom/Lit/TemplateName"
{
    Properties
    {
        [MainTexture] _BaseMap ("Base Map", 2D) = "white" {}
        [MainColor]   _BaseColor ("Base Color", Color) = (1, 1, 1, 1)
        _Smoothness ("Smoothness", Range(0, 1)) = 0.5
        [Normal] _NormalMap ("Normal Map", 2D) = "bump" {}
        _NormalStrength ("Normal Strength", Range(0, 2)) = 1.0
    }

    SubShader
    {
        Tags
        {
            "RenderPipeline" = "UniversalPipeline"
            "RenderType" = "Opaque"
            "Queue" = "Geometry"
        }

        // ================================================
        // Pass 1: ForwardLit — メイン描画
        // ================================================
        Pass
        {
            Name "ForwardLit"
            Tags { "LightMode" = "UniversalForward" }

            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            // [学習メモ] 影を受けるために必要なキーワード
            // _MAIN_LIGHT_SHADOWS: メインライトの影
            // _MAIN_LIGHT_SHADOWS_CASCADE: カスケードシャドウマップ
            // _ADDITIONAL_LIGHTS: 追加ライト対応
            #pragma multi_compile _ _MAIN_LIGHT_SHADOWS _MAIN_LIGHT_SHADOWS_CASCADE
            #pragma multi_compile _ _ADDITIONAL_LIGHTS_VERTEX _ADDITIONAL_LIGHTS
            #pragma multi_compile_fragment _ _SHADOWS_SOFT

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"

            struct Attributes
            {
                float4 positionOS : POSITION;
                float3 normalOS   : NORMAL;
                float4 tangentOS  : TANGENT;
                float2 uv         : TEXCOORD0;
            };

            struct Varyings
            {
                float4 positionHCS  : SV_POSITION;
                float2 uv           : TEXCOORD0;
                float3 positionWS   : TEXCOORD1;
                float3 normalWS     : TEXCOORD2;
                float3 tangentWS    : TEXCOORD3;
                float3 bitangentWS  : TEXCOORD4;
            };

            TEXTURE2D(_BaseMap);
            SAMPLER(sampler_BaseMap);
            TEXTURE2D(_NormalMap);
            SAMPLER(sampler_NormalMap);

            CBUFFER_START(UnityPerMaterial)
                float4 _BaseMap_ST;
                float4 _NormalMap_ST;
                half4 _BaseColor;
                half _Smoothness;
                half _NormalStrength;
            CBUFFER_END

            Varyings vert(Attributes input)
            {
                Varyings output;

                // [学習メモ] 各種座標変換
                // ワールド座標はライティング計算で必要（ライト方向との計算に使う）
                VertexPositionInputs posInputs = GetVertexPositionInputs(input.positionOS.xyz);
                output.positionHCS = posInputs.positionCS;
                output.positionWS = posInputs.positionWS;

                // [学習メモ] 法線・接線のワールド空間変換
                // 法線マップを使うにはTBN行列（Tangent, Bitangent, Normal）が必要
                VertexNormalInputs normalInputs = GetVertexNormalInputs(input.normalOS, input.tangentOS);
                output.normalWS = normalInputs.normalWS;
                output.tangentWS = normalInputs.tangentWS;
                output.bitangentWS = normalInputs.bitangentWS;

                output.uv = TRANSFORM_TEX(input.uv, _BaseMap);

                return output;
            }

            half4 frag(Varyings input) : SV_Target
            {
                // --- テクスチャサンプリング ---
                half4 baseColor = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, input.uv) * _BaseColor;

                // --- 法線マップの適用 ---
                // [学習メモ] 法線マップはTangent Spaceで格納されている
                // TBN行列を使ってWorld Spaceに変換する必要がある
                half3 normalTS = UnpackNormalScale(
                    SAMPLE_TEXTURE2D(_NormalMap, sampler_NormalMap, input.uv),
                    _NormalStrength
                );
                half3x3 TBN = half3x3(
                    normalize(input.tangentWS),
                    normalize(input.bitangentWS),
                    normalize(input.normalWS)
                );
                half3 normalWS = normalize(mul(normalTS, TBN));

                // --- Shadow Coord ---
                // [学習メモ] 影を受けるにはフラグメントのワールド座標から
                // シャドウマップ上の座標を計算する
                float4 shadowCoord = TransformWorldToShadowCoord(input.positionWS);

                // --- メインライト ---
                Light mainLight = GetMainLight(shadowCoord);
                half3 lightDir = normalize(mainLight.direction);
                half3 lightColor = mainLight.color;
                half shadow = mainLight.shadowAttenuation * mainLight.distanceAttenuation;

                // [学習メモ] カメラへの方向ベクトル（鏡面反射に使う）
                half3 viewDir = normalize(GetWorldSpaceNormalizeViewDir(input.positionWS));

                // --- Lambert 拡散反射 ---
                // [学習メモ] N dot L: 法線とライト方向の内積
                // 面がライトを向いているほど明るい。負の値はライトの裏側なので0にクランプ
                half NdotL = saturate(dot(normalWS, lightDir));
                half3 diffuse = baseColor.rgb * lightColor * NdotL * shadow;

                // --- Blinn-Phong 鏡面反射 ---
                // [学習メモ] ハーフベクトル H = normalize(L + V)
                // N dot H が1に近いほどハイライトが強い
                // Smoothnessで指数を制御し、高いほどハイライトが鋭くなる
                half3 halfDir = normalize(lightDir + viewDir);
                half NdotH = saturate(dot(normalWS, halfDir));
                half specPower = exp2(10.0 * _Smoothness + 1.0);
                half3 specular = lightColor * pow(NdotH, specPower) * shadow;

                // --- アディショナルライト ---
                half3 additionalColor = half3(0, 0, 0);
                int lightsCount = GetAdditionalLightsCount();
                for (int i = 0; i < lightsCount; i++)
                {
                    Light addLight = GetAdditionalLight(i, input.positionWS);
                    half addNdotL = saturate(dot(normalWS, addLight.direction));
                    half addAtten = addLight.shadowAttenuation * addLight.distanceAttenuation;
                    additionalColor += baseColor.rgb * addLight.color * addNdotL * addAtten;
                }

                // --- 環境光 ---
                // [学習メモ] SampleSH: 球面調和関数で環境光を近似取得
                // シーンのライティング設定（Environment Lighting）が反映される
                half3 ambient = SampleSH(normalWS) * baseColor.rgb;

                half3 finalColor = diffuse + specular + additionalColor + ambient;
                return half4(finalColor, baseColor.a);
            }
            ENDHLSL
        }

        // ================================================
        // Pass 2, 3, 4: Shadow / Depth パス
        // ================================================
        // ShadowCaster, DepthOnly, DepthNormals は
        // references/template-shadow-passes.md を参照して追加する
    }

    Fallback "Universal Render Pipeline/Lit"
}
```

## 法線マップなしの簡易版

Phase 2の序盤では法線マップを省略して、以下を簡略化できる。

- Attributes から `tangentOS` を削除
- Varyings から `tangentWS`, `bitangentWS` を削除
- TBN行列の構築を省略し、`normalWS = normalize(input.normalWS)` をそのまま使う
- `_NormalMap` 関連のプロパティ・テクスチャ宣言・CBUFFER変数を削除
