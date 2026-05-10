# Units Generation Assessment

> **作成日**: 2026-05-09T03:30:00Z
> **目的**: Units Generation ステージの実行根拠と方法論判定 (User Stories / Application Design と同じ「Part 1 スキップ + Part 2 のみ実施」方針) を文書化

## Decision

**Execute Units Generation**: **Yes** (重点ステージ)

**標準 Step 2-11 (Part 1 Planning) のスキップ**: **Yes** (User Stories / Application Design と同じ方法論を継続)

---

## Execute 判定の根拠

| 条件 | 該当 | 根拠 |
|---|---|---|
| マルチコンポーネント構成 | **Yes** | Application Design で C-01〜C-07 の 7 コンポーネントを確定。各コンポーネントは独立にデプロイ可能 |
| Construction Phase の per-Unit Loop が必要 | **Yes** | Functional Design / NFR Requirements / NFR Design / Infrastructure Design / Code Generation を Unit 単位で回す前提 |
| PBT 対象 3 純粋関数の Unit 帰属確定が必要 | **Yes** | FR-04 / FR-05 / FR-10 の Unit 帰属を明示しないと Code Generation での PBT 適用範囲が曖昧 |
| Open Items の Unit 担当一覧化が必要 | **Yes** | Application Design Section 16 の O-01〜O-15 を Functional Design (per-Unit) で扱うため、担当 Unit を確定する必要 |

**Skip 条件への該当**: なし (Application Design で既にコンポーネント定義済み / Construction Phase が per-Unit 構成 / Greenfield)

---

## Step 2-11 (Part 1 Planning) スキップの根拠

User Stories / Application Design ステージと同じ方法論を継続する。

### 標準 Step 3 のカテゴリで本来問うべきことが既に解決済み

| Step 3 のカテゴリ | 本プロジェクトでの解決状況 |
|---|---|
| **Story Grouping** (グルーピング戦略) | Application Design で 7 コンポーネントの担当ストーリーが確定済み (`components.md` 全 21 ストーリーのコンポーネント割当表 / `application-design.md` Section 4)。Unit は **コンポーネントと 1:1 対応** とすることでグルーピング戦略が自明 |
| **Dependencies** (依存関係) | Application Design で `component-dependency.md` に依存マトリックス + データフロー図 + 通信パターン + 障害伝播 + デプロイ順序を確定済み。Unit 間依存は同じ構造を継承 |
| **Team Alignment** (チーム構造) | 本プロジェクトの規模 (小規模チーム) のため Unit ごとの責務分担を厳密に分ける必要なし。AI-DLC ステージとしての Functional Design / Code Generation を per-Unit で回せれば十分 |
| **Technical Considerations** (スケーラビリティ・デプロイ要件) | Application Design で AWS Serverless 構成と Bolt 1 デプロイ順序 (`component-dependency.md` Section 6) を確定済み。Unit 単位のデプロイ独立性は本ステージで再記述 |
| **Business Domain** (ドメイン境界) | Application Design Section 8 ドメインモデルで「ジャッジ × 悪魔 × 動的トーンシフト × 称号」の境界を確定済み |
| **Code Organization** (Greenfield コード組織化) | iOS Swift/SwiftUI (Mobile) + AWS CDK (Infrastructure) という Q4 確定 + Q5 確定で大枠決定済み。Unit ごとのディレクトリ構造は本ステージで補足 |

### ユーザーの優先事項 6 件で代替

ユーザーが Application Design 完了承認時に提示した 6 件の優先事項が、Step 3 で本来問うべき内容を **指示として直接与えている**:

1. 7 コンポーネント → 7 Unit 確定 (Unit 1〜7 の対応一覧明示)
2. PBT 対象 3 純粋関数の Unit 帰属 (FR-04→Unit 3 / FR-05→Unit 2 / FR-10→Unit 4)
3. Open Items O-01〜O-15 の Unit 担当一覧化
4. Inception 成果物として簡潔 / 軽め / 新規発明なし
5. 整合性維持 (7 Unit / 17 Must / 21 ストーリー / AWS 6 サービス / PBT 3 / 機微データ境界)
6. Inception フェーズ完了への道筋 (Units Generation 完了 → PRFAQ ADDITIONAL DELIVERABLE)

---

## 標準ルールステップの取り扱い

| 標準ステップ | 本プロジェクトでの取り扱い |
|---|---|
| Step 1: Create Unit of Work Plan (Part 1) | 本ファイル (assessment) で実施 (Execute 判定 + Step 2-11 スキップ判定) |
| Step 2-11: Plan + Q&A loop + 承認 | **スキップ** (上記根拠 + ユーザー優先事項 6 件で代替) |
| Step 12-15: Generation (3 ファイル生成) | 実施: `unit-of-work.md` / `unit-of-work-dependency.md` / `unit-of-work-story-map.md` を `aidlc-docs/inception/application-design/` 配下に配置 (ルール Step 2 の指定パス準拠) |
| Step 16-18: 完了承認 | 通常どおり実施 |
| Step 19: Update Progress | aidlc-state.md 更新で実施 |

---

## Methodology Choice

### Unit 分解戦略: コンポーネント 1:1 マッピング

| 観点 | 採用方針 |
|---|---|
| **マッピング** | Application Design の **7 コンポーネント (C-01〜C-07)** を **7 Unit (Unit 1〜Unit 7)** に 1:1 でマッピング |
| **Unit 境界** | コンポーネント境界を踏襲 (再分解なし) |
| **デプロイ独立性** | コンポーネント独立性を Unit 独立性として継承 (Bolt 1 順序は `component-dependency.md` Section 6 を参照) |
| **Construction Phase 構成** | per-Unit Loop で Functional Design / NFR Requirements / NFR Design / Infrastructure Design / Code Generation を回す |

### 用語 (ルール Overview の Terminology に準拠)

- **Service**: 独立にデプロイ可能なコンポーネント (本プロジェクトでは Lambda 関数 / S3+CloudFront などの AWS サービス境界に対応)
- **Module**: Service 内の論理グループ (本プロジェクトでは C-02 内の Risk Calculator (C-03) のように一つの Lambda にバンドルされたライブラリ群が該当)
- **Unit of Work**: 計画上の単位 (本ステージでは 7 Unit を計画上の単位として扱う)

### Methodology Choice 補足: Module の独立 Unit 化の根拠

AI-DLC 専門評価者が抱きうる疑問「**Module (Unit-3 Risk Calculator / Unit-5 External Client) を独立 Unit として扱うのは適切か / 親 Service (Unit-2 Dialogue API) に統合した方が適切ではないか**」に対する **先回り回答**:

#### (1) 物理境界 vs 論理境界の使い分け

- **物理境界 (デプロイ単位)**: Unit-3 / Unit-5 は Unit-2 にバンドルされる (CDK スタックでは同一 Lambda 関数として配置)
- **論理境界 (Unit of Work 単位)**: 計画 / 設計 / 検証は **独立 Unit として扱う**
- 両者は矛盾せず、AI-DLC ルール (Overview / Terminology) の「Service / Module / Unit of Work」3 区分に対応

#### (2) Module を独立 Unit 化する 4 つの根拠

| # | 根拠 | 詳細 |
|---|---|---|
| (a) | **PBT 適用範囲の明確化** | Unit-3 (FR-04 calculateAnnoyanceRisk) は純粋関数 / Unit-5 は I/O 含み。Compliance Pre-Check (PBT-02/03/07/08/09 適用) の判定境界が Unit 単位で明確 / Construction Phase の Functional Design / Code Generation で PBT 実装責務が一意に確定 |
| (b) | **Bolt 1 順序での並行検証可能性** | R9 (Bedrock 申請) 待ち中、Unit-3 (純粋関数 PBT 先行検証) と Unit-5 (天気 API キー申請 + クライアント単体実装) を **Unit-2 とは独立に並行実装可能**。Unit-2 に統合した粒度では「R9 承認まで全部待ち」となり Bolt 1 が空回り |
| (c) | **Open Items per-Unit Loop での担当明示** | Open Item O-01 (迷惑リスク判定の具体閾値) は Unit-3 担当 / O-16 (METs ベース定量化) も Unit-3 主担当 / 等。Construction Phase の per-Unit Functional Design でクローズする際、Unit 粒度が小さい方が責務が明確 |
| (d) | **責務境界の明確化** | 純粋関数 (Unit-3) / 外部 API クライアント (Unit-5) / オーケストレーション (Unit-2) という **3 つの異なる責務** を独立に表現できる。Unit-2 に統合すると「Dialogue Lambda が全部やる」という巨大 Unit になり、テスタビリティ・追跡可能性が低下 |

#### (3) AI-DLC ルール「subdomain 相当」解釈との整合

AI-DLC ルール (units-generation.md Overview): 「For microservices, each unit becomes an independently deployable service. For monoliths, the single unit represents the entire application with logical modules.」

本プロジェクトは **microservices と monolith の中間**:
- 物理的には 4 つの独立デプロイ可能 Service (Mobile / Dialogue Lambda / History & Title Lambda / Catalog Distribution + Infrastructure 横断)
- 論理的には **subdomain 相当**として 7 つの Unit of Work (純粋関数 / 外部クライアント / 統合的責務 / 各々独立した Functional Design 対象)

per-Unit Loop (Construction Phase の Functional Design / NFR Requirements / NFR Design / Infrastructure Design / Code Generation) を **subdomain 単位で回す** ことで、設計の追跡可能性と PBT の適用粒度が両立する。

#### (4) コンポーネント 1:1 マッピングの優位性

| 観点 | 1:1 マッピングの利点 |
|---|---|
| **追跡可能性** | C-N → Unit-N の 1:1 対応により、Application Design ステージから Construction Phase まで識別子が一貫 (例: C-03 = Unit-3 = Risk Calculator) |
| **責務再分解の回避** | Application Design で確定したコンポーネント境界をそのまま継承 / Units Generation での再判定なし |
| **per-Unit Loop の運用** | Functional Design / NFR Requirements / NFR Design / Infrastructure Design / Code Generation を **同一の単位** で 7 回繰り返す / 単位境界の混乱なし |
| **外部ステークホルダーへの可読性** | 「7 コンポーネント → 7 Unit」のシンプルな対応関係 / 不要な抽象化レイヤーなし |

> **結論**: Module (Unit-3 / Unit-5) を独立 Unit 化する判断は、AI-DLC ルール (subdomain 相当) と整合し、PBT 適用 / Bolt 1 並行実装 / Open Items per-Unit 担当 / 責務境界の 4 観点で明確な利益がある。物理境界 (Lambda バンドル) との矛盾もない。

---

## Expected Outcomes

- 外部ステークホルダーが `unit-of-work.md` 単体で全 Unit の責務・コンポーネント対応・担当 FR・担当ストーリーを把握できる
- `unit-of-work-dependency.md` で Unit 間の依存関係 + Bolt 1 デプロイ順序が明示される
- `unit-of-work-story-map.md` で全 21 ストーリー → 7 Unit のマッピングが完備
- PBT 対象 3 純粋関数の Unit 帰属が明示され、Code Generation での PBT 適用範囲が確定
- Open Items O-01〜O-15 の担当 Unit が一覧化され、Construction Phase Functional Design (per-Unit) での担当が明確
- Construction Phase の per-Unit Loop に直接接続できる

---

## Compliance Pre-Check

| 拡張 | 適用範囲 | このステージでの遵守事項 |
|---|---|---|
| Security Baseline (Yes / All blocking) | Unit 境界が機微データ境界 (NFR-DAT-02 / R3 / SECURITY-13) と整合していることを確認 | `unit-of-work.md` で Unit 1 (Mobile) の機微データ境界遵守を明示 / `unit-of-work-dependency.md` でクラウド側 Unit (2〜7) が集計値のみ受け取ることを明示 |
| Property-Based Testing (Partial) | PBT 対象 3 純粋関数 (FR-04 / FR-05 / FR-10) の Unit 帰属を確定 | `unit-of-work.md` で Unit 3 / Unit 2 / Unit 4 の責務として明示 |

> いずれもブロッキング所見なし。設計内容に変更なし (Application Design の継承のみ)。
