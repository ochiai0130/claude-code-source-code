# システムプロンプト

> 主要ソースファイル:
> - `src/utils/systemPrompt.ts` (システムプロンプト構築ロジック)
> - `src/utils/systemPromptType.ts` (ブランド型定義)
> - `src/utils/queryContext.ts` (キャッシュキープレフィックス構築)
> - `src/context.ts` (ユーザー/システムコンテキスト生成)
> - `src/constants/prompts.ts` (プロンプトテンプレート)

## 概要

Claude Codeのシステムプロンプトは、複数の構成要素を動的に組み合わせて構築される。プロンプトは`SystemPrompt`ブランド型（`readonly string[]`のブランド付き配列）で表現され、APIレイヤーでキャッシュ制御ブレークポイントの挿入に使われる。

```
システムプロンプト構成
├── defaultSystemPrompt (または customSystemPrompt)
│     ├── イントロセクション
│     ├── ツール説明
│     ├── システム規則
│     ├── タスク実行指示
│     ├── MCP指示
│     ├── 出力スタイル
│     └── 言語設定
├── memoryMechanicsPrompt (条件付き)
├── appendSystemPrompt (オプション)
├── userContext (プリペンド)
│     ├── CLAUDE.md
│     └── 現在の日付
└── systemContext (アペンド)
      ├── git status
      └── キャッシュブレーカー (ant-only)
```

## SystemPrompt ブランド型

`src/utils/systemPromptType.ts` で定義される。依存関係を最小化するため、意図的に独立したモジュールとして分離されている。

```typescript
// src/utils/systemPromptType.ts
export type SystemPrompt = readonly string[] & {
  readonly __brand: 'SystemPrompt'
}

export function asSystemPrompt(value: readonly string[]): SystemPrompt {
  return value as SystemPrompt
}
```

文字列配列にブランドを付けることで、型安全にシステムプロンプトの配列を扱える。配列の各要素がAPIレイヤーでの`cache_control`ブレークポイント挿入の単位となる。

## プロンプト構築プロセス

### fetchSystemPromptParts() (src/utils/queryContext.ts, L44-74)

`QueryEngine`から呼び出される最初のステップ。3つの構成要素を並行して取得する:

```typescript
const [defaultSystemPrompt, userContext, systemContext] = await Promise.all([
  customSystemPrompt !== undefined
    ? Promise.resolve([])
    : getSystemPrompt(tools, mainLoopModel, additionalWorkingDirectories, mcpClients),
  getUserContext(),
  customSystemPrompt !== undefined ? Promise.resolve({}) : getSystemContext(),
])
```

**重要な動作**: `customSystemPrompt`が設定されている場合:
- `getSystemPrompt()`はスキップされ、空配列が返る
- `getSystemContext()`もスキップされる（デフォルトプロンプトと紐づくコンテキストが不要なため）
- `getUserContext()`は常に取得される（CLAUDE.mdと日付は常に必要）

### getSystemPrompt() (src/constants/prompts.ts)

デフォルトシステムプロンプトの本体を構築する。以下のセクションで構成される:

1. **イントロセクション** (`getSimpleIntroSection`): エージェントの基本的な役割定義、サイバーリスク指示
2. **システムセクション** (`getSimpleSystemSection`): ツール表示、権限モード、フック、自動要約に関する規則
3. **タスク実行セクション** (`getSimpleDoingTasksSection`): コードスタイル、ファイル操作、Git操作の指示
4. **ツール固有セクション**: 各ツール（Bash、Read、Edit、Write、Glob、Grep等）の使用方法
5. **MCPインストラクション** (`getMcpInstructionsSection`): MCPサーバーのツール使用指示
6. **出力スタイル** (`getOutputStyleSection`): ユーザー設定のスタイル指示
7. **言語設定** (`getLanguageSection`): 応答言語の指定
8. **動的境界マーカー** (`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`): キャッシュのスコープ境界

```
__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__
```

この境界マーカーより前のコンテンツは`scope: 'global'`でキャッシュ可能（組織横断）。境界より後はユーザー/セッション固有のコンテンツ。

### buildEffectiveSystemPrompt() (src/utils/systemPrompt.ts, L41-123)

最終的なシステムプロンプトを決定する関数。優先順位に基づく選択ロジック:

```
優先順位:
0. overrideSystemPrompt → 他のすべてを置換（ループモード用）
1. コーディネーターモード → コーディネータープロンプトに置換
2. エージェントシステムプロンプト:
   - プロアクティブモード: デフォルト + エージェント指示を追記
   - 通常モード: エージェント指示で置換
3. customSystemPrompt → デフォルトを置換
4. defaultSystemPrompt → 標準プロンプト

+ appendSystemPrompt は常に末尾に追加（override時を除く）
```

**コーディネーターモード** (L62-75):

```typescript
if (feature('COORDINATOR_MODE') && isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)) {
  const { getCoordinatorSystemPrompt } = require('../coordinator/coordinatorMode.js')
  return asSystemPrompt([getCoordinatorSystemPrompt(), ...(appendSystemPrompt ? [appendSystemPrompt] : [])])
}
```

**プロアクティブモード** (L103-113):
エージェント指示がデフォルトプロンプトに追記される（置換ではない）。自律エージェントのアイデンティティ + ドメイン固有の指示。

## CLAUDE.mdの読み込みと統合

### getUserContext() (src/context.ts, L155-189)

CLAUDE.mdの内容はユーザーコンテキストとして読み込まれる:

```typescript
export const getUserContext = memoize(async () => {
  const shouldDisableClaudeMd =
    isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_CLAUDE_MDS) ||
    (isBareMode() && getAdditionalDirectoriesForClaudeMd().length === 0)
  
  const claudeMd = shouldDisableClaudeMd
    ? null
    : getClaudeMds(filterInjectedMemoryFiles(await getMemoryFiles()))
  
  setCachedClaudeMdContent(claudeMd || null)
  
  return {
    ...(claudeMd && { claudeMd }),
    currentDate: `Today's date is ${getLocalISODate()}.`,
  }
})
```

**無効化条件**:
- `CLAUDE_CODE_DISABLE_CLAUDE_MDS`環境変数が`true`
- `--bare`モードかつ追加ディレクトリの指定なし

**重要**: `memoize`されているため、同一セッション内でCLAUDE.mdは1回しか読み込まれない。セッション開始後にCLAUDE.mdを変更しても、コンパクション後のSessionStartフックで再読み込みされるまで反映されない。

### メモリファイルの探索

`getMemoryFiles()` (`src/utils/claudemd.ts`から) は以下のパスを探索する:

- カレントディレクトリの`.claude/settings.md`、`CLAUDE.md`等
- 親ディレクトリチェーン上のCLAUDE.mdファイル
- ホームディレクトリの設定ファイル
- `--add-dir`で指定された追加ディレクトリ

## システムコンテキスト

### getSystemContext() (src/context.ts, L116-150)

Git状態を含むシステムレベルのコンテキスト:

```typescript
export const getSystemContext = memoize(async () => {
  const gitStatus = isEnvTruthy(process.env.CLAUDE_CODE_REMOTE) || !shouldIncludeGitInstructions()
    ? null
    : await getGitStatus()
  
  const injection = feature('BREAK_CACHE_COMMAND') ? getSystemPromptInjection() : null
  
  return {
    ...(gitStatus && { gitStatus }),
    ...(injection ? { cacheBreaker: `[CACHE_BREAKER: ${injection}]` } : {}),
  }
})
```

**Git状態** (`getGitStatus()`, L36-111): 以下の情報を並行取得:
- 現在のブランチ
- メインブランチ
- `git status --short`（2,000文字で切り詰め）
- 直近5コミットのログ
- gitユーザー名

**スキップ条件**:
- `CLAUDE_CODE_REMOTE` (CCR環境)
- git指示が無効化されている場合

## 動的プロンプト生成

### QueryEngine内での最終組み立て (src/QueryEngine.ts, L288-325)

`QueryEngine.submitMessage()`内で最終的なシステムプロンプトが組み立てられる:

```typescript
// 1. 基本パーツの取得
const { defaultSystemPrompt, userContext: baseUserContext, systemContext } =
  await fetchSystemPromptParts({ tools, mainLoopModel, ... })

// 2. コーディネーターモードのユーザーコンテキスト追加
const userContext = {
  ...baseUserContext,
  ...getCoordinatorUserContext(mcpClients, scratchpadDir),
}

// 3. メモリメカニクスプロンプト（条件付き）
const memoryMechanicsPrompt =
  customPrompt !== undefined && hasAutoMemPathOverride()
    ? await loadMemoryPrompt()
    : null

// 4. 最終組み立て
const systemPrompt = asSystemPrompt([
  ...(customPrompt !== undefined ? [customPrompt] : defaultSystemPrompt),
  ...(memoryMechanicsPrompt ? [memoryMechanicsPrompt] : []),
  ...(appendSystemPrompt ? [appendSystemPrompt] : []),
])
```

### query.ts内でのコンテキスト適用 (src/query.ts, L449-451)

`queryLoop()`内で、`systemContext`が`systemPrompt`に追加される:

```typescript
const fullSystemPrompt = asSystemPrompt(
  appendSystemContext(systemPrompt, systemContext),
)
```

`userContext`はメッセージ配列にプリペンドされる:

```typescript
messages: prependUserContext(messagesForQuery, userContext),
```

この分離により、`userContext`（CLAUDE.md、日付）はメッセージの先頭に配置され、`systemContext`（git status）はシステムプロンプトの末尾に配置される。APIのプロンプトキャッシュにおいて、ユーザーコンテキストはメッセージ側のキャッシュに、システムコンテキストはシステムプロンプト側のキャッシュに属する。

## キャッシュ戦略

### 静的/動的の分離

`SYSTEM_PROMPT_DYNAMIC_BOUNDARY` (src/constants/prompts.ts, L114-115) を境界として:

- **静的部分** (境界より前): ツール説明、基本規則、タスク指示 → `scope: 'global'`でキャッシュ可能
- **動的部分** (境界より後): ユーザー設定、環境固有情報 → セッション固有キャッシュ

### memoizeによるセッション内キャッシュ

`getUserContext()`と`getSystemContext()`は`lodash/memoize`でキャッシュされ、セッション内で1回のみ実行される。キャッシュブレーカーの注入（`setSystemPromptInjection`）時にはキャッシュがクリアされる (src/context.ts, L32-33)。
