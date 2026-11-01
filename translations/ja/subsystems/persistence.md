# セッション永続化

日本語 | [English](../../../docs/subsystems/persistence.md) | [中文](../../../docs/subsystems/persistence.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/persistence.md](../../../docs/subsystems/persistence.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

イベントログの**永続性 seam**。[session.md](session.md) はインメモリの `Session` — 唯一の真実の源である追記専用の `SessionEvent` ログ — を説明する。このページはそのログがどのように永続化されるかを説明する。抽象 `SessionPersistence` サービス、その provider モデルと同梱の JSONL バックエンド、flush の checkpoint、クラッシュ復旧、そしてログとともに移動するメタデータヘッダーである。ログが運ぶイベント語彙は、生成された [永続化ログのイベントカタログ](../../../docs/persistence-catalog.md) にメンバーごとに列挙されている。

この seam は [capability seam](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-06-13-capability-seams.md) である。既存の `SessionEvent` に対する `create`/`open`/`stat`/`list` を公開する 1 つの抽象サービス（[dsh-session-persistence](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/session/session-persistence)、`ctx.sessionPersistence`）であり、**並行する永続イベント型は存在しない**。`create` と `open` はセッションごとの `SessionHandle`（`read`/`append`/`flush`/`close`）を返し、これがすべてのログアクセスと単一ライター所有権を運ぶ。リポジトリは [dsh-session-persistence-jsonl](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/session/session-persistence-jsonl) を provider として同梱する。ツリー外の provider が同じサービス契約を実装してもよい。[handle ベース永続化の Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-08-27-handle-based-session-persistence.md) と [session-persistence リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-persistence/README.md)を参照。

## `SessionHandle` — 保存済みセッションへの 1 つのオープンチャネル

すべてのログ読み取りと書き込みはハンドルを通り、id でアドレス指定するサービスのメソッドを通ることは決してない。ハンドルはプロセス間の書き込みリースが守る唯一の扉である。読み取りは呼び出し元が所有する外側のスライスと、プロデューサーが確立したイベント値のエイリアシング状態を返す。1 つのハンドル型が両方のアクセスに使われる。`read` ハンドルに対する変更は型で分かれたものではなく実行時の `SessionReadOnlyError` となり、プロセス内の単一ライター所有権により、所有者が活動中の 2 回目の `open(id, 'write')` は `SessionAlreadyOwnedError` で拒否される。

```ts type-equiv
/** One persistence event slice returned by {@link SessionHandle.read}. */
interface SessionHandleReadResult {
  /**
   * Whether event values are exclusively owned or shared only after deep
   * freezing. Slicing preserves the producer's state even when no events remain.
   */
  readonly eventState: SessionSeedEventState
  /** Event values in a caller-owned outer array. */
  readonly events: readonly SessionEvent[]
}
```

```ts type-equiv
/**
 * One open channel onto a stored session. A handle is single-owner state, not
 * a shared service: `read` never backtracks below what this handle already
 * observed, a `write` handle reads its own successful appends, and `close()`
 * is the one teardown (idempotent, uncancellable; `Symbol.asyncDispose`
 * delegates to it). Every operation on a closed handle rejects with
 * `SessionHandleClosedError`.
 *
 * Freshness across handles: once an `append` or `flush` resolves on a write
 * handle, every read STARTED afterwards on the same backend instance — on any
 * handle, or through `stat`/`list` — observes at least that prefix.
 * Reads concurrent with a mutation carry no ordering promise beyond the valid
 * contiguous prefix.
 */
interface SessionHandle extends AsyncDisposable {
  /** The stored session this handle addresses. */
  readonly id: SessionId
  /** The immutable stored header, fixed at `create`/`open`. */
  readonly header: SessionHeader
  /**
   * Exact fork-inherited prefix length stored with the log; `0` when
   * `header.isSeeded` is false. Storage metadata paired with the header for
   * every body read, never part of the replayable event log.
   */
  readonly inheritedEventCount: SessionLogOffset
  /** Whether this handle may mutate the log. */
  readonly access: SessionAccess

  /**
   * Read a slice of the valid contiguous logical log. The slice is a legal log
   * prefix segment: a torn physical tail is never returned, and repeated reads
   * on this handle never observe an older state than a prior read.
   * @param offset - first logical event seq to include; defaults to `0`.
   * @param length - maximum number of events to return; defaults to the rest
   *   of the log. An offset at or past the end returns an empty list.
   * @param options - optional cancellation.
   * @returns the caller-owned outer slice plus the ownership state of its event values.
   */
  read(offset?: number, length?: number, options?: SessionHandleReadOptions): Promise<SessionHandleReadResult>

  /**
   * Append a contiguous batch continuing the current logical end. The first
   * event's `seq` MUST equal the stored next-seq; committed events are never
   * rewritten. Persistence is best-effort: on resolution the batch is
   * accepted, ordered, and visible to reads on this backend instance, but
   * only a resolved {@link flush} promises it survives a crash — a backend
   * may buffer or batch physical writes behind append. Rejects with
   * `SessionReadOnlyError` on a read handle and `SessionOwnershipLostError`
   * when write ownership is gone.
   * @param events - the contiguous batch, in seq order.
   * @param options - optional cancellation observed before the write starts.
   */
  append(events: readonly SessionEvent[], options?: SessionHandleAppendOptions): Promise<void>

  /**
   * The durability barrier — the one operation that promises storage: on
   * resolution every acknowledged append is durable and the session is
   * materialized for other processes; an empty created session becomes
   * durably listable here. Callers that must survive a crash flush; a backend
   * whose `append` already persists on resolution treats this as
   * materialize-if-needed. Rejects with `SessionReadOnlyError` on a read
   * handle.
   * @param options - optional cancellation observed before the barrier starts.
   */
  flush(options?: SessionHandleFlushOptions): Promise<void>

  /**
   * Release the handle: a read handle frees local resources; a write handle
   * completes pending durability and releases write ownership. Idempotent,
   * asynchronous, and deliberately not cancellable.
   */
  close(): Promise<void>
}
```

作成された Session は `create` が解決した瞬間からこのプロセスで観測可能である。一方、バックエンドは物理的な実体化（純粋な最適化）を最初の `append` または `flush` まで遅延させてもよい。他のプロセスは実体化された Session だけを見て、クラッシュ前に実体化されなかった Session は決して存在しなかったことになる。

## flush の checkpoint

`session/event` は*同期的*な通知である。マウントされたバックエンドはセッション id でそれをアクティブな write ハンドルの有界な write-behind ウィンドウに振り分け、プロデューサーをブロックしない（永続化はすでに id ごとに 1 つのアクティブ write ハンドルを強制するため、バックエンドはこれらのリスナーを 1 回だけインストールする）。最初の保留イベントが固定の内部バッチウィンドウを開始し、後続のイベントは期限をリセットせずに参加する。期限切れでセッションの write ハンドルを通じて 1 つの永続 `append` が開始される。その書き込み中に受け入れられたイベントは独自の期限を受け取り、後続バッチを形成する。`session/flush` は待機をキャンセルし、静止するまでドレインするため、ループは依然として、次の通常の turn を主張する前の順序付けとエラー観測の checkpoint としてそれを使う。拒否されたバックグラウンド書き込みはイベントを順序どおりに保持し、自動経路を一時停止し、ロガーを通じて報告される。次の明示的な flush が再試行し、呼び出し元へ大声で拒否を返す。`session/disposed` は同じ最終ドレインを実行してハンドルを閉じ、`close()` 自体もまだ開いているストレージを通じてルーティング済みバッファをドレインするため、バックエンドの teardown の close スイープは何も失わない。ウィンドウが制限するのは意図的なバッチ待機だけであり、イベントループのスケジューリングやバックエンドの永続化レイテンシではない。

## クラッシュ復旧は中断された turn を保持する

turn の途中でクラッシュしたログは、開かれた `turn/start` と `turn/end` なしで終わる。永続化はそれを**切り詰めたり修復したりしない** — 長期的なタスクでは 1 つの turn が非常に大きくなる可能性があり（多くの step、大きな tool 出力）、それらのイベントはクラッシュ前に永続的に追記されていた。物理的に有効な連続ログを返し、決して解決しなかった append に属する破損した物理テールの不完全な断片だけが破棄される。そこから回復された完全なレコード（JSONL バックエンドは破損した Zstandard フレームを部分的にデコードする）は、ハンドルの最初の新しい append の前に書き込み経路によって永続的に書き直される。修復は読み取り側の仕事である。resume（agent-loop）は write ハンドルを通じて保存ログを読み、`interruptedTurnClosers` — 欠落した tool エラー、開いているすべての `step/end`、合成の `turn/end { reason: { kind: 'interrupted' } }` — を計算し、Session を公開する前に同じハンドルを通じて通常のバッチとして追記する。`interrupted` はループが発火しない唯一の `TurnEndReason` である（[session.md](session.md#why-a-turn-ended-turnendreasonmap)を参照）。

したがって修復は write 所有権の下でのみ書き込む。ライブ Session の write ハンドルはそのライフサイクル所有者が保持するため、同時の `open(id, 'write')` は `SessionAlreadyOwnedError` で拒否され、修復がライブの turn と競合しない。読み取り専用の観測者（session-query）は、中断された cold ログを同じクローザーでメモリ内だけバランスを取り、何も書き戻さない。

読み取り専用の観測は `open(id, 'read')` である。ハンドルは検証済みの連続プレフィックススライスを提供し、破損したテールは決して返さず、1 つのハンドルでの繰り返し読み取りは以前の読み取りより古い状態を観測しない。永続化側の prepared Session キャッシュは存在しない。session-query がその cold 読み取りキャッシュを所有し、`stat().revision` の変更トークンで id ごとに 1 つのバランスを取った cold Session をキー付けし、トークンが変わったときだけ再読み取りする。[handle ベース永続化の Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-08-27-handle-based-session-persistence.md) がこのライフサイクルを所有し、アーカイブされた [Session preparation の記録](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/archived/architecture/2026-08-05-session-preparation.md) が元の公開境界 `SessionPreparation` の決定を記録している。

## `SessionLocation` — Session ごとのアーティファクトメタデータ

`SessionLocation` は診断とエクスポートのための観測メタデータである。ログアクセスは依然としてセッションハンドルの `read` を通る。実体化された永続化スナップショットは選択されたアーティファクトパスを運び、`SessionFormatUnsupportedError` はビルドが解釈を拒否した生のログを名指しできる。JSONL はそのプロジェクト/セッションディレクトリ内の絶対トランスクリプトパスを提供する。セッションごとに 1 つのアーティファクトを持たないバックエンドは何も提供しない。

```ts type-equiv
/**
 * A backend-resolved, per-Session local artifact path used by observational
 * snapshots and refusal diagnostics. It names a backend target for the Session
 * and does not authorize reads; log access still goes through a session
 * handle's `read`. It is not persisted Session content.
 */
interface SessionLocation {
  /** Backend-specific artifact kind, for example `jsonl`. */
  readonly kind: string
  /** Absolute path to this session's backend-owned artifact. */
  readonly path: string
}
```

<a id="sessionheader--metadata-beside-the-log"></a>

## `SessionHeader` — ログのそばのメタデータ

セッションごとのメタデータはイベントログから**別に**移動する。ヘッダーはフォーマットバージョン、cwd、`isSeeded` 系譜ビットを運び、本体を持つストレージ値はそのそばに正確な継承カットを運ぶ。どちらも `SessionEventMap` に属さず、`deriveMessages()` に届かない。論理ヘッダーは `session.header` を通じて添付され、Session はそのカットを `inheritedEventCount` として公開する。

ソース: [`packages/core/session/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/session/src/types.ts)

```ts type-equiv
/**
 * Immutable validated storage metadata, kept outside the conversation event log.
 */
interface SessionHeader {
  /**
   * Current logical format version, stamped from {@link SESSION_FORMAT_VERSION}.
   * Historical physical headers are translated before entering this interface.
   */
  readonly version: typeof SESSION_FORMAT_VERSION
  /** The session's id (mirrors the {@link Session}'s id). */
  readonly id: SessionId
  /** Non-negative safe-integer Unix epoch milliseconds when the session was created. */
  readonly createdAt: number
  /** Absolute working directory the session was created in (if any). */
  readonly cwd?: string
  /**
   * Direct creator of a subagent Session. Ordinary forks omit this field;
   * historical ordinary forks may retain their source id here. Presence alone
   * does not classify a Session as a subagent; use `origin` for that distinction.
   */
  readonly parentSession?: SessionId
  /**
   * Whether this Session contains a fork-inherited event prefix. The exact prefix
   * length is Session state rather than ordinary header metadata.
   */
  readonly isSeeded: boolean
  /**
   * Coarse product classification for a session created as a subagent child.
   * This is presentation metadata, not proof that the child is continuable.
   */
  readonly origin?: 'subagent'
  /**
   * Delegation depth: absent (zero) for a top-level session, parent depth + 1
   * for a subagent child. Persisted so a recursion budget survives restart and
   * resume — a runtime-only depth would reset a resumed child to top-level.
   */
  readonly delegationDepth?: number
  /**
   * Id of the agent preset this session's agent was composed from, when the
   * deployment composes per session. Durable because the preset decides the
   * session's tools and prompt: a resume that restored a different composition
   * would replay history the model can no longer act on.
   */
  readonly agentPreset?: string
}
```

## フォーマット拒否 — ビルドが忠実に読めないログ

バックエンドは忠実に解釈できないログを `SessionFormatUnsupportedError` で拒否する。損傷は何もないため `SessionPersistenceCorruptionError` とは別である。`stat` と `list` は最高の正規世代を分類し、サポートされた履歴ヘッダーをその本体を読んだり変更したりせずに変換する。履歴の `open` 呼び出しは、現在の論理値を返す前にセッションごとの 1 つの移行準備を共有し、すべてのソースパス、バイト、inode を変更しないままにする。JSONL provider はそのインメモリ結果から read ハンドルを公開せずに返す。write open は単一ライターの主張とファイルリースを保持したまま準備を再利用し、最終的な現在世代を排他的に公開し、それからようやく書き込み可能ハンドルを返す。将来の最高世代は、より古い読み取り可能な世代が残っていても拒否する。現在フォーマットの復元はインストール済みの拡張と `ignorable: true` を持つ未知のイベントを保持する。履歴 v0/v1/v2 の移行は無視可能とマークされていても未知の型を拒否する。メッセージはバックエンドがセッションごとに 1 つのアーティファクトを保持する場合、選択された生のログパスを追加する。ツリー外のバックエンドは、その物理フォーマットの入口で同等の current-only ハンドル値と方向を意識した拒否を強制しなければならない。[リリース済みフォーマット移行の決定](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-08-31-released-session-format-migrations.md)がチェーンと不変公開のルールを所有する。

## `CreateSessionOptions` — シードとメタデータ

ストアを通じて `Session` を作成するには、`seed`（初期リプレイまたは fork 履歴）、任意の正確な `inheritedEventCount`、`meta`（ストアが `SessionHeader` に折り込むストレージレベルのフィールド）を取る。ストアは `version`/`id` を埋め、`createdAt` をデフォルト設定する。呼び出し元は検証済みの絶対 `cwd`、`parentSession` 系譜、`isSeeded` 系譜ビット、任意の粗い `origin`、`delegationDepth`、`agentPreset`、既存の `createdAt` を指定できる。シード付き作成は、継承プレフィックスに等しい明示的な seed と正確なカットを要求する。コンストラクタはそのカットに子所有のタグ付き end-seed マーカーを追記してから、セットアップが子所有のイベントを追加する。`origin: 'subagent'` はプロダクトのナビゲーションが重複する子行を隠すことを可能にする。それは記述子が有効であることや、子が再開できることを証明しない。

```ts type-equiv
/**
 * Options for creating a {@link Session} via the store. `seed` replays/forks
 * an existing event log; `meta` carries the caller-supplied storage fields the
 * store folds into a {@link SessionHeader}.
 */
interface CreateSessionOptions {
  /** Initial replay or fork history supplied at construction. */
  readonly seed?: readonly SessionEvent[]
  /**
   * Exact fork-inherited prefix length when `meta.isSeeded` is true. The
   * constructor appends the child-owned tagged marker at the cut unless
   * the seed already includes it followed by child-owned fork closers.
   */
  readonly inheritedEventCount?: SessionLogOffset
  /**
   * Storage metadata read once before publication. `isSeeded` marks fork
   * lineage; supplying replay history alone does not make it inherited.
   */
  readonly meta?: {
    readonly cwd?: string
    readonly parentSession?: SessionId
    readonly createdAt?: number
    readonly isSeeded?: boolean
    readonly origin?: 'subagent'
    readonly delegationDepth?: number
    readonly agentPreset?: string
  }
}
```

したがってリプレイ/fork は `ctx.agents.create({ sessionId, seed, meta })` である。fork はさらに `meta.isSeeded: true` とともに `inheritedEventCount` を指定し、agent-loop が公開したセッションだけが永続化され、ループは公開前に新しいセッションの write ハンドルを通じて seed を保存する。*永続化された*セッションをライブ agent に再開するのは `ctx.agents.resume({ resumeSessionId })` である。

## 準備と復元の所有権

`SessionStore.prepare()` は通常の作成オプション、または `RestoredSessionOptions` を通じた採用可能な seed を受け入れる。その `eventState` は、イベント値が独立して所有されているか、深い凍結の後だけ共有されるかを示す。プロデューサーがその状態を確立し、スライスは結果の長さから別の状態を推測しない。復元は別のコピーや凍結パスなしでそれらの値を検証して採用する。その後 `SessionPreparation` が公開またはロールバックまで正確な未公開 Session を所有する。破棄は同期的かつ冪等である。agent-loop の resume はセッションの write ハンドルを通じてこの結果を読み、準備の前に独立して所有する `interruptedTurnClosers` を追記する。

```ts type-equiv
/**
 * Aliasing state of an adoptable Session seed. `shared-frozen` permits deeply
 * frozen aliases plus independently owned unfrozen values in the same seed.
 */
type SessionSeedEventState = 'detached' | 'shared-frozen'
```

```ts type-equiv
/**
 * Adoptable storage values transferred to {@link SessionStore.prepare}
 * without another copy or freeze pass.
 */
interface RestoredSessionOptions {
  /** Events that are independently owned or already deeply frozen. */
  readonly seed: SessionEvent[]
  /** Independently owned storage metadata to validate and freeze in place. */
  readonly meta: SessionHeader
  /** Exact number of fork-inherited leading events decoded from storage. */
  readonly inheritedEventCount: SessionLogOffset
  /** Aliasing state carried from the operation that produced the seed. */
  readonly eventState: SessionSeedEventState
}
```

```ts type-equiv
/** Inputs accepted while constructing an unpublished Session. */
type PrepareSessionOptions =
  | (CreateSessionOptions & { readonly eventState?: undefined })
  | RestoredSessionOptions
```

```ts type-equiv
/** Options for a preparation whose provider retains unpublished state. */
interface SessionPreparationOptions {
  /** Release provider-owned state when the Session was not published. */
  readonly release?: () => void
}
```

```ts public-api
/**
 * One exact unpublished Session and the provider state that keeps it usable.
 * Disposal is synchronous and idempotent. Providers decide whether release
 * returns the Session to a cache or discards it; publication may consume that
 * state before disposal, making the callback a no-op.
 */
declare class SessionPreparation implements Disposable {
  /** The exact Session to use for setup and publication. */
  readonly session: Session;
  /**
   * Wrap an unpublished Session in one preparation lifetime.
   * @param session - exact unpublished Session.
   * @param options - optional provider release behavior.
   * @returns a preparation disposed after publication or rollback.
   */
  static create(session: Session, options?: SessionPreparationOptions): SessionPreparation;
  /** Release provider state once when this preparation leaves its caller. */
  [Symbol.dispose](): void;
}
```

## 軽量なソースリビジョン

派生読み取りモデルの consumer は、完全なイベントログを読み込む前に安価で不透明なリビジョンを比較する。リビジョンは `stat`/`list` からのバックエンドインスタンスごとの変更トークンである。等しいリビジョンは変更されていないログとして扱ってよく、等しくないリビジョンは何も約束せず、write 所有権の入れ替わりは決してリビジョンを変えない。session-query は cold 読み取りキャッシュをそれでキー付けする。トークンは open、read、resume では何の役割も果たさない。

```ts type-equiv
/**
 * Backend-owned token that identifies both one storage source and one revision
 * of a persisted session log.
 */
type SessionPersistenceRevision = Branded<'SessionPersistenceRevision'>
```

```ts type-equiv
/**
 * Lightweight stored-session observation returned by {@link SessionPersistence.stat}
 * and {@link SessionPersistence.list} without reading the full event log.
 */
interface SessionPersistenceSnapshot {
  /** Detached metadata for one stored session. */
  readonly header: SessionHeader
  /** Header-only format classification; absent without versioned artifact metadata. */
  readonly formatStatus?: 'current' | 'migration-required'
  /** Opaque change token; see {@link SessionPersistence.stat}. */
  readonly revision: SessionPersistenceRevision
  /** Logical event count, when the backend can provide it cheaply from metadata; otherwise absent. */
  readonly eventCount?: number
  /** Physical artifact byte size, when the backend can provide it cheaply (JSONL); otherwise absent. */
  readonly sizeBytes?: number
  /**
   * Backend-resolved location of the selected materialized per-Session
   * artifact, for diagnostics and export. Absent when the backend has no
   * materialized per-Session file, including an unmaterialized pending Session.
   */
  readonly location?: SessionLocation
}
```

任意の `eventCount`/`sizeBytes` フィールドは、明示的に必要とする consumer のための安価なバックエンド観測のままである。Session の一覧表示は cold ログを開くためにどちらのフィールドも使わない。ヘッダーと、アイデンティティ検証済みの projection キャッシュヒントだけを読むため、キャッシュや Session フォーマットのアップグレードが起動を本体スキャンに変えることは決してない。

## バックエンド

同梱の provider は抽象 `SessionPersistence` 契約（`create`/`open`/`stat`/`list`、`read`/`append`/`flush`/`close` と随所の任意キャンセルを運ぶセッションごとの `SessionHandle`）を実装し、共有の永続化契約スイートに合格する。

- **[dsh-session-persistence-jsonl](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/session/session-persistence-jsonl)** — セッションごとの追記専用論理 JSONL ログ。デフォルトではチェックサム付きで連結された Zstandard フレームとして、設定により生の行として保存され、クラッシュに安全な原子的実体化、バッチごとの `fsync` 追記、最初の新しい append 前の断裂テールの切り詰めを備える。`stat`/`list` は `sizeBytes` と、`fs.stat` から導出したベストエフォートのリビジョンを運ぶ。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxsessionpersistence--sessionpersistence-abstract-seam"></a>

### `ctx.sessionPersistence` — `SessionPersistence`（抽象 seam）

セッションごとのハンドルを通じてアドレス指定される永続的な追記専用セッションストレージ。

すべてのバックエンドが共有するストレージ意味論: イベントは seq 0 から連続し、決して書き換えられない。破損した物理テールは読み取り者に決して返されず、書き込み経路が最初の append の前に切り詰める。読み取りは現在フォーマットのレコードだけを検証し、未知の語彙を fail-closed で拒否する。`append` はベストエフォートで永続化し、`flush` — ハンドルごとまたはサービス全体 — が永続性の障壁である。

可視性: 作成されたセッションは `create` が解決した瞬間からこのプロセスで `stat`/`list`/`open` を通じて観測可能であり、バックエンドが物理的な実体化を遅延させている間もそうである（純粋な最適化）。他のプロセスはセッションが実体化されて初めてそれを見て、クラッシュ前に実体化されなかったセッションは決して存在しなかったことになる。`SessionHandle.flush` は実体化を強制する。

鮮度: `append` または `flush` が解決すると、その後このバックエンドインスタンスで開始された読み取りは少なくともそのプレフィックスを観測する。

```ts cordis-catalog
/**
 * Create a new stored session and take its write ownership.
 * @param header - the immutable header (id, version, cwd, lineage) to store.
 * @param options - optional cancellation.
 * @returns a `write` handle owned by the caller; close it to release ownership.
 * @throws {SessionAlreadyExistsError} when the id already exists.
 */
abstract create(header: SessionHeader, options?: SessionPersistenceCreateOptions): Promise<SessionHandle>

/**
 * Open an existing stored session.
 *
 * `read` never takes ownership and works while another handle (or process)
 * holds write ownership. `write` atomically claims single-writer ownership;
 * an existing active owner rejects.
 * @param id - the stored session to open.
 * @param access - `read` or `write`.
 * @param options - optional cancellation.
 * @returns the open handle.
 * @throws {SessionPersistenceNotFoundError} when the session does not exist.
 * @throws {SessionAlreadyOwnedError} for `write` when ownership is taken.
 */
abstract open(id: SessionId, access: SessionAccess, options?: SessionPersistenceOpenOptions): Promise<SessionHandle>

/**
 * Flush every active write handle owned by this service instance in one
 * durability barrier: each handle's routed live events drain durably and
 * its session materializes, exactly as that handle's own
 * `SessionHandle.flush` would. Read handles buffer nothing and are
 * untouched. A handle closed concurrently counts as flushed — close itself
 * drains durably.
 * @returns resolution once every write handle active at the call has flushed.
 * @throws {AggregateError} naming each session whose flush failed; the
 *   remaining handles still flush.
 */
abstract flush(): Promise<void>

/**
 * Observe one stored session without reading its event log or taking
 * ownership.
 *
 * The snapshot's `revision` is an opaque change token comparable only
 * against revisions from the same service instance and session id: equal
 * revisions may be treated as an unchanged log; unequal revisions promise
 * nothing. Write-ownership churn does not change a revision. It exists for
 * derived read-model caches keyed off `stat`/`list`; it plays no part in
 * open, read, or resume.
 * @param id - the stored session to observe.
 * @param options - optional cancellation.
 * @returns the snapshot, or `undefined` when the session does not exist.
 */
abstract stat(id: SessionId, options?: SessionPersistenceStatOptions): Promise<SessionPersistenceSnapshot | undefined>

/**
 * List every stored session visible to this process, in no promised order.
 * @param options - optional cancellation.
 * @returns one snapshot per stored session.
 */
abstract list(options?: SessionPersistenceListOptions): Promise<readonly SessionPersistenceSnapshot[]>
```

型: [SessionId](core.md)

ソース: [`packages/session/session-persistence/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-persistence/src/index.ts)
<!-- END GENERATED cordis-surface -->
