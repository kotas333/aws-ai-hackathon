# Components — コンポーネント定義と高レベル責務

> **作成日**: 2026-05-07T15:00:47Z
> **対応ルール**: `.aidlc-rule-details/inception/application-design.md` Step 10
> **基底ドキュメント**:
> - `aidlc-docs/inception/requirements/requirements.md` (FR-01〜14 / NFR / R1〜R14)
> - `aidlc-docs/inception/user-stories/user-stories.md` (全 21 ストーリー / AC)
> - `aidlc-docs/inception/plans/execution-plan.md` Section 3.1
> **関連**: `component-methods.md` (メソッドシグネチャ) / `services.md` (サービス層) / `component-dependency.md` (依存関係)

## 命名規則

- コンポーネント ID: `C-NN` (NN: 二桁ゼロ埋め)
- Unit との対応関係は **Units Generation ステージで定義** される (本ステージはコンポーネント定義に責務を限定)

---

## コンポーネント一覧

| ID | コンポーネント名 | 配置 | 主要 FR | 主要ストーリー |
|---|---|---|---|---|
| **C-01** | Mobile Client | **iOS のみ (撤退ルートは Section 11 参照)** | FR-01, FR-02, FR-11, **FR-12**, **FR-13**, **FR-14 (Mobile 側)** | US-1.1, US-1.2, **US-1.6**, US-3.1, **US-3.3 (Mobile 側)**, US-5.1〜5.6, **US-5.7** |
| **C-02** | Dialogue API (API Gateway + Lambda) | AWS / ap-northeast-1 | FR-05, FR-06, FR-07 | US-2.1, US-2.2, US-2.4 |
| **C-03** | Risk Calculator | AWS / Lambda 内ライブラリ (純粋関数) | FR-04 | US-1.5 |
| **C-04** | History & Title Service | AWS / Lambda + DynamoDB (META#AFFIRMATIONS / S-02 AWS-shift / **achieved フィールド FR-14** / **getLastBathTime DD-03**) | FR-08, FR-09, FR-10 (動的判定), **FR-14 (達成フラグ記録)** | US-3.1, US-3.2, **US-3.3 (記録側)**, US-4.1, US-4.2 |
| **C-05** | External Client | AWS / Lambda 内ライブラリ | FR-03 | US-1.3 |
| **C-06** | Infrastructure (CDK/CloudFormation) | AWS / IaC | NFR-DAT-03, NFR-DAT-05 | (横断的) |
| **C-07** | Title Catalog Distribution (S-04 AWS-shift) | AWS / S3 + CloudFront | FR-10 (静的メタ配信) | US-4.2 (称号 name/description) |

---

## C-01: Mobile Client

### 配置と前提
- **プラットフォーム**: **iOS 確定 (Q4=A)** / 撤退ルートは「iOS のまま擬似データモード (DEBUG ビルド)」(Web 版廃止 / Section 11 参照)
- **言語**: **Swift / SwiftUI 確定**
- **オフライン要件**: 機微データ (ヘルスケア生データ / **カレンダー生情報 FR-12**) は **端末ローカルにのみ保持** / NFR-DAT-02 / R3

### 責務 (What)

1. **オンボーディング** — US-5.1, US-5.5
   - 初回起動時のキャラクター紹介
   - **コンセプト明示** (人をダメにするおふざけアプリ / 真剣な健康相談には使わない / R11 対策方針 (a) / requirements.md 参照)
   - 同意ボタン必須 (US-5.5 AC)

2. **ヘルスケアデータ取得** — US-1.1, US-1.2 (FR-01, FR-02)
   - HealthKit から 6 項目取得 (歩数 / アクティブエネルギー / 運動時間 / 心拍数 / 睡眠時間 / 立ち上がり時間)
   - 失敗時のフォールバック (ヘルスケアなしで対話生成可)
   - **HealthKit ラッパー (HealthDataAdapter)** で抽象化 → 擬似データ層と差し替え可能 (Q4 確定 / Section 11 参照)

3. **位置情報取得** — US-1.3 (FR-03)
   - 取得時のみ使用、永続保存しない (NFR-DAT-02)
   - 失敗時のフォールバック

3.5. **カレンダーデータ取得** — US-1.6 (FR-12) / **FR-12**
   - iOS EventKit から翌日の予定サマリを取得 (`CalendarSummary { isHoliday, earliestEventTime, eventCount, asOf }`)
   - 機微情報 (タイトル / 場所 / 参加者) は **端末ローカル限定** / AWS には集計値のみ送信
   - 失敗時のフォールバック: 全フィールド null で対話生成継続
   - **CalendarDataAdapter** で抽象化 → DEBUG ビルドで PseudoCalendarAdapter に差し替え可能 (Section 11 参照)

4. **Dialogue API 呼び出し** — US-2.1 (FR-05, FR-06)
   - ヘルスケア要約 (生データではなく統計値) + 環境データ + 端末 UUID を `POST /dialogue` に送信
   - 応答を受け取り、ジャッジと悪魔のメッセージを表示 (US-2.1 AC: 識別可能形式 + **悪魔最後** + 責めない表現)

5. **対話表示** — US-2.1, US-2.2 (FR-06)
   - 4〜6 ターンの対話表示
   - ストリーミング表示も可
   - エラー時のフォールバック表示「ちょっと考え事してます…」

6. **選択 (入る/サボる) 送信** — US-3.1 (FR-08)
   - `POST /selections` に送信
   - 選択後に両キャラから全肯定メッセージ表示

7. **履歴表示** — US-3.2 (FR-09)
   - `GET /history?from=YYYY-MM&to=YYYY-MM` を呼び出し
   - カレンダー UI で 3 状態色分け (中立配色 / R11 対応)

8. **称号表示** — US-4.1, US-4.2 (FR-10) / **S-04 AWS-shift 反映**
   - 選択送信レスポンスに含まれる新規称号通知 (id のみ) を画面表示
   - 獲得済み称号 ID 一覧 (`GET /titles`) を取得
   - 称号 **名前/説明** は **C-07 (CloudFront)** から取得した `TitleCatalog` をローカルキャッシュで結合
   - 起動時 + 設定画面の「カタログ更新」ボタンで catalog refresh

8.5. **選択後の全肯定メッセージ表示** — US-3.1 / **S-02 AWS-shift 反映**
   - 選択送信レスポンスに含まれる `affirmation: { judgeMessage, devilMessage }` を画面に表示
   - DDB 由来 (META#AFFIRMATIONS パーティション) のテンプレートを Lambda がランダム選択した結果を表示するだけ / 固定文言を Mobile に持たない

8.6. **30 分後の達成確認通知スケジュール** — US-3.3 (FR-14) / **FR-14**
   - `choice='bath'` の選択完了レスポンスに含まれる `achievementCheckScheduledAt` を受け、**iOS UNUserNotificationCenter** にローカル通知をスケジュール
   - 30 分後に通知発火 → ユーザーが Yes/まだを選択 → `submitAchievement(selectionId, achieved)` で C-04 に記録
   - 通知許可フローを **オンボーディングに追加**
   - **NotificationScheduler** で抽象化 → DEBUG ビルドで LoggingNotificationScheduler に差し替え可能 (Section 11 参照)
   - `choice='skip'` の場合はスケジュールしない (US-3.3 AC5)

9. **設定画面** — US-5.3
   - 許可状態表示 / バージョン情報 / コンセプト再表示 (US-5.5 から再表示)
   - キャラクター切替 UI (US-2.3 Should: ドメインモデル拡張点として用意)

10. **ダラけ感のある UI 演出** — US-5.6 (FR-11 / NFR-USA-03)
    - キャラがソファに沈む / ぬるっと動く / ゆるいフォント等のうち最低 3 つ

11. **翌日予定のホーム画面ミニ表示** — US-5.7 (FR-13) / **FR-12**
    - ホーム画面の片隅に翌日予定サマリ (最早開始時刻 / イベント数) を表示
    - タップで **iOS 標準カレンダーアプリに遷移** (アプリ内に詳細画面を持たない)
    - `getTomorrowMiniSummary()` で取得 (US-1.6 と同じ EventKit を再利用 / キャッシュ戦略は Functional Design / O-14)
    - カレンダー権限拒否時は表示しない (画面ノイズ回避)

### 抽象化点 (iOS 確定 / DEBUG ビルド擬似データモード / Section 11 Q4 Implementation Plan 参照)

| インターフェース | Production 実装 (iOS Release) | DEBUG ビルド擬似データ | テスト用 (Mock) |
|---|---|---|---|
| `HealthDataAdapter` | HealthKitAdapter (HealthKit から 6 項目取得) | PseudoHealthAdapter (固定 JSON または UI で擬似値入力) | MockHealthAdapter |
| `LocationDataAdapter` | CoreLocationAdapter | PseudoLocationAdapter (固定値) | MockLocationAdapter |
| `CalendarDataAdapter` (**FR-12**) | EventKitAdapter (EventKit から翌日予定サマリ集計) | PseudoCalendarAdapter (固定 JSON) | MockCalendarAdapter |
| `NotificationScheduler` (**FR-14**) | LocalNotificationScheduler (UNUserNotificationCenter) | LoggingNotificationScheduler (発火せずログ出力のみ) | MockNotificationScheduler |
| `WeatherDataAdapter` | (Mobile では使わない / Dialogue API 側で取得) | 同上 | — |
| `DialogueAPIClient` | URLSession で HTTPS | (差し替え不要 / 同じ実装で AWS と通信) | MockDialogueAPIClient |
| `TitleCatalogClient` | URLSession で CloudFront GET (ETag 対応) | (差し替え不要) | MockTitleCatalogClient |

> **設計原則**: アプリケーションコア (UI / ViewModel / オンボーディング) は Adapter インターフェースにのみ依存し、HealthKit 固有 API には直接触れない。

### 責務外
- ヘルスケア生データの AWS 送信 (NFR-DAT-02 / R3 違反)
- LLM 直接呼び出し (Lambda 経由のみ / SECURITY-06)
- 称号付与ロジック (Server 側 / R13: LLM 動的生成禁止)
- 称号メタの保有 (S-04 AWS-shift により C-07 から取得)
- 選択後全肯定文言の保有 (S-02 AWS-shift により C-04 経由 DDB から取得)

---

## C-02: Dialogue API

### 配置と前提
- **AWS サービス**: API Gateway + Lambda
- **リージョン**: ap-northeast-1 (Tokyo / NFR-DAT-05)
- **モデル**: Amazon Bedrock / Claude 3.5 Sonnet 等 (予選通過後の Bolt でモデルアクセス申請 / R9)

### 責務 (What)

1. **対話生成エンドポイント** — `POST /dialogue` (FR-05, FR-06)
   - リクエスト: ヘルスケア要約 + 環境データ + 端末 UUID
   - **Risk Calculator (C-03) を呼び出して** 迷惑リスクフラグを取得 (FR-04)
   - **External Client (C-05) を呼び出して** 天気データ補完 (位置情報あれば / FR-03)
   - **History & Title Service (C-04) を呼び出して** 連続サボり長期化検知用の最近の履歴サマリ取得 (FR-07)
   - **プロンプト構築** (純粋関数 / **PBT 対象 / FR-05**) — 綺麗度判定の入力組み立て
   - **Bedrock Claude 呼び出し** で対話生成
   - レスポンス: 4〜6 ターンの対話 (ジャッジ・悪魔交互 + **悪魔最後**)

2. **プロンプト構造の責務** — FR-05, FR-06, FR-07
   - **System Prompt**: キャラクター設定 + 設計原則 (悪魔最後 / 責めない / 全肯定 / 健康配慮ポリシー)
   - **Dynamic Input**: ヘルスケア要約 + 環境データ + 迷惑リスクフラグ + 連続サボり長期化フラグ
   - **動的トーンシフト** (US-2.4 Must): フラグに応じて悪魔のトーンが柔らかく健康寄りにシフト
   - 詳細プロンプトは `application-design.md` Section 9 (LLM プロンプト構造) と Functional Design で詳細化

3. **エラーハンドリング**
   - Bedrock 障害時: 503 + フォールバック用テンプレ対話 (オプション)
   - リクエスト過多: NFR-USA-02 (数秒以内) を維持するためのスロットリング (R10 LLM 料金予算超過対策)

### 責務外
- 履歴永続化 (C-04 の責務)
- 称号付与判定 (C-04 の責務)
- 迷惑リスク数値計算 (C-03 の責務)

---

## C-03: Risk Calculator

### 配置と前提
- **AWS サービス**: Lambda 内の **純粋関数ライブラリ** (独立 Lambda ではなく C-02 にバンドル)
- **特性**: I/O 副作用なし / **PBT 対象 (Partial) / FR-04**

### 責務 (What)

1. **迷惑リスク判定** — US-1.5 (FR-04)
   - 入力: プロキシ指標 (最終入浴経過時間 / 運動量合算スコア / 気温・湿度 / 心拍数)
   - 出力: リスクレベル (`low` / `medium` / `high`) + 各指標の値 + 欠損ありフラグ
   - **複合条件で計算** (詳細閾値は Construction Phase / Functional Design で詳細化)
   - **欠損部分は暫定スコア** (US-1.5 AC)

### 責務外
- LLM 呼び出し (C-02 の責務)
- 履歴取得 (C-04 の責務)

### PBT 対象 (Partial 適用範囲) — PBT-02, 03, 07, 08, 09
- 入力空間 (ヘルスケア + 環境のすべての値域) でフラグ計算が冪等であること
- リスクレベルが `low | medium | high` のいずれかに必ず分類されること
- 欠損ありフラグが正しく立つこと

---

## C-04: History & Title Service

### 配置と前提
- **AWS サービス**: Lambda + DynamoDB
- **DynamoDB スキーマ**: シングルテーブル設計を推奨 (Construction Phase で確定)
- **暗号化**: 保管時暗号化 (Server-Side Encryption 有効 / SECURITY-07)

### 責務 (What)

1. **選択記録エンドポイント** — `POST /selections` (FR-08) / **S-02 AWS-shift 反映**
   - 入力: 端末 UUID / 日時 / 選択内容 (入る/サボる) / ヘルスケア要約 / 天気要約 / 迷惑リスクレベル
   - DynamoDB に保存
   - **称号付与判定** を実行 → 新規称号 ID をレスポンスに含める (静的メタは C-07 から)
   - **Affirmation 取得** (S-02 AWS-shift): META#AFFIRMATIONS パーティションから choice 別にランダム 1 件選び `affirmation: { judgeMessage, devilMessage }` をレスポンスに含める

2. **履歴取得エンドポイント** — `GET /history?from=YYYY-MM&to=YYYY-MM` (FR-09)
   - 端末 UUID で範囲取得
   - カレンダー表示用に日単位集計 (各日: 入った / サボった / 未利用 / 詳細リンク)
   - 全期間データ保持 (NFR-DAT-01)

3. **獲得称号一覧エンドポイント** — `GET /titles` (FR-10) / **S-04 AWS-shift 反映**
   - 獲得済み称号 ID + 獲得日時のみを返す (静的メタ name/description は含まない / C-07 で配信)

4. **称号付与判定 (動的)** — US-4.1, US-4.2 (FR-10) / **PBT 対象 / 純粋関数**
   - 入力: 端末 UUID の累計履歴サマリ
   - 出力: 付与すべき新規称号 ID リスト (事前リスト 10 件以上 / R13: LLM 動的生成禁止)
   - 注: 名前/説明等の **静的メタ** は C-07 (Title Catalog Distribution) で別途配信

5. **連続サボり長期化サマリ提供** (内部 API / C-02 から呼ばれる) — US-2.4 (FR-07)
   - 直近 N 日の選択分布を返す
   - C-02 が動的トーンシフトのプロンプト入力に使う

5.5. **最終 'bath' タイムスタンプ提供** (内部 API / C-02 から呼ばれる) — **DD-03 / FR-04**
   - `getLastBathTime(deviceUUID)` で最新の `choice='bath'` 選択時刻を返す
   - C-02 が現在時刻との差分から `hoursSinceLastBath` を計算
   - 履歴に 'bath' なしの場合は null

6. **Affirmation テンプレート参照** (S-02 AWS-shift / FR-08 補助) — US-3.1 全肯定原則
   - DynamoDB Query: `PK='META#AFFIRMATIONS'`, `SK='AFFIRMATION#<choice>#*'`
   - 取得結果からランダム 1 件選択
   - 文言は事前審査済みテンプレート (NFR-CON-03 品位 / R13 と同種の運用ガードレール)

7. **達成フラグ記録エンドポイント** — `POST /selections/{selectionId}/achievement` (FR-14 / **FR-14**) / US-3.3
   - 入力: `selectionId` / `achieved: bool`
   - DynamoDB UpdateItem: 該当 SelectionRecord の `achieved` を更新
   - 元の `choice='bath'` は維持 (上書きしない / US-3.3 AC4)
   - `choice='skip'` の SelectionRecord に対する呼び出しは 400 エラー (US-3.3 AC5)

### 責務外
- LLM 呼び出し (C-02 の責務)
- 迷惑リスク計算 (C-03 の責務)

### PBT 対象 (Partial 適用範囲) — PBT-02, 03, 07, 08, 09
- 称号付与の条件評価が累計履歴に対して冪等であること
- 称号 ID が事前リスト 10 件以上のみに含まれること (R13 担保 / 事前リストは C-07 catalog のソース・オブ・トゥルース)
- 同一履歴に対して同じ称号を複数回付与しないこと

注: `pickAffirmation()` は I/O (DDB Query) を含むため PBT 対象外。テストは Construction Phase で個別に設計。

---

## C-05: External Client

### 配置と前提
- **AWS サービス**: Lambda 内のクライアントライブラリ (C-02 にバンドル)
- **外部 API**: 無料枠のある天気 API (OpenWeatherMap 等) — 予選通過後の Bolt で確定

### 責務 (What)

1. **天気 API クライアント** — US-1.3 (FR-03)
   - 入力: 緯度経度
   - 出力: 気温 (℃) / 天気 (晴れ/曇り/雨/雪等) / 湿度
   - リトライ (1 回まで) + タイムアウト (1 秒)
   - キャッシュ (同一座標 / 30 分以内は再利用 / R10 料金対策にも貢献)

2. **エラー時のフォールバック**
   - 失敗時は環境データなしで C-02 の対話生成に進める (US-1.3 AC)

### 責務外
- 位置情報取得 (Mobile 側 C-01 の責務)
- ヘルスケア API (Mobile 側 C-01 の責務)

---

## C-07: Title Catalog Distribution (新規 / S-04 AWS-shift)

### 配置と前提
- **AWS サービス**: S3 (Bucket) + CloudFront (Distribution / OAI 経由) — **本設計で初出 / S-04 AWS-shift で追加**
- **配信内容**: 称号メタ情報 (id / name / description / category) を JSON 1 ファイル `titles-catalog.json` で配信

### 責務 (What)

1. **称号メタ静的配信** — US-4.2 (FR-10 静的メタ部) — S-04 AWS-shift
   - `https://<cloudfront-domain>/titles-catalog.json` を Mobile に配信
   - CloudFront edge cache (TTL: 1 時間推奨)
   - ETag 対応で Mobile 側のキャッシュヒット時は 304 返却

2. **更新フロー** (運用)
   - 開発者が S3 にアップロード → CloudFront invalidation 実行
   - S3 Versioning 有効でロールバック容易

3. **可用性とフェイルオープン**
   - CloudFront 障害時、Mobile はローカルキャッシュ (前回値) を継続使用
   - 初回起動時のみ取得失敗で「カタログ取得失敗」表示

### 責務外
- 称号付与判定 (C-04 の責務)
- 獲得済み称号の永続化 (C-04 + DynamoDB の責務)
- ユーザー固有データの配信 (本コンポーネントは公開静的メタのみ)

### セキュリティ・プライバシー
- **公開コンテンツのみ** (機微情報なし / NFR-DAT-02 / SECURITY-13 と整合)
- TLS 1.2+ (CloudFront 標準)
- S3 SSE-S3 (静的メタとはいえ暗号化は標準で適用)
- S3 Bucket Policy: CloudFront OAI からのみ許可 (Direct S3 GET をブロック)

---

## C-06: Infrastructure

### 配置と前提
- **AWS サービス**: CloudFormation または CDK (NFR-DAT-03 / 主催規約準拠)
- **主管対象**: API Gateway / Lambda / DynamoDB / IAM / CloudWatch / Secrets Manager (天気 API キー)

### 責務 (What)

1. **AWS リソース定義 (IaC)** — `requirements.md` Technical Context
   - API Gateway: REST API + 認可 (端末 UUID ベース軽量識別 / SECURITY-04)
   - Lambda: C-02 (Dialogue) と C-04 (History & Title) の関数定義 / IAM Role 最小権限 (SECURITY-05)
   - DynamoDB: シングルテーブル / Server-Side Encryption (SECURITY-07) / Point-in-Time Recovery / META#AFFIRMATIONS パーティションを含む (S-02 AWS-shift)
   - **S3 + CloudFront** (S-04 AWS-shift / C-07): Catalog Bucket (Versioning ON / SSE-S3) / Distribution (OAI / TLS 1.2+ / TTL 1h)
   - IAM: Bedrock InvokeModel 権限 / DynamoDB CRUD 権限 / S3 GetObject 権限 (CloudFront OAI 経由のみ) / 最小権限の原則

2. **監視・ログ** — `requirements.md` Technical Context (CloudWatch)
   - CloudWatch Logs (構造化ログ / SECURITY-13: 機微情報をログに残さない)
   - メトリクス: Lambda 実行時間 / Bedrock 呼び出し回数 / DynamoDB 消費 RCU/WCU
   - アラーム: コスト閾値超過 (R10 / NFR 範囲外だが運用必須)

3. **シークレット管理** — SECURITY-09
   - 天気 API キーは Secrets Manager / Lambda 環境変数経由で取得
   - Bedrock はモデルアクセスを IAM で制御 (キー不要)

4. **デプロイパイプライン** (予選通過後の Bolt で構築 / 概略)
   - CDK Synth → CloudFormation Deploy
   - Bolt 1 で AWS アカウント / Bedrock モデルアクセス申請 (R8 / R9 / 最優先タスク)

### 責務外
- アプリケーションロジック (C-01〜C-05 の責務)

---

## カバレッジ確認

### FR-01〜11 のコンポーネント割当

| FR | 内容 | 担当 |
|---|---|---|
| FR-01 | ヘルスケア連携 | C-01 |
| FR-02 | ヘルスケア取得項目 6 種 | C-01 |
| FR-03 | 天気・気温連携 | C-01 (位置情報) + C-05 (天気 API) |
| FR-04 | 迷惑リスク閾値判定 | C-03 |
| FR-05 | 綺麗度判定 (LLM) | C-02 |
| FR-06 | ジャッジ・悪魔対話生成 | C-02 |
| FR-07 | 悪魔の発言シフト | C-02 (プロンプト) + C-04 (履歴サマリ提供) |
| FR-08 | 選択の記録 + Affirmation 配信 | C-04 (S-02 AWS-shift) |
| FR-09 | サボり履歴 | C-04 |
| FR-10 | 称号・バッジ (動的判定) | C-04 |
| FR-10 | 称号・バッジ (静的メタ配信) | C-07 (S-04 AWS-shift) |
| FR-11 | ダラけ感のある UI 演出 | C-01 |
| **FR-12** | **カレンダー連携 (EventKit)** | **C-01 (CalendarDataAdapter / FR-12)** + C-02 (User Prompt 取り込み) |
| **FR-13** | **翌日予定のホーム画面ミニ表示** | **C-01 (`getTomorrowMiniSummary()` + iOS 標準カレンダー遷移 / FR-12)** |
| **FR-14** | **入浴決意の達成確認通知** | **C-01 (NotificationScheduler / FR-14)** + **C-04 (markAchievement / S-06 / FR-14)** |

すべての FR がいずれかのコンポーネントに割り当てられていることを確認。

### 全 21 ストーリーのコンポーネント割当

| ストーリー | 担当 |
|---|---|
| US-1.1, US-1.2 | C-01 |
| US-1.3 | C-01 + C-05 |
| US-1.4 (S) | C-01 |
| US-1.5 | C-03 (純粋関数) + C-02 (呼び出し) |
| US-1.6 (M / Revision 2 / FR-12) | C-01 (CalendarDataAdapter) + C-02 (User Prompt 取り込み) |
| US-2.1, US-2.2 | C-02 + C-01 (表示) |
| US-2.3 (S) | C-01 (UI) + C-02 (プロンプトに反映) — Should ストーリー / 拡張点として担保 |
| US-2.4 | C-02 (プロンプト) + C-04 (履歴サマリ提供) — Must / 動的トーンシフトの中核 |
| US-3.1 | C-01 + C-04 (S-02 AWS-shift で affirmation を C-04 経由 DDB から取得) |
| US-3.2 | C-01 + C-04 |
| US-3.3 (S / Revision 2 / FR-14) | C-01 (NotificationScheduler) + C-04 (markAchievement / S-06) |
| US-4.1, US-4.2 | C-04 (動的判定) + C-07 (静的メタ配信 / S-04 AWS-shift) + C-01 (結合表示) |
| US-5.1, US-5.2, US-5.3 | C-01 |
| US-5.5 | C-01 (オンボーディング画面) — R11 対策方針 (a)(c): コンセプト明示 + 予選通過後 Bolt で法務観点レビュー |
| US-5.6 | C-01 |
| US-5.7 (S / Revision 2 / FR-12) | C-01 (`getTomorrowMiniSummary()` + iOS 標準カレンダーへの遷移) |

すべての 21 ストーリーが 1 つ以上のコンポーネントに割り当てられていることを確認。

---

## 拡張容易性 (US-2.3 Should の拡張点)

### キャラクター切替 (US-2.3) の拡張点

- **C-01 (Mobile Client)**: 設定画面でキャラクターセット ID を選択 → ローカル設定に保存 → `POST /dialogue` リクエストに `characterSetId` を含める
- **C-02 (Dialogue API)**: System Prompt をキャラクターセット ID で切り替え (プロンプトテンプレートをパラメータ化)
- **拡張時の作業**: System Prompt のテンプレートに新セット (例: 関西弁) を追加するだけ。コード変更は最小限

> US-2.3 を Should に保ったまま将来の Must 昇格にも対応できる条件 = ドメインモデルが上記の拡張点を持つこと。本ドキュメントで担保済み。
