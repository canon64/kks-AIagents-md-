# 本編「女キャラの腰IKだけ有効化」調査メモ

- 調査日: 2026-02-24
- 対象: コイカツサンシャイン本編 (`KoikatsuSunshine`)

## 結論

- 実現は可能。
- ただし本編の標準IK制御だけでは「腰だけON」は直接サポートされていない。
- 実装には `BepInEx + Harmony` で `FullBodyBipedIK` の effector weight を上書きする処理が必要。

## デコンパイルで確認した根拠

1. 本編キャラは `FullBodyBipedIK` にアクセス可能
- `../_decomp/_tmp_MotionIK.cs:162`
  - `ik = animBody.GetComponent<FullBodyBipedIK>();`

2. 既定の `MotionIK` が直接扱うIKターゲットは手足4点のみ
- `../_decomp/_tmp_MotionIK.cs:15`
  - `IKTarget` enum
- `../_decomp/_tmp_MotionIK.cs:17`
  - `LeftHand`
- `../_decomp/_tmp_MotionIK.cs:18`
  - `RightHand`
- `../_decomp/_tmp_MotionIK.cs:19`
  - `LeftFoot`
- `../_decomp/_tmp_MotionIK.cs:20`
  - `RightFoot`

3. 標準処理は effector / bend の weight を更新する
- `../_decomp/_tmp_MotionIK.cs:330`
  - `effector.positionWeight = 0f;`
- `../_decomp/_tmp_MotionIK.cs:335`
  - `effector.positionWeight = param.weightPos;`
- `../_decomp/_tmp_MotionIK.cs:366`
  - `bend.weight = 0f;`
- `../_decomp/_tmp_MotionIK.cs:370`
  - `bend.weight = param2.weight;`

4. 腰/腰周辺フレーム定義は存在する
- `../_decomp/_tmp_IKCorrect.cs:7`
  - `cf_t_hips`, `cf_t_waist_L`, `cf_t_waist_R`

5. ADVコマンド側 `ActiveIK` は `ik.enabled` の全体ON/OFF
- `../_decomp/_tmp_ADV_ActiveIK.cs:18`
  - `...ikMotion.motionIK.ik.enabled = bool.Parse(...)`

6. 性別フィルタは `ChaControl` 側で可能（女=1）
- `../_decomp/_tmp_ChaControl_from_kksmain.cs:296`
  - `_sex == 0 ? "ill_Default_Male" : "ill_Default_Female"`
- `../_decomp/_tmp_ChaControl_from_kksmain.cs:256`
  - `if (1 == _sex)`

7. `FullBodyBipedEffector` を任意指定して weight を設定する経路は存在
- `../_decomp/_tmp_Utils_IKLoader.cs:43`
  - `ik.solver.GetEffector(FullBodyBipedEffector...)`
- `../_decomp/_tmp_Utils_IKLoader.cs:48`
  - `eff.positionWeight = ...`
- `../_decomp/_tmp_Utils_IKLoader.cs:49`
  - `eff.rotationWeight = ...`

## 実装方針（最小）

1. 女キャラ (`sex == 1`) の `ChaControl` のみ対象化。
2. `animBody` から `FullBodyBipedIK` を取得。
3. 毎フレーム（`LateUpdate` 相当）で以下を再適用:
- `ik.enabled = true`
- `bodyEffector` の weight を有効化
- 手足 effector の weight を 0 固定
4. モーション切替/キャラ再読込後に上書きが戻るため、再適用フックを入れる。

## 注意点

- 標準 `MotionIK` の更新と競合しやすい（weight が戻される）。
- 腰だけ有効化すると既存アニメ/当たり判定との干渉が出る場合がある。
- シーンや状態遷移（ADV/H/Action）ごとに再初期化タイミングが異なるため、適用ポイントを分ける必要がある。

## 手順参照

- 実装手順は `../MAIN_GIRL_HIPS_IK_PROCEDURE_2026-02-24.md` を参照。
