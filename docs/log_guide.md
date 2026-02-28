# ログ実装ガイド

## 基本方針

BepInExの`LogOutput.log`は全プラグインの出力が混在して見づらい。開発中は専用ログファイルをプラグインフォルダ内に出力する。

## ログレベル

| レベル | 用途 |
|--------|------|
| `Logger.LogDebug` | 開発中の詳細な値確認。リリース時は削除 |
| `Logger.LogInfo` | 正常な動作の記録（起動、初期化完了等） |
| `Logger.LogWarning` | 想定外だが続行可能な状態 |
| `Logger.LogError` | エラー。処理が失敗した場合 |

BepInExの`Logger.*`はLogOutput.logに書かれる。細かいデバッグログは`LogToFile()`で専用ファイルに書く。

## 実装パターン

```csharp
[BepInPlugin(GUID, PluginName, Version)]
public class MyPlugin : BaseUnityPlugin
{
    private static string _logFilePath;

    private void Awake()
    {
        _logFilePath = Path.Combine(Path.GetDirectoryName(Info.Location), "MyPlugin.log");
        LogToFile("=== Plugin Started ===");
    }

    private static void LogToFile(string message)
    {
        try
        {
            var line = $"[{DateTime.Now:HH:mm:ss.fff}] {message}";
            File.AppendAllText(_logFilePath, line + Environment.NewLine);
        }
        catch { }
    }
}
```

## ログファイルの場所

```
BepInEx/plugins/MyPlugin/MyPlugin.log
```

## 注意

- `using System.IO;` を追加すること
- `LogToFile()` は全ての重要な処理に使用する
- リアルタイム監視: `tail -f BepInEx/plugins/MyPlugin/MyPlugin.log`
- リリース時はファイル出力を無効化するか削減する
