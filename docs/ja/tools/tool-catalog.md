# ツールカタログ - Claude Code v2.1.88

## 概要

Claude Codeは`src/tools/`ディレクトリに40のツールディレクトリと共有ユーティリティを持つ。各ツールは`buildTool()`関数を通じて`Tool`型として登録され、統一的なインターフェースでモデルから呼び出される。

---

## ツール一覧（全40ツール）

### ファイル操作ツール（5ツール）

| ツール名 | ディレクトリ | 概要 |
|---|---|---|
| **FileReadTool** | `src/tools/FileReadTool/` | ファイル内容の読み取り。行番号付き出力、オフセット/リミット指定、画像・PDF・Jupyterノートブック対応。`maxResultSizeChars`はInfinity（永続化による循環参照を防止） |
| **FileWriteTool** | `src/tools/FileWriteTool/` | ファイルへの書き込み。新規作成または完全上書き。既存ファイルは事前にReadツールでの読み取りが必須 |
| **FileEditTool** | `src/tools/FileEditTool/` | 既存ファイル内の文字列置換による編集。差分のみ送信するため効率的。`old_string`のユニーク性検証、`replace_all`オプション対応 |
| **GlobTool** | `src/tools/GlobTool/` | ファイルパターンマッチング検索。`**/*.js`等のGlobパターンで高速にファイルを検索。結果は更新日時順 |
| **GrepTool** | `src/tools/GrepTool/` | ripgrep(rg)ベースのコンテンツ検索。正規表現、ファイルタイプフィルタ、コンテキスト行数指定、複数出力モード対応 |

### コード実行ツール（4ツール）

| ツール名 | ディレクトリ | 概要 |
|---|---|---|
| **BashTool** | `src/tools/BashTool/` | Bashコマンドの実行。最も複雑なツール（636KB）。サンドボックス、AST解析、権限管理、バックグラウンド実行、タイムアウト制御を包含。詳細は`bash-tool-security.md`を参照 |
| **PowerShellTool** | `src/tools/PowerShellTool/` | Windows環境でのPowerShellコマンド実行。BashToolのWindows版に相当 |
| **REPLTool** | `src/tools/REPLTool/` | 対話型REPL（Read-Eval-Print Loop）環境。透過ラッパー（`isTransparentWrapper`）として内部ツール呼び出しを委譲 |
| **NotebookEditTool** | `src/tools/NotebookEditTool/` | Jupyterノートブック(.ipynb)のセル編集。遅延ロード対象（`shouldDefer`） |

### AI・エージェントツール（3ツール）

| ツール名 | ディレクトリ | 概要 |
|---|---|---|
| **AgentTool** | `src/tools/AgentTool/` | サブエージェントの起動と管理。`src/tools/AgentTool/loadAgentsDir.js`からエージェント定義を読み込み、独立したコンテキストでタスクを実行 |
| **AskUserQuestionTool** | `src/tools/AskUserQuestionTool/` | ユーザーへの質問を投げかけ回答を待つ。`requiresUserInteraction()`がtrueを返す |
| **SendMessageTool** | `src/tools/SendMessageTool/` | ユーザーへのメッセージ送信。質問ではなく情報提供用 |

### 外部統合ツール（5ツール）

| ツール名 | ディレクトリ | 概要 |
|---|---|---|
| **MCPTool** | `src/tools/MCPTool/` | Model Context Protocolサーバーとの通信。`mcpInfo`プロパティでサーバー名・ツール名を保持。`isMcp`フラグがtrue |
| **WebSearchTool** | `src/tools/WebSearchTool/` | Web検索の実行。遅延ロード対象 |
| **WebFetchTool** | `src/tools/WebFetchTool/` | WebページのURL取得。遅延ロード対象 |
| **RemoteTriggerTool** | `src/tools/RemoteTriggerTool/` | リモートシステムへのトリガー送信 |
| **LSPTool** | `src/tools/LSPTool/` | Language Server Protocol連携。`isLsp`フラグがtrue |

### MCP関連ツール（3ツール）

| ツール名 | ディレクトリ | 概要 |
|---|---|---|
| **ListMcpResourcesTool** | `src/tools/ListMcpResourcesTool/` | MCPサーバーのリソース一覧取得 |
| **ReadMcpResourceTool** | `src/tools/ReadMcpResourceTool/` | MCPリソースの内容読み取り |
| **McpAuthTool** | `src/tools/McpAuthTool/` | MCPサーバーへの認証処理 |

### タスク管理ツール（6ツール）

| ツール名 | ディレクトリ | 概要 |
|---|---|---|
| **TaskCreateTool** | `src/tools/TaskCreateTool/` | バックグラウンドタスクの作成。`LocalShellTask`と連携 |
| **TaskUpdateTool** | `src/tools/TaskUpdateTool/` | タスクの状態更新 |
| **TaskListTool** | `src/tools/TaskListTool/` | タスク一覧の取得 |
| **TaskGetTool** | `src/tools/TaskGetTool/` | 特定タスクの詳細取得 |
| **TaskStopTool** | `src/tools/TaskStopTool/` | 実行中タスクの停止 |
| **TaskOutputTool** | `src/tools/TaskOutputTool/` | タスク出力の読み取り |

### チーム管理ツール（2ツール）

| ツール名 | ディレクトリ | 概要 |
|---|---|---|
| **TeamCreateTool** | `src/tools/TeamCreateTool/` | チームの作成 |
| **TeamDeleteTool** | `src/tools/TeamDeleteTool/` | チームの削除 |

### ワークツリー管理ツール（2ツール）

| ツール名 | ディレクトリ | 概要 |
|---|---|---|
| **EnterWorktreeTool** | `src/tools/EnterWorktreeTool/` | Gitワークツリーへの進入 |
| **ExitWorktreeTool** | `src/tools/ExitWorktreeTool/` | Gitワークツリーからの退出 |

### 設定・モード管理ツール（4ツール）

| ツール名 | ディレクトリ | 概要 |
|---|---|---|
| **ConfigTool** | `src/tools/ConfigTool/` | 設定の読み取り・変更 |
| **EnterPlanModeTool** | `src/tools/EnterPlanModeTool/` | プランモードへの移行。`ToolPermissionContext.prePlanMode`に元のモードを保存 |
| **ExitPlanModeTool** | `src/tools/ExitPlanModeTool/` | プランモードからの復帰 |
| **SkillTool** | `src/tools/SkillTool/` | スキルの検索・呼び出し。`dynamicSkillDirTriggers`と`discoveredSkillNames`で管理 |

### スケジュール管理ツール（1ツール）

| ツール名 | ディレクトリ | 概要 |
|---|---|---|
| **ScheduleCronTool** | `src/tools/ScheduleCronTool/` | 定期実行ジョブのスケジューリング |

### ユーティリティツール（5ツール）

| ツール名 | ディレクトリ | 概要 |
|---|---|---|
| **SleepTool** | `src/tools/SleepTool/` | 指定時間の待機 |
| **TodoWriteTool** | `src/tools/TodoWriteTool/` | ToDoリストの書き込み・更新。遅延ロード対象 |
| **ToolSearchTool** | `src/tools/ToolSearchTool/` | 遅延ロードツールの検索・発見。キーワードベースで`searchHint`を使ったマッチング |
| **BriefTool** | `src/tools/BriefTool/` | 簡潔な出力モード制御 |
| **SyntheticOutputTool** | `src/tools/SyntheticOutputTool/` | 合成出力の生成（内部利用） |

---

## 共有リソース

| パス | 概要 |
|---|---|
| `src/tools/shared/` | ツール間で共有されるユーティリティ（例: `gitOperationTracking.ts`） |
| `src/tools/testing/` | ツールテスト用ユーティリティ |
| `src/tools/utils.ts` | ツール汎用ユーティリティ関数 |

---

## カテゴリ別統計

| カテゴリ | ツール数 | 主な特徴 |
|---|---|---|
| ファイル操作 | 5 | 読み取り専用操作が多く、並行実行安全 |
| コード実行 | 4 | セキュリティが最重要、サンドボックス利用 |
| AI・エージェント | 3 | ユーザー対話やサブエージェント管理 |
| 外部統合 | 5 | MCP/LSP/Web等の外部プロトコル連携 |
| MCP関連 | 3 | MCPリソース管理専用 |
| タスク管理 | 6 | バックグラウンドタスクのライフサイクル管理 |
| チーム管理 | 2 | チーム操作 |
| ワークツリー管理 | 2 | Git ワークツリー操作 |
| 設定・モード | 4 | 動作モードと設定の制御 |
| スケジュール | 1 | 定期実行ジョブ |
| ユーティリティ | 5 | 汎用的な補助ツール |

---

## ツールの共通パターン

### 1. buildTool()による登録

全ツールは`buildTool()`を使って`Tool`型のインスタンスを生成する:

```typescript
// 典型的なツール定義パターン
export const MyTool = buildTool({
  name: 'MyTool',
  inputSchema: myInputSchema,
  // デフォルト値で十分なメソッドは省略可能:
  // isEnabled, isConcurrencySafe, isReadOnly, isDestructive,
  // checkPermissions, toAutoClassifierInput, userFacingName
  call: async (args, context, canUseTool, parentMessage, onProgress) => {
    // ツール実行ロジック
    return { data: result }
  },
  // ... 必須メソッド
})
```

### 2. 遅延ロードパターン

頻繁に使用されないツール（NotebookEdit、TodoWrite、WebFetch、WebSearch等）は`shouldDefer: true`を設定し、初期プロンプトサイズを削減する。モデルは`ToolSearchTool`を使ってこれらを検索・ロードしてから利用する。

### 3. 読み取り専用ツールの特性

読み取り専用ツール（FileRead、Glob、Grep等）は:
- `isReadOnly()`が`true`を返す
- `isConcurrencySafe()`が`true`を返す（並行実行可能）
- `checkPermissions()`はパス検証のみ（書き込み権限は不要）

### 4. UI表現

各ツールは豊富なUI表現メソッドを持ち、以下の状態に応じた表示を行う:
- **使用開始時**: `renderToolUseMessage()`（部分入力対応）
- **実行中**: `renderToolUseProgressMessage()`
- **完了時**: `renderToolResultMessage()`
- **拒否時**: `renderToolUseRejectedMessage()`
- **エラー時**: `renderToolUseErrorMessage()`
- **キュー待ち**: `renderToolUseQueuedMessage()`

### 5. セキュリティ分類

`toAutoClassifierInput()`メソッドにより、自動モードのセキュリティ分類器にコマンドのコンパクト表現を提供する。セキュリティ関連のツールはこれを必ずオーバーライドする。

---

## MCPツールの特殊性

MCPツールは外部サーバーから動的に登録され、以下の特殊プロパティを持つ:

- `isMcp: true` フラグ
- `mcpInfo: { serverName, toolName }` - 元のサーバー名・ツール名
- `inputJSONSchema` - Zodではなく直接JSON Schema形式
- `alwaysLoad` - `_meta['anthropic/alwaysLoad']`で制御可能
- ツール名は`mcp__serverName__toolName`形式（`CLAUDE_AGENT_SDK_MCP_NO_PREFIX`モードではプレフィックスなし）
