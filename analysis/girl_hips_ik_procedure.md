# 本編 女キャラ腰IK + ゲージ/速度乗っ取り DLL実装手順（フェーズ1/2/3/4/5）

- 作成日: 2026-02-24
- 対象: `KoikatsuSunshine` 本編
- この手順は「DLLを作る時の実装手順」
- フェーズ1範囲:
1. ショートカットでUI表示
2. UIの有効ボタン押下
3. 押下時に腰座標取得
4. 腰IK有効化
5. 左コントローラーと親子付け
- フェーズ2範囲:
1. 女/男ゲージ加算の乗っ取り
2. UIから加算乗っ取りON/OFF
3. 加算倍率・加算バイアスの適用
- フェーズ3範囲:
1. 腰IK移動量を軽量に毎フレーム検知
2. 動いている間だけゲージ加算
3. 停止で減衰開始
4. 微小な動きは閾値で除外
- フェーズ4範囲:
1. 腰追従の調整変数をDLLに導入
2. UIで調整変数をリアルタイム変更
3. 変更値を即フレーム反映
- フェーズ5範囲:
1. 速さゲージ（`HSprite.imageSpeed`）の乗っ取り
2. 速さゲージ値（`speed/speedCalc`）の制御
3. UIに速さゲージ横の数値テキストエリア表示

## 前提

1. BepInExプラグインとして実装する。
2. `BepInProcess("KoikatsuSunshine")` を付ける。
3. 本編Hシーンで女キャラ `ChaControl` にアクセスできる状態で動かす。

## 実装手順（フェーズ1）

1. プラグイン骨組みを作る。  
`Awake()` で初期化、`Update()` でショートカット入力を処理する。

2. ショートカットでUIを表示/非表示できるようにする。  
例: `F8` でパネル表示トグル。

3. UIに「有効」ボタンを1つ置く。  
ボタン押下時に `EnableHipsIkLink()` を呼ぶ。

4. `EnableHipsIkLink()` の先頭で対象を解決する。  
女キャラ `ChaControl` を取得し、失敗時は中断する。

5. 腰Transformを取得する。  
優先 `A_ROOTBONE`（実体 `cf_j_hips`）を使う。  
見つからない場合のみ `cf_t_hips(work)` → `cf_t_hips` の順でフォールバックする。

6. 腰候補の座標ログを有効化時に1回出す。  
`cf_j_hips` / `cf_t_hips(work)` / `cf_t_hips` の採用結果と座標を高精度で記録し、  
「吹っ飛び」がどの基準点で起きるかを判定できるようにする。

7. 左コントローラーを取得する。  
`VRGIN_Camera (origin)/Left Controller` のTransformを取る。  
取れなければ中断する。

8. 有効ボタン押下の瞬間に基準腰座標を保存する。  
`baseHipsPos = hips.position`  
`baseHipsRot = hips.rotation`

9. 同時に基準コントローラ座標を保存する。  
`baseCtrlPos = leftController.position`  
`baseCtrlRot = leftController.rotation`

10. 腰IK本体を取得する。  
`FullBodyBipedIK` を `animBody` から取得し、`solver.bodyEffector` を使う。

11. 腰IKを有効化する。  
`ik.enabled = true`  
`bodyEffector.target` 用の `proxy` を作り、`proxy` を基準腰座標に置く。  
`bodyEffector.target = proxy` を設定する。

12. 左コントローラーと親子付けする。  
`proxy.SetParent(leftController, true)` を実行する。  
フェーズ1ではここまでで止める（追従補正・減衰は次フェーズ）。

13. 二重有効化ガードを入れる。  
すでに `proxy`/IKが有効なら再作成しない。

14. ログを出す。  
有効化時に「対象キャラ名」「基準腰座標」「基準コントローラ座標」「IK有効化成功」を記録する。

## フェーズ1の完了条件

1. ショートカットでUIが出る。
2. 有効ボタン押下でエラーなく処理が通る。
3. 押下瞬間の腰座標がログに出る。
4. 腰IKが有効になる。
5. `proxy` が左コントローラーの子として作成される。

## フェーズ1でまだやらないこと

1. `SmoothDamp` 追従
2. 差分計算（`deltaPos`/`deltaRot`）
3. 再センター
4. ふっとび検知
5. 無効化時の完全復帰

## 実装手順（フェーズ2: ゲージ加算乗っ取り）

1. プラグイン状態を追加する。  
`gaugeHijackEnabled`（bool）を追加する。

2. パラメータを追加する。  
`femaleAddScale`, `maleAddScale`（倍率）  
`femaleAddBias`, `maleAddBias`（固定加算）  
を設定値として持つ。

3. UIに加算乗っ取りトグルを追加する。  
「ゲージ加算乗っ取り ON/OFF」ボタンを追加し、`gaugeHijackEnabled` を切り替える。

4. `HFlag.FemaleGaugeUp` に Harmony Prefix を当てる。  
対象シグネチャ: `FemaleGaugeUp(float _addPoint, bool _force, bool _isIdle = true)`  
`gaugeHijackEnabled == false` なら元処理を通す。

5. `HFlag.MaleGaugeUp` に Harmony Prefix を当てる。  
対象シグネチャ: `MaleGaugeUp(float _addPoint)`  
`gaugeHijackEnabled == false` なら元処理を通す。

6. ON時は `_addPoint` を乗っ取る。  
女: `_addPoint = _addPoint * femaleAddScale + femaleAddBias`  
男: `_addPoint = _addPoint * maleAddScale + maleAddBias`

7. 乗っ取り方式は「加算値差し替え」を採用する。  
Prefixで `_addPoint` を上書きして `return true` し、  
元メソッドの既存クランプ処理（0-100や条件）をそのまま使う。

8. ログを追加する。  
ON中は、元加算値と乗っ取り後加算値をデバッグログに出す。

9. パラメータ初期値を決める。  
`femaleAddScale=1.0`, `maleAddScale=1.0`, `femaleAddBias=0.0`, `maleAddBias=0.0`。

10. 安全ガードを入れる。  
倍率が極端な値にならないようにUI入力を制限する。

## フェーズ2の完了条件

1. UIで加算乗っ取りをON/OFFできる。
2. ON時に `FemaleGaugeUp/MaleGaugeUp` の入力加算値が置き換わる。
3. OFF時は純正挙動に戻る。
4. 乗っ取り後もゲージが0-100の範囲を外れない。

## フェーズ2の注意

1. これは「加算経路」の乗っ取り。  
`flags.gaugeFemale = 0` などの直接代入は別経路。

2. 直接代入も含めて完全に乗っ取る場合は、  
`HSceneProc.Update` 後段で最終値を再上書きするフェーズ3を追加する。

## 実装手順（フェーズ3: 軽量移動検知 + 動作時加算 + 停止時減衰）

1. 毎フレーム処理の前提を固定する。  
`Transform`、`HFlag`、`ChaControl` 参照は有効化時にキャッシュして使い回す。  
毎フレーム `GameObject.Find` とリフレクションを禁止する。

2. 追跡状態を追加する。  
`lastHipsPos`、`lastMoveTime`、`isMoving`、`moveSqrThreshold` を持つ。  
しきい値は二乗距離で比較する。

3. しきい値パラメータを追加する。  
`movementThreshold`（微小動作除外）  
`idleDelay`（停止判定までの猶予）  
`moveAddPerSecondFemale`、`moveAddPerSecondMale`  
`decayPerSecondFemale`、`decayPerSecondMale`

4. 毎フレームで移動量を軽量計算する。  
`delta = currentHipsPos - lastHipsPos`  
`moveSqr = delta.sqrMagnitude`  
`lastHipsPos = currentHipsPos`

5. 微小動作を除外する。  
`moveSqr < movementThreshold * movementThreshold` なら「移動なし」扱い。  
このとき加算しない。

6. 移動中のみ加算する。  
移動ありなら `isMoving = true`、`lastMoveTime = now`。  
`flags.FemaleGaugeUp(Time.deltaTime * moveAddPerSecondFemale, false, false)`  
`flags.MaleGaugeUp(Time.deltaTime * moveAddPerSecondMale)`

7. 停止判定を入れる。  
移動なしが続き `now - lastMoveTime >= idleDelay` なら `isMoving = false`。

8. 停止中は減衰を適用する。  
`isMoving == false` の間だけ負加算を入れる。  
`flags.FemaleGaugeUp(-Time.deltaTime * decayPerSecondFemale, false, false)`  
`flags.MaleGaugeUp(-Time.deltaTime * decayPerSecondMale)`

9. 競合回避を入れる。  
フェーズ2の加算乗っ取りON時は、フェーズ3が投入する加算値だけを対象にする。  
二重で倍率をかけないように入口を1箇所に集約する。

10. ログは間引く。  
毎フレームログ禁止。  
状態遷移時（移動開始/停止開始）だけ1行記録する。

11. GC発生を抑える。  
毎フレーム `new`、LINQ、文字列連結を禁止する。

12. 安全初期値を設定する。  
`movementThreshold = 0.003f`  
`idleDelay = 0.15f`  
`moveAddPerSecondFemale = 12f`  
`moveAddPerSecondMale = 8f`  
`decayPerSecondFemale = 6f`  
`decayPerSecondMale = 5f`

## フェーズ3の完了条件

1. 微小揺れではゲージが増えない。
2. 明確な腰移動中だけゲージが増える。
3. 停止後 `idleDelay` 経過で減衰に切り替わる。
4. フレーム落ちが発生しない（毎フレーム重処理なし）。

## 実装手順（フェーズ4: 調整変数の導入 + UIリアルタイム変更）

1. 腰追従の調整変数を追加する。  
`changeFactorXYZ`（`Vector3`）  
`dampen`（`float`）

2. 初期値を `vr_test.cs` に合わせる。  
`changeFactorXYZ = (3, 3, 10)`  
`dampen = 0.1`

3. UIに調整コントロールを追加する。  
`changeFactorX`、`changeFactorY`、`changeFactorZ` 用の入力欄またはスライダー  
`dampen` 用の入力欄またはスライダー  
加えて、ゲージ加算乗っ取りの以下もUIに出す。  
`movementThreshold`、`idleDelay`  
`moveAddPerSecondFemale`、`moveAddPerSecondMale`  
`decayPerSecondFemale`、`decayPerSecondMale`  
`femaleAddScale`、`maleAddScale`  
`femaleAddBias`、`maleAddBias`

4. UI変更イベントで値を即保存する。  
UI操作時に内部パラメータへ即代入する。  
Applyボタン方式にしない。

5. 毎フレームロジックは常に最新値を参照する。  
`scaledDelta = new Vector3(delta.x * changeFactorXYZ.x, delta.y * changeFactorXYZ.y, delta.z * changeFactorXYZ.z)`  
`SmoothDamp(..., dampen)`  
ゲージ加算/減衰/しきい値判定もUIで変更した最新値を毎フレーム参照する。

6. UI変更がその場で挙動に反映されることを保証する。  
値変更後、次フレームで腰追従が変化する設計にする。

7. 安全クランプを入れる。  
`changeFactorXYZ` は `0.0` 以上、上限を決める。  
`dampen` は `0.0` 以上で上限を決める。

8. `dampen == 0` の分岐を入れる。  
0のときは `SmoothDamp` を使わず即時反映に切り替える。

9. 設定永続化を入れる。  
終了後も値が残るように `ConfigEntry` へ保存する。

10. 軽量制約を維持する。  
UI更新時以外に文字列処理・ログ出力を増やさない。  
毎フレームは数値演算のみで完結させる。

## UI公開項目（必須）

1. 腰追従:
`changeFactorX`、`changeFactorY`、`changeFactorZ`、`dampen`

2. 移動検知:
`movementThreshold`（微小動作カット）  
`idleDelay`（停止判定遅延）

3. 加算:
`moveAddPerSecondFemale`、`moveAddPerSecondMale`  
`femaleAddScale`、`maleAddScale`  
`femaleAddBias`、`maleAddBias`

4. 減衰:
`decayPerSecondFemale`、`decayPerSecondMale`

## フェーズ4の完了条件

1. UIで `changeFactorXYZ` と `dampen` を変更できる。
2. 変更が次フレームで反映される。
3. `dampen = 0` で即時追従に切り替わる。
4. 設定を再起動後も保持できる。

## 実装手順（フェーズ5: 速さゲージ乗っ取り）

1. 速さゲージの表示元を固定で押さえる。  
`HSprite.Update()` の `imageSpeed.fillAmount` は以下を参照する。  
- Aibu: `Mathf.InverseLerp(0, speedMaxAibuBody, flags.speed)`  
- それ以外: `flags.speedCalc`

2. 速さ制御メソッドをパッチ対象にする。  
`HFlag.SpeedUpClick(...)` / `HFlag.WaitSpeedProc(...)`  
`HFlag.SpeedUpClickAibu(...)` / `HFlag.WaitSpeedProcAibu(...)`

3. 乗っ取り方式を決める。  
「入力率を差し替える方式」か「最終値をクランプして再代入する方式」のどちらかに統一する。  
混在させると二重補正になる。

4. `speed` / `speedCalc` の補正点を1か所に集約する。  
`Prefix` で引数補正、または `Postfix` で最終値補正のどちらか1つだけ使う。

5. UIに速さゲージ表示を追加する。  
バーの横に数値テキストエリアを必ず置く（`fill` 値を表示）。  
必要なら下段に `speed` / `speedCalc` の実値も表示する。

6. 速さゲージ専用ログを出す。  
`mode`、`speed`、`speedCalc`、`imageSpeed.fillAmount` 相当値を同時記録して、  
「見た目」と「内部値」のズレを追えるようにする。

7. 感度ゲージ乗っ取りと独立にON/OFFできるようにする。  
速さゲージだけを使う運用を前提に、トグルを分離する。

## フェーズ5の完了条件

1. 速さゲージ（`imageSpeed`）が想定通りに変化する。
2. バー横の数値テキストで現在値を確認できる。
3. モード切替（Aibu/非Aibu）でも破綻しない。
4. 感度ゲージ乗っ取りをOFFでも速さゲージだけ制御できる。

## 他AI実装用 固定仕様（必須）

この章は「別AIがそのまま実装に入るための固定仕様」。  
クラス名・メソッド名・パッチ名はここに合わせて統一する。

### 1. デコンパイルで確認済みの実在メソッド（必須参照）

1. `HFlag.FemaleGaugeUp(float _addPoint, bool _force, bool _isIdle = true)`
2. `HFlag.MaleGaugeUp(float _addPoint)`
3. `HSceneProc` フィールド: `public HFlag flags`
4. `HSceneProc.Update()`（private）
5. `HSceneProc.LateUpdate()`（private）
6. `HSprite.Update()`（感度バーは `flags.gaugeFemale / flags.gaugeMale`、速さバーは `flags.speed / flags.speedCalc` 参照）
7. `MotionIK` 内: `ik = animBody.GetComponent<FullBodyBipedIK>()`
8. `HFlag.SpeedUpClick(float _rateSpeedUp, float _rateSpeedMax)`
9. `HFlag.WaitSpeedProc(bool _isLock, AnimationCurve _curve)`
10. `HFlag.SpeedUpClickAibu(float _rateSpeedUp, float _rateSpeedMax, bool _drag)`
11. `HFlag.WaitSpeedProcAibu()`

### 2. DLL側の固定クラス名

1. `MainGirlHipsIkPlugin : BaseUnityPlugin`
2. `MainGirlHipsIkState`（状態保持専用）
3. `MainGirlHipsIkPatches`（Harmonyパッチ集約）

### 3. DLL側の必須メソッド名（この名前で作る）

`MainGirlHipsIkPlugin`:

1. `void Awake()`
2. `void OnDestroy()`
3. `void Update()`
4. `void LateUpdate()`
5. `void OnGUI()`
6. `void ToggleUi()`
7. `void EnableHipsIkLink()`
8. `void DisableHipsIkLink()`
9. `bool TryResolveRuntimeRefs()`
10. `Transform ResolveHipsTransform(ChaControl chaCtrl)`
11. `void CaptureBaseline()`
12. `void UpdateHipsProxyFollow(float dt)`
13. `void TickMovementAndGauge(float dt)`
14. `bool IsHipsMoving(float dt)`
15. `float BuildFemaleInjectedAdd(float baseAdd)`
16. `float BuildMaleInjectedAdd(float baseAdd)`
17. `void ApplyConfigClamp()`

`MainGirlHipsIkPatches`:

1. `static bool Prefix_FemaleGaugeUp(HFlag __instance, ref float _addPoint, bool _force, bool _isIdle)`
2. `static bool Prefix_MaleGaugeUp(HFlag __instance, ref float _addPoint)`
3. `static void Postfix_HSceneProcUpdate(HSceneProc __instance)`（必要時のみ）
4. `static void Postfix_HSceneProcLateUpdate(HSceneProc __instance)`（必要時のみ）

### 4. Harmonyパッチ指定（必須）

1. `HFlag.FemaleGaugeUp` への Prefix
2. `HFlag.MaleGaugeUp` への Prefix
3. 速さゲージ乗っ取り時は `HFlag.SpeedUpClick/WaitSpeedProc/SpeedUpClickAibu/WaitSpeedProcAibu` の Prefix/Postfix を追加
4. 直接代入対策が必要な場合のみ `HSceneProc.Update` / `LateUpdate` Postfix を追加
5. パッチクラス内で `MainGirlHipsIkPlugin.Instance` の状態を参照してON/OFF判定する

### 5. 必須状態変数（MainGirlHipsIkState）

1. IK参照:
`ChaControl targetFemaleCha`, `Transform hips`, `Transform leftController`, `Transform proxy`, `FullBodyBipedIK fbbik`
2. 基準姿勢:
`Vector3 baseHipsPos`, `Quaternion baseHipsRot`, `Vector3 baseCtrlPos`, `Quaternion baseCtrlRot`
3. 追従:
`Vector3 lastHipsPos`, `Vector3 smoothVelocity`, `float lastMoveTime`, `bool isMoving`
4. UI:
`bool uiVisible`, `bool hipsIkEnabled`, `bool gaugeHijackEnabled`
5. 調整値:
`Vector3 changeFactorXYZ`, `float dampen`, `float movementThreshold`, `float idleDelay`
6. 加算/減衰:
`float moveAddPerSecondFemale`, `float moveAddPerSecondMale`, `float decayPerSecondFemale`, `float decayPerSecondMale`
7. 乗っ取り係数:
`float femaleAddScale`, `float maleAddScale`, `float femaleAddBias`, `float maleAddBias`

### 6. 有効ボタン押下時の固定フロー（順番固定）

1. `TryResolveRuntimeRefs()` で `ChaControl` / 腰 / 左コントローラ / `FullBodyBipedIK` を解決
2. `CaptureBaseline()` で腰基準とコントローラ基準を保存
3. `proxy` を作成し、初期位置を `baseHipsPos/baseHipsRot` に合わせる
4. `proxy.SetParent(leftController, true)` を実行
5. `fbbik.enabled = true`
6. `solver.bodyEffector.target = proxy`
7. `hipsIkEnabled = true`

### 7. 毎フレームの固定フロー（軽量）

`Update()`:

1. ショートカット入力（UIトグル）
2. UI値のクランプ反映

`LateUpdate()`:

1. `hipsIkEnabled` なら `UpdateHipsProxyFollow(Time.deltaTime)`
2. `gaugeHijackEnabled` なら `TickMovementAndGauge(Time.deltaTime)`
3. 毎フレーム `GameObject.Find` を禁止（キャッシュ参照のみ）

### 8. ゲージ計算の固定ルール

1. `moveSqr = (hips.position - lastHipsPos).sqrMagnitude`
2. `moveSqr < movementThreshold^2` は移動なし
3. 移動あり:
`FemaleGaugeUp(dt * moveAddPerSecondFemale, false, false)`  
`MaleGaugeUp(dt * moveAddPerSecondMale)`
4. 停止判定成立後:
`FemaleGaugeUp(-dt * decayPerSecondFemale, false, false)`  
`MaleGaugeUp(-dt * decayPerSecondMale)`
5. Harmony Prefixで `_addPoint` に `scale + bias` を適用
6. 二重乗算を防ぐため、適用入口はPrefix 1箇所に限定

### 9. UI必須項目（再掲・固定）

1. `Enable Hips IK`（ON/OFF）
2. `Gauge Hijack`（ON/OFF）
3. `changeFactorX/Y/Z`
4. `dampen`
5. `movementThreshold`
6. `idleDelay`
7. `moveAddPerSecondFemale/Male`
8. `decayPerSecondFemale/Male`
9. `femaleAddScale/maleAddScale`
10. `femaleAddBias/maleAddBias`
11. `Speed Gauge Hijack`（ON/OFF）
12. 速さゲージバー + 横の数値テキストエリア
13. `speed/speedCalc` 実値表示

### 10. 受け入れテスト（他AIが実装完了判定に使う）

1. F8でUI表示/非表示が切り替わる
2. `Enable Hips IK` 押下で腰基準ログが1回出る
3. 押下直後に `solver.bodyEffector.target == proxy` になる
4. 左コントローラ移動で腰が追従する
5. 微小揺れではゲージ増加しない
6. 明確な移動中のみ男女ゲージが増加
7. 停止後 `idleDelay` 経過で減衰に切り替わる
8. UIスライダー変更が次フレームで反映される
9. ゲーム再起動後に設定値が保持される
10. 毎フレームログやGCスパイクが発生しない

### 11. 失敗しやすい点（固定注意）

1. `HFlag` のみパッチでは直接代入経路を完全には潰せない
2. `LateUpdate` 以外でIK反映すると純正更新に上書きされやすい
3. 腰Transform探索は `A_ROOTBONE(cf_j_hips)` 優先、失敗時のみ `cf_t_hips(work)` などへフォールバックする
4. `dampen == 0` は `SmoothDamp` を使わず即時代入に分岐
5. 毎フレームの文字列生成・反射・Findは禁止
