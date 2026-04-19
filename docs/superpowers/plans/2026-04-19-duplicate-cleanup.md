# JLPT 問題データ重複対策 実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 本番Firestore上の重複問題 547件 (選択肢セット+正解値+level_id が一致する sub_question) を掃除し、パイプライン側 (`2_duplicate.rs`) を改修して再発を防止する。

**Architecture:** jlpt-app-scripts に 3本の one-shot バイナリを追加 (`99_normalize_levels.rs`, `99_report_duplicates.rs`, `99_apply_dedup.rs`) + 既存 `2_duplicate.rs` の dedup_key を共通ヘルパー (`dedup_common.rs`) 経由に差し替える。tiebreaker には Firestore の `createTime` システムメタデータを REST API 経由で取得する。すべての破壊的操作に dry-run モードを用意し、JSON レポート目視承認を挟む。

**Tech Stack:** Rust 2024, firestore 0.44, tokio, serde, unicode-normalization (新規依存), chrono, dotenv, env_logger, reqwest (新規依存)

**Design Doc:** `docs/superpowers/specs/2026-04-19-duplicate-cleanup-design.md`

**ブランチ:** `feature/dedup-cleanup` を `jlpt-app-scripts` に作成 (refresh-token とは別リポジトリなので衝突なし)

---

## Phase 0: 準備

### Task 1: 作業ブランチの作成とバックアップ

**Files:**
- なし (環境操作のみ)

- [ ] **Step 1: jlpt-app-scripts で作業ブランチを作成**

Run:
```bash
cd /home/o9oem/workspace/mine/jlpt_base/jlpt-app-scripts
git checkout master
git pull origin master
git checkout -b feature/dedup-cleanup
```
Expected: Switched to a new branch 'feature/dedup-cleanup'

- [ ] **Step 2: Firestore Export でバックアップ**

Run:
```bash
gcloud config configurations activate jlpt
# バックアップ先バケット存在確認 (なければ作成)
BUCKET="gs://argon-depth-446413-t0-firestore-backup"
gcloud storage buckets describe $BUCKET 2>&1 | head -3 || \
  gcloud storage buckets create $BUCKET --project=argon-depth-446413-t0 --location=asia-northeast1

# Export 実行 (questions コレクションのみでなく全体、rollback 時に備える)
TIMESTAMP=$(date -u +"%Y%m%d-%H%M%S")
gcloud firestore export $BUCKET/backup-$TIMESTAMP \
  --collection-ids=questions,votes,user_answers \
  --project=argon-depth-446413-t0
```
Expected: metadata.operationId が表示される。`gcloud firestore operations list` で SUCCESSFUL になるまで待機。

- [ ] **Step 3: バックアップ完了確認**

Run:
```bash
gcloud firestore operations list --project=argon-depth-446413-t0 --limit=1 --format="value(state,done)"
```
Expected: `SUCCESSFUL` (または `done=True`)

- [ ] **Step 4: gcloud config を default に戻す**

Run:
```bash
gcloud config configurations activate default
```
Expected: `Activated [default].`

- [ ] **Step 5: コミット (ブランチのみ、コード変更なし)**

この task にコミット対象はない。ブランチ作成と Export はリモート操作。

---

## Phase 4: 共通ヘルパーとパイプライン改修 (TDD先行)

Phase 1-3 が破壊的な本番操作のため、**共通ヘルパー (dedup_common.rs) をテストで確立してから既存データに触る**。設計ドキュメントの記載順序 (Phase 1-5) から意図的に変更。

### Task 2: 依存追加とテストディレクトリ準備

**Files:**
- Modify: `jlpt-app-scripts/Cargo.toml`
- Create: `jlpt-app-scripts/tests/dedup_common_tests.rs`

- [ ] **Step 1: Cargo.toml に依存を追加**

`jlpt-app-scripts/Cargo.toml` の `[dependencies]` セクション末尾に追加:

```toml
unicode-normalization = "0.1"
reqwest = { version = "0.12", features = ["json"] }
```

- [ ] **Step 2: `[[bin]]` セクションに 3 本追加**

`jlpt-app-scripts/Cargo.toml` に追記:

```toml
# レベル名の大文字統一 (one-shot)
[[bin]]
name = "normalize_levels"
path = "bin/99_normalize_levels.rs"

# 重複レポート生成 (読み取り専用 one-shot)
[[bin]]
name = "report_duplicates"
path = "bin/99_report_duplicates.rs"

# dedup レポート適用 (破壊的 one-shot)
[[bin]]
name = "apply_dedup"
path = "bin/99_apply_dedup.rs"
```

- [ ] **Step 3: tests ディレクトリを作成し空のテストファイル追加**

Run:
```bash
mkdir -p /home/o9oem/workspace/mine/jlpt_base/jlpt-app-scripts/tests
```

Create `jlpt-app-scripts/tests/dedup_common_tests.rs`:
```rust
// Phase 4 Task 3 以降でテストを埋める
```

- [ ] **Step 4: ビルド確認**

Run:
```bash
cd /home/o9oem/workspace/mine/jlpt_base/jlpt-app-scripts
cargo check 2>&1 | tail -10
```
Expected: "Compiling scripts" → "Finished" (`[[bin]]` が指すファイルはまだ無いので warning は出るが、エラーにはならない)

実際にはこの時点では新 bin 本体が未作成のため、`cargo check` は各 bin のコンパイルに失敗する可能性がある。Step 4 では以下のコマンドで既存 bin のみを確認:

Run:
```bash
cargo check --bin duplicate 2>&1 | tail -5
```
Expected: `Finished`

- [ ] **Step 5: コミット**

Run:
```bash
git add Cargo.toml tests/dedup_common_tests.rs
git commit -m "chore: dedup 対策用の依存追加 (unicode-normalization, reqwest)"
```

---

### Task 3: dedup_common.rs — normalize_text 関数 (TDD)

**Files:**
- Modify: `jlpt-app-scripts/tests/dedup_common_tests.rs`
- Create: `jlpt-app-scripts/bin/dedup_common.rs`

- [ ] **Step 1: 失敗するテストを書く**

`jlpt-app-scripts/tests/dedup_common_tests.rs`:

```rust
#[path = "../bin/dedup_common.rs"]
mod dedup_common;

use dedup_common::normalize_text;

#[test]
fn normalize_text_trims_whitespace() {
    assert_eq!(normalize_text("  hello  "), "hello");
}

#[test]
fn normalize_text_preserves_middle_spaces() {
    // 設計上、中間空白は trim しない (単語区切りとして意味を持つ場合がある)
    // → 変更: 実データで「つくえの うえに ほんが」のような空白は残す
    assert_eq!(normalize_text("hello world"), "hello world");
}

#[test]
fn normalize_text_nfkc_fullwidth_digit() {
    // 全角 "１" → 半角 "1"
    assert_eq!(normalize_text("１"), "1");
}

#[test]
fn normalize_text_nfkc_halfwidth_kana() {
    // 半角カナ "ｱ" → 全角 "ア" (NFKC は半角カナを全角に合成)
    assert_eq!(normalize_text("ｱ"), "ア");
}

#[test]
fn normalize_text_preserves_hiragana_kanji() {
    // ひらがな/漢字はそのまま
    assert_eq!(normalize_text("役割"), "役割");
    assert_eq!(normalize_text("やくわり"), "やくわり");
}

#[test]
fn normalize_text_empty() {
    assert_eq!(normalize_text(""), "");
    assert_eq!(normalize_text("   "), "");
}
```

- [ ] **Step 2: テストを実行し FAIL を確認**

Run:
```bash
cd /home/o9oem/workspace/mine/jlpt_base/jlpt-app-scripts
cargo test --test dedup_common_tests 2>&1 | tail -10
```
Expected: コンパイルエラー (`bin/dedup_common.rs` が存在しない)

- [ ] **Step 3: bin/dedup_common.rs を作成し最小実装**

Create `jlpt-app-scripts/bin/dedup_common.rs`:

```rust
//! 重複検出用の共通ヘルパー。
//!
//! 複数のバイナリ (`2_duplicate.rs`, `99_report_duplicates.rs`, `99_apply_dedup.rs`) から
//! `#[path = "dedup_common.rs"] mod dedup_common;` のパターンで読み込まれる。

#![allow(dead_code)]

use unicode_normalization::UnicodeNormalization;

/// 選択肢値・正解値などのテキストを正規化する。
///
/// - Unicode NFKC 正規化 (全角→半角、半角カナ→全角カナ等)
/// - 前後空白 trim
/// - 中間空白は保持 (JLPTでは意味を持つケースがあるため)
///
/// ひらがな/漢字の表記揺れは**意図的に吸収しない** (JLPTの出題要素のため)。
pub fn normalize_text(s: &str) -> String {
    s.nfkc().collect::<String>().trim().to_string()
}
```

- [ ] **Step 4: テストを実行し PASS を確認**

Run:
```bash
cargo test --test dedup_common_tests 2>&1 | tail -15
```
Expected: `test result: ok. 6 passed`

- [ ] **Step 5: コミット**

Run:
```bash
git add bin/dedup_common.rs tests/dedup_common_tests.rs
git commit -m "feat(scripts): dedup_common.rs に normalize_text を追加 (TDD)"
```

---

### Task 4: dedup_common.rs — dedup_key 関数 (TDD)

**Files:**
- Modify: `jlpt-app-scripts/tests/dedup_common_tests.rs`
- Modify: `jlpt-app-scripts/bin/dedup_common.rs`

- [ ] **Step 1: テストを追加**

`jlpt-app-scripts/tests/dedup_common_tests.rs` の末尾に追加:

```rust
use dedup_common::{dedup_key, SubLike};

fn mk_sub(options: &[(&str, &str)], answer: &str) -> SubLike {
    SubLike {
        options: options.iter().map(|(k,v)| (k.to_string(), v.to_string())).collect(),
        answer: answer.to_string(),
    }
}

#[test]
fn dedup_key_basic() {
    let sub = mk_sub(&[("1","役割"),("2","役目"),("3","配役"),("4","役者")], "1");
    let key = dedup_key(5, &sub).expect("should produce a key");
    // sorted option values + normalized answer value + level
    assert_eq!(key, "L5|OPT[役割,役者,役目,配役]|ANS[役割]");
}

#[test]
fn dedup_key_same_options_different_answer_produces_different_keys() {
    let sub_a = mk_sub(&[("1","が"),("2","で"),("3","に"),("4","を")], "1");
    let sub_b = mk_sub(&[("1","が"),("2","で"),("3","に"),("4","を")], "4");
    let key_a = dedup_key(5, &sub_a).unwrap();
    let key_b = dedup_key(5, &sub_b).unwrap();
    assert_ne!(key_a, key_b);
}

#[test]
fn dedup_key_excludes_numeric_options() {
    // category=9 並び替え問題は選択肢が '1','2','3','4' で値が意味を持たないため None を返す
    let sub = mk_sub(&[("1","1"),("2","2"),("3","3"),("4","4")], "1");
    assert!(dedup_key(3, &sub).is_none());
}

#[test]
fn dedup_key_excludes_numeric_options_after_shuffle() {
    // key と value が食い違っていても値がすべて '1'〜'4' ならスキップ
    let sub = mk_sub(&[("1","3"),("2","1"),("3","4"),("4","2")], "2");
    assert!(dedup_key(3, &sub).is_none());
}

#[test]
fn dedup_key_returns_none_when_answer_value_missing() {
    // answer="5" だが選択肢に key="5" がない → None
    let sub = mk_sub(&[("1","a"),("2","b"),("3","c"),("4","d")], "5");
    assert!(dedup_key(1, &sub).is_none());
}

#[test]
fn dedup_key_respects_level_id() {
    let sub = mk_sub(&[("1","a"),("2","b"),("3","c"),("4","d")], "1");
    let k1 = dedup_key(1, &sub).unwrap();
    let k2 = dedup_key(2, &sub).unwrap();
    assert_ne!(k1, k2);
}

#[test]
fn dedup_key_normalizes_fullwidth() {
    let sub_half = mk_sub(&[("1","1A"),("2","2B"),("3","3C"),("4","4D")], "1");
    let sub_full = mk_sub(&[("1","１Ａ"),("2","２Ｂ"),("3","３Ｃ"),("4","４Ｄ")], "1");
    // ただし値が全て '1'〜'4' で始まるが "1A" など別文字列なので除外ルールには該当しない
    assert_eq!(dedup_key(1, &sub_half), dedup_key(1, &sub_full));
}
```

- [ ] **Step 2: テスト実行で FAIL 確認**

Run:
```bash
cargo test --test dedup_common_tests 2>&1 | tail -10
```
Expected: コンパイルエラー (`dedup_key`, `SubLike` 未定義)

- [ ] **Step 3: dedup_common.rs に実装追加**

`jlpt-app-scripts/bin/dedup_common.rs` の末尾に追加:

```rust
/// dedup 判定に必要な SubQuestion の部分ビュー。
///
/// `bin/utils.rs` の `SubQuestion` とは独立に定義し、テストしやすくする。
/// 実際の呼び出し側で `SubQuestion` から変換する。
pub struct SubLike {
    /// (key, value) のペア。順不同。
    pub options: Vec<(String, String)>,
    /// 正解のキー (例: "1"〜"4")
    pub answer: String,
}

/// dedup キーを生成する。
///
/// 形式: `"L{level_id}|OPT[{sorted_normalized_values}]|ANS[{normalized_answer_value}]"`
///
/// 以下のケースでは `None` を返す (dedup 対象外):
/// - 選択肢値が正規化後すべて `"1"`, `"2"`, `"3"`, `"4"` (並び替え問題のプレースホルダ)
/// - `answer` キーに対応する value が選択肢に存在しない (不正データ)
pub fn dedup_key(level_id: u32, sub: &SubLike) -> Option<String> {
    // すべての選択肢値を正規化
    let normalized: Vec<(String, String)> = sub.options.iter()
        .map(|(k, v)| (k.clone(), normalize_text(v)))
        .collect();

    // 値を sort
    let mut sorted_values: Vec<String> = normalized.iter().map(|(_, v)| v.clone()).collect();
    sorted_values.sort();

    // '1'〜'4' のみからなる選択肢は除外
    if sorted_values == vec!["1".to_string(), "2".to_string(), "3".to_string(), "4".to_string()] {
        return None;
    }

    // answer キーから value を引く
    let answer_value = normalized.iter()
        .find(|(k, _)| k == &sub.answer)
        .map(|(_, v)| v.clone())?;

    Some(format!(
        "L{}|OPT[{}]|ANS[{}]",
        level_id,
        sorted_values.join(","),
        answer_value
    ))
}
```

- [ ] **Step 4: テスト実行で PASS 確認**

Run:
```bash
cargo test --test dedup_common_tests 2>&1 | tail -15
```
Expected: `test result: ok. 13 passed`

- [ ] **Step 5: コミット**

Run:
```bash
git add bin/dedup_common.rs tests/dedup_common_tests.rs
git commit -m "feat(scripts): dedup_common.rs に dedup_key を追加 (選択肢+正解+level, 数字選択肢は除外)"
```

---

### Task 5: dedup_common.rs — tiebreaker (prefer_keep_order)

**Files:**
- Modify: `jlpt-app-scripts/tests/dedup_common_tests.rs`
- Modify: `jlpt-app-scripts/bin/dedup_common.rs`

- [ ] **Step 1: テストを追加**

`jlpt-app-scripts/tests/dedup_common_tests.rs` の末尾に追加:

```rust
use chrono::{TimeZone, Utc};
use dedup_common::{prefer_keep_order, Candidate};

#[test]
fn prefer_keep_order_older_create_time_wins() {
    let older = Candidate {
        parent_id: "z".into(),
        sub_idx: 0,
        create_time: Utc.with_ymd_and_hms(2026, 3, 1, 0, 0, 0).unwrap(),
        sentence_len: 10,
    };
    let newer = Candidate {
        parent_id: "a".into(),
        sub_idx: 0,
        create_time: Utc.with_ymd_and_hms(2026, 3, 15, 0, 0, 0).unwrap(),
        sentence_len: 100,
    };
    let mut v = vec![newer.clone(), older.clone()];
    v.sort_by(prefer_keep_order);
    assert_eq!(v[0].parent_id, "z"); // older wins
}

#[test]
fn prefer_keep_order_same_time_longer_sentence_wins() {
    let time = Utc.with_ymd_and_hms(2026, 3, 1, 0, 0, 0).unwrap();
    let short = Candidate {
        parent_id: "z".into(), sub_idx: 0, create_time: time, sentence_len: 10,
    };
    let long = Candidate {
        parent_id: "a".into(), sub_idx: 0, create_time: time, sentence_len: 100,
    };
    let mut v = vec![short.clone(), long.clone()];
    v.sort_by(prefer_keep_order);
    assert_eq!(v[0].parent_id, "a"); // longer sentence wins
}

#[test]
fn prefer_keep_order_same_time_same_length_lexical_parent_id() {
    let time = Utc.with_ymd_and_hms(2026, 3, 1, 0, 0, 0).unwrap();
    let a = Candidate { parent_id: "a".into(), sub_idx: 0, create_time: time, sentence_len: 10 };
    let b = Candidate { parent_id: "b".into(), sub_idx: 0, create_time: time, sentence_len: 10 };
    let mut v = vec![b.clone(), a.clone()];
    v.sort_by(prefer_keep_order);
    assert_eq!(v[0].parent_id, "a"); // lexical order
}
```

- [ ] **Step 2: テスト実行で FAIL 確認**

Run:
```bash
cargo test --test dedup_common_tests 2>&1 | tail -10
```
Expected: コンパイルエラー (`Candidate`, `prefer_keep_order` 未定義)

- [ ] **Step 3: dedup_common.rs に実装追加**

`jlpt-app-scripts/bin/dedup_common.rs` の末尾に追加:

```rust
use chrono::{DateTime, Utc};

/// tiebreaker のための候補情報。
#[derive(Clone, Debug)]
pub struct Candidate {
    pub parent_id: String,
    pub sub_idx: usize,
    pub create_time: DateTime<Utc>,
    pub sentence_len: usize,
}

/// 重複グループ内で「残すレコード」を決めるための比較関数。
///
/// 優先順位: createTime 古い方 → sentence 長い方 → parent_id 辞書順
///
/// `Vec::sort_by(prefer_keep_order)` でソートすると、先頭が「残すべきレコード」になる。
pub fn prefer_keep_order(a: &Candidate, b: &Candidate) -> std::cmp::Ordering {
    // 1. createTime 古い方を先頭に
    match a.create_time.cmp(&b.create_time) {
        std::cmp::Ordering::Equal => {}
        ord => return ord,
    }
    // 2. sentence 長い方を先頭に (降順なので b.len.cmp(a.len))
    match b.sentence_len.cmp(&a.sentence_len) {
        std::cmp::Ordering::Equal => {}
        ord => return ord,
    }
    // 3. parent_id 辞書順
    a.parent_id.cmp(&b.parent_id)
}
```

- [ ] **Step 4: テスト実行で PASS 確認**

Run:
```bash
cargo test --test dedup_common_tests 2>&1 | tail -15
```
Expected: `test result: ok. 16 passed`

- [ ] **Step 5: コミット**

Run:
```bash
git add bin/dedup_common.rs tests/dedup_common_tests.rs
git commit -m "feat(scripts): dedup_common.rs に prefer_keep_order tiebreaker 追加"
```

---

### Task 6: 既存 2_duplicate.rs を dedup_common ベースに改修

**Files:**
- Modify: `jlpt-app-scripts/bin/2_duplicate.rs`

- [ ] **Step 1: 現状の `2_duplicate.rs` を読む (参考: bin/2_duplicate.rs:1-159)**

既存実装は `dedup_key = sentence + "||" + correct_value` + Levenshtein 0.85 類似判定。これを dedup_common の `dedup_key` ベースに置き換える。文類似度は使わない (新dedupは選択肢ベースで十分保守的)。

- [ ] **Step 2: 新しい `2_duplicate.rs` に差し替え**

`jlpt-app-scripts/bin/2_duplicate.rs` の全内容を次に置き換える:

```rust
use std::collections::HashSet;

use log::{info, warn};

mod utils;
#[path = "dedup_common.rs"]
mod dedup_common;

use crate::utils::{
    read_questions_from_stage, write_questions_to_stage, Question, LEVELS, STAGE_2_OUTPUT,
};
use crate::dedup_common::{dedup_key, SubLike};

fn main() {
    crate::utils::init_logger();

    let start = std::time::Instant::now();

    for level in LEVELS {
        // シャッフル済みファイルを優先
        let input_file = "1_7_shuffled.json";
        let questions = match read_questions_from_stage(level, input_file) {
            Ok(q) => q,
            Err(e) => {
                warn!("[{}] {}", level, e);
                continue;
            }
        };

        if questions.is_empty() {
            warn!("[{}] 問題データなし、スキップ", level);
            continue;
        }

        let original_count = questions.len();
        let original_sub_count: usize = questions.iter().map(|q| q.sub_questions.len()).sum();

        // dedup_key での重複検出
        let mut seen_keys: HashSet<String> = HashSet::new();
        let mut deduplicated: Vec<Question> = Vec::new();
        let mut removed_as_dup = 0usize;
        let mut excluded_numeric = 0usize;
        let mut invalid = 0usize;

        for question in questions {
            let mut kept_subs = Vec::new();
            for sub_q in question.sub_questions {
                let sub_like = SubLike {
                    options: sub_q.select_answer.iter()
                        .map(|sa| (sa.key.clone(), sa.value.clone()))
                        .collect(),
                    answer: sub_q.answer.clone(),
                };
                match dedup_key(question.level_id, &sub_like) {
                    Some(key) => {
                        if seen_keys.contains(&key) {
                            removed_as_dup += 1;
                            continue;
                        }
                        seen_keys.insert(key);
                        kept_subs.push(sub_q);
                    }
                    None => {
                        // dedup 対象外 (数字選択肢 or 正解値取得不可)
                        // 数字選択肢のどちらかを判別
                        let all_values_numeric = sub_q.select_answer.iter()
                            .all(|sa| matches!(sa.value.trim(), "1"|"2"|"3"|"4"));
                        if all_values_numeric && sub_q.select_answer.len() == 4 {
                            excluded_numeric += 1;
                        } else {
                            invalid += 1;
                        }
                        kept_subs.push(sub_q);
                    }
                }
            }

            if kept_subs.is_empty() {
                continue;
            }

            deduplicated.push(Question {
                sub_questions: kept_subs,
                ..question
            });
        }

        let remaining_sub: usize = deduplicated.iter().map(|q| q.sub_questions.len()).sum();
        info!(
            "[{}] questions: {} → {}, sub_questions: {} → {} (重複除外={}, 数字選択肢スキップ={}, 不正データスキップ={})",
            level,
            original_count,
            deduplicated.len(),
            original_sub_count,
            remaining_sub,
            removed_as_dup,
            excluded_numeric,
            invalid,
        );

        match write_questions_to_stage(level, STAGE_2_OUTPUT, &deduplicated) {
            Ok(_) => info!("[{}] wrote {}", level, STAGE_2_OUTPUT),
            Err(e) => warn!("[{}] 書込失敗: {}", level, e),
        }
    }

    info!("done, elapsed: {:?}", start.elapsed());
}
```

- [ ] **Step 3: ビルド確認**

Run:
```bash
cargo check --bin duplicate 2>&1 | tail -5
```
Expected: `Finished`

- [ ] **Step 4: 既存テストが引き続き PASS することを確認**

Run:
```bash
cargo test --test dedup_common_tests 2>&1 | tail -5
```
Expected: `test result: ok. 16 passed`

- [ ] **Step 5: コミット**

Run:
```bash
git add bin/2_duplicate.rs
git commit -m "refactor(scripts): 2_duplicate.rs を dedup_common ベースに改修 (選択肢セット込みkey)"
```

---

## Phase 1: level_name 正規化 (one-shot)

### Task 7: 99_normalize_levels.rs — 共通基盤

**Files:**
- Create: `jlpt-app-scripts/bin/99_normalize_levels.rs`

- [ ] **Step 1: ファイル作成**

Create `jlpt-app-scripts/bin/99_normalize_levels.rs`:

```rust
//! 本番 Firestore の `questions.level_name` が "N5"/"n5" のように大小文字混在しているため、
//! すべて大文字 ("N1" 〜 "N5") に統一する one-shot スクリプト。
//!
//! --dry-run (デフォルト): 対象件数のみを報告、書き込みなし
//! --execute: 実際に Firestore を更新

use futures_util::StreamExt;
use log::{error, info, warn};
use std::env;

mod utils;

#[tokio::main]
async fn main() {
    dotenv::dotenv().ok();
    utils::init_logger();

    let execute = env::args().any(|a| a == "--execute");
    let mode = if execute { "EXECUTE" } else { "DRY-RUN" };
    info!("=== 99_normalize_levels [{}] ===", mode);

    let project_id = env::var("PROJECT_ID").expect("PROJECT_ID must be set");
    let db = match firestore::FirestoreDb::new(&project_id).await {
        Ok(d) => d,
        Err(e) => {
            error!("FirestoreDb::new failed: {:?}", e);
            return;
        }
    };

    // 全 questions を取得 (doc 全体を保持)
    let stream_result = db
        .fluent()
        .list()
        .from("questions")
        .obj::<serde_json::Value>()
        .stream_all_with_errors()
        .await;

    let mut stream = match stream_result {
        Ok(s) => s,
        Err(e) => {
            error!("stream_all_with_errors failed: {:?}", e);
            return;
        }
    };

    let mut total = 0usize;
    let mut needs_update: Vec<(String, String, String, serde_json::Value)> = Vec::new();
    // (id, old_level, new_level, full_doc)

    while let Some(item) = stream.next().await {
        let doc = match item {
            Ok(d) => d,
            Err(e) => {
                warn!("stream item error: {:?}", e);
                continue;
            }
        };
        total += 1;
        let id = doc.get("id").and_then(|v| v.as_str()).unwrap_or("").to_string();
        let old_level = doc.get("level_name").and_then(|v| v.as_str()).unwrap_or("").to_string();
        let new_level = old_level.to_uppercase();
        if old_level != new_level && !id.is_empty() {
            needs_update.push((id, old_level, new_level, doc));
        }
    }

    info!("total docs: {}, needs update: {}", total, needs_update.len());

    // 変更内訳を level_name 別にカウント
    let mut breakdown: std::collections::HashMap<String, usize> = std::collections::HashMap::new();
    for (_, old, _, _) in &needs_update {
        *breakdown.entry(old.clone()).or_insert(0) += 1;
    }
    let mut sorted: Vec<_> = breakdown.into_iter().collect();
    sorted.sort();
    for (old, count) in sorted {
        info!("  {} → {}: {}件", old, old.to_uppercase(), count);
    }

    if !execute {
        info!("(dry-run: 変更は適用されません。--execute で実行)");
        return;
    }

    // 実行: 既存 backend パターン (full object replace) を踏襲
    info!("実行中...");
    let mut ok = 0usize;
    let mut err = 0usize;
    for (id, _old, new_level, doc) in &needs_update {
        let mut new_doc = doc.clone();
        if let Some(obj) = new_doc.as_object_mut() {
            obj.insert("level_name".to_string(), serde_json::Value::String(new_level.clone()));
        }
        match db
            .fluent()
            .update()
            .in_col("questions")
            .document_id(id)
            .object(&new_doc)
            .execute::<serde_json::Value>()
            .await
        {
            Ok(_) => ok += 1,
            Err(e) => {
                error!("update failed id={}: {:?}", id, e);
                err += 1;
            }
        }
        if (ok + err) % 500 == 0 {
            info!("進捗: {} / {}", ok + err, needs_update.len());
        }
    }
    info!("done: ok={}, err={}", ok, err);
}
```

- [ ] **Step 2: dry-run 実行 (本番DB)**

Run:
```bash
gcloud config configurations activate jlpt
cd /home/o9oem/workspace/mine/jlpt_base/jlpt-app-scripts
export PROJECT_ID=argon-depth-446413-t0
export RUST_LOG=info
cargo run --bin normalize_levels 2>&1 | tail -30
```
Expected:
- `total docs: 14857`
- `needs update: 4992` (または近い値)
- レベル別内訳で `n1 → N1: 1090`, `n2 → N2: 1209`, `n3 → N3: 549`, `n4 → N4: 672`, `n5 → N5: 1472` が表示される
- `(dry-run: 変更は適用されません)` で終了

- [ ] **Step 3: dry-run 出力をユーザーに提示し承認を得る**

コンソール出力をそのままユーザーに報告。承認が得られたら Step 4 へ。

- [ ] **Step 4: --execute で実行**

Run:
```bash
cargo run --bin normalize_levels -- --execute 2>&1 | tail -30
```
Expected: `done: ok=4992, err=0` (err が 0 でない場合は原因調査、再実行)

- [ ] **Step 5: 検証 (再度 dry-run 実行)**

Run:
```bash
cargo run --bin normalize_levels 2>&1 | tail -10
```
Expected:
- `total docs: 14857` (近い値)
- `needs update: 0`
- ※ `needs update: 0` が小文字表記が残っていないことの直接的な証拠

gcloud config を default に戻す:
```bash
gcloud config configurations activate default
```

- [ ] **Step 6: コミット**

Run:
```bash
cd /home/o9oem/workspace/mine/jlpt_base/jlpt-app-scripts
git add bin/99_normalize_levels.rs
git commit -m "feat(scripts): 99_normalize_levels.rs — level_name を大文字統一する one-shot"
```

---

## Phase 2: 重複レポート生成

### Task 8: 99_report_duplicates.rs

**Files:**
- Create: `jlpt-app-scripts/bin/99_report_duplicates.rs`

- [ ] **Step 1: ファイル作成**

Create `jlpt-app-scripts/bin/99_report_duplicates.rs`:

```rust
//! 本番 Firestore から全 questions を取得し、dedup_common::dedup_key でグルーピング。
//! tiebreaker (createTime 古い方 > sentence 長 > qid 辞書順) で残すレコードを決め、
//! `reports/duplicates.json` に削除候補を出力する (読み取り専用)。

use chrono::{DateTime, Utc};
use log::{error, info, warn};
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::env;
use std::fs;
use std::path::PathBuf;

mod utils;
#[path = "dedup_common.rs"]
mod dedup_common;

use crate::dedup_common::{dedup_key, prefer_keep_order, Candidate, SubLike};

/// Firestore REST API で createTime を取るためのドキュメントラッパー
#[derive(Deserialize, Debug)]
struct RestDocument {
    name: String,
    #[serde(default)]
    fields: serde_json::Value,
    #[serde(rename = "createTime")]
    create_time: String,
}

#[derive(Deserialize, Debug)]
struct RestListResponse {
    #[serde(default)]
    documents: Vec<RestDocument>,
    #[serde(rename = "nextPageToken", default)]
    next_page_token: Option<String>,
}

#[derive(Serialize)]
struct ReportRow {
    parent_id: String,
    sub_idx: usize,
    sentence: String,
    correct_value: String,
    create_time: String,
}

#[derive(Serialize)]
struct ReportGroup {
    dedup_key: String,
    keep: ReportRow,
    remove: Vec<ReportRow>,
}

#[derive(Serialize)]
struct Report {
    generated_at: String,
    source: String,
    total_parents: usize,
    total_sub_questions: usize,
    dedup_groups: usize,
    rows_in_dup_groups: usize,
    removable_subs: usize,
    groups: Vec<ReportGroup>,
}

#[tokio::main]
async fn main() {
    dotenv::dotenv().ok();
    utils::init_logger();

    let project_id = env::var("PROJECT_ID").expect("PROJECT_ID must be set");
    let token = get_access_token();

    info!("=== 99_report_duplicates ===");
    info!("Firestore REST API で questions を取得中...");

    let docs = fetch_all_questions(&project_id, &token).await;
    info!("取得完了: {} docs", docs.len());

    // 全 sub_question を dedup_key でグルーピング
    let mut total_subs = 0usize;
    let mut groups: HashMap<String, Vec<(Candidate, String, String)>> = HashMap::new();
    //                          key    (cand, sentence, correct_value)

    for doc in &docs {
        let fields = &doc.fields;
        let parent_id = extract_string(fields, "id").unwrap_or_else(|| {
            // fallback: name 末尾
            doc.name.rsplit('/').next().unwrap_or("").to_string()
        });
        let level_id = extract_int(fields, "level_id").unwrap_or(0) as u32;
        let create_time: DateTime<Utc> = doc.create_time.parse().unwrap_or_else(|_| Utc::now());

        let subs_arr = extract_array(fields, "sub_questions");
        for (idx, sub_mv) in subs_arr.iter().enumerate() {
            total_subs += 1;
            let sub_fields = sub_mv.get("mapValue")
                .and_then(|m| m.get("fields"))
                .cloned()
                .unwrap_or(serde_json::Value::Null);
            let sentence = extract_string(&sub_fields, "sentence").unwrap_or_default();
            let answer = extract_string(&sub_fields, "answer").unwrap_or_default();
            let options = extract_options(&sub_fields);

            let sub_like = SubLike {
                options: options.clone(),
                answer: answer.clone(),
            };
            let Some(key) = dedup_key(level_id, &sub_like) else { continue };

            let correct_value = options.iter()
                .find(|(k, _)| k == &answer)
                .map(|(_, v)| v.clone())
                .unwrap_or_default();

            let cand = Candidate {
                parent_id: parent_id.clone(),
                sub_idx: idx,
                create_time,
                sentence_len: sentence.chars().count(),
            };
            groups.entry(key).or_insert_with(Vec::new).push((cand, sentence, correct_value));
        }
    }

    // 重複グループのみ抽出
    let mut dup_groups_out: Vec<ReportGroup> = Vec::new();
    let mut rows_in_dup_groups = 0usize;
    let mut removable = 0usize;

    for (key, mut rows) in groups {
        if rows.len() <= 1 { continue; }
        rows.sort_by(|a, b| prefer_keep_order(&a.0, &b.0));
        let keep_tuple = rows.remove(0);
        let keep = ReportRow {
            parent_id: keep_tuple.0.parent_id,
            sub_idx: keep_tuple.0.sub_idx,
            sentence: keep_tuple.1,
            correct_value: keep_tuple.2,
            create_time: keep_tuple.0.create_time.to_rfc3339(),
        };
        let remove_rows: Vec<ReportRow> = rows.into_iter().map(|(cand, sent, cv)| ReportRow {
            parent_id: cand.parent_id,
            sub_idx: cand.sub_idx,
            sentence: sent,
            correct_value: cv,
            create_time: cand.create_time.to_rfc3339(),
        }).collect();
        rows_in_dup_groups += remove_rows.len() + 1;
        removable += remove_rows.len();
        dup_groups_out.push(ReportGroup { dedup_key: key, keep, remove: remove_rows });
    }

    // create_time 降順でソート (新しい重複ほど上位表示、レビューしやすい)
    dup_groups_out.sort_by(|a, b| b.keep.create_time.cmp(&a.keep.create_time));

    let report = Report {
        generated_at: Utc::now().to_rfc3339(),
        source: format!("firestore:{}/(default)", project_id),
        total_parents: docs.len(),
        total_sub_questions: total_subs,
        dedup_groups: dup_groups_out.len(),
        rows_in_dup_groups,
        removable_subs: removable,
        groups: dup_groups_out,
    };

    // 出力
    let out_dir = PathBuf::from("reports");
    fs::create_dir_all(&out_dir).expect("failed to create reports dir");
    let out_path = out_dir.join("duplicates.json");
    let json = serde_json::to_string_pretty(&report).expect("serialize failed");
    fs::write(&out_path, json).expect("write failed");

    info!("=== Report ===");
    info!("total_parents:          {}", report.total_parents);
    info!("total_sub_questions:    {}", report.total_sub_questions);
    info!("dedup_groups:           {}", report.dedup_groups);
    info!("rows_in_dup_groups:     {}", report.rows_in_dup_groups);
    info!("removable_subs:         {}", report.removable_subs);
    info!("出力: {}", out_path.display());
}

fn get_access_token() -> String {
    let output = std::process::Command::new("gcloud")
        .args(["auth", "print-access-token"])
        .output()
        .expect("gcloud auth print-access-token failed");
    String::from_utf8(output.stdout).expect("invalid utf8").trim().to_string()
}

async fn fetch_all_questions(project_id: &str, token: &str) -> Vec<RestDocument> {
    let client = reqwest::Client::new();
    let mut out: Vec<RestDocument> = Vec::new();
    let mut page_token: Option<String> = None;
    loop {
        let mut url = format!(
            "https://firestore.googleapis.com/v1/projects/{}/databases/(default)/documents/questions?pageSize=300",
            project_id
        );
        if let Some(t) = &page_token {
            url.push_str(&format!("&pageToken={}", t));
        }
        let resp = match client.get(&url).bearer_auth(token).send().await {
            Ok(r) => r,
            Err(e) => { error!("fetch failed: {:?}", e); break; }
        };
        let body: RestListResponse = match resp.json().await {
            Ok(b) => b,
            Err(e) => { error!("json parse failed: {:?}", e); break; }
        };
        out.extend(body.documents);
        match body.next_page_token {
            Some(ref t) if !t.is_empty() => { page_token = Some(t.clone()); }
            _ => break,
        }
        info!("  取得中... {} docs", out.len());
    }
    out
}

fn extract_string(fields: &serde_json::Value, key: &str) -> Option<String> {
    fields.get(key)?.get("stringValue")?.as_str().map(|s| s.to_string())
}

fn extract_int(fields: &serde_json::Value, key: &str) -> Option<i64> {
    fields.get(key)?.get("integerValue")?.as_str()?.parse().ok()
}

fn extract_array(fields: &serde_json::Value, key: &str) -> Vec<serde_json::Value> {
    fields.get(key)
        .and_then(|v| v.get("arrayValue"))
        .and_then(|v| v.get("values"))
        .and_then(|v| v.as_array())
        .cloned()
        .unwrap_or_default()
}

fn extract_options(sub_fields: &serde_json::Value) -> Vec<(String, String)> {
    let values = extract_array(sub_fields, "select_answer");
    values.iter().filter_map(|opt| {
        let of = opt.get("mapValue")?.get("fields")?;
        let k = of.get("key")?.get("stringValue")?.as_str()?.to_string();
        let v = of.get("value")?.get("stringValue")?.as_str()?.to_string();
        Some((k, v))
    }).collect()
}
```

- [ ] **Step 2: ビルド確認**

Run:
```bash
cargo check --bin report_duplicates 2>&1 | tail -10
```
Expected: `Finished`

- [ ] **Step 3: 実行 (本番DB読み取りのみ)**

Run:
```bash
gcloud config configurations activate jlpt
export PROJECT_ID=argon-depth-446413-t0
export RUST_LOG=info
cargo run --bin report_duplicates 2>&1 | tail -20
```
Expected:
- `total_parents:          14857`
- `total_sub_questions:    21809` (近い値)
- `dedup_groups:           328` (近い値)
- `removable_subs:         547` (近い値)
- `出力: reports/duplicates.json`

- [ ] **Step 4: 出力レポートをユーザー目視レビュー**

Run:
```bash
# 先頭10グループをユーザーに表示
python3 -c "
import json
d = json.load(open('reports/duplicates.json'))
print(f'total groups: {len(d[\"groups\"])}')
print(f'removable_subs: {d[\"removable_subs\"]}')
print()
for g in d['groups'][:10]:
    print(f'--- {g[\"dedup_key\"]}')
    print(f'  KEEP: [{g[\"keep\"][\"parent_id\"][:8]}] {g[\"keep\"][\"sentence\"][:60]}  (correct={g[\"keep\"][\"correct_value\"]})')
    for r in g['remove'][:3]:
        print(f'  REMOVE: [{r[\"parent_id\"][:8]}] {r[\"sentence\"][:60]}  (correct={r[\"correct_value\"]})')
"
```
ユーザーが出力を確認し、サンプル群が本当に重複として妥当か判断する。判断に問題があれば計画を見直し。

- [ ] **Step 5: gcloud config を default に戻す**

Run:
```bash
gcloud config configurations activate default
```

- [ ] **Step 6: コミット**

Run:
```bash
cd /home/o9oem/workspace/mine/jlpt_base/jlpt-app-scripts
git add bin/99_report_duplicates.rs
git commit -m "feat(scripts): 99_report_duplicates.rs — 本番DB重複レポート生成 (読み取り専用)"
```

ただし `reports/duplicates.json` は `.gitignore` に追加して commit しない (大きく、本番データの生コピーのため)。

- [ ] **Step 7: .gitignore に reports/ を追加**

`jlpt-app-scripts/.gitignore` に `reports/` を追記 (すでにあれば skip)。

Run:
```bash
grep -q "^reports/" .gitignore || echo "reports/" >> .gitignore
git add .gitignore
git commit -m "chore: reports/ を .gitignore に追加"
```

---

## Phase 3: dedup 適用 (破壊的)

### Task 9: 99_apply_dedup.rs

**Files:**
- Create: `jlpt-app-scripts/bin/99_apply_dedup.rs`

- [ ] **Step 1: ファイル作成**

Create `jlpt-app-scripts/bin/99_apply_dedup.rs`:

```rust
//! `reports/duplicates.json` を入力として、重複 sub_question を Firestore から除去する。
//!
//! --dry-run (デフォルト): 操作ログのみ出力
//! --execute: 実際に Firestore を更新
//!
//! ロジック:
//!   1. report.groups から (parent_id, sub_idx) の削除集合を構築
//!   2. 対象 parent_id ごとに、Firestore から現行 Question を取得
//!   3. sub_questions から該当 sub を除外した配列で update
//!   4. sub_questions が空なら delete (user_answers 連鎖削除は別途 backend の既存 API を利用 or 省略)

use futures_util::StreamExt;
use log::{error, info, warn};
use serde::Deserialize;
use std::collections::HashMap;
use std::env;
use std::fs;

mod utils;

#[derive(Deserialize, Debug)]
struct ReportRow {
    parent_id: String,
    sub_idx: usize,
    #[serde(default)]
    sentence: String,
}

#[derive(Deserialize, Debug)]
struct ReportGroup {
    #[serde(default)]
    dedup_key: String,
    keep: ReportRow,
    remove: Vec<ReportRow>,
}

#[derive(Deserialize, Debug)]
struct Report {
    removable_subs: usize,
    groups: Vec<ReportGroup>,
}

#[tokio::main]
async fn main() {
    dotenv::dotenv().ok();
    utils::init_logger();

    let execute = env::args().any(|a| a == "--execute");
    let mode = if execute { "EXECUTE" } else { "DRY-RUN" };
    info!("=== 99_apply_dedup [{}] ===", mode);

    let report_path = env::args().skip(1).find(|a| !a.starts_with("--"))
        .unwrap_or_else(|| "reports/duplicates.json".to_string());
    info!("レポート読込: {}", report_path);

    let content = match fs::read_to_string(&report_path) {
        Ok(c) => c,
        Err(e) => {
            error!("レポート読込失敗: {:?}", e);
            return;
        }
    };
    let report: Report = match serde_json::from_str(&content) {
        Ok(r) => r,
        Err(e) => {
            error!("レポートJSON parse失敗: {:?}", e);
            return;
        }
    };
    info!("removable_subs (期待値): {}", report.removable_subs);

    // parent_id -> Vec<sub_idx to remove>
    let mut remove_map: HashMap<String, Vec<usize>> = HashMap::new();
    for g in &report.groups {
        for r in &g.remove {
            remove_map.entry(r.parent_id.clone()).or_insert_with(Vec::new).push(r.sub_idx);
        }
    }
    info!("対象 parent 数: {}", remove_map.len());

    let project_id = env::var("PROJECT_ID").expect("PROJECT_ID must be set");
    let db = match firestore::FirestoreDb::new(&project_id).await {
        Ok(d) => d,
        Err(e) => { error!("FirestoreDb::new failed: {:?}", e); return; }
    };

    let mut updated = 0usize;
    let mut parents_deleted = 0usize;
    let mut subs_removed = 0usize;
    let mut errors: Vec<(String, String)> = Vec::new();

    for (parent_id, sub_indices_to_remove) in &remove_map {
        // 現行 Question を取得
        let doc = match db.fluent()
            .select()
            .by_id_in("questions")
            .obj::<serde_json::Value>()
            .one(parent_id)
            .await
        {
            Ok(Some(d)) => d,
            Ok(None) => {
                warn!("parent not found: {}", parent_id);
                continue;
            }
            Err(e) => {
                error!("fetch failed {}: {:?}", parent_id, e);
                errors.push((parent_id.clone(), format!("{:?}", e)));
                continue;
            }
        };

        // sub_questions を抽出
        let subs = doc.get("sub_questions").and_then(|v| v.as_array()).cloned().unwrap_or_default();

        // 残す sub を構築 (sub_idx を除外)
        let remove_set: std::collections::HashSet<usize> = sub_indices_to_remove.iter().cloned().collect();
        let kept: Vec<serde_json::Value> = subs.iter().enumerate()
            .filter(|(i, _)| !remove_set.contains(i))
            .map(|(_, s)| s.clone())
            .collect();

        let removed_count = subs.len() - kept.len();
        if removed_count == 0 {
            warn!("parent {}: 対象 sub_idx がすでに存在しない (report 過時か?)", parent_id);
            continue;
        }
        subs_removed += removed_count;

        if !execute {
            if kept.is_empty() {
                info!("[dry-run] would DELETE parent={} (all {} subs)", parent_id, subs.len());
            } else {
                info!("[dry-run] would UPDATE parent={} ({} → {} subs)", parent_id, subs.len(), kept.len());
            }
            continue;
        }

        // 実行
        if kept.is_empty() {
            match db.fluent().delete().from("questions").document_id(parent_id).execute().await {
                Ok(_) => { parents_deleted += 1; }
                Err(e) => { error!("delete {}: {:?}", parent_id, e); errors.push((parent_id.clone(), format!("{:?}", e))); }
            }
        } else {
            // 現行ドキュメントの sub_questions を置き換えた object で全体を書き戻す
            // firestore 0.44 の fluent API に `.fields([...])` 相当が確実に無いため、
            // 既存 backend パターン (full object replace) を踏襲する
            let mut new_doc = doc.clone();
            if let Some(obj) = new_doc.as_object_mut() {
                obj.insert("sub_questions".to_string(), serde_json::Value::Array(kept));
            }
            match db.fluent()
                .update()
                .in_col("questions")
                .document_id(parent_id)
                .object(&new_doc)
                .execute::<serde_json::Value>()
                .await
            {
                Ok(_) => { updated += 1; }
                Err(e) => { error!("update {}: {:?}", parent_id, e); errors.push((parent_id.clone(), format!("{:?}", e))); }
            }
        }
        if (updated + parents_deleted) % 50 == 0 {
            info!("進捗: updated={}, deleted={}, subs_removed={}", updated, parents_deleted, subs_removed);
        }
    }

    info!("=== Summary ===");
    info!("subs_removed (計画): {}", subs_removed);
    info!("updated parents:      {}", updated);
    info!("deleted parents:      {}", parents_deleted);
    info!("errors:               {}", errors.len());

    if !errors.is_empty() {
        let err_path = "reports/apply_dedup_errors.json";
        let _ = fs::write(err_path, serde_json::to_string_pretty(&errors).unwrap());
        warn!("エラー詳細: {}", err_path);
    }

    if !execute {
        info!("(dry-run: 変更は適用されていません。--execute で実行)");
    }
}
```

- [ ] **Step 2: ビルド確認**

Run:
```bash
cargo check --bin apply_dedup 2>&1 | tail -10
```
Expected: `Finished`

- [ ] **Step 3: dry-run 実行 (本番DB読み取りのみ)**

Run:
```bash
gcloud config configurations activate jlpt
export PROJECT_ID=argon-depth-446413-t0
export RUST_LOG=info
cargo run --bin apply_dedup 2>&1 | tee /tmp/apply_dedup_dryrun.log | tail -40
```
Expected:
- `対象 parent 数: 約500`
- `[dry-run] would UPDATE parent=...` または `would DELETE parent=...` のログが並ぶ
- `subs_removed (計画): 547` (report の removable_subs と一致)

- [ ] **Step 4: ユーザー承認**

dry-run の `subs_removed` が report と一致し、`errors: 0` であることをユーザーに確認する。

- [ ] **Step 5: --execute で実行**

Run:
```bash
cargo run --bin apply_dedup -- --execute 2>&1 | tee /tmp/apply_dedup_exec.log | tail -40
```
Expected:
- `updated parents:` + `deleted parents:` の合計 ≒ 対象 parent 数
- `subs_removed (計画): 547`
- `errors: 0` (エラーあれば reports/apply_dedup_errors.json を確認しリトライ)

- [ ] **Step 6: 検証 (削除後の総数)**

Run:
```bash
# 削除後 sub_questions 総数 = 21809 - 547 を期待
cargo run --bin report_duplicates 2>&1 | grep "total_sub_questions\|removable_subs"
```
Expected:
- `total_sub_questions: 21262` (= 21809 - 547 付近)
- `removable_subs: 0` (または極少、もし残るなら race condition 等)

- [ ] **Step 7: gcloud config を default に戻す**

Run:
```bash
gcloud config configurations activate default
```

- [ ] **Step 8: コミット**

Run:
```bash
cd /home/o9oem/workspace/mine/jlpt_base/jlpt-app-scripts
git add bin/99_apply_dedup.rs
git commit -m "feat(scripts): 99_apply_dedup.rs — レポート適用で重複 sub_question を除去"
```

---

## Phase 5: ドキュメント更新

### Task 10: scripts/docs/pipeline.md と jlpt_base/docs/gcp-environment.md を更新

**Files:**
- Modify: `jlpt-app-scripts/docs/pipeline.md`
- Modify: `jlpt_base/docs/gcp-environment.md` (親リポジトリ側)

- [ ] **Step 1: scripts/docs/pipeline.md の dedup ステージ記述を更新**

`jlpt-app-scripts/docs/pipeline.md` を開き、`2_duplicate` に関する章 (現状の記述) を以下の内容で置き換える (章のタイトルは保持):

```markdown
## Stage 2: 重複排除 (`bin/2_duplicate.rs`)

### 目的
同一 level 内で「選択肢セット + 正解値」が重複する sub_question を除去する。

### dedup キー
```
L{level_id}|OPT[{sorted_normalized_values}]|ANS[{normalized_answer_value}]
```

- 選択肢値は Unicode NFKC 正規化 + 前後空白 trim + sort 済み
- 選択肢値がすべて `"1","2","3","4"` の場合は並び替え問題とみなし除外
- 正解キーに対応する値が見つからない場合も除外 (不正データ)

### 過去の実装との差分
旧 dedup_key は `sentence + "||" + correct_value` で選択肢を見ていなかった。
新 key は選択肢セット込みのため、文のバリエーションを変えただけの類問を検出できる。

### 共通ヘルパー
`bin/dedup_common.rs` に `normalize_text`, `dedup_key`, `prefer_keep_order`, `Candidate`, `SubLike` を配置。
`#[path = "dedup_common.rs"] mod dedup_common;` で各バイナリから参照。
テストは `tests/dedup_common_tests.rs`。
```

- [ ] **Step 2: jlpt_base/docs/gcp-environment.md に既存データ掃除の記録を追加**

`docs/gcp-environment.md` (親リポジトリ) の末尾に章を追加:

```markdown
## 2026-04-19: 既存データ重複対策

本番 Firestore `questions` コレクションの重複を以下の手順で掃除:

1. バックアップ: `gs://argon-depth-446413-t0-firestore-backup/backup-YYYYMMDD-HHMMSS`
2. level_name 大文字統一: 4,992件 ("n1"〜"n5" → "N1"〜"N5")
3. dedup: 選択肢セット+正解値+level_id キーで **約547 sub_questions** を除去
   (328 重複グループ)
4. パイプライン再発防止: `bin/2_duplicate.rs` の dedup_key を選択肢込みに改修

詳細設計: `docs/superpowers/specs/2026-04-19-duplicate-cleanup-design.md`
実装計画: `docs/superpowers/plans/2026-04-19-duplicate-cleanup.md`

### 残課題 (別Issue化)
- N5 生成プロンプト改善 (削除候補の46%が N5)
- category=9 並び替え問題の key==value バグ (21件)
- category=7 に紛れた不正フォーマット (qid=b1f3dcbe, 5件)
- 管理画面UI (重複監視)
```

- [ ] **Step 3: 両リポジトリでコミット**

Run (scripts 側):
```bash
cd /home/o9oem/workspace/mine/jlpt_base/jlpt-app-scripts
git add docs/pipeline.md
git commit -m "docs: pipeline.md に新 dedup ロジックを反映"
```

Run (親リポジトリ側):
```bash
cd /home/o9oem/workspace/mine/jlpt_base
git add docs/gcp-environment.md
git commit -m "docs: 2026-04-19 重複対策の作業記録を gcp-environment に追記"
```

---

## 実装順序まとめ

| Task | Phase | 内容 | 副作用 |
|------|-------|------|--------|
| 1 | 0 | ブランチ作成 + Firestore Export | リモートのみ |
| 2 | 4 | Cargo.toml 依存追加 + テスト雛形 | ローカル |
| 3 | 4 | normalize_text (TDD) | ローカル |
| 4 | 4 | dedup_key (TDD) | ローカル |
| 5 | 4 | prefer_keep_order (TDD) | ローカル |
| 6 | 4 | 2_duplicate.rs 改修 | ローカル |
| 7 | 1 | level_name 正規化 one-shot | **本番DB 4,992件更新** |
| 8 | 2 | レポート生成 | 本番DB読み取りのみ |
| 9 | 3 | dedup 適用 | **本番DB 547 sub 削除** |
| 10 | 5 | ドキュメント更新 | ローカル |

## 成功基準 (K2)

- Task 3-5: `cargo test --test dedup_common_tests` が 16 テスト PASS
- Task 6: `cargo check --bin duplicate` が PASS
- Task 7 実行後: `level_name="n*"` の件数が 0
- Task 8 実行後: `reports/duplicates.json` が生成され、`removable_subs ≈ 547`
- Task 9 実行後: `total_sub_questions` が実行前比で約 547 減少、`errors: 0`
- Task 10: 2 リポジトリの docs コミット

## リスクへの備え

- **Task 7/9 の途中失敗**: errors リスト (`reports/apply_dedup_errors.json`) を確認、該当 parent をリトライ
- **予期しない dedup 結果**: report の `removable_subs` が 400 未満 or 700 超なら実行を中止、原因調査
- **Firestore rate limit**: 既定 500 writes/sec。スクリプトは逐次なので問題ないが、エラーログで `RESOURCE_EXHAUSTED` を検知したらスリープ挿入
- **ロールバック**: Task 1 の Export から `gcloud firestore import` で復元可能
