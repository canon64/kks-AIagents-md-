# 本編H ゲージ周辺データフロー解析と乗っ取りポイント（2026-02-25）

- 対象: `KoikatsuSunshine` / `KoikatsuSunshine_VR`
- 目的: 女/男感度ゲージを「どこで」「どう変化しているか」を分解し、確実に乗っ取るための実務手順を固定する。

## 1. 確認済みの基幹メソッド（デコンパイル裏取り済み）

- `HFlag.FemaleGaugeUp(float _addPoint, bool _force, bool _isIdle = true)`
  - 実処理: `gaugeFemale += _addPoint * potion;` → `Mathf.Clamp`
  - 参照: `../_decomp/HFlag.latest.cs`（`FemaleGaugeUp`）
- `HFlag.MaleGaugeUp(float _addPoint)`
  - 実処理: `gaugeMale += _addPoint * feelMaleCheat;` → `Mathf.Clamp`
  - 参照: `../_decomp/HFlag.latest.cs`（`MaleGaugeUp`）
- `HSprite.Update()`
  - 表示処理: `flags.gaugeFemale` / `flags.gaugeMale` を `imgBar.fillAmount` に反映
  - 参照: `../_decomp/HSprite.latest.cs`（`Update`）

## 2. ゲージ変化の経路（重要）

### 経路A: 通常加算/減衰（メソッド経由）

- 各Hモード処理（`HAibu` / `HHoushi` / `HSonyu` / `H3P...`）から `flags.FemaleGaugeUp(...)` / `flags.MaleGaugeUp(...)` が呼ばれる。
- ここは Harmony Prefix で `_addPoint` を差し替えれば乗っ取り可能。

### 経路B: 直接代入（メソッドを通らない）

- 一部処理で `flags.gaugeFemale = ...` / `flags.gaugeMale = ...` の直接代入がある。
- 例:
  - `HSprite.OnImmediatelyFinishMale/Female` で 70 へ直置き
  - `HSonyu` / `H3PSonyu` 系で 0 リセットや Lerp
- ここは `HFlag.*GaugeUp` パッチだけでは乗っ取れない。

### 経路C: UI表示

- `HSprite.Update()` が `flags.gaugeFemale/gaugeMale` をバーへ反映するだけ。
- 見た目が動くかどうかは、最終的に `flags` の値をどれだけ確保できたかで決まる。

## 3. 今回導入した「ゲージ流れ専用ログ」

プラグイン: `../MainGirlHipsIkHijack`

### 追加済みログファイル

- `F:/kks/BepInEx/plugins/MainGirlHipsIkHijack/_logs/gauge_flow.txt`
- 併せて `all.txt` にも `[GAUGEFLOW]` で出力

### 設定/UI

- UIトグル: `ゲージ流れログを有効（gauge_flow.txt）`
- UIトグル: `ゲージメソッド詳細ログを有効`
- UIスライダ: `ゲージ流れログ間隔(秒)`
- `GaugeFlow.Enabled` はデフォルト `false`（必要時のみON）

### ログが示す内容

- `Update ... deltaF/deltaM`:
  - `HSceneProc.Update` 1周の前後差分
- `LateUpdate ... deltaF/deltaM`:
  - `HSceneProc.LateUpdate` 1周の前後差分
- `byMethodF/byMethodM`:
  - `FemaleGaugeUp/MaleGaugeUp` の戻り値差分を積算した量
- `directF/directM`:
  - `delta - byMethod`（= 直接代入・別経路影響の推定量）

判定の見方:
- `directF/directM ≈ 0` なら、ほぼメソッド経由だけで動いている。
- `directF/directM` が大きく跳ねるなら、直接代入経路が効いている。

## 4. 乗っ取りの実務手順（この順でやる）

1. `ゲージ加算乗っ取り` をON（`HFlag.FemaleGaugeUp/MaleGaugeUp` Prefix有効化）。
2. `ゲージ流れログ` をON（必要ならメソッド詳細もON）。
3. H中に問題操作を再現し、`gauge_flow.txt` を確認。
4. `directF/directM` が非ゼロになる操作を特定。
5. その操作の発火元メソッド（例: `HSprite.OnImmediatelyFinish*`, `HSonyu`系）に追加パッチ。
6. 追加パッチ後、再度 `directF/directM` を見てゼロ寄りになるか確認。

## 5. どこをどう乗っ取るか（結論）

- 基本層: `HFlag.FemaleGaugeUp` / `HFlag.MaleGaugeUp` Prefixで `_addPoint` 差し替え。
- 補助層: `HSceneProc.LateUpdate` Postfixで最終値補正（必要時のみ）。
- 例外層: `HSprite.OnImmediatelyFinish*` や `HSonyu/H3P` の直接代入点を個別パッチ。

この3層で、メソッド経由と直接代入の両方を実質支配できる。
