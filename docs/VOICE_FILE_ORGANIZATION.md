# KKS 音声ファイル整理ガイド

## 概要

KKSの音声ファイルをCSVデータに基づいて、セリフをファイル名にして階層的に整理するシステム。

## 音声ファイルの命名規則

### 元のファイル名形式

```
h_{voice_type}_{character_id}_{category}_{sequential_id}.wav
```

**例:**
```
h_ai_13_00_000.wav
h_so_13_03_025.wav
h_ka3p_13_01_010.wav
```

### 構成要素

| 要素 | 説明 | 例 |
|------|------|-----|
| `h_` | Hシーン音声の接頭辞 | 固定 |
| `voice_type` | ボイスタイプ（2-4文字） | ai, so, ka3p |
| `character_id` | キャラクターID（2桁） | 13 |
| `category` | 興奮度カテゴリ（2桁） | 00-03 |
| `sequential_id` | 連番（3桁） | 000-999 |

## ボイスタイプ分類（10種）

| コード | 日本語名 | 説明 |
|--------|---------|------|
| `ai` | AI_喘ぎ | 喘ぎ声 |
| `fe` | FE_前戯 | 前戯シーン |
| `hh` | HH_奉仕 | 奉仕行為 |
| `hh3p` | HH3P_3P奉仕 | 3P奉仕シーン |
| `ka` | KA_愛撫 | 愛撫シーン |
| `ka3p` | KA3P_3P愛撫 | 3P愛撫シーン |
| `ko` | KO_行為中 | 行為中の掛け合い |
| `on` | ON_オナニー | オナニーシーン |
| `so` | SO_挿入 | 挿入シーン |
| `so3p` | SO3P_3P挿入 | 3P挿入シーン |

## 興奮度カテゴリ分類（4段階）

| コード | 日本語名 | 説明 |
|--------|---------|------|
| `00` | 00_通常 | 通常状態 |
| `01` | 01_興奮 | やや興奮 |
| `02` | 02_高揚 | かなり興奮 |
| `03` | 03_絶頂 | 絶頂状態 |

## CSV形式

### 構造

```csv
filename.wav|character_code|language|dialogue_text
```

**例:**
```csv
h_ai_13_00_000.wav|c13|JP|も～、恥ずかしいじゃん……？
h_ai_13_00_001.wav|c13|JP|ん……んん……
h_so_13_03_025.wav|c13|JP|あっ、あっ、イクッ！
```

### フィールド

| 位置 | 名前 | 説明 |
|------|------|------|
| 0 | filename | 元のファイル名 |
| 1 | character_code | キャラクターコード（c13など） |
| 2 | language | 言語コード（JP/EN） |
| 3 | dialogue | セリフテキスト |

## 整理スクリプト

### 実装

`F:/kks/work/rename_voice_detailed.py`

### 処理フロー

```python
1. CSV読み込み（UTF-8 BOM付き）
   ├─ ファイル名 → セリフ のマッピング作成
   └─ 2,735エントリ

2. 音声ファイル走査
   ├─ ソースディレクトリから *.wav を取得
   └─ 3,059ファイル

3. ファイルごとに処理
   ├─ CSVからセリフ取得
   ├─ ファイル名からボイスタイプ/カテゴリ抽出
   ├─ セリフをファイル名として無害化
   │   ├─ 禁止文字 (\/:*?"<>|) → _
   │   ├─ 空白 → _
   │   └─ 長さ制限（200文字）
   ├─ 出力フォルダ作成
   │   └─ {OUTPUT}/{ボイスタイプ}/{カテゴリ}/
   ├─ 重複名には連番付与
   └─ コピー実行

4. 統計出力
```

### 処理結果（c13の場合）

```
成功: 2,554
CSVに無し: 503
パターン不一致: 2
```

### 出力ディレクトリ構造

```
AudioClip_renamed/
├── AI_喘ぎ/
│   ├── 00_通常/
│   │   ├── も～、恥ずかしいじゃん……？.wav
│   │   ├── ん……んん…….wav
│   │   └── ...
│   ├── 01_興奮/
│   ├── 02_高揚/
│   └── 03_絶頂/
├── FE_前戯/
│   └── ...
├── SO_挿入/
│   ├── 00_通常/
│   ├── 01_興奮/
│   ├── 02_高揚/
│   └── 03_絶頂/
│       ├── あっ、あっ、イクッ！.wav
│       └── ...
└── ...
```

## ファイル名の無害化

### 処理内容

```python
def sanitize_filename(text):
    # Windowsで使えない文字を置換
    invalid_chars = r'[\\/:*?"<>|]'
    text = re.sub(invalid_chars, '_', text)

    # 空白をアンダースコアに
    text = text.replace(' ', '_')

    # 長すぎる場合は切り詰め
    if len(text) > 200:
        text = text[:200]

    return text
```

### 変換例

| 元のセリフ | ファイル名 |
|-----------|-----------|
| `も～、恥ずかしいじゃん……？` | `も～、恥ずかしいじゃん……？.wav` |
| `あっ、あっ、イクッ！` | `あっ、あっ、イクッ！.wav` |
| `ん♥ んん♥♥` | `ん♥_んん♥♥.wav` |

**注意:** 日本語文字は問題ないが、記号類は置換される。

## 重複ファイル名の処理

### 問題

異なる音声ファイルでも、セリフが同じ場合がある：
```
h_ai_13_00_005.wav → "ん……"
h_ai_13_00_012.wav → "ん……"
h_ai_13_01_003.wav → "ん……"  （カテゴリ違い）
```

### 解決策

同じフォルダ内で重複する場合、連番を付ける：

```python
counter = 1
while os.path.exists(output_path):
    new_filename = f"{safe_text}_{counter}.wav"
    output_path = os.path.join(output_folder, new_filename)
    counter += 1
```

**結果:**
```
00_通常/
  ├── ん…….wav
  ├── ん……_1.wav
  └── ん……_2.wav
01_興奮/
  └── ん…….wav  （別カテゴリなので番号なし）
```

## CSV読み込みの注意点

### BOM付きUTF-8

```python
with open(csv_path, 'r', encoding='utf-8-sig') as f:
```

`utf-8-sig`を指定することで、BOM（Byte Order Mark）を自動で除去。

### 区切り文字

```python
parts = line.split('|')
```

CSVだが区切り文字はパイプ（`|`）。カンマ（`,`）ではない。

### エンコーディングエラー対策

```python
line = line.strip()
if not line:
    continue
```

空行は無視する。

## 統計情報の活用

### ボイスタイプ別統計

```
AI_喘ぎ: 1,234ファイル
  00_通常: 308ファイル
  01_興奮: 312ファイル
  02_高揚: 305ファイル
  03_絶頂: 309ファイル

SO_挿入: 678ファイル
  00_通常: 169ファイル
  01_興奮: 170ファイル
  02_高揚: 169ファイル
  03_絶頂: 170ファイル

...
```

これにより：
- どのボイスタイプが多いか把握
- カテゴリごとのバランス確認
- 欠損データの検出

## トラブルシューティング

### CSVに存在しないファイル

**原因:**
- CSVが古い（新しいパッチで追加された音声）
- キャラクターIDが違う（c13用のCSVでc14のファイルを処理）

**対処:**
- 正しいCSVを使用
- スキップされたファイルをログで確認

### パターン不一致

**原因:**
- ファイル名が規則から外れている
- 正規表現パターンが不十分

**パターン:**
```python
r'h_([a-z0-9]+)_\d+_(\d+)_\d+\.wav'
```

このパターンに合わないファイルはスキップされる。

### 文字化け

**原因:**
- CSVのエンコーディングが違う
- `utf-8-sig`ではなく`shift-jis`等

**対処:**
```python
# 自動検出を試す
import chardet
with open(csv_path, 'rb') as f:
    result = chardet.detect(f.read())
    encoding = result['encoding']
```

## 使用例

### 基本的な使い方

```python
# 設定変更
CSV_PATH = r"F:\kks\voice_extract\voice_csv\c13.csv"
SOURCE_DIR = r"C:\Users\youzo\Desktop\新しいフォルダー (8)\AudioClip"
OUTPUT_DIR = r"C:\Users\youzo\Desktop\新しいフォルダー (8)\AudioClip_renamed"

# 実行
python rename_voice_detailed.py
```

### 複数キャラクター処理

```bash
# c13
python rename_voice_detailed.py

# c14用に設定変更
# CSV_PATH = r"F:\kks\voice_extract\voice_csv\c14.csv"
python rename_voice_detailed.py
```

### 統計のみ取得

```python
# コピーせずに統計だけ出す
# shutil.copy2(source_file, output_path)
# ↑この行をコメントアウト
```

## 応用

### 1. プラグインへの組み込み

整理後の音声ファイルを、StudioVoicePluginのような外部音声再生プラグインで使用。

**構造的利点:**
- フォルダ階層がゲージ色と対応
- セリフが見えるので選択しやすい

### 2. 音声データベース作成

ファイル名（セリフ）を検索可能にする：

```python
# セリフ → ファイルパス のデータベース
{
    "あっ、あっ、イクッ！": [
        "AI_喘ぎ/03_絶頂/あっ、あっ、イクッ！.wav",
        "SO_挿入/03_絶頂/あっ、あっ、イクッ！.wav"
    ]
}
```

### 3. 字幕システム

セリフがファイル名なので、再生時に字幕として表示可能。

## 関連ファイル

- **スクリプト本体**: `F:/kks/work/rename_voice_detailed.py`
- **簡易版**: `F:/kks/work/rename_voice.py` （カテゴリのみ分類）
- **CSVサンプル**: `F:/kks/voice_extract/voice_csv/c13.csv`
- **開発ガイド**: `F:/kks/CLAUDE.md`
