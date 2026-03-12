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
     ├── jlpt.howlrs.net ──→ [GCS] gs://jlpt.howlrs.net/
     │                        静的サイトホスティング (Vite React PWA)
     │
     └── backend API ──────→ [Cloud Run] backend
                              asia-northeast1
                              https://backend-652691189545.asia-northeast1.run.app
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

### GCS - フロントエンド

| 項目 | 値 |
|------|-----|
| バケット名 | `gs://jlpt.howlrs.net/` |
| ストレージクラス | STANDARD |
| メインページ | `index.html` |
| 404ページ | `index.html` (SPA対応) |

**バケット内容:**
```
index.html
icon.ico
sitemap.xml
assets/
  index-*.js    (11ファイル, Viteビルド出力)
  index-*.css
```

### Firestore

| 項目 | 値 |
|------|-----|
| データベース | `(default)` |
| タイプ | FIRESTORE_NATIVE |
| ロケーション | `asia-northeast1` |

**コレクション:**
- `questions` - JLPT問題データ
- `levels` - レベルマスタ
- `categories` - カテゴリマスタ
- `users` - ユーザーデータ
- `votes` - 問題評価データ
- `categories_raw` - カテゴリ生データ (scripts投入用)

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

### フロントエンド (GCS)

```bash
gcloud config configurations activate jlpt
# ビルド後
gsutil -m rsync -r -d dist/ gs://jlpt.howlrs.net/
```
