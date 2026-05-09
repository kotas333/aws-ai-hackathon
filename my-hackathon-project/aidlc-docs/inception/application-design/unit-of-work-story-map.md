# Unit of Work — ストーリー → Unit マッピング

> **作成日**: 2026-05-09T03:30:00Z
> **対応ルール**: `.aidlc-rule-details/inception/units-generation.md` Step 12-15
> **基底ドキュメント**:
> - `aidlc-docs/inception/user-stories/user-stories.md` (全 21 ストーリー / v3 / M:17, S:4)
> - `aidlc-docs/inception/application-design/components.md` (全 21 ストーリーのコンポーネント割当表)
> **関連**: `unit-of-work.md` (Unit 定義) / `unit-of-work-dependency.md` (Unit 間依存)

---

## ストーリー数の前提

- **総数**: **21** (M:17, S:4 / AI-DLC v3 / Application Design Revision 2 で確定)
- **v1 → v2**: 17 → 18 (削除 3 + 新規 4)
- **v2 → v3**: 18 → 21 (新規 3 / US-1.6, US-3.3, US-5.7 / Application Design Revision 2 で追加)
- **詳細**: `user-stories.md` 「v1 → v2 差分計算」+「v2 → v3 差分計算」セクション参照

---

## ストーリー → Unit マッピング (全 21 件)

| ID | 優先度 | タイトル (短縮) | 主担当 Unit | 副担当 Unit | 主要 FR |
|---|---|---|---|---|---|
| **US-1.1** ★ | M | ヘルスケアデータの自動取得 | **Unit-1** | — | FR-01, 02 |
| **US-1.2** | M | ヘルスケアアクセス許可フロー | **Unit-1** | — | FR-01 |
| **US-1.3** ★ | M | 位置情報と天気データの取得 | **Unit-1** | Unit-5 (天気 API) | FR-03 |
| **US-1.4** | S | 取得データのプレビュー表示 | **Unit-1** | — | FR-01, 03 |
| **US-1.5** ★ | M | 迷惑リスク判定 | **Unit-3** (PBT) | Unit-2 (呼び出し側) | FR-04 |
| **US-1.6** | M | カレンダー連携 (翌日予定) | **Unit-1** (CalendarDataAdapter) | Unit-2 (User Prompt 取り込み) | FR-12 |
| **US-2.1** ★ | M | ジャッジと悪魔の対話メッセージ生成 | **Unit-2** | Unit-1 (表示) | FR-05, 06 |
| **US-2.2** | M | 対話のターン数 (4-6) | **Unit-2** | Unit-1 (表示) | FR-06 |
| **US-2.3** | S | キャラクターの口調を選べる (拡張点) | **Unit-2** (プロンプト) | Unit-1 (UI) | FR-06 |
| **US-2.4** ★ | M | 悪魔の発言トーンシフト | **Unit-2** (プロンプト) | Unit-4 (履歴サマリ提供 / FR-07) | FR-07 |
| **US-3.1** | M | 入る/サボるの選択 | **Unit-4** | Unit-1 (UI) | FR-08 |
| **US-3.2** ★ | M | サボり履歴のカレンダー表示 | **Unit-4** | Unit-1 (UI / 中立配色) | FR-09 |
| **US-3.3** | S | 30 分後の入浴達成確認通知 | **Unit-1** (NotificationScheduler) | Unit-4 (markAchievement) | FR-14 |
| **US-4.1** | M | 称号・バッジの自動付与 | **Unit-4** (PBT) | Unit-1 (表示) | FR-10 |
| **US-4.2** | M | 過激でユーモラスな称号セット | **Unit-4** (動的判定 / PBT) | Unit-7 (静的メタ配信) + Unit-1 (結合表示) | FR-10 |
| **US-5.1** | M | オンボーディング | **Unit-1** | — | NFR-CON-04 |
| **US-5.2** | M | ホーム画面 | **Unit-1** | — | NFR-USA-01 |
| **US-5.3** | M | 設定画面 | **Unit-1** | — | — |
| **US-5.5** ★ | M | コンセプト明示オンボーディング | **Unit-1** | — (R11 対策方針 (a)(c) 参照) | NFR-CON-04 |
| **US-5.6** | M | ダラけ感のある UI 演出 | **Unit-1** | — | FR-11 / NFR-USA-03 |
| **US-5.7** | S | 翌日予定のホーム画面ミニ表示 | **Unit-1** (`getTomorrowMiniSummary` + iOS 標準カレンダー遷移) | — | FR-13 |

> ★ = Mob レビュー重点項目 (`user-stories.md` 「Mob レビュー重点項目」表 参照)

---

## Unit ごとのストーリー集計

### Unit-1: Mobile Client Unit (主担当 12 件)

| 優先度 | ストーリー ID | 件数 |
|---|---|---|
| Must | US-1.1, US-1.2, US-1.3, US-1.6, US-5.1, US-5.2, US-5.3, US-5.5, US-5.6 | 9 |
| Should | US-1.4, US-3.3, US-5.7 | 3 |
| **計 (主担当)** | | **12** |

加えて副担当として US-2.1, US-2.2, US-2.3, US-3.1, US-3.2, US-4.1, US-4.2 (UI 表示) の 7 件 → Unit-1 が UI で関与する全体は 19 件。**Unit-1 は UI 集約 Unit** として最も広範な関与。

### Unit-2: Dialogue API Unit (主担当 4 件)

| 優先度 | ストーリー ID | 件数 |
|---|---|---|
| Must | US-2.1, US-2.2, US-2.4 | 3 |
| Should | US-2.3 | 1 |
| **計 (主担当)** | | **4** |

加えて副担当として US-1.5 (Unit-3 を呼び出し) / US-1.6 (CalendarSummary を User Prompt に取り込み)。

### Unit-3: Risk Calculator Unit (主担当 1 件 / PBT)

| 優先度 | ストーリー ID | 件数 |
|---|---|---|
| Must | US-1.5 (PBT) | 1 |
| **計 (主担当)** | | **1** |

### Unit-4: History & Title Service Unit (主担当 4 件)

| 優先度 | ストーリー ID | 件数 |
|---|---|---|
| Must | US-3.1, US-3.2, US-4.1, US-4.2 | 4 |
| Should | — (主担当なし) | 0 |
| **計 (主担当)** | | **4** |

加えて副担当として US-3.3 (markAchievement 提供 / FR-14) / US-2.4 (履歴サマリ提供 / FR-07) / US-4.2 (称号 ID 一覧 / 静的メタは Unit-7) に関与。

### Unit-5: External Client Unit (副担当のみ / 主担当 0 件)

US-1.3 の天気 API 部分を **副担当** として担当 (Unit-1 が位置情報取得 + 天気データ表示の主担当 / Unit-5 は天気 API クライアントとして Unit-1 + Unit-2 を支援)。

### Unit-6: Infrastructure Unit (横断 / 主担当 0 件)

NFR-DAT-03, NFR-DAT-05 / Security Baseline 横断 / 全 Unit のデプロイを担う。個別ストーリーの主担当は持たない。

### Unit-7: Title Catalog Distribution Unit (副担当のみ / 主担当 0 件)

US-4.2 の称号メタ静的配信部分を **副担当** として担当 (Unit-4 が動的判定の主担当 / Unit-7 は静的メタ配信 / Unit-1 が結合表示)。

---

## カバレッジ集計

| 観点 | 件数 |
|---|---|
| 全ストーリー数 | 21 (M:17, S:4) |
| 主担当が割当済みのストーリー | 21 ✓ (全件) |
| 副担当も含めて整合性確認済み | 21 ✓ |
| 1 つの Unit に主担当のみ | Unit-1: 12 件 (M:9, S:3) / Unit-2: 4 件 (M:3, S:1) / Unit-3: 1 件 (M:1) / Unit-4: 4 件 (M:4) / Unit-5: 0 件 (副担当のみ) / Unit-6: 0 件 (横断) / Unit-7: 0 件 (副担当のみ) |
| 主担当総和 | 12 + 4 + 1 + 4 + 0 + 0 + 0 = **21** ✓ |

**整合確認**: 全 21 ストーリーがいずれかの Unit に主担当として割当済み ✓ (Mobile UI 中心の Unit-1 が 12 件 / Cloud Service 側に 9 件 / 純粋関数 Unit に 1 件)

---

## PBT 対象 3 純粋関数の Unit 帰属確定

| FR | 純粋関数 | 担当 Unit | 関連ストーリー | PBT 検証ルール |
|---|---|---|---|---|
| FR-04 | `calculateAnnoyanceRisk(input)` | **Unit-3** | US-1.5 | PBT-02, 03, 07, 08, 09 |
| FR-05 | `buildPrompt(input)` | **Unit-2** | US-2.1, US-2.4 | PBT-02, 03, 07, 08, 09 |
| FR-10 (動的判定) | `evaluateNewTitles(input)` | **Unit-4** | US-4.1, US-4.2 | PBT-02, 03, 07, 08, 09 |

> 詳細プロパティ識別 (PBT-01) は Functional Design (per-Unit) で実施。Code Generation で PBT 実装。Build & Test でブロッキング検証。

---

## Open Items 担当一覧 (Application Design Section 16 / O-01〜O-15)

Open Items O-01〜O-15 (O-08 Closed / 残り 14 件) の担当 Unit 一覧および詳細は **`unit-of-work.md` 末尾の「Open Items 担当一覧」セクション** を参照してください。Construction Phase の per-Unit Functional Design でクローズしていく対象です。

---

## 全体整合性の最終確認 (ストーリー → Unit マッピング観点)

- ✓ 全 **21 ストーリー** が主担当 Unit に割当済み
- ✓ **17 Must / 4 Should** の内訳保持
- ✓ 主担当総和 = Unit-1: 12 + Unit-2: 4 + Unit-3: 1 + Unit-4: 4 + Unit-5/6/7: 0 = **21** ✓
- ✓ PBT 対象 **3 純粋関数** が Unit-2 / Unit-3 / Unit-4 に帰属確定

> Unit 定義整合 (FR-01〜14 / NFR-USA-DAT-CON / DD-01〜03 / AWS 6 サービスの Unit 配置) は **`unit-of-work.md` の「カバレッジ確認」セクション** を、Unit 間の依存関係・機微データ境界・Bolt 1 デプロイ順序の整合は **`unit-of-work-dependency.md` の「カバレッジ確認」セクション** を参照してください。
