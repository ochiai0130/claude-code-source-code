# Inkコンポーネントアーキテクチャ

Claude Code v2.1.88 は **Ink** (React Terminal UI) のフォーク版をベースに、144以上のアプリケーションコンポーネントとカスタムInk基盤コンポーネントで構成されたターミナルUIを実現している。

---

## 1. Inkフレームワーク統合

**ディレクトリ:** `src/ink/`

Claude Codeは[Ink](https://github.com/vadimdemedes/ink)をフォーク・大幅拡張したカスタム版を内包している。50以上のモジュールで構成される。

### 1.1 コアモジュール

| ファイル | 役割 |
|---------|------|
| `src/ink/ink.tsx` | メインのInkクラス。React reconcilerの初期化、フレームレンダリングループ、入出力管理 |
| `src/ink/reconciler.ts` | `react-reconciler`ベースのカスタムレコンサイラー。DOM操作（`createNode`, `appendChildNode`等）を仮想DOMに変換 |
| `src/ink/renderer.ts` | 仮想DOMから画面出力への変換 |
| `src/ink/dom.ts` | カスタムDOMノード実装（`DOMElement`, `TextNode`） |
| `src/ink/output.ts` | 出力バッファリング |
| `src/ink/screen.ts` | セル単位の画面バッファ（`CellWidth`, `CharPool`, `StylePool`, `HyperlinkPool`） |
| `src/ink/optimizer.ts` | 出力の最適化 |
| `src/ink/terminal.ts` | ターミナル制御（ディフベースの出力、kittyプロトコル検出） |

### 1.2 レイアウトシステム

| ファイル | 役割 |
|---------|------|
| `src/ink/layout/` | Yogaレイアウトエンジン統合 |
| `src/ink/styles.ts` | Box/Textスタイル定義（`Styles`, `TextStyles`型） |
| `src/ink/render-border.ts` | ボーダーレンダリング |
| `src/ink/render-node-to-output.ts` | ノード→出力変換、スクロール追従 |
| `src/ink/render-to-screen.ts` | 出力→画面バッファ変換、位置ハイライト |
| `src/ink/measure-text.ts` / `src/ink/measure-element.ts` | テキスト/要素のサイズ測定 |
| `src/ink/wrap-text.ts` / `src/ink/wrapAnsi.ts` | テキストラッピング |
| `src/ink/stringWidth.ts` / `src/ink/widest-line.ts` | 文字幅計算（CJK対応） |

### 1.3 入力システム

| ファイル | 役割 |
|---------|------|
| `src/ink/parse-keypress.ts` | キー入力のパース |
| `src/ink/events/` | イベントシステム（`keyboard-event.ts`等） |
| `src/ink/focus.ts` | フォーカスマネージャー |
| `src/ink/selection.ts` | テキスト選択（マウス対応、URL検出、範囲選択） |
| `src/ink/hit-test.ts` | マウスクリック/ホバーのヒットテスト |

### 1.4 ターミナルIO

| ファイル | 役割 |
|---------|------|
| `src/ink/termio/` | ターミナル制御シーケンス |
| `src/ink/termio/csi.ts` | CSI (Control Sequence Introducer) — カーソル移動、画面消去 |
| `src/ink/termio/dec.ts` | DEC private modes — マウストラッキング、alt-screen |
| `src/ink/termio/osc.ts` | OSC (Operating System Command) — iTerm2プログレス、クリップボード、タブステータス |
| `src/ink/terminal-querier.ts` | ターミナル機能問い合わせ |
| `src/ink/terminal-focus-state.ts` | ターミナルフォーカス状態 |

---

## 2. Ink基盤コンポーネント

**ディレクトリ:** `src/ink/components/`

18個のコアUIプリミティブを提供:

| コンポーネント | 説明 |
|--------------|------|
| `Box.tsx` | Flexboxレイアウトコンテナ（Yogaベース） |
| `Text.tsx` | テキスト表示（色、太字、斜体、下線、取消線等） |
| `App.tsx` | Ink内部のルートアプリラッパー（`CursorDeclarationContext`等を提供） |
| `ScrollBox.tsx` | スクロール可能なコンテナ（仮想スクロール対応） |
| `Button.tsx` | ボタンコンポーネント |
| `Link.tsx` | ハイパーリンク（OSCハイパーリンクプロトコル対応） |
| `Spacer.tsx` | フレックスspacer |
| `Newline.tsx` | 改行 |
| `RawAnsi.tsx` | 生のANSIシーケンスを直接出力 |
| `NoSelect.tsx` | テキスト選択を無効化するラッパー |
| `AlternateScreen.tsx` | alt-screen切り替え |
| `ErrorOverview.tsx` | エラー表示 |
| `AppContext.ts` | アプリケーションコンテキスト |
| `ClockContext.tsx` | 時計/タイマーコンテキスト |
| `StdinContext.ts` | 標準入力コンテキスト |
| `TerminalSizeContext.tsx` | ターミナルサイズコンテキスト |
| `TerminalFocusContext.tsx` | ターミナルフォーカスコンテキスト |
| `CursorDeclarationContext.ts` | カーソル宣言コンテキスト |

---

## 3. アプリケーションコンポーネント（144+）

**ディレクトリ:** `src/components/`

### 3.1 分類一覧

#### 画面・レイアウト

| コンポーネント | 説明 |
|--------------|------|
| `App.tsx` | トップレベルプロバイダー統合（`FpsMetricsProvider` → `StatsProvider` → `AppStateProvider`） |
| `FullscreenLayout.tsx` | フルスクリーンレイアウト（トランスクリプト + モーダルスロット） |
| `Messages.tsx` | メッセージリスト表示 |
| `VirtualMessageList.tsx` | 仮想スクロール付きメッセージリスト |
| `OffscreenFreeze.tsx` | 画面外コンテンツのフリーズ（レンダリング最適化） |

#### メッセージ表示

| コンポーネント | 説明 |
|--------------|------|
| `Message.tsx` | 単一メッセージの表示 |
| `MessageRow.tsx` | メッセージ行レイアウト |
| `MessageResponse.tsx` | アシスタント応答の表示 |
| `MessageModel.tsx` | モデル名表示 |
| `MessageTimestamp.tsx` | タイムスタンプ表示 |
| `MessageSelector.tsx` | メッセージ選択（rewind用） |
| `CompactSummary.tsx` | 要約済み会話の表示（`BLACK_CIRCLE`アイコン + メタデータ） |
| `messages/` | メッセージ種別ごとのサブコンポーネント群 |

#### ツール表示

| コンポーネント | 説明 |
|--------------|------|
| `ToolUseLoader.tsx` | ツール使用中のローダー表示 |
| `FileEditToolDiff.tsx` | ファイル編集の差分表示 |
| `FileEditToolUpdatedMessage.tsx` | ファイル更新完了メッセージ |
| `FallbackToolUseErrorMessage.tsx` | ツールエラーのフォールバック表示 |
| `FallbackToolUseRejectedMessage.tsx` | ツール拒否のフォールバック表示 |
| `FileEditToolUseRejectedMessage.tsx` | ファイル編集拒否メッセージ |
| `NotebookEditToolUseRejectedMessage.tsx` | ノートブック編集拒否メッセージ |
| `StructuredDiff.tsx` / `StructuredDiffList.tsx` | 構造化差分表示 |
| `diff/` | 差分関連コンポーネント群 |

#### 権限・承認ダイアログ

| コンポーネント | 説明 |
|--------------|------|
| `permissions/` | 権限要求UI群 |
| `TrustDialog/` | 信頼設定ダイアログ |
| `ApproveApiKey.tsx` | APIキー承認 |
| `BypassPermissionsModeDialog.tsx` | バイパスモード確認 |
| `AutoModeOptInDialog.tsx` | 自動モードオプトイン |
| `ManagedSettingsSecurityDialog/` | 管理設定セキュリティ |

#### 設定・ピッカー

| コンポーネント | 説明 |
|--------------|------|
| `Settings/` | 設定画面群 |
| `ModelPicker.tsx` | モデル選択 |
| `ThemePicker.tsx` | テーマ選択 |
| `OutputStylePicker.tsx` | 出力スタイル選択 |
| `LanguagePicker.tsx` | 言語選択 |
| `LogSelector.tsx` | ログ選択 |

#### MCPサーバー管理

| コンポーネント | 説明 |
|--------------|------|
| `mcp/` | MCP関連コンポーネント群 |
| `MCPServerApprovalDialog.tsx` | MCPサーバー承認 |
| `MCPServerDesktopImportDialog.tsx` | デスクトップインポート |
| `MCPServerDialogCopy.tsx` | MCPダイアログコピー |
| `MCPServerMultiselectDialog.tsx` | MCPサーバー複数選択 |

#### プログレス・ステータス

| コンポーネント | 説明 |
|--------------|------|
| `Spinner.tsx` / `Spinner/` | スピナー表示（`SpinnerWithVerb`, `BriefIdleStatus`） |
| `AgentProgressLine.tsx` | エージェント進行状況 |
| `BashModeProgress.tsx` | Bashモード進行状況 |
| `StatusLine.tsx` | ステータスライン |
| `StatusNotices.tsx` | ステータス通知 |
| `CoordinatorAgentStatus.tsx` | コーディネーターステータス |
| `MemoryUsageIndicator.tsx` | メモリ使用量表示 |

#### 検索・ナビゲーション

| コンポーネント | 説明 |
|--------------|------|
| `GlobalSearchDialog.tsx` | グローバル検索（`ctrl+shift+f`） |
| `QuickOpenDialog.tsx` | クイックオープン（`ctrl+shift+p`） |
| `HistorySearchDialog.tsx` | 履歴検索（`ctrl+r`） |
| `SearchBox.tsx` | 検索ボックス |

#### 入力

| コンポーネント | 説明 |
|--------------|------|
| `PromptInput/` | プロンプト入力コンポーネント群 |
| `TextInput.tsx` | テキスト入力 |
| `VimTextInput.tsx` | Vimモード付きテキスト入力 |
| `BaseTextInput.tsx` | 基盤テキスト入力 |
| `ContextSuggestions.tsx` | コンテキストサジェスト |
| `ContextVisualization.tsx` | コンテキスト表示 |

#### デスクトップ統合

| コンポーネント | 説明 |
|--------------|------|
| `DesktopHandoff.tsx` | デスクトップ連携 |
| `DesktopUpsell/` | デスクトップアップセル |
| `IdeAutoConnectDialog.tsx` | IDE自動接続 |
| `IdeOnboardingDialog.tsx` | IDEオンボーディング |
| `IdeStatusIndicator.tsx` | IDE接続状態 |

#### テレポート・リモート

| コンポーネント | 説明 |
|--------------|------|
| `TeleportProgress.tsx` | テレポート進行 |
| `TeleportError.tsx` | テレポートエラー |
| `TeleportRepoMismatchDialog.tsx` | リポジトリ不一致 |
| `TeleportResumeWrapper.tsx` | テレポート再開 |
| `RemoteCallout.tsx` | リモート通知 |
| `RemoteEnvironmentDialog.tsx` | リモート環境 |

#### チーム・エージェント

| コンポーネント | 説明 |
|--------------|------|
| `agents/` | エージェント関連 |
| `teams/` | チーム関連 |
| `tasks/` | タスク管理 |
| `TeammateViewHeader.tsx` | チームメイトビューヘッダー |
| `TaskListV2.tsx` | タスクリストv2 |

#### UI部品・デザインシステム

| コンポーネント | 説明 |
|--------------|------|
| `design-system/` | デザインシステムプリミティブ |
| `CustomSelect/` | カスタムセレクト |
| `TagTabs.tsx` | タブ表示 |
| `ConfigurableShortcutHint.tsx` | カスタマイズ可能なショートカットヒント |
| `FastIcon.tsx` | 高速アイコン表示 |
| `FilePathLink.tsx` | ファイルパスリンク |
| `ClickableImageRef.tsx` | クリック可能な画像参照 |
| `PressEnterToContinue.tsx` | Enter続行プロンプト |

#### Markdown・コード表示

| コンポーネント | 説明 |
|--------------|------|
| `Markdown.tsx` | Markdownレンダリング |
| `MarkdownTable.tsx` | テーブル表示 |
| `HighlightedCode/` / `HighlightedCode.tsx` | シンタックスハイライト |

#### 更新・診断

| コンポーネント | 説明 |
|--------------|------|
| `AutoUpdater.tsx` / `AutoUpdaterWrapper.tsx` | 自動更新 |
| `NativeAutoUpdater.tsx` | ネイティブ自動更新 |
| `PackageManagerAutoUpdater.tsx` | パッケージマネージャー経由の更新 |
| `DiagnosticsDisplay.tsx` | 診断表示 |
| `Stats.tsx` | 統計表示 |
| `DevBar.tsx` | 開発者バー（ANT-ONLY: 遅い同期操作の警告表示、500ms間隔で更新） |

---

## 4. 画面構成

**ディレクトリ:** `src/screens/`

3つの画面コンポーネントが存在:

| 画面 | ファイル | 説明 |
|------|---------|------|
| REPL | `src/screens/REPL.tsx` | メインの対話画面。80以上のインポート、最大規模のコンポーネント |
| Doctor | `src/screens/Doctor.tsx` | 診断画面 |
| ResumeConversation | `src/screens/ResumeConversation.tsx` | 会話再開画面 |

### REPL画面の構成要素

`REPL.tsx`は以下を統合する:
- メッセージリスト（仮想スクロール対応）
- プロンプト入力（Vim対応）
- 権限ダイアログ
- コスト閾値・アイドル復帰ダイアログ
- スピナー表示
- MCP Elicitation
- セッションフック通知
- スウォーム/チームメイト管理
- 検索UI（`ctrl+r`履歴検索、`/`トランスクリプト検索）

---

## 5. ターミナルレンダリングの仕組み

### 5.1 フレームレンダリングパイプライン

`src/ink/ink.tsx`の`Ink`クラスが中心:

```
React State変更
  → Reconciler (react-reconciler) がDOMノード更新
  → Yoga Layout でレイアウト計算
  → renderNodeToOutput() でDOMノードを出力バッファに変換
  → optimize() で出力最適化
  → Screen (セルバッファ) に書き込み
  → writeDiffToTerminal() で前フレームとの差分のみ出力
```

**フレーム間隔:** `FRAME_INTERVAL_MS`（`src/ink/constants.ts`で定義）でスロットリング

### 5.2 差分レンダリング

`writeDiffToTerminal()`は前フレームと現フレームのセルバッファを比較し、変更されたセルのみターミナルに出力する。これにより:
- ちらつき防止
- 帯域幅削減（SSH等で重要）
- 高いフレームレート維持

### 5.3 Alt-Screen サポート

フルスクリーンモードではalt-screen（DEC private mode 47/1047）を使用:
- カーソルは常に非表示（`ALT_SCREEN_ANCHOR_CURSOR`）
- 画面全体を使用
- 終了時に元の画面を復元

### 5.4 選択・マウスサポート

`src/ink/selection.ts`で実装:
- マウスでのテキスト選択
- ダブルクリックで単語選択
- URL検出（`findPlainTextUrlAt`）
- スクロール追従（`shiftSelectionForFollow`）

---

## 6. ANSIカラーサポート

**ファイル:** `src/ink/colorize.ts`

### 6.1 チョークレベルの自動調整

3段階の適応ロジック:

1. **VS Code/xterm.js検出** (`boostChalkLevelForXtermJs`):
   - `TERM_PROGRAM=vscode` + chalk level 2 → level 3にブースト
   - code-server等でCOLORTERMが未設定の環境に対応

2. **tmux検出**:
   - tmux内ではlevel 2にクランプ（truecolor → 256色）
   - tmuxの`terminal-overrides`設定に依存しない安定動作

3. **最低レベル保証**:
   - `NO_COLOR` / `FORCE_COLOR=0` は尊重（level 0）

### 6.2 テーマシステム

`Text`コンポーネントの`color`プロパティは`Theme`キー名を受け付ける:
```tsx
<Text color="warning">警告メッセージ</Text>
<Text color="text">通常テキスト</Text>
```

テーマはRGB値で定義され、chalk levelに応じて自動的にダウングレードされる。

### 6.3 Inkスタイル属性

`src/ink/styles.ts`で定義されるスタイル:

```typescript
type TextStyles = {
  color?: Color        // 前景色（名前/hex/rgb）
  backgroundColor?: Color  // 背景色
  bold?: boolean
  italic?: boolean
  underline?: boolean
  strikethrough?: boolean
  inverse?: boolean
  dimColor?: boolean
  wrap?: 'wrap' | 'truncate' | 'truncate-end'
}
```

---

## 7. 主要コンポーネントの解説

### 7.1 App (`src/components/App.tsx`)

最上位のプロバイダーラッパー。3つのコンテキストをネスト:

```tsx
<FpsMetricsProvider getFpsMetrics={getFpsMetrics}>
  <StatsProvider store={stats}>
    <AppStateProvider initialState={initialState} onChangeAppState={onChangeAppState}>
      {children}
    </AppStateProvider>
  </StatsProvider>
</FpsMetricsProvider>
```

### 7.2 CompactSummary (`src/components/CompactSummary.tsx`)

要約済み会話の表示。`summarizeMetadata`の有無で分岐:
- メタデータあり: `BLACK_CIRCLE`アイコン + 「Summarized conversation」+ 要約統計
- トランスクリプトモード: 元のテキストをそのまま表示
- `ConfigurableShortcutHint`でカスタマイズ可能なショートカットヒントを表示

### 7.3 DevBar (`src/components/DevBar.tsx`)

開発者専用の遅延操作モニター:
- `production` !== `development` かつ `external` !== `ant` の場合は非表示
- 500msポーリングで`getSlowOperations()`を取得
- 最新3件を`operation (Xms)`形式で表示
- `<Text wrap="truncate-end" color="warning">`で一行に収まるよう切り詰め

### 7.4 React Compilerによる最適化

多くのコンポーネントにReact Compilerの出力が見られる:
```typescript
import { c as _c } from "react/compiler-runtime";
const $ = _c(9);  // メモ化キャッシュスロット
```

手動の`useMemo`/`useCallback`なしに、コンパイラが自動的にメモ化を適用。`$[n]`パターンでキャッシュスロットを管理し、依存値が変わった場合のみ再計算する。
