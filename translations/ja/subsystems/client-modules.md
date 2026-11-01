# クライアント・モジュール

日本語 | [English](../../../docs/subsystems/client-modules.md) | [中文](../../../docs/subsystems/client-modules.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/client-modules.md](../../../docs/subsystems/client-modules.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

Web プラグイン・テーブル。[dsh-client-modules](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/client/modules) にあるクライアント・モジュール・システムの Node 側であり、`ctx.clientModules`（`ClientModuleRegistry`）として提供される。host Loader のエントリーから `dsh.client` を宣言するパッケージをスキャンし、`window.__DSH_BOOT__` のエントリー・グラフを合成し、`/plugins` 配下でバージョン付きの 1 つ以上のリソースからなるコンボ・スクリプトを配信し、すべてのインデックス・インジェクション・コレクションに boot プロトコルの行で応答する。1 つのサービスの 4 つの顔である。これは web GUI スタックの任意のケイパビリティであり、agent ループの背骨の一部ではない。また [dsh-host-webserver](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/host/webserver) のコンシューマーでもある。[web-server.md](web-server.md) で説明されるキャリアが、このサービスが応答するプレフィックス・ルートと `webserver/index-inject` イベントを提供する。同じパッケージのブラウザー側（`ctx.modules`、これらの bundle を取得して実体化する遅延 CJS モジュール・テーブル）はカーネル機構であり、ここではなく[パッケージ README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/modules/README.md) で説明される。

ソース: [`packages/client/modules/src/client/manifest.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/modules/src/client/manifest.ts)

## ワイヤー

グラフは、Node 側とブラウザー側の間のワイヤーの単一の源である。host はスキャンしたパッケージから `WebBootEntry` の行と `WebBootBatch` の記述子を合成し、Vite エントリーの前に、登録ファサード、アプリケーション・プリロード、bootstrap スクリプト、グラフ・グローバルを構造化インデックス・インジェクション・テーブルにコントリビュートする。`global` 行は `globalThis["__DSH_BOOT__"]` としてレンダリングされ、`<` はエスケープされるので、プラグインが制御する文字列が script 要素の外へ抜け出すことはできない。有効なマニフェストを持たないページは boot できない。ブラウザーのパーサーは、不正な行やバッチ、未知のメンバー、初期コンボ記述子をちょうど 1 つ持たないエントリーを拒否する。

```ts type-equiv
/**
 * One composed client entry pushed by the host (a graph row). Wire
 * single source: the host node half (package root) produces this same shape.
 * `immediately` marks stage-one prefetch. `inject` names package rows whose
 * factories must arrive before this row materializes, while Cordis separately
 * uses the same package edges to compose entries. `external` carries exact
 * non-inject module requests (see {@link WebBootGraph.entries}).
 */
interface WebBootEntry {
  /** Entry name == package name. */
  id: string
  /**
   * Revisioned single-resource combo reference used by HMR. It is relative to
   * the document, so the browser resolves it under whatever mount served the page.
   */
  url: string
  /** Opaque plugin-artifact revision used for HMR cache busting. */
  rev: string
  /** Package-name dependency edges used for factory arrival and plugin composition. */
  inject?: string[]
  /** Stage-one prefetch mark: load the script for factory registration during module-face boot. */
  immediately?: boolean
  /** Non-baseline module specifiers this row requests; omitted when it requests none. */
  external?: string[]
}
```

```ts type-equiv
/** Initial scheduling phase for one revisioned combo script. */
type WebBootBatchPhase = 'bootstrap' | 'application'
```

```ts type-equiv
/** One initial combo script; a scheduling phase may span several descriptors. */
interface WebBootBatch {
  /** Parser-blocking bootstrap or preloaded application scheduling. */
  phase: WebBootBatchPhase
  /** Content-addressed combo script reference, document-relative like {@link WebBootEntry.url}. */
  url: string
  /** Revision derived from the ordered entry revisions. */
  rev: string
  /** Graph entry ids whose factories the script registers, in execution order. */
  entries: string[]
}
```

```ts type-equiv
/** Host release identity; two releases are the same when both fields are equal. */
interface WebBootRelease {
  /** Release version label. */
  version: string
  /** Source commit fixed in when the Host was built; absent when the build recorded none. */
  commit?: string
}
```

```ts type-equiv
/** The composed client entry graph the host injects as `window.__DSH_BOOT__`. */
interface WebBootGraph {
  /** Consistency anchor over the current entry and batch descriptors. */
  rev: string
  /** Host release identity; absent from a Host that publishes none. */
  release?: WebBootRelease
  /**
   * Composed entries in module-graph order — a dynamic package row precedes
   * rows whose `external` requests that package. Cordis activation order is
   * unrelated and remains owned by fiber service waiting.
   */
  entries: WebBootEntry[]
  /** Initial combo descriptors; every entry belongs to exactly one descriptor. */
  batches: WebBootBatch[]
}
```

初回公開と HMR は、実行可能バイトをハッシュせずに、エントリーの mtime、ctime、サイズから各行の `rev` を導出する。同じアーティファクトは Host の再起動をまたいでリビジョンを保持する。初期記述子は行を bootstrap と application のスケジューリング・フェーズに分割し、どちらのフェーズも複数の記述子を含みうる。それらの URL は順序付きのパッケージ・リソース・リストと、それらの行リビジョンから導出されたリビジョンだけを含む。フェーズ名はルートに入らない。グラフ合成は行順を保ちつつ、map 形式の URL が 3 KiB を超える前に貪欲に分割し、スクリプトの連結や map の読み取りは行わない。グラフ・リビジョンはエントリー記述子とバッチ記述子をハッシュする。`immediately` は第 1 段階の登録バリアを示す。1 つのコンボ内の行はそのスクリプト搬送を共有し、別々のコンボは独立にロードされる。

`release` はこのパッケージのバージョンと、クライアント・ビルドがビルド済み Host アーティファクトに固定するソース・コミットを対にし、グラフ・リビジョンの外側に留まる。ページは、Host がそのページの boot リリースを配信している間だけ、グラフと再ビルド・フレームを適用する。別のリリースはページのリロードを待つ（[Host リリースの変更](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/modules/README.md#host-release-changes)）。

## スキャン

パッケージは、package.json で `dsh.client`（`platform: 'web'`、任意の `inject` エッジ、任意の `immediately`）を宣言し、ビルド済み bundle を `exports["./client"]` でエクスポートすることでテーブルに参加する。各ライブ行は、自身の Loader 指定子と所有ツリーの `baseUrl` から、利用可能な場合はその Host フェイスをインポートするのと同じ `loader.internal.resolveSync` 実装を通じて解決される。最も近い所有パッケージのマニフェストがブラウザー・モジュール id を提供するので、相対ソースとビルド済みオーバーレイはパッケージのアイデンティティを保つ。1 つのパッケージ名に解決される別個のアクティブな Loader ソースは合成に失敗する。1 つのソースがアンロードされた後は、生存するソースが fiber を再起動せずに行を提供する。

パッケージの照合は Loader エントリー名ごとにインクリメンタルである。すべての Cordis `internal/plugin` の発行（fiber の構築または破棄）は、その fiber のエントリー名をダーティにする。空でない各 flush は、すべての Loader エントリーを 1 回列挙し、flush 開始時に名前で索引する。保留中の名前は索引されたエントリー（同じ名前のすべてのソースを含む）を使い、見つからない名前は空のリストを使ってアンロード済みソースを除去する。パッケージ解決は照合中に同期的に Loader エントリーを追加または削除してはならない。アクティベーションは現在のすべての名前をシードして同期的に flush する。以降の通知はマイクロタスクの flush をキューに入れる。

アクティベーション時、不正な宣言、欠落した bundle、flush の列挙失敗は 1 つの合成エラーに集約される。fiber は失敗し、boot がそれを報告する。定常状態では、パッケージの失敗は警告をログに記録するが、他の名前の照合を妨げない。列挙失敗は失敗したバッチを消費し、警告を 1 つログに記録し、パッケージ状態と公開済みグラフを変更しない。影響を受けた各名前は、再試行する前に別の通知を待つ。

各照合はパッケージ・マニフェストと bundle メタデータを再度読む。同じソースからの fiber 再起動が同じ宣言と bundle メタデータを見つけた場合、その行と rev を保持し、bundle のバイトを読まない。宣言、クライアント・エクスポート、bundle のいずれかが変わっていれば、その照合によって公開される。エントリーを再起動せずに再ビルドされた bundle は `rebuilt()` を通じてグラフに到達する。

## バンドル・ルートとインデックス・インジェクション

`GET`/`HEAD /plugins/??<package-a>/client.js,<package-b>/client.js&rev=<rev>` は 1 つの生成されたコンボ・スクリプトを指定する。1 リソースのリクエストも同じ形式を使い、それが HMR 経路である。`/plugins/<package>/assets/<path>` はパッケージの公開済み画像を配信する。そのルートは[パッケージ README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/modules/README.md#package-assets) が所有する。スクリプトは最初の `GET` で 1 回だけ連結され、末尾にコンボ・クエリだけを運ぶ `sourceMappingURL` が付く。すなわち `??<package-a>/client.js.map,<package-b>/client.js.map&rev=<rev>` であり、ドキュメントではなくスクリプト自身のディレクトリーを基準に解決される。map ファイルは、スタートアップ、インデックス・レンダリング、スクリプトの `GET`、`HEAD` では読まれない。最初の map の `GET` がそれらを読んで検証し、1 つの Indexed Source Map v3 を合成して、そのボディをキャッシュする。作成済みのコンポーネント map はそのセクションを提供する。それを持たないコンポーネントは、`sourcesContent` がキャプチャされた bundle であり、ソース名がパッケージ化された `sourceURL` またはプラグイン・ルートである identity セクションを受け取る。すべてのスタートアップ・リクエスト URL は UTF-8 バイトで測定して最大 3 KiB であり、分割にはより長い map 形式を使う。すべてのアプリケーション URL はプリロードされ、すべての bootstrap URL はグラフ・グローバルと Vite エントリーの前に実行される。実体化されたレスポンスは長期間有効なイミュータブル・キャッシュを使う。未知または改変されたリソース・リスト、欠落したリビジョン、陳腐化したリビジョンには、異なるバイトを配信したり SPA フォールバックに HTML を JavaScript として返させたりする代わりに 404 で応答する。他のメソッドは 405 である。インジェクション行はインデックスのレンダリングごとに現在のグラフを運ぶので、リロードは常にライブの合成に対して boot する。

## サービス

```ts type-equiv
/** Filesystem baseline captured before a client artifact snapshot is read. */
interface ClientArtifactBaseline {
  /** Absolute path of the client bundle. */
  readonly path: string
  /** Bundle modification time in milliseconds. */
  readonly mtimeMs: number
  /** Bundle status-change time in milliseconds, including writes that preserve mtime. */
  readonly ctimeMs: number
  /** Bundle size in bytes. */
  readonly size: number
}
```

`ClientModuleRegistry`（`ctx.clientModules`。[`packages/client/modules/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/modules/src/index.ts) で定義）は読み取りと再ビルドのフェイスを公開する。シグネチャは生成された[サービス・カタログ](#ctxclientmodules--clientmoduleregistry) にある。`graph()` は現在の合成済みグラフを返し（変更間では安定したオブジェクト）、`clientPath(id)` は bundle の絶対パスを返し、`artifactBaseline(id)` は現在のスナップショットを読む前にキャプチャした bundle の stat 値を返す。`fetchBundle()` は HTTP ルートと同じレスポンスを返す。遅延 bundle のレスポンスと、リクエストごとに読まれるパッケージ・アセットである。`rebuilt(id)` はエントリーを再起動せずに再ビルドされた bundle を公開する。ファイルシステムのメタデータからリビジョンを導出し、リビジョンが変わったときだけ新しいバイトを読み、グラフを再合成して通知する。エントリーの再起動は代わりに照合を通じて公開する。`onRebuilt` は変更された bundle ごとに新しいリビジョンとともに発火する。`onGraphChanged` はグラフを再合成した flush（行の追加または削除、あるいは再ビルドのリビジョン変更）の後に発火し、プル・モデルである。リスナーは `graph()` を読み直す。どちらの通知経路もリスナーの例外を封じ込めるので、例外を投げる購読者が後続の購読者を飛ばしたり、flush を引き起こした何かを停止させたりすることはない。

[`dsh-client-hmr`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/hmr/README.md) は、出荷される Web 合成でライブのグラフ・スナップショットを配信する。Host は既存のグラフ変更通知を即座に転送し、再接続時には現在の完全なグラフを送る。グラフは望ましいブラウザー・エントリーを記述するもので、Host のクリーンアップが完了したことを主張するものではない。そのアーティファクト・ポーリングは再ビルドされたリビジョンを別途報告する。ソース・マップだけの変更はリロードを引き起こさない。新しいコンボ map の URL は bundle のリビジョンが変わった後にだけ現れ、各 map のボディは最初の `GET` で固定される。クライアント・モジュールはスナップショットを検証し、照合をそれらの再ビルドと直列化する。boot で作成されたエントリー・マップ、単一リソースの到着、非同期の削除、未使用モジュールとスタイルのクリーンアップ、ページ・ローカルのリトライ状態を所有する。静的なプラットフォーム・モジュールと bootstrap はページ・ライフタイムを保持する。Electron のインストールは別のフローである。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。dispatch モードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxclientmodules--clientmoduleregistry"></a>

### `ctx.clientModules` — `ClientModuleRegistry`

Web プラグイン・テーブル・サービス。インクリメンタルな `dsh.client` スキャン + ワイヤー合成 + bundle とパッケージ・アセットのルート + インデックス・インジェクション行。構築時にはアクティベーション・スキャンが同期的に実行される。既にロード済みのエントリーの中に不正な宣言や欠落した bundle があると、1 つの大きな例外に集約される（FAILED fiber。boot アクティベーション監査がそれを報告する）。

```ts cordis-catalog
/**
 * Current composed entry graph (stable object between changes).
 * @returns the graph served as `window.__DSH_BOOT__`.
 */
graph(): WebBootGraph

/**
 * Absolute path of an entry's client bundle.
 * @param id - entry id (package name).
 * @returns the path, or undefined for an unknown id.
 */
clientPath(id: string): string | undefined

/**
 * Serve an advertised revisioned bundle, source map, chunk, or package asset
 * without a Web server. Unknown URLs return 404, unsupported methods return
 * 405, and `HEAD` returns the same immutable headers without materializing a
 * body. Each bundle body is built once on its first `GET`; script
 * construction never reads maps. A package asset is read on every request
 * and answers 304 without a body when `If-None-Match` equals its entity tag;
 * its body holds the file open until the caller consumes or cancels it. A
 * bundle, map, or chunk body is the held buffer itself, not a copy, so a
 * caller must not modify its bytes.
 * @param request - shell-carrier request for a `/plugins` resource.
 * @returns the exact response also exposed by the optional Web route.
 */
async fetchBundle(request: Request): Promise<Response>

/**
 * Filesystem baseline captured before an entry's current bytes were read.
 * HMR compares it with the live files when installing a watch, so a write
 * between startup composition and watch installation cannot disappear into
 * the watcher's initial state.
 * @param id - entry id (package name).
 * @returns the path and baseline, or undefined for an unknown id.
 */
artifactBaseline(id: string): ClientArtifactBaseline | undefined

/**
 * Publish one completed bundle generation (the HMR watch's registration
 * hook; an entry restart publishes through reconciliation instead).
 * Unchanged mtime, ctime and size preserve the graph without reading the bundle.
 * @param id - entry id (package name).
 * @returns the current artifact rev, or undefined for an unknown id.
 */
rebuilt(id: string): string | undefined

/**
 * Subscribe to bundle rebuilds; fires when {@link rebuilt} changes a rev. A rev an
 * entry restart publishes reaches only {@link onGraphChanged} listeners.
 * @param listener - receives the entry id and its new bundle rev.
 * @returns the unsubscriber.
 */
onRebuilt(listener: (id: string, rev: string) => void): () => void

/**
 * Fires after any flush that recomposed the graph (row added/removed, or a
 * rebuilt rev change). Pull model: listeners re-read {@link graph}.
 * @param listener - notified with no payload.
 * @returns the unsubscriber.
 */
onGraphChanged(listener: () => void): () => void
```

ソース: [`packages/client/modules/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/modules/src/index.ts)
<!-- END GENERATED cordis-surface -->
