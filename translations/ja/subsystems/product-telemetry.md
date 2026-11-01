# 製品テレメトリー

日本語 | [English](../../../docs/subsystems/product-telemetry.md) | [中文](../../../docs/subsystems/product-telemetry.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/product-telemetry.md](../../../docs/subsystems/product-telemetry.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

[製品テレメトリー・プラグイン](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/host/product-telemetry-otel/README.md)は、明示的に選択された分析イベントを OTLP/HTTP で送信する。その `productTelemetry` サービスは送信だけを所有し、イベントがいつ発生するか、どのフィールドが承認されるかは製品のコンシューマーが所有する。エクスポーターは Session データや識別子を自動的に収集しない。[Desktop 分析コンシューマー](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/product-analytics/README.md)はインタラクションとライブ圧縮イベントを選択し、キャッシュされた静的環境と利用可能なログイン・アイデンティティを加え、Desktop の起動時コレクション・スイッチに従う。通常の Web クライアントは製品イベントを収集しない。

`ProductTelemetryRecord` はイベント名、文字列ボディ、Unix ミリ秒タイムスタンプを必要とする。属性はスカラーの文字列、数値、真偽値、およびそれらの値の 1 レベルのオブジェクト（`ProductTelemetryScalar`）を受け入れる。任意の severity は OTel の severity 番号を使い、デフォルトは INFO である。観測時刻はレコードがエンキューされたときに割り当てられる。

エンキューは同期的で、配信を確認しない。バッチ処理と再試行は SDK が所有し、ローカル診断がエクスポートの失敗を報告する。設定、シャットダウン、損失の制限についてはパッケージ README を参照。

## レコード型

```ts type-equiv
/** Scalar values accepted in product attributes. */
type ProductTelemetryScalar = OTelEventScalar
```

```ts type-equiv
/** Caller-selected ordinary analytics record. */
type ProductTelemetryRecord = OTelEventRecord
```

ソース: [`packages/host/product-telemetry-otel/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/host/product-telemetry-otel/src/index.ts)

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxproductanalytics--productanalytics"></a>

### `ctx.productAnalytics` — `ProductAnalytics`

認証済みのイベント受け入れ。無効化されたインスタンスはアイデンティティを検査せず、新しいイベントも受け入れない。

```ts cordis-catalog
/**
 * Read the collection policy.
 * @returns whether this Host currently accepts Desktop analytics.
 */
@Remote enabled(): boolean

/**
 * Stream the effective policy initially and after live configuration edits.
 * @param signal - subscriber lifetime.
 * @returns current policy values until cancellation or service disposal.
 */
@Remote({ mode: 'stream' }) async *watchPolicy(signal: AbortSignal): AsyncIterable<boolean>

/**
 * Submit selected Desktop fields with cached static environment and fresh identity.
 * The first intake waits for bounded environment queries; concurrent intakes share them.
 * Missing identity is omitted; failed environment fields carry fixed reasons.
 * @param event - typed product event without message contents or credentials.
 * @returns after local submission; no delivery or warehouse acknowledgement.
 */
@Remote async report(event: ProductEvent): Promise<void>
```

ソース: [`packages/client/product-analytics/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/product-analytics/src/index.ts)

<a id="ctxproducttelemetry--producttelemetry"></a>

### `ctx.productTelemetry` — `ProductTelemetry`

Host の分析送信者。マウントするだけでは何も送信せず、所有する fiber がアンロード時にそれを排出する。

```ts cordis-catalog
/**
 * Enqueue one selected product event without waiting for network delivery.
 * Queue admission and shutdown completion are not collector or warehouse acknowledgements.
 * @param record - caller-owned event containing only approved analytics fields.
 */
emit(record: ProductTelemetryRecord): void
```

ソース: [`packages/host/product-telemetry-otel/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/host/product-telemetry-otel/src/index.ts)
<!-- END GENERATED cordis-surface -->
