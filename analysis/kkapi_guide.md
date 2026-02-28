# KKSAPI 完全調査ノート

- 調査日: 2026-02-28
- バージョン: 1.42.2
- GUID: `marco.kkapi`
- DLL: `F:/kks/BepInEx/plugins/KKSAPI.dll`

---

## 概要

KKSの公式MOD APIフレームワーク。イベント駆動型のコントローラー登録システム。

用途:
- スタジオのカスタムUI追加
- キャラクターへのデータ付与と永続化
- キャラメイカーのUI拡張
- 本編イベント（Hシーン、セーブ、時間帯等）のフック

---

## KoikatuAPI（ルートクラス）

```csharp
KoikatuAPI.GetCurrentGameMode()  // → GameMode (Studio/Maker/MainGame/Unknown)
KoikatuAPI.GetGameVersion()      // → Version
KoikatuAPI.IsVR()                // → bool
KoikatuAPI.Logger                // ManualLogSource
KoikatuAPI.IsQuitting            // bool
```

---

## StudioAPI

```csharp
StudioAPI.InsideStudio   // bool: CharaStudioプロセスか
StudioAPI.StudioLoaded   // bool: スタジオが完全に読み込まれたか

// イベント
StudioAPI.StudioLoadedChanged += (s, e) => { /* ここからStudio.Info等にアクセス可能 */ };

// キャラ・オブジェクト取得
StudioAPI.GetSelectedCharacters()              // → IEnumerable<OCIChar>
StudioAPI.GetSelectedObjects()                 // → IEnumerable<ObjectCtrlInfo>
StudioAPI.GetSelectedControllers<T>()          // → IEnumerable<T>

// カスタムUI
StudioAPI.GetOrCreateCurrentStateCategory("カテゴリ名")  // → CurrentStateCategory
```

### CurrentStateCategory に追加できるコントロール

- `CurrentStateCategoryToggle`
- `CurrentStateCategorySlider`
- `CurrentStateCategoryDropdown`
- `CurrentStateCategoryColorPicker`
- `CurrentStateCategorySwitch`

---

## CharacterApi（キャラクター拡張）

```csharp
// 全キャラクター（スタジオ・メイカー・本編）に自動でコントローラーを付与
CharacterApi.RegisterExtraBehaviour<T>(string extendedDataId)
    // T: CharaCustomFunctionController の派生クラス（引数なしコンストラクタ必須）
```

### CharaCustomFunctionController（基底クラス）

```csharp
// プロパティ
ChaControl ChaControl           // このコントローラーが付いているキャラ
ChaFileControl ChaFileControl
string ExtendedDataId
bool Started

// 拡張データの読み書き
GetExtendedData(bool getFromLoadedChara) → PluginData
SetExtendedData(PluginData data)
    // 本編では Heroine/Player コピーへ自動同期

GetCoordinateExtendedData(ChaFileCoordinate) → PluginData
SetCoordinateExtendedData(ChaFileCoordinate, PluginData)

// 実装必須
protected abstract void OnCardBeingSaved(GameMode currentGameMode)
    // ここで SetExtendedData() を呼ぶ

// 任意オーバーライド
protected virtual void OnReload(GameMode mode, bool maintainState)
    // キャラ状態変化時。Awake/Start の代わりに使う
    // maintainState=true のときは状態保持（リロード不要）

protected virtual void OnCoordinateBeingLoaded(GameMode mode, CoordinateEventArgs args)
protected virtual void OnCharacterReloading(GameMode mode)
```

---

## StudioSaveLoadApi（シーンセーブ/ロード）

```csharp
// シーンに付くコントローラーを登録（インスタンスは1つ）
StudioSaveLoadApi.RegisterExtraBehaviour<T>(string extendedDataId)
    // T: SceneCustomFunctionController の派生クラス

// イベント
StudioSaveLoadApi.SceneLoadComplete    // ロード/インポート/クリア後
StudioSaveLoadApi.ObjectAdded
StudioSaveLoadApi.ObjectDeleted
StudioSaveLoadApi.ObjectsCopied
StudioSaveLoadApi.ObjectsSelected
StudioSaveLoadApi.ObjectVisibilityToggled

// インポート時のID対応表
StudioSaveLoadApi.ImportDictionary
    // Key = 現在シーンのID, Value = セーブファイル上の元ID
```

### SceneCustomFunctionController（基底クラス）

```csharp
// 実装必須
protected internal abstract void OnSceneLoad(
    SceneOperationKind operation,           // Load / Import / Clear
    ReadOnlyDictionary<int, ObjectCtrlInfo> loadedItems)  // 元IDでキー
protected internal abstract void OnSceneSave()

// 任意オーバーライド
protected internal virtual void OnObjectsCopied(ReadOnlyDictionary<int, ObjectCtrlInfo>)
protected internal virtual void OnObjectDeleted(ObjectCtrlInfo)
protected internal virtual void OnObjectVisibilityToggled(ObjectCtrlInfo, bool visible)
protected internal virtual void OnObjectsSelected(List<ObjectCtrlInfo>)

// 拡張データ
GetExtendedData() → PluginData
SetExtendedData(PluginData data)
```

---

## MakerAPI（キャラメイカー）

```csharp
MakerAPI.InsideMaker        // bool
MakerAPI.InsideAndLoaded    // bool
MakerAPI.GetMakerSex()      // int (0=male, 1=female)
MakerAPI.GetCharacterControl() // ChaControl（プレビューキャラ）
MakerAPI.GetCurrentCoordinateType() // CoordinateType

// コントロール追加
MakerAPI.AddControl<T>(T control) → T
MakerAPI.AddSidebarControl<T>(T control) → T         // 右サイドバー
MakerAPI.AddAccessoryWindowControl<T>(T control) → T
MakerAPI.AddEditableAccessoryWindowControl<T, TVal>() // スロット別値
```

### イベントの実行順序（重要）

1. `RegisterCustomSubCategories` — サブカテゴリ追加（ここだけ）
2. `MakerStartedLoading` — 早期（一部コンポーネント未初期化）
3. `MakerBaseLoaded` — コントロール追加のベストタイミング
4. `MakerFinishedLoading` — ユーザー操作開始後（重い処理は避ける）
5. `ReloadCustomInterface` — キャラ/コーデ読み込み後にUI更新

### コントロール種類

`MakerButton`, `MakerToggle`, `MakerSlider`, `MakerDropdown`,
`MakerText`, `MakerTextbox`, `MakerColor`, `MakerImage`,
`MakerSeparator`, `MakerRadioButtons`,
`MakerLoadToggle`, `MakerCoordinateLoadToggle`,
`SidebarToggle`, `SidebarSeparator`

---

## GameAPI（本編・Hシーン）

```csharp
GameAPI.InsideHScene   // bool
GameAPI.GameBeingSaved // bool

// コントローラー登録（本編専用。スタジオでは動作しない）
GameAPI.RegisterExtraBehaviour<T>(string extendedDataId)

// シーン取得
GameAPI.GetActionControl()
GameAPI.GetADVScene()
GameAPI.GetActionScene()
GameAPI.GetTalkScene()
GameAPI.GetCurrentHeroine() // Heroine（フォーカス中のキャラ）

// カスタムアクションポイント追加
GameAPI.AddActionIcon(mapNo, position, icon, color, text, onOpen, ...) → IDisposable
GameAPI.AddTouchIcon(icon, onCreated, row, order) → IDisposable
```

### GameAPI イベント

```csharp
GameAPI.StartH      // HシーンStart
GameAPI.EndH        // HシーンEnd
GameAPI.GameLoad    // セーブロード後
GameAPI.GameSave    // セーブ前
GameAPI.DayChange   // 曜日変化
GameAPI.PeriodChange // 時間帯変化（朝/昼/夕/夜）
GameAPI.NewGame     // 新規ゲーム開始
```

### GameCustomFunctionController（基底クラス）

```csharp
protected virtual void OnGameLoad(GameSaveLoadEventArgs)
protected virtual void OnGameSave(GameSaveLoadEventArgs)
protected virtual void OnStartH(MonoBehaviour baseLoader, HFlag hFlag, bool freeH)
protected virtual void OnEndH(MonoBehaviour baseLoader, HFlag hFlag, bool freeH)
protected virtual void OnDayChange(Week day)
protected virtual void OnPeriodChange(Type period)
protected virtual void OnNewGame()
```

---

## SceneApi（ゲーム問わず共通）

```csharp
SceneApi.GetAddSceneName()       // オーバーレイシーン名（設定/ダイアログ等）
SceneApi.GetLoadSceneName()      // メインシーン名（メイカー/H/ADV等）
SceneApi.GetIsNowLoadingFade()   // ローディング+フェード中
SceneApi.GetIsNowLoading()       // ローディング中のみ
SceneApi.GetIsFadeNow()          // フェード中のみ
SceneApi.GetIsOverlap()          // ダイアログ/オーバーレイ表示中
```

---

## StudioObjectExtensions（ヘルパー）

```csharp
obj.GetObjectCtrlInfo()                         // ObjectInfo → ObjectCtrlInfo
obj.GetSceneId()                                // 現在のシーンID
chaControl.GetOCIChar()                         // ChaControl → OCIChar
ociChar.GetChaControl()                         // OCIChar → ChaControl
tno.TryGetObjectCtrlInfo(out ObjectCtrlInfo)    // TreeNodeObject → ObjectCtrlInfo
```

---

## PluginData（拡張データの型）

```csharp
var data = new PluginData();
data.data["key"] = value;  // object型で何でも入れられる

// 読み込み
var data = GetExtendedData();
if (data?.data.TryGetValue("key", out var val) == true)
{
    // val を使う
}
```

---

## 落とし穴

| やってはいけない | 理由 | 正しい方法 |
|---|---|---|
| `Awake()` でスタジオデータにアクセス | 未初期化 | `StudioLoadedChanged` イベントを待つ |
| `CharaCustomFunctionController` で `Awake/Start` | 動作しない | `OnReload()` を使う |
| `RegisterCustomSubCategories` でコントロール追加 | 早すぎる | `MakerBaseLoaded` で追加 |
| `InsideStudio` だけチェック | ロード前の可能性 | `InsideStudio && StudioLoaded` |
| `GameAPI` をスタジオで使う | 動作しない | 本編専用と割り切る |

---

## 実装パターン集

### スタジオプラグインの基本形

```csharp
[BepInProcess("CharaStudio")]
public class MyPlugin : BaseUnityPlugin
{
    private void Awake()
    {
        StudioAPI.StudioLoadedChanged += OnStudioLoaded;
    }

    private void OnStudioLoaded(object sender, EventArgs e)
    {
        var category = StudioAPI.GetOrCreateCurrentStateCategory("MyPlugin");
        var toggle = category.AddControl(new CurrentStateCategoryToggle("ON/OFF", 2, null));
        toggle.Value.Subscribe(v => { /* 処理 */ });
    }
}
```

### キャラデータ永続化の基本形

```csharp
public class MyController : CharaCustomFunctionController
{
    private float _myValue;

    protected override void OnReload(GameMode mode, bool maintainState)
    {
        if (maintainState) return;
        var data = GetExtendedData();
        _myValue = data?.data.TryGetValue("val", out var v) == true ? (float)v : 0f;
    }

    protected override void OnCardBeingSaved(GameMode mode)
    {
        var data = new PluginData();
        data.data["val"] = _myValue;
        SetExtendedData(data);
    }
}
// 登録
CharacterApi.RegisterExtraBehaviour<MyController>("com.author.myplugin");
```

### シーンデータ永続化の基本形

```csharp
public class MySceneController : SceneCustomFunctionController
{
    protected internal override void OnSceneLoad(
        SceneOperationKind op, ReadOnlyDictionary<int, ObjectCtrlInfo> items)
    {
        var data = GetExtendedData();
        // items のキーは元ID（セーブファイル上のID）
    }

    protected internal override void OnSceneSave()
    {
        var data = new PluginData();
        SetExtendedData(data);
    }
}
// 登録
StudioSaveLoadApi.RegisterExtraBehaviour<MySceneController>("com.author.myplugin");
```
