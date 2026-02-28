# 本編Hシーン 男女感度ゲージ 乗っ取り調査

- 調査日: 2026-02-24
- 対象: `KoikatsuSunshine` 本編Hシーン（Aibu/Houshi/Sonyu/3P）

## 結論（先に要点）

- 男女の感度ゲージ本体は `HFlag` の `gaugeFemale` / `gaugeMale`。
- UI表示は `HSprite.Update()` が `flags.gaugeFemale/gaugeMale` を毎フレーム参照して描画。
- ゲージ更新は `HSceneProc.Update()` -> `lstProc[mode].Proc()`（`HAibu/HHoushi/HSonyu/H3P...`）で実行。
- 「乗っ取り」を確実にやるなら、`HFlag` の加算メソッドだけでは不十分（各モードで直接代入があるため）。

## 主要クラスと責務

1. `HFlag`（値保持・共通更新）
- `gaugeFemale`, `gaugeMale`, `lockGugeFemale`, `lockGugeMale`
  - `../_decomp/_tmp_HFlag.cs:417`
  - `../_decomp/_tmp_HFlag.cs:423`
- 加算API
  - `FemaleGaugeUp(float _addPoint, bool _force, bool _isIdle = true)`
    - `../_decomp/_tmp_HFlag.cs:956`
  - `MaleGaugeUp(float _addPoint)`
    - `../_decomp/_tmp_HFlag.cs:974`

2. `HSprite`（ゲージUI）
- ゲージバー描画:
  - 女: `gauge[0].imgBar.fillAmount = InverseLerp(... flags.gaugeFemale)`
    - `../_decomp/_tmp_HSprite.cs:628`
  - 男: `gauge[1].imgBar.fillAmount = InverseLerp(... flags.gaugeMale)`
    - `../_decomp/_tmp_HSprite.cs:632`
- 即終了ボタンで70%へ直接代入:
  - 男 `OnImmediatelyFinishMale()`
    - `../_decomp/_tmp_HSprite.cs:2413`
    - `../_decomp/_tmp_HSprite.cs:2415`
  - 女 `OnImmediatelyFinishFemale()`
    - `../_decomp/_tmp_HSprite.cs:2428`
    - `../_decomp/_tmp_HSprite.cs:2430`

3. `HSceneProc`（フレーム進行の中心）
- `public HFlag flags;`
  - `../_decomp/_tmp_HSceneProc.cs:282`
- モード別処理呼び出し:
  - `Update()` 内で `lstProc[mode].Proc();`
    - `../_decomp/_tmp_HSceneProc.cs:1419`
  - `LateUpdate()` 内で `lstProc[mode].LateProc();`
    - `../_decomp/_tmp_HSceneProc.cs:1482`
- モード実体登録:
  - `HAibu`, `HHoushi`, `HSonyu`, `H3PHoushi`, `H3PSonyu`, `H3PDarkHoushi`, `H3PDarkSonyu`
    - `../_decomp/_tmp_HSceneProc.cs:1136`
    - `../_decomp/_tmp_HSceneProc.cs:1149`

## 直接代入（メソッド経由をバイパスする箇所）

`HFlag.FemaleGaugeUp/MaleGaugeUp` をパッチしても、以下の直接代入は残る。

- `HAibu`: 女ゲージを `Lerp(100->0)` / `0` 代入
  - `../_decomp/_tmp_HAibu.cs:472`
  - `../_decomp/_tmp_HAibu.cs:475`
- `HHoushi`: 男ゲージを `0` / `Lerp(100->0)` 代入
  - `../_decomp/_tmp_HHoushi.cs:341`
  - `../_decomp/_tmp_HHoushi.cs:567`
- `HSonyu`: 男女ゲージ `0` / `Lerp(old->0)` 代入多数
  - `../_decomp/_tmp_HSonyu.cs:512`
  - `../_decomp/_tmp_HSonyu.cs:1118`
  - `../_decomp/_tmp_HSonyu.cs:1122`
- 3P系/3PDark系にも同パターン
  - `../_decomp/_tmp_H3PSonyu.cs:547`
  - `../_decomp/_tmp_H3PSonyu.cs:1159`
  - `../_decomp/_tmp_H3PDarkSonyu.cs:464`
  - `../_decomp/_tmp_H3PDarkSonyu.cs:1026`

## 乗っ取り実装の推奨フック

### A. 強制上書き型（最も確実）

1. `HSceneProc.Update` Postfix で `__instance.flags.gaugeFemale/gaugeMale` を毎フレーム強制値に設定。
2. 必要なら `HSceneProc.LateUpdate` Postfix でも同じ上書き。
3. `HSprite.Update` Prefix/Postfix でバー表示を同期（通常は flags 上書きだけで追従）。

利点:
- モード別クラスの直接代入をまとめて打ち消せる。

欠点:
- 常時上書きなので、純正遷移（絶頂演出時の減衰）も潰れる。

### B. 差分介入型（自然挙動寄り）

1. `HFlag.FemaleGaugeUp` / `MaleGaugeUp` を Prefix で差し替え。
2. 直接代入を行う箇所（`HAibu/HHoushi/HSonyu/3P系`）を個別パッチ。

利点:
- 挙動を細かく維持しやすい。

欠点:
- パッチ対象が多く保守コストが高い。

## ゲーム側ロジックにおける70%境界

- 男ゲージ70%以上で操作分岐する箇所がある（例: 挿入系アクションボタン切替）。
  - `../_decomp/_tmp_HSprite.cs:3934`
  - `../_decomp/_tmp_HSprite.cs:3639`

つまり、男ゲージを乗っ取ると「UIで押せる行動」も変わる。

## 最短PoCの実装方針（次アクション）

1. `BepInProcess("KoikatsuSunshine")` で本編専用プラグイン作成。
2. Harmonyで `HSceneProc.Update` Postfix を当てる。
3. `flags` が有効な間、`gaugeFemale/gaugeMale` を設定値へ強制。
4. トグルキーでON/OFF可能にする。

