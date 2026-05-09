# Component Dependency — 依存マトリックス・通信パターン・データフロー

> **作成日**: 2026-05-07T15:00:47Z
> **対応ルール**: `.aidlc-rule-details/inception/application-design.md` Step 10
> **関連**: `components.md` (コンポーネント定義) / `component-methods.md` (メソッド契約) / `services.md` (サービス層)

## 注意

- 本ファイルは **依存関係 + 通信パターン + データフロー** を視覚化する
- **機微データ (ヘルスケア生データ / 位置情報) のローカル境界 (R3 / NFR-DAT-02 / SECURITY-13)** を明示することが本ファイルの中核責務
- ASCII 図は `common/ascii-diagram-standards.md` 準拠 (`+` `-` `|` `^` `v` `<` `>` とスペースのみ)

---

## 1. 依存マトリックス (Compile-time / Runtime)

行: 利用側 (Caller) / 列: 被利用側 (Callee) / `o`: 直接依存 / `-`: 依存なし

| Caller \ Callee | C-01 | C-02 | C-03 | C-04 | C-05 | C-06 | C-07 | Bedrock | DynamoDB | 外部天気 API | S3 | CloudFront | EventKit | UNUserNotif |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| C-01 (Mobile) | — | o | - | o | - | - | o (via CF) | - | - | - | - | o | o (FR-12) | o (FR-14) |
| C-02 (Dialogue API) | - | — | o | o | o | - | - | o | - | - | - | - | - | - |
| C-03 (Risk Calculator) | - | - | — | - | - | - | - | - | - | - | - | - | - | - |
| C-04 (History & Title) | - | - | - | — | - | - | - | - | o | - | - | - | - | - |
| C-05 (External Client) | - | - | - | - | — | - | - | - | - | o | - | - | - | - |
| C-06 (Infrastructure) | - | (deploys) | - | (deploys) | - | — | (deploys) | (perms) | (defines) | - | (defines) | (defines) | - | - |
| C-07 (Title Catalog) | - | - | - | - | - | - | — | - | - | - | o | (fronted by) | - | - |

### 依存の凡例

- **C-01 → C-02**: HTTPS (API Gateway 経由 / `POST /dialogue` `POST /selections` `GET /history` `GET /titles`)
  - 注: Mobile から見ると C-04 への依存は **API Gateway 経由で間接的**。実体は API Gateway → Dialogue Lambda or History Lambda
- **C-01 → CloudFront (C-07)**: HTTPS (`GET /titles-catalog.json`) — **S-04 AWS-shift / API Gateway 不経由 / Lambda 不経由**
- **C-02 → C-03**: 同一 Lambda 内のローカル関数呼び出し (純粋関数)
- **C-02 → C-05**: 同一 Lambda 内のクライアントライブラリ呼び出し
- **C-02 → C-04**: クロス Lambda Direct Invoke (内部 API / API Gateway を経由しない / レイテンシ回避)
- **C-04 → DynamoDB**: AWS SDK (META#AFFIRMATIONS パーティションへの Query を含む / S-02 AWS-shift)
- **C-05 → 外部天気 API**: HTTPS
- **C-07 → S3 / CloudFront**: 配信構成 (CloudFront が S3 origin を fronts / OAI 経由)
- **C-01 → EventKit (FR-12)**: iOS OS API (端末ローカル / 機微データはここから外に出ない)
- **C-01 → UNUserNotificationCenter (FR-14)**: iOS OS API (端末ローカル / 通知はクラウド経由しない)
- **C-02 → C-04 (DD-03)**: getRecentSkipPattern() に加え getLastBathTime() も Direct Invoke (両者並列で取得)

### 循環依存チェック

依存グラフは DAG (有向非巡回) であることを確認:

```
C-01 -> C-02 -> { C-03, C-05, Bedrock }
              -> C-04 -> DynamoDB
                 (getRecentSkipPattern + getLastBathTime / DD-03 / 並列 Direct Invoke)
C-01 -> C-04 (via API Gateway) -> DynamoDB (Selection / META#AFFIRMATIONS / achievement)
C-01 -> C-07 (via CloudFront) -> S3
C-01 -> EventKit (FR-12 / OS API / on-device)
C-01 -> UNUserNotificationCenter (FR-14 / OS API / on-device)
```

循環依存なし。

---

## 2. データフロー図 (機微データの境界明示)

### 凡例
- `[LOCAL]` = 端末ローカル限定 / AWS には送信しない
- `[CLOUD]` = AWS に送信される
- `[SUMMARY]` = 統計値のみ (生レコードを含まない)
- `==>` = HTTPS over TLS
- `-->` = 同一プロセス内呼び出し
- `~~~` = 機微データ境界 (越えてはいけない)

```
+--------------------------------------------------------------+
|                         iOS DEVICE                           |
|                                                              |
|   +---------------+        +-----------------------+         |
|   | HealthKit     |  raw   | C-01 Mobile Client    |         |
|   | (OS provides) |------->|  HealthDataAdapter    |         |
|   +---------------+ [LOCAL]|  - HealthKit raw data |         |
|                            |    is RESIDENT here   |         |
|   +---------------+        |  - Reduces to         |         |
|   | CoreLocation  |  raw   |    HealthSummary      |         |
|   | (OS provides) |------->|    [LOCAL][SUMMARY]   |         |
|   +---------------+        |                       |         |
|   +---------------+ raw    |  CalendarDataAdapter  | [FR-12]  |
|   | EventKit      |------->|  - title/loc/attendee|         |
|   | (OS provides) | [LOCAL]|    is RESIDENT here   |         |
|   +---------------+        |  - Reduces to         |         |
|                            |    CalendarSummary    |         |
|                            |    [LOCAL][SUMMARY]   |         |
|                            |                       |         |
|   +-------------------+    |  Notification         | [FR-14]  |
|   | UNUser            |<---|  Scheduler            |         |
|   | NotificationCtr   |    |  (schedules local     |         |
|   | (OS provides)     |    |   notif at +30min)    |         |
|   +-------------------+    |                       |         |
|                            |  Onboarding (US-5.5)  |         |
|                            |  Calendar UI (US-3.2) |         |
|                            |  Tomorrow Mini Display|         |
|                            |  (US-5.7 / FR-12)      |         |
|                            |  TitleCatalogClient   |         |
|                            |  (local cached JSON)  |         |
|                            +-----------+-----------+         |
|                                        |                     |
+----------------------------------------|---------------------+
                                         |
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~|~~~~~~~~~~~~~~~~~~~~~~ <-- 機微データ境界
                                         |
                              POST /dialogue / /selections
                              { deviceUUID, HealthSummary,    [SUMMARY only]
                                location, choice, ... }
                                         |
                                         |   GET catalog (公開静的 / 機微情報なし)
                                         |   ----------------------+
                                         v                         v
+----------------------------------------|-------------------------|---+
|                          AWS CLOUD                                   |
|                          ap-northeast-1                              |
|                                                                      |
|              +------------+                  +-----------------+     |
|              | API Gateway|                  | CloudFront      |     |
|              | (REST)     |                  | (edge cached)   |     |
|              +-----+------+                  +--------+--------+     |
|                    |                                  |              |
|         +----------+----------+                       v              |
|         |                     |                  +---------+         |
|         v                     v                  | S3      |         |
| +---------------+     +-----------------+        | Bucket  |         |
| | C-02 Dialogue |     | C-04 History &  |        | (catalog|         |
| | Lambda        |     | Title Lambda    |        |  JSON)  |         |
| |               |     |                 |        | SSE-S3  |         |
| | + C-03 Risk   |     | + evaluateNewTitles      | Version |         |
| |   Calculator  |     |   (PBT FR-10)   |        +---------+         |
| | + C-05 Ext.   |     | + pickAffirmation                            |
| |   Client      |     |   (S-02 AWS-shift)                           |
| |               |     +--------+--------+                            |
| | buildPrompt() |              |                                     |
| | (PBT FR-05)   |              v                                     |
| +-------+-------+     +-----------------+                            |
|         |             | DynamoDB        |                            |
|         |             | (SSE encrypted) |                            |
|         |             | [CLOUD][SUMMARY]|                            |
|         |             | + META#         |                            |
|         |             |   AFFIRMATIONS  |                            |
|         |             +-----------------+                            |
|         |                                                            |
|         |  Direct Invoke (FR-07: getRecentSkipPattern)               |
|         +-----------------------------> C-04                         |
|         |                                                            |
|         v                                                            |
| +---------------+                                                    |
| | Bedrock       |                                                    |
| | Claude        |                                                    |
| | Sonnet 4.6    |                                                    |
| | InvokeModel   |                                                    |
| +-------+-------+                                                    |
|         |                                                            |
|         v                                                            |
|   (dialogue text / no PII echoed)                                    |
|         |                                                            |
+---------|------------------------------------------------------------+
          |
          v
    +-------------+
    | C-05 -> 外部 |
    | 天気 API     |
    +-------------+
```

### 境界線 (`~~~`) の意味

- **HealthKit / CoreLocation / EventKit の生データ** は端末ローカルから出ない (NFR-DAT-02 / R3 / SECURITY-13 / **FR-12 で EventKit 追加**)
- **AWS に送信されるのは `HealthSummary` + `CalendarSummary` (集計値) のみ**:
  - HealthSummary: 歩数 (合計値) / アクティブエネルギー (合計 kcal) / 運動時間 (合計分) / 心拍数 (平均値) / 睡眠時間 (合計 h) / 立ち上がり時間 (合計分)
  - **CalendarSummary** (FR-12): isHoliday / earliestEventTime / eventCount のみ。**タイトル / 場所 / 参加者は送信しない**
  - **個別タイムスタンプ付きレコードは送信しない** (再識別リスク回避 / SECURITY-13)
- **位置情報の緯度経度** は対話生成リクエストに含まれるが、**永続保存はしない** (Lambda 実行内で天気 API 呼び出しに使うだけ / NFR-DAT-02)
- **DynamoDB に保存される機微データ**: SelectionRecord 内の `health` (HealthSummary 集計値のみ) / `environment` / `riskLevel` / **`achieved` (FR-14 / true/false/null)**。**生データは保存しない**
- **C-07 (S3 + CloudFront) は機微情報を扱わない** (公開静的メタのみ): 称号 name/description は機微性ゼロ。AWS-shift によるサービス追加でも、機微データ境界は移動しない
- **FR-14 (UNUserNotificationCenter)**: ローカル通知のみ。APNS (リモート) は使わない / 通知データは端末から外に出ない

---

## 3. 通信パターンの分類

| 経路 | プロトコル | 認可 | TLS | 備考 |
|---|---|---|---|---|
| C-01 → API Gateway | HTTPS | 端末 UUID (リクエストヘッダ) | TLS 1.2+ | NFR-DAT-02 / SECURITY-04, 06 |
| API Gateway → Lambda | AWS 内部 | IAM Role | (AWS 内部) | SECURITY-05 (最小権限) |
| Dialogue Lambda → History Lambda (Direct Invoke) | AWS Lambda Invoke API | IAM Role | (AWS 内部) | レイテンシ短縮 |
| Lambda → Bedrock | AWS SDK (HTTPS) | IAM Role (InvokeModel) | TLS | SECURITY-05, 06 |
| Lambda → DynamoDB | AWS SDK (HTTPS) | IAM Role (CRUD on table / META#AFFIRMATIONS Query 含む) | TLS | SECURITY-07 (SSE) |
| Lambda → Secrets Manager | AWS SDK (HTTPS) | IAM Role (GetSecretValue) | TLS | SECURITY-09 |
| Lambda → 外部天気 API | HTTPS | API Key (Secrets Manager 経由) | TLS | API キーをコードに含めない / SECURITY-09 |
| **C-01 → CloudFront (C-07)** | HTTPS | **不要** (公開静的メタ / 機微情報なし) | TLS 1.2+ | edge cached / ETag 対応 |
| **CloudFront → S3 (C-07 内部)** | AWS 内部 | OAI (Origin Access Identity) | (AWS 内部) | Direct S3 GET をブロック / SECURITY-06 |
| **C-01 → API Gateway (S-06 / FR-14)** | HTTPS | 端末 UUID (リクエストヘッダ) | TLS 1.2+ | `POST /selections/{selectionId}/achievement` / NFR-DAT-02 / SECURITY-04, 06 |
| **C-01 ↔ EventKit (FR-12)** | iOS OS API | iOS の Permission (要許可) | (端末ローカル) | 機微データ境界 `~~~` の内側で完結 |
| **C-01 ↔ UNUserNotificationCenter (FR-14)** | iOS OS API | iOS の Permission (要許可) | (端末ローカル) | ローカル通知のみ / APNS リモートなし |

---

## 4. データ保持先と保持期間

| データ | 保持先 | 保持期間 | 暗号化 | 関連 NFR/SECURITY |
|---|---|---|---|---|
| ヘルスケア生データ (個別レコード) | 端末ローカルのみ | (HealthKit 管理) | OS 標準 | NFR-DAT-02 / R3 / SECURITY-13 |
| **カレンダー生データ (タイトル/場所/参加者) (FR-12)** | 端末ローカルのみ | (EventKit 管理) | OS 標準 | NFR-DAT-02 / R3 / SECURITY-13 |
| 位置情報 (緯度経度) | (永続保存しない) | 取得から API 呼び出しまでの数秒のみ | TLS 通信時 | NFR-DAT-02 / R3 |
| HealthSummary (集計値) | DynamoDB | 無限保持 (NFR-DAT-01) | SSE | SECURITY-07 |
| **CalendarSummary (集計値 / FR-12)** | (Lambda 実行内のみ / 永続保存しない) | リクエスト内のみ | TLS 通信時 | NFR-DAT-02 |
| EnvironmentData | DynamoDB | 無限保持 | SSE | SECURITY-07 |
| 選択 (choice) | DynamoDB | 無限保持 (NFR-DAT-01) | SSE | SECURITY-07 |
| **入浴決意の達成フラグ (achieved / FR-14)** | DynamoDB (SelectionRecord 内) | 無限保持 | SSE | SECURITY-07 |
| **iOS ローカル通知の予約 (FR-14)** | 端末ローカル (UNUserNotificationCenter) | 発火まで (30 分) | OS 標準 | (機微情報なし) |
| **獲得称号履歴 (id + awardedAt のみ)** | DynamoDB | 無限保持 | SSE | SECURITY-07 |
| **Affirmation テンプレート** (S-02 AWS-shift) | DynamoDB (META#AFFIRMATIONS) | 永続 (運用更新) | SSE | NFR-CON-03 (品位) |
| **Title Catalog** (S-04 AWS-shift) | S3 (titles-catalog.json / Versioning ON) | 永続 (運用更新) | SSE-S3 | (機微情報なし) |
| **Title Catalog edge cache** | CloudFront edge | TTL 1h | TLS 通信時 | NFR-USA-02 (応答速度) |
| 端末 UUID | DynamoDB (PK) | 無限保持 | SSE | SECURITY-04 |
| API キー (天気) | Secrets Manager | (運用管理) | KMS | SECURITY-09 |
| CloudWatch Logs | CloudWatch | 30 日 (推奨) | KMS | SECURITY-13 (機微情報除外) |
| **Title Catalog ローカルキャッシュ** | 端末ローカル (アプリ Documents) | 起動時更新 | OS 標準 | (機微情報なし) |

---

## 5. 障害伝播と分離 (Resilience)

| 障害 | 分離レベル | フォールバック |
|---|---|---|
| HealthKit 権限拒否 / 取得失敗 | C-01 内 | `HealthSummary=null` で対話生成継続 (US-1.1 AC) |
| 位置情報拒否 / 取得失敗 | C-01 内 | `location=null` で対話生成継続 (US-1.3 AC) |
| 天気 API 失敗 (C-05) | C-02 Lambda 内 | `EnvironmentData=null` で対話生成継続 (US-1.3 AC) |
| Bedrock 障害 (C-02) | S-01 のみ | フォールバック対話テンプレ or 503 + 再試行案内 (US-2.1 AC) |
| DynamoDB 障害 (C-04) | S-02/S-03/S-04 のみ | 5xx (S-01 は影響なし) |
| **DynamoDB META#AFFIRMATIONS Query 失敗** (S-02 AWS-shift) | S-02 内のみ | Lambda バンドルの最小限フォールバック文言 1 件で `affirmation` を返す |
| History Lambda 障害 | S-01 の一部 | `recentSkipPattern={skipDays:0, consecutiveSkipDays:0}` で対話生成継続 (Functional Design で確定) |
| **CloudFront 障害 (C-07)** | S-05 のみ / Mobile UX への影響 | Mobile はローカルキャッシュ (前回値) を継続使用 / 初回起動時のみ「カタログ取得失敗」表示 |
| **S3 障害 (C-07 origin)** | S-05 のみ | CloudFront cache が生きていればしばらく継続 / 完全障害時は Mobile キャッシュ継続 |
| **EventKit 権限拒否 / 取得失敗 (FR-12)** | C-01 内 | `CalendarSummary=null` で対話生成継続 (US-1.6 AC2) / US-5.7 ミニ表示は非表示 |
| **UNUserNotificationCenter 権限拒否 (FR-14)** | C-01 内 | scheduleAchievementCheck() がスキップ / S-06 は呼ばれない / `achieved=null` のまま |
| **S-06 (Achievement) Lambda 障害** | S-06 のみ | Mobile 側で再試行案内 / `achieved` は null のまま (将来の称号判定に影響なし) |

---

## 6. デプロイ単位 (Bolt 1 で R8/R9 着手後の順序)

予選通過後の最初の Bolt で着手する順序:

1. **C-06 Infrastructure**: AWS アカウント・Bedrock モデルアクセス申請 (R8/R9 / 最優先)
2. **C-04 History & Title** (DynamoDB 含む / META#AFFIRMATIONS パーティション初期投入を含む) — Bedrock 不要なので独立して着手可能
3. **C-07 Title Catalog Distribution** (S3 + CloudFront) — Bedrock 不要 / Lambda 不要 / IaC のみで完結。R8 の AWS アカウント取得後すぐに着手可能
4. **C-05 External Client** (天気 API キー取得) — 外部 API キー申請を並行
5. **C-03 Risk Calculator** (純粋関数) — 単体テスト + PBT で先行検証可能
6. **C-02 Dialogue API** (Bedrock 利用) — モデルアクセス承認後
7. **C-01 Mobile Client** — UI / Adapter / API 呼び出し統合 (CloudFront catalog URL 設定込み)

> **R8/R9 リスク管理**: C-02 が最も R9 (Bedrock 申請) リスクを抱えるため、C-03/C-04/C-05/C-07 を先行する分離が有効。**C-07 は Lambda 経由しないため R9 とは完全独立**で、AWS アカウント (R8) 取得後すぐに価値が出る。

---

## 7. 拡張点 (US-2.3 Should の拡張点)

US-2.3 (キャラクター切替) の拡張点を依存関係レベルで明示:

```
C-01 (UI で characterSetId 選択)
   |  POST /dialogue { ..., characterSetId: 'kansai' }
   v
C-02 buildPrompt(input)
   |  characterSetId に応じて systemPrompt テンプレートを選択
   |  - 'standard' (MVP のみ)
   |  - 'kansai' (拡張時に追加)
   |  - ... (将来追加)
   v
Bedrock
```

**拡張時の作業**: C-01 の設定画面に新 ID を追加 + C-02 の systemPrompt テンプレートに新エントリを追加 (コード追加のみ / 既存メソッドシグネチャ不変 / US-2.3 拡張点として担保)。

---

## 8. 依存関係のカバレッジ確認

- すべてのコンポーネント (C-01〜C-07) が依存マトリックスに登場 (C-07 は S-04 AWS-shift で追加 / 7 のまま維持)
- **Revision 2 で OS API 依存を 2 件追加** (EventKit / FR-12 + UNUserNotificationCenter / FR-14): 端末ローカル境界の内側で完結、新規コンポーネント (C-XX) は不要
- 機微データ境界 (`~~~`) がデータフロー図に明示 / C-07 は機微情報を扱わないため境界の外側 (公開静的メタ) / **EventKit 生データ (タイトル/場所/参加者) も境界の内側で完結**
- すべての保存先 (DynamoDB / Secrets Manager / CloudWatch / 端末ローカル / S3 / CloudFront edge cache / **iOS UNUserNotificationCenter 予約**) のデータと暗号化方針が表に登場
- 障害伝播がコンポーネント間の分離レベルとともに表に登場 (新規: META#AFFIRMATIONS / CloudFront / S3 / **EventKit / UNUserNotificationCenter / S-06**)
- US-2.3 (Should) の拡張点が依存関係レベルで担保されている
- **Unit 数 7 維持の根拠が依存マトリックスで可視化**: CalendarDataAdapter / NotificationScheduler は C-01 内、getLastBathTime / markAchievement は C-04 内に追加されており、新規コンポーネントの追加なし
