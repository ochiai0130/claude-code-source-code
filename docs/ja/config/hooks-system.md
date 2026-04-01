# Hooks システム

Claude Code v2.1.88 ソースコード分析

## 概要

Hooksシステムは、Claude Codeのライフサイクル各段階でユーザー定義のシェルコマンド、LLMプロンプト、HTTPリクエスト、またはエージェント検証を実行するための拡張メカニズムである。`settings.json` の `hooks` セクションで定義され、ツール実行の前後、セッション開始/終了時など多様なイベントに対応する。

**主要ファイル:** `src/utils/hooks.ts`（メイン実行エンジン、1000行以上）

---

## 1. ライフサイクルコールバック

### 1.1 フックイベント一覧

`src/entrypoints/agentSdkTypes.ts` で定義される `HOOK_EVENTS` に基づき、以下のイベントが利用可能:

| イベント | タイミング | 入力型 |
|----------|-----------|--------|
| `PreToolUse` | ツール実行前 | `PreToolUseHookInput` |
| `PostToolUse` | ツール実行後（成功時） | `PostToolUseHookInput` |
| `PostToolUseFailure` | ツール実行後（失敗時） | `PostToolUseFailureHookInput` |
| `Notification` | 通知送信時 | `NotificationHookInput` |
| `Stop` | モデル停止時 | `StopHookInput` |
| `StopFailure` | 停止失敗時 | `StopFailureHookInput` |
| `PreCompact` | コンパクト処理前 | `PreCompactHookInput` |
| `PostCompact` | コンパクト処理後 | `PostCompactHookInput` |
| `SessionStart` | セッション開始時 | `SessionStartHookInput` |
| `SessionEnd` | セッション終了時 | `SessionEndHookInput` |
| `Setup` | 初回セットアップ時 | `SetupHookInput` |
| `SubagentStart` | サブエージェント開始時 | `SubagentStartHookInput` |
| `SubagentStop` | サブエージェント停止時 | `SubagentStopHookInput` |
| `TeammateIdle` | チームメイトアイドル時 | `TeammateIdleHookInput` |
| `TaskCreated` | タスク作成時 | `TaskCreatedHookInput` |
| `TaskCompleted` | タスク完了時 | `TaskCompletedHookInput` |
| `ConfigChange` | 設定変更時 | `ConfigChangeHookInput` |
| `CwdChanged` | 作業ディレクトリ変更時 | `CwdChangedHookInput` |
| `FileChanged` | ファイル変更時 | `FileChangedHookInput` |
| `InstructionsLoaded` | 指示読み込み時 | `InstructionsLoadedHookInput` |
| `UserPromptSubmit` | ユーザープロンプト送信時 | `UserPromptSubmitHookInput` |
| `PermissionRequest` | 権限リクエスト時 | `PermissionRequestHookInput` |
| `PermissionDenied` | 権限拒否時 | `PermissionDeniedHookInput` |
| `Elicitation` | 質問応答時 | `ElicitationHookInput` |
| `ElicitationResult` | 質問応答結果時 | `ElicitationResultHookInput` |

---

## 2. フック定義と設定

### 2.1 フックコマンドの4つの型

**ファイル:** `src/schemas/hooks.ts`

#### (a) BashCommandHook（シェルコマンド）

```json
{
  "type": "command",
  "command": "npm test",
  "if": "Bash(npm *)",
  "shell": "bash",
  "timeout": 60,
  "statusMessage": "テスト実行中...",
  "once": false,
  "async": false,
  "asyncRewake": false
}
```

| フィールド | 説明 |
|------------|------|
| `command` | 実行するシェルコマンド |
| `if` | 権限ルール構文によるフィルタ（例: `"Bash(git *)"`, `"Read(*.ts)"`） |
| `shell` | シェル種別（`bash` または `powershell`、デフォルト: `bash`） |
| `timeout` | タイムアウト秒数 |
| `statusMessage` | スピナーに表示するカスタムメッセージ |
| `once` | trueの場合、1回実行後に削除 |
| `async` | trueの場合、バックグラウンドで非ブロッキング実行 |
| `asyncRewake` | trueの場合、バックグラウンド実行し、終了コード2でモデルを再起動 |

#### (b) PromptHook（LLMプロンプト）

```json
{
  "type": "prompt",
  "prompt": "このコード変更を確認してください: $ARGUMENTS",
  "model": "claude-sonnet-4-6",
  "timeout": 30
}
```

`$ARGUMENTS` プレースホルダーにフック入力JSONが埋め込まれる。未指定時はデフォルトの小型高速モデルが使用される。

#### (c) HttpHook（HTTP POST）

```json
{
  "type": "http",
  "url": "https://hooks.example.com/webhook",
  "headers": {
    "Authorization": "Bearer $MY_TOKEN"
  },
  "allowedEnvVars": ["MY_TOKEN"],
  "timeout": 10
}
```

ヘッダー内の `$VAR_NAME` / `${VAR_NAME}` は `allowedEnvVars` に列挙された環境変数のみ展開される。SSRF防御として `ssrfGuard.ts` が実装されている。

#### (d) AgentHook（エージェント検証）

```json
{
  "type": "agent",
  "prompt": "ユニットテストが成功したことを確認してください。",
  "model": "claude-sonnet-4-6",
  "timeout": 60
}
```

独立したエージェントセッションで検証タスクを実行する。`execAgentHook.ts` が処理を担当。

### 2.2 settings.json でのフック定義

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "type": "command",
        "command": "echo 'ツール実行前チェック'",
        "if": "Bash(rm *)"
      }
    ],
    "PostToolUse": [
      {
        "type": "http",
        "url": "https://hooks.internal.corp/audit",
        "headers": { "X-Api-Key": "$AUDIT_KEY" },
        "allowedEnvVars": ["AUDIT_KEY"]
      }
    ],
    "SessionStart": [
      {
        "type": "command",
        "command": "echo 'セッション開始'"
      }
    ]
  }
}
```

---

## 3. フック実行のタイミングとフロー

### 3.1 タイムアウト

```typescript
// hooks.ts L166
const TOOL_HOOK_EXECUTION_TIMEOUT_MS = 10 * 60 * 1000  // 10分（通常フック）

// hooks.ts L175-176
const SESSION_END_HOOK_TIMEOUT_MS_DEFAULT = 1500  // 1.5秒（SessionEndフック）
```

SessionEndフックはシャットダウン/クリア時に実行されるため、非常に短いタイムアウトが設定されている。環境変数 `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` でオーバーライド可能。

### 3.2 信頼チェック

`hooks.ts` L267-296の `shouldSkipHookDueToTrust()`:

- **全てのフック**はワークスペース信頼を必要とする（defense-in-depth）
- 非対話モード（SDK）では信頼は暗黙的に付与される
- 対話モードでは `checkHasTrustDialogAccepted()` で確認

歴史的脆弱性の対策:
- 信頼ダイアログを拒否した際のSessionEndフック実行
- 信頼前にサブエージェントが完了した際のSubagentStopフック実行

### 3.3 共通入力（Base Hook Input）

`createBaseHookInput()`（L301-328）が全フック型の共通入力を生成:

```typescript
{
  session_id: string,
  transcript_path: string,  // セッショントランスクリプトのパス
  cwd: string,              // 現在の作業ディレクトリ
  permission_mode?: string,
  agent_id?: string,
  agent_type?: string,
}
```

### 3.4 非同期フック実行

`executeInBackground()`（L184-265）:

- `async: true` のフックはバックグラウンドで実行され、完了を待たない
- `asyncRewake: true` のフックはバックグラウンド実行し、終了コード2（ブロッキングエラー）時にモデルを再起動させる
- 非同期フックは `AsyncHookRegistry`（`src/utils/hooks/AsyncHookRegistry.ts`）で管理される

---

## 4. フックからのフィードバック処理

### 4.1 JSON出力スキーマ

フックは標準出力にJSONを返すことで動作を制御できる。`hooks.ts` L382-451の `parseHookOutput()`:

```json
{
  "continue": true,
  "suppressOutput": false,
  "stopReason": "テスト失敗",
  "decision": "approve" | "block",
  "reason": "理由の説明",
  "systemMessage": "システムメッセージ",
  "permissionDecision": "allow" | "deny" | "ask",
  "hookSpecificOutput": { ... }
}
```

### 4.2 イベント固有の出力

各イベントで `hookSpecificOutput` に追加フィールドを返せる:

| イベント | 追加フィールド |
|----------|---------------|
| `PreToolUse` | `permissionDecision`, `permissionDecisionReason`, `updatedInput`, `additionalContext` |
| `PostToolUse` | `additionalContext`, `updatedMCPToolOutput` |
| `UserPromptSubmit` | `additionalContext`（必須） |
| `SessionStart` | `additionalContext`, `initialUserMessage`, `watchPaths` |
| `PermissionRequest` | `decision` (behavior + updatedInput) |
| `PermissionDenied` | `retry` |
| `Elicitation` | `action`, `content` |

### 4.3 HookResult の構造

`hooks.ts` L338-357:

```typescript
export interface HookResult {
  message?: HookResultMessage
  systemMessage?: string
  blockingError?: HookBlockingError
  outcome: 'success' | 'blocking' | 'non_blocking_error' | 'cancelled'
  preventContinuation?: boolean
  stopReason?: string
  permissionBehavior?: 'ask' | 'deny' | 'allow' | 'passthrough'
  updatedInput?: Record<string, unknown>
  // ...
}
```

---

## 5. フックイベントシステム

**ファイル:** `src/utils/hooks/hookEvents.ts`

ブロードキャスト型のイベントシステム。メインメッセージストリームとは独立。

### 5.1 イベント型

```typescript
type HookStartedEvent = { type: 'started', hookId, hookName, hookEvent }
type HookProgressEvent = { type: 'progress', hookId, hookName, hookEvent, stdout, stderr, output }
type HookResponseEvent = { type: 'response', hookId, hookName, hookEvent, output, stdout, stderr, exitCode, outcome }
```

### 5.2 常時発行イベント

`hookEvents.ts` L18:

```typescript
const ALWAYS_EMITTED_HOOK_EVENTS = ['SessionStart', 'Setup'] as const
```

これらは `includeHookEvents` オプションに関係なく常に発行される。他のイベントは `setAllHookEventsEnabled(true)` が呼ばれた場合（SDK `includeHookEvents` オプションまたは `CLAUDE_CODE_REMOTE` モード時）のみ発行。

### 5.3 プログレス通知

`startHookProgressInterval()`（L124-151）は1秒間隔でフックの進捗をポーリングし、出力が変化した場合にのみプログレスイベントを発行する。

---

## 6. フック設定のスナップショット管理

**ファイル:** `src/utils/hooks/hooksConfigSnapshot.ts`

### 6.1 スナップショットの目的

アプリケーション起動時にフック設定のスナップショットを取得し、セッション中の設定変更による予期しないフック実行を防止する。

### 6.2 マネージドフックのポリシー制御

`hooksConfigSnapshot.ts` L18-53 の `getHooksFromAllowedSources()`:

1. `policySettings.disableAllHooks === true` → 空のフック設定を返す
2. `policySettings.allowManagedHooksOnly === true` → マネージドフックのみ
3. `strictPluginOnlyCustomization` でフックがロック → マネージドフックのみ
4. 非マネージド設定で `disableAllHooks === true` → マネージドフックは引き続き実行（非マネージド設定はマネージドフックを無効化できない）
5. それ以外 → 全ソースからのマージ済みフック

### 6.3 スナップショットの更新

```typescript
captureHooksConfigSnapshot()  // 起動時に呼び出し
updateHooksConfigSnapshot()   // /hooks コマンド等で設定変更時に呼び出し
```

`updateHooksConfigSnapshot()` はディスクから最新の設定を読み直すため、`resetSettingsCache()` を先に実行する。

---

## 7. カスタムフックの作成

### 7.1 基本的なコマンドフック

`~/.claude/settings.json` に追加:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "type": "command",
        "command": "echo '{\"decision\": \"approve\"}' ",
        "if": "Bash(git commit *)"
      }
    ]
  }
}
```

### 7.2 条件付きフック（if フィルタ）

`if` フィールドは権限ルール構文を使用し、フック起動前にマッチングを行う（`src/schemas/hooks.ts` L17-27）:

```
"if": "Bash(git *)"       → gitコマンド実行時のみ
"if": "Read(*.ts)"        → TypeScriptファイル読み取り時のみ
"if": "Edit(src/**)"      → srcディレクトリ配下の編集時のみ
```

これにより、不要なフックプロセスの起動を回避できる。

### 7.3 HTTP Webhookフック

外部サービスとの連携:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "type": "http",
        "url": "https://api.example.com/audit",
        "headers": {
          "Authorization": "Bearer $API_TOKEN"
        },
        "allowedEnvVars": ["API_TOKEN"]
      }
    ]
  }
}
```

**セキュリティ:** `allowedHttpHookUrls` 設定（`types.ts` L479-489）でURL許可リストを定義可能。`httpHookAllowedEnvVars`（L491-499）で環境変数の使用を制限可能。

### 7.4 シェル選択

`hooks.ts` L780-792:
- `hook.shell` → `DEFAULT_HOOK_SHELL`（`bash`）の優先順位で決定
- PowerShellフックは `pwsh -NoProfile -NonInteractive -Command` で実行
- WindowsではGit Bash（Cygwin）経由で実行（cmd.exeではない）

---

## 関連ファイル

| ファイル | 説明 |
|----------|------|
| `src/utils/hooks.ts` | メインフック実行エンジン |
| `src/schemas/hooks.ts` | フックZodスキーマ定義 |
| `src/utils/hooks/hookEvents.ts` | イベントブロードキャストシステム |
| `src/utils/hooks/hooksConfigSnapshot.ts` | 設定スナップショット管理 |
| `src/utils/hooks/AsyncHookRegistry.ts` | 非同期フック登録・管理 |
| `src/utils/hooks/execAgentHook.ts` | エージェントフック実行 |
| `src/utils/hooks/execPromptHook.ts` | プロンプトフック実行 |
| `src/utils/hooks/execHttpHook.ts` | HTTPフック実行 |
| `src/utils/hooks/hooksSettings.ts` | フック設定ヘルパー |
| `src/utils/hooks/sessionHooks.ts` | セッションフック・関数フック管理 |
| `src/utils/hooks/ssrfGuard.ts` | SSRF防御 |
| `src/utils/hooks/fileChangedWatcher.ts` | ファイル変更監視 |
| `src/utils/hooks/postSamplingHooks.ts` | サンプリング後フック |
| `src/utils/hooks/registerFrontmatterHooks.ts` | フロントマターフック登録 |
| `src/utils/hooks/registerSkillHooks.ts` | スキルフック登録 |
| `src/types/hooks.ts` | フック関連の型定義 |
