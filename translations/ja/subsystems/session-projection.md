# セッション投影

日本語 | [English](../../../docs/subsystems/session-projection.md) | [中文](../../../docs/subsystems/session-projection.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/session-projection.md](../../../docs/subsystems/session-projection.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

セッション投影 seam — [capability seam](../capability-seams.md) — は、ドメインの Host プラグインが、ログ由来のセッションごとの状態の現在の完全な値をクライアント側のキャリアへ提供する経路である。Service Definition と registry（[dsh-session-projection](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/session/session-projection)、`ctx.sessionProjections`）、ドメインの貢献者（それぞれが 1 つの純粋な unit を登録する）、キャリア（[dsh-session-controller](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/api/session-controller) の履歴テールページと `session/projection` push フレーム）からなる。Host の状態読み取り者はアクティベーション中に registry を要求する。フレームワークが駆動し、ドメインが計算する。registry は `session/event` を 1 回購読し、コミットされたすべてのイベントをすべての unit で畳み込む。ドメインは購読を持たず、クライアントはドメインイベントを決して畳み込まない — 完成した値を受け取る。設計の権威: [session-projection RFC](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/proposed/architecture/2026-07-27-session-projection-and-command-log.md)。drive/cache/feed の契約: [パッケージ README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-projection/README.md)。

ソース: [`packages/session/session-projection/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-projection/src/index.ts)

`ProjectionHandle` は登録解除後も同期リーダーを保持する。`stateOf(session)` は新しくコミットされたイベントを元の unit で畳み込む。グローバルスナップショット、checkpoint、通知に再参加することはない。登録自体はハンドルを使用可能になる前に完了している必要がある。[登録ライフサイクル](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-projection/README.md)を参照。

```ts type-equiv
/** An effect-owned registration with a reader retained independently of registration. */
interface ProjectionHandle<S> {
  /**
   * Read the current state, including after unregistering, by folding new
   * Session events with this handle's original unit. Do not mutate the live value.
   * @param session - Session whose committed events are folded.
   * @returns the projection state at the Session's current cursor.
   */
  stateOf(session: Session): S
  /**
   * Create a reader for one exact Session without reading state or preparing it.
   * The reader retains this unit after unregistration and owns no disposal.
   * @param session - Session whose committed events the bound reader folds.
   * @returns a synchronous callback with the same state and failures as `stateOf(session)`.
   */
  readerFor(session: Session): () => S
  /**
   * Unregister from global drives and snapshots without invalidating this reader.
   * @returns a promise that resolves when this registration is released.
   */
  dispose(): Promise<void>
}
```

[互換性ルール](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-projection/README.md#deprecated-plugin-compatibility)は、レガシーの破棄、キー付き読み取り、およびキーごとに 1 回の非推奨警告を定義する。

`ProjectionHandle.readerFor(session)` は 1 つの正確な Session に対する `() => S` コールバックを返す。呼び出すと元の unit を読み取り、借用状態、準備完了要件、同期的な失敗を保持する。所有者は複数のリーダーを型付きオブジェクトに合成できる。ドメインサービスはそれらをプライベートに保つ。[複数の状態の束縛](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-projection/README.md#bind-several-states-to-one-session)を参照。

## 計算単位（unit）

`ProjectionDefinition<K, S>` は各 unit の Host 状態型をローカルに保持する。登録は `stateSchema` から `S` を推論し、`ProjectionHandle<S>` を返す。クライアント可視のキーだけが `SessionProjectionMap` にマージされ、その全体の値を型付けする。`wire` ブロックは unit をクライアント可視にし、レンダリングは slot システムに属し、決してこの層には属さない。

```ts type-equiv
/**
 * One domain's state-driven computation unit: a pure synchronous fold plus
 * declarations and an optional client view — never an opaque getter. The framework drives
 * `apply` on every committed session event; the domain holds no
 * subscriptions and owns only the computation. All functions MUST be
 * synchronous (an async unit would tear the carriers' consistency cut), and
 * `state` MUST be plain JSON (the persisted-cache precondition).
 */
interface ProjectionDefinition<K extends string, S> {
  /** The projection key this unit owns. */
  key: K
  /** Validates persisted state before it seeds a fold. */
  stateSchema: ZodType<S>
  /**
   * State for the empty log and its immutable Session metadata.
   * @param header - immutable metadata for the Session being projected.
   * @param inheritedEventCount - exact fork-inherited prefix length.
   * @returns the initial state.
   */
  init(header: SessionHeader, inheritedEventCount: SessionLogOffset): NoInfer<S>
  /**
   * Pure transition: previous state + one committed event → next state. A
   * unit uninterested in an event MUST return the same state reference — an
   * unchanged reference (`Object.is`) produces zero downstream work.
   * @param state - the state covering all prior events.
   * @param event - the next committed session event.
   * @returns the next state (same reference when the event is not the unit's).
   */
  apply(state: NoInfer<S>, event: SessionEvent): NoInfer<S>
  /** Client view. Omit for host-only units. */
  wire?: K extends keyof SessionProjectionMap ? {
    /** Validates the wire payload before it leaves the host. */
    viewSchema: ZodType<SessionProjectionViewValue<SessionProjectionMap[K]>>
    /**
     * State → wire payload (the read-side projection). The live drive keeps
     * the two latest raw results and compares them with `Object.is`; an
     * object-valued view must reuse its reference to suppress publication
     * across internal-only state changes.
     * @param state - the current state.
     * @returns the whole current value for this unit's key.
     */
    view(state: NoInfer<S>): SessionProjectionViewValue<SessionProjectionMap[K]>
    /**
     * Whether a Session summary carries this unit; omitted means `true`.
     * Declare `false` on a key whose map entry carries the
     * `SessionProjectionSummaryExcluded` mark: the runtime key set follows this
     * field, while the client view type follows the mark.
     * @default true
     */
    summary?: boolean
  } : never
  /**
   * Persisted-cache invalidation version: bump whenever the serialized state fields or the
   * fold semantics change, so persisted `(sessionId, key, ver, seq, val)`
   * rows from an older unit are discarded instead of being forward-applied
   * into garbage. Non-negative integer.
   */
  stateVersion: number
}
```

提供されるすべての projection 値は完全な読み取りモデルである。ソースイベントは全体の値またはドメイン所有の操作を運ぶことがあり、unit の決定的な `apply` がリプレイを所有し、checkpoint と前方テールのリプレイが同じ状態を再構築する。

## スナップショットと変更フィード

```ts type-equiv
/**
 * One consistent read cut over every registered client-visible unit for one session.
 * `asOfSeq` is the shared watermark — the seq of the last event every value
 * reflects (`-1` for an empty log).
 */
interface ProjectionSnapshot {
  /** Seq of the last event the values reflect; -1 for an empty log. */
  asOfSeq: SessionSeqCursor
  /** Whole current client value per registered key. */
  values: Partial<SessionProjectionMap>
}
```

```ts type-equiv
/**
 * Change-feed listener: one unit's raw `view` result changed by `Object.is`
 * for one session. `value` is the schema-validated output; `seq` is the
 * unit's watermark at emission (the seq of the event that caused the change).
 */
type ProjectionChangeListener = (
  session: Session,
  key: Extract<keyof SessionProjectionMap, string>,
  value: unknown,
  seq: SessionSeq,
) => void
```

`snapshot(session)` は完全に同期的である。キャリアは自身のページスライスと同じティックでそれを読むため、`asOfSeq` は両方の読み取りを 1 つのシーケンス番号で覆う。クライアントビューだけを返し、すべての値は返す前に自身の unit の `viewSchema` を通る。`handle.stateOf(session)` は無関係なビューを計算せずに 1 つのライブな Host 状態を読む。呼び出し元は借用した参照を変更してはならない。状態参照が変わると 1 つのキャッシュされた生のビューが計算され、変更フィードはその結果が `Object.is` で変わったときだけ発火する。オブジェクト値のビューは、内部のみの状態変更にわたって公開を抑制するために自身の参照を保持しなければならない。

## サマリー読み取り

Session のサマリーは、`SessionProjectionMap` のエントリが `SessionProjectionSummaryExcluded<T>` マークを持つキーを除く、登録済みのすべてのクライアントビューを運ぶ。マークには実行時の表現がない。クライアントビューの型を決める一方、`wire.summary` が実行時のキー集合を決め、省略は `true` を意味する。マークを持つエントリのキーは両者が一致するように `wire.summary: false` を宣言する。フィールドは単なる boolean であるため、型はそれを強制しない。

`summaryKeys()` はライブの登録からキー集合を導出し、登録またはその破棄がキーテーブルを変えるまで同じインスタンスを返し続ける。サマリー consumer はそれを既存のキー指定を受け付ける読み取り面 — キャッシュされたセルには `cachedSnapshot(session, keys)`、checkpoint 行には `viewCheckpoint(rows, keys)` — に渡す。どちらも自身のウォーターマーク規則を保ち、コピーせずに集合を使うため、サマリー専用の読み取り面も、集合の 2 つ目の定義も存在しない。

リスト行はそのサマリー投影を運び、完全な投影は決して運ばない。サマリーのキー集合だけを約束し、それ以上は約束しない。`snapshot`、`cachedSnapshot`、`viewCheckpoint` はキー部分集合なしで呼ばれたとき完全なままであり、開かれた Session、そのフォローベースライン、制御ベースラインがすべての projection 値を保つのはこのためである。`@` 参照は同じキー指定を受け付ける読み取り面を通じて自身のラベルキーだけを読む。

### サマリーの語彙

以下の宣言はサマリー規則を述べる。マーク、その剥離、除外キーのテスト、サマリー値の型は純粋型の出口で宣言される。両方の読み取り面が受け入れるキー選択はサービスモジュールで宣言される。

```ts type-equiv
/**
 * One `SessionProjectionMap` entry excluded from Session summary payloads
 * (the Session list row and any other summary read). The mark is type-only;
 * `T` keeps its exact runtime value and wire representation. Wrap only the
 * excluded contributor's own map entry, and place a nullable entry's `null`
 * outside the wrapper (`SessionProjectionSummaryExcluded<Item[]> | null`) so
 * the required symbol cannot absorb that member.
 */
type SessionProjectionSummaryExcluded<T> = T & {
  readonly [sessionProjectionSummaryExcluded]: true
}
```

```ts type-equiv
/**
 * One map entry's client view value: the summary mark stripped. The
 * conditional distributes over a nullable entry's union, so a `| null` member
 * written outside the wrapper survives unchanged.
 */
type SessionProjectionViewValue<T> =
  T extends SessionProjectionSummaryExcluded<infer Value> ? Value : T
```

```ts type-equiv
/** Every client view value, keyed by projection key, with no summary mark. */
type SessionProjectionViewValues = {
  [K in keyof SessionProjectionMap]: SessionProjectionViewValue<SessionProjectionMap[K]>
}
```

```ts type-equiv
/**
 * Keys a Session summary omits: map entries whose non-nullish value carries
 * the summary mark. Nullish members are ignored, so a nullable entry is still
 * recognized when its `null` lives outside the wrapper.
 */
type SessionProjectionSummaryExcludedKey = {
  [K in keyof SessionProjectionMap]:
  NonNullable<SessionProjectionMap[K]> extends SessionProjectionSummaryExcluded<unknown> ? K : never
}[keyof SessionProjectionMap]
```

```ts type-equiv
/**
 * The client values a Session summary carries: every client view value except
 * {@link SessionProjectionSummaryExcludedKey}. A summary read returns these
 * keys only; an opened Session's full snapshot keeps every view value.
 */
type SessionProjectionSummaryValues =
  Omit<SessionProjectionViewValues, SessionProjectionSummaryExcludedKey>
```

```ts type-equiv
/**
 * One read's client-visible key selection: an explicit key list, or a derived
 * set such as {@link SessionProjectionRegistry.summaryKeys}. A set is used as
 * given; only a list is copied into one.
 */
type ProjectionKeySelection =
  | readonly Extract<keyof SessionProjectionMap, string>[]
  | ReadonlySet<Extract<keyof SessionProjectionMap, string>>
```

## registry: `ctx.sessionProjections`

`SessionProjectionRegistry`（[シグネチャ](#ctxsessionprojections--sessionprojectionregistry)）が drive を所有する。1 つの `session/event` 購読、登録済みのすべての unit に対する先行 `apply`、セッションごと・unit ごとのウォーターマークセルである。セルは遅延して構築される — イベントが流れた後に登録された unit、または registry より古い Session は、最初の接触（イベントまたは読み取り）でインメモリログに対して `init` を畳み込む。ドメインはアクティベーション中に `register(definition)` を待機し、そのハンドルを保持する。登録は呼び出し元のファイバーに属する。最後の解放がグローバル drive と snapshot からキーを削除する一方、保持されたハンドルは読み取り可能なままである。同じキー・同じバージョンの登録者は 1 つの unit を共有し、互換性のないバージョンは登録を拒否する。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxsessionprojectioncache--sessionprojectioncache"></a>

### `ctx.sessionProjectionCache` — `SessionProjectionCache`

永続化された projection キャッシュサービス。初期化時に `session_projcache` ドメインを開き、ライブ Session をスロットルされた write-behind（Config の count/interval トリガー）と 3 つの必須時点 — Session 作成、`turn/end`、Session 破棄（live から cold への瞬間） — で checkpoint し、Session ヘッダーに対するキャッシュ行を提供する。すべての永続書き込みは fail-soft である。失敗は警告をログに記録し、キャッシュは次の書き込みで自己修復する。

```ts cordis-catalog
/**
 * The zero-I/O listing read: whole values viewed straight from the stored
 * rows (version-matching keys only) of the record bound to the caller's
 * lifecycle. The header is the only identity witness a listing holds, so
 * this face matches the lifecycle identity (`formatVersion`, `createdAt`,
 * `cwd`, `isSeeded`) and not the inherited cut: within one format
 * generation the cut is fixed at fork time, so it distinguishes no
 * lifecycle the other fields do not, and a viewed value never seeds a fold.
 * The view is as stale as the last durable checkpoint but never wrong and
 * never from an unrelated log. Its `asOfSeq` is the lowest watermark among
 * the served rows: the stored record's own position, which the header
 * cannot relate to the log the caller later opens. The Session list
 * therefore labels the block as cached, and the client lets every value the
 * connected Session produces supersede it whatever this number says.
 * @param meta - the listed session's header (identity witness; no log read).
 * @param keys - optional projection keys required by the caller's audience.
 * @returns the viewed block, or `undefined` when no usable row exists for
 *   this lifecycle at the current Session format.
 */
cachedSnapshot( meta: SessionHeader, keys?: ProjectionKeySelection, ): ProjectionSnapshot | undefined

/**
 * Read only a predecessor checkpoint's title as a zero-I/O listing hint.
 *
 * The authoritative Session header supplies the lifecycle identity. A cache
 * checkpoint can lag that log but cannot lead it because writes flush the
 * log first, so a matching predecessor title is a genuine (possibly stale)
 * fact from this Session. The registry still requires the current title
 * projection's row version and schema. No other predecessor projection is
 * exposed: format normalization can change their current meaning, and the
 * {@link cachedSnapshot} / hydration paths continue to reject them.
 * @param meta - authoritative listed Session header.
 * @returns a title-only block at the stored title row's watermark, or
 *   `undefined` when the record is current, newer, unrelated, missing, or
 *   incompatible with the title unit.
 */
cachedPredecessorTitle(meta: SessionHeader): ProjectionSnapshot | undefined

/**
 * Hydrate projection cells for an already-prepared Session without another
 * persistence read. The cache seeds matching rows; the supplied exact log
 * advances every unit to the observation cut. No checkpoint is written
 * because the logical observation may contain recovery events not yet durable.
 * @param session - exact unpublished Session retained by persistence.
 * @param events - exact logical event prefix represented by the observation.
 * @returns all projection values at the event cut.
 */
hydratePrepared( session: Session, events: readonly SessionEvent[], ): ProjectionSnapshot

/**
 * Durably checkpoint one live session NOW (all mandatory points call
 * this; tests and carriers may too). The registry cut is snapshotted at
 * this boundary (states are live references), then the session's record is
 * replaced on the domain's write chain. NOT fail-soft — callers on the
 * fail-soft paths contain it.
 * @param session - the live session to checkpoint.
 * @returns resolution after durability and event emission.
 */
async write(session: Session): Promise<void>

/**
 * Cold-read one session's projections from its complete log. Each unit is
 * seeded from the identity-checked cached rows — the registry skips `apply`
 * for the already-folded prefix (events at or below the row's `seq`) — and
 * the refreshed checkpoint is written back (fail-soft, fire-and-forget), so
 * the first cold read creates the cache row and later ones seed from it.
 * The caller supplies the complete log in seq order: this service never
 * consults the persistence layer.
 * @param meta - the stored session header (identity witness).
 * @param inheritedEventCount - exact inherited prefix length for projection initialization and identity.
 * @param events - the session's complete log, in seq order.
 * @returns the projection cut at the log end.
 */
coldSnapshot( meta: SessionHeader, inheritedEventCount: SessionLogOffset, events: readonly SessionEvent[], ): ProjectionSnapshot
```

型: [Session](session.md) · [SessionEvent](session.md) · [SessionHeader](persistence.md) · [SessionLogOffset](session.md)

ソース: [`packages/session/session-projection-cache/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-projection-cache/src/index.ts)

<a id="ctxsessionprojections--sessionprojectionregistry"></a>

### `ctx.sessionProjections` — `SessionProjectionRegistry`

`ctx.sessionProjections`: projection unit のテーブルとその drive。サービスは `session/event` を 1 回購読し、コミットされたすべてのイベントが登録済みのすべての unit の `apply` を通る（先行 drive）。状態参照が変わると次のクライアントビューが計算され、変更フィードはその生の結果が `Object.is` で変わったときだけ通知される。セルは遅延して構築される — イベントが流れた後に登録された unit、または registry より古い Session は、最初の接触（イベントまたは読み取り）でインメモリログに対して `init` を畳み込む。consumer は `sessionProjections` を必須の注入として宣言し、アクティベーション中に登録を待機し、準備完了のハンドルを保持する。キーを共有する登録者は 1 つの unit を共有し、カウントされる。最後の解放が unit をライブの drive、checkpoint、クライアント snapshot から削除する。保持されたハンドルは、元の unit を使って要求に応じて新しいイベントを畳み込み続ける。

```ts cordis-catalog
/**
 * Register one domain's unit on the calling context's fiber. Fiber or handle
 * disposal releases one reference; the final release removes live participation.
 * Retained handles remain readable after release.
 * Callers must await registration before publishing consumers; registration may perform asynchronous work.
 * @param definition - key, state schema, pure unit functions, and stateVersion.
 * @returns an awaitable yielding a ready handle; the callable runtime value is deprecated.
 */
register<K extends keyof SessionProjectionMap, S>( definition: Omit<ProjectionDefinition<K, S>, 'wire'> & { wire: NonNullable<ProjectionDefinition<K, S>['wire']> }, ): Promise<ProjectionHandle<S>>

/**
 * Register one host-only unit. Its state is omitted from client snapshots
 * and always checkpointed like every other unit.
 * Callers must await registration before publishing consumers; registration may perform asynchronous work.
 * @param definition - key, state schema, pure unit functions, and stateVersion.
 * @returns an awaitable yielding a ready handle; the callable runtime value is deprecated.
 */
register<K extends string, S>( definition: ProjectionDefinition<K, S> & { key: Exclude<K, keyof SessionProjectionMap> }, ): Promise<ProjectionHandle<S>>

/**
 * Subscribe to the change feed. The registration is an effect on the
 * calling context's fiber.
 * @param listener - called once per client-visible unit whose raw view changed by `Object.is`, per committed event.
 * @returns the exact disposer that unsubscribes.
 */
onChanged(listener: ProjectionChangeListener): () => void

/**
 * One consistent cut over every registered client-visible unit for one session, read from
 * the watermark cache (missing cells fold lazily over the in-memory log).
 * Fully synchronous — every value and `asOfSeq` reflect the same log
 * position. Each value passes its unit's `viewSchema` before leaving.
 * @param session - the session whose projection values are read.
 * @param keys - optional client-visible outputs; state materialization remains complete.
 * @returns the snapshot; `values` is empty when no selected client-visible unit is registered.
 */
snapshot( session: Session, keys?: ProjectionKeySelection, ): ProjectionSnapshot

/**
 * Read only already-materialized client-visible cells without folding history.
 * Values may trail the live Session and are therefore hints, not a complete
 * baseline. Missing cells are omitted.
 * @param session - attached Session whose cached cells are inspected.
 * @param keys - optional wire keys to view.
 * @returns the lowest common cached cut, or `undefined` when no wire cell exists.
 */
cachedSnapshot( session: Session, keys?: ProjectionKeySelection, ): ProjectionSnapshot | undefined

/**
 * The keys a Session summary carries: every registered client-visible key
 * whose unit does not declare `summary: false`. The returned set is stable —
 * the same instance on every call — until a registration or its disposal
 * changes the key table, so a consumer may hold it across rows. Callers pass
 * it to a key-taking read face such as `cachedSnapshot(session, summaryKeys())`
 * or `viewCheckpoint(rows, summaryKeys())`; those faces use the set as-is.
 * @returns the current summary key set; do not mutate it.
 */
summaryKeys(): ReadonlySet<Extract<keyof SessionProjectionMap, string>>

/**
 * State-level checkpoint of every persisted unit for one session, read
 * from the watermark cache (missing cells fold lazily over the in-memory
 * log). This is the write side of the persisted projection cache: the
 * returned rows are the `(key → {ver, seq, val})` part of the durable
 * `(sessionId, key, ver, seq, val)`
 * rows. Every `val` is a DETACHED structured clone — never the live
 * cell reference: the watermark cache is this registry's authoritative
 * mutable state, and a caller reaching the live reference could corrupt
 * every subsequent snapshot and frame through it (plain JSON by the unit
 * contract, so the clone is total).
 * @param session - the session whose unit states are checkpointed.
 * @returns one row per registered key.
 */
checkpoint(session: Session): ProjectionCheckpoint

/**
 * The stored seq a {@link restore} tail read over `checkpoint` must start
 * at: one event BELOW the lowest usable watermark (a row is usable when
 * its `ver` matches the live unit's `stateVersion`; an absent or mismatched row
 * pulls the floor to `0` — that key must refold the full log). The
 * one-below anchor is load-bearing: the tail then proves how far the
 * stored log still extends, so {@link restore} can detect a log that
 * shrank below a row's watermark (crash-repair truncation) instead of
 * serving the stale row as current — an empty tail read from the anchor
 * yields an end below every watermark and the restore rejects for a full
 * re-read.
 * @param checkpoint - persisted rows for one session (possibly stale or empty).
 * @returns the offset for the stored-log suffix read (`SessionHandle.read`),
 *   or `undefined` when no unit is registered (no read needed —
 *   {@link restore} would serve empty values regardless).
 */
restoreFloor(checkpoint: ProjectionCheckpoint): SessionLogOffset | undefined

/**
 * View a checkpoint's rows without any log read: for every registered
 * client-visible unit whose row's `ver` matches, serve the schema-validated
 * `view` of the schema-validated stored state; mismatched, malformed, or absent rows leave their key
 * absent (a cold or listing consumer treats it as not-yet-available and a
 * fuller read path refolds it). The zero-I/O rung of the read ladder —
 * values are as stale as their rows, never wrong.
 * @param checkpoint - persisted rows for one session (possibly stale or empty).
 * @param keys - optional wire keys to view.
 * @returns whole values per key with a usable row; empty when none.
 */
viewCheckpoint( checkpoint: ProjectionCheckpoint, keys?: ProjectionKeySelection, ): Partial<SessionProjectionMap>

/**
 * Cold read: fold every persisted unit over a stored log suffix, seeding
 * each from its checkpoint row when usable — the one read recipe (cached
 * state + forward tail replay + `view`) applied without a live `Session`.
 * Call with the stored events at or past `restoreFloor(checkpoint)` (a
 * `SessionHandle.read` slice) and that same floor as
 * `baseSeq`; the floor's one-below anchor makes the supplied end honest,
 * so a shrunk log is detected here. A row is usable iff its
 * `ver` matches the live unit's `stateVersion`, it does not predate `baseSeq`
 * (`seq >= baseSeq - 1`), and it does not claim events past the
 * supplied end (`seq <= endSeq`); an unusable row is discarded
 * and its key refolds from `init` — which is only sound over the full
 * log, so a discarded row with `baseSeq > 0` throws (the caller re-reads
 * from seq 0, e.g. after a crash-repair truncation shrank the log below
 * a row's watermark).
 * @param checkpoint - persisted rows for one session (possibly stale or empty).
 * @param events - the stored events with `seq >= baseSeq`, in seq order.
 * @param baseSeq - the seq `events` starts at (its first event's seq when non-empty).
 * @param header - immutable metadata for the Session being restored.
 * @param inheritedEventCount - exact fork-inherited prefix length supplied to unit initialization.
 * @returns the snapshot cut at the supplied log end (`asOfSeq` is the last
 *   supplied event's seq, `baseSeq - 1` for an empty tail) plus the
 *   refreshed checkpoint rows at that cut, ready for a durable write-back.
 */
restore( checkpoint: ProjectionCheckpoint, events: readonly SessionEvent[], baseSeq: SessionLogOffset, header: SessionHeader, inheritedEventCount: SessionLogOffset, ): { snapshot: ProjectionSnapshot; checkpoint: ProjectionCheckpoint }

/**
 * Restore an exact cut and install its states on the supplied prepared Session.
 * A later publication reuses these cells; ordinary live reads and event drive
 * advance any constructor-owned suffix exactly once.
 * @param session - exact prepared Session that owns the restored log prefix.
 * @param checkpoint - persisted rows for this Session lifecycle.
 * @param events - exact events at the observation cut.
 * @param baseSeq - first supplied event sequence.
 * @returns all projection values at the supplied cut.
 */
hydrate( session: Session, checkpoint: ProjectionCheckpoint, events: readonly SessionEvent[], baseSeq: SessionLogOffset, ): ProjectionSnapshot
```

型: [Session](session.md) · [SessionEvent](session.md) · [SessionHeader](persistence.md) · [SessionLogOffset](session.md)

ソース: [`packages/session/session-projection/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-projection/src/index.ts)
<!-- END GENERATED cordis-surface -->
