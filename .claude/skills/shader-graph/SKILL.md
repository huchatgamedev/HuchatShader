---
name: shader-graph
description: "Unity URP向けのShader Graphに関するスキル。ユーザーが「Shader Graphを作って」「ノードベースで」「ShaderGraphで」「サブグラフを作って」「ノードの繋ぎ方」「Shader Graphの設定」「ビジュアルシェーダー」などと言ったときにこのスキルを使う。Shader Graphの作成ガイド、ノード構成の提案、コードシェーダーとの比較解説、SubGraphの設計などが対象。"
---

# Shader Graph Guide (URP)

Unity 6 + URP 環境でのShader Graph作成を支援するスキル。
コードシェーダーとの比較学習を重視し、ノード構成をテキストで明確に記述する。

## Shader Graphの制約事項

Shader Graphファイル（`.shadergraph`）はUnityエディター上のGUIツールで編集するため、テキストエディターからの直接編集は推奨されない。
このスキルでは以下の形で支援する:

1. **ノード構成の設計図**をテキスト・表・Mermaidで提示する
2. **Graph Settings**の推奨値を明示する
3. **コードシェーダーとの対応関係**を示す
4. **SubGraph設計**のベストプラクティスを提供する

## 回答テンプレート

Shader Graphの作成を求められた場合、以下の構成で回答する。

### 1. Graph Settings（グラフ設定）

Graph Inspectorで設定すべき項目を表で示す。

| 設定項目 | 値 | 理由 |
|---------|-----|------|
| Material | Unlit / Lit | 用途に応じて |
| Surface Type | Opaque / Transparent | 透過の有無 |
| Render Face | Front / Both | 裏面描画の要否 |
| Alpha Clipping | On / Off | Alpha Testの有無 |

### 2. プロパティ（Blackboard）

Blackboardに追加するプロパティを一覧で示す。

| プロパティ名 | 型 | デフォルト値 | Reference Name | 説明 |
|-------------|-----|------------|----------------|------|
| Base Map | Texture2D | white | _BaseMap | ベーステクスチャ |
| Base Color | Color | (1,1,1,1) | _BaseColor | ベースカラー |

**Reference Nameの規約**: `_PascalCase` で統一する（コードシェーダーと合わせる）。

### 3. ノードフロー

ノードの接続を以下の形式で記述する:

```
[ノードA] → (出力ポート) → [ノードB](入力ポート)
```

例:
```
[Sample Texture 2D] → (RGBA) → [Multiply](A)
[Color Property: _BaseColor] → (Out) → [Multiply](B)
[Multiply] → (Out) → [Fragment](Base Color)
```

複雑なグラフの場合は、機能ブロックごとにセクション分けする。

### 4. コードシェーダーとの対応

以下の形式で対応関係を示す:

| Shader Graphノード | HLSLコード相当 |
|-------------------|----------------|
| Sample Texture 2D | `SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, uv)` |
| Transform (Object→World) | `TransformObjectToWorld(positionOS)` |
| Dot Product | `dot(a, b)` |
| Saturate | `saturate(x)` |

## よく使うノード一覧

### 入力ノード
| ノード | 機能 | コード相当 |
|--------|------|-----------|
| Position | 各空間の座標を取得 | `input.positionOS` / `positionWS` |
| Normal Vector | 法線ベクトル取得 | `input.normalOS` / `normalWS` |
| UV | UV座標取得 | `input.uv` |
| View Direction | カメラへの方向 | `GetWorldSpaceNormalizeViewDir(positionWS)` |
| Time | 時間値 | `_Time` |
| Screen Position | スクリーン座標 | `ComputeScreenPos()` |

### テクスチャ
| ノード | 機能 | コード相当 |
|--------|------|-----------|
| Sample Texture 2D | テクスチャサンプリング | `SAMPLE_TEXTURE2D()` |
| Tiling And Offset | UV変換 | `TRANSFORM_TEX()` / 手動計算 |
| Normal From Texture | ノーマルマップ読み取り | `UnpackNormal()` |

### 数学
| ノード | 機能 | コード相当 |
|--------|------|-----------|
| Multiply / Add / Subtract | 四則演算 | `*`, `+`, `-` |
| Dot Product | 内積 | `dot(a, b)` |
| Cross Product | 外積 | `cross(a, b)` |
| Lerp | 線形補間 | `lerp(a, b, t)` |
| Smoothstep | 滑らかなステップ | `smoothstep(edge0, edge1, x)` |
| Step | ステップ関数 | `step(edge, x)` |
| Saturate | 0-1クランプ | `saturate(x)` |
| Fresnel Effect | フレネル効果 | `pow(1 - saturate(dot(N, V)), power)` |

### ライティング（Lit Graph用）
Lit Shader Graphでは、Fragment出力に接続するだけでPBRライティングが適用される。
独自ライティングが必要な場合はCustom Functionノードを使う。

## SubGraphの設計指針

### 命名規則
`SG_機能名` の形式で統一する（例: `SG_FresnelEffect`, `SG_UVScroll`）。

### 再利用の基準
以下の条件を満たす場合にSubGraph化を検討する:
- 3つ以上のノードで構成される処理ブロック
- 複数のShader Graphで同じ処理を使い回す可能性がある
- 入出力が明確に定義できる

### SubGraphの構造
```
入力ポート（パラメータ）
  ↓
処理ノード群
  ↓
出力ポート（結果）
```

### 推奨SubGraph例
| SubGraph名 | 機能 | 入力 | 出力 |
|------------|------|------|------|
| SG_UVScroll | UVスクロール | UV, Speed | UV |
| SG_UVRotate | UV回転 | UV, Angle, Pivot | UV |
| SG_FresnelEffect | フレネル効果 | Normal, ViewDir, Power | Float |
| SG_ToonRamp | トゥーンランプ | NdotL, Steps | Float |
| SG_Dissolve | ディゾルブ | Noise, Threshold, EdgeWidth | Alpha, EdgeMask |

## Custom Functionノードの活用

Shader Graphの標準ノードでは表現できない処理がある場合、Custom Functionノードを使ってHLSLコードを直接記述できる。

### 使用場面
- URPのLight構造体への直接アクセス
- 複雑な数学関数
- 既存のHLSLインクルードファイルの利用

### 形式
- **String**: ノード内にHLSLコードを直接記述（短い処理向け）
- **File**: 外部の `.hlsl` ファイルを参照（再利用・保守性向上）

```hlsl
// Custom Function用のHLSLファイル例
// Assets/HuchatShader/Shaders/Common/CustomFunctions.hlsl

void MainLightCustom_float(float3 WorldPos, out float3 Direction, out float3 Color, out float Attenuation)
{
    #ifdef SHADERGRAPH_PREVIEW
    // プレビュー用のダミー値
    Direction = float3(0.5, 0.5, 0);
    Color = float3(1, 1, 1);
    Attenuation = 1;
    #else
    Light mainLight = GetMainLight();
    Direction = mainLight.direction;
    Color = mainLight.color;
    Attenuation = mainLight.distanceAttenuation * mainLight.shadowAttenuation;
    #endif
}
```

**注意**: Custom Functionでは `#ifdef SHADERGRAPH_PREVIEW` でプレビュー用の分岐を必ず入れること。プレビューではURPのライブラリが利用できないため。

## グラフ整理のベストプラクティス

1. **グループ化**: 関連ノードをグループ（右クリック → Group Selection）でまとめる
2. **Sticky Note**: 各グループにStickyNoteで処理の説明を書く
3. **Redirect Node**: 長い接続線にはRedirect Nodeを挟んで視認性を上げる
4. **左→右の流れ**: 入力を左、出力を右に配置する
5. **命名**: プロパティ名はコードシェーダーと統一する

## コードシェーダーとの比較学習

Shader Graphで作成したものと同等のコードシェーダーを並べて比較するのが効果的。
比較時のポイント:

| 観点 | Shader Graph | コードシェーダー |
|------|-------------|----------------|
| 学習コスト | ノードを繋ぐだけで直感的 | HLSL構文の理解が必要 |
| デバッグ | ノードごとにプレビュー可能 | Frame Debugger等が必要 |
| パフォーマンス | 冗長なコードが生成されることがある | 手書きで最適化可能 |
| 柔軟性 | Custom Function以外は制約あり | 何でもできる |
| バリアント | KeywordノードでGUI管理 | #pragma で手動管理 |
| SRP Batcher | 自動で互換 | 手動でCBUFFER管理が必要 |
