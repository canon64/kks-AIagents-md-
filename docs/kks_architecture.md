# KKS内部アーキテクチャ

## データリスト構造（共通パターン）

KKSの多くの機能（ボイス、アニメーション、アイテム等）は同じパターンで管理される。

```
Studio.Info に辞書がある
  dic{Feature}LoadInfo[group][category][no] = LoadCommonInfo
  dic{Feature}GroupCategory[group] = GroupInfo

GroupInfo:
  .name: string (グループ表示名)
  .dicCategory: Dictionary<int, string> (カテゴリID→表示名)

LoadCommonInfo (FileInfo を継承):
  .name: string (表示名)
  .bundlePath: string (AssetBundleパス)
  .fileName: string (バンドル内アセット名)
  .manifest: string
```

データはExcel形式のTextAsset（`*.unity3d` AssetBundle内）から読み込まれる。

## UI構造（共通パターン）

スタジオの選択UIは3段階リスト:
- `{Feature}GroupList` → グループ選択
- `{Feature}CategoryList` → カテゴリ選択
- `{Feature}List` → 個別アイテム選択

**辞書にエントリを追加すれば対応するUIリストに自動で反映される。UI側のパッチは不要。**

## シングルトンアクセス

```csharp
Singleton<Info>.Instance           // Studio.Info
Singleton<Studio.Studio>.Instance  // Studio本体
SingletonInitializer<Voice>        // Manager.Voice (initialized プロパティあり)
```

## アセット読み込み

```csharp
// ゲーム標準（AssetBundleから）
new AssetBundleData(bundlePath, assetName).GetAsset<T>();

// カスタム（ディスクから）→ Harmonyで差し替える
```
