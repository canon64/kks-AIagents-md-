# 作業開始時に読むファイル

新規プラグイン・アプリを作るときは以下を確認する。

- 既存調査の確認 → 下記ファイル一覧

---

# Analysis 消化進捗

## 消化の定義
- **未消化**: 調査ノートのまま。docsに相当する内容がない
- **消化中**: docsへの整理が進行中
- **消化済み**: 重要な知見がdocsに移済み。このファイルは削除可能

---

## ファイル一覧

| ファイル | 内容 | 状態 | 移行先 |
|---------|------|------|--------|
| `h_voice_system.md` | 本編HシーンのボイスシステムComplete解析。HVoiceCtrl/VoiceAllData/PlayVoiceの全フロー | 未消化 | - |
| `h_voice_conditions.md` | ボイス条件分岐（cond_0〜14）の詳細。各モードでの条件判定の実装 | 未消化 | - |
| `h_gauge_hijack.md` | 本編HシーンのゲージHFlag/HSprite/HSceneProcの構造と乗っ取りポイント調査 | 未消化 | - |
| `h_gauge_flow.md` | ゲージ変化のデータフロー解析。FemaleGaugeUp/MaleGaugeUpの経路 | 未消化 | - |
| `girl_hips_ik.md` | 女キャラ腰IK有効化の調査。FullBodyBipedIK/MotionIKへのアクセス方法 | 未消化 | - |
| `girl_hips_ik_procedure.md` | 腰IK+ゲージ乗っ取りDLL実装手順書（フェーズ1〜5）。他AI実装用の固定仕様も含む | 未消化 | - |
| `girl_hips_ik_progress.md` | 腰IK実装の進捗メモ | 未消化 | - |
| `female_anim_speed.md` | 女キャラ速度反映の切断調査。HActionBase.SetAnimatorFloatがキーポイント | 未消化 | - |
| `h_list_01_complete_dump.txt` | hリスト（h/list/01）の全データダンプ。生テキスト | 未消化 | - |
| `StudioCustomVoice_PlayPrefix.il.txt` | カスタムボイス再生のHarmonyパッチILコード | 未消化 | - |
| `StudioCustomVoice_RegisterCustomVoices.il.txt` | カスタムボイス登録処理のILコード | 未消化 | - |
| `StudioCustomVoice_RegisterCustomVoices.il2.txt` | 同上の別バージョン | 未消化 | - |

---

## プラグイン雛形

```csharp
using BepInEx;
using BepInEx.Logging;
using HarmonyLib;
using KKAPI.Studio;
using Studio;
using System;
using System.Collections;
using System.IO;
using UnityEngine;

namespace MyPlugin
{
    [BepInPlugin(GUID, PluginName, Version)]
    [BepInProcess("CharaStudio")]  // スタジオ専用。本編は "KoikatsuSunshine"
    public class MyPlugin : BaseUnityPlugin
    {
        public const string GUID = "com.author.myplugin";
        public const string PluginName = "My Plugin";
        public const string Version = "1.0.0";

        internal static new ManualLogSource Logger;
        private static string _logFilePath;

        private void Awake()
        {
            Logger = base.Logger;
            _logFilePath = Path.Combine(Path.GetDirectoryName(Info.Location), "MyPlugin.log");
            LogToFile("=== Plugin Started ===");
            Harmony.CreateAndPatchAll(typeof(MyPlugin), GUID);
            StartCoroutine(WaitForStudio());
        }

        private IEnumerator WaitForStudio()
        {
            while (!StudioAPI.StudioLoaded)
                yield return null;
            // ここでスタジオのデータにアクセスできる
        }

        private static void LogToFile(string message)
        {
            try
            {
                File.AppendAllText(_logFilePath, $"[{DateTime.Now:HH:mm:ss.fff}] {message}{Environment.NewLine}");
            }
            catch { }
        }
    }
}
```

```
BepInEx/plugins/MyPlugin/
├── MyPlugin.dll
├── MyPlugin.log      ← 開発時のみ
├── Settings.json     ← 設定ファイル
└── data/             ← リソース（必要に応じて）
```

---

## 注意

- **Voice.Playは同期メソッド** — UnityWebRequest等の非同期APIは使えない。WAVはバイト配列から `AudioClip.Create()` + `SetData()` で同期的に生成する
