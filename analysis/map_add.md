# 本編マップ追加調査

- 調査日: 2026-02-28
- 対象: `KoikatsuSunshine` 本編アクションシーン

---

## 結論

マップデータは `map/list/mapinfo/` 以下の全AssetBundleを自動列挙して読み込む。
スタジオのアイテム追加と同じパターン。カスタムABを置けば自動認識される。

---

## データの流れ

1. `BaseMap.Start()` → `LoadMapInfo()` を呼ぶ
2. `LoadMapInfo()` が `map/list/mapinfo/` 以下を `CommonLib.GetAssetBundleNameListFromPath` で列挙
3. 各ABから `MapInfo` (ScriptableObject) を取り出して `infoDic[No] = Param` に格納
4. サムネイルは `map/list/mapthumbnailinfo/` から同様に自動読み込み
5. `MapSelectMenuScene.Create()` が `infoDic.Values` を `Sort` 順に並べて表示

---

## MapInfo.Param フィールド

| フィールド | 型 | 意味 |
|---|---|---|
| `No` | int | マップID。重複禁止 |
| `MapName` | string | 内部名 |
| `DisplayName` | string | UI表示名 |
| `Sort` | int | UI表示順 |
| `AssetBundleName` | string | マップSceneのABパス |
| `AssetName` | string | Scene名 |
| `isGate` | bool | マップ移動選択肢に出るか |
| `isFreeH` | bool | フリーH選択肢に出るか |
| `isH` | bool | Hシーン可能か |
| `isOutdoors` | bool | 屋外か |
| `isSky` | bool | 空あり |
| `ThumbnailMorningID` 等 | int | サムネイルID（-1=なし） |

---

## マップ追加の2アプローチ

### A. 純正（カスタムAssetBundle）

- Unityプロジェクトでマップシーンを作り、ABとしてビルド
- `map/list/mapinfo/` にカスタムABを追加
- `MapInfo` ScriptableObjectにParamを設定
- 難易度高（Unity環境必須）

### B. プラグイン（既存マップの再利用）

- `BaseMap.LoadMapInfo` Postfix で戻り値の辞書にエントリを追加
- `AssetBundleName` / `AssetName` は既存マップのものを流用、`No` だけ変える
- `isH` / `isFreeH` などのフラグを自由に設定できる
- 難易度低

Postfixの書き方（概要）:
```csharp
[HarmonyPostfix]
[HarmonyPatch(typeof(BaseMap), "LoadMapInfo")]
static void LoadMapInfo_Postfix(ref Dictionary<int, MapInfo.Param> __result)
{
    // 既存マップを複製して別IDで登録
    var original = __result[1]; // 既存マップNo=1を流用
    var custom = new MapInfo.Param {
        No = 900,
        MapName = "custom_map",
        DisplayName = "カスタムマップ",
        AssetBundleName = original.AssetBundleName,
        AssetName = original.AssetName,
        isGate = true,
        isFreeH = true,
        isH = true,
        Sort = 900
    };
    __result[900] = custom;
}
```

---

## マップ選択UIの表示条件

`MapSelectMenuScene` の Switch文でNoによる特殊条件がある:

- `No=9` (シークレットビーチ): `isSecretBeachOpen && isSecretBeachInvasion` が必要、夜のみ
- `No=36` (スイートルーム): `isSuiteRoomOpen && isSuiteRoomInvasion` が必要
- その他: 条件なしで表示

カスタムマップはNo=9/36を避ければ条件なしで表示される。

---

## 参照デコンパイルファイル

- `../_decomp/_tmp_BaseMap.cs` — LoadMapInfo/LoadMapThumbnailInfo実装
- `../_decomp/_tmp_MapInfo.cs` — MapInfo ScriptableObject定義
- `../_decomp/_tmp_MapSelectMenuScene.cs` — マップ選択UI実装
