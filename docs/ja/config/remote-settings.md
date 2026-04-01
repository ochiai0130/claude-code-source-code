# リモートマネージド設定

Claude Code v2.1.88 ソースコード分析

## 概要

リモートマネージド設定は、エンタープライズ顧客向けにサーバーサイドから設定を配布・管理するシステムである。APIを通じて設定を取得し、ローカルキャッシュとセッションキャッシュの2層キャッシュで高効率な運用を実現する。チェックサムベースのHTTPキャッシングにより、ネットワークトラフィックを最小化している。

**主要ファイル:** `src/services/remoteManagedSettings/index.ts`

---

## 1. リモート設定のポーリング

### 1.1 ポーリング間隔

`index.ts` L54:

```typescript
const POLLING_INTERVAL_MS = 60 * 60 * 1000  // 1時間
```

バックグラウンドポーリングにより、セッション中の設定変更をキャッチする。

### 1.2 初期化フロー

```
init.ts
  → initializeRemoteManagedSettingsLoadingPromise()  // Promise作成
main.tsx
  → loadRemoteManagedSettings()                       // 実際の取得と適用
    → fetchAndLoadRemoteManagedSettings()
      → fetchWithRetry(cachedChecksum)
        → fetchRemoteManagedSettings()                // 単一APIコール
    → startBackgroundPolling()                        // 1時間ごとのポーリング開始
```

### 1.3 キャッシュファースト起動

`index.ts` L526-532:

起動時、ディスクキャッシュ（`~/.claude/remote-settings.json`）が存在する場合は即座に適用し、待機中のシステムをアンブロックする。バックグラウンドでAPI取得が並行して進み、変更があれば後から通知される。これにより、print-mode起動時の約77msのフェッチ待ちを削減。

### 1.4 ホットリロード

設定が変更された場合、`settingsChangeDetector.notifyChange('policySettings')` が呼ばれ、設定キャッシュがリセットされる。環境変数、テレメトリ、権限は次回の読み取り時に更新される。

---

## 2. 適格性チェック

**ファイル:** `src/services/remoteManagedSettings/syncCache.ts`

### 2.1 適格ユーザー

`isRemoteManagedSettingsEligible()`（syncCache.ts L49-112）:

| ユーザー種別 | 条件 | 適格性 |
|-------------|------|--------|
| コンソールユーザー（APIキー） | 実際のキーが存在 | 適格 |
| OAuth（Enterprise/C4E） | `subscriptionType === 'enterprise'` | 適格 |
| OAuth（Team） | `subscriptionType === 'team'` | 適格 |
| OAuth（外部注入トークン） | `subscriptionType === null` | 適格（APIが判定） |
| サードパーティプロバイダー | `getAPIProvider() !== 'firstParty'` | 不適格 |
| カスタムベースURL | `!isFirstPartyAnthropicBaseUrl()` | 不適格 |
| Cowork（ローカルエージェント） | `CLAUDE_CODE_ENTRYPOINT === 'local-agent'` | 不適格 |

### 2.2 循環依存の回避

適格性チェックは `getSettings()` を呼び出してはならない。`syncCache.ts` と `syncCacheState.ts` の分離により、`settings.ts → syncCache.ts → auth.ts → settings.ts` の循環を防止している（`syncCacheState.ts` L1-22のコメント参照）。

---

## 3. API通信

### 3.1 エンドポイント

```typescript
`${getOauthConfig().BASE_API_URL}/api/claude_code/settings`
```

### 3.2 認証

`getRemoteSettingsAuthHeaders()`（index.ts L166-203）:

1. **APIキー優先:** `x-api-key` ヘッダー（`apiKeyHelper` はスキップ、循環依存回避）
2. **OAuthフォールバック:** `Authorization: Bearer <token>` + `anthropic-beta` ヘッダー

### 3.3 チェックサムベースキャッシング

`computeChecksumFromSettings()`（index.ts L131-137）:

```typescript
// Pythonサーバーと同一の正規化: json.dumps(sort_keys=True, separators=(",", ":"))
const sorted = sortKeysDeep(settings)
const normalized = jsonStringify(sorted)
const hash = createHash('sha256').update(normalized).digest('hex')
return `sha256:${hash}`
```

HTTP `If-None-Match` / `304 Not Modified` によるETagキャッシングを実装。サーバーサイドのPython実装と互換性のあるチェックサム計算を行う。

### 3.4 レスポンス処理

| ステータス | 意味 | 処理 |
|-----------|------|------|
| `200` | 新しい設定 | スキーマバリデーション後に適用 |
| `204` | 設定なし | 空オブジェクト `{}` を返す |
| `304` | 変更なし | キャッシュ有効、`settings: null` を返す |
| `404` | 未設定/機能フラグオフ | 空オブジェクト `{}` を返す |
| `401/403` | 認証エラー | リトライなし（`skipRetry: true`） |

### 3.5 リトライロジック

`fetchWithRetry()`（index.ts L209-242）:

- 最大5回リトライ（`DEFAULT_MAX_RETRIES = 5`）
- 指数バックオフ（`getRetryDelay(attempt)`）
- タイムアウト: 10秒（`SETTINGS_TIMEOUT_MS = 10000`）
- 認証エラーはリトライしない

---

## 4. レスポンスのスキーマ

**ファイル:** `src/services/remoteManagedSettings/types.ts`

```typescript
export const RemoteManagedSettingsResponseSchema = lazySchema(() =>
  z.object({
    uuid: z.string(),                                    // 設定UUID
    checksum: z.string(),                                // チェックサム
    settings: z.record(z.string(), z.unknown()),         // 設定本体
  })
)
```

レスポンス解析後、`SettingsSchema.safeParse()` で完全なバリデーションが行われる（index.ts L322-332）。

---

## 5. 管理対象設定の種類

リモート設定は `SettingsSchema`（`src/utils/settings/types.ts`）の全フィールドを配布可能。エンタープライズ管理者が特に使用する設定:

### 5.1 権限制御

```json
{
  "permissions": {
    "allow": [{ "tool": "Read", "pattern": "**" }],
    "deny": [{ "tool": "Bash", "pattern": "rm -rf *" }],
    "defaultMode": "normal",
    "disableBypassPermissionsMode": "disable"
  }
}
```

### 5.2 フック制御

```json
{
  "hooks": { ... },
  "allowManagedHooksOnly": true,
  "disableAllHooks": false,
  "allowedHttpHookUrls": ["https://hooks.corp.example.com/*"]
}
```

### 5.3 モデル制限

```json
{
  "model": "claude-opus-4-6",
  "availableModels": ["opus", "sonnet"],
  "modelOverrides": {
    "claude-opus-4-6": "arn:aws:bedrock:..."
  }
}
```

### 5.4 MCP制御

```json
{
  "allowedMcpServers": [
    { "serverName": "approved-server" },
    { "serverUrl": "https://*.corp.example.com/*" }
  ],
  "deniedMcpServers": [
    { "serverName": "blocked-server" }
  ],
  "allowManagedMcpServersOnly": true
}
```

### 5.5 プラグイン制御

```json
{
  "strictPluginOnlyCustomization": ["skills", "agents", "hooks", "mcp"],
  "strictKnownMarketplaces": [{ "source": "settings", "name": "corp-marketplace" }],
  "blockedMarketplaces": [{ "source": "settings", "name": "untrusted-marketplace" }]
}
```

---

## 6. キルスイッチ機能

### 6.1 全フック無効化

マネージド設定で `disableAllHooks: true` を配布すると、マネージドフックを含む全フックが無効化される（`hooksConfigSnapshot.ts` L83-88）。

**注意:** 非マネージド設定で `disableAllHooks: true` を設定した場合、マネージドフックは引き続き実行される。これは設計上の意図的な動作。

### 6.2 バイパスモードの無効化

```json
{
  "permissions": {
    "disableBypassPermissionsMode": "disable"
  }
}
```

### 6.3 自動モードの無効化

内部機能フラグ `TRANSCRIPT_CLASSIFIER` が有効な場合:

```json
{
  "permissions": {
    "disableAutoMode": "disable"
  }
}
```

---

## 7. セキュリティチェック

**ファイル:** `src/services/remoteManagedSettings/securityCheck.tsx`

### 7.1 危険な設定変更の検出

`checkManagedSettingsSecurity()`（securityCheck.tsx L22-61）:

1. 新しい設定に「危険な設定」が含まれるか確認
2. 危険な設定が前回から変更されたか確認
3. 変更がある場合、対話モードでブロッキングダイアログを表示
4. ユーザーの承認/拒否を待機

### 7.2 処理フロー

```
新しい設定を取得
  → hasDangerousSettings(extractDangerousSettings(newSettings))
    → 危険な設定なし → 'no_check_needed'
    → 危険な設定あり
      → hasDangerousSettingsChanged(cached, new)
        → 変更なし → 'no_check_needed'
        → 変更あり
          → 非対話モード → 'no_check_needed'
          → 対話モード → ダイアログ表示
            → 承認 → 'approved' → 設定適用
            → 拒否 → 'rejected' → gracefulShutdownSync(1)
```

### 7.3 セキュリティダイアログ

`ManagedSettingsSecurityDialog` コンポーネント（Ink/React）で承認UIを表示。分析イベント:

- `tengu_managed_settings_security_dialog_shown` - ダイアログ表示
- `tengu_managed_settings_security_dialog_accepted` - 承認
- `tengu_managed_settings_security_dialog_rejected` - 拒否

---

## 8. 組織レベル設定の適用

### 8.1 設定の優先順位における位置

リモートマネージド設定は `policySettings` として扱われ、全ソース中**最高優先度**を持つ:

```
userSettings < projectSettings < localSettings < flagSettings < policySettings
```

### 8.2 policySettings の特権

- `--setting-sources` フラグで無効化不可
- `allowManagedHooksOnly` でマネージドフック以外を無効化可能
- `allowManagedPermissionRulesOnly` でマネージド権限ルール以外を無効化可能
- `allowManagedMcpServersOnly` でマネージドMCP許可リスト以外を無効化可能
- `strictPluginOnlyCustomization` でプラグイン以外のカスタマイズをロック可能

### 8.3 Fail-Open設計

`index.ts` L414-503の `fetchAndLoadRemoteManagedSettings()`:

- API取得失敗時、キャッシュが存在すれば古いキャッシュを使用（graceful degradation）
- キャッシュも存在しない場合、リモート設定なしで続行（fail-open）
- ブロッキングなし — セッションの開始を妨げない

### 8.4 認証状態変更時の更新

`refreshRemoteManagedSettings()`（index.ts L562-579）:

ログイン/ログアウト時に呼ばれ、キャッシュをクリアして再取得する。`settingsChangeDetector.notifyChange('policySettings')` によりホットリロードが発火する。

---

## 9. キャッシュアーキテクチャ

### 9.1 2層キャッシュ

**ファイル:** `src/services/remoteManagedSettings/syncCacheState.ts`

| 層 | 保存先 | 用途 |
|----|--------|------|
| セッションキャッシュ | メモリ（`sessionCache` 変数） | 高速アクセス |
| ファイルキャッシュ | `~/.claude/remote-settings.json` | プロセス間共有・永続化 |

### 9.2 キャッシュ読み込み

`getRemoteManagedSettingsSyncFromCache()`（syncCacheState.ts L70-96）:

```
1. eligible !== true → null を返す（不適格）
2. sessionCache が存在 → sessionCache を返す
3. ファイルから読み込み → sessionCache に保存 → 設定キャッシュをリセット → 返す
```

設定キャッシュのリセット（L92: `resetSettingsCache()`）は、policySettings層が初めて可視になった際に1回だけ実行され、以前のマージ結果（policySettings不在）を無効化する。

### 9.3 ファイル保存

`saveSettings()`（index.ts L367-386）:

```typescript
const handle = await open(path, 'w', 0o600)  // 権限: owner読み書きのみ
await handle.writeFile(jsonStringify(settings, null, 2))
await handle.datasync()  // データの永続化を保証
```

### 9.4 ローディングプロミスのタイムアウト

```typescript
const LOADING_PROMISE_TIMEOUT_MS = 30000  // 30秒（index.ts L66）
```

`loadRemoteManagedSettings()` が呼ばれない場合（Agent SDKテスト等）のデッドロック防止。

---

## 関連ファイル

| ファイル | 説明 |
|----------|------|
| `src/services/remoteManagedSettings/index.ts` | メインサービス（取得、ポーリング、キャッシュ管理） |
| `src/services/remoteManagedSettings/types.ts` | レスポンススキーマ、結果型 |
| `src/services/remoteManagedSettings/syncCache.ts` | 適格性チェック（auth.ts依存） |
| `src/services/remoteManagedSettings/syncCacheState.ts` | キャッシュ状態管理（リーフモジュール） |
| `src/services/remoteManagedSettings/securityCheck.tsx` | 危険な設定変更のセキュリティダイアログ |
| `src/utils/settings/settings.ts` | 設定マージでpolicySettingsソースを参照 |
| `src/utils/settings/changeDetector.ts` | 設定変更の検出・通知 |
| `src/utils/settings/constants.ts` | SETTING_SOURCES定義（policySettingsの位置） |
