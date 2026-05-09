# Services — サービス層定義とオーケストレーション

> **作成日**: 2026-05-07T15:00:47Z
> **対応ルール**: `.aidlc-rule-details/inception/application-design.md` Step 10
> **関連**: `components.md` (コンポーネント定義) / `component-methods.md` (メソッド契約) / `component-dependency.md` (依存関係)

## 注意

- 本ファイルは **コンポーネントを跨いだサービス層 (オーケストレーション)** を定義する
- AWS サーバーレス構成では **API Gateway がエッジ層 / Lambda がユースケース単位** という標準パターンに従う
- 詳細実装は Functional Design (per-Unit / Construction Phase) で詳細化される

---

## サービス一覧

| ID | サービス名 | エッジ | ユースケース | 担当 |
|---|---|---|---|---|
| **S-01** | Dialogue Service | API Gateway: `POST /dialogue` | ジャッジと悪魔の対話生成 (FR-12 で CalendarSummary 入力 / DD-03 で hoursSinceLastBath 並列取得) | Dialogue Lambda (C-02) |
| **S-02** | Selection Service | API Gateway: `POST /selections` | 選択記録 + 称号付与判定 + **両キャラ全肯定文言の DDB 取得** (AWS-shift) + **FR-14 達成確認スケジュール時刻返却** | History Lambda (C-04) |
| **S-03** | History Service | API Gateway: `GET /history` | 履歴取得 (カレンダー用) | History Lambda (C-04) |
| **S-04** | Awarded Titles Service | API Gateway: `GET /titles` | **獲得称号 ID 一覧** (静的メタ name/description は含まない / AWS-shift) | History Lambda (C-04) |
| **S-05** | Title Catalog Distribution (S-04 AWS-shift) | CloudFront: `GET /titles-catalog.json` | **称号メタ静的配信** (id → name/description) | C-07 (S3 + CloudFront / Lambda 不経由) |
| **S-06** | Achievement Service (**新規 / FR-14 / FR-14**) | API Gateway: `POST /selections/{selectionId}/achievement` | 30 分後通知の **達成 (Yes) / 未達 (まだ) フラグ** を SelectionRecord に記録 | History Lambda (C-04) |

---

## S-01: Dialogue Service (`POST /dialogue`)

### ユーザーシナリオ
US-2.1 + US-2.2 + US-2.4 + US-1.5 を統合して動作する **コア体験のオーケストレーション**

### オーケストレーションフロー

```
[Mobile (C-01)]
   |  POST /dialogue
   |  { deviceUUID, health, location, calendar, characterSetId? }
   |                          ^^^^^^^^ FR-12 (FR-12 / US-1.6)
   v
[API Gateway]
   |  認可: 端末 UUID (SECURITY-04 軽量識別)
   |  スロットリング (R10 料金対策)
   v
[Dialogue Lambda (C-02)]
   |
   +-- (1) location ありなら External (C-05) で fetchWeather()
   |       └ 失敗時は EnvironmentData=null で進行 (US-1.3 AC)
   |
   +-- (2) Risk Calculator (C-03) で calculateAnnoyanceRisk()
   |       └ 純粋関数 / PBT 対象 (FR-04)
   |       └ hoursSinceLastBath は (3-2) で取得した値を渡す (DD-03)
   |
   +-- (3) History & Title (C-04) の getRecentSkipPattern() を内部呼び出し
   |       └ 連続サボり長期化フラグ用 (US-2.4 / FR-07)
   |       └ 注: ここは S-01 の内部呼び出し (S-02/S-03/S-04 を経由しない)
   |
   +-- (3-2) History & Title (C-04) の getLastBathTime() も並列で内部呼び出し (DD-03)
   |       └ 最終 'bath' タイムスタンプ取得 → 現在時刻との差分で
   |          hoursSinceLastBath を計算
   |       └ 履歴に 'bath' なしの場合は null
   |       └ (3) と同じ Direct Invoke パターン (1 回の Lambda invoke で
   |          両方を取得する設計を Functional Design で確定: O-07 関連)
   |
   +-- (4) buildPrompt() で system + user prompt を構築
   |       └ 純粋関数 / PBT 対象 (FR-05)
   |       └ 動的トーンシフト適用 (US-2.4)
   |       └ FR-12: CalendarSummary を User Prompt の [Calendar Summary]
   |          ブロックに含める (タイトル/場所/参加者は含めない)
   |       └ DD-03: hoursSinceLastBath を User Prompt の [Last Bath Time]
   |          ブロックに含める
   |
   +-- (5) Bedrock InvokeModel (Claude Sonnet 4.6) で対話生成
   |       └ 4〜6 ターン (US-2.2)
   |       └ 悪魔最後 (US-2.1 AC) はプロンプトで担保
   |
   +-- (6) レスポンス整形
   |
   v
[Mobile (C-01)]
   <- DialogueResponse { dialogue, riskLevel }
```

### 設計上の重要ポイント

- **応答時間目標**: 数秒以内 (NFR-USA-02) → Bedrock 呼び出しが律速。失敗時はフォールバックテンプレ対話を返す案を Functional Design で詳細化
- **動的トーンシフト** (US-2.4 Must): プロンプト構築時に以下フラグを反映
  - `riskLevel === 'high'` → 悪魔のトーンを健康寄りにシフト
  - `recentSkipPattern.consecutiveSkipDays >= 閾値` → 同上
  - 「悪魔はジャッジのような厳しい説教はしない / 柔らかさを保ちながら内容のみシフト」(US-2.4 AC)
- **キャラクター切替** (US-2.3 Should の拡張点): `characterSetId` で System Prompt テンプレートを切り替え
- **機微情報の最小化** (SECURITY-13): Lambda → CloudWatch Logs には統計値のみ記録、生の health 個別レコードはログに残さない

### 内部依存

- C-03 (Risk Calculator) — 同一 Lambda 内ライブラリ
- C-05 (External Client) — 同一 Lambda 内ライブラリ
- C-04 (History & Title) — **クロス Lambda 内部 API 呼び出し** (Direct Lambda Invoke / 同 VPC 内)

> **トレードオフ**: 「(3) 履歴サマリ取得」を S-03 (`GET /history`) 経由にせず、Dialogue Lambda が History Lambda を直接呼び出すのは、API Gateway → API Gateway の二段ホップを避けてレイテンシを抑えるため (NFR-USA-02 への対応)。代替案として History データを DynamoDB から Dialogue Lambda が直接読む案もあるが、責務分離 (C-04 の称号付与ロジックとデータ層の知識を Dialogue Lambda に漏らさない) を優先して Direct Invoke 案を推奨。Functional Design で確定。

---

## S-02: Selection Service (`POST /selections`) — **AWS-shift 反映**

### ユーザーシナリオ
US-3.1 + US-4.1 + US-4.2 を統合 + **両キャラ全肯定文言を DDB から配信** (AWS-shift)

### オーケストレーションフロー

```
[Mobile (C-01)]
   |  POST /selections
   |  { deviceUUID, choice, health, environment, riskLevel }
   v
[API Gateway]
   |  認可 + スロットリング
   v
[History Lambda (C-04)]
   |
   +-- (1) DynamoDB に SelectionRecord を保存 (FR-08)
   |       └ SSE 有効 (SECURITY-07)
   |       └ 機微情報 (個別 health レコード) は **保存しない** (HealthSummary の統計値のみ / NFR-DAT-02)
   |
   +-- (2) 累計統計を再計算 (DynamoDB から累計値を取得)
   |
   +-- (3) evaluateNewTitles() で新規称号判定
   |       └ 純粋関数 / PBT 対象 (FR-10 動的判定)
   |       └ 事前リスト 10 件以上のみから選択 (R13)
   |
   +-- (4) 新規称号があれば DynamoDB に awardedTitle レコードを保存 (id + awardedAt のみ)
   |
   +-- (5) **pickAffirmation(choice, riskLevel)** で DDB から全肯定文言を取得 (AWS-shift)
   |       └ DynamoDB Query: PK=`META#AFFIRMATIONS`, SK begins_with `AFFIRMATION#<choice>#`
   |       └ 取得結果からランダム 1 件選択 (Lambda 内乱数)
   |       └ DDB 障害時は最小限の Lambda バンドル文言にフォールバック
   |
   |
   +-- (6) **FR-14**: choice='bath' の場合のみ achievementCheckScheduledAt を計算
   |       (recordedAt + 30 分) してレスポンスに含める
   |       (choice='skip' の場合は null)
   v
[Mobile (C-01)]
   <- SelectionResponse {
        selectionId,
        recordedAt,
        newTitles: [{ id, awardedAt }, ...],   // 静的メタは catalog から
        affirmation: { judgeMessage, devilMessage },
        achievementCheckScheduledAt: ISODateTime | null  // FR-14 / choice='bath' のみ
      }

[Mobile (C-01)] (レスポンス受領後)
   |
   +-- choice='bath' で achievementCheckScheduledAt が非 null の場合:
   |   NotificationScheduler.scheduleAchievementCheck({
   |     selectionId,
   |     fireAt: achievementCheckScheduledAt,
   |     bodyText: '<悪魔キャラ寄り文言>',
   |     actionYes: { ... }, actionNo: { ... }
   |   })
   |
   +-- 30 分後 / 通知発火 → ユーザーが Yes/まだ を選択 → submitAchievement(selectionId, achieved)
       → POST /selections/{selectionId}/achievement → S-06 へ
```

### 設計上の重要ポイント

- **アトミシティ**: SelectionRecord 保存と AwardedTitle 付与は **同一 DynamoDB トランザクション** (TransactWriteItems) を推奨。Affirmation 取得は別トランザクションでよい (失敗してもフォールバック可)。Functional Design で詳細化
- **両キャラから全肯定** (US-3.1 AC): **AWS-shift により文言は DDB 由来** (META#AFFIRMATIONS パーティション)。choice 別 (bath / skip) に 10〜20 件のテンプレートを事前投入し、ランダム選択でレスポンス
  - 旧設計 (Mobile 固定文言) からの変更理由: AWS サービス活用度の向上 (DynamoDB 利用箇所増) + 文言更新を Mobile アプリ更新なしで反映可能
  - LLM 不使用のためレイテンシ問題なし (DDB Query は数 ms)
- **「ストーリーの設計原則」と「コア体験の選択原則」の両立**: トーンシフト発動時でも、選択完了後は両キャラ全肯定 (US-2.4 AC)
- **文言の品位担保** (NFR-CON-03): META#AFFIRMATIONS のテンプレートは **事前審査済み** (R13 称号と同種の運用ガードレール)

---

## S-03: History Service (`GET /history`)

### ユーザーシナリオ
US-3.2 (カレンダー表示)

### オーケストレーションフロー

```
[Mobile (C-01)]
   |  GET /history?deviceUUID=XXX&from=YYYY-MM&to=YYYY-MM
   v
[API Gateway]
   v
[History Lambda (C-04)]
   |
   +-- (1) DynamoDB Query (PK=deviceUUID, SK range)
   |       └ 範囲は from-to の月境界
   |
   +-- (2) 日単位集計 (status: bath / skip / none)
   |       └ 同日複数選択 (理論上ありうる) は最後を採用
   |
   v
[Mobile (C-01)]
   <- HistoryResponse { days: [...] }
```

### 設計上の重要ポイント

- **無限保持** (NFR-DAT-01): 削除・期限なし。DynamoDB TTL は使わない
- **中立配色** (US-3.2 / R11): 配色は Mobile 側で実装 (本サービスは状態のみ返す)

---

## S-04: Awarded Titles Service (`GET /titles`) — **AWS-shift 反映**

### ユーザーシナリオ
US-4.1, US-4.2 (称号一覧 / **獲得 ID + 獲得日時のみ**)

### オーケストレーションフロー

```
[Mobile (C-01)]
   |  GET /titles?deviceUUID=XXX
   v
[API Gateway]
   v
[History Lambda (C-04)]
   |
   +-- (1) DynamoDB Query (PK=deviceUUID, SK prefix=TITLE#)
   |
   v
[Mobile (C-01)]
   <- TitlesResponse { titles: [{ id, awardedAt }, ...] }
   |
   |  (Mobile 側で C-07 catalog をキャッシュから参照し、
   |   id → name/description を結合)
```

### 設計上の重要ポイント

- 称号メタ情報 (name / description) は **C-07 (S3 + CloudFront) で配信** (S-04 AWS-shift)
- 旧設計 (Mobile 側のリソースバンドルで管理) からの変更理由: AWS サービス活用度向上 (S3 + CloudFront 追加) + 称号セットの版管理を AWS 側で完結可能
- 採用案の判断: **案 A (S3 + CloudFront)** を採用 (案 B DynamoDB META との比較は S-05 / 本ファイル下部参照)

---

## S-05: Title Catalog Distribution (`GET /titles-catalog.json`) — **新規 / S-04 AWS-shift**

### ユーザーシナリオ
US-4.2 称号メタ情報 (name / description / category) の AWS 側集中管理 + Mobile 側結合表示

### オーケストレーションフロー

```
[Mobile (C-01)]
   |  起動時 + 設定画面の「カタログ更新」ボタン
   |  GET https://<cloudfront-domain>/titles-catalog.json
   |  If-None-Match: <cached ETag>
   v
[CloudFront] -- edge cache hit / TTL 1h
   |
   +- cache miss --> [S3 Bucket / titles-catalog.json (Versioning ON / SSE-S3)]
   |
   v
[Mobile (C-01)]
   <- 200 TitleCatalog { version, entries: [{ id, name, description, category? }] }
   または
   <- 304 Not Modified (cached のまま継続)
```

### 設計上の重要ポイント

- **Lambda / API Gateway を経由しない**: CloudFront → S3 の直接配信。リクエスト数による Lambda コストが発生しない
- **公開コンテンツ**: 機微情報を含まない (NFR-DAT-02 / SECURITY-13 と整合)
- **更新フロー**: 開発者が S3 にアップロード → CloudFront invalidation 実行 (秒〜分単位で edge に反映)
- **可用性**: Mobile はローカルキャッシュ (前回値) を継続使用可能。初回起動時のみ取得失敗が UX に影響
- **ETag 利用**: Mobile 側で ETag を保存し、条件付き GET (If-None-Match) で帯域節約

### 採用案の比較 (案 A vs 案 B / S-04 AWS-shift の検討対象 2)

| 観点 | 案 A: S3 + CloudFront | 案 B: DynamoDB META パーティション |
|---|---|---|
| AWS サービス数 | **+2 (S3 + CloudFront)** | +0 (既存 DynamoDB を拡張) |
| 配信レイテンシ | **edge cache でほぼ瞬時** | DDB Query (warm Lambda で数 ms / cold start で数百 ms) |
| 配信コスト | CloudFront (低単価) + S3 (極低単価) | DDB RCU |
| 更新時の伝搬 | invalidation 必須 (秒〜分) | 即時 (DDB write 完了で次の Query から反映) |
| 版管理 | **S3 versioning + CloudFront invalidation** | DDB Item revision 自管理 |
| 機微情報の混在 | **なし** (専用ストア) | user data DB と同居 (Bucket 分離なし) |
| Mobile 実装 | URL fetch + local cache | API GW + Lambda 経由のため `GET /titles` を拡張 |
| IaC 複雑度 | +S3 Bucket / +CloudFront Distribution / +OAI | +DDB Item の初期投入のみ |
| AWS マネージドサービス利用範囲 | **+2 サービス (S3 + CloudFront)** | 既存 DynamoDB の利用範囲を拡張 |

**推奨: 案 A (S3 + CloudFront)** を採用。

理由:
1. **NFR-DAT-03 (AWS サービスのみ) との整合**: AWS マネージドサービス (S3 + CloudFront) で完結
2. **配信レイテンシ改善 (NFR-USA-02 への寄与)**: edge cache で ms オーダーで配信
3. **責務分離**: 静的メタは専用ストア / user data DB との混在回避
4. **称号メタの版管理を Mobile アプリ更新なしで反映可能** (運用上の柔軟性 / S3 versioning + CloudFront invalidation で完結)
5. **IaC 増分は限定的**: CDK で 30〜50 行程度

トレードオフ受容:
- invalidation の運用手順が必要 (運用ドキュメント化で吸収)
- IaC が +S3/+CloudFront 分増える (Bolt 1 で実装)

---

## S-06: Achievement Service (`POST /selections/{selectionId}/achievement`) — **新規 / FR-14 / FR-14 / US-3.3**

### ユーザーシナリオ
US-3.3 (S) — 30 分後の通知から達成 (Yes) / 未達 (まだ) を SelectionRecord に記録

### オーケストレーションフロー

```
[Mobile (C-01)]
   |  (30 分後の iOS ローカル通知発火 → ユーザーが Yes/まだ を選択)
   |  POST /selections/{selectionId}/achievement
   |  { achieved: true | false }
   v
[API Gateway]
   |  認可: 端末 UUID
   |  注: selectionId が他端末のものでないことを Lambda 内で確認 (SECURITY-04)
   v
[History Lambda (C-04)]
   |
   +-- (1) DynamoDB UpdateItem: 該当 SelectionRecord の achieved フィールド更新
   |       └ 元の choice='bath' は維持 (上書きしない / US-3.3 AC4)
   |       └ Last-write-wins (同一 selectionId に対する複数回呼び出し)
   |
   v
[Mobile (C-01)]
   <- AchievementResponse { recordedAt }
```

### 設計上の重要ポイント

- **元の `choice='bath'` 記録を維持** (US-3.3 AC4): 達成失敗 (achieved=false) でも `choice='skip'` には書き換えない
- **`choice='skip'` の SelectionRecord に対する achievement 呼び出しは無効** (US-3.3 AC5): Lambda 内で choice='bath' であることを検証し、不一致なら 400 を返す (Functional Design で確定)
- **権限チェック**: 通知許可 (UNUserNotificationCenter) はオンボーディングで取得 (US-3.3 備考)。権限拒否時は scheduleAchievementCheck() がスキップされ、本サービスは呼ばれない (achieved=null のまま)
- **achieved=null の意味**: 未確認 (通知が発火していない / ユーザーが応答していない)。称号付与判定 (FR-10) で「決意倒れマスター」のような将来称号を実装する場合に活用可能 (本変更ではスコープ外)

---

## サービス間の独立性 (R8/R9 への配慮)

各サービスは **独立にデプロイ可能** な構成を維持する:

- S-01 (Dialogue) と S-02/S-03/S-04 (History) は別 Lambda (Bedrock 権限 + 重い依存性 vs. DynamoDB のみ + 軽い)
- S-05 (Title Catalog) は **Lambda を経由しない** (S3 + CloudFront のみ) ため Bedrock R9 とも完全に独立
- S-01 が Bedrock モデルアクセス申請 (R9) 待ちでも、S-02/S-03/S-04/S-05 は先行してデプロイ・検証可能 (Bolt 1 で価値を出せる)

---

## サービス層のカバレッジ確認

| サービス | 対応 FR | 対応ストーリー | 対応 NFR |
|---|---|---|---|
| S-01 Dialogue | FR-03, FR-04, FR-05, FR-06, FR-07, **FR-12 (Calendar 入力)** | US-1.3, US-1.5, **US-1.6**, US-2.1, US-2.2, US-2.3 (S), US-2.4 | NFR-USA-02 (応答速度) / NFR-CON-01〜04 (トーン) |
| S-02 Selection | FR-08, FR-10 (動的判定), **FR-14 (achievementCheckScheduledAt 計算)** | US-3.1, **US-3.3 (連携)**, US-4.1, US-4.2 | NFR-DAT-01 (永続性) / NFR-CON-03 (品位 / Affirmation 文言) |
| S-03 History | FR-09 | US-3.2 | NFR-DAT-01 (永続性) |
| S-04 Awarded Titles | FR-10 (獲得 ID 一覧) | US-4.1, US-4.2 | NFR-DAT-01 (永続性) |
| S-05 Title Catalog | FR-10 (静的メタ配信) | US-4.2 | NFR-USA-02 (edge cache) / NFR-CON-03 (品位 / 称号メタ) |
| **S-06 Achievement** | **FR-14 (達成フラグ記録)** | **US-3.3** | NFR-DAT-01 (永続性) |

すべての FR (FR-01, FR-02, FR-11, **FR-13** を除く) が API レベルで提供されることを確認。
- FR-01, FR-02 は Mobile 内部 (`HealthDataAdapter`) のため API 不要
- FR-11 は UI 演出のため API 不要
- **FR-13 (翌日予定ミニ表示) は Mobile 内部 (`getTomorrowMiniSummary()` + iOS 標準カレンダーへの遷移)** のため API 不要 (US-1.6 の取得結果を再利用)
- **AWS マネージドサービス利用範囲**: S-05 追加により S3 + CloudFront を新規にスタックへ追加 (Bedrock + Lambda + DynamoDB + API Gateway + S3 + CloudFront / NFR-DAT-03 整合)
- **FR-14 で API Gateway エンドポイントが 1 つ増加** (`POST /selections/{id}/achievement` / S-06)。Lambda は既存の History Lambda (C-04) を拡張するため新規 Lambda 不要
