# セッションテレメトリ（SessionTelemetryBackend）

日本語 | [English](../../../docs/subsystems/session-telemetry.md) | [中文](../../../docs/subsystems/session-telemetry.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/session-telemetry.md](../../../docs/subsystems/session-telemetry.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

[キャプチャ・サービス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-telemetry/README.md)は正規イベントのキャプチャ、秘匿化、ハンドオフ・カーソルを所有する。[フィードバック・バックエンド](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-telemetry-otel/README.md)は Session プレフィックスを認可し、その独立したキュー、リクエストごとのスケジューリング、4,000,000 バイトの上限を所有する。レポート・データがモデル・リクエストに届くことはない。

ソース: [`packages/session/session-telemetry/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-telemetry/src/index.ts)

## 論理レコード

```ts type-equiv
/**
 * Severity of a telemetry record, pre-mapped at capture so a receiver can
 * alert with zero configuration: `error` for events whose own outcome flag
 * says so (the tool message's `isError`, `turn/end` error reasons) and for
 * `agent-error` operational records. Captured events otherwise default to
 * `info`; `warn` remains available to `session-telemetry/record` policies and
 * backends.
 */
type SessionTelemetrySeverity = 'info' | 'warn' | 'error'
```

```ts type-equiv
/**
 * One logical record handed to a backend — the capture contract's whole outbound
 * vocabulary. Ledger records mirror session-log events one-to-one;
 * operational records (`channel: 'ops'`) carry the two signals with no log
 * home (`agent-error`, `shutdown`) and deliberately omit `event.seq`-style
 * identity so they can never be mistaken for ledger rows.
 */
interface SessionTelemetryRecord {
  /** Canonical envelope without data; body carries the separately redacted payload. Absent for operational records. */
  sourceEvent?: { sessionId: SessionId; envelope: Omit<SessionEvent, 'data'> }
  /** Ledger (session-log mirror) or ops (operational signal) channel; backends keep the two under separate instrumentation scopes. */
  channel: 'ledger' | 'ops'
  /** Unix epoch milliseconds — the source event's append time for ledger records, the emission time for ops records. */
  time: number
  /** Pre-mapped alerting severity; see {@link SessionTelemetrySeverity}. */
  severity: SessionTelemetrySeverity
  /**
   * Identity attributes, deliberately minimal: ledger records carry
   * `session.id`, `session.format_version`, `event.type`, `event.seq`, plus optional
   * `session.cwd` / `session.parent_id`; a seeded Session also carries
   * `session.seed_length` from its exact inherited event count;
   * ops records carry `telemetry.op`, `session.id`, and (for `agent-error`)
   * `agent.id`, `turn`, `step`, `error.name`. Anything recoverable from the
   * body is intentionally NOT duplicated here.
   */
  attributes: Record<string, string | number>
  /**
   * The complete payload: a deep copy of the session event's `data` for
   * ledger records (JSON-serializable by `Session.append`'s own
   * validation), or the op payload for ops records. Never mutated after
   * handoff.
   */
  body: unknown
}
```

すべての正規の[セッション・イベント](session.md)は、完全なコンパクト・ストリームを伴う各 `assistant/message` や `assistant/attempt`、および seam が関知しないプラグイン・マージ型も含めて、1 つの順序付き ledger レコードとしてそのまま通過する。プロセス内の `agent/assistant-stream` フレームはこの永続フィードに入らない。新しい fork は、継承マーカーと fork クローザーを含む子所有のサフィックスから始まる。復元された Session は、復元された fork を含め、保存済みプレフィックスの後から始まる。バックエンドは `includeHistory` を選んでプレフィックス全体を含められる。同じオブジェクトを再採用すると、そのハンドオフ・カーソルの後から再開する。配送はベスト・エフォートである。カーソルは「配送済み」ではなく「ハンドオフ済み」を示し、レコードは失われたり（クラッシュ、リロードの窓）、重複したり（新しいオブジェクトでのリプレイ、SDK のリトライ）しうる。そのため受信側は ledger レコードを `(session.id, session.format_version, event.seq)` で重複排除する。ops レコードはそのアイデンティティを意図的に省略する。それらは合計する項目ではなくアラートを出すシグナルであり、重複を許容する。

## 共有の開示

すべてのバックエンドは、`ctx.sessionTelemetry` の必須の抽象メンバー `sharing` を通じて、デプロイが選択したモードを公開する（[サービス定義 README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-telemetry/README.md#the-sharing-disclosure)）。これは Session ごとの受入判断でも配送の受領証でもない。`/feedback` の確認応答はこれを参照しない。

```ts type-equiv
/**
 * Deployment-selected session-sharing mode, not confirmation of SDK delivery.
 */
type SessionTelemetrySharingStatus = 'full' | 'feedback-only' | 'disabled'
```

## キャプチャ・ポリシー

```ts type-equiv
/** Whether capture follows live events or reads the canonical log only when requested. */
type SessionTelemetryCapture = 'live' | 'on-demand'
```

```ts type-equiv
/** Backend-selected capture mode and history policy. */
interface SessionTelemetryCaptureOptions {
  /** Follow live events, or wait for explicit capture; defaults to live. */
  capture?: SessionTelemetryCapture
  /** Include inherited fork history and stored history from earlier lifecycles; defaults to false. */
  includeHistory?: boolean
}
```

`includeHistory` は保存済みレコードと継承レコードを許可するが、それ自体がキャプチャを認可するわけではない。[OTel バックエンド](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-telemetry-otel/README.md)はオンデマンド・キャプチャを使い、新たに明示的なフィードバックを必要とする。すべてのプロバイダーについて、そのフィードバックを通じてプレフィックス全体だけを解放する。

## バックエンド契約

```ts type-equiv
/**
 * The minimum backend contract the coordinator requires. {@link SessionTelemetryBackend} is
 * its service-registered form; tests compose the coordinator with a bare
 * implementation of this interface.
 */
interface SessionTelemetrySink {
  /**
   * Hand one record to the backend's pipeline. MUST be a non-blocking
   * enqueue — the coordinator calls this synchronously from the
   * `session/event` hot path or an explicit canonical-log capture, so anything
   * slower than a queue push would tax the agent loop or feedback handling.
   * Errors thrown here are contained by the coordinator and logged; they
   * never reach the loop.
   * @param record - the logical record to report; owned by the backend after the call.
   */
  emit(record: SessionTelemetryRecord): void
  /**
   * Optional hint that a turn ended. A backend may forward it to its SDK's
   * flush so records are exported after each turn. Called
   * fire-and-forget; implementations must not block and must not throw
   * meaningfully (the coordinator contains exceptions). Most backends should
   * leave this unimplemented and let their SDK's own batching cadence govern
   * export timing: a backend that does implement it owns the interaction
   * between its concurrent flushes and {@link shutdown}'s drain (the OTel
   * backend leaves it unimplemented for exactly that hazard — see the
   * revival Agent Note).
   */
  flush?(): void
  /**
   * Forward the fiber's disposal to the SDK: flush whatever is queued and
   * reach quiescence, per the SDK's own shutdown contract. Everything
   * emitted before this call must still be delivered — including records
   * enqueued while a {@link flush} hint is in flight, so a backend whose SDK
   * guards against concurrent flushes orders behind the outstanding one (the
   * coordinator emits its dispose-time `shutdown` markers immediately before
   * calling this). Awaited by the coordinator's dispose; a rejection is
   * logged as a warning and never fails application teardown.
   * The coordinator captures dispose-time shutdown markers immediately before
   * this call for live capture; on-demand capture creates no ops records.
   * @returns resolves when the backend's pipeline has quiesced.
   */
  shutdown(): Promise<void>
}
```

`SessionTelemetryBackend`（`ctx.sessionTelemetry`、[シグネチャ](#ctxsessiontelemetry--sessiontelemetrybackend-abstract-seam)）は契約のロード可能な形である。コンテキストごとに 1 実装で、重複してロードすると例外を投げる。バックエンドはコンストラクターで seam の `SessionTelemetryCoordinator` を合成し、キャプチャ側を設置する。

## 秘匿化ウォーターフォール: `session-telemetry/record`

すべてのレコードは、正規イベントのコピーと `emit()` の間で `session-telemetry/record` [ウォーターフォール](../cordis-primer.md#cordis-waterfall-semantics)を通過する（[イベント項目](#session-telemetryrecord--waterfall)）。seam 自体はルールを一切提供しない。リスナーが 1 つもマウントされていなければ、レコードはキャプチャされたままバックエンドに届くので、エクスポートされるデータの清浄さはデプロイがマウントするルールと正確に一致する。リスナーは `next()` の戻り値を変換することで積み重なる。`next()` を呼ばずに戻ると、その下のすべてを置き換える。例外を投げるリスナーは、コーディネーターの封じ込めの中で、その 1 レコードを fail-closed で保留する。秘匿化はエクスポート用コピーにのみ適用され、正規のセッション・ログが書き換えられることはない。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。dispatch モードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxsessiontelemetry--sessiontelemetrybackend-abstract-seam"></a>

### `ctx.sessionTelemetry` — `SessionTelemetryBackend` (抽象 seam)

バックエンド契約のロード可能な形。コンテキストごとに 1 実装で、`telemetry` キー配下の cordis `Service` 登録は重複時に例外を投げる（cordis の標準挙動）。バックエンドはコンストラクターで SessionTelemetryCoordinator を合成し、キャプチャ側を設置する。

```ts cordis-catalog
/**
 * See {@link SessionTelemetrySink.emit} — that declaration is the contract's one home.
 * @param record - the logical record to report; owned by the backend after the call.
 */
abstract emit(record: SessionTelemetryRecord): void

/** See {@link SessionTelemetrySink.flush}. */
flush?(): void

/**
 * See {@link SessionTelemetrySink.shutdown}.
 * @returns resolves when the backend's pipeline has quiesced.
 */
abstract shutdown(): Promise<void>
```

ソース: [`packages/session/session-telemetry/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-telemetry/src/index.ts)

<a id="session-telemetry-events"></a>

### `session-telemetry/*` イベント

<a id="session-telemetryrecord--waterfall"></a>

#### `session-telemetry/record` — waterfall

バックエンドに届く前に、送信される 1 レコードを変換する。このウォーターフォールはサービス定義の秘匿化拡張点である。seam 自体はルールを一切提供しない。最内の `next()` はレコードをそのまま通し、リスナーがマウントされていなければレコードはキャプチャされたままバックエンドに届くので、エクスポートされるデータの清浄さはデプロイがマウントするルールと正確に一致する。リスナーは `next()` の戻り値を変換することで積み重なる。`next()` を呼ばずに戻ると、その下のすべてを置き換える。コーディネーターの封じ込めの中でキャプチャのホット・パス上で同期的に dispatch される。例外を投げるリスナーはその 1 レコードを保留し（fail-closed）、エージェント・ループには決して届かない。ライブ・キャプチャは追記時に dispatch し、オンデマンド・キャプチャは正規ログを読む間に dispatch する。秘匿化はエクスポート用コピーにのみ適用され、正規のセッション・ログが書き換えられることはない。

```ts cordis-catalog
/**
 * Transform one outbound record before it reaches the backend. This
 * waterfall is the Service Definition's redaction extension point. It ships NO rules
 * of its own: the
 * innermost `next()` passes the record through unchanged, and with no
 * listener mounted records reach the backend as captured, so exported
 * data is exactly as clean as the rules a deployment mounts. Listeners
 * stack by transforming `next()`'s return value; returning without
 * `next()` replaces everything beneath. Dispatched synchronously on the
 * capture hot path inside the coordinator's containment: a throwing
 * listener withholds that one record (fail-closed) and never reaches the
 * agent loop. Live capture dispatches at append time; on-demand capture
 * dispatches while reading the canonical log. Redaction applies to the
 * exported copy only; the canonical session log is never rewritten.
 * @param record - the candidate record, already the coordinator's own deep
 *   copy; listeners return a (possibly new) record and must not mutate it.
 * @mode waterfall
 */
'session-telemetry/record'(record: SessionTelemetryRecord, next: () => SessionTelemetryRecord): SessionTelemetryRecord
```

ソース: [`packages/session/session-telemetry/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-telemetry/src/index.ts)
<!-- END GENERATED cordis-surface -->
