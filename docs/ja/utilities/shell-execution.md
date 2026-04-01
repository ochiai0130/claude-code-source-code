# シェル実行 (Shell.ts / ShellCommand.ts)

Claude Code v2.1.88 のシェルコマンド実行基盤に関する包括的な分析ドキュメント。

## 関連ソースファイル

| ファイル | サイズ | 役割 |
|---------|--------|------|
| `src/utils/Shell.ts` | ~16KB | シェルの検出、設定、コマンド実行 |
| `src/utils/ShellCommand.ts` | ~14KB | プロセスラッパー、ライフサイクル管理 |

---

## 1. Shellクラスの構造と責務

### シェルの検出と選択

`findSuitableShell()` 関数（`Shell.ts:73-137`）が利用可能なシェルを検出する。検出ロジックは以下の優先順位:

1. **CLAUDE_CODE_SHELL 環境変数**: 明示的なシェル指定（bash/zshのみサポート）
2. **SHELL 環境変数**: ユーザーの優先シェル（bash/zshのみ考慮）
3. **which によるパス検出**: `zsh` と `bash` を `which` コマンドで検索
4. **フォールバックパス**: `/bin`, `/usr/bin`, `/usr/local/bin`, `/opt/homebrew/bin` を順次探索

**実行可能性の検証** (`Shell.ts:50-68`):
- `accessSync(shellPath, fsConstants.X_OK)` でアクセス権をチェック
- 失敗時のフォールバック: `execFileSync(shellPath, ['--version'])` を1秒タイムアウトで実行（Nix環境等での互換性対応）

### ShellConfig とプロバイダシステム

```typescript
// Shell.ts:46-48
export type ShellConfig = {
  provider: ShellProvider
}
```

シェルプロバイダは `ShellProvider` インタフェースを実装し、シェル固有のコマンド構築・引数生成を担当:

- **bashProvider**: bash/zsh 用（`Shell.ts:141-143`）
- **powershellProvider**: PowerShell 用（`Shell.ts:148-154`）

`getShellConfig()` は `memoize` でラップされ、セッション中1回だけ初期化される（`Shell.ts:146`）。

### resolveProvider マッピング

```typescript
// Shell.ts:156-159
const resolveProvider: Record<ShellType, () => Promise<ShellProvider>> = {
  bash: async () => (await getShellConfig()).provider,
  powershell: getPsProvider,
}
```

---

## 2. ShellCommandの実行モデル

### exec() 関数

`exec()` 関数（`Shell.ts:181-442`）がシェルコマンド実行のエントリポイント。

**ExecOptions 型** (`Shell.ts:161-175`):

| オプション | 型 | 説明 |
|-----------|-----|------|
| `timeout` | `number` | タイムアウト（デフォルト: 30分） |
| `onProgress` | callback | 出力の進捗コールバック |
| `preventCwdChanges` | `boolean` | cwdの変更を抑止 |
| `shouldUseSandbox` | `boolean` | サンドボックス使用 |
| `shouldAutoBackground` | `boolean` | 自動バックグラウンド化 |
| `onStdout` | callback | stdoutのリアルタイムコールバック |

**実行フロー:**

1. **プロバイダ解決**: `resolveProvider[shellType]()` でシェルプロバイダを取得
2. **コマンド構築**: `provider.buildExecCommand()` でシェル固有のコマンド文字列を生成
3. **cwd検証**: `realpath(cwd)` で作業ディレクトリの存在を確認、不存在時は `getOriginalCwd()` にフォールバック（`Shell.ts:218-238`）
4. **アボートチェック**: シグナルが既にアボート済みの場合は `createAbortedCommand()` を返す
5. **サンドボックスラップ**: `shouldUseSandbox` の場合、`SandboxManager.wrapWithSandbox()` でコマンドをラップ
6. **プロセス生成**: `spawn()` で子プロセスを生成
7. **wrapSpawn()**: `ShellCommand` インスタンスとしてラップ
8. **cwd追跡**: コマンド完了後に `pwd -P` の出力からcwdを更新

### 出力モード: ファイルモード vs パイプモード

**ファイルモード**（デフォルト、`Shell.ts:289-313`）:
- stdout/stderr を同一のファイルディスクリプタに直接書き出し
- POSIX の `O_APPEND` で原子的な書き込みを保証
- JS を経由しないため高パフォーマンス
- セキュリティ: `O_NOFOLLOW` でシンボリックリンク攻撃を防止

**パイプモード**（`onStdout` 指定時、`Shell.ts:284`）:
- `stdio: ['pipe', 'pipe', 'pipe']` でストリーム経由
- `StreamWrapper` クラスでリアルタイムデータを `TaskOutput` に転送
- フック実行時に使用

### 環境変数の設定

`spawn()` に渡される環境変数（`Shell.ts:316-328`）:

```typescript
env: {
  ...subprocessEnv(),        // 親プロセスの環境変数
  SHELL: binShell,           // bashの場合のみ設定
  GIT_EDITOR: 'true',        // gitのエディタ起動を抑止
  CLAUDECODE: '1',           // Claude Code内であることを示すフラグ
  ...envOverrides,           // プロバイダ固有のオーバーライド
  // Ant環境ではセッションIDも追加
}
```

---

## 3. プロセス管理（生成、監視、終了）

### ShellCommandImpl クラス

`ShellCommandImpl` クラス（`ShellCommand.ts:114-382`）が子プロセスのライフサイクル全体を管理する。

**ステータス遷移:**
```
running → backgrounded → completed
running → completed
running → killed
backgrounded → completed
backgrounded → killed
```

### ShellCommand インタフェース

```typescript
// ShellCommand.ts:32-47
export type ShellCommand = {
  background: (backgroundTaskId: string) => boolean
  result: Promise<ExecResult>
  kill: () => void
  status: 'running' | 'backgrounded' | 'completed' | 'killed'
  cleanup: () => void
  onTimeout?: (callback: (backgroundFn: (taskId: string) => boolean) => void) => void
  taskOutput: TaskOutput
}
```

### ExecResult 型

```typescript
// ShellCommand.ts:13-30
export type ExecResult = {
  stdout: string
  stderr: string
  code: number
  interrupted: boolean
  backgroundTaskId?: string
  backgroundedByUser?: boolean
  assistantAutoBackgrounded?: boolean
  outputFilePath?: string      // 大出力のファイルパス
  outputFileSize?: number
  outputTaskId?: string
  preSpawnError?: string       // 生成前エラー
}
```

### プロセスの終了

`#doKill()` メソッド（`ShellCommand.ts:337-343`）:
- `tree-kill` ライブラリで子プロセスツリー全体を `SIGKILL` で終了
- 孫プロセスの孤児化を防止

**アボート処理** (`ShellCommand.ts:186-193`):
- `AbortSignal` の `reason` が `'interrupt'`（ユーザーが新メッセージを送信）の場合はキルせず、バックグラウンド化を許可
- それ以外の場合は即座にキル

---

## 4. 出力のキャプチャとストリーミング

### StreamWrapper クラス

`StreamWrapper` クラス（`ShellCommand.ts:66-104`）はパイプモードでストリームからデータを読み取り `TaskOutput` に転送する:

- `stream.setEncoding('utf-8')` で文字列モードに設定（Buffer→String変換のオーバーヘッドを削減）
- `#dataHandler()` でstdout/stderrを区別して `TaskOutput` に書き込み
- `cleanup()` でイベントリスナーを解除し、参照を解放してGCを促進

### TaskOutput の役割

`TaskOutput` クラス（`src/utils/task/TaskOutput.ts`）が出力の永続化と進捗報告を管理:

- ファイルモード: ディスク上のファイルからデータを読み取り
- パイプモード: インメモリバッファにデータを蓄積
- `onProgress` コールバックで最新の出力行と統計情報を通知

### 大出力の処理

コマンド完了時（`ShellCommand.ts:291-315`）:
- `taskOutput.outputFileRedundant` が `true`（小ファイル）: 出力全体が `result.stdout` に格納され、ファイルは削除
- `false`（大ファイル）: `result.outputFilePath` でファイルパスを通知、`result.outputFileSize` でサイズを報告

---

## 5. タイムアウト処理

### タイムアウトの設定と動作

デフォルトタイムアウト: **30分**（`Shell.ts:44`）

`ShellCommandImpl.#handleTimeout()` 静的メソッド（`ShellCommand.ts:135-141`）:

1. **自動バックグラウンドが有効** かつ **コールバックが登録済み**: `onTimeoutCallback` を呼び出し、コールバックがバックグラウンド化を決定
2. **それ以外**: `SIGTERM`（コード143）でプロセスを終了

タイムアウト時のエラーメッセージ:
```
Command timed out after ${formatDuration(timeout)}
```

### サイズウォッチドッグ

バックグラウンドタスクにはサイズウォッチドッグが設定される（`ShellCommand.ts:239-261`）:

- **監視間隔**: 5秒（`SIZE_WATCHDOG_INTERVAL_MS`）
- **上限**: `MAX_TASK_OUTPUT_BYTES`（ディスク容量保護、768GBインシデントの再発防止）
- 超過時: `SIGKILL` で強制終了、stderrに `Background command killed: output file exceeded ...` を追記

---

## 6. バックグラウンド実行

### background() メソッド

`background()` メソッド（`ShellCommand.ts:348-366`）:

1. ステータスが `'running'` の場合のみ有効
2. `backgroundTaskId` を設定、ステータスを `'backgrounded'` に変更
3. タイムアウトとアボートリスナーをクリーンアップ
4. **ファイルモード**: サイズウォッチドッグを開始
5. **パイプモード**: インメモリバッファをディスクにスピル（`taskOutput.spillToDisk()`）

### AbortedShellCommand / createFailedCommand

**AbortedShellCommand** (`ShellCommand.ts:408-435`): 実行前にアボートされたコマンドの表現。デフォルト終了コード145。

**createFailedCommand** (`ShellCommand.ts:447-465`): spawn前にエラーが発生した場合（例: cwdが存在しない）の表現。`preSpawnError` フィールドにエラーメッセージを格納。

---

## 7. cwd（作業ディレクトリ）の追跡

コマンド完了後の cwd 更新ロジック（`Shell.ts:385-421`）:

1. `cwdFilePath` から `pwd -P` の出力を読み取り（同期的 `readFileSync` — microtaskバウンダリでのレース条件を防止）
2. Windows環境ではPOSIXパスをWindowsパスに変換
3. NFC正規化でUnicode比較の誤判定を防止（macOS APFSのNFD問題）
4. cwd変更時: `setCwdState()`, `invalidateSessionEnvCache()`, `onCwdChangedForHooks()` を呼び出し
5. 一時ファイルをクリーンアップ

### setCwd() 関数

`setCwd()` 関数（`Shell.ts:447-474`）:
- 相対パスを絶対パスに解決
- `realpathSync()` でシンボリックリンクを解決
- `setCwdState()` でグローバル状態を更新
- ENOENT時はユーザーフレンドリーなエラーメッセージ

---

## サンドボックス統合

サンドボックスが有効な場合（`Shell.ts:259-273`）:

1. `SandboxManager.wrapWithSandbox()` でコマンドをラップ
2. サンドボックス一時ディレクトリを `0o700` 権限で作成
3. PowerShellのサンドボックス: `/bin/sh` を内部シェルとして使用（pwshのプロファイル読み込みによる遅延を回避）
4. コマンド完了後: `SandboxManager.cleanupAfterCommand()` でLinuxの bwrap が作成したゴーストドットファイルをクリーンアップ
