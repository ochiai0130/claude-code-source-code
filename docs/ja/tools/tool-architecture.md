# ツールアーキテクチャ - Claude Code v2.1.88

## 概要

Claude Codeのツールシステムは、`src/Tool.ts`（約792行）に定義された型システムを基盤とし、40以上のツールを統一的なインターフェースで管理する。本ドキュメントでは、ツールの定義・登録・実行・権限管理のアーキテクチャを詳細に解析する。

---

## 1. ToolInputJSONSchema型

```typescript
// src/Tool.ts:15-21
export type ToolInputJSONSchema = {
  [x: string]: unknown
  type: 'object'
  properties?: {
    [x: string]: unknown
  }
}
```

MCP（Model Context Protocol）ツールがZodスキーマではなくJSON Schema形式で直接入力スキーマを指定するために使用される。`type: 'object'`が必須であり、ツールの入力は常にオブジェクト形式である。

---

## 2. Tool型インターフェースの詳細分析

`Tool`型は`src/Tool.ts:362-695`に定義されており、ジェネリクスで入力スキーマ型`Input`、出力型`Output`、プログレス型`P`をパラメータ化している。

### 2.1 基本プロパティ

| プロパティ | 型 | 説明 |
|---|---|---|
| `name` | `string` (readonly) | ツールの一意識別名 |
| `aliases` | `string[]` (任意) | ツール名変更時の後方互換エイリアス |
| `searchHint` | `string` (任意) | ToolSearchでのキーワードマッチング用フレーズ（3-10語） |
| `inputSchema` | `Input` (readonly) | Zodスキーマによる入力バリデーション定義 |
| `inputJSONSchema` | `ToolInputJSONSchema` (任意) | MCPツール用JSON Schemaフォーマット |
| `outputSchema` | `z.ZodType` (任意) | 出力スキーマ定義 |
| `maxResultSizeChars` | `number` | ツール結果の最大文字数。超過時はディスクに永続化 |
| `strict` | `boolean` (任意) | API側でツール指示・パラメータスキーマを厳格に遵守させるモード |

### 2.2 コアメソッド

#### `call()` - ツール実行

```typescript
// src/Tool.ts:379-385
call(
  args: z.infer<Input>,
  context: ToolUseContext,
  canUseTool: CanUseToolFn,
  parentMessage: AssistantMessage,
  onProgress?: ToolCallProgress<P>,
): Promise<ToolResult<Output>>
```

ツールの主要実行メソッド。5つの引数を受け取る:
- `args`: バリデーション済みの入力パラメータ
- `context`: ツール実行コンテキスト（後述）
- `canUseTool`: ネストされたツール呼び出し時の権限チェック関数
- `parentMessage`: このツール呼び出しを含むアシスタントメッセージ
- `onProgress`: プログレス通知コールバック（任意）

#### `description()` - 動的説明文生成

```typescript
// src/Tool.ts:386-393
description(
  input: z.infer<Input>,
  options: {
    isNonInteractiveSession: boolean
    toolPermissionContext: ToolPermissionContext
    tools: Tools
  },
): Promise<string>
```

入力パラメータと実行オプションに基づいて動的にツール説明文を生成する。非対話型セッションや権限コンテキストによって説明内容が変わる場合がある。

#### `prompt()` - システムプロンプト生成

```typescript
// src/Tool.ts:518-523
prompt(options: {
  getToolPermissionContext: () => Promise<ToolPermissionContext>
  tools: Tools
  agents: AgentDefinition[]
  allowedAgentTypes?: string[]
}): Promise<string>
```

モデルに送信するシステムプロンプト内のツール使用ガイドラインを生成する。

### 2.3 権限・バリデーション系メソッド

| メソッド | 戻り値 | 説明 |
|---|---|---|
| `validateInput()` | `Promise<ValidationResult>` | 入力の妥当性検証。モデルにエラーを通知（UI表示はしない） |
| `checkPermissions()` | `Promise<PermissionResult>` | ツール固有の権限チェック。`validateInput()`通過後に呼ばれる |
| `preparePermissionMatcher()` | `Promise<(pattern: string) => boolean>` | フック`if`条件のパターンマッチャーを準備 |
| `isEnabled()` | `boolean` | ツールが有効かどうか |
| `isReadOnly()` | `boolean` | 読み取り専用操作かどうか |
| `isDestructive()` | `boolean` | 不可逆操作（削除・上書き・送信）かどうか |
| `isConcurrencySafe()` | `boolean` | 並行実行が安全かどうか |

### 2.4 UI・レンダリング系メソッド

| メソッド | 説明 |
|---|---|
| `renderToolUseMessage()` | ツール呼び出しメッセージのUI表示（部分入力対応） |
| `renderToolResultMessage()` | ツール結果メッセージのUI表示 |
| `renderToolUseProgressMessage()` | 実行中のプログレスUI表示 |
| `renderToolUseRejectedMessage()` | 権限拒否時のUI表示 |
| `renderToolUseErrorMessage()` | エラー時のUI表示 |
| `renderGroupedToolUse()` | 並列ツール呼び出しのグループ表示 |
| `renderToolUseTag()` | ツール呼び出し後のメタデータタグ表示 |
| `renderToolUseQueuedMessage()` | キュー待ち状態の表示 |
| `userFacingName()` | ユーザー向け表示名 |
| `getToolUseSummary()` | コンパクトビュー用の短い要約文字列 |
| `getActivityDescription()` | スピナー表示用のアクティビティ説明 |
| `extractSearchText()` | トランスクリプト検索用のテキスト抽出 |
| `isResultTruncated()` | 非verboseモードで結果が切り詰められているか |

### 2.5 分類・セキュリティ系メソッド

| メソッド | 説明 |
|---|---|
| `isSearchOrReadCommand()` | UIの折り畳み表示判定（検索/読み取り/一覧） |
| `isOpenWorld()` | オープンワールド操作かどうか |
| `toAutoClassifierInput()` | セキュリティ分類器向けの入力コンパクト表現 |
| `interruptBehavior()` | ユーザー中断時の挙動（`'cancel'`または`'block'`） |
| `requiresUserInteraction()` | ユーザー対話が必要かどうか |

### 2.6 特殊プロパティ

| プロパティ | 説明 |
|---|---|
| `isMcp` | MCPツールかどうか |
| `isLsp` | LSPツールかどうか |
| `shouldDefer` | 遅延読み込みツール（ToolSearch経由で利用） |
| `alwaysLoad` | 遅延を無効化し常にスキーマを初期プロンプトに含める |
| `mcpInfo` | MCPツールのサーバー名・ツール名情報 |
| `backfillObservableInput()` | オブザーバー向け入力にレガシー/派生フィールドを追加 |
| `getPath()` | ファイルパスを操作するツールのパス取得 |
| `inputsEquivalent()` | 入力の等価性判定 |

---

## 3. ToolUseContext - 実行コンテキスト

`ToolUseContext`（`src/Tool.ts:158-300`）はツール実行時に渡されるコンテキスト情報の集合体である。

### 3.1 主要フィールド

#### options（設定情報）

```typescript
// src/Tool.ts:159-179
options: {
  commands: Command[]              // 利用可能なコマンド一覧
  debug: boolean                   // デバッグモード
  mainLoopModel: string            // 使用モデル名
  tools: Tools                     // 利用可能なツール一覧
  verbose: boolean                 // 冗長モード
  thinkingConfig: ThinkingConfig   // 思考設定
  mcpClients: MCPServerConnection[] // MCP接続一覧
  mcpResources: Record<string, ServerResource[]> // MCPリソース
  isNonInteractiveSession: boolean // 非対話型セッション
  agentDefinitions: AgentDefinitionsResult // エージェント定義
  maxBudgetUsd?: number            // 最大予算（USD）
  customSystemPrompt?: string      // カスタムシステムプロンプト
  appendSystemPrompt?: string      // 追加システムプロンプト
  refreshTools?: () => Tools       // ツール一覧の動的更新
}
```

#### 状態管理

| フィールド | 説明 |
|---|---|
| `abortController` | 中断制御用AbortController |
| `readFileState` | ファイル読み取り状態キャッシュ（LRU） |
| `getAppState()` / `setAppState()` | アプリケーション状態のget/set |
| `setAppStateForTasks()` | タスク用の常時共有setAppState（サブエージェントでも有効） |
| `messages` | 会話メッセージ履歴 |
| `fileReadingLimits` | ファイル読み取り制限（トークン数・バイト数） |
| `globLimits` | Glob結果の最大数制限 |

#### UI・通知

| フィールド | 説明 |
|---|---|
| `setToolJSX` | ツールカスタムJSX表示設定 |
| `addNotification` | 通知追加 |
| `appendSystemMessage` | UI専用システムメッセージ追加 |
| `sendOSNotification` | OS通知送信（iTerm2、Kitty等） |
| `setStreamMode` | スピナーモード設定 |
| `setSDKStatus` | SDK状態設定 |

#### エージェント・タスク関連

| フィールド | 説明 |
|---|---|
| `agentId` | サブエージェントID |
| `agentType` | サブエージェントのタイプ名 |
| `queryTracking` | クエリチェーン追跡（chainId、depth） |
| `toolUseId` | 現在のツール使用ID |
| `toolDecisions` | ツール決定のキャッシュ |
| `localDenialTracking` | 非同期サブエージェント用の拒否追跡状態 |

#### メモリ・スキル管理

| フィールド | 説明 |
|---|---|
| `nestedMemoryAttachmentTriggers` | ネストメモリアタッチメントのトリガー |
| `loadedNestedMemoryPaths` | 読み込み済みCLAUDE.mdパスの重複排除 |
| `dynamicSkillDirTriggers` | 動的スキルディレクトリトリガー |
| `discoveredSkillNames` | セッション中に発見されたスキル名（テレメトリ用） |

#### パフォーマンス・最適化

| フィールド | 説明 |
|---|---|
| `requireCanUseTool` | canUseToolを常に呼び出すフラグ（投機実行時のパス書き換え） |
| `contentReplacementState` | ツール結果のコンテンツ置換状態（バジェット管理） |
| `renderedSystemPrompt` | 親のレンダリング済みシステムプロンプト（キャッシュ共有用） |

---

## 4. ToolPermissionContext - 権限コンテキスト

```typescript
// src/Tool.ts:123-138
export type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  isBypassPermissionsModeAvailable: boolean
  isAutoModeAvailable?: boolean
  strippedDangerousRules?: ToolPermissionRulesBySource
  shouldAvoidPermissionPrompts?: boolean
  awaitAutomatedChecksBeforeDialog?: boolean
  prePlanMode?: PermissionMode
}>
```

`DeepImmutable`で保護された不変オブジェクトとして定義されている。

### 主要フィールド

| フィールド | 説明 |
|---|---|
| `mode` | 権限モード（`'default'`等） |
| `alwaysAllowRules` | 常に許可するルール（ソース別） |
| `alwaysDenyRules` | 常に拒否するルール（ソース別） |
| `alwaysAskRules` | 常に確認するルール（ソース別） |
| `shouldAvoidPermissionPrompts` | 権限プロンプトを自動拒否（バックグラウンドエージェント用） |
| `awaitAutomatedChecksBeforeDialog` | ダイアログ表示前に自動チェックを待つ |
| `prePlanMode` | プランモード進入前の権限モード保存 |

---

## 5. ツール登録の仕組み - buildTool

### 5.1 ToolDefとデフォルト値

`ToolDef`型（`src/Tool.ts:721-726`）は`Tool`型の部分的な定義であり、以下のメソッドを省略可能にしている:

```typescript
// src/Tool.ts:707-715
type DefaultableToolKeys =
  | 'isEnabled'
  | 'isConcurrencySafe'
  | 'isReadOnly'
  | 'isDestructive'
  | 'checkPermissions'
  | 'toAutoClassifierInput'
  | 'userFacingName'
```

### 5.2 デフォルト値（フェイルクローズ設計）

```typescript
// src/Tool.ts:757-769
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?) => false,    // 安全でないと仮定
  isReadOnly: (_input?) => false,            // 書き込みありと仮定
  isDestructive: (_input?) => false,
  checkPermissions: (input, _ctx?) =>        // 汎用権限システムに委譲
    Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: (_input?) => '',    // 分類器をスキップ
  userFacingName: (_input?) => '',
}
```

セキュリティ上重要な`isConcurrencySafe`と`isReadOnly`は**フェイルクローズ**（fail-closed）設計となっている。デフォルトでは並行実行は安全でなく、読み取り専用でないと仮定する。

### 5.3 buildTool関数

```typescript
// src/Tool.ts:783-792
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

`BuiltTool<D>`型（`src/Tool.ts:735-741`）はTypeScriptの型レベルスプレッドを実現し、ツール定義がデフォルトを上書きする場合はその型を使い、省略する場合はデフォルトの型を使う。全60以上のツールがこの関数を通じて登録される。

---

## 6. ToolResult - ツール実行結果

```typescript
// src/Tool.ts:321-336
export type ToolResult<T> = {
  data: T
  newMessages?: (UserMessage | AssistantMessage | AttachmentMessage | SystemMessage)[]
  contextModifier?: (context: ToolUseContext) => ToolUseContext
  mcpMeta?: {
    _meta?: Record<string, unknown>
    structuredContent?: Record<string, unknown>
  }
}
```

| フィールド | 説明 |
|---|---|
| `data` | ツール固有の出力データ |
| `newMessages` | 会話に追加するメッセージ（任意） |
| `contextModifier` | コンテキスト変更関数（非並行安全ツールのみ有効） |
| `mcpMeta` | MCPプロトコルメタデータ（SDK消費者向けパススルー） |

---

## 7. ツールルーティングと実行フロー

### 7.1 ツール検索

```typescript
// src/Tool.ts:348-360
export function toolMatchesName(
  tool: { name: string; aliases?: string[] },
  name: string,
): boolean {
  return tool.name === name || (tool.aliases?.includes(name) ?? false)
}

export function findToolByName(tools: Tools, name: string): Tool | undefined {
  return tools.find(t => toolMatchesName(t, name))
}
```

ツール名またはエイリアスでツールを検索する。`Tools`型は`readonly Tool[]`として定義されており（`src/Tool.ts:701`）、不変のコレクションとして扱われる。

### 7.2 実行フロー概要

1. モデルが`tool_use`ブロックを生成
2. `findToolByName()`でツールを特定
3. `inputSchema`による入力バリデーション
4. `validateInput()`による追加バリデーション
5. `checkPermissions()`による権限チェック
6. `call()`によるツール実行
7. `mapToolResultToToolResultBlockParam()`で結果をAPI形式に変換

### 7.3 遅延ロード（Deferred Loading）

`shouldDefer`フラグが`true`のツールは、初期プロンプトには軽量な定義のみ含まれる。モデルは`ToolSearch`ツールを使って遅延ツールを検索・ロードしてから使用する。`alwaysLoad`フラグで遅延を無効化できる。

---

## 8. プログレス通知

```typescript
// src/Tool.ts:306-319
export type Progress = ToolProgressData | HookProgress

export type ToolProgress<P extends ToolProgressData> = {
  toolUseID: string
  data: P
}

export function filterToolProgressMessages(
  progressMessagesForMessage: ProgressMessage[],
): ProgressMessage<ToolProgressData>[] {
  return progressMessagesForMessage.filter(
    (msg): msg is ProgressMessage<ToolProgressData> =>
      msg.data?.type !== 'hook_progress',
  )
}
```

プログレスデータ型は`src/types/tools.ts`に集約定義されている:
- `BashProgress`: Bash実行のプログレス
- `AgentToolProgress`: エージェントツールのプログレス
- `MCPProgress`: MCPツールのプログレス
- `REPLToolProgress`: REPLツールのプログレス
- `WebSearchProgress`: Web検索のプログレス
- `SkillToolProgress`: スキルツールのプログレス
- `TaskOutputProgress`: タスク出力のプログレス

---

## 9. 権限チェックとツールフィルタリング

### 9.1 権限チェックの流れ

1. **validateInput()**: 入力の構造的・論理的妥当性を検証
2. **checkPermissions()**: ツール固有の権限ロジックを実行
3. **汎用権限システム**: `ToolPermissionContext`のルール評価
4. **フック**: PreToolUse/PostToolUseフックによる外部制御

### 9.2 権限結果（PermissionResult）

権限チェックの結果は以下のbehaviorを持つ:
- `'allow'`: 実行を許可（`updatedInput`で入力を変更可能）
- `'ask'`: ユーザーに確認を求める
- `'deny'`: 実行を拒否
- `'passthrough'`: このチェックでは判断せず次に委譲

### 9.3 ValidationResult

```typescript
// src/Tool.ts:95-101
export type ValidationResult =
  | { result: true }
  | {
      result: false
      message: string
      errorCode: number
    }
```

バリデーション失敗時はモデルにエラーメッセージとコードを返し、UIには直接表示しない。

---

## 10. SetToolJSXFn - カスタムUI表示

```typescript
// src/Tool.ts:103-114
export type SetToolJSXFn = (
  args: {
    jsx: React.ReactNode | null
    shouldHidePromptInput: boolean
    shouldContinueAnimation?: true
    showSpinner?: boolean
    isLocalJSXCommand?: boolean
    isImmediate?: boolean
    clearLocalJSX?: boolean
  } | null,
) => void
```

ツールがカスタムReact JSXをUI層に挿入するためのコールバック。プロンプト入力の非表示化、アニメーション制御、ローカルJSXコマンドの管理が可能。

---

## まとめ

Claude Codeのツールアーキテクチャは以下の設計原則に基づいている:

1. **型安全性**: Zodスキーマとジェネリクスによる厳密な型定義
2. **フェイルクローズ**: セキュリティ関連のデフォルト値は安全側に倒す
3. **関心の分離**: 入力バリデーション・権限チェック・実行・UI表示が明確に分離
4. **拡張性**: `buildTool()`によるデフォルト埋め込みで新規ツール追加が容易
5. **不変性**: `DeepImmutable`や`readonly`による状態の保護
6. **遅延ロード**: `shouldDefer`/`alwaysLoad`による初期プロンプトサイズの最適化
