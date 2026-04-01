# 状態管理アーキテクチャ

Claude Code v2.1.88 のUI層は、カスタム軽量ストアとReact Contextプロバイダーを組み合わせて状態管理を行っている。Zustandに類似した設計だが、完全に独自実装であり、`useSyncExternalStore`によるReact 18統合を特徴とする。

---

## 1. Storeの基盤: `createStore`

**ファイル:** `src/state/store.ts`

アプリケーション全体の状態管理の基盤となる汎用ストア実装。わずか34行で構成される極めてシンプルな設計。

```typescript
export type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}
```

### 主要な設計判断

| 特性 | 説明 |
|------|------|
| **イミュータブル更新** | `setState`はupdater関数を受け取り、`Object.is`で同一性を検査。同一オブジェクトが返された場合、リスナー通知をスキップ |
| **onChange コールバック** | ストア生成時にオプショナルな`onChange`を受け取り、状態変更の副作用（永続化等）を一元管理 |
| **サブスクリプション** | `Set<Listener>`によるO(1)の登録/解除。返り値のクリーンアップ関数でReact統合 |

---

## 2. AppState: アプリケーション状態の全体像

**ファイル:** `src/state/AppStateStore.ts` (型定義 + デフォルト値)

`AppState`型は`DeepImmutable<{...}>`で定義され、450行以上に及ぶ巨大な状態ツリーである。主要なセクションを以下に分類する。

### 2.1 コア設定・モデル

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `settings` | `SettingsJson` | ユーザー設定（`settings.json`由来） |
| `verbose` | `boolean` | 詳細ログ表示フラグ |
| `mainLoopModel` | `ModelSetting` | 使用中のモデル名（エイリアスまたはフルネーム） |
| `mainLoopModelForSession` | `ModelSetting` | セッションスコープのモデルオーバーライド |
| `toolPermissionContext` | `ToolPermissionContext` | 権限モード（`default`, `plan`, `bubble`等） |
| `thinkingEnabled` | `boolean \| undefined` | thinking機能の有効/無効 |
| `effortValue` | `EffortValue` | 推論努力レベル |
| `fastMode` | `boolean` | 高速モード |

### 2.2 UI表示状態

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `expandedView` | `'none' \| 'tasks' \| 'teammates'` | 展開パネルの種類 |
| `isBriefOnly` | `boolean` | ブリーフ表示モード |
| `footerSelection` | `FooterItem \| null` | フォーカス中のフッターピル |
| `statusLineText` | `string \| undefined` | ステータスライン表示文字列 |
| `activeOverlays` | `ReadonlySet<string>` | アクティブなオーバーレイ（Escキー制御用） |
| `coordinatorTaskIndex` | `number` | CoordinatorTaskPanelの選択インデックス |

### 2.3 MCP・プラグインシステム

```typescript
mcp: {
  clients: MCPServerConnection[]  // 接続中MCPサーバー一覧
  tools: Tool[]                    // MCPツール一覧
  commands: Command[]              // MCPコマンド一覧
  resources: Record<string, ServerResource[]>
  pluginReconnectKey: number       // /reload-plugins でインクリメント
}
plugins: {
  enabled: LoadedPlugin[]
  disabled: LoadedPlugin[]
  commands: Command[]
  errors: PluginError[]
  installationStatus: { marketplaces: [...], plugins: [...] }
  needsRefresh: boolean
}
```

### 2.4 タスク・エージェント管理

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `tasks` | `{ [taskId: string]: TaskState }` | 全タスクのレジストリ（`DeepImmutable`から除外） |
| `agentNameRegistry` | `Map<string, AgentId>` | エージェント名→ID の逆引きマップ |
| `foregroundedTaskId` | `string \| undefined` | フォアグラウンドのタスクID |
| `viewingAgentTaskId` | `string \| undefined` | 閲覧中のエージェントタスクID |
| `teamContext` | オブジェクト | チーム/スウォーム構成情報 |

### 2.5 投機実行 (Speculation)

```typescript
type SpeculationState =
  | { status: 'idle' }
  | { status: 'active'; id: string; abort: () => void; startTime: number;
      messagesRef: { current: Message[] };  // ミュータブルref（配列コピー回避）
      writtenPathsRef: { current: Set<string> };
      boundary: CompletionBoundary | null;
      // ... 他フィールド
    }
```

`SpeculationState`はパフォーマンス上の理由から内部に`Ref`パターン（ミュータブル参照）を使用。メッセージ配列の展開コストを回避している。

### 2.6 ブリッジ・リモート接続

`replBridge*`プレフィックスのフィールド群（15個以上）が常時接続ブリッジの状態を管理:

- `replBridgeEnabled` / `replBridgeConnected` / `replBridgeSessionActive`
- `replBridgeConnectUrl` / `replBridgeSessionUrl`
- `replBridgeReconnecting` / `replBridgeError`

### 2.7 デフォルト状態

`getDefaultAppState()` （`src/state/AppStateStore.ts` L456-569）が初期状態を生成。注目点:

- `thinkingEnabled` は `shouldEnableThinkingByDefault()` で動的に決定
- `toolPermissionContext.mode` はチームメイト設定に基づき `'plan'` または `'default'` を選択
- `sessionHooks` は `Map` で初期化（`DeepImmutable`の例外）

---

## 3. AppStateProvider と React統合

**ファイル:** `src/state/AppState.tsx`

### 3.1 プロバイダーの階層構造

`AppStateProvider`はアプリケーションのルートに位置し、以下のコンテキストをネストして提供:

```
HasAppStateContext.Provider (二重ネスト防止ガード)
  └── AppStoreContext.Provider (Store<AppState> を提供)
        └── MailboxProvider (メッセージボックス)
              └── VoiceProvider (音声入力状態 — feature('VOICE_MODE')でDCE)
```

### 3.2 状態の購読: `useAppState`

`src/state/AppState.tsx` L142-163

```typescript
export function useAppState<T>(selector: (state: AppState) => T): T {
  const store = useAppStore()
  const get = () => selector(store.getState())
  return useSyncExternalStore(store.subscribe, get, get)
}
```

**使用パターン:**
```typescript
// 良い例: 既存のサブオブジェクト参照を返す
const { text, promptId } = useAppState(s => s.promptSuggestion)

// 悪い例: 毎回新しいオブジェクトを生成する（常に再レンダリング）
const data = useAppState(s => ({ a: s.verbose, b: s.model })) // NG
```

### 3.3 その他のフック

| フック | 説明 |
|--------|------|
| `useSetAppState()` | updater関数を返す。購読なし。安定した参照 |
| `useAppStateStore()` | Store自体を返す（非React コードへの受け渡し用） |
| `useAppStateMaybeOutsideOfProvider()` | Provider外でも安全に使えるバージョン（`undefined`を返す） |

---

## 4. 状態変更のサブスクリプション: `onChangeAppState`

**ファイル:** `src/state/onChangeAppState.ts`

`createStore`のonChangeコールバックとして登録され、すべての状態変更を監視して副作用を実行する。

### 4.1 権限モードの同期 (L43-92)

```typescript
if (prevMode !== newMode) {
  const prevExternal = toExternalPermissionMode(prevMode)
  const newExternal = toExternalPermissionMode(newMode)
  if (prevExternal !== newExternal) {
    notifySessionMetadataChanged({ permission_mode: newExternal, ... })
  }
  notifyPermissionModeChanged(newMode)
}
```

これは8箇所以上ある権限モード変更パスの**単一チョークポイント**として機能する。以前はCCR（Claude Code Remote）への通知漏れが多発していた箇所を一元化した設計。

### 4.2 モデル設定の永続化 (L94-112)

`mainLoopModel`の変更を検出し、`updateSettingsForSource('userSettings', { model: ... })`で設定ファイルに反映。

### 4.3 表示設定の永続化 (L115-128)

`expandedView`の変更を`saveGlobalConfig()`でグローバル設定に保存:
- `showExpandedTodos` / `showSpinnerTree` フラグとして後方互換を維持

### 4.4 verbose設定の永続化 (L131-140)

### 4.5 認証キャッシュのクリア (L155-170)

`settings`オブジェクト自体の変更を検出し:
- `clearApiKeyHelperCache()` / `clearAwsCredentialsCache()` / `clearGcpCredentialsCache()` を呼び出し
- `settings.env`の変更時に`applyConfigEnvironmentVariables()`で環境変数を再適用

---

## 5. セレクター

**ファイル:** `src/state/selectors.ts`

純粋関数として状態から派生データを抽出する。

### `getViewedTeammateTask`
```typescript
function getViewedTeammateTask(
  appState: Pick<AppState, 'viewingAgentTaskId' | 'tasks'>
): InProcessTeammateTaskState | undefined
```
`viewingAgentTaskId`がセットされ、対応タスクが`InProcessTeammateTask`型の場合にのみ返す。

### `getActiveAgentForInput`
```typescript
type ActiveAgentForInput =
  | { type: 'leader' }
  | { type: 'viewed'; task: InProcessTeammateTaskState }
  | { type: 'named_agent'; task: LocalAgentTaskState }
```
ユーザー入力のルーティング先を判別。リーダー / 閲覧中エージェント / 名前付きエージェント の判別型ユニオンを返す。

---

## 6. React Contextプロバイダー一覧

`src/context/`ディレクトリに9つの特化型コンテキストが存在する。

### 6.1 notifications (`src/context/notifications.tsx`)

**役割:** トースト通知のキュー管理と表示制御

| 型 | 説明 |
|-----|------|
| `Notification` | `TextNotification \| JSXNotification` |
| `Priority` | `'low' \| 'medium' \| 'high' \| 'immediate'` |

- 通知はAppState内の`notifications.queue`と`notifications.current`で管理
- `fold`関数により同一キーの通知をマージ可能（`Array.reduce`ライク）
- `invalidates`フィールドで他の通知を無効化
- デフォルトタイムアウト: 8000ms
- `immediate`優先度の通知は既存通知を即座に置換

**提供フック:**
- `useNotifications()` → `{ addNotification, removeNotification }`

### 6.2 stats (`src/context/stats.tsx`)

**役割:** メトリクス収集（カウンター、ヒストグラム、セット）

```typescript
type StatsStore = {
  increment(name: string, value?: number): void  // カウンター
  set(name: string, value: number): void          // ゲージ
  observe(name: string, value: number): void       // ヒストグラム（レザバーサンプリング）
  add(name: string, value: string): void           // 一意値セット
  getAll(): Record<string, number>
}
```

- ヒストグラムは**Algorithm R**（レザバーサンプリング）を使用、サイズ上限1024
- パーセンタイル計算（p50/p90/p99）に対応
- `StatsProvider`でコンテキスト提供、`useStats()`で取得

### 6.3 modalContext (`src/context/modalContext.tsx`)

**役割:** FullscreenLayoutのモーダルスロットのサイズ情報提供

```typescript
type ModalCtx = {
  rows: number
  columns: number
  scrollRef: RefObject<ScrollBoxHandle | null> | null
}
```

- `useIsInsideModal()` — モーダル内かどうかの判定
- `useModalOrTerminalSize(fallback)` — モーダル内ならモーダルサイズ、外ならfallbackを返す
- `useModalScrollRef()` — モーダルのスクロールハンドル取得

### 6.4 overlayContext (`src/context/overlayContext.tsx`)

**役割:** Escapeキーの協調制御（オーバーレイ vs リクエストキャンセル）

- `useRegisterOverlay(id, enabled?)` — オーバーレイの自動登録/解除（マウント/アンマウント連動）
- `useIsOverlayActive()` — モーダルオーバーレイがアクティブかを判定
- `NON_MODAL_OVERLAYS` — `autocomplete`等のTextInputフォーカスを奪わないオーバーレイ
- AppStateの`activeOverlays: ReadonlySet<string>`で管理

### 6.5 voice (`src/context/voice.tsx`)

**役割:** 音声入力の状態管理（`feature('VOICE_MODE')`でDCE対象）

```typescript
type VoiceState = {
  voiceState: 'idle' | 'recording' | 'processing'
  voiceError: string | null
  voiceInterimTranscript: string
  voiceAudioLevels: number[]
  voiceWarmingUp: boolean
}
```

- 独自の`Store<VoiceState>`を使用（AppStateとは独立）
- `useVoiceState(selector)` — スライス購読（`useSyncExternalStore`ベース）
- `useSetVoiceState()` — 同期的setter（`VoiceKeybindingHandler`が即座に読み取る）
- `useGetVoiceState()` — コールバック内での同期読み取り

### 6.6 QueuedMessageContext (`src/context/QueuedMessageContext.tsx`)

**役割:** キューされたメッセージのレイアウト情報を子コンポーネントに伝達

```typescript
type QueuedMessageContextValue = {
  isQueued: boolean
  isFirst: boolean
  paddingWidth: number  // コンテナパディング幅（paddingX={2}なら4）
}
```

- `QueuedMessageProvider` — `Box paddingX={padding}`でラップ
- ブリーフモードではパディングをゼロに（二重インデント防止）

### 6.7 mailbox (`src/context/mailbox.tsx`)

**役割:** プロセス間メッセージ配信（スウォーム/チーム間通信）

- `MailboxProvider` — `useMemo(() => new Mailbox(), [])`で一度だけ生成
- `useMailbox()` — `Mailbox`インスタンスを取得
- `Mailbox`の実装は`src/utils/mailbox.ts`に分離

### 6.8 fpsMetrics (`src/context/fpsMetrics.tsx`)

**役割:** FPSメトリクスのゲッター関数を提供

- `FpsMetricsProvider` — `getFpsMetrics: () => FpsMetrics | undefined`を配信
- `useFpsMetrics()` — ゲッター関数を返す（呼び出し時に最新値を取得）
- DevBar等のパフォーマンス表示で使用

### 6.9 promptOverlayContext (`src/context/promptOverlayContext.tsx`)

**役割:** プロンプト上部に浮遊するオーバーレイ（サジェスト・ダイアログ）のポータル

```typescript
type PromptOverlayData = {
  suggestions: SuggestionItem[]
  selectedSuggestion: number
  maxColumnWidth?: number
}
```

FullscreenLayoutの`overflowY:hidden`クリップを回避するため、2つのチャネルを提供:
- `useSetPromptOverlay(data)` — スラッシュコマンドサジェストデータ
- `useSetPromptOverlayDialog(node)` — 任意のダイアログノード

data/setterコンテキストのペアに分割し、書き込み側が自身の書き込みで再レンダリングしない設計。

---

## 7. 状態の永続化フロー

状態の永続化は`onChangeAppState`内の差分検出で駆動される:

```
AppState変更
  ↓ setState()
  ↓ Object.is チェック
  ↓ onChange コールバック
  ├── permission mode → notifySessionMetadataChanged() → CCR/SDK
  ├── mainLoopModel → updateSettingsForSource() → settings.json
  ├── expandedView → saveGlobalConfig() → global config
  ├── verbose → saveGlobalConfig() → global config
  └── settings変更 → clearAuthCaches() + applyConfigEnvVars()
```

### グローバル設定 vs ユーザー設定

| 永続化先 | 管理対象 | 関数 |
|---------|---------|------|
| グローバル設定 (`~/.claude/config.json`) | `verbose`, `expandedView`, `tungstenPanelVisible` | `saveGlobalConfig()` |
| ユーザー設定 (`settings.json`) | `mainLoopModel`, `settings` | `updateSettingsForSource()` |
| CCR external_metadata | `permission_mode`, `is_ultraplan_mode` | `notifySessionMetadataChanged()` |

---

## 8. Appコンポーネントでのプロバイダー統合

**ファイル:** `src/components/App.tsx`

トップレベルの`App`コンポーネントがプロバイダーを統合:

```
FpsMetricsProvider
  └── StatsProvider
        └── AppStateProvider (+ MailboxProvider + VoiceProvider 内包)
              └── children (REPL等)
```

`onChangeAppState`は`App.tsx`で`AppStateProvider`に注入され、ストア生成時にバインドされる。
