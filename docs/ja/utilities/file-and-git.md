# ファイル操作とGit統合

Claude Code v2.1.88 におけるファイルシステム操作、Git リポジトリ統合、ファイル履歴追跡の技術解析。

## 概要

Claude Code はファイル操作とバージョン管理を抽象化した複数のユーティリティモジュールを持つ。

| モジュール | ファイルパス | 役割 |
|---|---|---|
| file.ts | `src/utils/file.ts` | ファイル読み書き、エンコーディング検出 |
| git.ts | `src/utils/git.ts` | Git リポジトリの検出・操作 |
| fileHistory.ts | `src/utils/fileHistory.ts` | ファイル変更のバックアップと履歴管理 |
| fsOperations.ts | `src/utils/fsOperations.ts` | ファイルシステム操作の抽象化インターフェース |

---

## 1. ファイル操作ユーティリティ (`file.ts`)

### 基本型

```typescript
type File = {
  filename: string
  content: string
}
```

### ファイルの存在確認

`pathExists()` (`src/utils/file.ts` L39-46) は非同期の存在確認。`stat()` を呼び出し、エラー時は `false` を返す。

### ファイル読み取り

- `readFileSafe()` (L50-58): 同期読み取り。`getFsImplementation()` を経由して抽象化された FS 実装を使用する。失敗時は `null` を返す。
- `MAX_OUTPUT_SIZE` (L48): 0.25 MB。出力サイズの上限。

### ファイル変更時刻

タイムスタンプの精度問題に対処するため、`Math.floor()` で正規化を行う:

- `getFileModificationTime()` (L66-69): 同期版。`statSync().mtimeMs` を切り捨て。
- `getFileModificationTimeAsync()` (L77-82): 非同期版。ネットワークドライブや低速ディスクでの slow-operation インジケータ発生を回避するため、非同期パスで使用する。

### ファイル書き込み

`writeTextContent()` (L84-98) はエンコーディングと改行コードを考慮した書き込み:

- CRLF モードの場合、まず既存の `\r\n` を `\n` に正規化してから再度 `\r\n` に変換する。これにより `\r\r\n` の二重変換を防止する。

### エンコーディング検出

- `detectFileEncoding()` (L100-118): シンボリックリンクを解決した上でエンコーディングを検出。エラー時はデフォルトで `utf8` を返す。
- `detectLineEndings()` (L120-135): ファイル先頭 4096 バイトから改行コードを検出。`LF` または `CRLF` を返す。

### パス処理

- `convertLeadingTabsToSpaces()` (L137-142): 行頭のタブをスペース (2文字) に変換。タブが含まれない場合は正規表現をスキップして高速に返す。
- `getAbsoluteAndRelativePaths()` (L144-150): パスの絶対・相対パスのペアを計算。

---

## 2. Git 操作統合 (`git.ts`)

### Git ルートの検出

#### `findGitRoot()` (`src/utils/git.ts` L27-109)

ディレクトリツリーを上方向に走査して `.git` ディレクトリまたはファイルを探す。

- `.git` がディレクトリの場合: 通常のリポジトリ
- `.git` がファイルの場合: worktree またはサブモジュール
- **メモ化**: LRU キャッシュ (最大50エントリ) で結果をキャッシュ。`gitDiff` が `dirname(file)` で呼び出すため、多数のディレクトリでのエントリ蓄積を防止する。
- **NFC 正規化**: macOS の Unicode 正規化の問題に対応して `.normalize('NFC')` を適用。
- **Sentinel パターン**: `GIT_ROOT_NOT_FOUND` シンボルにより `null` と未検出を区別。

#### `findCanonicalGitRoot()` (L185-210)

worktree をメインリポジトリのルートに解決する。`.git` ファイル -> `gitdir:` -> `commondir` チェーンを辿る。

**セキュリティ検証** (L143-176):
悪意のあるリポジトリによるパストラバーサル攻撃を防止するため、二重の検証を実施:

1. `worktreeGitDir` が `<commonDir>/worktrees/` の直接の子であることを確認
2. `<worktreeGitDir>/gitdir` が `<gitRoot>/.git` にバックリンクしていることを確認

両方の検証が必須。(1) のみでは被害者が信頼済みリポジトリの worktree を持つ場合に失敗し、(2) のみでは攻撃者が `worktreeGitDir` を制御するため失敗する。

### Git 実行ファイルの解決

`gitExe()` (L212-216) は `which` コマンドで `git` のパスを一度だけ解決し、メモ化する。これにより毎回のプロセス起動時のパス検索を回避する。

### Git 状態の確認

- `getIsGit()` (L218-229): 現在の作業ディレクトリが Git リポジトリ内にあるかを判定。メモ化により一度だけ実行。
- `getGitDir()` (L231-233): `resolveGitDir()` への委譲。
- `isAtGitRoot()` (L235-249): 現在の作業ディレクトリが Git リポジトリのルートと一致するかを判定。シンボリックリンクを解決して正確に比較する。

### キャッシュ活用

`src/utils/git/gitFilesystem.ts` から以下のキャッシュ済み関数をインポート:

- `getCachedBranch()` -- ブランチ名
- `getCachedDefaultBranch()` -- デフォルトブランチ名
- `getCachedHead()` -- HEAD の SHA
- `getCachedRemoteUrl()` -- リモート URL
- `getWorktreeCountFromFs()` -- Worktree 数
- `isShallowClone` -- シャロークローンかの判定

---

## 3. ファイル履歴追跡 (`fileHistory.ts`)

### 概要

ファイルの変更をバックアップし、元に戻す機能を提供する。`~/.claude/` 配下にバックアップファイルを保存する。

### データ型

```typescript
// バックアップエントリ (src/utils/fileHistory.ts L32-37)
type FileHistoryBackup = {
  backupFileName: string | null  // null = ファイルが存在しないバージョン
  version: number
  backupTime: Date
}

// スナップショット (L39-43)
type FileHistorySnapshot = {
  messageId: UUID
  trackedFileBackups: Record<string, FileHistoryBackup>
  timestamp: Date
}

// 状態管理 (L45-52)
type FileHistoryState = {
  snapshots: FileHistorySnapshot[]
  trackedFiles: Set<string>
  snapshotSequence: number  // 単調増加カウンタ (古いスナップショットの退避後も増加)
}
```

### スナップショット上限

`MAX_SNAPSHOTS` (L54): **100** スナップショットまで保持。超過した場合は古いものが退避される。`snapshotSequence` は退避後も増加し続け、`useGitDiffStats` がアクティビティシグナルとして使用する。

### 有効化条件

`fileHistoryEnabled()` (L63-71):

1. **非インタラクティブセッション (SDK)**: `CLAUDE_CODE_ENABLE_SDK_FILE_CHECKPOINTING` 環境変数が truthy
2. **インタラクティブセッション**: グローバル設定の `fileCheckpointingEnabled` が `false` でなく、`CLAUDE_CODE_DISABLE_FILE_CHECKPOINTING` が設定されていないこと

### ファイル編集の追跡

`fileHistoryTrackEdit()` (L86-100) はファイル編集前に呼び出され、現在の内容のバックアップを作成する。

- バックアップファイル名はハッシュベース (例: `{hash}@v1`)
- 既存バックアップの存在を確認し、重複書き込みを防止

### 差分統計

```typescript
type DiffStats = {
  filesChanged?: string[]
  insertions: number
  deletions: number
} | undefined
```

`diffLines` (diff ライブラリ) を使用した行単位の差分計算を行う。

---

## 4. ファイルシステム操作の抽象化 (`fsOperations.ts`)

### FsOperations インターフェース

`src/utils/fsOperations.ts` L23-123 で定義される `FsOperations` 型は、Node.js `fs` モジュールのサブセットを型安全に抽象化する。

**目的**: モック、仮想ファイルシステム等の代替実装への差し替えを可能にする。

#### 同期操作

| メソッド | 説明 |
|---|---|
| `existsSync(path)` | ファイル/ディレクトリの存在確認 |
| `statSync(path)` | ファイル情報取得 |
| `lstatSync(path)` | シンボリックリンクを辿らずにファイル情報取得 |
| `readFileSync(path, options)` | ファイル内容を文字列として読み取り |
| `readFileBytesSync(path)` | ファイル内容を Buffer として読み取り |
| `readSync(path, options)` | 指定バイト数の読み取り |
| `appendFileSync(path, data)` | ファイルへの追記 |
| `copyFileSync(src, dest)` | ファイルのコピー |
| `unlinkSync(path)` | ファイルの削除 |
| `renameSync(old, new)` | ファイルの移動/リネーム |
| `linkSync(target, path)` | ハードリンクの作成 |
| `symlinkSync(target, path)` | シンボリックリンクの作成 |
| `readlinkSync(path)` | シンボリックリンクの読み取り |
| `realpathSync(path)` | シンボリックリンクを解決した正規パスの取得 |
| `mkdirSync(path, options)` | ディレクトリの再帰作成 |
| `readdirSync(path)` | ディレクトリ内容の一覧 (Dirent) |
| `readdirStringSync(path)` | ディレクトリ内容の一覧 (文字列) |
| `isDirEmptySync(path)` | ディレクトリが空かの判定 |
| `rmdirSync(path)` | 空ディレクトリの削除 |
| `rmSync(path, options)` | ファイル/ディレクトリの削除 (recursive オプション対応) |
| `createWriteStream(path)` | 書き込みストリームの作成 |

#### 非同期操作

| メソッド | 説明 |
|---|---|
| `stat(path)` | ファイル情報取得 |
| `readdir(path)` | ディレクトリ内容の一覧 |
| `unlink(path)` | ファイルの削除 |
| `rmdir(path)` | 空ディレクトリの削除 |
| `rm(path, options)` | ファイル/ディレクトリの削除 |
| `mkdir(path, options)` | ディレクトリの作成 |
| `readFile(path, options)` | ファイル内容の読み取り |
| `rename(old, new)` | ファイルの移動/リネーム |
| `readFileBytes(path, maxBytes?)` | バイナリ読み取り (上限バイト数指定可) |

### safeResolvePath()

`safeResolvePath()` (L138-178) はパス解決のセーフティラッパー:

1. **UNC パスのブロック** (L144-146): `//` または `\\` で始まるパスは DNS/SMB リクエストを防止するためにそのまま返す
2. **特殊ファイルの検出** (L149-161): FIFO、ソケット、デバイスファイルの場合は `realpathSync` が FIFO のライターを待ってハングするため、元のパスを返す
3. **シンボリックリンクの解決** (L163-171): `realpathSync` で正規パスに解決。`isCanonical: true` フラグにより、呼び出し元が追加のシンボリックリンク解決をスキップできる
4. **エラーハンドリング** (L172-177): `ENOENT`、壊れたシンボリックリンク、`EACCES`、`ELOOP` 等のすべてのエラーで元のパスを返し、操作を継続させる

### isDuplicatePath()

`isDuplicatePath()` (L187-198) はシンボリックリンクを解決して重複ファイルを検出する。`loadedPaths` セットで既に処理済みのパスを追跡する。

---

## 5. パス操作とバリデーション

### パス展開

`expandPath()` (`src/utils/path.ts` からインポート) はチルダ (`~`) をホームディレクトリに展開する。

### パスの正規化

`file.ts` では `path` モジュールの以下の関数を活用:

- `basename()` -- ファイル名の抽出
- `dirname()` -- ディレクトリ部分の抽出
- `extname()` -- 拡張子の抽出
- `isAbsolute()` -- 絶対パスの判定
- `join()` -- パスの結合
- `normalize()` -- パスの正規化
- `relative()` -- 相対パスの計算
- `resolve()` -- 絶対パスへの解決
- `sep` -- パス区切り文字

### バイナリファイルの検出

`git.ts` では `hasBinaryExtension()` と `isBinaryContent()` (`src/constants/files.ts`) をインポートしてバイナリファイルの判定を行う。これにより、テキスト処理が不適切なファイルを自動的にスキップする。
