# ショートカットキー一覧

## スタジオ（CharaStudio）

| キー | 用途 | プラグイン |
|------|------|-----------|
| Ctrl+J | UI表示切替 | StudioGaugeBar |
| Ctrl+G | UI表示切替 | StudioFacePresetTool |
| Ctrl+B | UI表示切替 | BlinkTest |
| F4 | 接続 | HandyPlugin |
| F7 | キャリブレーション | HandyPlugin |
| F8 | モーション停止 / 女性キャラ登録 | HandyPlugin / StudioHipsTrackerProbe ⚠️競合 |
| F9 | モーション再開 / 女性キャラ削除 | HandyPlugin / StudioHipsTrackerProbe ⚠️競合 |
| F10 | UI表示切替 / 詳細ログ切替 | HandyPlugin / StudioHipsTrackerProbe ⚠️競合 |
| F11 | UI表示切替 | FaceCatalogTool |

## 本編（KoikatsuSunshine）

| キー | 用途 | プラグイン |
|------|------|-----------|
| F8 | UI表示切替 | MainGirlHipsIkHijack |
| F9 | UI表示切替 | FemaleAnimSpeedCutProbe |
| F10 | UI表示切替 | HVoiceAdder |

## 注意

- HandyPluginとStudioHipsTrackerProbeがF8/F9/F10で競合している（同時使用不可）
- 新規プラグインのキーは上記と被らないように選ぶこと
- `Input.GetKeyDown(KeyCode.X)` のハードコードより `Config.Bind<KeyboardShortcut>` で設定化を推奨
