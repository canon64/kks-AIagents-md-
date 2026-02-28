# MainGirlHipsIkHijack 進捗メモ (2026-02-25)

## 対象
- プロジェクト: `../MainGirlHipsIkHijack`
- 実行先DLL: `F:/kks/BepInEx/plugins/MainGirlHipsIkHijack/MainGirlHipsIkHijack.dll`

## 今回までに完了したこと
1. 本編VR向けの腰IK制御を実装済み
- 腰IK有効/無効
- 左コントローラー連動の開始/停止
- 腰基準は `A_ROOTBONE(cf_j_hips)` 優先で解決

2. 速さゲージ系の乗っ取りを実装済み
- 腰移動中だけ `speed/speedCalc` を加算
- 停止後は減衰
- 微小移動は `movementThreshold` で無視

3. 今回の追加実装（重要）
- 要件: 「待機状態で座標が動いたら1回だけピストン開始」
- 実装内容:
  - `sonyu系` かつ `InsertIdle / A_InsertIdle` 中
  - 移動量が閾値超え
  - その待機区間で未送信
  - `flags.click == none`
  - 上記を満たすときだけ `flags.click = HFlag.ClickKind.speedup` を1回送信
- 待機状態を抜けたら送信フラグをリセット

## 変更したファイル
- `../MainGirlHipsIkHijack/MainGirlHipsIkState.cs`
  - `speedupBootstrapSentInInsertIdle` を追加
- `../MainGirlHipsIkHijack/MainGirlHipsIkPlugin.Runtime.cs`
  - `TryQueueSpeedupBootstrap(...)` を追加
  - `TickMovementAndGauge(...)` に待機中1回クリック送信判定を追加
  - IK/連動のON/OFF時にブートストラップ状態を初期化
- `../MainGirlHipsIkHijack/CODEBASE_STATE.md`
  - 上記仕様を追記

## ビルド・配置状況
- `dotnet build -c Release` 成功（警告0 / エラー0）
- DLL配置済み:
  - `F:/kks/BepInEx/plugins/MainGirlHipsIkHijack/MainGirlHipsIkHijack.dll`
  - 更新時刻: 2026-02-25 04:47:17

## ログ確認先
- 専用ログフォルダ:
  - `F:/kks/BepInEx/plugins/MainGirlHipsIkHijack/_logs/`
- 主に見るファイル:
  - `info.txt`
  - `debug.txt`
- 目印ログ:
  - `擬似クリック送信 click=speedup anim=InsertIdle ...`
  - `擬似クリック送信 click=speedup anim=A_InsertIdle ...`

## 次回の確認手順
1. 本編VRでHシーン開始
2. F8でUI表示
3. 腰IK有効 → 左コントローラー連動開始
4. `InsertIdle / A_InsertIdle` 中に腰を動かす
5. 1回だけループ入りするか確認
6. `info.txt` に擬似クリック送信ログが出るか確認

## 次回の着手候補
1. 擬似クリック送信にクールダウン秒を追加（必要なら）
2. 体位/モードごとのブートストラップ有効・無効のUI切替
3. ログ粒度の調整（通常運用は `info` だけ、調査時のみ `debug`）

## 2026-02-25 20:12 追記（今回）
1. `FemaleAnimSpeedCutProbe` の機能を `MainGirlHipsIkHijack` に統合
- UIに「女アニメ速度切断 ON/OFF」ボタンを追加
- Configに `General.CutFemaleAnimSpeedEnabled` を追加（デフォルトOFF）

2. Harmonyパッチ追加
- `HActionBase.SetAnimatorFloat(string,float,bool,bool)` Prefixを追加
- 切断ONかつ `_param` が `speed` / `speedBody` のとき:
  - female/female1 への反映をスキップ
  - male/male1/item は従来どおり反映
  - `__result=true` で元メソッドをスキップ

3. 変更ファイル
- `../MainGirlHipsIkHijack/MainGirlHipsIkPlugin.cs`
- `../MainGirlHipsIkHijack/MainGirlHipsIkState.cs`
- `../MainGirlHipsIkHijack/MainGirlHipsIkPlugin.Config.cs`
- `../MainGirlHipsIkHijack/MainGirlHipsIkPlugin.UI.cs`
- `../MainGirlHipsIkHijack/MainGirlHipsIkPlugin.Runtime.cs`
- `../MainGirlHipsIkHijack/MainGirlHipsIkPatches.cs`
- `../MainGirlHipsIkHijack/CODEBASE_STATE.md`

4. ビルド・配置
- `dotnet build -c Release` 成功（警告0 / エラー0）
- DLL配置済み:
  - `F:/kks/BepInEx/plugins/MainGirlHipsIkHijack/MainGirlHipsIkHijack.dll`
  - 更新時刻: 2026-02-25 20:12:29
