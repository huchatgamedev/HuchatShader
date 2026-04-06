---
name: shader-debug
description: "Shaderのデバッグ・トラブルシュートを支援するスキル。ユーザーが「マゼンタになる」「ピンクになる」「描画されない」「影が出ない」「SRP Batcher互換じゃない」「シェーダーエラー」「コンパイルエラー」「表示がおかしい」「バグ」「壊れた」「正しく表示されない」「ちらつく」「Zファイティング」「描画順がおかしい」などと言ったときにこのスキルを使う。シェーダーの問題診断、エラー解析、修正提案を行う。"
---

# Shader Debug & Troubleshoot

Unity 6 + URP 環境でのシェーダーのデバッグ・トラブルシュートを支援するスキル。
症状から原因を素早く特定し、修正方法を提示する。

## 診断フロー

シェーダーの問題が報告された場合、以下の順序で診断する。

### Step 1: 症状の分類

まず症状を特定する。以下のどれに当たるか判断する:

1. **マゼンタ（ピンク）表示** → シェーダーが読み込めない / コンパイルエラー
2. **真っ黒** → ライティング計算の問題 / テクスチャ未設定
3. **真っ白** → 値の飽和 / テクスチャ未設定
4. **描画されない（透明）** → Render Queue / Blend / Cull設定
5. **影が落ちない** → ShadowCaster Pass不足
6. **影を受けない** → Shadow Coord / multi_compile不足
7. **ちらつき・Zファイティング** → 深度バッファの精度問題
8. **パフォーマンス低下** → → shader-optimizeスキルに委譲
9. **SRP Batcher非互換** → CBUFFER構造の問題

### Step 2: 関連ファイルの確認

問題のシェーダーファイルを読み込み、以下をチェックする:

- [ ] `"RenderPipeline" = "UniversalPipeline"` タグがあるか
- [ ] CBUFFER_START(UnityPerMaterial) と CBUFFER_END で囲まれているか
- [ ] テクスチャ宣言がCBUFFER外にあるか
- [ ] PropertiesとCBUFFER内の変数が一致しているか
- [ ] 必要なインクルードがあるか
- [ ] セマンティクスが正しいか

### Step 3: 修正提案

原因を特定したら、修正コードを提示する。

## 症状別トラブルシュートガイド

### マゼンタ（ピンク）表示

**原因一覧**（頻度順）:
1. コンパイルエラー（構文エラー、未定義変数、インクルード不足）
2. `"RenderPipeline" = "UniversalPipeline"` タグ忘れ
3. ビルトインRP用シェーダーをURPで使おうとしている
4. シェーダーファイルのパスが壊れている

**診断手順**:
```
1. Unityコンソールでエラーメッセージを確認
2. シェーダーをインスペクターで選択 → "Compile and show code" でエラー詳細を確認
3. エラー行番号をもとにコードを修正
```

**よくあるコンパイルエラー**:
| エラーメッセージ | 原因 | 修正 |
|--------------|------|------|
| `undeclared identifier` | 変数未宣言 | 宣言を追加。CBUFFERに入れ忘れていないか確認 |
| `cannot match any overloaded function` | 引数の型不一致 | half/float/float3等の型を確認 |
| `redefinition of '...'` | 二重定義 | インクルードの重複を `#ifndef` ガードで防ぐ |
| `syntax error: unexpected token '...'` | 構文エラー | セミコロン忘れ、括弧の対応を確認 |
| `'POSITION' : semantic is missing` | セマンティクス不備 | Attributes構造体のセマンティクスを確認 |

### 真っ黒表示

**原因一覧**:
1. ライティング計算で法線が (0,0,0) になっている
2. `normalOS` を正規化していない
3. テクスチャが未設定（マテリアルで None）
4. `dot(N, L)` の結果が常に0以下（法線の向きが逆）
5. Light取得関数のインクルード不足
6. `shadowAttenuation` が0（影の中にいる）

**デバッグ手法**: フラグメントシェーダーで中間値を色として出力する
```hlsl
// デバッグ: 法線をビジュアライズ
return half4(normalWS * 0.5 + 0.5, 1);

// デバッグ: NdotLをビジュアライズ
half NdotL = saturate(dot(normalWS, lightDir));
return half4(NdotL, NdotL, NdotL, 1);

// デバッグ: UV座標をビジュアライズ
return half4(input.uv, 0, 1);

// デバッグ: ワールド座標をビジュアライズ
return half4(frac(input.positionWS), 1);
```

### 影が落ちない

**原因**: ShadowCaster Passが存在しない。

**修正**: ShadowCaster Passを追加する。`references/shadowcaster-pass.md` を参照。

最小限のShadowCaster Pass:
```hlsl
Pass
{
    Name "ShadowCaster"
    Tags { "LightMode" = "ShadowCaster" }

    ZWrite On
    ZTest LEqual
    ColorMask 0  // 色は書かない。深度のみ。
    Cull Back

    HLSLPROGRAM
    #pragma vertex ShadowPassVertex
    #pragma fragment ShadowPassFragment

    #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
    #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/ShadowCasterPass.hlsl"
    ENDHLSL
}
```

**注意**: URP標準の `ShadowCasterPass.hlsl` を使うには、`Attributes` / `Varyings` の構造がURP標準に準拠している必要がある。カスタム構造体を使っている場合は自前で実装する。

### 影を受けない

**原因一覧**:
1. Shadow Coordの計算がない
2. `#pragma multi_compile` でシャドウキーワードが定義されていない
3. `GetMainLight(shadowCoord)` ではなく `GetMainLight()` を使っている

**必要なmulti_compile**:
```hlsl
#pragma multi_compile _ _MAIN_LIGHT_SHADOWS _MAIN_LIGHT_SHADOWS_CASCADE
#pragma multi_compile_fragment _ _SHADOWS_SOFT
```

**Shadow Coordの計算**:
```hlsl
// 頂点シェーダーで
float4 shadowCoord = TransformWorldToShadowCoord(positionWS);

// フラグメントシェーダーで
Light mainLight = GetMainLight(shadowCoord);
half shadow = mainLight.shadowAttenuation;
```

### SRP Batcher非互換

**確認方法**:
1. Frame Debugger（Window > Analysis > Frame Debugger）を開く
2. Draw Callを選択して "SRP Batch" の状態を確認
3. 非互換の場合、理由が表示される

**よくある原因と修正**:
| 原因 | 修正 |
|------|------|
| CBUFFERがない | `CBUFFER_START(UnityPerMaterial)` / `CBUFFER_END` を追加 |
| テクスチャがCBUFFER内 | `TEXTURE2D` / `SAMPLER` をCBUFFER外に移動 |
| PropertiesとCBUFFERの不一致 | Propertiesの全変数に対応するCBUFFER内変数があるか確認 |
| CBUFFER_START(UnityPerMaterial)が複数回 | 1つにまとめる |
| グローバル変数がCBUFFER内 | `_Time`, `_WorldSpaceCameraPos` 等はCBUFFERから除外 |

### ちらつき・Zファイティング

**原因**: 同一平面上の2つのメッシュの深度値が競合している。

**修正方法**:
```hlsl
// 方法1: Offset命令でバイアスを掛ける
Pass
{
    Offset -1, -1  // 手前側に微小オフセット
    // ...
}

// 方法2: 頂点シェーダーで法線方向に微小オフセット
output.positionHCS = TransformObjectToHClip(input.positionOS.xyz + input.normalOS * 0.001);
```

### 透過が正しくない

**チェックリスト**:
- [ ] `"Queue" = "Transparent"` が設定されているか
- [ ] `"RenderType" = "Transparent"` が設定されているか
- [ ] `Blend SrcAlpha OneMinusSrcAlpha` （または適切なBlend命令）があるか
- [ ] `ZWrite Off` になっているか
- [ ] フラグメントシェーダーのアルファ値が正しいか

## デバッグ用ヘルパー

以下のデバッグテクニックを状況に応じて提案する。

### 値のビジュアライズパターン
```hlsl
// 0〜1の値をグレースケールで表示
half debugVal = /* 調べたい値 */;
return half4(debugVal, debugVal, debugVal, 1);

// -1〜1の値を0〜1にリマップして表示
half3 debugVec = /* 調べたい法線等 */;
return half4(debugVec * 0.5 + 0.5, 1);

// 値の範囲をヒートマップで表示（赤=大、青=小）
half debugVal = /* 調べたい値 */;
return half4(debugVal, 0, 1 - debugVal, 1);
```

### UnityエディターでのFrame Debugger活用
1. **Window > Analysis > Frame Debugger** を開く
2. **Enable** をクリックして描画コールを一覧表示
3. 各Draw Callで以下を確認:
   - ShaderProperties: 各プロパティの値が正しいか
   - RenderTarget: 正しいバッファに描画しているか
   - SRP Batch: バッチングが効いているか

### RenderDocとの連携
高度なデバッグが必要な場合:
1. Scene ViewまたはGame Viewを右クリック → "Capture RenderDoc"
2. ピクセルを選択して描画履歴を確認
3. シェーダーのステップ実行でデバッグ
