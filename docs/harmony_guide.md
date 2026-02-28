# Harmonyパッチ ガイド

## 基本パターン

```csharp
// Prefix: 元のメソッドの前に実行。return false で元をスキップ。
[HarmonyPrefix]
[HarmonyPatch(typeof(TargetClass), "MethodName", new Type[] { typeof(ArgType) })]
static bool Prefix(ArgType arg, ref ReturnType __result)
{
    if (IsCustom(arg)) {
        __result = CustomImplementation(arg);
        return false; // 元のメソッドをスキップ
    }
    return true; // 通常処理
}
```

## privateメンバーへのアクセス

```csharp
// フィールド
var field = AccessTools.Field(typeof(TargetClass), "_privateField");
var value = field.GetValue(instance);

// プロパティ
var prop = AccessTools.Property(typeof(TargetClass), "_privateProperty");
var value = prop.GetValue(null); // staticの場合

// メソッド
var method = AccessTools.Method(typeof(TargetClass), "PrivateMethod");
method.Invoke(null, new object[] { arg1, arg2 }); // staticの場合
```

## KKSでの典型的な使い方

1. `Studio.Info` の辞書に直接エントリを追加する（ゲームUIに自動で反映される）
2. 読み込み処理（AssetBundle→AudioClip等）をPrefixで横取りする
3. カスタムエントリかどうかは `bundlePath` にマーカー文字列を入れて判別する

## 注意

- `__result` で戻り値を書き換えるときは `ref` が必要
- `return false` すると元のメソッドは実行されない
- パッチの適用は `Harmony.CreateAndPatchAll(typeof(MyPlugin), GUID)` で一括適用
