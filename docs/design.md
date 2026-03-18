# 基本設計書: UptimePulse (確定版)

## 1. プロジェクト方針
- **フェーズ**: MVP -> v1 -> v2 (段階的開発)
- **監視内容**: HTTP GETによる死活監視、1/5/10分間隔、メール/Webhook通知

## 2. 確定した技術構成
「安定性と信頼性」を最優先し、以下の構成で進める。

- **Framework**: **Next.js 15.5 (Stable)** / **React 19**
    - **選定理由**: 最新の Next.js 16 よりも動作が安定しており、React 19 の機能を活用した確実な開発が可能なため。
- **Database**: **PostgreSQL** (Prisma 6.x)
    - **選定理由**: Next.js との親和性が高く、完全な型安全性とリレーショナルデータベースとしての信頼性を両立できるため。
- **Styling**: **Tailwind CSS v4**
    - **選定理由**: ビルド速度の大幅な向上と、CSS-first な設計による設定（`tailwind.config.js` 等）の簡略化が可能なため。
- **Coding Rules**: **Biome 1.x / 2.x**
    - **選定理由**: Rust 製の高速なツール（Lint/Format）の導入により、一貫したコーディングの質を担保するため。
- **Testing**: **Vitest 3.x**
    - **選定理由**: プロジェクトの要件に合わせたユニットテスト/統合テストを高速に実行するため。
- **Scheduler**: **Upstash Workflow (@upstash/workflow)**
    - **選定理由**: サーバーレス環境での 1 分周期監視の設定が非常に簡易であり、以下の根拠に基づき高い信頼性を持つため。
        - **状態の永続化 (Durable State)**: ステップごとの実行状態を保存し、タイムアウトやエラー後も中断箇所から再開可能。
        - **自動リトライ**: 指数バックオフによる自動再試行機能。
        - **QStash 基盤**: メッセージの「少なくとも 1 回 (At-least-once)」の配信保証。

## 3. アーキテクチャ
- **方式**: 共通キッカーによる1分周期パルス監視
- **フロー**: 
  1. Upstashから1分ごとにAPIをキック
  2. Next.jsがDBを確認し、対象モニターを実行
  3. 結果をDB保存し、異常があれば通知

## 4. ディレクトリ構成
```text
uptimepulse/
├── app/                # Next.js App Router
│   ├── auth/           # 認証関連のルート
│   ├── components/     # アプリ固有の共通コンポーネント (UI, Layout等)
│   ├── constants/      # アプリ内で使用する定数定義
│   └── schema/         # Zod 等のバリデーションスキーマ定義
├── lib/                # 共通ユーティリティ、Prisma クライアント等
├── prisma/             # Prisma Schema、マイグレーションファイル
└── docs/               # 設計・実装計画ドキュメント
```

## 5. データベース設計 (Prisma Schema 最終定義)

各テーブルの主要カラムを以下のように定義する。

### ユーザー・組織関連
- **User / Account** (Auth.js / NextAuth): GitHub & Google 連携をサポート。
    - 同一メールアドレスでの複数プロバイダー連携を許容し、ユーザーの利便性を向上。
- **Profile** (プロフィール):
    - `id`, `userId`, `displayName` (表示名), `image`, `role` (権限), `updatedAt`

### 監視・状態管理 (設定と動的な状態を分離)
- **Monitor** (監視設定):
    - `id`, `userId`, `createdId`, `name` (**サイト名/表示名**), `url`, `method` (GET/POST), `interval` (1/5/10), `expectedCode`, `shouldMatch` (レスポンス内に含まれるべき文字列 - オプション), `timeoutMs`
    - `@@index([userId])` / `@@index([status, lastCheckedAt])`
- **MonitorStatus** (現在のリアルタイム状況):
    - `id`, `monitorId`, `status` (UP, DOWN, DEGRADED, PAUSED)
    - `lastCheckedAt` (最終チェック日時)
    - `lastLatencyMs` (最新の応答速度), `lastStatusCode` (最新のステータスコード)

### 実行履歴・障害管理
- **Heartbeat** (全実行ログ):
    - `id`, `monitorId`, `statusCode`, `latencyMs`, `message` (エラー詳細など), `timestamp`
    - `@@index([monitorId, timestamp])`
- **Incident** (障害イベント):
    - `id`, `monitorId`, `status` (OPEN, RESOLVED)
    - `startedAt`, `resolvedAt`, `cause` (原因メモ)

### 通知設定
- **NotificationChannel** (通知先):
    - `id`, `userId`, `name`, `type` (EMAIL, SITE), `config` (JSON形式の設定)
- **MonitorNotification** (紐付け):
    - `monitorId`, `channelId` (どの監視の結果をどこに送るかの設定)

## 6. テスト・運用方針

監視サービスの信頼性を担保するため、以下の体制で品質管理を行う。

- **ユニットテスト**: **Vitest**
    - **選定理由**: 死活判定や通知の「計算ロジック」が正しいかを高速に検証するため。
- **統合テスト**: **Playwright** (v1〜)
    - **選定理由**: ユーザーの操作フロー（ダッシュボード面）や公開ページの表示が正常であることを、実際のブラウザ挙動で検証するため。
- **CIでの自動実行**: **GitHub Actions** (v1〜)
    - **選定理由**: コードのプッシュ時にテストを自動実行し、デプロイ前に障害のリスク（デグレード）を最小限に抑えるため。
- **品質基準**: **Biome** による規約遵守、**Vitest** によるカバレッジ管理。
    - **選定理由**: 自動化された整形と解析により、属人性を排除した一貫性のあるクリーンなコードベースを維持するため。

## 7. 共通要件・環境変数

プロジェクトの実行に必要な主要な環境変数を定義する。

- `DATABASE_URL`: PostgreSQL 接続用。
- `AUTH_SECRET`: Auth.js 用のシークレットキー。
- `AUTH_GITHUB_ID` / `AUTH_GITHUB_SECRET`: GitHub 認証用。
- `AUTH_GOOGLE_ID` / `AUTH_GOOGLE_SECRET`: Google 認証用。
- `UPSTASH_WORKFLOW_URL`: スケジューラーからのキックを受ける API エンドポイント。
- `UPSTASH_TOKEN`: Upstash 連携認証用。

## 8. デザインシステム（暫定）

ステータスページやダッシュボードの一貫性を保つための基本要素。

- **配色（セマンティックカラー）**:
    - **Operational (正常)**: `emerald-500` (緑)
    - **Degraded (遅延/一部障害)**: `amber-500` (橙)
    - **Outage (完全停止)**: `rose-500` (赤)
    - **Maintenance (メンテ中)**: `blue-500` (青)
- **UI コンポーネント**: Tailwind CSS v4 をベースに、miniZenn を踏襲した清潔感のあるモダンなデザインを目指す。

## 9. 今後の展望 (Future Outlook / v2〜)

将来的な拡張性として以下の機能を検討している。

### 組織（マルチテナント）対応
複数人での監視共有や権限管理を可能にする。
- **Organization** (組織/テナント):
    - `id`, `name` (組織名), `slug` (URL用一意識別子), `createdAt`
- **Member** (メンバーシップ):
    - `id`, `orgId`, `userId`, `role` (OWNER, ADMIN, VIEW)
- **Monitor への展開**:
    - `userId` を `orgId` に拡張し、組織単位での監視管理を実現。
