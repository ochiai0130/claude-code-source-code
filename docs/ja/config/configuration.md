# 設定と構成管理

Claude Code v2.1.88 ソースコード分析

## 概要

Claude Codeの設定システムは、複数のレイヤーから成る階層的な構成管理アーキテクチャを採用している。グローバル設定（`~/.claude/config.json`）、プロジェクト設定（`settings.json`）、CLAUDE.mdファイル群、環境変数、およびリモートマネージド設定が統合され、最終的な動作設定が決定される。

---

## 1. config.ts の構造と責務

**ファイル:** `src/utils/config.ts`（約51KB、580行以上の型定義）

### 1.1 主要な型定義

#### GlobalConfig（L183-578）

グローバル設定の完全な型定義。`~/.claude/config.json` に永続化される。主要フィールド:

```typescript
export type GlobalConfig = {
  projects?: Record<string, ProjectConfig>  // プロジェクト別設定
  numStartups: number                       // 起動回数
  theme: ThemeSetting                       // テーマ設定
  autoCompactEnabled: boolean               // 自動コンパクト
  verbose: boolean                          // 詳細ログ
  env: { [key: string]: string }            // 環境変数
  fileCheckpointingEnabled: boolean         // ファイルチェックポイント
  cachedStatsigGates: Record<string, boolean> // 機能フラグキャッシュ
  cachedGrowthBookFeatures?: Record<string, unknown> // GrowthBook機能キャッシュ
  // ... 他多数のフィールド
}
```

#### ProjectConfig（L76-136）

プロジェクトごとの設定。`config.json` 内の `projects[normalizedPath]` に保存される:

```typescript
export type ProjectConfig = {
  allowedTools: string[]             // 許可済みツール
  mcpContextUris: string[]           // MCPコンテキストURI
  mcpServers?: Record<string, McpServerConfig>  // MCPサーバー
  hasTrustDialogAccepted?: boolean   // 信頼ダイアログ承認
  // ... セッションメトリクス、MCP設定等
}
```

### 1.2 デフォルト値の管理

`createDefaultGlobalConfig()`（L585-623）はファクトリ関数としてデフォルト値を生成する。共有定数のディープクローンではなく、毎回新しいオブジェクトを返すことで参照の安全性を確保している:

```typescript
function createDefaultGlobalConfig(): GlobalConfig {
  return {
    numStartups: 0,
    theme: 'dark',
    autoCompactEnabled: true,
    verbose: false,
    fileCheckpointingEnabled: true,
    // ...
  }
}
```

### 1.3 再入ガード

`config.ts` L48-51には再入ガードが実装されている:

```typescript
let insideGetConfig = false
```

これは `getConfig → logEvent → getGlobalConfig → getConfig` の無限再帰を防止する。`logEvent` のサンプリングチェックがGrowthBook機能をグローバル設定から読み取るため、設定ファイルが破損している場合にこの循環が発生し得る。

---

## 2. 設定ファイル体系

### 2.1 settings.json ファイルの階層

**ファイル:** `src/utils/settings/constants.ts`

設定ソースは以下の優先順位で定義される（後のソースが前を上書き）:

```typescript
export const SETTING_SOURCES = [
  'userSettings',      // ~/.claude/settings.json（ユーザーグローバル）
  'projectSettings',   // .claude/settings.json（プロジェクト共有）
  'localSettings',     // .claude/settings.local.json（gitignore対象）
  'flagSettings',      // --settings CLIフラグ
  'policySettings',    // managed-settings.json またはリモートAPI
] as const
```

**重要:** `policySettings` と `flagSettings` は常に有効であり、`--setting-sources` フラグで無効化できない（`constants.ts` L159-167）。

### 2.2 ファイルパスの解決

| ソース | パス |
|--------|------|
| ユーザー設定 | `~/.claude/settings.json` |
| プロジェクト設定 | `<project>/.claude/settings.json` |
| ローカル設定 | `<project>/.claude/settings.local.json` |
| マネージド設定（ファイル） | プラットフォーム依存（`/etc/claude-code/managed-settings.json` 等） |
| マネージド設定（ドロップイン） | `managed-settings.d/*.json` |
| リモート設定 | `~/.claude/remote-settings.json`（キャッシュ） |

### 2.3 マネージド設定のドロップイン方式

`settings.ts` L62-121の `loadManagedFileSettings()` は、systemd/sudoersのドロップイン規約に従う:

1. `managed-settings.json`（基本ファイル、最低優先度）をマージ
2. `managed-settings.d/*.json` をアルファベット順にソートしてマージ（後のファイルが優先）

```
例: 10-otel.json, 20-security.json
→ 独立チームが単一ファイルを調整せずにポリシーを配布可能
```

---

## 3. 設定スキーマとバリデーション

### 3.1 SettingsSchema

**ファイル:** `src/utils/settings/types.ts` L255以降

Zod v4を使用したスキーマ定義。主要セクション:

| セクション | 説明 |
|------------|------|
| `apiKeyHelper` | 認証値を出力するスクリプトのパス |
| `env` | セッション環境変数 |
| `permissions` | ツール使用権限（allow/deny/ask） |
| `hooks` | ライフサイクルフック定義 |
| `model` | デフォルトモデルのオーバーライド |
| `availableModels` | エンタープライズ向けモデル許可リスト |
| `mcpServers` | MCPサーバー設定 |
| `allowedMcpServers` / `deniedMcpServers` | MCP許可/拒否リスト |
| `disableAllHooks` | 全フック無効化 |
| `allowManagedHooksOnly` | マネージドフックのみ許可 |
| `strictPluginOnlyCustomization` | プラグイン以外のカスタマイズをロック |

### 3.2 後方互換性ポリシー

`types.ts` L209-241のコメントに明記:

- **許可される変更:** 新しいオプショナルフィールド追加、enum値の追加、バリデーションの緩和
- **禁止される変更:** フィールドの削除、enum値の削除、オプショナルを必須に変更、型の厳格化
- 無効なフィールドはファイルに保持され（削除されない）、ユーザーが修正可能

### 3.3 PermissionsSchema

`types.ts` L42-85で定義:

```typescript
export const PermissionsSchema = lazySchema(() =>
  z.object({
    allow: z.array(PermissionRuleSchema()).optional(),
    deny: z.array(PermissionRuleSchema()).optional(),
    ask: z.array(PermissionRuleSchema()).optional(),
    defaultMode: z.enum(PERMISSION_MODES).optional(),
    disableBypassPermissionsMode: z.enum(['disable']).optional(),
    additionalDirectories: z.array(z.string()).optional(),
  }).passthrough()
)
```

---

## 4. CLAUDE.md の解析と統合

**ファイル:** `src/utils/claudemd.ts`

### 4.1 ファイル読み込み順序

`claudemd.ts` L1-26のコメントに明記されている読み込み順序:

1. **マネージドメモリ** (`/etc/claude-code/CLAUDE.md`) - 全ユーザー向けグローバル指示
2. **ユーザーメモリ** (`~/.claude/CLAUDE.md`) - 個人のグローバル指示
3. **プロジェクトメモリ** (`CLAUDE.md`, `.claude/CLAUDE.md`, `.claude/rules/*.md`) - リポジトリにコミットされる指示
4. **ローカルメモリ** (`CLAUDE.local.md`) - 個人のプロジェクト固有指示

ファイルは**逆優先順位**で読み込まれる。後で読み込まれたファイルほどモデルが重視する。

### 4.2 ディレクトリ走査

カレントディレクトリからルートに向かって上方向に走査し、各ディレクトリで以下を確認:
- `CLAUDE.md`
- `.claude/CLAUDE.md`
- `.claude/rules/*.md`（全.mdファイル）

カレントディレクトリに近いファイルほど優先度が高い（後で読み込まれる）。

### 4.3 @include ディレクティブ

`claudemd.ts` L18-25:

```
構文: @path, @./relative/path, @~/home/path, @/absolute/path
- @path（プレフィックスなし）は相対パスとして扱われる
- リーフテキストノードでのみ動作（コードブロック内では無効）
- インクルードされたファイルは、インクルード元の前に独立エントリとして追加
- 循環参照は処理済みファイルの追跡により防止
- 存在しないファイルはサイレントに無視
```

### 4.4 テキストファイル制限

`claudemd.ts` L96-227: `@include` で読み込めるのはテキストファイル拡張子のみ。`.md`, `.ts`, `.py`, `.json`, `.yaml` 等、100種以上の拡張子がホワイトリストで定義されており、バイナリファイル（画像、PDF等）のメモリへの読み込みを防止している。

### 4.5 HTMLコメントの除去

`stripHtmlComments()`（L292-334）は、ブロックレベルのHTMLコメント（`<!-- ... -->`）をマークダウンから除去する。markedレキサーを使用してブロックレベルのコメントのみを対象とし、インラインコードやフェンスドコードブロック内のコメントは保持される。

### 4.6 文字数制限

```typescript
export const MAX_MEMORY_CHARACTER_COUNT = 40000  // L93
```

---

## 5. 環境変数によるランタイムオーバーライド

### 5.1 settings.json の env セクション

各設定ソースの `env` フィールドで環境変数を定義可能:

```json
{
  "env": {
    "ANTHROPIC_MODEL": "claude-opus-4-6",
    "CLAUDE_CODE_MAX_TOKENS": "8192"
  }
}
```

`EnvironmentVariablesSchema`（`types.ts` L35-37）により、値は `z.coerce.string()` で文字列に変換される。

### 5.2 主要な環境変数

| 環境変数 | 用途 |
|----------|------|
| `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` | SessionEndフックのタイムアウト（デフォルト1500ms） |
| `CLAUDE_CODE_ENTRYPOINT` | エントリポイント識別（`local-agent` でリモート設定を無効化） |
| `CLAUDE_CODE_ENABLE_XAA` | XAA IdP設定の有効化 |
| `CLAUDE_CODE_OAUTH_TOKEN` | OAuth トークンの外部注入 |

---

## 6. 設定の優先順位

最終的な設定マージの優先順位（低→高）:

```
1. userSettings     (~/.claude/settings.json)
2. projectSettings  (.claude/settings.json)
3. localSettings    (.claude/settings.local.json)
4. flagSettings     (--settings CLIフラグ)
5. policySettings   (managed-settings.json + remote API)
```

**注意事項:**
- `policySettings` が最高優先度を持ち、エンタープライズ管理者のポリシーを強制する
- `policySettings` と `flagSettings` は `--setting-sources` フラグで無効化不可（`constants.ts` L159-167）
- 配列フィールド（`allowedMcpServers` 等）はソース間でマージされる
- `disableAllHooks` が非マネージド設定で設定された場合でも、マネージドフックは実行される（`hooksConfigSnapshot.ts` L47-49）

### GlobalConfig vs Settings の関係

- **GlobalConfig** (`config.json`): UIの状態、セッションメトリクス、キャッシュ等の永続化データ
- **Settings** (`settings.json`): 権限、フック、環境変数、ポリシー等の設定

両者は異なるファイルに保存され、異なるマージ戦略を持つ。Settingsは5つのソースからマージされるのに対し、GlobalConfigは単一ファイル。

---

## 関連ファイル

| ファイル | 説明 |
|----------|------|
| `src/utils/config.ts` | GlobalConfig/ProjectConfig型定義、設定読み書き |
| `src/utils/settings/settings.ts` | 設定ソースの読み込み・マージロジック |
| `src/utils/settings/types.ts` | SettingsSchema（Zodスキーマ）定義 |
| `src/utils/settings/constants.ts` | SETTING_SOURCES、優先順位定義 |
| `src/utils/settings/settingsCache.ts` | 設定キャッシュ管理 |
| `src/utils/settings/managedPath.ts` | マネージド設定のパス解決 |
| `src/utils/settings/validation.ts` | バリデーションとエラーフォーマット |
| `src/utils/claudemd.ts` | CLAUDE.mdファイルの解析と統合 |
