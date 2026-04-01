# 権限システム

Claude Code v2.1.88 の権限システムは、ツール実行の安全性を多層的に管理するセキュリティ基盤である。

## 概要

権限システムは `src/utils/permissions/` ディレクトリに実装され、以下の要素で構成される:

- **Permission Modes** - セッション全体の権限レベル
- **Permission Rules** - ツール単位の allow/deny/ask ルール
- **Permission Classifiers** - AI ベースの自動権限判定
- **Denial Tracking** - 拒否の追跡とフォールバック
- **Permission Setup** - 権限コンテキストの初期化

---

## Permission Modes

### 定義

**ソース**: `src/types/permissions.ts:16-38`, `src/utils/permissions/PermissionMode.ts`

```typescript
// 外部公開モード
const EXTERNAL_PERMISSION_MODES = [
  'acceptEdits',       // 編集を自動承認
  'bypassPermissions', // 全権限をバイパス
  'default',           // デフォルト (手動承認)
  'dontAsk',           // 確認なし
  'plan',              // プランモード (読み取り専用)
] as const

// 内部モード (上記 + auto)
type InternalPermissionMode = ExternalPermissionMode | 'auto' | 'bubble'
```

`auto` モードは `TRANSCRIPT_CLASSIFIER` フィーチャーゲートで保護されており、内部ビルドでのみ有効:

```typescript
// src/types/permissions.ts:33-36
const INTERNAL_PERMISSION_MODES = [
  ...EXTERNAL_PERMISSION_MODES,
  ...(feature('TRANSCRIPT_CLASSIFIER') ? (['auto'] as const) : ([] as const)),
] as const
```

### モード設定

各モードにはタイトル、アイコン、カラーが紐付けられる:

| モード | タイトル | シンボル | カラー |
|---|---|---|---|
| `default` | Default | (なし) | text |
| `plan` | Plan Mode | 一時停止アイコン | planMode |
| `acceptEdits` | Accept edits | `>>` | autoAccept |
| `bypassPermissions` | Bypass Permissions | `>>` | error |
| `dontAsk` | Don't Ask | `>>` | error |
| `auto` | Auto mode | `>>` | warning |

---

## Permission Rules

### ルール構造

**ソース**: `src/types/permissions.ts:50-79`

```typescript
type PermissionRuleValue = {
  toolName: string        // ツール名 (例: "Bash", "Edit")
  ruleContent?: string    // オプション: ルールの内容 (例: "prefix:npm *")
}

type PermissionRule = {
  source: PermissionRuleSource   // ルールの出所
  ruleBehavior: PermissionBehavior  // 'allow' | 'deny' | 'ask'
  ruleValue: PermissionRuleValue
}
```

### ルールソース

```typescript
// src/types/permissions.ts:54-63
type PermissionRuleSource =
  | 'userSettings'      // ~/.claude/settings.json
  | 'projectSettings'   // .claude/settings.json
  | 'localSettings'     // .claude/settings.local.json
  | 'flagSettings'      // GrowthBook フラグ設定
  | 'policySettings'    // 組織ポリシー (マネージド)
  | 'cliArg'            // CLI 引数
  | 'command'           // コマンド経由
  | 'session'           // セッション中の一時設定
```

### ルールのロード

**ソース**: `src/utils/permissions/permissionsLoader.ts:120-133`

```typescript
function loadAllPermissionRulesFromDisk(): PermissionRule[] {
  // allowManagedPermissionRulesOnly が有効な場合、ポリシー設定のみ使用
  if (shouldAllowManagedPermissionRulesOnly()) {
    return getPermissionRulesForSource('policySettings')
  }
  // 通常は全ソースからロード
  const rules: PermissionRule[] = []
  for (const source of getEnabledSettingSources()) {
    rules.push(...getPermissionRulesForSource(source))
  }
  return rules
}
```

`allowManagedPermissionRulesOnly` が `policySettings` で有効な場合、ユーザー・プロジェクトレベルのルールは完全に無視される。これは企業環境でのセキュリティポリシー強制に使用される。

### ルールのマッチング

**ソース**: `src/utils/permissions/permissions.ts:238-269`

ルールマッチングは以下の優先順位で実行:

1. **ツール全体のマッチ**: ルールにコンテンツがなく、ツール名が一致
2. **MCP サーバーレベル**: `mcp__server1` が `mcp__server1__tool1` にマッチ
3. **コンテンツベースのマッチ**: `Bash(prefix:npm *)` のようなツール固有ルール

```typescript
function toolMatchesRule(tool, rule): boolean {
  if (rule.ruleValue.ruleContent !== undefined) return false
  const nameForRuleMatch = getToolNameForPermissionCheck(tool)
  if (rule.ruleValue.toolName === nameForRuleMatch) return true
  // MCP サーバーレベルのマッチ
  // ...
}
```

---

## 権限リクエストと承認フロー

### 権限判定の結果型

**ソース**: `src/types/permissions.ts:174-266`

```typescript
type PermissionDecision<Input> =
  | PermissionAllowDecision<Input>   // 許可
  | PermissionAskDecision<Input>     // ユーザーに確認
  | PermissionDenyDecision           // 拒否

// 追加: passthrough オプション
type PermissionResult<Input> =
  | PermissionDecision<Input>
  | { behavior: 'passthrough'; message: string; ... }
```

### 判定理由 (PermissionDecisionReason)

**ソース**: `src/types/permissions.ts:271-324`

判定理由は以下のタイプに分類される:

| タイプ | 説明 |
|---|---|
| `rule` | 権限ルールによる判定 |
| `mode` | 権限モードによる判定 |
| `subcommandResults` | 複合コマンドのサブコマンド結果 |
| `permissionPromptTool` | 権限プロンプトツール |
| `hook` | フックによる判定 |
| `asyncAgent` | 非同期エージェント |
| `sandboxOverride` | サンドボックスオーバーライド |
| `classifier` | 分類器による判定 |
| `workingDir` | 作業ディレクトリ制限 |
| `safetyCheck` | 安全性チェック |
| `other` | その他 |

### PermissionAskDecision の拡張フィールド

```typescript
type PermissionAskDecision<Input> = {
  behavior: 'ask'
  message: string
  suggestions?: PermissionUpdate[]           // 推奨される権限更新
  blockedPath?: string                       // ブロックされたパス
  pendingClassifierCheck?: PendingClassifierCheck  // 非同期分類器チェック
  isBashSecurityCheckForMisparsing?: boolean // パース不正によるセキュリティチェック
  contentBlocks?: ContentBlockParam[]        // 画像フィードバック用
}
```

---

## ツールレベルの権限チェック

### チェック関数群

**ソース**: `src/utils/permissions/permissions.ts:275-302`

```typescript
// ツール全体が常に許可されているか
function toolAlwaysAllowedRule(context, tool): PermissionRule | null

// ツール全体が拒否されているか
function getDenyRuleForTool(context, tool): PermissionRule | null

// ツール全体が確認要求されているか
function getAskRuleForTool(context, tool): PermissionRule | null

// エージェントタイプの拒否チェック
function getDenyRuleForAgent(context, agentToolName, agentType): PermissionRule | null
```

### コンテンツベースのルール検索

```typescript
// src/utils/permissions/permissions.ts:349-390
function getRuleByContentsForTool(
  context: ToolPermissionContext,
  tool: Tool,
  behavior: PermissionBehavior,
): Map<string, PermissionRule>
```

例: `Bash(prefix:npm *)` ルールの場合、キーは `prefix:npm *`、値はルールオブジェクト。

### ToolPermissionContext

**ソース**: `src/types/permissions.ts:427-441`

```typescript
type ToolPermissionContext = {
  readonly mode: PermissionMode
  readonly additionalWorkingDirectories: ReadonlyMap<string, AdditionalWorkingDirectory>
  readonly alwaysAllowRules: ToolPermissionRulesBySource
  readonly alwaysDenyRules: ToolPermissionRulesBySource
  readonly alwaysAskRules: ToolPermissionRulesBySource
  readonly isBypassPermissionsModeAvailable: boolean
  readonly strippedDangerousRules?: ToolPermissionRulesBySource
  readonly shouldAvoidPermissionPrompts?: boolean
  readonly awaitAutomatedChecksBeforeDialog?: boolean
  readonly prePlanMode?: PermissionMode
}
```

---

## 自動モードと分類器

### Auto Mode State

**ソース**: `src/utils/permissions/autoModeState.ts`

自動モードの状態はモジュールレベルの変数で管理:

```typescript
let autoModeActive = false         // 自動モードがアクティブか
let autoModeFlagCli = false        // CLI フラグで指定されたか
let autoModeCircuitBroken = false  // サーキットブレーカーが発動したか
```

サーキットブレーカーは GrowthBook の `tengu_auto_mode_config.enabled === 'disabled'` を検出した場合に発動し、自動モードへの再入を防止する。

### Bash 分類器

`BASH_CLASSIFIER` フィーチャーゲートで保護されるコマンド安全性分類器:

**ソース**: `src/utils/permissions/yoloClassifier.ts`

```typescript
type YoloClassifierResult = {
  thinking?: string
  shouldBlock: boolean
  reason: string
  unavailable?: boolean
  transcriptTooLong?: boolean
  model: string
  usage?: ClassifierUsage
  durationMs?: number
  stage?: 'fast' | 'thinking'    // 2段階 XML 分類器
}
```

### トランスクリプト分類器

`TRANSCRIPT_CLASSIFIER` フィーチャーゲートで保護される自動モード用分類器:

```typescript
// src/utils/permissions/permissions.ts:59-64
const classifierDecisionModule = feature('TRANSCRIPT_CLASSIFIER')
  ? require('./classifierDecision.js')
  : null
const autoModeStateModule = feature('TRANSCRIPT_CLASSIFIER')
  ? require('./autoModeState.js')
  : null
```

---

## 自動モード拒否の追跡

### Denial Tracking

**ソース**: `src/utils/permissions/denialTracking.ts`

連続拒否と総拒否数を追跡し、しきい値を超えた場合にユーザープロンプトへフォールバック:

```typescript
type DenialTrackingState = {
  consecutiveDenials: number  // 連続拒否数
  totalDenials: number        // 総拒否数
}

const DENIAL_LIMITS = {
  maxConsecutive: 3,   // 連続 3 回
  maxTotal: 20,        // 合計 20 回
}
```

```typescript
function shouldFallbackToPrompting(state: DenialTrackingState): boolean {
  return (
    state.consecutiveDenials >= DENIAL_LIMITS.maxConsecutive ||
    state.totalDenials >= DENIAL_LIMITS.maxTotal
  )
}
```

成功時は `consecutiveDenials` がリセットされるが、`totalDenials` は累積される。

---

## 権限更新の永続化

### PermissionUpdate 型

**ソース**: `src/types/permissions.ts:98-131`

```typescript
type PermissionUpdate =
  | { type: 'addRules'; destination: PermissionUpdateDestination; rules: PermissionRuleValue[]; behavior: PermissionBehavior }
  | { type: 'replaceRules'; ... }
  | { type: 'removeRules'; ... }
  | { type: 'setMode'; destination: PermissionUpdateDestination; mode: ExternalPermissionMode }
  | { type: 'addDirectories'; ... }
  | { type: 'removeDirectories'; ... }
```

### 永続化先

```typescript
type PermissionUpdateDestination =
  | 'userSettings'     // ユーザー設定ファイル
  | 'projectSettings'  // プロジェクト設定ファイル
  | 'localSettings'    // ローカル設定ファイル
  | 'session'          // セッション (メモリのみ)
  | 'cliArg'           // CLI 引数 (変更不可)
```

### マネージドポリシーによる制限

**ソース**: `src/utils/permissions/permissionsLoader.ts:31-43`

```typescript
function shouldAllowManagedPermissionRulesOnly(): boolean {
  return getSettingsForSource('policySettings')
    ?.allowManagedPermissionRulesOnly === true
}

// マネージドルールのみの場合、"always allow" オプションを非表示
function shouldShowAlwaysAllowOptions(): boolean {
  return !shouldAllowManagedPermissionRulesOnly()
}
```

### ルール追加時の安全策

**ソース**: `src/utils/permissions/permissionsLoader.ts:229-296`

ルール追加時には:
1. マネージドポリシーが有効なら新ルール追加を拒否
2. 既存ルールの正規化 (レガシー名の変換) で重複チェック
3. バリデーションエラー時にはレニエントパーサーにフォールバックし既存ルールを保護

---

## ヘッドレスエージェントの権限処理

**ソース**: `src/utils/permissions/permissions.ts:400-460`

UI を持たない非同期/ヘッドレスエージェントでは、権限プロンプトを表示できないため:

1. `PermissionRequest` フックを実行
2. フックが allow/deny を決定した場合はその結果を採用
3. フックが `interrupt` を返した場合はエージェントを中断
4. フックが判定しなかった場合は自動拒否にフォールバック

---

## 権限チェックのレイヤー構造

権限チェックは以下の順序で評価される:

```
1. Permission Mode チェック
   └── plan / bypassPermissions / dontAsk / auto の特殊処理

2. Deny Rules チェック
   └── policySettings > projectSettings > userSettings の順

3. Allow Rules チェック
   └── ツール全体 or コンテンツベースのマッチ

4. Ask Rules チェック
   └── 強制確認ルール

5. Classifier チェック (auto モードのみ)
   └── Bash 分類器 / トランスクリプト分類器

6. Default: ユーザーにプロンプト表示
```
