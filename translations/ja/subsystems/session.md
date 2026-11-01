# セッション

日本語 | [English](../../../docs/subsystems/session.md) | [中文](../../../docs/subsystems/session.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/session.md](../../../docs/subsystems/session.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

[dsh-session](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/core/session) のインメモリのイベントソーシングモデル。`Session` は型付き `SessionEvent` の**追記専用ログ**であり、agent の相互作用履歴全体に対する唯一の真実の源である。LLM のメッセージ履歴はログから*派生*するもので、別に保存されることはなく、リプレイは同じイベントからの再派生である。ログを**永続**にする方法（永続化 seam、バックエンド、クラッシュ復旧）は [persistence.md](persistence.md) で扱う姉妹の関心事である。

ソース: [`packages/core/session/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/session/src/types.ts)

## `SessionEventMap` — イベント語彙

追記専用のイベント型。マージ拡張可能であり、プラグインは宣言マージで追加のイベント型を宣言する。たとえば [compaction seam](compaction.md) は `compaction/start` / `compaction/summary` / `compaction/end` を追加し、`@deepseek-ai/dsh-hook-protocol` は hook ブリッジ用のログ専用レコード `hook/invoked` / `hook/result` を追加する。`compaction/*` と同様、これらは `SurfaceEventType` では**ない**（`surfaceOp` を持たない）。生成された [永続化ログのイベントカタログ](../../../docs/persistence-catalog.md) は、core とマージ済みのすべてのメンバーを、ペイロード、サーフェスバッジ、宣言箇所とともに列挙する。

```ts type-equiv
/** A user-role specialization of the shared message representation. */
interface UserMessage extends MessageBase {
  readonly role: 'user'
}
```

```ts type-equiv
/**
 * The merge-extensible, append-only source of truth for an agent interaction.
 * Message history is derived from this log. Every event is lossless JSON and
 * sequence numbers stay contiguous. Assistant attempt events embed their exact
 * compact raw streams so persistence stores one durable settlement per attempt.
 */
interface SessionEventMap {
  /**
   * Opens turn `turn` before the loop claims queued input or runs pre-step.
   * Rejection, empty input, cancellation, or failure may close it with no
   * step; otherwise the following identified `user/message` event or batch
   * records the messages entering the step.
   */
  'turn/start': { turn: number }
  /**
   * Closes turn `turn` with the {@link TurnEndReason} that ended it. A turn
   * with no entered step has no `step/start` or `step/end`. The loop does not await a
   * flush at turn boundaries: `dsh-session-checkpoint-policy` owns the
   * per-request durability checkpoint, and consumers that read storage after
   * `whenIdle()` flush themselves. Success commits the turn; rejection is
   * reported live and does not prevent later work.
   */
  'turn/end': { turn: number; reason: TurnEndReason }
  /** Opens step `step` of turn `turn` — one model call plus the tool executions it requested. */
  'step/start': { turn: number; step: number }
  /** Closes step `step` of turn `turn`. */
  'step/end': { turn: number; step: number }
  /**
   * A user-role message on the model-visible surface: a direct human prompt
   * (the queued message claimed for this turn), a synthetic `agent.inject()`
   * context (file-change notices, subdir AGENTS.md, skill content, cron
   * notifications, …), or an entered goal continuation round. All three
   * project their `content` verbatim; `source` tells them apart.
   */
  'user/message': UserMessage
  /** An incremental agent session change admitted at the named turn and step. */
  'developer/message': {
    turn: number
    step: number
    message: DeveloperMessage
    /** Earlier request/header defining every tool addition; required exactly when additions are present. */
    headerSeq?: SessionSeq
  }
  /**
   * The rendered system prompt on the model-visible surface. The loop appends
   * the first one as surface node 0 before the step's first `user/message`.
   * A prepared in-history route can append nonempty changes in a continuing
   * series. An incapable route, or a new series whose effective text differs,
   * normalizes text to the first system node; a series start whose latest
   * surviving node already holds the rendering leaves it in place.
   * Normalization empties nonempty later nodes, then rewrites the head if
   * needed, through logged per-node replacements. An empty rendering always
   * clears all active system nodes, leaving no older instructions model-visible.
   * Empty later nodes are dormant and project to no message; an empty head with
   * no active later node records "no system prompt". Restored nonempty text follows
   * the same route and series rule; empty nodes never restore older text.
   */
  'system/message': { turn: number; step: number; message: SystemMessage }
  /**
   * Assembled assistant message for one step (derived history uses this).
   * Carries the step's `usage` when the adapter reported token accounting, so
   * the model output and its accounting travel together (there is no separate
   * usage record). `usage` is absent when the adapter reported none. A turn
   * cancelled mid-stream finalizes its delivered text/reasoning prefix as this
   * event with `interrupted: true`; undispatched tool calls are absent. The
   * marker distinguishes that prefix without re-deriving interruption from turn
   * boundaries. An aborted turn with no such event streamed no visible content.
   */
  'assistant/message': {
    turn: number
    step: number
    message: AssistantMessage
    /** Exact timed model stream, compacted without joining delta boundaries. */
    stream: AssistantStreamRecord[]
    usage?: TokenUsage
    interrupted?: true
  }
  /**
   * One model attempt that committed no surface message. The embedded stream
   * preserves a failed, retried, cancelled, or stream-error attempt that
   * reached settlement without fabricating model-visible history.
   */
  'assistant/attempt': { turn: number; step: number; stream: AssistantStreamRecord[] }
  /**
   * The model requested one tool invocation: `name` with the raw `arguments`
   * JSON string exactly as the model produced it (unparsed). `callId` pairs the
   * call with its `tool/result`.
   */
  'tool/call': { turn: number; step: number; callId: ToolCallId; name: string; arguments: string }
  /**
   * A completed tool call's model-facing result, optional internal failure
   * identity and user-facing reason, and optional tool-private `meta`
   * presentation payload. The reason remains outside the model-facing message.
   * `meta` is
   * opaque to the core (the producing tool owns its shape and reads it back in
   * `presentResult`) but MUST be JSON-serializable: `Session.append`
   * runtime-validates all event data with `isJsonValue`, so a non-serializable
   * `meta` is rejected at the source, and the durable log reproduces the
   * identical card on replay. Absent
   * unless the tool attaches one (e.g. `dsh-tool-fs` carries its result-time
   * contextual diff here).
   */
  'tool/result': {
    turn: number
    step: number
    message: ToolResultMessage
    /**
     * Optional failure identity and raw user-facing reason, outside model content;
     * allowed only when the message has `isError: true`.
     */
    error?: { name: string; code: string; reason?: string }
    meta?: JsonValue
  }
  /**
   * Full header for the next request, appended inside its step before dispatch.
   * It is log-only; the latest snapshot reconstructs the request header.
   */
  'request/header': {
    header: EpochHeader
    reason: RequestHeaderReason
    /** This request begins a distinct model-message series, independently of the header reason. */
    startsSeries?: true
  }
  /**
   * Route metadata for the next request, logged only when the route, capacity,
   * or system prompt update mode changes. It does not participate in request
   * reconstruction or header equality. Prompt admission uses the bound prepared
   * call's capability, not this snapshot from an earlier request.
   */
  'request/context': RequestContext
  /**
   * Separates inherited or restored history from later lifecycle-owned work.
   * This log-only marker need not be at {@link Session.firstLiveSeq}: a fork
   * seed can already contain its tagged marker and child-owned synthetic
   * closers before construction.
   *
   * A fresh fork child owns one `{ inherited: true }` marker at its exact
   * inherited-prefix cut, even when that prefix ends in an ancestor marker.
   * `buildForkSeed` appends that marker before any synthetic closers; the
   * `Session` constructor supplies it when given only the inherited prefix.
   * The last tagged marker is the current Session's cut; untagged markers
   * keep ordinary restore and replay lifecycle boundaries.
   *
   * Only the `Session` constructor and `buildForkSeed` may create this marker.
   * Session append does not reject other writers, so a plugin appending one
   * would silently classify every live bracket before it as seed history.
   *
   * An owner of a standalone open/close bracket (`compaction/start` …
   * `compaction/end`) reads it because seed history and live work are otherwise
   * byte-identical: an unmatched opening marker before this event belongs to
   * an ended lifecycle, whatever ended it. NOT a liveness signal about other
   * writers — a concurrently live session holds its own boundary elsewhere,
   * so tolerating concurrent writers needs a signal beyond the log.
   */
  'session/end-seed': { inherited?: true }
}
```

`UserMessage` は、通常のプロンプト、注入されたコンテキスト、ステアリング、ライブの inbox イベントで共有される、識別情報を持つ凍結済みの user ロールの値である。イベントラッパーはイベントローカルの位置または結果の事実だけを追加し、ループは項目が保留中である間、driver が所有するルーティング状態だけを追加する。

<a id="the-request-header-event-requestheader"></a>

### リクエストヘッダーのイベント: `request/header`

リクエストエンベロープ、すなわち `EpochHeader`（呼び出し設定 + アダプターが供給するデフォルトのマーカー + 組み立て済みのツールスキーマ）は、ログに記録されたセッション状態である。したがって、すべての会話リクエストはログの純粋関数になる（再構成可能性の Agent Note）。レンダリングされたシステムプロンプトはヘッダーの一部ではない。それは派生履歴であり、サーフェスノード 0 の `system/message` イベントと、それ以降の履歴内システムノードである（[決定](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-09-02-system-prompt-as-surface-node.md)）。そのためプロンプトの変更はシステムノードを置換または追加し、ヘッダーは変えない。理由 `'initial'` または `'resume'` を持つ完全な `request/header` スナップショットは、各ループインスタンスの境界を記録する。変更されたリクエストは理由 `'change'` のスナップショットを追記する。明示的に宣言されたメッセージシリーズを開始する、またはサーフェス置換に続く、変更のないエンベロープは、理由 `'series'` のスナップショットを追記する。`'initial'`、`'resume'`、`'change'` のスナップショットは、そのリクエストがシリーズも開始する場合に `startsSeries: true` を持つ。理由 `'series'` はすでにその事実を記録している。通常の追記専用の後続ターン、さらなるステップ、同じモデルメッセージシリーズ内のリトライは、最新のスナップショットを継承する。`foldRequestHeader(events)` は最新のスナップショットを選択してヘッダーを再構成する。このイベントは `SurfaceEventType` ではない。LLM メッセージを生成しない。

```ts type-equiv
/**
 * Logged request state outside derived history: call config and tools. The
 * system prompt is derived history — surface node 0, a `system/message` event.
 * The latest full `request/header` snapshot reconstructs the header; canonical
 * empty optional fields are absent.
 */
interface EpochHeader {
  /** The conversation's call configuration (provider, model, reasoning effort, and sampling scalars). */
  config: LlmCallConfig
  /** Effective config fields materialized from the exact adapter rather than proposed by a caller. */
  adapterDefaults?: LlmCallConfigAdapterDefaults
  /** Assembled tool schemas; absent for a tool-less request. */
  tools?: ToolSchema[]
  /** Retired request text; system prompts belong to system/message events.
   * @persistenceReserved
   */
  system?: never
}
```

現在のイベント受け入れでは、正規の `request/header.header` を要求する。`system` フィールドは禁止され、`tools: []` と `adapterDefaults: {}` は省略しなければならない。空白だけの system メッセージ内容、`config.stop: []`、ネストした拡張は変更されない。シード、追記、現在の永続化読み取りは、非正規のヘッダーを黙って正規化せず拒否する。履歴の変換は [V3 エンベロープの決定](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-09-06-v3-canonical-session-envelopes.md) が担う。`request/header-delta` またはその完全スナップショットの `fallback` 理由を含むレガシー v0 ログは、不完全にリプレイされるのではなく拒否される。

### ルート容量のイベント: `request/context`

リクエストが解決したルートのコンテキストメタデータは、別のログ状態である。同じステップ内で `request/header` の隣に追記され、プロバイダー、モデル、容量、または `systemPromptUpdate` モードが前のレコードと異なる場合に限られる。これが `EpochHeader` の外にあるのは、その型が `headerEquals` でフィールドごとに比較される再構成契約だからである。容量と更新モードはリクエスト入力ではなくルートを記述するため、これらを折り込むとルートの変更がリクエストエンベロープの `change` として登録され、アダプターメタデータがループのリクエスト再構成に引き込まれてしまう。`request/header` と同様、これは `SurfaceEventType` ではなく、LLM メッセージを生成しない。`session.requestContext()` は最新のレコードを増分的に畳み込む。agent ループは、変更されたシステムプロンプトが最新のシステムノードを置換するか、キャッシュされた履歴の後に追記されるかを決めるとき、そのレコードの `systemPromptUpdate` を読む（[決定ルール](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent-loop/README.md#understand-the-implementation)）。アダプターが容量を通知しないルートは `contextWindow` を欠いた状態で記録されるため、新しいレコードは古いルートの容量をクリアする。更新モードを宣言しないルートも同様に、古いルートの `systemPromptUpdate` をクリアする。

```ts type-equiv
/** Registration-bound metadata for one resolved model route. */
interface RequestContext {
  /** Registered provider route the metadata belongs to. */
  provider: string
  /** Provider-owned model id the metadata belongs to. */
  model: string
  /** Maximum combined request and response context in tokens, when advertised. */
  contextWindow?: number
  /** `'in-history'` when the route reads the latest `system` message at any position as the effective system prompt. */
  systemPromptUpdate?: SystemPromptUpdate
}
```

## `SessionEvent<T>` — 1 つのログエントリー

`type` に関する適切な判別可能ユニオン（独立した `type`/`data` ユニオンではない）であるため、`switch (event.type)` はキャストなしで `event.data` を絞り込む。`seq` はログ内の単調増加する位置（`seq = log.length`）で、`time` はエポックミリ秒である。

```ts type-equiv
/** Sequence number of one existing event in a Session log. */
type SessionSeq = BrandedNumber<'SessionSeq'>
```

```ts type-equiv
/** A Session log gap, prefix length, or read offset, which may equal the event count. */
type SessionLogOffset = BrandedNumber<'SessionLogOffset'>
```

```ts type-equiv
/** Inclusive Session event watermark, or `-1` before any event exists. */
type SessionSeqCursor = SessionSeq | -1
```

```ts type-equiv
/** One existing Session event position, or explicit absence. */
type OptionalSessionSeq = SessionSeq | null
```

`SessionSeq(value)` と `SessionLogOffset(value)` は非負の安全な整数だけを受け入れ、負のゼロを拒否する。これらはシリアライズされた数値を変えずにコンパイル時のブランドを追加する。算術演算は通常の `number` を返すため、呼び出し側は意図する役割に応じてコンストラクター経由で再度受け入れなければならない。

```ts type-equiv
/**
 * One immutable entry in the session log.
 *
 * A proper discriminated union over `type` (not independent `type`/`data`
 * unions), so `switch (event.type)` narrows `event.data` without casts.
 *
 * The {@link sourceEventSeqs} and {@link surfaceOp} fields are conditional:
 * they only exist on {@link SurfaceEventType} variants (`system/message`, `user/message`,
 * `assistant/message`, `tool/result`).
 * Non-surface events (boundary markers, attempts, errors) never carry
 * surface metadata — the compiler enforces this at `Session.append()`
 * call sites.
 */
type SessionEvent<T extends SessionEventType = SessionEventType> = {
  [K in SessionEventType]: {
    type: K
    /** Monotonic sequence number within the session. */
    seq: SessionSeq
    /** Unix epoch milliseconds. */
    time: number
    data: SessionEventMap[K]
    /**
     * Marks an event a reader may safely skip when it does not recognize
     * `type`. Absent means required: a reader meeting an unrecognized type
     * without this marker MUST refuse to reconstruct the session instead of
     * silently dropping the event, because an unrecognized required event may
     * change how the rest of the log is interpreted. A writer sets `true` only
     * on purely informational records whose loss cannot affect reconstruction;
     * defaulting to required means a forgotten marker over-refuses (an
     * inconvenience) rather than silently resuming a gutted session.
     */
    ignorable?: true
  } & (K extends SurfaceEventType ? SurfaceIntent<K> : {
    surfaceOp?: never
    sourceEventSeqs?: never
  })
}[T]
```

`SessionEventType = keyof SessionEventMap`。`SessionEventMap` はマージ拡張可能であるため、`SessionEvent` に対する switch は `assertNever` を使っては**ならない**。プラグインが追加したバリアントは有効な未知の値である。既知のケースを処理し、`default` にフォールスルーする。

すべてのサーフェスイベントは `surfaceOp` を必須とし、既知のログ専用イベントは両方のサーフェスメタデータフィールドを禁止する。ネイティブの未知または廃止された ignorable エンベロープは不透明なまま残る。`assistant/message` はプロバイダーのストリームを埋め込み、`sourceEventSeqs` を禁止する。system、user、tool のサーフェスイベントは、出所の帰属または置換範囲の網羅に必要な場合、一意な先行イベントの完全で空でない集合を引用できる。`tool/result` は、そのメッセージが `isError: true` を持つ場合に限り `data.error` を持てる。失敗の識別情報は失敗した結果でも任意のままである。

## サーフェスの型

メッセージを生成する型（`SurfaceEventType` — `system/message`、`developer/message`、`user/message`、`assistant/message`、`tool/result`）は、順序付きの派生サーフェスにどのように参加するかを宣言するサーフェスメタデータを持つ。`system/message` はレンダリングされたシステムプロンプトを保持する。ループは最初のものをサーフェスノード 0 として追記し、プロンプトが変わると、履歴内ルートでは最新のシステムノードを正確に置換するか、新しいノードを追記する。サーフェスの畳み込みは、ノード 0 の `system/message` を対象とする他の置換を拒否する。一方、後続のシステムノードは通常の履歴であり、コンパクションの置換がそれをシャドウしてよい。詳しくは [session surface の Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-06-18-session-surface.md) を参照。

### `SurfaceEventType` — メッセージを生成するイベント型の部分集合

```ts type-equiv
/**
 * The subset of {@link SessionEventType} values whose events produce LLM
 * messages and are eligible to appear on the ordered surface. Only these
 * event types may carry {@link SurfaceOp}; system, user, and tool events may also cite
 * earlier sources through {@link SessionEvent.sourceEventSeqs}.
 */
type SurfaceEventType =
  | 'system/message'
  | 'developer/message'
  | 'user/message'
  | 'assistant/message'
  | 'tool/result'
```

### `SurfaceOp` — イベントがサーフェスに入った方法

```ts type-equiv
/**
 * How a session event entered the ordered surface. Only valid on
 * {@link SurfaceEventType} events.
 *
 * - `'append'`: added to the tail — normal path for user/assistant/tool
 *   messages.
 * - `{ op: 'replace', startSeq, endSeq }`: replaces surface nodes from `startSeq`
 *   (inclusive) through `endSeq` (inclusive) with this node. Both must exist as
 *   surface nodes in the current surface. `startSeq === endSeq` replaces a single
 *   node. The node's {@link SessionEvent.sourceEventSeqs} must include every
 *   shadowed surface node. Used by compaction; any surface-replacing producer
 *   may use it.
 */
type SurfaceOp =
  | 'append'
  | { op: 'replace'; startSeq: SessionSeq; endSeq: SessionSeq }
```

`'append'` は通常の末尾追記パスである。`replace` は `op`、`startSeq`、`endSeq` だけを含み、エイリアスや余分なキーを持たない。現在のサーフェスイベントシーケンス間の包含範囲をシャドウし、その位置に新しいイベントを挿入する。端点が等しい場合は 1 つのエントリーを置換する。端点は置換するイベントより前になければならないが、その相対順序は数値のシーケンス順ではなくサーフェス順である。

### `SurfaceIntent` — `session.append()` の引数

```ts type-equiv
/**
 * Surface placement and cited source-event seqs for {@link Session.append}. Required on
 * message-producing events and forbidden on log-only events.
 */
type SurfaceIntent<T extends SurfaceEventType = SurfaceEventType> = {
  surfaceOp: SurfaceOp
} & (T extends 'assistant/message' ? {
  /** Assistant messages embed their provider stream instead of citing source events. */
  sourceEventSeqs?: never
} : {
  /** Complete non-empty set of known earlier source-event seqs. */
  sourceEventSeqs?: SessionSeq[]
})
```

`SurfaceEventType` イベントには必須である。メッセージを生成するすべてのイベントは、派生モデル履歴の唯一の源であるサーフェスにどのように参加するかを宣言しなければならない。人間向けのトランスクリプトはもう 1 つのプロジェクションであり、代わりにログの追記起点イベントを読む。サーフェスは置換が要約する範囲を意図的にシャドウするからである（[dsh-session](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/session/README.md) の `isAppendSurfaceEvent`）。非サーフェス型はコンパイル時にこれを拒否する。

`assistant/message` は `sourceEventSeqs` を持てない。その `stream` が正確なプロバイダーの証拠を所有する。他のサーフェスイベントは、先行イベントを引用しない場合はフィールドを省略し、引用する場合は完全で空でないリストを使う。

<a id="plugin-owned-message-projections"></a>
### プラグイン所有のメッセージプロジェクション

内容を変更するプラグインは、イベント宣言に `@messageProjection` を付け、`ctx.sessions.registerMessageProjection()` を通じて純粋な定義を登録する。Session はコミット前にその定義を通じて完全な決定を検証し、不変のメッセージ更新を適用し、`contentGeneration` を進める。復元時やデタッチされた畳み込み時も含め、インタープリターの欠落を拒否する。使用中の定義をアンロードすると、キャッシュされた読み取りがブロックされる。デタッチされたリーダーは `foldSurface(events, projections)` に明示的な定義を渡し、`deriveEventMessage()` を通じて `projectedMessages` を適用する。インストール済みの形式カタログは、オフラインリーダー用のファーストパーティ定義を組み立てる。[画像オフロードプラグイン](compaction.md#image-offload) は、画像固有のイベントと解釈を所有する。

```ts type-equiv
/** Readonly history immediately before a message-projection event. */
interface SessionMessageProjectionContext {
  /** Current message-producing event sequences in model-visible order. */
  nodes: readonly SessionSeq[]
  /** Contiguous event window; entries at or beyond the candidate seq are not committed inputs. */
  events: readonly SessionEvent[]
  /** Absolute sequence of the window's first event. */
  baseSeq: SessionLogOffset
  /** Previously projected messages keyed by their original event sequences. */
  messages: ReadonlyMap<SessionSeq, Message>
}
```

```ts type-equiv
/** Pure interpretation of one plugin-owned event that changes existing message content. */
interface SessionMessageProjection<T extends SessionEventType = SessionEventType> {
  /** Event interpreted by this definition; declare it with `@messageProjection` in SessionEventMap. */
  type: T
  /**
   * Validate the complete durable decision before returning any updates. Preserve
   * message identities and publish immutable copies without mutating the input.
   * @param event - candidate event, not yet applied to the supplied history.
   * @param context - history preceding this decision.
   * @returns changed current messages keyed by their original sequences.
   * @throws when the durable decision cannot be applied to this history.
   */
  project(event: SessionEvent<T>, context: SessionMessageProjectionContext): ReadonlyMap<SessionSeq, Message>
}
```

### `SessionSurface` — ライブの読み取り専用サーフェスプロジェクション

`Session.surface` は、セッションの安定した `SessionSurface` ビューを返す。同じ増分マネージャーが、コミット前に追記候補を検証し、コミット済みイベントからこのプロジェクションを進める。呼び出し側はメンバーシップと置換世代を観察できるが、検証を呼び出すことはできない。

`SurfaceManager(log, baseSeq?, projections?)` は代わりに、最初のイベントが絶対シーケンス `baseSeq` を持つ、連続してロードされたウィンドウを畳み込める。すべてのイベントはその絶対シーケンス空間で連続したままであり、ウィンドウの先頭をまたぐ置換は、宣言された範囲が存在しないため失敗する。

```ts type-equiv
/** Readonly live projection of the message-producing session events. */
interface SessionSurface {
  /** Current surface event sequences in model-visible order. */
  readonly nodes: readonly SessionSeq[]
  /** Monotonic count of committed positional replacements. */
  readonly replaceGeneration: number
  /** Monotonic count of committed replacements and plugin-owned message changes. */
  readonly contentGeneration: number
}
```

### `SurfaceFoldReplacement` と `SurfaceFoldResult` — 完全なサーフェスリプレイ

`foldSurface(events, projections)` は、デタッチされた現在のイベントシーケンスと、各宣言済み置換範囲が実際にシャドウしたシーケンスを返す。ライブマネージャーは置換履歴を保持せずに同じ遷移を使う。その `replaceGeneration` はコミットされた置換ごとに増加するため、増分コンシューマーは純粋な末尾成長と書き換えを区別できる。

```ts type-equiv
/** One replacement operation observed while folding a session surface. */
interface SurfaceFoldReplacement {
  /** Seq of the event that replaced the prior surface range. */
  seq: SessionSeq
  /** Declared inclusive start seq of the replaced surface range. */
  start: SessionSeq
  /** Declared inclusive end seq of the replaced surface range. */
  end: SessionSeq
  /** Actual surface entries removed by the operation, in surface order. */
  shadowedSeqs: SessionSeq[]
}
```

```ts type-equiv
/** Complete result of replaying the surface operations in a session log. */
interface SurfaceFoldResult {
  /** Current surface event sequences in model-visible order. */
  nodes: SessionSeq[]
  /** Replacement operations in event order. */
  replacements: SurfaceFoldReplacement[]
  /** Immutable projected messages, keyed by their original event sequences. */
  projectedMessages: ReadonlyMap<SessionSeq, Message>
}
```

## `Session` の公開 API

本体を除いた宣言は、通常のクラスのデタッチされたファクトリー、状態アクセサー、追記メソッド、履歴プロジェクションをソースと同期させる。ストア操作は生成された [`ctx.sessions` セクション](#ctxsessions--sessionstore) に残る。

```ts public-api
/**
 * An event-sourced session: an append-only log of {@link SessionEvent}s.
 *
 * Plain class (not a Service) — create live instances via
 * `ctx.sessions.create()` and detached instances via {@link create}.
 * Seeding with an existing event log replays/forks a session.
 * @typert object
 */
declare class Session {
  /** The ordered surface over this session's event log. */
  get surface(): SessionSurface;
  /**
   * Detached, deep-frozen creation metadata (format version, cwd, lineage,
   * and whether fork history exists). Supplied by the store via `ctx.sessions.create()`. When a
   * `Session` is created without a store-owned header, a minimal header is
   * synthesized (stamped with the current {@link SESSION_FORMAT_VERSION}) so
   * `session.header` is always present. Kept out of the event log — it is a
   * storage concern, not replayable conversation state.
   */
  readonly header: SessionHeader;
  /** Number of leading events inherited from this Session's fork parent. */
  readonly inheritedEventCount: SessionLogOffset;
  /** The session identity, derived from its durable header's single copy. */
  get id(): SessionId;
  /**
   * The constructor seed length (0 without one), before any marker appended
   * during construction. Seed events never publish on `session/event`. A
   * marker appended before the store attaches occupies this seq without
   * publishing either; otherwise this seq is available for the next append.
   *
   * This in-process offset is not persisted. A fork seed can already contain
   * the child's inherited marker and synthetic closers, so its child-owned
   * history starts at {@link inheritedEventCount}, before this offset. A
   * resumed Session's seed contains its full stored log, while its inherited
   * count keeps the durable fork cut. Consumers needing complete canonical
   * history start at seq 0.
   */
  readonly firstLiveSeq: SessionLogOffset;
  /**
   * First event produced for this object lifecycle. A new fork includes its
   * child-owned seed marker and closers; a restored Session starts after its
   * complete stored prefix. This in-process capture offset is not persisted.
   */
  readonly firstLifecycleSeq: SessionLogOffset;

  /**
   * Create a detached session by validating and snapshotting borrowed seed
   * events and storage metadata.
   * @param id - session identity.
   * @param seed - optional borrowed replay or fork events.
   * @param header - optional borrowed storage metadata.
   * @param inheritedEventCount - exact fork-inherited prefix length for a seeded header.
   * @param projections - pure interpreters for plugin-owned message changes.
   * @returns a detached session.
   * @throws when a seed event requires a missing message interpreter or fails validation.
   */
  static create(
    id: SessionId,
    seed?: readonly SessionEvent[],
    header?: SessionHeader,
    inheritedEventCount?: SessionLogOffset,
    projections?: readonly SessionMessageProjection[],
    ): Session;
  /**
   * Restore a detached session by adopting an independently owned or deeply frozen seed.
   * Runtime-required event fields, event envelopes, sequence continuity, surface
   * transitions, and header fields are validated without copying or freezing events.
   * Embedded Assistant streams remain opaque until a stream consumer or storage
   * verifier reads them.
   * @param id - restored session identity.
   * @param seed - independently owned or deeply frozen events.
   * @param header - independently owned storage metadata.
   * @param inheritedEventCount - exact fork-inherited prefix length decoded from storage.
   * @param eventState - aliasing state carried from the operation that produced the seed.
   * @param projections - pure interpreters for plugin-owned message changes.
   * @returns a restored detached session.
   * @throws when a seed event requires a missing message interpreter or fails validation.
   */
  static fromRestore(
    id: SessionId,
    seed: readonly SessionEvent[],
    header: SessionHeader,
    inheritedEventCount: SessionLogOffset,
    eventState: SessionSeedEventState,
    projections?: readonly SessionMessageProjection[],
    ): Session;
  /**
   * Return the immutable event stored at one exact sequence number.
   * @deprecated Existing logic may remain unmigrated for now, but new calls are prohibited.
   * See the [Agent Note](../../../../.agents/notes/implemented/architecture/2026-09-09-deprecate-synchronous-session-event-reads.md).
   * @param seq - event sequence number.
   * @returns the accepted event, or undefined when the log does not contain it.
   */
  eventAt(seq: SessionSeq): SessionEvent | undefined;
  /**
   * Materialize an immutable snapshot of a half-open event sequence range.
   * A full current snapshot is reused until the next append; every previously
   * returned snapshot remains stable after later appends.
   * @deprecated Existing logic may remain unmigrated for now, but new calls are prohibited.
   * See the [Agent Note](../../../../.agents/notes/implemented/architecture/2026-09-09-deprecate-synchronous-session-event-reads.md).
   * @param fromSeq - non-negative inclusive sequence number; defaults to the log start.
   * @param toSeqExclusive - non-negative exclusive sequence number; defaults to the current end.
   * @returns a frozen array of the selected deeply frozen events.
   */
  snapshotEvents(
    fromSeq: SessionLogOffset = SessionLogOffset(0),
    toSeqExclusive: SessionLogOffset = this.seq,
    ): readonly SessionEvent[];
  /**
   * Return this Session's events after its fork-inherited prefix.
   * @deprecated Existing logic may remain unmigrated for now, but new calls are prohibited.
   * See the [Agent Note](../../../../.agents/notes/implemented/architecture/2026-09-09-deprecate-synchronous-session-event-reads.md).
   * @returns a fresh array containing child-owned events in log order.
   */
  ownEvents(): readonly SessionEvent[];
  /**
   * Whether one existing event position is outside the fork-inherited prefix.
   * @param seq - event position in this Session.
   * @returns true when the event belongs to this Session rather than its parent.
   */
  isOwnSeq(seq: SessionSeq): boolean;
  /** The next event's sequence number — always the log length (the `seq = log.length` contiguity contract). */
  get seq(): SessionLogOffset;
  /**
   * Append one typed event to the log and synchronously notify observers via
   * the store-owned, module-private publication hooks. The hot path never blocks
   * on I/O — persistence plugins buffer asynchronously. Once the event enters
   * the log, the append is committed: observer failures are logged and
   * contained per listener, so they do not change the return value or prevent
   * later listeners from observing the same accepted event.
   * Independent Sessions append without notifying store observers. Once a store
   * prepares or enters the instance, writes require a completed creation
   * announcement and remain disabled after removal.
   *
   * @param type - The event type (key of {@link SessionEventMap}).
   * @param data - The event payload; must be JSON-serializable.
   * @param opts - Surface metadata: `surfaceOp` controls how the event enters
   *   the ordered surface; `sourceEventSeqs` lists the seq numbers of earlier
   *   events this one derives from. REQUIRED for
   *   {@link SurfaceEventType} events (every message-producing event must
   *   declare how it joins the surface, the sole source of derived model
   *   history) and
   *   rejected by the compiler for non-surface types like `turn/start` or
   *   `assistant/attempt`. Assistant messages embed their exact provider
   *   stream and cannot cite top-level source events.
   * @returns the logged event — its assigned `seq`/`time` plus the SNAPSHOT of
   *   `data` that entered the log, so reading `event.data` back sees the logged
   *   value, never the caller's still-mutable input.
   * @throws if a store-owned Session is unentered or its creation announcement
   *   has not completed, or if `data` or surface
   *   metadata is not losslessly JSON-serializable
   *   (BigInt, function, symbol, undefined, negative zero, non-finite number,
   *   circular reference, sparse array, or an exotic object such as
   *   Map/Set/Date/class instance), or when the candidate violates the
   *   request-header empty-field or tool-error consistency rules, or the
   *   canonical surface contract (marker shape and eligibility, unique
   *   earlier source-event references, positional replacement validity, and complete
   *   shadowed-node coverage). One iterative pass reads, validates, and
   *   copies each nested value once, so a stateful getter cannot supply one value
   *   to validation and another to storage. The event log is the durable source
   *   of truth, so a bad event fails at the append site rather than later during
   *   a backend flush. A synchronous internal dispatch validation failure or an
   *   append reentered while this acceptance/publication boundary is open also
   *   rejects before the log changes.
   */
  append<T extends SessionEventType>(
    type: T,
    data: SessionEventMap[T],
    ...opts: T extends SurfaceEventType ? [opts: SurfaceIntent<T>] : []
    ): SessionEvent<T>;
  /**
   * The {@link EpochHeader} in force after the log's last header event — the
   * header the NEXT request will be compared against — or undefined before
   * the first `request/header` snapshot. The live, incrementally-maintained
   * form of `foldRequestHeader(session.snapshotEvents())`: each header event is folded
   * once, when first seen, so a per-step read costs O(new events).
   * @returns the folded header, or undefined when no header event exists yet.
   */
  requestHeader(): EpochHeader | undefined;
  /**
   * Return the latest resolved route metadata, or `undefined` before the first
   * `request/context` event. Each event is folded once.
   * @returns the latest immutable route metadata.
   */
  requestContext(): RequestContext | undefined;
  /**
   * Fold unseen committed events into capability-independent tool history.
   * Initial access reconstructs inherited history; later reads consume only new events.
   * @returns an immutable snapshot for LLM request projection, including historical addition definitions.
   */
  toolHistory(): ToolHistory;
  /**
   * Derive the LLM message history by walking the ordered sequences of
   * message-producing events maintained by `surfaceOp` markers. The
   * surface is the single source of derived history: every message-producing
   * append records its `surfaceOp`, so a raw event with no marker (a chunk, a
   * turn boundary) is correctly absent, and a compaction `replace` deletes the
   * shadowed nodes from the derivation. The projection rules are
   * {@link deriveEventMessage}, with logged message projections applied
   * without changing node membership or message identity.
   *
   * CACHED: pure tail growth costs O(new nodes); a replacement or message projection
   * ({@link SessionSurface.contentGeneration}) rebuilds. The returned array is
   * a fresh snapshot per call (later appends never grow an array a caller
   * already holds); the `Message` objects in it are SHARED and **deep-frozen**.
   * Unchanged content reuses frozen event data; projected blocks are frozen
   * derived copies. Consumers cannot mutate the log through either form.
   * @returns a fresh array of the shared, frozen derived history.
   */
  deriveMessages(): Message[];
  /**
   * Project one event with all committed message projections applied.
   * The original durable event remains unchanged.
   * @param event - the event to project.
   * @returns the derived message, or null when the event produces none.
   */
  deriveEventMessage(event: SessionEvent): Message | null;
}
```

## 派生履歴: `deriveMessages()` と `deriveEventMessage()`

`Session.deriveMessages()` はイベントログを、モデルが見る `Message[]` に投影する。キャッシュされ（各サーフェスノードは最初に見られたときに 1 回だけ投影され、サーフェスの書き換えで再構築される）、凍結される（呼び出しごとに新しい配列を返し、メッセージは共有されて深く凍結されるため、プロジェクションを通じてログ履歴を変更することは表現できない）。`deriveEventMessage(event)` は畳み込みが適用するノードごとの純粋関数であり、外部の再構成器がまったく同じルールでログのプレフィックスを投影し、キャッシュと食い違わないように公開されている。投影のルールは次のとおり。

- `user/message` → 正確な `content` を持つ user メッセージ。任意のエンベロープはログ専用の表示メタデータとして残る。
- `assistant/message` → それを生成したプロバイダーとモデル、および任意のアダプター専用リプレイ状態を持つ assistant メッセージ。埋め込まれたコンパクトストリームは 2 つ目のメッセージではなく、リプレイ、usage、UI の証拠である。**内容が空**の `assistant/message` もスキップされる。内容なしで打ち切られた max-tokens のステップは、ストリーム、usage、プロバイダー、モデルを保持するために `assistant/message` を記録するが、内容のない assistant ターンがプロバイダーのトランスクリプトに入ってはならない。
- `tool/result` → 結果の内容とツール呼び出しの識別情報を持つ、第一級の tool ロールメッセージ。
- `user/message`（注入されたコンテキスト、つまり `user` 以外のソース）→ 時系列上の位置に `content` をそのまま持つ user ロールメッセージ。その型付きソースが生成元を名指しし、生成元固有のデータを運ぶ。

それ以外（`turn/*`、`step/*`、`assistant/attempt`、プラグイン所有の `llm/retry`）は構造的であり、メッセージに投影されない。トークン会計は各 `assistant/message` または `assistant/attempt` の埋め込みストリームを展開するが、メッセージのトップレベル `usage` は存在する場合、コミット済みメッセージの権威のままである。したがって、失敗したモデルリクエストの試行は、assistant メッセージを捏造せずにプロバイダーの usage を保持する。現在の論理検証は、プロバイダー／モデルを欠くリクエストヘッダーと assistant メッセージを、ルートを推測せずに拒否する。サポートされる履歴表現は、現在の Session が存在する前に、隣接する形式エッジによって正規化され検証される。

## ライブセッションのフォーク API

`ctx.sessions.create(id, { seed, meta })` は低レベルのリプレイ／フォークプリミティブである。通常のライブセッションフォークでは、`SessionStore` が 1 つのポリシー API を公開する。

- `fork(source, boundary?, childSessionId?)` はライブの `Session` オブジェクトまたはライブの `SessionId` を受け取り、包含的な `SessionSeq` 境界（デフォルト: 現在の最後のイベント）までソースイベントを選択し、開いている末尾を閉じてから、深くクローンしたシードイベント、`isSeeded: true`、正確な `inheritedEventCount`、継承した `cwd` を持つ独立したライブセッションを、`parentSession` なしで作成する。

明示的な `boundary` により、呼び出し側はソースに新しいイベントや開いている現在のターンがあっても、前の `turn/end` や後続の単独ログ専用イベントなど、任意の安定したターン間の位置からフォークできる。API は、開いているターンの内部で終わるプレフィックスを黙って切り詰めるのではなく拒否する。`dsh-subagent-fork-in-process` は完了済みプレフィックスの切り詰めを維持する。ツール実行時の委譲は通常、親ターンが開いている間に始まるからである。通常のセッション分岐では、要求する境界を明示すべきである。

## ターンが終了した理由: `TurnEndReasonMap`

`turn/start` にはトリガーフィールドがない。入った `user/message` バッチが各ステップに入ったものを記録し、`llm/retry` がリクエストの回復を記録し、アイドル注入は、起動する配信が後続の pre-step に到達するまで保留のままである。ライブのターンは、driver を停止させた型付きの [`AgentCancelCause`](core.md#the-agent-handle) を保持する。永続化は、呼び出し元を保存していないサポート対象の粗いキャンセルレコードをインポートする場合に限り、追加の `{ kind: 'legacy' }` 原因を使う。

```ts type-equiv
/** Durable cancellation cause, including imports whose original coarse record carried no cause. */
type TurnEndCancelCause = AgentCancelCause | { readonly kind: 'legacy' }
```

```ts type-equiv
/**
 * Why a turn ended. Merge-extensible sum type.
 */
interface TurnEndReasonMap {
  completed: { kind: 'completed' }
  /** A cancellation request interrupted the live turn. */
  aborted: { kind: 'aborted'; reason: TurnEndCancelCause }

  blocked: { kind: 'blocked' }
  /**
   * The turn failed. `error` is always a structured failure: the `LlmError`
   * facts verbatim, or `{ message: errorChain(error), code }` flattened from
   * any other error, where `code` is `TOOL_ACTIVATION_FAILED` for a failed
   * tool condition and `UNKNOWN` otherwise.
   */
  error: { kind: 'error'; error: LlmFailure }
  /** At least one step reached its output-token ceiling, even if a plugin continued the turn. */
  'max-tokens': { kind: 'max-tokens' }
  /**
   * A crash-orphaned turn was closed after the fact: agent-loop resume appends
   * this closer for a stored log whose last turn never ended, and session-query
   * synthesizes it on cold reads. The loop never emits this marker live, and
   * the events recorded before the crash remain intact.
   */
  interrupted: { kind: 'interrupted' }
  /**
   * Fork-seed construction closed a turn that was still open at the fork
   * boundary in the source session. Only fork seeds carry this marker — the
   * loop never emits it — and the source events before the boundary remain
   * intact in the child.
   */
  forked: { kind: 'forked' }
}
```

`max-tokens` は同名のモデル呼び出しの `FinishReason` を反映する。ターン内のいずれかのステップが `max-tokens` であれば、ターン全体は `completed` ではなく `max-tokens` で終了する（打ち切られた事実が後の継続より優先される）。これにより、コンシューマーは正常な停止と切り詰められた停止を区別できる。キャンセルとエラーは別個の結果のままである。ループは `interrupted` も `forked` もライブでは発行しない。クラッシュ復旧が `interrupted` を合成し（[persistence.md](persistence.md) を参照）、フォークシードの構築が `forked` を合成する。このマップはマージ拡張可能である。

## 実行の囲い込みとスタンドアロンイベント

ターンが囲い込むのは 1 回のモデルループ実行であり、セッションログ全体ではない。AgentLoop は、ターン内で pre-step バッチに入ったものからのみ、注入された `user/message` イベントを記録する。プラグイン所有のログ専用イベントは、`turn/end` と次の `turn/start` の間に現れることがあり、ターン番号を増やさずにイベント seq を消費する。永続化は、連続して受け入れられたすべてのイベントを境界付きの永続バッチに受け入れ、クラッシュ修復は真に開いている末尾のターンだけを閉じる。即時の永続性バリアを必要とする生成元は、明示的に `ctx.sessions.flush(session)` を待つ。

マージ拡張可能なイベント関係は、それを宣言するプラグインに属する。そのため core は、ターンが開いていないという理由だけで未知のイベントを拒否しない。[スタンドアロンイベントの決定](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/simplification/2026-07-28-remove-synthetic-log-only-turns.md) を参照。

## end-seed 境界: `session/end-seed`

`buildForkSeed` は、合成のフォーククローザーを追加する前に、正確な継承プレフィックスの切断位置で `session/end-seed { inherited: true }` を追記する。`Session` コンストラクターは、その準備済みマーカーを保持するか、継承プレフィックスだけが与えられた場合に 1 つ追記する。復元はタグ付きマーカーを保持し、完全な保存済みシードがすでにマーカーで終わっていない場合に限り、通常の `session/end-seed {}` を追記する。どちらの形式もログ専用で、メッセージを生成しない。作成できるのはコンストラクターと `buildForkSeed` だけである。

フォーク系譜では、ペイロードが `inherited: true` を持つ**最後**のマーカーを特定する。現在の形式のデコードは、`SessionHeader.isSeeded` が true のときに正確にそれを要求し、その seq から `inheritedEventCount` を導出する。ライフサイクルの所有関係では、どちらかの形式の最後の `session/end-seed` を特定する。すでに任意のマーカーで終わるシードを再度開いても、別の通常マーカーは追記されない。

これが存在するのは、シード履歴とライブの作業がそうでなければバイト単位で同一になり、単独の開閉ブラケットを所有するプラグインを無力化するからである。一致しない `compaction/start` は、書き込み側がコンパクションの途中でクラッシュした場合も、今まさにコンパクションしている場合も同じに読める。`session/end-seed` より前の開始マーカーはコンストラクターのシードに由来し、何がそれを終わらせたか（クラッシュ、成功したプロセス、まだ実行中の親からのフォーク）に関係なく、終了したライフサイクルに属する。したがってその所有者は、それを終了済みとして扱ってよい。これは*この*セッションが継承したブラケットだけを対象とする。同じ履歴上で開いたブラケットを保持する同時実行のライブセッションは、別の場所に独自の境界を持つため、同時書き込みを許容するにはログを超えた生存シグナルが必要である。core は境界を書き込み、そこから何も読まない。ブラケットの語彙は所有するプラグインに残る。だからこそクラッシュ修復は turn/step/tool の境界を閉じ、`compaction/*` は決して閉じない。

人間のアクティビティで Session を並べるコンシューマーは、この境界を除外する。Session を開くことは作業ではないため、ログの末尾で並べると、開かれたすべての Session が先頭に浮上してしまう。

## プラグインが提供するログ専用イベント

プラグインは追加の `SessionEventMap` 型を宣言マージしてよい。これらは**ログ専用**である。`SurfaceEventType` では**なく**（`surfaceOp` を持たず、派生履歴に何も寄与しない）、所有者が、開いている実行ターンに属するか、ターン間に存在してよいかを決める。生成された [永続化ログのイベントカタログ](../../../docs/persistence-catalog.md) は、core とプラグイン提供のすべてのイベントを列挙する。compaction seam の `compaction/*` のセマンティクスは [compaction.md](compaction.md) で論じる。

1 つのプラグイン所有ファミリー内の複数のイベントが 1 つの Web Client Conversation Node に組み立てられる場合、そのファミリー内のすべての開始、更新、結果、リソース、中断イベントは、同じ安定したビジネス ID を保持するか、独立に導出する。この要件は相関する Node ファミリーに適用され、すべての Session イベントには適用されない。これによりクライアントは、隣接関係から推測したり履歴を走査したりせずに、各イベントをグループ化できる。[Conversation サブシステム](conversation.md) を参照。

hook ブリッジの `hook/invoked` / `hook/result` ペア（`@deepseek-ai/dsh-hook-protocol` 由来）は `handlerId` で相関する。`UserPromptSubmit`、`PreToolUse`、`PostToolUse`、`Stop` はループの開いているターン内で発火するため、それらの `hook/*` レコードは構造的にターンに囲い込まれる。`SessionStart` はターン 1 より前に実行されるため `hook/*` レコードを持たない。そのコンテキストは、起動する配信がターンを開くまで inbox に保留されたままである。

### プラグイン記録

実験的パッケージは `PluginRecordMap` でプラグイン所有の状態を宣言する。`appendPluginRecord()` は各宣言を使って名前とペイロードの型を付け、`plugin:` 名前空間で `ignorable: true` とマークされたイベントを追記する。`pluginRecordOf()` は、現在の宣言に関係なく、未知のペイロードを持つ保存済みレコードを読む。[実験的永続化カタログ](../../../docs/experimental-persistence-catalog.md) は現在の宣言を列挙する。[パッケージ README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/session/README.md#write-experimental-plugin-records) が、呼び出し元の制限、保持、読み取りパスを所有する。

```ts type-equiv
/**
 * Event type of an experimental plugin record: `plugin:` followed by
 * slash-separated segments of lowercase letters, digits, `.`, `_`, and `-`,
 * each starting with a letter or digit. A record type is never a
 * {@link SessionEventMap} member.
 */
type PluginRecordType = `plugin:${string}`
```

```ts type-equiv
/**
 * Payloads written by experimental packages, keyed by their `plugin:` record
 * names. Packages augment this map to type writes and enter the current plugin
 * record catalog; these declarations do not enter {@link SessionEventMap} or
 * released persistence schemas. Stored records still require owner validation.
 */
interface PluginRecordMap {}
```

```ts type-equiv
/**
 * One committed experimental plugin record, read from the log by
 * `pluginRecordOf`. The record's owner validates `data` before use, because a
 * restored record carries whatever JSON an earlier build of its owner wrote,
 * or a V3 event that a format migration renamed into the `plugin:` namespace held.
 */
interface PluginRecord {
  /** The record type, chosen by the owning plugin. */
  readonly type: PluginRecordType
  /** The record's position in the Session log. */
  readonly seq: SessionSeq
  /** Unix epoch milliseconds at append. */
  readonly time: number
  /** The JSON payload as committed. */
  readonly data: unknown
}
```

## 永続性の契約

永続化バックエンドが依存するもの: 永続ログはすべてのイベントを損失なく永続化し、すべての Assistant 試行は、埋め込まれたコンパクトストリームが元のタイミング付きチャンクを保持する 1 つの `assistant/message` または `assistant/attempt` である。`seq` はこれらの確定と、間に挟まるすべてのイベントを通じて連続したままである。バックエンドはイベントバッチに独自のストレージフレーミングを選んでよいが、ハンドルの `read()` が追記されたイベントを正確に返す必要がある。現在の JSONL はイベントごとに 1 行を書き込む（[persistence.md](persistence.md) を参照）。すべての `event.data` は JSON シリアライズ可能でなければならず、`Session.append` がソースでこれを強制する（シリアライズできないデータでは例外を投げる）。そのため不正なイベントがログに入ることはなく、`session.snapshotEvents()` は常にバックエンドが永続化できるものと等しい。シリアライズできないデータを運ぶイベント型の追加、core の実行ネストの破壊、所有者が宣言した関係の違反は、オンディスク形式に対する破壊的変更である。

この契約を消費するバックエンドは [persistence.md](persistence.md) にある。

## リモートカタログとワークスペースのオープン

`ModelCatalog` は `session/modelCatalog` が返す Host 世代のモデルディレクトリであり、デプロイメントのデフォルト、ルーティング可能なプロバイダー ID、成功したプロバイダーグループ、分離されたプロバイダー障害を運ぶ。これは 1 つの Session から派生するものではなく、Session のプロジェクションとは別のままである。

`SessionOpenWorkspacePathRequest` は絶対パスまたはワークスペースで解決された `path` を運ぶ。任意の `action: "reveal"` は、デフォルトアプリケーションでのオープンではなくファイルマネージャーのナビゲーションを選ぶ。`SessionOpenWorkspacePathValue` は、Host がネイティブのハンドオフを受け入れたことを確認する。Session を認識する Client は、既知の場合、相対パスを現在の Session の cwd に対して解決する。コントローラーはパスをそのままオープナーに渡し、無効なリクエスト、キャンセル、オープナーの失敗を Session Remote のエラー語彙で報告する。

任意の `application` は、システムのデフォルトを変えずに登録済みのファイルハンドラーを選ぶ。`workspacePathApplications({ path })` は、Host のパス検証後に、現在のハンドラー、名前、アイコン、デフォルト選択を返す。

## セッションログのアップロード除外

`ctx.sessionLogDeepSeek.registerExclusion()` は、`SessionLogUploadRequest` に対する呼び出し元所有の述語を登録する。一致した場合、Session イベントを読む前に、ログの寄与とその受け入れコールバックを省略する。一致しない場合は通常のアップロード適格性が保たれる。[パッケージ README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-log-deepseek/README.md#upload-exclusions) が失敗時と破棄時の挙動を定義する。

```ts type-equiv
/** Request identity inspected before preparing a Session-log contribution. */
interface SessionLogUploadRequest {
  /** Live Session whose canonical log would be contributed. */
  readonly sessionId: SessionId
  /** Auxiliary request classification; absent for ordinary conversation requests. */
  readonly purpose?: string
}
```

```ts type-equiv
/**
 * Synchronous, side-effect-free upload exclusion owned by its registering plugin.
 * @param request - request identity; the predicate must not mutate it.
 * @returns `true` to omit the log field and acceptance callback, or `false` to keep checking eligibility.
 */
type SessionLogUploadExclusion = (request: SessionLogUploadRequest) => boolean
```

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxsessioncontroller--sessioncontroller"></a>

### `ctx.sessionController` — `SessionController`

生成された `ctx.remote.session` 名前空間を支える Host サービス。

```ts cordis-catalog
/**
 * Resolve or resume one ordinary Session for another Host API domain.
 * @param sessionId - Session identity whose Agent owns the operation.
 * @returns the live Agent or the stable Session-domain failure.
 */
resolveAgent(sessionId: SessionId): Promise<ApiSessionAgentResult>

/**
 * Read all visible Session rows without resuming an Agent.
 * @param _request - reserved empty list request.
 * @param signal - cancellation for persistence reads and summary generation.
 * @returns visible Session summaries ordered by activity.
 */
@Remote('list') async list(_request: SessionListRequest, signal: AbortSignal): Promise<SessionListValue>

/**
 * Search visible Session content without resuming an Agent. A non-owner
 * receives only the hits whose Session `allows()` admits; `hasMore`
 * describes the bounded result before that filtering.
 * @param request - literal message-content query.
 * @param signal - cancellation for list and search reads.
 * @returns authorized bounded Session search results.
 */
@Remote('search') async search(request: SessionSearchRequest, signal: AbortSignal): Promise<SessionSearchValue>

/**
 * Create one ordinary Session for a new conversation, reusing the named blank Session while it is reusable.
 * @param request - optional blank Session to reuse, location, and Agent preset.
 * @returns the created or reused Session identity and its preset when configured.
 */
@Remote('create') create(request: SessionCreateRequest): Promise<SessionCreateValue>

/**
 * Select one Session-local model after explicitly resuming the Session; save the default in the background.
 * @param request - Session identity and requested model selection.
 * @returns the normalized selection installed for the Session, without waiting for default persistence.
 */
@Remote('selectModel') selectModel(request: SessionSelectModelRequest): Promise<SessionSelectModelValue>

/**
 * Select the first available account model after login when no provider API key is configured.
 * @returns after saving the first available model or retaining the existing default.
 */
@Remote async initializeDefaultModel(): Promise<void>

/**
 * Describe every currently routable model for Host-generation selectors.
 * @returns provider-grouped models, the deployment default, and isolated provider failures.
 */
@Remote('modelCatalog') modelCatalog(): Promise<ModelCatalog>

/**
 * Report whether this deployment can hand a Session workspace path to a native desktop.
 * @returns true when the matching open operation is available.
 */
@Remote canOpenWorkspacePath(): boolean

/**
 * Describe the serving desktop for authenticated file-action routes.
 * @returns Host name, configured availability, and platform-specific file-manager behavior.
 */
workspaceDesktop(): { name: string; available: boolean; fileManager: 'finder' | 'explorer' | 'directory' | null }

/**
 * Verify one path through the composed filesystem and open it on the Host desktop.
 * @param request - path after best-effort Session workspace resolution.
 * @param signal - caller lifetime; abort terminates the native command.
 * @returns confirmation after the native opener accepts the path.
 * @throws RemoteError when the request is invalid, has no verified Host mapping, is cancelled, or the opener fails.
 */
@Remote('openWorkspacePath') async openWorkspacePath( request: SessionOpenWorkspacePathRequest, signal: AbortSignal, ): Promise<SessionOpenWorkspacePathValue>

/**
 * Query current file handlers on the serving desktop without activating an Agent.
 * @param request - file path in Host filesystem syntax.
 * @param signal - caller lifetime, propagated to filesystem and desktop queries.
 * @returns OS application names, icons, and default selection; empty when desktop opening is unavailable.
 * @throws RemoteError when the path is invalid, the query is cancelled, or native discovery fails.
 */
@Remote('workspacePathApplications') async workspacePathApplications( request: { readonly path: string }, signal: AbortSignal, ): Promise<readonly SessionWorkspacePathApplication[]>

/**
 * Rename one Session after explicitly resuming it.
 * @param request - Session identity and proposed title.
 * @returns the accepted title and durable event sequence.
 */
@Remote('rename') rename(request: SessionRenameRequest): Promise<SessionRenameValue>

/**
 * Fork one cold-readable exact event prefix into a new Session. An omitted
 * boundary selects the latest completed-turn prefix; an open cut receives
 * synthetic fork closers.
 * @param request - source Session, optional exact inclusive event boundary, and migration preflight choice.
 * @returns the new Session identity.
 */
@Remote('fork') async fork(request: SessionForkRequest): Promise<SessionForkValue>

/**
 * Admit one prompt after explicitly resuming its Session.
 * @param request - Session identity, prompt content, source metadata, and delivery mode.
 * @param signal - caller cancellation before prompt admission begins.
 * @returns acknowledgement that the Agent accepted the prompt.
 */
@Remote('prompt') async prompt(request: SessionPromptRequest, signal: AbortSignal): Promise<SessionPromptValue>

/**
 * Read one image from the attachment store after gateway authorization.
 * @param request - image reference and gateway permission scope; Session existence is not checked.
 * @returns the durable attachment reference and base64-encoded bytes.
 */
@Remote('attachment') attachment(request: SessionAttachmentRequest): Promise<SessionAttachmentValue>

/**
 * Mutate one still-pending queue occurrence, resuming a cold Agent first.
 * @param request - Session, queue item, and requested mutation.
 * @returns acknowledgement that the queue mutation was applied.
 */
@Remote('updateQueue') updateQueue(request: SessionUpdateQueueRequest): Promise<SessionUpdateQueueValue>

/**
 * Cancel one active Agent turn without dropping its pending inbox.
 * @param request - Session whose active Agent turn is cancelled.
 * @returns acknowledgement that cancellation was requested.
 */
@Remote('cancel') cancel(request: SessionCancelRequest): SessionCancelValue

/**
 * Read one cold-safe, message-aligned Session history page.
 * @param request - durable address, optional fixed cut, backward cursor, and page budget.
 * @param signal - cancellation for persistence reads.
 * @returns one chronological page with its inclusive cursor.
 */
@Remote events(request: SessionPageRequest, signal: AbortSignal): Promise<SessionEventsPage>

/**
 * Subscribe to Session events without reading a snapshot.
 * @param request - Session address.
 * @param signal - subscription cancellation.
 * @returns readiness and subsequent events, without a snapshot.
 */
@Remote({ mode: 'stream' }) observeEvent(request: SessionObserveEventRequest, signal: AbortSignal): AsyncIterable<SessionEventFrame>

/**
 * Subscribe to sparse Session projection changes.
 * @param request - target Session.
 * @param signal - subscription cancellation.
 * @returns readiness and sparse projection changes.
 */
@Remote({ mode: 'stream' }) observeProjections(request: SessionProjectionsRequest, signal: AbortSignal): AsyncIterable<SessionProjectionFrame>

/**
 * Read passive Session state.
 * @param request - target Session.
 * @param signal - read cancellation.
 * @returns current passive state.
 */
@Remote state(request: SessionStateRequest, signal: AbortSignal): Promise<SessionState>

/**
 * Subscribe to complete Session state replacements.
 * @param request - target Session.
 * @param signal - subscription cancellation.
 * @returns complete passive state replacements.
 */
@Remote({ mode: 'stream' }) observeState(request: SessionStateRequest, signal: AbortSignal): AsyncIterable<SessionState>

/**
 * Wait for Session quiescence or pending input with bounded assistant output.
 * @param request - Session, wait bounds and output window.
 * @param signal - caller cancellation.
 * @returns stopping state and bounded assistant output.
 */
@Remote wait(request: SessionWaitRequest, signal: AbortSignal): Promise<SessionWaitValue>

/**
 * Follow one Session log from its opening or resume cursor.
 * @param request - durable address and last committed sequence already held by the caller.
 * @param signal - cancellation owned by the Remote stream carrier.
 * @returns a complete opening snapshot followed by gap-free durable event
 *   frames and optional cursorless assistant-stream frames.
 */
@Remote({ mode: 'stream' }) follow(request: SessionFollowRequest, signal: AbortSignal): AsyncIterable<SessionFollowFrame>

/**
 * Read exact projections, returning cached hints only when migration is required.
 * @param request - Session whose current values are required.
 * @param signal - cancellation for the Session read.
 * @returns a sequenced baseline, cached hints when migration is deferred, or null when absent.
 */
@Remote('projections') projections(request: SessionProjectionsRequest, signal: AbortSignal): Promise<SessionProjectionsValue>

/**
 * Stream a complete live-control baseline followed by replacement frames.
 * @param signal - cancellation owned by the Remote stream carrier.
 * @returns one complete baseline followed by live replacement frames.
 */
@Remote({ mode: 'stream' }) async *control(signal: AbortSignal): AsyncIterable<SessionControlFrame>
```

型: [SessionId](core.md) · [SessionSearchRequest](session-query.md)

ソース: [`packages/api/session-controller/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/session-controller/src/index.ts)

<a id="ctxsessionlogdeepseek--sessionlogdeepseekservice"></a>

### `ctx.sessionLogDeepSeek` — `SessionLogDeepSeekService`

呼び出し元が所有するアップロード除外を備えた、増分の Session ログコントリビューター。

```ts cordis-catalog
/**
 * Exclude matching requests before reading or preparing their log contribution.
 * Any matching rule suppresses both the field and its acceptance callback. With
 * no matching rules, normal upload eligibility applies. A throwing rule fails
 * this contribution; the request-extension registry omits it and reports the error.
 * @param exclude - synchronous, side-effect-free predicate; `true` skips this upload.
 * @returns effect disposer that removes this registration when called or when the caller unloads.
 */
registerExclusion(exclude: SessionLogUploadExclusion): () => Promise<void>
```

ソース: [`packages/session/session-log-deepseek/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-log-deepseek/src/index.ts)

<a id="ctxsessions--sessionstore"></a>

### `ctx.sessions` — `SessionStore`

インメモリのセッションストア（`ctx.sessions`）。

永続化はここでは意図的に実装されていない。agent ライフサイクルが、公開された各セッションの書き込みハンドルにセッションログライターを接続する。そのライフサイクルの外で公開されたセッションは何も永続化しない。

```ts cordis-catalog
/**
 * Register one event interpreter for live creation, restore, and fork.
 * Disposing the contribution makes sessions that used it refuse further derivation.
 * @param projection - pure definition owned by the event's plugin.
 * @returns the fiber-owned disposer.
 * @throws when another definition already owns this event type.
 */
registerMessageProjection(projection: SessionMessageProjection): () => Promise<void>

/**
 * Create a session owned by the calling fiber: disposing that fiber stops
 * event notification and removes the session from the store. `options.seed`
 * populates the session with a copy of those events (replay/fork);
 * `options.meta` attaches creation metadata (validated absolute `cwd`, seed
 * and parent lineage, and delegation depth) as the immutable
 * {@link SessionHeader} (the store fills `version`/`id`/`createdAt`).
 *
 * For an agent whose session must be torn down IN ORDER with its loop (so the
 * loop's final events are published before the store attachment ends), do NOT use this
 * — fold the session lifecycle into the agent's own effect via
 * {@link prepare} + {@link enter} + {@link announce} (see
 * `dsh-agent-loop`'s creation transaction).
 *
 * @param id - the session id; omitted, the store mints `session-<n>`.
 * @param options - seed events and/or creation metadata for the header.
 * @returns the live session, already entered and announced.
 * @throws if a session with `id` already exists, metadata is not a plain
 *   lossless-JSON record with valid scalar fields, or `meta.cwd` is a
 *   non-absolute path (storage backends key directories off it).
 */
create(id?: SessionId, options?: CreateSessionOptions): Session

/**
 * Build a session WITHOUT entering it into the store — validate the id/cwd and
 * construct the {@link Session} (with its immutable {@link SessionHeader}).
 * Pairs with {@link enter} + {@link announce}: a caller that owns a composite
 * `ctx.effect` (the agent factory) folds the session lifecycle into that ONE
 * effect so a fiber unload tears the session + agent down as a single ORDERED
 * chain rather than as racing sibling effects — which would remove the publication hooks
 * before the driver's closing events commit, dropping them.
 *
 * @param id - the session id; omitted, the store mints `session-<n>`.
 * @param options - seed events and/or creation metadata for the header. With
 *   `eventState`, every seed event is either independently owned or any
 *   shared value is deeply frozen; {@link Session.fromRestore} validates and
 *   adopts those values without copying or freezing them.
 * @returns the store-owned session, not yet entered and read-only until announced.
 * @throws if a session with `id` already exists, metadata is not a plain
 *   lossless-JSON record with valid scalar fields, or `meta.cwd` is a
 *   non-absolute path.
 */
prepare(id?: SessionId, options?: PrepareSessionOptions): Session

/**
 * Enter a session into the store: install the module-private
 * append publication hooks and add it to the store. Writes remain disabled
 * until {@link announce} completes successfully. Returns the DETACH
 * disposer (hooks + store removal). Does NOT emit `session/created` —
 * the caller yields this disposer inside its effect and THEN calls
 * {@link announce}, so a throwing `session/created` listener rolls the attach
 * back instead of leaking it.
 *
 * Re-checks the id for a duplicate: `prepare` and `enter` are public
 * cross-package primitives and a caller may interleave arbitrary work (or
 * another create) between them, so a stale prepared session must NOT overwrite
 * a live store entry of the same id — its detach disposer would later delete
 * the REAL session. The {@link create} convenience and the agent factory call
 * the two back-to-back so they never trip this, but the public API cannot
 * assume that.
 *
 * @param session - an unentered session; independent instances become store-owned.
 * @returns the detach disposer (publication hooks + store removal). When called from
 *   a synchronous `session/created` listener, removal and disposal wait until
 *   that creation dispatch unwinds.
 * @throws if a session with this id is already in the store.
 */
enter(session: Session): () => void

/** Emit `session/created` exactly once for an {@link enter}ed session (with
 * the carrier {@link enter} captured). Separate from {@link enter} so the
 * caller can yield the detach disposer first (rollback safety — see
 * {@link enter}). The Session remains read-only throughout synchronous
 * dispatch; successful completion enables appends. A failed announcement
 * leaves it read-only and cannot be retried.
 * @param session - the entered session to announce to listeners.
 * @throws if the session is not live or its announcement already began,
 *   including a reentrant call from a creation listener. */
announce(session: Session): void

/**
 * Dispatch the awaited `session/flush` durability checkpoint for `session`,
 * with the carrier captured at {@link enter}. THE flush entry point: the
 * store owns the carrier, so callers (the checkpoint policy's per-request
 * barrier, goal-round-driver's idle checkpoint, teardown drains, and consumers
 * that flush themselves before reading storage) must come through here
 * rather than dispatch a raw `ctx.parallel('session/flush', …)` — one owner
 * and one spelling.
 * @param session - the session whose buffered events must reach durable storage.
 * @returns whether at least one durability listener participated, after every
 *   listener has settled successfully.
 * @throws the first registered listener failure after every listener settles.
 */
async flush(session: Session): Promise<boolean>

/**
 * Look up a live session.
 * @param id - the session id to look up.
 * @returns the session, or undefined when no live session has that id.
 */
get(id: SessionId): Session | undefined

/**
 * All live sessions, in creation order.
 * @returns a fresh array; mutating it does not affect the store.
 */
list(): Session[]

/**
 * Create an independent live session from an exact prefix of a live source.
 * `boundary` is an inclusive source event seq; omitted means the source's
 * current last event. An open tail receives synthetic tool results and
 * step/turn closers with the forked cause. Closed steps and turns remain
 * unchanged, including any failed tool calls already missing results.
 * `inheritedEventCount` counts only copied source events, excluding these closers.
 *
 * @param source - Live source session object or id.
 * @param boundary - Inclusive source event seq to fork through; omitted means
 *   the source's current last event, and omitted on an empty source forks an
 *   empty child.
 * @param childSessionId - Optional child session id; omitted delegates to
 *   `SessionStore`'s id policy.
 * @returns The created live child session.
 */
fork(source: SessionForkSource, boundary?: SessionSeq, childSessionId?: SessionId): Session
```

型: [CreateSessionOptions](persistence.md) · [PrepareSessionOptions](persistence.md) · [SessionId](core.md)

ソース: [`packages/core/session/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/session/src/index.ts)

<a id="api-session-events"></a>

### `api-session/*` イベント

<a id="api-sessionactivity--emit"></a>

#### `api-session/activity` — emit

ユーザーが作成した永続メッセージ 1 件が Session リストのアクティビティを進めた。

```ts cordis-catalog
/**
 * One user-authored durable message advanced Session list activity.
 * @mode emit
 * @param sessionId - addressed Session identity.
 * @param updatedAt - durable message time used for list ordering.
 */
'api-session/activity'(sessionId: SessionId, updatedAt: number): void
```

型: [SessionId](core.md)

ソース: [`packages/api/session-controller/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/session-controller/src/types.ts)

<a id="api-sessionadded--emit"></a>

#### `api-session/added` — emit

Session が可視になったか、その Agent が作成または破棄された。コンシューマーはサマリーを upsert し、現在の実行状態と可用性状態を置き換える。

```ts cordis-catalog
/**
 * A Session became visible or its Agent was created or disposed.
 * Consumers upsert the summary and replace its current running and availability state.
 * @mode emit
 * @param summary - current list row for the Session.
 */
'api-session/added'(summary: SessionSummary): void
```

ソース: [`packages/api/session-controller/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/session-controller/src/types.ts)

<a id="api-sessionerror--emit"></a>

#### `api-session/error` — emit

1 つの Agent が、永続的なターン位置の外で失敗した。

```ts cordis-catalog
/**
 * One Agent failed outside a durable turn position.
 * @mode emit
 * @param sessionId - Agent and Session identity.
 * @param message - user-safe failure chain.
 */
'api-session/error'(sessionId: SessionId, message: string): void
```

型: [SessionId](core.md)

ソース: [`packages/api/session-controller/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/session-controller/src/types.ts)

<a id="api-sessionremoved--emit"></a>

#### `api-session/removed` — emit

Session がライブの Host レジストリから離れた。

```ts cordis-catalog
/**
 * A Session left the live Host registry.
 * @mode emit
 * @param sessionId - removed Session identity.
 */
'api-session/removed'(sessionId: SessionId): void
```

型: [SessionId](core.md)

ソース: [`packages/api/session-controller/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/session-controller/src/types.ts)

<a id="api-sessionstatus--emit"></a>

#### `api-session/status` — emit

1 つの Agent が実行状態を変えた。

```ts cordis-catalog
/**
 * One Agent changed running state.
 * @mode emit
 * @param sessionId - Agent and Session identity.
 * @param running - whether the Agent is running.
 */
'api-session/status'(sessionId: SessionId, running: boolean): void
```

型: [SessionId](core.md)

ソース: [`packages/api/session-controller/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/session-controller/src/types.ts)

<a id="session-events"></a>

### `session/*` イベント

<a id="sessioncreated--emit"></a>

#### `session/created` — emit

ストアに入った後の読み取り専用の作成アナウンス。すべての同期リスナーが正常に戻るまで、新しいイベントは拒否される。同期的なスローは作成を拒否し、対になる破棄でロールバックする。ディスパッチ中に要求されたデタッチは延期される。返された Promise の拒否はログに記録されるが、この同期境界を遡って拒否することはできない。スコープでフィルターされたディスパッチ（`@deepseek-ai/dsh-scope`）: agent スコープのリスナーは、その agent のコンテキストを通じて入ったセッションだけを受け取る。

```ts cordis-catalog
/**
 * Read-only creation announcement after store entry.
 * New events reject until every synchronous listener has returned successfully.
 * A synchronous throw vetoes creation and rolls back with a paired disposal;
 * detach requested during dispatch is deferred.
 * A returned-promise rejection is logged but cannot retroactively veto this
 * synchronous boundary.
 * Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): agent-scoped listeners
 * receive only sessions entered through that agent's context.
 * @param session - the session just entered and announced.
 * @mode emit
 */
'session/created'(this: Scoped<Session>, session: Session): void
```

型: [Scoped](scope.md)

ソース: [`packages/core/session/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/session/src/index.ts)

<a id="sessiondisposed--emit"></a>

#### `session/disposed` — emit

アナウンス済みのセッションがストアを離れたときに 1 回発行される。公開のロールバックも含むが、作成アナウンスが始まらなかったエントリーでは決して発行されない。リスナーの失敗はログに記録され、封じ込められる。スコープでフィルターされたディスパッチ（`@deepseek-ai/dsh-scope`）は所有者のスコープを再利用する。

```ts cordis-catalog
/**
 * Emitted once when an announced session leaves the store, including
 * publication rollback, but never for an entry whose creation announcement
 * did not begin. Listener failures are logged and contained.
 * Scope-filtered dispatch (`@deepseek-ai/dsh-scope`) reuses the owner scope.
 * @param session - the session that is no longer live in the store.
 * @mode emit
 */
'session/disposed'(this: Scoped<Session>, session: Session): void
```

型: [Scoped](scope.md)

ソース: [`packages/core/session/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/session/src/index.ts)

<a id="sessionevent--emit"></a>

#### `session/event` — emit

コミット後のファイア・アンド・フォーゲットの追記フィード。リスナーのスナップショットはログへのプッシュ前に解決されるが、コールバックはその後に実行される。オブザーバーの失敗はログに記録され、コミット済みの追記を失敗させずに封じ込められる。スコープでフィルターされたディスパッチ（`@deepseek-ai/dsh-scope`）: agent スコープのリスナーは、その agent のコンテキストを通じて入ったセッションのイベントだけを受け取る。

```ts cordis-catalog
/**
 * Post-commit, fire-and-forget append feed. The listener snapshot resolves
 * before the log push, but callbacks run after it; observer failures are
 * logged and contained without making the committed append fail.
 * Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): agent-scoped listeners
 * receive only events from sessions entered through that agent's context.
 * @param session - the session whose log grew.
 * @param event - the appended event, exactly as recorded.
 * @mode emit
 */
'session/event'(this: Scoped<Session>, session: Session, event: SessionEvent): void
```

型: [Scoped](scope.md)

ソース: [`packages/core/session/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/session/src/index.ts)

<a id="sessionflush--parallel"></a>

#### `session/flush` — parallel

待機される並列の永続性チェックポイント。すべてのリスナーが実行され、呼び出し元はそれらすべてを待つ。ウォーターフォールの拒否権はない。スコープでフィルターされたディスパッチ（`@deepseek-ai/dsh-scope`）は、セッションの所有者スコープを再利用する。

```ts cordis-catalog
/**
 * Awaited parallel durability checkpoint: every listener runs and the
 * caller awaits all of them, with no waterfall veto. Scope-filtered dispatch
 * (`@deepseek-ai/dsh-scope`) reuses the session's owner scope.
 * @param session - the session whose buffered events must reach durable storage.
 * @mode parallel
 */
'session/flush'(this: Scoped<Session>, session: Session): Promise<void> | void
```

型: [Scoped](scope.md)

ソース: [`packages/core/session/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/session/src/index.ts)
<!-- END GENERATED cordis-surface -->
