# サブエージェント

日本語 | [English](../../../docs/subsystems/subagent.md) | [中文](../../../docs/subsystems/subagent.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/subagent.md](../../../docs/subsystems/subagent.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

統一されたアクティベーション API は、ローカル会話と外部実行を管理する。

## ケイパビリティとアクティベーション要求

すべてのバックエンドは startActivation を使う。ケイパビリティ・フラグは要求オプションを検証し、prepareContinuable はローカル Agent の作成を選択し、start は 1 つの外部実行を選択する。モデル向けツールは親配信を使う。ワークフローとコードモードの Consumer は、別の親メッセージを注入せずに呼び出し元の結果を await できる。利用できない記述子を持つ履歴上の子は、継続ケイパビリティを付与されることなく `mode: 'unknown'` として発見可能なままである。[パッケージ README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/subagent/subagent/README.md) がカタログの永続性を定義する。

任意の `request.cwd` は子の初期ディレクトリを選択し、相対パスは親の現在の有効ディレクトリに対して解決される。省略すると、受け入れ時にそのディレクトリを捕捉する。ローカルの子 Session は親の元のプロジェクトを保持し、コールド・レジュームは記録されたディレクトリ状態を保持する。

```ts type-equiv
/**
 * Start-time options supported by a registered backend. The manager checks
 * every requested option before preparing a local child or starting an external
 * execution. depthLimit controls maxDepth; the remaining flag names match
 * SubagentStartRequest fields.
 */
interface SubagentCapabilities {
  readonly agentOptions: boolean
  /** Supports selecting a preset independently of the parent's composition. */
  readonly agentPreset: boolean
  readonly outputSchema: boolean
  readonly depthLimit: boolean
}
```

```ts type-equiv
/** One managed child execution and its result delivery policy. */
interface SubagentActivationSpec {
  /** Registered backend to use. */
  readonly provider: string
  /** Short task label retained in cataloged child membership. */
  readonly label: string
  /** Optional reserved identity for a local child; external backends allocate their own ids. */
  readonly childId?: SessionId
  /** Task, parent, and backend-supported execution options. */
  readonly request: Omit<SubagentStartRequest, 'label' | 'signal'>
  /** Cancellation before publication; callers own later cancellation through dispose. */
  readonly signal: AbortSignal
  /** Parent delivery notifies the model; caller delivery only returns the result. Local children always enter the parent catalog. */
  readonly delivery: 'parent' | 'caller'
}
```

```ts type-equiv
/** A managed execution; disposal addresses this exact activation, never a later resume. */
interface SubagentActivation {
  /** Stable identity of the child. */
  readonly childId: SessionId
  /** Accepted inbox message, when the backend has a local inbox. */
  readonly messageId?: MessageId
  /**
   * Execution result after teardown settles and notifications are sent; capture failures reject.
   * Teardown failures are reported by dispose() without replacing a captured result.
   */
  readonly result: Promise<SubagentResult>
  /** Stop and release this activation and its owned descendants; rejects on teardown failure. */
  dispose(): Promise<void>
}
```

```ts type-equiv
/**
 * Task and optional capabilities supplied to a backend. startActivation carries
 * these fields in request while owning its label, cancellation, and delivery
 * policy separately.
 */
interface SubagentStartRequest {
  /** Initial child directory; relative paths resolve against the parent's current directory. Omitted inherits that directory at start. */
  readonly cwd?: string
  /**
   * Independent preset id; omission joins the parent's current generation.
   * Requires a provider with `capabilities.agentPreset` that does not inherit
   * parent context.
   */
  readonly agentPreset?: string
  /** Optional short display label persisted with a session-backed child. */
  readonly label?: string
  /** Content delivered as the child's user message. */
  readonly prompt: ContentBlock[]
  /**
   * The spawning agent. Its effective directory supplies the default cwd;
   * in-process children retain its origin, lineage, and delegation depth.
   */
  readonly parent: Agent
  /**
   * Cancellation signal from the spawning context (the tool's `exec.signal`).
   * This is the canonical cancellation channel both before and after startup:
   * a provider rejects `start()` after cleaning partial resources when it
   * fires before the run is published, and cancels the published run's
   * remaining turn work when it fires afterward.
   */
  readonly signal: AbortSignal
  /**
   * Optional host-Agent provider, model, reasoning-effort, and output-token
   * overrides. Requires {@link SubagentCapabilities.agentOptions}; in-process
   * providers merge them over the parent Agent's options when they create the
   * child, while the DSH SDK provider merges them over its instance defaults
   * before initializing the separate child runtime.
   */
  readonly agentOptions?: AgentOptions
  /**
   * Object-rooted JSON Schema within `assertObjectJsonSchema`'s enforced subset. Start rejects
   * unsupported schemas or providers without the capability. Data must be plain host-realm JSON;
   * a successful child returns the matching value as {@link SubagentResult.structured}.
   */
  readonly outputSchema?: ObjectJsonSchema
  /**
   * Optional absolute delegation-depth cap for the child being started: its
   * computed depth must be less than or equal to this non-negative safe
   * integer. Requires {@link SubagentCapabilities.depthLimit}; rejected at
   * start otherwise.
   */
  readonly maxDepth?: number
}
```

```ts type-equiv
/** Provider-facing request with an absolute directory selected before startup. */
interface ResolvedSubagentStartRequest extends SubagentStartRequest {
  /** Absolute child directory captured from the parent or explicit request. */
  readonly cwd: string
}
```

## ローカルの子とアクティベーション

Spawn だけが独立した `agentPreset` をサポートする。省略時と fork は親のアクティブな構成を継承する。[子プリセット規則](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/subagent/subagent/README.md#restoring-child-composition) が明示的な選択、イベント順序、履歴の復元を定義する。

ローカルの子は永続的な Session と最大 1 つのライブ・アクティベーションを持つ。マネージャーは同一性と容量を予約し、バックエンドを通じて子を準備し、入力の受け入れ、コールド・レジューム、子優先の破棄を所有する。アクティベーション結果はリソース解放と完了通知の後に確定する。破棄はキャプチャ済みの結果を置き換えずにクリーンアップ失敗を別途報告する。構造化キャプチャはそのアクティベーションの入力を閉じ、後のコールド・レジュームは出力スキーマを継承しない。

```ts type-equiv
/**
 * What the continuation manager asks a provider for while materializing one
 * continuable child's FIRST activation. The manager has already reserved the
 * durable child identity and owns every later operation, so this request
 * carries only what distinguishes a fresh child from one seeded with parent
 * history.
 */
interface ContinuableCreateRequest {
  /** Absolute initial directory captured before provider preparation. */
  readonly cwd: string
  /** The reserved durable child session id, for provider diagnostics. */
  readonly sessionId: SessionId
  /** The delegating parent agent whose history a seeding provider reads. */
  readonly parent: Agent
  /**
   * Caller cancellation, which owns preparation only until the manager accepts
   * the initial prompt into the child's inbox.
   */
  readonly signal: AbortSignal
}
```

```ts type-equiv
/**
 * A provider's detached contribution to one continuable child's creation. This
 * is DATA, never a capability: it carries no Agent, `AgentHandle`, prompt
 * delivery, result, disposal, or resume operation, because the continuation
 * manager owns the child's whole lifecycle after preparation.
 */
interface ContinuableCreateSpec {
  /**
   * Completed-turn prefix of the parent's log to seed the child session with,
   * or absent for a fresh child. Same durable contract as
   * `CreateAgentOptions.seed`: contiguous from seq 0, lossless JSON, balanced.
   */
  readonly seed?: readonly SessionEvent[]
}
```

## メッセージと中断

sendMessage は正確なライブ送信者を認可し、隣接するローカル agent のみを許可する。実行中の子はステアリング入力を受け取り、存在しない継続可能な子は永続化からレジュームする。外部実行はフォローアップを拒否する。ローカルと外部の両方の実行は、最終テキスト、構造化結果、診断、状態を親の確定通知で自動的に報告する。ローカルの子は send_message を通じてメッセージを送ることもできる。caller 配信は確定通知を抑制する。drainDescendants と drainChildren は、ローカルと外部のアクティベーションについて所有作業の終了を待機する。

```ts type-equiv
/** Durable attribution for one model-authored message between Agents. */
interface AgentMessageSource {
  readonly kind: 'agent-message'
  /** A message another agent addressed to this one (`relay` context form). */
  readonly form: 'relay'
  /** Session id of the Agent whose tool call produced the message. */
  readonly senderSessionId: SessionId
}
```

```ts type-equiv
/**
 * Durable attribution for the runtime's own account of a continuable child
 * settling. Deliberately a different kind from
 * {@link AgentMessageSource}: an Agent message is content the sender chose,
 * while this message is the manager stating what became of the child, and a
 * transcript that merged them would credit the child with words it never wrote.
 */
interface SubagentSettledMessageSource {
  readonly kind: 'subagent-settled'
  /** A runtime account shown without expanding the row (`notice` context form). */
  readonly form: 'notice'
  /** One-line account of how the child ended. */
  readonly summary: string
  /** Session id of the child that settled. */
  readonly senderSessionId: SessionId
}
```

```ts type-equiv
/** Options for one model-authored message between adjacent Agents. */
interface SubagentSendMessageOptions {
  /** Caller cancellation, owning the operation only until inbox acceptance. */
  readonly signal: AbortSignal
}
```

```ts type-equiv
/**
 * Authority under which one interrupt request is admitted. `user` carries the
 * durable direct-parent address a human client presented; `ancestor` carries
 * the exact live Agent object whose recorded lineage must contain the caller.
 */
type SubagentInterruptAuthority =
  | { readonly kind: 'user'; readonly parentSessionId: SessionId }
  | { readonly kind: 'ancestor'; readonly agent: Agent }
```

## 永続的な列挙

`listChildren` は、すべてのローカルの子と親配信の外部の子からなる親所有のカタログを読む。caller 配信の外部実行は、メンバーシップと結果の収集を呼び出し元に委ねる。ローカルのワークフローの子は完了後も発見可能なままである。外部エントリーは `mode: 'external'` を持ち、子 Session を開くこともフォローアップを受け入れることもできない。カタログ・エントリーは作成時のメンバーシップを記録し、実行状態は記録しない。`list_agents` は `running` または `inactive` のアクティビティを持つ直接の継続可能な子を公開する。[コントロール・ツール](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/subagent/tool-subagent-control/README.md#list_agents) を参照。

## 結果とバックエンド・ハンドル

SubagentResult はプログラムの Consumer が利用できる。アクティベーション・レジストリは破棄が完了するまで外部バックエンド・ハンドルを保持する。クリーンアップ失敗はキャプチャ済みの実行結果を置き換えない。アクティベーション結果と親通知はそれを保持し、dispose は拒否し、ライブの subagent/end イベントは error を報告する。バックエンド所有の診断は、下記の安全な詳細の要件を満たさなければならない。

```ts type-equiv
/**
 * The terminal outcome of a subagent run, resolved by {@link SubagentRun.result}.
 */
interface SubagentResult {
  /**
   * The child's final assistant output is the content of its last non-empty
   * assistant message. Empty-content messages, including usage-only messages,
   * are skipped. Without a non-empty message, the output is its accumulated
   * assistant text stream, or `[]` when the child produced neither.
   */
  readonly output: readonly ContentBlock[]
  /**
   * The structured result after a requested `outputSchema` was successfully
   * satisfied. Requesting a schema does not guarantee presence: a provider can
   * end with `stopReason: 'error'` when the child fails or finishes without a
   * valid capture. The structured value is validated against the requested
   * output schema by the provider; `unknown` here because the seam is
   * schema-agnostic.
   */
  readonly structured?: unknown
  /**
   * Provider-authored, non-assistant failure detail for a non-`completed`
   * result. Providers keep this text free of tool inputs, file contents,
   * environment values, credentials, and raw protocol payloads, and limit it
   * to 4096 UTF-8 bytes. Consumers present it separately from {@link output}.
   */
  readonly diagnostic?: string
  /** Why the run ended. A non-`completed` reason means `output` may be partial. */
  readonly stopReason: SubagentStopReason
}
```

```ts type-equiv
/**
 * Why a subagent run ended. Merge-extensible (a backend may add variants);
 * consumers branch on the known cases and fall through `default`. The known
 * cases mirror the harness turn-end vocabulary so the tool layer can map a
 * non-`completed` result to an `isError` tool result.
 */
interface SubagentStopReasonMap {
  /** The child finished its turn normally. */
  completed: 'completed'
  /** Cancelled through the request signal or disposal. */
  aborted: 'aborted'
  /** Model or transport failure. */
  error: 'error'
  /** The child hit its token ceiling before finishing. */
  'max-tokens': 'max-tokens'
  /** The child declined the task. */
  refusal: 'refusal'
}
```

```ts type-equiv
/**
 * Backend execution handle owned directly by the activation registry.
 * A result may become available before resource release; the manager always
 * disposes the handle and awaits cleanup. Backend startup failures clean up
 * partial resources before rejecting, while accepted execution failures settle
 * through result.
 */
interface SubagentRun {
  /** Provider-minted id, unique across all parents, providers, and local Sessions in this runtime. */
  readonly id: SessionId
  /**
   * Resolves with the child's terminal {@link SubagentResult} when the run
   * settles. Does NOT reject on a child-level failure — a model/transport
   * failure resolves with `stopReason: 'error'` so the consumer maps it to an
   * `isError` tool result. Rejects on an infrastructure fault the seam cannot
   * represent as a stop reason.
   */
  readonly result: Promise<SubagentResult>
  /**
   * Cancel remaining work, reach child quiescence, and release resources.
   * Idempotent.
   */
  dispose(): Promise<void>
}
```

## プロバイダー契約: SubagentProvider

`SubagentManager` は起動、メッセージの受け入れ、実行の寿命を所有する。Spawn と Fork はローカル Agent のための分離された作成入力を提供する。Codex、Claude Code、ACP、DSH SDK は 1 つの実行ハンドルを提供し、複数のターンを得ることはない。各アクティベーションは、その AgentHandle または SubagentRun を直接保持する実行レコードである。外部の完了はハンドルの result promise に従う。inbox とアイドルの受け入れはローカル実行だけに属する。両方のパスは受け入れ、容量、親の所有権、キャンセル、破棄を共有する。[パッケージ・リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/subagent/subagent/README.md) が構成とデプロイの要件を説明する。

```ts type-equiv
/**
 * One registered transport for running child agents. Providers are trusted
 * same-process implementations; callers treat descriptors and returned values
 * as borrowed immutable data. The service may call one provider concurrently
 * for distinct children. Providers isolate operation-local mutable state; a
 * shared capacity controller may delay an operation but must not couple its
 * settlement or cleanup to a sibling.
 */
interface SubagentProvider {
  /** Unique registry name (e.g. `spawn`, `fork`, `acp`). */
  readonly name: string
  /** The start-time features this provider supports (see {@link SubagentCapabilities}). */
  readonly capabilities: SubagentCapabilities
  /**
   * Whether the child sees the parent's completed-turn prefix. This is descriptive, not a
   * service-validated start capability: the model-facing tool derives truthful wording from it.
   * It says nothing about tool registration, injected services, or authority inheritance.
   */
  readonly inheritsParentContext: boolean
  /**
   * Optional static provider-owned provider/model route for child Agent
   * options. Consumers merge tool/model overrides over these values before
   * preflight; providers whose route derives from the parent omit it. The value
   * is detached immutable data and requires `agentOptions` support.
   */
  readonly agentRouteDefaults?: Readonly<{ provider: string; model: string }>
  /**
   * Establish one external execution and return its owned handle.
   * The service has already validated that every requested start-time
   * capability is supported. Before fulfillment, the provider owns setup and
   * cleans any unpublished partial resources before rejecting. Ownership transfers on
   * fulfillment; subsequent turn or infrastructure failure settles through
   * the returned run. Distinct starts may overlap; cancellation, failure,
   * result settlement, and disposal remain independent for each run.
   */
  start?(request: ResolvedSubagentStartRequest): Promise<SubagentRun>
  /**
   * OPTIONAL (continuable-creation capability): contribute the detached
   * creation inputs that distinguish this provider's continuable children —
   * only whether the child session is seeded with parent history. Method
   * presence selects local activation execution. Providers without it execute
   * through start and do not accept subsequent messages.
   *
   * This is the provider's ONLY participation in a continuable child. The
   * continuation manager owns identity reservation, composition, Agent
   * creation, prompt delivery, cold resume, ownership, and disposal, so a
   * provider never sees the child's Agent, handle, turns, or teardown.
   * Distinct preparations may overlap; each follows its own signal and returns
   * data belonging only to `request.sessionId`.
   */
  prepareContinuable?(request: ContinuableCreateRequest): Promise<ContinuableCreateSpec>
}
```

## 永続的な列挙: `listChildren()`、`listDescendants()` とそのエントリー

モデル向けの `list_agents` アダプターは、現在のアクティビティを `running` または `inactive` として報告する。これらの値はタスクの完了を表さず、`send_message` が成功することを保証しない。

`SubagentRuntime.listChildren(parentSessionId, signal?)` は、ライブ優先の Session 観測を通じて親の `subagentCatalog` ビューを読み、成功または失敗時にその観測を解放する。子ログを読んだり Session コーパスを列挙したりせずに、親のイベント順で直接の子エントリーを返す。クエリの失敗は伝播し、カタログ投影の欠落は明示的に失敗する。ブラウザの行は共有投影ストアからメンバーシップを導出し、Session の状態からアクティビティを追加する。コントロール・ストリームは完全なカタログ更新をプッシュする。[親カタログの Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-09-01-parent-owned-subagent-catalog.md) が作成、フォーク分離、順序付け、永続化コストを所有する。

`SubagentRuntime.listDescendants(rootSessionId)` は同じカタログ・リーダーを安定した事前順序で再帰的に呼び出し、各親のイベント順を保つ。外部エントリーはローカル Session を持たない葉である。ワンショットと不明モードのエントリーはトラバーサル・ノードとして残り、不明モードは `unsupported` 診断を生成する。読めない子カタログは `corrupt` または `unavailable` を生成し、そのブランチだけを停止する。ルート読み取りの失敗、サービスや投影の欠落、キャンセルは一覧全体を拒否する。到達可能な各カタログは一度観測され、次の読み取りの前に解放される。繰り返された ID と循環はスキップされる。到達可能なカタログにない Session は、通常の Session フォークやそれらのフォーク下のサブエージェントを含め、発見されない。各行はカタログ上の親とルートからの相対深さを持つ:

```ts type-equiv
/** One catalog descendant with its direct parent and edge distance from the requested root. */
type SubagentDescendantListEntry = SubagentListEntry & {
  /** Parent whose catalog contains this child. */
  readonly parentId: SessionId
  /** Edge distance from the requested root; direct children are `1`. */
  readonly depth: number
}
```



<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxsubagentmodelselection--subagentmodelselectionconfig"></a>

### `ctx.subagentModelSelection` — `SubagentModelSelectionConfig`

最初のモデル・リクエストの前に Session の認可を記録する Host 設定所有者。

ソース: [`packages/subagent/tool-subagent/src/model-selection-settings.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/subagent/tool-subagent/src/model-selection-settings.ts)

<a id="ctxsubagents--subagentruntime"></a>

### `ctx.subagents` — `SubagentRuntime`

管理されたアクティベーション、永続的な発見、ローカルの子メッセージングを備えた名前付きプロバイダー・レジストリ。

```ts cordis-catalog
/**
 * Resolve a delegation tool's depth policy against the current user setting.
 * @param configured - Explicit tool limit, or provider-managed for external delegation.
 * @returns The numeric limit, or undefined when the provider owns depth enforcement.
 */
resolveMaxDepth(configured?: number | 'provider-managed'): number | undefined

/**
 * Start a local child under its reserved identity.
 * @param spec - local task, reserved child id, and result recipient.
 * @returns activation with its accepted initial message id.
 */
startActivation(spec: SubagentActivationSpec & { readonly childId: SessionId }): Promise<SubagentActivation & { readonly messageId: MessageId }>

/**
 * Start a local or external child.
 * @param spec - task, backend, and result recipient.
 * @returns activation with a message id only for local children.
 */
startActivation(spec: SubagentActivationSpec): Promise<SubagentActivation>

/**
 * Join progressing descendants without cancelling them. Idle descendants whose
 * inboxes require a later wake stay resident and do not delay host completion.
 * @param parent - the exact parent whose descendant work is observed.
 * @returns whether work was joined; hosts recheck parent idle after true.
 */
async waitForChildren(parent: Agent): Promise<boolean>

/**
 * Steer one model-authored message to the sender's direct parent or direct
 * continuable child. A running target admits it at the nearest step boundary;
 * an idle target starts a turn, and an absent direct child cold-resumes from
 * persistence. The service derives durable sender attribution from the exact
 * live sender. Caller cancellation stops only pre-acceptance work.
 * @param sender - exact live Agent authorizing and originating the message.
 * @param targetId - durable direct-parent or direct-child session id.
 * @param content - model-authored content to deliver.
 * @param options - caller cancellation before inbox acceptance.
 * @returns the accepted message's inbox id.
 * @throws when continuation services are unavailable, adjacency is rejected,
 *   or the message was not admitted.
 */
async sendMessage( sender: Agent, targetId: SessionId, content: ContentBlock[], options: SubagentSendMessageOptions, ): Promise<MessageId>

/**
 * Interrupt one live child's current execution under a human parent
 * address or an exact live ancestor Agent. Fire-and-return: the cancel
 * signal is issued before this returns, but the target may keep running
 * until it observes the signal. Unclaimed pending inbox work, the Activation,
 * and published descendants are preserved; claimed work is not requeued.
 * Once the interrupted Agent is idle, a waking send resumes the parked FIFO
 * queue. External backends stop their single execution. An absent target
 * is an accepted no-op, as is a manager-less composition, which cannot own a
 * live Activation.
 * @param targetSessionId - the durable child session id to interrupt.
 * @param authority - the human parent address or exact live ancestor Agent.
 * @throws {SubagentError} `UNAUTHORIZED` when the authority does not own the
 *   live target.
 */
interrupt(targetSessionId: SessionId, authority: SubagentInterruptAuthority): void

/**
 * Close subagent admission below exact live parent Agents, stop only their
 * visible descendant Activations synchronously, then await admitted scoped
 * materializations and release those forests child-first. The scoped cutoff
 * lasts until each exact parent leaves the registry; unrelated parent trees
 * remain live.
 * @param parents - exact host-owned parent Agents entering teardown.
 * @returns once every retained descendant activation released its execution handle.
 * @throws an aggregate error after all branches settle when any failed.
 */
async drainDescendants(parents: readonly Agent[]): Promise<void>

/**
 * Release selected resident direct children of one exact live
 * parent. Other children of the same parent remain admitted and resident.
 * Absent targets and a manager-less composition are accepted no-ops.
 * @param parent - exact live direct parent authorizing the selected release.
 * @param childIds - durable direct-child ids to release when resident.
 * @returns once every selected activation released its execution handle.
 * @throws {SubagentError} `UNAUTHORIZED` when a resident target belongs to a
 *   different parent or the supplied parent identity is stale.
 */
async drainChildren(parent: Agent, childIds: readonly SessionId[]): Promise<void>

/**
 * Read the parent's durable direct-child catalog without loading or resuming an Agent.
 * The service owns and releases the live-preferred Session observation.
 * @param parentSessionId - parent whose direct children are requested.
 * @param signal - cancellation forwarded to the Session query.
 * @returns catalog children in parent event order.
 * @throws {@link SubagentError} when query or catalog projection is unavailable.
 * @throws SessionQueryError when the parent cannot be read or the query is cancelled.
 */
listChildren(parentSessionId: SessionId, signal?: AbortSignal): Promise<SubagentCatalogEntry[]>

/**
 * Recursively list reachable parent catalogs in stable pre-order, preserving
 * each catalog's event order. Each row carries its catalog parent and depth;
 * external children are leaves; one-shot and unknown-mode children remain
 * traversal nodes. Unknown modes
 * produce unsupported diagnostics. Unreadable child catalogs produce corrupt
 * or unavailable diagnostics and stop only that branch. Root read failures,
 * missing services or projections, and cancellation reject the whole listing.
 * Each catalog is observed once and released before the next read. No Agent
 * is loaded or resumed; Sessions absent from reachable catalogs are omitted.
 * @param rootSessionId - session whose catalog starts descendant discovery.
 * @param signal - cancellation forwarded to and checked around each catalog read.
 * @returns children and branch diagnostics in parent-catalog pre-order.
 * @throws {@link SubagentError} when listing dependencies are unavailable or the caller cancels.
 * @throws SessionQueryError when the root catalog cannot be read.
 */
listDescendants(rootSessionId: SessionId, signal?: AbortSignal): Promise<SubagentDescendantListEntry[]>

/**
 * Deliver one browser-authored message to a continuable child through the
 * exact live direct parent, retaining the caller-minted request identity and
 * validated browser zone on the accepted message. Success identifies the
 * message the child's inbox accepted; later execution is independent of this
 * call. Queue delivery targets a later turn; steer delivery targets the
 * nearest step and retains the Agent loop's best-effort fallback semantics.
 * Image parts are admitted and persisted through the attachment store
 * before delivery, and the child's model must accept image input.
 * Cold resume at capacity rejects with `subagent/delivery-unavailable`.
 * @param request - durable address, delivery, minted identity, content, and optional browser zone.
 * @param signal - carrier cancellation, owning the call until inbox acceptance.
 * @returns the accepted message's inbox identity.
 * @throws {RemoteError} `gateway/bad-request`, `subagent/attachment-invalid`,
 *   `subagent/invalid-time-zone`, `subagent/parent-unavailable`,
 *   `subagent/not-resumable`, `subagent/unauthorized`,
 *   `subagent/delivery-unavailable`, `gateway/cancelled`, or `gateway/internal`.
 */
@Remote('prompt') async prompt(request: SubagentPromptRequest, signal: AbortSignal): Promise<SubagentPromptReceipt>

/**
 * Remote face of {@link interrupt} under one durable parent address. No
 * catalog, history, persistence, or parent Agent lookup runs: the core
 * primitive alone authorizes the address against the live Activation, which
 * is what keeps a live child interruptible while its parent Agent is offline.
 * Absent, idle, and already-completed targets are accepted no-ops there.
 * @param childSessionId - durable child session id to interrupt.
 * @param parentSessionId - durable direct parent whose authority is claimed.
 * @param mode - required continuable-address discriminator.
 * @returns acknowledgement that the cancel signal was admitted, not that the target is quiescent.
 * @throws {RemoteError} `gateway/bad-request` for an empty id,
 *   `subagent/unauthorized` when the address does not own the live target,
 *   otherwise `gateway/internal`.
 */
@Remote('interruptByParent') interruptByParent( childSessionId: SessionId, parentSessionId: SessionId, mode: 'continuable', ): SubagentInterruptReceipt

/**
 * Register a provider under its name. Registration is effect-scoped and HMR
 * safe; removing a provider blocks new starts but does not revoke runs that
 * were already returned to their holders. Providers without either execution
 * method are rejected with UNSUPPORTED_CAPABILITY before registration.
 * @param provider - the trusted provider implementation.
 * @returns the exact Cordis effect disposer.
 */
registerProvider(provider: SubagentProvider): () => void

/**
 * Look up a provider by name.
 * @param name - the provider name.
 * @returns the provider, or undefined when absent.
 */
getProvider(name: string): SubagentProvider | undefined

/**
 * List registered provider names in insertion order.
 * @returns the registered names.
 */
list(): string[]
```

型: [Agent](core.md) · [ContentBlock](llm-streaming.md) · [MessageId](llm-streaming.md) · [SessionId](core.md)

ソース: [`packages/subagent/subagent/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/subagent/subagent/src/index.ts)

<a id="subagent-events"></a>

### `subagent/*` イベント

<a id="subagentend--emit"></a>

#### `subagent/end` — emit

公開された子が確定した。スコープでフィルタリングされたディスパッチは `subagent/start` と同じ委譲元の親キャリアを使うため、ライフサイクルのペアは同じスコープのオーディエンスに届く。

```ts cordis-catalog
/**
 * A published child settled. Scope-filtered dispatch uses the same delegating
 * parent carrier as `subagent/start`, so the lifecycle pair reaches the
 * same scoped audience.
 * @param info - the run identity and terminal outcome.
 * @mode emit
 */
'subagent/end'(this: Scoped<SubagentRuntime>, info: SubagentRunEndInfo): void
```

型: [Scoped](scope.md)

ソース: [`packages/subagent/subagent/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/subagent/subagent/src/index.ts)

<a id="subagentprovider-added--emit"></a>

#### `subagent/provider-added` — emit

プロバイダーがレジストリで解決可能になった。

```ts cordis-catalog
/**
 * A provider became resolvable in the registry.
 * @param provider - the registered provider.
 * @mode emit
 */
'subagent/provider-added'(provider: SubagentProvider): void
```

ソース: [`packages/subagent/subagent/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/subagent/subagent/src/index.ts)

<a id="subagentprovider-removed--emit"></a>

#### `subagent/provider-removed` — emit

プロバイダーがレジストリを離れた。受け入れ済みの実行は引き続きホルダーが所有する。

```ts cordis-catalog
/**
 * A provider left the registry. Accepted runs remain holder-owned.
 * @param name - the provider name that no longer resolves.
 * @mode emit
 */
'subagent/provider-removed'(name: string): void
```

ソース: [`packages/subagent/subagent/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/subagent/subagent/src/index.ts)

<a id="subagentstart--emit"></a>

#### `subagent/start` — emit

プロバイダーが公開された子を確立した。インプロセス・プロバイダーでは、この通知中に `ctx.agents.get(info.id)` が解決する。スコープでフィルタリングされたディスパッチは委譲元の親でキャリアをキー付けするため、親スコープのリスナーは自身の委譲だけを観測する。`subagent/end` と対になる。

```ts cordis-catalog
/**
 * A provider established a published child. For in-process providers,
 * `ctx.agents.get(info.id)` resolves during this notification.
 * Scope-filtered dispatch keys the carrier by the delegating parent, so a
 * parent-scoped listener observes only its own delegations. Paired with
 * `subagent/end`.
 * @param info - the provider and published child identity.
 * @mode emit
 */
'subagent/start'(this: Scoped<SubagentRuntime>, info: SubagentRunInfo): void
```

型: [Scoped](scope.md)

ソース: [`packages/subagent/subagent/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/subagent/subagent/src/index.ts)
<!-- END GENERATED cordis-surface -->
