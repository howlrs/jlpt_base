# GCP 環境構成

## プロジェクト情報

| 項目 | 値 |
|------|-----|
| プロジェクトID | `argon-depth-446413-t0` |
| アカウント | `sharebook.amazon@gmail.com` |
| gcloud構成名 | `jlpt` |
| リージョン | `asia-northeast1` (東京) |

## インフラ構成

```
[Cloudflare]  DNS/CDN
     │
     └── jlpt.howlrs.net ──→ [Cloud Run] frontend
                              Next.js 16 (App Router)
                              asia-northeast1
                                    │
                                    │ Next.js rewrites (/api/* → backend)
                                    ▼
                              [Cloud Run] backend
                              Rust Axum API
                              asia-northeast1
                                    │
                                    └──→ [Firestore] (default)
                                          FIRESTORE_NATIVE / asia-northeast1
```

**API通信方式:** フロントエンドのNext.js rewritesにより `/api/*` リクエストをバックエンドにプロキシ。ブラウザから見ると全て `jlpt.howlrs.net` への同一オリジンリクエストとなり、Cookieがファーストパーティとして正常に動作する。

## サービス詳細

### Cloud Run - backend

| 項目 | 値 |
|------|-----|
| サービス名 | `backend` |
| リージョン | `asia-northeast1` |
| URL | `https://backend-652691189545.asia-northeast1.run.app` |
| CPU | 1000m (1 vCPU) |
| メモリ | 128Mi |
| 最大インスタンス | 2 |
| スタートアップCPUブースト | 有効 |
| イメージ | `asia-northeast1-docker.pkg.dev/argon-depth-446413-t0/cloud-run-source-deploy/backend` |

**環境変数:**

| 変数 | 値 |
|------|-----|
| `FRONTEND_URL` | `https://jlpt.howlrs.net` |
| `RUST_LOG` | `error` |
| `PROJECT_ID` | `argon-depth-446413-t0` |
| `JWT_SECRET` | (設定済み) |
| `ADMIN_EMAILS` | (設定済み) |

### Cloud Run - frontend

| 項目 | 値 |
|------|-----|
| サービス名 | `frontend` |
| リージョン | `asia-northeast1` |
| 内容 | Next.js 16 (App Router) |
| ドメイン | `jlpt.howlrs.net` |
| API プロキシ | Next.js rewrites で `/api/*` → backend URL |

**環境変数:**

| 変数 | 値 |
|------|-----|
| `NEXT_PUBLIC_API_URL` | `https://backend-652691189545.asia-northeast1.run.app` |

### Firestore

| 項目 | 値 |
|------|-----|
| データベース | `(default)` |
| タイプ | FIRESTORE_NATIVE |
| ロケーション | `asia-northeast1` |

**コレクション:**
- `questions` - JLPT問題データ
  - ドキュメントID: UUID v4
  - フィールド: id, level_id, level_name, category_id, category_name, sentence, prerequisites, sub_questions[], generated_by
  - 複合インデックス: `level_id` (ASC) + `category_id` (ASC)
- `levels` - レベルマスタ（N1〜N5の5件）
  - ドキュメントID: level_id (1-5)
  - フィールド: id, name
- `categories` - カテゴリマスタ（reten=子問題数付き）
  - ドキュメントID: UUID
  - フィールド: level_id, id, name, reten
- `users` - ユーザーデータ
  - ドキュメントID: email
  - フィールド: id, user_id, email, password(hashed), ip, language, country, created_at
- `user_answers` - 不正解回答履歴
  - ドキュメントID: `{user_id}_{question_id}_{sub_question_id}`（決定的ID、重複防止）
  - フィールド: id, user_id, question_id, sub_question_id, level_id, category_name, selected_answer, correct_answer, is_correct, answered_at
  - 複合インデックス: `user_id` (ASC) + `answered_at` (DESC)
- `user_stats` - ユーザー別統計（レベル・カテゴリ別正答率）
  - ドキュメントID: user_id
- `votes` - 問題評価データ
  - フィールド: id, vote, where_to, parent_id, child_id, created_at

## セキュリティ構成

### 認証・認可

| 項目 | 内容 |
|------|------|
| 認証方式 | httpOnly Cookie (JWT) |
| Cookie属性 | `HttpOnly; Secure; SameSite=Lax; Path=/api; Max-Age=86400` |
| JWT有効期限 | 24時間 |
| パスワードハッシュ | Argon2id |
| 管理者判定 | `ADMIN_EMAILS` 環境変数（カンマ区切り） |

**認証エンドポイント:**
- `POST /api/signin` — ログイン + Cookie設定
- `GET /api/auth/me` — 認証状態確認
- `POST /api/auth/logout` — Cookie クリア

**認証フロー:**
1. ブラウザ → `jlpt.howlrs.net/api/signin` (同一オリジン)
2. Next.js rewrites → backend (Set-Cookie レスポンス)
3. ブラウザがCookieを `jlpt.howlrs.net` ドメインに保存（ファーストパーティ）
4. 以降のAPIリクエストで自動送信

### セキュリティ対策

| 対策 | 内容 |
|------|------|
| レート制限 | tower_governor + SmartIpKeyExtractor（signin/signup: 5バースト, evaluate: 10バースト） |
| セキュリティヘッダー | X-Frame-Options: DENY, X-Content-Type-Options: nosniff, HSTS |
| 入力バリデーション | メール形式、パスワード8〜128文字、vote enum型安全化 |
| ユーザー列挙防止 | 統一エラーメッセージ + ダミーArgon2比較によるタイミング均一化 |
| CORS | 単一オリジン + allow_credentials(true) |
| monitor.rs | JWT_SECRET を claim.rs の定数から参照（空シークレット脆弱性修正済み） |

## パフォーマンス

| エンドポイント | 応答時間 | 備考 |
|---------------|---------|------|
| `/api/public/health` | ~0.31s | ネットワーク往復のみ |
| `/api/meta` | ~0.44s | Firestore 2コレクション読み取り |
| `/api/level/*/categories/*/questions` | ~0.45-0.60s | Firestore複合インデックス使用 |
| 管理API群 | ~0.30s | ネットワーク往復のみ |

問題取得は複合インデックス（`level_id` + `category_id`）を活用し、レベル全問題取得からカテゴリ直接フィルタに最適化済み。

## 有効化済みAPI

| API | 用途 |
|-----|------|
| `run.googleapis.com` | Cloud Run |
| `firestore.googleapis.com` | Firestore |
| `storage.googleapis.com` | Cloud Storage |
| `cloudbuild.googleapis.com` | Cloud Build (Cloud Runデプロイ時) |
| `artifactregistry.googleapis.com` | Artifact Registry (Dockerイメージ) |

## デプロイ手順

各 `deploy.sh` にはプリデプロイチェック（ビルド検証・疎通確認）とポストデプロイヘルスチェックが組み込まれている。

### バックエンド (Cloud Run)

```bash
gcloud config configurations activate jlpt
cd jlpt-app-backend
./deploy.sh  # cargo check → デプロイ → /api/meta + /api/questions スモークテスト
```

### フロントエンド (Cloud Run)

```bash
gcloud config configurations activate jlpt
cd jlpt-app-frontend
./deploy.sh  # .env.local確認 → npm run build → バックエンド疎通 → デプロイ → ヘルスチェック
```

> **注意:** フロントエンドは `.env.local` に `NEXT_PUBLIC_API_URL` が必要。未設定の場合 `deploy.sh` がエラーで停止する。
