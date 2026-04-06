# Unlit Shader テンプレート

学習Phase 1で使用する最も基本的なシェーダー。ライティング計算なし。
UV操作、テクスチャサンプリング、色計算の基礎を学ぶのに最適。

## 基本テンプレート

```hlsl
// ============================================================
// [学習メモ] このShaderで学べること:
// - ShaderLabの基本構文（Properties, SubShader, Pass）
// - 頂点シェーダーでの座標変換
// - フラグメントシェーダーでのテクスチャサンプリング
// - SRP Batcher互換のCBUFFER構造
// ============================================================
Shader "Custom/Unlit/TemplateName"
{
    Properties
    {
        // [学習メモ] Propertiesはマテリアルインスペクターに表示されるパラメータ
        // 書式: _変数名 ("表示名", 型) = デフォルト値
        [MainTexture] _BaseMap ("Base Map", 2D) = "white" {}
        [MainColor]   _BaseColor ("Base Color", Color) = (1, 1, 1, 1)
    }

    SubShader
    {
        // [学習メモ] Tags はレンダリングパイプラインにこのシェーダーの性質を伝える
        // RenderPipeline: URPで使うことを宣言（これがないとマゼンタになる）
        // RenderType: カメラのReplacement Shader機能で使われる分類
        // Queue: 描画順。Geometryは不透明オブジェクトの標準キュー (2000)
        Tags
        {
            "RenderPipeline" = "UniversalPipeline"
            "RenderType" = "Opaque"
            "Queue" = "Geometry"
        }

        Pass
        {
            Name "Unlit"
            Tags { "LightMode" = "UniversalForward" }

            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            // [学習メモ] Core.hlslはURPの基本機能を提供する
            // 座標変換関数、共通変数（_Time等）、テクスチャマクロが含まれる
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            // --- 構造体定義 ---

            // [学習メモ] Attributes: メッシュから頂点シェーダーに渡されるデータ
            // セマンティクス（: POSITION等）がGPUに各データの意味を伝える
            struct Attributes
            {
                float4 positionOS : POSITION;   // オブジェクト空間の頂点座標
                float2 uv         : TEXCOORD0;  // UV座標（テクスチャ座標）
            };

            // [学習メモ] Varyings: 頂点シェーダーからフラグメントシェーダーに渡すデータ
            // GPU がラスタライズ時にピクセルごとに補間（Interpolation）する
            struct Varyings
            {
                float4 positionHCS : SV_POSITION; // クリップ空間座標（GPU必須）
                float2 uv          : TEXCOORD0;
            };

            // --- テクスチャ・サンプラー宣言（CBUFFER外） ---
            TEXTURE2D(_BaseMap);
            SAMPLER(sampler_BaseMap);

            // --- マテリアルプロパティ（SRP Batcher互換） ---
            CBUFFER_START(UnityPerMaterial)
                float4 _BaseMap_ST;  // xy: Tiling, zw: Offset
                half4 _BaseColor;
            CBUFFER_END

            // --- 頂点シェーダー ---
            Varyings vert(Attributes input)
            {
                Varyings output;

                // [学習メモ] TransformObjectToHClip:
                // オブジェクト空間 → ワールド空間 → ビュー空間 → クリップ空間
                // この一連の変換を1つの関数で行う（内部でM*V*P行列を掛けている）
                output.positionHCS = TransformObjectToHClip(input.positionOS.xyz);

                // [学習メモ] TRANSFORM_TEX: UV にTilingとOffsetを適用する
                // _BaseMap_STのxyがTiling、zwがOffset
                output.uv = TRANSFORM_TEX(input.uv, _BaseMap);

                return output;
            }

            // --- フラグメントシェーダー ---
            half4 frag(Varyings input) : SV_Target
            {
                // [学習メモ] SAMPLE_TEXTURE2D: テクスチャからピクセルの色を取得する
                // GPUがUV座標に基づいてテクセルを読み取り、フィルタリングして返す
                half4 texColor = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, input.uv);

                return texColor * _BaseColor;
            }
            ENDHLSL
        }
    }

    Fallback "Universal Render Pipeline/Unlit"
}
```

## バリエーション: UV スクロール

Unlitテンプレートに時間ベースのUVスクロールを追加する例。

Properties に追加:
```hlsl
_ScrollSpeed ("Scroll Speed", Vector) = (0.5, 0, 0, 0)
```

CBUFFER に追加:
```hlsl
half2 _ScrollSpeed;
```

頂点シェーダーでUV加算:
```hlsl
// [学習メモ] _Time.y はUnityが毎フレーム更新する経過秒数
// floatで受けること（halfだと約65秒で精度が破綻する）
float2 scrollUV = input.uv + _ScrollSpeed * _Time.y;
output.uv = TRANSFORM_TEX(scrollUV, _BaseMap);
```

## バリエーション: 頂点カラー

Attributes に追加:
```hlsl
float4 color : COLOR;  // メッシュに埋め込まれた頂点カラー
```

Varyings に追加:
```hlsl
half4 vertexColor : COLOR;
```

フラグメントシェーダーで乗算:
```hlsl
return texColor * _BaseColor * input.vertexColor;
```
