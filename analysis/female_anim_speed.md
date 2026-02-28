# 女キャラ速度反映切断 調査メモ

調査日: 2026-02-25  
対象: KKS 本編H (`KoikatsuSunshine` / `KoikatsuSunshine_VR`)

## 目的

- 速さゲージ (`speed/speedCalc`) は動かしたまま、
- 女キャラ本体モーションへの「速度反映」だけ切断できるかを確認する。

## 事実（デコンパイル裏取り）

### 1. 速さゲージUIは `flags.speed / flags.speedCalc` を直接表示

- Aibu: `imageSpeed.fillAmount = InverseLerp(..., flags.speed)`
- それ以外: `imageSpeed.fillAmount = flags.speedCalc`

参照:
- `../_decomp/HSprite.latest.cs:605`
- `../_decomp/HSprite.latest.cs:609`

### 2. 速度値は `HFlag.WaitSpeedProc*` で更新

- `WaitSpeedProc(...)` が `speedCalc` を更新し、`speed = curve.Evaluate(speedCalc)` する
- `WaitSpeedProcAibu()` は `speed` を直接更新

参照:
- `../_decomp/HFlag.latest.cs:791`
- `../_decomp/HFlag.latest.cs:808`
- `../_decomp/HFlag.latest.cs:819`
- `../_decomp/HFlag.latest.cs:869`
- `../_decomp/HFlag.latest.cs:883`

### 3. 女キャラへの速度反映は `HActionBase.SetAnimatorFloat` 経由

`SetAnimatorFloat(string _param, float _value, bool _isMale=true, bool _isFemale1=true)` 内で:

- `female.setAnimatorParamFloat(hash, value)` を常に実行
- `female1` は `_isFemale1` 条件で実行
- `male/male1` は `_isMale` 条件で実行

参照:
- `../_decomp/HActionBase.latest.cs:186`
- `../_decomp/HActionBase.latest.cs:189`
- `../_decomp/HActionBase.latest.cs:192`
- `../_decomp/HActionBase.latest.cs:196`

### 4. 各モードは `speed` / `speedBody` を毎フレーム投入

- Sonyu/Houshi/3P系は `SetAnimatorFloat("speed", flags.speed)`
- Aibuは `SetAnimatorFloat("speedBody", 1f + flags.speed)`

参照:
- `../_decomp/_tmp_HSonyu.cs:925`
- `../_decomp/_tmp_HHoushi.cs:453`
- `../_decomp/_tmp_H3PSonyu.cs:985`
- `../_decomp/_tmp_H3PHoushi.cs:482`
- `../_decomp/_tmp_HAibu.cs:525`

### 5. `ChaControl.setAnimatorParamFloat` は最終的に `Animator.SetFloat`

参照:
- `../_decomp/ChaControl.latest.full.cs:807`
- `../_decomp/ChaControl.latest.full.cs:815`

## 結論

- 「速さゲージ」と「女キャラモーション速度反映」は分離可能。
- 実装ポイントは `HActionBase.SetAnimatorFloat(...)`。
- ここで `_param == "speed"` / `"speedBody"` のときだけ女キャラへの `setAnimatorParamFloat` をスキップすれば、
  - `flags.speed/speedCalc`（=ゲージ）は動く
  - 女キャラの速度反映だけ切れる

## 実装方針（テストプラグイン）

1. Harmonyで `HActionBase.SetAnimatorFloat(string,float,bool,bool)` をPrefixパッチ。
2. 切断ON時かつ `_param` が `speed` または `speedBody` のとき:
   - 元メソッドをスキップ
   - 男性側・item側の反映は維持
   - 女性側（`female`, `female1`）だけ反映しない
3. IMGUIで切断ON/OFFボタンを提供（検証用）。

## 注意点

- 切断ON時は「女の速度更新が止まる」ため、ON切替時点の値で体感固定される挙動になる可能性がある。
- 3Pの `female1` も同様に切る設計にする（女全体対象）。
