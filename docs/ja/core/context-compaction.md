# コンテキストコンパクション

> 主要ソースファイル:
> - `src/services/compact/autoCompact.ts` (自動コンパクト制御)
> - `src/services/compact/compact.ts` (コンパクション実行)
> - `src/services/compact/microCompact.ts` (マイクロコンパクト)
> - `src/services/compact/sessionMemoryCompact.ts` (セッションメモリコンパクト)
> - `src/services/compact/prompt.ts` (コンパクトプロンプト)
> - `src/services/compact/grouping.ts` (メッセージグルーピング)
> - `src/utils/tokens.ts` (トークン推定)
> - `src/utils/tokenBudget.ts` (トークンバジェット解析)

## 概要

Claude Codeは複数層のコンテキスト管理戦略を持ち、コンテキストウィンドウの制限内で会話を継続可能にする。各戦略は異なるトリガー条件と圧縮強度を持つ。

```
コンテキスト管理の階層（軽量 → 重量）
──────────────────────────────────────
1. マイクロコンパクト (microCompact)
   - 古いツール結果の内容クリア
   - ローカルまたはキャッシュ編集

2. スニップコンパクト (snipCompact) [HISTORY_SNIP]
   - 古いメッセージの除去
   - トークン推定ベース

3. コンテキストコラプス (contextCollapse) [CONTEXT_COLLAPSE]
   - メッセージの折りたたみ
   - 段階的な圧縮

4. オートコンパクト (autoCompact)
   - セッションメモリコンパクト（優先）
   - レガシーコンパクト（フォールバック）

5. リアクティブコンパクト (reactiveCompact) [REACTIVE_COMPACT]
   - API 413エラーからの回復
   - 最後の手段
```

## 自動コンパクト (autoCompact)

### 閾値とバッファ (autoCompact.ts)

```typescript
// 主要な定数
AUTOCOMPACT_BUFFER_TOKENS = 13_000        // オートコンパクト閾値バッファ
WARNING_THRESHOLD_BUFFER_TOKENS = 20_000   // 警告閾値バッファ
ERROR_THRESHOLD_BUFFER_TOKENS = 20_000     // エラー閾値バッファ
MANUAL_COMPACT_BUFFER_TOKENS = 3_000       // 手動コンパクト用予約
MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3   // サーキットブレーカーの閾値
```

### 有効コンテキストウィンドウの計算 (L33-49)

```typescript
function getEffectiveContextWindowSize(model: string): number {
  const reservedTokensForSummary = Math.min(
    getMaxOutputTokensForModel(model),
    MAX_OUTPUT_TOKENS_FOR_SUMMARY,  // 20,000
  )
  let contextWindow = getContextWindowForModel(model, getSdkBetas())
  // CLAUDE_CODE_AUTO_COMPACT_WINDOW で上書き可能
  return contextWindow - reservedTokensForSummary
}
```

実効コンテキストウィンドウ = モデルのコンテキストウィンドウ - 要約出力用の予約トークン (20,000)

### オートコンパクト閾値 (L72-91)

```
オートコンパクト閾値 = 有効コンテキストウィンドウ - 13,000
```

`CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`環境変数でパーセンテージベースのオーバーライドが可能。

### トークン警告状態の計算 (L93-145)

`calculateTokenWarningState()`は複数の閾値を同時に評価:

| 状態 | 条件 |
|---|---|
| `isAboveWarningThreshold` | tokens >= threshold - 20,000 |
| `isAboveErrorThreshold` | tokens >= threshold - 20,000 |
| `isAboveAutoCompactThreshold` | オートコンパクト有効 かつ tokens >= 閾値 |
| `isAtBlockingLimit` | tokens >= 有効ウィンドウ - 3,000 |

### shouldAutoCompact() (L160-239)

コンパクトをトリガーすべきか判断する。以下の条件で**スキップ**される:

- `querySource`が`session_memory`、`compact`、`marble_origami`（再帰ガード）
- `DISABLE_COMPACT`または`DISABLE_AUTO_COMPACT`環境変数
- リアクティブオンリーモード (`tengu_cobalt_raccoon`フラグ)
- コンテキストコラプスが有効（コラプスがコンテキスト管理を担当）
- トークン数が閾値未満

### autoCompactIfNeeded() (L241-351)

実際のコンパクション実行:

1. **サーキットブレーカー**: `consecutiveFailures >= 3`で即座にスキップ
2. **セッションメモリコンパクト**を最初に試行 (L288-310)
3. 失敗時は**レガシーコンパクト** (`compactConversation()`) にフォールバック (L312-332)
4. 失敗時はサーキットブレーカーカウンタを増分

## セッションメモリコンパクト (sessionMemoryCompact.ts)

セッションメモリ（`SessionMemory`機能）が有効な場合に使われる軽量なコンパクション方式。

### 設計思想

APIにサマリーリクエストを送る代わりに、セッション中に蓄積されたセッションメモリの内容をサマリーとして使用する。これによりAPIコール（コスト、レイテンシ）を削減できる。

### 設定 (L47-66)

```typescript
const DEFAULT_SM_COMPACT_CONFIG = {
  minTokens: 10_000,              // 保持する最小トークン数
  minTextBlockMessages: 5,        // 保持するテキストブロックメッセージの最小数
  maxTokens: 40_000,              // 保持する最大トークン数（ハードキャップ）
}
```

リモート設定（GrowthBook）から上書き可能。

### calculateMessagesToKeepIndex() (L324-397)

コンパクション後に保持するメッセージの開始インデックスを計算:

1. `lastSummarizedMessageId`の位置から開始
2. `minTokens`と`minTextBlockMessages`の両方を満たすまで後方に拡張
3. `maxTokens`に達したら停止
4. `adjustIndexToPreserveAPIInvariants()`でtool_use/tool_resultペアの分断を防止

### adjustIndexToPreserveAPIInvariants() (L232-313)

API不変条件を維持するための調整:

1. **ツールペア保全**: 保持範囲にtool_resultが含まれる場合、対応するtool_useを含むアシスタントメッセージまで拡張
2. **思考ブロック保全**: 同じ`message.id`を共有するアシスタントメッセージ（思考ブロックを含む）を一緒に保持

## レガシーコンパクト (compact.ts)

### compactConversation()

フル会話のコンパクション処理:

1. **PreCompactフック**の実行
2. **サマリーリクエスト**の生成（`getCompactPrompt()`によるプロンプト）
3. **prompt-too-longリトライ** (L450-491): コンパクトリクエスト自体が長すぎる場合、`truncateHeadForPTLRetry()`で最古のAPIラウンドグループを削除して再試行（最大3回）
4. **ポストコンパクト処理**: ファイル添付の復元、スキル添付、MCP指示の再注入
5. **SessionStartフック**の実行

### CompactionResult (L299-310)

```typescript
interface CompactionResult {
  boundaryMarker: SystemMessage       // コンパクト境界マーカー
  summaryMessages: UserMessage[]       // サマリーメッセージ
  attachments: AttachmentMessage[]     // ファイル添付等
  hookResults: HookResultMessage[]     // フック結果
  messagesToKeep?: Message[]           // 保持するメッセージ（部分コンパクト）
  preCompactTokenCount?: number        // コンパクト前トークン数
  postCompactTokenCount?: number       // コンパクト後トークン数
  truePostCompactTokenCount?: number   // 真のポストコンパクトトークン数
  compactionUsage?: Usage              // コンパクションAPIの使用量
}
```

### buildPostCompactMessages() (L330-338)

コンパクション結果からメッセージ配列を構築:

```typescript
function buildPostCompactMessages(result: CompactionResult): Message[] {
  return [
    result.boundaryMarker,      // 境界マーカー
    ...result.summaryMessages,  // サマリー
    ...(result.messagesToKeep ?? []),  // 保持メッセージ
    ...result.attachments,      // 添付ファイル
    ...result.hookResults,      // フック結果
  ]
}
```

### truncateHeadForPTLRetry() (L243-291)

コンパクトリクエストがprompt-too-longになった場合の回復:

1. メッセージを`groupMessagesByApiRound()`でAPIラウンド単位にグルーピング
2. `tokenGap`（超過トークン数）に応じてグループを削除
3. `tokenGap`が不明な場合は全グループの20%を削除
4. 最低1グループは残す
5. アシスタント開始のメッセージ配列になった場合、合成ユーザーメッセージを先頭に挿入

## マイクロコンパクト (microCompact.ts)

### 概要

マイクロコンパクトは、フルコンパクション（サマリー生成）を行わずに、古いツール結果の内容をクリアしてトークンを節約する軽量な手法。

### 対象ツール (L41-50)

```typescript
const COMPACTABLE_TOOLS = new Set([
  FILE_READ_TOOL_NAME, ...SHELL_TOOL_NAMES,
  GREP_TOOL_NAME, GLOB_TOOL_NAME,
  WEB_SEARCH_TOOL_NAME, WEB_FETCH_TOOL_NAME,
  FILE_EDIT_TOOL_NAME, FILE_WRITE_TOOL_NAME,
])
```

### 時間ベースマイクロコンパクト (L446-530)

最後のアシスタントメッセージからの経過時間が閾値を超えた場合にトリガーされる。サーバーキャッシュが失効しているため、ツール結果のクリアによるキャッシュヒットへの影響がない。

```typescript
function evaluateTimeBasedTrigger(messages, querySource): {
  gapMinutes: number; config: TimeBasedMCConfig
} | null
```

動作:
1. `gapThresholdMinutes`を超えた場合にトリガー
2. `keepRecent`個の最近のツール結果は保持
3. それ以外のツール結果の内容を`'[Old tool result content cleared]'`に置換
4. キャッシュMC状態をリセット

### キャッシュマイクロコンパクト (L305-399)

`CACHED_MICROCOMPACT`機能（ant-only）有効時に使用。キャッシュ編集APIを使用してサーバー側でツール結果を削除する。

**レガシーとの違い**:
- ローカルメッセージ内容を**変更しない**（`cache_reference`と`cache_edits`はAPIレイヤーで追加）
- カウントベースのトリガー/キープ閾値をGrowthBook設定から取得
- ディスク永続化なし

### トークン推定 (estimateMessageTokens, L164-205)

メッセージ配列からトークン数を推定:

```typescript
function estimateMessageTokens(messages: Message[]): number {
  // 各メッセージの各ブロックについて:
  // - text → roughTokenCountEstimation(text)
  // - tool_result → calculateToolResultTokens(block)
  // - image/document → 2,000 tokens (固定)
  // - thinking → roughTokenCountEstimation(thinking)
  // - tool_use → roughTokenCountEstimation(name + JSON.stringify(input))
  
  // 4/3でパディング（保守的な推定）
  return Math.ceil(totalTokens * (4 / 3))
}
```

## コンパクトプロンプト (prompt.ts)

### プロンプト構造

コンパクトプロンプトは3部構成:

1. **NO_TOOLS_PREAMBLE**: ツール使用禁止の強い指示
2. **本文** (BASE/PARTIAL/PARTIAL_UP_TO): サマリー生成の詳細指示
3. **NO_TOOLS_TRAILER**: ツール使用禁止のリマインダー

### サマリーの9セクション

```
1. Primary Request and Intent    - ユーザーの明示的な要求と意図
2. Key Technical Concepts        - 重要な技術概念
3. Files and Code Sections       - ファイルとコードセクション
4. Errors and fixes              - エラーと修正
5. Problem Solving               - 問題解決
6. All user messages             - すべてのユーザーメッセージ
7. Pending Tasks                 - 未完了タスク
8. Current Work                  - 現在の作業
9. Optional Next Step            - 次のステップ（オプション）
```

### formatCompactSummary() (L311-335)

生のサマリーから`<analysis>`セクション（下書き用スクラッチパッド）を除去し、`<summary>`タグをヘッダーに変換:

```typescript
function formatCompactSummary(summary: string): string {
  // 1. <analysis>...</analysis> を除去
  // 2. <summary>...</summary> を "Summary:\n..." に変換
  // 3. 余分な空行を除去
}
```

### 部分コンパクト (Partial Compact)

2つの方向をサポート:

- **`from`** (`PARTIAL_COMPACT_PROMPT`): 最近のメッセージのみをサマリー化。以前のメッセージはそのまま保持。
- **`up_to`** (`PARTIAL_COMPACT_UP_TO_PROMPT`): サマリーは継続セッションの冒頭に配置される前提で作成。後続のメッセージとの接続を意識。

## トークン推定の仕組み (tokens.ts)

### tokenCountWithEstimation() (L226-261)

コンテキストサイズを測定するための**正準関数**。閾値チェック（オートコンパクト、セッションメモリ等）には常にこの関数を使用する。

```
tokenCountWithEstimation =
  最後のAPIレスポンスのトークン数 (input + output + cache)
  + それ以降に追加されたメッセージのラフ推定
```

**並列ツール呼び出しへの対応** (L219-225):
ストリーミングコードはコンテンツブロックごとに個別のAssistantMessageレコードを発行し、ToolResultがインターリーブされる:

```
[..., assistant(id=A), user(result), assistant(id=A), user(result), ...]
```

最後のアシスタントレコードだけを見ると、間のツール結果を見逃す。そのため、同じ`message.id`を持つ最初のレコードまで遡って推定範囲を拡張する。

### getTokenCountFromUsage() (L46-53)

APIレスポンスのusageデータからコンテキストトークン数を計算:

```typescript
function getTokenCountFromUsage(usage: Usage): number {
  return usage.input_tokens
    + (usage.cache_creation_input_tokens ?? 0)
    + (usage.cache_read_input_tokens ?? 0)
    + usage.output_tokens
}
```

### finalContextTokensFromLastResponse() (L79-112)

タスクバジェット計算用。`usage.iterations[-1]`（サーバーサイドツールループの最終イテレーション）を使用し、キャッシュトークンを除外。

## トークンバジェット (tokenBudget.ts)

ユーザーが「+500k」や「use 2M tokens」のような表記でトークンバジェットを指定できる機能。

### parseTokenBudget() (L21-29)

3つのパターンを認識:

1. **先頭ショートハンド**: `+500k` → 500,000
2. **末尾ショートハンド**: `...some text +500k` → 500,000
3. **冗長形式**: `use 2M tokens` → 2,000,000

サフィックスの乗数: `k` = 1,000、`m` = 1,000,000、`b` = 1,000,000,000

### getBudgetContinuationMessage() (L66-73)

バジェット継続時のナッジメッセージ:

```
Stopped at 42% of token target (210,000 / 500,000). Keep working — do not summarize.
```

## メッセージグルーピング (grouping.ts)

### groupMessagesByApiRound() (L22-63)

メッセージをAPIラウンド単位でグループ化する。`message.id`の変化を境界として使用:

```typescript
function groupMessagesByApiRound(messages: Message[]): Message[][] {
  // 新しいアシスタントレスポンス（異なるmessage.id）で境界を切る
  // 同一message.idのチャンクはインターリーブされたtool_resultと共に1グループ
}
```

**用途**:
- `truncateHeadForPTLRetry()`: コンパクトリクエストがprompt-too-longの際に、最古のラウンドから削除
- リアクティブコンパクト: 同様のグルーピングで部分的な会話を要約

## コンテキストウィンドウの設定 (context.ts)

### モデル別コンテキストウィンドウ (src/utils/context.ts)

```typescript
MODEL_CONTEXT_WINDOW_DEFAULT = 200_000  // デフォルト
```

1M対応の判定:
- `[1m]`サフィックス付きモデル → 1,000,000
- `claude-sonnet-4`、`opus-4-6`を含むモデル → `modelSupports1M()`で判定
- `CONTEXT_1M_BETA_HEADER`ベータ → 1,000,000
- `CLAUDE_CODE_MAX_CONTEXT_TOKENS`環境変数でオーバーライド可能

### 出力トークン上限 (L149-210)

モデル別のデフォルト/上限:

| モデル | デフォルト | 上限 |
|---|---|---|
| opus-4-6 | 64,000 | 128,000 |
| sonnet-4-6 | 32,000 | 128,000 |
| opus-4-5 / sonnet-4 / haiku-4 | 32,000 | 64,000 |
| その他 | 32,000 | 64,000 |

### スロット予約最適化

```typescript
CAPPED_DEFAULT_MAX_TOKENS = 8_000   // キャップされたデフォルト
ESCALATED_MAX_TOKENS = 64_000       // エスカレーション時の上限
```

BQデータでp99出力が4,911トークンであるため、8kのキャップで99%以上のリクエストをカバー。上限に達した場合は64kに自動エスカレーション（`query.ts`の`max_output_tokens_escalate`遷移）。
