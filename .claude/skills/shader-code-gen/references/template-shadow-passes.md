# Shadow / Depth パステンプレート

ForwardLit以外のパスのテンプレート集。
これらを追加することで影・Depth Prepass・SSAO等が正しく動作する。

## なぜ追加パスが必要なのか

URPは複数のパスでシーンを描画する。ForwardLitだけでは「このオブジェクトの見た目」しか描けない。
影を落としたり、ポストエフェクトが正しく動くには、専用のパスでDepthやNormalを書き出す必要がある。

---

## ShadowCaster Pass

影を落とすために必要。このパスがないとオブジェクトが影を生成しない。

```hlsl
Pass
{
    Name "ShadowCaster"
    Tags { "LightMode" = "ShadowCaster" }

    // [学習メモ] シャドウマップはデプスだけ書くのでColorMask 0
    // 色は書かず、深度値のみをレンダーターゲットに出力する
    ZWrite On
    ZTest LEqual
    ColorMask 0
    Cull Back

    HLSLPROGRAM
    #pragma vertex ShadowPassVertex
    #pragma fragment ShadowPassFragment

    #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
    #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Shadows.hlsl"

    // [学習メモ] ShadowCasterでもCBUFFERが必要（SRP Batcher互換のため）
    // ForwardLitと同じCBUFFER内容を宣言する
    CBUFFER_START(UnityPerMaterial)
        float4 _BaseMap_ST;
        half4 _BaseColor;
        // ... ForwardLitと同じプロパティをすべて含める
    CBUFFER_END

    struct Attributes
    {
        float4 positionOS : POSITION;
        float3 normalOS   : NORMAL;
    };

    struct Varyings
    {
        float4 positionHCS : SV_POSITION;
    };

    Varyings ShadowPassVertex(Attributes input)
    {
        Varyings output;

        float3 positionWS = TransformObjectToWorld(input.positionOS.xyz);
        float3 normalWS = TransformObjectToWorldNormal(input.normalOS);

        // [学習メモ] ApplyShadowBias:
        // シャドウマップのアーティファクト（シャドウアクネ）を軽減するため
        // 法線方向とライト方向にバイアスをかけて頂点をずらす
        float4 positionCS = TransformWorldToHClip(
            ApplyShadowBias(positionWS, normalWS, _MainLightPosition.xyz)
        );

        // [学習メモ] プラットフォームによるクリップ空間のZ範囲の違いを吸収
        #if UNITY_REVERSED_Z
            positionCS.z = min(positionCS.z, UNITY_NEAR_CLIP_VALUE);
        #else
            positionCS.z = max(positionCS.z, UNITY_NEAR_CLIP_VALUE);
        #endif

        output.positionHCS = positionCS;
        return output;
    }

    half4 ShadowPassFragment(Varyings input) : SV_TARGET
    {
        return 0;
    }
    ENDHLSL
}
```

### Alpha Test付きShadowCaster

CutoutシェーダーではShadowCasterでもclipを行う:

```hlsl
// Attributesにuv追加
struct Attributes
{
    float4 positionOS : POSITION;
    float3 normalOS   : NORMAL;
    float2 uv         : TEXCOORD0;
};

struct Varyings
{
    float4 positionHCS : SV_POSITION;
    float2 uv          : TEXCOORD0;
};

// テクスチャ宣言追加
TEXTURE2D(_BaseMap);
SAMPLER(sampler_BaseMap);

// フラグメントでclip
half4 ShadowPassFragment(Varyings input) : SV_TARGET
{
    half alpha = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, input.uv).a * _BaseColor.a;
    clip(alpha - _Cutoff);
    return 0;
}
```

---

## DepthOnly Pass

Depth Prepass やカメラのDepth Texture生成に使われる。
ポストプロセス（DOF、Fog等）が正しく動作するために必要。

```hlsl
Pass
{
    Name "DepthOnly"
    Tags { "LightMode" = "DepthOnly" }

    ZWrite On
    ColorMask R
    Cull Back

    HLSLPROGRAM
    #pragma vertex DepthOnlyVertex
    #pragma fragment DepthOnlyFragment

    #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

    CBUFFER_START(UnityPerMaterial)
        float4 _BaseMap_ST;
        half4 _BaseColor;
        // ... ForwardLitと同じプロパティ
    CBUFFER_END

    struct Attributes
    {
        float4 positionOS : POSITION;
    };

    struct Varyings
    {
        float4 positionHCS : SV_POSITION;
    };

    Varyings DepthOnlyVertex(Attributes input)
    {
        Varyings output;
        output.positionHCS = TransformObjectToHClip(input.positionOS.xyz);
        return output;
    }

    half DepthOnlyFragment(Varyings input) : SV_TARGET
    {
        return input.positionHCS.z;
    }
    ENDHLSL
}
```

---

## DepthNormals Pass

SSAOなどのスクリーンスペースエフェクトが法線情報を必要とする場合に使う。

```hlsl
Pass
{
    Name "DepthNormals"
    Tags { "LightMode" = "DepthNormalsOnly" }

    ZWrite On
    Cull Back

    HLSLPROGRAM
    #pragma vertex DepthNormalsVertex
    #pragma fragment DepthNormalsFragment

    #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

    CBUFFER_START(UnityPerMaterial)
        float4 _BaseMap_ST;
        half4 _BaseColor;
        // ... ForwardLitと同じプロパティ
    CBUFFER_END

    struct Attributes
    {
        float4 positionOS : POSITION;
        float3 normalOS   : NORMAL;
    };

    struct Varyings
    {
        float4 positionHCS : SV_POSITION;
        float3 normalWS    : TEXCOORD0;
    };

    Varyings DepthNormalsVertex(Attributes input)
    {
        Varyings output;
        output.positionHCS = TransformObjectToHClip(input.positionOS.xyz);
        output.normalWS = TransformObjectToWorldNormal(input.normalOS);
        return output;
    }

    half4 DepthNormalsFragment(Varyings input) : SV_TARGET
    {
        // [学習メモ] 法線をエンコードしてレンダーターゲットに書き出す
        float3 normalWS = normalize(input.normalWS);
        return half4(normalWS, 0.0);
    }
    ENDHLSL
}
```

---

## 重要: CBUFFERの一致

すべてのPassで `CBUFFER_START(UnityPerMaterial)` の中身を完全に一致させること。
ForwardLitにあるプロパティがShadowCasterのCBUFFERに無いとSRP Batcherが壊れる。

実際に使わないプロパティであっても、宣言だけは全パスに含める。
