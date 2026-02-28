# 本編Hボイス 条件分岐ディープダイブ

- 調査日: 2026-02-25
- 対象DB: `../kks_voices.db`
- 対象コード: `../_decomp/HVoiceCtrl.decompiled.cs`

---

## 1. まず結論

`cond_desc` は空ではない。  
`idx=0..14` で定義があり、`voices.cond_0..cond_14` は **全行0/1のみ** で埋まっている（NULLなし）。

ただし `cond` 条件が立っている行は少ない。

- `voices` 総行数: `348,823`
- `condが1つでも立つ行`: `3,995`
- `cond全ゼロ行`: `344,828`

つまり `cond_desc` は「全ボイス共通の主分類」ではなく、**一部の条件付きボイスの追加ゲート**。

---

## 2. cond_desc と cond_i の実体

`cond_desc`:

| idx | desc |
|---|---|
| 0 | 初挿入(コーカン+アナル未) |
| 1 | 股間処女かつ未挿入 |
| 2 | アナル処女かつ未挿入 |
| 3 | 挿入OK状態 |
| 4 | 股間絶頂0回 |
| 5 | アナル絶頂0回 |
| 6 | 股間絶頂0+アナル絶頂1 |
| 7 | アナル外出し0+アナル中出し1 |
| 8 | 股間絶頂1回 |
| 9 | アナル絶頂1回 |
| 10 | 股間挿入中(股間専用) |
| 11 | コンドームなし |
| 12 | アナル挿入中(アナル専用) |
| 13 | 弱点3番 |
| 14 | 彼女フラグ |

`voices.cond_0..14` は全列で `0/1` のみ（`other=0`, `null=0`）。

---

## 3. 実行時の判定順（重要）

`HVoiceCtrl.VoiceProc()` → `GetPLayNumVoiceList()` の流れで、実際は2段ゲート。

1. **パターン条件**（`VoicePatternData` 側）
- `IsPlayVoicePtn(...)`
- `lstAnimID` と `lstConditions` で候補パターンを絞る
- 判定関数: `IsPtnConditions*`（Start/Aibu/Houshi/Sonyu/...）

2. **ボイス条件**（`VoiceAllData` 側）
- `VoiceInfo.isPlayConditions`（bool[15]）
- 判定関数: `IsVoiceConditions*`
- ここで `cond_0..14` 相当が評価される

3. その後
- `isPlay` / `isOneShot` / `notOverwrite` / `priority` / ランダム順で最終1本決定

---

## 4. cond_0..14 はどこで効くか

`IsVoiceConditionsSonyu` / `IsVoiceConditions3PSonyu` が主。

### Sonyu系（mode=3, 8）での実コード条件

- `cond_0`: `sonyuKokanPlay + sonyuAnalPlay == 0`
- `cond_1`: `isVirgin && !isInsertKokanVoiceCondition`
- `cond_2`: `isAnalVirgin && !isInsertAnalVoiceCondition`
- `cond_3`: `isInsertOK[_main]`
- `cond_4`: `sonyuOrg == 0`
- `cond_5`: `sonyuAnalOrg == 0`
- `cond_6`: `sonyuOrg == 0 && sonyuAnalOrg == 1`
- `cond_7`: `sonyuAnalOutside == 0 && sonyuAnalInside == 1`
- `cond_8`: `sonyuOrg == 1`
- `cond_9`: `sonyuAnalOrg == 1`
- `cond_10`: `!isAnalPlay && 生理Sではない`
- `cond_11`: `!isCondom`
- `cond_12`: `isAnalPlay`
- `cond_13`: `weakPoint == 3`
- `cond_14`: `isGirlfriend`

### 他モード

- `Houshi` / `3PHoushi` は `cond_0` のみ使用
- `Aibu`, `Masturbation`, `Peeping`, `Lesbian` は `IsVoiceConditions*` 側で実質未使用（空switch）

---

## 5. DB実測: condが立つボイスの分布

### mode_name別（condが1の行）

- ほぼ `sonyu`
- 一部 `aibu_start` / `houshi` にも存在（主に `cond_0..4`）

### file_type別（condが1の行）

- 主体: `sonyu`
- 少量: `aibu_touch`, `foreplay`, `houshi`

### cond使用率の高いもの

- `cond_10` (股間挿入中): 800
- `cond_3` (挿入OK): 754
- `cond_0` (初挿入系): 713
- `cond_14` (彼女): 365
- `cond_13` (弱点3): 2（極少）

---

## 6. 「厳密分類」する時の正しい軸

分類は以下の優先順で切ると、再生決定ロジックに近くなる。

1. `mode` / `mode_name`（行為モード）
2. `patternKey` 相当（`playVoices % 100` に対応するパターン）  
   - 現DBには直接列なし。`VoicePatternData` 側解析が必要
3. `state`（歓喜度4段階）
4. `cond_0..14`（VoiceInfoの追加条件）
5. `file_type` / `insert_type` / `houshi_type` / `aibu_type` / `situation_type` / `condom_type`
6. `voice_id` / `filename` / `wav_path`

ポイント:

- `file_type` 等は分類に便利だが、再生決定の最終ゲートは `pattern条件 + cond条件 + state`。
- したがって「厳密分類」を目指すなら、`VoicePatternData.lstConditions` 解析が必須。

---

## 7. 追加で見えたこと

### filenameトークンと file_type は実質1対1

- `so -> sonyu`
- `hh -> houshi`
- `ai -> aibu`
- `so3p -> sonyu_3p`
- `ka3p -> aibu_3p`
- `hh3p -> houshi_3p`
- `fe -> foreplay`
- `ka -> aibu_touch`
- `on -> masturbation`
- `ko -> act_common`

### `voices.mode` は 0..6 まで

- DB上は `aibu_start, aibu, houshi, sonyu, masturbation, peeping, lesbian`
- 3P/3PDark専用mode（7/8/9/10）は、このDBには直接出ていない  
  （コード上は `IsPtnConditions3P*`, `IsVoiceConditions3P*` が存在）

---

## 8. 未解決（次調査）

1. `VoicePatternData` をDB化して `patternKey -> lstConditions` を完全展開
2. 3P/3PDarkの `mode` 実データがどこにあるか（別抽出か共用か）確定
3. `situation_type` 列を、コードの `IsPtnConditions*` 条件IDと逆対応付け

---

## 参照

- `../_decomp/HVoiceCtrl.decompiled.cs`
- `../kks_voices.db`
- `../MAIN_H_VOICE_BINDING_ANALYSIS.md`
