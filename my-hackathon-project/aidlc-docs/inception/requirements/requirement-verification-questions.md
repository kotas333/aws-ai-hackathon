# Requirements Verification Questions

> **作成日**: 2026-05-07
> **対象**: Mob Elaboration v4 確定版 (`intent.md` / `references/user_stories_v2.md`) を AI-DLC に取り込むにあたり、
> 既存内容を作り直さず **不足観点だけ** を確認するための質問。
>
> **回答方法**: 各質問の `[Answer]:` の右に英大文字 (A, B, C, …) を記入してください。
> X を選ぶ場合は X の右に詳細を記述してください。
> 全問記入後、「回答完了しました」「done」など完了の合図をください。

---

## Question 1: User Stories 数の不一致確認

`references/user_stories_v2.md` を機械的に集計したところ、以下の不一致を検出しました。

| 観点 | サマリ記載 (456-467 行) | 実物の集計 |
|---|---|---|
| 合計 | 17 件 | **18 件** (US-1.1〜US-5.6) |
| Must (M) | 14 件 | **16 件** |
| Should (S) | 3 件 | **2 件** (US-1.4, US-2.3) |

実際の見出し列挙: US-1.1【M】, US-1.2【M】, US-1.3【M】, US-1.4【S】, US-1.5【M】, US-2.1【M】, US-2.2【M】,
US-2.3【S】, US-2.4【M】, US-3.1【M】, US-3.2【M】, US-4.1【M】, US-4.2【M】, US-5.1【M】, US-5.2【M】,
US-5.3【M】, US-5.5【M】, US-5.6【M】 (計 18 / M:16, S:2)

どの解釈を正とするか確定してください。

A) 実物どおり **18 件 (M:16, S:2)** を正とし、サマリ記載側 (17 件 / M:14, S:3) を誤記として修正する
B) サマリ記載どおり **17 件 (M:14, S:3)** を正とし、実物のいずれかを削除/降格して整合させる
   (削除候補がある場合は X で具体的に指定)
C) どちらでもない — 別の数え方で整合させたい (X で詳細記述)
X) Other (please describe after [Answer]: tag below)

[Answer]: A
**追加指示** (2026-05-07T13:05:25Z 受領):
1. user_stories_v2.md は references/ 側 (Mob Elaboration 確定版) なので **改変しない**
2. aidlc-docs/inception/user-stories/user-stories.md にインポートする際、サマリ数値を「18 件 (M:16, S:2)」に修正
3. v1→v2 の差分計算 (v1=17 - 削除3 + 新規4 = 18) が成立することを user-stories.md に明記

---

## Question 2: Security Baseline 拡張のオプトイン

Security Baseline 拡張ルールを本プロジェクトで強制するかどうかを確定してください。

A) Yes — すべての SECURITY ルールをブロッキング制約として強制する (本番品質アプリ向け推奨)
B) No — すべての SECURITY ルールをスキップする (PoC・プロトタイプ・実験プロジェクト向け)
X) Other (please describe after [Answer]: tag below)

> 補足: 本プロジェクトはハッカソンのプロトタイプですが、ヘルスケア・位置情報を扱うためプライバシー観点は重要です。
> Mob Elaboration v4 でも「ローカル保存基本、AWS 送信は最小限」というポリシーは確定済みです。

[Answer]: A
**理由**: intent.md の R3「機微情報の取り扱い」が重要度: 高と明記されているため、拡張ルールをオフにすると Risk Register との整合性が崩れる。

---

## Question 3: Property-Based Testing (PBT) 拡張のオプトイン

PBT 拡張ルールを本プロジェクトで強制するかどうかを確定してください。

A) Yes — すべての PBT ルールをブロッキング制約として強制する
   (ビジネスロジック・データ変換・シリアライズ・ステートフル要素のあるプロジェクト向け推奨)
B) Partial — 純粋関数とシリアライズの round-trip にのみ PBT を強制する
   (アルゴリズム的複雑度が限定的なプロジェクト向け)
C) No — すべての PBT ルールをスキップする
   (シンプル CRUD・UI のみ・薄い統合層のプロジェクト向け)
X) Other (please describe after [Answer]: tag below)

> 補足: 本プロジェクトには「迷惑リスク判定 (プロキシ指標 → フラグ計算)」「綺麗度判定 (LLM 入力組み立て)」
> 「称号付与の条件評価」など、純粋関数で表現可能なロジックが含まれます。

[Answer]: B
**PBT 対象 (純粋関数 / round-trip 限定)**:
- 迷惑リスク判定 (プロキシ指標 → フラグ計算)
- 綺麗度判定の入力組み立て (LLM 入力構築)
- 称号付与の条件評価
**理由**: プロトタイプ全体で PBT を強制するとハッカソンの開発速度を阻害するが、上記 3 つは純粋関数で表現可能なロジックなのでここだけ堅牢性を担保したい。

---

## Question 4: プラットフォーム選択 (intent.md「未確定」項目)

intent.md「技術スタック」では「プラットフォーム: チームの実装スキルで決定 (最初の Bolt)」とされていますが、
書類審査用 PRFAQ・Application Design では具体的なターゲットを明示する必要があります。
予選提出物として PRFAQ に記載するプラットフォームを 1 つ選んでください。

A) iOS (HealthKit) のみ — Swift / SwiftUI を想定。HealthKit が成熟しているため実装容易
B) Android (Health Connect) のみ — Kotlin / Jetpack Compose を想定
C) iOS + Android 両対応 — Flutter または React Native を想定 (実装負荷高)
D) Web デモ + 擬似ヘルスケアデータ — 端末認可不要、デモ動画が撮りやすい (R2 撤退案)
X) Other (please describe after [Answer]: tag below)

[Answer]: X — 書類審査 (Inception) 段階では確定しない
**取り扱い方針**:
- PRFAQ には「iOS (HealthKit) を第一候補、擬似データ Web デモを撤退案 (R2 対策)」と記載
- 最終確定は予選通過後の最初の Bolt で実装スキル評価とともに実施 (R5 対策と整合)
- Application Design は **iOS HealthKit を仮の前提** とし、撤退案として **擬似データ層への抽象化ポイントを明記** すること

**【後日確定 / 2026-05-08T02:48:55Z】 最終回答: A — iOS (HealthKit) のみ確定**
- Q4 の最終確定回答は **A (iOS 確定)**。Swift / SwiftUI を採用と確定
- 撤退ルートを「擬似データ Web デモ」から **「iOS のまま擬似データモード (DEBUG ビルド)」** に再定義 (Web 版撤退案は採用しない / R2 改定 / R5 解消)
- Adapter パターンの責務: (1) ユニットテストでのモック注入 (2) DEBUG ビルドでの擬似データモード差し替え
- 詳細は `requirements.md` Q4 セクション + `application-design.md` Section 11 (Q4 Implementation Plan) 参照
- 上記 [Answer]: X (書類審査段階では確定しない) の取り扱い方針は **履歴として残す** が、最終確定は本補記の通り

---

## Question 5: AWS アカウント・Bedrock モデルアクセスの準備状況

intent.md の Risk Register R8 (AWS アカウント準備の遅延 / 重要度 高) と R9 (Bedrock モデルアクセス申請 / 重要度 中)
の現状を確認させてください。Application Design / Units Generation で前提として扱う必要があるためです。

A) 両方準備済み — AWS アカウントあり、Bedrock (Claude) モデルアクセス申請も承認済み
B) アカウントのみ準備済み — AWS アカウントはあるが、Bedrock モデルアクセスは未申請または審査中
C) 両方未準備 — Inception 完了までに準備する想定 (タスクとして明示する必要あり)
D) PRFAQ 段階では「想定環境」として記載し、実装段階で確定する
X) Other (please describe after [Answer]: tag below)

[Answer]: D
**取り扱い方針** (PRFAQ・Application Design への反映指示):
- AWS アカウント: 「予選通過後の最初の Bolt で準備」と明記 (R8 対策と整合)
- Bedrock Claude モデルアクセス: 「ap-northeast-1 (Tokyo) リージョンで Claude モデルアクセスを予選通過後の最初の Bolt で申請・承認取得する」と明記 (R9 対策と整合)
- リスク: 申請承認に時間がかかる可能性 (R9) は Risk Register で既に重要度: 中で記載済みのため、予選通過直後の最優先タスクとして明示する

---

## 回答ガイド

- 不明な点があれば X (Other) を選び、詳細を記述してください
- 質問 1 (Story 数) は **正本の確定** が User Stories ステージの前提となります
- 質問 2/3 は extensions の有効化を確定し、以後の全ステージで適用ルールを決定します
- 質問 4/5 は Application Design / Units Generation で前提条件として参照します
