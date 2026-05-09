# Unit of Work — Unit 定義と責務 (Inception / Units Generation ステージ成果物)

> **作成日**: 2026-05-09T03:30:00Z
> **対応ルール**: `.aidlc-rule-details/inception/units-generation.md` Step 12-15
> **基底ドキュメント**:
> - `aidlc-docs/inception/application-design/components.md` (C-01〜C-07 の責務定義)
> - `aidlc-docs/inception/application-design/services.md` (S-01〜S-06 のサービス層)
> - `aidlc-docs/inception/application-design/component-methods.md` (メソッドシグネチャ)
> - `aidlc-docs/inception/application-design/component-dependency.md` (依存マトリックス + デプロイ順序)
> **関連**: `unit-of-work-dependency.md` (Unit 間依存) / `unit-of-work-story-map.md` (ストーリー → Unit マッピング)

---

## 命名規則と方針

- Unit ID: `Unit-N` (1 桁)
- **マッピング方針**: Application Design の **7 コンポーネント (C-01〜C-07)** を **7 Unit (Unit-1〜Unit-7) に 1:1 でマッピング**
- 用語 (units-generation.md ルール準拠): Service = 独立デプロイ可能 / Module = Service 内論理グループ

---

## Unit 一覧

| Unit ID | Unit 名 | 対応コンポーネント | 種別 | 主要 FR | 主担当ストーリー数 |
|---|---|---|---|---|---|
| **Unit-1** | Mobile Client Unit | C-01 | Mobile Service (iOS / Swift / SwiftUI) | FR-01, 02, 11, 12, 13, 14 (Mobile 側) | 12 (M:9 + S:3) |
| **Unit-2** | Dialogue API Unit | C-02 | Cloud Service (API Gateway + Lambda) | FR-05, 06, 07 | 4 (M:3 + S:1) |
| **Unit-3** | Risk Calculator Unit | C-03 | Module (Lambda 内純粋関数 / Unit-2 にバンドル) | FR-04 | 1 (M:1) |
| **Unit-4** | History & Title Service Unit | C-04 | Cloud Service (Lambda + DynamoDB) | FR-08, 09, 10 (動的判定), 14 (達成フラグ記録) | 4 (M:4) |
| **Unit-5** | External Client Unit | C-05 | Module (Lambda 内ライブラリ / Unit-2 にバンドル) | FR-03 | 0 (副担当のみ) |
| **Unit-6** | Infrastructure Unit | C-06 | IaC (CDK / CloudFormation / 横断) | NFR-DAT-03, 05 | 0 (横断) |
| **Unit-7** | Title Catalog Distribution Unit | C-07 | Static Service (S3 + CloudFront) | FR-10 (静的メタ配信) | 0 (副担当のみ) |

> 担当ストーリーの詳細マッピングは `unit-of-work-story-map.md` を参照。
> **主担当総和**: 12 + 4 + 1 + 4 + 0 + 0 + 0 = **21** ✓ で全 21 ストーリー (M:17, S:4) をカバー。Unit-5 / Unit-7 は副担当のみ / Unit-6 は横断。

---

## Unit-1: Mobile Client Unit

### 種別と配置
- **種別**: Mobile Service (端末ローカル実行 / iOS Native)
- **言語 / フレームワーク**: Swift / SwiftUI (Q4 = A 確定)
- **OS API 依存**: HealthKit / CoreLocation / EventKit / UNUserNotificationCenter

### 責務 (What)
- オンボーディング (US-5.1, US-5.5) / コンセプト明示 (R11 対策方針 (a))
- ヘルスケアデータ取得 (FR-01, 02 / `HealthDataAdapter`) / **HealthKit 生データの端末ローカル限定**
- 位置情報取得 (FR-03 一部 / `LocationDataAdapter`)
- カレンダーデータ取得 (FR-12 / `CalendarDataAdapter` / EventKit)
- 翌日予定ホーム画面ミニ表示 (FR-13 / iOS 標準カレンダーへの遷移)
- Dialogue API 呼び出し / 対話表示 (4-6 ターン / 悪魔最後)
- 選択 (入る/サボる) 送信 (FR-08)
- 履歴カレンダー表示 (US-3.2 / 中立配色)
- 称号表示 (FR-10 / Unit-7 catalog 結合)
- 設定画面 (US-5.3) / コンセプト再表示 (R11 対策方針 (a) 再表示経路)
- 30 分後の達成確認通知スケジュール (FR-14 / `NotificationScheduler` / iOS UNUserNotificationCenter / `choice='bath'` のみ)
- ダラけ感のある UI 演出 (FR-11 / NFR-USA-03)

### 機微データ境界 (NFR-DAT-02 / R3 / SECURITY-13)
- HealthKit 生データ + EventKit 生情報 (タイトル/場所/参加者) + CoreLocation の永続データは **端末ローカル限定**
- AWS への送信は集計値 (HealthSummary / CalendarSummary) のみ
- 位置情報は対話生成リクエスト時のみ送信、永続保存しない

### Adapter パターン (DEBUG ビルド擬似データモード)
- Production (Release): HealthKitAdapter / CoreLocationAdapter / EventKitAdapter / LocalNotificationScheduler
- DEBUG ビルド: PseudoHealthAdapter / PseudoLocationAdapter / PseudoCalendarAdapter / LoggingNotificationScheduler
- Test: Mock 注入

### 外部 Unit との連携
- Unit-2 (Dialogue API) を `POST /dialogue` で呼び出し
- Unit-4 (History & Title Service) を `POST /selections` / `GET /history` / `GET /titles` / `POST /selections/{id}/achievement` で呼び出し
- Unit-7 (Title Catalog Distribution) を CloudFront URL で直接 GET (API Gateway 不経由)

### コード組織化 (Greenfield)
```
mobile-client/                  ← Unit-1 のリポジトリ
├── App/                        Application Core (UI / ViewModel / Onboarding / Calendar UI)
├── Adapters/                   HealthDataAdapter / LocationDataAdapter / CalendarDataAdapter / NotificationScheduler
│   ├── Production/             HealthKitAdapter / CoreLocationAdapter / EventKitAdapter / LocalNotificationScheduler
│   └── Pseudo/                 PseudoXXXAdapter (DEBUG ビルド差し替え可能)
├── ApiClient/                  DialogueAPIClient / HistoryAPIClient / TitleCatalogClient
├── Resources/                  キャラ画像 (Visual Asset Plan / Section 18) / フォント / 文言リソース
└── Tests/                      ユニットテスト (Mock 注入)
```

---

## Unit-2: Dialogue API Unit

### 種別と配置
- **種別**: Cloud Service (API Gateway REST + Lambda)
- **AWS リージョン**: ap-northeast-1
- **同梱 Module**: Unit-3 (Risk Calculator) / Unit-5 (External Client) を同一 Lambda にバンドル
- **使用モデル**: Bedrock Claude 3.5 Sonnet (R9 申請後)

### 責務 (What)
- 対話生成エンドポイント `POST /dialogue` (FR-05, 06)
- ヘルスケア要約 + 環境データ + カレンダーサマリ + 端末 UUID を受信
- Unit-5 の `fetchWeather()` で天気取得 (FR-03 / 失敗時は環境データなしで継続)
- Unit-3 の `calculateAnnoyanceRisk()` で迷惑リスク判定 (PBT 対象 / FR-04)
- Unit-4 の `getRecentSkipPattern()` + `getLastBathTime()` を Direct Invoke で並列取得 (FR-07 / DD-03)
- `buildPrompt()` で System + User Prompt 構築 (PBT 対象 / FR-05)
- 動的トーンシフト (US-2.4 Must / FR-07): `riskLevel='high'` または連続サボり長期化フラグで悪魔のトーンを健康寄りにシフト
- Bedrock Claude 呼び出し
- レスポンス整形 (4-6 ターン / 悪魔最後 / 責めない / `riskLevel` 付き)

### PBT 対象 (Partial 適用範囲) — PBT-02, 03, 07, 08, 09
- `buildPrompt(input)` (FR-05): 冪等性 / 機微情報除外 / characterSetId フォールバック / systemPromptVersion 常在
- 詳細プロパティ識別は Functional Design (per-Unit / PBT-01) で実施

### コード組織化 (Greenfield)
```
dialogue-lambda/                ← Unit-2 のリポジトリ
├── handler.ts                  POST /dialogue ハンドラ
├── prompt/                     buildPrompt() / プロンプトテンプレート
├── modules/                    Unit-3 (Risk Calculator) + Unit-5 (External Client) を内包
└── tests/                      PBT (PBT-02/03/07/08/09)
```

---

## Unit-3: Risk Calculator Unit

### 種別と配置
- **種別**: Module (Unit-2 Dialogue Lambda 内の純粋関数ライブラリ)
- **特性**: I/O 副作用なし

### 責務 (What)
- 迷惑リスク判定 (FR-04 / US-1.5): `calculateAnnoyanceRisk(input)`
- 入力: プロキシ指標 (`hoursSinceLastBath` / `health: HealthSummary` / `environment: EnvironmentData`)
- 出力: `AnnoyanceRiskFlag { level: 'low'|'medium'|'high', hoursSinceLastBath, movementScore, ..., hasMissingData }`
- `hoursSinceLastBath` は Unit-2 が Unit-4 `getLastBathTime()` 経由で取得した値を渡す (DD-03)

### PBT 対象 (Partial 適用範囲) — PBT-02, 03, 07, 08, 09
- 出力 `level` は `low | medium | high` のいずれか (全域性)
- 同一入力 → 同一出力 (冪等性)
- 全データ欠損時は `hasMissingData=true` で `level='low'` (安全側)
- `hoursSinceLastBath > 72` のときは `level >= 'medium'` (US-1.5 AC)
- 詳細プロパティ識別は Functional Design (per-Unit / PBT-01) で実施

### コード組織化 (Greenfield)
- Unit-2 のリポジトリ内 `modules/risk-calculator/` として配置 (独立ライブラリ)

---

## Unit-4: History & Title Service Unit

### 種別と配置
- **種別**: Cloud Service (Lambda + DynamoDB)
- **AWS リージョン**: ap-northeast-1
- **DynamoDB**: シングルテーブル / Server-Side Encryption (SSE) 有効 / Point-in-Time Recovery / META#AFFIRMATIONS パーティション含む

### 責務 (What)
- 選択記録エンドポイント `POST /selections` (FR-08): SelectionRecord 保存 + 称号付与判定 + Affirmation 取得 (DD-02)
- 履歴取得エンドポイント `GET /history` (FR-09): カレンダー UI 用日単位集計
- 獲得称号一覧エンドポイント `GET /titles` (FR-10): 獲得 ID + awardedAt のみ (静的メタは Unit-7)
- **達成フラグ記録エンドポイント** `POST /selections/{selectionId}/achievement` (FR-14): `achieved: bool` を SelectionRecord に Update / 元の `choice='bath'` は維持
- 称号付与判定 (PBT 対象 / FR-10): `evaluateNewTitles()` / 事前リスト 10+ 件のみから選択 (称号メタの源泉は Unit-7)
- Affirmation テンプレート参照 (DD-02): META#AFFIRMATIONS パーティションから choice 別ランダム選択 / `pickAffirmation()`
- 連続サボり長期化サマリ提供 (FR-07): `getRecentSkipPattern()` (Unit-2 から Direct Invoke)
- 最終 'bath' タイムスタンプ提供 (DD-03): `getLastBathTime()` (Unit-2 から Direct Invoke)

### PBT 対象 (Partial 適用範囲) — PBT-02, 03, 07, 08, 09
- `evaluateNewTitles(input)` (FR-10): 出力が事前リスト 10+ 件のみ / 重複付与禁止 / 冪等性
- 詳細プロパティ識別は Functional Design (per-Unit / PBT-01) で実施

### DynamoDB スキーマ (概要)
```
PK = <deviceUUID>  (ユーザーデータ)
SK = SELECTION#<ISO8601>#<random>  → SelectionRecord (achieved フィールド含む)
SK = TITLE#<titleId>                → AwardedTitle

PK = META#AFFIRMATIONS  (運用メタデータ / DD-02)
SK = AFFIRMATION#<choice>#<id>      → { judgeMessage, devilMessage, ... }
```

### コード組織化 (Greenfield)
```
history-title-lambda/           ← Unit-4 のリポジトリ
├── handler.ts                  POST /selections / GET /history / GET /titles / POST /selections/{id}/achievement ハンドラ
├── domain/                     evaluateNewTitles / pickAffirmation / getRecentSkipPattern / getLastBathTime / markAchievement
├── persistence/                DynamoDB アクセス (シングルテーブル)
└── tests/                      PBT (FR-10) + 統合テスト (DDB Local)
```

---

## Unit-5: External Client Unit

### 種別と配置
- **種別**: Module (Unit-2 Dialogue Lambda 内のクライアントライブラリ)
- **外部 API**: 無料枠のある天気 API (例: OpenWeatherMap)

### 責務 (What)
- 天気 API クライアント (FR-03 / US-1.3): `fetchWeather(latitude, longitude)`
- 出力: `EnvironmentData { tempC, weather, humidityPct, asOf } | null`
- リトライ (1 回まで) + タイムアウト (1 秒) + キャッシュ (同一座標 30 分以内 / R10 料金対策)
- 失敗時は `null` (Unit-2 が環境データなしで対話生成継続 / US-1.3 AC)

### コード組織化 (Greenfield)
- Unit-2 のリポジトリ内 `modules/external-client/` として配置

---

## Unit-6: Infrastructure Unit

### 種別と配置
- **種別**: IaC (横断 / アプリケーションロジックなし)
- **ツール**: CloudFormation または CDK (主催規約準拠)

### 責務 (What)
- AWS リソース定義 (NFR-DAT-03 / Technical Context):
  - **API Gateway**: REST API + 認可 (端末 UUID 軽量識別 / SECURITY-04) + スロットリング (R10)
  - **Lambda**: Unit-2 (Dialogue) と Unit-4 (History & Title) の関数定義 / IAM Role 最小権限 (SECURITY-05)
  - **DynamoDB**: シングルテーブル / SSE (SECURITY-07) / Point-in-Time Recovery / META#AFFIRMATIONS パーティション含む (DD-02)
  - **S3 + CloudFront** (Unit-7 用 / DD-01): Catalog Bucket (Versioning ON / SSE-S3) / Distribution (OAI / TLS 1.2+ / TTL 1h)
  - **IAM**: Bedrock InvokeModel 権限 / DynamoDB CRUD 権限 / S3 GetObject 権限 (CloudFront OAI 経由のみ) / 最小権限の原則
- 監視・ログ:
  - CloudWatch Logs (構造化 / SECURITY-13: 機微情報除外)
  - メトリクス + アラーム (R10 料金監視)
- シークレット管理 (SECURITY-09):
  - 天気 API キーは Secrets Manager
  - Bedrock はモデルアクセスを IAM で制御
- デプロイパイプライン (Bolt 1 で構築 / 概略): CDK Synth → CloudFormation Deploy

### Bolt 1 最優先タスク (R8 / R9 への対応)
- AWS アカウント準備 (R8 / 最優先)
- Bedrock Claude モデルアクセス申請 (R9 / 最優先 / 申請待ち中は他 Unit を並行実装)
- 詳細は `unit-of-work-dependency.md` Section「Bolt 1 デプロイ順序」参照

### コード組織化 (Greenfield)
```
infrastructure/                 ← Unit-6 のリポジトリ
├── lib/
│   ├── api-gateway-stack.ts
│   ├── lambda-stack.ts         (Unit-2 + Unit-4)
│   ├── dynamodb-stack.ts
│   ├── catalog-stack.ts        (Unit-7 用 S3 + CloudFront)
│   ├── monitoring-stack.ts     (CloudWatch + アラーム)
│   └── iam-stack.ts
├── bin/                        CDK エントリポイント
└── docs/
    └── catalog-update-runbook.md  (CloudFront invalidation 手順 / Open Item O-11)
```

---

## Unit-7: Title Catalog Distribution Unit

### 種別と配置
- **種別**: Static Service (S3 + CloudFront / Lambda 経由しない)
- **AWS リージョン**: ap-northeast-1 (S3) + Edge (CloudFront)
- **由来**: Application Design DD-01 (S-04 AWS-shift) で新規追加

### 責務 (What)
- 称号メタ静的配信 (FR-10 静的メタ部 / S-05): `https://<cloudfront-domain>/titles-catalog.json`
- CloudFront edge cache (TTL: 1 時間)
- ETag 対応 (Mobile 側のキャッシュヒット時は 304 Not Modified)
- 更新フロー (運用): 開発者が S3 にアップロード → CloudFront invalidation 実行
- S3 versioning 有効でロールバック容易

### 機微情報の境界
- **公開コンテンツのみ** (機微情報なし) / NFR-DAT-02 / SECURITY-13 と整合
- TLS 1.2+ (CloudFront 標準) / S3 SSE-S3
- S3 Bucket Policy: CloudFront OAI からのみ許可 (Direct S3 GET をブロック)

### コード組織化 (Greenfield)
```
title-catalog/                  ← Unit-7 のリポジトリ
├── titles-catalog.json         初期メタデータ (Open Item O-10 / 10+ 件)
├── schema.json                 TitleCatalogEntry 型定義
└── runbook.md                  S3 アップロード + CloudFront invalidation 手順
```

> **注**: AWS リソース (S3 Bucket / CloudFront Distribution) の定義は **Unit-6 (Infrastructure)** が担当。Unit-7 は配信コンテンツ + 運用手順を持つ。

---

## カバレッジ確認

### FR カバレッジ

| FR | 担当 Unit |
|---|---|
| FR-01, 02 | Unit-1 |
| FR-03 | Unit-1 (位置情報) + Unit-5 (天気 API) |
| FR-04 | Unit-3 (PBT) |
| FR-05 | Unit-2 (PBT) |
| FR-06 | Unit-2 (Bedrock 呼び出し) |
| FR-07 | Unit-2 (プロンプト) + Unit-4 (履歴サマリ提供) |
| FR-08 | Unit-4 |
| FR-09 | Unit-4 |
| FR-10 | Unit-4 (動的判定 / PBT) + Unit-7 (静的メタ配信) |
| FR-11 | Unit-1 (UI 演出) |
| FR-12 | Unit-1 (CalendarDataAdapter) + Unit-2 (User Prompt 取り込み) |
| FR-13 | Unit-1 (ホーム画面ミニ表示 + iOS 標準カレンダー遷移) |
| FR-14 | Unit-1 (NotificationScheduler) + Unit-4 (markAchievement) |

すべての FR (FR-01〜14) がいずれかの Unit に割当済み ✓

### NFR カバレッジ

| NFR | 担当 Unit |
|---|---|
| NFR-USA-01 (片手操作) | Unit-1 |
| NFR-USA-02 (応答速度数秒以内) | Unit-2 (Bedrock 呼び出し / Direct Invoke) + Unit-7 (edge cache) |
| NFR-USA-03 (ダラけ感演出) | Unit-1 |
| NFR-DAT-01 (永続性無限保持) | Unit-4 |
| NFR-DAT-02 (機微データローカル) | Unit-1 (HealthKit/EventKit/CoreLocation 生データ保護) |
| NFR-DAT-03 (AWS のみ) | Unit-6 (IaC で全 AWS リソース定義) |
| NFR-DAT-04 (日本語のみ) | Unit-1 (UI) + Unit-2 (プロンプト) |
| NFR-DAT-05 (ap-northeast-1) | Unit-6 |
| NFR-CON-01〜04 (トーン/品位/安全配慮) | Unit-2 (LLM プロンプト) + Unit-1 (オンボーディング) + Unit-4 (Affirmation 文言事前審査) |

### 設計判断 (DD-XX) との整合

| DD | 担当 Unit |
|---|---|
| DD-01 (C-07 追加) | Unit-7 + Unit-6 (S3+CloudFront 定義) |
| DD-02 (META#AFFIRMATIONS) | Unit-4 |
| DD-03 (hoursSinceLastBath 取得元) | Unit-2 (呼び出し側) + Unit-4 (`getLastBathTime()` 提供側) |

### Open Items 担当一覧 (Application Design Section 16 / O-01〜O-15)

| Open Item | 内容 | 担当 Unit (Functional Design 段階で扱う) |
|---|---|---|
| O-01 | 迷惑リスク判定の具体閾値 | Unit-3 |
| O-02 | 動的トーンシフト発火条件 + プロンプト具体例 | Unit-2 |
| O-03 | DynamoDB シングルテーブルの PK/SK 設計 (`achieved` フィールド + META#AFFIRMATIONS 含む) | Unit-4 |
| O-04 | 称号付与条件の具体ロジック | Unit-4 |
| O-05 | フォールバック対話テンプレ (Bedrock 障害時) | Unit-2 |
| O-06 | キャラクター切替 UI の MVP 扱い (プレースホルダ vs. 後置) | Unit-1 |
| O-07 | History → Dialogue Direct Invoke vs. DynamoDB 直読みの最終判断 | Unit-2 + Unit-4 |
| ~~O-08~~ | (Closed / Q4=A iOS 確定で Web UX 設計不要) | — |
| O-09 | META#AFFIRMATIONS choice 別テンプレート初期 10〜20 件文言 | Unit-4 |
| O-10 | `titles-catalog.json` 初期 10+ 件メタ確定 | Unit-7 |
| O-11 | CloudFront invalidation 運用手順ドキュメント化 | Unit-7 + Unit-6 |
| O-12 | 起動時の TitleCatalogClient タイムアウト・リトライ・ETag 戦略 | Unit-1 |
| O-13 | 達成確認通知の文言 + 発火タイミング詳細 | Unit-1 |
| O-14 | カレンダー (EventKit) アクセス時のキャッシュ戦略 | Unit-1 |
| O-15 | 擬似データモード (DEBUG ビルド) の擬似値分布設計 | Unit-1 |

> O-08 は Closed (Q4=A 確定により Web 撤退案廃止)。残り 14 Open Items はすべていずれかの Unit に割当済み。
