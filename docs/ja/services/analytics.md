# アナリティクスと可観測性

> Claude Code v2.1.88 ソースコード解析ドキュメント

## 概要

Claude Code のアナリティクス基盤は `src/services/analytics/` 配下の9ファイルで構成され、**デュアルシンクアーキテクチャ**（Anthropic 1st Party + Datadog）によるイベントロギング、GrowthBook によるフィーチャーフラグ管理、および使用量統計の収集を担う。

### ファイル構成

| ファイル | 役割 |
|---|---|
| `index.ts` | 公開 API。イベントキューイングとシンク接続 |
| `sink.ts` | シンク実装。Datadog と 1P へのルーティング |
| `datadog.ts` | Datadog Logs API へのバッチ送信 |
| `firstPartyEventLogger.ts` | OpenTelemetry ベースの 1st Party イベントロギング |
| `firstPartyEventLoggingExporter.ts` | 1P イベントの HTTP エクスポーター |
| `growthbook.ts` | GrowthBook フィーチャーフラグクライアント |
| `metadata.ts` | イベントメタデータのエンリッチメント |
| `config.ts` | アナリティクス有効/無効の共通判定 |
| `sinkKillswitch.ts` | 個別シンクの動的キルスイッチ |

---

## デュアルアナリティクスシンク

### アーキテクチャ

```
logEvent() / logEventAsync()
    |
    v
[index.ts] イベントキュー (シンク未接続時)
    |
    v
[sink.ts] logEventImpl()
    |
    +---> イベントサンプリング判定
    |
    +---> [datadog.ts] Datadog Logs API
    |         (PROTO フィールド除去済み)
    |
    +---> [firstPartyEventLogger.ts] 1P Event Logging
              (PROTO フィールド含む完全ペイロード)
```

### イベントキューイング（`index.ts`）

アナリティクスモジュールは**依存関係ゼロ**で設計されており、インポートサイクルを回避する（`index.ts` L7-8）。アプリ起動前に発生したイベントは内部キューに蓄積され、`attachAnalyticsSink()` の呼び出し時に `queueMicrotask` で非同期にドレインされる（`index.ts` L95-123）。これによりスタートアップパスにレイテンシを追加しない。

```typescript
// index.ts L95-98
export function attachAnalyticsSink(newSink: AnalyticsSink): void {
  if (sink !== null) { return }  // 冪等性の保証
  sink = newSink
```

### Datadog シンク（`datadog.ts`）

- **エンドポイント**: `https://http-intake.logs.us5.datadoghq.com/api/v2/logs`（L12）
- **バッチ処理**: 最大100件、15秒間隔でフラッシュ（L14-15）
- **ネットワークタイムアウト**: 5秒（L16）
- **許可イベントリスト**: `DATADOG_ALLOWED_EVENTS` セットで64種類のイベントのみ送信（L19-64）
- **タグフィールド**: `arch`, `platform`, `model`, `provider`, `version` 等13フィールドをDDタグとして送信（L66-83）

Datadog への送信は GrowthBook のフィーチャーゲート `tengu_log_datadog_events` で制御される（`sink.ts` L20-43）。ゲートの状態が未初期化の場合、前回セッションのキャッシュ値にフォールバックする。

### 1st Party イベントロギング（`firstPartyEventLogger.ts`）

OpenTelemetry SDK（`@opentelemetry/sdk-logs`）を基盤とし、`BatchLogRecordProcessor` によるバッチ処理と `FirstPartyEventLoggingExporter` による HTTP エクスポートを行う。

- **バッチ設定**: GrowthBook の `tengu_1p_event_batch_config` で動的に設定可能（L87-102）
  - `scheduledDelayMillis`: バッチ送信間隔
  - `maxExportBatchSize`: 最大バッチサイズ
  - `maxQueueSize`: 最大キューサイズ
  - `maxAttempts`: リトライ回数
  - `path` / `baseUrl`: エンドポイントのカスタマイズ
- **シャットダウン**: `shutdown1PEventLogging()` がプロセス終了前に呼ばれ、未送信イベントをフラッシュ（L116-128）

### シンクキルスイッチ（`sinkKillswitch.ts`）

GrowthBook の動的設定 `tengu_frond_boric`（難読化されたキー名）により、個別のシンク（`datadog` / `firstParty`）を実行時に無効化できる（`sinkKillswitch.ts` L1-25）。フェイルオープン設計で、設定が欠落または不正な場合はシンクは有効のままとなる。

### イベントサンプリング（`firstPartyEventLogger.ts` L30-85）

`tengu_event_sampling_config` による動的サンプリング設定:

- イベント名ごとに `sample_rate`（0-1）を指定可能
- 設定のないイベントは100%記録
- `sample_rate: 0` は完全ドロップ、`sample_rate: 1` はメタデータ付加なしの完全記録
- サンプリングされたイベントには `sample_rate` メタデータが自動付加（`sink.ts` L58-61）

---

## GrowthBook フィーチャーフラグの統合

### クライアント管理（`growthbook.ts`）

GrowthBook SDK (`@growthbook/growthbook`) を使用し、リモート評価（Remote Eval）によるフィーチャーフラグ管理を行う。

#### ユーザー属性（`GrowthBookUserAttributes` 型、L32-47）

ターゲティングに使用される属性:

| 属性 | 内容 |
|---|---|
| `id` | ユーザーID |
| `sessionId` | セッションID |
| `deviceID` | デバイスID |
| `platform` | `win32` / `darwin` / `linux` |
| `organizationUUID` | 組織UUID |
| `accountUUID` | アカウントUUID |
| `userType` | ユーザータイプ |
| `subscriptionType` | サブスクリプション種別 |
| `rateLimitTier` | レート制限ティア |
| `email` | メールアドレス |
| `appVersion` | アプリバージョン |
| `github` | GitHub Actions メタデータ |

#### キャッシュと初期化（L59-94）

- **リモート評価キャッシュ**: `remoteEvalFeatureValues` マップにサーバー応答をキャッシュ
- **エクスポージャーログの重複排除**: `loggedExposures` セットで同一セッション内の重複を防止（L88-89）
- **再初期化追跡**: `reinitializingPromise` で認証変更時のセキュリティゲートチェックが古い値を返さないよう保証（L93-94）
- **リフレッシュ通知**: `onGrowthBookRefresh()` によるサブスクライバーパターンで、フィーチャー値の変更を監視可能（L139-157）

#### 環境変数オーバーライド（L159-192）

`CLAUDE_INTERNAL_FC_OVERRIDES` 環境変数により、ant ユーザーのみフィーチャーフラグの値をローカルでオーバーライド可能。評価ハーネスでの特定フラグ構成テストに使用。

```bash
CLAUDE_INTERNAL_FC_OVERRIDES='{"my_feature": true}' claude
```

---

## イベントロギングと PII スクラビング

### PII 保護の型システム

Claude Code は TypeScript の型システムを利用した独自の PII 保護メカニズムを持つ。

#### `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`（`index.ts` L19）

`never` 型のマーカーで、文字列値をアナリティクスメタデータに含める際に開発者が「コードやファイルパスを含まない」ことを明示的にキャストにより宣言する必要がある。実行時の制約ではなくコードレビュー時の注意喚起として機能する。

#### `AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED`（`index.ts` L33）

PII タグ付きの proto カラムに送信される値のマーカー型。1P エクスポーターのみが `_PROTO_*` キーを参照し、他のシンク（Datadog）には `stripProtoFields()` により除去されて送信される（`index.ts` L45-58）。

#### `_PROTO_*` キーの分離

```
イベントペイロード { event: "...", _PROTO_user_name: "..." }
    |
    +---> Datadog: { event: "..." }  // _PROTO_ 除去済み
    |
    +---> 1P: { event: "...", _PROTO_user_name: "..." }  // proto フィールドにマッピング
```

`stripProtoFields()` は `_PROTO_` プレフィックスのキーがない場合は同一参照を返し、不要なオブジェクトコピーを回避する（`index.ts` L45-58）。

#### MCP ツール名のサニタイズ（`metadata.ts` L70-77）

MCP ツール名は `mcp__<server>__<tool>` 形式でユーザー固有のサーバー構成を含む可能性があるため、`sanitizeToolNameForAnalytics()` で一律 `'mcp_tool'` に置換される。ビルトインツール名（`Bash`, `Read`, `Write` 等）はそのまま記録される。

### アナリティクス無効化条件（`config.ts`）

以下の場合にアナリティクスが完全無効化される:

1. **テスト環境**: `NODE_ENV === 'test'`
2. **サードパーティクラウドプロバイダー**: `CLAUDE_CODE_USE_BEDROCK`, `CLAUDE_CODE_USE_VERTEX`, `CLAUDE_CODE_USE_FOUNDRY`
3. **プライバシー設定**: テレメトリ無効化（`no-telemetry` / `essential-traffic`）

---

## 使用量集計

### ClaudeCodeStats（`src/utils/stats.ts`）

セッション横断的な使用量統計を収集・集計するモジュール。JSONL 形式のトランスクリプトファイルを解析して以下の統計を算出する。

#### 主要な統計データ（`ClaudeCodeStats` 型、`stats.ts` L53-87）

| カテゴリ | データ |
|---|---|
| アクティビティ概要 | 総セッション数、総メッセージ数、総日数、アクティブ日数 |
| ストリーク | 現在の連続日数、最長連続日数、開始/終了日 |
| 日次アクティビティ | 日付ごとのメッセージ数、セッション数、ツールコール数（ヒートマップ用） |
| 日次モデルトークン | 日付・モデルごとのトークン使用量（チャート用） |
| モデル使用量 | モデルごとの集約使用量（`ModelUsage`型） |
| 時間統計 | 最初/最後のセッション日、ピークアクティビティ日/時間帯 |
| 投機実行 | `totalSpeculationTimeSavedMs` - 投機的実行により節約された時間 |
| ショット統計 | `shotDistribution` / `oneShotRate` - ant ユーザーのみ |

#### キャッシュ戦略（`statsCache.ts` 連携）

`stats.ts` は `statsCache.ts` と連携してパフォーマンスを最適化する:

- `loadStatsCache()` / `saveStatsCache()`: ディスクキャッシュの読み書き
- `mergeCacheWithNewStats()`: キャッシュと新規データのマージ
- `withStatsCacheLock()`: 並行アクセスのロック制御
- 日付文字列ヘルパー: `getTodayDateString()`, `getYesterdayDateString()`, `isDateBefore()`

---

## 診断ログとプロファイリング

### イベントメタデータのエンリッチメント（`metadata.ts`）

すべてのアナリティクスイベントに以下のコンテキスト情報が自動付加される:

- **環境情報**: プラットフォーム、アーキテクチャ、WSL バージョン、Linux ディストロ情報、VCS 種別
- **セッション情報**: セッションID、親セッションID、クライアントタイプ、インタラクティブモード
- **モデル情報**: メインループモデル、モデルベータ
- **ユーザー情報**: サブスクリプションタイプ、ユーザータイプ、Kairos アクティブ状態
- **リポジトリ情報**: リモートハッシュ（`getRepoRemoteHash()`）
- **チーム情報**: チーム名、エージェントID、チームメイトフラグ

### スタートアップ時の初期化フロー

```
アプリ起動
    |
    v
[sink.ts] initializeAnalyticsSink()
    |  - attachAnalyticsSink() を呼び出し
    |  - キュー内イベントをドレイン
    |
    v
[sink.ts] initializeAnalyticsGates()
    |  - Datadog ゲートの状態を更新
    |
    v
[growthbook.ts] GrowthBook クライアント初期化
    |  - リモート評価の取得
    |  - キャッシュの更新
    |  - 保留中のエクスポージャーログの送信
    |
    v
通常運用（イベント送信開始）
```

初期化完了前のイベントはキャッシュされた設定値を使用する。これにより初期化中のデータ損失を防止する（`sink.ts` L37-42、`growthbook.ts` L93-94）。

---

## セキュリティと信頼性の設計原則

1. **PII 保護**: 型システムによる明示的検証、`_PROTO_*` キーの自動分離、MCP ツール名のサニタイズ
2. **フェイルオープン**: キルスイッチ設定の欠落時はシンクを維持（データ損失より過剰送信を許容）
3. **冪等性**: `attachAnalyticsSink()` の複数呼び出しに対する安全性
4. **遅延初期化**: イベントキューイングにより起動パスへの影響を最小化
5. **動的制御**: GrowthBook によるサンプリング率、バッチ設定、シンク有効/無効の実行時変更
6. **プライバシー尊重**: サードパーティプロバイダーおよびテレメトリ無効化設定の厳密な遵守
