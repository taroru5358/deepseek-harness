# バックグラウンド・ジョブ・ランタイム

日本語 | [English](../../../docs/subsystems/jobs.md) | [中文](../../../docs/subsystems/jobs.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/jobs.md](../../../docs/subsystems/jobs.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

長時間実行されるプロデューサー、`ctx.jobs`、ジョブ制御が共有する型。[seam consolidation Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-09-03-jobs-seam-consolidation.md)が現在の設計を、[ランタイム・リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/jobs/jobs/README.md)がその起源を所有する。このページは [`packages/jobs/jobs/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/jobs/jobs/src/types.ts) とクライアント・セーフな [`view.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/jobs/jobs/src/view.ts) リーフから、正確なフィールドとバリアントを記録する。

## ID とステータス

`JobId` は[ブランド付き ID](core.md#branded-ids) であり、`<kind>-<short-uuid>` として生成され、kind の後ろに 16 文字の小文字 16 進 UUID 文字が続く。アクセス制御は ID の秘匿ではなく、所有者の認可に依拠する。`JobKind` はマージ拡張可能なマップから導出され、レジストリは kind を不透明な ID 名前空間として扱う。

```ts type-equiv
/**
 * Producer-defined job kinds. Plugins extend this map by declaration merging;
 * the registry treats every value as an opaque id namespace.
 */
interface JobKindMap {
  bash: 'bash'
}
```

`JobStatus` は `'running' | 'stopping' | 'completed' | 'killed' | 'failed'` である。プロデューサー固有の事実は、ジョブの実行中は `JobView.progress` に、決着後は `JobView.detail` に属する。

<a id="session-lifecycle-records"></a>

## セッション・ライフサイクル・レコード

所有ジョブは、元の所有者 Session に `job/registered` と `job/settled` を発行する。グローバルに一意な `jobId` がそれらのレコードを対にし、ライブ・ジョブを制御する。SDK はこれらのログ専用レコードを、出力の読み取りとは独立に `session.event` を通じて転送する。タイムスタンプはレジストリのコミットを表し、CPU 時間や正確な OS プロセス境界を表すものではない。決着レコードが欠けていれば、不完全なままである。`producerReported` は、プロデューサーが終端結果とともに `done` を履行したことを意味する。強制 teardown や拒否の場合は false であり、リソースの解放を独立に証明するものではない。

`job/settled` は `turn/end` の後に続くことがある。SDK の実行結果は自身の観測ウィンドウだけを対象とする。後の決着を必要とするコンシューマーは、Session のサブスクリプションを保持するか、永続化された履歴を読む。turn の完了は、所有ジョブがすべて停止したことを含意しない。

アプリケーションの teardown は、所有ジョブの最後の決着がストレージに届く前に、Session のイベント・ルーティングを停止したり、そのライターを閉じたりすることがある。これには通常のワンショット完了やシグナルによるシャットダウン、さらにプロセスの突然の消失も含まれる。`job/settled` が欠けている場合は、ライブ・レジストリが終了を報告していても不完全なままである。コンシューマーは終了タイムスタンプを捏造してはならない。

```ts type-equiv
/** Durable identity and registration time of one owned job execution. */
interface JobRegisteredData {
  /** Unique execution id accepted by job controls and retained in Session history. */
  readonly jobId: JobId
  /** Original owner; copied history does not transfer execution ownership. */
  readonly owner: SessionId
  /** Producer kind. */
  readonly kind: string
  /** Producer-supplied label. */
  readonly label: string
  /** Unix epoch milliseconds when registration committed. */
  readonly startedAt: number
  /** Creating tool invocation, when the producer was called through a tool. */
  readonly toolCall?: JobToolCall
}
```

```ts type-equiv
/** Registry settlement, including whether the producer reported a terminal outcome. */
interface JobSettledData extends JobRegisteredData {
  /** Unix epoch milliseconds when the registry committed the terminal record. */
  readonly finishedAt: number
  /** Recorded terminal outcome. */
  readonly status: JobOutcome['status']
  /** Source of the termination request, or producer completion. */
  readonly cause: JobSettleCause
  /** True when done fulfilled; false for rejection or forced failure. Does not certify resource release. */
  readonly producerReported: boolean
  /** Terminal reason, including a recorded kill reason when applicable. */
  readonly detail?: string
}
```

## プロデューサー契約

`JobSpec` は同一性、所有するセッション、任意の pull 型 `output` ソース、およびスターターを宣言する。ランタイムはプリフライトを終えてから、ジョブの `JobHandle` を伴って `run()` を呼び、後から失敗し得るステップなしでコミットする。実行リソースはプロデューサーが所有し、同一性、アクセス、ライフサイクル状態、出力リングはランタイムが所有する。

```ts type-equiv
/** Tool dispatch that created a job; nested calls retain their root invocation. */
interface JobToolCall {
  /** Exact invocation, including a PTC sub-call when applicable. */
  readonly callId: SessionEventMap['tool/call']['callId']
  /** Root model-requested invocation owning this execution. */
  readonly rootCallId: SessionEventMap['tool/call']['callId']
}
```

```ts type-equiv
/**
 * Producer declaration passed to {@link JobRegistry.start}. The runtime
 * preflights access and cleanup before invoking {@link run}; the producer owns
 * execution resources while the runtime owns identity, lifecycle state, and
 * the output ring.
 */
interface JobSpec {
  /** Producer kind — also the id prefix (`bash`, …). */
  kind: JobKind
  /** One-line model-facing label (the command; the delegation description). */
  label: string
  /**
   * Owning session. Access is fenced by it, and the owner's live Agent must be
   * the one currently registered under that id: its disposal cancels and
   * awaits the job. Omitting the owner creates an unowned job, open to any
   * caller until service disposal.
   */
  owner?: SessionId
  /** Creating tool invocation, retained in the durable lifecycle records. */
  toolCall?: JobToolCall
  /**
   * Optional UTF-8 byte cap for each complete model-facing completion notice or
   * output read, including controller status metadata. Independent of ring
   * retention: it bounds the consuming model surface, never observers.
   */
  outputLimitBytes?: number
  /**
   * Pull sources the registry pumps into the ring at its own cadence.
   * Producers that narrate their own progress use {@link JobHandle.append}
   * instead; a job may use both.
   */
  output?: readonly JobOutputSource[]
  /**
   * Start the work after preflight and synchronously return its hooks. Called
   * once with the job's producer face; a throw leaves nothing registered (the
   * spent ordinal is skipped), and the producer must clean up any partially
   * started resources.
   * @param job - the issued id plus the ring append and progress writers.
   */
  run(job: JobHandle): JobHooks
}
```

```ts type-equiv
/**
 * Producer face of one registered job, handed to {@link JobSpec.run} and
 * valid for the job's whole life. All methods are synchronous. Writes staged
 * inside the starter call are retained and become visible with the
 * registration commit; after settlement — the producer's own outcome, a kill,
 * or a registry-forced teardown end — writes log and drop instead of
 * throwing, so a producer's trailing flush cannot break its own teardown path.
 */
interface JobHandle {
  /** The registry-issued id (`<kind>-<short-uuid>`). */
  readonly id: JobId
  /**
   * Append one chunk to the output ring. Offsets advance by the chunk's UTF-8
   * byte length; an empty chunk is dropped without waking observers.
   * @param text - the chunk text, exactly as produced.
   * @param options - stream label and gap marker.
   */
  append(text: string, options?: JobAppendOptions): void
  /**
   * Replace the live progress line (`3/10`, the current phase). Settlement
   * clears it; the terminal reason travels in {@link JobOutcome.detail}.
   * @param line - the new progress line.
   */
  updateProgress(line: string): void
}
```

`JobHooks.done` は、作業が終わっただけではなく、プロデューサーがリソースを解放した後に解決する。結果がストリームではなく値であるジョブ（ワークフローのレンダリング結果など）は、それを `JobOutcome.result` として返す。決着後のモデルの最初の読み取りが、それを 1 回だけ運ぶ。

```ts type-equiv
/** Hooks through which the runtime controls and observes producer work. */
interface JobHooks {
  /**
   * Request termination. Must be synchronous, idempotent, and eventually settle
   * {@link done}; throws propagate. The optional reason is forwarded verbatim.
   */
  cancel(reason?: string): void
  /**
   * Resolves after the producer releases its resources, not merely when work
   * finishes. Must not reject; the runtime converts a rejection to `failed`.
   * If teardown cancellation throws, the runtime may force-fail only the
   * registry record without claiming that the work stopped.
   */
  done: Promise<JobOutcome>
}
```

```ts type-equiv
/** Terminal result supplied by a producer through {@link JobHooks.done}. */
interface JobOutcome {
  /** How the job ended: finished (`completed`), cancelled (`killed`), or broke (`failed`). */
  status: 'completed' | 'killed' | 'failed'
  /**
   * Terminal reason rendered into status lines (`exit code: 3`, `max-tokens`).
   * When the job settles `killed` after a {@link JobRegistry.kill} with a
   * reason, the registry appends that reason.
   */
  detail?: string
  /**
   * Return value for jobs whose result is a value rather than a stream (a
   * workflow's rendered result, a subagent's report). The output ring carries
   * the stream; this is handed out once by the model's next {@link JobRegistry.read}.
   */
  result?: string
}
```

pull ソースはレジストリに、消費しないオフセット・リーダー（サブプロセスの `readFrom` ファミリー）を渡す。レジストリは各ソースを独自のケイデンス（`dsh-jobs-local` では `pumpPollMs`）で汲み出し、決着がリングを閉じる前にもう一度ドレインする。したがってプロデューサーは `done` に何も畳み込まない。

```ts type-equiv
/**
 * A pull source the registry pumps into the job's output ring — the subprocess
 * `readFrom` family. The registry owns the cadence and drains every source one
 * last time before settlement closes the ring, so a producer folds nothing
 * into its `done`.
 */
interface JobOutputSource {
  /** Stream label attached to every chunk this source yields. */
  channel?: JobChannel
  /**
   * Read everything captured since `fromByte` without consuming it.
   * @param fromByte - whole-stream offset to resume from (a prior read's `nextOffset`; 0 first).
   * @returns the delta text, the next offset, the lossy flag, and the spill path the source currently keeps.
   */
  read(fromByte: number): JobSourceRead
}
```

```ts type-equiv
/** One incremental read from a {@link JobOutputSource}. */
interface JobSourceRead {
  /** Text captured since the requested offset (the whole retained tail when lossy). */
  text: string
  /** Whole-stream offset to resume from on the next read. */
  nextOffset: number
  /** True when the requested offset slid out of the source's retained window. */
  lossy: boolean
  /**
   * Host path of a file holding the complete stream, when the source
   * currently keeps an intact one. Reported on every read, so the registry
   * tracks it as source metadata: a later read without it withdraws the file.
   */
  spillPath?: string
}
```

## 出力リング

すべてのジョブは 1 つの境界付きリングを所有する。pull ソースはそこへ汲み出され、`JobHandle.append` の push はそのまま着地する。モデルはレジストリが保持するカーソル（`JobRegistry.read`）を通じてリングを消費し、任意の数の観測者が絶対バイト・オフセット（`JobRegistry.readAt`）でそれを読み、どちらも他方を妨げない。`JobChannel` は `stdout`、`stderr`、`log` にラベルを付ける。`log` はプロデューサーの語りであり、観測者にだけ届き、モデルの消費読み取りには決して届かない。決着はストリームを終わらせ、保持量を決着後の上限まで切り詰める。リングには別個のライフサイクルはない。pull ソースが保持するスピル・ファイルはジョブのメタデータ（`JobView.output.spillPaths`。汲み出しの読み取りのたびに更新される）であり、チャンク単位のメタデータではない。したがってモデルのドロップ出力通知は、リングがバイトを追い出した後も、ギャップ・チャンク自体がなくなった後でも、そのファイルを名指せる。ブラウザは `job.list` と `job.follow`（[`dsh-api-job-controller`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/job-controller/README.md) の Remote ストリーム）を通じて名簿とリングに到達する。それらのフレームは下記の Cordis API セクションに列挙されている。

```ts type-equiv
/** One chunk of a job's output ring: absolute offset, text, channel, and loss marker. */
interface JobChunk {
  /** Absolute offset of the chunk's first byte; offsets never move once assigned. */
  readonly at: number
  /** Chunk text exactly as appended (possibly tail-trimmed by retention). */
  readonly text: string
  /** Stream label, when the producer supplied one. */
  readonly channel?: JobChannel
  /** Bytes immediately before this chunk were lost, at the producer or to retention. */
  readonly gapBefore?: true
}
```

```ts type-equiv
/** Result of one non-consuming {@link JobRegistry.readAt}. */
interface JobOutputRead {
  /** Retained chunks overlapping `[from, total)`, in offset order. */
  chunks: readonly JobChunk[]
  /**
   * Offset to resume from — the ring's current `total`. Always a chunk
   * boundary: appends land whole and trimming only advances chunk starts, and
   * consumers concatenate `chunks` under that assumption, so a provider
   * serving partial chunks would silently duplicate text.
   */
  next: number
  /** True when `from` fell below the oldest retained byte, so bytes are missing before `chunks`. */
  lossy: boolean
}
```

## コンシューマー・ビュー

`JobView` は、すべての読み手が消費する唯一の投影である。モデル向けツール、ブラウザの名簿、観測ストリームである。`owner` はアクセスを囲い込むセッション ID を運び、レジストリはライフサイクル・クリーンアップのためにその背後のライブ `Agent` を解決するが、オブジェクト自体を外に渡すことはない。`progress` はプロデューサーのライブ行であり、決着時にクリアされる。`detail` は終端理由であり、記録された kill 理由がマージされる。

```ts type-equiv
/**
 * Read-only projection of one job — a fresh object per call, never live
 * registry state. The model tools, the browser roster, and the observation
 * stream all consume this one shape.
 */
interface JobView {
  /** The registry-issued id (`<kind>-<short-uuid>`). */
  readonly id: JobId
  /**
   * The producer kind the job was registered with: a Host-registered
   * `JobKind`, carried as an open string because a browser bundle or a Remote
   * codec sees only the `JobKindMap` merges its own program compiles.
   */
  readonly kind: string
  /** The producer-supplied one-line label. */
  readonly label: string
  /** Owning session; absent for an unowned job, which every caller can see. */
  readonly owner?: SessionId
  /** Producer-owned cap for complete model-facing notices and reads, in UTF-8 bytes. */
  readonly outputLimitBytes?: number
  /** Current lifecycle state. */
  readonly status: JobStatus
  /** The producer's live progress line (`3/10`, the current phase); cleared at settlement. */
  readonly progress?: string
  /** Terminal reason (`exit code: 3`); a recorded kill reason is merged in. */
  readonly detail?: string
  /** Epoch ms when the job was registered. */
  readonly startedAt: number
  /** Epoch ms when the job settled; absent while live. */
  readonly finishedAt?: number
  /**
   * The output ring's absolute coordinates and the complete-stream files
   * behind it. `total` is the offset the next chunk starts at (0 while
   * nothing was written); `earliest` is the oldest retained byte, greater
   * than zero exactly when retention dropped the head. `spillPaths` lists the
   * spill files the job's pull sources currently keep, in source order and
   * deduplicated, and is absent while no source keeps one: it outlives any
   * chunk, so a reader below `earliest` can still name where the bytes went.
   */
  readonly output: { readonly total: number; readonly earliest: number; readonly spillPaths?: readonly string[] }
}
```

`JobRegistry.list`、`get`、`read`、`readAt`、`kill`、`wait` はそれぞれ呼び出し元の `SessionId` を受け取る。省略すると、所有されていないジョブだけにアクセスでき、セッションは自身のジョブにもアクセスできる。

```ts type-equiv
/** Output and post-read state returned by the consuming {@link JobRegistry.read}. */
interface JobRead {
  /** Ring chunks appended since the model cursor, in offset order; every channel included. */
  chunks: readonly JobChunk[]
  /** True when the cursor fell below the oldest retained byte, so bytes are missing before `chunks`. */
  lossy: boolean
  /** The producer's {@link JobOutcome.result}, handed out by the first read after settlement only. */
  result?: string
  /** The job's state at read time. */
  job: JobView
}
```

## イベント

レジストリはすべてのコミットを 1 つのフィルター済みストリームで通知する。ライフサイクル・イベントは、それが通知するコミット後の投影を運ぶ。`settled` はその原因を名指すため、完了レポーターは teardown をスキップできる。`output` は ID と新しい total だけを運ぶため、観測者は自身のカーソルから読み取り、レジストリがペイロードを push することは決してない。

```ts type-equiv
/**
 * One lifecycle or output event. Lifecycle events carry the job's projection
 * after the commit they announce; `output` carries only the id and the new
 * total, so an observer schedules a {@link JobRegistry.readAt} from its own
 * cursor and the registry never pushes payloads.
 */
type JobEvent =
  | {
    /** Registration commit, progress line change, stopping transition, or removal from the visible set. */
    readonly type: 'registered' | 'progress' | 'stopping' | 'removed'
    readonly job: JobView
  }
  | {
    readonly type: 'settled'
    readonly job: JobView
    readonly cause: JobSettleCause
    /**
     * Whether this settlement released a live {@link JobRegistry.wait}. That
     * waiter's caller receives the terminal projection as its own result, so
     * a completion reporter treats an awaited settlement as already delivered
     * and reports only the unawaited ones. A wait that timed out or was
     * aborted before the settlement does not count.
     */
    readonly awaited: boolean
  }
  | {
    readonly type: 'output'
    readonly id: JobId
    /** Owning session, absent for an unowned job. */
    readonly owner?: SessionId
    /** The ring's total after the append (or at settlement, which ends the stream). */
    readonly total: number
  }
```

```ts type-equiv
/**
 * Who a subscription hears about. `{ owner }` delivers that session's jobs
 * plus every unowned job (the set that session can see). `{ owners: 'scope' }`
 * delivers the owners composed under the subscribing context — one registry
 * serves every composition in the process, and a mount under one preset must
 * not hear another preset's agents. `{ owners: 'all' }` delivers everything.
 */
type JobEventFilter =
  | { readonly owner: SessionId }
  | { readonly owners: 'all' | 'scope' }
```

## サービス挙動

抽象的な [`JobRegistry`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/jobs/jobs/src/index.ts) Service Definition は、アトミックな `start`、呼び出し元スコープの `list`、`get`、消費型の `read`、非消費型の `readAt`、`kill`、境界付きの `wait`、フィルター済みの `events` ストリーム、`attachController` を規定する。[`LocalJobRegistry`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/jobs/jobs-local/src/index.ts) はプロセス・ローカルの Service Provider である。認可は所有者セッションを比較し、所有者のクリーンアップと受け入れは、ジョブ開始時にその所有者セッションの下で登録されているライブ `Agent` を使う。ローカル・プロバイダーの `maxConcurrentJobsPerOwner` 設定（正の安全な整数）のデフォルトは `10` で、正確な所有者ごとに `running` と `stopping` のレコードを数え、所有されていないジョブには 1 つの共有バケットを使う。プロデューサーの終端決着が容量を解放する。`retainBytes`（デフォルト 262144）と `settledRetainBytes`（デフォルト 16384）は各リングのライブ保持量と決着後保持量を制限し、`pumpPollMs`（デフォルト 150）は pull のケイデンスである。Service Definition の契約は [`dsh-jobs`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/jobs/jobs/README.md)、レジストリのライフサイクルと受け入れポリシーは [`dsh-jobs-local`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/jobs/jobs-local/README.md)、モデル向けコンシューマーは [`dsh-tool-jobs`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/jobs/tool-jobs/README.md) を参照。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxjobcontroller--jobcontroller"></a>

### `ctx.jobController` — `JobController`

生成された `ctx.remote.job` 名前空間を支える Host サービス。

```ts cordis-catalog
/**
 * Stream the jobs one session can see — its own plus every unowned job —
 * as whole-set frames: one on open, then one after each coalesced burst of
 * lifecycle commits. The stream has no natural end; the carrier closes it.
 * @param request - the session whose visible set to mirror.
 * @param signal - cancellation owned by the Remote stream carrier.
 * @returns the roster frames.
 */
@Remote({ mode: 'stream' }) list(request: JobListRequest, signal: AbortSignal): AsyncIterable<JobListFrame>

/**
 * Stream one job's retained output from an absolute byte offset, then its
 * terminal projection once settled and drained. Non-consuming: the
 * model-facing cursor and notice state never observe these reads. The
 * request's session is the fenced read's caller; the registry rejects a
 * job the session cannot see and an unknown job.
 * @param request - target job, owning session, and optional resume offset.
 * @param signal - cancellation owned by the Remote stream carrier.
 * @returns anchor, coalesced output frames, and the terminal status.
 */
@Remote({ mode: 'stream' }) follow(request: JobFollowRequest, signal: AbortSignal): AsyncIterable<JobFollowFrame>

/**
 * Kill one background job on a human's behalf. The request's session is
 * the fenced read's caller, so the job must be one that session can see:
 * the registry's owner fence is the only access rule, and a child session's
 * own jobs are killable from its list like any other. The kill records
 * `cancelled by the user` as its reason; it is not one the model requested,
 * so the owning agent still receives the completion notice, and a shell
 * tool waiting on that job reads the reason in its own result.
 * @param request - Session whose job list carries the job, and the job id.
 * @returns the registry's admission of the kill request.
 */
@Remote('kill') kill(request: JobKillRequest): JobKillValue
```

ソース: [`packages/api/job-controller/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/job-controller/src/index.ts)

<a id="ctxjobs--jobregistry-abstract-seam"></a>

### `ctx.jobs` — `JobRegistry`（抽象 seam）

抽象的なバックグラウンド・ジョブ・レジストリ。サブクラス化し、抽象メンバーを実装して、そのサブクラスを plugin としてロードする。`ctx.jobs` として登録される（コンテキストごとに 1 実装。2 つ目をロードするとスローする。これは cordis 標準の重複サービス挙動である）。

実装は次のセマンティクスを守らなければならない。

- 登録はプロデューサーとコントローラーの fiber より長生きする。所有者とサービスの破棄はライブ作業をキャンセルし、準拠したプロデューサーを待つ。teardown のキャンセルがスローした場合、レコードだけが強制的に失敗する。そのような決着は `cause: 'teardown'` を通知する。所有者が破棄されているジョブには読み手が残っていないからである。
- 所有ジョブへのアクセスは所有者のセッション ID で囲い込まれる。ID は予測可能であるため、境界は秘匿ではなく認可である。
- 決着は先勝ちである。1 つの終端レコード、解放された待機者、続いて 1 回のコンテインメントされたイベント配信が、遅れて届くプロデューサーの結果に対しても行われる。`settled` イベントは解放されたすべての待機者の後に続き、待機者を解放したか（`awaited`）を報告する。したがって完了レポーターは、待機中の呼び出し元がすでに収集した決着をスキップできる。
- 決着済みレコードは、その所有者の破棄、サービスの破棄、または終端状態を自ら収集して ID を外に渡さなかった呼び出し元による明示的な削除まで、一覧に残る。
- start は、spec の所有者にサービスを提供する接続済みジョブ・コントローラーがない間は作業を拒否する。したがってプロデューサーは、所有者が収集も停止もできない作業を開始できない。1 つのレジストリがプロセス内のすべての構成にサービスを提供するため、この問いと `{ owners: 'scope' }` の下でのイベント配信は、プロセス全体ではなく所有者相対である。スコープのないコンテキストから行われた登録はすべての所有者にサービスを提供し、agent 構成のスコープの下で行われた登録は、その下で構成された agent に正確にサービスを提供する。
- すべてのジョブは 1 つの出力リングを所有する。spec が名指す pull ソースはレジストリが汲み出し、決着前にさらに 1 回ドレインする。push された append はそのまま着地する。モデルの消費カーソルと観測者の絶対オフセットは同じバイトを読み、互いを妨げない。
- リングの保持量は境界付きである。ライブ上限を超える append は最も古い保持バイトを落とす。保持ウィンドウより下の読み手は、エラーではなくロッシーな読み取りを得る。決着は保持量を決着後の上限まで切り詰めてストリームを終わらせる。リングには別個のライフサイクルはない。

```ts cordis-catalog
/**
 * Preflight access, validation, owner cleanup, and implementation-owned
 * admission before starting and atomically registering work. Any preflight
 * rejection leaves no job id or execution resource. A throwing starter
 * leaves nothing registered; after it returns, registration cannot fail.
 * @param spec - job identity, owner, output sources, and synchronous starter.
 * @returns the registry-issued `<kind>-<short-uuid>` id.
 */
abstract start(spec: JobSpec): JobId

/**
 * List caller-owned and unowned jobs in registration order.
 * @param caller - reading session; omission sees only unowned jobs.
 * @returns fresh projections.
 */
abstract list(caller?: SessionId): JobView[]

/**
 * Project one job without changing its cursor. Throws for an unknown or
 * foreign job.
 * @param id - job to look up.
 * @param caller - reading session checked against the owner.
 * @returns a fresh projection.
 */
abstract get(id: JobId, caller?: SessionId): JobView

/**
 * Consume the ring from the model cursor and advance it to the current
 * total. After settlement the first read also carries the producer's
 * result. Throws for an unknown or foreign job.
 * @param id - job to read.
 * @param caller - reading session checked against the owner.
 * @returns the chunks since the cursor, the lossy flag, the result once, and the post-read projection.
 */
abstract read(id: JobId, caller?: SessionId): JobRead

/**
 * Read retained ring output without moving the model cursor. Resume with
 * a previous read's `next`; an offset inside a retained chunk returns the
 * whole chunk (its `at` may precede `from`). Throws for a negative or
 * non-integer offset, or an unknown or foreign job.
 * @param id - job to read.
 * @param from - absolute byte offset to read from (0 for the retained head).
 * @param caller - reading session checked against the owner.
 * @returns retained chunks overlapping `[from, total)`, the resume offset, and the lossy flag.
 */
abstract readAt(id: JobId, from: number, caller?: SessionId): JobOutputRead

/**
 * Request cancellation, then mark the job stopping. A producer throw
 * propagates without changing job state. A supplied reason is merged into
 * terminal `detail` when the job settles `killed`. Throws for an unknown
 * or foreign job.
 * @param id - job to cancel.
 * @param caller - killing session checked against the owner.
 * @param reason - cancellation reason forwarded verbatim to the producer.
 * @returns `requested` for live work, otherwise `already-finished`.
 */
abstract kill(id: JobId, caller?: SessionId, reason?: string): 'requested' | 'already-finished'

/**
 * Wait for settlement or timeout without cancelling the job. Caller abort
 * rejects only while the job is live; after settlement the terminal
 * projection wins. Rejects for an invalid timeout or an unknown or foreign
 * job.
 * @param id - job to wait for.
 * @param timeoutMs - positive finite wait bound in milliseconds.
 * @param caller - waiting session checked against the owner.
 * @param signal - optional cancellation of the wait itself.
 * @returns projection at settlement or timeout.
 */
abstract wait(id: JobId, timeoutMs: number, caller?: SessionId, signal?: AbortSignal): Promise<JobView>

/**
 * Drop one settled job's record from the visible set and announce
 * `removed`. For a caller that collected the terminal state through its own
 * {@link wait} and never handed the id to the model, such as a shell tool's
 * foreground call. Throws for a job that is still live, unknown, or foreign.
 * @param id - settled job to drop.
 * @param caller - removing session checked against the owner.
 */
abstract remove(id: JobId, caller?: SessionId): void

/**
 * Attach an effect-scoped controller that can read and stop jobs. It serves the
 * owners its registering context's scope covers, and {@link start} refuses an
 * owner no attached controller serves.
 * @param name - diagnostic label; duplicate names remain independent.
 * @returns disposer that detaches this controller.
 */
abstract attachController(name: string): () => void
```

型: [SessionId](core.md)

ソース: [`packages/jobs/jobs/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/jobs/jobs/src/index.ts)
<!-- END GENERATED cordis-surface -->
