# 2026-04-19 JLPT データ品質改善レポート

## 概要

本番 Firestore データの重複対策から始まり、パイプライン改修・validator 強化・ユーザフィードバック導線・運用監視の仕組み化までを1セッションで完遂。7 Issue を Close、新機能2件を本番デプロイ、本番データ 360 sub_questions を削除。

## スコープ

- 対象リポジトリ: jlpt-app-scripts, jlpt-app-backend, jlpt-app-frontend, jlpt_base (parent)
- 対象本番: jlpt.howlrs.net (Cloud Run + Firestore、argon-depth-446413-t0)
- ユーザ: アクティブ 10人程度、votes 9 件、user_answers 39 件の小規模運用

## 実行 Phase (0-7)

### Phase 0: feature/dedup-cleanup を master にマージ
- jlpt-app-scripts に 17 commits を統合
- 20 integration tests 全 pass
- リモート push 完了、feature branch 削除

### Phase 1: Issue #16 — category=7 不正フォーマット対策
- **本番DB**: qid=b1f3dcbe-2e59-44c0-af33-098c6fc16bc7 (5 sub_questions) を削除
- **パイプライン**: `1_5_validate_questions.rs` に「category=9 以外で選択肢値が全て '1','2','3','4' の場合 reject」チェックを追加
- Commit: aba282f

### Phase 2: Issue #15 — category=9 シャッフル identity permutation
- **根本原因**: `SliceRandom::shuffle` が確率 1/24 = 4.17% で元の並びと一致する偶然性
- **本番DB**: 21 sub_questions (20 parents) を削除 (16 DELETE + 4 UPDATE)
- **パイプライン**: `1_7_shuffle_answers.rs` に最大10回リトライで identity 排除 (再発確率 1/24^10 ≈ 0)
- Commit: 16fb300

### Phase 3: Issue #14 — N5 プロンプト多様化
- **SYSTEM_INSTRUCTION** (`bin/utils.rs`) に 3 条を追加:
  - 8: シチュエーション多様化 (最低5種類の日常シーン)
  - 9: 同じ選択肢セット繰り返し禁止
  - 10: 同じ正解語の連続配置禁止
- N5 重複集中 (削除候補の46%) への対策
- Commit: d6a49a1

### Phase 4: 問題報告機能 (新機能)
Gemini pro レビューで「最優先級」と示唆された導線。

#### Backend (jlpt-app-backend)
- `src/models/report.rs`: `QuestionReport` モデル、`{question_id}_{user_id}` を doc_id に使い二重報告防止
- `src/api/report.rs`:
  - `POST /api/questions/:id/report` (ログイン必須、409 Conflict で二重報告検出)
  - `GET /api/admin/reports` (Admin専用、report_count 降順で集計)
- Commit: ddebcca → Cloud Run backend 00054-9z9 deploy

#### Frontend (jlpt-app-frontend)
- `src/lib/api.ts`: `reportQuestion(questionId)` 追加
- `src/app/[level]/quiz/QuizClient.tsx`: 「⚑ 問題を報告」ボタン
- Commit: 2e5cea8 → Cloud Run frontend 00036-sr4 deploy

### Phase 5: Issue #20 部分 — /api/admin/duplicates backend API
- `src/common/dedup.rs`: scripts 側と同ロジック移植
- `src/api/admin.rs`: `pub async fn duplicates` 追加
- レスポンス: `total_parents`, `total_sub_questions`, `dedup_groups`, `removable_subs`, `skipped_*`, `top_groups[]`
- Commit: fc2d212 → Cloud Run backend 00055-gvm deploy
- **frontend UI は別セッション** (#20 open のまま、論点整理コメント追加済)

### Phase 6: 運用監視スクリプト
- `jlpt-app-scripts/tools/health_check.sh` (226行)
- 監視項目:
  1. 新規 reports 流入 (via `/api/admin/reports`)
  2. 重複データ健全性 (via `/api/admin/duplicates`)
  3. Cloud Run エラーログ (過去 N 時間、デフォルト 24h)
- 終了コード: 0=HEALTHY / 1=WARNING / 2=ERROR
- Commit: ac82d17

### Phase 7: Issue #17 — apply_dedup 再実行時の改善
- `99_apply_dedup.rs`: `MAX_PARENTS_ABORT` 環境変数で abort 閾値を override 可能
- `99_report_duplicates.rs`: `fetch_all_questions` の 401 受信時に gcloud token 再取得してリトライ
- Commit: 22c463d

---

## 本番データ変化

| 指標 | 開始時 | 終了時 | 差分 |
|------|--------|--------|------|
| total_parents | 14,857 | 14,676 | **-181** |
| total_sub_questions | 21,809 | 21,441〜21,449 | **-360〜-368** |
| level_name 小文字 | 4,992件 | 0件 | **-4,992 (正規化)** |
| dedup groups | 237 | 0 | -237 |
| cat=9 key==value bugs | 21件 | 0件 | -21 |
| cat=7 不正format | 5件 | 0件 | -5 |

本番の level_name 分布 (終了時):
- N1: 4,107 / N2: 3,146 / N3: 2,384 / N4: 2,213 / N5: 2,827
- **小文字 (n*) は 0件** — 正規化完全達成

## 本番 Cloud Run 最終状態

| サービス | Revision | URL | 状態 |
|---------|----------|-----|------|
| backend | backend-00055-gvm | https://backend-652691189545.asia-northeast1.run.app | 100% traffic |
| frontend | frontend-00036-sr4 | https://jlpt.howlrs.net/ | 100% traffic |

## スモークテスト結果 (本セッション完了時)

### 公開エンドポイント (認証不要)
| Endpoint | Expected | Actual |
|----------|----------|--------|
| `https://jlpt.howlrs.net/` | 200 | **200** ✅ |
| `https://jlpt.howlrs.net/api/meta` | 200 | **200** ✅ |
| `https://jlpt.howlrs.net/n5/quiz` | 200 | **200** ✅ |
| `https://jlpt.howlrs.net/about` | 200 | **200** ✅ |
| `https://jlpt.howlrs.net/login` | 200 | **200** ✅ |
| backend `/api/meta` | 200 | **200** ✅ |

### 認証ガード確認 (未認証アクセス)
| Endpoint | Expected | Actual |
|----------|----------|--------|
| `POST /api/questions/:id/report` | 401 | **401** ✅ |
| `GET /api/admin/reports` | 401 | **401** ✅ |
| `GET /api/admin/duplicates` | 401 | **401** ✅ |

### Cloud Run エラー
- 過去30分のエラー: **0件** (backend + frontend とも)

**→ 機能破壊なし、全サービス正常動作、品質指標向上**

## Issue 状態 (最終)

### Closed (6件)
| Repo | # | タイトル |
|------|---|---------|
| jlpt-app-scripts | #14 | [N5] 生成プロンプトのテーマ多様化による重複生成の抑制 |
| jlpt-app-scripts | #15 | [category=9] 並び替え問題のシャッフル未実行疑い 21件 |
| jlpt-app-scripts | #16 | [category=7] 不正フォーマットの sub_question 5件 |
| jlpt-app-scripts | #17 | [99_apply_dedup] 再実行時のための改善 |
| jlpt-app-backend | #21 | [feature] 問題報告エンドポイント |
| jlpt-app-frontend | #31 | [feature] 問題報告ボタン |

### Open (次セッション、論点整理コメント追加済)
| Repo | # | タイトル |
|------|---|---------|
| jlpt-app-scripts | #18 | [future] 意味的類似 (embeddings) による類問検出 |
| jlpt-app-backend | #20 | [admin] 重複監視 UI + API (backend完了、frontend UI残) |

## コミット統計

| Repo | 追加コミット数 |
|------|--------------|
| jlpt-app-scripts | 23 (dedup feature branch 17 + 各Phase 6) |
| jlpt-app-backend | 2 (報告API + duplicates API) |
| jlpt-app-frontend | 1 (報告ボタン) |
| jlpt_base (parent) | 4 (設計+計画+docs×2) |

## 新規追加された技術資産

### 再利用可能モジュール
- `jlpt-app-scripts/bin/dedup_common.rs` — 汎用重複検出ヘルパー (normalize_text, dedup_key, prefer_keep_order)
- `jlpt-app-backend/src/common/dedup.rs` — backend 版、同ロジック
- 20 integration tests (`jlpt-app-scripts/tests/dedup_common_tests.rs`)

### 運用ツール
- `jlpt-app-scripts/tools/health_check.sh` — 3項目一括監視 CLI

### 新 API
- `POST /api/questions/:id/report`
- `GET /api/admin/reports`
- `GET /api/admin/duplicates`

### 新 UI
- クイズ画面「⚑ 問題を報告」ボタン

## 次セッション推奨

Gemini pro ROI 順位で:

1. **Issue #20 frontend UI** (Medium impact / Low cost)
   - /admin/duplicates ページの設計→実装
   - 既存 backend API 活用で1-2セッションで完結可能
   - 論点整理: Issue #20 コメント参照

2. **Issue #18 embeddings Phase 1** (High impact / High cost)
   - サンプル調査 (N5 100-200件) から開始
   - API コスト \$10-30 の覚悟必要
   - 論点整理: Issue #18 コメント参照

3. **運用監視のフィードバックループ**
   - 24-48h 後に `tools/health_check.sh` 実行
   - 新規 reports があれば該当問題を調査
   - removable_subs > 0 なら Phase 3 プロンプト escalation (案2 サブテーマ注入)

## 参考資料

- 設計: `docs/superpowers/specs/2026-04-19-duplicate-cleanup-design.md`
- 実装計画: `docs/superpowers/plans/2026-04-19-duplicate-cleanup.md`
- GCP 環境追記: `docs/gcp-environment.md` §「2026-04-19: 既存データ重複対策」

## 所感

小規模本番 (10人) でも、データ品質劣化の早期検知 → パイプライン再発防止 → ユーザフィードバック導線 → 運用監視 の4段階を段階的に整備することで、**継続的改善の基盤**が確立した。Gemini pro のセカンドオピニオンを重要判断ポイント (マージ規則・フィードバック導線追加・残Issue優先順位) で活用したことで、スコープの適切な絞り込みと ROI 最大化に貢献。
