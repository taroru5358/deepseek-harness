# コア

日本語 | [English](../../../docs/subsystems/core.md) | [中文](../../../docs/subsystems/core.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/core.md](../../../docs/subsystems/core.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

**core** サブシステムは [`packages/core`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/README.md) である。これはすべてのコンポジションが起動するパッケージ群であり、イベントソーシングのセッションログ、システムプロンプトの組み立て、ツールレジストリ、agent の型、およびそれらを駆動する具体的なループを含む。このページでは、`agent`/`agent-loop` の組が宣言するもの、すなわち agent がどのように生成・所有されるか、`Agent` ハンドルの配信・キャンセル・傍受の契約、さらにすべてのサブシステムが従う 2 つの型パターンを説明する。このグループの専用ページとフォルダーの残りは [subsystems README](index.md) に索引がある。

## パッケージごとの骨格

1 つのターンは 6 つのパッケージを 1 つのループで流れる。[`agent-loop`](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/core/agent-loop) のドライバーがキュー済みのプロンプトを取り出し、[セッションログ](session.md)（`ctx.sessions`）でターンを開き、[system-prompt](system-prompt.md)（`ctx.systemPrompt`）を通じてリクエストプレフィックスを組み立ててログから履歴を派生させ、モデル応答を [LLM seam](llm-streaming.md) を通じてストリームし、ツール呼び出しを[ツールレジストリ](tools.md)（`ctx.tools`）を通じてディスパッチし、モデルから見えるすべての事実を、次のステップがそこから派生する前にログへ追記する。ループが動かす会話の語彙（`Message`、`ContentBlock`、`StreamChunk`、モデルリクエスト）は [`packages/llm`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/README.md) が宣言し、[llm-streaming.md](llm-streaming.md) で文書化されている。

| パッケージ | 責務 | ページ |
|---|---|---|
| `session/` | 追記専用の `SessionEvent` ログとインメモリストア — 唯一の真実の源（`ctx.sessions`） | [session.md](session.md) |
| `system-prompt/` | プロンプトセクションとツールスキーマの組み立て（`ctx.systemPrompt`） | [system-prompt.md](system-prompt.md) |
| `tools/` | スコープ付きツールレジストリとガード付き実行パイプライン（`ctx.tools`） | [tools.md](tools.md) |
| `agent/` | `Agent` インターフェース、ライブレジストリ、イニシエータースコープ、`agent/*` イベント語彙（`ctx.agents`） | このページ |
| `agent-loop/` | 公開 `Agent` 契約を実装する具体的なドライバー（`ctx.agentLoop`） | このページ |
| `scope/` | レジストリとループが agent ごとのスコープを構築する基盤のスコープ付き登録プリミティブ | [scope.md](scope.md) |

`scope/` は唯一の非サービスパッケージである。依存関係のないライブラリ（`createScope`/`scopeOf`/`scopeTarget`）であり、モジュールグラフ上で `session/` と `system-prompt/` の下に位置する。これはまさに、それらが循環なしで消費できるようにするためである。`agent-loop` は公開 `Agent` 契約の唯一の具体的な実装であり、harness のデフォルト製品ループであるためここに存在する。各ドライバーを `ctx.agents.withInitiator()` の中で実行する。拡張プラグインは `agent` に依存し（開始元の Agent が必要な場合も含む）、`agent-loop` に直接依存することは決してないため、ループは差し替え可能なままである。[`dsh-base`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/base/README.md) はデフォルトの製品コンポジションであり、[`dsh-sdk-minimal`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/sdk-minimal/README.md) はより小さなスタンドアロンツリーを宣言する。

## 生成と所有

コンシューマーは `ctx.agents` を通じて agent を生成する（`create()` は呼び出し元が供給した 1 つの `SessionId` の下で新しいセッションと agent を構築し、`resume()` は先に永続化されたセッションをロードする）。またはループの設定エントリーを通じて宣言的に生成する。プログラムによる生成は所有者のハンドルを返す:

ソース: [`packages/core/agent/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent/src/index.ts)

```ts type-equiv
/**
 * An owned agent plus its disposer, returned by {@link AgentRegistry.create} /
 * {@link AgentRegistry.resume}. The disposer is a CAPABILITY: among consumers,
 * only the holder can tear this agent down. The registered factory provider is
 * also a structural owner because the scoped agent depends on that provider's
 * service API; provider unload stops and drains every live handle it made.
 * `dispose()` stops the loop, awaits its exit, unregisters the agent, removes
 * its session from the store, and finally unwinds its scoped world.
 *
 * `ctx.agents.get(id)` still returns a bare {@link Agent} — the handle is
 * exposed only to the consumer owner that created it; the structural provider
 * reaches the same teardown internally. Config-created agents (the loop's own
 * startup) are owned by the loop fiber and never need a handle.
 */
interface AgentHandle {
  agent: Agent
  dispose(): Promise<void>
}
```

`CreateAgentOptions` は共有の同一性と、新しい agent が公開前に必要とするすべてを運ぶ。任意のライブ `parentAgent`、セッションメタデータ（`meta` — 検証済みの `cwd`、フォーク系譜、`isSeeded` マーカー、origin 分類、委譲の深さ、`agentPreset`）、兄弟フィールド `inheritedEventCount` における正確なフォークの切断位置、任意の `seed` リプレイプレフィックス、agent ごとの `AgentOptions`、生成専用のキャンセル `signal`、および `setup` である。`ResumeAgentOptions` は永続同一性の対応物であり、`resumeSessionId`、`parentAgent`、`agentOptions`、`signal`、`setup` を持つ。`setup` コールバック（`AgentSetup`）は、Session ストアへのエントリーと `session/created` の完了後、Agent 登録の前に `(agentCtx, agent)` を受け取る。コンテキストはスコープ付き登録を所有し、明示的な Agent は Context 上の逆プロパティなしで正確な子 Session を供給する。`agentCtx` を通じて登録されたすべては、`agent/created` と最初のプロンプト組み立ての前に存在する。setup は公開直前に呼び出される同期コミットを返してよい。setup の拒否、コミットのスロー、所有者の破棄はトランザクションを巻き戻し、Session を削除し、復元された保留入力とすでに保存されたイベントを保持する。

`AgentFactory` はレジストリの背後にある生成インターフェースである。ループは `ctx.agents.setFactory()` を通じて自身のファクトリーを登録するため、コンシューマーは具体的なループパッケージに依存せずに `ctx.agents` を使う。ランタイムの子生成者は `options.parentAgent` を設定し、レジストリは一方から他方を導出することなくオプションと呼び出し元 Context をファクトリーに渡す。正確な `create`/`resume` のシグネチャとロールバック契約は、下記の[生成セクション](#ctxagents--agentregistry)にある。

## agent ハンドル

`Agent` はすべてのプラグイン（UI、hook、オーケストレーター）がプログラムする対象のサーフェスである。`ctx.agents.get(id)` がそれを返し、[イニシエータースコープ](#initiating-agent)がそれを運ぶ。具体的な実装は dsh-agent-loop のパッケージ内部であり、ループの外の何もそれに依存しない。統一された `send` メソッドはターゲットとウェイクアップのルーティングを直接公開し、`followup`、`steer`、`inject` は固定プリセットのエイリアスである。

ソース: [`packages/core/agent/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent/src/types.ts)

```ts type-equiv
/** Public live-agent handle; the runtime face augments its live capabilities. */
interface Agent {
  /** Session-backed Agent identity. */
  readonly id: SessionId
  /** Route fallback and output controls; live accessors apply until the Session records a selection or request. */
  readonly options: AgentOptions
  /** The live session this agent drives; its log is the durable source of truth. */
  readonly session: Session
  /** Agent-owned access to durable model selection for future steps. */
  readonly model: Model
  /** Agent-owned access to durable pending work. */
  readonly inbox: Inbox
  /** The current lifecycle state, mirrored on every `agent/status` transition. */
  readonly status: AgentStatus
  /** Agent-scoped context; its contributions are agent-local, unwind on disposal, and reject registration afterward. */
  readonly ctx: Context

  /**
   * Clear queued and steering work — unless `keepInbox` — and abort the active
   * turn or between-turn task. The first cause wins for that activity. With no
   * active activity, cancellation is a no-op and does not arm later work.
   * @param cause - the stable caller intent carried by the active operation signal.
   * @param options - cancellation options; `keepInbox` preserves pending work.
   */
  cancel(cause: AgentCancelCause, options?: CancelOptions): void

  /**
   * Resolve after the current whole-agent activity reaches quiescence. This
   * follows replacement work started before the observed driver retires,
   * but does not identify the settlement of any particular message.
   * @returns fulfillment after no active driver or maintenance task remains.
   */
  whenIdle(): Promise<void>

  /**
   * Run one non-turn maintenance task from the true idle phase. The task starts
   * synchronously after claiming that phase; later waking input remains in the
   * inbox until the task settles, while public status stays `idle`.
   * `whenIdle()` follows both the task and any waking work released behind it.
   * @param task - operation whose fulfillment or rejection is preserved, with a signal aborted by {@link cancel}.
   * @throws synchronously when turn-driving or another maintenance task already owns the agent.
   * @returns the task promise.
   */
  runMaintenance<T>(task: (signal: AbortSignal) => Promise<T>): Promise<T>

  /**
   * Route identified input to an inbox boundary and optionally wake the driver.
   * Waking input submitted after active cancellation is queued for the next
   * turn and runs when the aborted activity converges to idle; a `disposed`
   * cancel leaves it parked. A wake submitted while already idle always opens
   * its turn boundary, even when its message is cleared before the driver
   * claims ([driver wake convergence](../../agent-loop/src/agent.ts)).
   * @param message - identified content and the source that supplied it.
   * @param target - the preferred next-turn or next-step inbox boundary.
   * @param wakeup - whether delivery may wake the driver.
   */
  send(message: UserMessage, target: InboxTarget, wakeup: boolean): void

  /**
   * Queue an ordinary follow-up turn and wake the driver. The item becomes the
   * sole ordinary message of its own turn.
   * @param message - identified prompt content and the source that supplied it.
   */
  followup(message: UserMessage): void

  /**
   * Submit steering for the nearest step. An idle driver starts a turn;
   * a running driver consumes it at its next step boundary.
   * A rejected step leaves steering parked in the inbox until the next
   * wake; cancellation or disposal may discard pending steering.
   * @param message - identified steering content and the source that supplied it.
   */
  steer(message: UserMessage): void

  /**
   * Queue model-facing context for the next pre-step without waking the
   * driver. A running driver claims it at the nearest later step boundary;
   * idle drivers leave it pending until follow-up or steering
   * wakes them. It may miss a request whose pre-step already claimed its
   * batch. Cancellation or disposal may discard pending context.
   * @param message - identified injected context and the source that supplied it.
   */
  inject(message: UserMessage): void
}
```

```ts type-equiv
/**
 * An agent's lifecycle state, emitted on every transition as `agent/status`:
 * `idle` means no driver is active; `running` begins when waking input starts
 * cancellable pre-step processing and lasts while the driver drains,
 * closes, or checkpoints turns. Disposal removes the agent from its registry;
 * it is not a third observable status.
 */
type AgentStatus = 'idle' | 'running'
```

```ts type-equiv
/** One process-local live assistant streaming publication. */
type AssistantStreamFrame =
  | {
    readonly type: 'start'
    readonly attemptId: LlmAttemptId
    /** Monotone within one attached Agent lifecycle; replacement restarts at 1. */
    readonly revision: number
    readonly turn: number
    readonly step: number
  }
  | {
    readonly type: 'chunk'
    readonly attemptId: LlmAttemptId
    readonly revision: number
    /** Dense zero-based position within the attempt. */
    readonly index: number
    /** Safe-integer timestamp reused by the durable embedded stream. */
    readonly time: number
    readonly chunk: StreamChunk
  }
  | {
    readonly type: 'end'
    readonly attemptId: LlmAttemptId
    readonly revision: number
    /** Number of chunk frames emitted by this attempt. */
    readonly index: number
    /** Durable settlement committed before this notification, or live abandonment without one. */
    readonly outcome:
      | {
        readonly kind: 'committed'
        readonly eventType: 'assistant/message' | 'assistant/attempt'
        readonly seq: SessionSeq
      }
      | { readonly kind: 'abandoned' }
  }
```

`running` はドライバー全体のドレイン期間を表し、連続するキュー済みターンにまたがってよい。ターンがまだ開いていることを証明するものではない。破棄は agent をレジストリから削除し `agent/disposed` を発行する。終端のステータス値ではない。`followup()` はハンドルを返さない。その `MessageId` は永続インボックスの挿入、取り出し、破棄の事実を識別するものであり、後の assistant 出力やターン終了を識別するものではない。`whenIdle()` は agent 全体を観察するため、呼び出し元はその区間を明示的に所有する場合に限り、受領から idle までの区間を 1 回の実行と呼んでよい（[決定](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-07-30-followup-enqueue-and-owned-runs.md)）。

```ts type-equiv
/**
 * Merge-extensible agent options. Route fields may be live accessors until the
 * Session records a selection or request; step preparation captures their value.
 * Persona belongs to system-prompt sections.
 */
interface AgentOptions {
  /** Provider route (must have a registered adapter at call time). */
  provider?: string
  /** Model id interpreted by the selected provider adapter. */
  model?: string
  /** Adapter-owned reasoning effort for the selected provider/model route. */
  reasoningEffort?: ReasoningEffortId | undefined
  /** Maximum output tokens for each conversation-model request. */
  maxTokens?: number
}
```

ステップの準備は、プロンプト組み立ての前に `provider` と `model` を必要とする。非推奨の `agent/request` hook はリクエストコントロールを変更してよいが、そのルートを保持しなければならない。明示的な `reasoningEffort` はそのルートの最初のリクエストをシードし、正確なモデル解決がそれを検証する。省略した場合はアダプターのデフォルトが実体化できる。存在する場合、`maxTokens` は正の安全な整数でなければならず、すべての会話モデルリクエストを上限付けする。省略した場合は、リクエストヘッダーの前に正確なモデルのアダプターデフォルトが実体化できるか、そうでなければプロバイダーの振る舞いは変わらない。agent スコープの `deployment:persona-prefix` プロンプトセクションは、グローバルなデフォルトペルソナをシャドウしてよい。

`AgentLoop` は `ctx.sessionProjections` に `modelSelection` を登録し、各 `agent.model` インスタンスを通じて選択操作を公開する。`model/selection` はモデル履歴に入ることなく完全な選択を記録する。一致するリクエストヘッダーが `pending` をクリアし、最新のリクエストが `lastUsed` のまま残る。アダプターがデフォルト化したエフォートは省略され、後のリクエストが再びデフォルトを解決する。`next` が null になるのは、選択もリクエストも記録される前だけであり、その場合 `AgentOptions` がルートを供給する。[Agent のモデル API](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent/README.md)は選択の優先順位と再開ポリシーを説明する。

```ts type-equiv
/** Agent-owned access to the model choice for future steps. */
interface Model {
  /**
   * Resolve the choice for the next step to begin preparation. Recorded choices
   * take precedence over the last request and then the Agent's live options.
   * This read neither starts a step nor changes an already-prepared step.
   * @returns a detached choice, or undefined when no source names both provider and model.
   */
  current(): Readonly<ModelSelection> | undefined

  /**
   * Replace the next step's complete selection. Omitted effort follows adapter
   * defaults without inheriting a previous effort or resolving defaults here.
   * An equal recorded choice is a no-op; an options-only fallback is
   * always recorded. Callers own model availability and capability validation.
   * The append uses the Session's existing persistence and failure guarantees.
   * @param selection - complete provider, model, and optional explicit effort.
   */
  select(selection: Readonly<ModelSelection>): void

}
```

```ts type-equiv
/** Complete provider, model, and optional reasoning effort chosen for one Session's model steps. */
interface ModelSelection {
  /** Registered provider route. */
  provider: string
  /** Provider-owned model id. */
  model: string
  /** Adapter-owned reasoning effort, or provider/default behavior when absent. */
  reasoningEffort?: ReasoningEffortId
}
```

```ts type-equiv
/**
 * Durable model-selection state for one Session. An effort the adapter
 * defaulted on a recorded request is omitted, so later requests resolve the
 * default again instead of pinning it as a choice.
 */
interface ModelSelectionProjectionState {
  /** Selection used by the latest recorded model request. */
  readonly lastUsed: ModelSelection | null
  /** Later recorded selection not yet used by a matching model request. */
  readonly pending: ModelSelection | null
}
```

```ts type-equiv
/** Client view of the durable model-selection fold. */
interface ModelSelectionProjection {
  /** Selection used by the latest recorded model request. */
  readonly lastUsed: ModelSelection | null
  /** Selection the next model step uses; null while the Agent's initial options supply the route. */
  readonly next: ModelSelection | null
}
```

インボックスは配信の語彙である。agent が永続プロジェクションとして所有する、順序付きの 2 つの保留メッセージリストである:

```ts type-equiv
/** Agent-owned access to pending work; concrete storage belongs to the driver. */
interface Inbox {
  /** Prompts awaiting individual turns. */
  readonly nextTurn: readonly UserMessage[]
  /** Input awaiting the next step boundary. */
  readonly nextStep: readonly UserMessage[]

  /** Durably cancel all pending input, clearing next-step before next-turn. */
  clear(): void

  /**
   * Append one message to a pending list.
   * @param target - pending list to extend.
   * @param message - message to append.
   */
  append(target: InboxTarget, message: UserMessage): void

  /**
   * Prepend one message to a pending list.
   * @param target - pending list to extend.
   * @param message - message to prepend.
   */
  prepend(target: InboxTarget, message: UserMessage): void

  /**
   * Replace one pending message in place.
   * @param messageId - identity of the pending message to replace.
   * @param newMessage - replacement message.
   * @returns whether the message was still pending.
   */
  replace(messageId: MessageId, newMessage: UserMessage): boolean

  /**
   * Remove one pending message.
   * @param messageId - identity of the pending message to remove.
   * @returns whether the message was still pending.
   */
  remove(messageId: MessageId): boolean

  /**
   * Apply standard splice semantics and durably record the normalized result.
   * @param target - pending list to mutate.
   * @param start - splice position.
   * @param deleteCount - maximum number of messages to remove.
   * @param inserted - messages to insert at the resolved position.
   * @returns messages removed by the splice.
   */
  splice(
    target: InboxTarget,
    start: number,
    deleteCount: number,
    inserted: UserMessage[],
  ): UserMessage[]
}
```

```ts type-equiv
/** One of the two ordered pending-message lists owned by an agent. */
type InboxTarget = 'next-turn' | 'next-step'
```

保留中の各出現はその `UserMessage` であり、`MessageId` が唯一の同一性である。構造的な `Inbox` メソッドは、正規化された永続 `agent/inbox/spliced` 変更を記録し、重複する保留 ID を拒否する。`replace(messageId, newMessage)` と `remove(messageId)` は両方のリストにわたって保留メッセージを探す。置換は同一性を変えてよく、古いメッセージを discarded として、続いて新しいメッセージを inserted として発行する。通常の削除と `clear()` はキャンセルである。ステップ境界では、dsh-agent-loop のパッケージ内部 `ReactLoopInbox` が提案されたバッチ（すべての `next-step` 入力と、ターン境界では 1 つの `next-turn` メッセージ）を、discarded 通知のない純粋な削除 splice で取り除き、その後メッセージごとの claimed 通知を発行する。ループ専用の保留検出と取り出しは `Agent.inbox` の一部ではない。`AgentLoop` サービスはファクトリーを公開する前に標準の `inbox` プロジェクションを登録する。そのセルが唯一のライブ状態であり、同じフォールドが Agent が存在しない場合でもコールドコンシューマーにサービスする。フォールドは安全でない、または範囲外の splice 座標と、両方のリストにまたがる重複同一性を拒否し、不正な永続履歴をイベント seq で識別する。1 つのメッセージを追うコンシューマーは、正確な `agent/inbox/inserted`、`claimed`、`discarded` 通知を使う。

キャンセル:

```ts type-equiv
/** Options for {@link Agent.cancel}. */
interface CancelOptions {
  /**
   * Preserve queued and steering inbox items instead of discarding them. The
   * active turn is still aborted, but un-started and pending work survives for a
   * later turn and no canceled inbox splice is logged.
   */
  keepInbox?: boolean | undefined
}
```

```ts type-equiv
/** Why an active agent driver was cancelled. */
type AgentCancelCause =
  | { readonly kind: 'user' }
  | { readonly kind: 'parent' }
  | { readonly kind: 'hook'; readonly reason: string }
  | { readonly kind: 'disposed' }
```

cause は TypeScript が強制する同一プロセス内の入力である。アクティブなキャンセル保持者は同じオブジェクトをランタイム専用の `AbortSignal.reason` として公開する。シグナルは協力的なリスナーに分類の権限を与えない。永続的な `turn/end` は結果を `{ kind: 'aborted', reason: TurnEndCancelCause }` として記録するため、キャンセル原因は終端結果に残る。

[イベント分類](../architecture.md#events)が `agent/*` のライフサイクル、チェックポイント、waterfall 契約を所有する。ターンとステップの境界は agent の emit ではなく、永続セッションイベントである。

## 開始元の Agent

`ctx.agents` が運ぶプロセスローカルのイニシエーターは、上記の正確な `Agent` であり、別のフレームやコピーされた同一性ではない。アンビエントな存在はライブ性の証明でも認可でもない。[イニシエータースコープの決定](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-07-15-agent-initiator-scope.md)がその寿命とスコープの規則を定義する。

## 傍受の決定

ステップ前の決定は、永続的な user ロール入力と同じ識別済み `UserMessage` 型を使う。入力されたバッチが権威であり、すべてのメッセージの `id` と `source` を保持する。hook ブリッジはネイティブの決定フィールドをこの型付き結果にマップする。

ソース: [`packages/core/agent/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent/src/types.ts)

`agent/pre-step` は、排他的に取り出されたバッチ（`messages`）、提案されたステップの座標（`turn`、`step`）、現在のターンのキャンセル `signal` を運ぶ 1 つのペイロードを受け取る。最初の提案は、いずれかのステップの前に開いているターンの中で実行される。ツールの継続は、ステップ間で空のバッチを取り出して提出してよい:

それは `PreStepDecision` を返す。reject はステップを開かない。enter は `step/start` の後に追記される完全なメッセージバッチを供給する。最終決定で省略された取り出し済みメッセージは削除されたままになり、取り出し後に挿入された入力は保留のままになる:

```ts type-equiv
/** Whether and with which messages the loop enters a proposed step. */
type PreStepDecision =
  | { kind: 'reject' }
  | {
    kind: 'enter'
    messages: UserMessage[]
    /** Start a distinct model-message series before this step's admitted messages. */
    startsRequestSeries?: true
  }
```

`agent/request-error` は、失敗したモデルステップが閉じた後、そのターンが閉じる前に実行される。リスナーは、失敗したターンのシグナルがまだライブである間に、永続状態を修復したりポリシー作業を待機したりできる。ハンドリングリスナーは `next()` を呼ばずに `{ kind: 'retry' }` を返す。デフォルトの `undefined` は障害を終端のままにする。

```ts type-equiv
/** Action returned by a listener that owns model-request recovery. */
type RequestErrorAction = { kind: 'retry' } | undefined
```

`agent/pre-step` はリクエスト派生前の唯一の waterfall リスナーチェーンである。`agent/turn-stopping` は、ターンにツールやステアリングの継続がないときに、最後のステアリングドレインの前に実行される。

`agent/created` は `SessionStartSource` を運ぶ（セッションライフサイクルが始まった理由であり、ブリッジは SessionStart マッチャーのキーにする）:

```ts type-equiv
/** Why a session lifecycle began; seeded creates are `startup`, while persisted loads are `resume`. */
type SessionStartSource = 'startup' | 'resume' | 'clear' | 'compact'
```

## セッション

`Session` は型付き `SessionEvent` の**追記専用ログ**であり、唯一の真実の源である。LLM のメッセージ履歴はログから*派生*するもので（`deriveMessages()`）、別に保存されない。すべてのエントリーは単調な `seq`、`time`、`type` で判別される `data` ペイロードを運ぶ。サーフェスバリアントはさらに、`sourceEventSeqs` で引用された以前のイベントを列挙し、`surfaceOp` を運んでよい。

`SessionEvent` エンベロープの正確な条件付きフィールド、13 のコアイベントバリアント（`turn/start`、`turn/end`、`step/start`、`step/end`、`user/message`、`system/message`、`assistant/message`、`assistant/attempt`、`tool/call`、`tool/result`、`request/header`、`request/context`、`session/end-seed`）、`deriveMessages()` のプロジェクション規則、`TurnEndReason` の理由、実行囲い込みとスタンドアロンイベントの規則は **[session.md](session.md)** にある。ログを永続化する方法（`SessionPersistence` インターフェース、JSONL プロバイダー、`session/flush` チェックポイント、クラッシュ復旧、`SessionHeader`）は **[persistence.md](persistence.md)** にある。

## `ToolDefinition`

コアにある唯一のパイプライン作成用の型であり、登録されたすべてのツールが*何であるか*を表す。モデル向けの `ToolSchema` に、`execute` 関数と任意の最終コンテンツおよび UI コールバックを加えたものである。ツール作者が手で構築することはまれだが（`defineTool` DSL が型付き引数で構築する）、これはレジストリが保持し、ループがディスパッチする契約である。

その完全なフィールド、`defineTool`/`ValueSchemaSpec`/`ParameterSchemaSpec` の型付きスキーマ DSL、`ToolExecution`/`ToolExecutionResult` の waterfall 型、ツール提示の UI 型は **[tools.md](tools.md)** にある。

## リポジトリ全体の型パターン

すべてのサブシステムに共通して現れる 2 つのパターンを、ここで一度だけ文書化する。

<a id="the-map--derived-union-pattern"></a>

### `…Map → 派生ユニオン` パターン

harness のほとんどすべての拡張可能な合併型は 1 つのパターンに従う。判別タグでキー付けされたインターフェース（`…Map`）があり、そこから `keyof` でユニオンが派生する。プラグインは**宣言マージ**でバリアントを追加し、所有パッケージを編集しない。

```ts ignore-check
// The pattern, schematically:
interface ThingMap {
  'a': { kind: 'a'; /* … */ }
  'b': { kind: 'b'; /* … */ }
}
type ThingKind = keyof ThingMap          // 'a' | 'b'
type Thing = ThingMap[keyof ThingMap]    // the discriminated union

// A plugin extends it without touching the source package:
declare module '@deepseek-ai/dsh-llm' {
  interface ThingMap {
    'c': { kind: 'c'; /* … */ }
  }
}
```

6 つの正規マップがこのパターンを使う。プラグイン作者はこれらを拡張する:

| マップ | パッケージ | 派生 | カタログ |
|---|---|---|---|
| `ContentBlockMap` | dsh-llm | `ContentBlock` | [llm-streaming.md](llm-streaming.md#content-blocks-and-messages) |
| `MessageSourceMap` | dsh-llm | `MessageSource` | [llm-streaming.md](llm-streaming.md#content-blocks-and-messages) |
| `LlmRequestPurposeMap` | dsh-llm | `LlmRequestPurpose` | [llm-streaming.md](llm-streaming.md#the-model-request-and-result) |
| `FinishReasonMap` | dsh-llm | `FinishReason` | [llm-streaming.md](llm-streaming.md#the-model-request-and-result) |
| `TurnEndReasonMap` | dsh-session | `TurnEndReason` | [session.md](session.md) |
| `SessionEventMap` | dsh-session | `SessionEvent` | [session.md](session.md) |

コンシューマーが最も `switch` する 2 つの大きな判別可能合併型は、**`StreamChunk`**（ストリーミングプロトコル）と **`SessionEvent`**（ログエントリー）である。リポジトリの慣習に従い、タグで `switch` し、`if` を連鎖させない。そうすることで各アームが絞り込まれ、誤ったタグはコンパイルに失敗する。

### ブランド付き ID

パッケージ間で渡される ID は**ブランド付き**である。構造的には文字列だが、型レベルでは交換できない（`ToolCallId` が期待される場所に `SessionId` を渡すことはできない）。構築には共有の `brandString<T>()` ヘルパーか、所有者定義の検証ファクトリーを使う。比較、ログ、JSON は通常の文字列として振る舞う。

`Branded<B>` プリミティブとステートレスコンストラクターは [dsh-brand](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/util/brand) にあり、harness のケイパビリティ依存を持たない。`brandString<T>()` はコンパイル時のみの文字列ブランドを適用する。

ソース: [`packages/util/brand/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/util/brand/src/index.ts)

```ts type-equiv
/** A string carrying a compile-time-only brand `B`. */
type Branded<B extends string> = string & { readonly [BRAND]: B }
```

2 つのコア ID は `ToolCallId`（ツール呼び出しをその結果と相関させる。dsh-llm）と `SessionId`（ライブ agent と永続セッションの共有同一性。dsh-session）である。ケイパビリティパッケージも独自の ID にブランドを付ける。たとえば [jobs.md](jobs.md) の `JobId` である。

## `TurnBoundaryProjection`

`ctx.agents.turnBoundaryOf(session)` は、ライブ Agent を必要とせずに、記録されたターンとステップの位置を読む。レジストリがこの状態を所有し、コンシューマーはそのプロジェクション定義を登録しない。

ソース: [`packages/core/agent/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent/src/types.ts)

```ts type-equiv
/** Turn and step positions recorded in a Session. */
interface TurnBoundaryProjection {
  /** Seq of the open turn's `turn/start`, or null between turns. */
  readonly openTurnStartSeq: OptionalSessionSeq
  /** Seq of the latest `step/start` event, or null before the first step. */
  readonly lastStepStartSeq: OptionalSessionSeq
  /** The latest step boundary (`step/start` or `step/end`) and its seq, or null before the first step boundary. */
  readonly lastStepBoundary: { readonly kind: 'start' | 'end'; readonly seq: SessionSeq } | null
  /** Turn number of the latest `turn/start`; 0 before the first turn. */
  readonly lastTurn: number
}
```

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxagentdefaultmodel--agentdefaultmodelconfig"></a>

### `ctx.agentDefaultModel` — `AgentDefaultModelConfig`

Host やトランスポートから独立してデフォルトのモデル選択を所有する。各操作は所有する Config 参照を読む。

```ts cordis-catalog
/**
 * Read the current default model selection.
 * @returns a detached provider, model, and optional reasoning selection.
 */
currentSelection(): ModelSelection

/**
 * Save the complete default model selection. A deployment without a configuration
 * editor keeps its composition entry. Saves commit in submission order; a failed
 * save rejects its caller without blocking later saves.
 * @param next - resolved selection accepted by an entry point.
 * @returns fulfillment after the optional profile write settles.
 */
async saveSelection(next: ModelSelection): Promise<void>
```

ソース: [`packages/core/agent-default-model/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent-default-model/src/index.ts)

<a id="ctxagentloop--agentloop"></a>

### `ctx.agentLoop` — `AgentLoop`

具体的な agent ファクトリとドライバーサービス。

```ts cordis-catalog
/**
 * Create an agent and session under one caller-supplied identity, owned by
 * the accessing fiber. Constructor-driven config calls mint a fresh combined
 * id before entering this boundary. When a persistence backend is mounted,
 * the session's durable identity and any seed are stored before publication.
 * @param id - shared agent/session identity.
 * @param options - concrete loop options.
 * @param meta - optional fresh-session workspace metadata.
 * @returns the published running agent.
 */
async create(id: SessionId, options: AgentOptions = {}, meta: Pick<SessionHeader, 'cwd'> = {}): Promise<Agent>

/**
 * Create an owned agent on a caller-supplied session id.
 * @param ownerCtx - caller context that structurally owns the lifecycle.
 * @param options - identities, optional live parent, session seed/metadata, loop options, setup, and cancellation.
 * @returns the published handle.
 */
async createAgent(ownerCtx: Context, options: CreateAgentOptions): Promise<AgentHandle>

/**
 * Resume an owned agent from the configured persistence service.
 * @param ownerCtx - caller context that owns load, setup, and the live lifecycle.
 * @param options - persisted identity, optional live parent, loop options, setup, and cancellation.
 * @returns the published handle.
 */
async resume(ownerCtx: Context, options: ResumeAgentOptions): Promise<AgentHandle>
```

型: [SessionHeader](persistence.md)

ソース: [`packages/core/agent-loop/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent-loop/src/index.ts)

<a id="ctxagentpresets--agentpresetregistry"></a>

### `ctx.agentPresets` — `AgentPresetRegistry`

YAML で宣言されたプリセットと、ライブ Agent が保持するリビジョンのレジストリ。

```ts cordis-catalog
/**
 * Read the preset recorded for a Session, including selections no longer in the roster.
 * @param session - Session whose committed preset selection is read.
 * @returns the recorded preset, or null when none was selected.
 */
presetForSession(session: Session): string | null

/** Register and eagerly load a definition; activation failure remains visible in the roster.
 * @param definition Parsed configuration supplied by the declaring plugin.
 * @returns Definition disposer after activation or its diagnostic settles; the declaring plugin owns it.
 */
async register(definition: PresetDefinition): Promise<() => Promise<void>>

/** Inspect retained revisions, or the exact revision an Agent joined.
 * @param ctx - optional Agent context; omission includes all retained revisions.
 * @returns detached module references and isolation diagnostics; no match returns an empty list.
 */
inspectCompositions(ctx?: Context): AgentPresetInspection[]

/** Read every declared preset, including activation failures.
 * @returns Display metadata and loading diagnostics.
 */
async list(): Promise<AgentPreset[]>

/** Read the selection roster.
 * @returns Current presets, each marked when it is the default.
 */
@Remote('list') async remoteExportList(): Promise<AgentPresetRoster>

/** Resolve an identity without starting an Agent.
 * @param id Explicit preset or the current default.
 * @returns Current metadata, including failure when activation failed.
 */
async resolve(id?: string): Promise<AgentPreset>

/** Read one declaration's child plugin list as YAML, for viewing only.
 * @param agentPreset Preset identity.
 * @returns The declared composition beside its published metadata.
 */
@Remote('read') readDocument(agentPreset: string): Promise<AgentPresetDocument>

/** Bind an unpublished Agent to the current preset revision.
 * @param ctx Agent context from its setup callback.
 * @param id Requested preset, or the default.
 * @returns Bound preset identity.
 */
async mount(ctx: Context, id?: string): Promise<AgentPreset>

/** Join a child to the exact revision retained by its parent.
 * @param ctx Child Agent context.
 * @param parent Parent Agent context.
 * @returns Inherited preset id, or undefined in a preset-free composition.
 */
composeFrom(ctx: Context, parent: Context): string | undefined

/** Read the preset a live Agent uses.
 * @param ctx Agent context.
 * @returns Its preset id, if bound.
 */
composedPreset(ctx: Context): string | undefined

/** Read a service supplied inside an Agent's isolated preset group.
 * @param agent Agent whose composition is queried.
 * @param name Cordis service name.
 * @returns The service, or undefined.
 */
serviceFor<K extends string & keyof Context>(agent: { ctx: Context }, name: K): Context[K] | undefined

/** Rebind a blank Agent; the caller owns the blank-session check.
 * @param ctx Agent context.
 * @param id Requested preset.
 * @returns The bound identity.
 */
async recompose(ctx: Context, id: string): Promise<AgentPreset>

/** Select a preset before a session starts its first turn. A Session whose
 * log records the preset and whose Agent runs its current revision keeps its
 * composition; nothing is logged or announced.
 * @param agent Target Agent.
 * @param agentPreset Requested identity.
 * @returns Committed preset identity.
 */
@Remote('select') async select(agent: Agent, agentPreset: string): Promise<string>

/** Read current registrations for cold transcript presentation.
 * @param id Preset identity or the default.
 * @returns A revision lease; dispose it after the scoped read completes.
 */
async acquireScope(id?: string): Promise<{ key: ScopeKey } & AsyncDisposable>

/** Read plugin rows without creating an Agent.
 * @returns Current declaration metadata and activation states.
 */
compositionInventory(): Promise<AgentPresetComposition[]>
```

型: [ScopeKey](scope.md) · [Session](session.md)

ソース: [`packages/preset/agent-preset-registry/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/preset/agent-preset-registry/src/index.ts)

<a id="ctxagents--agentregistry"></a>

### `ctx.agents` — `AgentRegistry`

Agent サービス（`ctx.agents`）: ライブ agent を追跡し、プロセスローカルの非同期ドライバーチェーンを通じて開始元の Agent を運ぶ。Agent の*生成*は、`setFactory` を通じて登録された、AgentFactory を実装するプラグイン（`@deepseek-ai/dsh-agent-loop`）が提供する。

イニシエーターメソッドは同一プロセス内の因果的帰属のみを提供する。アンビエントな存在はライブ性の証明でも認可でもない。サブジェクトと所有者は明示的なままであり、worker、プロセス、永続化、ワイヤ境界での同一性も同様である。返された Promise 境界はティアダウン中にドレインされる。ただし、所有ファイバーのアンロードを開始するネストされた系譜は、自身のドレインから除外される。

```ts cordis-catalog
/**
 * Read the turn and step positions recorded in a Session.
 * @param session - Session whose committed events are read.
 * @returns the current turn and step positions; do not mutate the returned value.
 */
turnBoundaryOf(session: Session): TurnBoundaryProjection

/**
 * Read the Agent that initiated the inherited asynchronous driver chain.
 * Use this optional form for logging, tracing, metrics, or host attribution
 * that also supports agentless calls. When a parent creates a child, setup
 * reports the causal parent while the setup callback's Agent parameter
 * identifies the child.
 * @returns the inherited Agent, or `undefined` outside an initiator boundary
 *   and inside an explicit clearing boundary.
 * @throws when this service instance has been disposed.
 */
currentInitiator(): Agent | undefined

/**
 * Read the initiating Agent and fail when no initiator boundary is active.
 * Use this for private helpers contractually below a driver, or for a
 * deployment-owned outbound request whose contract forbids agentless calls.
 * Generic or direct-call paths use optional lookup or explicit request fields.
 * @returns the inherited Agent.
 * @throws when no initiator is active or this service instance has been disposed.
 */
requireInitiator(): Agent

/**
 * Run an operation with one exact Agent as its process-local initiator. The
 * exact synchronous value or Promise returned by the operation is preserved.
 * Custom drivers and test harnesses wrap their complete returned foreground
 * lifetime.
 * A queue or wire receiver may establish this boundary only after validating
 * explicit identity and resolving the exact live Agent; this method does neither.
 * Detached work remains owned by the subsystem that starts it.
 * @param agent - initiating Agent to inherit; presence is neither liveness proof nor authorization.
 * @param operation - synchronous or asynchronous operation to invoke.
 * @returns the exact value returned by `operation`.
 * @throws when the initiator scope is closing/disposed, or when `operation` throws.
 */
withInitiator<T>(agent: Agent, operation: () => T): T

/**
 * Run an operation inside a boundary that hides any inherited initiating
 * Agent. The exact synchronous value or Promise is preserved.
 * Use this while creating lazy shared timers, queue pumps, pool maintenance,
 * watchers, or exporters so they do not inherit the first Agent that happens
 * to initialize them. It clears only initiator attribution, not explicit
 * fields, and does not own or drain detached resources.
 * @param operation - synchronous or asynchronous operation to invoke without an initiator.
 * @returns the exact value returned by `operation`.
 * @throws when the initiator scope is closing/disposed, or when `operation` throws.
 */
withoutInitiator<T>(operation: () => T): T

/**
 * Register the agent-creation factory (the loop calls this on construction,
 * effect-scoped). A traced Cordis service is canonicalized to its concrete
 * target; each create/resume call is then traced through that caller's
 * context so ownership follows the caller without stacking proxy layers.
 * Throws if a factory is already registered. Returns the disposer; on
 * dispose the factory slot is cleared.
 * @param factory - the loop-owned factory {@link create}/{@link resume} delegate to.
 * @returns the disposer that clears the factory slot. The exact
 *   Cordis effect disposer (single-shot): composite (generator) effects may
 *   yield it directly — exact identity nests the teardown in order.
 */
setFactory(factory: AgentFactory): () => void

/**
 * Create and publish a new agent through the registered factory.
 * Distinct from {@link register} (which records an already-constructed
 * agent): this constructs the agent and its session. Rejects if no factory is
 * registered or creation/setup fails. The resolved {@link AgentHandle} lets
 * the owner tear down exactly this agent.
 * @param options - shared identity, optional live parent, session seed/metadata, and agent options.
 * @returns the handle after setup, rollback-covered publication, and loop start complete.
 */
async create(options: CreateAgentOptions): Promise<AgentHandle>

/**
 * Load a persisted session and resume an agent on it through the registered
 * factory. Rejects if no factory is registered; the factory rejects if
 * session persistence is not configured or persistence/setup fails.
 * @param options - persisted identity, optional live parent, configuration, and setup.
 * @returns the handle after setup, rollback-covered publication, and loop start complete.
 */
async resume(options: ResumeAgentOptions): Promise<AgentHandle>

/**
 * Register a live agent with source `startup`. Rejects if the id is already registered or a
 * serial `agent/created` listener fails. Emits `agent/disposed`
 * when the calling fiber is disposed — both with the agent's scope carrier
 * (`scopeTarget(agent, agent)`): the subject is the agent in hand, so the
 * emits are scope-filtered regardless of which context invoked `register`
 * (calling through `agent.ctx` scopes EFFECTS; dispatch scoping always
 * requires passing the carrier). The entry is a runtime root; factory-backed
 * creation uses `options.parentAgent` for child ownership. Await the registration before using the agent.
 * @param agent - the already-constructed agent to record in the store.
 * @returns the awaitable Cordis effect disposer (single-shot; a repeat call
 *   returns undefined without awaiting an in-flight teardown). Exact
 *   identity is load-bearing: a composite (generator) effect that owns a
 *   teardown ORDER — the agent factory's lifecycle chain — must yield THIS
 *   function so Cordis nests the unregistration at that yield position;
 *   yielding a wrapper would leave it disposing as a concurrent sibling on
 *   owner unload, unregistering the agent (and emitting `agent/disposed`)
 *   while its final turn is still draining.
 */
register(agent: Agent): ReturnType<Context['effect']>

/**
 * Insert an already-constructed agent without announcing it. This is the
 * advanced ordered-lifecycle primitive used by the async agent factory: it
 * first completes setup while the agent is unpublished, then assigns the
 * returned detach closure into its pre-installed composite teardown before
 * calling {@link announce}. Ordinary callers use {@link register}.
 * @param agent - the prepared, unpublished agent.
 * @param owner - explicitly supplied live runtime owner, or
 *   undefined for a top-level runtime root. This is runtime ownership, not
 *   the resumed session's durable parent lineage.
 * @returns an idempotent closure that removes this exact entry and emits
 *   `agent/disposed` with listener failures contained. When called from a
 *   `agent/created` listener, removal and disposal wait until the serial
 *   creation dispatch settles.
 */
enter(agent: Agent, owner: Agent | undefined): () => void

/**
 * Announce an agent previously inserted with {@link enter}.
 * @param agent - the live inserted agent to announce.
 * @param source - fresh creation, resume, clear, or compaction source.
 * @param signal - optional factory initialization cancellation signal passed to listeners.
 * @returns completion of the serial creation listeners; a listener failure rejects.
 * @throws if `agent` is not the exact live registry entry for its id, or its
 *   creation announcement already began (including a reentrant call from a
 *   creation listener).
 */
async announce(agent: Agent, source: SessionStartSource, signal?: AbortSignal): Promise<void>

/**
 * Look up a live agent.
 * @param id - the shared agent/session id to look up.
 * @returns the agent, or undefined when no live agent has that id.
 */
get(id: SessionId): Agent | undefined

/**
 * Test whether a live agent was created through one exact parent agent's
 * scoped context. Runtime ownership is independent of durable session
 * lineage and remains unambiguous when unrelated providers reuse an id.
 * @param id - the candidate child agent's shared agent/session id.
 * @param owner - the expected runtime creator agent.
 * @returns true only while the exact child entry is live under that owner.
 */
isOwnedBy(id: SessionId, owner: Agent): boolean

/**
 * All live agents, in registration order.
 * @returns a fresh array; mutating it does not affect the registry.
 */
list(): Agent[]

/**
 * All live top-level agents in registration order. A top-level agent was
 * created without an owning agent context; durable session lineage does not
 * affect this runtime relation, so a resumed fork may still be a root.
 * @returns a fresh array; mutating it does not affect the registry.
 */
roots(): Agent[]
```

型: [Session](session.md)

ソース: [`packages/core/agent/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent/src/index.ts)

<a id="agent-events"></a>

### `agent/*` イベント

<a id="agentassistant-stream--emit"></a>

#### `agent/assistant-stream` — emit

プロセスローカルの assistant ストリーム公開。チャンクフレームは一時的である。ループは、コミットされた end フレームの前に、同じストリームを持つ最終 v2 の `assistant/message` または `assistant/attempt` を 1 つ追記する。

```ts cordis-catalog
/**
 * Process-local assistant-stream publication. Chunk frames are transient;
 * the loop appends one final v2 `assistant/message` or `assistant/attempt`
 * with the same stream before a committed end frame.
 * @param payload.agent - the agent whose attempt produced the frame.
 * @param payload.frame - one ordered start, chunk, or end publication.
 * Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): agent-scoped listeners receive only that agent.
 * @mode emit
 */
'agent/assistant-stream'(this: Scoped<Agent>, payload: { agent: Agent; frame: AssistantStreamFrame }): void
```

型: [Scoped](scope.md)

ソース: [`packages/core/agent/src/runtime-types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent/src/runtime-types.ts)

<a id="agentcreated--serial"></a>

#### `agent/created` — serial

エントリーされた agent は、ファクトリの setup 後に agent ごとの初期化の準備が整う。リスナーは順番に実行され、生成が解決する前に待機される。AgentLoop はすべてのリスナーが終了するまでキュー済み入力を保持する。スローまたは拒否は生成を失敗させ、以降のリスナーをスキップする。破棄はディスパッチが決着するまでスコープとセッションを保持する。リスナーは `agent.whenIdle()` や自身の所有者の破棄を待機してはならない。

```ts cordis-catalog
/**
 * An entered agent is ready for per-agent initialization after factory setup.
 * Listeners run in order and are awaited before creation resolves. AgentLoop
 * holds queued input until all listeners finish. A throw or rejection fails
 * creation and skips later listeners. Disposal retains the scope and session
 * until dispatch settles; listeners must not await agent.whenIdle() or their
 * own owner's disposal.
 * @param payload.agent - the newly registered agent with its live session and completed setup.
 * @param payload.source - fresh creation, resume, clear, or compaction source.
 * @param payload.signal - factory initialization cancellation signal, when provided.
 * Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): agent-scoped listeners receive only that agent.
 * @mode serial
 */
'agent/created'(this: Scoped<Agent>, payload: { agent: Agent; source: SessionStartSource; signal?: AbortSignal }): undefined | Promise<undefined>
```

型: [Scoped](scope.md)

ソース: [`packages/core/agent/src/runtime-types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent/src/runtime-types.ts)

<a id="agentdisposed--emit"></a>

#### `agent/disposed` — emit

agent がレジストリを離れた。AgentLoop はドライバーの静止とスコープ付き登録の巻き戻しの後に、ただしセッションのデタッチの前にこれを発行する。カスタムレジストリの利用者は、自身のドライバー順序契約を所有する。

```ts cordis-catalog
/**
 * An agent left the registry; AgentLoop emits this after driver quiescence
 * and scoped-registration unwind, but before session detachment. Custom
 * registry users own their driver-ordering contract.
 * @param payload.agent - the exact agent removed from the registry.
 * Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): agent-scoped listeners receive only that agent.
 * @mode emit
 */
'agent/disposed'(this: Scoped<Agent>, payload: { agent: Agent }): void
```

型: [Scoped](scope.md)

ソース: [`packages/core/agent/src/runtime-types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent/src/runtime-types.ts)

<a id="agenterror--emit"></a>

#### `agent/error` — emit

ステップまたはターンがエラーになった。エラーが永続レコードのためのターン内位置を持たない場合でも、マシンはここで障害を報告する。

```ts cordis-catalog
/**
 * A step or turn errored. The machine reports a failure here even when
 * the error has no in-turn position for a durable record.
 * @param payload.agent - the agent whose turn errored.
 * @param payload.turn - the turn in which the failure surfaced.
 * @param payload.step - the step at which the failure surfaced.
 * @param payload.error - the failure, verbatim.
 * Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): agent-scoped listeners receive only that agent.
 * @mode emit
 */
'agent/error'(this: Scoped<Agent>, payload: { agent: Agent; turn: number; step: number; error: unknown }): void
```

型: [Scoped](scope.md)

ソース: [`packages/core/agent/src/runtime-types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent/src/runtime-types.ts)

<a id="agentinboxclaimed--emit"></a>

#### `agent/inbox/claimed` — emit

1 つのメッセージが、開いているターンの中でインボックスを離れた。提案されたステップが拒否された場合、取り出されたメッセージはここで終わる。それは破棄もされず、user/message として再発行もされず、ターンはステップなしで閉じる。

```ts cordis-catalog
/**
 * One message left the inbox inside its open turn. If the proposed step
 * is rejected, the claimed message ends here: it is neither discarded nor
 * re-emitted as a user/message, and the turn closes without a step.
 * @param payload.agent - the agent whose inbox changed.
 * @param payload.message - the claimed message.
 * @param payload.turn - the owning turn.
 * Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): agent-scoped listeners receive only that agent.
 * @mode emit
 */
'agent/inbox/claimed'(this: Scoped<Agent>, payload: { agent: Agent; message: UserMessage; turn: number }): void
```

型: [Scoped](scope.md) · [UserMessage](session.md)

ソース: [`packages/core/agent/src/runtime-types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent/src/runtime-types.ts)

<a id="agentinboxdiscarded--emit"></a>

#### `agent/inbox/discarded` — emit

1 つのメッセージがライブインボックスから破棄された。

```ts cordis-catalog
/**
 * One message was discarded from the live inbox.
 * @param payload.agent - the agent whose inbox changed.
 * @param payload.message - the discarded message.
 * Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): agent-scoped listeners receive only that agent.
 * @mode emit
 */
'agent/inbox/discarded'(this: Scoped<Agent>, payload: { agent: Agent; message: UserMessage }): void
```

型: [Scoped](scope.md) · [UserMessage](session.md)

ソース: [`packages/core/agent/src/runtime-types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent/src/runtime-types.ts)

<a id="agentinboxinserted--emit"></a>

#### `agent/inbox/inserted` — emit

1 つのメッセージがライブインボックスに入った。

```ts cordis-catalog
/**
 * One message entered the live inbox.
 * @param payload.agent - the agent whose inbox changed.
 * @param payload.message - the inserted message.
 * Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): agent-scoped listeners receive only that agent.
 * @mode emit
 */
'agent/inbox/inserted'(this: Scoped<Agent>, payload: { agent: Agent; message: UserMessage }): void
```

型: [Scoped](scope.md) · [UserMessage](session.md)

ソース: [`packages/core/agent/src/runtime-types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent/src/runtime-types.ts)

<a id="agentpre-step--waterfall"></a>

#### `agent/pre-step` — waterfall

提案されたステップを拒否するか、そこに入るメッセージを置き換える。`next()` を呼ぶと現在のメッセージが保持される。

```ts cordis-catalog
/**
 * Reject a proposed step or replace the messages that enter it. Calling
 * `next()` preserves the current messages.
 * @param payload.agent - the agent proposing the step.
 * @param payload.messages - messages removed from the inbox for this step.
 * @param payload.turn - the turn that will own the step.
 * @param payload.step - the step proposed by the loop.
 * @param payload.signal - the current turn's cancellation signal.
 * @param payload.modelSelection - captured step selection, retained across retries; omitted effort uses adapter defaults.
 * Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): agent-scoped listeners receive only that agent.
 * @mode waterfall
 */
'agent/pre-step'(this: Scoped<Agent>, payload: { agent: Agent; messages: UserMessage[]; turn: number; step: number; signal: AbortSignal; modelSelection: Readonly<ModelSelection> }, next: () => Promise<PreStepDecision>): Promise<PreStepDecision>
```

型: [Scoped](scope.md) · [UserMessage](session.md)

ソース: [`packages/core/agent/src/runtime-types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent/src/runtime-types.ts)

<a id="agentrequest-error--waterfall"></a>

#### `agent/request-error` — waterfall

ループがリトライするかステップを閉じる前に、失敗した 1 回のモデルリクエスト試行を処理する。リスナーは回復を所有する場合、`next()` を呼ばずに `{ kind: 'retry' }` を返すか、委譲するために `next()` を呼ぶ。デフォルトの `undefined` は障害を終端のままにする。

```ts cordis-catalog
/**
 * Handle one failed model-request attempt before the loop retries or closes
 * its step. A listener returns `{ kind: 'retry' }` without calling `next()`
 * when it owns recovery, or calls `next()` to delegate. The default
 * `undefined` leaves the failure terminal.
 * @param payload.agent - the agent whose request failed.
 * @param payload.turn - the turn containing the failed request.
 * @param payload.step - the step containing the failed request attempt.
 * @param payload.provider - the provider selected for the failed request.
 * @param payload.failure - serializable facts normalized at the final adapter boundary.
 * @param payload.retryPolicy - the policy of the adapter registration that served the failed request.
 * @param payload.signal - the turn abort signal.
 * Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): agent-scoped listeners receive only that agent.
 * @mode waterfall
 */
'agent/request-error'(this: Scoped<Agent>, payload: { agent: Agent; turn: number; step: number; provider: string; failure: LlmFailure; retryPolicy: ResolvedRetryPolicy | undefined; signal: AbortSignal }, next: () => Promise<RequestErrorAction>): Promise<RequestErrorAction>
```

型: [LlmFailure](llm-streaming.md) · [ResolvedRetryPolicy](llm-streaming.md) · [Scoped](scope.md)

ソース: [`packages/core/agent/src/runtime-types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent/src/runtime-types.ts)

<a id="agentstatus--emit"></a>

#### `agent/status` — emit

agent のステータスが変わった（`idle` ⇄ `running`）。ウェイクアップ配信はキャンセルを予約した後、同期的に `running` に入る。`idle` はスケジュール済みまたはアクティブなドライバーが残っていないことを意味する。

```ts cordis-catalog
/**
 * Agent status changed (`idle` ⇄ `running`). A waking delivery enters
 * `running` synchronously after reserving cancellation; `idle` means no
 * driver remains scheduled or active.
 * @param payload.agent - the agent whose status flipped.
 * @param payload.status - the status just entered (the transition's destination).
 * Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): agent-scoped listeners receive only that agent.
 * @mode emit
 */
'agent/status'(this: Scoped<Agent>, payload: { agent: Agent; status: AgentStatus }): void
```

型: [Scoped](scope.md)

ソース: [`packages/core/agent/src/runtime-types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent/src/runtime-types.ts)

<a id="agentturn-stopping--serial"></a>

#### `agent/turn-stopping` — serial

ターンが閉じようとしている。モデルは応答を負っていない（ライブのツール呼び出しも新しいステアリングもない）。境界がコミットされる前に待機される。異議を唱えるリスナーはステアリングし（`agent.steer(...)`）、マシンはインボックスを再読する。新しいステアリングがあれば別のステップを実行し、なければターンを閉じる。データが決定するため、リスナーの順序は結果を変えられない。逆方向の制御（ツールループを早期に停止する）もデータである。`concludesTurn` を運ぶツール結果は、そのステップでターンを終了させる。この結論は、すでに提出された next-step 作業をショートサーキットしない。同じステップの `additionalContexts` や競合するステアリングは依然として実行され、ターンはそのインボックスがドレインしたときにのみ閉じる。

```ts cordis-catalog
/**
 * The turn is about to close: the model owes no response (no live tool
 * calls, no fresh steering). Awaited before the boundary commits — a
 * listener that objects steers (`agent.steer(...)`) and the machine
 * re-reads its inbox: fresh steering runs another step, none closes the
 * turn. Data decides, so listener order cannot change the outcome. The
 * inverse control (stop a tool loop early) is data too: a tool result
 * carrying `concludesTurn` ends the turn at its step. The conclusion
 * never short-circuits already-submitted next-step work: same-step
 * `additionalContexts` or racing steering still runs, and the turn
 * closes only when that inbox drains.
 * @param payload.agent - the agent whose turn is at its stop boundary.
 * @param payload.turn - the turn about to close.
 * @param payload.signal - the current turn's explicit abort signal.
 * Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): agent-scoped listeners receive only that agent.
 * @mode serial
 */
'agent/turn-stopping'(this: Scoped<Agent>, payload: { agent: Agent; turn: number; signal: AbortSignal }): Promise<void> | void
```

型: [Scoped](scope.md)

ソース: [`packages/core/agent/src/runtime-types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent/src/runtime-types.ts)

<a id="agent-loop-events"></a>

### `agent-loop/*` イベント

<a id="agent-loopconfig-start-failed--emit"></a>

#### `agent-loop/config-start-failed` — emit

宣言的な agent エントリーが、ライブ agent を公開する前に失敗した。設定された同一性のために作業をバッファリングするコンシューマーは、この一時的なシグナルを使って、永遠に待つのではなくその作業を拒否する。通常のファクトリティアダウンは、キャンセルされた起動試行からの失敗を抑制する。

```ts cordis-catalog
/**
 * A declarative agent entry failed before it could publish a live agent.
 * Consumers that buffer work for the configured identity use this
 * transient signal to reject that work instead of waiting forever. Normal
 * factory teardown suppresses failures from the cancelled startup attempt.
 * @param payload.sessionId - exact shared agent/session identity that failed startup.
 * @param payload.error - persistence, setup, or publication failure.
 * @mode emit
 */
'agent-loop/config-start-failed'(payload: { sessionId: SessionId; error: unknown }): void
```

ソース: [`packages/core/agent-loop/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent-loop/src/index.ts)

<a id="agent-preset-events"></a>

### `agent-preset/*` イベント

<a id="agent-presetselected--emit"></a>

#### `agent-preset/selected` — emit

1 つのセッションが、異なる agent プリセットを永続ログにコミットした。コンシューマーはそのセッションのコンポジションから派生した状態だけを無効化する。

```ts cordis-catalog
/**
 * One session committed a different agent preset to its durable log.
 * Consumers invalidate only state derived from that session's composition.
 * @mode emit
 * @param sessionId - the session whose composition changed.
 * @param agentPreset - the preset recorded by the committed selection.
 */
'agent-preset/selected'(sessionId: SessionId, agentPreset: string): void
```

ソース: [`packages/preset/agent-preset-registry/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/preset/agent-preset-registry/src/types.ts)
<!-- END GENERATED cordis-surface -->
