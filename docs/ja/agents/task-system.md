# タスクシステム

Claude Code v2.1.88 のタスクシステムは、バックグラウンドで実行される各種処理（シェルコマンド、サブエージェント、リモートセッション等）を統一的に管理するフレームワークである。

## アーキテクチャ概要

タスクシステムは以下の3層構造で構成される。

1. **タスク定義層** (`src/Task.ts`) - タスクの型、ステータス、ID生成などの基本定義
2. **タスクレジストリ層** (`src/tasks.ts`) - 全タスク実装の登録と検索
3. **タスク実装層** (`src/tasks/` ディレクトリ) - 各タスクタイプの具体的な実装

```
src/
├── Task.ts                          # 基本型定義・ID生成
├── tasks.ts                         # タスクレジストリ（getAllTasks, getTaskByType）
└── tasks/
    ├── types.ts                     # TaskState ユニオン型
    ├── pillLabel.ts                 # フッターピルのラベル生成
    ├── stopTask.ts                  # タスク停止の共通ロジック
    ├── LocalMainSessionTask.ts      # メインセッションのバックグラウンド化
    ├── LocalShellTask/              # ローカルシェルコマンド
    ├── LocalAgentTask/              # ローカルサブエージェント
    ├── RemoteAgentTask/             # リモートエージェント（CCR環境）
    ├── InProcessTeammateTask/       # インプロセスチームメイト
    ├── DreamTask/                   # バックグラウンドメモリ統合
    ├── LocalWorkflowTask/           # ワークフロースクリプト（フィーチャーフラグ制御）
    └── MonitorMcpTask/              # MCPモニター（フィーチャーフラグ制御）
```

## タスクタイプ一覧

`src/Task.ts` (6-13行目) で定義される7つのタスクタイプ:

| タイプ | IDプレフィックス | 説明 | 実装ファイル |
|--------|-----------------|------|-------------|
| `local_bash` | `b` | ローカルシェルコマンドの非同期実行 | `src/tasks/LocalShellTask/LocalShellTask.tsx` |
| `local_agent` | `a` | ローカルサブエージェント（AgentTool経由） | `src/tasks/LocalAgentTask/LocalAgentTask.tsx` |
| `remote_agent` | `r` | リモートCCR環境でのエージェント実行 | `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx` |
| `in_process_teammate` | `t` | 同一プロセス内チームメイト（Swarm） | `src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx` |
| `local_workflow` | `w` | ワークフロースクリプト | フィーチャーフラグ `WORKFLOW_SCRIPTS` 制御 |
| `monitor_mcp` | `m` | MCPモニタリング | フィーチャーフラグ `MONITOR_TOOL` 制御 |
| `dream` | `d` | バックグラウンドメモリ統合（auto-dream） | `src/tasks/DreamTask/DreamTask.ts` |

### 条件付きタスクの読み込み

`src/tasks.ts` (9-14行目) では、`LocalWorkflowTask` と `MonitorMcpTask` はフィーチャーフラグによって条件付きで読み込まれる:

```typescript
const LocalWorkflowTask: Task | null = feature('WORKFLOW_SCRIPTS')
  ? require('./tasks/LocalWorkflowTask/LocalWorkflowTask.js').LocalWorkflowTask
  : null
const MonitorMcpTask: Task | null = feature('MONITOR_TOOL')
  ? require('./tasks/MonitorMcpTask/MonitorMcpTask.js').MonitorMcpTask
  : null
```

## タスクライフサイクル

### ステータス遷移

`src/Task.ts` (15-20行目) で定義される5つのステータス:

```
pending ──> running ──> completed
                    ──> failed
                    ──> killed
```

- **pending**: タスク登録直後の初期状態
- **running**: タスク実行中
- **completed**: 正常完了
- **failed**: エラーによる失敗
- **killed**: ユーザーまたはシステムによる強制停止

終端ステータスの判定は `isTerminalTaskStatus()` 関数 (`src/Task.ts` 27-29行目) で行われる:

```typescript
export function isTerminalTaskStatus(status: TaskStatus): boolean {
  return status === 'completed' || status === 'failed' || status === 'killed'
}
```

### バックグラウンドタスク判定

`src/tasks/types.ts` (37-46行目) の `isBackgroundTask()` は、タスクがフッターピルに表示されるべきかを判定する:

- ステータスが `running` または `pending` であること
- `isBackgrounded` が `false` でないこと（フォアグラウンド実行中のタスクは除外）

## タスクID生成

`src/Task.ts` (79-106行目) でタスクIDが生成される。

### ID構造

タスクIDは **タイプ別プレフィックス1文字 + ランダム8文字** の9文字で構成される:

```
b3k7f2m9x   ← 'b' (local_bash) + 8文字のランダム文字列
a9p2n5q1w   ← 'a' (local_agent) + 8文字のランダム文字列
```

### プレフィックス一覧

`src/Task.ts` (79-87行目):

| タスクタイプ | プレフィックス |
|-------------|--------------|
| `local_bash` | `b` |
| `local_agent` | `a` |
| `remote_agent` | `r` |
| `in_process_teammate` | `t` |
| `local_workflow` | `w` |
| `monitor_mcp` | `m` |
| `dream` | `d` |
| (不明なタイプ) | `x` |

### 特殊なID: メインセッションタスク

`src/tasks/LocalMainSessionTask.ts` (75-82行目) では、メインセッションのバックグラウンド化用にプレフィックス `s` を使用:

```typescript
function generateMainSessionTaskId(): string {
  const bytes = randomBytes(8)
  let id = 's'
  // ...
}
```

### ランダム文字の生成

`src/Task.ts` (94-106行目):

- アルファベット: `0123456789abcdefghijklmnopqrstuvwxyz` （36文字）
- `crypto.randomBytes(8)` で暗号学的に安全な乱数を使用
- 36^8 = 約2.8兆通りの組み合わせ（シンボリックリンク攻撃への耐性）

## タスク状態の基本構造

`src/Task.ts` (45-57行目) の `TaskStateBase`:

```typescript
export type TaskStateBase = {
  id: string           // タスクID
  type: TaskType       // タスクタイプ
  status: TaskStatus   // 現在のステータス
  description: string  // タスクの説明
  toolUseId?: string   // 関連するtool_useのID
  startTime: number    // 開始時刻（Date.now()）
  endTime?: number     // 終了時刻
  totalPausedMs?: number // 一時停止の合計時間
  outputFile: string   // 出力ファイルパス
  outputOffset: number // 出力ファイルの読み取りオフセット
  notified: boolean    // 通知済みフラグ
}
```

## ディスクバックアップと出力ファイル

### 出力ディレクトリ

`src/utils/task/diskOutput.ts` (36-55行目):

- パス: `<projectTempDir>/<sessionId>/tasks/`
- セッションIDごとにディレクトリが分離（並行セッションの衝突を防止）
- セッションIDは初回呼び出し時にキャプチャされ、`/clear` でも変更されない

### セキュリティ対策

`src/utils/task/diskOutput.ts` (20-21行目):

```typescript
const O_NOFOLLOW = fsConstants.O_NOFOLLOW ?? 0
```

`O_NOFOLLOW` フラグにより、シンボリックリンク経由のファイルオープンを防止。サンドボックス内の攻撃者がtasksディレクトリにシンボリックリンクを作成して任意ファイルへの書き込みを誘導する攻撃を防ぐ。

### 出力サイズ制限

`src/utils/task/diskOutput.ts` (30-31行目):

```typescript
export const MAX_TASK_OUTPUT_BYTES = 5 * 1024 * 1024 * 1024  // 5GB
```

### シンボリックリンクによるトランスクリプト連携

`src/tasks/LocalMainSessionTask.ts` (107-109行目):

エージェントタスクの出力ファイルは、エージェントのトランスクリプトファイルへのシンボリックリンクとして作成される。これにより `TaskOutput` コンポーネントからライブ進捗を確認できる。

## タスクの作成・実行・監視フロー

### 1. タスク登録

`src/utils/task/framework.ts` (77-80行目) の `registerTask()`:

```typescript
export function registerTask(task: TaskState, setAppState: SetAppState): void {
  setAppState(prev => {
    // AppState.tasks に新しいタスクを追加
  })
}
```

### 2. 状態更新

`src/utils/task/framework.ts` (48-72行目) の `updateTaskState()`:

- ジェネリック型パラメータで型安全な更新を実現
- アップデータが同じ参照を返した場合は状態変更をスキップ（不要な再レンダリング防止）

### 3. タスク停止

`src/tasks/stopTask.ts` (38-100行目) の `stopTask()`:

1. AppStateからタスクIDでタスクを検索
2. ステータスが `running` であることを検証
3. タスクタイプに対応する `Task.kill()` を呼び出し
4. シェルタスクの場合は通知を抑制（exit code 137のノイズ防止）
5. SDK向けに `task_terminated` イベントを発行

### 4. 通知メカニズム

タスク完了時に `<task-notification>` XML形式の通知が生成される:

```xml
<task-notification>
  <task-id>{taskId}</task-id>
  <tool-use-id>{toolUseId}</tool-use-id>
  <output-file>{outputPath}</output-file>
  <status>completed|failed|killed</status>
  <summary>{概要メッセージ}</summary>
</task-notification>
```

通知の重複防止は `notified` フラグのアトミックなチェック・セットで実現される（`src/tasks/LocalMainSessionTask.ts` 231-239行目）。

## 各タスクタイプの詳細

### LocalShellTask (`local_bash`)

`src/tasks/LocalShellTask/LocalShellTask.tsx`:

- シェルコマンドのバックグラウンド実行
- `kind` フィールドで `'bash'`（通常）と `'monitor'`（モニタリング用）を区別
- ストール検出: 45秒間出力がなく、最終行がインタラクティブプロンプトに見える場合に通知
- エージェントが生成したシェルタスクは `agentId` で紐づけ、エージェント終了時に孤立タスクを自動停止

### LocalAgentTask (`local_agent`)

`src/tasks/LocalAgentTask/LocalAgentTask.tsx`:

- サブエージェントのバックグラウンド実行を管理
- `AgentProgress` で進捗を追跡（ツール使用回数、トークン数、最近のアクティビティ）
- `isBackgrounded` フラグでフォアグラウンド/バックグラウンドの切り替え
- `pendingMessages` キューで実行中エージェントへのメッセージ送信をサポート
- `retain` / `diskLoaded` でUI表示時のメモリ管理を最適化
- `evictAfter` タイムスタンプで終了後のパネル表示期間を制御（デフォルト30秒: `PANEL_GRACE_MS`）

### RemoteAgentTask (`remote_agent`)

`src/tasks/RemoteAgentTask/RemoteAgentTask.tsx`:

- リモートCCR（Claude Code Remote）環境でのエージェント実行
- `remoteTaskType`: `'remote-agent'`, `'ultraplan'`, `'ultrareview'`, `'autofix-pr'`, `'background-pr'`
- ポーリングベースの進捗監視（`pollRemoteSessionEvents`）
- Ultraplanモード: `ultraplanPhase` で `'needs_input'`, `'plan_ready'` などのフェーズ管理
- セッションメタデータをディスクに永続化（`--resume` でのセッション復元対応）
- カスタム完了チェッカー登録（`registerCompletionChecker`）

### InProcessTeammateTask (`in_process_teammate`)

`src/tasks/InProcessTeammateTask/types.ts`:

- 同一Node.jsプロセス内で実行されるチームメイトエージェント
- `TeammateIdentity`: `agentId`, `agentName`, `teamName`, `color`, `planModeRequired`, `parentSessionId`
- プランモード承認フロー（`awaitingPlanApproval`）
- 独立した権限モード（`permissionMode`）
- メッセージUIキャップ: 50件（`TEAMMATE_MESSAGES_UI_CAP`）でメモリ使用量を制限
  - BQ分析（2026-03-20）で500+ターンセッション時に約20MB RSS/エージェント、Swarmバーストで約125MB/並行エージェントを記録
  - Whale session 9a990de8: 2分間で292エージェント起動、36.8GB到達

### DreamTask (`dream`)

`src/tasks/DreamTask/DreamTask.ts`:

- auto-dreamメモリ統合サブエージェントのUI表面化
- `DreamPhase`: `'starting'` -> `'updating'`（最初のEdit/Writeツール使用で遷移）
- `filesTouched`: Edit/Write tool_useで検出されたファイルパス（不完全な反映）
- `turns`: 最大30件の最近のアシスタントターンを保持
- kill時に `consolidationLock` のmtimeをロールバック（次セッションでのリトライを可能に）

### LocalMainSessionTask

`src/tasks/LocalMainSessionTask.ts`:

- メインセッションクエリのバックグラウンド化（Ctrl+B 2回押し）
- `LocalAgentTaskState` を拡張し `agentType: 'main-session'` で区別
- IDプレフィックス `s`（エージェントの `a` と区別）
- 独立したトランスクリプトファイルに出力（`/clear` 後の破損を防止）
- フォアグラウンド復帰（`foregroundMainSessionTask`）: メッセージ履歴の復元と表示

## フッターピル表示

`src/tasks/pillLabel.ts` (10-67行目):

バックグラウンドタスクの種類と数に応じたコンパクトなラベルを生成:

| タスクタイプ | 表示例 |
|-------------|-------|
| `local_bash` | `"1 shell"`, `"3 shells, 2 monitors"` |
| `local_agent` | `"1 local agent"`, `"3 local agents"` |
| `remote_agent` | `"◇ 1 cloud session"`, `"◆ ultraplan ready"` |
| `in_process_teammate` | `"1 team"`, `"2 teams"` |
| `local_workflow` | `"1 background workflow"` |
| `monitor_mcp` | `"1 monitor"` |
| `dream` | `"dreaming"` |
| 混在 | `"5 background tasks"` |

Ultraplanの場合、フェーズに応じてダイヤモンド記号が変化:
- `◇`（open diamond）: 実行中 / 入力待ち
- `◆`（filled diamond）: プラン承認待ち

## タスクフレームワークの定数

`src/utils/task/framework.ts`:

| 定数 | 値 | 用途 |
|------|-----|------|
| `POLL_INTERVAL_MS` | 1000ms | タスクポーリング間隔 |
| `STOPPED_DISPLAY_MS` | 3000ms | 停止タスクの表示期間 |
| `PANEL_GRACE_MS` | 30000ms | コーディネーターパネルでの終了タスク猶予期間 |
