# マルチエージェント

Claude Codeは単一のエージェントだけでなく、複数のサブエージェントを生成・協調させるマルチエージェントアーキテクチャを備えている。本ドキュメントでは、AgentToolによるサブエージェント生成、コーディネーターモード、DreamTask、InProcessTeammate、KAIROSの各機構を解説する。

---

## AgentToolによるサブエージェント生成

### 概要

`AgentTool`はサブエージェントの生成と実行を担う中核ツールである。親エージェントが`Agent`ツールを呼び出すと、独立したコンテキストを持つ子エージェントが起動し、指定されたタスクを実行する。

**主要ファイル:**

- `src/tools/AgentTool/AgentTool.tsx` -- ツール定義、入出力スキーマ、呼び出しエントリポイント
- `src/tools/AgentTool/runAgent.ts` -- エージェント実行ループの本体
- `src/tools/AgentTool/forkSubagent.ts` -- フォーク型サブエージェント（会話コンテキスト継承）
- `src/tools/AgentTool/builtInAgents.ts` -- 組み込みエージェント定義の集約
- `src/tools/AgentTool/prompt.ts` -- エージェント一覧のプロンプト生成

### 入力スキーマ

`AgentTool`の入力は以下のフィールドで構成される（`src/tools/AgentTool/AgentTool.tsx`）:

| フィールド | 説明 |
|---|---|
| `description` | タスクの短い説明（3-5語） |
| `prompt` | エージェントに実行させるタスク内容 |
| `subagent_type` | 使用する特化エージェントの種類（省略時はgeneral-purpose） |
| `model` | モデルオーバーライド（`sonnet`, `opus`, `haiku`） |
| `run_in_background` | バックグラウンド実行フラグ |
| `isolation` | 分離モード（`worktree`: gitワークツリー分離、`remote`: リモート環境） |
| `cwd` | 作業ディレクトリの絶対パス指定 |
| `name` | スポーンされるエージェントの名前（SendMessageでアドレス可能） |

### 組み込みエージェントの種類

`src/tools/AgentTool/builtInAgents.ts`で定義される組み込みエージェント:

- **GENERAL_PURPOSE_AGENT** -- 汎用タスク実行エージェント（常に有効）
- **EXPLORE_AGENT** -- コードベース探索用エージェント
- **PLAN_AGENT** -- 計画策定用エージェント
- **VERIFICATION_AGENT** -- 検証用エージェント（フィーチャーゲート制御）
- **CLAUDE_CODE_GUIDE_AGENT** -- ガイドエージェント（非SDK環境のみ）
- **STATUSLINE_SETUP_AGENT** -- ステータスライン設定用エージェント

### フォーク型サブエージェント

`src/tools/AgentTool/forkSubagent.ts`で実装されるフォーク機構は、`subagent_type`を省略した場合に親の会話コンテキストを丸ごと継承する子エージェントを生成する。

```
isForkSubagentEnabled() = true かつ subagent_type 省略
  -> 親の会話履歴とシステムプロンプトをバイト単位で複製
  -> プロンプトキャッシュ共有のため全フォーク子は同一プレフィックスを維持
  -> コーディネーターモードとは排他（coordinator が有効なら fork は無効）
```

フォーク子エージェントは `FORK_AGENT` 定義に従い、`tools: ['*']`（親と同一ツールプール）、`permissionMode: 'bubble'`（権限リクエストを親に伝播）で動作する。再帰的フォークは `isInForkChild()` で防止される。

### エージェントメモリ

`src/tools/AgentTool/agentMemory.ts`はエージェント固有の永続メモリを管理する。スコープは3種類:

- `user` -- `~/.claude/agent-memory/`（ユーザーグローバル）
- `project` -- `.claude/agent-memory/`（プロジェクト共有）
- `local` -- `.claude/agent-memory-local/`（プロジェクトローカル、VCS対象外）

---

## マルチエージェント協調（Coordinator / Swarm）

### コーディネーターモード

`src/coordinator/coordinatorMode.ts`で制御されるコーディネーターモードは、親エージェントがオーケストレーション専任となり、実作業をワーカーエージェントに委譲するアーキテクチャである。

**有効化条件:**

```typescript
// src/coordinator/coordinatorMode.ts
export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}
```

環境変数 `CLAUDE_CODE_COORDINATOR_MODE=1` とフィーチャーゲート `COORDINATOR_MODE` の両方が必要。

**主な機能:**

- ワーカーエージェントが利用可能なツール一覧をコンテキストに注入（`getCoordinatorUserContext()`）
- セッション再開時のモード自動同期（`matchSessionMode()`）
- スクラッチパッド（`tengu_scratch`ゲート）によるエージェント間の共有メモ領域
- コーディネーター専用の組み込みエージェント（`getCoordinatorAgents()`）

### InProcessTeammate

`src/utils/swarm/`配下で実装されるインプロセスチームメイトは、同一Node.jsプロセス内で複数のエージェントを並行実行する仕組みである。

**主要ファイル:**

- `src/utils/swarm/spawnInProcess.ts` -- チームメイトの生成と登録
- `src/utils/swarm/inProcessRunner.ts` -- 実行ループ（runAgent()のラッパー）
- `src/utils/swarm/backends/InProcessBackend.ts` -- バックエンド実装
- `src/tasks/InProcessTeammateTask/` -- タスク状態管理

**設計の特徴:**

- `AsyncLocalStorage`によるコンテキスト分離（`runWithTeammateContext()`）
- プロセス分離（tmux/iTerm2ベース）とは異なり、メモリ空間を共有
- リンクされた`AbortController`による協調的キャンセル
- リーダーエージェントへのアイドル通知
- Planモード承認フローのサポート
- `SendMessage`ツールによるエージェント間メッセージング

```
SpawnContext -> TeammateContext作成 -> AbortController連携
  -> InProcessTeammateTaskState登録 -> runAgent()実行
  -> 完了時にリーダーへ通知
```

---

## DreamTask（バックグラウンド推論）

### 概要

DreamTaskは「自動メモリ統合」を実行するバックグラウンドタスクである。セッション間で蓄積された知見を整理・統合するサブエージェントをバックグラウンドで起動する。

**主要ファイル:**

- `src/tasks/DreamTask/DreamTask.ts` -- タスク状態定義とライフサイクル管理
- `src/services/autoDream/autoDream.ts` -- 自動起動ロジック
- `src/services/autoDream/consolidationPrompt.ts` -- 統合プロンプト生成

### 起動条件（ゲート順序）

`src/services/autoDream/autoDream.ts`に定義されたゲートを安価な順に評価する:

1. **時間ゲート** -- `lastConsolidatedAt`からの経過時間が`minHours`以上
2. **セッションゲート** -- `lastConsolidatedAt`以降に変更されたトランスクリプト数が`minSessions`以上
3. **ロックゲート** -- 他プロセスが統合処理中でないこと

### タスク状態

```typescript
// src/tasks/DreamTask/DreamTask.ts
type DreamTaskState = TaskStateBase & {
  type: 'dream'
  phase: DreamPhase        // 'starting' | 'updating'
  sessionsReviewing: number
  filesTouched: string[]   // Edit/Writeで検出されたパス（不完全）
  turns: DreamTurn[]       // アシスタント応答ターン
  abortController?: AbortController
  priorMtime: number       // キル時のロック巻き戻し用
}
```

- `phase`は`'starting'`から始まり、最初のEdit/Writeツール使用で`'updating'`に遷移
- `filesTouched`はツール呼び出しのパターンマッチで検出されるため、bash経由の書き込みは捕捉されない
- フッターのピルとShift+Downダイアログに進捗が表示される

### KAIROSモードとの関係

KAIROSモードがアクティブな場合、自動Dreamは無効化される（`getKairosActive()`チェック）。KAIROSは独自のdisk-skill dreamを使用するためである。

---

## KAIROS（自律エージェントモード）

### 概要

KAIROSはClaude Codeの自律エージェントモードであり、フィーチャーゲート`KAIROS`および`KAIROS_CHANNELS`で制御される。プロアクティブな動作を可能にし、ユーザーの明示的な指示なしに推論・行動を開始できる。

**関連フィーチャーゲート:**

- `KAIROS` -- コア機能の有効化
- `KAIROS_CHANNELS` -- チャネル機能
- `KAIROS_BRIEF` -- 簡潔応答モード
- `PROACTIVE` -- プロアクティブ動作（KAIROSと共通のコードパスを持つ）

### 主な機能

- **プロアクティブモジュール** -- `src/proactive/index.js`で実装。KAIROSまたはPROACTIVEフィーチャーが有効な場合にのみロードされる（`src/tools/AgentTool/AgentTool.tsx`、`src/screens/REPL.tsx`）
- **cwdオーバーライド** -- AgentToolの`cwd`パラメータはKAIROSビルドでのみ公開される（`src/tools/AgentTool/AgentTool.tsx` L111-113）
- **永続的実行** -- bridgeコマンドで`perpetual=true`を設定し、継続的な実行を実現（`src/commands/bridge/bridge.tsx`）
- **アシスタント履歴** -- 過去の会話履歴の遅延ロード（`src/screens/REPL.tsx`）
- **自動Dream無効化** -- KAIROSモードでは組み込みの自動Dreamを使わず、独自のdisk-skill dreamで統合を行う

### アーキテクチャ上の位置づけ

```
                    KAIROS（自律モード）
                         |
            +------------+------------+
            |                         |
     Proactive Module          Bridge (perpetual)
            |                         |
    REPL統合 / 履歴ロード      永続的セッション管理
            |
    AgentTool (cwd override)
            |
    Dream (disk-skill方式)
```

KAIROSはビルド時のフィーチャーゲートで制御されるため、非KAIROSビルドではデッドコード除去により関連コードが完全に除外される。`feature('KAIROS')`の結果に基づき`require()`による遅延ロードが行われる設計である。

---

## アーキテクチャ全体像

```
ユーザー入力
    |
    v
メインエージェント（REPL）
    |
    +-- AgentTool呼び出し
    |       |
    |       +-- subagent_type指定 --> 組み込み/カスタムエージェント
    |       +-- subagent_type省略 --> フォーク（親コンテキスト継承）
    |       +-- isolation: worktree --> gitワークツリー分離
    |       +-- isolation: remote --> リモート環境実行
    |       +-- run_in_background --> 非同期実行 + タスク通知
    |
    +-- Coordinatorモード
    |       |
    |       +-- ワーカーエージェント委譲
    |       +-- スクラッチパッド共有
    |       +-- SendMessage相互通信
    |
    +-- InProcessTeammate（Swarm）
    |       |
    |       +-- AsyncLocalStorageコンテキスト分離
    |       +-- 同一プロセス内並行実行
    |       +-- リーダー/メンバー構造
    |
    +-- DreamTask（バックグラウンド）
    |       |
    |       +-- 時間/セッション/ロックゲート
    |       +-- メモリ統合サブエージェント
    |
    +-- KAIROS（自律モード）
            |
            +-- プロアクティブ動作
            +-- 永続セッション
            +-- disk-skill dream
```
