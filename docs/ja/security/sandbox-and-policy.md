# サンドボックスとポリシー

Claude Code v2.1.88 は、OS レベルのサンドボックス隔離と組織ポリシーによるリモート制限の二層でセキュリティを実現する。

## 概要

セキュリティレイヤーの階層構造:

```
┌─────────────────────────────────────────┐
│         組織ポリシー (Policy Limits)      │  リモート API で管理
├─────────────────────────────────────────┤
│         マネージド設定 (policySettings)    │  管理者がデプロイ
├─────────────────────────────────────────┤
│         権限ルール (Permission Rules)     │  ユーザー/プロジェクト設定
├─────────────────────────────────────────┤
│         サンドボックス (OS レベル隔離)      │  bubblewrap / sandbox-exec
└─────────────────────────────────────────┘
```

---

## サンドボックスメカニズムの実装

### アーキテクチャ

**ソース**: `src/utils/sandbox/sandbox-adapter.ts`

サンドボックスは `@anthropic-ai/sandbox-runtime` パッケージを基盤とし、Claude Code 固有のアダプター層でラップされている:

```typescript
import {
  SandboxManager as BaseSandboxManager,
  SandboxRuntimeConfigSchema,
  SandboxViolationStore,
} from '@anthropic-ai/sandbox-runtime'
```

### プラットフォームサポート

| プラットフォーム | サンドボックス技術 | 状態 |
|---|---|---|
| macOS | `sandbox-exec` (Seatbelt) | サポート済み |
| Linux | `bubblewrap` (bwrap) | サポート済み |
| WSL2 | `bubblewrap` | サポート済み |
| WSL1 | - | 非サポート |

**ソース**: `src/utils/sandbox/sandbox-adapter.ts:491-493`

```typescript
const isSupportedPlatform = memoize((): boolean => {
  return BaseSandboxManager.isSupportedPlatform()
})
```

### 有効化条件

サンドボックスが有効になるための条件チェーン:

```typescript
// src/utils/sandbox/sandbox-adapter.ts:532-547
function isSandboxingEnabled(): boolean {
  if (!isSupportedPlatform()) return false
  if (checkDependencies().errors.length > 0) return false
  if (!isPlatformInEnabledList()) return false
  return getSandboxEnabledSetting()
}
```

1. **プラットフォームサポート**: macOS, Linux, WSL2
2. **依存関係チェック**: bubblewrap, socat 等がインストール済み
3. **enabledPlatforms 制限**: 任意のプラットフォームに限定可能 (undocumented)
4. **設定で明示的に有効化**: `sandbox.enabled: true`

### サンドボックス設定変換

**ソース**: `src/utils/sandbox/sandbox-adapter.ts:172-381`

Claude Code の設定形式を `SandboxRuntimeConfig` に変換する `convertToSandboxRuntimeConfig()` 関数が中核:

```typescript
function convertToSandboxRuntimeConfig(settings: SettingsJson): SandboxRuntimeConfig {
  // 以下を統合:
  // 1. 権限ルールからのネットワーク/ファイルシステム制限
  // 2. sandbox.* 設定
  // 3. セキュリティハードコードルール
  return {
    network: { allowedDomains, deniedDomains, ... },
    filesystem: { denyRead, allowRead, allowWrite, denyWrite },
    ignoreViolations: ...,
    enableWeakerNestedSandbox: ...,
    enableWeakerNetworkIsolation: ...,
    ripgrep: ...,
  }
}
```

---

## ファイルアクセス制御

### 書き込み許可パス

デフォルトで書き込みが許可されるパス:

```typescript
// src/utils/sandbox/sandbox-adapter.ts:225
const allowWrite: string[] = ['.', getClaudeTempDir()]
```

追加の書き込み許可:
- `--add-dir` や `/add-dir` で追加されたディレクトリ
- `permissions.allow` の `Edit(path)` ルールに一致するパス
- `sandbox.filesystem.allowWrite` に指定されたパス
- Git worktree のメインリポジトリパス

### 書き込み拒否パス (セキュリティハードコード)

**ソース**: `src/utils/sandbox/sandbox-adapter.ts:232-255`

設定ファイルへの書き込みを常に拒否し、サンドボックスエスケープを防止:

```typescript
// 全設定ソースの settings.json ファイル
const settingsPaths = SETTING_SOURCES.map(source =>
  getSettingsFilePathForSource(source),
).filter(Boolean)
denyWrite.push(...settingsPaths)
denyWrite.push(getManagedSettingsDropInDir())

// .claude/skills への書き込みも拒否
denyWrite.push(resolve(originalCwd, '.claude', 'skills'))
```

### ベアリポジトリ攻撃の防御

**ソース**: `src/utils/sandbox/sandbox-adapter.ts:259-280`

Git の `is_git_directory()` がカレントディレクトリをベアリポジトリとして誤認させる攻撃 (HEAD + objects/ + refs/ の設置) を防御:

```typescript
const bareGitRepoFiles = ['HEAD', 'objects', 'refs', 'hooks', 'config']
for (const dir of [originalCwd, cwd]) {
  for (const gitFile of bareGitRepoFiles) {
    const p = resolve(dir, gitFile)
    try {
      statSync(p)       // 存在する場合: 読み取り専用マウント
      denyWrite.push(p)
    } catch {
      bareGitRepoScrubPaths.push(p)  // 存在しない場合: コマンド後にスクラブ
    }
  }
}
```

コマンド実行後に `scrubBareGitRepoFiles()` で不正に作成されたファイルを削除。

### パスパターンの解決

**ソース**: `src/utils/sandbox/sandbox-adapter.ts:99-119`

Claude Code 固有のパス記法をサンドボックスランタイム用に変換:

| パターン | 意味 | 変換結果 |
|---|---|---|
| `//path` | ファイルシステムルートからの絶対パス | `/path` |
| `/path` | 設定ファイルディレクトリからの相対パス | `$SETTINGS_DIR/path` |
| `~/path` | ホームディレクトリ (パススルー) | そのまま |
| `./path` | カレントディレクトリ相対 (パススルー) | そのまま |

`sandbox.filesystem.*` 設定では別のセマンティクス (`/path` = 絶対パス) を使用:

```typescript
// src/utils/sandbox/sandbox-adapter.ts:138-146
function resolveSandboxFilesystemPath(pattern: string, source: SettingSource): string {
  if (pattern.startsWith('//')) return pattern.slice(1)
  return expandPath(pattern, getSettingsRootPathForSource(source))
}
```

---

## ネットワーク制御

### ドメインベースのフィルタリング

```typescript
// SandboxRuntimeConfig.network の構造
{
  allowedDomains: string[]           // 許可ドメインリスト
  deniedDomains: string[]            // 拒否ドメインリスト
  allowUnixSockets?: boolean         // Unix ソケット許可
  allowAllUnixSockets?: boolean      // 全 Unix ソケット許可
  allowLocalBinding?: boolean        // ローカルバインド許可
  httpProxyPort?: number             // HTTP プロキシポート
  socksProxyPort?: number            // SOCKS プロキシポート
}
```

### マネージドドメインのみ制限

**ソース**: `src/utils/sandbox/sandbox-adapter.ts:152-157`

```typescript
function shouldAllowManagedSandboxDomainsOnly(): boolean {
  return getSettingsForSource('policySettings')
    ?.sandbox?.network?.allowManagedDomainsOnly === true
}
```

有効時はポリシー設定のドメインのみがサンドボックスに許可される。

---

## コマンド除外とサンドボックスオーバーライド

### 除外コマンド

**ソース**: `src/tools/BashTool/shouldUseSandbox.ts`

ユーザーの便宜のため、特定コマンドをサンドボックスから除外可能:

```typescript
// 注意: セキュリティ境界ではない
// "NOTE: excludedCommands is a user-facing convenience feature, not a security boundary."
function containsExcludedCommand(command: string): boolean {
  // 1. 動的設定 (内部のみ): tengu_sandbox_disabled_commands
  // 2. ユーザー設定: sandbox.excludedCommands
}
```

### dangerouslyDisableSandbox

ツール入力に `dangerouslyDisableSandbox: true` を指定するとサンドボックスをバイパスできるが、これは権限プロンプトの対象となる:

```typescript
// 判定理由タイプ: sandboxOverride
type PermissionDecisionReason = {
  type: 'sandboxOverride'
  reason: 'excludedCommand' | 'dangerouslyDisableSandbox'
}
```

---

## サンドボックスの初期化と状態管理

### 初期化フロー

```typescript
// src/utils/sandbox/sandbox-adapter.ts:387-394
let initializationPromise: Promise<void> | undefined
let settingsSubscriptionCleanup: (() => void) | undefined
let worktreeMainRepoPath: string | null | undefined
```

初期化時に:
1. Git worktree の検出 (`detectWorktreeMainRepoPath`)
2. 依存関係チェック (`checkDependencies`)
3. 設定変更の監視 (settings change detector)
4. `convertToSandboxRuntimeConfig` による設定生成

### 利用不能時の警告

**ソース**: `src/utils/sandbox/sandbox-adapter.ts:562-592`

ユーザーが `sandbox.enabled: true` を設定したにもかかわらずサンドボックスが利用できない場合、起動時に明確な理由を表示:

```typescript
function getSandboxUnavailableReason(): string | undefined {
  // プラットフォーム非サポート
  // enabledPlatforms に含まれない
  // 依存関係の不足
}
```

### 自動許可設定

```typescript
// src/utils/sandbox/sandbox-adapter.ts:469-477
function isAutoAllowBashIfSandboxedEnabled(): boolean {
  return settings?.sandbox?.autoAllowBashIfSandboxed ?? true  // デフォルト true
}

function areUnsandboxedCommandsAllowed(): boolean {
  return settings?.sandbox?.allowUnsandboxedCommands ?? true
}
```

---

## ポリシーリミット (組織ポリシー)

### 概要

**ソース**: `src/services/policyLimits/index.ts`

組織レベルのポリシー制限を API から取得し、CLI 機能を制限するサービス。

### 対象ユーザー

```typescript
// src/services/policyLimits/index.ts:167-211
function isPolicyLimitsEligible(): boolean {
  // 3P プロバイダーユーザー: 対象外
  // カスタム base URL: 対象外
  // Console ユーザー (API キー): 対象
  // OAuth ユーザー: Team/Enterprise のみ対象
}
```

### ポリシーレスポンス形式

**ソース**: `src/services/policyLimits/types.ts`

```typescript
const PolicyLimitsResponseSchema = z.object({
  restrictions: z.record(z.string(), z.object({ allowed: z.boolean() })),
})
```

ブロックされたポリシーのみがレスポンスに含まれる。キーが存在しない = 許可。

### フェイルオープン設計

```typescript
// src/services/policyLimits/index.ts:510-526
function isPolicyAllowed(policy: string): boolean {
  const restrictions = getRestrictionsFromCache()
  if (!restrictions) {
    // 例外: essential-traffic-only モードでは特定ポリシーをフェイルクローズ
    if (isEssentialTrafficOnly() && ESSENTIAL_TRAFFIC_DENY_ON_MISS.has(policy)) {
      return false
    }
    return true  // フェイルオープン
  }
  const restriction = restrictions[policy]
  if (!restriction) return true  // 未知のポリシー = 許可
  return restriction.allowed
}
```

HIPAA 対応組織向けに、`allow_product_feedback` ポリシーは essential-traffic-only モードでキャッシュミス時にフェイルクローズする。

### キャッシュ戦略

| レイヤー | 説明 |
|---|---|
| セッションキャッシュ | メモリ内の `sessionCache` 変数 |
| ファイルキャッシュ | `~/.claude/policy-limits.json` |
| ETag キャッシュ | SHA-256 チェックサムによる 304 Not Modified 対応 |

### バックグラウンドポーリング

```typescript
// src/services/policyLimits/index.ts:57-58
const POLLING_INTERVAL_MS = 60 * 60 * 1000  // 1 時間
const FETCH_TIMEOUT_MS = 10000               // 10 秒
const DEFAULT_MAX_RETRIES = 5                // 最大 5 回リトライ
```

ポーリングタイマーは `unref()` されており、プロセスの終了を妨げない。

### 初期化プロミス

**ソース**: `src/services/policyLimits/index.ts:94-114`

他のシステムがポリシーロードの完了を待機できるよう、ローディングプロミスを提供:

```typescript
function initializePolicyLimitsLoadingPromise(): void {
  // タイムアウト付き (30秒) でデッドロック防止
  setTimeout(() => {
    if (loadingCompleteResolve) {
      loadingCompleteResolve()
    }
  }, LOADING_PROMISE_TIMEOUT_MS)
}
```

### 認証とリトライ

**ソース**: `src/services/policyLimits/index.ts:227-262`

API キーと OAuth の両方に対応:

```typescript
function getAuthHeaders(): { headers: Record<string, string>; error?: string } {
  // 1. API キーを試行
  // 2. OAuth トークンにフォールバック
}
```

エクスポネンシャルバックオフによるリトライ:

```typescript
async function fetchWithRetry(cachedChecksum?: string): Promise<PolicyLimitsFetchResult> {
  for (let attempt = 1; attempt <= DEFAULT_MAX_RETRIES + 1; attempt++) {
    const result = await fetchPolicyLimits(cachedChecksum)
    if (result.success || result.skipRetry) return result
    await sleep(getRetryDelay(attempt))
  }
}
```

---

## セキュリティレイヤーの相互作用

### 権限チェックからサンドボックスへの流れ

```
ツール実行リクエスト
  │
  ├── 1. 権限ルールチェック (allow/deny/ask)
  │     └── policySettings > projectSettings > userSettings
  │
  ├── 2. ポリシーリミットチェック
  │     └── isPolicyAllowed() でリモートポリシー確認
  │
  ├── 3. サンドボックス判定
  │     ├── shouldUseSandbox() で対象コマンドか確認
  │     ├── excludedCommands チェック
  │     └── dangerouslyDisableSandbox チェック
  │
  └── 4. サンドボックス実行
        ├── ファイルシステム隔離 (allowWrite/denyWrite/denyRead)
        ├── ネットワーク隔離 (allowedDomains/deniedDomains)
        └── 違反検出・記録
```

### 設定の優先順位

```
policySettings (組織ポリシー)    [最高優先度]
  ↓
flagSettings (GrowthBook)
  ↓
projectSettings (.claude/settings.json)
  ↓
localSettings (.claude/settings.local.json)
  ↓
userSettings (~/.claude/settings.json)   [最低優先度]
```

`allowManagedPermissionRulesOnly` が有効な場合、policySettings 以外の権限ルールは無視される。
`allowManagedSandboxDomainsOnly` が有効な場合、policySettings 以外のネットワークドメインは無視される。
`allowManagedReadPathsOnly` が有効な場合、policySettings 以外の読み取りパスは無視される。

---

## サンドボックス違反の処理

### 違反イベント

```typescript
// @anthropic-ai/sandbox-runtime から
type SandboxViolationEvent = {
  // 違反の詳細 (ファイルアクセス、ネットワークアクセス等)
}
```

### UI 表示

**ソース**: `src/utils/sandbox/sandbox-ui-utils.ts`

```typescript
function removeSandboxViolationTags(text: string): string {
  return text.replace(/<sandbox_violations>[\s\S]*?<\/sandbox_violations>/g, '')
}
```

サンドボックス違反情報はエラーメッセージ内の `<sandbox_violations>` タグで伝達され、UI 表示時に除去される。
