# 認証システム (auth.ts)

Claude Code v2.1.88 の認証基盤に関する包括的な分析ドキュメント。

## 関連ソースファイル

| ファイル | サイズ | 役割 |
|---------|--------|------|
| `src/utils/auth.ts` | ~65KB | 認証システムのメインモジュール |
| `src/utils/authPortable.ts` | ~0.5KB | macOSキーチェーン操作・APIキー正規化 |
| `src/utils/authFileDescriptor.ts` | ~5KB | ファイルディスクリプタ経由のトークン取得 |

---

## 1. 認証システムの全体像

Claude Code の認証システムは複数の認証ソースを優先順位に従って処理する多層アーキテクチャを採用している。主要な認証パスは以下の通り。

### 認証ソースの優先順位

`getAuthTokenSource()` 関数（`auth.ts:153`）が認証トークンのソースを決定する。優先順位は以下の通り:

1. **ANTHROPIC_AUTH_TOKEN** 環境変数（マネージドOAuthコンテキスト外のみ）
2. **CLAUDE_CODE_OAUTH_TOKEN** 環境変数
3. **ファイルディスクリプタ経由のOAuthトークン**（CCR環境向け）
4. **apiKeyHelper** 設定（マネージドOAuthコンテキスト外のみ）
5. **Claude.ai OAuth トークン**（ローカル設定から）

APIキーの優先順位は `getAnthropicApiKeyWithSource()` 関数（`auth.ts:226`）が決定する:

1. **ANTHROPIC_API_KEY** 環境変数
2. **ファイルディスクリプタ経由のAPIキー**
3. **apiKeyHelper** コマンド出力
4. **macOSキーチェーン / 設定ファイル**（`/login` で管理されるキー）

### bareモードの特殊処理

`--bare` フラグ使用時は認証が最小限に制限される（`auth.ts:101-103`, `auth.ts:235-248`）:
- OAuthは無効化
- `ANTHROPIC_API_KEY` 環境変数または `--settings` フラグで指定された `apiKeyHelper` のみ使用可能
- キーチェーン、設定ファイル、承認リストは参照されない

### マネージドOAuthコンテキスト

`isManagedOAuthContext()` 関数（`auth.ts:91-96`）は、CCR（Claude Code Remote）やClaude Desktopから起動された場合を検出する:

```
CLAUDE_CODE_REMOTE=true  →  マネージドコンテキスト
CLAUDE_CODE_ENTRYPOINT=claude-desktop  →  マネージドコンテキスト
```

この場合、ユーザーのローカル設定（`apiKeyHelper`、`env.ANTHROPIC_API_KEY` など）は無視され、渡されたOAuthトークンのみが使用される。

---

## 2. Claude.ai OAuth認証フロー

### Anthropic認証の有効/無効判定

`isAnthropicAuthEnabled()` 関数（`auth.ts:100-149`）が1P（ファーストパーティ）認証の有効性を判定する。以下の場合に無効化される:

- `--bare` モード
- **ANTHROPIC_UNIX_SOCKET** が設定されている場合（SSHプロキシ経由）: `CLAUDE_CODE_OAUTH_TOKEN` が設定されている場合のみ有効
- サードパーティサービス使用時（Bedrock / Vertex / Foundry）
- 外部APIキーまたは外部認証トークンが存在し、かつマネージドOAuthコンテキスト外の場合

### OAuthトークンの取得と管理

OAuth トークンは `getClaudeAIOAuthTokens()` で取得され（`auth.ts:200-201`付近で参照）、`shouldUseClaudeAIAuth()` でスコープの検証が行われる。トークン管理の主要な機能:

- **トークン期限切れチェック**: `isOAuthTokenExpired()` を使用
- **トークンリフレッシュ**: `refreshOAuthToken()` を使用
- **プロファイル取得**: `getOauthProfileFromOauthToken()` でユーザー情報を取得

---

## 3. APIキー管理

### apiKeyHelper メカニズム

`apiKeyHelper` は設定ファイルで指定された外部コマンドを実行してAPIキーを取得する仕組み。SWR（Stale-While-Revalidate）キャッシュパターンを採用している（`auth.ts:452-589`）。

**キャッシュアーキテクチャ:**
- `_apiKeyHelperCache`: 最後に取得した値とタイムスタンプ
- `_apiKeyHelperInflight`: 実行中のPromise（重複実行防止）
- `_apiKeyHelperEpoch`: エポックカウンタ（キャッシュ無効化時にインクリメント）

**TTL管理** (`auth.ts:435-450`):
- デフォルトTTL: 5分（`DEFAULT_API_KEY_HELPER_TTL = 5 * 60 * 1000`）
- `CLAUDE_CODE_API_KEY_HELPER_TTL_MS` 環境変数でカスタマイズ可能

**セキュリティ検証** (`auth.ts:546-556`):
- プロジェクト設定由来の `apiKeyHelper` は、ワークスペーストラストが確認済みの場合のみ実行
- `execa` でシェル経由実行、タイムアウト10分

**SWR動作** (`auth.ts:469-499`):
1. キャッシュヒット（TTL内）: キャッシュ値を即座に返す
2. キャッシュヒット（TTL超過）: 古い値を返しつつバックグラウンドで更新
3. キャッシュミス: 実行完了まで待機（重複呼び出しはデデュプリケーション）

### ファイルディスクリプタ経由のキー取得

`authFileDescriptor.ts` は CCR 環境でのセキュアなトークン受け渡しを実装する。

**`getCredentialFromFd()` の優先順位** (`authFileDescriptor.ts:97-166`):
1. ファイルディスクリプタ（パイプ）: Go環境マネージャーが `cmd.ExtraFiles` で渡す
2. well-knownファイル: FD読み取り成功時にディスクに永続化されたトークン

**well-knownパス:**
- OAuth トークン: `/home/claude/.claude/remote/.oauth_token`
- API キー: `/home/claude/.claude/remote/.api_key`
- セッションイングレストークン: `/home/claude/.claude/remote/.session_ingress_token`

**セキュリティ**: ファイル権限は `0o700`（ディレクトリ）/ `0o600`（ファイル）で作成される。

---

## 4. トークン管理（取得、更新、失効）

### AWS認証リフレッシュ

`refreshAndGetAwsCredentials()` 関数（`auth.ts:787-807`）は `memoizeWithTTLAsync` でラップされ、TTL 1時間（`DEFAULT_AWS_STS_TTL`）でキャッシュされる:

1. `runAwsAuthRefresh()`: STS caller identity を確認し、失敗時に `awsAuthRefresh` コマンドを実行（タイムアウト3分）
2. `getAwsCredsFromCredentialExport()`: `awsCredentialExport` コマンドでJSON形式の認証情報を取得
3. `clearAwsIniCache()`: AWS INI キャッシュをクリア

### GCP認証リフレッシュ

`refreshGcpCredentialsIfNeeded()` 関数（`auth.ts:974-981`）も同様にTTL 1時間でキャッシュ:

- `checkGcpCredentialsValid()`: `google-auth-library` を動的インポートしてアクセストークンの有効性を検証（タイムアウト5秒）
- `refreshGcpAuth()`: 無効な場合に `gcpAuthRefresh` コマンドを実行

### セキュリティチェック共通パターン

AWS/GCP の認証リフレッシュコマンドはすべて以下のセキュリティチェックを実施:
1. プロジェクト設定由来かチェック（`isXxxFromProjectSettings()`）
2. ワークスペーストラストが確認済みか検証（`checkHasTrustDialogAccepted()`）
3. 非インタラクティブセッションの場合は例外を設ける

---

## 5. 認証状態の永続化

### macOSキーチェーン連携

`getApiKeyFromConfigOrMacOSKeychain()` 関数（`auth.ts:1050-1087`）:

1. **プリフェッチ結果の利用**: `keychainPrefetch.ts` が起動時に非同期でキーチェーンを読み取り、完了していればそれを使用（同期的な `security` サブプロセス生成を回避、約33ms節約）
2. **直接キーチェーンアクセス**: プリフェッチ未完了の場合、`security find-generic-password` コマンドで取得
3. **設定ファイルフォールバック**: `getGlobalConfig().primaryApiKey` から取得

### APIキーの保存

`saveApiKey()` 関数（`auth.ts:1094`付近）:
- APIキーのバリデーション: 英数字、ダッシュ、アンダースコアのみ許可
- `isValidApiKey()` によるフォーマット検証

### authPortable.ts

`authPortable.ts` は以下の2つのユーティリティを提供:

- `maybeRemoveApiKeyFromMacOSKeychainThrows()`: macOSの場合、`security delete-generic-password` でキーチェーンからAPIキーを削除
- `normalizeApiKeyForConfig()`: APIキーの末尾20文字を設定ファイル用に正規化（承認リストでの比較に使用）

---

## 6. セキュアストレージ

### SecureStorage アーキテクチャ

`getSecureStorage()` 関数（`auth.ts:62` でインポート）はプラットフォームに応じたセキュアストレージ実装を提供する。関連モジュール:

- `src/utils/secureStorage/index.ts`: ストレージプロバイダの選択
- `src/utils/secureStorage/macOsKeychainHelpers.ts`: macOS Keychain ヘルパー
- `src/utils/secureStorage/keychainPrefetch.ts`: 起動時のキーチェーンプリフェッチ

### CCR環境でのトークン永続化

`maybePersistTokenForSubprocesses()` 関数（`authFileDescriptor.ts:30-49`）:
- CCR環境（`CLAUDE_CODE_REMOTE=true`）の場合のみ動作
- FDから読み取ったトークンをwell-knownパスにファイルとして書き出し
- サブプロセス（tmux、別シェル）がパイプFDを継承できない問題を解決

---

## アーキテクチャ図

```
認証リクエスト
    │
    ├─ bareモード? ──→ ANTHROPIC_API_KEY / apiKeyHelper のみ
    │
    ├─ マネージドOAuth? ──→ 環境変数のOAuthトークンのみ
    │
    └─ 通常モード
        │
        ├─ getAuthTokenSource()
        │   ├─ ANTHROPIC_AUTH_TOKEN
        │   ├─ CLAUDE_CODE_OAUTH_TOKEN
        │   ├─ FD経由OAuthトークン
        │   ├─ apiKeyHelper
        │   └─ Claude.ai OAuthトークン
        │
        └─ getAnthropicApiKeyWithSource()
            ├─ ANTHROPIC_API_KEY 環境変数
            ├─ FD経由APIキー
            ├─ apiKeyHelper (SWRキャッシュ)
            └─ macOSキーチェーン / 設定ファイル
```
