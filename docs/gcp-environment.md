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
     ├── jlpt.howlrs.net ──→ [Cloud Run] frontend
     │                        Next.js 16 (App Router)
     │                        asia-northeast1
     │
     └── api.jlpt.howlrs.net ──→ [Cloud Run] backend
                                  Rust Axum API
                                  asia-northeast1
                                        │
                                        └──→ [Firestore] (default)
                                              FIRESTORE_NATIVE / asia-northeast1
```

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
| 現行リビジョン | `backend-00019-69m` |
| イメージ | `asia-northeast1-docker.pkg.dev/argon-depth-446413-t0/cloud-run-source-deploy/backend` |

**環境変数:**

| 変数 | 値 |
|------|-----|
| `FRONTEND_URL` | `https://jlpt.howlrs.net` |
| `RUST_LOG` | `error` |
| `PROJECT_ID` | `argon-depth-446413-t0` |
| `JWT_SECRET` | (設定済み) |

### Cloud Run - frontend

| 項目 | 値 |
|------|-----|
| サービス名 | `frontend` |
| リージョン | `asia-northeast1` |
| 内容 | Next.js 16 (App Router) |
| サブドメイン | `jlpt.howlrs.net` |

### GCS - フロントエンド（旧・廃止）

| 項目 | 値 |
|------|-----|
| バケット名 | `gs://jlpt.howlrs.net/` |
| 状態 | 廃止済み（Cloud Runに移行完了） |

### Firestore

| 項目 | 値 |
|------|-----|
| データベース | `(default)` |
| タイプ | FIRESTORE_NATIVE |
| ロケーション | `asia-northeast1` |

**コレクション:**
- `questions` - JLPT問題データ（10,481親問題 / 2026-03-14時点）
  - ドキュメントID: UUID v4
  - フィールド: id, level_id, level_name, category_id, category_name, sentence, prerequisites, sub_questions[], generated_by
- `levels` - レベルマスタ（N1〜N5の5件）
  - ドキュメントID: level_id (1-5)
  - フィールド: id, name
- `categories` - カテゴリマスタ（reten=子問題数付き、94件）
  - ドキュメントID: `{level_id}_{category_id}`
  - フィールド: level_id, id, name, reten
- `categories_raw` - カテゴリ生データ（scripts投入用）
  - ドキュメントID: 自動UUID
  - フィールド: level_id, id, name
- `users` - ユーザーデータ
- `votes` - 問題評価データ

## 有効化済みAPI

| API | 用途 |
|-----|------|
| `run.googleapis.com` | Cloud Run |
| `firestore.googleapis.com` | Firestore |
| `storage.googleapis.com` | Cloud Storage |
| `cloudbuild.googleapis.com` | Cloud Build (Cloud Runデプロイ時) |
| `artifactregistry.googleapis.com` | Artifact Registry (Dockerイメージ) |
| `containerregistry.googleapis.com` | Container Registry |

## gcloud 構成

```bash
# jlpt構成の確認
gcloud config configurations list

# jlpt構成に切替
gcloud config configurations activate jlpt

# 確認
gcloud config list
```

## デプロイ手順

### バックエンド (Cloud Run)

```bash
gcloud config configurations activate jlpt
cd jlpt-app-backend
gcloud run deploy backend --source . --region=asia-northeast1
```

### フロントエンド (Cloud Run)

```bash
gcloud config configurations activate jlpt
cd jlpt-app-frontend
gcloud run deploy frontend --source . --region=asia-northeast1
```
