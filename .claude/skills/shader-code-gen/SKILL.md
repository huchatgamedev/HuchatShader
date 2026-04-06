---
name: shader-code-gen
description: "Unity URP向けのHLSLコードシェーダーを生成・修正するスキル。ユーザーが「シェーダーを作って」「Shaderを書いて」「.shaderファイルを作成して」「頂点シェーダーを実装して」「フラグメントシェーダーを追加して」「マテリアルを作りたい」「描画を変えたい」「見た目を変えたい」などと言ったときにこのスキルを使う。Unlit/Lit/透過/ポストプロセスなどシェーダーの種類を問わず、コードベースのシェーダー生成・修正であればすべてこのスキルの対象。Shader Graphではなくコードで書く場合に使用する。"
---

# Shader Code Generator (URP)

Unity 6 + URP 環境でのHLSLコードシェーダーを生成するスキル。
生成するすべてのシェーダーは、SRP Batcher互換・モバイル対応・学習コメント付きを基本とする。

## 生成前チェックリスト

シェーダーを書き始める前に、以下を必ず確認する。

1. **シェーダーの種類** — Unlit / Lit / 透過 / ポストプロセス / Compute のどれか
2. **必要なPass** — 種類に応じて必要なPassが異なる（後述の一覧を参照）
3. **ライティングの要否** — ライティングが必要なら `Lighting.hlsl` をインクルードする
4. **透過かどうか** — Blend命令、RenderQueue、ZWriteの設定が変わる
5. **モバイル対応** — half精度を積極的に使い、分岐を避ける

## SRP Batcher互換ルール（絶対に守る）

SRP Batcher互換を壊すとバッチングが効かずドローコールが増える。これはパフォーマンスに直結するため、以下を厳守する。

- マテリアルプロパティは必ず `CBUFFER_START(UnityPerMaterial)` / `CBUFFER_END` で囲む
- CBUFFER内に入れるのはスカラー・ベクター・行列のみ（float, half, float4, float4x4 等）
- `TEXTURE2D` / `SAMPLER` 宣言はCBUFFERの外に置く
- `_ST`（Tiling/Offset）変数はCBUFFER内に入れる
- 1シェーダー内でCBUFFER_START(UnityPerMaterial)は1回だけ

```hlsl
// ✅ 正しい
TEXTURE2D(_MainTex);
SAMPLER(sampler_MainTex);

CBUFFER_START(UnityPerMaterial)
    float4 _MainTex_ST;
    half4 _BaseColor;
    half _Smoothness;
CBUFFER_END

// ❌ 間違い — テクスチャがCBUFFER内にある
CBUFFER_START(UnityPerMaterial)
    TEXTURE2D(_MainTex);        // ダメ
    SAMPLER(sampler_MainTex);   // ダメ
    float4 _MainTex_ST;
CBUFFER_END
```

## Pass構成ガイド

シェーダーの種類ごとに必要なPassが異なる。Passが足りないと影が落ちない、Depth Prepassに映らないなどの問題が起きる。

### Opaque（不透明）シェーダー
| Pass | LightMode Tag | 必須度 | 役割 |
|------|--------------|--------|------|
| ForwardLit | `UniversalForward` | 必須 | メイン描画 |
| ShadowCaster | `ShadowCaster` | 強く推奨 | 影を落とす |
| DepthOnly | `DepthOnly` | 推奨 | Depth Prepass / Depth Texture生成 |
| DepthNormals | `DepthNormalsOnly` | 推奨 | SSAO等のポストエフェクト用 |
| Meta | `Meta` | 任意 | ライトマップ / GI用 |

### Transparent（透過）シェーダー
| Pass | LightMode Tag | 必須度 | 備考 |
|------|--------------|--------|------|
| ForwardLit | `UniversalForward` | 必須 | Blend命令を追加 |
| ShadowCaster | — | 通常不要 | 半透明は影を落とさないことが多い |
| DepthOnly | — | 状況次第 | 必要ならAlpha Testと組み合わせ |

### Unlitシェーダー（学習初期）
ForwardLitパスだけでも動作する。学習が進んだらShadowCaster等を追加していく。

## 精度修飾子の使い分け

モバイルGPUではhalf（16bit）とfloat（32bit）で実際に演算精度が異なる。PC GPUでは通常どちらもfloat扱い。

| 用途 | 精度 | 理由 |
|------|------|------|
| 色・ライティング結果 | `half` / `half3` / `half4` | 0〜1の範囲で十分 |
| 法線ベクトル | `half3` | 正規化済みなら-1〜1で十分 |
| ワールド座標 | `float3` | 値が大きくなるため精度が必要 |
| UV座標 | `float2` | タイリングで大きな値になる場合がある |
| 行列演算 | `float` | 精度不足でアーティファクトが出る |
| 時間（_Time） | `float` | 累積値なのでhalfだとすぐ精度が破綻する |

## 座標変換の命名規則

URPでは以下の接尾辞を使う。独自変数でもこれに倣うことでコードの可読性を保つ。

- `OS` — Object Space（モデルのローカル座標）
- `WS` — World Space
- `VS` — View Space（カメラ基準）
- `HCS` — Homogeneous Clip Space（射影後）
- `NDC` — Normalized Device Coordinates
- `SS` — Screen Space

```hlsl
float3 positionWS = TransformObjectToWorld(input.positionOS.xyz);
float4 positionHCS = TransformWorldToHClip(positionWS);
float3 normalWS = TransformObjectToWorldNormal(input.normalOS);
```

## 構造体の命名規則

頂点シェーダーの入力は `Attributes`、出力は `Varyings` とする。これはURP公式の命名に従ったもの。

```hlsl
struct Attributes
{
    float4 positionOS : POSITION;
    float3 normalOS   : NORMAL;
    float4 tangentOS  : TANGENT;
    float2 uv         : TEXCOORD0;
};

struct Varyings
{
    float4 positionHCS : SV_POSITION;
    float2 uv          : TEXCOORD0;
    float3 normalWS    : TEXCOORD1;
    float3 positionWS  : TEXCOORD2;
};
```

## テクスチャサンプリング

URPではマクロ経由でテクスチャを扱う。直接 `tex2D` は使わない。

```hlsl
// 宣言
TEXTURE2D(_BaseMap);
SAMPLER(sampler_BaseMap);

// サンプリング
half4 color = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, uv);

// LOD指定（ミップレベル指定）
half4 color = SAMPLE_TEXTURE2D_LOD(_BaseMap, sampler_BaseMap, uv, mipLevel);
```

## ライティング取得パターン

URPでメインライトとアディショナルライトを取得する方法。`Lighting.hlsl` のインクルードが必要。

```hlsl
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"

// メインライト（ディレクショナルライト）
Light mainLight = GetMainLight();
half3 lightDir = mainLight.direction;
half3 lightColor = mainLight.color;
half shadowAtten = mainLight.shadowAttenuation;
half distAtten = mainLight.distanceAttenuation;

// 影付きメインライト（Shadow Coordが必要）
Light mainLight = GetMainLight(shadowCoord);

// アディショナルライト
int additionalLightsCount = GetAdditionalLightsCount();
for (int i = 0; i < additionalLightsCount; i++)
{
    Light light = GetAdditionalLight(i, positionWS);
    // light.direction, light.color, light.shadowAttenuation, light.distanceAttenuation
}
```

## よくあるミスと対策

| ミス | 症状 | 対策 |
|------|------|------|
| CBUFFERにテクスチャを入れる | SRP Batcher非互換 | テクスチャ宣言はCBUFFER外に |
| `RenderPipeline`タグ忘れ | マゼンタ表示 | `"RenderPipeline" = "UniversalPipeline"` を必ず書く |
| ShadowCaster Pass忘れ | オブジェクトが影を落とさない | ShadowCaster Passを追加 |
| normalOSの正規化忘れ | ライティングが崩れる | `normalize()` を忘れずに |
| _Timeをhalfで受ける | 時間経過で値が飛ぶ | `float` で受ける |
| ZWrite Offの透過でDepth書く | 描画順が壊れる | 透過では `ZWrite Off` |
| Blend命令忘れの透過 | 不透明として描画される | `Blend SrcAlpha OneMinusSrcAlpha` 等 |
| TRANSFORM_TEX忘れ | TilingとOffsetが反映されない | `TRANSFORM_TEX(uv, _MainTex)` を使う |

## 学習コメントの方針

生成するシェーダーには、以下の箇所に日本語で学習コメントを付ける。

- **ファイル先頭**: このシェーダーで学べること（概要）
- **Properties**: 各プロパティの役割
- **Tags**: 各タグの意味と省略した場合の影響
- **頂点シェーダー**: 座標変換の各ステップで何が起きているか
- **フラグメントシェーダー**: 色の計算ロジックの解説
- **新しい概念が初出の箇所**: その概念の背景説明

## テンプレート

シェーダーの種類ごとに詳細なテンプレートを `references/` に用意している。
生成時は該当するテンプレートを読み込んでからコードを書くこと。

- `references/template-unlit.md` — Unlitシェーダーのテンプレート
- `references/template-lit.md` — Litシェーダー（Lambert + Blinn-Phong）のテンプレート
- `references/template-transparent.md` — 透過シェーダーのテンプレート
- `references/template-shadow-passes.md` — ShadowCaster / DepthOnly / DepthNormals Passのテンプレート

生成するシェーダーがどの種類に該当するか判断し、該当するテンプレートファイルを `view` ツールで読み込んでから作業を始める。
