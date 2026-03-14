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
     │                        静的サイトホスティング
     │                        ※ Next.js 16 (Cloud Run) への移行予定
     │
     ├── api.jlpt.howlrs.net ──→ [Cloud Run] backend（予定）
     │                            バックエンドAPI専用サブドメイン
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

### Cloud Run - frontend（予定）

| 項目 | 値 |
|------|-----|
| サービス名 | `frontend`（予定） |
| リージョン | `asia-northeast1` |
| 内容 | Next.js 16 (App Router) のCloud Runホスティング |
| 移行元 | GCS (`gs://jlpt.howlrs.net/`) |
| サブドメイン | `jlpt.howlrs.net`（現行GCSから移行予定） |

**備考:** 現在はGCSで静的ホスティング（旧Vite版）しているが、Next.js 16版をCloud Runへ移行予定。バックエンドAPIは `api.jlpt.howlrs.net` サブドメインに分離予定。

### GCS - フロントエンド（現行）

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

### フロントエンド (GCS - 現行)

```bash
gcloud config configurations activate jlpt
# ビルド後
gsutil -m rsync -r -d dist/ gs://jlpt.howlrs.net/
```

### フロントエンド (Cloud Run - 予定)

```bash
gcloud config configurations activate jlpt
cd jlpt-app-frontend
gcloud run deploy frontend --source . --region=asia-northeast1
```
