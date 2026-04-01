# セッションとメモリ管理

Claude Code v2.1.88 におけるセッション永続化、メモリシステム、およびセッション復元の技術解析。

## 概要

Claude Code は会話の状態を `~/.claude/` 配下に永続化し、セッションの中断・再開、メモリの自動抽出、ワークツリー管理などを実現している。主要コンポーネントは以下の通り。

| モジュール | ファイルパス | 役割 |
|---|---|---|
| SessionStorage | `src/utils/sessionStorage.ts` | トランスクリプト (JSONL) の読み書き |
| SessionState | `src/utils/sessionState.ts` | セッション状態の管理・通知 |
| SessionRestore | `src/utils/sessionRestore.ts` | セッション再開時の状態復元 |
| SessionMemory | `src/services/SessionMemory/` | 自動メモリ抽出サブエージェント |
| Worktree | `src/utils/worktree.ts` | Git worktree の作成・管理 |

---

## 1. セッション永続化 (`~/.claude/sessions/`)

### ストレージ構造

セッションデータはプロジェクト単位で管理される。パスは `getProjectsDir()` (`src/utils/sessionStorage.ts` L198-200) により `~/.claude/projects/` 配下に生成される。

```
~/.claude/projects/<sanitized-project-path>/
  <session-id>.jsonl          # メイントランスクリプト
  <session-id>/subagents/     # サブエージェントのトランスクリプト
    agent-<agent-id>.jsonl
    agent-<agent-id>.meta.json
```

各トランスクリプトは **JSONL 形式** (1行1エントリ) で保存され、`Entry` 型のレコードが追記される。

### トランスクリプトパスの解決

- `getTranscriptPath()` (L202-205): 現在のセッションのトランスクリプトパスを返す。`sessionProjectDir` が設定されている場合はそちらを優先する。
- `getTranscriptPathForSession(sessionId)` (L207-225): 指定セッションのパスを返す。現在のセッションの場合は `getTranscriptPath()` へ委譲し、パスの不一致を防止する (gh-30217 修正)。
- `getAgentTranscriptPath(agentId)` (L247-258): サブエージェントのトランスクリプトパス。`agentTranscriptSubdirs` マップによりサブディレクトリのグルーピングが可能。

### トランスクリプトメッセージの判定

`isTranscriptMessage()` (L139-146) が単一の真実源 (single source of truth) として、どのエントリがトランスクリプトに含まれるかを定義する。対象は `user`、`assistant`、`attachment`、`system` の4種類。**`progress` メッセージは含まれない** -- これらはエフェメラルな UI 状態であり、永続化すると親UUID チェーンのフォークが発生し、実メッセージが孤立する問題があった (#14373, #23537)。

### サイズ制限

- `MAX_TRANSCRIPT_READ_BYTES`: 50 MB (L229)。セッションファイルは数 GB に成長しうるため、OOM 防止のために読み取り上限が設定されている。
- `MAX_TOMBSTONE_REWRITE_BYTES`: 50 MB (L123)。トゥームストーン書き換え時の上限。

---

## 2. セッション状態管理 (`sessionState.ts`)

### 状態モデル

セッション状態は3値で管理される (`src/utils/sessionState.ts` L1):

```typescript
type SessionState = 'idle' | 'running' | 'requires_action'
```

### RequiresActionDetails

`requires_action` 状態のコンテキスト情報 (L15-24):

| フィールド | 説明 |
|---|---|
| `tool_name` | ブロックしているツール名 |
| `action_description` | 人間可読な要約 (例: "Editing src/foo.ts") |
| `tool_use_id` | ツール使用 ID |
| `request_id` | リクエスト ID |
| `input` | ツール入力 (フロントエンドが `pending_action` から読み取る) |

### 外部メタデータ

`SessionExternalMetadata` (L37-45) はセッションに紐づく外部公開メタデータ:

- `permission_mode`: 権限モード
- `is_ultraplan_mode`: ウルトラプランモードフラグ
- `model`: 使用モデル
- `pending_action`: ブロック中のアクション詳細
- `post_turn_summary` / `task_summary`: ターン後サマリー / タスクサマリー (長時間実行時の進捗表示)

### リスナーパターン

状態変更は3つのリスナーを通じて通知される (L56-83):

1. `SessionStateChangedListener` -- 状態遷移 (`idle` / `running` / `requires_action`)
2. `SessionMetadataChangedListener` -- メタデータ更新
3. `PermissionModeChangedListener` -- 権限モード変更 (Shift+Tab、スラッシュコマンド等)

---

## 3. メッセージ履歴の管理

### parentUuid チェーン

メッセージは `uuid` / `parentUuid` のチェーンで関連付けられる。`isChainParticipant()` (L154-156) により `progress` タイプのメッセージはチェーンから除外される。

### レガシー Progress エントリの処理

PR #24099 以前のトランスクリプトには `progress` エントリが UUID チェーンに含まれていた。`isLegacyProgressEntry()` (L169-178) で検出し、`loadTranscriptFile` がチェーンのブリッジ処理を行う。

### エフェメラルツール進捗

`EPHEMERAL_PROGRESS_TYPES` (L186-193) で定義される以下のタイプはUIのみで使用:

- `bash_progress`
- `powershell_progress`
- `mcp_progress`
- `sleep_progress` (PROACTIVE / KAIROS フィーチャーフラグ有効時)

---

## 4. メモリシステム

### SessionMemory サービス

`src/services/SessionMemory/sessionMemory.ts` は、会話中に自動的にメモリを抽出・保存するサブエージェントを管理する。

**動作原理** (L1-5):
> Session Memory automatically maintains a markdown file with notes about the current conversation. It runs periodically in the background using a forked subagent to extract key information without interrupting the main conversation flow.

### 抽出トリガー条件

`shouldExtractMemory()` (L134-150) が以下の条件を評価:

1. **初期化しきい値**: `minimumMessageTokensToInit` (デフォルト 10,000 トークン) 以上のコンテキストウィンドウトークン数
2. **更新間隔しきい値**: 前回抽出以降 `minimumTokensBetweenUpdate` (デフォルト 5,000 トークン) 以上のコンテキスト成長
3. **ツールコール数**: 前回以降 `toolCallsBetweenUpdates` (デフォルト 3 回) 以上のツール呼び出し

### 設定 (`SessionMemoryConfig`)

`src/services/SessionMemory/sessionMemoryUtils.ts` L18-29 で定義:

```typescript
type SessionMemoryConfig = {
  minimumMessageTokensToInit: number    // デフォルト: 10000
  minimumTokensBetweenUpdate: number    // デフォルト: 5000
  toolCallsBetweenUpdates: number       // デフォルト: 3
}
```

### フィーチャーゲート

- `tengu_session_memory`: GrowthBook によるフィーチャーフラグ (L81)
- `tengu_sm_config`: リモート設定のオーバーライド (L88-93)

### 抽出の同期制御

`waitForSessionMemoryExtraction()` (L89-100) が進行中の抽出完了を待機:

- タイムアウト: 15秒 (`EXTRACTION_WAIT_TIMEOUT_MS`)
- 古い抽出の無視: 60秒以上 (`EXTRACTION_STALE_THRESHOLD_MS`) 経過した抽出はスキップ

### メモリタイプ

`src/utils/memory/types.ts` で定義される `MemoryType`:

| タイプ | 説明 |
|---|---|
| `User` | ユーザーレベルの記憶 |
| `Project` | プロジェクトレベルの記憶 |
| `Local` | ローカルの記憶 |
| `Managed` | 管理された記憶 |
| `AutoMem` | 自動生成された記憶 |
| `TeamMem` | チームメモリ (TEAMMEM フィーチャーフラグ時) |

---

## 5. セッションのリストアと復元

### 復元フロー (`sessionRestore.ts`)

`restoreSessionStateFromLog()` (L99) がセッション再開時の状態復元を統括する。

**入力** (`ResumeResult` 型, L64-70):

```typescript
type ResumeResult = {
  messages?: Message[]
  fileHistorySnapshots?: FileHistorySnapshot[]
  attributionSnapshots?: AttributionSnapshotMessage[]
  contextCollapseCommits?: ContextCollapseCommitEntry[]
  contextCollapseSnapshot?: ContextCollapseSnapshotEntry
}
```

### 復元される状態

1. **ファイル履歴**: `fileHistoryRestoreStateFromLog()` で復元
2. **コミット帰属**: `attributionRestoreStateFromLog()` / `restoreAttributionStateFromSnapshots()` で復元
3. **TODO リスト**: `extractTodosFromTranscript()` (L77-93) がトランスクリプト末尾から最後の `TodoWrite` ツール使用を探索し、TODO 状態を再構築
4. **メモリファイルキャッシュ**: `clearMemoryFileCaches()` で初期化
5. **コスト状態**: `restoreCostStateForSession()` で復元
6. **エージェント定義**: `getAgentDefinitionsWithOverrides()` で復元
7. **セッションファイルポインタ**: `adoptResumedSessionFile()` / `resetSessionFilePointer()` で復元
8. **録画ファイル**: `renameRecordingForSession()` でセッション ID に紐付け

### 依存モジュール

セッション復元は `src/utils/sessionStorage.ts` の以下の関数に依存:

- `adoptResumedSessionFile()` -- 既存セッションファイルの引き継ぎ
- `resetSessionFilePointer()` -- ファイルポインタのリセット
- `restoreSessionMetadata()` -- メタデータの復元
- `saveMode()` -- モード状態の保存
- `saveWorktreeState()` -- ワークツリー状態の保存

---

## 6. Worktree 管理

### 概要

`src/utils/worktree.ts` は Git worktree の作成・管理を担当する。メインリポジトリから分離した作業ディレクトリでの並行作業を可能にする。

### Worktree スラグの検証

`validateWorktreeSlug()` (L66-87) がセキュリティ上の検証を実施:

- **最大長**: 64文字 (`MAX_WORKTREE_SLUG_LENGTH`)
- **パストラバーサル防止**: `..` や絶対パスの拒否
- **文字制限**: `[a-zA-Z0-9._-]` のみ (`VALID_WORKTREE_SLUG_SEGMENT`)
- **ネスト許可**: `/` 区切りで階層化可能 (例: `asm/feature-foo`)。各セグメントが独立に検証される

### ディレクトリのシンボリックリンク

`mkdirRecursive()` と関連ヘルパーにより、メインリポジトリの `node_modules` 等の大規模ディレクトリをシンボリックリンクで共有し、ディスク容量の膨張を防止する (L94-100)。

### フック連携

Worktree の作成・削除時にフックが実行される:

- `executeWorktreeCreateHook()` -- 作成時フック
- `executeWorktreeRemoveHook()` -- 削除時フック
- `hasWorktreeCreateHook()` -- 作成フックの存在確認

### Git 統合

`src/utils/git/gitFilesystem.ts` の以下の関数と連携:

- `getCommonDir()` -- 共通 Git ディレクトリの取得
- `readWorktreeHeadSha()` -- Worktree の HEAD SHA 読み取り
- `resolveGitDir()` -- Git ディレクトリの解決
- `resolveRef()` -- 参照の解決

---

## 7. トランスクリプト記録

### JSONL フォーマット

トランスクリプトは JSONL (JSON Lines) 形式で記録される。各行は `Entry` 型のレコードで、以下のタイプが含まれる:

- `TranscriptMessage`: `user` | `assistant` | `attachment` | `system`
- `FileHistorySnapshotMessage`: ファイル変更のスナップショット
- `AttributionSnapshotMessage`: コミット帰属のスナップショット
- `ContextCollapseCommitEntry` / `ContextCollapseSnapshotEntry`: コンテキスト圧縮関連
- `ContentReplacementEntry`: コンテンツ置換記録

### 最初のプロンプトの抽出

`SKIP_FIRST_PROMPT_PATTERN` (L125-127) により、IDE コンテキスト、フック出力、中断マーカー等の非意味的メッセージをスキップして最初の実質的なプロンプトを抽出する。

```typescript
const SKIP_FIRST_PROMPT_PATTERN =
  /^(?:\s*<[a-z][\w-]*[\s>]|\[Request interrupted by user[^\]]*\])/
```

### サブエージェントのトランスクリプト

サブエージェントのトランスクリプトはセッションディレクトリ配下の `subagents/` ディレクトリに保存される。`agentTranscriptSubdirs` マップ (L234) によりサブディレクトリのグルーピングが管理される (例: ワークフロー実行)。

メタデータは `.meta.json` ファイルに保存され、`AgentMetadata` 型 (L264) で `agentType`、`worktreePath` 等が記録される。
