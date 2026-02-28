# 本編Hシーン ボイスデータ紐づけ完全解析

- 調査日: 2026-02-25
- 対象: `KoikatsuSunshine` 本編Hシーン（Aibu/Houshi/Sonyu/3P/3PDark）
- デコンパイル元: `KoikatsuSunshine_Data/Managed/Assembly-CSharp.dll`

---

## 結論（先に要点）

本編Hシーンのボイス再生は `HVoiceCtrl` クラスが一手に担う。
データは **`h/list/*.unity3d`** に入った2種類のScriptableObject（`VoiceAllData` / `VoicePatternData`）から読み込まれる。
「どのボイスを鳴らすか」は **`mode × 100 + patternKey`** という整数で `flags.voice.playVoices[n]` に設定され、`HVoiceCtrl.PlayVoice()` が解釈して実ファイルを再生する。

---

## 1. 全体フロー

```
HScene.Awake() / Start()
  └─ HFlag.lstABName = CommonLib.GetAssetBundleNameListFromPath("h/list/")
       ↓ h/list/00.unity3d, 01.unity3d, 30.unity3d, ... を収集

HSceneProc.Init() / HVoiceCtrl.Init(_chaFolder, _chaFolder1)
  └─ h/list/*.unity3d から 4種のアセットを読み込み
       ├─ personality_voice_{chaFolder}  → VoiceAllData  (キャラ別ボイス定義)
       ├─ voice_{mode:00}_00            → VoicePatternData (モード別パターン)
       ├─ personality_breath_{chaFolder} → BreathAllData  (キャラ別呼吸ボイス)
       └─ breath_{mode:00}_00           → BreathPatternData (呼吸パターン)

毎フレーム: HSceneProc.Update() → lstProc[mode].Proc()
  └─ 各Hモードクラス (HAibu/HHoushi/HSonyu 等)
       └─ flags.voice.playVoices[0] = mode*100 + patternKey を設定

HVoiceCtrl.PlayVoice(female, ...)
  └─ playVoices[n] を解釈 → dicVoicePtn → dicVoiceIntos → pathAsset/nameFile 取得
  └─ Utils.Voice.OncePlayChara(Setting) → Manager.Voice.Play() → AudioSource再生
  └─ female.SetLipSync() で口パク連動
```

---

## 2. データ構造

### 2.1 `VoiceAllData` (personality_voice_{chaFolder})

キャラクター別のボイスファイル定義。各ボイスIDに対してファイルパスと再生条件を格納。

```
VoiceAllData
  param: List<Param>
    Param
      mode: int               ← Hモード番号 (0-10)
      data: List<Voicedata>
        Voicedata
          id: int             ← ボイスID (VoicePatternData の lstVoice に対応)
          info[0..3]: ...     ← 歓喜度(0=控えめ/1=通常/2=興奮/3=絶頂)別データ
            pathAsset: string ← AssetBundleパス (例: "sound/data/pcm/c13/h/01")
            nameFile: string  ← ファイル名 (例: "h_so_13_03_025")
            priority: int     ← 優先度 (高いほど先に再生)
            playConditions[15]: bool[] ← 再生条件フラグ15個
            face: int         ← 再生時に適用する表情ID (-1=変更なし)
            eyeneck: int      ← 再生時に適用する目・首方向ID (-1=変更なし)
            isOneShot: bool   ← 一度再生したら以降再生しない
            notOverwrite: bool ← 上書き禁止
            word: string      ← セリフテキスト
```

**キャラ識別子 `_chaFolder`**: `Manager.Voice.infoTable[personalityNo].FileName` から取得。
（例: パーソナリティ0番なら `"c00"` に対応するファイル名）

### 2.2 `VoicePatternData` (voice_{mode:00}_00)

モード別のボイスパターン定義。「どのアニメーション/状態のときにどのボイスIDを再生するか」のマッピング。

```
VoicePatternData
  param: List<Param>
    Param
      id: int               ← パターンキー (playVoices % 100 に対応)
      lstInfo: List<VoicePtnInfo>
        VoicePtnInfo
          lstAnimID: List<int>     ← 対象アニメーションID (空なら全アニメ対象)
          lstConditions: List<int> ← 再生条件IDリスト
          lstVoice: List<int>      ← 再生候補ボイスIDリスト → VoiceAllDataのIDに対応
          lstSecondConditions: List<int> ← 2人目用条件 (3P)
          lstSecondVoice: List<int>      ← 2人目用ボイスID (3P)
          state: int              ← 使用する歓喜度インデックス
          link: LinkInfo          ← 連鎖再生設定
```

---

## 3. playVoices の値とモード対応

`flags.voice.playVoices[n]` に設定される値の意味：

```
value / 100 = mode  (dicVoicePtn / dicVoiceIntos のキー)
value % 100 = patternKey  (dicVoicePtn[main][mode] のキー)
```

### Hモードと値の対応表

| Hモードクラス | playVoices の値 | mode | voice_{mode:00}_00 |
|-------------|----------------|------|--------------------|
| (共通)       | 0xx            | 0    | voice_00_00        |
| `HAibu`     | 1xx            | 1    | voice_01_00        |
| `HHoushi`   | 2xx            | 2    | voice_02_00        |
| `HSonyu`    | 3xx            | 3    | voice_03_00        |
| (未確認)     | 4xx            | 4    | voice_04_00        |
| (未確認)     | 5xx            | 5    | voice_05_00        |
| (未確認)     | 6xx            | 6    | voice_06_00        |
| `H3PHoushi` | 7xx            | 7    | voice_07_00        |
| `H3PSonyu`  | 8xx            | 8    | voice_08_00        |
| `H3PDarkHoushi` | 9xx        | 9    | voice_09_00 (?)    |
| `H3PDarkSonyu`  | 10xx       | 10   | voice_10_00 (?)    |

**注意**: `HVoiceCtrl.Init` の初期化ループは `k < 9`（0〜8）だが、H3PDark系は 9xx/10xx を使用。
この矛盾については要追加調査。

---

## 4. HSonyu (挿入) のボイス設定パターン

`_tmp_HSonyu.cs` で確認した主なパターン：

| playVoices の値 | 意味 |
|----------------|------|
| `300 + num2`   | 通常ループ呼吸中のセリフ (SONYU_WAIT_IDLE) |
| `303 + num2`   | **膣（コーカン）挿入アクション** |
| `306 + num2`   | **アナル挿入アクション** |
| `307 + num2`   | 絶頂直前 (終了系ボイス) |
| `308 + num2`   | 絶頂直前 (別パターン) |
| `309 + num2`   | 速度変化対応セリフ |
| `324 + num2`   | ループ開始 (low speed) |
| `325 + num2`   | ループ中間 |
| `326 + num2`   | ループ激し系 |
| `334 + num2`   | ループ (別バリエーション) |

`num2` = `flags.nowAnimationInfo.stateType` または類似フィールドで歓喜度/アニメ状態を示す小値。

### 膣 vs アナルの分岐

`flags.isAnalPlay` が切り替えのフラグ:

```csharp
// 挿入直後
if (click == ClickKind.insert)       flags.isAnalPlay = false;
if (click == ClickKind.insert_anal)  flags.isAnalPlay = true;

// ボイス設定
flags.voice.playVoices[0] = ((!flags.isAnalPlay) ? 303 : 306) + num2;

// 再生条件も別々に設定
if (!flags.isAnalPlay) flags.SetInsertKokanVoiceCondition();
else                   flags.SetInsertAnalVoiceCondition();
```

どちらも **mode=3 (`voice_03_00`)** を使うが、パターンキーと再生条件で別ボイスに振り分けられる。

---

## 5. PlayVoice の実行ロジック

```csharp
// HVoiceCtrl.PlayVoice (line 1472+)
int mode   = flags.voice.playVoices[_main] / 100;
int key    = flags.voice.playVoices[_main] % 100;

VoicePtn voicePtn = dicVoicePtn[_main][mode][key];
Dictionary<int, Dictionary<int, VoiceInfo>> voiceDict = dicVoiceIntos[_main][mode];

// VoicePtnInfo の lstConditions を評価して再生するボイスIDリストを作成
List<VoiceSelect> playList = GetPLayNumVoiceList(matchedInfos, voiceDict, mode, _female, _main);

// priority 順 + ランダムでソートして先頭を選択
nowVoices[_main].voiceInfo = voiceDict[playList[0].state][playList[0].playnum];

// 実際の再生
Utils.Voice.Setting s = new Utils.Voice.Setting {
    no             = flags.lstHeroine[_main].voiceNo,    // パーソナリティ番号
    assetBundleName = nowVoices[_main].voiceInfo.pathAsset,
    assetName      = nowVoices[_main].voiceInfo.nameFile,
    pitch          = flags.lstHeroine[_main].voicePitch,
    voiceTrans     = flags.transVoiceMouth[_main]        // 口の Transform
};
_female.SetLipSync(Utils.Voice.OncePlayChara(s));        // 再生 + 口パク
```

再生後：
- `faceLists[_main].SetFace(voiceInfo.face, ...)` で表情変更
- `flags.voice.eyenecks[_main] = voiceInfo.eyeneck` で目線変更

---

## 6. 状態 (歓喜度) の決定

`VoiceInfo` は `info[0..3]` の4段階（歓喜度）に分かれる。
どの段階が選ばれるかは `GetStateSelect(state, _main)` が決定：

```csharp
// state は VoicePtnInfo.state から来る
// _main は 0(ヒロイン1人目) / 1(2人目, 3P時)
int stateSelect = GetStateSelect(list[j].state, _main);
// → dicVoiceIntos[_main][mode][stateSelect][voiceId] の pathAsset/nameFile を参照
```

歓喜度ステート:
- 0: 控えめ (gauge 低)
- 1: 通常
- 2: 興奮
- 3: 絶頂寸前 (gauge 高)

これは `flags.gaugeFemale` の現在値から算出される。

---

## 7. `_chaFolder` の決定方法

`HVoiceCtrl.Init(string _chaFolder, string _chaFolder1)` の引数。

- `fixPersonal` 辞書（パーソナリティ変換テーブル）を通して正規化
- 例: キャラのパーソナリティが `"00"` なら `personality_voice_00` を検索
- `Manager.Voice.infoTable[heroineVoiceNo].FileName` から取得される（HSceneProc 初期化時）

---

## 8. データの在処

全データは `h/list/` 以下の AssetBundle:

```
h/list/
  00.unity3d  ← 本体定義 (メイン)
  01.unity3d  ← 追加定義
  30.unity3d  ← DLC30
  31.unity3d  ← DLC31
  60.unity3d  ← DLC60
  70.unity3d  ← DLC70
  71.unity3d
  81.unity3d
  91.unity3d
```

各 .unity3d に含まれるアセット（TextResourceRedirector で差し替え可能）：

| アセット名                        | 型                | 用途 |
|----------------------------------|-------------------|------|
| `personality_voice_{chaFolder}`  | VoiceAllData      | キャラ別ボイスファイル定義 |
| `personality_breath_{chaFolder}` | BreathAllData     | キャラ別呼吸ボイス定義 |
| `voice_{mode:00}_00`             | VoicePatternData  | モード別ボイスパターン |
| `breath_{mode:00}_00`            | BreathPatternData | 呼吸パターン |
| `voice_{mode:00}_00` (short)     | ShortBreathData   | 短い呼吸データ |
| `shortbreath_00`                 | ShortBreathPatternData | 短い呼吸パターン |

---

## 9. MOD開発への応用

### カスタムボイスを挿入に追加する場合

1. **VoiceAllData のエントリを追加**（または TextResourceRedirector で差し替え）
   - `personality_voice_{chaFolder}` に新規ボイスIDを登録
   - `pathAsset`, `nameFile` にカスタムWAVのパスをセット
   - 再生条件 `playConditions[15]` を適切に設定

2. **VoicePatternData の lstVoice に追加**
   - 挿入 (mode=3) の対象パターン (`voice_03_00`) の `lstVoice` に新IDを追加

3. **HVoiceCtrl.LoadVoice / LoadVoicePtn を Harmony Prefix でフック**
   - カスタムデータを dicVoiceIntos / dicVoicePtn に直接注入することが可能

### ボイス再生をフックする最も確実な場所

```csharp
// Utils.Voice.Setting が組み立てられた直後 → OncePlayChara の Prefix
[HarmonyPrefix]
[HarmonyPatch(typeof(Utils.Voice), "OncePlayChara")]
static bool Prefix(Utils.Voice.Setting _setting, ref AudioSource __result)
{
    // _setting.assetBundleName, _setting.assetName で再生対象を確認・置換
}
```

または

```csharp
// HVoiceCtrl.PlayVoice の Postfix で nowVoices[_main].voiceInfo を書き換え
```

---

## 10. 関連ファイル

| ファイル | 用途 |
|---------|------|
| `../_decomp/_tmp_HSonyu.cs` | 挿入Hモードの playVoices 設定 |
| `../_decomp/_tmp_HAibu.cs` | 愛撫Hモードの playVoices 設定 |
| `../_decomp/_tmp_HHoushi.cs` | 奉仕Hモードの playVoices 設定 |
| `../_decomp/_tmp_H3PSonyu.cs` | 3P挿入の playVoices 設定 |
| `../_decomp/_tmp_H3PDarkSonyu.cs` | 3PDark挿入の playVoices 設定 |
| `../_decomp/HActionBase.latest.cs` | PlayVoice / IsVoiceWait 実装 |
| `../_decomp/HFlag.latest.cs` | HFlag.lstABName, VoiceFlag |
| `../_decomp/_tmp_HScene.cs` | lstABName の初期化 (h/list/) |

---

## 11. lstProc とボイスモード番号の正確な対応

調査日: 2026-02-27（`HSceneProc.latest.cs` + `HVoiceCtrl.decompiled.cs` から確認）

```
lstProc[0] = HAibu         → playVoices = 1xx  → dicVoicePtn[_main][1]
lstProc[1] = HHoushi       → playVoices = 2xx  → dicVoicePtn[_main][2]
lstProc[2] = HSonyu        → playVoices = 3xx  → dicVoicePtn[_main][3]
lstProc[3] = HMasturbation → playVoices = 4xx  → dicVoicePtn[_main][4]
lstProc[4] = HPeeping      → playVoices = 5xx  → dicVoicePtn[_main][5]
lstProc[5] = HLesbian      → playVoices = 6xx  → dicVoicePtn[_main][6]
lstProc[6] = H3PHoushi     → playVoices = 7xx  → dicVoicePtn[_main][7]
lstProc[7] = H3PSonyu      → playVoices = 8xx  → dicVoicePtn[_main][8]
lstProc[8] = H3PDarkHoushi → playVoices = 9xx  → ★ dicVoicePtn[9] は存在しない（後述）
lstProc[9] = H3PDarkSonyu  → playVoices = 10xx → ★ dicVoicePtn[10] は存在しない（後述）
```

---

## 12. 女性上位オフセット `num2 = isFemaleInitiative ? 38 : 0`

各モードクラスは `flags.nowAnimationInfo.isFemaleInitiative` を見て
patternKey に **+38 のオフセット** を加える。

```
通常体位 (num2=0):  playVoices = mode*100 + 基本キー
女性上位 (num2=38): playVoices = mode*100 + 基本キー + 38
```

カスタムボイスを追加する場合、**通常・女性上位の両方のパターンキー**に登録が必要。

---

## 13. 各モード playVoices パターンキー 完全表

### HSonyu (dicVoicePtn mode=3)  ※ `+ num2` = 女性上位時 +38

| playVoices 値 | 状況 |
|---|---|
| `300 + num2` | 待機中（挿入状態・息のみ） |
| `303 + num2` | 膣挿入ループ（通常） |
| `306 + num2` | アナル挿入ループ |
| `307 + num2` | 挿入直後（初回） |
| `308 + num2` | 挿入直後（繰り返し） |
| `309 + num2` | 速度変化ループ |
| `310 + num2` | ループ低速 |
| `311 + num2` | ループ低速・速め |
| `312 + num2` | ループ通常 |
| `313 + num2` | ループ通常・速め |
| `314 + num2` | ゲージ70%超え（スロー） |
| `315 + num2` | ゲージ70%超え（速め） |
| `316 + num2` | 男ゲージ70%超え（スロー） |
| `317 + num2` | 男ゲージ70%超え（速め） |
| `318 + num2` | 絶頂 orgW/orgS |
| `319 + num2` | 絶頂 sameS/sameW |
| `320 + num2` | 中出し（コンドームなし） |
| `321 + num2` | 外出し |
| `322 + num2` | アナル中出し |
| `323 + num2` | 中出し（コンドームあり） |
| `324 + num2` | 射精開始 inside |
| `325 + num2` | 射精開始 orgW/sameW |
| `326 + num2` | 射精開始 orgS/sameS |
| `327 + num2` | 絶頂後ループ |
| `328 + num2` | 絶頂後ループ orgW |
| `329 + num2` | 絶頂後ループ orgS |
| `330 + num2` | 絶頂後ループ sameS |
| `331 + num2` | 射精後セリフ（inside/outside） |
| `332 + num2` | 射精後セリフ（sameW/sameS） |
| `333 + num2` | 射精後セリフ（その他） |
| `334 + num2` | 射精開始 outside |
| `335 + num2` | 外出し後セリフ |
| `336 + num2` | 抜いた直後 |

### HAibu (dicVoicePtn mode=1)

| playVoices 値 | 状況 |
|---|---|
| `100` | 待機中（低速・息のみ） |
| `102` | Orgasm後待機 |
| `104` | 嫌がり（胸L） |
| `105` | 嫌がり（股間） |
| `106` | 嫌がり（アナル） |
| `107` | 嫌がり（お尻） |
| `108` | 嫌がり（胸R） |
| `109` | 嫌がり（後ろ系） |
| `110` | 嫌がり（前面系） |
| `141` | ゲージ70%超え |
| `142` | 絶頂 start |
| `143` | 絶頂 A（後） |
| `1` / `11` | 再開始（通常/特殊カテゴリ） |
| `12` | 速度アップ（特殊カテゴリ） |

### HHoushi (dicVoicePtn mode=2)

| playVoices 値 | 状況 |
|---|---|
| `200` | 待機中 |
| `201` | ループ低速 |
| `202` | ループ速め |
| `203` | ループ stop |
| `204` | 外出し |
| `205` | 口内 |
| `206` | パイ擦り finish |
| `207` | 口外（後始末） |
| `208` | 飲む |
| `209` | 吐く |
| `1` / `11` | 再開始（通常/特殊カテゴリ） |

### H3PSonyu (dicVoicePtn mode=8)  ※ `+ num4` = 女性上位時 +38

構造は HSonyu と同じ。base が 800 番台。

| HSonyu キー | H3PSonyu キー |
|---|---|
| `300 + num2` | `800 + num4` |
| `303 + num2` | `803 + num4` |
| `306 + num2` | `806 + num4` |
| `307 + num2` | `807 + num4` |
| `308 + num2` | `808 + num4` |
| `309 + num2` | `809 + num4` |
| `310-336 + num2` | `810-836 + num4` |

H3PDarkSonyu は `1000 + xx` を設定するが、dicVoicePtn[10] が存在しないため **実際には再生されない**（後述）。

---

## 14. VoiceAllData の mode 番号共有（重要）

`LoadVoice` 内のモード振り分けロジック（`HVoiceCtrl.decompiled.cs`）：

```csharp
// i = dicVoiceIntos のキー (0..8)
// item2.mode = VoiceAllData.param[*].mode
item2.mode != (MathfEx.IsRange(7, i, 8, isEqual: true) ? (i - 5) : i)
```

| dicVoiceIntos キー i | 読み込む VoiceAllData.mode |
|---|---|
| 0 | 0（共通/Start） |
| 1 | 1（Aibu） |
| 2 | 2（Houshi） |
| 3 | 3（Sonyu） |
| 4 | 4（Masturbation） |
| 5 | 5（Peeping） |
| 6 | 6（Lesbian） |
| 7 | **2**（H3PHoushi は Houshi のデータを流用） |
| 8 | **3**（H3PSonyu は Sonyu のデータを流用） |

**実装上の影響**: Sonyu (mode=3) にカスタムボイスIDを追加すれば、
`dicVoiceIntos[_main][3]` と `[_main][8]` の両方に同じデータが入る。
→ **HSonyu と H3PSonyu は共通のボイスプールを使う**。パターン側（dicVoicePtn）で分岐。

---

## 15. isPlay フラグの動作

```
VoiceInfo.isPlay = false  → 再生候補（GetPLayNumVoiceList に入る）
VoiceInfo.isPlay = true   → 再生済み、候補外

候補がゼロになったとき:
  → isOneShot=false の voiceInfo をまとめてリセット (isPlay=false)
  → 再度候補を作り直す → 再生再開
```

**カスタムボイス設定指針:**

| フィールド | 推奨値 | 理由 |
|---|---|---|
| `isOneShot` | `false` | プール方式で繰り返し再生 |
| `isOneShot` | `true` | 初挿入など1シーンで1回限りの場合 |
| `notOverwrite` | `false` | 常に上書き可能（推奨） |
| `isPlayConditions[0..14]` | 全 `false` | 条件なし、常に候補に入る |
| `priority` | `0` | 既存ボイスと同等の優先度 |

---

## 16. H3PDark（mode=8/9 in lstProc）の結論

| 項目 | 結論 |
|---|---|
| H3PDarkHoushi の playVoices | `900 + n` → voice mode = 9 |
| H3PDarkSonyu の playVoices | `1000 + n` → voice mode = 10 |
| dicVoicePtn の初期化範囲 | `for(k<9)` → keys 0-8 のみ |
| VoiceProc での挙動 | `ContainsKey(9/10)` が false → return false → **実質無音** |
| `InitD()` の用途 | 同期版の Init（Coroutine なし）。0-8 の初期化は同じ |
| MOD の優先度 | **低い**（標準 dicVoicePtn に mode 9/10 がないため乗れない） |

H3PDark に声を追加したい場合は dicVoicePtn に mode 9/10 を手動で Add してから
LoadVoicePtn を呼ぶ必要があるが、実際に PlayPtnCheckD 経由で動くかは要検証。

---

## 17. Harmony 注入の具体的なフック箇所

```csharp
// ① VoiceInfo の注入（LoadVoice Postfix）
// private bool LoadVoice(string _chaFolder, int _main)
var mLoadVoice = AccessTools.Method(typeof(HVoiceCtrl), "LoadVoice",
    new Type[] { typeof(string), typeof(int) });
// Postfix で __instance.dicVoiceIntos[_main][mode][state][customId] = new VoiceInfo{...}

// ② VoicePtn への ID 追加（LoadVoicePtn Postfix）
// private bool LoadVoicePtn(int _mode, int _main)
var mLoadVoicePtn = AccessTools.Method(typeof(HVoiceCtrl), "LoadVoicePtn",
    new Type[] { typeof(int), typeof(int) });
// Postfix で __instance.dicVoicePtn[0][_mode][patternKey].lstInfo[n].lstVoice.Add(customId)
// ※ 女性上位も忘れずに (patternKey+38 のパターンにも追加)

// ③ 実ファイル再生のインターセプト（OncePlayChara Prefix）
// public static AudioSource Utils.Voice.OncePlayChara(Utils.Voice.Setting)
[HarmonyPatch(typeof(Utils.Voice), "OncePlayChara",
    new Type[] { typeof(Utils.Voice.Setting) })]
static bool Prefix(Utils.Voice.Setting _setting, ref AudioSource __result)
{
    if (!_setting.assetBundleName.StartsWith("CUSTOM/")) return true;
    // ディスクから WAV を同期読み込み → AudioClip.Create + SetData → Play
    return false;
}
```

**注入タイミング**: `HVoiceCtrl.Init()` はコルーチンなので、
`HSceneProc.Start()` の Postfix などで `StartCoroutine` の完了を待つか、
`LoadVoiceList` の末尾を Postfix でフックする。

---

## 18. プラグイン設計（2026-02-27 策定）

### 設計の核心：キャラ別処理が必要な理由

```
dicVoiceIntos[_main][mode][state][voiceId] = VoiceInfo
             ↑
        ヒロインスロット (0=1人目, 1=2人目)
```

- `LoadVoice(_chaFolder, _main)` はスロット別・キャラ別に呼ばれる
- `LoadVoicePtn(_mode, _main)` はモード別に呼ばれ、パターンは全スロット共有
- `GetPLayNumVoiceList` は `_dicUseVoiceInfo[stateSelect].ContainsKey(voiceId)` で存在確認 →
  **IDが存在しなければ静かにスキップ**（ContainsKey確認済み）

→ **パターン側には全キャラ共通のIDを入れ、VoiceInfoの実態だけキャラ別スロットに入れる**設計が成立する。

### 処理フロー

```
[起動時] VoiceRegistry.Scan()
  voices/ フォルダを走査
  キャラ×モード×state → WAVファイルリストを構築
  全キャラ横断での最大WAV数を記録 → カスタムID上限を決定
  例: sonyu で最大4ファイル → ID 9000〜9003 を使う

[Hシーン Init 時]
  Patch① LoadVoice("c13", _main=0) Postfix
    → dicVoiceIntos[0][3][0..3][9000..9002] = VoiceInfo{pathAsset="CUSTOM", nameFile="c13/sonyu/001.wav"}

  Patch① LoadVoice("c00", _main=1) Postfix  ← 3P時
    → dicVoiceIntos[1][3][0..3][9000..9003] = VoiceInfo{pathAsset="CUSTOM", nameFile="c00/sonyu/001.wav"}

  Patch② LoadVoicePtn(_mode=3, _main) Postfix
    → 該当パターンの lstVoice に 9000〜9003 を追加（全キャラ共通）
    → c13(3ファイル)は 9003 が dicVoiceIntos に存在しない → スキップ（正常）

[再生時]
  Patch③ Utils.Voice.OncePlayChara Prefix
    → assetBundleName == "CUSTOM" なら
      pluginDir/voices/{assetName} からWAVをディスク読み
      AudioClip 生成 → AudioSource 再生 → return false
```

### フォルダ構造

```
BepInEx/plugins/HVoiceAdder/
  HVoiceAdder.dll
  voices/
    c13/                ← _chaFolder と完全一致
      sonyu/            ← ループ挿入中に再生 (mode=3, keys 3/6/41/44)
        001.wav
        002.wav
        003.wav
      aibu/             ← 愛撫中 (mode=1, key 0)
        001.wav
      houshi/           ← 奉仕中 (mode=2, keys 1/2)
        001.wav
    c00/
      sonyu/
        001.wav
        002.wav
        003.wav
        004.wav
    c24/
      sonyu/
        001.wav
```

**state（歓喜度）別に分けたい場合（任意）**:

```
voices/c13/sonyu/
  st0/   ← 控えめキャラのみ (HExperience=0)
    001.wav
  st3/   ← 最積極的キャラのみ (HExperience=3)
    001.wav
  001.wav  ← state指定なし → 全state(0-3)に登録
```

### カスタムID割当ルール

```
モードごとに独立した ID スペース
  dicVoiceIntos[_main][mode] がモードごと別辞書 → mode間で ID が重複しても問題なし

  sonyu (dicMode=3): 9000〜
  aibu  (dicMode=1): 9000〜  ← 同じ番号でもmode辞書が別なので衝突なし
  houshi(dicMode=2): 9000〜

全キャラの sonyu ファイル数の最大 = 4 (c00)
→ lstVoice に [9000, 9001, 9002, 9003] を登録
→ c13(3ファイル) は 9000〜9002 のみ dicVoiceIntos に入る
→ 9003 は lookup miss → スキップ（正常）
```

### パターンキーへの注入対象（デフォルト）

```
Sonyu (mode=3):
  通常体位 膣ループ:    key 3  (playVoices=303)
  通常体位 アナルループ: key 6  (playVoices=306)
  女性上位 膣ループ:    key 41 (playVoices=341 = 303+38)
  女性上位 アナルループ: key 44 (playVoices=344 = 306+38)
  速度変化系も追加したければ key 10〜13, 48〜51

Aibu (mode=1):
  待機:  key 0 (playVoices=100)

Houshi (mode=2):
  低速:  key 1 (playVoices=201)
  速め:  key 2 (playVoices=202)
```

Settings.json で変更可能にする。

### VoiceInfo の設定値

```csharp
new HVoiceCtrl.VoiceInfo {
    id              = 9000,
    priority        = 0,            // 既存と同等の優先度
    pathAsset       = "CUSTOM",     // 再生時の判別マーカー
    nameFile        = "c13/sonyu/001.wav",  // voices/ 配下の相対パス
    isPlayConditions = new bool[15],// 全 false = 条件なし・常に候補
    isOneShot       = false,        // プール方式で繰り返し再生
    notOverwrite    = false,
    face            = -1,           // 表情変更なし
    eyeneck         = -1,
}
```

### クラス構成

```
HVoiceAdderPlugin : BaseUnityPlugin
  [BepInProcess("KoikatsuSunshine")]  ← 本編専用
  Awake() → VoiceRegistry.Scan() → Harmony.PatchAll()

VoiceRegistry (static)
  Scan(pluginDir)
    voices/ を走査してキャラ×モード×state → List<string>(WAVパス) を構築
    全モードの最大WAV数 → customIdCount[modeName] を決定
  GetWavPaths(chaFolder, modeName, state) → List<string>
  GetMaxCount(modeName) → int

HVoicePatches
  LoadVoice_Postfix   (__instance, _chaFolder, _main)
  LoadVoicePtn_Postfix(__instance, _mode, _main)
  OncePlayChara_Prefix(_setting, ref __result)

WavLoader (static)
  Load(string path) → AudioClip   ← File.ReadAllBytes → 16bit PCM パース
```

---

## 19. 確認済み vs 未確認（実装前チェックリスト）

### 確認済み（コードで裏取り済み）

| 項目 | 根拠ファイル |
|------|-------------|
| `dicVoiceIntos` / `dicVoicePtn` の型・構造 | HVoiceCtrl.decompiled.cs |
| `LoadVoice(string, int)` の引数・挙動 | 同上 |
| `LoadVoicePtn(int, int)` の引数・挙動 | 同上 |
| パターンキー全表（303+num2 等） | _tmp_HSonyu/HAibu/HHoushi.cs |
| `ContainsKey` miss → 静かにスキップ | GetPLayNumVoiceList 実装 |
| `isPlay` フラグのリセット動作 | VoiceProc 実装 |
| VoiceAllData の mode 共有（i=7→2, i=8→3） | LoadVoice ループ |
| `GetStateSelect` の動作（HExperience → state 0-3） | HVoiceCtrl.decompiled.cs |
| lstProc と mode 番号の対応 | HSceneProc.latest.cs |

### 未確認（実装前に要デコンパイル）

| 項目 | 理由 | コマンド |
|------|------|---------|
| `Utils.Voice.OncePlayChara` のシグネチャ | Harmony パッチに必須 | `ilspycmd -t "Utils.Voice" Assembly-CSharp.dll` |
| `Utils.Voice.Setting` の全フィールド | VoiceInfo → Setting の組み立てに必要 | 同上 |
| `Manager.Voice.IsPlay` の仕組み | カスタム AudioSource が IsPlay チェックを通過するか | `ilspycmd -t "Manager.Voice" Assembly-CSharp.dll` |

**実装の分割方針:**
- **Patch①②（注入）**: 今すぐ書ける。確認不要
- **Patch③（WAV再生）**: `Utils.Voice` のデコンパイル後に実装する


---

## 20. Utils.Voice 解析結果（確認済み）

`Utils.Voice` は **`Illusion.Game.Utils` の内部ネストクラス**（`Illusion.Game.Utils+Voice`）。
`ilspycmd -t "Utils.Voice"` では見つからず、`ilspycmd -t "Illusion.Game.Utils"` で発見。

### Utils.Voice.Setting フィールド（全項目）

| フィールド | 型 | デフォルト | 説明 |
|-----------|------|-----------|------|
| assetBundleName | string | "" | AssetBundle パス |
| assetName | string | "" | Bundle 内アセット名 |
| type | Manager.Voice.Type | PCM | enum 値は PCM のみ |
| no | int | - | キャラ番号（personalities）|
| pitch | float | 1f | 再生ピッチ |
| voiceTrans | Transform | null | 口元 Transform（3D 音源位置） |
| delayTime | float | 0f | 再生遅延 |
| fadeTime | float | 0f | フェードイン時間 |
| isAsync | bool | true | 非同期読み込みフラグ |
| settingNo | int | -1 | SoundSettingData 番号（-1=不使用） |
| isPlayEndDelete | bool | true | 再生終了時に GameObject 削除 |
| isBundleUnload | bool | false | 再生後に AssetBundle をアンロードするか |
| is2D | bool | false | 2D 音源強制フラグ |

### Utils.Voice.OncePlayChara の実装

```
OncePlayChara(Setting s):
  Manager.Voice.Loader loader を Setting から組み立て
  Manager.Voice.OncePlayChara(loader) を呼ぶだけ（薄いラッパー）
```

**重要**: Utils.Voice.OncePlayChara は単なるラッパー。
パッチ対象として Utils.Voice.OncePlayChara を hook しても良いが、
**Manager.Voice.Play(Loader) を hook する方が IsPlay 互換性の観点で優れる**（§24 参照）。

---

## 21. Manager.Voice 解析結果（確認済み）

デコンパイル済みソース: `../_decomp/Manager.Voice.decompiled.cs`（477行）

### _transTable の構造

```
Dictionary<int, Transform> _transTable   (private static)
  key   = キャラ番号（personality number）
  value = 再生管理用 Transform（子 GameObject = 再生中の AudioSource）
```

Initialize() で VoiceInfo の全 personality 番号 → Transform を登録する。
AudioSource は `_transTable[no]` の **子 GameObject** として生成される。

### Manager.Voice.Play(Loader) の実装フロー

```
Play(Loader loader):
  1. _transTable.TryGetValue(loader.no, out Transform vt) → なければ null return
  2. new AssetBundleData(loader.bundle, loader.asset).GetAsset<AudioClip>() → null なら return
  3. AudioSource src = Create(vt)   ← settingObjects[0] を vt の子として Instantiate
  4. src.clip = asset
  5. Play_Standby(src, loader)      ← private static
  6. return src
```

**パッチポイント**: ステップ 2 の AssetBundleData.GetAsset を差し替える。
Prefix で loader.bundle が "CUSTOM_" で始まる場合はカスタム実装に切り替え。

### Manager.Voice.IsPlay(Transform voiceTrans) の実装

```
foreach Transform value in _transTable.Values:
  for i in 0..value.childCount:
    child = value.GetChild(i)
    if child.GetComponent<Voice_Component>().voiceTrans == voiceTrans:
      return true
return false
```

**結論**: IsPlay は _transTable 値の全子を走査して Voice_Component.voiceTrans を比較する。
カスタム AudioSource も Manager.Voice.Create(vt) で生成し、Voice_Component.Bind(loader) を呼べば
**IsPlay と完全に互換する**。

### Voice_Component（確認済み）

```
Voice_Component : MonoBehaviour
  bundle     : string (private set)
  asset      : string (private set)
  voiceTrans : Transform (private set)
  Bind(Voice.Loader loader): bundle/asset/voiceTrans を loader から設定
```

IsPlay チェックが Voice_Component.voiceTrans を使うため、Bind 呼び出しが必須。

### Play_Standby (private static) の動作概要

| ステップ | 内容 |
|---------|------|
| 1 | src.name = clip.name |
| 2 | Sound.ClipAutoRelease(src) — clip 参照カウント登録 + OnDestroy 解放 |
| 3 | Sound.AudioSettingData(src, settingNo) — settingNo=-1 なら null → no-op |
| 4 | settingNo<0 なら spatialBlend = voiceTrans!=null ? 1 : 0 |
| 5 | src.pitch = loader.pitch |
| 6 | UpdateAsObservable で毎フレーム src.volume = Voice.GetVolume(no) |
| 7 | voiceTrans!=null なら毎フレーム src の position/rotation を voiceTrans に同期 |
| 8 | src.GetOrAddComponent<Voice_Component>().Bind(loader) |
| 9 | clip.loadState == Loaded になった次フレームで Play（or PlayFade） |
| 10 | loop=false なら PlayEndDestroy で終了時 GameObject 自動破棄 |

**リフレクションでの呼び出し**:
```csharp
static MethodInfo _playStandby = AccessTools.Method(
    typeof(Manager.Voice), "Play_Standby",
    new[] { typeof(AudioSource), typeof(Manager.Voice.Loader) });

_playStandby.Invoke(null, new object[] { audioSource, loader });
```

---

## 22. Manager.Sound 解析結果（確認済み）

デコンパイル済みソース: `../_decomp/Manager.Sound.decompiled.cs`（613行）

### ClipAutoRelease の動作

```
ClipAutoRelease(AudioSource src):
  clip = src.clip
  Register(clip)   // _useAudioClipList.Add(clip)
  src.OnDestroyAsObservable().Subscribe(() => Remove(clip))
```

### Register / Remove の実装

```
Register(AudioClip clip): _useAudioClipList.Add(clip)

Remove(AudioClip clip):
  _useAudioClipList から 1 件削除
  残件数 == 0 なら Resources.UnloadAsset(clip)
```

**カスタム WAV への影響**:
AudioClip.Create() + SetData() で生成したクリップは Resources システム外。
Resources.UnloadAsset(clip) は**ドキュメント上 no-op**（AssetBundle/Resources 以外には作用しない）。
→ ClipAutoRelease はカスタム WAV に対して**安全に呼び出し可能**。
クリップは参照消滅時に GC 回収される。

### PlayEndDestroy の動作

```
PlayEndDestroy(AudioSource src, float fadeTime):
  UpdateAsObservable.TakeWhile(_ => src.isPlaying).Subscribe(
    onNext     = no-op,
    onCompleted = () => {
      if src == null: return
      FadePlayer fp = src.GetComponent<FadePlayer>()
      if fp != null: fp.Stop(fadeTime)
      else: Destroy(src.gameObject)
    })
```

**カスタム WAV への影響**: isPlaying は標準 Unity AudioSource プロパティ。
カスタム WAV でも再生終了時に正しく GameObject が破棄される。**完全互換**。

### AudioDataLoadState の注意

Play_Standby は clip.loadState == Loaded をポーリングして Play する。
AudioClip.Create(stream=false) + SetData() の場合、**loadState は即座に Loaded**。
→ 次の Update フレーム（約 1 フレーム、16ms 程度）で Play が呼ばれる。実用上問題なし。

---

## 23. 確認済み vs 未確認（最終版）

§19 の「未確認」リストはすべて解決済み。

### 追加で確認済みになった項目

| 項目 | 根拠 |
|------|------|
| Utils.Voice.Setting 全フィールド | Illusion.Game.Utils デコンパイル |
| Utils.Voice.OncePlayChara シグネチャ | 同上 |
| Manager.Voice.IsPlay(Transform) の仕組み | Manager.Voice.decompiled.cs |
| Manager.Voice._transTable 構造 | 同上 |
| Manager.Voice.Play_Standby の全動作 | 同上 |
| Voice_Component.Bind の動作 | 同上 |
| Sound.ClipAutoRelease の安全性（custom WAV） | Manager.Sound.decompiled.cs |
| Sound.PlayEndDestroy の動作 | 同上 |
| Resources.UnloadAsset は custom WAV に no-op | Unity ドキュメント + 実装照合 |

### 残り未確認（影響度低・回避策あり）

| 項目 | 影響度 | 回避策 |
|------|--------|--------|
| Sound.FadePlayer.Play/Stop の実装 | 低 | 初期実装は fadeTime=0 固定 |
| SoundSettingData.Param の具体値 | 低 | settingNo=-1 で bypass |
| VoiceInfo.Param の全フィールド型 | 中 | 既存 dicVoiceIntos の Param を複製すれば型一致 |

---

## 24. 最終確認済み Hook 戦略

### 選択した方式: Manager.Voice.Play(Loader) Prefix

**理由**:
Utils.Voice.OncePlayChara(Setting) を hook した場合、独自 AudioSource は
Manager.Voice.Create(vt) を呼ばないため _transTable[no] の子にならない
→ IsPlay(voiceTrans) が false を返す → ゲームの重複再生防止ロジックが壊れる。
Manager.Voice.Play(Loader) を hook すれば、Create(vt) を自分で呼ぶことで
AudioSource が正しく _transTable[no] の子になり IsPlay と完全互換。

### 実装フロー（確定版）

```
HVoiceCtrl.VoiceProc()
  → Utils.Voice.OncePlayChara(Setting)
    → Manager.Voice.OncePlayChara(Loader)
      → Stop(no, voiceTrans)    ← 既存ボイス停止
      → Play(Loader)            ← ★ここに Prefix パッチを入れる

[Prefix] Play(Loader loader):
  if NOT loader.bundle.StartsWith("CUSTOM_"): return true  // 通常処理

  1. _transTable から Transform を取得
     Transform vt = (AccessTools.Property(typeof(Manager.Voice),"_transTable")
                      .GetValue(null) as Dictionary<int,Transform>)[loader.no]

  2. WAV ファイルパスを決定
     loader.asset = "c13/sonyu/2/h_so_13_02_007" 形式
     string wavPath = Path.Combine(pluginVoiceDir, loader.asset + ".wav")

  3. ディスクから WAV を読み込み AudioClip を生成
     AudioClip clip = WavLoader.Load(wavPath)
     if clip == null: return false  // ファイルなし → 無音（エラーにしない）

  4. AudioSource を _transTable[no] の子として生成
     AudioSource src = Manager.Voice.Create(vt)
     src.clip = clip

  5. Play_Standby でフル初期化
     _playStandbyMethod.Invoke(null, new object[]{ src, loader })
     ← 音量・位置同期・IsPlay追跡・自動破棄 を全て担う

  6. __result = src; return false   ← 元の Play をスキップ
```

### カスタム Loader のフィールド値

| フィールド | 値 | 説明 |
|-----------|-----|------|
| bundle | "CUSTOM_VOICE" | マーカー文字列（実在しない AssetBundle 名） |
| asset | "c13/sonyu/2/h_so_13_02_007" | プラグイン相対パス（.wav 省略） |
| no | 13 | キャラ番号 |
| pitch | 1f | 通常速度 |
| voiceTrans | heroine の口元 Transform | 3D 音源・IsPlay 追跡に使用 |
| settingNo | -1 | SoundSettingData 不使用（デフォルト適用） |
| fadeTime | 0f | フェードなし（初期実装） |

### WAV ファイルの格納構造（確定）

```
BepInEx/plugins/HVoiceAdder/
  voice/
    c00/
      aibu/
        0/ *.wav   (state 0 = 控えめ)
        1/ *.wav
        2/ *.wav
        3/ *.wav   (state 3 = 絶頂)
      houshi/
        0..3/ *.wav
      sonyu/
        0..3/ *.wav
      sonyu_3p/    (mode 8 = H3PSonyu)
        0..3/ *.wav
      houshi_3p/   (mode 7 = H3PHoushi)
        0..3/ *.wav
    c13/
      sonyu/
        2/ h_so_13_02_007.wav
        ...
```

mode 番号と mode 名の対応（最終版）:

| mode 番号 | mode 名 | lstProc index | 備考 |
|-----------|---------|---------------|------|
| 0 | aibu_start | - | 開始時 |
| 1 | aibu | 0 (HAibu) | |
| 2 | houshi | 1 (HHoushi) | |
| 3 | sonyu | 2 (HSonyu) | |
| 4 | masturbation | 3 | |
| 5 | peeping | 4 | |
| 6 | lesbian | 5 | |
| 7 | houshi_3p | 6 (H3PHoushi) | VoiceAllData.mode=2 共用 |
| 8 | sonyu_3p | 7 (H3PSonyu) | VoiceAllData.mode=3 共用 |
| 9 | (H3PDark Houshi) | 8 | dicVoicePtn に key 9 なし → **未対応確定** |
| 10 | (H3PDark Sonyu) | 9 | dicVoicePtn に key 10 なし → **未対応確定** |

---

## 参照ファイル一覧（最終版）

| ファイル | 内容 |
|----------|------|
| ../_decomp/HVoiceCtrl.decompiled.cs | H ボイス中心ロジック全体 |
| ../_decomp/HSceneProc.latest.cs | lstProc, lstHProc 定義 |
| ../_decomp/_tmp_HSonyu.cs | Sonyu 系 playVoices キー表 |
| ../_decomp/_tmp_HAibu.cs | Aibu 系 playVoices キー表 |
| ../_decomp/_tmp_HHoushi.cs | Houshi 系 playVoices キー表 |
| ../_decomp/_tmp_H3PSonyu.cs | 3P Sonyu 系 playVoices キー表 |
| ../_decomp/_tmp_H3PDarkSonyu.cs | 3PDark Sonyu（mode=10 → 未対応確定） |
| ../_decomp/_tmp_H3PDarkHoushi.cs | 3PDark Houshi（mode=9 → 未対応確定） |
| ../_decomp/Manager.Voice.decompiled.cs | Voice 再生システム全体（477行） |
| ../_decomp/Manager.Sound.decompiled.cs | Sound システム（ClipAutoRelease 等）（613行） |

## 注意

- **_transTableの初期化タイミング** — `Manager.Voice.Initialize()` で VoiceInfo ScriptableObject から読み込まれる。パーソナリティ番号がテーブルにないとボイス再生不可
- **Play_StandbyはPrivate** — `AccessTools.Method` で取得してリフレクション呼び出し。音量・空間音響・自動破棄・フェード・口パク連動を全て担当する重要メソッド
