# GUI実装ガイド（IMGUI）

BepInExプラグインのGUIはUnityのIMGUI（`OnGUI`方式）を使う。

## 基本構造

```csharp
private const int WindowId = 0x46415343;  // ユニークな整数（他と被らないようにする）
private Rect _windowRect = new Rect(40f, 40f, 460f, 300f);  // 初期位置・サイズ
private bool _uiVisible;
private ConfigEntry<KeyboardShortcut> _cfgToggleUiKey;

private void Awake()
{
    _cfgToggleUiKey = Config.Bind("Input", "ToggleUiKey",
        new KeyboardShortcut(KeyCode.F9), "UI表示切替キー");
}

private void Update()
{
    if (_cfgToggleUiKey.Value.IsDown())
        _uiVisible = !_uiVisible;
}

private void OnGUI()
{
    if (!_uiVisible) return;
    _windowRect = GUI.Window(WindowId, _windowRect, DrawWindow, PluginName);
}

private void DrawWindow(int id)
{
    GUILayout.BeginVertical();

    GUILayout.Label("ラベルテキスト");

    if (GUILayout.Button("ボタン", GUILayout.Height(34f)))
    {
        // ボタン処理
    }

    GUILayout.EndVertical();
    GUI.DragWindow(new Rect(0f, 0f, 10000f, 22f));  // 必ず末尾に呼ぶ
}
```

## ルール

- UIは起動時に非表示にする（`_uiVisible = false` がデフォルト）
- キーで表示切替する設計にする

## ポイント

- `WindowId` は他プラグインと被らないようにユニークな値にする
- `GUI.DragWindow()` は `DrawWindow` の**末尾**に呼ぶ（順序が重要）
- キーバインドは `Config.Bind` で設定化する（ハードコードしない）
- `GUILayout.Height()` でボタンの高さを指定できる
