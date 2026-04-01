# クエリ実行 (query.ts)

> ソースファイル: `src/query.ts` (約1,729行)

## 概要

`query.ts`はClaude Codeのエージェント推論ループの本体を実装する。モデルへのAPI呼び出し、ストリーミングレスポンスの処理、ツール実行のオーケストレーション、エラーリカバリ、コンテキスト管理を一体的に行う`while(true)`ループで構成される。

```
query()
  └── queryLoop() [while(true)]
        ├── 前処理
        │     ├── ツール結果バジェット適用
        │     ├── スニップコンパクト
        │     ├── マイクロコンパクト
        │     ├── コンテキストコラプス
        │     └── オートコンパクト
        ├── APIストリーミング呼び出し
        │     ├── ストリーミングツール実行
        │     └── フォールバック処理
        ├── 後処理
        │     ├── ストップフック
        │     ├── リアクティブコンパクト
        │     └── max_output_tokensリカバリ
        └── ツール実行 → 次のターンへ
```

## エントリポイント

### query() 関数 (L219-239)

公開APIであり、内部の`queryLoop()`を`yield*`で委譲する。完了時に消費したコマンドのUUIDに対して`notifyCommandLifecycle('completed')`を呼び出す。

### QueryParams 型 (L181-199)

```typescript
export type QueryParams = {
  messages: Message[]
  systemPrompt: SystemPrompt
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
  canUseTool: CanUseToolFn
  toolUseContext: ToolUseContext
  fallbackModel?: string
  querySource: QuerySource
  maxOutputTokensOverride?: number
  maxTurns?: number
  taskBudget?: { total: number }
}
```

## queryLoop() の内部構造 (L241-1729)

### ループ状態 (State型, L204-217)

各イテレーション間で引き継がれる可変状態:

| フィールド | 説明 |
|---|---|
| `messages` | 現在のメッセージ配列 |
| `toolUseContext` | ツール実行コンテキスト |
| `autoCompactTracking` | オートコンパクトの追跡状態 |
| `maxOutputTokensRecoveryCount` | max_output_tokensリカバリの試行回数 |
| `hasAttemptedReactiveCompact` | リアクティブコンパクトを試みたかどうか |
| `turnCount` | 現在のターン数 |
| `transition` | 前回のイテレーションが継続した理由 |

### 状態遷移 (transitions)

ループの`continue`は常に`state = { ..., transition: { reason: '...' } }`で理由を明示する:

- `next_turn`: ツール実行後の通常の次ターン
- `reactive_compact_retry`: リアクティブコンパクト後の再試行
- `collapse_drain_retry`: コンテキストコラプスのドレイン後の再試行
- `max_output_tokens_escalate`: 出力トークン上限のエスカレーション
- `max_output_tokens_recovery`: 出力トークンリカバリ
- `stop_hook_blocking`: ストップフックがブロッキングエラーを返した場合
- `token_budget_continuation`: トークンバジェット継続

## メッセージの前処理パイプライン

各イテレーションの冒頭で、APIに送信するメッセージに対して複数の前処理が適用される。

### 1. ツール結果バジェット (L377-394)

`applyToolResultBudget()`により、集約的なツール結果サイズにメッセージ単位の上限を適用する。マイクロコンパクトより先に実行され、`contentReplacementState`を通じて内容の置換を永続化できる。

### 2. スニップコンパクト (L401-409)

`HISTORY_SNIP`機能が有効な場合、`snipCompactIfNeeded()`で古いメッセージをスニップし、解放されたトークン数(`snipTokensFreed`)をオートコンパクトの閾値判定に反映する。

### 3. マイクロコンパクト (L414-426)

`microcompact()`により、古いツール結果の内容をクリアする軽量なコンパクション。2つのパスがある:

- **時間ベースマイクロコンパクト**: 最後のアシスタントメッセージからの経過時間が閾値を超えた場合にトリガー
- **キャッシュマイクロコンパクト** (`CACHED_MICROCOMPACT`): キャッシュ編集APIを使い、ローカルメッセージ内容を変更せずにサーバー側でツール結果を削除

### 4. コンテキストコラプス (L440-447)

`CONTEXT_COLLAPSE`機能が有効な場合、`applyCollapsesIfNeeded()`でメッセージのコラプス（折りたたみ）を適用。オートコンパクトより先に実行され、コラプスで十分にトークンが削減できればオートコンパクトはno-opとなる。

### 5. オートコンパクト (L454-543)

`autocompact()`により、トークン使用量が閾値を超えた場合に会話を要約して圧縮する。成功時にはコンパクト境界メッセージとポストコンパクトメッセージを`yield`する。

## StreamingToolExecutor (L560-568)

`StreamingToolExecutor`は、モデルのストリーミングレスポンスと並行してツールを実行する最適化機能である。

```typescript
let streamingToolExecutor = useStreamingToolExecution
  ? new StreamingToolExecutor(
      toolUseContext.options.tools,
      canUseTool,
      toolUseContext,
    )
  : null
```

### 動作原理

1. ストリーミング中に`tool_use`ブロックを検出するたびに`addTool()`で登録 (L841-843)
2. `getCompletedResults()`で完了した結果を随時取得して`yield` (L851-862)
3. ストリーミング完了後、`getRemainingResults()`で残りの結果を取得 (L1380-1382)

ストリーミング中のフォールバック発生時には`discard()`で状態をリセットし、新しいインスタンスを作成する (L733-740)。

## APIストリーミングとフォールバック

### モデル呼び出し (L659-708)

`deps.callModel()`でAPIストリーミングを開始。主要なオプション:

- `model`: 現在のモデル（`getRuntimeMainLoopModel()`で決定）
- `fastMode`: 高速モード設定
- `fallbackModel`: フォールバックモデル
- `taskBudget`: タスクバジェット（remaining含む）
- `maxOutputTokensOverride`: 出力トークン上限オーバーライド

### ストリーミングフォールバック (L709-741)

ストリーミング中にフォールバックが発生した場合:

1. 既存のアシスタントメッセージにトゥームストーンを発行 (L716-718)
2. `assistantMessages`、`toolResults`、`toolUseBlocks`をクリア
3. `StreamingToolExecutor`を再作成
4. ストリーミングを新しいモデルで継続

### FallbackTriggeredError (L893-953)

`catch`ブロックで`FallbackTriggeredError`を捕捉し:

1. モデルを`fallbackModel`に切り替え
2. 思考ブロックの署名を除去（`stripSignatureBlocks`）
3. `attemptWithFallback = true`でリトライ

## エラーリカバリ機構

### prompt-too-long リカバリ (L1062-1183)

3段階の回復戦略:

1. **コンテキストコラプスドレイン**: ステージされたコラプスをすべてコミット (L1089-1117)
2. **リアクティブコンパクト**: `tryReactiveCompact()`で即座にコンパクション (L1119-1166)
3. **エラー表面化**: 回復不能な場合、保留していたエラーメッセージを`yield`

### max_output_tokens リカバリ (L1188-1256)

`MAX_OUTPUT_TOKENS_RECOVERY_LIMIT` (=3) 回まで自動リカバリ:

1. **エスカレーション** (L1195-1221): デフォルト8kから64kへの自動引き上げ（1回限り）
2. **マルチターンリカバリ** (L1223-1252): メタメッセージ「Output token limit hit. Resume directly...」を注入して継続

### メディアサイズエラーリカバリ

画像/PDFサイズエラーもリアクティブコンパクトで回復を試みる (L1082-1084)。コラプスドレインはスキップ（コラプスは画像を除去しない）。

## ツール実行のオーケストレーション

### ツール結果の収集 (L1360-1408)

`StreamingToolExecutor`または`runTools()`からツール結果を収集:

```typescript
const toolUpdates = streamingToolExecutor
  ? streamingToolExecutor.getRemainingResults()
  : runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)
```

### ツール使用サマリー生成 (L1411-1482)

ツールバッチ完了後、`generateToolUseSummary()`で非同期にサマリーを生成。次のイテレーションの冒頭で`await`される（ストリーミング中に並行解決）。

### フック停止チェック (L1518-1521)

ツール結果に`hook_stopped_continuation`アタッチメントが含まれる場合、`shouldPreventContinuation = true`でループを終了。

## 自動コンパクトメカニズム

オートコンパクトの詳細は`src/services/compact/autoCompact.ts`で定義されるが、`query.ts`での統合ポイントは以下の通り:

### コンパクト結果の処理 (L470-543)

コンパクションが成功した場合:

1. タスクバジェットの残量を計算 (`taskBudgetRemaining`) (L508-515)
2. トラッキング状態をリセット (L521-526)
3. `buildPostCompactMessages()`でポストコンパクトメッセージを構築
4. ポストコンパクトメッセージを`yield`
5. `messagesForQuery`をポストコンパクトメッセージで置換

### ブロッキングリミット (L628-648)

オートコンパクトがオフの場合、`calculateTokenWarningState()`でブロッキングリミットをチェック。超過時は`PROMPT_TOO_LONG_ERROR_MESSAGE`の合成アシスタントメッセージを発行して終了。

## MCPリソースとツール検索

### ツールのリフレッシュ (L1660-1671)

ターン間で`refreshTools()`により、新規接続されたMCPサーバーのツールを反映:

```typescript
if (updatedToolUseContext.options.refreshTools) {
  const refreshedTools = updatedToolUseContext.options.refreshTools()
  if (refreshedTools !== updatedToolUseContext.options.tools) {
    updatedToolUseContext = { ...updatedToolUseContext, options: { ...updatedToolUseContext.options, tools: refreshedTools } }
  }
}
```

### アタッチメント収集 (L1580-1590)

ツール実行後、`getAttachmentMessages()`で以下を収集:

- ファイル変更アタッチメント
- キューされたコマンド
- メモリプリフェッチ結果 (L1599-1614)
- スキルディスカバリプリフェッチ結果 (L1620-1628)

### キューコマンドの消費 (L1548-1578)

`getCommandsByMaxPriority()`でキューからコマンドを取得。スラッシュコマンドは除外され、メインスレッドとサブエージェントでフィルタリングされる。

## 最大ターン数とタスクバジェット

### 最大ターン数チェック (L1704-1712)

```typescript
if (maxTurns && nextTurnCount > maxTurns) {
  yield createAttachmentMessage({ type: 'max_turns_reached', maxTurns, turnCount: nextTurnCount })
  return { reason: 'max_turns', turnCount: nextTurnCount }
}
```

### トークンバジェット継続 (L1308-1355)

`TOKEN_BUDGET`機能が有効な場合、`checkTokenBudget()`で出力トークンの消費率を評価:

- `action === 'continue'`: ナッジメッセージを注入して継続
- `action === 'stop'`: 完了イベントをログして終了
- `diminishingReturns`: 収穫逓減を検出した場合の早期停止

## ウィズホールド（保留）パターン

回復可能なエラー（prompt-too-long、max_output_tokens、メディアサイズエラー）は、ストリーミング中に検出されても即座に`yield`されず、「保留」される (L799-825):

```typescript
let withheld = false
if (reactiveCompact?.isWithheldPromptTooLong(message)) withheld = true
if (mediaRecoveryEnabled && reactiveCompact?.isWithheldMediaSizeError(message)) withheld = true
if (isWithheldMaxOutputTokens(message)) withheld = true
if (!withheld) yield yieldMessage
```

保留されたメッセージは、後続のリカバリが成功すればSDKクライアントに表示されず、リカバリが失敗した場合のみ表面化される。
