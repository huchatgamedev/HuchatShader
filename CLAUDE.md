# Unity Shader Learning Project

## プロジェクト概要
Unity 6 (6000.3.11f1) + URP を使用したShader学習プロジェクト。
コードシェーダーとShader Graphの両方を扱い、基礎から応用へ段階的に学習する。

## 技術スタック
- **Unity**: 6000.3.11f1
- **Render Pipeline**: URP (Universal Render Pipeline)
- **Shader言語**: HLSL / ShaderLab
- **Shader Graph**: 併用（ノードベースでのプロトタイピングやビジュアル確認に使用）
- **カラースペース**: Linear
- **ターゲットプラットフォーム**: PC (Windows/Mac) + モバイル (iOS/Android)

## レンダリング設定
- SRP Batcher: 有効（すべてのコードシェーダーでCBUFFER互換を必須とする）
- GPU Instancing: モバイル向けShaderでは積極的に対応する
- Dynamic Batching: 原則無効（SRP Batcher優先）

## ディレクトリ構成
```
Assets/HuchatShader/
├── Shaders/
│   ├── Common/            # 共有インクルード、ユーティリティ関数
│   ├── Unlit/             # Unlitシェーダー（学習初期）
│   ├── Lit/               # ライティング対応シェーダー
│   ├── Toon/              # トゥーンシェーダー
│   ├── PostProcess/       # ポストプロセス用
│   ├── Compute/           # Compute Shader
│   └── Experimental/      # 実験・検証用（自由に壊してよい）
├── ShaderGraphs/
│   ├── SubGraphs/         # 再利用可能なサブグラフ
│   └── (トピック別フォルダ)
├── Materials/
│   └── (Shaderと対応するフォルダ構成)
├── Scenes/
│   ├── ShaderTest/        # Shader単体テスト用シーン
│   └── Showcase/          # 成果物の展示用シーン
├── Textures/              # テスト用テクスチャ
├── Models/                # テスト用メッシュ（Sphere, Cube, キャラモデル等）
└── Docs/
    └── LearningLog.md     # 学習ログ・メモ
```

## コーディング規約

### ShaderLab / HLSL
- ファイル命名: PascalCase（例: `ToonLighting.shader`）
- Shader名: `Shader "Custom/{カテゴリ}/{名前}"` の形式で統一
- Properties名: `_PascalCase`（例: `_MainTex`, `_BaseColor`）
- 変数名: camelCase（例: `float3 worldNormal`）
- 各Shaderファイルの先頭に学習メモをコメントで記述する
- Fallbackは必ず指定する（`Fallback "Universal Render Pipeline/Lit"` 等）

### SRP Batcher互換ルール（必須）
コードシェーダーでは、マテリアルプロパティを必ず `CBUFFER_START(UnityPerMaterial)` / `CBUFFER_END` で囲む。
テクスチャ宣言（`TEXTURE2D`, `SAMPLER`）はCBUFFERの外に置く。
```hlsl
// 正しい例
TEXTURE2D(_MainTex);
SAMPLER(sampler_MainTex);

CBUFFER_START(UnityPerMaterial)
    float4 _MainTex_ST;
    half4 _BaseColor;
CBUFFER_END
```

### Shader Graph
- SubGraphは再利用を前提に命名する（例: `SG_FresnelEffect`）
- ノードのグループ化とStickyNoteによるメモを積極的に使う
- コードシェーダーと同等のものをShader Graphで作成して比較学習を行う

### 精度修飾子の方針
- モバイル対応を意識し、可能な限り `half` を使用する
- ワールド座標系の計算やUV座標の精度が必要な箇所では `float` を使用する
- 迷ったら `float` を使い、最適化フェーズで `half` に置き換える

### Shader Variant / Keyword管理
- 学習用途では `shader_feature` を基本とする（ビルドに不要なバリアントを含めない）
- 常にすべてのバリアントが必要な場合のみ `multi_compile` を使用する
- キーワードの数は1シェーダーあたり最大4つ程度に抑える（バリアント爆発を防ぐ）

## URP固有の情報

### 主要インクルードパス
```hlsl
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Input.hlsl"
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/ShaderVariablesFunctions.hlsl"
```

### URPのPass構造テンプレート
```hlsl
Shader "Custom/Category/Name"
{
    Properties
    {
        _BaseMap ("Base Map", 2D) = "white" {}
        _BaseColor ("Base Color", Color) = (1,1,1,1)
    }

    SubShader
    {
        Tags
        {
            "RenderPipeline" = "UniversalPipeline"
            "RenderType" = "Opaque"
            "Queue" = "Geometry"
        }

        Pass
        {
            Name "ForwardLit"
            Tags { "LightMode" = "UniversalForward" }

            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            struct Attributes
            {
                float4 positionOS : POSITION;
                float2 uv : TEXCOORD0;
            };

            struct Varyings
            {
                float4 positionHCS : SV_POSITION;
                float2 uv : TEXCOORD0;
            };

            TEXTURE2D(_BaseMap);
            SAMPLER(sampler_BaseMap);

            CBUFFER_START(UnityPerMaterial)
                float4 _BaseMap_ST;
                half4 _BaseColor;
            CBUFFER_END

            Varyings vert(Attributes input)
            {
                Varyings output;
                output.positionHCS = TransformObjectToHClip(input.positionOS.xyz);
                output.uv = TRANSFORM_TEX(input.uv, _BaseMap);
                return output;
            }

            half4 frag(Varyings input) : SV_Target
            {
                half4 texColor = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, input.uv);
                return texColor * _BaseColor;
            }
            ENDHLSL
        }
    }

    Fallback "Universal Render Pipeline/Lit"
}
```

## デバッグ・検証ツール
- **Frame Debugger**: Unity内蔵。Draw Callの確認、SRP Batcher互換性の確認に使用
- **RenderDoc**: GPU側のデバッグが必要な場合に使用
- **Shader Variants検証**: `Edit > Project Settings > Graphics` でログ出力を有効化して確認
- **Stats Window**: Game Viewで描画負荷を簡易確認

## 学習カリキュラム（段階的）

### Phase 1: 基礎
- ShaderLabの構文理解（Properties, SubShader, Pass）
- Unlit Shaderの作成（色、テクスチャ表示）
- 座標空間の変換（Object → World → View → Clip）
- UV操作（スクロール、タイリング、回転）
- 同等のShader Graphも作成して比較

### Phase 2: ライティング
- Lambert拡散反射
- Blinn-Phong鏡面反射
- 法線マッピング（Tangent Space）
- URPのLight構造体の利用
- 複数ライト対応

### Phase 3: 応用表現
- トゥーンシェーディング（ランプテクスチャ、アウトライン）
- Rim Light / Fresnel効果
- Dissolve / ディゾルブエフェクト
- 透過・半透明（Alpha Blending, Alpha Test）
- Stencilバッファの活用

### Phase 4: 高度な技術
- ポストプロセス（Custom Renderer Feature）
- Compute Shader
- テッセレーション（対応環境のみ）
- GPU Instancingの実装
- モバイル最適化テクニック（ALU削減、テクスチャ帯域最適化）

### Phase 5: 統合・実践
- キャラクター向けシェーダー（肌、髪、衣服）
- 環境向けシェーダー（水面、植生、空）
- VFXシェーダー（パーティクル、歪みエフェクト）
- パフォーマンスプロファイリングと最適化

## Claudeへの指示

### コード生成時のルール
- 新しいShaderを作成する際は、必ず上記のURPテンプレートをベースにする
- 各セクションに日本語で学習用コメントを付ける
- SRP Batcher互換性を常に維持する
- モバイル対応を考慮した精度修飾子を使用する
- 既存のShaderを修正する場合は、変更点と理由を説明する

### 説明時のルール
- GPU上で何が起きているかを概念的に説明する
- 新しい概念が出てきたら、図解や具体的な数値例を用いて説明する
- 「なぜこうするのか」の理由を必ず添える
- パフォーマンスへの影響がある場合は言及する
- Shader GraphとHLSLコードの対応関係を示す

### 学習サポート
- 現在のPhaseに合わせた難易度で回答する
- Phase外の高度な概念が必要な場合は、簡潔に触れつつ「Phase Xで詳しく学ぶ」と案内する
- 各トピックの学習完了時に、理解度確認のための練習課題を提案する