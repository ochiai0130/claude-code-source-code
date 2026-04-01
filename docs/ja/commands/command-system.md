# コマンドシステム

Claude Code v2.1.88 のコマンドシステムは、スラッシュコマンドのレジストリ、ディスパッチ、動的ロードを統合的に管理する。

## 概要

コマンドシステムの中核は `src/commands.ts` (約755行) にあり、以下を管理する:

- 組み込みコマンドの定義・登録
- フィーチャーゲート付きコマンドの条件付きロード
- スキル、プラグイン、ワークフローからの動的コマンド統合
- コマンドのフィルタリング・検索・ディスパッチ

---

## コマンド型の定義

### Command 型

**ソース**: `src/types/command.ts`

コマンドは `CommandBase` と3つの実行型の Union として定義される:

```typescript
type Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)
```

| コマンド型 | 説明 | 特徴 |
|---|---|---|
| `prompt` | プロンプト型 | モデルにテキストを送信。スキルの基本型 |
| `local` | ローカル型 | ローカルで実行し `LocalCommandResult` を返す |
| `local-jsx` | ローカル JSX 型 | Ink UI をレンダリング。遅延ロード対応 |

### CommandBase のプロパティ

```typescript
// src/types/command.ts:175-203
type CommandBase = {
  availability?: CommandAvailability[]  // 'claude-ai' | 'console'
  description: string
  isEnabled?: () => boolean    // フィーチャーフラグ等による動的有効化
  isHidden?: boolean           // ヘルプ/タイプアヘッドから隠す
  name: string
  aliases?: string[]
  whenToUse?: string           // スキル仕様: いつ使うかの説明
  disableModelInvocation?: boolean  // モデルによる呼び出し禁止
  loadedFrom?: 'commands_DEPRECATED' | 'skills' | 'plugin' | 'managed' | 'bundled' | 'mcp'
  kind?: 'workflow'
  immediate?: boolean          // 即時実行 (キュー待ちなし)
  isSensitive?: boolean        // 引数を会話履歴から除外
}
```

### 可用性 (Availability)

コマンドのアクセス制御は `availability` フィールドで管理:

```typescript
// src/commands.ts:417-443
function meetsAvailabilityRequirement(cmd: Command): boolean {
  if (!cmd.availability) return true  // 未指定 = 全員利用可能
  for (const a of cmd.availability) {
    switch (a) {
      case 'claude-ai':     // Claude.ai OAuth サブスクライバー
      case 'console':       // Console API キーユーザー (1P 直接)
    }
  }
  return false
}
```

---

## コマンドレジストリの構造

### 静的コマンド登録

`COMMANDS()` 関数 (メモ化済み) が組み込みコマンドの一覧を返す:

**ソース**: `src/commands.ts:258-346`

```typescript
const COMMANDS = memoize((): Command[] => [
  addDir, advisor, agents, branch, btw, chrome, clear, color, compact,
  config, copy, desktop, context, contextNonInteractive, cost, diff,
  doctor, effort, exit, fast, files, heapDump, help, ide, init,
  keybindings, installGitHubApp, installSlackApp, mcp, memory, mobile,
  model, outputStyle, remoteEnv, plugin, pr_comments, releaseNotes,
  reloadPlugins, rename, resume, session, skills, stats, status,
  statusline, stickers, tag, theme, feedback, review, ultrareview,
  rewind, securityReview, terminalSetup, upgrade, extraUsage,
  extraUsageNonInteractive, rateLimitOptions, usage, usageReport, vim,
  // フィーチャーゲート付きコマンド (後述)
  ...(webCmd ? [webCmd] : []),
  ...(forkCmd ? [forkCmd] : []),
  ...(buddy ? [buddy] : []),
  // ... その他
  // 内部専用コマンド (USER_TYPE === 'ant' の場合のみ)
  ...(process.env.USER_TYPE === 'ant' && !process.env.IS_DEMO
    ? INTERNAL_ONLY_COMMANDS : []),
])
```

### 内部専用コマンド

**ソース**: `src/commands.ts:225-254`

```typescript
const INTERNAL_ONLY_COMMANDS = [
  backfillSessions, breakCache, bughunter, commit, commitPushPr,
  ctx_viz, goodClaude, issue, initVerifiers, mockLimits, bridgeKick,
  version, onboarding, share, summary, teleport, antTrace, perfIssue,
  env, oauthRefresh, debugToolCall, agentsPlatform, autofixPr,
  // フィーチャーゲート付き
  ...(forceSnip ? [forceSnip] : []),
  ...(ultraplan ? [ultraplan] : []),
  ...(subscribePr ? [subscribePr] : []),
  resetLimits, resetLimitsNonInteractive,
]
```

---

## コマンド一覧 (カテゴリ別)

### セッション管理

| コマンド名 | ソース | 説明 |
|---|---|---|
| `clear` | `src/commands/clear/` | 画面・会話クリア |
| `compact` | `src/commands/compact/` | コンテキスト圧縮 |
| `resume` | `src/commands/resume/` | セッション再開 |
| `session` | `src/commands/session/` | セッション管理 |
| `exit` | `src/commands/exit/` | 終了 |
| `rename` | `src/commands/rename/` | セッション名変更 |

### コンテキスト・ファイル

| コマンド名 | ソース | 説明 |
|---|---|---|
| `context` | `src/commands/context/` | コンテキスト表示 |
| `add-dir` | `src/commands/add-dir/` | ディレクトリ追加 |
| `files` | `src/commands/files/` | トラッキングファイル一覧 |
| `diff` | `src/commands/diff/` | 差分表示 |
| `branch` | `src/commands/branch/` | ブランチ操作 |

### 設定・環境

| コマンド名 | ソース | 説明 |
|---|---|---|
| `config` | `src/commands/config/` | 設定管理 |
| `color` | `src/commands/color/` | エージェントカラー変更 |
| `theme` | `src/commands/theme/` | ターミナルテーマ |
| `vim` | `src/commands/vim/` | Vim モード切替 |
| `keybindings` | `src/commands/keybindings/` | キーバインド管理 |
| `model` | `src/commands/model/` | モデル選択 |
| `output-style` | `src/commands/output-style/` | 出力スタイル |
| `effort` | `src/commands/effort/` | 推論努力レベル |
| `fast` | `src/commands/fast/` | 高速モード |

### 認証・アカウント

| コマンド名 | ソース | 説明 |
|---|---|---|
| `login` | `src/commands/login/` | ログイン |
| `logout` | `src/commands/logout/` | ログアウト |
| `usage` | `src/commands/usage/` | 使用量表示 |
| `cost` | `src/commands/cost/` | セッションコスト表示 |
| `passes` | `src/commands/passes/` | パス管理 |

### ツール・統合

| コマンド名 | ソース | 説明 |
|---|---|---|
| `mcp` | `src/commands/mcp/` | MCP サーバー管理 |
| `ide` | `src/commands/ide/` | IDE 拡張インストール |
| `desktop` | `src/commands/desktop/` | デスクトップアプリ |
| `mobile` | `src/commands/mobile/` | モバイル QR コード |
| `chrome` | `src/commands/chrome/` | Chrome 連携 |
| `install-github-app` | `src/commands/install-github-app/` | GitHub App インストール |
| `install-slack-app` | `src/commands/install-slack-app/` | Slack App インストール |

### 権限・セキュリティ

| コマンド名 | ソース | 説明 |
|---|---|---|
| `permissions` | `src/commands/permissions/` | 権限管理 |
| `sandbox-toggle` | `src/commands/sandbox-toggle/` | サンドボックス切替 |
| `privacy-settings` | `src/commands/privacy-settings/` | プライバシー設定 |
| `hooks` | `src/commands/hooks/` | フック管理 |

### コード操作

| コマンド名 | ソース | 説明 |
|---|---|---|
| `review` | `src/commands/review.ts` | コードレビュー |
| `ultrareview` | `src/commands/review.ts` | 詳細レビュー |
| `security-review` | `src/commands/security-review.ts` | セキュリティレビュー |
| `plan` | `src/commands/plan/` | プランモード切替 |
| `rewind` | `src/commands/rewind/` | 変更巻き戻し |

### 情報・ヘルプ

| コマンド名 | ソース | 説明 |
|---|---|---|
| `help` | `src/commands/help/` | ヘルプ表示 |
| `doctor` | `src/commands/doctor/` | 診断 |
| `status` | `src/commands/status/` | ステータス表示 |
| `release-notes` | `src/commands/release-notes/` | リリースノート |
| `feedback` | `src/commands/feedback/` | フィードバック送信 |
| `btw` | `src/commands/btw/` | クイックノート |
| `stats` | `src/commands/stats/` | 統計情報 |
| `insights` | `src/commands/insights.ts` | 使用レポート (遅延ロード, 113KB) |

### スキル・拡張

| コマンド名 | ソース | 説明 |
|---|---|---|
| `skills` | `src/commands/skills/` | スキル管理 |
| `plugin` | `src/commands/plugin/` | プラグイン管理 |
| `reload-plugins` | `src/commands/reload-plugins/` | プラグインリロード |
| `agents` | `src/commands/agents/` | エージェント管理 |
| `tasks` | `src/commands/tasks/` | タスク管理 |

### その他

| コマンド名 | ソース | 説明 |
|---|---|---|
| `copy` | `src/commands/copy/` | 最後のメッセージをコピー |
| `memory` | `src/commands/memory/` | CLAUDE.md メモリ管理 |
| `export` | `src/commands/export/` | エクスポート |
| `init` | `src/commands/init.ts` | プロジェクト初期化 |
| `stickers` | `src/commands/stickers/` | ステッカー |
| `tag` | `src/commands/tag/` | タグ管理 |
| `terminal-setup` | `src/commands/terminalSetup/` | ターミナルセットアップ |
| `upgrade` | `src/commands/upgrade/` | アップグレード |
| `thinkback` | `src/commands/thinkback/` | 思考振り返り |
| `thinkback-play` | `src/commands/thinkback-play/` | 思考再生 |
| `statusline` | `src/commands/statusline.tsx` | ステータスライン切替 |
| `advisor` | `src/commands/advisor.ts` | アドバイザー |
| `heapdump` | `src/commands/heapdump/` | ヒープダンプ |
| `rate-limit-options` | `src/commands/rate-limit-options/` | レート制限オプション |
| `remote-env` | `src/commands/remote-env/` | リモート環境変数 |
| `pr_comments` | `src/commands/pr_comments/` | PR コメント |
| `summary` | `src/commands/summary/` | 会話サマリー |

### Feature-gated コマンド

| コマンド名 | フィーチャーゲート | 説明 |
|---|---|---|
| `proactive` | `PROACTIVE \|\| KAIROS` | プロアクティブ動作 |
| `brief` | `KAIROS \|\| KAIROS_BRIEF` | ブリーフモード |
| `assistant` | `KAIROS` | アシスタントモード |
| `bridge` | `BRIDGE_MODE` | ブリッジ接続 |
| `remote-control-server` | `DAEMON && BRIDGE_MODE` | リモートコントロールサーバー |
| `voice` | `VOICE_MODE` | 音声モード |
| `force-snip` | `HISTORY_SNIP` | 強制スニップ |
| `workflows` | `WORKFLOW_SCRIPTS` | ワークフロー |
| `remote-setup` | `CCR_REMOTE_SETUP` | リモートセットアップ |
| `subscribe-pr` | `KAIROS_GITHUB_WEBHOOKS` | PR 購読 |
| `ultraplan` | `ULTRAPLAN` | ウルトラプラン |
| `torch` | `TORCH` | Torch |
| `peers` | `UDS_INBOX` | ピア通信 |
| `fork` | `FORK_SUBAGENT` | サブエージェントフォーク |
| `buddy` | `BUDDY` | バディ |

---

## コマンドのロード・ディスパッチフロー

### 1. コマンドロード

`getCommands()` が全コマンドソースを統合する:

```
getCommands(cwd)
  ├── loadAllCommands(cwd) [メモ化]
  │   ├── getSkills(cwd)
  │   │   ├── getSkillDirCommands(cwd)     // .claude/skills/ ディレクトリ
  │   │   ├── getPluginSkills()             // プラグインスキル
  │   │   ├── getBundledSkills()            // バンドル済みスキル
  │   │   └── getBuiltinPluginSkillCommands() // 組み込みプラグインスキル
  │   ├── getPluginCommands()               // プラグインコマンド
  │   ├── getWorkflowCommands(cwd)          // ワークフロー [WORKFLOW_SCRIPTS]
  │   └── COMMANDS()                        // 組み込みコマンド
  ├── getDynamicSkills()                    // ファイル操作で発見されたスキル
  ├── meetsAvailabilityRequirement()        // 認証要件チェック
  └── isCommandEnabled()                    // フィーチャーフラグチェック
```

**ソース**: `src/commands.ts:449-517`

### 2. コマンド検索

```typescript
// src/commands.ts:688-698
function findCommand(commandName: string, commands: Command[]): Command | undefined {
  return commands.find(
    _ => _.name === commandName ||
         getCommandName(_) === commandName ||
         _.aliases?.includes(commandName),
  )
}
```

コマンドは名前、表示名、エイリアスのいずれかで検索される。

### 3. コマンドのキャッシュ管理

```typescript
// src/commands.ts:523-539
function clearCommandMemoizationCaches(): void {
  loadAllCommands.cache?.clear?.()
  getSkillToolCommands.cache?.clear?.()
  getSlashCommandToolSkills.cache?.clear?.()
  clearSkillIndexCache?.()
}

function clearCommandsCache(): void {
  clearCommandMemoizationCaches()
  clearPluginCommandCache()
  clearPluginSkillsCache()
  clearSkillCaches()
}
```

動的スキルが追加された場合やプラグインがリロードされた場合にキャッシュがクリアされる。

---

## リモート・ブリッジ対応

### リモートセーフコマンド

`--remote` モードで使用可能なコマンドは明示的に制限される:

**ソース**: `src/commands.ts:619-637`

```typescript
const REMOTE_SAFE_COMMANDS: Set<Command> = new Set([
  session, exit, clear, help, theme, color, vim, cost,
  usage, copy, btw, feedback, plan, keybindings, statusline,
  stickers, mobile,
])
```

### ブリッジセーフコマンド

モバイル/Web クライアントからのブリッジ経由実行が許可されるコマンド:

**ソース**: `src/commands.ts:651-660`

```typescript
const BRIDGE_SAFE_COMMANDS: Set<Command> = new Set([
  compact, clear, cost, summary, releaseNotes, files,
])
```

ブリッジ経由の安全性判定ルール (`src/commands.ts:672-676`):
- `local-jsx` 型: 常にブロック (Ink UI をレンダリングするため)
- `prompt` 型: 常に許可 (テキスト展開のみ)
- `local` 型: `BRIDGE_SAFE_COMMANDS` に明示的に含まれている場合のみ許可

---

## 遅延ロードパターン

大きなモジュールは遅延ロードで初期化コストを低減:

```typescript
// src/commands.ts:190-202 (insights.ts は 113KB, 3200行)
const usageReport: Command = {
  type: 'prompt',
  name: 'insights',
  async getPromptForCommand(args, context) {
    const real = (await import('./commands/insights.js')).default
    if (real.type !== 'prompt') throw new Error('unreachable')
    return real.getPromptForCommand(args, context)
  },
}
```

`local-jsx` 型コマンドも `load()` メソッドで遅延ロードを標準サポート:

```typescript
type LocalJSXCommand = {
  type: 'local-jsx'
  load: () => Promise<LocalJSXCommandModule>
}
```

---

## スキルツール統合

### モデル呼び出し可能なコマンド

`getSkillToolCommands()` はモデルが呼び出せるコマンドをフィルタリング:

**ソース**: `src/commands.ts:563-581`

条件:
- `type === 'prompt'`
- `disableModelInvocation` が false
- `source !== 'builtin'`
- `loadedFrom` が `bundled`, `skills`, `commands_DEPRECATED` のいずれか、または `hasUserSpecifiedDescription` / `whenToUse` が設定済み

### MCP スキルコマンド

**ソース**: `src/commands.ts:547-559`

```typescript
function getMcpSkillCommands(mcpCommands: readonly Command[]): readonly Command[] {
  if (feature('MCP_SKILLS')) {
    return mcpCommands.filter(cmd =>
      cmd.type === 'prompt' &&
      cmd.loadedFrom === 'mcp' &&
      !cmd.disableModelInvocation,
    )
  }
  return []
}
```
