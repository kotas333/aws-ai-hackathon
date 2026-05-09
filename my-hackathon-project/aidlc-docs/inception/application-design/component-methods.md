# Component Methods — メソッドシグネチャと I/O 仕様

> **作成日**: 2026-05-07T15:00:47Z
> **対応ルール**: `.aidlc-rule-details/inception/application-design.md` Step 10
> **関連**: `components.md` (コンポーネント定義) / `services.md` (サービス層) / `component-dependency.md` (依存関係)

## 注意

- **本ファイルは I/O とメソッド契約のみを定義** する。詳細業務ルール (例: 迷惑リスク判定の具体閾値、称号付与の具体条件、トーンシフト発動の具体条件) は **Functional Design (per-Unit / Construction Phase)** で詳細化される
- 型表記は擬似 TypeScript 風 (言語非依存 / 概念表現用)
- **PBT 対象** マーカーは Property-Based Testing (Partial 適用) の対象メソッド

---

## 共通データ型

```typescript
type DeviceUUID = string;  // 端末 UUID (端末識別 / SECURITY-04)
type ISODate = string;     // YYYY-MM-DD
type ISODateTime = string; // ISO 8601

type HealthSummary = {
  steps: number;                // 歩数
  activeEnergyKcal: number;     // アクティブエネルギー (kcal)
  exerciseMinutes: number;      // 運動時間 (分)
  heartRateAvg: number | null;  // 心拍数 (平均 bpm)
  sleepHours: number | null;    // 睡眠時間 (h)
  standMinutes: number;         // 立ち上がり時間 (分)
  // 注: 個別の生レコードは含まない (NFR-DAT-02 / R3: 端末ローカル限定)
  asOf: ISODateTime;
};

type EnvironmentData = {
  tempC: number;
  weather: 'clear' | 'cloudy' | 'rain' | 'snow' | 'other';
  humidityPct: number;
  asOf: ISODateTime;
};

// CalendarSummary: FR-12 で追加 (FR-12 / US-1.6)
// 機微情報 (タイトル / 場所 / 参加者) は端末ローカル限定 / AWS には集計値のみ送信
type CalendarSummary = {
  isHoliday: boolean | null;          // 翌日が休日かどうか (null = 取得不可 / 権限拒否)
  earliestEventTime: ISODateTime | null;  // 翌日の最早開始時刻
  eventCount: number | null;          // 翌日のイベント総数
  asOf: ISODateTime;
};

type RiskLevel = 'low' | 'medium' | 'high';

type AnnoyanceRiskFlag = {
  level: RiskLevel;
  hoursSinceLastBath: number | null;
  movementScore: number;  // 運動量合算 (純粋関数)
  tempC: number | null;
  humidityPct: number | null;
  heartRateAvg: number | null;
  hasMissingData: boolean;
};

type Choice = 'bath' | 'skip';  // 入る / サボる

type DialogueTurn = {
  speaker: 'judge' | 'devil';   // 改名: 'angel' → 'judge' (Application Design Section 1.5 命名変更記録)
  message: string;
};

type Dialogue = {
  turns: DialogueTurn[];   // 4〜6 ターン / 最後は必ず 'devil' (US-2.1 AC)
  generatedAt: ISODateTime;
};

// AwardedTitle: AWS 側で永続化される「ユーザーが獲得した称号」レコード
// AWS-shift により name/description は含まない (静的メタは C-07 から取得)
type AwardedTitle = {
  id: string;             // 事前リスト 10 件以上のいずれか (R13)
  awardedAt: ISODateTime;
};

// TitleCatalogEntry: C-07 Title Catalog Distribution (S3+CloudFront) が配信する静的メタ
type TitleCatalogEntry = {
  id: string;
  name: string;
  description: string;
  category?: string;      // 例: '歓迎' / '連続系' / '累計系' / '過激系' / '状況系' / 'パターン系' / '特殊'
};

type TitleCatalog = {
  version: string;        // 例: '2026-05-08T00:00:00Z' / S3 オブジェクトメタの最終更新時刻
  entries: TitleCatalogEntry[];
};

// AffirmationMessage: S-02 が DDB から取得する選択後全肯定メッセージ (両キャラ分)
type AffirmationMessage = {
  id: string;
  judgeMessage: string;   // 改名後: 元 angel
  devilMessage: string;
};

type SelectionRecord = {
  deviceUUID: DeviceUUID;
  selectedAt: ISODateTime;
  choice: Choice;
  healthSummary: HealthSummary | null;
  environmentData: EnvironmentData | null;
  riskLevel: RiskLevel | null;
};
```

---

## C-01: Mobile Client

### Adapter インターフェース (iOS 確定 / DEBUG ビルド擬似データモード / Section 11 Q4 Implementation Plan 参照)

```typescript
interface HealthDataAdapter {
  isAvailable(): Promise<boolean>;
  requestPermission(): Promise<boolean>;
  fetchTodaySummary(): Promise<HealthSummary | null>;
}

interface LocationDataAdapter {
  isAvailable(): Promise<boolean>;
  requestPermission(): Promise<boolean>;
  fetchCurrentLocation(): Promise<{ latitude: number; longitude: number } | null>;
}

// CalendarDataAdapter: FR-12 で追加 (FR-12 / US-1.6)
// iOS EventKit から翌日の予定サマリを取得
interface CalendarDataAdapter {
  isAvailable(): Promise<boolean>;
  requestPermission(): Promise<boolean>;
  fetchTomorrowSummary(): Promise<CalendarSummary | null>;
  // 振る舞い:
  // - EventKit から翌日 (00:00 〜 23:59 端末ローカル時刻) のイベントを取得
  // - 集計のみ実施 (タイトル/場所/参加者は読まない / 端末ローカルからも出さない)
  // - 終日イベントが「休日」相当の場合 isHoliday=true
}

// NotificationScheduler: FR-14 で追加 (FR-14 / US-3.3)
// iOS UNUserNotificationCenter でローカル通知をスケジュール
interface NotificationScheduler {
  isAvailable(): Promise<boolean>;
  requestPermission(): Promise<boolean>;
  scheduleAchievementCheck(input: {
    selectionId: string;
    fireAt: ISODateTime;       // 30 分後を想定
    bodyText: string;          // 悪魔キャラ寄りの文言 (詳細は Functional Design)
    actionYes: { label: string; identifier: 'YES_DID_BATH' };
    actionNo: { label: string; identifier: 'NO_NOT_YET' };
  }): Promise<void>;
  cancel(selectionId: string): Promise<void>;
}

// 実装パターン (iOS 確定 / Section 11 Q4 Implementation Plan 参照):
// - Production (Release): HealthKitAdapter / CoreLocationAdapter / EventKitAdapter / LocalNotificationScheduler
// - DEBUG ビルド (擬似データモード): PseudoHealthAdapter / PseudoLocationAdapter / PseudoCalendarAdapter / LoggingNotificationScheduler
// - Test: Mock 注入 (Adapter インターフェースに対する Stub/Mock)
```

### 主要画面ロジックメソッド

```typescript
// US-5.1, US-5.5
function showOnboarding(): Promise<{ accepted: boolean }>;

// US-1.1, US-1.2, US-1.3, US-1.6 (Mobile 側のデータ取得 / FR-12 でカレンダー追加)
function collectInputs(): Promise<{
  health: HealthSummary | null;
  location: { latitude: number; longitude: number } | null;
  calendar: CalendarSummary | null;       // FR-12: FR-12 / US-1.6
}>;

// US-2.1 (Dialogue API 呼び出し)
function requestDialogue(input: {
  deviceUUID: DeviceUUID;
  health: HealthSummary | null;
  location: { latitude: number; longitude: number } | null;
  calendar: CalendarSummary | null;       // FR-12: FR-12 / US-1.6
  characterSetId?: string;  // US-2.3 Should: キャラ切替の拡張点
}): Promise<Dialogue>;

// US-5.7 (翌日予定ホーム画面ミニ表示 / FR-12 / FR-13)
// US-1.6 と同じ EventKit 経由の取得を再利用 (キャッシュ戦略は Functional Design / O-14)
function getTomorrowMiniSummary(): Promise<CalendarSummary | null>;

// US-3.1 (選択送信) — AWS-shift: affirmation を DDB 由来でレスポンスに含める / FR-14: 達成確認スケジュール
function submitSelection(input: {
  deviceUUID: DeviceUUID;
  choice: Choice;
  health: HealthSummary | null;
  environment: EnvironmentData | null;
  riskLevel: RiskLevel | null;
}): Promise<{
  selectionId: string;                       // FR-14: 達成確認エンドポイント呼び出し用
  newTitles: AwardedTitle[];                 // id + awardedAt のみ (静的メタは catalog から)
  affirmation: AffirmationMessage;           // S-02 AWS-shift: DDB 由来の両キャラ全肯定メッセージ
  achievementCheckScheduledAt: ISODateTime | null;  // FR-14: choice='bath' 時のみ非 null (現在時刻 + 30 分)
}>;

// US-3.3 (達成確認応答 / FR-14 / FR-14)
function submitAchievement(input: {
  selectionId: string;
  achieved: boolean;
}): Promise<{ recordedAt: ISODateTime }>;

// 通知スケジュール (FR-14 / FR-14): submitSelection レスポンス受領後に Mobile 側で実行
// - choice='bath' の場合のみ NotificationScheduler.scheduleAchievementCheck() を呼ぶ
// - 通知発火後にユーザーが Yes/まだ を選択すると submitAchievement() を呼ぶ

// US-3.2 (履歴取得)
function fetchHistory(input: {
  deviceUUID: DeviceUUID;
  fromMonth: string;  // YYYY-MM
  toMonth: string;
}): Promise<{ days: Array<{ date: ISODate; status: 'bath' | 'skip' | 'none' }> }>;

// US-4.2 (称号一覧) — AWS-shift: 静的メタは C-07 catalog から取得し Mobile 側で結合
function fetchAwardedTitles(deviceUUID: DeviceUUID): Promise<{ titles: AwardedTitle[] }>;

// C-07 Title Catalog Distribution からの取得 (CloudFront / Mobile 起動時に取得しローカルキャッシュ)
function fetchTitleCatalog(): Promise<TitleCatalog>;
// 振る舞い:
// - URL: https://<cloudfront-domain>/titles-catalog.json (環境変数で設定)
// - HTTP If-None-Match (ETag) でキャッシュ最適化
// - 失敗時はローカルキャッシュ (前回値) を継続使用
// - 起動時 + 設定画面からの「カタログ更新」ボタンで取得
```

---

## C-02: Dialogue API

### エンドポイント

```typescript
// US-2.1 / FR-05, FR-06
// POST /dialogue
type DialogueRequest = {
  deviceUUID: DeviceUUID;
  health: HealthSummary | null;
  location: { latitude: number; longitude: number } | null;
  calendar: CalendarSummary | null;       // FR-12: FR-12 / US-1.6
  characterSetId?: string;  // 省略時は 'standard'
};

type DialogueResponse = {
  dialogue: Dialogue;
  riskLevel: RiskLevel | null;
  // 注: 環境データ自体は返さない (Mobile が必要なら別途取得 / プライバシー最小化)
};
```

### 内部メソッド (純粋関数 / **PBT 対象 / FR-05**)

```typescript
// 綺麗度判定の入力組み立て (LLM 入力構築)
// PBT-02, 03, 07, 08, 09 適用範囲
function buildPrompt(input: {
  health: HealthSummary | null;
  environment: EnvironmentData | null;
  riskFlag: AnnoyanceRiskFlag;
  recentSkipPattern: { skipDays: number; consecutiveSkipDays: number };
  calendarSummary: CalendarSummary | null;   // FR-12: FR-12 / 翌日予定の文脈付与
  hoursSinceLastBath: number | null;         // DD-03: C-04 getLastBathTime() の結果
  characterSetId: string;
  systemPromptVersion: string;
}): { systemPrompt: string; userPrompt: string };

// プロンプト構築の特性 (PBT で検証):
// - 同一入力 → 同一出力 (冪等)
// - 機微情報 (生 health 個別レコード等) を含まない (SECURITY-13)
// - characterSetId が未知の場合は 'standard' にフォールバック
// - systemPromptVersion がレスポンス追跡用に常にプロンプトに含まれる
```

### 内部呼び出し

```typescript
// → C-03: 迷惑リスク計算 (calculateAnnoyanceRisk / hoursSinceLastBath は C-04 から取得した値を渡す)
// → C-05: 天気データ取得
// → C-04: 連続サボり長期化サマリ取得 (US-2.4 / FR-07 / getRecentSkipPattern)
// → C-04: 最終 'bath' タイムスタンプ取得 (DD-03 / getLastBathTime / 並列で呼び出し)
// → Bedrock: Claude 呼び出し
```

---

## C-03: Risk Calculator

### 主要メソッド (純粋関数 / **PBT 対象 / FR-04**)

```typescript
// US-1.5 / FR-04
// PBT-02, 03, 07, 08, 09 適用範囲
function calculateAnnoyanceRisk(input: {
  hoursSinceLastBath: number | null;   // DD-03: C-04 getLastBathTime() 経由で C-02 が取得して渡す
  health: HealthSummary | null;
  environment: EnvironmentData | null;
}): AnnoyanceRiskFlag;

// 特性 (PBT で検証):
// - 出力 level は 'low' | 'medium' | 'high' のいずれか (全域性)
// - 同一入力 → 同一出力 (冪等)
// - 全データ欠損時 hasMissingData=true で level='low' (安全側)
// - hoursSinceLastBath > 72 のときは level='medium' 以上 (US-1.5 AC)
// - 詳細閾値は Functional Design で確定
```

---

## C-04: History & Title Service

### エンドポイント

```typescript
// US-3.1 / FR-08 — AWS-shift: affirmation を DDB から取得しレスポンスに含める
// POST /selections
type SelectionRequest = {
  deviceUUID: DeviceUUID;
  choice: Choice;
  health: HealthSummary | null;
  environment: EnvironmentData | null;
  riskLevel: RiskLevel | null;
};

type SelectionResponse = {
  selectionId: string;                       // FR-14: 達成確認エンドポイント呼び出し用
  recordedAt: ISODateTime;
  newTitles: AwardedTitle[];                 // id + awardedAt のみ (静的メタは catalog から)
  affirmation: AffirmationMessage;           // S-02 AWS-shift: DDB 由来の両キャラ全肯定メッセージ
  achievementCheckScheduledAt: ISODateTime | null;  // FR-14: choice='bath' のみ非 null (recordedAt + 30min)
};

// FR-14 / FR-14 / US-3.3
// POST /selections/{selectionId}/achievement
type AchievementRequest = {
  selectionId: string;            // path parameter
  achieved: boolean;              // true = 入った / false = まだ
};

type AchievementResponse = {
  recordedAt: ISODateTime;
};

// US-3.2 / FR-09
// GET /history?deviceUUID=XXX&from=YYYY-MM&to=YYYY-MM
type HistoryResponse = {
  days: Array<{
    date: ISODate;
    status: 'bath' | 'skip' | 'none';
    selectionId?: string;  // 詳細表示用
  }>;
};

// US-4.2 / FR-10 — AWS-shift: 名前/説明は C-07 catalog から / API は獲得 ID のみ返却
// GET /titles?deviceUUID=XXX
type TitlesResponse = {
  titles: AwardedTitle[];
};

// C-07: 静的メタ配信 (CloudFront / S3)
// GET https://<cloudfront-domain>/titles-catalog.json
// レスポンス: TitleCatalog (id → name/description のマップ)
// 注: API Gateway を経由しない。Lambda 不要。エッジキャッシュ済み。
```

### 内部メソッド (純粋関数 / **PBT 対象 / FR-10**)

```typescript
// US-4.1, US-4.2 / FR-10
// PBT-02, 03, 07, 08, 09 適用範囲
function evaluateNewTitles(input: {
  cumulativeStats: {
    totalSkipCount: number;
    totalBathCount: number;
    consecutiveSkipDays: number;
    sameDayOfWeekConsecutiveSkip: number;  // ソファと一体化 用
    skippedOnHotDay: boolean;             // 真夏のサボり貴族 用
    skippedOnRainyConsecutiveDays: number; // 梅雨どきのサボりマスター 用
    isFirstSkip: boolean;                 // ダメ人間勲章 用
  };
  alreadyAwardedTitleIds: string[];
  todaySelection: { choice: Choice; weather: EnvironmentData | null };
}): { newTitleIds: string[] };

// 特性 (PBT で検証):
// - newTitleIds はすべて事前リスト (10 件以上) のいずれか (R13 担保)
// - alreadyAwardedTitleIds に含まれる ID は newTitleIds に含めない (重複付与禁止)
// - 同一入力 → 同一出力 (冪等)
// - 全条件外なら newTitleIds=[]
```

### 内部メソッド (履歴サマリ提供 / FR-07)

```typescript
// US-2.4 / FR-07: 連続サボり長期化サマリ (C-02 から呼ばれる)
function getRecentSkipPattern(input: {
  deviceUUID: DeviceUUID;
  asOf: ISODateTime;
  windowDays: number;  // 例: 14
}): {
  skipDays: number;
  consecutiveSkipDays: number;
};

// DD-03 / FR-04: 最終 'bath' タイムスタンプ取得 (C-02 から呼ばれる / hoursSinceLastBath 計算用)
function getLastBathTime(input: {
  deviceUUID: DeviceUUID;
}): Promise<ISODateTime | null>;
// 振る舞い:
// - DynamoDB Query: PK=deviceUUID, SK begins_with `SELECTION#`, FilterExpression: choice='bath'
// - 最新の 1 件 (descending) を返す
// - 履歴に 'bath' なしの場合は null (初回利用時など)
// - C-02 が現在時刻との差分を計算して hoursSinceLastBath を求める
```

### 内部メソッド (達成フラグ更新 / FR-14 / FR-14 / US-3.3)

```typescript
// 通知から「Yes (入った)」「まだ (入ってない)」を選択した結果を SelectionRecord に記録
function markAchievement(input: {
  selectionId: string;
  achieved: boolean;
}): Promise<void>;
// 振る舞い:
// - DynamoDB UpdateItem: 該当 SelectionRecord の achieved フィールドを更新
// - 元の choice='bath' は維持 (上書きしない / US-3.3 AC4)
// - 同一 selectionId に対する複数回呼び出しは最後の値で上書き (Last-write-wins / Functional Design で確定)
```

### DynamoDB スキーマ拡張 (FR-14 / FR-14 反映)

既存の SelectionRecord に `achieved` フィールドを追加:

```
PK: <deviceUUID>
SK: 'SELECTION#<ISO8601>#<random>'
   selectionId: string  (= SK の random 部分または独立フィールド)

Item attributes:
  selectedAt: ISODateTime
  choice: 'bath' | 'skip'
  health: HealthSummary | null
  environment: EnvironmentData | null
  riskLevel: RiskLevel | null
  achieved: boolean | null   ← FR-14 で追加 (null=未確認 / true=達成 / false=未達)
                              ← choice='bath' のみで意味を持つ
                              ← choice='skip' の場合は常に null
```

### 内部メソッド (Affirmation 取得 / S-02 AWS-shift / US-3.1)

```typescript
// US-3.1 AC: 選択完了時に両キャラから全肯定メッセージを返す
// AWS-shift により Mobile 固定文言ではなく DDB 由来の文言テンプレートからランダム選択
function pickAffirmation(input: {
  choice: Choice;
  riskLevel: RiskLevel | null;  // 将来拡張: choice + riskLevel で文言群を絞り込む
}): Promise<AffirmationMessage>;

// 振る舞い:
// - DynamoDB Query: PK=`META#AFFIRMATIONS`, SK=`AFFIRMATION#<choice>#*`
// - 取得結果からランダム 1 件選択
// - 失敗時は最小限の Lambda バンドル文言にフォールバック (運用初期の移行期間用 / R10 料金枠超過時にも使用)
// - 文言テンプレートは事前審査済み (NFR-CON-03 品位 / R13 と同種の運用ガードレール)
```

### DynamoDB スキーマ (META#AFFIRMATIONS パーティション / S-02 AWS-shift)

```
PK: 'META#AFFIRMATIONS'
SK: 'AFFIRMATION#<choice>#<id>'
   choice: 'bath' | 'skip'
   id: zero-padded sequence (例: '001')

Item attributes:
  judgeMessage: string    // 改名後: 元 angel
  devilMessage: string
  category?: string       // 'casual' | 'caring' | 'humor' 等 (将来絞り込み用)
  enabled: boolean        // false で停止可能 (運用ガードレール)

初期投入: 各 choice 10〜20 件 (合計 20〜40 件)
```

---

## C-05: External Client

### 主要メソッド

```typescript
// US-1.3 / FR-03
function fetchWeather(input: {
  latitude: number;
  longitude: number;
}): Promise<EnvironmentData | null>;

// 振る舞い:
// - 外部天気 API 呼び出し (リトライ 1 回 / タイムアウト 1 秒)
// - 同一座標 30 分以内はキャッシュ (R10 料金対策)
// - 失敗時は null (US-1.3 AC: 環境データなしで対話生成に進める)
```

---

## C-07: Title Catalog Distribution (新規 / S-04 AWS-shift)

### 配置と前提
- **AWS サービス**: S3 (Bucket) + CloudFront (Distribution / OAI 経由)
- **対応コンポーネント**: C-01 から CloudFront URL を直接取得 (API Gateway 不経由)

### 主要コンテンツ

```typescript
// S3 オブジェクト: titles-catalog.json (1 ファイル)
// レスポンスは TitleCatalog 型 (本ファイル冒頭の共通型を参照)

// 例:
{
  "version": "2026-05-08T02:21:51Z",
  "entries": [
    { "id": "DAMENINGEN_KUNSHO", "name": "ダメ人間勲章", "description": "初サボリの歓迎称号", "category": "歓迎" },
    { "id": "AKUMA_NO_IINARI",  "name": "悪魔の言いなり", "description": "連続サボリ 7 日達成", "category": "連続系" }
    // ...計 10+ 件
  ]
}
```

### 配信仕様

| 項目 | 値 |
|---|---|
| URL | `https://<cloudfront-domain>/titles-catalog.json` (環境変数で Mobile に設定) |
| HTTP メソッド | GET のみ (公開読み取り) |
| 認証 | 不要 (内容に機微情報を含まない / 静的メタのみ) |
| キャッシュ | CloudFront edge cache (TTL: 1 時間推奨 / 更新時は invalidation) |
| ETag / If-None-Match | サポート (Mobile は ETag を保存し条件付き GET) |
| 暗号化 | TLS 1.2+ (CloudFront 標準) / S3 SSE-S3 |
| 更新フロー | 開発者が S3 にアップロード → CloudFront invalidation 実行 |
| バージョン管理 | S3 versioning 有効 |

### Mobile (C-01) との連携

```typescript
// C-01: アプリ起動時 + 設定画面の「カタログ更新」ボタンで実行
async function refreshTitleCatalog(): Promise<void> {
  const cached = readLocalCatalogCache();
  try {
    const fresh = await httpGetWithETag(catalogUrl, cached?.etag);
    if (fresh.notModified) return;       // 304: cached のまま継続
    writeLocalCatalogCache(fresh.body, fresh.etag);
  } catch {
    // ネットワーク失敗時はキャッシュ継続 / 初回のみ「カタログ取得失敗」表示
  }
}

// 称号一覧表示時:
async function displayTitles(deviceUUID: DeviceUUID) {
  const [{ titles }, catalog] = await Promise.all([
    fetchAwardedTitles(deviceUUID),
    readLocalCatalogCache(),
  ]);
  return titles.map(t => ({
    ...t,
    name: catalog.entries.find(e => e.id === t.id)?.name ?? t.id,
    description: catalog.entries.find(e => e.id === t.id)?.description ?? '',
  }));
}
```

---

## C-06: Infrastructure

### 主要 IaC 定義 (CDK Construct 概念レベル)

> 詳細は Infrastructure Design (per-Unit / Construction Phase) で詳細化。本ステージでは責務と境界のみ定義。

```typescript
// CDK Stack 構成例 (概念のみ / 実装は Construction Phase)
class FuroskanSupporterStack {
  apiGateway: RestApi;          // POST /dialogue, POST /selections, GET /history, GET /titles
  dialogueLambda: Function;     // C-02 + C-03 + C-05 をバンドル
  historyLambda: Function;      // C-04 (META#AFFIRMATIONS 読み取りも担当 / S-02 AWS-shift)
  table: Table;                 // DynamoDB シングルテーブル / SSE 有効 / META#AFFIRMATIONS パーティションを含む
  bedrockPolicy: PolicyStatement;  // InvokeModel 権限 (Claude Sonnet 4.6 / Opus 4.7 拡張可能)
  weatherApiSecret: Secret;     // Secrets Manager
  logGroup: LogGroup;           // CloudWatch Logs (構造化 / 機微情報除外)
  costAlarm: Alarm;             // R10 料金監視

  // C-07 Title Catalog Distribution (S-04 AWS-shift)
  catalogBucket: Bucket;        // titles-catalog.json / Versioning ON / SSE-S3
  catalogDistribution: Distribution;  // CloudFront / OAI 経由 / TTL 1h / TLS 1.2+
  catalogInvalidationDoc: Document;   // 更新時手順 (s3 cp + cloudfront create-invalidation)
}
```

---

## メソッド契約のカバレッジ確認

| FR | 主要メソッド | 担当 | PBT |
|---|---|---|---|
| FR-01, FR-02 | `HealthDataAdapter.fetchTodaySummary()` | C-01 | — |
| FR-03 | `LocationDataAdapter.fetchCurrentLocation()` + `External.fetchWeather()` | C-01 + C-05 | — |
| FR-04 | `calculateAnnoyanceRisk()` | C-03 | ✅ |
| FR-05 | `buildPrompt()` (LLM 入力組み立て) | C-02 | ✅ |
| FR-06 | `POST /dialogue` (Bedrock 呼び出しを含む) | C-02 | — (LLM 自体は外部) |
| FR-07 | `buildPrompt()` 内のフラグ反映 + `getRecentSkipPattern()` | C-02 + C-04 | — |
| FR-08 | `POST /selections` (+ `pickAffirmation()` / S-02 AWS-shift) | C-04 | — |
| FR-09 | `GET /history` | C-04 | — |
| FR-10 | `evaluateNewTitles()` (動的判定) + `fetchTitleCatalog()` (静的メタ / S-04 AWS-shift) | C-04 + C-07 | ✅ (FR-10 動的判定のみ) |
| FR-11 | UI 演出メソッド群 (画面コンポーネント内に内包) | C-01 | — |
| FR-12 | `CalendarDataAdapter.fetchTomorrowSummary()` + `buildPrompt()` 入力 | C-01 + C-02 | — (Adapter は I/O / buildPrompt は既に PBT) |
| FR-13 | `getTomorrowMiniSummary()` (Mobile UI 内 / iOS 標準カレンダーに遷移) | C-01 | — |
| FR-14 | `NotificationScheduler.scheduleAchievementCheck()` + `submitAchievement()` + `markAchievement()` + 新エンドポイント `POST /selections/{id}/achievement` | C-01 + C-04 | — (I/O 含むため対象外) |

すべての FR が 1 つ以上のメソッドに対応していることを確認。
PBT 対象 3 純粋関数 (FR-04 / FR-05 / FR-10 動的判定) が C-03 / C-02 / C-04 に明確に配置されている。
S-02 / S-04 AWS-shift により、選択後文言と称号メタが AWS 側 (DDB + S3/CloudFront) で管理される構成。
