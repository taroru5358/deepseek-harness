# Office から PDF へ

日本語 | [English](../../../docs/subsystems/office-to-pdf.md) | [中文](../../../docs/subsystems/office-to-pdf.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/office-to-pdf.md](../../../docs/subsystems/office-to-pdf.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

[ドキュメント・パッケージ・ファミリー](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/document/README.md)は、Node Host 上で Office ファイルを PDF に変換する。コンシューマーはソースの読み取りを認可し、プレゼンテーションを所有する。共有プロバイダーは変換、境界付きのアドミッション、一時的な PDF の再利用を所有する。このサブシステムはモデル向けのツールも Session イベントも作成しない。

## 所有権

| 所有者 | 責務 |
|---|---|
| [office-to-pdf](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/document/office-to-pdf/README.md) | `ctx.officeToPdf`：共有の LibreOffice 変換、境界付きのアドミッション、PDF キャッシュ |
| [Web バンドル](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/web-app/README.md) | Host のコンシューマーが共有する 1 つの設定可能な変換プロバイダー |
| [Office プレビュー Client](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-sidebar-documentpreview/README.md#office-preview) | Office 拡張子の選択、PDF の再利用、欠落フォントの通知 |

## 要求と結果

[`OfficeToPdfRequest`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/document/office-to-pdf/src/types.ts)は、すでに認可されたソース・キー／バージョン、任意の stat サイズ、遅延される `read(signal, maxBytes)` コールバック、フォアグラウンド／バックグラウンドの優先度、および `OfficeExtension`（`doc`、`docx`、`xls`、`xlsx`、`ppt`、`pptx`）を含む。`OfficeToPdf.convert(request, signal?)` は 1 つの完全な PDF 結果を返す。キャンセルは呼び出し元とプロバイダーの寿命に従い、検証、出力、エンジンの失敗は分類された `OfficeToPdfError` で拒否される。

`OfficeToPdfPriority` は、要求されたプレビュー／QA では `foreground`、投機では `background` である。`OfficeSourceKey` は呼び出し元が所有する認可済みソース・ロケーターをブランド化する。`OfficeToPdfGeneration` はプロバイダーの寿命を、`OfficeToPdfKey` はそのコンテンツ・アイデンティティをブランド化する。どちらの不透明な値もコンシューマーによってパースされない。

| 結果フィールド | 意味 |
|---|---|
| `pdf` | 完全な PDF を含む、呼び出し元所有の `Uint8Array` |
| `missingFonts` | 要求されたドキュメントのフォント・ファミリーのうち、この変換で利用できないもの |
| `cacheKey` | 不透明なコンバーター世代と拡張子／ソース・コンテンツ・アイデンティティ |
| `generation` | プロバイダーの寿命。置き換えはキャッシュされた PDF の再利用を無効化する |

プロバイダーはソース・バイトを割り当てる前に遅延読み取りをアドミットし、コンテンツ・アイデンティティで変換を共有し、返す前に自身のプライベートなスクラッチ・ディレクトリを削除する。返された PDF バイトはプロバイダーの破棄後も有効である。ソース・バイトと PDF バイトは Session ストレージに入らない。コンシューマーは認可された境界付き読み取りに [Workspace Files](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/workspace-files/README.md)を使える。

## プレビューの読み取り

`RenderedDocumentBytes` はワークスペース・ファイルのメタデータ、ネイティブ PDF の `data`、`missingFonts`、`generation` を運ぶ。元のソース・アイデンティティが変換された PDF に伴う。

`officeToPdf.render` Remote メソッドは、Session の [Workspace Files](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/workspace-files/README.md)サービスを通じてソースの認可とバージョンを確認する。変換のアドミッション後、`fs.readBytes` が予約されたバイト容量の範囲で生の入力を供給し、Office の入力制限がこの読み取りを統治する。バイナリ Remote は PDF をマルチパート添付に投影し、Client 上で `ArrayBuffer` に支えられた `Uint8Array` を復元する。ソース・アクセスの失敗はそのまま透過し、サイズとエンジンの失敗は診断なしで分類された理由を公開する。変換は Agent をアクティブ化せず、イベントを追加しない。

`api/remotes` アセンブリは変換サービスの生成された Remote 記述子をマウントする。共有の Document Preview パッケージは Office 形式を完全なバイトのロードと既存の PDF.js Worker で登録する。各プレビュー読み取りは、実行中の変換やキャッシュされた PDF を共有する前に、レンダラーの世代、ソースの認可、バージョンを再確認する。接続のリセットとプラグインの破棄は要求をキャンセルし、キャッシュされたバイトをクリアする。サービスが欠けている場合はローカライズされた設定ガイダンスを表示する。

## エンジンの選択と制限

外部の [`@deepseek-ai/libreoffice-kit`](https://github.com/deepseek-ai/dsh-libreoffice-kit) Node API が、プリコンパイルされたエンジンを選択する。キットは独立したバージョンとリリース・ワークフローを持ち、[リリース所有権の決定](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-09-14-independent-libreoffice-kit.md)で定義される。アプリケーションのビルドは公開 npm パッケージをインストールする。アプリケーションのパッケージングには、ターゲットが宣言するネイティブ・エンジン、またはキットがそのターゲットにネイティブ・エンジンを宣言しない場合は Node WASM が必要である。[プラットフォーム・エンジン・リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/document/office-to-pdf/README.md)がインストールとパッケージングを定義する。無効なメタデータ、必須アセットの欠落、変換エラーはエンジンを切り替えずに拒否される。変換は Host 上のディスク入出力パスを使い、ブラウザー側の変換エンジンやフォント RPC を持たない。

[Host プロバイダー設定](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/document/office-to-pdf/README.md#use-this-package)は、並行性、デッドライン、入出力の制限、アーカイブの制限、画像解像度、フォント・アクセスを所有する。ネイティブ／WASM の実装とアセット配布はキット・ワークスペースに属する。システム LibreOffice の検出、ランタイムのエンジン・ダウンロード、永続的な PDF キャッシュ、モデル向けのレンダリングはこのプロバイダーの外である。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxofficetopdf--officetopdf"></a>

### `ctx.officeToPdf` — `OfficeToPdf`

1 つのプロバイダー寿命が、すべてのコンバーター、キューに入った呼び出し、一時ファイルを所有する。

```ts cordis-catalog
/**
 * Convert Office bytes without modifying the source or writing Session events.
 * @param request - authorized metadata and deferred bounded source read.
 * @param signal - caller cancellation; provider disposal also stops active work.
 * @returns caller-owned PDF bytes after conversion and scratch cleanup settle; canceled readers reject independently.
 * @throws {OfficeToPdfError} Invalid input, unusable output, or engine failure; cancellation rejects with its reason.
 */
convert(request: OfficeToPdfRequest, signal?: AbortSignal): Promise<OfficeToPdfResult>

/**
 * Read and convert one Office file using the Session's ordinary filesystem authorization.
 * @param workspaceFileScope - Session header lookup shared with workspaceFiles.
 * @param path - absolute or workspace-relative Office path.
 * @param priority - foreground preview or speculative background work.
 * @param signal - Remote cancellation; disposal also cancels outstanding reads and conversions.
 * @returns complete PDF bytes with original source identity and missing font families.
 */
@Remote async render( workspaceFileScope: WorkspaceFileScope, path: string, priority: OfficeToPdfPriority, signal: AbortSignal, ): Promise<RenderedDocumentBytes>

/**
 * Read the current rendering generation before reusing a Client PDF.
 * @param signal - Remote caller cancellation.
 * @returns provider lifetime, replaced with rendering, font, or engine configuration.
 */
@Remote('generation') getGeneration(signal: AbortSignal): OfficeToPdfGeneration
```

ソース: [`packages/document/office-to-pdf/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/document/office-to-pdf/src/index.ts)
<!-- END GENERATED cordis-surface -->
