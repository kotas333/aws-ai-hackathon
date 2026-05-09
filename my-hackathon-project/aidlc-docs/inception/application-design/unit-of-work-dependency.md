# Unit of Work — 依存マトリックス + Bolt 1 デプロイ順序

> **作成日**: 2026-05-09T03:30:00Z
> **対応ルール**: `.aidlc-rule-details/inception/units-generation.md` Step 12-15
> **基底ドキュメント**: `aidlc-docs/inception/application-design/component-dependency.md` (本ファイルはコンポーネント単位の依存マップを Unit 単位で再記述)
> **関連**: `unit-of-work.md` (Unit 定義) / `unit-of-work-story-map.md` (ストーリー → Unit マッピング)

---

## 1. Unit 間依存マトリックス

行: 利用側 (Caller) / 列: 被利用側 (Callee) / `o`: 直接依存 / `-`: 依存なし

| Caller \ Callee | Unit-1 | Unit-2 | Unit-3 | Unit-4 | Unit-5 | Unit-6 | Unit-7 | Bedrock | DynamoDB | 外部天気 API | S3 | CloudFront | EventKit | UNUserNotif | HealthKit | CoreLocation |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **Unit-1** (Mobile) | — | o (HTTPS) | - | o (HTTPS) | - | - | o (HTTPS via CF) | - | - | - | - | o | o | o | o | o |
| **Unit-2** (Dialogue) | - | — | o (in-process) | o (Direct Invoke) | o (in-process) | - | - | o | - | - | - | - | - | - | - | - |
| **Unit-3** (Risk) | - | - | — | - | - | - | - | - | - | - | - | - | - | - | - | - |
| **Unit-4** (History) | - | - | - | — | - | - | - | - | o | - | - | - | - | - | - | - |
| **Unit-5** (External) | - | - | - | - | — | - | - | - | - | o | - | - | - | - | - | - |
| **Unit-6** (Infra) | - | (deploys) | - | (deploys) | - | — | (deploys) | (perms) | (defines) | - | (defines) | (defines) | - | - | - | - |
| **Unit-7** (Catalog) | - | - | - | - | - | - | — | - | - | - | o | (fronted by) | - | - | - | - |

### 1.1 主要依存パスの解説

- **Unit-1 → Unit-2**: HTTPS (API Gateway 経由 / `POST /dialogue`)
- **Unit-1 → Unit-4**: HTTPS (API Gateway 経由 / `POST /selections` `GET /history` `GET /titles` `POST /selections/{id}/achievement`)
- **Unit-1 → Unit-7**: HTTPS (CloudFront 経由 / `GET /titles-catalog.json` / **API Gateway 不経由 / Lambda 不経由**)
- **Unit-2 → Unit-3**: 同一 Lambda 内のローカル関数呼び出し (純粋関数)
- **Unit-2 → Unit-5**: 同一 Lambda 内のクライアントライブラリ呼び出し
- **Unit-2 → Unit-4**: クロス Lambda Direct Invoke (`getRecentSkipPattern()` + `getLastBathTime()` 並列 / FR-07 + DD-03)
- **Unit-4 → DynamoDB**: AWS SDK (META#AFFIRMATIONS パーティションへの Query 含む / DD-02)
- **Unit-5 → 外部天気 API**: HTTPS (API キーは Secrets Manager / SECURITY-09)
- **Unit-6 → 各 Unit**: IaC でデプロイ + IAM 権限定義 (横断的責務)
- **Unit-7 → S3 / CloudFront**: 配信構成 (CloudFront が S3 origin を fronts / OAI 経由 / SECURITY-06)
- **Unit-1 → OS API** (HealthKit / CoreLocation / EventKit / UNUserNotificationCenter): 端末ローカル / 機微データ境界の内側

### 1.2 循環依存チェック

```
Unit-1 → Unit-2 → { Unit-3, Unit-5, Bedrock }
              → Unit-4 → DynamoDB
                 (getRecentSkipPattern + getLastBathTime / DD-03 / 並列 Direct Invoke)
Unit-1 → Unit-4 (via API Gateway) → DynamoDB (Selection / META#AFFIRMATIONS / achievement)
Unit-1 → Unit-7 (via CloudFront) → S3
Unit-1 → OS API (HealthKit / CoreLocation / EventKit / UNUserNotif / on-device)
```

依存グラフは DAG (有向非巡回) であり、**循環依存なし** ✓

---

## 2. データフロー (Unit 単位の機微データ境界)

```
+--- iOS DEVICE (Unit-1) ----------------------------------------+
|                                                                |
|  HealthKit raw → HealthDataAdapter → HealthSummary [SUMMARY]   |
|  EventKit raw  → CalendarDataAdapter → CalendarSummary [SUMMARY]|
|  CoreLocation  → LocationDataAdapter → lat/lon (永続保存しない) |
|  UNUserNotif   → NotificationScheduler (ローカル通知のみ)        |
|                                                                |
|  POST /dialogue { HealthSummary, CalendarSummary, location, deviceUUID }
|  POST /selections { ..., choice, ... }                          |
|  POST /selections/{id}/achievement { achieved }                 |
|  GET /titles-catalog.json (CloudFront 経由 / 機微情報なし)       |
+--------------------|-------------------|-----------------------+
                     |                   |
~~~~~~~~~~~~~~~~~~~~~|~~~~~~~~~~~~~~~~~~~|~~~~~~~~~~~~~~~~~~~~~~~~~ <-- 機微データ境界
                     |                   |
                     v                   v
+- AWS Cloud (Unit-2 / Unit-4 / Unit-5 / Unit-7) ---------------+
|                                                                |
|  Unit-2 Dialogue Lambda                                        |
|    + Unit-3 Risk Calculator (PBT / FR-04)                      |
|    + Unit-5 External Client (天気 API)                          |
|    + buildPrompt (PBT / FR-05)                                 |
|        ↓ Direct Invoke (DD-03 並列)                             |
|        Unit-4 (getRecentSkipPattern + getLastBathTime)         |
|        ↓                                                       |
|        Bedrock InvokeModel (Claude 3.5 / no PII echoed)        |
|                                                                |
|  Unit-4 History & Title Lambda                                 |
|    + evaluateNewTitles (PBT / FR-10)                           |
|    + pickAffirmation (DD-02)                                   |
|    + markAchievement (FR-14)                                   |
|        ↓                                                       |
|        DynamoDB (SSE / Single Table)                           |
|        + META#AFFIRMATIONS (DD-02)                             |
|        + SelectionRecord (achieved 含む / FR-14)                |
|                                                                |
|  Unit-7 Title Catalog Distribution                             |
|        CloudFront (TTL 1h / OAI) → S3 (Versioning / SSE-S3)    |
|        ※ 機微情報なし / 公開静的メタのみ                          |
|                                                                |
|  Unit-6 Infrastructure (CDK / IaC / 横断)                       |
+----------------------------------------------------------------+

External:
  Unit-5 → 外部天気 API (Secrets Manager 経由)
```

### 機微データ境界の Unit 単位での担保

| Unit | 機微情報の扱い |
|---|---|
| **Unit-1** | HealthKit / EventKit / CoreLocation の生データを **端末ローカル限定で保持** / AWS には集計値のみ送信 |
| **Unit-2** | HealthSummary + CalendarSummary + 位置情報 (緯度経度) を受信 / **位置情報は Lambda 実行内のみ使用、永続保存しない** / Bedrock 呼び出し時に PII を含めない (buildPrompt の PBT 検証) / CloudWatch Logs に生レコードを残さない (SECURITY-13) |
| **Unit-3** | 純粋関数 / I/O なし / 機微情報非依存 |
| **Unit-4** | DynamoDB に **集計値 (HealthSummary / EnvironmentData / riskLevel) + 選択 + 称号 + achieved** のみ保存 / **生データは保存しない** / SSE 暗号化 / 端末 UUID を PK |
| **Unit-5** | 緯度経度 → 天気 API へ送信 / 取得 + キャッシュ (30 分) / 永続保存しない |
| **Unit-6** | アプリケーションロジックなし / IAM 最小権限 + Secrets Manager + CloudWatch Logs (機微情報除外) を IaC で定義 |
| **Unit-7** | **公開静的メタのみ** (機微情報なし) / TLS 1.2+ / S3 SSE-S3 / OAI |

---

## 3. 障害伝播 (Unit 間分離)

| 障害 | 分離レベル | フォールバック |
|---|---|---|
| HealthKit 権限拒否 / 取得失敗 | Unit-1 内 | `HealthSummary=null` で対話生成継続 (US-1.1 AC) |
| EventKit 権限拒否 / 取得失敗 | Unit-1 内 | `CalendarSummary=null` で対話生成継続 (US-1.6 AC) / US-5.7 ミニ表示は非表示 |
| CoreLocation 拒否 / 取得失敗 | Unit-1 内 | `location=null` で対話生成継続 (US-1.3 AC) |
| UNUserNotif 権限拒否 | Unit-1 内 | scheduleAchievementCheck() スキップ / `achieved=null` のまま |
| 外部天気 API 失敗 (Unit-5) | Unit-2 内 | `EnvironmentData=null` で対話生成継続 |
| Bedrock 障害 | Unit-2 のみ | フォールバック対話テンプレ or 503 + 再試行案内 (US-2.1 AC) |
| DynamoDB 障害 | Unit-4 のみ (Unit-2 のメイン経路は影響なし) | 5xx |
| DynamoDB META#AFFIRMATIONS Query 失敗 | Unit-4 内 | Lambda バンドルの最小限フォールバック文言で `affirmation` を返す (DD-02) |
| Unit-4 Lambda 障害 (Direct Invoke) | Unit-2 の一部影響 | `recentSkipPattern={skipDays:0, consecutiveSkipDays:0}` + `lastBathTime=null` で対話生成継続 (Functional Design で確定) |
| CloudFront 障害 (Unit-7) | Unit-7 のみ / Mobile UX への影響 | Unit-1 ローカルキャッシュ (前回値) を継続使用 |
| S3 障害 (Unit-7 origin) | Unit-7 のみ | CloudFront cache が生きていればしばらく継続 |

---

## 4. Bolt 1 デプロイ順序 (R8 / R9 への対応)

予選通過後の最初の Bolt で着手する順序。**R8 (AWS アカウント準備) と R9 (Bedrock モデルアクセス申請) を最優先 / 申請待ち中も他 Unit を並行実装可能** にする分離戦略。

| 順序 | Unit | 着手条件 | 並行実装可否 |
|---|---|---|---|
| **1** | **Unit-6 Infrastructure** (の最小スタック) | AWS アカウント取得直後 (R8 最優先) | — |
| **1' (並行)** | (Bedrock 申請) | R9 最優先タスク / Unit-6 着手と同時に申請 | (申請待ち中は 2-5 を並行実装) |
| **2** | **Unit-4 History & Title** (DynamoDB 含む / META#AFFIRMATIONS パーティション初期投入) | Unit-6 で DynamoDB プロビジョン後 / **Bedrock 不要なので独立** | ✓ |
| **3** | **Unit-7 Title Catalog Distribution** (S3 + CloudFront) | Unit-6 で S3+CloudFront プロビジョン後 / **Lambda 経由しない / Bedrock 不要** | ✓ (Unit-4 と並行可能) |
| **4** | **Unit-5 External Client** (天気 API キー取得) | 天気 API キー申請完了後 / Unit-2 にバンドルされる予定だが先行ライブラリ実装可能 | ✓ |
| **5** | **Unit-3 Risk Calculator** (純粋関数 / PBT) | Bedrock 不要 / **単体テスト + PBT で先行検証可能** | ✓ (Unit-2 にバンドルされる予定だが純粋関数として独立検証) |
| **6** | **Unit-2 Dialogue API** (Bedrock 利用) | **Bedrock モデルアクセス承認後 (R9 解消後)** / Unit-3, Unit-5 を内包 | — (Bedrock 待ち) |
| **7** | **Unit-1 Mobile Client** (UI / Adapter / API 呼び出し統合) | Unit-2/4/7 の API 契約が確定後 / Adapter は擬似データモードで先行実装可能 | ✓ (擬似データ DEBUG ビルドで先行) |

### Bolt 1 リスク管理の要点

- **Unit-2 が最大の R9 (Bedrock 申請) リスクを抱える** → Unit-3/4/5/7 を先行する分離が有効
- **Unit-7 は Lambda 経由しないため R9 と完全独立** → AWS アカウント (R8) 取得後すぐに価値が出る
- **Unit-1 は擬似データモード (DEBUG ビルド) で先行実装可能** → R9 申請承認待ち中も UI / 統合テストを進められる
- 並行実装可能な Unit は 2/3/4/5/7 の **5 つ** / R9 申請待ちで詰まらない

---

## 5. Construction Phase per-Unit Loop の対象

各 Unit ごとに以下の per-Unit ステージを回す (構成パターン):

| Unit | Functional Design | NFR Requirements | NFR Design | Infrastructure Design | Code Generation |
|---|---|---|---|---|---|
| Unit-1 | ✓ (UI ロジック / Adapter 戦略 / 擬似データ分布) | ✓ (NFR-USA-01/02/03 / NFR-DAT-02 ローカル境界) | ✓ | — (アプリ単体 / インフラなし) | ✓ |
| Unit-2 | ✓ (プロンプト + buildPrompt PBT) | ✓ (NFR-USA-02 数秒以内 / NFR-CON-01〜04 / SECURITY-13) | ✓ | ✓ (Lambda + API Gateway) | ✓ |
| Unit-3 | ✓ (リスク判定式 + 閾値) | ✓ (純粋関数特性 / PBT) | ✓ | — (Module / Unit-2 にバンドル) | ✓ |
| Unit-4 | ✓ (称号判定 + DDB スキーマ + Affirmation テンプレ) | ✓ (NFR-DAT-01/07 / SECURITY-04/07) | ✓ | ✓ (Lambda + DynamoDB) | ✓ |
| Unit-5 | ✓ (リトライ・キャッシュ戦略) | ✓ (R10 料金対策) | ✓ | — (Module / Unit-2 にバンドル) | ✓ |
| Unit-6 | — (アプリロジックなし) | ✓ (NFR-DAT-03/05 / Security 横断) | ✓ | ✓ (CDK / 全 AWS リソース定義) | ✓ |
| Unit-7 | ✓ (catalog JSON スキーマ + 運用フロー) | ✓ (NFR-USA-02 edge cache / SECURITY-06 OAI) | ✓ | ✓ (S3 + CloudFront / Unit-6 と連携) | ✓ |
| **Build & Test** | — | — | — | — | (全 Unit 横断 / Build + Unit Test + Integration Test + PBT 統合 + プロンプトテスト) |

---

## 6. カバレッジ確認

- すべての Unit (Unit-1〜Unit-7) が依存マトリックスに登場 ✓
- 機微データ境界 (`~~~`) が Unit 単位で明示 ✓
- すべての保存先 (DynamoDB / Secrets Manager / CloudWatch / 端末ローカル / S3 / CloudFront edge cache / iOS UNUserNotif 予約) が記載 ✓
- Bolt 1 デプロイ順序が R8 / R9 リスク管理を踏まえて確定 ✓
- 循環依存なし (DAG 構造) ✓
