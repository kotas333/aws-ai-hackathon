# Application Design Assessment

> **作成日**: 2026-05-07T15:00:47Z
> **目的**: 本プロジェクトで Application Design ステージを実行する正当性、および標準ルール (Step 1-15) のうち Step 2-9 (Plan + [Answer]: tag Q&A loop) をスキップする根拠を文書化

## Decision

**Execute Application Design**: **Yes** (重点ステージ)

**標準 Step 2-9 のスキップ**: **Yes** (User Stories ステージと同じ方法論を継続)

---

## Execute 判定の根拠 (workflow-planning.md Section 3.2 と整合)

| 条件 | 該当 | 根拠 |
|---|---|---|
| New components or services needed | **Yes** | Mobile Client / API Gateway+Lambda (対話生成) / Risk Calculator / History+Title / External Client / Infrastructure の 6 コンポーネントが新規 |
| Component methods and business rules need definition | **Yes** | LLM プロンプト構築・迷惑リスク判定・称号付与等の主要メソッド契約を Functional Design に渡す前に定義する必要 |
| Service layer design required | **Yes** | Mobile から見た API オーケストレーション (対話生成 / 履歴記録 / 履歴取得 / 設定取得) を明確化する必要 |
| Component dependencies need clarification | **Yes** | 機微データ (ヘルスケア/位置) のローカル境界 (R3 / NFR-DAT-02) を依存関係マップで視覚化する必要 |

**Skip 条件への該当**: なし (既存コンポーネント境界変更ではない / 新規メソッドあり / Greenfield)

---

## Step 2-9 スキップの根拠 (User Stories ステージと同じ方法論)

User Stories ステージでは Mob Elaboration 確定版がすでに存在することと Q1-Q5 で残課題が解決済みであることを根拠に Step 2-14 (Planning) をスキップし、Step 15+ (Generation) のみ実施した。

Application Design ステージでも以下の理由で **同じ方法論を継続** する:

### Step 2-9 で問うべきことが既に解決済みである根拠

| Step 4 のカテゴリ | 本プロジェクトでの解決状況 |
|---|---|
| **Component Identification** (境界・組織・グループ化) | `execution-plan.md` Section 3.1.B で **6 Unit 暫定分解** が確定済み (Mobile Client / Dialogue API / Risk Calculator / History & Title / External Client / Infrastructure)。本ステージはこの分解を「コンポーネント」として継承し、Units Generation で「Unit」として確定させる構造 |
| **Component Methods** (メソッド契約・I/O) | FR-01〜11 と全 18 ストーリーの AC で I/O が AC 単位で定義済み。本ステージは AC をメソッドシグネチャに翻訳する作業 |
| **Service Layer Design** (オーケストレーション) | `requirements.md` Technical Context で AWS サーバーレス構成が確定済み (API Gateway + Lambda)。サービス層は API Gateway がオーケストレーション、Lambda が個別ユースケースを担当する標準パターンで自明 |
| **Component Dependencies** (通信パターン・結合) | `requirements.md` NFR-DAT-02 で「ヘルスケア生データ = ローカル限定 / サボり履歴 = AWS」の境界が確定済み。本ステージはこの境界を依存関係マップに翻訳する作業 |
| **Design Patterns** (アーキテクチャ様式) | AWS 主催規約 (NFR-DAT-03) と Bedrock + Lambda + DynamoDB の固定スタックで、サーバーレス + イベント駆動 + Adapter パターン (HealthKit/擬似データ切替) という様式が自明 |

### ユーザーの優先事項 5 件で代替

ユーザーが Workflow Planning 完了承認時に提示した 5 件の優先事項が、Step 4 で本来問うべき内容を **指示として直接与えている**:

1. **持ち越し 3 判断 (D-01/D-02/D-03)** の決着 → 本ステージで決着 (後続 Phase B で上流ステージに最初から確定された記述として統合)
2. **Q4 設計実体化** (iOS HealthKit + 擬似データ抽象化) → 本ステージで Adapter パターン設計
3. **設計の中核訴求点の明示** (LLM プロンプト構造 / 機微データフロー / ドメインモデル) → 本ステージで Section 9 / Section 10 / Section 8 として記述
4. **コンポーネント分解の方針** → 本ステージで定義 (Unit 分解は Units Generation の責務)
5. **自己完結型ポリシー** → application-design.md を自己完結型で作成

---

## 標準ルールステップの取り扱い

| 標準ステップ | 本プロジェクトでの取り扱い |
|---|---|
| Step 1: Analyze Context | 本ファイル (assessment) で実施 (Execute 判定 + Step 2-9 スキップ判定) |
| Step 2-9: Plan + Q&A loop | **スキップ** (上記根拠 + ユーザー優先事項 5 件で代替) |
| Step 10: Generate Application Design Artifacts | 5 ファイル生成 (components.md / component-methods.md / services.md / component-dependency.md / application-design.md) |
| Step 11: Log Approval | audit.md に Generation 完了エントリを追記 |
| Step 12-14: 完了承認 | 通常どおり実施 |
| Step 15: Update Progress | aidlc-state.md 更新で実施 |

---

## Methodology Choice

### Architecture Style: AWS Serverless + Event-Driven + Adapter Pattern

| 様式 | 選択理由 |
|---|---|
| **Serverless** (Lambda + API Gateway + DynamoDB) | NFR-DAT-03 「AWS サービスのみ」 / R8 R9 の予選通過後最優先タスク (アカウント・モデルアクセス) と整合 / 開発期間制約 (1 ヶ月) 内での実装可能性を高める |
| **Event-Driven** (対話生成 / 選択保存) | Mobile からのリクエストを Lambda が個別に処理する単純なフロー / 状態を持たない LLM 呼び出しに最適 |
| **Adapter Pattern** (HealthKit / EventKit / 擬似データモード) | Q4 (iOS 確定 + DEBUG ビルド擬似データモード) を Mobile 内で抽象化 / R2 (ヘルスケア API 連携の実装難易度) への対応 |
| **Domain-Driven の軽量適用** | Mob Elaboration の二人キャラクター (ジャッジ・悪魔) と動的トーンシフト (US-2.4) を中核ドメインとして明示 / US-2.3 (Should) 維持の根拠 (キャラ拡張容易性) を担保 |

### 設計の優先順位 (明示的制約に基づく順位付け)

詳細は `application-design.md` Section 2.2 を参照。明示的制約 (NFR / Risk Register / 主催規約) に基づく順位は以下:

1. **機微情報のローカル保護** (NFR-DAT-02 / R3 / SECURITY-13)
2. **応答速度の確保** (NFR-USA-02)
3. **AWS マネージドサービスでの構築** (NFR-DAT-03)
4. **MVP 17 件 (Must) の実装可能性** — 7 コンポーネントの境界が明確で並行実装可能
5. **R8 / R9 への対応容易性** — コンポーネント独立性によりデプロイ単位を分離可能
6. **R1 (スコープ過大) のリスク管理** — Should ストーリーを Must 増の代わりに保留
7. **拡張容易性** — US-2.3 (キャラ切替 / Should) の拡張点をドメインモデルで担保

---

## Expected Outcomes

- 外部ステークホルダーが `application-design.md` 単体で全コンポーネント・主要メソッド・サービス層・依存関係を把握できる
- LLM プロンプト構造が中核ドメインとして視覚化される (ジャッジと悪魔 + 動的トーンシフトが Section 9 で記述)
- 機微データフロー図で R3 / NFR-DAT-02 のローカル境界が明示される (Section 10 / プライバシー設計の根拠記述)
- Units Generation ステージで本ステージの 7 コンポーネントを直接 Unit 候補としてマッピングできる
- 持ち越し 3 判断 (D-01/D-02/D-03) が本ステージで決着し、後続 Phase B で `requirements.md` / `user-stories.md` に最初から確定された記述として統合される

## Compliance Pre-Check (Application Design ステージ時点)

| 拡張 | 適用範囲 | このステージでの遵守事項 |
|---|---|---|
| Security Baseline (Yes / All blocking) | 設計レベルで以下を明示する: (a) 機微データ (ヘルスケア/位置) のローカル境界 (R3 / NFR-DAT-02 / SECURITY-03 機微情報の最小化) / (b) 端末 UUID 認証 (SECURITY-04 認証境界) / (c) Bedrock 呼び出しは Lambda 経由のみ (SECURITY-06 公開エンドポイント最小化) / (d) DynamoDB 暗号化方針 (SECURITY-07 保管時暗号化) | application-design.md Section 10 (機微データフロー) で視覚化 |
| Property-Based Testing (Partial) | 純粋関数 3 件の所属コンポーネントを設計で明示 | application-design.md Section 14 (PBT Target Mapping) で明示 |

> Security Baseline は設計レベルでの「方針」を確定する責任があるため、**Application Design ステージでは N/A ではなく "方針確定責任あり"**。Code Generation で実装検証を行う。
