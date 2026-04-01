# MCP (Model Context Protocol) 統合

> Claude Code v2.1.88 ソースコード解析 - `src/services/mcp/` ディレクトリ

## 概要

Claude Code は Model Context Protocol (MCP) を通じて外部ツールサーバーと統合する。MCP サーバーは stdio、SSE、HTTP、WebSocket、SDK などの複数トランスポートをサポートし、ツール、プロンプト（コマンド）、リソースを動的に提供する。

## ファイル構成

| ファイル | 役割 |
|---|---|
| `MCPConnectionManager.tsx` | React コンテキストプロバイダ。再接続・トグル機能を子コンポーネントに提供 |
| `useManageMCPConnections.ts` | MCP 接続のライフサイクル管理フック |
| `client.ts` | MCP クライアント生成、ツール/コマンド/リソースの取得 |
| `config.ts` | MCP サーバー設定の読み込み・書き込み |
| `types.ts` | 型定義・Zod スキーマ |
| `auth.ts` | OAuth 認証フロー（PKCE、トークンリフレッシュ） |
| `normalization.ts` | サーバー名の正規化 |
| `utils.ts` | ユーティリティ関数（フィルタリング、分類） |
| `headersHelper.ts` | カスタムヘッダーの取得 |
| `elicitationHandler.ts` | MCP Elicitation（対話型入力要求）処理 |
| `channelNotification.ts` | チャネル通知処理 |
| `channelPermissions.ts` | チャネル権限管理 |
| `claudeai.ts` | Claude.ai プロキシサーバー設定取得 |
| `officialRegistry.ts` | 公式 MCP レジストリ |
| `envExpansion.ts` | 環境変数展開 |
| `oauthPort.ts` | OAuth リダイレクト URI / ポート管理 |
| `xaa.ts` / `xaaIdpLogin.ts` | Cross-App Access (XAA) 認証 |
| `SdkControlTransport.ts` | SDK 制御用トランスポート |
| `InProcessTransport.ts` | インプロセストランスポート |
| `vscodeSdkMcp.ts` | VS Code SDK MCP 統合 |

## MCP サーバーの型定義 (`types.ts`)

### トランスポート種別

```typescript
export const TransportSchema = z.enum(['stdio', 'sse', 'sse-ide', 'http', 'ws', 'sdk'])
```

### サーバー設定スキーマ

| 型 | 説明 | 主要フィールド |
|---|---|---|
| `McpStdioServerConfig` | ローカルプロセス（stdio） | `command`, `args`, `env` |
| `McpSSEServerConfig` | Server-Sent Events | `url`, `headers`, `oauth` |
| `McpSSEIDEServerConfig` | IDE 用 SSE（内部用） | `url`, `ideName` |
| `McpHTTPServerConfig` | Streamable HTTP | `url`, `headers`, `oauth` |
| `McpWebSocketServerConfig` | WebSocket | `url`, `headers` |
| `McpSdkServerConfig` | SDK 内蔵 | `name` |
| `McpClaudeAIProxyServerConfig` | Claude.ai プロキシ | `url`, `id` |

### 設定スコープ (`ConfigScope`)

```typescript
z.enum(['local', 'user', 'project', 'dynamic', 'enterprise', 'claudeai', 'managed'])
```

- **local**: `.mcp.json`（プロジェクトディレクトリ）
- **user**: `~/.claude/` ユーザー設定
- **project**: `.claude/` プロジェクト設定
- **enterprise**: 管理者が配布するマネージド設定
- **claudeai**: Claude.ai から取得するプロキシサーバー
- **dynamic**: ランタイム動的設定
- **managed**: マネージドプラグイン提供

### 接続状態

```typescript
type ConnectedMCPServer = {
  client: Client          // @modelcontextprotocol/sdk Client
  name: string
  type: 'connected'
  capabilities: ServerCapabilities
  serverInfo?: { name: string; version: string }
  instructions?: string   // サーバー提供のモデル向け指示
  config: ScopedMcpServerConfig
  cleanup: () => Promise<void>
}

type FailedMCPServer = {
  name: string
  type: 'failed'
  config: ScopedMcpServerConfig
  error?: string
}
```

## MCPサーバーのライフサイクル管理

### MCPConnectionManager (`MCPConnectionManager.tsx`)

React コンテキストプロバイダとして機能し、以下の操作を子コンポーネントに公開する:

- `reconnectMcpServer(serverName)` - サーバーの再接続
- `toggleMcpServer(serverName)` - サーバーの有効/無効切り替え

```typescript
export function MCPConnectionManager({ children, dynamicMcpConfig, isStrictMcpConfig }) {
  const { reconnectMcpServer, toggleMcpServer } = useManageMCPConnections(dynamicMcpConfig, isStrictMcpConfig)
  // ...
}
```

### useManageMCPConnections フック (`useManageMCPConnections.ts`)

MCP 接続の中核ロジックを管理する React フック:

**初期化と再接続:**
- 設定変更時に自動的にサーバーを再接続
- `MAX_RECONNECT_ATTEMPTS = 5`、指数バックオフ（`INITIAL_BACKOFF_MS = 1000`, `MAX_BACKOFF_MS = 30000`）
- プラグインの MCP サーバーも統合管理

**通知ハンドリング:**

MCP SDK の通知スキーマに基づき、サーバー側の変更を自動検出:

- `ToolListChangedNotificationSchema` - ツールリスト変更
- `ResourceListChangedNotificationSchema` - リソースリスト変更
- `PromptListChangedNotificationSchema` - プロンプトリスト変更

**チャネル権限 (Kairos):**

`channelPermissions.ts` と連携し、MCP チャネル経由のツール実行に対する権限制御を行う。`ChannelPermissionNotificationSchema` / `ChannelMessageNotificationSchema` で通知をリレーする。

### エラー管理

プラグインエラーの重複排除:

```typescript
function addErrorsToAppState(setAppState, newErrors: PluginError[]): void {
  // 既存エラーキーのセットを構築し、重複を除外
}
```

## 接続プーリングと管理

### MCP クライアント生成 (`client.ts`)

`client.ts` がトランスポート別のクライアント生成を行う:

**サポートトランスポート:**

1. **StdioClientTransport** - ローカルプロセスの stdio 通信
2. **SSEClientTransport** - Server-Sent Events
3. **StreamableHTTPClientTransport** - Streamable HTTP (新プロトコル)
4. **WebSocketTransport** - WebSocket（`src/utils/mcpWebSocketTransport.ts`）
5. **SdkControlClientTransport** - SDK 内部制御用

**セッション管理:**

- `McpSessionExpiredError` - セッション期限切れ検出（HTTP 404 + JSON-RPC -32001）
- `McpAuthError` - OAuth トークン失効時の再認証要求
- `clearServerCache()` でキャッシュされた接続をクリア

### ツール・コマンド・リソースの取得

```typescript
fetchToolsForClient(client, serverName, config)    // → Tool[]
fetchCommandsForClient(client, serverName)          // → Command[]
fetchResourcesForClient(client, serverName)         // → ServerResource[]
getMcpToolsCommandsAndResources(configs, isStrict)  // 全サーバーから一括取得
reconnectMcpServerImpl(serverName, configs)         // 再接続 + 全取得
```

### MCPTool クラス

`src/tools/MCPTool/MCPTool.ts` の `MCPTool` クラスが各 MCP ツールを `Tool` インターフェースにラップする。ツール名は `mcp__<normalizedServerName>__<toolName>` 形式で一意に識別される。

## リソースハンドリング

### リソース型

```typescript
type ServerResource = {
  server: string          // サーバー名
  resource: Resource      // MCP Resource 型
}
```

### リソース関連ツール

- `ListMcpResourcesTool` - MCP サーバーのリソース一覧表示
- `ReadMcpResourceTool` - MCP リソースの読み取り

### 出力処理

大規模なツール出力は以下の方法で処理される:

- `mcpContentNeedsTruncation()` - トランケーション必要性の判定
- `truncateMcpContentIfNeeded()` - 必要に応じてコンテンツを切り詰め
- `persistBinaryContent()` - バイナリコンテンツのディスク保存
- `maybeResizeAndDownsampleImageBuffer()` - 画像のリサイズ・ダウンサンプリング

## ツール同期

### サーバー名正規化 (`normalization.ts`)

API パターン `^[a-zA-Z0-9_-]{1,64}$` に準拠するよう名前を正規化:

```typescript
export function normalizeNameForMCP(name: string): string {
  let normalized = name.replace(/[^a-zA-Z0-9_-]/g, '_')
  // claude.ai サーバーは連続アンダースコアを圧縮
  if (name.startsWith('claude.ai ')) {
    normalized = normalized.replace(/_+/g, '_').replace(/^_|_$/g, '')
  }
  return normalized
}
```

### ツールフィルタリング (`utils.ts`)

サーバー別のフィルタリングユーティリティ:

```typescript
filterToolsByServer(tools, serverName)       // サーバーのツールを抽出
excludeToolsByServer(tools, serverName)      // サーバーのツールを除外
filterCommandsByServer(commands, serverName) // コマンドをフィルタ
filterResourcesByServer(resources, serverName) // リソースをフィルタ
```

### MCP スキル統合

フィーチャーフラグ `MCP_SKILLS` 有効時、MCP サーバーからスキル（コマンド）を取得する:

```typescript
const fetchMcpSkillsForClient = feature('MCP_SKILLS')
  ? require('../../skills/mcpSkills.js').fetchMcpSkillsForClient
  : null
```

## OAuth 認証フロー (`auth.ts`)

### ClaudeAuthProvider

`OAuthClientProvider` インターフェースを実装し、MCP サーバーの OAuth 認証を管理する。

**認証フロー:**

1. **メタデータ検出**: `discoverOAuthServerInfo()` / `discoverAuthorizationServerMetadata()` で認証サーバーの情報を取得
2. **クライアント登録**: Dynamic Client Registration (DCR)
3. **PKCE 認証**: `code_challenge` / `code_verifier` による PKCE フロー
4. **ローカル HTTP サーバー**: コールバック受信用の一時 HTTP サーバーを起動
5. **トークン交換**: 認可コードをアクセストークンに交換
6. **トークンリフレッシュ**: 期限切れトークンの自動リフレッシュ

### セキュリティ対策

- **タイムアウト**: `AUTH_REQUEST_TIMEOUT_MS = 30000`（各 OAuth リクエストに 30 秒タイムアウト）
- **URL パラメータのリダクション**: `redactSensitiveUrlParams()` が `state`, `nonce`, `code_challenge`, `code_verifier`, `code` をログから除外
- **ロック機構**: `MAX_LOCK_RETRIES = 5` でトークンリフレッシュの競合を防止
- **セキュアストレージ**: トークンは `secureStorage`（macOS Keychain 等）に保存

### OAuth エラー正規化

`normalizeOAuthErrorBody()` が非標準の OAuth エラーレスポンスを正規化する（L157-190）:

- Slack 等の「200 OK + エラーボディ」パターンを 400 レスポンスに変換
- 非標準エラーコード（`invalid_refresh_token`, `expired_refresh_token`, `token_expired`）を RFC 6749 の `invalid_grant` に正規化

### Cross-App Access (XAA)

`xaa.ts` / `xaaIdpLogin.ts` が Cross-App Access (SEP-990) を実装:

- IdP (Identity Provider) へのトークン交換
- OIDC ディスカバリー
- 設定は `settings.xaaIdp` で一元管理

### アナリティクスイベント

OAuth 関連のアナリティクスイベント:

| イベント名 | 説明 |
|---|---|
| `tengu_mcp_oauth_refresh_failure` | トークンリフレッシュ失敗（理由コード付き） |
| `tengu_mcp_oauth_flow_error` | OAuth フロー全体のエラー |

失敗理由は安定した文字列として定義されている（`MCPRefreshFailureReason`, `MCPOAuthFlowErrorReason`）。

## 設定管理 (`config.ts`)

### 設定の読み込み

`getClaudeCodeMcpConfigs()` が全ソースから MCP 設定を収集する:

1. **エンタープライズ**: `getEnterpriseMcpFilePath()` → `<managed>/managed-mcp.json`
2. **ユーザー設定**: `getGlobalConfig()` からの mcpServers
3. **プロジェクト設定**: `.claude/settings.json` / `.mcp.json`
4. **プラグイン**: `getPluginMcpServers()` でプラグイン提供サーバーを統合
5. **Claude.ai**: `fetchClaudeAIMcpConfigsIfEligible()` で動的取得

### 設定の書き込み

`.mcp.json` への書き込みはアトミック操作:

```typescript
// 1. 一時ファイルに書き込み
// 2. datasync() でディスクフラッシュ
// 3. rename() でアトミック置換
// 4. 失敗時は一時ファイルをクリーンアップ
```

### サーバーの有効/無効

```typescript
setMcpServerEnabled(serverName, enabled)  // 設定に保存
isMcpServerDisabled(serverName)           // 無効状態の確認
```

## Elicitation ハンドリング (`elicitationHandler.ts`)

MCP サーバーからの対話型入力要求（Elicitation）を処理する。`ElicitRequestSchema` に基づいてユーザーにパラメータ入力を促し、`ElicitResult` を返す。フック（`runElicitationHooks` / `runElicitationResultHooks`）による前後処理も可能。
