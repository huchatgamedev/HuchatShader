# Transparent Shader テンプレート

透過・半透明シェーダーのテンプレート。
Alpha Blending と Alpha Test（Cutout）の2パターンを収録。

## 透過で変わるポイント（Opaqueとの差分）

不透明シェーダーからの変更点のみをまとめる。全体構造はLitテンプレートと同じ。

### Tags の変更
```hlsl
Tags
{
    "RenderPipeline" = "UniversalPipeline"
    "RenderType" = "Transparent"        // ← Opaque → Transparent
    "Queue" = "Transparent"             // ← Geometry → Transparent (3000)
}
```

### Pass内のレンダーステート追加
```hlsl
Pass
{
    Name "ForwardLit"
    Tags { "LightMode" = "UniversalForward" }

    // [学習メモ] Alpha Blendingの設定
    // SrcAlpha: フラグメントの色にアルファを掛ける
    // OneMinusSrcAlpha: 背景の色に (1 - アルファ) を掛ける
    // 最終色 = フラグメント色 × α + 背景色 × (1 - α)
    Blend SrcAlpha OneMinusSrcAlpha

    // [学習メモ] ZWrite Off:
    // 半透明はDepthバッファに書き込まない
    // 書き込むと後ろの半透明オブジェクトが描画されなくなる
    ZWrite Off

    // Cull Back（デフォルト。両面描画したい場合は Cull Off）

    HLSLPROGRAM
    // ... シェーダー本体（Litテンプレートと同様）
    ENDHLSL
}
```

### フラグメントシェーダーの出力
```hlsl
half4 frag(Varyings input) : SV_Target
{
    half4 baseColor = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, input.uv) * _BaseColor;

    // ... ライティング計算 ...

    // [学習メモ] アルファ値をそのまま出力する
    // Blend命令がGPU側で背景との合成を行う
    return half4(finalColor, baseColor.a);
}
```

---

## Alpha Test（Cutout）パターン

完全な透過ではなく、アルファ値に基づいて「描画する/しない」を二値で決める方式。
草や葉、フェンスなど、穴の空いたテクスチャに使う。

### Opaqueとの差分

Tags は Opaque のまま使える（ただしQueueを `AlphaTest` にするのが一般的）:
```hlsl
Tags
{
    "RenderPipeline" = "UniversalPipeline"
    "RenderType" = "TransparentCutout"
    "Queue" = "AlphaTest"               // 2450（GeometryとTransparentの間）
}
```

Properties に追加:
```hlsl
_Cutoff ("Alpha Cutoff", Range(0, 1)) = 0.5
```

CBUFFER に追加:
```hlsl
half _Cutoff;
```

フラグメントシェーダーでクリップ:
```hlsl
half4 frag(Varyings input) : SV_Target
{
    half4 baseColor = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, input.uv) * _BaseColor;

    // [学習メモ] clip(): 引数が負ならこのピクセルの描画を破棄する
    // アルファがCutoff未満なら描画しない
    clip(baseColor.a - _Cutoff);

    // ... ライティング計算 ...

    return half4(finalColor, 1.0); // 残ったピクセルは不透明として出力
}
```

Alpha TestではZWriteはOnのまま、ShadowCasterパスも有効にできる。
ShadowCasterでも同じ `clip()` を行うことで、穴の部分から影が透ける。

---

## Blend命令のリファレンス

| Blend設定 | 用途 |
|-----------|------|
| `Blend SrcAlpha OneMinusSrcAlpha` | 標準的な半透明 |
| `Blend One One` | 加算合成（発光エフェクト、パーティクル） |
| `Blend One OneMinusSrcAlpha` | プリマルチプライドアルファ |
| `Blend DstColor Zero` | 乗算合成（影デカール等） |
| `Blend SrcAlpha One` | 加算半透明（ソフトな発光） |

---

## 透過シェーダーの描画順の注意

半透明オブジェクトは奥から手前の順に描画しないと正しく合成されない。
Unityは`Queue`の値でソートするが、同一Queue内のオブジェクト同士は
カメラからの距離でソートされる。ただしメッシュ単位のソートなので、
大きなメッシュが交差する場合は描画順の破綻が起きうる。
これはリアルタイムレンダリングの根本的な制限であり、OIT（Order Independent Transparency）以外では完全な解決は難しい。
