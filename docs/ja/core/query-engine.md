# クエリエンジン (QueryEngine)

> ソースファイル: `src/QueryEngine.ts` (約1,295行)

## 概要

`QueryEngine`クラスは、Claude Codeにおける会話のライフサイクルとセッション状態を管理する中核コンポーネントである。1つの会話につき1つの`QueryEngine`インスタンスが生成され、`submitMessage()`の呼び出しごとに同一会話内の新しい「ターン」が開始される。メッセージ履歴、ファイルキャッシュ、使用量統計などの状態はターン間で永続化される。

```
QueryEngine (1会話 = 1インスタンス)
  ├── submitMessage() ─ ターンの開始
  │     ├── システムプロンプト構築
  │     ├── ユーザー入力処理 (processUserInput)
  │     ├── query() 呼び出し (推論ループ)
  │     └── 結果の yield
  ├── interrupt() ─ 中断
  ├── getMessages() ─ メッセージ取得
  └── setModel() ─ モデル変更
```

## クラス構造

### QueryEngineConfig (L130-173)

`QueryEngine`の初期設定を定義する型。主要なフィールドは以下の通り。

| フィールド | 説明 |
|---|---|
| `cwd` | 作業ディレクトリ |
| `tools` | 利用可能なツール群 |
| `commands` | スラッシュコマンド群 |
| `mcpClients` | MCPサーバー接続 |
| `canUseTool` | ツール使用許可判定関数 |
| `maxTurns` | 最大ターン数 |
| `maxBudgetUsd` | USD予算上限 |
| `taskBudget` | タスクトークンバジェット |
| `thinkingConfig` | 思考モード設定 |
| `snipReplay` | スニップ境界ハンドラ (HISTORY_SNIP機能) |

### プライベート状態 (L184-197)

```typescript
private mutableMessages: Message[]        // 会話メッセージの可変配列
private abortController: AbortController  // 中断制御
private permissionDenials: SDKPermissionDenial[]  // 権限拒否の追跡
private totalUsage: NonNullableUsage      // 累計トークン使用量
private readFileState: FileStateCache     // ファイル読み取り状態キャッシュ
private discoveredSkillNames: Set<string> // ターンごとのスキル発見追跡
```

## submitMessage() メソッド (L209-1156)

これが`QueryEngine`の中核メソッドであり、ユーザーの入力を受け取り、推論・ツール実行・結果返却の全フローを制御する。`AsyncGenerator`として実装され、`SDKMessage`を逐次`yield`する。

### 処理フロー

#### 1. 初期化フェーズ (L213-326)

- 設定値の展開（`cwd`, `tools`, `maxTurns`など）
- `discoveredSkillNames`のクリア（ターンごとにリセット）
- 権限拒否を追跡する`wrappedCanUseTool`のラッピング (L244-271)
- メインループモデルの決定 (`userSpecifiedModel`があればパース、なければデフォルト) (L273-276)
- 思考設定の決定 (`adaptive`がデフォルト) (L278-283)
- **システムプロンプトの構築** (L288-325):
  - `fetchSystemPromptParts()`でデフォルトプロンプト、ユーザーコンテキスト、システムコンテキストを取得
  - コーディネーターモードのユーザーコンテキスト追加
  - メモリメカニクスプロンプトの条件付き注入
  - `asSystemPrompt()`で最終的なシステムプロンプト配列を構築

#### 2. ユーザー入力処理フェーズ (L335-487)

- `ProcessUserInputContext`の構築 (L335-395)
- 孤立した権限（`orphanedPermission`）の処理 (L398-408)
- `processUserInput()`によるスラッシュコマンド解析・メッセージ正規化 (L410-428)
- メッセージの永続化（トランスクリプト記録） (L450-463)
- ツール権限コンテキストの更新 (L477-486)

#### 3. スキルとプラグインのロード (L529-551)

```typescript
const [skills, { enabled: enabledPlugins }] = await Promise.all([
  getSlashCommandToolSkills(getCwd()),
  loadAllPluginsCacheOnly(),
])
```

キャッシュオンリーでロードされ、ネットワークリクエストをブロックしない。

#### 4. システム初期化メッセージの送出 (L540-554)

`buildSystemInitMessage()`により、利用可能なツール、MCPクライアント、モデル、権限モード等の情報をSDKクライアントに通知する。

#### 5. query() ループの実行とメッセージディスパッチ (L675-1049)

`query()`関数（`src/query.ts`で定義）を`for await`で消費し、受信したメッセージを種類に応じて処理する。

主要なメッセージタイプと処理:

| メッセージタイプ | 処理内容 |
|---|---|
| `assistant` | `mutableMessages`に追加、正規化して`yield` |
| `user` | `mutableMessages`に追加、ターンカウント増加 |
| `stream_event` | `message_start`/`message_delta`/`message_stop`でトークン使用量を追跡 |
| `system` (compact_boundary) | コンパクト境界処理、前境界メッセージのGC解放 |
| `system` (api_error) | APIリトライ情報の送出 |
| `attachment` | 構造化出力、最大ターン到達、キューコマンド等の処理 |
| `tombstone` | メッセージ削除シグナル（スキップ） |
| `tool_use_summary` | ツール使用サマリーのSDK送出 |

#### 6. 予算チェック (L972-1001)

`maxBudgetUsd`が設定されている場合、`getTotalCost()`で現在の総コストと比較し、超過時は`error_max_budget_usd`結果を返す。

#### 7. 構造化出力リトライ制限 (L1004-1048)

`jsonSchema`が設定されている場合、`SYNTHETIC_OUTPUT_TOOL_NAME`のツール呼び出し回数を追跡し、最大リトライ数（デフォルト5回）を超えたら`error_max_structured_output_retries`を返す。

#### 8. 結果の生成 (L1058-1156)

- `isResultSuccessful()`で結果の成功判定
- テキスト結果の抽出
- `result`メッセージ（`success`または`error_during_execution`）の`yield`

## コスト追跡とトークンバジェット

### 使用量の追跡

トークン使用量は2段階で追跡される:

1. **メッセージレベル** (L658, L788-816): `stream_event`の`message_start`と`message_delta`で`currentMessageUsage`を更新し、`message_stop`で`totalUsage`に累積。
2. **セッションレベル** (L189): `totalUsage`フィールドが全ターンの累計を保持。

### コスト関連の外部関数

- `getTotalCost()` (`src/cost-tracker.ts`): 全APIコールの累計コスト
- `getTotalAPIDuration()`: 全API呼び出しの累計時間
- `getModelUsage()`: モデル別の使用量統計

## モデルリトライとエラーリカバリ

### モデルフォールバック

`QueryEngine`自体はフォールバックの設定を`query()`に渡すのみ。実際のフォールバック処理は`query.ts`の`queryLoop()`内で行われる（`FallbackTriggeredError`のキャッチ）。

### 権限拒否の追跡

`wrappedCanUseTool` (L244-271) がツール使用判定をラップし、拒否された場合に`permissionDenials`配列に記録。これはSDKの結果メッセージに含まれる。

## 会話履歴の管理

### メッセージの永続化

- ユーザーメッセージ: API呼び出し前にトランスクリプトに記録 (L450-463)
- アシスタントメッセージ: fire-and-forget で記録 (L727-728)
- コンパクト境界: 保存セグメントの尾部までフラッシュしてから記録 (L699-715)

### メッセージのGC (ガベージコレクション)

コンパクト境界メッセージ受信時 (L927-933)、境界より前のメッセージを`mutableMessages`と`messages`の両方から`splice`で削除し、メモリを解放する。

### スニップリプレイ

`HISTORY_SNIP`機能が有効な場合、`snipReplay`コールバック (L905-915) がスニップ境界メッセージを検出し、`mutableMessages`を置き換える。

## ask() 便利関数 (L1186-1295)

`ask()`は`QueryEngine`のワンショット利用向けラッパーである。`QueryEngine`インスタンスを作成し、`submitMessage()`を1回呼び出して`yield*`で全メッセージを転送する。SDK/ヘッドレスパスの主要なエントリポイント。

```typescript
export async function* ask({ ... }): AsyncGenerator<SDKMessage, void, unknown> {
  const engine = new QueryEngine({ ... })
  try {
    yield* engine.submitMessage(prompt, { uuid: promptUuid, isMeta })
  } finally {
    setReadFileCache(engine.getReadFileState())
  }
}
```

`HISTORY_SNIP`機能有効時、`snipReplay`コールバックが注入される (L1276-1284)。

## プロンプトキャッシュ管理

システムプロンプトの構築は`fetchSystemPromptParts()` (`src/utils/queryContext.ts`) を通じて行われ、以下の3要素をキャッシュキープレフィックスとして構成する:

1. **defaultSystemPrompt**: `getSystemPrompt()`から取得（ツール説明、モデル固有指示等）
2. **userContext**: CLAUDE.mdの内容、現在の日付
3. **systemContext**: git status、キャッシュブレーカー

これらは会話開始時に`memoize`でキャッシュされ、同一セッション内では再計算されない。`asSystemPrompt()`で`SystemPrompt`ブランド型に変換され、APIレイヤーで`cache_control`ブレークポイントの挿入に使われる。
