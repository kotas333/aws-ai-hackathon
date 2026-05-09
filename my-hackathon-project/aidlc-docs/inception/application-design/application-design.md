# Application Design (Inception / Application Design ステージ成果物 / 統合ドキュメント)

> **作成日**: 2026-05-07T15:00:47Z
> **最終更新**: 2026-05-07T15:00:47Z
> **状態**: Draft (完了承認待ち)
> **手法**: User Stories ステージと同じ方法論 (Step 1 + Step 10-15 / Step 2-9 はユーザー優先事項 5 件で代替)
> **基底ドキュメント**:
> - `aidlc-docs/inception/requirements/requirements.md` (FR-01〜11 / NFR / R1〜R14 / Q1〜Q5 回答)
> - `aidlc-docs/inception/user-stories/user-stories.md` (全 21 ストーリー / AC)
> - `aidlc-docs/inception/user-stories/personas.md` (P1 主 + P2 想定外)
> - `aidlc-docs/inception/plans/execution-plan.md` (重点 3 ステージ + 持ち越し 3 判断 D-01/D-02/D-03)
> - `aidlc-docs/inception/plans/application-design-assessment.md` (Step 2-9 スキップ根拠)
> **対応ルール**: `.aidlc-rule-details/inception/application-design.md` Step 10

---

> **Section 1 (Pending Decisions Resolution) について**: かつて本セクションには User Stories ステージから持ち越された 3 件の判断 (D-01: US-2.4 / D-02: US-5.5 / D-03: US-2.3) を決着させる章が存在したが、AI-DLC のステージ責務分担に従い、これらの判断は本来の上流ステージ (User Stories / Requirements) で確定された記述に統合した。このため Section 1 は欠番。Section 番号 (Section 1.5 / 1.6 / Section 2 以降) は読み手の参照容易性のため不変としている。
>
> - US-2.4 (悪魔のトーンシフト / Must) → `user-stories.md` US-2.4 を参照
> - US-5.5 (コンセプト明示 / Must) + 法的妥当性 → `requirements.md` Risk Register R11 対策方針 (3 段構え) を参照
> - US-2.3 (キャラクター切替 / Should) → `user-stories.md` US-2.3 を参照

---

## Section 1.5. Naming Decision (設計上の命名変更)

### N-01: 「天使」→「ジャッジ」への命名変更 — **【決定】 2026-05-08T02:21:51Z / Application Design ステージ**

| 観点 | 内容 |
|---|---|
| **背景** | Mob Elaboration v4 (`references/intent_v4.md` / `references/user_stories_v2.md`) では「天使と悪魔」として命名されていた |
| **指摘** | Application Design ステージで「天使」が担う役割 (データを根拠に判定する側) と命名 (宗教的メタファー) の乖離が指摘された |
| **決定** | **「天使」を「ジャッジ」に変更** |
| **変更理由** | (1) **役割と命名の一致** — 「データを根拠に判定する」という役割に「ジャッジ (判定者)」が直結 / (2) **R12 (ユーザー層ミスマッチ) リスクの軽減** — 真剣な健康相談アプリと誤解される可能性を下げる (宗教的メタファーは医療文脈と紛れやすい) / (3) **データ駆動 LLM 活用という差別化要因 (`requirements.md` Differentiators) との直接的な整合** — 「ジャッジ」のほうが「データを根拠に判定する」役割を直接的に表現する |
| **キャラクター性** | データを根拠に淡々と判定する **無機質寄りの存在**。人格は持つが、感情的になることはなく、ユーザーを責めない。**悪魔との対比は「無機質な判定者 vs. 人格的な誘惑者」** |
| **変更範囲** | **`aidlc-docs/` 配下の AI-DLC ステージ成果物のみ**: `requirements.md` / `user-stories.md` / `personas.md` / `execution-plan.md` / `application-design.md` (本ファイル) / `components.md` / `services.md` / `component-methods.md` / `component-dependency.md` / `application-design-assessment.md` / `user-stories-assessment.md` |
| **変更しないファイル** | `references/intent_v4.md` および `references/user_stories_v2.md` は **Mob Elaboration 確定版として不改変** (本プロジェクトの基本ポリシー)。`aidlc-docs/inception/requirements/intent.md` は元資料コピーのため不改変 (役割分担表に従う) |
| **型値の変更** | `'angel'` → `'judge'` / `AngelPersona` → `JudgePersona` |
| **System Prompt の変更** | 旧 `Angel: gentle health advisor based on data` → 新 `Judge: data-driven analyst who calmly explains health/environmental data, never blames the user` |

> **Section 1.5 のスコープ**: 本セクションは **命名変更 (Naming Decision)** のみを扱う。設計判断 (Design Decisions) は Section 1.6 を参照。FR / US の追加・確定 (FR-12 / FR-13 / FR-14 / US-1.6 / US-3.3 / US-5.7 / Q4=A 確定など) は本ステージの判断記録ではなく `requirements.md` / `user-stories.md` に最初から確定された記述として組み込まれている。

---

## Section 1.6. Design Decisions (Application Design ステージで確定した設計判断)

本ステージで確定した **設計判断** を記録する。FR / US の追加・確定は上流 (Requirements / User Stories) の責務に属するため本セクションには含まない (上流ステージ成果物を参照)。

### DD-01: コンポーネント追加 — C-07 Title Catalog Distribution (旧 N-02 / S-04 AWS-shift)

| 観点 | 内容 |
|---|---|
| **背景** | 称号メタ (id / name / description / category) を Mobile 側のリソースバンドルから AWS 側集中管理に移行する。称号は機微情報を含まない静的データであり、運用上は文言更新を Mobile アプリ更新なしで反映可能にしたい |
| **決定** | **新規コンポーネント C-07 (S3 + CloudFront) を追加し、称号メタを AWS 側で管理する** |
| **採用案** | **案 A (S3 + CloudFront)** — 詳細トレードオフは `services.md` S-05 セクション参照 |
| **採用理由** | (1) **NFR-DAT-03 (AWS サービスのみ) との整合** — AWS マネージドサービス (S3 + CloudFront) で完結 / (2) **edge cache でレイテンシ改善 (NFR-USA-02 への寄与)** — CloudFront edge から ETag 対応で配信 / (3) **静的メタと user data DB の責務分離** — DynamoDB (ユーザー個別データ) と混在させない / (4) **称号メタの版管理を Mobile アプリ更新なしで反映可能** (運用上の柔軟性 / S3 versioning + CloudFront invalidation で完結) |
| **影響範囲** | C-07 を新規追加 / S-04 (`GET /titles`) のレスポンスを軽量化 (`AwardedTitle = { id, awardedAt }`) / Mobile が C-07 catalog をローカルキャッシュで結合 / **コンポーネント数 7** (Unit 分解は Units Generation ステージの責務) |

### DD-02: S-02 全肯定文言の AWS 寄せ (旧 N-03 / DDB META#AFFIRMATIONS)

| 観点 | 内容 |
|---|---|
| **背景** | 選択後の両キャラ全肯定メッセージ (US-3.1 AC) の文言を **Mobile アプリ更新なしで反映可能** にする運用要請。LLM 再呼び出しは NFR-USA-02 (応答速度: 数秒以内) の観点で不適 |
| **決定** | **選択後の両キャラ全肯定メッセージを DDB 由来 (META#AFFIRMATIONS パーティション) で管理する** |
| **実装** | DynamoDB シングルテーブルに **META#AFFIRMATIONS パーティション** を配置。choice 別 (bath / skip) に 10〜20 件のテンプレート。Selection Lambda がランダム 1 件選びレスポンスに含める |
| **採用理由** | (1) **NFR-DAT-03 (AWS サービスのみ) との整合** — DynamoDB の既存利用範囲内で完結 / (2) **文言更新を Mobile アプリ更新なしで反映可能** (運用柔軟性 / 文言は事前審査済みテンプレートとして DDB Item を追加・編集) / (3) **LLM 不使用のためレイテンシ問題なし** (DDB Query は数 ms / NFR-USA-02 への影響なし) / (4) **既存の DynamoDB シングルテーブル設計との整合性** (新規スタック追加なしで実現可能 / META#AFFIRMATIONS パーティションを単一テーブル内に配置) |
| **影響範囲** | C-04 に `pickAffirmation()` を配置 / S-02 のレスポンスに `affirmation: { judgeMessage, devilMessage }` を含める / DDB スキーマに META#AFFIRMATIONS を含める |

### DD-03: hoursSinceLastBath の取得元 (旧 N-06)

| 観点 | 内容 |
|---|---|
| **背景** | 迷惑リスク判定 (FR-04 / US-1.5) の入力 `hoursSinceLastBath` の取得元を本ステージで明確化する |
| **設計** | (1) ユーザーが決意ボタン (`POST /selections { choice: 'bath' }`) をタップした時刻を **C-04 が DynamoDB に記録** (既存の SelectionRecord に該当) / (2) `POST /dialogue` 時に **C-02 が C-04 から最終 'bath' タイムスタンプを取得** (内部 API: `getLastBathTime(deviceUUID)` を C-04 に配置) / (3) 取得時刻と現在時刻の差分から `hoursSinceLastBath` を計算 / (4) 初回利用時 / 履歴に 'bath' なしの場合は `null` |
| **設計反映** | `component-methods.md` C-04 に `getLastBathTime()` メソッド配置 / `services.md` S-01 オーケストレーションに「(3-2) C-04 の `getLastBathTime()` も内部呼び出し」を `getRecentSkipPattern()` と並列で配置 |

> **設計判断 DD-01〜DD-03 共通の制約**: ヘルスケア生データ + カレンダー生データのローカル限定原則 (NFR-DAT-02 / R3 / SECURITY-13) は **絶対に維持**。HealthSummary / CalendarSummary への集計処理は Mobile 側 / HealthKit / CoreLocation / EventKit / UNUserNotificationCenter / UI / オンボーディング / カレンダー表示は Mobile 側に正しく配置される構成を維持。AWS 寄せの対象は機微情報を含まない処理 (Affirmation テンプレート / 称号メタ) のみ。

---

## Section 2. Methodology Choice (設計様式)

`application-design-assessment.md` の Methodology Choice 章と整合。

### 2.1 アーキテクチャ様式

| 様式 | 採用理由 |
|---|---|
| **AWS Serverless** (Lambda + API Gateway + DynamoDB) | NFR-DAT-03 「AWS サービスのみ」/ R8・R9 への対応が容易 / ハッカソン速度 |
| **Event-Driven** (リクエスト駆動 / 状態を持たない LLM 呼び出し) | Mobile からのリクエスト単位で完結する単純なフロー |
| **Adapter Pattern** (HealthKit / EventKit / 擬似データモード) | Q4 回答 (iOS 確定 + DEBUG ビルド擬似データモード) を Mobile 内で抽象化 |
| **Domain-Driven の軽量適用** (ジャッジ・悪魔 / 動的トーンシフト / 称号) | Mob Elaboration の中核ドメインを言語化 / US-2.3 拡張容易性を担保 |

### 2.2 設計の優先順位 (明示的制約に基づく順位付け)

各設計判断は以下の優先順位で評価する。優先順位は明示的制約 (NFR / Risk Register / 主催規約) に基づき、上位ほど厳格に守る:

1. **機微情報のローカル保護** (NFR-DAT-02 / R3 / SECURITY-13) — ヘルスケア生データ + カレンダー生データ (タイトル / 場所 / 参加者) は端末ローカル限定。AWS には集計値 (HealthSummary / CalendarSummary) のみ送信。Section 10 機微データフローで端末ローカル境界を視覚化して担保
2. **応答速度の確保** (NFR-USA-02 / 数秒以内) — Direct Invoke (Dialogue Lambda → History Lambda) / 並列化 (`getRecentSkipPattern()` + `getLastBathTime()`) / CloudFront edge cache (C-07 catalog 配信) / DDB 由来 Affirmation (LLM 再呼び出し回避) を Section 6 + `component-dependency.md` で担保
3. **AWS マネージドサービスでの構築** (NFR-DAT-03 / 主催規約準拠) — AWS Serverless アーキテクチャを採用し、Bedrock / Lambda / DynamoDB / API Gateway / S3 / CloudFront のマネージドサービスで実現
4. **MVP 17 件 (Must) の実装可能性** — Section 4 コンポーネント分解で 7 コンポーネント並行実装可能
5. **R8 (AWS アカウント準備遅延) / R9 (Bedrock モデルアクセス申請遅延) への対応容易性** — `component-dependency.md` Section 6 (デプロイ単位) でコンポーネント独立性を確保。R9 申請待ち中も C-04 / C-05 / C-03 / C-07 を先行実装可能 (C-07 は Lambda 経由しないため R9 と完全独立)
6. **R1 (スコープ過大) のリスク管理** — Should ストーリー (US-1.4 / US-2.3 / US-3.3 / US-5.7) を Must 増の代わりに保留することで、実装スコープを管理可能範囲に維持
7. **拡張容易性** — US-2.3 (キャラ切替 / Should) の拡張点を Section 8 ドメインモデル / `component-dependency.md` Section 7 で明示

---

## Section 3. Architecture Overview (アーキテクチャ俯瞰)

```
+--------------------------------------------------------------+
|                         iOS DEVICE                           |
|                                                              |
|  +-----------------------+   +--------------------------+    |
|  | C-01 Mobile Client    |   | OS-Provided              |    |
|  |  - Onboarding (US-5.5)|<--+ - HealthKit (FR-01,02)   |    |
|  |  - Dialogue UI        |   | - CoreLocation (FR-03)   |    |
|  |  - Calendar (US-3.2)  |   +--------------------------+    |
|  |  - Settings (US-5.3)  |                                   |
|  |  - Adapter Pattern    |                                   |
|  |    (Q4 撤退案対応)    |                                   |
|  |  - TitleCatalogClient |                                   |
|  |    (local cached JSON)|                                   |
|  +-----------+-----------+                                   |
|              |                                               |
+--------------|-----------------------------------------------+
               |                                  |
   API: HTTPS  |  端末 UUID 認可                  | catalog: HTTPS
   [SUMMARY    |  (機微生データは越えない)        | (公開静的メタ)
    only]      |                                  |
               |                                  |
+--------------|----------------------------------|--------------+
|              v          AWS / ap-northeast-1    v              |
|  +-----------------+                  +-----------------+      |
|  | API Gateway     |                  | CloudFront      |      |
|  | (REST)          |                  | (edge cached)   |      |
|  +--+--------------+                  +--------+--------+      |
|     |                                          |               |
|     +---> [C-02 Dialogue Lambda]                v              |
|     |       + C-03 Risk Calc (PBT)        +---------+          |
|     |       + C-05 External Client        | C-07 S3 |          |
|     |       + buildPrompt (PBT)           | Catalog |          |
|     |              |                       | Bucket  |          |
|     |              +---> Bedrock Claude   | Version |          |
|     |              |          3.5         +---------+          |
|     |              +-Direct Invoke->[C-04]                     |
|     |                                                          |
|     +---> [C-04 History & Title Lambda]                        |
|             + evaluateNewTitles (PBT FR-10 動的判定)           |
|             + pickAffirmation (S-02 AWS-shift)                 |
|                    |                                           |
|                    +---> DynamoDB (SSE)                        |
|                            + META#AFFIRMATIONS                 |
|                                                                |
|  +---------------------------------------+                     |
|  | C-06 Infrastructure (CDK / IaC)       |                     |
|  | - IAM (最小権限) / Secrets / CloudWatch|                     |
|  | - S3 Bucket / CloudFront / OAI (C-07) |                     |
|  +---------------------------------------+                     |
|                                                                |
+----------------------------------------------------------------+

External:
  C-05 -> 天気 API (OpenWeatherMap 等 / Secrets Manager 経由)
```

---

## Section 4. Component Decomposition (コンポーネント分解)

| ID | コンポーネント | 配置 | 主要 FR | 担当ストーリー |
|---|---|---|---|---|
| C-01 | Mobile Client | iOS のみ (撤退ルートは Section 11 参照) | FR-01, FR-02, FR-11, FR-12, FR-13, FR-14 (Mobile 側) | US-1.1, 1.2, 1.6, 3.1, 3.3 (Mobile 側), 5.1〜5.6, 5.7 |
| C-02 | Dialogue API | API Gateway + Lambda | FR-05, 06, 07 | US-2.1, 2.2, 2.4 |
| C-03 | Risk Calculator | Lambda 内純粋関数 (PBT) | FR-04 | US-1.5 |
| C-04 | History & Title Service | Lambda + DynamoDB (META#AFFIRMATIONS / S-02 AWS-shift / `achieved` フィールド (FR-14) / `getLastBathTime` (DD-03)) | FR-08, 09, 10 (動的判定), FR-14 (達成フラグ記録) | US-3.1, 3.2, 3.3 (記録側), 4.1, 4.2 |
| C-05 | External Client | Lambda 内ライブラリ | FR-03 | US-1.3 |
| C-06 | Infrastructure | CDK / IaC | NFR-DAT-03, 05 | (横断) |
| **C-07** | **Title Catalog Distribution (S-04 AWS-shift)** | **S3 + CloudFront** | FR-10 (静的メタ配信) | US-4.2 |

> 詳細は `components.md` を参照。本ステージでは **コンポーネント定義** に責務を限定する。Unit との対応関係 (どのコンポーネントが Unit Generation でどのような Unit になるか) は **Units Generation ステージで定義** される。

---

## Section 5. Component Methods (主要メソッド契約 / 抜粋)

詳細は `component-methods.md` を参照。**PBT 対象 3 純粋関数 (FR-04 / FR-05 / FR-10)** を本サマリで強調する。

### PBT 対象 (Partial 適用 / PBT-02, 03, 07, 08, 09)

| FR | メソッド | 所属 | 検証する特性 (PBT) |
|---|---|---|---|
| FR-04 | `calculateAnnoyanceRisk(input)` | C-03 | level ∈ {low, medium, high} の全域性 / 冪等性 / 全欠損時の安全側挙動 / hoursSinceLastBath > 72 で level >= medium |
| FR-05 | `buildPrompt(input)` | C-02 | 同一入力 → 同一出力 (冪等) / 機微情報 (生 health 個別レコード) を含まない / characterSetId 未知時のフォールバック / systemPromptVersion 常在性 |
| FR-10 | `evaluateNewTitles(input)` | C-04 | 出力が事前リスト 10+ 件のみ / 重複付与禁止 / 冪等性 |

### 公開 HTTP API

| Method / Path | 担当 | リクエスト | レスポンス |
|---|---|---|---|
| `POST /dialogue` | C-02 | `{ deviceUUID, health (Summary), location, characterSetId? }` | `{ dialogue (4-6 turns), riskLevel }` |
| `POST /selections` (S-02 AWS-shift) | C-04 | `{ deviceUUID, choice, health, environment, riskLevel }` | `{ recordedAt, newTitles: AwardedTitle[], affirmation: { judgeMessage, devilMessage } }` |
| `GET /history` | C-04 | `?deviceUUID=&from=YYYY-MM&to=YYYY-MM` | `{ days[] }` |
| `GET /titles` (S-04 AWS-shift) | C-04 | `?deviceUUID=` | `{ titles: AwardedTitle[] }` (id + awardedAt のみ / 静的メタは catalog から) |
| **`GET /titles-catalog.json`** (新規 / S-05) | **C-07 (CloudFront)** | (なし) | `TitleCatalog { version, entries: [{ id, name, description, category? }] }` |

> 4 + 1 エンドポイント構成で MVP 全 17 ストーリー (Must) をカバー。`/titles-catalog.json` のみ **API Gateway 不経由 / Lambda 不経由** で CloudFront → S3 の直接配信 (S-04 AWS-shift)。US-2.3 のキャラ切替は `POST /dialogue` のオプションパラメータで対応 (US-2.3 拡張点)。

---

## Section 6. Service Layer (サービス層 / オーケストレーション)

詳細は `services.md` を参照。

| ID | サービス | エッジ | ユースケース |
|---|---|---|---|
| S-01 | Dialogue | API Gateway: `POST /dialogue` | ジャッジと悪魔の対話生成 (US-2.1, 2.2, 2.4 + US-1.5 統合) |
| S-02 | Selection (**AWS-shift**) | API Gateway: `POST /selections` | 選択記録 + 称号付与 + **両キャラ全肯定文言の DDB 取得** (US-3.1, 4.1, 4.2 統合) |
| S-03 | History | API Gateway: `GET /history` | カレンダー履歴 (US-3.2) |
| S-04 | Awarded Titles (**AWS-shift**) | API Gateway: `GET /titles` | **獲得称号 ID 一覧** (静的メタは含まない) |
| **S-05** | Title Catalog Distribution (**新規**) | CloudFront: `GET /titles-catalog.json` | **称号メタ静的配信** (id → name/description / API Gateway/Lambda 不経由) |

### S-01 のオーケストレーション (コア体験)

```
Mobile -> POST /dialogue
  -> API Gateway (認可 + スロットリング)
  -> Dialogue Lambda
       (1) External で天気取得 (location ありなら)
       (2) Risk Calculator で迷惑リスク判定 (PBT FR-04)
       (3) History Lambda Direct Invoke で連続サボりサマリ取得 (FR-07)
       (4) buildPrompt() で System+User Prompt 構築 (PBT FR-05)
            └ 動的トーンシフト適用 (US-2.4 Must)
       (5) Bedrock Claude 呼び出し
       (6) レスポンス整形 (悪魔最後 / 責めない)
  -> Mobile
```

### S-02 のオーケストレーション (AWS-shift 反映)

```
Mobile -> POST /selections
  -> API Gateway (認可 + スロットリング)
  -> History Lambda
       (1) DynamoDB に SelectionRecord 保存
       (2) 累計統計を再計算
       (3) evaluateNewTitles() で新規称号 ID 判定 (PBT FR-10)
       (4) 新規称号があれば AwardedTitle レコード保存 (TransactWriteItems)
       (5) pickAffirmation(choice, riskLevel) で META#AFFIRMATIONS から
            ランダム 1 件取得 (DDB Query)
  -> Mobile
   <- { recordedAt, newTitles: [{id, awardedAt}], affirmation: {judgeMessage, devilMessage} }
```

### S-05 のオーケストレーション (新規 / S-04 AWS-shift)

```
Mobile (起動時 + 設定画面の「カタログ更新」ボタン)
  -> GET https://<cloudfront-domain>/titles-catalog.json
     (If-None-Match: <cached ETag>)
  -> CloudFront (edge cache hit / TTL 1h)
        |
        +-- cache miss --> S3 (titles-catalog.json / Versioning ON)
  <- 200 TitleCatalog または 304 Not Modified

(API Gateway 不経由 / Lambda 不経由 / 機微情報なし)
```

### Direct Invoke の理由 (S-01 (3) について)

API Gateway → API Gateway の二段ホップを避けてレイテンシを抑える (NFR-USA-02 数秒以内)。代替案あり (DynamoDB 直読み) だが、責務分離を優先し Direct Invoke を推奨。Functional Design で確定。

---

## Section 7. Component Dependencies (依存関係 / 機微データ境界)

詳細は `component-dependency.md` を参照。本サマリで境界線のみ強調。

```
端末ローカル境界 (NFR-DAT-02 / R3 / SECURITY-13):
   - HealthKit / CoreLocation の生データ = 端末から出ない
   - AWS には HealthSummary (集計値) のみ送信
   - 位置情報 (lat/lon) は対話生成時のみ AWS で使用、永続保存しない
   - DynamoDB に保存されるのは集計値・選択・称号付与履歴のみ
```

依存関係グラフは DAG (循環なし)、Direct Invoke は Dialogue Lambda → History Lambda の 1 経路のみ。

---

## Section 8. Domain Model (ドメインモデル)

### 8.1 主要ドメイン概念の関係

```
Persona (P1: 疲弊した社会人 / P2: 想定外利用者 [離脱誘導])
   |
   v
DeviceUUID (端末識別 / SECURITY-04)
   |
   v
DialogueSession (対話セッション)                         <-- 永続化しない
   |   - Inputs: HealthSummary, EnvironmentData, AnnoyanceRiskFlag, RecentSkipPattern
   |   - Output: Dialogue (4-6 turns / 悪魔最後 / 責めない)
   |   - Modulator: ToneShiftFlag (US-2.4)
   |   - Selectable: CharacterSet (US-2.3 拡張点)
   |
   +--> uses --> CharacterSet { judge: JudgePersona, devil: DevilPersona }
   |              |
   |              v
   |     JudgePersona  (データ駆動アナリスト / 無機質寄りの判定者 / 責めない)
   |     DevilPersona  (サボり推奨側 / 人格的な誘惑者 / 動的トーンシフト / 最後の発言権)
   |
   v
Selection (入る / サボる)                                <-- DynamoDB 永続化
   |
   v
SelectionHistory (端末 UUID 単位 / 全期間保持 / NFR-DAT-01)
   |
   v
TitleAwardEvaluator (純粋関数 / PBT FR-10)
   |   - Input: CumulativeStats, AlreadyAwardedTitleIds, TodaySelection
   |   - Output: NewTitleIds (事前リスト 10+ から / R13)
   |
   v
AwardedTitle (称号付与履歴 / DynamoDB 永続化)
```

### 8.2 ドメインの中核: 二人キャラクター + 動的トーンシフト

| 概念 | 役割 |
|---|---|
| **JudgePersona (ジャッジ)** | データを根拠に淡々と判定する **無機質寄りの存在**。ヘルスケア + 環境データを根拠に状況を説明 / 感情的にならない / **責めない** (NFR-CON-01) / 過剰な健康指導をしない (NFR-CON-02) / 「ジャッジ」の名は役割と直結する命名 (Section 1.5 N-01 参照) |
| **DevilPersona (悪魔)** | **人格的な誘惑者**。サボり推奨が基本 / **常に最後の発言権** / 全肯定 / **動的トーンシフトで健康寄りにシフトすることがある** (US-2.4 Must) |
| **対比** | **「無機質な判定者 vs. 人格的な誘惑者」** という対比構造を持つ。ユーザーは無機質なデータ提示 (ジャッジ) と人格的な肯定 (悪魔) の両方を同時に受け取る |
| **CharacterSet** | (ジャッジ, 悪魔) のペア。MVP は 'standard' のみ。**ID パラメータで拡張可能** (US-2.3 Should 拡張点 / Section 9 LLM プロンプト構造 参照) |
| **ToneShiftFlag** | データ駆動の入力フラグ。`riskLevel === 'high'` または `recentSkipPattern.consecutiveSkipDays >= 閾値` で発火。System Prompt の振る舞い定義に影響を与える |

### 8.3 P2 (想定外利用者) への設計責務

P2 が誤って利用した場合に害を与えない設計を以下で担保:

| 仕組み | 責務 |
|---|---|
| US-5.5 オンボーディングのコンセプト明示 (R11 対策方針 (a) / requirements.md 参照) | 真剣な健康相談アプリと誤解させない |
| LLM の System Prompt | 真剣な医療アドバイスを返さない (NFR-CON-02) |
| 称号は事前リスト (R13) | 不快な体験を防ぐ |
| 設定画面から US-5.5 再表示 | 使い始めから時間が経った後でも認識できる |

---

## Section 9. LLM Prompt Architecture (中核ドメイン)

### 9.1 プロンプト構造の概念

```
+--- System Prompt (CharacterSet 別 / 固定テンプレート) -------+
|                                                              |
| [Identity]                                                   |
|   You are a dialogue system with two personas:               |
|   - Judge: data-driven analyst who calmly explains health/   |
|            environmental data, never blames the user.        |
|            Inorganic-leaning, factual, non-emotional.        |
|   - Devil: humanlike tempter, laziness advocate,             |
|            always speaks last, affirms whichever choice the  |
|            user makes, never blames the user                 |
|                                                              |
| [Design Principles] (NFR-CON-01〜04 / R7)                    |
|   - Never seriously criticize the user                       |
|   - Devil always has the final word                          |
|   - Both characters affirm whichever choice the user makes   |
|   - Avoid medical-grade advice                               |
|                                                              |
| [Health Consideration Policy] (US-2.4 Must / FR-07)          |
|   When `riskLevel === "high"` OR                             |
|   `consecutiveSkipDays >= threshold`:                        |
|     -> Devil shifts to a softer, health-leaning tone         |
|     -> Devil still maintains affectionate language           |
|     -> Devil does NOT lecture sternly                        |
|                                                              |
| [Tone & Style] (US-2.1 / US-5.5 / NFR-CON-03)                |
|   - Humorous but tasteful                                    |
|   - Self-deprecating-friendly                                |
|   - Never discriminatory or vulgar                           |
|                                                              |
| [Output Format]                                              |
|   - 4-6 turns alternating judge/devil                        |
|   - The last turn MUST be devil                              |
|   - Plain text per turn (no JSON wrapping)                   |
|                                                              |
| [System Prompt Version]                                      |
|   - Always include version string for traceability           |
|                                                              |
+--------------------------------------------------------------+

+--- User Prompt (動的データ / フラグ駆動) ---------------------+
|                                                              |
| [Health Summary] (集計値のみ / NFR-DAT-02)                   |
|   steps, activeEnergyKcal, exerciseMinutes,                  |
|   heartRateAvg, sleepHours, standMinutes                     |
|                                                              |
| [Environment]                                                |
|   tempC, weather, humidityPct                                |
|                                                              |
| [Annoyance Risk Flag] (PBT FR-04 出力)                       |
|   level: low | medium | high                                 |
|   hoursSinceLastBath, movementScore, ...                     |
|                                                              |
| [Recent Skip Pattern] (FR-07 / US-2.4)                       |
|   skipDays, consecutiveSkipDays                              |
|                                                              |
| [Calendar Summary] (FR-12 / US-1.6)                          |
|   isHoliday: bool | null                                     |
|   earliestEventTime: ISO8601 string | null                   |
|   eventCount: number | null                                  |
|   注: タイトル/場所/参加者は端末ローカル限定 / 含めない         |
|                                                              |
| [Last Bath Time] (FR-04 / US-1.5 / Section 1.6 DD-03 参照)   |
|   hoursSinceLastBath: number | null                          |
|   注: C-04 の getLastBathTime() で取得した値                  |
|                                                              |
| [Tone Shift Triggers]                                        |
|   isAnnoyanceHigh: bool                                      |
|   isLongConsecutiveSkip: bool                                |
|   注: Calendar Summary も間接的にトーン判断材料となる         |
|       (例: 翌日休日なら悪魔のトーンがより緩む / Functional    |
|        Design で詳細化)                                      |
|                                                              |
| [Character Set] (US-2.3 拡張点)                              |
|   characterSetId: 'standard' | (future: 'kansai', ...)       |
|                                                              |
+--------------------------------------------------------------+
```

### 9.2 動的トーンシフトの状態遷移

```
基本トーン (Default):
  Judge: 健康データを根拠に淡々と現状を説明する (無機質寄り / 感情なし)
  Devil: 「今日もダラけよう」「何もしないことの価値」を全力推奨 (人格的)

   |
   |  isAnnoyanceHigh=true (riskLevel === 'high')
   |  または isLongConsecutiveSkip=true
   v

シフトトーン (Health-Leaning Devil):
  Judge: 不変 (データ駆動アナリストとして一貫 / 数値を提示)
  Devil: 「さすがに今日は入っといた方がいいよ、明日仕事でしょ?」
         (柔らかさは保ったまま、内容だけ健康寄りにシフト)
         (ジャッジのような厳しい説教はしない / US-2.4 AC)

   |
   |  ユーザーが選択 (入る / サボる)
   v

選択後 (常に):
  両キャラから選択を全肯定する短いメッセージ
  (S-02 AWS-shift により DDB META#AFFIRMATIONS から取得 /
   { judgeMessage, devilMessage } を Selection Lambda がランダム選択 /
   LLM 再呼び出し不要 / Section 6 S-02 オーケストレーション参照)
```

### 9.3 プロンプト品質保証 (R4 / R14 への対応)

| 観点 | 対応 |
|---|---|
| **PBT (FR-05)** | `buildPrompt()` の純粋関数性を担保 (冪等 / 機微情報除外 / フォールバック) |
| **プロンプトテスト** (Bolt 2-3 / R14) | (a) 各 ToneShift 状態で responses を採取 / (b) 「責めない」ガードレールの違反検出 / (c) systemPromptVersion を変えた A/B 比較 |
| **System Prompt の版管理** | `systemPromptVersion` 文字列をプロンプトに常在させ、CloudWatch Logs と紐付けて応答品質をトラッキング |
| **言語モデルの選定** | **Claude Sonnet 4.6** を第一候補 (Sonnet 4.5 の直接後継 / Opus 4.6 相当の知能を低コストで実現 / NFR-USA-02 数秒以内応答 + R10 料金予算と整合)。**Claude Opus 4.7** を高品質オプションとして拡張可能 (1M token context / より複雑な指示への追従性 / 東京リージョン available)。R9 申請完了後にモデル間比較を Bolt で実施 |

---

## Section 10. Sensitive Data Flow (機微データフロー / R3 / NFR-DAT-02)

### 10.1 データ境界の原則

> **原則**: ヘルスケア生データ (HealthKit の個別タイムスタンプ付きレコード) / 位置情報の永続データ / **カレンダー生情報 (タイトル/場所/参加者 / FR-12)** は **端末ローカルから出ない**。AWS に送信されるのは集計値 (HealthSummary / **CalendarSummary**) のみで、これは再識別性が低い。

> **カレンダーデータ (FR-12) も、HealthSummary と完全に同じ境界ポリシーを適用**。

### 10.2 視覚化図 (機微データ境界 `~~~`)

```
+-- iOS DEVICE -----------------------------------------------+
|                                                             |
|  HealthKit (raw records, GPS-tagged steps, etc.)            |
|     |                                                       |
|     v  reduce/aggregate                                     |
|  C-01 HealthDataAdapter                                     |
|     |                                                       |
|     v  produce                                              |
|  HealthSummary { steps, activeEnergyKcal,                   |
|                  exerciseMinutes, heartRateAvg,             |
|                  sleepHours, standMinutes, asOf }           |
|                  ^^^^^^^^^^^                                |
|                  集計値のみ / 個別レコード非含有             |
|                                                             |
|  EventKit (raw events: title, location, attendees)  [FR-12]  |
|     |                                                       |
|     v  reduce/aggregate                                     |
|  C-01 CalendarDataAdapter                                   |
|     |                                                       |
|     v  produce                                              |
|  CalendarSummary { isHoliday, earliestEventTime,            |
|                    eventCount, asOf }                       |
|                    ^^^^^^^^^^^                              |
|                    集計値のみ / タイトル/場所/参加者 非含有  |
|                                                             |
|  CoreLocation (lat/lon)                                     |
|     |  no persistence / use only for next API call          |
|     v                                                       |
|  UNUserNotificationCenter (local notification, on-device)   |
|     | [FR-14] schedule 30-min delayed notification            |
|     v   on choice='bath' / never crosses to AWS             |
|                                                             |
|  POST /dialogue {                                           |
|    HealthSummary, CalendarSummary, location, deviceUUID }   |
|  POST /selections { ..., choice, ... }                      |
|  POST /selections/{id}/achievement { achieved } [FR-14]      |
|                                                             |
+-------------------|-----------------------------------------+
                    |
~~~~~~~~~~~~~~~~~~~~|~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ <--- 機微データ境界
                    |
                    v
+-- AWS / ap-northeast-1 -------------------------------------+
|                                                             |
|  Dialogue Lambda                                            |
|    Use HealthSummary + CalendarSummary for buildPrompt      |
|    Use location only to call External (Weather API)         |
|    location is NOT persisted                                |
|    Calls C-04 getLastBathTime() for hoursSinceLastBath [DD-03]|
|                                                             |
|    Bedrock InvokeModel (no PII echoed)                      |
|    CloudWatch Logs (no raw health/calendar records / no PII)|
|                                                             |
|  History Lambda                                             |
|    Persist SelectionRecord:                                 |
|      { deviceUUID, selectedAt, choice,                      |
|        health (Summary), environment, riskLevel,            |
|        achieved: bool|null }            [FR-14]              |
|    Persist AwardedTitle:                                    |
|      { deviceUUID, titleId, awardedAt }                     |
|    Provide getLastBathTime(deviceUUID)  [DD-03]              |
|    Handle markAchievement(selectionId, achieved) [FR-14]     |
|                                                             |
|  DynamoDB (SSE encrypted, point-in-time recovery)           |
|    Stores ONLY summary values, choice, titles, achieved     |
|                                                             |
+-------------------------------------------------------------+
```

### 10.3 セキュリティ対応マッピング (Security Baseline)

| 拡張ルール | 本ステージでの設計対応 |
|---|---|
| SECURITY-03 (機微情報の最小化) | HealthSummary に集計値のみ含める / 生レコードは AWS に送らない |
| SECURITY-04 (認証境界) | 端末 UUID を API Gateway で認可ヘッダとして検証 / ユーザー登録なし (intent.md 端末 UUID ベース軽量識別) |
| SECURITY-05 (最小権限) | Lambda IAM Role を機能別に分離 (Dialogue Lambda は Bedrock InvokeModel + DynamoDB 不要 / History Lambda は DynamoDB のみ) |
| SECURITY-06 (公開エンドポイント最小化) | API Gateway の 4 エンドポイントのみ公開 / Bedrock は Lambda 経由でのみアクセス |
| SECURITY-07 (保管時暗号化) | DynamoDB SSE 有効 / Secrets Manager で API キー管理 |
| SECURITY-09 (シークレット管理) | 天気 API キーは Secrets Manager / Lambda 環境変数経由で取得 / コードに含めない |
| SECURITY-13 (ログの機微情報除外) | CloudWatch Logs には統計値のみ / 生レコード・端末識別子のフルダンプは記録しない / プロンプトテスト時は別ログ系統 |

> **N/A の Security ルール**: 本ステージで設計範囲外のもの (例: WAF, DDOS 対応の詳細) は Infrastructure Design (per-Unit / Construction Phase) で詳細化。

### 10.4 PBT (Partial) 適用範囲との整合

機微データフローと PBT 対象は以下のように整合:
- C-03 `calculateAnnoyanceRisk()` の入力は HealthSummary (機微情報除去済み) のみ
- C-02 `buildPrompt()` の出力は機微情報を含まない (PBT 検証特性)
- C-04 `evaluateNewTitles()` は端末 UUID を入力に取らず、累計統計のみで動作 (機微情報非依存)

---

## Section 11. Q4 Implementation Plan (iOS 確定)

### 11.1 確定したプラットフォーム (Q4 = A)

| 項目 | 内容 |
|---|---|
| **プラットフォーム** | **iOS のみ確定** (Q4 = A) |
| **言語 / フレームワーク** | **Swift / SwiftUI** |
| **OS API 依存** | HealthKit (FR-01,02) / CoreLocation (FR-03 の位置取得) / **EventKit (FR-12)** / **UNUserNotificationCenter (FR-14)** |
| **撤退ルート** | **「iOS のまま擬似データモード (DEBUG ビルド)」** (Web 版撤退案は採用しない / R2 と整合) |
| **R5 (プラットフォーム判断の遅延)** | **N/A (解消)** |

### 11.2 Adapter パターンの責務

Adapter は以下 2 つの責務を持つ:

1. **テスタビリティ**: ユニットテスト時のモック注入
2. **デモ用擬似データモード**: HealthKit (および EventKit / CoreLocation) が予選実装で難航した場合、**DEBUG ビルドで擬似データを返す `PseudoXXXAdapter` に差し替えてデモ実施可能** (撤退ではなく「デモ用モード」)

```
                +-----------------------------+
                | C-01 Application Core       |
                | (UI / ViewModel /           |
                |  Onboarding / Calendar /    |
                |  NotificationScheduler)     |
                +--------------+--------------+
                               |
                  depends on (interface only)
                               |
                +--------------v--------------+
                | HealthDataAdapter (IF)      |
                | LocationDataAdapter (IF)    |
                | CalendarDataAdapter (IF) (FR-12)
                | NotificationScheduler (IF) (FR-14)
                +--------------+--------------+
                               |
                  implemented by (substitutable)
                               |
            +------------------+------------------+
            |                                     |
+-----------v-----------+         +---------------v---------------+
| Production (Release)  |         | DEBUG Build (Pseudo Mode)     |
|  - HealthKitAdapter   |         |  - PseudoHealthAdapter        |
|  - CoreLocationAdapter|         |  - PseudoLocationAdapter      |
|  - EventKitAdapter    |         |  - PseudoCalendarAdapter      |
|  - LocalNotification  |         |  - LoggingNotification        |
|    Scheduler          |         |    (logs instead of firing)   |
+-----------------------+         +-------------------------------+
                                  | Test Build (Mocked)           |
                                  |  - MockHealthAdapter          |
                                  |  - ... (per test fixture)     |
                                  +-------------------------------+
```

### 11.3 デモ用擬似データモード発動時の影響範囲

| 影響 | 内容 |
|---|---|
| **C-01 内のみ** | Application Core (UI / ViewModel) は不変 / Adapter 実装のみ差し替え (DEBUG ビルド) |
| **C-02〜C-07 不変** | API 契約・サーバーレス構成・DynamoDB スキーマすべて不変 |
| **デプロイ形態** | iOS の DEBUG ビルド (TestFlight 内輪配信 or Xcode 直配布)。**Web 版は作らない** |
| **PRFAQ への記載** | Internal FAQ で「**撤退ではなくデモ用モード**: 実装難航時に DEBUG ビルドで擬似データに差し替えてデモ実施可能」と記載 |

### 11.4 設計上の利点

- **R5 (プラットフォーム判断の遅延) 解消**: Bolt 1 開始時点で iOS 確定 / 比較検討の時間を省略
- **R2 リスクは緩和**: iOS HealthKit 一択化により Health Connect 比較検討は不要 / 撤退ルートも iOS 内で完結
- **テスト容易性**: 同じ Adapter パターンで Mock 注入が可能 → ユニットテスト品質向上
- **PBT 対象 3 関数 (FR-04/05/10) はプラットフォーム非依存**: 純粋関数として変更なし

---

## Section 12. Q5 Implementation Plan (AWS 想定環境 + 予選通過後の最優先タスク)

### 12.1 書類審査段階での記載 (想定環境)

| 項目 | 想定環境 | 予選通過後のアクション |
|---|---|---|
| AWS アカウント | 予選通過後の最初の Bolt で準備 | Bolt 1 最優先 / R8 |
| Bedrock モデルアクセス (Claude Sonnet 4.6 第一候補 + Claude Opus 4.7 拡張オプション) | ap-northeast-1 で予選通過後の Bolt で申請 | Bolt 1 最優先 / R9 |
| リージョン | ap-northeast-1 (Tokyo) 確定 (NFR-DAT-05) | — |
| IAM 構成 | 最小権限ロール (Dialogue Lambda / History Lambda 別) | Bolt 1 で IaC として実装 |
| シークレット | 天気 API キーを Secrets Manager で管理 | Bolt 1 で天気 API も申請 |

### 12.2 Bolt 1 タスク順序 (R8/R9 リスク並行緩和)

1. AWS アカウント作成 / 請求アラーム設定 (R10 対策)
2. Bedrock Claude モデルアクセス申請 (R9) **← 並行で 3-5 を進める**
3. **C-06 Infrastructure**: 基本 IaC 雛形作成
4. **C-04 + C-05 + C-03** 先行実装 (Bedrock 不要)
5. C-01 の Adapter インターフェース + UI 雛形
6. (Bedrock 承認後) **C-02** 実装 + プロンプトテスト

> R9 申請待ちの 1〜数日を C-04/C-05/C-03/C-01 の先行実装に充てることで、Bolt 1 を空回りさせない。

### 12.3 PRFAQ への記載

PRFAQ ステージで上記を「Internal FAQ: 予選通過後の最優先タスク」として明記する (R11 対策方針 (c) 法務観点レビューと並んで Bolt 1 最優先タスク群)。

---

## Section 13. NFR Mapping (NFR の設計反映)

| NFR ID | 内容 | 本設計での対応 |
|---|---|---|
| NFR-USA-01 | 片手操作・最小タップ | C-01 ホーム画面でメインボタンが大きく中央配置 (US-5.2 AC) |
| NFR-USA-02 | 応答速度: 数秒以内 | S-01 Direct Invoke / 並列化 / Bedrock タイムアウト管理 |
| NFR-USA-03 | ダラけ感のある演出 | C-01 で US-5.6 の演出を最低 3 つ実装 |
| NFR-DAT-01 | データ永続性: 無限保持 | DynamoDB TTL なし / カレンダーは月単位スクロールで全期間アクセス可 |
| NFR-DAT-02 | プライバシー: ヘルスケア生データはローカル | Section 10 機微データフローで担保 |
| NFR-DAT-03 | AWS サービスのみ | C-02〜C-07 すべて AWS マネージドサービスを採用: Bedrock / Lambda / DynamoDB / API Gateway / S3 / CloudFront |
| NFR-DAT-04 | 日本語のみ | プロンプトテンプレート・UI コピー・称号名すべて日本語 |
| NFR-DAT-05 | リージョン: ap-northeast-1 | C-06 で全リソースを ap-northeast-1 に固定 |
| NFR-CON-01 | 真剣に責めない | LLM System Prompt の Design Principles で担保 (Section 9) |
| NFR-CON-02 | 過剰な健康指導をしない | LLM System Prompt の Tone & Style で担保 (Section 9) |
| NFR-CON-03 | ユーモアは品位の範囲内 | 称号事前リスト (C-07 catalog) + LLM プロンプトの Tone & Style + **Affirmation テンプレート事前審査** (META#AFFIRMATIONS / S-02 AWS-shift) |
| NFR-CON-04 | 真剣相談ユーザーへの安全配慮 | US-5.5 オンボーディング + Section 8.3 P2 への責務 + R11 対策方針 (c) 法務観点レビュー |

---

## Section 14. PBT Target Mapping (PBT 対象 3 関数の設計上の位置)

| PBT 対象 | 純粋関数 | 所属コンポーネント | 検証する特性 (PBT-02, 03, 07, 08, 09) |
|---|---|---|---|
| FR-04 | `calculateAnnoyanceRisk(input)` | C-03 Risk Calculator | level の全域性 / 冪等性 / 全欠損時の安全側挙動 / 経過時間 > 72h で level >= medium |
| FR-05 | `buildPrompt(input)` | C-02 Dialogue API | 冪等性 / 機微情報除外 / characterSetId フォールバック / systemPromptVersion 常在 |
| FR-10 (動的判定) | `evaluateNewTitles(input)` | C-04 History & Title | 出力が事前リスト 10+ のみ (C-07 catalog のソース・オブ・トゥルース) / 重複付与禁止 / 冪等性 |

> **注**: `pickAffirmation()` (S-02 AWS-shift) は I/O (DDB Query) を含むため PBT 対象外。Functional Design で個別テスト戦略を策定。FR-10 静的メタ配信 (C-07) はデータ配信のみで純粋関数を含まないため PBT 対象外。

> **Functional Design (per-Unit) で PBT-01 のプロパティ識別を実施し、Code Generation で PBT 実装を行う。本ステージでは「どの関数が PBT 対象か」と「どんな特性を検証するか」までを確定**。

---

## Section 15. Compliance Summary (Application Design ステージ時点)

### 15.1 Security Baseline (Yes / All blocking)

| Rule | Status | Rationale |
|---|---|---|
| SECURITY-03 (機微情報の最小化) | **Compliant (設計方針確定)** | Section 10 で HealthSummary 集計値のみ AWS 送信 / 生レコード非送信を明記 |
| SECURITY-04 (認証境界) | **Compliant** | 端末 UUID 認可を API Gateway で検証する設計 |
| SECURITY-05 (最小権限) | **Compliant** | Dialogue Lambda / History Lambda の IAM Role を機能別に分離 |
| SECURITY-06 (公開エンドポイント最小化) | **Compliant** | API Gateway 4 エンドポイントのみ公開 / Bedrock は Lambda 経由のみ |
| SECURITY-07 (保管時暗号化) | **Compliant** | DynamoDB SSE 有効 / Secrets Manager で API キー管理 |
| SECURITY-09 (シークレット管理) | **Compliant** | 天気 API キーは Secrets Manager / コードに含めない |
| SECURITY-13 (ログの機微情報除外) | **Compliant** | CloudWatch Logs に統計値のみ記録する方針 |
| SECURITY-01, 02, 08, 10, 11, 12, 14, 15 | **N/A at this stage** | Infrastructure Design / Code Generation / Build & Test で詳細化 |

> ブロッキング所見なし。本ステージで設計レベルの方針を確定。

### 15.2 Property-Based Testing (Partial)

| Rule | Status | Rationale |
|---|---|---|
| PBT-01 (プロパティ識別) | **Pending Functional Design** | 本ステージでは PBT 対象 3 関数を確定。プロパティ識別は Functional Design (per-Unit) で実施 |
| PBT-02, 03, 07, 08, 09 | **Pending Code Generation / Build & Test** | テスト実装は Code Generation で実施 |

> ブロッキング所見なし。

---

## Section 16. Open Items (Functional Design へ持ち越し)

Application Design ステージ完了時点で、以下の領域は次ステージ以降で詳細化される未解決事項として残されています:

- 迷惑リスク判定の具体閾値 (FR-04 関連)
- 動的トーンシフト発火条件 + プロンプト具体例 (FR-07 関連)
- DynamoDB シングルテーブル PK/SK 設計 (`achieved` フィールド (FR-14) + META#AFFIRMATIONS パーティション (DD-02) 含む)
- 称号付与条件の具体ロジック (FR-10 関連)
- フォールバック対話テンプレ (Bedrock 障害時)
- キャラクター切替 UI の MVP 扱い (プレースホルダ vs. 後置)
- History → Dialogue Direct Invoke vs. DynamoDB 直読みの最終判断
- META#AFFIRMATIONS choice 別テンプレート初期文言 (NFR-CON-03 品位)
- `titles-catalog.json` 初期メタ確定 (name / description / category)
- CloudFront invalidation 運用手順ドキュメント化
- TitleCatalogClient タイムアウト・リトライ・ETag 戦略
- 達成確認通知の文言 (悪魔キャラ寄り) + 発火タイミング詳細
- EventKit アクセス時のキャッシュ戦略 (US-1.6 と US-5.7 の重複取得回避)
- 擬似データモード (DEBUG ビルド) の擬似値分布設計

> 詳細な担当 Unit 一覧 (O-01〜O-15 の Unit 帰属確定 / O-08 Closed) は Units Generation ステージ成果物 **`unit-of-work.md` 末尾の「Open Items 担当一覧」セクション** を参照してください。Construction Phase の per-Unit Functional Design でクローズしていく対象です。

### 16.1 本ステージで Closed にした項目 (Section 16 から本文に昇格)

| # | 項目 | 昇格先 |
|---|---|---|
| **DD-03** | hoursSinceLastBath の取得元: C-04 が SelectionRecord ('bath' タイムスタンプ) を保持し、`getLastBathTime(deviceUUID)` で C-02 から取得可能に | Section 1.6 DD-03 / Section 6 S-01 オーケストレーション (3-2) / `component-methods.md` C-04 / `services.md` S-01 |

---

## Section 17. 次のステップ

1. ユーザーが本 `application-design.md` および個別 4 ファイル (`components.md` / `component-methods.md` / `services.md` / `component-dependency.md`) と `application-design-assessment.md` を最終レビュー
2. **Section 1.5 (Naming Decision / N-01)** と **Section 1.6 (Design Decisions / DD-01〜DD-03)** について承認確認 (旧持ち越し判断は上流ステージ user-stories.md / requirements.md に最初から確定された記述として組み込み済み)
3. 承認後、**Units Generation** ステージへ進行 — 本ステージで定義した **7 コンポーネント** (C-01〜C-07) を **Units Generation ステージで Unit として正式に分解・確定** する (本ステージはコンポーネント定義に責務を限定 / Unit 分解は Units Generation の責務)
4. その後、PRFAQ ステージで本ファイルの以下のセクションを Internal FAQ および提出パッケージに翻訳する:
   - Section 9 (LLM プロンプト構造) — AI 活用の中核ドメイン
   - Section 10 (機微データフロー) — プライバシー設計 (NFR-DAT-02 / R3 / SECURITY-13 への対応)
   - Section 11 (Q4 Implementation Plan) — iOS / Swift / SwiftUI 採用と擬似データモードの設計
   - Section 12 (Q5 Implementation Plan) — AWS 想定環境と予選通過後のタスク順序
   - Section 1.5 (Naming Decision) — ジャッジ命名の経緯
   - Section 1.6 (Design Decisions / DD-01〜DD-03) — Application Design ステージで確定した設計判断
   - Section 18 (Visual Asset Plan) — キャラデザの設計レベル要件

---

## Section 18. Visual Asset Plan (キャラデザの設計レベル要件 / Application Design Revision 2 で新規追加)

> **本セクションの粒度**: 印象が伝わる粒度のみを定義する。詳細な絵柄やデザイン案、イラスト発注先、ライセンスは **PRFAQ ステージまたは書類審査提出パッケージ** で決定する。本ステージでは「どのコンポーネントが何を持つか」の責務分担と、各キャラの設計要件のみを明文化する。

### 18.1 キャラデザの責務分担

- **C-01 Mobile Client** が画像/イラストアセットを表示する責任を持つ
- アセットは **アプリバンドル内** に格納 (R13 / 版管理しやすさ / オフライン動作 / 起動時のレイテンシ回避)
- **NFR-USA-03 (ダラけ感のある UI 演出)** と連動: 画面の他要素 (US-5.6 のソファに沈むアニメ等) と統一感を持たせる
- ジャッジ/悪魔の **System Prompt の Identity 記述 (Section 9.1)** とビジュアルが食い違わないよう、Visual Asset と Prompt は **同じ Truth ソース (本セクション + Section 8.2 ドメインモデル)** から派生させる

### 18.2 ジャッジのデザイン要件

- **データ判定者という役割を反映した無機質寄りのキャラ** (Section 8.2 ドメインモデルのキャラ性と整合)
- **タブレットや計測機器を持つ等、データを扱う雰囲気**を演出する小道具
- **表情は冷静・中立寄り** / 感情的な表現を抑制
- カラートーンはクール系 (青〜白〜グレー寄り) を推奨 (詳細は PRFAQ で確定)
- **System Prompt の `Judge: data-driven analyst, inorganic-leaning, factual, non-emotional`** と一貫性を持たせる

### 18.3 悪魔のデザイン要件

- **親しみやすさのある誘惑者キャラ** (Section 8.2 「人格的な誘惑者」と整合)
- **ソファでくつろぐ姿勢など、ダラけ感を体現** する立ち姿
- **動的トーンシフト時 (US-2.4) に表情変化の余地** がある (例: 通常時はニヤけ顔 / シフト時は心配そうな苦笑)
- カラートーンは暖色系 (赤〜紫〜ピンク寄り) を推奨 (詳細は PRFAQ で確定)
- **悪魔最後の発言権原則 (US-2.1 AC)** を Visual でも強調: 対話表示で悪魔のアバターが下部 (最後) に来る配置

### 18.4 各キャラの状態バリエーション (最低限)

各キャラ少なくとも 3 種類の状態バリエーションを用意する:

| 状態 | ジャッジ | 悪魔 |
|---|---|---|
| **通常時** | データ提示中の中立表情 | サボり推奨ニヤけ顔 / ソファに沈む |
| **トーンシフト時 (US-2.4 Must)** | 不変 (常にデータ駆動 / 中立) | 心配そうな苦笑 / 諭し顔 |
| **選択後肯定モード (US-3.1 / S-02)** | 「結果を承認する」中立的なうなずき | 全肯定の親指立て / 寄り添う表情 |

> 上記 3 状態は最小限のバリエーション。アニメーション (NFR-USA-03 ダラけ感の演出) は Construction Phase で詳細化。

### 18.5 アセット管理

| 観点 | 方針 |
|---|---|
| **格納先** | **アプリバンドル内** (R13 / 版管理しやすさ / オフライン動作 / 起動時のレイテンシ回避) |
| **C-07 (Title Catalog) との対比** | **C-07 (称号メタ) は S3+CloudFront 配信**、**Visual Asset (キャラ画像) はアプリバンドル内** という違い。理由: キャラ画像は容量が大きく更新頻度も低いため CloudFront 経由よりバンドル同梱が UX に優れる / 称号メタは小さく更新があり得るため CDN 配信が適する |
| **形式** | PNG (透過) / 必要に応じて Lottie (ダラけ感アニメ用) |
| **配色 / フォント** | NFR-USA-03 ゆるいフォントとの統一 (Construction Phase の UI 設計で詳細化) |
| **キャラ拡張時の対応 (US-2.3 Should 延長)** | 新キャラセット追加時は: (a) アセットをアプリバンドルに追加 / (b) `characterSetId` をプロンプトテンプレートに紐付け (Section 9.1) / (c) Mobile 設定画面に新 ID を露出 |

### 18.6 PRFAQ への引き継ぎ

以下を **PRFAQ ステージまたは書類審査提出パッケージ** で決定 (本ステージのスコープ外):

- 具体的なデザイン案 (絵コンテ / モックアップ)
- イラスト発注先 (社内デザイナー / 外注 / 生成 AI 等)
- ライセンス (商用利用可 / 著作権帰属)
- ジャッジ/悪魔の名前・愛称 (UI 上の表示名 / プロンプト内 vs. Mobile UI 文言の使い分け)
- 動的トーンシフト時の遷移演出の詳細
- ホーム画面の翌日予定ミニ表示 (US-5.7 / FR-13) のレイアウト案

> 書類審査提出時には **本セクションを根拠にデザインモックアップを添付** する想定。
