# Hooksとキーバインド

Claude Codeのフロントエンドは、80以上のReact hooksと柔軟なキーバインディングシステムで構成されている。本ドキュメントではhooksの分類、主要hooksの解説、キーバインディングアーキテクチャ、およびVimモードの実装について説明する。

## React Hooks一覧と分類

すべてのhooksは `src/hooks/` ディレクトリに配置されている。以下にカテゴリ別で分類する。

### 権限・セキュリティ

| Hook | ファイル | 概要 |
|------|---------|------|
| `useCanUseTool` | `src/hooks/useCanUseTool.tsx` | ツール使用権限の判定と許可フロー |
| `useSwarmPermissionPoller` | `src/hooks/useSwarmPermissionPoller.ts` | Swarmモードの権限ポーリング |

関連サブディレクトリ: `src/hooks/toolPermission/` にハンドラ群（`coordinatorHandler`、`interactiveHandler`、`swarmWorkerHandler`）が配置されている。

### UIインタラクション

| Hook | ファイル | 概要 |
|------|---------|------|
| `useTextInput` | `src/hooks/useTextInput.ts` | テキスト入力のコア処理 |
| `useVimInput` | `src/hooks/useVimInput.ts` | Vim風テキスト入力の実装 |
| `useArrowKeyHistory` | `src/hooks/useArrowKeyHistory.tsx` | 矢印キーによる履歴ナビゲーション |
| `useSearchInput` | `src/hooks/useSearchInput.ts` | 検索入力の管理 |
| `useHistorySearch` | `src/hooks/useHistorySearch.ts` | コマンド履歴検索 |
| `useInputBuffer` | `src/hooks/useInputBuffer.ts` | 入力バッファ管理 |
| `useTypeahead` | `src/hooks/useTypeahead.tsx` | 先行入力（タイプアヘッド）処理 |
| `usePasteHandler` | `src/hooks/usePasteHandler.ts` | ペースト操作のハンドリング |
| `useCopyOnSelect` | `src/hooks/useCopyOnSelect.ts` | 選択テキストの自動コピー |
| `useVirtualScroll` | `src/hooks/useVirtualScroll.ts` | 仮想スクロール制御 |
| `useBlink` | `src/hooks/useBlink.ts` | カーソル点滅制御 |
| `useDoublePress` | `src/hooks/useDoublePress.ts` | ダブルプレス検出 |

### キーバインディング

| Hook | ファイル | 概要 |
|------|---------|------|
| `useCommandKeybindings` | `src/hooks/useCommandKeybindings.tsx` | コマンド用キーバインド登録 |
| `useGlobalKeybindings` | `src/hooks/useGlobalKeybindings.tsx` | グローバルキーバインド登録 |
| `useExitOnCtrlCD` | `src/hooks/useExitOnCtrlCD.ts` | Ctrl+C/Dによる終了処理 |
| `useExitOnCtrlCDWithKeybindings` | `src/hooks/useExitOnCtrlCDWithKeybindings.ts` | キーバインド連携の終了処理 |

### ツール・コマンド

| Hook | ファイル | 概要 |
|------|---------|------|
| `useMergedTools` | `src/hooks/useMergedTools.ts` | ツール定義のマージ |
| `useMergedCommands` | `src/hooks/useMergedCommands.ts` | コマンド定義のマージ |
| `useMergedClients` | `src/hooks/useMergedClients.ts` | クライアント定義のマージ |
| `useCommandQueue` | `src/hooks/useCommandQueue.ts` | コマンドキュー管理 |
| `useQueueProcessor` | `src/hooks/useQueueProcessor.ts` | キュー処理の実行制御 |
| `useManagePlugins` | `src/hooks/useManagePlugins.ts` | プラグイン管理 |

### IDE連携

| Hook | ファイル | 概要 |
|------|---------|------|
| `useIDEIntegration` | `src/hooks/useIDEIntegration.tsx` | IDE自動接続とMCP設定 |
| `useIdeAtMentioned` | `src/hooks/useIdeAtMentioned.ts` | IDE側からのメンション処理 |
| `useIdeConnectionStatus` | `src/hooks/useIdeConnectionStatus.ts` | IDE接続状態の監視 |
| `useIdeLogging` | `src/hooks/useIdeLogging.ts` | IDEログ統合 |
| `useIdeSelection` | `src/hooks/useIdeSelection.ts` | IDE側での選択テキスト取得 |
| `useDiffInIDE` | `src/hooks/useDiffInIDE.ts` | IDE内でのdiff表示 |
| `useDiffData` | `src/hooks/useDiffData.ts` | diffデータの管理 |
| `useLspPluginRecommendation` | `src/hooks/useLspPluginRecommendation.tsx` | LSPプラグインの推奨 |

### 音声

| Hook | ファイル | 概要 |
|------|---------|------|
| `useVoice` | `src/hooks/useVoice.ts` | ホールド・トゥ・トーク音声入力のコアhook |
| `useVoiceEnabled` | `src/hooks/useVoiceEnabled.ts` | 音声機能の有効/無効判定 |
| `useVoiceIntegration` | `src/hooks/useVoiceIntegration.tsx` | 音声機能のUI統合 |

`useVoice`はAnthropicの`voice_stream`エンドポイントを使用したSTT（Speech-to-Text）を実装している。キーバインドを押し続けて録音し、離すと自動送信される。

### セッション・状態管理

| Hook | ファイル | 概要 |
|------|---------|------|
| `useSettings` | `src/hooks/useSettings.ts` | 設定値の読み取り |
| `useSettingsChange` | `src/hooks/useSettingsChange.ts` | 設定変更の監視 |
| `useAssistantHistory` | `src/hooks/useAssistantHistory.ts` | アシスタント会話履歴 |
| `useSessionBackgrounding` | `src/hooks/useSessionBackgrounding.ts` | セッションのバックグラウンド化 |
| `useRemoteSession` | `src/hooks/useRemoteSession.ts` | リモートセッション管理 |
| `useSSHSession` | `src/hooks/useSSHSession.ts` | SSHセッション管理 |
| `useTerminalSize` | `src/hooks/useTerminalSize.ts` | ターミナルサイズの検出と監視 |
| `useMemoryUsage` | `src/hooks/useMemoryUsage.ts` | メモリ使用量の監視 |
| `useElapsedTime` | `src/hooks/useElapsedTime.ts` | 経過時間の計測 |

### ネットワーク・API

| Hook | ファイル | 概要 |
|------|---------|------|
| `useApiKeyVerification` | `src/hooks/useApiKeyVerification.ts` | APIキーの検証 |
| `useDirectConnect` | `src/hooks/useDirectConnect.ts` | 直接接続の管理 |
| `useMainLoopModel` | `src/hooks/useMainLoopModel.ts` | メインループのモデル選択 |
| `useDynamicConfig` | `src/hooks/useDynamicConfig.ts` | 動的設定の取得 |
| `useInboxPoller` | `src/hooks/useInboxPoller.ts` | 受信ボックスのポーリング |
| `useMailboxBridge` | `src/hooks/useMailboxBridge.ts` | メールボックスブリッジ |

### 通知・サジェスト

| Hook | ファイル | 概要 |
|------|---------|------|
| `useUpdateNotification` | `src/hooks/useUpdateNotification.ts` | アプリ更新通知 |
| `useNotifyAfterTimeout` | `src/hooks/useNotifyAfterTimeout.ts` | タイムアウト後の通知 |
| `useChromeExtensionNotification` | `src/hooks/useChromeExtensionNotification.tsx` | Chrome拡張通知 |
| `useOfficialMarketplaceNotification` | `src/hooks/useOfficialMarketplaceNotification.tsx` | マーケットプレイス通知 |
| `usePromptSuggestion` | `src/hooks/usePromptSuggestion.ts` | プロンプトサジェスト |
| `useClaudeCodeHintRecommendation` | `src/hooks/useClaudeCodeHintRecommendation.tsx` | ヒント推奨表示 |
| `useClipboardImageHint` | `src/hooks/useClipboardImageHint.ts` | クリップボード画像のヒント |
| `useIssueFlagBanner` | `src/hooks/useIssueFlagBanner.ts` | 問題フラグバナー |

### Swarm・タスク管理

| Hook | ファイル | 概要 |
|------|---------|------|
| `useSwarmInitialization` | `src/hooks/useSwarmInitialization.ts` | Swarmモードの初期化 |
| `useScheduledTasks` | `src/hooks/useScheduledTasks.ts` | スケジュールタスク管理 |
| `useTasksV2` | `src/hooks/useTasksV2.ts` | タスク管理v2 |
| `useTaskListWatcher` | `src/hooks/useTaskListWatcher.ts` | タスクリストの監視 |
| `useBackgroundTaskNavigation` | `src/hooks/useBackgroundTaskNavigation.ts` | バックグラウンドタスクのナビゲーション |
| `useTeammateViewAutoExit` | `src/hooks/useTeammateViewAutoExit.ts` | チームメイトビューの自動終了 |

### その他ユーティリティ

| Hook | ファイル | 概要 |
|------|---------|------|
| `useAfterFirstRender` | `src/hooks/useAfterFirstRender.ts` | 初回レンダリング後のフラグ |
| `useTimeout` | `src/hooks/useTimeout.ts` | タイムアウト管理 |
| `useMinDisplayTime` | `src/hooks/useMinDisplayTime.ts` | 最小表示時間の保証 |
| `useCancelRequest` | `src/hooks/useCancelRequest.ts` | リクエストキャンセル |
| `useLogMessages` | `src/hooks/useLogMessages.ts` | ログメッセージ管理 |
| `useTurnDiffs` | `src/hooks/useTurnDiffs.ts` | ターンごとのdiff管理 |
| `useFileHistorySnapshotInit` | `src/hooks/useFileHistorySnapshotInit.ts` | ファイル履歴スナップショット初期化 |
| `usePrStatus` | `src/hooks/usePrStatus.ts` | PRステータス監視 |
| `useAwaySummary` | `src/hooks/useAwaySummary.ts` | 離席時サマリー |
| `useTeleportResume` | `src/hooks/useTeleportResume.tsx` | テレポート再開 |
| `useReplBridge` | `src/hooks/useReplBridge.tsx` | REPLブリッジ |
| `useSkillImprovementSurvey` | `src/hooks/useSkillImprovementSurvey.ts` | スキル改善サーベイ |
| `useSkillsChange` | `src/hooks/useSkillsChange.ts` | スキル変更検出 |
| `useDeferredHookMessages` | `src/hooks/useDeferredHookMessages.ts` | 遅延hookメッセージ |
| `usePromptsFromClaudeInChrome` | `src/hooks/usePromptsFromClaudeInChrome.tsx` | Chrome経由プロンプト |

サジェスト関連ユーティリティとして `src/hooks/fileSuggestions.ts`、`src/hooks/unifiedSuggestions.ts`、`src/hooks/renderPlaceholder.ts` も存在する。

---

## 主要Hooks解説

### useCanUseTool

**ファイル:** `src/hooks/useCanUseTool.tsx`

ツール実行前の権限チェックを担うhook。型シグネチャ:

```typescript
type CanUseToolFn<Input> = (
  tool: ToolType,
  input: Input,
  toolUseContext: ToolUseContext,
  assistantMessage: AssistantMessage,
  toolUseID: string,
  forceDecision?: PermissionDecision<Input>
) => Promise<PermissionDecision<Input>>
```

処理フロー:
1. `hasPermissionsToUseTool()` で設定ベースの権限チェック
2. `allow` の場合 -- 即座に許可を返却（classifier承認の記録を含む）
3. `deny` の場合 -- 拒否を返却、auto-modeの場合は拒否通知を表示
4. `ask` の場合 -- ユーザーに対話的に確認を求める

サブハンドラは `src/hooks/toolPermission/handlers/` 以下で、coordinatorモード、interactiveモード、swarm workerモードの3つのパターンに分岐する。

### useCommandKeybindings

**ファイル:** `src/hooks/useCommandKeybindings.tsx`

`command:*` アクションパターンに基づいてキーバインドハンドラを登録するコンポーネント。キーバインド設定から `command:` プレフィックスを持つアクションを抽出し、対応するスラッシュコマンド（例: `command:commit` -> `/commit`）を即座に実行する。

特徴:
- キーバインド経由のコマンドは「即時実行」として扱われる
- 入力中のテキストは保持される（プロンプトがクリアされない）
- モーダルオーバーレイがアクティブな場合は無効化される

### useGlobalKeybindings

**ファイル:** `src/hooks/useGlobalKeybindings.tsx`

グローバルキーバインドのハンドラを登録する。主な対応キー:
- `Ctrl+T`: タスクリストの切り替え（none -> tasks -> teammates -> none のサイクル）
- `Ctrl+O`: トランスクリプトモードの切り替え
- `Ctrl+E`: トランスクリプト内全メッセージ表示の切り替え
- `Ctrl+C` / `Escape`: トランスクリプトモードの終了

### useVimInput

**ファイル:** `src/hooks/useVimInput.ts`

`useTextInput`を拡張してVimモードを追加するhook。INSERTモードとNORMALモードの切り替え、状態マシン遷移、dot-repeat（`.` コマンド）のための変更記録などを管理する。

---

## キーバインディングシステム

キーバインディングシステムは `src/keybindings/` ディレクトリに実装されている。

### アーキテクチャ

```
~/.claude/keybindings.json  (ユーザー設定)
         │
         ▼
loadUserBindings.ts         ユーザー設定の読み込み
         │
         ▼
defaultBindings.ts          デフォルト設定とマージ
         │
         ▼
resolver.ts                 キー入力とアクションの解決
         │
         ▼
match.ts                    パターンマッチング
         │
         ▼
KeybindingContext.tsx        Reactコンテキスト経由で全体に提供
         │
         ▼
useKeybinding.ts            各コンポーネントでのハンドラ登録
```

### 主要ファイル

| ファイル | 役割 |
|---------|------|
| `src/keybindings/defaultBindings.ts` | デフォルトキーバインド定義 |
| `src/keybindings/loadUserBindings.ts` | `~/.claude/keybindings.json` の読み込み |
| `src/keybindings/schema.ts` | Zodスキーマによるバリデーション |
| `src/keybindings/parser.ts` | キー入力文字列のパース |
| `src/keybindings/match.ts` | キーシーケンスのマッチング |
| `src/keybindings/resolver.ts` | アクションの解決 |
| `src/keybindings/reservedShortcuts.ts` | 予約済みショートカットの定義 |
| `src/keybindings/shortcutFormat.ts` | ショートカット表示フォーマット |
| `src/keybindings/template.ts` | テンプレート処理 |
| `src/keybindings/validate.ts` | 設定のバリデーション |
| `src/keybindings/KeybindingContext.tsx` | Reactコンテキストプロバイダ |
| `src/keybindings/KeybindingProviderSetup.tsx` | プロバイダのセットアップ |
| `src/keybindings/useKeybinding.ts` | キーバインド登録hook |
| `src/keybindings/useShortcutDisplay.ts` | ショートカット表示用hook |

### コンテキスト

キーバインドは「コンテキスト」に基づいてスコープ付けされる（`src/keybindings/schema.ts`）:

| コンテキスト | 説明 |
|-------------|------|
| `Global` | どこでも有効 |
| `Chat` | チャット入力フォーカス時 |
| `Autocomplete` | オートコンプリートメニュー表示時 |
| `Confirmation` | 確認/権限ダイアログ表示時 |
| `Help` | ヘルプオーバーレイ表示時 |
| `Transcript` | トランスクリプト表示時 |
| `HistorySearch` | 履歴検索時（Ctrl+R） |
| `Task` | タスク/エージェント実行時 |
| `ThemePicker` | テーマピッカー表示時 |
| `Settings` | 設定メニュー表示時 |
| `Tabs` | タブナビゲーション時 |
| `DiffDialog` | diffダイアログ表示時 |
| `ModelPicker` | モデルピッカー表示時 |
| `Select` | セレクトコンポーネントフォーカス時 |

### デフォルトキーバインド

`src/keybindings/defaultBindings.ts` で定義されている主要バインド:

**Global:**
- `Ctrl+C` -- `app:interrupt`（予約済み、再バインド不可）
- `Ctrl+D` -- `app:exit`（予約済み、再バインド不可）
- `Ctrl+L` -- `app:redraw`
- `Ctrl+T` -- `app:toggleTodos`
- `Ctrl+O` -- `app:toggleTranscript`
- `Ctrl+R` -- `history:search`
- `Ctrl+Shift+F` / `Cmd+Shift+F` -- `app:globalSearch`
- `Ctrl+Shift+P` / `Cmd+Shift+P` -- `app:quickOpen`

**Chat:**
- `Escape` -- `chat:cancel`
- `Ctrl+X Ctrl+K` -- `chat:killAgents`（コードキー）
- `Shift+Tab` -- `chat:cycleMode`（Windowsでは`Meta+M`）
- `Meta+P` -- `chat:modelPicker`
- `Meta+O` -- `chat:fastMode`
- `Meta+T` -- `chat:thinkingToggle`
- `Enter` -- `chat:submit`
- `Up` / `Down` -- `history:previous` / `history:next`

プラットフォーム別の差異:
- 画像ペースト: Windowsでは `Alt+V`、その他は `Ctrl+V`
- モードサイクル: Windows（VTモード非対応）では `Meta+M`、その他は `Shift+Tab`

### アクション一覧

`src/keybindings/schema.ts` の `KEYBINDING_ACTIONS` に全アクションが定義されている。主要カテゴリ:

- **app:** `interrupt`, `exit`, `toggleTodos`, `toggleTranscript`, `redraw`, `globalSearch`, `quickOpen`
- **chat:** `cancel`, `killAgents`, `cycleMode`, `modelPicker`, `submit`, `newline`, `undo`, `imagePaste`
- **autocomplete:** `accept`, `dismiss`, `previous`, `next`
- **confirm:** `yes`, `no`, `previous`, `next`, `toggle`
- **transcript:** `toggleShowAll`, `exit`
- **historySearch:** `next`, `accept`, `cancel`, `execute`
- **task:** `background`

---

## Vimモードの実装

Vimモードは `src/vim/` ディレクトリに状態マシンとして実装されている。

### ディレクトリ構成

| ファイル | 役割 |
|---------|------|
| `src/vim/types.ts` | 状態マシンの型定義 |
| `src/vim/transitions.ts` | 状態遷移テーブル |
| `src/vim/motions.ts` | モーション（移動コマンド）の実装 |
| `src/vim/operators.ts` | オペレータ（操作コマンド）の実装 |
| `src/vim/textObjects.ts` | テキストオブジェクトの実装 |

コマンド有効化: `src/commands/vim/vim.ts`、`src/commands/vim/index.ts`

### 状態マシン設計

Vimモードは2つのモードで構成される:

```
VimState
├── INSERT -- insertedText を追跡（dot-repeat用）
└── NORMAL -- CommandState状態マシン
```

NORMALモード内の`CommandState`遷移図（`src/vim/types.ts`より）:

```
idle ──┬─[d/c/y]──► operator
       ├─[1-9]────► count
       ├─[fFtT]───► find
       ├─[g]──────► g
       ├─[r]──────► replace
       └─[><]─────► indent

operator ─┬─[motion]──► execute
           ├─[0-9]────► operatorCount
           ├─[ia]─────► operatorTextObj
           └─[fFtT]───► operatorFind
```

### CommandState型

`src/vim/types.ts` で定義されている各状態:

```typescript
type CommandState =
  | { type: 'idle' }
  | { type: 'count'; digits: string }
  | { type: 'operator'; op: Operator; count: number }
  | { type: 'operatorCount'; op: Operator; count: number; digits: string }
  | { type: 'operatorFind'; op: Operator; count: number; find: FindType }
  | { type: 'operatorTextObj'; op: Operator; count: number; scope: TextObjScope }
  | { type: 'find'; find: FindType; count: number }
  | { type: 'g'; count: number }
  | { type: 'operatorG'; op: Operator; count: number }
  | { type: 'replace'; count: number }
  | { type: 'indent'; dir: '>' | '<'; count: number }
```

### 遷移テーブル

`src/vim/transitions.ts` がメインのディスパッチ関数を提供する:

```typescript
function transition(
  state: CommandState,
  input: string,
  ctx: TransitionContext
): TransitionResult
```

`TransitionResult`は次の状態（`next`）と実行する関数（`execute`）の組み合わせで構成される。各状態に対応する`fromXxx`関数が個別の遷移ロジックを実装している。

### サポートされている機能

- **オペレータ:** `d`（delete）、`c`（change）、`y`（yank）
- **モーション:** `h`, `l`, `w`, `b`, `e`, `0`, `$`, `^` など（SIMPLE_MOTIONS定数で定義）
- **検索モーション:** `f`, `F`, `t`, `T`
- **テキストオブジェクト:** `i`（inner）/ `a`（around）スコープ
- **その他:** `x`（delete char）、`r`（replace）、`J`（join）、`~`（toggle case）、`p`（paste）、`>`/`<`（indent）、`u`（undo）、`.`（dot-repeat）
- **PersistentState:** ヤンクレジスタ、最後の検索、最後の変更がコマンド間で保持される

### useVimInput hookとの連携

`src/hooks/useVimInput.ts` は `useTextInput` を内部で使用しつつ、Vimモードの状態管理を上に重ねる:

1. `vimStateRef` でVim状態を管理（React refで高頻度更新に対応）
2. `persistentRef` でヤンクレジスタや最後の変更を保持
3. `switchToInsertMode` / `switchToNormalMode` でモード切り替え
4. NORMALモードからINSERTモードに戻る際、Vimの慣例に従いカーソルを1文字左に移動
5. INSERTモードで入力されたテキストは `insertedText` として記録され、`.` コマンドによるリピートに使用される
