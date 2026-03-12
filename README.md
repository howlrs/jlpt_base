# JLPT Base

JLPT（日本語能力試験）[非公式] 対策学習アプリ

**本番URL:** https://jlpt.howlrs.net/
**紹介動画:** https://www.youtube.com/watch?v=I4o_v7d3yR8

## リポジトリ構成

本プロジェクトは役務・責務別に3つのリポジトリで構成される。

| リポジトリ | 役割 | 技術 | ドキュメント |
|-----------|------|------|-------------|
| [japanese-app](./japanese-app/) | 問題生成スクリプト（初期版） | Rust + Gemini API | [docs](./japanese-app/docs/) |
| [jlpt-app-backend](./jlpt-app-backend/) | バックエンドAPI | Rust Axum + Firestore | [docs](./jlpt-app-backend/docs/) |
| [jlpt-app-scripts](./jlpt-app-scripts/) | データパイプライン（本番版） | Rust + Gemini API + Firestore | [docs](./jlpt-app-scripts/docs/) |

## 各リポジトリの詳細

### japanese-app
Google Gemini APIを利用してJLPT問題をJSON形式で自動生成するスクリプト群。N2/N3レベルに対応した初期版。問題生成→ファイル結合→構造化パースの3段階パイプライン。

-> [概要](./japanese-app/docs/overview.md) | [アーキテクチャ](./japanese-app/docs/architecture.md) | [セットアップ](./japanese-app/docs/setup.md)

### jlpt-app-backend
Rust AxumフレームワークとGoogle Firestoreで構築されたREST API。問題配信（レベル別・カテゴリ別）、JWT認証、問題評価機能を提供する。Google Cloud Runにデプロイ。

-> [概要](./jlpt-app-backend/docs/overview.md) | [API仕様](./jlpt-app-backend/docs/api-spec.md) | [データモデル](./jlpt-app-backend/docs/data-models.md) | [セットアップ](./jlpt-app-backend/docs/setup.md)

### jlpt-app-scripts
N1〜N5全レベル対応の本番データパイプライン。AI問題生成（1000リクエスト/レベル）→構造化→重複排除→ID採番→レベル正規化→カテゴリ抽出→Firestore投入の全工程を管理する。

-> [概要](./jlpt-app-scripts/docs/overview.md) | [パイプライン](./jlpt-app-scripts/docs/pipeline.md) | [データ構造](./jlpt-app-scripts/docs/data-models.md) | [プロンプト設計](./jlpt-app-scripts/docs/prompts.md) | [セットアップ](./jlpt-app-scripts/docs/setup.md)

## 統合ドキュメント

詳細は [docs/](./docs/) を参照。
