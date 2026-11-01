# HTTP サーバー

日本語 | [English](../../../docs/subsystems/web-server.md) | [中文](../../../docs/subsystems/web-server.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/web-server.md](../../../docs/subsystems/web-server.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

[dsh-host-webserver](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/host/webserver) は GUI ホスト向けのブラウザ HTTP キャリアである。1 つの `node:http` サーバー（`tls` が設定されていれば `node:https`）が `ctx.webServer`、名前付きルート・レジストリ、任意の gzip レスポンス圧縮、index.html 変換コールバック、そしてプラグインが要求できる 1 つのフォールバック（ハンドラーまたはリソース）を提供する。これは agent ループの一部ではなく、ケイパビリティ seam でもない。その唯一の harness 概念は index レンダリングが代弁する Peer であり、`/api` ブリッジ、プラグイン・バンドル、HMR イベント・ストリームを含むすべての機能ルートは別のプラグインが登録する（[レイヤリング・リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/app-boot/README.md)）。ブラウザにのみサービスを提供する。Electron はビルド済みファイルを `file://` 経由でロードし、fetch リクエストをこのサーバーではなく IPC ブリッジ経由で送る。

ソース: [`packages/host/webserver/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/host/webserver/src/index.ts)

## ルート

```ts type-equiv
/** Route match kind: 'exact' matches the pathname verbatim; 'prefix' p matches p and p/<anything>. */
type WebRouteKind = 'exact' | 'prefix'
```

```ts type-equiv
/** One named route registration. */
interface WebRoute {
  kind: WebRouteKind
  /** Absolute pathname, no trailing slash. */
  path: string
  /** Owns the full response lifecycle (may hold the response open, e.g. SSE). */
  handler: (req: IncomingMessage, res: ServerResponse) => void | Promise<void>
}
```

```ts type-equiv
/**
 * Answers one GET or HEAD request with a Fetch response. An HTTP request
 * arrives without a Peer and carries the client's headers; an in-process
 * {@link WebServer.readResource} read passes the Peer its caller admitted.
 */
type WebResource = (request: Request, peer?: PeerScope) => Promise<Response>
```

```ts type-equiv
/** One named route answered by a resource instead of a handler. */
interface WebResourceRoute {
  kind: WebRouteKind
  /** Absolute pathname, no trailing slash. */
  path: string
  /** Answers HTTP GET and HEAD requests and in-process reads. */
  resource: WebResource
}
```

マッチ順は固定である。まず完全一致テーブル、次に最も長く一致するプレフィックス、最後に登録済みのフォールバック。登録順はリクエストに面するセマンティクスを持たない。名前付きルートは互いに素になるよう構成され、フォールバック席は名前付きルートが要求しないあらゆるものに応答する。所有者は 1 つだけで、2 回目の登録はスローする。出荷される Web コンポジションはこの席を [`dsh-host-frontend-static`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/host/frontend-static/src/index.ts)（SPA dist サーバー）で要求し、そのセマンティクスは固定されている。Connection は dist ルートと設定済み index の HTML が読まれる前にそれらを認証する。index 以外のアセットは公開のままである。GET/HEAD 以外は 405、dist ルート外へのトラバーサルは 403、存在するファイルは直接配信され、存在しない対象またはファイルでない対象は空の 404 レスポンスとなり、未知の拡張子は octet-stream として送られる。

ルートまたはフォールバックは、ハンドラーの代わりにリソース（Fetch の `Request` から `Response` への関数）を渡すことができる。HTTP の `GET` と `HEAD` リクエストは Peer なしでそれに到達し、`readResource(request, peer)` は呼び出し元が受け入れた Peer のためにプロセス内でそれを読む。読み取りは同じマッチ順に従うが、到達するのはリソースだけである。したがって、所有者プライベートなデータを返したりアクションを実行したりするルートは、`GET` に応答する場合でもハンドラーのままである。

<a id="config"></a>

## 設定

```ts type-equiv
/** Web server listen, TLS, and response-compression config. */
interface Config {
  /**
   * Listen address: a concrete IPv4 or IPv6 literal of one local interface,
   * for example the container's own Pod address from `hostname -i`. A loopback
   * literal (any address in 127/8, `::1`, or a mapped form of either) keeps the
   * server on this machine; any other literal serves the network that address
   * belongs to, over plain HTTP unless `tls` is set. The unspecified address —
   * IPv4 any, IPv6 any, and the IPv4-mapped forms of IPv4 any — is rejected at
   * load: it would expose the port on every interface at once.
   */
  host: string
  /** Listen port; zero requests an OS-assigned port. */
  port: number
  /**
   * Serve HTTPS with this certificate and key instead of plain HTTP. Both files
   * are read once, before the listener binds: an unreadable or empty file,
   * invalid PEM, or a key that does not match the certificate rejects
   * initialization rather than falling back to HTTP. The material is never
   * re-read, so replacing a certificate takes a reload. Omitted or null listens
   * over plain HTTP.
   */
  tls?: TlsConfig
  /** Response compression for socket-backed HTTP requests. @default 'none' */
  compression?: 'none' | 'gzip'
  /** Gzip DEFLATE level from 0 through 9. @default 1 */
  compressionLevel?: number
  /** Minimum known response length eligible for gzip; unknown-length streams are eligible. @default 1024 */
  compressionThresholdBytes?: number
}
```

```ts type-equiv
/**
 * TLS material for the HTTPS listener: one certificate chain file and the
 * private key file it pairs with. Both hold PEM text and both resolve against
 * the process working directory.
 */
interface TlsConfig {
  /** Certificate chain file, leaf certificate first, PEM, no passphrase. */
  certFile: string
  /** Private key file for the chain's leaf certificate; unencrypted PEM. */
  keyFile: string
}
```

どのコンポジションもすべてのインターフェースを一度にバインドできない。未指定アドレスはロード時に拒否される。エクスポートされる `isWildcardHost` と `isLoopbackHost` は、解析済みの値からバインド・アドレスを分類する。IPv6 の `%zone` は `listen` のために保持され、分類はアドレスだけを読む。キャリアは認証も Origin ポリシーも所有せず、`tls` が証明書と鍵を供給しない限りプレーン HTTP を提供する。`compression` のデフォルトは `none` で、出荷される Web バンドルは gzip レベル 1 と 1024 バイトのしきい値を選ぶ。出荷される `dsh web` コマンドはデフォルトでループバックを選び、1 つの具体的な `--host` アドレスを受け付ける。非ループバック・バインドがプレーン HTTP を提供する場合は起動時に警告する。その Connection プラグインは Host/Origin チェックと、すべての Host API ルートおよびストリームに対するブラウザ Session 認証を提供し、バインド・アドレスと `--trusted-host` がどのブラウザ Host を受け入れるかを左右する。`--public-url` は広告される URL だけを変える。他のコンポジションは自身のバインドとルート認証ポリシーを所有する。dist の位置は、席を要求するフロントエンド・プラグインのアセンブリ上の事実である。

## サービス

`WebServer`（`ctx.webServer`）はアクティブ化で即座に listen する。listen の失敗（EADDRINUSE など）は初期化を拒否し、boot プロセスが失敗した fiber を報告する。`tls` のペアはリスナーがバインドする前に一度だけ読まれ、読めない、空、または不一致のペアは、HTTP にフォールバックする代わりに同じように初期化を拒否する。`register(route)` は 1 つの名前付きルートを追加し、その disposer を返す。重複する `(kind, path)` はスローする。ルート・パターンはコンポジション・レベルの契約であり、衝突は設定ミスだからである。Gzip は対象となるソケット裏付けのレスポンスをサーバー内でラップするため、ハンドラーは `ServerResponse` の直接所有権を保ち、リソースは書き戻すべき `Response` をサービスに渡す。既存のコンテンツ・エンコーディング、`Cache-Control: no-transform`、レンジ、SSE、ZIP、およびパッケージ済みの `.gz` Worker イメージは identity レスポンスのままである。`collectIndexInjections(peer)` は、ページが代弁する Peer を渡す 1 回の `webserver/index-inject` emit で構造化された `IndexInjection` 行を集め、`renderIndex(html, peer)` はそれらを成功したルートおよび設定済み index レスポンスへ描画してから、生の `tapIndex(transform)` エスケープハッチ変換を登録順に適用する。[dsh-client-modules](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/client/modules) がそのイベントに boot マニフェスト行で応答する。`port` はリッスン中のポート（`config.port` が 0 のときに OS が割り当てたポートを含む）を読み、`protocol` はリスナーが提供するスキーム（`http:` または `https:`）を報告する。

処理がスローするリクエスト（`decodeURIComponent` に当たる不正な % エスケープ、本文途中で切断するクライアント）は警告としてログに記録され、400 で応答される。ヘッダーがすでに出ている場合はソケットが破棄される。プロセスが終了することは決してない。破棄は `close()` と `closeAllConnections()` を対にする。ハンドラーがレスポンスを開いたままにする（SSE）ことがあり、そのような接続は自力では終わらないためである。強制クローズがなければ、teardown はハングする。TLS ソケットはハンドシェイク後に初めて接続として数えられるため、サーバーは生の TLS ソケットをアップグレード済みのものと並べて追跡し、破棄時に両方を破棄する。このパッケージは決して出力しない。URL 行はシェルに属する。dev モードのバンドル監視パイプラインを含むパッケージごとの運用詳細は、[README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/host/webserver/README.md) にある。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxconnection--hostconnectionhandle"></a>

### `ctx.connection` — `HostConnectionHandle`

トランスポート非依存のアダプターが消費するホスト `ctx.connection` のメンバー。

```ts cordis-catalog
/**
 * Compose exact Fetch routes and the shared-channel RPC interceptor.
 * @param channel - shared channel mounted by Connection.
 * @returns Fetch handler for trusted, authenticated requests.
 */
createSharedFetchHandler(channel: '/api'): ConnectionFetchHandler

/**
 * Apply Connection's Host/Origin checks and browser authentication to
 * another Web route.
 * @param request - request headers from the HTTP or upgrade request.
 * @returns rejection status, or undefined when the route may accept the request.
 */
requestRejection(request: ConnectionTrustRequest): ConnectionRequestRejection

/**
 * Admit an HTTP request against its signed browser or API credential and live Peer.
 * @param request - request headers from the HTTP or upgrade request.
 * @returns the credential's Peer, or the rejection status.
 */
admit(request: ConnectionTrustRequest): PeerAdmission

/**
 * Create an independent in-process Peer, disposed with the Context that read `ctx.connection`.
 * @param options - fixed principal and optional interaction participation (default `answer`).
 * @returns a new Peer distinct from every other Peer.
 */
createPeer(options: { readonly principal: Principal; readonly interactions?: 'answer' | 'none' }): PeerScope

/**
 * Claim a physical socket for an API Peer; browser sockets share their owner Peer.
 * @param request - authenticated upgrade request headers.
 * @returns the Peer and idempotent release, or 409 for duplicate API ownership.
 * The carrier releases on socket termination or failed upgrade; release removes
 * API admission synchronously without cancelling independent HTTP calls.
 */
admitConnection(request: ConnectionTrustRequest): ConnectionAdmission

/**
 * Authenticate one frontend index request, owning a token redirect or 401.
 * The cookie this mints is `Secure` when the mounted Web carrier serves TLS,
 * which only that listener's protocol decides.
 * @param request - root or configured-index HTTP request.
 * @param response - response owned when the result is false.
 * @returns true only when the frontend may serve index.html.
 */
authorizeIndex(request: ConnectionIndexRequest, response: ConnectionIndexResponse): boolean

/**
 * Add the fresh process token to an ordinary Web application URL.
 * @param baseUrl - clean application URL whose authority and mount are preserved.
 * @returns tokenized URL for initial login; a mount proxy strips its prefix before {@link authorizeIndex}.
 */
authenticatedUrl(baseUrl: string): string
```

型: [Principal](typert.md)

ソース: [`packages/client/connection/src/rpc.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/connection/src/rpc.ts)

<a id="ctxwebapp--webapp"></a>

### `ctx.webApp` — `WebApp`

このプロセスが提供する Web GUI。`web-runtime` 行によって提供される。

```ts cordis-catalog
/**
 * The URL this GUI advertises, also used for `DSH_WEB_URL` and the startup URL line.
 * @returns the configured `publicUrl`, otherwise the listener origin.
 */
getAdvertisedUrl(): string
```

ソース: [`packages/bundle/web-app/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/web-app/src/index.ts)

<a id="ctxwebserver--webserver"></a>

### `ctx.webServer` — `WebServer`

ブラウザ HTTP キャリア・サービス。アクティブ化は設定済みの TLS マテリアルをロードしてから即座に listen する。マテリアルまたは listen の失敗は初期化を拒否し、boot プロセスが失敗した fiber を報告する。設定済みの名前付きルートは相異なる必要があるため、ルート登録順はリクエストに影響しない。名前付きルートに一致しないリクエストは、フォールバック所有者が登録するまで 404 で応答される。

```ts cordis-catalog
/**
 * The root URL local clients use to reach the bound listener: the protocol,
 * the {@link bindUrlHost} form of the bind address, and the listening port.
 * @returns the origin, for example `http://127.0.0.1:3080`.
 * @throws before `listen` binds a port, or when the bind address has no URL form.
 */
getOrigin(): string

/**
 * Register a named route. Duplicate (kind, path) throws — route patterns are
 * a composition-level contract, so a collision is a misconfiguration.
 * @param route - kind, path, and the owning handler or resource.
 * @returns the disposer removing the route.
 */
register(route: WebRoute | WebResourceRoute): () => void

/**
 * Register an exact-path HTTP upgrade route. Duplicate paths throw because
 * one socket can have only one protocol owner.
 * @param route - pathname and handler owning negotiation plus socket use.
 * @returns the disposer removing the route.
 */
registerUpgrade(route: WebUpgradeRoute): () => void

/**
 * Claim the fallback seat: the handler or resource answering every request
 * no named route matches (the SPA dist server in the shipped Web
 * composition). One owner only — a second registration throws, because two
 * fallbacks cannot compose.
 * @param fallback - a handler owning the full response lifecycle of
 * unmatched requests, or `{ resource }` answering them.
 * @returns the disposer releasing the seat.
 */
registerFallback(fallback: WebRoute['handler'] | Pick<WebResourceRoute, 'resource'>): () => void

/**
 * Read the response an HTTP request for the same URL would receive, as the
 * resource computes it for `peer`, without an HTTP exchange. The request
 * path picks the responder by the HTTP match order — exact, longest prefix,
 * then the fallback — and only a resource answers. This method checks no
 * cookie, Host, or Origin: the caller admits the reader before reading.
 * @param request - the read; its URL path selects the responder.
 * @param peer - the Peer the read speaks for, passed to the resource.
 * @returns the resource's response, without a body for HEAD as over HTTP;
 * 405 with `Allow: GET, HEAD` for any other method, checked before matching,
 * where over HTTP a handler route or an unmatched path answers such a
 * method itself; 404 when the path matches a handler route, or reaches the
 * fallback seat while it is unclaimed or holds a handler. A resource that
 * throws rejects the read, where an HTTP request would receive 400.
 */
async readResource(request: Request, peer: PeerScope): Promise<Response>

/**
 * Register a raw-HTML index transform, the escape hatch for markup no
 * {@link IndexInjection} row expresses: {@link renderIndex} applies taps in
 * registration order after rendering the structured rows.
 * @param transform - pure html-to-html function.
 * @returns the disposer removing the transform.
 */
tapIndex(transform: (html: string) => string): () => void

/**
 * Run an index.html body through the registered taps in registration order
 * — called by the fallback owner on every index response it renders.
 * @param html - the raw index.html body.
 * @returns the transformed body.
 */
applyIndexTaps(html: string): string

/**
 * Gather the structured injection table for one Peer: one
 * `webserver/index-inject` emit, every subscriber pushes its current rows.
 * Fresh per call, so subscribers read live state (module graph, theme
 * preference, the Peer's access) at emit time.
 * @param peer - the Peer the page speaks for.
 * @returns rows in subscriber activation order.
 */
collectIndexInjections(peer: PeerScope): IndexInjection[]

/**
 * Render one index.html body for one Peer: the structured injection table
 * first, then the raw `tapIndex` transforms over the result.
 * @param html - the raw index.html body.
 * @param peer - the Peer the page speaks for.
 * @returns the transformed body.
 */
renderIndex(html: string, peer: PeerScope): string
```

ソース: [`packages/host/webserver/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/host/webserver/src/index.ts)

<a id="connection-events"></a>

### `connection/*` イベント

<a id="connectionrequest--waterfall"></a>

#### `connection/request` — waterfall

認証済みの共有 API リクエストを、本文転送を含めて受け入れるかラップする。リスナーが後続のリクエストを拒否しても、既存のリクエストは継続する。

```ts cordis-catalog
/**
 * Admit or wrap an authenticated shared API request, including body transfer.
 * Existing requests continue when a listener refuses subsequent requests.
 * @param request - Authenticated incoming HTTP request.
 * @param response - Response owned until the delegated bridge settles.
 * @param next - Delegate to the next listener or the shared API bridge.
 * @mode waterfall
 */
'connection/request'(request: IncomingMessage, response: ServerResponse, next: () => Promise<void>): Promise<void>
```

ソース: [`packages/client/connection/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/connection/src/index.ts)

<a id="webserver-events"></a>

### `webserver/*` イベント

<a id="webserverindex-inject--emit"></a>

#### `webserver/index-inject` — emit

`peer` が代弁するページの構造化 index インジェクション・テーブルを集める。index レンダリングのたび、および worker の boot ペイロード・リクエストのたびに発行され、リスナーは自身の現在の行を push する。したがって行のデータは emit 時に新鮮に読まれる。

```ts cordis-catalog
/**
 * Collect the structured index injection table for the page `peer` speaks
 * for. Emitted on every index render and every worker boot-payload
 * request; listeners push their current rows, so a row's data is read
 * fresh at emit time.
 * @param table - Mutable row table; listeners append in activation order.
 * @param peer - The Peer the rendered page speaks for.
 * @mode emit
 */
'webserver/index-inject'(table: IndexInjection[], peer: PeerScope): void
```

ソース: [`packages/host/webserver/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/host/webserver/src/index.ts)
<!-- END GENERATED cordis-surface -->
