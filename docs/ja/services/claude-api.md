# Claude API 統合

> Claude Code v2.1.88 ソースコード解析 - `src/services/api/` ディレクトリ

## 概要

Claude Code の API 層は `src/services/api/` 配下に構成され、Anthropic の `@anthropic-ai/sdk` を通じて Claude モデルとの通信を管理する。ストリーミング応答処理、ツール呼び出しの調整、プロンプトキャッシング、コスト追跡、リトライロジックなどを包括的に実装している。

## ファイル構成

| ファイル | 役割 |
|---|---|
| `claude.ts` | メインクエリエンジン。モデルへのリクエスト組み立て・送信・応答処理 |
| `client.ts` | Anthropic SDK クライアントの生成・認証管理 |
| `withRetry.ts` | リトライロジック（指数バックオフ、529/429 対応） |
| `errors.ts` | API エラーの分類・メッセージ生成 |
| `logging.ts` | API リクエスト/レスポンスのロギング・ゲートウェイ検出 |
| `usage.ts` | 使用量（利用率）の取得 |
| `errorUtils.ts` | 接続エラー詳細の抽出 |
| `emptyUsage.ts` | 空の Usage オブジェクト定数 |
| `promptCacheBreakDetection.ts` | プロンプトキャッシュ破壊の検出 |
| `bootstrap.ts` | API 初期化処理 |

## Claude API クライアントの構造

### クライアント生成 (`client.ts`)

`getAnthropicClient()` 関数がプロバイダに応じた SDK クライアントを生成する（L88-190）。

```typescript
export async function getAnthropicClient({
  apiKey, maxRetries, model, fetchOverride, source
}: { ... }): Promise<Anthropic>
```

**サポートするプロバイダ:**

| プロバイダ | 環境変数 | SDK |
|---|---|---|
| Direct API (1P) | `ANTHROPIC_API_KEY` | `Anthropic` |
| AWS Bedrock | `CLAUDE_CODE_USE_BEDROCK` | `AnthropicBedrock` |
| Azure Foundry | `CLAUDE_CODE_USE_FOUNDRY` | `AnthropicFoundry` |
| Google Vertex AI | `CLAUDE_CODE_USE_VERTEX` | `AnthropicVertex` |

各クライアントには以下の共通ヘッダーが付与される（`client.ts` L105-116）:

- `x-app: cli` - アプリ識別子
- `User-Agent` - ユーザーエージェント文字列
- `X-Claude-Code-Session-Id` - セッション ID
- `ANTHROPIC_CUSTOM_HEADERS` 環境変数によるカスタムヘッダー

### 認証フロー

1. **OAuth トークン**: `checkAndRefreshOAuthTokenIfNeeded()` で自動リフレッシュ（L132-133）
2. **API キー**: `ANTHROPIC_API_KEY` 環境変数または API キーヘルパーから取得
3. **Bedrock**: AWS 認証情報（アクセスキー / セッショントークン）を動的にリフレッシュ
4. **Vertex**: Google Auth ライブラリ経由の GCP 認証
5. **Foundry**: Azure AD トークンプロバイダまたは API キー

## ストリーミングメッセージ処理

### メインクエリ関数 (`claude.ts`)

2 つのエントリポイントが提供される:

```typescript
// ストリーミングなし（完了まで待機）
export async function queryModelWithoutStreaming({ ... }): Promise<AssistantMessage>

// ストリーミングあり（逐次イベント返却）
export async function* queryModelWithStreaming({ ... }): AsyncGenerator<StreamEvent | AssistantMessage | SystemAPIErrorMessage, void>
```

両方とも内部的に `queryModel()` ジェネレータ関数を使用する（L752-780）。`queryModelWithStreaming` は `AsyncGenerator` を返し、以下の型のイベントを yield する:

- `StreamEvent` - ストリーミング中のデルタイベント（テキスト、thinking ブロック等）
- `AssistantMessage` - 完成したアシスタントメッセージ
- `SystemAPIErrorMessage` - API エラーメッセージ

### VCR (Video Cassette Recorder) パターン

`withStreamingVCR` / `withVCR` ラッパーにより、API リクエストの録画・再生が可能（テスト・デバッグ用）。

## ツール呼び出し/結果の調整

### Options 型 (`claude.ts` L676-707)

`Options` 型がクエリの全設定を定義する:

```typescript
export type Options = {
  getToolPermissionContext: () => Promise<ToolPermissionContext>
  model: string
  toolChoice?: BetaToolChoiceTool | BetaToolChoiceAuto
  mcpTools: Tools              // MCP サーバー由来のツール
  extraToolSchemas?: BetaToolUnion[]
  agents: AgentDefinition[]    // サブエージェント定義
  querySource: QuerySource     // クエリの発信源（repl_main_thread, agent:custom 等）
  taskBudget?: { total: number; remaining?: number }  // API 側タスクバジェット
  // ...
}
```

### ツールスキーマの変換

`toolToAPISchema()` ユーティリティ（`src/utils/api.ts`）が内部 `Tool` 型を API の `BetaToolUnion` に変換する。MCP ツールは `mcp__<server>__<tool>` 形式の名前で登録される。

### ツール検索 (Tool Search / Deferred Tools)

LSP 初期化が完了していないツールは `defer_loading: true` フラグ付きで送信される（`shouldDeferLspTool()`, L786-793）。`TOOL_SEARCH_TOOL_NAME` によるツール検索機能も統合されている。

## ビジョン（画像）処理

### 画像関連のエラーハンドリング (`errors.ts`)

以下のメディアエラーが定義されている:

- **画像サイズ超過**: `getImageTooLargeErrorMessage()` (L186-189)
- **PDF ページ数超過**: `getPdfTooLargeErrorMessage()` - 最大ページ数と最大ファイルサイズ制限 (L170-175)
- **PDF パスワード保護**: `getPdfPasswordProtectedErrorMessage()` (L176-179)
- **リクエストサイズ超過**: `getRequestTooLargeErrorMessage()` (L192-196)

`isMediaSizeError()` 関数（L133-139）がメディアサイズ拒否を検出し、`stripImagesFromMessages` によるリトライを可能にする。

### メディア制限定数

`API_MAX_MEDIA_PER_REQUEST`（`src/constants/apiLimits.ts`）がリクエストあたりの最大メディア数を制限する。

## コスト追跡と使用量管理

### コスト追跡 (`cost-tracker.ts`)

セッション単位でトークン使用量とコストを集計する:

```typescript
// bootstrap/state.ts の状態管理関数群
getTotalCostUSD()
getTotalInputTokens()
getTotalOutputTokens()
getTotalCacheReadInputTokens()
getTotalCacheCreationInputTokens()
getModelUsage()          // モデル別使用量
getTotalAPIDuration()    // API 呼び出し総時間
```

`addToTotalSessionCost()` が各 API レスポンス後に呼ばれ、`calculateUSDCost()` で USD コストを算出する。

### 使用量取得 (`usage.ts`)

Claude.ai サブスクライバー向けに利用率情報を取得する:

```typescript
type Utilization = {
  five_hour?: RateLimit | null     // 5 時間レート制限
  seven_day?: RateLimit | null     // 7 日間レート制限
  seven_day_opus?: RateLimit | null // Opus 7 日間制限
  seven_day_sonnet?: RateLimit | null // Sonnet 7 日間制限
  extra_usage?: ExtraUsage | null  // 超過利用情報
}
```

### プロンプトキャッシング

`getPromptCachingEnabled()` (L333-356) がモデル別のプロンプトキャッシュ制御を行う:

- `DISABLE_PROMPT_CACHING` - グローバル無効化
- `DISABLE_PROMPT_CACHING_HAIKU` / `_SONNET` / `_OPUS` - モデル別無効化

`getCacheControl()` (L358-374) がキャッシュ制御ヘッダーを生成し、1 時間 TTL の条件判定（`should1hCacheTTL`, L393-434）を行う。GrowthBook のアロウリストに基づいてクエリソースごとに TTL を適用する。

## レート制限とリトライ

### リトライエンジン (`withRetry.ts`)

```typescript
export async function* withRetry<T>(
  getClient: () => Promise<Anthropic>,
  operation: (client: Anthropic, attempt: number, context: RetryContext) => Promise<T>,
  options: RetryOptions,
): AsyncGenerator<SystemAPIErrorMessage, T>
```

**主要パラメータ:**

| 定数 | 値 | 説明 |
|---|---|---|
| `DEFAULT_MAX_RETRIES` | 10 | デフォルト最大リトライ回数 |
| `MAX_529_RETRIES` | 3 | 529 Overloaded の最大リトライ |
| `BASE_DELAY_MS` | 500 | 基本遅延（ミリ秒） |
| `FLOOR_OUTPUT_TOKENS` | 3000 | 最小出力トークン数 |

**リトライ戦略:**

1. **529 Overloaded**: フォアグラウンドクエリソースのみリトライ（`FOREGROUND_529_RETRY_SOURCES`, L62-82）。バックグラウンドタスク（サマリー、タイトル生成等）は即座に失敗させ、ゲートウェイ増幅を防ぐ
2. **429 Rate Limit**: 標準のバックオフリトライ
3. **接続エラー**: `ECONNRESET` / `EPIPE` はステール接続として処理（`isStaleConnectionError`, L112-118）
4. **モデルフォールバック**: `FallbackTriggeredError` で代替モデルへの切り替え

**持続リトライモード** (`CLAUDE_CODE_UNATTENDED_RETRY`):

無人セッション向けに 429/529 を無限リトライ:
- 最大バックオフ: 5 分 (`PERSISTENT_MAX_BACKOFF_MS`)
- リセットキャップ: 6 時間 (`PERSISTENT_RESET_CAP_MS`)
- ハートビート間隔: 30 秒 (`HEARTBEAT_INTERVAL_MS`)

### CannotRetryError / FallbackTriggeredError

```typescript
class CannotRetryError extends Error {
  constructor(public readonly originalError: unknown, public readonly retryContext: RetryContext)
}

class FallbackTriggeredError extends Error {
  constructor(public readonly originalModel: string, public readonly fallbackModel: string)
}
```

## エラーハンドリング

### エラー分類 (`errors.ts`)

主要なエラーメッセージ定数:

| 定数 | 説明 |
|---|---|
| `API_ERROR_MESSAGE_PREFIX` | "API Error" |
| `PROMPT_TOO_LONG_ERROR_MESSAGE` | "Prompt is too long" |
| `CREDIT_BALANCE_TOO_LOW_ERROR_MESSAGE` | クレジット不足 |
| `INVALID_API_KEY_ERROR_MESSAGE` | API キー無効 |
| `TOKEN_REVOKED_ERROR_MESSAGE` | OAuth トークン失効 |
| `REPEATED_529_ERROR_MESSAGE` | 繰り返し 529 エラー |
| `CUSTOM_OFF_SWITCH_MESSAGE` | Opus 高負荷時の Sonnet 切り替え推奨 |
| `API_TIMEOUT_ERROR_MESSAGE` | タイムアウト |

### プロンプト長超過の解析

```typescript
export function parsePromptTooLongTokenCounts(rawMessage: string): {
  actualTokens: number | undefined
  limitTokens: number | undefined
}
```

`getPromptTooLongTokenGap()` がリアクティブコンパクションのグループスキップ量を決定する。

### ゲートウェイ検出 (`logging.ts`)

レスポンスヘッダーから AI ゲートウェイ（プロキシ）を自動検出する:

| ゲートウェイ | 検出方法 |
|---|---|
| LiteLLM | `x-litellm-*` ヘッダー |
| Helicone | `helicone-*` ヘッダー |
| Portkey | `x-portkey-*` ヘッダー |
| Cloudflare AI Gateway | `cf-aig-*` ヘッダー |
| Kong | `x-kong-*` ヘッダー |
| Braintrust | `x-bt-*` ヘッダー |
| Databricks | ホスト名サフィックス |

## Effort パラメータと Beta ヘッダー

### Effort 設定 (`claude.ts` L440-466)

`configureEffortParams()` がモデルの推論努力レベルを設定する:

- 文字列 effort → `output_config.effort` に直接設定
- 数値 effort → `anthropic_internal.effort_override`（ant-only）
- 未指定 → `EFFORT_BETA_HEADER` を追加

### Task Budget (`claude.ts` L479-501)

API 側のトークンバジェット認識機能:

```typescript
type TaskBudgetParam = {
  type: 'tokens'
  total: number
  remaining?: number
}
```

`configureTaskBudgetParams()` が `output_config.task_budget` を設定し、`TASK_BUDGETS_BETA_HEADER` を追加する。

### Beta ヘッダー一覧

`src/constants/betas.ts` で定義される主要な Beta ヘッダー:

- `AFK_MODE_BETA_HEADER` - AFK モード
- `CONTEXT_1M_BETA_HEADER` - 1M コンテキスト
- `CONTEXT_MANAGEMENT_BETA_HEADER` - コンテキスト管理
- `EFFORT_BETA_HEADER` - 推論 effort
- `FAST_MODE_BETA_HEADER` - 高速モード
- `PROMPT_CACHING_SCOPE_BETA_HEADER` - キャッシュスコープ
- `STRUCTURED_OUTPUTS_BETA_HEADER` - 構造化出力
- `TASK_BUDGETS_BETA_HEADER` - タスクバジェット
- `ADVISOR_BETA_HEADER` - アドバイザー

## API メタデータ

`getAPIMetadata()` (L503-528) がリクエストに付与するメタデータを生成する:

```typescript
{
  user_id: JSON.stringify({
    device_id: getOrCreateUserID(),
    account_uuid: getOauthAccountInfo()?.accountUuid ?? '',
    session_id: getSessionId(),
    // CLAUDE_CODE_EXTRA_METADATA 環境変数からの追加フィールド
  })
}
```

## 反蒸留対策

`getExtraBodyParams()` 内で、1P CLI かつフィーチャーフラグ有効時に `anti_distillation: ['fake_tools']` を送信する（L301-313）。偽ツールインジェクションによりモデル出力の蒸留を防止する。
