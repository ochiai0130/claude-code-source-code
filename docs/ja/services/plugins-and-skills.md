# プラグイン・スキルシステム

> Claude Code v2.1.88 ソースコード解析 - `src/plugins/`, `src/skills/`, `src/utils/plugins/`, `src/services/plugins/`

## 概要

Claude Code のプラグイン・スキルシステムは、CLI の機能を拡張するための2つの仕組みを提供する。**プラグイン**はマーケットプレイス経由でインストール可能なパッケージで、スキル・フック・MCP サーバーなどの複数コンポーネントを含む。**スキル**はスラッシュコマンドとして呼び出し可能なプロンプトで、バンドル版・ユーザー定義版・MCP 提供版がある。

## ファイル構成

### プラグイン関連

| ファイル / ディレクトリ | 役割 |
|---|---|
| `src/plugins/builtinPlugins.ts` | ビルトインプラグインレジストリ |
| `src/plugins/bundled/` | バンドルプラグインの実装 |
| `src/utils/plugins/pluginLoader.ts` | プラグインの発見・読み込み・バリデーション |
| `src/utils/plugins/schemas.ts` | プラグイン関連の Zod スキーマ |
| `src/utils/plugins/marketplaceManager.ts` | マーケットプレイスデータ管理 |
| `src/utils/plugins/pluginDirectories.ts` | プラグインディレクトリのパス管理 |
| `src/utils/plugins/pluginVersioning.ts` | バージョン管理 |
| `src/utils/plugins/dependencyResolver.ts` | 依存関係の解決 |
| `src/utils/plugins/installedPluginsManager.ts` | インストール済みプラグイン管理 |
| `src/utils/plugins/pluginBlocklist.ts` | プラグインブロックリスト |
| `src/utils/plugins/pluginPolicy.ts` | プラグインポリシー |
| `src/utils/plugins/mcpPluginIntegration.ts` | プラグイン由来 MCP サーバーの統合 |
| `src/utils/plugins/pluginAutoupdate.ts` | 自動アップデート |
| `src/utils/plugins/zipCache.ts` | ZIP キャッシュ |
| `src/services/plugins/PluginInstallationManager.ts` | インストール管理サービス |
| `src/services/plugins/pluginCliCommands.ts` | CLI コマンド (`/plugin`) |
| `src/services/plugins/pluginOperations.ts` | プラグイン操作 |

### スキル関連

| ファイル / ディレクトリ | 役割 |
|---|---|
| `src/skills/bundledSkills.ts` | バンドルスキルのレジストリ |
| `src/skills/loadSkillsDir.ts` | ファイルベーススキルの読み込み |
| `src/skills/mcpSkillBuilders.ts` | MCP スキルビルダー登録 |
| `src/skills/bundled/` | バンドルスキルの実装 |

## プラグインシステムのアーキテクチャ

### プラグインの種類

#### 1. ビルトインプラグイン (`src/plugins/builtinPlugins.ts`)

CLI にバンドルされ、ユーザーが `/plugin` UI から有効/無効を切り替え可能:

```typescript
type BuiltinPluginDefinition = {
  name: string
  description: string
  version: string
  defaultEnabled?: boolean          // デフォルトで有効か（true がデフォルト）
  isAvailable?: () => boolean       // 利用可能条件
  skills?: BundledSkillDefinition[] // 提供するスキル
  hooks?: HooksSettings             // フック設定
  mcpServers?: Record<string, McpServerConfig> // MCP サーバー
}
```

**プラグイン ID 形式**: `{name}@builtin`

```typescript
// 登録
registerBuiltinPlugin(definition)

// 取得（有効/無効で分類）
getBuiltinPlugins(): { enabled: LoadedPlugin[], disabled: LoadedPlugin[] }

// スキルコマンド取得（有効プラグインのみ）
getBuiltinPluginSkillCommands(): Command[]
```

有効/無効状態はユーザー設定の `enabledPlugins` に永続化される:

```typescript
const userSetting = settings?.enabledPlugins?.[pluginId]
const isEnabled = userSetting !== undefined ? userSetting === true : (definition.defaultEnabled ?? true)
```

#### 2. マーケットプレイスプラグイン

外部マーケットプレイスから Git/NPM 経由でインストールされるプラグイン。

**プラグイン ID 形式**: `{name}@{marketplace}`

**ディレクトリ構造:**

```
my-plugin/
├── plugin.json          # オプションのマニフェスト
├── commands/            # カスタムスラッシュコマンド
│   ├── build.md
│   └── deploy.md
├── agents/              # カスタム AI エージェント
│   └── test-runner.md
└── hooks/               # フック設定
    └── hooks.json
```

#### 3. セッションプラグイン

`--plugin-dir` CLI フラグまたは SDK `plugins` オプションでセッション単位で有効化されるプラグイン。

### プラグインの読み込み (`src/utils/plugins/pluginLoader.ts`)

#### 発見ソース（優先順位順）

1. **マーケットプレイスベース**: 設定内の `plugin@marketplace` 形式のエントリ
2. **セッション限定**: `--plugin-dir` フラグまたは SDK plugins オプション

#### キャッシュ構造

バージョン管理付きキャッシュ:

```
~/.claude/plugins/cache/{marketplace}/{plugin}/{version}/
```

```typescript
// バージョン付きキャッシュパス
getVersionedCachePath(pluginId, version): string

// レガシーキャッシュパス（後方互換性）
getLegacyCachePath(pluginName): string

// パス解決（バージョン付き → レガシー → 新規）
resolvePluginPath(pluginId, version?): Promise<string>
```

#### シードキャッシュ

事前配置されたプラグインキャッシュ（BYOC 環境向け）:

```typescript
// シードディレクトリを優先順位順にプローブ
probeSeedCache(pluginId, version): Promise<string | null>

// バージョン不明時のプローブ（単一バージョンのみマッチ）
probeSeedCacheAnyVersion(pluginId): Promise<string | null>
```

#### ZIP キャッシュ

フィーチャーフラグ有効時、プラグインを ZIP 形式でキャッシュ:

```typescript
getVersionedZipCachePath(pluginId, version): string
// → `${getVersionedCachePath(pluginId, version)}.zip`
```

### プラグインの読み込み結果

```typescript
type LoadedPlugin = {
  name: string
  manifest: PluginManifest
  path: string          // ファイルシステムパスまたは 'builtin' センチネル
  source: string        // pluginId (e.g., 'slack@anthropic')
  repository: string
  enabled: boolean
  isBuiltin?: boolean
  hooksConfig?: HooksSettings
  mcpServers?: Record<string, McpServerConfig>
}

type PluginLoadResult = {
  enabled: LoadedPlugin[]
  disabled: LoadedPlugin[]
  errors: PluginError[]
}
```

### プラグインの実行

#### フック (`src/utils/plugins/loadPluginHooks.ts`)

プラグインが提供するフック（コマンド実行前後の処理）:

```typescript
type HooksSettings = {
  preToolExecution?: HookDefinition[]
  postToolExecution?: HookDefinition[]
  // ...
}
```

#### MCP サーバー統合 (`src/utils/plugins/mcpPluginIntegration.ts`)

プラグインが MCP サーバーを提供する場合、`getPluginMcpServers()` で MCP 設定に統合される。`ScopedMcpServerConfig.pluginSource` にプラグインのソースが記録される。

#### エージェント (`src/utils/plugins/loadPluginAgents.ts`)

プラグインが AI エージェント定義を提供する。

#### コマンド (`src/utils/plugins/loadPluginCommands.ts`)

プラグインがカスタムスラッシュコマンドを提供する。

### マーケットプレイス管理

#### マーケットプレイスヘルパー (`src/utils/plugins/marketplaceHelpers.ts`)

```typescript
getBlockedMarketplaces()              // ブロックされたマーケットプレイス
getStrictKnownMarketplaces()          // 許可されたマーケットプレイス
isSourceAllowedByPolicy(source)       // ポリシーによる許可チェック
isSourceInBlocklist(source)           // ブロックリストチェック
```

#### 公式マーケットプレイス (`src/utils/plugins/officialMarketplace.ts`)

Anthropic 公式マーケットプレイスの管理。`officialMarketplaceGcs.ts` で GCS からのデータ取得を行う。

### プラグインセキュリティ

- **パスバリデーション**: `validatePathWithinBase()` でパストラバーサル攻撃を防止
- **ポリシー制御**: `pluginPolicy.ts` でエンタープライズポリシーに基づく制限
- **ブロックリスト**: `pluginBlocklist.ts` でブロック対象プラグインを管理
- **フラグ付け**: `pluginFlagging.ts` で危険なプラグインにフラグを付与
- **バリデーション**: `validatePlugin.ts` でプラグインの整合性を検証

## スキルシステムの仕組み

### スキルの種類

| 種類 | ソース | `loadedFrom` | 登録方法 |
|---|---|---|---|
| バンドルスキル | CLI に同梱 | `'bundled'` | `registerBundledSkill()` |
| ユーザー定義スキル | `.claude/skills/` 等のマークダウン | `'skills'` | ディレクトリスキャン |
| プラグインスキル | プラグインのマニフェスト | `'plugin'` | プラグイン読み込み時 |
| MCP スキル | MCP サーバー | `'mcp'` | MCP 接続時 |
| マネージドスキル | エンタープライズ管理 | `'managed'` | 管理設定から |
| レガシーコマンド | `commands/` ディレクトリ | `'commands_DEPRECATED'` | ディレクトリスキャン |

### バンドルスキル (`src/skills/bundledSkills.ts`)

CLI にコンパイルされ、全ユーザーが利用可能:

```typescript
type BundledSkillDefinition = {
  name: string
  description: string
  aliases?: string[]
  whenToUse?: string             // モデルが自動呼び出しする条件
  argumentHint?: string
  allowedTools?: string[]
  model?: string                 // 使用するモデル指定
  disableModelInvocation?: boolean
  userInvocable?: boolean
  isEnabled?: () => boolean      // 動的有効/無効判定
  hooks?: HooksSettings
  context?: 'inline' | 'fork'   // 実行コンテキスト
  agent?: string
  files?: Record<string, string> // 参照ファイル（ディスクに展開）
  getPromptForCommand: (args: string, context: ToolUseContext) => Promise<ContentBlockParam[]>
}
```

**登録と取得:**

```typescript
registerBundledSkill(definition)  // 起動時に登録
getBundledSkills(): Command[]     // 全バンドルスキル取得
```

**参照ファイルの展開:**

`files` フィールドが指定された場合、初回呼び出し時にディスクに展開される:

```typescript
// 展開先: ~/.claude/bundled-skills/{skillName}/
getBundledSkillExtractDir(skillName): string

// 遅延展開（初回のみ）
async function extractBundledSkillFiles(skillName, files): Promise<string | null>
```

展開後、プロンプトの先頭に `Base directory for this skill: <dir>` が付加され、モデルが Read/Grep で参照可能になる。

#### バンドルスキル一覧 (`src/skills/bundled/`)

| ファイル | スキル名 | 説明 |
|---|---|---|
| `claudeApi.ts` / `claudeApiContent.ts` | claude-api | Claude API / SDK ガイド |
| `simplify.ts` | simplify | コード簡素化レビュー |
| `loop.ts` | loop | 定期実行 |
| `updateConfig.ts` | update-config | 設定更新 |
| `verify.ts` / `verifyContent.ts` | verify | 変更検証 |
| `debug.ts` | debug | デバッグ支援 |
| `remember.ts` | remember | メモリー管理 |
| `batch.ts` | batch | バッチ処理 |
| `keybindings.ts` | keybindings | キーバインド |
| `stuck.ts` | stuck | スタック時の支援 |
| `claudeInChrome.ts` | claude-in-chrome | Chrome 統合 |
| `scheduleRemoteAgents.ts` | schedule-remote-agents | リモートエージェント |
| `skillify.ts` | skillify | スキル化 |
| `loremIpsum.ts` | lorem-ipsum | テストデータ |

### ユーザー定義スキル (`src/skills/loadSkillsDir.ts`)

ファイルシステムからマークダウンファイルを読み込み、スキルとして登録する。

#### スキルの検索パス

```typescript
function getSkillsPath(source: SettingSource | 'plugin', dir: 'skills' | 'commands'): string {
  switch (source) {
    case 'policySettings':   return join(getManagedFilePath(), '.claude', dir)
    case 'userSettings':     return join(getClaudeConfigHomeDir(), dir)
    case 'projectSettings':  return `.claude/${dir}`
    case 'plugin':           return 'plugin'
  }
}
```

#### マークダウンフロントマター

各スキルファイルのフロントマターから以下のメタデータを抽出:

- `name` - スキル名
- `description` - 説明文
- `whenToUse` - 自動呼び出し条件
- `argumentHint` - 引数ヒント
- `allowedTools` - 許可するツール
- `model` - 使用モデル
- `hooks` - フック設定
- `context` - 実行コンテキスト (`inline` / `fork`)
- `agent` - エージェント定義

#### 重複検出

`getFileIdentity()` が symlink を解決し、異なるパスから同一ファイルが読み込まれるのを防止する。

#### トークン推定

```typescript
function estimateSkillFrontmatterTokens(skill: Command): number {
  const frontmatterText = [skill.name, skill.description, skill.whenToUse].filter(Boolean).join(' ')
  return roughTokenCountEstimation(frontmatterText)
}
```

### MCP スキル (`src/skills/mcpSkillBuilders.ts`)

フィーチャーフラグ `MCP_SKILLS` 有効時、MCP サーバーのプロンプトをスキルとして登録:

```typescript
registerMCPSkillBuilders()  // MCP スキルビルダーの初期化
```

MCP スキルの `loadedFrom` は `'mcp'` で、`Command.source` は `'bundled'` ではなくサーバー名ベースの値になる。

### スキルの検索と呼び出し

#### Skill Tool (`src/tools/ToolSearchTool/`)

モデルが `Skill` ツールを通じてスキルを検索・呼び出しできる。`isDeferredToolsDeltaEnabled()` や `isToolSearchEnabled()` のフィーチャーフラグで制御される。

#### 呼び出しフロー

1. ユーザーがスラッシュコマンド (`/skill-name args`) を入力、またはモデルが Skill ツールを使用
2. `Command.getPromptForCommand(args, context)` が呼ばれ、プロンプトブロックを生成
3. プロンプトがユーザーメッセージとして挿入される
4. モデルが応答を生成

#### 実行コンテキスト

- `'inline'` - 現在の会話内で実行（デフォルト）
- `'fork'` - 新しいコンテキストで実行（サブエージェント的）

## サービス層 (`src/services/plugins/`)

### PluginInstallationManager (`PluginInstallationManager.ts`)

プラグインのインストール・アンインストール・アップデートを管理するサービスクラス。

### pluginCliCommands (`pluginCliCommands.ts`)

`/plugin` スラッシュコマンドの実装。プラグインの一覧表示、有効/無効切り替え、インストール/アンインストール操作を提供する。

### pluginOperations (`pluginOperations.ts`)

プラグインの高レベル操作（インストール、削除、有効化等）のロジック。
