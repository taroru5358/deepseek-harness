# LSP ナビゲーション

日本語 | [English](../../../docs/subsystems/lsp.md) | [中文](../../../docs/subsystems/lsp.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/lsp.md](../../../docs/subsystems/lsp.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

LSP seam。1 つの `ctx.lsp` サービス上でセマンティックなコード・ナビゲーションを公開する[ケイパビリティ seam](../glossary.md#capability-seam) であり、パッケージに分割されている。サービス定義（[dsh-lsp](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/lsp/lsp)、`ctx.lsp` + プロバイダー・レジストリ）、汎用サービス・プロバイダー（[dsh-lsp-stdio](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/lsp/lsp-stdio)、設定された stdio 言語サーバー host）、コンシューマー（[dsh-tool-lsp](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/lsp/tool-lsp)、`lsp` ツール・スキーマ）である。LSP は**任意のケイパビリティの 1 つ**であり、agent ループの背骨の一部ではない。だからその語彙は [core.md](core.md) ではなくここにある。プロバイダーを交換しても、モデルがナビゲーションを要求する方法は変わらない。

ソース: [`packages/lsp/lsp/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/lsp/lsp/src/types.ts)

## 操作と座標

seam とモデルが公開するセマンティック・クエリはちょうど 4 つである。ユニオンは閉じているので、1 つ追加することは seam、プロバイダー、ツールにまたがるコンパイル強制の変更になる。位置と範囲はゼロ基準の UTF-16 で、プロトコルに一致する。モデル向けツールが 1 基準のカーソル慣習を所有し、入力時と出力時に変換する。

```ts type-equiv
/**
 * The four semantic queries the seam and model expose. A closed union: adding an operation is a
 * compile-enforced change across the seam, providers, and the tool. Symbols and call hierarchy are
 * not operations here; they need different schemas.
 */
type LspOperation = 'goToDefinition' | 'findReferences' | 'goToImplementation' | 'hover'
```

```ts type-equiv
/** A zero-based UTF-16 cursor coordinate, matching the LSP wire convention. */
interface LspPosition {
  /** Zero-based line. */
  readonly line: number
  /** Zero-based UTF-16 code-unit offset within the line. */
  readonly character: number
}
```

```ts type-equiv
/** A zero-based UTF-16 half-open range `[start, end)`. */
interface LspRange {
  readonly start: LspPosition
  readonly end: LspPosition
}
```

## リクエスト

すべてのフィールドが必須である。`workspaceRoot` は呼び出し元が指定し、`languageId` はプロバイダーの登録（リクエストではない）から来る。タイムアウトと結果上限はコンシューマーが所有する。したがって実装側の既定値補完を必要とするフィールドはなく、`resolve()` ステップも存在しない。プロバイダーは呼び出し元のリクエストと、導出された `languageId` を受け取る。これは一時ドキュメントを同期するだけで、選択には一切関与しない。

```ts type-equiv
/**
 * A caller's normalized query. Every field is required: `workspaceRoot` is caller-supplied,
 * `languageId` comes from the provider registration (not here), and consumers own timeouts and
 * result limits — so no field needs implementation defaulting and there is no `resolve()` step.
 */
interface LspQueryRequest {
  /** Which semantic query to run. */
  readonly operation: LspOperation
  /** The source file to query (relative to `workspaceRoot` or absolute; the provider canonicalizes). */
  readonly filePath: string
  /** The zero-based UTF-16 cursor position to query at. */
  readonly position: LspPosition
  /** The workspace root the provider resolves against and indexes; required, never defaulted. */
  readonly workspaceRoot: string
}
```

```ts type-equiv
/**
 * A request as a provider receives it: the caller's {@link LspQueryRequest} plus the `languageId`
 * the seam derived from the provider's extension mapping. The language id only synchronizes the
 * transient document; it does not participate in selection.
 */
interface LspProviderQuery extends LspQueryRequest {
  /** The LSP language id for `filePath`, from this provider's extension mapping. */
  readonly languageId: string
}
```

## 結果

閉じた判別ユニオンである。ナビゲーション操作は `locations` に、`hover` はコンテンツまたは `null` に正規化される。コンシューマーは `kind` で網羅的に `switch` するので、新しいアームを処理するまでコンパイルが壊れる。`findReferences` は常に宣言を含む。プロバイダーが内部でこれを強制するので、呼び出し元にフラグはない。`locations` バリアントは `resolvedWorkspaceUri`（プロバイダーの正規のワークスペース `file:` URI）を運ぶ。ロケーション URI を相対化する呼び出し元は、シンボリックリンクの可能性があるリクエスト・ルートに host プラットフォームのパス規則を適用するのではなく、その座標を使う。

```ts type-equiv
/** One resolved location: a document URI and the range within it. */
interface LspLocation {
  /** The target document URI (`file:` or otherwise), verbatim from the server. */
  readonly uri: string
  /** The range within the target document. */
  readonly range: LspRange
}
```

```ts type-equiv
/** Normalized hover content, or `null` for no hover at the position. */
interface LspHover {
  /** The normalized hover text (markdown or plaintext, provider-joined). */
  readonly contents: string
  /** The range the hover applies to, when the server supplied one. */
  readonly range?: LspRange
}
```

```ts type-equiv
/**
 * The closed result union. Navigation operations (`goToDefinition`, `findReferences`,
 * `goToImplementation`) normalize to `locations`; `hover` normalizes to content or `null`.
 * Consumers `switch` on `kind` to exhaustiveness so a new arm breaks compilation until handled.
 *
 * The `locations` variant carries `resolvedWorkspaceUri`: the provider's canonical `file:` URI for
 * the request's workspace root. A caller that relativizes location URIs MUST use this, not parse the
 * request's possibly symlinked process path with host-platform rules; the execution platform may
 * differ from the caller's.
 */
type LspQueryResult =
  | { readonly kind: 'locations'; readonly locations: readonly LspLocation[]; readonly resolvedWorkspaceUri: string }
  | { readonly kind: 'hover'; readonly hover: LspHover | null }
```

## プロバイダーとサービス

プロバイダーは安定したブランド付き `id` と、排他的な小文字先頭ドットの拡張子マップを所有する。`registerProvider` は id とすべての拡張子をアトミックに予約する。無効または競合する登録は何も公開しない。そのディスポーザーはすべての予約を解放する。選択はクエリごとで順序に依存しない。一致がなければ `LspError` の `LSP_UNAVAILABLE` を投げる。seam はプロトコル型、プロセス/ドキュメント制御、汎用 JSON-RPC エスケープハッチを公開しない。

```ts type-equiv
/**
 * A language-server backend registered on `ctx.lsp`. Each provider owns a stable {@link
 * LspProviderId} and an extension-to-language-id map (lowercase, leading-dot keys).
 * `findReferences` always includes declarations — the provider enforces this internally; callers
 * get no flag.
 */
interface LspProvider {
  /** Stable provider identity, reserved atomically with the extension mappings. */
  readonly id: LspProviderId
  /** Lowercase leading-dot extension → LSP language id (e.g. `{ '.ts': 'typescript' }`). */
  readonly extensionToLanguage: Readonly<Record<string, string>>
  /**
   * Run one query. The seam has already selected this provider and derived `languageId`.
   * @param request - the resolved provider query (caller request + derived language id).
   * @param signal - optional cancellation; the provider stops its own work when it aborts.
   * @returns the normalized, closed-union result.
   */
  query(request: LspProviderQuery, signal?: AbortSignal): Promise<LspQueryResult>
}
```

```ts type-equiv
/**
 * The LSP capability seam (`ctx.lsp`). Owns provider registration/selection and normalized query
 * execution; exposes exactly the four operations and no protocol escape hatch.
 */
interface LspService {
  /**
   * Register a provider, atomically reserving its id and every normalized extension. Any conflict
   * or invalid input publishes nothing and throws `LspError`; the returned disposer releases all
   * reservations. Disposed with the calling fiber.
   * @param provider - the backend to register.
   * @returns a synchronous disposer releasing the id and all extension reservations.
   */
  registerProvider(provider: LspProvider): () => void
  /**
   * Select a provider by the file's extension and run one query. Selection is per-query and
   * order-independent; no match throws `LspError` `LSP_UNAVAILABLE`.
   * @param request - the normalized query.
   * @param signal - optional cancellation forwarded to the selected provider.
   * @returns the normalized, closed-union result.
   */
  query(request: LspQueryRequest, signal?: AbortSignal): Promise<LspQueryResult>
}
```

`LspProviderId` は seam のブランド付き id（[dsh-brand](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/util/brand) の `Branded<'LspProviderId'>`）である。`LspError` は `HarnessError` を継承し、`LSP_INVALID_PROVIDER`、`LSP_CONFLICT`、`LSP_UNAVAILABLE`、`LSP_DISPOSED`、`LSP_UNSUPPORTED_OPERATION`、`LSP_MALFORMED_RESPONSE` のような安定したコードを持つ。呼び出し元は `message` を解析するのではなく、これらのコードで分岐する。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。dispatch モードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxlsp--lspservice"></a>

### `ctx.lsp` — `LspService`

LSP ケイパビリティ seam（`ctx.lsp`）。プロバイダーの登録/選択と正規化クエリの実行を所有し、ちょうど 4 つの操作だけを公開してプロトコル・エスケープハッチは公開しない。

```ts cordis-catalog
/**
 * Register a provider, atomically reserving its id and every normalized extension. Any conflict
 * or invalid input publishes nothing and throws `LspError`; the returned disposer releases all
 * reservations. Disposed with the calling fiber.
 * @param provider - the backend to register.
 * @returns a synchronous disposer releasing the id and all extension reservations.
 */
registerProvider(provider: LspProvider): () => void

/**
 * Select a provider by the file's extension and run one query. Selection is per-query and
 * order-independent; no match throws `LspError` `LSP_UNAVAILABLE`.
 * @param request - the normalized query.
 * @param signal - optional cancellation forwarded to the selected provider.
 * @returns the normalized, closed-union result.
 */
query(request: LspQueryRequest, signal?: AbortSignal): Promise<LspQueryResult>
```

ソース: [`packages/lsp/lsp/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/lsp/lsp/src/types.ts)
<!-- END GENERATED cordis-surface -->
