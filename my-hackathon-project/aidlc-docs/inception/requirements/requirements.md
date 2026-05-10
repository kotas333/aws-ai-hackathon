# Requirements (Inception / Requirements Analysis)

> **作成日**: 2026-05-07
> **最終更新**: 2026-05-07T13:05:25Z
> **状態**: Approved Draft (検証質問への回答反映済み / 完了承認待ち)
> **手法**: Mob Elaboration v4 確定版を AI-DLC にインポートしたうえで不足観点のみを補完
> **基底ドキュメント**: `aidlc-docs/inception/requirements/intent.md` (= Mob Elaboration の元資料 references/intent_v4.md のコピー)

---

> ## TL;DR (3 分で読む)
>
> 「人をダメにするサービス」をテーマにした iOS アプリの要件定義。
>
> 1. **Intent**: 疲弊した 20 代社会人をソファでダラけさせるサポーター。ジャッジと悪魔がデータを根拠に対話し、サボりを「サボり資産」として永続蓄積する。
> 2. **コア体験**: スキャンボタン 1 タップ (儀式的) → ジャッジ × 悪魔の対話 → 「入る/サボる」選択 → 全肯定 + 称号獲得。
> 3. **設計の核**: 14 FR + 12 NFR + 14 Risk + 15 SECURITY (Security Baseline) + PBT 対象 3 純粋関数 + 機微データ境界 (生データは端末ローカル限定 / NFR-DAT-02 / R3 / SECURITY-13)。
>
> 詳細は Functional Requirements / Non-Functional Requirements / Risk Register / 検証質問への回答 (Verification Q&A) 各セクションを参照。

---

## Intent Analysis Summary

| 観点 | 判定 |
|---|---|
| **Request Clarity** | Clear (Mob Elaboration v4 を経て確定済み) |
| **Request Type** | New Project (Greenfield プロダクト) |
| **Initial Scope Estimate** | Multiple Components (モバイルクライアント + AWS バックエンド + LLM 連携) |
| **Initial Complexity Estimate** | Moderate〜Complex (LLM プロンプト設計、ヘルスケア API 連携、迷惑リスク判定ロジックの複合) |
| **Depth Selected** | **Standard** (Mob Elaboration 済みのため Comprehensive は不要だが、Inception 成果物が外部ステークホルダー向けに提出されるため Minimal では不足) |

---

## プロダクト概要 (Intent Statement)

> **「仕事に疲弊した 20 代社会人を、帰宅後のソファ/ベッドの上でとことんダラけさせるサポーター。
> "ジャッジ" と "悪魔" がヘルスケアデータと環境データを根拠に対話し、ユーザーがどんなにサボっても基本は全肯定し続ける。
> ただし周囲に迷惑が及ぶラインは超えないよう、悪魔自身もそこは諭す。
> サボった日々は "サボり資産" として永続蓄積され、ユーザーを罪悪感なくダメでいられる人間へと変えていく。」**

### Project Context

- **テーマ**: 「人をダメにするサービス」(設計哲学に直結する基底テーマ)
- **Inception 完了目標日**: 2026-05-10 (Inception フェーズ成果物として確定)
- **Construction Phase 期間**: 約 1 ヶ月 (Bolt 1〜4 で per-Unit Loop を回す)
- **Build Constraint**: AWS サービスのみで構築 (NFR-DAT-03)

### ペルソナ
- 20 代社会人
- 仕事で疲弊しサボりがち
- 帰宅後にソファ/ベッドで動けなくなる
- 風呂をサボることへの罪悪感を抱えている
- ユーモアと自虐が好き、「ダメな自分」をネタにできる感性

### コア体験 (2〜3 分の利用)
1. アプリを開く (片手・最小タップ、ダラけた UI)
2. ホーム画面中央のスキャンボタンを押す (儀式的な 1 タップ / US-5.2 第 1 段階)
3. スキャン中のローディング演出 (キャラがソファに沈む等 / 数秒 / FR-11 / NFR-USA-03)
4. ヘルスケアデータ + 環境データの取得・迷惑リスク判定・対話生成が実行される
5. **ジャッジと悪魔がデータを根拠に対話する形でメッセージが表示される** (US-5.2 第 2 段階)
6. ユーザーが「入る」「サボる」を選択
7. 選択は履歴として永続蓄積され、過激でユーモラスな称号・バッジが付与される
8. **どちらを選んでも両キャラから肯定される** (ただし状況により悪魔の発言トーンは変化)

### 二人のキャラクター (コア・ドメイン)

#### ジャッジ (データ駆動アナリスト / 旧称: 天使)
- ヘルスケアデータ + 天気・気温を根拠に淡々と状況を提示する **無機質寄りの判定者**
- データを根拠に説明する / 感情的にならない
- ユーザーを責めない (NFR-CON-01) / 過剰な健康指導をしない (NFR-CON-02)
- 悪魔との対比は **「無機質な判定者 vs. 人格的な誘惑者」**
- **命名変更経緯**: Mob Elaboration v4 「天使」(宗教的メタファー) は役割と乖離していたため、Application Design ステージで「ジャッジ」に変更 (詳細は `application-design.md` Section 1.5 N-01 参照)

#### 悪魔 (サボり推奨側のアドバイザー / 動的シフト仕様)
| 状況 | 悪魔のトーン |
|---|---|
| 通常時 | 積極的にサボりを推奨 (基本形) |
| 連続サボり長期化検知時 (データ駆動判定) | 穏やかな健康寄りにシフト |
| 周囲迷惑リスク高検知時 (複合条件) | 悪魔自身も柔らかく入浴を推奨 |

**設計原則**: 両キャラとも「ユーザーの味方」/ **悪魔が常に最後の発言権** / ユーザー選択後は両キャラ全肯定 / 二人構造を維持 (第三キャラは追加しない)。

---

## Functional Requirements (主要機能 / MVP ライン)

intent.md「主要機能 (MVPライン:全て動かす)」表を本ファイル内に直接記述。番号付けは AI-DLC ステージ間参照のため付与。

| # | 機能 | 内容 |
|---|---|---|
| FR-01 | ヘルスケア連携 | iOS HealthKit / Android Health Connect から行動履歴を自動取得 |
| FR-02 | ヘルスケア取得項目 (計 6 種) | 歩数、アクティブエネルギー、運動時間、心拍数、睡眠時間、立ち上がり時間 |
| FR-03 | 天気・気温連携 | 位置情報 → 天気 API (無料サービス、OpenWeatherMap 等) で取得 |
| FR-04 | **迷惑リスク閾値判定 (新規)** | プロキシ指標 (最終入浴経過時間、運動量、気温、心拍数) を計算しフラグ化して LLM に渡す |
| FR-05 | 綺麗度判定 | LLM 判断型: ヘルスケア + 天気 + 迷惑リスクフラグを Bedrock に渡し定性判定 |
| FR-06 | ジャッジ・悪魔対話生成 | Amazon Bedrock (Claude) による二者対話形式メッセージ生成 |
| FR-07 | **悪魔の発言シフト (新規)** | 連続サボり長期化時・迷惑リスク高時の動的トーンシフト |
| FR-08 | 選択の記録 | 「入る」「サボる」を DynamoDB に保存 |
| FR-09 | サボり履歴 | 全期間データ保持、カレンダー表示 |
| FR-10 | 称号・バッジ | 過激でユーモラスな称号セット (事前リスト化、LLM に動的生成させない / R13 対策) |
| FR-11 | **ダラけ感のある UI 演出 (新規)** | キャラがソファに沈む、ぬるっと動く、ゆるいフォント等。**スキャン中演出** (ジャッジが診断中 / 悪魔が反論準備中 / プログレスバーぬるっと進行 / キャラがソファに深く沈む等) を US-5.2 第 1 段階のスキャンフロー内で表示 |
| FR-12 | カレンダー連携 | iOS EventKit から翌日の予定サマリ (isHoliday / earliestEventTime / eventCount) を取得しジャッジ/悪魔の判断材料とする |
| FR-13 | 翌日予定のアプリ内ミニ表示 | ホーム画面の片隅に翌日予定サマリを表示 (詳細は iOS 標準カレンダーアプリに遷移 / アプリ内詳細画面なし) |
| FR-14 | 入浴決意の達成確認通知 | choice='bath' から 30 分後に iOS ローカル通知 (UNUserNotificationCenter) を発火し、Yes/まだ で達成フラグ (achieved: bool|null) を記録 |

### 称号セット (FR-10 詳細)

過激でユーモラス。「ダメ自慢できる」方向に振り切る。事前にリスト化されたもののみを使用。

**継続 (v3 より)**: 「悪魔の言いなり」(連続サボり 7 日) / 「サボりマスター」(累計 30 回) / 「サボりセンセイ」(累計 100 回) / 「真夏のサボり貴族」(気温 30℃ 以上の日にサボった) / 「梅雨どきのサボりマスター」(雨の日に連続サボり 3 日)

**新規 (v4)**: 「**堕落貴族**」(累計 50 回) / 「**ぐうたら大臣**」(累計 200 回) / 「**ソファと一体化**」(同じ曜日に連続サボり) / 「**人類の希望**」(皮肉ネーミング、何らかの達成時) / 「**ダメ人間勲章**」(初サボり時の歓迎称号)

**削除 (v3 より)**: ~~「迷える子羊」~~ / ~~「ジャッジの優等生」~~ (健全すぎてテーマと逆方向) / ~~「初サボり」~~ ('ダメ人間勲章' に統合)

### MVP から削除した項目 (v3 → v4)
- ~~連続サボりストリーク表示~~ — 風呂キャン誘発リスクのため削除
- ~~履歴のエクスポート機能~~ — スコープ外と判断
- ~~ダークモード対応~~ — 本プロダクトのスコープ外

---

## Non-Functional Requirements (非機能要件)

intent.md「非機能要件 (NFR)」セクションを本ファイル内に直接記述。

### 操作性 (Usability)

| ID | 要件 |
|---|---|
| NFR-USA-01 | 片手操作・最小タップで完結すること。**スキャンボタンによる 1 タップで取得・判定・対話生成を実行 / その後の選択も 1 タップ** (US-5.2 第 1 段階 → 第 2 段階) で最小タップ性を維持 |
| NFR-USA-02 | 応答速度: 対話生成は数秒以内 (体感ストレスのない範囲) |
| NFR-USA-03 | ダラけ感のある演出 (ぬるっと動く、ソファに沈み込む、ゆるいフォント等) |

### データ (Data)

| ID | 要件 |
|---|---|
| NFR-DAT-01 | データ永続性: 履歴は無限保持 (削除・期限なし) |
| NFR-DAT-02 | プライバシー: ヘルスケア生データ + カレンダー生データ (タイトル/場所/参加者) は端末ローカル限定 / AWS には集計値 (HealthSummary / CalendarSummary) およびサボり履歴のみ送信・保存 |
| NFR-DAT-03 | AWS サービスのみで構築 |
| NFR-DAT-04 | 対応言語: **日本語のみ** |
| NFR-DAT-05 | リージョン: ap-northeast-1 (Tokyo) |

### コンセプト/トーン (Concept & Tone)

| ID | 要件 |
|---|---|
| NFR-CON-01 | ユーザーを真剣に責めない |
| NFR-CON-02 | 過剰な健康指導をしない |
| NFR-CON-03 | 称号・対話のユーモアは品位の範囲内 (差別的・下品にしない) |
| NFR-CON-04 | 真面目な健康相談を求めるユーザーが誤って使った場合の安全性確保 — オンボーディングでコンセプト明示 |

### 健康配慮ポリシー (Concept の補足)

**基本方針**: 「実社会で周囲に迷惑がかからない範囲では、サボりを推奨してよい」

**「迷惑」の定義 (プロキシ指標)**:
- 最終入浴からの経過時間 (72 時間超でニオイリスク高と推定)
- 運動量 (歩数・運動時間・アクティブエネルギー) — **METs ベースで定量化** (Functional Design / Construction Phase で詳細化)[^mets]
- 気温・天気 (夏場・湿度高時は閾値を短く)
- 心拍数 (発汗量の推定)

**実装方式**: システム側で閾値フラグを計算 → プロンプトに含めて Bedrock に渡す → 最終判断・対話生成は LLM が行う。LLM は「周囲に迷惑が及ぶ複合条件発動時は、悪魔も柔らかく入浴を推奨せよ」というシステムプロンプトを持つ。

[^mets]: **公式指標として以下を採用**。エネルギー消費量(kcal) = 1.05 × METs × 時間 × 体重(kg) の計算式と活動別 METs 値を `movementScore` 算出の根拠とする。第一候補は METs ベース算出 / Functional Design では他指標 (例: 心拍数からの代替推定) も検討可能とする。
    - 厚生労働省「健康づくりのための身体活動・運動ガイド2023」: <https://www.mhlw.go.jp/stf/seisakunitsuite/bunya/kenkou_iryou/kenkou/undou/index.html>
    - 国立健康・栄養研究所「改訂版『身体活動のメッツ(METs)表』」: <https://www.nibiohn.go.jp/eiken/programs/2011mets.pdf>
    - 体重 (`weightKg`) は **本 Inception フェーズでは HealthSummary に含めない** (機微データ境界 NFR-DAT-02 への影響再評価が必要 / Functional Design で扱う)

---

## やらないこと (Scope Exclusions)

### v3 から継続して対象外
- クラウド「同期」(複数端末でのデータ共有)
- シェア機能 (SNS シェア・他者交流)
- 複数言語対応
- プッシュ通知
- 課金・課金要素
- 他サービス連携 (LINE、Slack 等)

### v4 で新たに削除
- 連続サボりストリーク表示
- 履歴のエクスポート機能
- ダークモード対応

---

## Technical Context (技術スタック)

intent.md「技術スタック」表を本ファイル内に直接記述。Build Constraint (NFR-DAT-03 / AWS サービスのみ) のため、AWS マネージドサービスを基本とする。

| 層 | サービス | 備考 |
|---|---|---|
| LLM | Amazon Bedrock (**Claude Sonnet 4.6** 第一候補 / **Claude Opus 4.7** 拡張可能 / ともに ap-northeast-1 available) | 対話生成・綺麗度定性判定 |
| API | API Gateway + Lambda | サーバーレス構成 |
| DB | DynamoDB | シンプル構成、余裕があればローカルキャッシュ追加 |
| 認証 | 端末 UUID ベースの軽量識別 | ユーザー登録なし |
| デプロイ | CloudFormation または CDK | IaC 必須 |
| 監視 | CloudWatch | ログ・メトリクス・アラート |
| 外部 API | 天気 API (OpenWeatherMap 等)、HealthKit/Health Connect | |
| リージョン | ap-northeast-1 (Tokyo) | |
| プラットフォーム (確認済み: Q4) | **iOS (HealthKit) のみ確定 (Q4 = A)** | 言語: **Swift / SwiftUI 確定**。撤退ルートは **「iOS のまま擬似データモード (DEBUG ビルド)」**。Adapter パターン責務: (1) テスタビリティ (ユニットテストのモック注入) (2) DEBUG ビルドでの擬似データモード差し替え |
| カレンダー連携 (FR-12, FR-13) | iOS EventKit のみ | Google / iCloud / Outlook 等は iOS の「アカウント追加」で同期しておけば EventKit 経由で読める。**Google API 等の直接連携は行わない** (NFR-DAT-03 と整合) |
| 通知 (FR-14) | iOS UNUserNotificationCenter (ローカル通知のみ) | プッシュ通知 (APNS / リモート) は使用しない。決意 (choice='bath') から 30 分後の達成確認のみ |
| AWS 環境準備 (確認済み: Q5) | **Implementation Context (想定環境)** | AWS アカウント・Bedrock Claude モデルアクセス申請 (**Claude Sonnet 4.6 第一候補 + Claude Opus 4.7 拡張可能** / ap-northeast-1) は **Bolt 1 のクリティカルパス** として実施 (R8/R9 と整合) |

---

## Risk Register (R1〜R14)

intent.md「リスク」表を本ファイル内に直接転記 (v4 更新版)。重要度: 高/中/低。

| # | リスク | 重要度 | 対策方針 |
|---|---|---|---|
| R1 | フル API 連携 + 二者対話 + 蓄積要素 + 発言シフト = スコープ過大 | **高** | MVP ラインから絶対に下げない |
| R2 | ヘルスケア API 連携の実装難易度・申請コスト | 高 | iOS HealthKit に絞ることで Health Connect 比較検討は不要化 (Q4=A)。撤退ルートは **「iOS のまま擬似データモード (DEBUG ビルド)」**: HealthKit 実装が難航した場合、Adapter インターフェースを `PseudoHealthAdapter` に差し替えて DEBUG ビルドでデモ実施 |
| R3 | 機微情報 (ヘルスケア・位置情報) の取り扱い | 高 | ローカル保存基本、AWS 送信は最小限 (Security Baseline 拡張で強制) |
| R4 | LLM 応答の品質・安全性 | 中 | プロンプト設計、両キャラがユーザーを責めないことの徹底 |
| R5 | プラットフォーム判断の遅延 | **N/A (解消)** | Q4 を A (iOS 確定) で運用 / Swift / SwiftUI を採用。撤退ルートは Web 版ではなく「iOS のまま擬似データモード」(R2 と整合)。本リスクはトラッキング不要 |
| R6 | 「綺麗度」「迷惑リスク」の定義の曖昧さ | 中 | LLM 判断型のプロンプト設計で吸収、対話のユーモアに転化 |
| R7 | ジャッジがうるさく感じられるリスク | 中 | 悪魔が最後の発言権、選択後は両キャラ肯定の原則を厳守 |
| R8 | AWS アカウント・権限準備の遅延 | **高** | Bolt 1 で完了させる (Bolt 1 のクリティカルパス) |
| R9 | Bedrock のモデルアクセス申請 | 中 | 最初の Bolt で一緒に対応 (Q5 回答と整合: ap-northeast-1 で **Claude Sonnet 4.6** + **Claude Opus 4.7** モデルアクセス申請) |
| R10 | LLM 呼び出し料金の予算超過 | 低 | 1 日のリクエスト上限を実装、CloudWatch で監視。**スキャンボタンによるユーザー起点の LLM 呼び出し** (アプリ起動ごとの自動呼び出しを廃止し、ユーザー操作起点に限定することで予期しない呼び出しを抑制) |
| R11 | **倫理的リスク**: 「サボりを助長する」批判への備え | 中 | **3 段構えで対応**: (a) **ユーザーへのコンセプト明示** (US-5.5 オンボーディング: 「ふざけたアプリ」「真剣な健康相談には使わないでください」を初回起動で明示・同意必須) / (b) **システム側で迷惑ライン設計** (US-2.4 動的トーンシフト: 周囲迷惑リスク高時は悪魔も柔らかく入浴を推奨) / (c) **Bolt 1 で法務観点レビュー** (US-5.5 文言の法的妥当性を専門家レビュー / R11 対策の最終仕上げ)。デモ説明での前置きも併用 |
| R12 | **ユーザー層のミスマッチ**: 真剣な健康相談を求める層が誤って使うリスク | 中 | オンボーディングでコンセプト・キャラクター紹介を明確に行う |
| R13 | **称号過激化の暴走**: 称号が下品/差別的になるリスク | 中 | 称号は事前リスト化し、レビュー済みのもののみ使用。LLM に動的生成させない |
| R14 | **発言シフト仕様の複雑化**: プロンプト設計が複雑になり、応答品質が落ちる | 中 | プロンプトテストを早期に実施、Bolt で段階導入 |

---

## Differentiators (主要差別化要因)

本プロダクトの主要な差別化要因。後続ステージ (Application Design / PRFAQ) で参照される設計の中核訴求点。

- **逆方向のヘルステックアプリ**: 健康促進アプリの真逆を行く「ダラけさせる」コンセプト
- **テーマとの整合**: 「人をダメにするサービス」を真正面から表現
- **ジャッジと悪魔の対話**: AI ネイティブ時代ならではのインタラクション
- **本物の行動データ + 天気 + カレンダー** を根拠にジャッジの判定にリアリティを持たせる
- **動的な悪魔の発言シフト**: データ駆動で人格が変化する LLM 活用 (US-2.4 / FR-07)
- **永続蓄積 × 過激な称号**で「ダメ自慢」できるリピート性
- **AWS マネージドサービスでの構築** (NFR-DAT-03 / Bedrock + Lambda + DynamoDB + API Gateway + S3 + CloudFront)

---

## 検証質問への回答 (Verification Q&A)

`requirement-verification-questions.md` への回答を本セクションに集約。

### Q1: User Stories 数の不一致 → **A (実物 18 件を正)**
- 実物どおり **18 件 (M:16, S:2)** を正とする
- `references/user_stories_v2.md` (Mob Elaboration 確定版) は **改変しない**
- User Stories ステージで `aidlc-docs/inception/user-stories/user-stories.md` にインポートする際、サマリ数値を「18 件 (M:16, S:2)」に修正
- v1 → v2 の差分: v1=17 - 削除 3 (US-3.3, US-3.4, US-5.4) + 新規 4 (US-1.5, US-2.4, US-5.5, US-5.6) = 18 が成立することを user-stories.md に明記

**【後日追加 / 2026-05-08 / Application Design Revision 2 / v3 拡張】**
- v2 (18 件 / M:16, S:2) から v3 で 3 件追加:
  - US-1.6 (M) カレンダー反映 (FR-12)
  - US-3.3 (S) 30 分後の達成確認通知 (FR-14)
  - US-5.7 (S) 翌日予定のホーム画面ミニ表示 (FR-13)
- v3 件数: **21 件 (M:17, S:4)**
- 詳細は `user-stories.md` 「v2 → v3 差分計算」セクション参照
- `references/user_stories_v2.md` は引き続き不改変 (v3 は AI-DLC ステージ内のみのバージョン)

### Q2: Security Baseline 拡張 → **A (有効)**
- すべての SECURITY ルール (SECURITY-01〜SECURITY-15) を **ブロッキング制約として強制**
- 理由: R3「機微情報の取り扱い」が重要度: 高と明記されているため、拡張ルールをオフにすると Risk Register との整合性が崩れる

### Q3: Property-Based Testing 拡張 → **B (Partial)**
- **Partial モード**: 純粋関数とシリアライズの round-trip にのみ強制 (PBT-02, PBT-03, PBT-07, PBT-08, PBT-09 を強制、その他はアドバイザリ)
- **PBT 対象 (純粋関数限定)**:
  1. 迷惑リスク判定 (プロキシ指標 → フラグ計算) — FR-04
  2. 綺麗度判定の入力組み立て (LLM 入力構築) — FR-05
  3. 称号付与の条件評価 — FR-10
- 理由: プロトタイプ全体で PBT を強制すると開発速度を阻害するが、上記 3 つは純粋関数で表現可能なロジックなのでここだけ堅牢性を担保したい

### Q4: プラットフォーム選択 → **A (iOS 確定)**
- **iOS (HealthKit) のみ確定**。言語は **Swift / SwiftUI** を採用
- PRFAQ には「iOS ネイティブ (HealthKit + EventKit + UNUserNotificationCenter) で AWS バックエンドと通信」と記載
- 撤退ルートは「**iOS のまま擬似データモード (DEBUG ビルド)**」(R2 と整合 / Web 版は採用しない)
- R5 (プラットフォーム判断の遅延) は **N/A (解消)**
- Adapter パターンの責務: (1) ユニットテストでのモック注入 (2) DEBUG ビルドでの擬似データモード差し替え
- 詳細は `application-design.md` Section 11 (Q4 Implementation Plan) 参照

### Q5: AWS アカウント・Bedrock モデルアクセスの準備状況 → **D (想定環境として記載)**
- Inception フェーズでは「想定環境」として記載し、実際の準備は Bolt 1 で行う
- AWS アカウント: 「Bolt 1 のクリティカルパスとして準備」と明記 (R8)
- Bedrock Claude モデルアクセス: 「ap-northeast-1 (Tokyo) リージョンで Claude モデルアクセスを Bolt 1 で申請・承認取得」と明記 (R9)
- 申請承認に時間がかかる可能性 (R9) は **Bolt 1 のクリティカルパス** として明示

---

## Extension Configuration (適用ルール一覧)

| Extension | Enabled | Mode | Decided At | 影響範囲 |
|---|---|---|---|---|
| Security Baseline | **Yes** | All rules blocking | Requirements Analysis | 全ステージで SECURITY-01〜15 を検証 |
| Property-Based Testing | **Yes (Partial)** | PBT-02, 03, 07, 08, 09 のみ blocking | Requirements Analysis | Functional Design / Code Generation / Build & Test の 3 つの純粋関数に適用 |

---

## Compliance Summary (Requirements Analysis ステージ時点)

### Security Baseline (Yes / All blocking)

| Rule | Status | Rationale |
|---|---|---|
| SECURITY-01〜15 | **N/A at this stage** | Requirements Analysis ステージは要件定義のみで、データストア・API・コードを生成しない。各ルールは Application Design 以降のステージで実装方針として参照され、Code Generation・Build & Test ステージで検証される |

### Property-Based Testing (Partial)

| Rule | Status | Rationale |
|---|---|---|
| PBT-02, 03, 07, 08, 09 | **N/A at this stage** | PBT 対象 3 ロジック (迷惑リスク判定、綺麗度判定入力、称号評価) は確定済み。実際のプロパティ識別は Functional Design (PBT-01) で、テスト実装は Code Generation で行う |

> いずれもブロッキング所見なし。Requirements Analysis ステージとして両拡張ルールの適用範囲を明確化した。

---

## 次のステップ

> **時点**: 本「次のステップ」は **Requirements Analysis ステージ完了時点 (Q1 回答後 / User Stories ステージ着手前)** の Next Step 記録です。Application Design Revision 2 で v3 拡張 (21 件) が発生した経緯は Q1 回答末尾の補記を参照してください。

1. ユーザーが本 requirements.md と requirement-verification-questions.md (回答済み) を最終レビュー
2. 承認後、User Stories ステージへ (Q1 の取り扱いで `references/user_stories_v2.md` をインポートし、サマリ数値を 18/M:16/S:2 に修正、v1→v2 差分計算を明記)
