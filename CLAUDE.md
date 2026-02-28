# KKS BepInExプラグイン開発

## 質問対応ルール
- 質問には先に答える
- 明示的に依頼されるまでコードを書かない
- 実装を依頼された場合も、まず方針を提案してから実行する

## 作業場所
- 作業は `F:/kks/work/` の中で行う
- ルート (`F:/kks/`) にはファイル・フォルダを置かない
- プロジェクト固有の情報は各プラグイン・GUIフォルダ内の `CODEBASE_STATE.md` に書く

## 環境
- ゲーム: KoikatsuSunshine (Unity 2019, Mono/.NET 4.7.2)
- フレームワーク: BepInEx 5.x
- ビルド: `dotnet build` (net472)
- デコンパイラ: `ilspycmd`

## 鉄則
1. コードを書く前に `analysis/` の調査ノートを確認する。なければ `ilspycmd` でデコンパイルする。推測しない
2. `StudioAPI.StudioLoaded` を待ってからデータにアクセスする（Awake時は未初期化）
3. `[BepInProcess]` は `"CharaStudio"` か `"KoikatsuSunshine"` を正しく指定する
4. 専用ログファイルをプラグインフォルダ内に出力する（`Path.GetDirectoryName(Info.Location)`）
5. 全パラメータはJSON設定ファイルで変更可能にする（ハードコード禁止）
6. リソース・設定ファイルはプラグインフォルダ内に置く（絶対パスをコードに書かない）

## ビルド & デプロイ
```bash
dotnet build MyPlugin.csproj -c Release
cp bin/Release/net472/MyPlugin.dll "F:/kks/BepInEx/plugins/MyPlugin/"
```
- ビルド後は即デプロイする
- 意味のある変更のたびに git commit する
- コミットメッセージは短く日本語で書く
- `git add` は対象ファイルを必ず明示する（`git add -A` や `git add .` は使わない）

## ファイル編集ルール
- ファイルの読み書きは必ずUTF-8で統一する
- mdファイルに追記する前に「どの章のどこに追加する」と提案してから実行する
- mdファイルが100行を超えたら責務ごとにサブファイルへの分割を提案する
- スクリプトが500行を超えたら分割を提案する

## 指示ファイルマップ
| 作業内容 | 参照ファイル |
|----------|-------------|
| 新規プラグイン・アプリ作成時 | `analysis/NEW_PLUGIN.md` |
| アセットバンドル構造 | `docs/ABDATA_FILE_MAP.md` |
| ゲームデータ所在地 | `docs/GAME_DATA_LOCATIONS.md` |
| 音声ファイル整理 | `docs/VOICE_FILE_ORGANIZATION.md` |
| ゲージ×表情連携仕様 | `docs/GAUGE_FACE_INTEGRATION.md` |
| KKS内部アーキテクチャ | `docs/kks_architecture.md` |
| Harmonyパッチの書き方 | `docs/harmony_guide.md` |
| デコンパイル手順 | `docs/decompile_guide.md` |
| ログ実装 | `docs/log_guide.md` |
| 設定ファイル実装時は必ず確認 | `docs/settings_guide.md` |
| GUI実装 | `docs/gui_guide.md` |
| ショートカットキー一覧（競合確認） | `docs/keybindings.md` |
| 設計の教訓 | `反省.md` |
