# 🚋 trafic-radar (transit-watch) - AI 開発・連携指示書 (GEMINI.md)

本ファイルは、本プロジェクトにおける AI（Gemini）および開発者が一貫した設計・実装基準、ディレクトリ構成、および開発プロセスを遵守するための指示書です。本プロジェクトは、新規（グリーンフィールド）開発の「スプリント 0（環境構築・着手準備）」フェーズにあります。

---

## 1. プロジェクト概要とアーキテクチャ

`trafic-radar` は、公共交通機関のオープンデータ（GTFS等）を取り込み、遅延・運行情報を集約して通知する Web アプリケーションです。

### 技術スタック
*   **バックエンド**: Java 21 / Spring Boot 3.x / Gradle / Spring Web (REST API) / Spring Data JPA / Spring Security + JWT / Spring Scheduler / Flyway
*   **フロントエンド**: React + TypeScript / Vite / TanStack Query (サーバー状態/ポーリング) / Zustand (クライアント状態) / React Router / Tailwind CSS + shadcn/ui / MapLibre GL JS + react-map-gl (地図表示)
*   **非同期処理・キュー**: Spring Event (開発初期) / Amazon SQS (本番構成)
*   **データベース**: PostgreSQL (ローカル開発: Docker Compose, 本番: Amazon RDS)
*   **インフラ**: AWS (ECS Fargate, RDS PostgreSQL, SQS, SES, S3 + CloudFront, EventBridge) / IaC: AWS CDK (Java)
*   **CI/CD & テスト**: GitHub Actions / JUnit 5 + Mockito / Testcontainers (PostgreSQL) / WireMock (外部APIモック)

### ディレクトリ構成（予定）
```
transit-watch/ (リポジトリルート)
├── backend/                # Spring Boot アプリケーション (Java 21, Gradle)
├── frontend/               # React + TypeScript (Vite) アプリケーション
├── infra/                  # AWS CDK (Java) コード
├── docker-compose.yml      # ローカル開発環境 (PostgreSQLなど)
├── .github/workflows/      # GitHub Actions 定義
├── README.md               # プロジェクト基本ドキュメント
└── GEMINI.md               # 本開発指示書（本ファイル）
```

---

## 2. ビルド・起動・テストコマンド

プロジェクトの各コンポーネントにおける標準的なコマンド群です。開発を進める際は、各ディレクトリに移動して実行します。

### 2.1 データベース・関連サービス（Docker Compose）
開発用 DB の起動・終了はリポジトリルートで行います。
*   **起動**: `docker compose up -d`
*   **停止**: `docker compose down`

### 2.2 バックエンド (Java / Gradle)
ディレクトリ `/backend` にて実行します。
*   **ビルド・コンパイル**: `./gradlew build`
*   **起動 (Live Reload 等)**: `./gradlew bootRun`
*   **テスト実行**: `./gradlew test`

### 2.3 フロントエンド (React / TypeScript / npm)
ディレクトリ `/frontend` にて実行します。
*   **依存関係インストール**: `npm install`
*   **ローカル開発サーバー起動**: `npm run dev`
*   **ビルド**: `npm run build`
*   **コード品質検証 (Lint/TypeCheck)**: `npm run lint`

---

## 3. 開発規約と AI 実装ガイドライン

AI アシスタントがコード生成・リファクタリング・修正を行う際は、以下の規約に厳格に従ってください。

### 3.1 設計・実装原則
1.  **モノレポの分離境界の維持**: `/backend` と `/frontend` は完全に独立したプロジェクトとして扱ってください。お互いのパッケージ管理（`package.json` や `build.gradle`）やコードを混在させないでください。
2.  **型安全性の確保**: OpenAPI（Swagger）定義をバックエンドのスキーマとし、フロントエンド側は `openapi-typescript` を用いて型定義を自動生成してください。フロントエンドの API クライアントは自動生成された型を前提に実装します。
3.  **データベースマイグレーションの厳格化**: DB スキーマの変更は、すべて Flyway のマイグレーションスクリプト（`backend/src/main/resources/db/migration/`）として作成します。すでに適用されたマイグレーションファイルは絶対に直接編集せず、必ず新しいバージョンファイル（`V<日付/連番>__<説明>.sql`）を追加してください。
4.  **権限検証の一貫性**: 一般ユーザー向け機能（お気に入り路線の操作など）は、リクエスト元のユーザーIDと操作対象データの所有者が一致しているかを、必ず Service 層のビジネスロジックで検証してください（Controller 層での検証漏れを防ぐため）。
5.  **外部サービス連携の非同期・疎結合化**: 運行情報（GTFS-RT）の取得・パースと、ユーザーへの通知判定（SES, Slack / Charwork）は密結合にせず、SQS やイベントバス（Spring Event）を挟んだ非同期メッセージング設計を徹底してください。

### 3.2 テスト方針
1.  **単体テスト**: ビジネスロジック（特に遅延判定や GTFS パース処理）は、JUnit 5 と Mockito を用いて網羅的に単体テストを記述してください。
2.  **結合テスト (Database)**: JPA リポジトリや DB の制約（外部キー、インデックスなど）が絡む処理のテストには H2 等のインメモリ DB は使用せず、`Testcontainers` を用いて PostgreSQL 実コンテナに対するテストを記述してください。
3.  **外部 API のモック**: GTFS-RT 配信サーバーや Slack / Charwork API、Amazon SES との通信を検証する際は、`WireMock` を用いて HTTP リクエスト/レスポンスをモックし、オフラインかつ確実なテストを保証してください。

### 3.3 コーディングスタイル
*   **Java**: Java 21 の最新機能を活用し、Record クラスやパターンマッチング、Stream API を効果的に使用して記述します。コンストラクタインジェクションを原則とし、フィールドへの `@Autowired` は避けてください。
*   **TypeScript / React**: Functional Components + React Hooks を標準とします。Tailwind CSS および shadcn/ui を用いたクリーンでモダンな UI 設計、および TanStack Query (React Query) による効率的なポーリング・キャッシュ管理を行ってください。

---

## 4. スプリント 0（環境構築・着手準備）のロードマップ
現在未実装のスケルトン状態であるため、AI は次の初期タスクの実行またはサポートに専念してください。
1.  **`docker-compose.yml` の作成**: PostgreSQL の立ち上げ
2.  **`/backend` の雛形作成**: Spring Initializr 経由での依存関係整備
3.  **`/frontend` の雛形作成**: Vite を用いた React TS アプリ起動確認
4.  **GTFS 静的データの調査**: ダウンロードした ZIP 内 CSV（`stops.txt` など）のパース方法検討

本ファイルは開発の進展（テーブル設計の決定、各種共通ユーティリティの実装など）に合わせて、随時更新してください。
