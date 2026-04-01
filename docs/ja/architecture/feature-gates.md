# フィーチャーゲートシステム

Claude Code v2.1.88 のフィーチャーゲートシステムは、コンパイル時のコード除去とランタイムフィーチャーフラグの二層構造で機能管理を実現している。

## 概要

フィーチャーゲートは以下の2つのメカニズムから構成される:

1. **`feature()` 関数によるコンパイル時コード除去** - ビルド時にデッドコードを完全に除去
2. **GrowthBook によるランタイムフィーチャーフラグ** - 実行時に動的に機能を切り替え

---

## feature() 関数によるコンパイル時コード除去

### 仕組み

`feature()` 関数は `bun:bundle` モジュールからインポートされるビルド時定数評価関数である。

```typescript
import { feature } from 'bun:bundle'
```

**ソース**: `src/commands.ts:59`, `src/entrypoints/cli.tsx:18`

Bun のバンドラーがビルド時に `feature('FLAG_NAME')` を `true` または `false` のリテラルに置換し、JavaScript エンジンのデッドコード除去 (DCE) により、`false` に評価されたブランチのコードがバンドルから完全に除去される。

### 重要な制約

`feature()` はインラインで呼び出す必要がある。変数に格納してから条件分岐に使用すると、バンドラーが最適化できない。

```typescript
// 正しい: インライン呼び出し
if (feature('DAEMON') && args[0] === '--daemon-worker') { ... }

// コメントで明示的に注意喚起されている箇所:
// src/entrypoints/cli.tsx:110
// "feature() must stay inline for build-time dead code elimination"
```

### 条件付きインポートパターン

モジュール全体をゲート付きで読み込む標準パターン:

```typescript
// src/commands.ts:62-123
const proactive =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('./commands/proactive.js').default
    : null

const bridge = feature('BRIDGE_MODE')
  ? require('./commands/bridge/index.js').default
  : null
```

`feature()` が `false` と評価された場合、`require()` 呼び出し自体がバンドルに含まれず、依存モジュールツリー全体が除去される。

---

## 主要なフィーチャーゲート一覧

ソースコード全体から検出されたフィーチャーゲートを機能カテゴリ別に分類する。

### コアインフラストラクチャ

| フィーチャーゲート | 説明 | 主な使用箇所 |
|---|---|---|
| `DAEMON` | デーモンプロセスモード | `src/entrypoints/cli.tsx:100,165` |
| `BRIDGE_MODE` | リモートコントロール/ブリッジ接続 | `src/entrypoints/cli.tsx:112`, `src/bridge/bridgeEnabled.ts` |
| `BG_SESSIONS` | バックグラウンドセッション (ps/logs/attach/kill) | `src/entrypoints/cli.tsx:185` |
| `DIRECT_CONNECT` | 直接接続機能 | `src/main.tsx:548,612` |
| `SSH_REMOTE` | SSH リモート機能 | `src/main.tsx:577,706` |
| `CCR_MIRROR` | CCR ミラーリング | `src/bridge/remoteBridgeCore.ts:732` |
| `CCR_AUTO_CONNECT` | CCR 自動接続 | `src/bridge/bridgeEnabled.ts:186` |
| `CCR_REMOTE_SETUP` | リモートセットアップ | `src/commands.ts:91` |
| `HARD_FAIL` | ハードフェイルモード | `src/main.tsx:3870` |

### AI アシスタント・プロアクティブ

| フィーチャーゲート | 説明 | 主な使用箇所 |
|---|---|---|
| `KAIROS` | アシスタントモード (総合) | `src/commands.ts:63,67,70`, `src/main.tsx` 多数 |
| `KAIROS_BRIEF` | ブリーフモード | `src/commands.ts:67`, `src/main.tsx:1728` |
| `KAIROS_CHANNELS` | チャンネル通知 | `src/services/mcp/channelNotification.ts` |
| `KAIROS_GITHUB_WEBHOOKS` | GitHub Webhook 連携 | `src/commands.ts:101` |
| `PROACTIVE` | プロアクティブ機能 | `src/commands.ts:63`, `src/screens/REPL.tsx:194` |
| `AGENT_TRIGGERS` | エージェントトリガー | `src/screens/REPL.tsx:199` |

### セキュリティ・権限

| フィーチャーゲート | 説明 | 主な使用箇所 |
|---|---|---|
| `TRANSCRIPT_CLASSIFIER` | トランスクリプト分類器 (自動モード) | `src/utils/permissions/permissions.ts:59`, `src/main.tsx:337` |
| `BASH_CLASSIFIER` | Bash コマンド分類器 | `src/tools/BashTool/bashPermissions.ts` 多数 |
| `TREE_SITTER_BASH_SHADOW` | Tree-sitter Bash シャドウパーサー | `src/tools/BashTool/bashPermissions.ts:1683` |

### ツール・UI

| フィーチャーゲート | 説明 | 主な使用箇所 |
|---|---|---|
| `VOICE_MODE` | 音声モード | `src/commands.ts:82`, `src/voice/voiceModeEnabled.ts` |
| `WORKFLOW_SCRIPTS` | ワークフロースクリプト | `src/commands.ts:86`, `src/commands.ts:401` |
| `WEB_BROWSER_TOOL` | Web ブラウザツール | `src/screens/REPL.tsx:272` |
| `CHICAGO_MCP` | コンピュータ使用 MCP | `src/entrypoints/cli.tsx:86`, `src/services/mcp/client.ts` |
| `MCP_SKILLS` | MCP スキル | `src/commands.ts:550`, `src/services/mcp/client.ts` |
| `COORDINATOR_MODE` | コーディネーターモード | `src/main.tsx:76`, `src/screens/REPL.tsx:119` |
| `MESSAGE_ACTIONS` | メッセージアクション | `src/screens/REPL.tsx:606` |

### 実験的・その他

| フィーチャーゲート | 説明 | 主な使用箇所 |
|---|---|---|
| `TEMPLATES` | テンプレート機能 | `src/entrypoints/cli.tsx:212` |
| `BYOC_ENVIRONMENT_RUNNER` | BYOC 環境ランナー | `src/entrypoints/cli.tsx:226` |
| `SELF_HOSTED_RUNNER` | セルフホストランナー | `src/entrypoints/cli.tsx:238` |
| `ULTRAPLAN` | ウルトラプラン | `src/commands.ts:104` |
| `TORCH` | Torch | `src/commands.ts:107` |
| `UDS_INBOX` | Unix ドメインソケット受信箱 | `src/tools/SendMessageTool/SendMessageTool.ts` |
| `FORK_SUBAGENT` | サブエージェントフォーク | `src/commands.ts:113` |
| `BUDDY` | バディ機能 | `src/commands.ts:118` |
| `EXPERIMENTAL_SKILL_SEARCH` | 実験的スキル検索 | `src/tools/SkillTool/SkillTool.ts` |
| `HISTORY_SNIP` | 履歴スニップ | `src/commands.ts:83` |
| `CONTEXT_COLLAPSE` | コンテキスト折りたたみ | `src/commands/context/context.tsx:20` |
| `ABLATION_BASELINE` | アブレーションベースライン | `src/entrypoints/cli.tsx:21` |
| `DUMP_SYSTEM_PROMPT` | システムプロンプトダンプ | `src/entrypoints/cli.tsx:53` |
| `EXTRACT_MEMORIES` | メモリ抽出 | `src/query/stopHooks.ts:42` |
| `COMMIT_ATTRIBUTION` | コミット帰属 | `src/commands/clear/caches.ts:105` |
| `REACTIVE_COMPACT` | リアクティブコンパクション | `src/commands/compact/compact.ts:35` |
| `PROMPT_CACHE_BREAK_DETECTION` | プロンプトキャッシュブレイク検出 | `src/commands/compact/compact.ts:67` |
| `VERIFICATION_AGENT` | 検証エージェント | `src/tools/TodoWriteTool/TodoWriteTool.ts:78` |
| `LODESTONE` | Lodestone | `src/main.tsx:647` |
| `TEAMMEM` | チームメモリ同期 | `src/services/teamMemorySync/watcher.ts:253` |
| `AWAY_SUMMARY` | 離席サマリー | `src/screens/REPL.tsx:1246` |
| `DOWNLOAD_USER_SETTINGS` | ユーザー設定ダウンロード | `src/commands/reload-plugins/reload-plugins.ts:25` |
| `UPLOAD_USER_SETTINGS` | ユーザー設定アップロード | `src/main.tsx:963` |
| `NEW_INIT` | 新しい init フロー | `src/commands/init.ts:230` |

---

## GrowthBook によるランタイムフィーチャーフラグ

### アーキテクチャ

GrowthBook は `@growthbook/growthbook` SDK を通じて統合されている。

**ソース**: `src/services/analytics/growthbook.ts`

```typescript
import { GrowthBook } from '@growthbook/growthbook'
```

### ユーザー属性 (ターゲティング)

GrowthBook にはユーザー属性が送信され、フィーチャーフラグのターゲティングに使用される:

```typescript
// src/services/analytics/growthbook.ts:32-47
type GrowthBookUserAttributes = {
  id: string
  sessionId: string
  deviceID: string
  platform: 'win32' | 'darwin' | 'linux'
  apiBaseUrlHost?: string
  organizationUUID?: string
  accountUUID?: string
  userType?: string
  subscriptionType?: string
  rateLimitTier?: string
  firstTokenTime?: number
  email?: string
  appVersion?: string
  github?: GitHubActionsMetadata
}
```

### API 関数

ランタイムフィーチャーフラグの取得には複数の関数が提供される:

| 関数名 | 動作 |
|---|---|
| `getFeatureValue_CACHED_MAY_BE_STALE()` | キャッシュ値を返す (古い可能性あり) |
| `getFeatureValue_CACHED_WITH_REFRESH()` | キャッシュ値を返しつつバックグラウンドでリフレッシュ |
| `checkStatsigFeatureGate_CACHED_MAY_BE_STALE()` | ゲートチェック (キャッシュ) |
| `checkSecurityRestrictionGate()` | セキュリティ制限ゲート |
| `getDynamicConfig_BLOCKS_ON_INIT()` | 動的設定 (初期化完了を待機) |

### 使用例

```typescript
// src/tools/BashTool/shouldUseSandbox.ts:1
import { getFeatureValue_CACHED_MAY_BE_STALE } from 'src/services/analytics/growthbook.js'

// サンドボックスで無効化されたコマンドの動的設定
const disabledCommands = getFeatureValue_CACHED_MAY_BE_STALE<{
  commands: string[]
  substrings: string[]
}>('tengu_sandbox_disabled_commands', { commands: [], substrings: [] })
```

### 実験のトラッキング

GrowthBook はフィーチャーフラグだけでなく A/B テストにも使用される。実験データはファーストパーティのイベントロガーに記録される:

```typescript
// src/services/analytics/growthbook.ts:70-77
type StoredExperimentData = {
  experimentId: string
  variationId: number
  inExperiment?: boolean
  hashAttribute?: string
  hashValue?: string
}
```

---

## 環境変数による制御

### ビルドタイプの制御

`USER_TYPE` 環境変数が内部ビルドと外部ビルドを区別する:

```typescript
// src/commands.ts:49-51
const agentsPlatform =
  process.env.USER_TYPE === 'ant'
    ? require('./commands/agents-platform/index.js').default
    : null
```

内部専用コマンド (`INTERNAL_ONLY_COMMANDS`) は `USER_TYPE === 'ant'` かつ `IS_DEMO` が未設定の場合のみ有効:

```typescript
// src/commands.ts:343-345
...(process.env.USER_TYPE === 'ant' && !process.env.IS_DEMO
  ? INTERNAL_ONLY_COMMANDS
  : []),
```

### 主要な環境変数

| 環境変数 | 用途 |
|---|---|
| `USER_TYPE` | ユーザータイプ (`ant` = 内部) |
| `IS_DEMO` | デモモード |
| `CLAUDE_CODE_ABLATION_BASELINE` | アブレーションベースライン有効化 |
| `CLAUDE_CODE_HOST_PLATFORM` | ホストプラットフォームの上書き |
| `CLAUDE_CODE_COORDINATOR_MODE` | コーディネーターモード有効化 |
| `CLAUDE_CONFIG_DIR` | 設定ディレクトリの上書き |

---

## コンパイル時とランタイムの使い分け

| 観点 | `feature()` (コンパイル時) | GrowthBook (ランタイム) |
|---|---|---|
| 評価タイミング | ビルド時 | 実行時 |
| コード除去 | バンドルから完全除去 | 条件分岐として残留 |
| 変更の反映 | 再ビルドが必要 | サーバー側の設定変更で即時反映 |
| 用途 | 内部/外部ビルドの分離、大規模機能の除去 | A/B テスト、段階的ロールアウト、動的設定 |
| パフォーマンス影響 | なし (コード自体が存在しない) | 軽微 (ネットワーク呼び出し + キャッシュ) |

この二層構造により、セキュリティ上重要なコードの完全な除去と、運用上の柔軟性の両方を実現している。
