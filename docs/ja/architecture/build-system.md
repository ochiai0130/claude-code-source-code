# Claude Code v2.1.88 ビルドシステム

## 概要

Claude Code のビルドシステムは、Bun ランタイムのコンパイル時機能（`feature()`, `MACRO`）に強く依存した設計となっている。ソースコードの研究・分析用に、esbuild を用いた代替ビルドパイプラインが提供されている。

---

## ビルドの全体像

```
src/ (TypeScriptソース)
  │
  ├─ prepare-src.mjs    (1) bun:bundle → スタブに置換, MACRO → リテラルに置換
  │
  ├─ build.mjs          (2) src/ → build-src/ コピー → 変換 → esbuild バンドル
  │
  └─ stub-modules.mjs   (3) 不足モジュールのスタブ自動生成
  │
  └─ dist/cli.js (~12MB, 単一ファイル)
```

---

## ビルドスクリプトの役割

### `scripts/build.mjs` — メインビルドスクリプト

**ファイル**: `scripts/build.mjs`

6つのフェーズで構成される:

#### Phase 1: ソースコピー (L56-59)
```
src/ → build-src/src/ (原本を汚さない)
```

#### Phase 2: ソース変換 (L65-117)
全 `.ts`/`.tsx` ファイルを走査し、以下の変換を適用:

1. **`feature('X')` → `false`**: Bun のコンパイル時分岐を全て `false` に置換。これにより、内部専用の機能コードがデッドコードとなる。
   ```javascript
   // 変換前
   if (feature('DAEMON') && args[0] === 'daemon') { ... }
   // 変換後
   if (false && args[0] === 'daemon') { ... }
   ```

2. **`MACRO.X` → リテラル値**: ビルド時定数をハードコードされた文字列に置換。
   ```javascript
   // 変換前
   console.log(`${MACRO.VERSION} (Claude Code)`)
   // 変換後
   console.log(`${'2.1.88'} (Claude Code)`)
   ```
   定義される MACRO (`build.mjs` L68-78):
   | MACRO | 値 |
   |-------|-----|
   | `MACRO.VERSION` | `'2.1.88'` |
   | `MACRO.BUILD_TIME` | `''` |
   | `MACRO.FEEDBACK_CHANNEL` | GitHub Issues URL |
   | `MACRO.ISSUES_EXPLAINER` | GitHub Issues URL |
   | `MACRO.NATIVE_PACKAGE_URL` | `'@anthropic-ai/claude-code'` |
   | `MACRO.PACKAGE_URL` | `'@anthropic-ai/claude-code'` |
   | `MACRO.VERSION_CHANGELOG` | `''` |

3. **`bun:bundle` インポート除去**: `import { feature } from 'bun:bundle'` をコメントに置換。

4. **`global.d.ts` インポート除去**: 型のみのインポートを削除。

#### Phase 3: エントリラッパー生成 (L123-128)
```javascript
#!/usr/bin/env node
// Claude Code v2.1.88 — built from source
import './src/entrypoints/cli.tsx'
```

#### Phase 4: 反復バンドル (L134-229)

最大5ラウンドの「ビルド → エラー解析 → スタブ生成 → リトライ」ループ:

1. esbuild でバンドルを試行
2. `Could not resolve "..."` エラーを解析
3. 不足モジュールのスタブファイルを自動生成
4. 成功するか、修復不能なエラーになるまで繰り返し

esbuild の設定 (`build.mjs` L149-166):
```
--bundle                    # 単一ファイルにバンドル
--platform=node             # Node.js 向け
--target=node18             # Node.js 18 互換
--format=esm                # ESM 形式
--packages=external         # 外部パッケージは除外
--external:bun:*            # Bun 固有モジュールは除外
--sourcemap                 # ソースマップ生成
```

---

### `scripts/prepare-src.mjs` — ソース前処理

**ファイル**: `scripts/prepare-src.mjs`

`npm run prepare-src` で実行される。`src/` ディレクトリ内のファイルを**直接**書き換える（`build.mjs` とは異なり、コピーを作らない）。

主な処理:
1. **`bun:bundle` → スタブ置換** (L40-51): 相対パスを計算し、`stubs/bun-bundle.js` へのインポートに書き換える。
   ```javascript
   // 変換前
   import { feature } from 'bun:bundle'
   // 変換後
   import { feature } from '../../stubs/bun-bundle.js'
   ```

2. **MACRO 置換** (L54-76): `build.mjs` と同様のリテラル置換。

3. **スタブファイル生成** (L93-113):
   - `stubs/bun-ffi.ts` — `bun:ffi` のスタブ
   - `stubs/global.d.ts` — MACRO の型宣言

---

### `scripts/stub-modules.mjs` — 不足モジュールスタブ生成

**ファイル**: `scripts/stub-modules.mjs`

esbuild のエラー出力を解析し、不足しているモジュールのスタブを自動生成する。フィーチャーゲートで除去されたコードが参照するモジュール（例: `./assistant/index.js`, `./coordinator/coordinatorMode.js`）は、ソースツリーに存在しない場合がある。

処理の流れ:
1. esbuild を実行してエラーを収集 (L22-28)
2. `Could not resolve` エラーからモジュールパスを抽出 (L30-39)
3. `grep` でインポート元ファイルを特定 (L51-55)
4. 適切なパスにスタブファイルを生成 (L59-121)
5. 再度 esbuild でバンドルを試行 (L127-158)

生成されるスタブの種類:
- `.d.ts` → `export {}`
- `.txt` / `.md` → 空ファイル
- `.ts` / `.tsx` → デフォルトエクスポート付きの空モジュール

---

### `scripts/transform.mjs` — 代替ビルドスクリプト

**ファイル**: `scripts/transform.mjs`

`build.mjs` の簡易版。同様のアプローチだが、MACRO をグローバル変数として注入する方式を採用:

```javascript
// エントリラッパー内で MACRO をグローバルに定義
const MACRO = {
  VERSION: '2.1.88',
  BUILD_TIME: '',
  ...
}
globalThis.MACRO = MACRO
import './src/entrypoints/cli.tsx'
```

esbuild の `--define` オプションも使用 (L122-123):
```
--define:process.env.USER_TYPE='"external"'
--define:process.env.CLAUDE_CODE_VERSION='"2.1.88"'
```

---

## stubs ディレクトリの役割

**ファイル**: `stubs/bun-bundle.ts`

```typescript
// Stub for bun:bundle — feature() is compile-time in Bun; replaced by build script
export function feature(_flag: string): boolean {
  return false
}
```

Bun ランタイムでは `bun:bundle` モジュールの `feature()` 関数はコンパイル時に評価され、結果に基づいてデッドコード除去が行われる。Node.js 環境ではこのモジュールが存在しないため、常に `false` を返すスタブを提供する。

`tsconfig.json` のパスマッピング (L19-22):
```json
{
  "paths": {
    "bun:bundle": ["stubs/bun-bundle.ts"],
    "src/*": ["src/*"]
  }
}
```

これにより、TypeScript の型チェック時に `bun:bundle` の参照が `stubs/bun-bundle.ts` に解決される。

---

## TypeScript 設定

**ファイル**: `tsconfig.json`

| 設定 | 値 | 説明 |
|------|-----|------|
| `target` | `ES2022` | 出力ターゲット |
| `module` | `ESNext` | ESM モジュール |
| `moduleResolution` | `bundler` | バンドラー向けモジュール解決 |
| `jsx` | `react-jsx` | React JSX 変換 (自動インポート) |
| `strict` | `false` | Strict モード無効 |
| `noEmit` | `false` | 宣言ファイル生成有効 |
| `declaration` | `true` | .d.ts 生成 |
| `sourceMap` | `true` | ソースマップ生成 |
| `lib` | `["ES2022", "DOM"]` | 型定義ライブラリ |

`include` 対象: `src/**/*`, `stubs/**/*`

---

## 依存関係の概要

**ファイル**: `package.json`

### devDependencies
| パッケージ | バージョン | 用途 |
|-----------|-----------|------|
| `esbuild` | `^0.27.4` | JavaScriptバンドラー |
| `typescript` | `^6.0.2` | TypeScript コンパイラ |

### npm scripts
| スクリプト | コマンド | 説明 |
|-----------|---------|------|
| `prepare-src` | `node scripts/prepare-src.mjs` | ソース前処理 |
| `build` | `npm run prepare-src && node scripts/build.mjs` | フルビルド |
| `check` | `npm run prepare-src && tsc --noEmit` | 型チェック |
| `start` | `node dist/cli.js` | ビルド済みバイナリ実行 |

### 主な実行時依存関係（ソースコードのインポートから推定）

| パッケージ | 用途 |
|-----------|------|
| `@anthropic-ai/sdk` | Anthropic API クライアント |
| `@commander-js/extra-typings` | CLI引数パーサー |
| `@growthbook/growthbook` | フィーチャーフラグ |
| `react` / `ink` | ターミナルUI |
| `chalk` | ターミナル色付け |
| `lodash-es` | ユーティリティ |
| `strip-ansi` | ANSIエスケープ除去 |

---

## Bun コンパイル時機能

本番ビルドでは Bun ランタイムの以下のコンパイル時機能が使用される:

### `feature()` — コンパイル時フィーチャーゲート
`bun:bundle` モジュールからインポートされる。ビルド時に `true`/`false` に評価され、バンドラーがデッドコード除去を行う。詳細は [feature-gates.md](./feature-gates.md) を参照。

### `MACRO` — コンパイル時定数
Bun の `--define` オプションで注入されるグローバル定数。`MACRO.VERSION`, `MACRO.BUILD_TIME` 等。esbuild ビルドではリテラル値に直接置換される。

### ビルド出力
- **本番 (Bun)**: 単一実行可能バイナリ（Bun compile）
- **研究用 (esbuild)**: `dist/cli.js` (~12MB, ESM 形式, ソースマップ付き)
