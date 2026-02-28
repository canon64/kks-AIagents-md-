# KKS ゲームデータ所在地マップ

## 概要

KKSのゲームデータ（AssetBundle）がどこに何として格納されているかの一覧。

## H表情データ

**ファイル**: `F:/kks/abdata/h/list/01.unity3d` (9.8MB)
**AssetBundle内の名前**: `HFaceList_cXX_XX_XX`
**内容**: H表情のパラメータリスト

### パラメータ構造
- `eyebrow` - 眉パターン
- `mouth` - 口パターン
- `openEye` - 目の開き
- `tears` - 涙レベル
- `idEyeNeck` - 目・首ID
- `pRange` - 範囲パラメータ
- `cheek` - 頬の赤み
- `eyesblink` - 瞬き制御

**用途**: StudioFacePresetToolでの表情プリセット作成時の参照

---

## 顔パーツプリセット

**ファイル**: `F:/kks/abdata/chara/face_preset_00.unity3d` (2.7KB)
**内容**: 顔パーツ（目、鼻、口等）の形状プリセット

**注意**: これは表情パターンではなく、顔のパーツ形状データ

### カスタムプリセット

**ファイル**:
- `F:/kks/abdata/custom/presets_f_00.unity3d` - 女性用カスタムプリセット
- `F:/kks/abdata/custom/presets_m_00.unity3d` - 男性用カスタムプリセット

**内容**: キャラメイカーでのカスタマイズプリセット

### 表情反映API

表情プリセットは「眉/目/口」の組み合わせIDとして `ChaControl` 経由で反映:

**パターン変更**:
- `Void ChangeEyebrowPtn(Int32, Boolean)` - 眉パターン変更
- `Void ChangeEyesPtn(Int32, Boolean)` - 目パターン変更
- `Void ChangeMouthPtn(Int32, Boolean)` - 口パターン変更

**パターン取得**:
- `Int32 GetEyebrowPtn()` - 現在の眉パターン取得
- `Int32 GetEyesPtn()` - 現在の目パターン取得
- `Int32 GetMouthPtn()` - 現在の口パターン取得

**補助API**:
- `Void ChangeEyesOpenMax(Single)` - 目の最大開き具合
- `Void ChangeMouthOpenMax(Single)` - 口の最大開き具合
- `Void ChangeMouthFixed(Boolean)` - 口を固定
- `Boolean GetMouthFixed()` - 口の固定状態取得
- `Single GetMouthOpenMax()` - 口の最大開き具合取得
- `Void DisableShapeMouth(Boolean)` - 口シェイプの無効化
- `FBSCtrlMouth get_mouthCtrl()` - 口コントロール取得

### 口の開度制御（重要）

**問題**: `ChangeMouthPtn()`で口パターンを変更しても、口は開かない。

**解決策**: `FBSCtrlMouth.FixedRate` フィールドに直接値を設定する。

```csharp
// 口パターンを変更
cha.ChangeMouthPtn(5, true);

// 口の開き範囲を設定
SetCtrlMinMax(cha, "mouthCtrl", 0.3f, 1.0f);
cha.ChangeMouthOpenMax(1.0f);

// ★重要: 口の開度を設定
cha.mouthCtrl.FixedRate = 0.3f;  // 0.0～1.0（30%開く）
```

#### FBSCtrlMouth の構造

`ChaControl.mouthCtrl` (FBSCtrlMouth型) の主要フィールド:

| フィールド | 型 | 説明 | デフォルト値 |
|-----------|-----|------|-------------|
| `FixedRate` | float | **口の現在の開度** (0.0～1.0) | 0 |
| `OpenMin` | float | 最小開度 | 0 |
| `OpenMax` | float | 最大開度 | 1 |
| `openRefValue` | float | 参照値 | 0.2 |
| `randTimeMin` | float | ランダム時間最小 | 0.5 |
| `randTimeMax` | float | ランダム時間最大 | 0.7 |
| `randScaleMin` | float | ランダムスケール最小 | 0.9 |
| `randScaleMax` | float | ランダムスケール最大 | 1.0 |
| `useAjustWidthScale` | bool | 幅調整使用 | true |

**FixedRate を設定することで、口が指定した開度で開く。**

#### 調査方法

ChaControlをリフレクションで調査:

```csharp
var mouthCtrl = cha.mouthCtrl;
var type = mouthCtrl.GetType();
var fields = type.GetFields(BindingFlags.Public | BindingFlags.Instance);
foreach (var field in fields)
{
    var value = field.GetValue(mouthCtrl);
    Logger.LogInfo($"{field.Name} = {value}");
}
```

**用途**: StudioFacePresetToolでの表情プリセット適用

---

## FaceBlendShape システム（表情・まばたき制御）

### クラス構成

```
ChaControl
  └─ FaceBlendShape (MonoBehaviour)
       ├─ BlinkCtrl (FBSBlinkControl) - まばたき計算エンジン
       ├─ EyesCtrl (FBSCtrlEyes) - 目の開閉制御
       ├─ EyebrowCtrl (FBSCtrlEyebrow) - 眉制御
       └─ MouthCtrl (FBSCtrlMouth) - 口制御
```

### FaceBlendShape.LateUpdate()（毎フレーム実行）

```csharp
private void LateUpdate()
{
    BlinkCtrl.CalcBlink();  // まばたき計算 → openRate更新

    FBSBlinkControl ctrl = BlinkCtrl;
    float num = ctrl.GetFixedFlags() != 0 ? -1f : ctrl.GetOpenRate();

    EyebrowCtrl.CalcBlend(num);
    EyesCtrl.CalcBlend(num);    // ★openRateが目に反映される
    MouthCtrl.CalcBlend(voiceValue);
}
```

**実行順序の重要性:**
- LateUpdate()の実行順序は不定（MonoBehaviourの追加順に依存）
- 他のスクリプトからopenRateを書き換えても、FaceBlendShape.LateUpdate()が後で上書きする可能性
- **解決策**: HarmonyパッチでCalcBlend()の引数を直接書き換える

### まばたきシステム（FBSBlinkControl）

#### 主要フィールド

```csharp
public class FBSBlinkControl
{
    private byte fixedFlags;         // 0=自動, 1=固定
    public byte BlinkFrequency = 30; // まばたき頻度（大きい=間隔長い、0-100）
    public float BaseSpeed = 0.15f;  // まばたき速度（秒、有効範囲0.0-0.5）
    private sbyte blinkMode;         // 0=開, 1=閉じ中, -1=開け中
    private float openRate = 1f;     // 現在の開き具合（0～1）
}
```

#### まばたきの自動ロジック

```
1. blinkMode = 0 (開いている) → ランダム待機
2. SetForceClose() → blinkMode = 1 (閉じる)
3. openRate: 1.0 → 0.0 (BaseSpeed秒かけて)
4. ランダムで1～3回繰り返し
5. SetForceOpen() → blinkMode = -1 (開ける)
6. openRate: 0.0 → 1.0 (BaseSpeed秒かけて)
7. 1に戻る
```

#### 制御API

```csharp
// まばたきON/OFF
cha.ChangeEyesBlinkFlag(true/false);

// 頻度変更（0-100、大きいほど間隔が長い）
cha.fbsCtrl.BlinkCtrl.SetFrequency(byte freq);

// 速度変更 - ★バグあり、直接代入すること
cha.fbsCtrl.BlinkCtrl.BaseSpeed = 0.15f;  // 直接代入

// 現在の開き具合取得
float openRate = cha.fbsCtrl.BlinkCtrl.GetOpenRate();
```

**ゲーム本体のバグ:**
```csharp
// SetSpeed()のバグ - 使用しないこと
public void SetSpeed(float value)
{
    BaseSpeed = Mathf.Max(1f, value);  // ★バグ！
}
```
- `BaseSpeed`の有効範囲は`0.0～0.5`
- `Mathf.Max(1f, value)`で1.0未満が全部1.0になる
- **解決策:** `BaseSpeed`に直接代入する

#### 目の開閉範囲制御（Min/Max OpenRate）

**失敗する方法:** リフレクションでopenRateを直接書き換え
```csharp
// ダメな例 - LateUpdate()の実行順序に依存
var field = blinkCtrl.GetType().GetField("openRate", ...);
field.SetValue(blinkCtrl, clampedValue);
```

**成功する方法:** Harmonyパッチで引数を書き換え
```csharp
[HarmonyPrefix]
[HarmonyPatch(typeof(FBSCtrlEyes), "CalcBlend")]
private static void FBSCtrlEyes_CalcBlend_Prefix(ref float blinkRate)
{
    blinkRate = Mathf.Clamp(blinkRate, minOpenRate, maxOpenRate);
}
```

**動作フロー:**
```
FaceBlendShape.LateUpdate()
  ↓
BlinkCtrl.CalcBlink() → openRate計算
  ↓
EyesCtrl.CalcBlend(openRate) ← ★Harmonyが引数を書き換え
  ↓
openRate = clampedValue
  ↓
CalculateBlendShape() → 確実に反映
```

**利点:**
- 実行順序に依存しない
- 引数を直接書き換えるため確実
- 副作用なし

### FBSCtrlEyes（目の開閉制御）

```csharp
public void CalcBlend(float blinkRate)
{
    if (0f <= blinkRate)
    {
        openRate = blinkRate;  // ★ここに値が入る
    }
    CalculateBlendShape();  // 実際にブレンドシェイプに反映
}
```

### 頬の赤み（Cheek）制御

**方法1:** ChaControl経由
```csharp
cha.ChangeHohoAkaRate(float value);  // 0.0～1.0
```

**方法2:** fileStatus直接操作（StudioGaugeBarで使用）
```csharp
cha.fileStatus.hohoAkaRate = value;  // 0.0～1.0
```

**リアルタイム連動の設計原則:**
- リアルタイム連動する値（Cheek、まばたき）→ 毎フレーム更新する側が制御
- シーン的な表情（Eyebrow、Eye、Mouth）→ プリセット側が制御
- 競合を避けるため、役割を明確に分離する

---

## 音声データ

（未調査 - 必要に応じて追記）

---

## アニメーションデータ

（未調査 - 必要に応じて追記）

---

## キャラクターボーン構造

### 主要ボーン

キャラクターの位置・姿勢トラッキングに使用される主要ボーン:

| ボーン名 | 用途 | 説明 |
|---------|------|------|
| `cf_j_root` | ルートボーン | キャラクター全体の基準点 |
| `cf_n_height` | 身長基準 | キャラクターの高さ基準点 |
| `cf_j_hips` | 腰ボーン | 腰の位置・動きトラッキング |

### アクセス方法

```csharp
// OCIChar経由でChaControlを取得
var ociChar = Studio.Studio.Instance.dicObjectCtrl[objectId] as OCIChar;
var cha = ociChar?.charInfo;

// ボーン階層からTransformを取得
Transform root = cha.objBodyBone.transform.Find("cf_j_root");
Transform height = cha.objBodyBone.transform.Find("cf_n_height");
Transform hips = root?.Find("cf_n_height/cf_j_hips");

// 位置トラッキング
Vector3 hipPosition = hips.position;
```

**用途例:**
- 外部デバイス連携（位置同期）
- モーションキャプチャ
- カメラ追従
- 物理演算

---

## その他のリストデータ

以下のディレクトリに各種リストデータが格納:
- `F:/kks/abdata/h/list/` - H関連
- `F:/kks/abdata/action/list/` - アクション
- `F:/kks/abdata/list/` - 全般（アイテム、パーツ等）
- `F:/kks/abdata/map/list/` - マップ

全て `.unity3d` AssetBundle形式。
