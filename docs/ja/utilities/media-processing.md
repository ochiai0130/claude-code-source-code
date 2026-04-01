# メディア処理

Claude Code v2.1.88 におけるファイル添付処理、画像リサイズ、ANSI→PNG 変換、画像バリデーションの技術解析。

## 概要

Claude Code はマルチモーダル入力をサポートしており、テキスト、画像、ファイルなどの多様なコンテンツタイプを処理する。

| モジュール | ファイルパス | 役割 |
|---|---|---|
| attachments.ts | `src/utils/attachments.ts` (~127KB) | 添付ファイルの統合処理 |
| ansiToPng.ts | `src/utils/ansiToPng.ts` (~214KB) | ANSI エスケープ → PNG 変換 |
| imageResizer.ts | `src/utils/imageResizer.ts` | 画像リサイズ・圧縮 |
| imageValidation.ts | `src/utils/imageValidation.ts` | API 送信前の画像サイズ検証 |

---

## 1. ファイル/画像添付処理 (`attachments.ts`)

### 概要

`attachments.ts` はシステム内で最も大きなユーティリティファイルの一つ (約127KB) であり、ユーザーメッセージに添付されるすべてのコンテキスト情報の生成を担当する。

### 依存モジュール

このファイルは非常に多くのモジュールに依存しており、Claude Code のメッセージング基盤の中核を成す。主要な依存先は以下の通り:

**ツール関連** (`src/utils/attachments.ts` L7-30):
- `FileReadTool` -- ファイル読み取り (`MaxFileReadTokenExceededError` を含む)
- `readImageWithTokenBudget` -- トークン予算内での画像読み取り
- `FileTooLargeError` / `readFileInRange` -- ファイル範囲読み取り
- `TODO_WRITE_TOOL_NAME`, `TASK_CREATE_TOOL_NAME`, `TASK_UPDATE_TOOL_NAME` -- タスク管理ツール
- `BASH_TOOL_NAME` -- Bash ツール
- `SKILL_TOOL_NAME` -- スキルツール
- `AGENT_TOOL_NAME` -- エージェントツール

**コンテキスト関連** (L38-46):
- `getMemoryFiles`, `getManagedAndUserConditionalRules` -- CLAUDE.md メモリファイル
- `filterInjectedMemoryFiles` -- メモリファイルのフィルタリング
- `getConditionalRulesForCwdLevelDirectory` -- ディレクトリレベルの条件付きルール

**画像処理** (L73):
- `maybeResizeAndDownsampleImageBlock` -- 画像リサイズ処理への委譲

### 添付ファイルの種類

ファイルは以下のようなコンテキスト情報を添付メッセージとして生成する:

1. **ファイル読み取り結果** -- `FileReadTool` 経由
2. **画像ペースト** -- `isValidImagePaste()`, `getImagePasteIds()` (L63)
3. **IDE セレクション** -- `IDESelection` 型 (L24)
4. **MCP リソース** -- `ReadResourceResult` 型 (L81)
5. **診断ファイル** -- `DiagnosticFile` 型 (L53)
6. **タスク出力** -- `generateTaskAttachments()`, `applyTaskOffsetsAndEvictions()` (L137-138)
7. **非同期フック応答** -- `checkForAsyncHookResponses()` (L183)
8. **LSP 診断** -- `checkForLSPDiagnostics()` (L188)
9. **ツール検索デルタ** -- `getDeferredToolsDelta()` (L163)
10. **MCP 命令デルタ** -- `getMcpInstructionsDelta()` (L171)

### 画像メッセージの型

`@anthropic-ai/sdk` から以下の型をインポート (L69-72):

```typescript
import type {
  ContentBlockParam,
  ImageBlockParam,
  Base64ImageSource,
} from '@anthropic-ai/sdk/resources/messages.mjs'
```

### フィーチャーフラグによる条件付き読み込み

パフォーマンスとバンドルサイズの最適化のため、一部のモジュールは条件付きで `require` される (L95-106):

- `EXPERIMENTAL_SKILL_SEARCH` -- スキル検索機能
- `TRANSCRIPT_CLASSIFIER` -- トランスクリプト分類器 (自動モード状態)

### ブートストラップ状態との統合

`src/bootstrap/state.js` から多数の状態関数をインポート (L143-160):

- `getTotalCostUSD()` -- 累計コスト
- `getTotalOutputTokens()` -- 累計出力トークン数
- `getCurrentTurnTokenBudget()` -- 現在のターンのトークン予算
- `getTurnOutputTokens()` -- 現在ターンの出力トークン数
- `hasExitedPlanModeInSession()` -- プランモード退出フラグ
- `getKairosActive()` -- Kairos アクティブ状態

---

## 2. ANSI ターミナル出力の PNG 変換 (`ansiToPng.ts`)

### 概要

`ansiToPng.ts` はANSI エスケープシーケンスを含むターミナルテキストを直接 PNG 画像にレンダリングする。以前の SVG 中間形式 (`ansiToSvg` → `@resvg/resvg-wasm`) パイプラインを置き換えたもの。

**設計思想** (`src/utils/ansiToPng.ts` L1-4):
> Render ANSI-escaped terminal text directly to a PNG image. Replaces the previous ansiToSvg → @resvg/resvg-wasm pipeline. The SVG was just a lossy intermediate format for what is fundamentally a grid of [colored characters].

### AnsiToPngOptions

`src/utils/ansiToPng.ts` L74-85 で定義:

| オプション | 型 | デフォルト | 説明 |
|---|---|---|---|
| `scale` | `number` | 1 | 整数ズーム倍率 (最近傍補間) |
| `paddingX` | `number` | 48 | 水平パディング (1x ピクセル) |
| `paddingY` | `number` | 48 | 垂直パディング (1x ピクセル) |
| `borderRadius` | `number` | 16 | 角丸半径 (1x ピクセル) |
| `background` | `AnsiColor` | ダークグレー | 背景色 |

### レンダリングパイプライン

`ansiToPng()` 関数 (L91-149) のレンダリングフロー:

1. **ANSI パース**: `parseAnsi(ansiText)` でテキストをスパン (テキスト + 色 + 太字フラグ) の行に分解
2. **末尾空白行のトリム**: 末尾の空行を除去 (L104-110)
3. **キャンバスサイズ計算**: グリフサイズ (`GLYPH_W`, `GLYPH_H`) とパディングからピクセルサイズを計算 (L115-119)
4. **RGBA バッファの初期化**: `Uint8Array` で背景色を塗りつぶし (L122-123)
5. **角丸処理**: `roundCorners()` で角丸マスクを適用 (L124-126)
6. **グリフのブリット**: 各文字をビットマップフォントでレンダリング (L131-149)
   - `SHADE_ALPHA` -- シェード文字 (ボックスドローイング等) はアルファブレンド
   - `FONT` マップ -- 組み込みビットマップフォントからグリフを取得
   - `FALLBACK_GLYPH` -- 未登録文字のフォールバック
   - ゼロ幅文字 (結合マーク等) はスキップ

### 出力形式

生の RGBA ピクセルデータから有効な PNG (RGBA, 8ビット) を生成して `Buffer` として返す。外部ライブラリ (sharp, canvas 等) に依存せず、完全に自己完結した実装。

---

## 3. 画像リサイズとバリデーション

### 画像リサイズ (`imageResizer.ts`)

#### API 制約定数

`src/constants/apiLimits.js` から以下の定数をインポート (L6-10):

| 定数 | 説明 |
|---|---|
| `API_IMAGE_MAX_BASE64_SIZE` | Base64 エンコード後の最大サイズ (API制限) |
| `IMAGE_MAX_HEIGHT` | 最大高さ |
| `IMAGE_MAX_WIDTH` | 最大幅 |
| `IMAGE_TARGET_RAW_SIZE` | 圧縮目標サイズ |

#### エラー分類

`classifyImageError()` (`src/utils/imageResizer.ts` L50-124) は画像処理エラーをアナリティクス向けに8カテゴリに分類:

| コード | 定数 | 説明 |
|---|---|---|
| 1 | `ERROR_TYPE_MODULE_LOAD` | モジュール読み込み失敗 |
| 2 | `ERROR_TYPE_PROCESSING` | フォーマット検出、破損データ等 |
| 3 | `ERROR_TYPE_UNKNOWN` | 未分類エラー |
| 4 | `ERROR_TYPE_PIXEL_LIMIT` | ピクセル/寸法制限超過 |
| 5 | `ERROR_TYPE_MEMORY` | メモリ割り当て失敗 |
| 6 | `ERROR_TYPE_TIMEOUT` | タイムアウト |
| 7 | `ERROR_TYPE_VIPS` | VIPS ライブラリ固有エラー |
| 8 | `ERROR_TYPE_PERMISSION` | アクセス権限エラー |

エラーコードの判定は Node.js の `error.code` を優先し、sharp ライブラリのようにエラーコードを公開しないケースではメッセージ文字列のマッチングにフォールバックする。

#### ImageResizeError

```typescript
class ImageResizeError extends Error {
  name = 'ImageResizeError'
}
```

画像リサイズの失敗かつ API サイズ制限超過時にスローされる (L37-42)。

#### リサイズ処理

`maybeResizeAndDownsampleImageBuffer()` (L169-199):

1. **空バッファチェック**: 0バイトの場合は即座にエラー (API が `image cannot be empty` を返す問題を防止)
2. **sharp によるメタデータ取得**: フォーマット、幅、高さを取得
3. **メディアタイプの正規化**: `jpg` → `jpeg` への変換
4. **寸法不明時の処理**: メタデータからサイズが取得できない場合、`IMAGE_TARGET_RAW_SIZE` を超えていれば JPEG 80% 品質で圧縮
5. **リサイズ実行**: `sharp` ライブラリで制約内にリサイズ

#### 型定義

```typescript
// L138-143
type ImageDimensions = {
  originalWidth?: number
  originalHeight?: number
  displayWidth?: number
  displayHeight?: number
}

// L145-149
interface ResizeResult {
  buffer: Buffer
  mediaType: string
  dimensions?: ImageDimensions
}
```

#### ハッシュ関数

`hashString()` (L130-136) はアナリティクスグルーピング用の djb2 ハッシュ (32ビット符号なし整数)。

### 画像バリデーション (`imageValidation.ts`)

#### OversizedImage 型

```typescript
// src/utils/imageValidation.ts L8-11
type OversizedImage = {
  index: number  // 画像のインデックス
  size: number   // Base64 サイズ (バイト)
}
```

#### ImageSizeError

`ImageSizeError` (L16-35) は API サイズ制限を超える画像が検出された場合にスローされるカスタムエラー:

- 単一画像の場合: サイズと上限を明示したメッセージ
- 複数画像の場合: 各画像のインデックスとサイズを列挙

#### Base64 画像ブロックの型ガード

`isBase64ImageBlock()` (L40-49) は型安全な画像ブロック判定:

```typescript
{ type: 'image', source: { type: 'base64', data: string } }
```

#### API バリデーション

`validateImagesForAPI()` (L65-99) は API 送信境界でのセーフティネット:

- **対象**: `user` タイプのメッセージのみ
- **制限**: `API_IMAGE_MAX_BASE64_SIZE` を超える Base64 ペイロードを検出
- **注意**: API の 5MB 制限はデコード後の生バイトではなく、Base64 エンコード文字列の長さに適用される
- **メッセージ形式対応**: ラップされた形式 `{ type: 'user', message: { role, content } }` と生の `MessageParam` 形式の両方をサポート
- **アナリティクス**: 検出時に `tengu_image_api_validation_failed` イベントを記録

---

## 4. マルチモーダル入力サポート

### サポートされる画像形式

`imageResizer.ts` で処理可能な画像形式 (L22):

```typescript
type ImageMediaType = 'image/png' | 'image/jpeg' | 'image/gif' | 'image/webp'
```

### 画像処理パイプライン

ユーザーが画像を添付した際の処理フロー:

```
画像入力
  ↓
isValidImagePaste() / getImagePasteIds() で検証
  ↓
readImageWithTokenBudget() でトークン予算内読み取り
  ↓
maybeResizeAndDownsampleImageBlock() でリサイズ
  ↓
validateImagesForAPI() で API 制限チェック
  ↓
Base64ImageSource として API に送信
```

### ANSI 出力の画像化フロー

Bash ツール等のターミナル出力を画像として処理する場合:

```
ANSI エスケープ付きターミナル出力
  ↓
ansiToPng() で PNG バッファに変換
  ↓
ImageBlockParam として API に送信
```

### ペーストされたコンテンツ

`PastedContent` 型 (`src/utils/config.ts`) でクリップボードからのペーストコンテンツを管理。画像ペーストは `getImagePasteIds()` で ID を取得し、添付メッセージとして処理される。

### IDE 統合

`IDESelection` 型 (`src/hooks/useIdeSelection.ts`) で IDE のテキスト選択状態を受け取り、コンテキストとして添付する。`getConnectedIdeName()` (`src/utils/ide.ts`) で接続中の IDE 名を取得する。
