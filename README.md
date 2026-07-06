# 🚋 trafic-radar（交通遅延・運行情報集約アプリ）

公共交通機関のオープンデータ（GTFS等）を取り込み、遅延・運行情報を集約して通知する Web アプリケーションです。
能開大 応用課程1年 有志チームによる就職活動向けポートフォリオプロジェクトです。

---

## 📌 概要

北海道の公共交通（JR北海道・路線バス等）は、天候や事故により遅延・運休が頻発しますが、情報が路線ごとに分散しており、
利用者がリアルタイムに全体像を把握しづらいという課題があります。

本プロジェクトは、公開されている運行データ（GTFS / GTFS-Realtime など）を定期的に取得・集約し、
利用者が登録した路線の遅延情報をダッシュボードで確認・通知として受け取れる仕組みを提供します。

交通インフラ領域への関心から着想したテーマであり、実際の業務システム開発（データ収集→非同期処理→通知→可視化）を
一通り経験できる設計を意識しています。

---

## ✨ 主な機能

- **運行情報の自動収集**：GTFS(静的)・GTFS-RT(動的)を定期取得し、遅延情報を蓄積
- **路線・区間の遅延ダッシュボード**：現在の遅延状況を地図・一覧形式で可視化
- **ユーザー登録路線の通知**：お気に入り路線に遅延が発生した際にメール／チャットで通知
- **認証・認可**：JWTベースのログイン、自分の登録路線のみ編集可能な権限設計
- **管理者機能**：データソースの管理、通知ルールの設定

---

## 🛠 技術スタック

### バックエンド
| 項目 | 技術 |
|---|---|
| 言語/フレームワーク | Java 21 + Spring Boot 3.x |
| ビルドツール | Gradle |
| Web | Spring Web（REST API） |
| DB操作 | Spring Data JPA |
| 認証 | Spring Security + JWT |
| 定期処理 | Spring Scheduler |
| マイグレーション | Flyway |

### 非同期処理
| 項目 | 技術 |
|---|---|
| 学習用の第一歩 | Spring Event（`ApplicationEventPublisher`） |
| 本格構成 | Amazon SQS（GTFS-RT取得→解析→通知判定→送信を分離） |

### データベース
| 項目 | 技術 |
|---|---|
| DB | PostgreSQL（余力があればPostGIS拡張で位置検索） |
| 開発環境 | Docker Compose |
| 本番環境 | Amazon RDS |

### 外部データ連携
| 項目 | 技術 |
|---|---|
| GTFS(静的) | 各バス会社配布のZIPを定期取得・パース |
| GTFS-RT(動的) | gtfs-realtime-bindings（Java用Protocol Buffersライブラリ） |
| JR北海道 | 公式運行情報の取得（構造化データ or 要調査のスクレイピング） |

### 通知機能
| 項目 | 技術 |
|---|---|
| メール | Amazon SES |
| チャットBot通知 | LINE Messaging API（LINE Notifyはサービス終了済みのため採用） |
| （余力あれば） | Firebase Cloud Messaging |

### フロントエンド
| 項目 | 技術 |
|---|---|
| フレームワーク | React + TypeScript |
| ビルドツール | Vite |
| サーバー状態管理 | TanStack Query（ポーリングでリアルタイム遅延情報取得） |
| クライアント状態管理 | Zustand |
| ルーティング | React Router |
| UI | Tailwind CSS + shadcn/ui |
| 地図表示 | MapLibre GL JS + react-map-gl |
| 型連携 | OpenAPI定義から openapi-typescript で型自動生成 |

### インフラ（AWS）
| 項目 | 技術 |
|---|---|
| APIサーバー | ECS(Fargate) |
| DB | RDS(PostgreSQL) |
| 非同期キュー | SQS |
| メール送信 | SES |
| フロント配信 | S3 + CloudFront |
| 定期実行 | EventBridge |
| IaC | AWS CDK（Java） |

### CI/CD・テスト
| 項目 | 技術 |
|---|---|
| CI/CD | GitHub Actions |
| 単体テスト | JUnit5 + Mockito |
| 結合テスト | Testcontainers（PostgreSQLコンテナ使用） |
| 外部APIモック | WireMock |

---

## 🏗 アーキテクチャ概要

```
[GTFS(静的ZIP)]        [GTFS-RT(Protocol Buffers)]      [JR北海道 運行情報]
      │                         │                              │
      └──────────────┬──────────┴──────────────────────────────┘
                      ▼
            [取得バッチ (Spring Scheduler)]
                      │
                      ▼
             [SQS] ─ 解析 → 通知判定 → 送信 を分離
                      │
                      ▼
          [Spring Boot API Server] ── [RDS(PostgreSQL)]
                      │  REST API (OpenAPI)
                      ▼
        [React + TypeScript SPA (Vite)]
                      │
        ┌─────────────┴─────────────┐
        ▼                           ▼
 [MapLibre 地図表示]         [TanStack Query ポーリング]

通知経路: SQS → 通知判定 →  SES(メール) / LINE Messaging API

本番想定: CloudFront → ALB → ECS(Fargate) → RDS
                              └→ S3(静的アセット) / SQS / EventBridge
```

---

## 📂 ディレクトリ構成（予定）

```
transit-watch/
├── backend/                # Spring Boot アプリケーション
│   ├── src/main/java/...
│   ├── src/main/resources/db/migration/  # Flyway マイグレーション
│   └── src/test/java/...
├── frontend/                # React + TypeScript (Vite) アプリケーション
│   ├── src/
│   └── package.json
├── infra/                   # AWS CDK (Java) コード
├── docker-compose.yml       # ローカル開発環境
├── .github/workflows/       # GitHub Actions 定義
└── README.md
```

---

## 🏁 まずやることリスト（スプリント0：着手準備）

設計より先に「本物のデータの形」を知っておくことが重要なため、DB設計より前にGTFSデータに触れる工程を先頭に置いています。

### 1. 環境構築
- [ ] JDK 21 インストール
- [ ] IntelliJ IDEA（or 使い慣れたIDE）セットアップ
- [ ] Docker Desktop インストール
- [ ] Node.js（LTS版）インストール（React用）
- [ ] GitHubリポジトリ作成（backend/frontendはmonorepoでも別リポジトリでもOK、まずはmonorepoが管理楽）

### 2. GTFSデータを実際に触ってみる
- [ ] 北海道中央バス or 道北バス or 拓殖バスのGTFS(静的)データをダウンロード
- [ ] ZIPを解凍してCSVの中身を確認（`stops.txt`, `routes.txt`, `trips.txt`, `stop_times.txt` など）
- [ ] GTFS-RT(動的)データも実際に1回取得してみる（Protocol Buffers形式なのでそのままでは読めないことを体感する）
- [ ] どのデータを自分のDBに落とし込むか、ざっくりメモする

### 3. Spring Bootプロジェクトの雛形作成
- [ ] Spring Initializr（https://start.spring.io）で雛形生成
  - 依存関係：Spring Web, Spring Data JPA, PostgreSQL Driver, Flyway, Spring Security, Validation
- [ ] ローカルで `./gradlew bootRun` して起動確認だけする（中身は空でOK）

### 4. Docker Compose環境作成
- [ ] PostgreSQLコンテナを立てる `docker-compose.yml` を書く
- [ ] Spring BootからそのPostgreSQLに接続できることを確認

### 5. Reactプロジェクトの雛形作成
- [ ] `npm create vite@latest` でReact+TypeScriptの雛形作成
- [ ] Tailwind CSSのセットアップ
- [ ] 適当なページを1つ表示してみて動作確認

### 6. プロダクトバックログ作成
- [ ] GitHub Projectsでカンバンボード作成
- [ ] ユーザーストーリーをIssueとして書き出す（主な機能の4項目をベースに）

### 7. ER図の設計
- [ ] 路線・停留所・便・遅延ログ・ユーザー・お気に入りのテーブルをざっくり書く
- [ ] draw.io等で図にする

---

## 🚀 セットアップ手順（ローカル開発）

### 前提
- Docker / Docker Compose
- Java 21
- Node.js（LTS）

### 手順
```bash
# リポジトリをクローン
git clone https://github.com/<org>/transit-watch.git
cd transit-watch

# DB・関連サービスをコンテナで起動
docker compose up -d

# バックエンド起動（Flywayマイグレーションは起動時に自動適用）
cd backend
./gradlew bootRun

# フロントエンド起動（別ターミナル）
cd frontend
npm install
npm run dev
```

---

## 👥 開発の進め方

- タスク管理：GitHub Issues + Projects
- ブランチ運用：`main` / `develop` / `feature/*`（Git Flow 簡易版）
- レビュー：Pull Request 必須、1名以上のレビュー承認後にマージ
- 進捗共有：週1回の定例（15〜30分）

---

## ✅ テスト戦略

- ビジネスロジックは JUnit5 + Mockito による単体テストでカバー
- DB を含む結合テストは Testcontainers を用いて本番に近い環境で検証
- GTFS-RT や外部APIとの連携部分は WireMock でモックし、外部サービスに依存せずテスト
- GitHub Actions で push / PR 時に自動実行

---

## 📅 開発スケジュール（目安・2〜3ヶ月）

| 期間 | 内容 |
|---|---|
| 0週目 | スプリント0（環境構築・GTFSデータ調査・雛形作成・ER図設計） |
| 1〜2週目 | 要件定義、画面設計、環境構築の仕上げ |
| 3〜6週目 | コア機能実装（GTFS取得バッチ、認証(JWT)、CRUD、Flywayマイグレーション） |
| 7〜9週目 | 非同期処理（SQS連携）、通知機能(SES/LINE)、ダッシュボードUI(地図表示)磨き込み |
| 10週目〜 | CI/CD整備、README・デモ動画作成、就活準備 |

---

## 🔭 今後の展望

- PostGISを用いた高度な位置検索（近隣路線・駅の検索）
- Firebase Cloud Messagingによるプッシュ通知対応
- 遅延傾向の簡易分析（曜日・天候別の集計、ダッシュボードでの可視化）

---

## 👤 チームメンバー

| 氏名 | 担当 |
|---|---|
| （記入） | バックエンド |
| （記入） | フロントエンド |
| （記入） | インフラ / CI-CD |

---

## 📄 ライセンス

MIT License（予定）
