# 設定ファイル実装ガイド（JSON）

## 基本方針

- 全パラメータ・閾値・定数はJSON設定ファイルで変更可能にする（ハードコード禁止）
- 設定ファイルはプラグインDLLと同じフォルダに配置する
- ファイルがない場合はデフォルト値で自動生成する
- `Awake()` で読み込む

## 実装パターン

```csharp
[Serializable]
private class Settings
{
    public float Speed = 1.0f;
    public float Threshold = 0.5f;
    public string BasePath = "";
}

private Settings _settings;

private void Awake()
{
    LoadSettings();
}

private void LoadSettings()
{
    try
    {
        string pluginDir = Path.GetDirectoryName(Info.Location);
        string settingsPath = Path.Combine(pluginDir, "Settings.json");

        if (!File.Exists(settingsPath))
        {
            _settings = new Settings();
            SaveSettings();
            return;
        }

        string json = File.ReadAllText(settingsPath, Encoding.UTF8);
        _settings = UnityEngine.JsonUtility.FromJson<Settings>(json);
        Logger.LogInfo("Settings loaded");
    }
    catch (Exception ex)
    {
        Logger.LogError($"Failed to load settings: {ex.Message}");
        _settings = new Settings();
    }
}

private void SaveSettings()
{
    try
    {
        string pluginDir = Path.GetDirectoryName(Info.Location);
        string settingsPath = Path.Combine(pluginDir, "Settings.json");
        string json = UnityEngine.JsonUtility.ToJson(_settings, true);
        File.WriteAllText(settingsPath, json, Encoding.UTF8);
    }
    catch (Exception ex)
    {
        Logger.LogError($"Failed to save settings: {ex.Message}");
    }
}
```

## JSON化すべき定数の例

- ゲージの上昇速度・減衰速度・閾値
- ボイス再生の条件・クールダウン時間
- UI位置・サイズ・色
- 色変化の閾値（Blue→Green等）
- 頬の赤み倍率
- 外部ファイルパス（表情JSON、ボイスフォルダ等）

単純なON/OFFはBepInEx Config (.cfg)でもよいが、複数パラメータはJSON推奨。

## 注意

- `using System.IO;` と `using System.Text;` を追加すること
- `UnityEngine.JsonUtility` を使用する（`Newtonsoft.Json` は不要）
- パス文字列はJSON内で `\\` でエスケープが必要
- 空文字列 `""` はデフォルト値として使用可能
