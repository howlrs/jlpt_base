# 設計ドキュメント: JLPT問題データ重複対策

> 作成日: 2026-04-19
> 関連Issue: (未起票 — 本設計承認後に起票)
> スコープ: 既存本番DB重複の検出・削除 + パイプライン再発防止
> 非スコープ: 管理画面UI (別Issue)、意味的類似検出 (別Issue)

---

## 問題の背景

ユーザーから「問題・選択肢の組み合わせが重複して見える」と指摘。
本番 Firestore DB 全量 (14,857 parent docs / 21,809 sub_questions) を取得し分析した結果、以下を確認:

- 現 dedup 実装 (`jlpt-app-scripts/bin/2_duplicate.rs`) の dedup_key は `sentence + "||" + correct_value` であり、**選択肢セットを判定に使用していない**
- 結果として削減実績は 30件のみ
- 一方、**選択肢セット + 正解値 + level_id 正規化 + category=9除外** での判定では **547 件** の重複 sub_questions が削除対象として特定される (328 重複グループ)
- `level_name` が `"N5"` と `"n5"` の混在バグにより、同一問題が複数レベル表記で二重登録。小文字表記は合計 4,992件 (n1〜n5)
- `category_id=9` (文の組み立て/並び替え問題) は select_answer の value が数字 ("1","2","3","4") でテキスト情報を持たないため、dedup 対象外 (総 511件中 506件が category=9)
- `votes` 9件、`user_answers` 39件と極小のため、連鎖削除の影響は実質なし

## 目標

1. **既存データの重複掃除**: 選択肢セットが一致し、かつ正解値・レベルが同じ sub_question を1つに統合
2. **再発防止**: `2_duplicate.rs` の dedup_key を選択肢セット込みに改修
3. **データ整合性**: `level_name` のケース正規化
4. **安全性**: dry-run、バックアップ、事前レポート、件数検証を備えた one-shot スクリプト

## 非目標

- 管理画面での重複監視UI (別Issue化)
- 意味的類似 (Gemini Embedding) による類問検出 (別Issue化、将来)
- `category_id=9` の `('1','2','3','4')` 選択肢問題の個別検証 (別Issue化、データ品質調査)
- `category_id=7 "用法"` に紛れ込んだ `('1','2','3','4')` 選択肢5件の調査 (別Issue化)
- N5 向け生成プロンプト改善 (別Issue化、削除候補のうちN5が50%を占めることから示唆)

---

## 設計方針の判断根拠

### 方針1: dedup キーは `(level_id, sorted_normalized_option_values, normalized_answer_value)`

**理由** (Gemini pro レビュー + 実データ分析の合意):

- 選択肢セットのみでは「助詞用法の別問題」まで誤削除するリスクが高い (例: `{が,で,に,を}` の選択肢で、正解が「が」の問題と「を」の問題は別の知識点を問う)
- 選択肢+正解値が一致する問題は、文のバリエーションを変えただけの類問でありUX上の重複感の主因
- `level_id` は大前提として含める (レベル違いの問題は統合してはならない)

**正規化ルール**:
1. Unicode NFKC 正規化
2. 前後・中間空白 trim
3. 英数字半角統一
4. ひらがな/漢字の表記揺れは吸収しない (JLPTでは表記自体が問題要素のため)

### 方針2: 削除単位は sub_question 単位、parent は全sub消失時のみ削除

**理由**:

- 長文読解系の parent は複数 sub_question を持つ (平均 1.47、最大数十)
- parent 単位削除は「無実の sub_question」を巻き添えにするリスク
- パイプライン側の `2_duplicate.rs` は既に `kept_subs` ベクタで sub 単位処理しており整合性取れる

**実装**:
- Firestore の `sub_questions` 配列フィールドを更新 (部分更新でなく全体書き換え)
- sub が 0 件になった parent は `delete_question` と同等の処理で削除

### 方針3: tiebreaker は `createTime 古い方 > sentence長 > qid 辞書順`

**理由**:

- `Question` モデルに `created_at` フィールドは**存在しない**が、Firestore のシステムメタデータ `createTime` が全ドキュメントに存在
- `generated_by` は全 14,857件が同一値 (`gemini-3.1-flash-lite-preview`) で差別化不可
- `votes` は 9件、`user_answers` は 39件と極小 → ユーザー評価・学習履歴ベースの tiebreaker は実質機能しない
- 古い方を残すことで、将来的な学習履歴の孤立を最小化
- sentence 長は情報量の proxy
- qid 辞書順は決定論的 tiebreak (冪等性のため)

### 方針4: level_name ケース正規化を dedup 前に実施

**理由**:

- 同一 `level_id=5` で `"N5"` 1,463件 / `"n5"` 1,472件と二重化
- dedup キーに `level_id` を使うため影響は小さいが、表示・検索で一貫性欠如
- 追加検出: level_name 正規化込みで +52件検出可能

**実装**: 全問題の `level_name` を `to_uppercase()` で統一する one-shot 更新スクリプト

### 方針5: ガードレール

- `--dry-run` デフォルト、`--execute` で実行
- 実行前に Firestore Export で全DBバックアップ取得
- 削除対象の JSON レポートを出力 (ユーザーレビュー可能)
- 削除後の件数検証 (期待値との照合)
- WriteBatch で複数操作をアトミック化

---

## アーキテクチャ

### コンポーネント

```
┌────────────────────────────────────────────────────────────┐
│ jlpt-app-scripts/bin/99_report_duplicates.rs (新規)        │
│   - 全 Question を取得                                     │
│   - dedup キー生成 + グルーピング                          │
│   - tiebreaker で残す1件を決定                             │
│   - JSON レポート出力 (削除対象 sub の一覧)                │
└────────────────────────────────────────────────────────────┘
                          ↓ reports/duplicates.json
┌────────────────────────────────────────────────────────────┐
│ ユーザー手動レビュー (サンプル確認)                        │
└────────────────────────────────────────────────────────────┘
                          ↓
┌────────────────────────────────────────────────────────────┐
│ jlpt-app-scripts/bin/99_apply_dedup.rs (新規)              │
│   - レポートを入力として受け取る                           │
│   - --dry-run (デフォルト) / --execute                     │
│   - WriteBatch で sub_questions フィールド更新             │
│   - sub が 0 件になった parent は削除                      │
│   - 件数検証                                               │
└────────────────────────────────────────────────────────────┘
                          ↓
┌────────────────────────────────────────────────────────────┐
│ jlpt-app-scripts/bin/99_normalize_levels.rs (新規)         │
│   - level_name を to_uppercase() で統一                    │
│   - dedup より**前**に実行                                 │
└────────────────────────────────────────────────────────────┘
                          ↓
┌────────────────────────────────────────────────────────────┐
│ jlpt-app-scripts/bin/2_duplicate.rs (改修)                 │
│   - dedup_key を選択肢セット込みに変更                     │
│   - 正規化ルールを新規ヘルパーに抽出 (テスト可能に)        │
└────────────────────────────────────────────────────────────┘
```

### データフロー (既存データ掃除)

```
Firestore (本番)
  ↓ GCS Export (バックアップ)
Firestore (読み取り)
  ↓ 99_report_duplicates.rs
duplicates_report.json (削除計画)
  ↓ ユーザーレビュー
  ↓ 99_apply_dedup.rs (--execute)
Firestore (更新: sub_questions部分 + 必要に応じ parent削除)
  ↓ 件数検証
完了ログ
```

### 再発防止フロー (パイプライン)

```
Gemini生成
  ↓ 0_create_questions.rs / 0_create_targeted.rs
  ↓ 1_json_read_to_struct.rs
  ↓ 1_5_validate_questions.rs
  ↓ 1_7_shuffle_answers.rs
  ↓ 2_duplicate.rs (改修: dedup_key = level_id + sorted_options + answer_value)
  ↓ 3_numbering.rs
  ↓ 4_leveling.rs
  ↓ 5_categories_to_meta.rs
  ↓ 99_questions_to_database.rs
Firestore (新規投入)
```

---

## 詳細設計

### dedup キー生成ロジック

```rust
fn normalize_text(s: &str) -> String {
    use unicode_normalization::UnicodeNormalization;
    s.nfkc().collect::<String>()
        .trim()
        .to_string()
}

fn dedup_key(level_id: u32, sub: &SubQuestion) -> Option<String> {
    // 並び替え問題 (選択肢がキー番号のみ) は dedup 対象外
    let option_values: Vec<String> = sub.select_answer.iter()
        .map(|sa| normalize_text(&sa.value))
        .collect();

    // ('1','2','3','4') 選択肢は対象外
    let mut sorted_opts = option_values.clone();
    sorted_opts.sort();
    if sorted_opts == vec!["1","2","3","4"] {
        return None;
    }

    // 正解値を取得
    let answer_value = sub.select_answer.iter()
        .find(|sa| sa.key == sub.answer)
        .map(|sa| normalize_text(&sa.value))?;

    Some(format!(
        "L{}|OPT[{}]|ANS[{}]",
        level_id,
        sorted_opts.join(","),
        answer_value
    ))
}
```

### tiebreaker 比較関数

```rust
struct Candidate {
    parent_id: String,
    sub_idx: usize,
    sub: SubQuestion,
    create_time: chrono::DateTime<chrono::Utc>, // Firestoreから取得
    sentence_len: usize,
}

fn prefer_this_over_that(a: &Candidate, b: &Candidate) -> std::cmp::Ordering {
    // 1. createTime 古い方を残す
    match a.create_time.cmp(&b.create_time) {
        std::cmp::Ordering::Equal => {}
        ord => return ord,
    }
    // 2. sentence 長い方を残す
    match b.sentence_len.cmp(&a.sentence_len) {
        std::cmp::Ordering::Equal => {}
        ord => return ord,
    }
    // 3. qid 辞書順
    a.parent_id.cmp(&b.parent_id)
}
```

### レポート JSON スキーマ

```json
{
  "generated_at": "2026-04-19T12:00:00Z",
  "source": "firestore:argon-depth-446413-t0/(default)",
  "total_parents": 14857,
  "total_sub_questions": 21809,
  "dedup_groups": 328,
  "rows_in_dup_groups": 875,
  "removable_subs": 547,
  "removable_parents": 0,
  "groups": [
    {
      "dedup_key": "L5|OPT[が,で,に,を]|ANS[が]",
      "keep": {
        "parent_id": "abc123...",
        "sub_idx": 0,
        "sentence": "きょうは あめ（　）ふっています",
        "create_time": "2026-03-10T00:00:00Z"
      },
      "remove": [
        {
          "parent_id": "def456...",
          "sub_idx": 1,
          "sentence": "テーブルの 上に 本（　）あります",
          "create_time": "2026-03-15T00:00:00Z"
        }
      ]
    }
  ]
}
```

### 削除実行スクリプトの動作

```
入力: duplicates_report.json
引数: --execute (なければ dry-run)

処理:
  1. レポートの groups をロード
  2. 削除対象の parent_id ごとに更新をグループ化
     - 単一 parent 内で複数 sub を削除するケースを1 write にまとめる
  3. 各 parent について:
     a. Firestore から最新の Question を取得
     b. remove リストの sub_idx を除外した新しい sub_questions を構築
     c. sub_questions が空なら parent 削除
     d. そうでなければ sub_questions フィールド更新
  4. WriteBatch (500 docs/batch) で送信
  5. 完了後、該当 parent を再取得し、期待通りに更新されているか検証
  6. 件数サマリ出力 (期待 vs 実際)

リスク低減:
  - dry-run モードでは書き込みせず、操作ログのみ出力
  - 1 batch = 最大 500 writes (Firestore制限)
  - エラー時は失敗した parent のリストを出力し、中断
```

### エラーハンドリング

- Firestore rate limit: リトライ with exponential backoff (3回まで)
- WriteBatch 失敗時: 当該 batch をスキップし、後でリトライ可能なよう失敗リスト出力
- ネットワーク切断: 中断して再実行可能に (冪等性: 既に削除済みの parent は skip)

### テスト戦略

単体テスト:
- `normalize_text`: NFKC 正規化、空白除去、英数字半角化
- `dedup_key`: パターンA/B/C の判別、'1234' 除外
- `prefer_this_over_that`: createTime 優先、同値時の sentence 長優先

統合テスト:
- Firestore エミュレータでの dry-run 実行
- レポート生成 → 適用 → 検証の一連

E2Eテスト:
- ステージング環境 (もしあれば) で小規模データでの動作確認
- 本番は最終的に dry-run → ユーザーレビュー → 実行

---

## 実行順序 (refresh-token Phase 1 完了後に着手)

1. **Phase 0: 準備**
   - Firestore Export でバックアップ (GCS バケットへ)
   - feature ブランチ作成: `feature/dedup-cleanup`

2. **Phase 1: level_name 正規化**
   - `99_normalize_levels.rs` 作成
   - dry-run で対象件数確認 (期待: 4,992件)
   - 実行
   - 検証: `level_name="n*"` の件数が全て 0 になる

3. **Phase 2: レポート生成**
   - `99_report_duplicates.rs` 作成
   - 実行 → `reports/duplicates.json` 出力 (期待: `removable_subs` ≈ 547)
   - ユーザーレビュー: サンプル10〜20件を目視確認

4. **Phase 3: dedup 実行**
   - `99_apply_dedup.rs` 作成
   - dry-run 実行
   - レビュー後、--execute で実行
   - 検証: 実削除数が dry-run 予測と一致

5. **Phase 4: パイプライン改修**
   - `2_duplicate.rs` の dedup_key を選択肢セット込みに改修
   - 正規化ヘルパーを `utils.rs` に追加
   - 単体テスト追加
   - 次回 Gemini 生成時に新 dedup ロジックが走ることを確認

6. **Phase 5: ドキュメント更新**
   - `jlpt-app-scripts/docs/pipeline.md` に新 dedup ロジック記載
   - `docs/gcp-environment.md` に既存データ掃除の記録

---

## リスクと対応

| リスク | 影響 | 対応 |
|--------|------|------|
| 削除対象選定ロジックのバグ | 正当な問題を誤削除 | dry-run + ユーザーレビュー + バックアップ |
| Firestore Export 失敗 | バックアップなしで削除 | Export 成功確認を前提条件に |
| user_answers 孤立 | 学習履歴のバグ | 現状39件と極小。削除後クリーンアップスクリプト実行 |
| level_name 正規化による既存インデックス破壊 | クエリパフォーマンス劣化 | Phase 1 着手時に `firestore.indexes.json` を確認し、level_name を含む複合インデックスがあれば事前に評価する (現時点未確認) |
| 並行する refresh-token 作業との衝突 | scripts と backend で独立なので問題なし | なし |

---

## 削減見込み (実データ実測)

- 削除される sub_questions: **547件** (21,809 中の 2.5%)
- 削除される parent Questions: **約0〜数十件** (大半は sub_questions 配列の更新のみ)
- level_name 正規化: **約4,000件** の文字列更新
- 連鎖削除される votes: **0件**
- 連鎖削除される user_answers: **極少 (推定10件未満)**

レベル別削除候補内訳:
- N5: 252件 (削除候補の46.1%)
- N4: 125件
- N3: 70件
- N1: 55件
- N2: 45件

→ N5 の重複集中は生成プロンプトの課題を示唆。別Issue化。

---

## 成功基準 (K2: 検証可能ゴール)

1. **Phase 1**: `level_name = "N5"` の件数が 2,935件 (= 1,463 + 1,472) になる。`"n5"` の件数が 0 件になる。
2. **Phase 2**: `duplicates_report.json` が生成され、`removable_subs` が 500〜600 の範囲。ユーザーが任意5サンプルを確認し重複として妥当と判断する。
3. **Phase 3**: dry-run 出力と実行後の Firestore カウントが一致。実行前後で `sub_questions` 総数が `removable_subs` の分だけ減少。
4. **Phase 4**: `2_duplicate.rs` の単体テストが pass し、新旧 dedup_key の挙動差がテストで示される。
5. **Phase 5**: 関連ドキュメント 2 ファイルが更新される。

---

## 確定事項 (ユーザー回答済み 2026-04-19)

1. **削除の最終承認形式**: レポートJSONを目視して「実行OK」と返す方式で進める。group単位の対象外マーク機構は作らない (YAGNI)。
2. **別Issue化する項目の推奨優先順位**:
   1. **N5 生成プロンプト改善** (削除候補の46%が N5 → ROI最大、新規生成の再発防止に直結)
   2. **category=9 並び替え問題の key==value 21件バグ調査** (生成パイプラインのバグ、影響は狭いが明確な不具合)
   3. **category=7 に紛れた不正フォーマット5件** (qid=b1f3dcbe の単一ドキュメント、目視確認で判定)
   4. **管理画面UI (重複監視)** (継続監視は必要だが、本設計の dedup + パイプライン改修で当面のリスクは十分低減されるため後回し可能)

---

## 付録: Gemini pro レビュー所見 (2026-04-19)

Gemini pro (gemini-3.1-pro-preview) に設計草案をレビューさせた結果:

- dedup キーは 案β (選択肢+正解値+level) 推奨 → **採用**
- 削除単位は sub_question 推奨 → **採用**
- tiebreaker は createTime 古い方 > sentence長 推奨 → **採用**
- WriteBatch アトミック化推奨 → **採用**
- level_name to_uppercase() 正規化必須 → **採用**
- ガードレール (dry-run + Export + レポート + 検証) 推奨 → **採用**

参照: SurrealDB review_log (tags: jlpt_base, dedup, gemini-pro)
