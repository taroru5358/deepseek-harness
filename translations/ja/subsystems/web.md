# Web アクセス

日本語 | [English](../../../docs/subsystems/web.md) | [中文](../../../docs/subsystems/web.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/web.md](../../../docs/subsystems/web.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

web アクセス seam — 1 つの `ctx.web` サービス上の **2 つの操作**（search と fetch）にまたがる [capability seam](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-06-24-web-capability-seam.md)。パッケージに分割される: Service Definition（[dsh-web](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/web/web)、`ctx.web` + provider レジストリ）、Service Providers（[dsh-web-search-exa](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/web/web-search-exa)、[dsh-web-search-perplexity](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/web/web-search-perplexity)、[dsh-web-search-deepseek](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/web/web-search-deepseek)、[dsh-web-deepseek-api](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/web/web-deepseek-api)、[dsh-web-fetch-http](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/web/web-fetch-http)）、Consumer（[dsh-tool-web](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/web/tool-web)、`web_search`/`web_fetch` ツールスキーマ）。Web は **1 つの任意 capability** であり、agent-loop の背骨の一部ではない — したがってその語彙は [core.md](core.md) ではなくここにある。検索 provider の交換はモデルがクエリを要求する方法を変えず、フェッチ provider の交換はモデルが URL を要求する方法を変えない。

ソース: [`packages/web/web/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/web/web/src/types.ts)

## なぜ 1 つの capability に 2 つの操作があるのか

検索とフェッチはリクエストスキーマもビジネスロジックも共有しないが、意図的に 1 つの `ctx.web` 中間層である: 1 つの provider 選択ポリシー所有者、1 つの abort/エラー語彙、1 つの製品向け「この harness がどのように web に到達するか」設定 API。コストはサービス上の並行する `searchX`/`fetchX` メソッドペアである。その並行性は意図的であり、抽出漏れではない。provider はツールではなく **capability**（`WebSearchProvider` または `WebFetchProvider`）を登録する。モデル向けの名前、スキーマ、プロンプトガイダンス、プレゼンテーションはすべて単一の `dsh-tool-web` コンシューマーにある。

## 検索リクエストと結果

各 seam リクエストはちょうど 1 つの `query` を運ぶ。`dsh-tool-web` コンシューマーは必須の `queries` 配列を受け入れ、それを別々の seam リクエストにファンアウトする。1 項目の配列は 1 回の検索を行う。`maxResults` はコンシューマー所有の境界（`dsh-tool-web` の `searchMaxResults` 設定、デフォルト `8`）であり、seam を通り抜けて返り道で強制される — provider が過剰に返すと、seam は `sources[]` を切り詰めて `truncated` を設定する。

```ts type-equiv
/**
 * What one search-capable backend is asked to search. Each request carries one
 * query; a consumer may issue several requests. `maxResults` is a
 * `dsh-tool-web`-layer bound passed through unchanged and enforced on the way
 * back by the seam (see {@link WebSearchResult}).
 */
interface WebSearchRequest {
  readonly query: string
  /**
   * Upper bound on returned sources; the seam truncates to it. Omitted = no
   * bound. `dsh-tool-web` always sets it. A provider whose API supports a
   * result-count control (Exa's `numResults`) should apply it at the request
   * layer as a cost/latency optimization; the seam enforces the bound
   * regardless.
   */
  readonly maxResults?: number
}
```

```ts type-equiv
/**
 * Normalized search outcome. `content` is optional provider-generated answer
 * text or summary (Exa and DeepSeek return none; Perplexity returns a
 * generated answer).
 * `sources[]` is the portable citation shape. `truncated` is set by the seam
 * when it cut `sources[]` down to `maxResults`.
 */
interface WebSearchResult {
  /** Optional provider-generated answer text, search context, or summary. */
  readonly content?: string
  /** Citeable sources, already truncated to the request's `maxResults`. */
  readonly sources: readonly WebSearchSource[]
  /** True when the seam dropped sources to honor `maxResults`. */
  readonly truncated: boolean
}
```

```ts type-equiv
/**
 * One citeable source. A source always has a URL; `title`, `snippet`, `summary`,
 * and `publishedAt` are optional because not every provider returns them — forcing
 * adapters to invent them would make the seam lie (Perplexity citations may be
 * URL-only). `dsh-tool-web` renders `title ?? hostname(url)` for display.
 */
interface WebSearchSource {
  readonly url: string
  readonly title?: string
  /** Short excerpt suited to display. */
  readonly snippet?: string
  /** Longer excerpt suited to model reading; consumers bound its length. */
  readonly summary?: string
  /** Publication/crawl timestamp as a provider-supplied ISO-8601 string. */
  readonly publishedAt?: string
}
```

## フェッチリクエストと結果

```ts type-equiv
/**
 * What one fetch-capable backend is asked to retrieve. The request deliberately
 * omits timeout, format, prompt, and extraction controls: cancellation is a
 * direct execution argument, while presentation and higher-level LLM concerns
 * belong outside safe retrieval.
 */
interface WebFetchRequest {
  readonly url: string
}
```

HTTP ステータスはフェッチされたリソース状態の一部であり、自動的に失敗ではない: `404`/`500` のネットワークフェッチ成功は、ステータスコードと上限付きのデコード済み本文を持つ `WebFetchResult` を返す。`url` は許可されたリダイレクト後の最終 URL である。`WebError` はリソースを安全に取得または表現できない失敗のために予約されている。

```ts type-equiv
/**
 * Normalized fetch outcome. A successful network fetch of a non-2xx response is
 * a result, not an error: the status code is part of the fetched resource
 * state. {@link WebError} is reserved for failures to safely retrieve or
 * represent the resource.
 */
interface WebFetchResult {
  /** The final URL after allowed redirects (the request URL is in the request). */
  readonly url: string
  /** HTTP status code of the fetched response. */
  readonly statusCode: number
  /** Decoded body, classified by content kind. */
  readonly body: WebFetchBody
  /** True when the provider capped the decoded body. */
  readonly truncated: boolean
}
```

```ts type-equiv
/**
 * The decoded body of a fetched resource. A CLOSED discriminated union owned by
 * `dsh-web`: the provider decodes the kind and `dsh-tool-web` renders it, so a
 * new kind is a coordinated change across known packages, not a plugin
 * extension. Consumers `switch` on `kind` ending in `default: assertNever(...)`
 * so adding a kind breaks compilation at every consumer until handled. Each arm
 * stays its own object literal even where fields coincide, so an arm can gain
 * fields the others lack.
 */
type WebFetchBody =
  | { readonly kind: 'html'; readonly content: string }
  | { readonly kind: 'text'; readonly content: string }
```

## Provider の可用性

provider の `available(): boolean` は安価なローカルチェック（認証情報の存在、解析可能な設定）であり、**ネットワーク呼び出しを行ってはならない**。それは実行時選択への入力であり、ヘルスシステムではない: `search()`/`fetch()` は使用可能な provider を選ぶためにそれを読み、選択失敗は呼び出し元がルーティングする構造化 `WebError` として表面化する — コードとメッセージに分岐可能な詳細（欠落した id または曖昧な候補集合）を運ぶ。

検索またはフェッチ provider は `ready(llmProvider): Promise<boolean>` も実装してよい。設定のみのチェックであり、ネットワーク呼び出しを行ってはならない: そのチャットルート上のモデルリクエストのために行われる操作に送信可能な認証情報があるかどうかを答える。`searchReady(llmProvider?)` と `fetchReady(llmProvider?)` は `search()` または `fetch()` が選択する provider だけに尋ね、`ready()` を持たない provider を ready として扱い、選択が失敗したときは true を答える。その操作がその失敗を `WebError` として報告するからである。Readiness は選択入力ではない。`dsh-tool-web` はそれを `web_search` と `web_fetch` のアクティベーション条件にするため、ルートが false と答えるモデルステップはツールを宣言もディスパッチもしない。

選択は登録、設定、HMR の順序に決して依存しない: capability は明示的な provider id（設定 `searchProvider`/`fetchProvider`、または同じフィールドを供給する対応する環境変数）を持つか、使用可能な provider がちょうど 1 つ登録されているときに自動選択する。設定された id なしで複数の使用可能な provider がある場合は `WEB_PROVIDER_AMBIGUOUS` であり、first-wins ではない。各操作は実行時に設定された id を読むため、保存された設定変更は `ctx.web` を再マウントせずに次の検索またはフェッチに適用される。

## フェッチのネットワークポリシー

出荷される Cordis、Code、Standard プリセットは、すべての sandbox と承認モードで呼び出しごとの確認なしに `web_fetch` を公開する。ファイル sandbox プリセットは Web のネットワークアクセスを統治しない。確認が必要なデプロイメントは `tools/pre-execute` ポリシーを追加するか、フェッチを無効化しなければならない。

HTTP provider は宛先アドレスを制限しない: `web_fetch` は他のホストと同様にループバック、プライベートネットワーク、クラウドメタデータホストに到達し、モデルは到達可能な任意の URL にデータを送信できる。クロスオリジンのリダイレクトには新しいツール呼び出しが必要である。モデルをプライベート宛先から遠ざけなければならないデプロイメントには、ネットワーク閉じ込め、`tools/pre-execute` ポリシー、またはフェッチツールなしが必要である。

## エラー

`WebError extends HarnessError`（[core.md](core.md) のエラー分類）は `code: string`（オープン、他のすべての seam のエラー — `LlmError`、`SubagentError` — と同様）を持ち、閉じた union ではない: provider は `dsh-web` を編集せずに独自のコードを発生させてよく、コンシューマーは未知のコードを許容しなければならない。コードは所有者によって分割される。Seam 中立のコードは共有 `WebRuntime` 契約によって発生する: `WEB_PROVIDER_UNAVAILABLE`、`WEB_PROVIDER_CONFIGURED_MISSING`、`WEB_PROVIDER_CONFIGURED_UNAVAILABLE`、`WEB_PROVIDER_AMBIGUOUS`、`WEB_DUPLICATE_PROVIDER`（登録時のプログラミングエラーであり、`LlmRuntime` の `DUPLICATE_ADAPTER` の類似物）、`WEB_ABORTED`、および `WEB_PROVIDER_ERROR`（seam を通じて表面化する provider 自身の失敗 — ネットワーク/トランスポート失敗を含む — の catch-all。DNS、接続拒否、TLS）。Fetch トランスポートのコードは `dsh-web-fetch-http` 実装が所有し、別のフェッチバックエンドがそれらを発生させる必要はない: `WEB_INVALID_URL`、`WEB_BLOCKED_URL`、`WEB_REDIRECT_BLOCKED`、`WEB_FETCH_TOO_LARGE`、`WEB_FETCH_TIMEOUT`、`WEB_UNSUPPORTED_CONTENT_TYPE`。

## サービス

`WebRuntime` は検索とフェッチの provider を登録し、重複 id を `WEB_DUPLICATE_PROVIDER` で拒否し、実行時に構造化された選択エラーで provider を解決する。ローカルフェッチバックエンドは HTTP(S) のみを受け入れ、認証情報を拒否し、各リクエストをインストールされたプロキシポリシーを通じて送信し、同一オリジンのリダイレクトホップごとに URL 検証を繰り返し、リダイレクト、バイト、文字、時間に上限を課し、本文をデコードする。ツールがプレゼンテーションを所有する。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxweb--webruntime"></a>

### `ctx.web` — `WebRuntime`

web アクセスサービス。`ctx.web` として登録される（コンテキストごとに 1 インスタンス）。

選択意味論（実行時に解決され、決して順序に依存しない）:

- 設定された id が登録済みで `available()` → その provider。
- 設定された id が未登録 → `WEB_PROVIDER_CONFIGURED_MISSING`。
- 設定された id が登録済みだが利用不可 → `WEB_PROVIDER_CONFIGURED_UNAVAILABLE`。
- id 未設定で、登録済みの使用可能な provider がちょうど 1 つ → その provider。
- id 未設定で、複数の使用可能な provider → `WEB_PROVIDER_AMBIGUOUS`。
- id 未設定で、使用可能な provider がない → `WEB_PROVIDER_UNAVAILABLE`。

```ts cordis-catalog
/**
 * Register a search provider. Throws {@link WebError} `WEB_DUPLICATE_PROVIDER`
 * if its id is already registered for search. Returns a disposer; disposed
 * with the calling fiber.
 * @param provider - the provider; its `id` is the registry key.
 * @returns the disposer that unregisters the provider.
 */
registerSearchProvider(provider: WebSearchProvider): () => void

/**
 * Register a fetch provider. Throws {@link WebError} `WEB_DUPLICATE_PROVIDER`
 * if its id is already registered for fetch. Returns a disposer; disposed
 * with the calling fiber.
 * @param provider - the provider; its `id` is the registry key.
 * @returns the disposer that unregisters the provider.
 */
registerFetchProvider(provider: WebFetchProvider): () => void

/**
 * Run one search through the selected provider. Resolves the provider at call
 * time with the selection rules above; throws {@link WebError} when the
 * capability cannot run. The seam enforces `request.maxResults` on the result:
 * if the provider over-returns, `sources[]` is truncated and `truncated` set.
 * @param request - the query and optional result limit.
 * @param signal - optional cancellation signal forwarded to the provider.
 * @returns the provider's results, capped to `request.maxResults`.
 */
async search(request: WebSearchRequest, signal?: AbortSignal): Promise<WebSearchResult>

/**
 * Whether the provider `search()` would select can authenticate a search made for a
 * model request on the `llmProvider` chat route. Asks only that provider's `ready()`;
 * one without `ready()` counts as ready. Makes no network calls. A selection failure
 * answers true, because `search()` reports that failure as a structured `WebError`.
 * @param llmProvider - the provider route id of the model request, when known.
 * @returns false only when the selected provider reports that it cannot authenticate.
 */
async searchReady(llmProvider?: string): Promise<boolean>

/**
 * Retrieve one URL through the selected provider. Resolves the provider at
 * call time with the selection rules above; throws {@link WebError} when the
 * capability cannot run. A non-2xx response is a result, not a throw.
 * @param request - the URL plus retrieval options.
 * @param signal - optional cancellation signal forwarded to the provider.
 * @returns the retrieval outcome; non-2xx responses resolve descriptively.
 */
async fetch(request: WebFetchRequest, signal?: AbortSignal): Promise<WebFetchResult>

/**
 * Whether the provider `fetch()` would select can authenticate a fetch made for a
 * model request on the `llmProvider` chat route. Asks only that provider's `ready()`;
 * one without `ready()` counts as ready. Makes no network calls. A selection failure
 * answers true, because `fetch()` reports that failure as a structured `WebError`.
 * @param llmProvider - the provider route id of the model request, when known.
 * @returns false only when the selected provider reports that it cannot authenticate.
 */
async fetchReady(llmProvider?: string): Promise<boolean>
```

ソース: [`packages/web/web/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/web/web/src/index.ts)
<!-- END GENERATED cordis-surface -->
