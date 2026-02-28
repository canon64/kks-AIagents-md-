# abdata ファイルマップ

KKS (コイカツサンシャイン) のアセットバンドルデータ全体構造。
調査日: 2026-02-24

> **Unity バージョン:** 2019.4.9f1
> **フォーマット:** UnityFS AssetBundle (.unity3d 拡張子、または拡張子なしバイナリ)

---

## トップレベル構造

```
F:/kks/abdata/
│
├── [ディレクトリ]
│   ├── action/          学校生活パート（アクションモード）
│   ├── adv/             ADVシナリオ・演出
│   ├── camera/          カメラプリセット
│   ├── chara/           キャラクターアセット（衣装・髪・体型等）
│   ├── communication/   コミュニケーション（会話ヒット判定等）
│   ├── custom/          メイクルームUI・カスタマイズデータ
│   ├── esthetic/        エステティックシーン
│   ├── etcetra/         その他リスト
│   ├── h/               Hシーンアニメーション・リスト
│   ├── list/            キャラカスタムリスト・名前等
│   ├── map/             マップデータ
│   ├── network/         ネットワーク関連
│   ├── shortcut/        ショートカット定義
│   ├── sound/           サウンド全般（BGM・SE・ボイス）
│   ├── studio/          キャラスタジオ専用アセット
│   ├── tutorial/        チュートリアル
│   ├── twoshot/         ツーショットイベント
│   ├── ui/              UI共通素材
│   └── vr/              VRモード専用データ
│
├── [DLC AssetBundle ファイル（拡張子なし・UnityFS形式）]
│   ├── add01 ～ add92   本編DLC追加データ
│   └── studio00 ～ studio110  スタジオDLC追加データ
```

---

## ディレクトリ詳細

### action/ — 学校生活パート（アクションモード）

自由行動パートで使うデータ全般。マップ・ショップ・会話・SE等。

```
action/
├── actioncontrol/
│   ├── 00.unity3d       デフォルトアクション制御設定
│   ├── 30.unity3d       DLC30 用
│   └── 70.unity3d       DLC70 用
├── animator/
│   ├── 00.unity3d       アニメーターコントローラー
│   └── 70.unity3d
├── chara.unity3d         キャラ関連共通アセット
├── fixchara/             固定キャラ（NPC等）データ
├── list/                 ← アクションモード用リスト群
│   ├── chara/            キャラリスト（00, 70）
│   ├── clubinfo/         部活情報
│   ├── monologue/        モノローグ台詞
│   ├── motionvoice/      動作中ボイス定義（00, 31, 70）
│   ├── playwater/        水遊びデータ
│   ├── prayinfo/         祈り（？）情報
│   ├── shopiconinfo/     ショップアイコン定義
│   ├── shopinfo/         ショップ品目定義
│   ├── sound/
│   │   ├── bgm/          アクションパートBGM定義
│   │   └── se/           アクションパートSE定義
│   ├── stepup/           ステップアップ（好感度段階）
│   ├── talklookbody/     会話中ボディ視線方向
│   ├── talklookneck/     会話中首視線方向
│   ├── topic/            会話トピック
│   └── wherelive/        居住地情報
├── map/
│   ├── minimap/          ミニマップ画像
│   ├── navmesh/          ナビメッシュ（AI経路）
│   └── semesh/           SEトリガーメッシュ
├── mapitem/              マップ上配置アイテム
├── menu/                 アクションメニューUI
├── playeraction/         プレイヤーアクション定義
├── playwater/            水遊びプレハブ等
└── sprite/shop/          ショップUI用スプライト
```

---

### adv/ — ADVシナリオ・演出

アドベンチャーパートのシナリオ・CG・モーション。

```
adv/
├── bg/                   ADV背景画像
├── directing/            演出データ（フェード・カメラ効果等）
├── eventcg/              イベントCG
├── motion/
│   ├── controller/adv/   ADV用アニメーターコントローラー
│   └── iklist/           IKリスト
└── scenario/             ← キャラ別シナリオデータ
    ├── c-13/             特殊キャラ（先生等）
    ├── c00/ ～ c43/      メインキャラ44人分
    ├── common/           共通シナリオ（全員共通イベント等）
    ├── common_param/     共通パラメータ定義
    ├── op/               オープニング
    └── suiteroomevent/   スイートルームイベント
```

---

### camera/ — カメラプリセット

```
camera/
├── action.unity3d        アクションパート用カメラ設定
└── twoshoot.unity3d      ツーショット撮影用カメラ設定
```

---

### chara/ — キャラクターアセット

体・衣装・髪・アクセサリのメッシュ・テクスチャ・マテリアル。
命名規則でカテゴリ判別可能。

```
chara/
├── ao_*.unity3d          アクセサリ（ao_arm_00.unity3d, ao_neck_88.unity3d 等）
├── bo_hair_*.unity3d     ボーン付き髪型
├── co_bot_*.unity3d      下半身衣装パーツ
├── co_top_*.unity3d      上半身衣装パーツ
├── cpo_jacket_*.unity3d  ジャケット系衣装
├── cpo_sailor_*.unity3d  セーラー系衣装
├── face_preset_00.unity3d 顔プリセット
├── mm_base.unity3d       男性ベースボディ
├── mt_*.unity3d          マテリアル・テクスチャ
├── oo_base.unity3d       女性ベースボディ
├── oo_hand.unity3d       手メッシュ
└── thumb/                サムネイル画像
```

**命名プレフィックス早見表:**

| プレフィックス | 意味 |
|--------------|------|
| `ao_` | Accessory（アクセサリ）|
| `bo_hair_` | Bone Hair（ボーン付き髪）|
| `co_` | Clothes（衣装）|
| `cpo_` | Clothes Part Other（その他衣装パーツ）|
| `mm_` | Male Model（男性ベース）|
| `mt_` | Material / Texture |
| `oo_` | Other Object（女性ベース・体部品）|

---

### communication/ — コミュニケーション

会話・好感度・ヒット判定データ。DLCごとに番号付き。

```
communication/
├── hit_00.unity3d        ヒット判定設定（本体）
├── info_00.unity3d       会話情報（本体）
├── info_31.unity3d       DLC31 用
├── info_50.unity3d       DLC50 用
├── info_60.unity3d       DLC60 用
├── info_63.unity3d       DLC63 用
├── info_70.unity3d       DLC70 用
└── info_110.unity3d      DLC110 用
```

---

### custom/ — メイクルーム・カスタマイズ

キャラクリエイト・メイクルームで使うUIデータ。

```
custom/
├── bgmlist/              BGMリスト（00, 70）
│   ├── 00.unity3d
│   └── 70.unity3d
├── colorsample.unity3d   カラーサンプル
├── cos_def_f_00.unity3d  デフォルト衣装（女）
├── cos_wed_*.unity3d     ウェディング衣装系
├── custom_etc.unity3d    カスタムその他
├── custompose.unity3d    カスタムポーズ
├── custompose_70.unity3d DLC70 ポーズ
├── customscenelist/      カスタムシーンリスト
│   ├── 00.unity3d, 01.unity3d, 70.unity3d
│   └── addpose00.unity3d
├── presets_f_00.unity3d  プリセット（女）
├── presets_m_00.unity3d  プリセット（男）
├── samplevoice_00.unity3d サンプルボイス
└── samplevoice_70.unity3d DLC70 サンプルボイス
```

---

### esthetic/ — エステティックシーン

エステ施術シーンで使うデータ。

```
esthetic/
├── animator/             エステ用アニメーター
├── prefabs/              エステ用プレハブ
└── list/                 ← エステリスト群
    ├── camera/           カメラ設定リスト
    ├── ctrlinfo/         制御情報
    ├── dynamicbonectrl/  動的ボーン制御
    ├── endadv/           終了ADV定義
    ├── eyeneck/          目・首の動き
    ├── hitcollision/     当たり判定
    ├── ik/               IK設定
    ├── reaction/         リアクション定義
    ├── se/               SE定義
    └── voice/            ボイス定義
```

---

### etcetra/ — その他リスト

```
etcetra/
└── list/                 分類外のリストデータ
```

---

### h/ — Hシーンデータ

Hシーンのアニメーション・リスト・スプライト。
`h/list/` の番号体系は DLC番号（00=本体, 70=DLC70等）と対応。

```
h/
├── anim/                 ← Hアニメーションデータ
│   ├── female/           女性モーション（00_01_00.unity3d 等 13ファイル）
│   ├── male/             男性モーション（7ファイル）
│   └── parts/            パーツ別モーション（01_00, 71_00）
├── common/               共通アセット（01, 71）
├── list/                 ← Hシーン定義リスト（重要）
│   ├── 00.unity3d        本体 Hシーン定義
│   ├── 01.unity3d        追加定義
│   ├── 30.unity3d        DLC30
│   ├── 31.unity3d        DLC31
│   ├── 60.unity3d        DLC60
│   ├── 70.unity3d        DLC70
│   ├── 71.unity3d        DLC71
│   ├── 81.unity3d        DLC81
│   └── 91.unity3d        DLC91
├── scene/                Hシーン選択UI（freehcharaselect.unity3d）
└── sprite/               Hシーン用スプライト
```

---

### list/ — キャラカスタムリスト

`Studio.Info` の `dicItemLoadInfo` 等が参照するリストデータ。
22ファイルは DLC番号に対応（00=本体, 01, 30, 35, 40...）。

```
list/
├── characustom/          キャラカスタムアイテムリスト（22ファイル）
│   ├── 00.unity3d ～ 90.unity3d  (本体+各DLC対応)
│   └── ... 計22ファイル
├── random_name/          ランダム名前リスト
│   └── random_name.unity3d
└── shapecorrect/         体型補正データ
    └── shapecorrect.unity3d
```

---

### map/ — マップデータ

各マップの配置情報・ナビゲーション・サウンドトリガー等。

```
map/
├── actionpoint/          アクションポイント（00/, 01/, 71/）
├── advpos/               ADVカメラ位置（00/, 01/）
├── areastatepoint/       エリア状態ポイント（00/）
├── common/               共通マップアセット
├── datepoint/            デートポイント（00/, 70/）
├── list/                 ← マップ定義リスト
│   ├── calcgateinfo/     ゲート計算情報
│   ├── mapinfo/          マップ情報定義
│   ├── mapthumbnailinfo/ マップサムネイル情報
│   └── navigationinfo/   ナビゲーション情報
├── materials/            マップマテリアル
├── object/               マップ上オブジェクト
├── placearea/            エリア定義
├── playeractionpoint/    プレイヤーアクションポイント
├── scene/                マップシーン（.unity3d）
├── sound/                マップBGM・SEトリガー
├── thumbnail/            マップサムネイル画像
└── waitpoint/            待機ポイント（00/, 70/, 71/）
```

---

### network/ — ネットワーク

オンライン機能用データ。

```
network/
├── entryhn.unity3d       Hシーンエントリー（ネットワーク）
├── networkcheck.unity3d  ネットワーク接続チェック
└── randomnetchara.unity3d ランダムネットキャラ
```

---

### shortcut/ — ショートカット

```
shortcut/
└── shortcut.unity3d      ショートカットキー定義
```

---

### sound/ — サウンドデータ全般

BGM・SE・キャラボイス（PCM）の全データ。

```
sound/
├── data/
│   ├── bgm/              BGMバンドル（26ファイル）
│   │   ├── kks_bgm_00.unity3d
│   │   └── ... （kks_bgm_XX.unity3d 形式）
│   ├── pcm/              ← キャラクター別ボイス（PCM・WAV）
│   │   ├── c-100/        特殊キャラ（adv/）
│   │   ├── c-13/         特殊キャラ（先生等）(adv/)
│   │   └── c00/ ～ c43/  メインキャラ44人
│   │       ├── adm/      雑多ボイス（日常会話等）
│   │       ├── adv/      ADVシナリオボイス
│   │       ├── esthetic/ エステボイス
│   │       ├── h/        Hシーンボイス（~153MB/キャラ）
│   │       └── namelist/ 名前呼びかけボイス
│   ├── se/               効果音
│   │   ├── esthetic/     エステSE
│   │   ├── h/            HシーンSE
│   │   └── map/env/      マップ環境SE
│   └── systemse/         システムSE
│       ├── brandcall/    ブランドコール（起動時）
│       └── titlecall/    タイトルコール
└── setting/              サウンド設定
    ├── sound3dsettingdata/  3Dサウンド設定
    └── soundsettingdata/    サウンド設定（00.unity3d）
```

**ボイスファイル命名規則（Hシーン）:**
`h_{type}_{char}_{level}_{seq}.wav`
→ 例: `h_so_13_03_025.wav` = 挿入(so), キャラ13, 絶頂(03), 25番目

| コード | シチュエーション |
|--------|----------------|
| `ai`   | 喘ぎ |
| `fe`   | 前戯 |
| `hh`   | 奉仕（手コキ・フェラ）|
| `hh3p` | 3P奉仕 |
| `ka`   | 愛撫 |
| `ka3p` | 3P愛撫 |
| `ko`   | 行為中掛け合い |
| `on`   | オナニー |
| `so`   | 挿入・ピストン |
| `so3p` | 3P挿入 |

興奮度: 00=控えめ, 01=通常, 02=興奮, 03=絶頂

---

### studio/ — スタジオアセット

キャラスタジオ専用データ。`Studio.Info` が参照するリスト・アイテム・BGM。

```
studio/
├── 00.unity3d ～ 90.unity3d   スタジオアイテム本体アセット（24ファイル・DLC対応）
├── anime/                      スタジオ用アニメーション
│   ├── 00.unity3d, 01.unity3d, 70.unity3d
├── base/                       スタジオベースアセット
│   └── 00.unity3d
├── filter/                     ポストエフェクトフィルター
├── info/                       ← スタジオアイテム定義リスト（重要）
│   ├── 00.unity3d ～ 110.unity3d  各DLC対応・25ファイル
│   └── （00,01,30,35,40,45,50,60-66,70-74,80,87,88,90,100,110）
├── map/                        スタジオ用マップ
│   ├── 00/                     本体マップ
│   ├── common/                 共通マップアセット
│   └── materials/              マップマテリアル
├── mat/                        マテリアル
│   ├── 00.unity3d, 01.unity3d
├── sky/                        スカイボックス
│   └── 01.unity3d
└── sound/                      スタジオ用サウンド
    ├── bgm/                    スタジオBGM（bgm_00 ～ bgm_09）
    └── song/                   スタジオ楽曲
```

**`studio/info/` との対応:**
`Studio.Info` の辞書 (`dicItemLoadInfo` 等) はこの `info/XX.unity3d` を読み込む。
`studio/XX.unity3d` (拡張子なし or .unity3d) が実アセット、`info/XX.unity3d` がメタ定義という対応。

---

### tutorial/ — チュートリアル

```
tutorial/
├── tutorial.unity3d        基本チュートリアル
├── tutorial_h.unity3d      Hチュートリアル
├── tutorialdialog.unity3d  チュートリアルダイアログ
└── tutorialprogress.unity3d チュートリアル進捗管理
```

---

### twoshot/ — ツーショットイベント

```
twoshot/
├── list/     ツーショットリスト定義
└── prefab/   ツーショット用プレハブ
```

---

### ui/ — UI共通素材

```
ui/
└── 00.unity3d    UI共通アセット（フォント・アイコン等）
```

---

### vr/ — VRモード

```
vr/
├── freehselect/  VR フリーHキャラ選択
└── h/            VR Hシーンデータ
```

---

## DLC AssetBundle ファイル（拡張子なし）

ルート直下にある拡張子なしファイルはすべて **UnityFS形式のAssetBundle**（Unity 2019.4.9f1）。
各ディレクトリ内の `XX.unity3d` に対応するDLCデータが格納されている。

### add** — 本編DLC追加データ

| ファイル | サイズ | 備考 |
|---------|-------|------|
| `add01` | ~3,650 B | DLC01 |
| `add02` | ~2,042 B | DLC02 |
| `add03` | ~1,933 B | DLC03 |
| `add04` | ~1,785 B | DLC04 |
| `add30` | ~6,596 B | DLC30 |
| `add31` | ~7,052 B | DLC31 |
| `add35` | ~5,556 B | DLC35 |
| `add40` | ~6,236 B | DLC40 |
| `add45` | ~（小）| DLC45 |
| `add50` | ~6,348 B | DLC50 |
| `add60` | ~5,316 B | DLC60 |
| `add63` | ~（小）| DLC63 |
| `add64` | ~（小）| DLC64 |
| `add70` | ~6,504 B | DLC70 |
| `add71`～`add75` | ~1,500-2,500 B | DLC71-75（小型）|
| `add80` | ~5,132 B | DLC80 |
| `add81` | ~（小）| DLC81 |
| `add88` | ~5,128 B | DLC88 |
| `add90` | ~5,012 B | DLC90 |
| `add91`～`add92` | ~（小）| DLC91-92 |
| `add100` | ~5,404 B | DLC100（大型）|
| `add110` | ~5,460 B | DLC110（大型）|

### studio** — スタジオDLC追加データ

| ファイル群 | サイズ | 備考 |
|-----------|-------|------|
| `studio00` | ~9,452 B | スタジオDLC00（最大）|
| `studio01` | ~7,024 B | スタジオDLC01 |
| `studio60`～`studio66` 等 | ~4,456 B | 多くが同一サイズ（共通テンプレート）|
| `studio70`～`studio74` | ~1,575-1,695 B | 小型 |
| `studio100`, `studio110` | ~4,456 B | 大型DLC |

**DLC番号体系の読み方:**

| 番号帯 | 意味 |
|-------|------|
| 00-04 | 本体・初期追加 |
| 30-35 | DLC第30章系 |
| 40-45 | DLC第40章系 |
| 50 | DLC第50章 |
| 60-66 | DLC第60章系 |
| 70-75 | DLC第70章系（衣装等細分化）|
| 80-88 | DLC第80章系 |
| 90-92 | DLC第90章系 |
| 100 | 大型DLC（追加キャラ・衣装一括）|
| 110 | 大型DLC（追加キャラ・衣装一括）|

---

## データアクセスパターン（MOD開発者向け）

### Studio.Info からデータを読む

スタジオのアイテムデータは `studio/info/XX.unity3d` から `Studio.Info` の辞書に読み込まれる:

```csharp
// アクセス例
Studio.Info info = Singleton<Studio.Info>.Instance;
// info.dicItemLoadInfo[group][category][no] → LoadCommonInfo
// info.dicItemGroupCategory[group]          → GroupInfo
```

### ボイスデータ

```
sound/data/pcm/c{XX}/h/ → WAVバンドル（AssetBundle内にAudioClip）
```

抽出済みWAV: `H:/kks_voice/c{XX}/h/01/` に格納済み。

### Hシーンリスト

```
h/list/00.unity3d → TextAsset (Excel/CSV形式) → Studio.Info.dicHListLoad 等
```

### キャラカスタムリスト

```
list/characustom/00.unity3d → TextAsset → Studio.Info.dicClothesLoadInfo 等
```

---

## 参考リンク

- ゲームデータ詳細: `F:/kks/GAME_DATA_LOCATIONS.md`（存在する場合）
- ボイス整理ガイド: `F:/kks/work/VOICE_FILE_ORGANIZATION.md`
- プラグイン開発ガイド: `F:/kks/CLAUDE.md`
- ボイスシステム解析: `F:/kks/test_plugin/StudioVoicePlugin/VOICE_SYSTEM_ANALYSIS.md`
