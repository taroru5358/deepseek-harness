# トークン・メーター

日本語 | [English](../../../docs/subsystems/token-meter.md) | [中文](../../../docs/subsystems/token-meter.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/token-meter.md](../../../docs/subsystems/token-meter.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

`@deepseek-ai/dsh-token-meter` は、リクエスト圧力と位置ベースのサーフェス価格付けのための、1 つの切り離されたリプレイ・スナップショットを公開する。`logRevision` は、その測定のすべてのフィールドについて消費された永続イベントの数である。

ソース: [`packages/llm/token-meter/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/token-meter/src/types.ts)

## `TokenMeasurement`

```ts type-equiv
/** Detached immutable request-pressure and surface snapshot at one consumed log revision. */
interface TokenMeasurement {
  /** Number of durable events consumed; equal to the next unread event seq. */
  readonly logRevision: SessionLogOffset
  /** Provider or heuristic anchor used for this measurement. */
  readonly baseline: TokenMeasurementBaseline
  /** Signed repricing of current surface content relative to the baseline anchor. */
  readonly surfaceDeltaTokens: number
  /** Non-negative current request-and-response pressure. */
  readonly totalTokens: number
  /** Total route-priced request tokens across the current surface; equals the sum of the node prices. */
  readonly surfaceTokens: number
  /** Current surface nodes in positional head-to-tail order. */
  readonly nodes: readonly TokenSurfaceNode[]
}
```

すべての測定は、有効なエンベロープのルーティング済みプロバイダー/モデルを `ctx.llm` を通じてそのルートが宣言するリクエスト画像価格に解決するので、画像の出現は、視覚トークンと、リクエストが実際に送るモデル可視テキストの合計として価格付けされる。宣言された価格を持たないルートと合成は固定ヒューリスティックを保つ。`baseline.kind === 'usage'` は、最新の成功したプロバイダー呼び出しが同じ正規リクエスト・エンベロープを持ち、その合計がその呼び出しの完全なルート価格アンカーを下回らないことを意味する。`estimated` は、再利用可能な保守的 usage アンカーが存在しないため、サービスが完全なエンベロープとサーフェス自体を価格付けしたことを意味する。後の成功したリクエストは前のアンカーを置き換え、符号付きの `surfaceDeltaTokens` は一致するアンカーに対する増減を保持し、両側を同じルートで再価格付けする。`totalTokens` はリクエストと応答の圧力のままであり、`surfaceTokens` はサーフェス専用のルート価格合計で、ノード価格の合計に等しい。

## `TokenSurfaceNode`

```ts type-equiv
/** One token-priced node in the current ordered session surface. */
interface TokenSurfaceNode {
  /** Durable sequence number of the surface event. */
  readonly seq: SessionSeq
  /**
   * Request-pressure tokens for the exact message projected by this node under
   * the measured route: image occurrences carry the route's declared visual
   * price when the routed adapter declares one, and the fixed heuristic
   * otherwise. Trigger, retention, and range selection all read this price.
   */
  readonly tokens: number
  /**
   * Fixed-heuristic tokens for the same message, independent of any route.
   * The shadow-price protocol prices replacements with this value so the O(1)
   * projection fold stays in agreement with its own appends.
   */
  readonly heuristicTokens: number
}
```

サーフェスの順序が権威である。置き換えノードは、後続の位置ノードより大きい永続 seq を持ちうる。スナップショットは不変であり、基になるリプレイ fold が進んでも成長しない。

## `ContextPressureProjection`

`ctx.tokenMeter.contextPressure(session)` は、最新のログ済み `pressureTokens` と `contextWindow` を同期的に読む。クライアント projection はサーフェスの変更から `projectedTokens` も導出する。これらの表示値は新しいリクエストの価格付けを行わない。リクエストの価格付けには `measure()` を使う。

ソース: [`packages/llm/token-meter/src/projection.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/token-meter/src/projection.ts)

```ts type-equiv
/**
 * Approximate context occupancy for a status display.
 *
 * The fields, when present, are deliberately NOT one atomic request
 * observation: each is a last-wins record of a different moment. Switching
 * models can therefore pair a fresh capacity with the previous route's
 * pressure until the next request reports usage. This is an intentional trade
 * — the value is a user-facing reference, not a billing or gating input. See
 * the token-meter README for the full rationale.
 */
interface ContextPressureProjection {
  /**
   * Provider-reported prompt size of the most recent request: uncached input
   * plus cache reads and writes. Response output is excluded, so this does not
   * grow as the current turn streams. Absent until a provider reports usage.
   */
  pressureTokens?: number
  /**
   * What the NEXT request's prompt would cost: {@link pressureTokens} plus the
   * heuristic repricing of everything the surface gained or lost since that
   * sample. Only the delta is estimated, so the figure stays anchored to the
   * provider while still reacting the moment a compaction shadows a span —
   * which `pressureTokens` alone cannot do, since compaction reports no usage
   * of its own. Absent until a provider reports usage.
   */
  projectedTokens?: number
  /** Newest recorded route capacity; absent when no adapter advertised one. */
  contextWindow?: number
}
```

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。dispatch モードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxtokenmeter--tokenmeter"></a>

### `ctx.tokenMeter` — `TokenMeter`

1 つのサービス全体の推定器と、分離されたセッションごとの fold のリプレイ・オーナー。

```ts cordis-catalog
/**
 * Read the latest logged provider prompt size and route capacity.
 * @param session - Session whose committed usage and request capacity are read.
 * @returns context pressure; fields remain absent until their source events are recorded.
 */
contextPressure(session: Session): Pick<ContextPressureProjection, 'pressureTokens' | 'contextWindow'>

/**
 * Measure current request pressure and surface through the durable tail.
 *
 * The effective envelope's routed provider/model selects the request-image
 * pricing every node is priced under: a route whose adapter declares image
 * pricing charges each retained image its visual tokens plus its
 * model-visible text, while other routes keep the fixed heuristic. Provider
 * usage is reused only when the latest successful call's canonical request
 * envelope matches `requestHeader` and its total is no lower than that
 * call's full route-priced anchor; otherwise the complete envelope and
 * surface are repriced. The anchor includes all surface nodes immediately
 * before the assistant message, including inputs admitted after step/start.
 *
 * `requestHeader` replaces the latest logged envelope for pressure and node
 * pricing; the node set always describes the current session surface. Every
 * call clones those positional nodes, so measurement is O(surface).
 *
 * @param session - session to replay through its current durable tail.
 * @param requestHeader - optional effective request envelope replacing the latest logged header.
 * @returns a detached deeply immutable pressure and surface measurement.
 */
measure(session: Session, requestHeader?: EpochHeader): TokenMeasurement

/**
 * Heuristically price one model-visible message (instance face of the pure
 * `estimateMessage` export from `estimate.ts`).
 * @param message - message to price without mutation.
 * @returns content and role-framing tokens under the fixed service heuristic.
 */
estimateMessage(message: Message): number
```

型: [EpochHeader](session.md) · [Message](llm-streaming.md) · [Session](session.md)

ソース: [`packages/llm/token-meter/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/token-meter/src/index.ts)
<!-- END GENERATED cordis-surface -->
