# JLPT Base プロジェクトドキュメント

JLPT（日本語能力試験）対策学習アプリの統合ドキュメント。

## リポジトリ別ドキュメント

### [japanese-app](../japanese-app/docs/)
問題生成スクリプト（初期版）。Rust + Google Gemini APIでJLPT問題をJSON形式で自動生成する。
- [概要](../japanese-app/docs/overview.md)
- [アーキテクチャ](../japanese-app/docs/architecture.md)
- [セットアップ](../japanese-app/docs/setup.md)

### [jlpt-app-frontend](../jlpt-app-frontend/docs/)
フロントエンドアプリ。Next.js 16 (App Router) + Tailwind CSSでJLPT問題の学習UIを提供する。
- [ドキュメント](../jlpt-app-frontend/docs/README.md)
- Cloud Runでホスティング

### [jlpt-app-backend](../jlpt-app-backend/docs/)
バックエンドAPI。Rust Axum + Google Firestoreで問題配信・ユーザー認証を提供する。
- [概要](../jlpt-app-backend/docs/overview.md)
- [API仕様](../jlpt-app-backend/docs/api-spec.md)
- [データモデル](../jlpt-app-backend/docs/data-models.md)
- [セットアップ](../jlpt-app-backend/docs/setup.md)

### [jlpt-app-scripts](../jlpt-app-scripts/docs/)
データパイプライン（本番版）。AI問題生成から加工・DB投入までの全工程を管理する。
- [概要](../jlpt-app-scripts/docs/overview.md)
- [パイプライン仕様](../jlpt-app-scripts/docs/pipeline.md)
- [データ構造](../jlpt-app-scripts/docs/data-models.md)
- [プロンプト設計](../jlpt-app-scripts/docs/prompts.md)
- [セットアップ](../jlpt-app-scripts/docs/setup.md)

## システム全体像

```
┌─────────────────────┐     ┌──────────────────────┐
│   japanese-app      │     │  jlpt-app-scripts    │
│  (初期版スクリプト)    │     │ (本番パイプライン)     │
│                     │     │                      │
│  Gemini API         │     │  Gemini API          │
│    ↓                │     │    ↓                 │
│  JSON生成(N2,N3)    │     │  JSON生成(N1-N5)     │
│    ↓                │     │    ↓                 │
│  構造化パース        │     │  構造化→類似排除      │
│                     │     │    ↓                 │
│                     │     │  ID採番→レベリング    │
│                     │     │    ↓                 │
│                     │     │  Firestore投入       │
└─────────────────────┘     └──────────┬───────────┘
                                       │
                                       ▼
                  ┌──────────────────────────────────────┐
                  │           Firestore                   │
                  └───────┬──────────────────┬────────────┘
                          │                  │
                          ▼                  ▼
               ┌──────────────────┐  ┌──────────────────┐
               │ jlpt-app-backend │  │ jlpt-app-frontend│
               │ (API サーバー)    │  │ (Next.js 16)     │
               │                  │  │                  │
               │ Axum + Firestore │  │ Next.js+Tailwind │
               │   ↓              │  │   ↓              │
               │ 問題配信API       │  │ 学習UI           │
               │ ユーザー認証      │  │                  │
               │ 問題評価         │  │                  │
               │   ↓              │  │   ↓              │
               │ Cloud Run        │  │ Cloud Run        │
               │ (backend)        │  │ (frontend)       │
               └──────────────────┘  └──────────────────┘
                          │                  │
                          └────────┬─────────┘
                                   ▼
                          ┌──────────────────┐
                          │   Cloudflare     │
                          │  (DNS/CDN/SSL)   │
                          └──────────────────┘
                                   ▼
                        https://jlpt.howlrs.net/
```

## インフラ・環境

- [GCP環境構成](./gcp-environment.md) - プロジェクト情報、Cloud Run / GCS / Firestore の構成詳細、デプロイ手順

## 技術スタック共通

| 技術 | 用途 |
|------|------|
| **Rust (Edition 2024)** | バックエンド・スクリプトの言語 |
| **Next.js 16 (App Router)** | フロントエンドフレームワーク |
| **Google Firestore** | NoSQLデータベース |
| **Google Gemini API** | AI問題生成 |
| **Google Cloud Run** | バックエンド・フロントエンドホスティング |
| **Recharts** | データ可視化（分析・カバレッジ） |
| **Serde** | JSONシリアライゼーション |
| **Tokio** | 非同期ランタイム |
