# Claude Code v2.1.88 アーキテクチャ概要

## プロジェクト全体像

Claude Code は Anthropic が開発した CLI ベースの AI コーディングアシスタントである。TypeScript で記述され、React/Ink を用いたターミナル UI を持つ。

| 項目 | 値 |
|------|-----|
| バージョン | 2.1.88 |
| 言語 | TypeScript (ES2022ターゲット) |
| UIフレームワーク | React + Ink (ターミナルUI) |
| ランタイム | Bun (本番) / Node.js >= 18 (フォールバック) |
| パッケージ名 | `@anthropic-ai/claude-code-source` |
| モジュール形式 | ESM (`"type": "module"`) |

---

## ディレクトリ構成

```
claude-code-source-code/
├── src/                    # メインソースコード
│   ├── entrypoints/        # エントリポイント (cli.tsx)
│   ├── cli/                # CLIコマンドローダー
│   ├── main.tsx            # メインUIループ・Commander定義
│   ├── QueryEngine.ts      # クエリエンジン（会話ループ制御）
│   ├── query.ts            # API呼び出し・ストリーム処理
│   ├── commands/           # スラッシュコマンド (~189ファイル)
│   ├── components/         # React/Ink UIコンポーネント (~389ファイル)
│   ├── hooks/              # React Hooks (~104ファイル)
│   ├── tools/              # ツール定義 (42ディレクトリ)
│   ├── services/           # サービス層 (20サブディレクトリ)
│   ├── utils/              # ユーティリティ (~564ファイル)
│   ├── types/              # 型定義
│   ├── state/              # 状態管理 (AppState, Store)
│   ├── bootstrap/          # 起動時状態初期化
│   ├── constants/          # 定数・プロンプト定義
│   ├── context/            # コンテキスト管理
│   ├── migrations/         # 設定マイグレーション
│   ├── plugins/            # プラグインシステム
│   ├── skills/             # スキルシステム
│   ├── assistant/          # Kairos アシスタントモード
│   ├── coordinator/        # コーディネーターモード
│   ├── bridge/             # ブリッジ/リモートコントロール
│   ├── daemon/             # デーモンプロセス
│   ├── remote/             # リモートセッション管理
│   ├── server/             # ダイレクトコネクト
│   ├── ink/                # Ink ターミナル描画拡張
│   ├── memdir/             # メモリディレクトリ
│   ├── voice/              # 音声モード
│   └── vim/                # Vim キーバインド
├── scripts/                # ビルドスクリプト
├── stubs/                  # Bun intrinsicsのスタブ
├── types/                  # 追加型定義
└── dist/                   # ビルド出力
```

### 各ディレクトリの役割

- **`src/entrypoints/`**: プロセスの最初の進入点。`cli.tsx`が `--version`等のファストパスを処理し、それ以外は `main.tsx` にディスパッチする。
- **`src/tools/`**: LLM が呼び出せるツール群。`BashTool`, `FileEditTool`, `FileReadTool`, `GrepTool`, `AgentTool`, `WebFetchTool` 等42種。各ツールは独立ディレクトリにカプセル化されている。
- **`src/commands/`**: ユーザーが `/` プレフィクスで呼び出すスラッシュコマンド。`/init`, `/mcp`, `/context`, `/bridge` 等。
- **`src/services/`**: 外部APIとの通信やバックグラウンド処理。`analytics/` (GrowthBook), `api/` (Anthropic API), `mcp/` (Model Context Protocol), `compact/` (コンテキスト圧縮), `lsp/` (Language Server Protocol) 等。
- **`src/components/`**: Ink ベースのターミナル UI コンポーネント群。
- **`src/hooks/`**: React Hooks。`useCanUseTool`, `useGlobalKeybindings`, `useVoiceIntegration` 等。
- **`src/utils/`**: 汎用ユーティリティ。認証(`auth.ts`), 設定(`config.ts`), Git操作(`git.ts`), モデル管理(`model/`), パーミッション(`permissions/`), プラグイン(`plugins/`) 等。
- **`src/state/`**: アプリケーション状態管理。`AppStateStore.ts` で状態の定義、`store.ts` でストア生成。
- **`stubs/`**: Bun ランタイム固有の API (`bun:bundle`, `bun:ffi`) のスタブ。Node.js 環境でのビルド互換性を提供。

---

## エントリポイントからの起動フロー

```
cli.tsx (エントリポイント)
  │
  ├─ ファストパス処理
  │   ├─ --version → MACRO.VERSION を出力して終了
  │   ├─ --daemon-worker → daemon/workerRegistry.js
  │   ├─ remote-control → bridge/bridgeMain.js
  │   ├─ daemon → daemon/main.js
  │   └─ ps/logs/attach/kill → cli/bg.js
  │
  └─ 通常パス → main.tsx をインポート
       │
       ├─ 副作用実行（起動プロファイリング、MDM読み込み、Keychain先読み）
       │   profileCheckpoint('main_tsx_entry')   ← L9-12
       │   startMdmRawRead()                     ← L15-16
       │   startKeychainPrefetch()               ← L19-20
       │
       ├─ Commander.js によるCLIオプション定義
       │
       └─ action handler
            │
            ├─ init() — 初期化処理
            ├─ GrowthBook初期化
            ├─ 設定読み込み・マイグレーション
            ├─ ツール/コマンド/MCP/プラグイン読み込み
            │
            └─ launchRepl() → REPL.tsx
                 │
                 └─ QueryEngine.ts (会話ループ)
                      │
                      └─ query.ts (API呼び出し)
                           │
                           ├─ システムプロンプト構築
                           ├─ Anthropic API ストリーミング
                           ├─ ツール実行 (StreamingToolExecutor)
                           └─ ストップフック処理
```

### 起動フローの詳細

#### 1. `src/entrypoints/cli.tsx` (エントリポイント)

プロセスの最初の入口。Bun の `feature()` を用いたコンパイル時分岐により、外部ビルドでは不要なコードパスが除去される。

```
// L1: feature() のインポート
import { feature } from 'bun:bundle';

// L33-42: main() 関数 — ファストパス分岐
async function main(): Promise<void> {
  // --version は即座に応答（モジュール読み込みゼロ）
  // その他は動的 import で main.tsx を読み込む
}
```

主なファストパス:
- `--version` / `-v`: モジュール読み込みなしで即応答 (L37-42)
- `--daemon-worker`: デーモンワーカー起動 (L100-106)
- `remote-control` / `bridge`: ブリッジモード (L112-162)
- `daemon`: デーモン本体 (L165-180)
- `ps` / `logs` / `attach` / `kill`: バックグラウンドセッション管理 (L185+)

#### 2. `src/main.tsx` (メインUIループ)

Commander.js による CLI オプション定義と、メインの action handler を含む大規模ファイル。起動時に以下の副作用を実行する:

- **プロファイリング開始** (`profileCheckpoint('main_tsx_entry')`, L12)
- **MDM設定の先読み** (`startMdmRawRead()`, L16) — macOS の `plutil`/Windows の `reg query` をバックグラウンドで並列実行
- **Keychain先読み** (`startKeychainPrefetch()`, L20) — OAuth/APIキーの読み込みを並列化（macOS起動時 ~65ms短縮）

#### 3. `src/QueryEngine.ts` (クエリエンジン)

会話のメインループを管理するクラス。以下の責務を持つ:
- ユーザー入力の処理 (`processUserInput`)
- API クエリの発行 (`query()` 呼び出し)
- ツール実行結果の処理
- セッション記録 (`recordTranscript`)
- コスト追跡 (`cost-tracker.ts`)
- ファイル状態キャッシュ (`fileStateCache`)
- 自動コンパクション制御

#### 4. `src/query.ts` (APIクエリ)

Anthropic API への実際のリクエストを行う。以下を処理する:
- システムプロンプトの構築
- メッセージの正規化
- ストリーミングレスポンスの処理
- ツール実行のオーケストレーション (`StreamingToolExecutor`, `runTools`)
- エラーハンドリング・リトライ
- トークン使用量の追跡
- 自動コンパクション (`autoCompact`)
- リアクティブコンパクション (`reactiveCompact` — フィーチャーゲート付き)
- コンテキストコラプス (`contextCollapse` — フィーチャーゲート付き)

---

## レイヤードアーキテクチャ

```
┌─────────────────────────────────────────────────┐
│                   CLI Layer                       │
│   entrypoints/cli.tsx → main.tsx (Commander.js)  │
├─────────────────────────────────────────────────┤
│                 Commands Layer                    │
│   /init, /mcp, /context, /bridge, ...           │
│   (~189ファイル, スラッシュコマンド)                │
├─────────────────────────────────────────────────┤
│               QueryEngine Layer                  │
│   QueryEngine.ts — 会話ループ制御                 │
│   query.ts — APIリクエスト・ストリーム             │
├─────────────────────────────────────────────────┤
│                 Tools Layer                       │
│   BashTool, FileEditTool, AgentTool, ...        │
│   (42ツールディレクトリ)                           │
├─────────────────────────────────────────────────┤
│               Services Layer                     │
│   api/ (Anthropic API), mcp/ (MCP),             │
│   analytics/ (GrowthBook), compact/,            │
│   lsp/, oauth/, plugins/, ...                   │
│   (20サブディレクトリ)                             │
├─────────────────────────────────────────────────┤
│            External APIs / Systems               │
│   Anthropic API, GrowthBook, MCP Servers,       │
│   Git, LSP Servers, File System                 │
└─────────────────────────────────────────────────┘
```

---

## 主要サブシステム一覧

### ツール (42ディレクトリ)

| カテゴリ | ツール |
|---------|-------|
| ファイル操作 | `FileReadTool`, `FileEditTool`, `FileWriteTool`, `GlobTool`, `GrepTool` |
| コマンド実行 | `BashTool`, `PowerShellTool`, `REPLTool` |
| エージェント | `AgentTool`, `SendMessageTool`, `TeamCreateTool`, `TeamDeleteTool` |
| タスク管理 | `TaskCreateTool`, `TaskGetTool`, `TaskListTool`, `TaskUpdateTool`, `TaskStopTool`, `TaskOutputTool` |
| MCP連携 | `MCPTool`, `McpAuthTool`, `ListMcpResourcesTool`, `ReadMcpResourceTool` |
| その他 | `WebFetchTool`, `WebSearchTool`, `NotebookEditTool`, `TodoWriteTool`, `SleepTool`, `ToolSearchTool`, `SkillTool` |
| ワークフロー | `EnterPlanModeTool`, `ExitPlanModeTool`, `EnterWorktreeTool`, `ExitWorktreeTool` |

### サービス (20サブディレクトリ)

`analytics/`, `api/`, `compact/`, `extractMemories/`, `lsp/`, `mcp/`, `oauth/`, `plugins/`, `policyLimits/`, `remoteManagedSettings/`, `settingsSync/`, `teamMemorySync/`, `tips/`, `tools/`, `toolUseSummary/`, `AgentSummary/`, `MagicDocs/`, `PromptSuggestion/`, `SessionMemory/`, `autoDream/`

### コンポーネント (~389ファイル)

React/Ink によるターミナル UI コンポーネント群。メッセージ表示、入力フォーム、権限ダイアログ、設定画面等。

### Hooks (~104ファイル)

React Hooks パターンによる状態管理・副作用処理。`useCanUseTool`, `useGlobalKeybindings`, `useVoiceIntegration`, `useAssistantHistory`, `useAutoModeUnavailableNotification` 等。

### ユーティリティ (~564ファイル)

認証 (`auth.ts`), 設定管理 (`config.ts`, `settings/`), Git操作 (`git.ts`), モデル管理 (`model/`), パーミッション (`permissions/`), プラグイン (`plugins/`), セッション管理 (`sessionStorage.ts`), ripgrep連携 (`ripgrep.ts`) 等。

---

## 技術的特徴

### パフォーマンス最適化
- **起動時の並列化**: MDM設定読み込みとKeychain先読みを `import` 前に開始（`main.tsx` L13-20）
- **ファストパス**: `--version` はモジュール読み込みゼロで応答
- **動的インポート**: フィーチャーゲートによるコード除去とレイジーローディングの併用
- **プロファイリング**: `startupProfiler.js` による起動パフォーマンス計測

### 設定管理
- **MDM (Mobile Device Management)**: エンタープライズ環境での一括設定
- **GrowthBook**: ランタイムフィーチャーフラグ
- **Policy Limits**: 組織ポリシーによる機能制限
- **Remote Managed Settings**: リモート設定管理

### 拡張性
- **MCP (Model Context Protocol)**: 外部ツールサーバーとの連携
- **プラグインシステム**: バンドルプラグインとバージョン管理プラグイン
- **スキルシステム**: ビルトインスキルとMCPスキル
- **マイグレーション**: `migrations/` ディレクトリに設定マイグレーション群（モデル名変更、設定キー移行等）
