---
name: shader-optimize
description: "シェーダーのパフォーマンス最適化を支援するスキル。ユーザーが「最適化」「パフォーマンス」「遅い」「重い」「モバイル対応」「描画負荷」「ドローコール」「バッチング」「GPU負荷」「ALU削減」「帯域」「バリアント爆発」「メモリ消費」「フレームレート」「プロファイリング」「オーバードロー」などと言ったときにこのスキルを使う。パフォーマンス分析、最適化提案、モバイル向け調整を行う。"
---

# Shader Performance Optimizer

Unity 6 + URP 環境でのシェーダーパフォーマンス最適化を支援するスキル。
モバイル（iOS/Android）とPC両対応を前提に、具体的な最適化手法を提示する。

## 最適化の優先順位

シェーダーの最適化は影響度の高い順に取り組む。

### 優先度: 高（まず確認）
1. **SRP Batcher互換** — ドローコール削減の最大要因
2. **オーバードロー** — 透過オブジェクトの重なり
3. **Shader Variant数** — ビルドサイズ・ロード時間に直結

### 優先度: 中（次に検討）
4. **テクスチャ帯域** — サンプリング回数とテクスチャ解像度
5. **精度修飾子** — half vs float（モバイルで効果大）
6. **数学演算の簡略化** — ALU命令数の削減

### 優先度: 低（最後に微調整）
7. **分岐の回避** — GPU特有の問題
8. **補間子（Varyings）の削減** — 帯域削減
9. **Vertex vs Fragment** — 計算の移動

## 最適化チェックリスト

シェーダーのレビュー時に以下を順にチェックする。

### SRP Batcher互換チェック
- [ ] CBUFFER_START(UnityPerMaterial) / CBUFFER_END で囲まれている
- [ ] テクスチャ宣言がCBUFFER外
- [ ] PropertiesとCBUFFER内変数が一致

### バリアント削減チェック
- [ ] `shader_feature` と `multi_compile` が適切に使い分けられている
- [ ] キーワード数が4個以下
- [ ] 不要なmulti_compileがない

### モバイル最適化チェック
- [ ] 色・ライティング値に `half` を使用
- [ ] ワールド座標・UV座標に `float` を使用
- [ ] フラグメントシェーダーで不必要に複雑な計算がない
- [ ] テクスチャサンプリング回数が最小限
- [ ] `discard` / `clip` の使用を最小限に

## 最適化テクニック集

### 1. 精度修飾子の最適化（モバイル効果: 大）

モバイルGPUでは `half`（16bit float）と `float`（32bit float）で実際にALU性能が異なる。
`half` は `float` の約2倍のスループットが出るGPUもある。

```hlsl
// ❌ 最適化前: すべてfloat
float3 normalWS = TransformObjectToWorldNormal(input.normalOS);
float NdotL = saturate(dot(normalWS, lightDir));
float3 diffuse = NdotL * lightColor * baseColor.rgb;

// ✅ 最適化後: 可能な箇所をhalfに
half3 normalWS = TransformObjectToWorldNormal(input.normalOS);
half NdotL = saturate(dot(normalWS, lightDir));
half3 diffuse = NdotL * lightColor * baseColor.rgb;
```

**halfが使える場所**: 色、法線、ライティング結果、0〜1のパラメータ  
**floatにすべき場所**: ワールド座標、UV座標、時間（_Time）、行列演算

### 2. 計算のVertex→Fragment移動（モバイル効果: 中〜大）

フラグメントシェーダーは頂点シェーダーより桁違いに多く実行される。
リニアに変化する値は頂点シェーダーで計算し、補間に任せる。

```hlsl
// ❌ フラグメントで毎ピクセル計算
half4 frag(Varyings input) : SV_Target
{
    float3 positionWS = TransformObjectToWorld(input.positionOS.xyz); // 重い
    half3 viewDirWS = normalize(_WorldSpaceCameraPos - positionWS);
    // ...
}

// ✅ 頂点で計算して補間
Varyings vert(Attributes input)
{
    output.positionWS = TransformObjectToWorld(input.positionOS.xyz);
    output.viewDirWS = normalize(_WorldSpaceCameraPos - output.positionWS);
    return output;
}
half4 frag(Varyings input) : SV_Target
{
    half3 viewDirWS = normalize(input.viewDirWS); // 再正規化のみ
    // ...
}
```

**注意**: 非リニアな値（法線マップ適用後の法線など）は補間すると不正確になるため、フラグメントで計算する必要がある。

### 3. テクスチャ帯域の最適化（モバイル効果: 大）

モバイルGPUでテクスチャアクセスはボトルネックになりやすい。

| 手法 | 効果 | 方法 |
|------|------|------|
| テクスチャ解像度削減 | 帯域削減 | 必要最小限の解像度にする |
| チャンネルパッキング | サンプリング回数削減 | R: Metallic, G: Roughness, B: AO を1枚に |
| ミップマップ有効化 | 帯域削減 | テクスチャのGenerate Mip Mapsを有効に |
| テクスチャ圧縮 | メモリ・帯域削減 | ASTC (モバイル), BC7 (PC) |
| サンプラー共有 | サンプラー数削減 | 同じ設定のテクスチャでサンプラーを共有 |

```hlsl
// サンプラー共有の例
TEXTURE2D(_BaseMap);
TEXTURE2D(_NormalMap);
TEXTURE2D(_MaskMap);
SAMPLER(sampler_BaseMap);  // 1つのサンプラーで3テクスチャを共有

half4 base = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, uv);
half4 normal = SAMPLE_TEXTURE2D(_NormalMap, sampler_BaseMap, uv);
half4 mask = SAMPLE_TEXTURE2D(_MaskMap, sampler_BaseMap, uv);
```

### 4. Shader Variant管理（ビルドサイズ効果: 大）

```hlsl
// shader_feature: ビルド時にマテリアルで使われているバリアントのみ含む
#pragma shader_feature_local _NORMALMAP_ON

// multi_compile: 全バリアントを常にビルドに含む（ランタイム切替用）
#pragma multi_compile _ _MAIN_LIGHT_SHADOWS

// _local: グローバルキーワード枠を消費しない（上限256個）
#pragma shader_feature_local _EMISSION_ON
#pragma multi_compile_local _ _DETAIL_ON
```

**バリアント数の計算**:
```
キーワード行1: 2バリアント（ON/OFF）
キーワード行2: 3バリアント（A/B/C）
キーワード行3: 2バリアント（ON/OFF）
→ 合計: 2 × 3 × 2 = 12バリアント
```

**ガイドライン**: 1シェーダーあたりキーワード4行以下。1行あたり2〜3バリアント。

### 5. 数学演算の簡略化

```hlsl
// ❌ pow は高コスト
half rim = pow(1.0 - saturate(dot(normalWS, viewDirWS)), 4.0);

// ✅ 2乗なら掛け算で代替（pow(x,2)）
half oneMinusNdotV = 1.0 - saturate(dot(normalWS, viewDirWS));
half rim = oneMinusNdotV * oneMinusNdotV; // pow(x, 2) 相当

// ❌ normalize は sqrt を含む
half3 dir = normalize(a - b);

// ✅ rsqrt を使う（一部GPUでより高速）
half3 diff = a - b;
half3 dir = diff * rsqrt(dot(diff, diff));
```

**コストの目安**（相対的）:
| 演算 | コスト | 備考 |
|------|--------|------|
| `+`, `-`, `*` | 1 | 基本演算 |
| `mad(a,b,c)` = a*b+c | 1 | GPUで1命令 |
| `dot` | 1 | GPU組み込み |
| `saturate`, `abs` | 0 | 無料（修飾子） |
| `/`, `rcp` | 2 | 除算は高め |
| `sqrt`, `rsqrt` | 2 | |
| `sin`, `cos` | 4 | 三角関数 |
| `pow` | 4+ | 内部でexp+log |
| `normalize` | 3 | rsqrt + mul |
| テクスチャサンプル | 10-100+ | 帯域依存 |

### 6. 分岐の最適化

GPUはSIMD（同一命令を並列実行）のため、分岐コストがCPUと根本的に異なる。

```hlsl
// ❌ 動的分岐（ピクセルごとに異なる条件）
if (input.uv.x > 0.5)
    color = red;
else
    color = blue;

// ✅ step/lerp で分岐を排除
half mask = step(0.5, input.uv.x);
color = lerp(blue, red, mask);

// ⚠️ Uniform分岐（マテリアルプロパティで切替）は比較的低コスト
// すべてのピクセルで同じ分岐を取るため、GPU最適化が効く
if (_UseNormalMap > 0.5)
{
    // Normal Map処理
}
```

### 7. オーバードローの削減

透過オブジェクトが重なるとフラグメントシェーダーが重複実行される。

**対策**:
- 透過オブジェクトの面積を最小限にする
- パーティクルのサイズ・数を制限する
- 不透明部分は `Alpha Test`（`clip()`）で処理する
- `ZWrite Off` の透過でも、大きな面には事前の `ZWrite On` パスを検討

## プロファイリング手法

### Unity Stats Window
Game ViewのStatsボタンで簡易確認:
- **Batches**: ドローコール数（少ないほど良い）
- **SetPass calls**: シェーダー切替回数
- **Tris / Verts**: ポリゴン数

### Frame Debugger
Window > Analysis > Frame Debugger:
- 各Draw Callの詳細を確認
- SRP Batchの有効性を確認
- レンダリング順序の確認

### Profiler (GPU)
Window > Analysis > Profiler → GPU:
- シェーダーごとのGPU時間
- フレーム内のボトルネック特定

### RenderDoc（高度）
- ピクセル単位のシェーダー実行追跡
- テクスチャフェッチの詳細分析
- GPU命令レベルのデバッグ

## モバイル向けガイドライン

### 推奨上限値（目安）
| 項目 | 推奨上限 |
|------|---------|
| フラグメントシェーダー命令数 | 〜50 ALU |
| テクスチャサンプリング回数 | 〜4回/パス |
| Varyings数 | 〜8個 |
| Shader Variant総数 | 〜32バリアント |
| マテリアルごとのパス数 | 2パス以下 |

### モバイルで避けるべきパターン
- `discard` / `clip` の多用（TBR GPUでタイル効率が低下）
- 依存テクスチャ読み（UV計算 → その結果でテクスチャ読み）
- 高解像度のフルスクリーンエフェクト
- 過度なアルファブレンディング
