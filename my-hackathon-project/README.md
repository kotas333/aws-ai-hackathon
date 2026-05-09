# 風呂キャンサポーター — AI-DLC プロジェクト

## プロジェクト概要

| 観点 | 内容 |
|---|---|
| **プロジェクト名** | 風呂キャンサポーター |
| **ハッカソン** | AWS Summit Japan 2026 ハッカソン |
| **テーマ** | 「人をダメにするサービス」 |
| **書類審査締切** | 2026-05-10 |
| **手法** | AI-DLC (AI-Assisted Development Lifecycle) + Mob Elaboration v4 確定版を AI-DLC 内に取り込み |
| **プロダクト要旨** | 仕事に疲弊した 20 代社会人を、帰宅後のソファ/ベッドの上でとことんダラけさせるサポーター。**ジャッジ** (旧称: 天使 / データ駆動アナリスト) と **悪魔** (人格的な誘惑者) がヘルスケア・カレンダー・天気を根拠に対話し、ユーザーがどんなにサボっても基本は全肯定する |

---

## ディレクトリ構成

```
my-hackathon-project/
├── README.md                              # ← 本ファイル / プロジェクト全体の入口
├── CLAUDE.md                              # AI-DLC ワークフロー (プロジェクト指示)
│
├── references/                            # Mob Elaboration 確定版 (★ 不改変)
│   ├── intent_v4.md                       # Intent v4 / Mob による合意形成済み
│   └── user_stories_v2.md                 # User Stories v2 / Mob による合意形成済み
│
└── aidlc-docs/                            # AI-DLC ステージ成果物 (本プロジェクトの作業領域)
    ├── aidlc-state.md                     # 各ステージの進捗 ([x]/[~]/[ ]) + 実時刻
    ├── audit.md                           # ユーザー入力 + AI 応答の完全な監査ログ
    │
    ├── inception/                         # 🔵 Inception フェーズ (書類審査対象)
    │   ├── plans/
    │   │   ├── execution-plan.md                    # Workflow Planning 成果物
    │   │   ├── user-stories-assessment.md           # User Stories ステージ実行根拠
    │   │   └── application-design-assessment.md     # Application Design ステージ実行根拠
    │   │
    │   ├── requirements/
    │   │   ├── intent.md                            # references/intent_v4.md のインポートコピー (★ 不改変)
    │   │   ├── requirements.md                      # Requirements Analysis 成果物 (自己完結型)
    │   │   └── requirement-verification-questions.md # Q1〜Q5 検証質問 + 回答
    │   │
    │   ├── user-stories/
    │   │   ├── user-stories.md                      # User Stories 成果物 (全 21 件 / AI-DLC v3)
    │   │   └── personas.md                          # P1 主 + P2 想定外
    │   │
    │   └── application-design/
    │       ├── application-design.md                # Application Design 統合ドキュメント
    │       ├── components.md                        # コンポーネント定義 (C-01〜C-07)
    │       ├── component-methods.md                 # メソッドシグネチャ + I/O
    │       ├── services.md                          # サービス層 (S-01〜S-06)
    │       └── component-dependency.md              # 依存マトリックス + データフロー
    │
    ├── construction/                      # 🟢 Construction フェーズ (予選通過後 / 概略のみ)
    │
    └── operations/                        # 🟡 Operations フェーズ (placeholder)
```

---

## ファイルの役割と参照関係

### Mob Elaboration 確定版 (`references/`) — ★ 不改変

| ファイル | 役割 | 改変ポリシー |
|---|---|---|
| `references/intent_v4.md` | Intent / Requirements に相当する Mob による合意形成済み版 | **不改変** (Mob 確定版) |
| `references/user_stories_v2.md` | User Stories の Mob 確定版 (18 件) | **不改変** (Mob 確定版) |

> **方針**: AI-DLC ステージで参照・取り込みするが、本物のソース・オブ・トゥルースは references/。AI-DLC 側で修正が必要な場合は `aidlc-docs/inception/` 配下のコピーを更新する (元資料は触らない)。

### AI-DLC ステージ成果物 (`aidlc-docs/`)

| ファイル | ステージ | 役割 |
|---|---|---|
| `aidlc-state.md` | 横断 | 各ステージの進捗 ([x]/[~]/[ ]) + 実時刻 + Extension Configuration |
| `audit.md` | 横断 | ユーザー入力 + AI 応答の完全な監査ログ (追記のみ / 過去ログ不改変) |
| `inception/plans/execution-plan.md` | Workflow Planning | 残ステージの実行可否・順序・粒度・成果物・持ち越し判断の決着場所 |
| `inception/plans/user-stories-assessment.md` | User Stories | ステージ実行根拠 + Step 2-14 スキップ根拠 |
| `inception/plans/application-design-assessment.md` | Application Design | ステージ実行根拠 + Step 2-9 スキップ根拠 |
| `inception/requirements/intent.md` | Requirements | references/intent_v4.md の **インポートコピー** (★ 不改変 / 役割: 元資料との一致を保つ) |
| `inception/requirements/requirements.md` | Requirements | Requirements Analysis 正規成果物 / 書類審査者が単体で読める自己完結型 |
| `inception/requirements/requirement-verification-questions.md` | Requirements | 検証質問 Q1〜Q5 + 回答 (Story 数 / Security / PBT / プラットフォーム / AWS) |
| `inception/user-stories/user-stories.md` | User Stories | 全 21 ストーリー (AI-DLC v3 / Application Design Revision 2 で 18→21 に拡張) |
| `inception/user-stories/personas.md` | User Stories | P1 主ペルソナ + P2 想定外ペルソナ (R12 対応) |
| `inception/application-design/application-design.md` | Application Design | 統合ドキュメント (Section 1〜18) / 持ち越し判断決着 / Naming Decision / LLM プロンプト / 機微データフロー / Visual Asset Plan |
| `inception/application-design/components.md` | Application Design | C-01〜C-07 のコンポーネント定義 + 高レベル責務 |
| `inception/application-design/component-methods.md` | Application Design | 共通データ型 + メソッドシグネチャ + I/O 仕様 |
| `inception/application-design/services.md` | Application Design | サービス層 S-01〜S-06 のオーケストレーション |
| `inception/application-design/component-dependency.md` | Application Design | 依存マトリックス + データフロー図 + 機微データ境界 |

### ファイル間の参照関係

```
references/intent_v4.md  ─(import)→  inception/requirements/intent.md
                         ─(基底)─→   inception/requirements/requirements.md
                                      └→ inception/user-stories/user-stories.md
                                      └→ inception/plans/execution-plan.md
                                      └→ inception/application-design/*.md

references/user_stories_v2.md  ─(基底)→ inception/user-stories/user-stories.md
                                          └→ inception/user-stories/personas.md

inception/plans/execution-plan.md  ─(計画)→ inception/application-design/*.md

inception/application-design/application-design.md  ─(統合)→
                          components.md / component-methods.md /
                          services.md / component-dependency.md

aidlc-state.md ←(状態更新)─ 各ステージ完了時
audit.md       ←(追記)─── ユーザー入力 + AI 応答 (毎回)
```

---

## ステージ別の成果物一覧

### 🔵 Inception フェーズ (書類審査対象)

| ステージ | 成果物 | 状態 |
|---|---|---|
| Workspace Detection | (state.md / audit.md 初期化) | ✅ 完了 |
| Reverse Engineering | (Skip / Greenfield) | — |
| Requirements Analysis | `inception/requirements/` 配下 3 ファイル | ✅ 完了 |
| User Stories | `inception/user-stories/` 配下 2 ファイル + `plans/user-stories-assessment.md` | ✅ 完了 |
| Workflow Planning | `inception/plans/execution-plan.md` | ✅ 完了 |
| Application Design | `inception/application-design/` 配下 5 ファイル + `plans/application-design-assessment.md` | ✅ 完了 (2026-05-09T03:30:00Z 承認 / Revision 1 + 2 + 3 (Phase A〜D) 完了) |
| Units Generation | `inception/application-design/unit-of-work*.md` 3 件 + `plans/units-generation-assessment.md` | ✅ 完了 (Generation + Phase E + F + G + 追加 X-3/X-6 完了) |

### 🟣 Additional Deliverable (AI-DLC 標準外)

| 成果物 | 状態 |
|---|---|
| PRFAQ | スコープ外 (主催者明示により書類審査提出対象は Inception フェーズまで / 本選通過後に検討) |

### 🟢 Construction フェーズ (予選通過後 / 概略)

| ステージ | 成果物 | 状態 |
|---|---|---|
| Functional Design (per Unit) | `construction/{unit}/functional-design/` | ⏳ 予選後 |
| NFR Requirements (per Unit) | `construction/{unit}/nfr-requirements/` | ⏳ 予選後 |
| NFR Design (per Unit) | `construction/{unit}/nfr-design/` | ⏳ 予選後 |
| Infrastructure Design (per Unit) | `construction/{unit}/infrastructure-design/` | ⏳ 予選後 |
| Code Generation (per Unit) | アプリ実装 (ワークスペースルート) + `construction/{unit}/code/` 概要 | ⏳ 予選後 |
| Build and Test | `construction/build-and-test/` | ⏳ 予選後 |

### 🟡 Operations フェーズ

placeholder (現状スコープ外)

---

## 改変ポリシーまとめ

| 対象 | ポリシー |
|---|---|
| `references/intent_v4.md` / `references/user_stories_v2.md` | **不改変** (Mob Elaboration 確定版) |
| `aidlc-docs/inception/requirements/intent.md` | **不改変** (元資料コピー / 元資料との一致を保つ) |
| `aidlc-docs/audit.md` | **追記のみ** (Edit/append) / 過去ログ不改変 / Write 全置換禁止 |
| `aidlc-docs/aidlc-state.md` | 各ステージ完了時に Edit で更新 / 実時刻記録 |
| その他 `aidlc-docs/` 配下のステージ成果物 | 該当ステージ進行中 + ステージ間整合のために更新 |

---

## 重要なメソドロジー注記

- **Application Design ステージはコンポーネント定義に責務を限定** する。Unit との対応関係は **Units Generation ステージ** で定義される
- **拡張機能 (Extension)**: Security Baseline (Yes / All blocking) + Property-Based Testing (Yes / Partial) を有効化済み (詳細は `aidlc-state.md`)
- **PBT 対象 3 純粋関数**: `calculateAnnoyanceRisk()` (FR-04 / C-03) / `buildPrompt()` (FR-05 / C-02) / `evaluateNewTitles()` (FR-10 / C-04)
- **機微データ境界**: ヘルスケア生データ + カレンダー生情報 (タイトル/場所/参加者) は端末ローカル限定 / AWS には集計値 (HealthSummary / CalendarSummary) のみ送信 (NFR-DAT-02 / R3 / SECURITY-13)
- **AWS マネージドサービスでの構築 (NFR-DAT-03)**: Bedrock + Lambda + DynamoDB + API Gateway + S3 + CloudFront を採用

---

## クイックリファレンス

- 何かを **設計レベルで知りたい** → `inception/application-design/application-design.md` (統合ドキュメント) から始める
- **要件** を確認したい → `inception/requirements/requirements.md`
- **ストーリー** を確認したい → `inception/user-stories/user-stories.md`
- **実行計画** を確認したい → `inception/plans/execution-plan.md`
- **進捗** を確認したい → `aidlc-state.md`
- **過去の意思決定** を確認したい → `audit.md` (時系列の監査ログ)
