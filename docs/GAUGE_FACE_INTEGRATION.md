# StudioGaugeBar 表情制御仕様

## 概要

**StudioGaugeBarがゲージ値に応じて自動的に表情を変化させます。**

- **ゲージ色変更時**: 表情パターン自動変更
- **常時**: 頬の赤みがゲージ値に連動

## 変更内容（2026-02-12）

**表情制御をStudioGaugeBarに統合しました。**

- **以前**: StudioFacePresetToolがStudioGaugeBarを監視して表情変更
- **現在**: StudioGaugeBarが独立して表情制御

### StudioGaugeBar（統合後）
- ゲージ値管理
- 頬の赤み（Cheek）をゲージ値（0-100%）に応じてリアルタイム制御
- **表情パターン（Eyebrow/Eye/Mouth）を色変化時に自動変更**
- **涙レベル（Tears）も連動**

### StudioFacePresetTool（汎用ツール化）
- 表情プリセットの保存/読込（手動操作用）
- ゲージ監視機能を削除（ゲージバーに依存しない）

## ゲージ色の定義

| ゲージ値 | 色 | ColorIndex |
|---------|-----|-----------|
| 0-25% | Blue | 0 |
| 25-50% | Green | 1 |
| 50-75% | Yellow | 2 |
| 75-100% | Red | 3 |

## 設定方法

### 表情パターンはハードコード

表情パターンは `StudioGaugeBar/Plugin.cs` の `GetFaceParamsForColor()` メソッドにハードコードされています。

| ゲージ色 | Eyebrow | Eye | Mouth | EyeMin | MouthMin | Tears |
|---------|---------|-----|-------|--------|----------|-------|
| Blue (0-25%) | 0 | 10 | 0 | 0.9 | 0.2 | 0 |
| Green (25-50%) | 1 | 11 | 1 | 0.8 | 0.3 | 0 |
| Yellow (50-75%) | 2 | 12 | 2 | 0.7 | 0.3 | 2 |
| Red (75-100%) | 3 | 13 | 3 | 0.6 | 0.5 | 3 |
| Black (100%+) | 4 | 25 | 43 | 0.5 | 0.7 | 3 |

**カスタマイズ方法:**
- `Plugin.cs` の `GetFaceParamsForColor()` を編集
- ビルド＆デプロイ

### 動作確認

1. StudioでキャラとIKガイドを登録
2. StudioGaugeBarで女キャラを登録
3. IKガイドを動かしてゲージを上昇
4. ゲージ色が変わると自動的に表情が切り替わる
5. 頬の赤みはゲージ値に応じて常時変化

## 動作仕様

### 表情パターン適用
- **トリガー**: ゲージ色が変わったとき（即座に切り替え）
- **対象**: 登録済みの女キャラ全員
- **適用内容**:
  - Eyebrow（眉パターン）
  - Eye（目パターン）
  - Mouth（口パターン）
  - EyeMin/EyeMax（目の開き範囲）
  - MouthMin（口の開き最小値）
  - **MouthMax（常に100%固定）**
  - **mouthCtrl.FixedRate（口の実際の開度）**
  - Tears（涙レベル）
  - ~~Cheek（頬の赤み）~~ ← 別途リアルタイム制御

### 頬の赤み制御
- **更新頻度**: 毎フレーム
- **対象**: 登録済みの女キャラ全員
- **計算式**: `Cheek = ゲージ値 × 0.5 (0.0 ~ 0.5)`
- **最大50%に制限**

## トラブルシューティング

### 表情が変わらない
- `BepInEx/plugins/StudioGaugeBar/StudioGaugeBar.log.txt` を確認
- `[Face] Color changed to X` のログが出ているか確認
- StudioGaugeBarで女キャラが登録されているか確認

### 頬の赤みが動かない
- `BepInEx/plugins/StudioGaugeBar/StudioGaugeBar.log.txt` を確認
- 女キャラ登録が完了しているか確認

### 表情パターンをカスタマイズしたい
- `F:/kks/work/StudioGaugeBar/Plugin.cs` の `GetFaceParamsForColor()` メソッドを編集
- `dotnet build -c Release` でビルド
- `StudioGaugeBar.dll` を `BepInEx/plugins/StudioGaugeBar/` にコピー

## 実装ファイル

- **StudioGaugeBar**: `F:/kks/work/StudioGaugeBar/Plugin.cs`
  - UpdateFaceForGirls() - 色変化検出＆表情適用
  - GetFaceParamsForColor() - 色ごとの表情パラメータ定義
  - ApplyFaceParams() - 表情適用ロジック
  - UpdateCheekForGirls() - 頬の赤み制御
- **StudioFacePresetTool**: `F:/kks/work/StudioFacePresetTool/Plugin.cs`
  - 表情プリセット管理（手動操作用）
  - ゲージ監視機能は削除済み
