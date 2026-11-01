# ツール

日本語 | [English](../../../docs/subsystems/tools.md) | [中文](../../../docs/subsystems/tools.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/tools.md](../../../docs/subsystems/tools.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

[dsh-tools](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/core/tools) のツール・パイプライン。[core.md](core.md) は、core パッケージ間で共有されるパイプライン作成用の型として `ToolDefinition` を紹介している。モデル向けの [`ToolSchema`](llm-streaming.md#the-model-request-and-result) ワイヤ型は、モデル・リクエストとともに宣言される。このページでは、すべての `ToolDefinition` フィールド、それを構築する型付きスキーマ DSL、ガード付き実行型、UI プレゼンテーション型を説明する。

ソース: [`packages/core/tools/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/tools/src/index.ts) · [`packages/core/tools/src/schema.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/tools/src/schema.ts) · [`packages/core/tools/src/presentation.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/tools/src/presentation.ts)

## `ToolDefinition` — 登録済みツール

`ToolSchema`（モデル向けのフィールド）に加えて、必須の正規出力宣言、`execute` 関数、ホスト専用のスケジューラ・メタデータ、任意の最終コンテンツ・コールバック、任意の UI プレゼンターを持つ。レジストリはこれらを保持し、ループはそれらを通じて呼び出しをディスパッチする。レジストリの `schemas()` は明示的な許可リストによってモデル向けの `ToolSchema[]` を構築する。`output`/`execute`/`projectContent`/`finalizeContent`/`timeoutMs`/`isConcurrencySafe`/`presentCall`/`presentResult` がモデル・リクエストに漏れてはならない。

```ts type-equiv
/** Tool-owned canonical output contract used after the body returns a JSON value. */
interface ToolOutputDefinition {
  /** Raw supported JSON Schema enforced against every successful canonical value. */
  readonly schema: JsonSchemaNode
  /** Pure projection from validated arguments and value to Native/model content. */
  render(args: unknown, value: JsonValue): ContentBlock[]
  /** Pure replayable presentation projection for native and nested calls. */
  presentationMeta?(args: unknown, value: JsonValue): JsonValue
}
```

```ts type-equiv
/** A registered tool: its schema plus the execution function. */
interface ToolDefinition extends ToolSchema {
  /** Mandatory canonical output declaration. */
  readonly output: ToolOutputDefinition
  /**
   * Run one accepted call and return only its canonical lossless-JSON value.
   * Async work must observe or forward `exec.signal` and settle only after its
   * owned work reaches quiescence. The executor preserves caller cancellation
   * through around-dispatch signal replacement and does not abandon this
   * promise, but it cannot hard-kill same-process code.
   * @param args - losslessly snapshotted, frozen model arguments.
   * @param exec - execution identity, cancellation signal, and context deferral.
   * @returns the canonical value declared by `output.schema`.
   */
  execute(args: unknown, exec: ToolRunContext): Promise<unknown>
  /**
   * Install execution-prepared content before `tools/post-execute` policies.
   * The callback is captured when the call starts and runs once for a
   * normalized outcome entering post-execute. Policy replacements remain
   * authoritative; pipeline failures that bypass post-execute skip projection.
   * @param exec - immutable execution identity and arguments.
   * @param result - normalized result before post-execute policy.
   * @returns replacement content, or undefined to preserve the renderer output.
   */
  projectContent?(exec: Readonly<ToolExecution>, result: Readonly<ToolExecutionResult>): ContentBlock[] | undefined
  /**
   * Synchronous last-mile transform for model-facing content. The executor
   * snapshots this callback when execution starts and invokes it exactly once
   * for every normalized outcome, including pipeline failures that bypass
   * `tools/post-execute`, immediately before lossless materialization.
   * Returning `undefined` preserves the content; every other result field
   * remains executor-owned. The callback must be total and must not throw.
   * @param exec - immutable execution identity and arguments.
   * @param result - complete normalized outcome before materialization.
   * @returns replacement content, or `undefined` to preserve it.
   */
  finalizeContent?(exec: Readonly<ToolExecution>, result: Readonly<ToolExecutionResult>): ContentBlock[] | undefined
  /**
   * Cooperative tool-call timeout budget in milliseconds. Omit for no deadline.
   * Enforced by `@deepseek-ai/dsh-tool-call-timeout-policy` (a `tools/execute` wrapper); it
   * is NEVER sent to the model — `schemas()` whitelists only name/description/
   * parameters. Declaring it asserts this tool forwards `exec.signal` to a
   * cooperative implementation that can reach quiescence when the signal aborts.
   */
  timeoutMs?: number
  /**
   * Pure synchronous classifier for overlap with sibling tool calls. Only
   * `true` opts in; omission, exceptions, non-`true` returns, and invalid
   * `defineTool` arguments are exclusive. This metadata is never model-visible.
   *
   * Opted-in executions must not mutate parent-owned state. Shared state must
   * tolerate concurrent dispatch; recorder races are permitted only when they
   * commute or fail closed. See the
   * [parallel-tool-call Agent Note](../../../../.agents/notes/implemented/feature/2026-07-10-parallel-tool-call-execution.md)
   * for the full contract.
   * @param args - parsed arguments; `defineTool` validates before calling.
   * @returns Whether this call may join a parallel group.
   */
  isConcurrencySafe?(args: unknown): boolean
  /**
   * Optional: how to present the PENDING state of one call in a UI, derived from
   * the call's `args` (parsed arguments, `unknown` — the tool validates/narrows
   * its own input). Returns a {@link ToolCallView} (a `card`-tagged render intent),
   * or `undefined` (or omit the method) to fall back to a generic presentation
   * (title = tool name, raw args as input). Pure and side-effect-free: a UI may
   * call it during live streaming AND a session-log replay, so it must depend
   * only on `args`.
   */
  presentCall?(args: unknown): ToolCallView | undefined
  /**
   * Optional: how to present the COMPLETED state, given the same `args` and the
   * durable result projection (`content`, failure state, and optional `meta`). Returns a
   * {@link ToolResultView}, or `undefined` (or omit the method) to keep the
   * pending title and render the raw result content. Pure and side-effect-free
   * for the same replay reason.
   */
  presentResult?(args: unknown, result: ToolResult): ToolResultView | undefined
}
```

`execute` は `args: unknown` を受け取る。生の `ToolDefinition` は自身の入力を検証する。ファーストパーティのツールはそれを手書きせず、`defineTool` を使う。これは引数を検証して絞り込み、`output.schema` から本体の戻り値を推論し、両方の出力プロジェクターに型を付ける。`finalizeContent` が型付き引数ではなく不変の実行を受け取るのは意図的である。無効な入力や外側のパイプライン失敗もそこに到達するためである。このコールバックは `isError`、正規値、構造化エラーの同一性、遅延コンテキスト、プレゼンテーション・メタデータを保ちながら、ツール所有のコンテンツ境界を適用してよい。

## 統一 JSON 値スキーマ DSL

プラグイン作者は、型付きパラメーターと型付き出力値に 1 つの語彙を使う。`ValueSchemaSpec` は `string`、`number`、`integer`、`boolean`、`null`、`array`、`object`、作者専用の `json`、および厳密に 1 つを選ぶ `oneOf` をサポートする。スカラーの `enum` と `const` の値は、そのノード型と一致しなければならない。明示的な object ノードは常に `additionalProperties: true | false` を宣言する。パラメーター定義は暗黙のオープンな object プロパティ・マップのままで、必須のプロパティごとに `required: true` が付く。

ソース: [`packages/core/tools/src/schema.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/tools/src/schema.ts)

```ts type-equiv
/** One author-facing schema for any lossless JSON value root. */
type ValueSchemaSpec =
  | StringValueSchemaSpec
  | NumberValueSchemaSpec
  | IntegerValueSchemaSpec
  | BooleanValueSchemaSpec
  | NullValueSchemaSpec
  | ArrayValueSchemaSpec
  | ObjectValueSchemaSpec
  | JsonValueSchemaSpec
  | OneOfValueSchemaSpec
```

```ts type-equiv
/** One implicit parameter-root property, optionally required. */
type ParameterPropertySpec = ValueSchemaSpec & { required?: true }
```

```ts type-equiv
/**
 * Tool parameter schema. The map itself is an implicit open object root;
 * requiredness remains a per-property `required: true` annotation.
 */
type ParameterSchemaSpec = {
  [key: string]: ParameterPropertySpec
  [key: symbol]: never
}
```

`{ type: 'json' }` は `JsonValue` を推論し、注釈のみの制約のない生スキーマにコンパイルする。出力ルートは object、array、スカラー、または null にできる。`InferValue<S>` はリテラル制約と object の開放性を 16 階層のコンテナまで尊重し、その後は TypeScript の型インスタンス化スタックを使い果たす代わりに `JsonValue` にフォールバックする。`InferArgs<P>` はプロパティごとの必須性を、必須と任意の文字列キーに変換する:

```ts type-equiv
/**
 * Infer the TypeScript value accepted by an author-facing value schema. Exact
 * inference is bounded to 16 container levels, then falls back to `JsonValue`.
 */
type InferValue<S> = InferValueAt<S, []>
```

```ts type-equiv
/** Infer the TypeScript argument object for an implicit parameter schema. */
type InferArgs<S> = InferProperties<S, []>
```

`defineTool({ name, description, parameters, output, execute, … })` は、パラメーター推論を `parameterSchemaSpecToJsonSchema()` と `validateArgs()` に結び付け、`execute`/`render`/`presentationMeta` を `InferValue<OutputSchema>` に結び付ける。スキーマ・レコードは自身の列挙可能な文字列キーのみを含み、スキーマ配列は密な組み込み配列であるため、推論、コンパイル、検証は同じ宣言を観察する。推論は 16 階層のコンテナまで正確なままで、その後 `JsonValue` に広がる。実行時の検証は完全なスキーマを歩き続ける。`valueSchemaSpecToJsonSchema()` は出力宣言を同じ強制の生サブセットを通じてコンパイルする。パラメーターの不一致は `ToolArgsError`（`INVALID_ARGS`）を throw する。無効な本体またはポリシー後の値は `ToolOutputError`（`INVALID_TOOL_OUTPUT`）を throw する。どちらも通常のツール・エラー経路を使う。生の JSON Schema はデフォルトでオープンである。サポートされないキーワードは、強制なしで受け入れられるのではなく拒否される。

登録は信頼された同一プロセスの契約である。レジストリは型付き定義を読み取り専用の入力として借用し、`output` を必須とし、その生スキーマを検証し、正の有限な `timeoutMs` などの意味要件をチェックする。`schemas()` はリクエストを構築するときにモデル向けのプロジェクションを構築するため、実行とプレゼンテーションはコールバックをワイヤに漏らすことなく、1 つの解決済み定義を共有する。

## `ToolRestriction` — 継承するものに対する 1 つのスコープのライブ・フィルター

`ToolRestriction` は、スコープが継承するツールに適用される。デプロイメント・グローバルのレイヤーと、そのチェーン上のすべての祖先スコープである。レジストリは読み取り専用の名前をプライベート・セットにコンパイルし、複数の制限を交差させ、その後、スコープ自身の登録を重ね合わせる。スコープ自身の登録は免除されたままで、委譲された子は自身が応答に使うツールを保持する。deny のみのフィルターは後から追加された未記載の継承ツールを許可し、allow リストはそれらを除外する。

```ts type-equiv
/**
 * Per-scope filter over global tools. Restrictions intersect and do not affect
 * scoped registrations or the reserved PTC mode transport.
 */
interface ToolRestriction {
  /** Global tool names that stay visible; everything else is removed. */
  readonly allow?: readonly string[]
  /** Global tool names removed from visibility. */
  readonly deny?: readonly string[]
}
```

## 条件付きアクティベーション — step 所有の受け入れ

登録は `when` 条件を持てる。AgentLoop は提案された step ごとに 1 回、キャプチャしたモデル選択とともに `ctx.tools.resolve()` を呼ぶ。返されたセットが、その step のツール宣言、executor セクション、およびそのバッチ内のすべての呼び出し（PTC プログラムのサブコールを含む）を決定する。`ctx.tools.executable()` は実行時にセットを適用し、セットがない場合は条件付き登録を除外する。同名の置換または新しい制限は、開始前の呼び出しが受け入れ済みの登録を使うことを止め、ループは step のバッチが決着した時点でセットを破棄する。

```ts type-equiv
/** Inputs for a tool's step-time activation condition. */
interface ToolActivationContext {
  /** Agent preparing the step. */
  readonly agent: Agent
  /** Provider, model, and optional reasoning effort captured for the step. */
  readonly modelSelection: Readonly<ModelSelection>
  /** Cancellation of step preparation; asynchronous conditions must observe it. */
  readonly signal: AbortSignal
}
```

```ts type-equiv
/** Lifecycle-owned conditions for a registered tool. */
interface ToolRegistrationOptions {
  /**
   * Decide whether the step's requests may call the tool. Omission enables
   * it. False excludes it from both declarations and dispatch; rejection fails
   * step preparation. Conditions must not mutate registrations or start tool work.
   * @param context - preparing agent, captured model selection, and cancellation signal.
   * @returns whether the tool participates in this step.
   */
  readonly when?: (context: ToolActivationContext) => boolean | Promise<boolean>
}
```

```ts type-equiv
/** One step's admitted registrations, also used by nested PTC calls. */
interface ResolvedToolSet {
  /** Close admission after the step's tool batch settles. */
  dispose(): void
  /**
   * Read admitted registrations that remain visible in their original scope.
   * @param scope - exact requesting agent; another scope is never admitted.
   * @returns original definitions excluding removed or restricted registrations.
   */
  view(scope: ScopeKey | undefined): ReadonlyMap<string, ToolDefinition>
}
```

## Executor と生の呼び出しバッチ

`ctx.tools.execute()` は完全な生のモデル呼び出しバッチを最も近いスコープ付き executor に委譲する。サービスは各バッチの開始時に現在のプロバイダーを選択し、プロンプトの組み立てはモデル入力のみを返す。`agent` は単独のサービス呼び出しでは存在しない。`log` は初期化フックなど、モデル step の外にあるプログラム的な呼び出しでは存在しない。`log` のない Agent もツール・アクセスをスコープするが、ルートの呼び出し／結果イベントを生成しない。ツール所有のイベントは引き続きその Session を使える。

```ts type-equiv
/** Model-call schemas produced by an executor. */
type ToolModelInput = Pick<ToolProviderResult, 'schemas' | 'knownNames'>
```

```ts type-equiv
/** Raw model calls, with optional Agent ownership and model-step logging. */
interface ToolBatchInput {
  readonly calls: readonly ToolCallBlock[]
  readonly signal: AbortSignal
  /**
   * Positive integer overriding the executor's concurrency limit for this batch.
   * Applies to concurrency-safe root calls; program subcall limits remain separate.
   * Undefined uses the executor's configured limit.
   */
  readonly maxParallelToolCalls?: number
  /**
   * Owning Agent supplies scoped tools and execution context, including outside a turn.
   * Undefined for standalone calls outside an Agent, such as a service invocation.
   */
  readonly agent?: Agent
  /**
   * Step admission for every call in the batch, including program subcalls.
   * Undefined admits only unconditional registrations.
   */
  readonly toolSet?: ResolvedToolSet
  /**
   * Committed model step receiving root call/result events; requires `agent`.
   * Undefined when `agent` is absent or for calls outside a model response, such
   * as session initialization hooks. Without `log`, root call/result events are
   * omitted while `agent`, when provided, still supplies scope and execution context.
   */
  readonly log?: {
    readonly turn: number
    readonly step: number
  }
}
```

```ts type-equiv
/** One settled result correlated with its original model call. */
interface ToolCallResult {
  readonly callId: ToolCallId
  readonly result: ToolExecutionResult
}
```

```ts type-equiv
/** Model-ordered outcomes and context accepted for the next step. */
interface ToolBatchResult {
  readonly results: readonly ToolCallResult[]
  readonly additionalContexts: readonly UserMessage[]
  readonly concluded: boolean
}
```

```ts type-equiv
/** A plugin owns argument parsing, dispatch, scheduling, cancellation, and logging. */
interface ToolExecutor {
  /** Guidance registered and disposed with this executor; absent means no executor-owned sections. */
  readonly sections?: readonly ToolPromptSection[]
  /**
   * Describe the model's callable tools.
   * @param tools - currently visible capability definitions, after scoped restrictions.
   * @returns schemas and names used to validate tool ordering.
   */
  describe(tools: readonly ToolDefinition[]): ToolModelInput
  /**
   * Execute a complete batch. With log, commit root call/result events to the
   * Agent's Session; without log, omit those events. Tool-owned events may still
   * use the owning Agent's Session. Return
   * one outcome per input call in input order, including skipped calls on abort.
   * Never settle while started work is still running. Internal failures reject
   * with ToolBatchExecutionError after draining, preserving committed outcomes
   * and context; callers must stop dependent work and repair missing log results.
   * An explicit maxParallelToolCalls takes precedence over executor configuration.
   * @param input - original calls, cancellation, optional concurrency override, Agent and logging step.
   * @returns settled results, deferred context, and whether a success concludes the turn.
   */
  execute(input: ToolBatchInput): Promise<ToolBatchResult>
}
```

## 実行: 拡張可能なウォーターフォールと単調ポリシー

組み込み executor は解析済みの `ToolExecutionInput` 値を共有の `ToolExecutionPipeline` に渡す。パイプラインは引数のスナップショット、実行の同一性、ポリシー・ウォーターフォール、ディスパッチ、最終結果の通知を所有し、呼び出しごとに 1 つの `ToolExecutionResult` を返す。カスタム executor は独自の実行を実装できる。公開 `ToolExecutor` が必要とするのは説明とバッチ実行だけである。

```ts type-equiv
/** Opaque call identity that permits correlation without exposing mutable execution state. */
type ToolExecutionToken = symbol & { readonly [toolExecutionTokenBrand]: true }
```

```ts type-equiv
/**
 * Parsed input to the shared invocation pipeline. The pipeline
 * adds its own token to form a {@link ToolExecution};
 * callers do not choose that token.
 */
interface ToolExecutionInput {
  readonly callId: ToolCallId
  /**
   * Root model-requested call owning this execution tree. Callers omit it for
   * a root execution; nested dispatchers propagate the enclosing value.
   */
  readonly rootCallId?: ToolCallId
  readonly name: string
  /** Binding-time tool schema for a PTC inner call; frozen by its producer and never logged. */
  readonly schema?: ToolSchema
  /** Losslessly JSON-serializable parsed arguments (tools validate their own schema). */
  readonly arguments: unknown
  /** The agent on whose behalf the call runs (set by the agent loop). */
  readonly agent?: Agent
  /** Step admission shared by Native and PTC dispatch. Conditional tools require it. */
  readonly toolSet?: ResolvedToolSet
  /**
   * Opaque token of the enclosing transport execution, when one exists. PTC
   * mode sets this on SDK sub-dispatches so commit-style observers can wait for
   * the outer `run_code` outcome without receiving its live mutable execution.
   * The token also marks the call as a transport sub-dispatch rather than a
   * model-direct call: with the PTC executor selected, only calls with a parent may
   * execute a native tool name — a model-direct call (no parent) is denied as
   * `UNKNOWN_TOOL` before the policy pipeline.
   */
  readonly parent?: ToolExecutionToken
  /** Required caller-owned cancellation for this invocation. */
  readonly signal: AbortSignal
}
```

ツール本体は実行時拡張を受け取る。`deferContext()` はコンテキストをその実行自身の結果に付加する。これは複合ツールのネスト・ディスパッチ・チャネルであり、プラグイン由来の命令を作り出すリーフ・ツールも使える。まだ開いている外側の呼び出しの内側に注入することはない。

```ts type-equiv
/**
 * Runtime context handed to a tool implementation after the executor has
 * accepted a {@link ToolExecution}. {@link deferContext} attaches context to
 * this execution's own result — a composite tool ferries nested-dispatch
 * context back to the outer result, and a leaf tool may mint a fresh
 * plugin-sourced instruction; the loop appends it only after the
 * `tool/result`.
 */
interface ToolRunContext extends ToolExecution {
  /**
   * Defer one context — typically a nested-dispatch context ferried by a
   * composite tool, or a fresh plugin-sourced instruction — until this tool's
   * final result reaches the agent loop. Contexts retain their individual
   * source and metadata and are emitted in call order.
   */
  deferContext(context: UserMessage): void
  /**
   * Mark a successful final result as terminal for the current agent turn.
   * The marker rides this execution's own result (`concludesTurn` exists only
   * on {@link ToolExecutionSuccess}); a composite that dispatches nested
   * calls forwards it from the nested result, exactly like
   * `additionalContexts`, so only an authoritative nested success can
   * conclude the enclosing run.
   */
  concludeTurn(): void
}
```

agent ループは保留中の各呼び出しの実行モードをレジストリに問い合わせ、排他的バリアとローリング・プールの並列実行を形成するために使う:

```ts type-equiv
/**
 * Scheduling mode for one pending call. `parallel` may overlap with siblings;
 * `exclusive` runs alone and forms an ordering barrier.
 */
type ToolExecutionMode =
  | { kind: 'parallel' }
  | { kind: 'exclusive' }
```

PTC バインディングは構築時に ToolSchema を凍結し、スケジューラを通じて `ToolExecution.schema` に渡す。これは一時的な実行メタデータである。ポリシーの前に、実際に開始された各サブディスパッチはペアリング ID、名前、正規化された引数のみを記録する。開始イベントも決着イベントも、説明、パラメーター、スキーマをシリアライズしない。ブリッジは後で決着した呼び出しを `tools/ptc-dispatch-log` ウォーターフォールに公開し、これはプログラム値、モデルから見える外側の結果、構造化失敗の同一性を保ちながら、永続コンテンツのコピーを変更してよい:

```ts type-equiv
/**
 * One settled `run_code` sub-dispatch about to be logged, as seen by the
 * `tools/ptc-dispatch-log` waterfall: the parent execution (session owner,
 * outer call identity), the sub-call identity, and the outcome whose durable
 * copy a listener may reshape. `content` is the RENDERED result projection
 * (what a native `tool/result` would carry) — the program itself received
 * the structured `value` (or just the error message on failure); only the
 * `tool/ptc-dispatch` event's copy changes.
 */
interface PtcDispatchLog {
  /** The outer `run_code` execution. */
  readonly exec: ToolExecution
  /** The calling agent (the scope routing key and the spill owner), when the outer call has one. */
  readonly agent?: Agent
  /** Opaque sub-call id; new calls use `<parent>:ptc:<n>`. */
  readonly subCallId: ToolCallId
  /** The dispatched sub-tool name. */
  readonly name: string
  /** Whether the sub-call settled as an error. */
  readonly isError: boolean
  /** The sub-call's complete model-facing content (the settle event's default payload). */
  readonly content: ContentBlock[]
}
```

```ts type-equiv
/**
 * One pending tool call inside the executor pipeline. Parsed arguments cross
 * one lossless-JSON materialization boundary before policy and are deep-frozen;
 * call identity, the caller signal, and the executor-assigned {@link token} are
 * readonly. The executor freezes the complete object before `tools/result`
 * observers run.
 */
interface ToolExecution extends ToolExecutionInput {
  /** Root model-requested call, resolved for every root and nested execution. */
  readonly rootCallId: ToolCallId
  /** Executor-assigned identity shared with nested calls only as their opaque `parent` token. */
  readonly token: ToolExecutionToken
}
```

```ts type-equiv
/**
 * Around-dispatch view of a {@link ToolExecution}. A `tools/execute` wrapper
 * may replace the signal for its delegated lifetime, but it cannot remove it.
 * The executor fuses every replacement with the captured caller signal.
 */
interface ToolDispatchExecution extends Omit<ToolExecution, 'signal'> {
  /** Cancellation signal visible to the next wrapper or tool body. */
  signal: AbortSignal
}
```

`ToolExecutionToken` は同一性の比較にのみ使われる不透明な実行時 `Symbol` である。ポリシーの前に、`execute()` は引数をマテリアライズして凍結し、JSON でない入力を拒否し、トークンを割り当てる。同一性フィールド、必須の呼び出し元シグナル、任意の親トークンは読み取り専用のままである。`ToolDispatchExecution` ラッパーはシグナルを置換できるが削除はできない。レジストリは本体を呼び出す前に呼び出し元シグナルを再融合する。最終オブザーバーは凍結された実行の同一性を受け取る。

`ToolGuard` はスコープを認識する最終的なディスパッチ前ポリシーである。その戻り値の型には意図的に allow の結果がない。`undefined` はウォーターフォールの決定を維持し、返された理由は許可を減らすことしかできないため、後続のリスナーがそれを取り消すことはできない。

```ts type-equiv
/**
 * A monotonic execution guard evaluated after every `tools/pre-execute`
 * listener and before the tool body. Returning a reason denies the call;
 * returning `undefined` leaves it unchanged. Because guards have no allow
 * result, listener ordering cannot turn a denial back into permission.
 * @param execution - the identity-protected call after extensible pre-execute policy completed.
 * @returns a final denial reason, or `undefined` to leave the call allowed.
 */
type ToolGuard = (execution: Readonly<ToolExecution>) => string | undefined
```

```ts type-equiv
/** Canonical failure detail; internal routing information remains optional. */
interface ToolFailure {
  /** Human-readable failure message without the Native `Error: ` envelope. */
  message: string
  /** Internal error class/code used by policy and durable diagnostics. */
  info?: ToolErrorInfo
}
```

```ts type-equiv
/** Successful canonical tool execution, including its Native/model projection. */
interface ToolExecutionSuccess {
  readonly isError: false
  /** Execution-local canonical value; deliberately omitted from durable events. */
  readonly value: JsonValue
  readonly content: ContentBlock[]
  readonly error?: never
  readonly meta?: JsonValue
  readonly additionalContexts?: UserMessage[]
  /** The agent loop stops after committing this successful result batch. */
  readonly concludesTurn?: true
}
```

```ts type-equiv
/** Failed canonical tool execution; failures never carry a successful value. */
interface ToolExecutionFailure {
  readonly isError: true
  readonly error: ToolFailure
  readonly value?: never
  readonly content: ContentBlock[]
  readonly meta?: JsonValue
  readonly additionalContexts?: UserMessage[]
  readonly concludesTurn?: never
}
```

```ts type-equiv
/** The discriminated, execution-local outcome of one tool call. */
type ToolExecutionResult = ToolExecutionSuccess | ToolExecutionFailure
```

```ts type-equiv
/** Structured error metadata for a failed tool call (alongside the model-facing text). */
interface ToolErrorInfo {
  name: string
  code: string
  /** Optional raw user-facing detail; durable projections preserve it but model-facing content does not include it. */
  reason?: string
}
```

結果は結果のみを含む。呼び出しの同一性は、すべてのフックを通じて付随する不変の `ToolExecution` と、永続的な `tool/call` / `tool/result` Session イベントに残るため、ラッパーが 2 つ目の食い違う同一性を作ることはできない。正規の `value` は実行ローカルである。ループは `content`、`error`、`meta` のみを永続化し、`tool/ptc-dispatch` はサブコールのレンダリング済み `content`、`isError`、任意の構造化 `error`、プレゼンテーション `meta` を保存する。リプレイはプレゼンテーションを再現するが、正規の中間値を再構築することはできない。任意の `ToolErrorInfo.reason` は、モデル向けコンテンツに追加せずに、生のユーザー向け詳細を保持する。

成功時、レジストリは本体の値をスナップショットして検証し、凍結し、ネイティブ呼び出しとネスト呼び出しのために純粋なレンダラーと任意のプレゼンテーション・メタデータ・プロジェクターを呼び出す。`tools/result` の直前に、永続プレゼンテーション・フィールドを別途マテリアライズする。無効な値、レンダラー／プロジェクターの失敗、または JSON でないプレゼンテーションは、JSON 安全な `isError` になる。したがって最終のライブ・オブザーバーは、後の永続追記に安全なフィールドの隣で、正確な実行ローカルの値を見る。

最終コンテンツの前に、レジストリは候補の結果をマテリアライズする。コンテンツ、構造化エラー、追加コンテキスト、プレゼンテーション・メタデータの失敗は、引き続き `finalizeContent` に到達する JSON 安全な `isError` 結果になる。レジストリはそのコールバックを正確に 1 回呼び出し、`tools/result` の直前に受け入れ済みの結果をマテリアライズして凍結するため、観察されるライブの結果は後の永続的な `tool/result` 追記に安全である。

各インターセプト・ウォーターフォールは型付きの **Decision** を返す（`agent/*` ウォーターフォールと共有されるイディオム）。`tools/pre-execute` のリスナーは `(exec, next)` を受け取り `PreToolDecision` を返す。`tools/execute` のラッパーは `ToolExecutionResult` を返す。`tools/post-execute` のリスナーは `(exec, result, next)` を受け取り `PostToolDecision` を返す:

```ts type-equiv
/**
 * Pre-dispatch decision. `allow` runs the call; `deny` materializes its
 * model-facing reason and optional structured error identity; `cancel` selects
 * the canonical cancellation result without presenting a policy denial; `ask`
 * runs only after an approval service returns `allowed-once` and otherwise
 * denies; its `reason` is the audited approval reason and its optional
 * `displayReason` is the localized prompt text. Input rewriting is excluded because arguments are already logged and
 * presented.
 */
type PreToolDecision =
  | { kind: 'allow' }
  | { kind: 'deny'; reason: string; info?: ToolErrorInfo }
  | { kind: 'cancel' }
  | { kind: 'ask'; reason?: string; displayReason?: { readonly en: string; readonly [locale: string]: string } }
```

```ts type-equiv
/**
 * Post-dispatch decision: accept, replace one projection, attach context for the
 * next request, or block by turning corrective feedback into an error result.
 */
type PostToolDecision =
  | { kind: 'accept'; content?: ContentBlock[]; value?: never; additionalContexts?: UserMessage[] }
  | { kind: 'accept'; value: JsonValue; content?: never; additionalContexts?: UserMessage[] }
  | { kind: 'block'; feedback: ContentBlock[]; additionalContexts?: UserMessage[] }
```

デフォルトには `next()` を呼び、ショートサーキットするには決定を返す。事前ポリシーは deny または ask を行える。`allowed-once` のみが続行し、許可されない場合、承認チャネルまたはサービスがない場合、agent のないリクエストは拒否になる。deny はモデル向けの理由を変えずに、構造化された同一性とユーザー向けの詳細を付加できる。ガードは引き続き最終的な拒否を課せる。履歴、監査、UI、実行が一致しなければならないため、引数は書き換えられない。

事後ポリシーはコンテンツまたは値のいずれかを置換でき、両方は置換できない。コンテンツの置換は正規値と既存のメタデータを保持する。値の置換は再検証され、コンテンツ／メタデータを再計算する。ブロックは値を削除し、修正フィードバックを含む `isError` になる。コンテンツの置換はプレゼンテーション・ポリシーであり、機密性ポリシーではない。プログラム値を隠す必要があるリスナーはブロックするか値を置換する。`tools/result` は正規化後の凍結された実行と結果を受け取る。オブザーバーはそれらを変換できず、オブザーバーの失敗は封じ込められる。不明なツールと throw するツールはどちらも構造化エラーになる（`ToolNotFoundError` は `UNKNOWN_TOOL` に写像される）。そのため呼び出しはターンを終了せずに失敗する。

## 強制される生 JSON Schema サブセット

サブエージェント、workflow、MCP、動的登録からの生スキーマは、作者 DSL のワイヤレベル相当を使う。`assertSupportedJsonSchema()` は任意の JSON ルートを受け入れ、`validateJsonSchemaValue()` がそれを強制し、`JsonSchemaError` がサポート外または不正なスキーマ・パスをすべて報告する。空の注釈のみのノードは、制約のないロスレス JSON を意味する。`oneOf` は少なくとも 2 つのブランチを必要とし、値は正確に 1 つに一致しなければならない。依然として object ルートを必要とする consumer は `assertObjectJsonSchema()` を呼び、`ObjectJsonSchema` を使う。これにより、共有語彙を制限せずに、サブエージェント／workflow の呼び出し元定義の構造化出力が object ルートのままになる。

```ts type-equiv
/** Scalar JSON values supported by `enum` and `const`. */
type JsonSchemaScalar = string | number | boolean | null
```

```ts type-equiv
/** Single-type keywords accepted by the enforced subset. */
type JsonSchemaType = 'object' | 'array' | 'string' | 'number' | 'integer' | 'boolean' | 'null'
```

```ts type-equiv
/**
 * One raw JSON Schema node in the enforced subset. The optional fields express
 * the external wire schema; {@link assertSupportedJsonSchema} rejects invalid
 * combinations before a caller treats the node as trusted.
 */
interface JsonSchemaNode {
  /** Omit with no constraints for any JSON value, or use `oneOf`. */
  type?: JsonSchemaType
  /** Exactly one branch must validate; at least two branches are required. */
  oneOf?: JsonSchemaNode[]
  /** Nested property schemas (`type: 'object'` only). */
  properties?: Record<string, JsonSchemaNode>
  /** Required property names; each must appear in `properties`. */
  required?: string[]
  /** `false` rejects undeclared keys; absent/`true` follows JSON Schema's open default. */
  additionalProperties?: boolean
  /** Item schema (`type: 'array'` only); absent accepts any JSON item. */
  items?: JsonSchemaNode
  /** Allowed values for a scalar node. */
  enum?: JsonSchemaScalar[]
  /** The single allowed value for a scalar node. */
  const?: JsonSchemaScalar
  /** Annotation, ignored for validation. */
  description?: string
  /** Annotation, ignored for validation. */
  title?: string
  /** Annotation, ignored for validation but required to be lossless JSON. */
  default?: JsonValue
  /** Annotation, ignored for validation but required to be lossless JSON. */
  examples?: JsonValue
}
```

```ts type-equiv
/** A consumer-constrained object-rooted schema. */
type ObjectJsonSchema = JsonSchemaNode & { type: 'object' }
```

## ツール・プレゼンテーションの UI 語彙

ツールが自身の呼び出しを UI（エディターのツール呼び出しカード、CLI のログ行）でどう表示してほしいか。プロバイダー中立であり、ツールはクライアント・プロトコルに依存せずに自身を記述できる。`presentCall`/`presentResult` は **`card` タグ付きレンダー・インテント**を返す。UI ブリッジが切り替える判別可能なユニオンである:

- `ToolCallView`（保留中）: `{ card: 'generic', title, kind?, rawInput?, content?, locations? }`（デフォルトのカード。`locations` は呼び出しが読み取り／変更するファイルの `{ path, line? }[]` で、エディターの追従用）、`{ card: 'terminal', title, description?, cwd? }`（シェル・コマンド → ターミナル・カード）、または `{ card: 'diff', title, diffs, locations? }`（ファイルの作成／変更 → インライン diff カード。`diffs` は `{ path, oldText, newText }[]` で、新しいファイルでは `oldText: null`）。
- `ToolResultView`（完了）: `{ card: 'generic', title?, content? }`、`{ card: 'terminal', title?, output?, exitCode?, signal? }`（キャプチャした実行出力 + 終了。能力のある UI は終了ステータスのピルを表示し、別の UI はフェンス付きの ` ```console ` フォールバックを導出してよい）、`{ card: 'diff', title?, diffs }`（完了したファイル変更 → 表示する変更。通常は適用済みの hunk と、前後のコンテンツから計算したコンテキスト行。前イメージがない場合はファイル全体の diff）、`{ card: 'search', shape, title?, truncated, total, … }`（完了した発見検索 → `shape: 'matches'`（grep）ではファイルごとにグループ化したマッチ、`shape: 'paths'`（glob）ではフラットなパス・リスト。`truncated`/`total` はインライン結果が上限に達したかどうかを報告し、UI が部分的な結果を完全なものとして提示しないようにする。ビューは結果テキストを含まない。検索カードのない UI は生の結果コンテンツにフォールバックする）、`{ card: 'read', title?, path, offset, lines, totalLines, lang?, content? }`（完了したファイル読み取り → 行番号付きで、任意で構文ハイライトされたコード・ビュー。`offset` はウィンドウが要求した 1 始まりの最初の行で、`lines` が空でも保持される。`lang` は拡張子からの言語ヒントで、`content` はエンベロープを除いたテキストであり、読み取りサポートのない UI がフォールバックする）、または `{ card: 'web', kind: 'search' | 'fetch', title?, … }`（完了した Web 取得。`kind: 'search'` は構造化された `sources`/`answer?`/`truncated` を含み、`kind: 'fetch'` は `url`/`statusCode`/`truncated` を含む。`web` ケイパビリティのない UI は生の結果コンテンツにフォールバックする。本文はビューに複製されない）。完了ビューは保留ビューを置き換えるため、変更ツールは呼び出し時のスニペットを複製する場合でも diff 結果を返す。検索と Web 取得には呼び出し時の `card` 相当がない（構造化結果は `execute` の後にのみ存在するため、保留状態は汎用カードのままである）。

`ToolCallKind`（`'read' | 'edit' | 'delete' | 'move' | 'search' | 'execute' | 'fetch' | 'other'`）は汎用カードのアイコンを選ぶ。`FileLocation`（`{ path, line? }`）、`FileDiff`（`{ path, oldText, newText }`）、`ReadFileLine`（`{ number, text }`、読み取りウィンドウの、1 始まりの番号が付いた 1 行）は共有のファイル・カード語彙である。設計は [render-intent-union Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-07-02-tool-render-intent-union.md) に固定されている。host/client のランタイムはこの中立語彙を自身のビューに投影する。

プレゼンテーション・フィールドの完全なドキュメントは [`packages/core/tools/src/presentation.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/tools/src/presentation.ts) にある。`bash` のスキーマと executor は [shell.md](shell.md) に、汎用のバックグラウンド制御は [jobs.md](jobs.md) にある。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxtools--toolsservice"></a>

### `ctx.tools` — `ToolsService`

具体的なツール・レジストリとスコープ付き executor ディスパッチャー。

```ts cordis-catalog
/**
 * Register globally or in the calling agent scope. Scoped tools shadow
 * globals; duplicates within one layer and the reserved `run_code` name fail.
 * @param definition - tool schema, execution, and optional finalization/presentation callbacks.
 * @param options - optional step-time activation condition.
 * @returns the exact disposer that unregisters the tool.
 */
register(definition: ToolDefinition, options: ToolRegistrationOptions = {}): () => void

/**
 * Restrict global tools for the calling agent scope. Empty filters, unknown
 * names, scope-local names, and reserved transport names fail. Restrictions
 * intersect; scoped registrations remain visible.
 * @param filter - global-tool mask: `allow` (keep only) and/or `deny` (remove).
 * @returns the exact disposer that lifts this restriction.
 */
restrict(filter: ToolRestriction): () => void

/**
 * Look up a tool as one scope sees it (scoped
 * shadows global; a restricted-away global reads as absent). Presenters pass
 * the calling agent so the rendered card matches the definition that
 * actually executed.
 * @param name - the tool name as registered.
 * @param scope - the viewing scope (the agent); omitted = the global view.
 * @param toolSet - step admission; omit only to inspect registered definitions.
 * @returns the definition the scope resolves, or undefined when none is visible.
 */
get(name: string, scope?: ScopeKey, toolSet?: ResolvedToolSet): ToolDefinition | undefined

/**
 * Resolve the definition a call may execute. With a step's admitted set, the
 * set decides; without one, conditional registrations are unavailable.
 * @param name - the tool name as registered.
 * @param scope - the calling agent; omitted = the global view.
 * @param toolSet - step admission, or omitted for a call outside a model step.
 * @returns the executable definition, or undefined when the call must fail as `UNKNOWN_TOOL`.
 */
executable(name: string, scope?: ScopeKey, toolSet?: ResolvedToolSet): ToolDefinition | undefined

/**
 * Resolve conditions once for this step without changing shared registrations.
 * Cancellation or a rejected condition publishes no partial selection. Removed
 * registrations cannot be restored by a late condition result.
 * @param context - preparing agent, captured model selection, and cancellation.
 * @returns the complete selection; the caller disposes it after its tool batch settles.
 */
async resolve(context: ToolActivationContext): Promise<ResolvedToolSet>

/**
 * Read the visible capability definitions for descriptions and program bindings.
 * @param scope - viewing Agent or preset; omitted selects the global view.
 * @param toolSet - step admission; omit only to inspect registered definitions.
 * @returns definitions after scoped shadowing and restrictions, limited to the step's admission when given.
 */
definitions(scope?: ScopeKey, toolSet?: ResolvedToolSet): readonly ToolDefinition[]

/**
 * Project visible definitions onto the allowlisted model-facing schema fields,
 * excluding execution and presentation callbacks.
 * @param scope - the viewing scope (the agent); omitted = the global view.
 * @param toolSet - step admission, or omitted for registered-definition inspection.
 * @returns one deep-cloned schema per visible tool.
 */
schemas(scope?: ScopeKey, toolSet?: ResolvedToolSet): ToolSchema[]

/**
 * Provide the executor for the calling scope; an unscoped contribution is the deployment default.
 * Rendering any executor-owned section requires the viewing scope's selection to remain available.
 * @param executor - initialized provider owned by the contributing plugin.
 * @returns the effect disposer. Unloading leaves the selection unavailable until it is replaced.
 */
setExecutor(executor: ToolExecutor): () => void

/**
 * Delegate a raw call batch to its Agent's provider or the deployment default.
 * Select at dispatch; accepted batches settle through that provider even if it unloads.
 * @param input - original calls, optional owning Agent, and optional logging step.
 * @returns the selected executor's settled batch; unavailable selections reject.
 */
async execute(input: ToolBatchInput): Promise<ToolBatchResult>

/**
 * Register a monotonic denial in the calling scope.
 * @param guard - evaluated after extensible pre-execute policy by the shared pipeline.
 * @returns the effect disposer that removes this guard.
 */
guard(guard: ToolGuard): () => void

/**
 * Check global and inherited guards after pre-execute policy and approval resolve.
 * @param exec - invocation whose guards are checked before dispatch.
 * @returns the first denial reason, or undefined when no guard denies.
 */
checkGuards(exec: ToolExecution): string | undefined
```

型: [ScopeKey](scope.md)

ソース: [`packages/core/tools/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/tools/src/index.ts)

<a id="tools-events"></a>

### `tools/*` イベント

<a id="toolschange--emit"></a>

#### `tools/change` — emit

ツールが登録または登録解除された、あるいはスコープ付き制限が変更された。フィルターされないレジストリ通知である。グローバルのコントリビューションはすべての agent の次回アセンブリに影響しうるため、スコープ付きリスナーはすべてのスコープからの変更を受け取る。

```ts cordis-catalog
/**
 * A tool was registered or unregistered, or a scoped restriction changed.
 * Unfiltered registry notification: scoped listeners receive changes from every
 * scope because a global contribution can affect every agent's next assembly.
 * @mode emit
 */
'tools/change'(): void
```

ソース: [`packages/core/tools/src/events.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/tools/src/events.ts)

<a id="toolsexecute--waterfall"></a>

#### `tools/execute` — waterfall

タイムアウト、リトライ、メトリクスのためのディスパッチ周辺ウォーターフォール。`next()` は正規化された結果を返す。ラッパーは `exec.signal` のみを変更でき、呼び出しの同一性は不変のままである。レジストリは本体の前に元の呼び出し元シグナルを再融合するため、置換によって呼び出し元のキャンセルを切り離すことはできない。ラッパーは依然として自身のシグナルを復元し、静止状態に到達しなければならない。スコープ・フィルター付きディスパッチ（`@deepseek-ai/dsh-scope`）: agent スコープのリスナーはその agent の呼び出しのみを受け取る。

```ts cordis-catalog
/**
 * Around-dispatch waterfall for timeout, retry, or metrics. `next()` returns
 * a normalized result; wrappers may change only `exec.signal`, while call
 * identity remains immutable. The registry re-fuses the original caller
 * signal before the body, so replacement cannot detach caller cancellation;
 * wrappers must still restore their signal and reach quiescence.
 * Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): agent-scoped listeners receive only that agent's calls.
 * @param exec - the allowed call about to dispatch (name, parsed arguments, caller agent, signal).
 * @mode waterfall
 */
'tools/execute'(this: Scoped<ToolsService>, exec: ToolDispatchExecution, next: () => Promise<ToolExecutionResult>): Promise<ToolExecutionResult>
```

型: [Scoped](scope.md)

ソース: [`packages/core/tools/src/events.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/tools/src/events.ts)

<a id="toolspost-execute--waterfall"></a>

#### `tools/post-execute` — waterfall

正規化されたディスパッチ結果を受け入れ、置換し、拡充し、またはブロックする。`next()` はそれを変更せずに受け入れる。throw するツールもエラーとしてこのウォーターフォールに到達する。非同期リスナーは `exec.signal` を監視しなければならない。それらが決着した後、呼び出し元のキャンセルは、成功して受け入れられた結果のみを、ツール本体が呼び出されたかどうかで選ばれるコードに置き換える。スコープ・フィルター付きディスパッチ（`@deepseek-ai/dsh-scope`）: agent スコープのリスナーはその agent の呼び出しのみを受け取る。

```ts cordis-catalog
/**
 * Accept, replace, enrich, or block a normalized dispatch result. `next()`
 * accepts it unchanged; thrown tools still reach this waterfall as errors. Async
 * listeners must observe `exec.signal`; after they settle, caller
 * cancellation replaces only a successful accepted outcome with the code
 * selected by whether the tool body was invoked.
 * Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): agent-scoped listeners receive only that agent's calls.
 * @param exec - the call that just ran (name, parsed arguments, caller agent).
 * @param result - the dispatch outcome a listener may accept, replace, or block.
 * @mode waterfall
 */
'tools/post-execute'(this: Scoped<ToolsService>, exec: ToolExecution, result: Readonly<ToolExecutionResult>, next: () => Promise<PostToolDecision>): Promise<PostToolDecision>
```

型: [Scoped](scope.md)

ソース: [`packages/core/tools/src/events.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/tools/src/events.ts)

<a id="toolspre-execute--waterfall"></a>

#### `tools/pre-execute` — waterfall

ディスパッチ前に許可、拒否、キャンセル、または ask を行う。`next()` は許可に委譲する。`cancel` は正規のディスパッチ前キャンセル結果を選択し、承認サポートがない場合は `ask` が拒否になる。非同期ゲートは `exec.signal` を監視しなければならない。レジストリはそれらが決着した後にキャンセルを再チェックするが、その promise を放棄することはない。スコープ・フィルター付きディスパッチ（`@deepseek-ai/dsh-scope`）: agent スコープのリスナーはその agent の呼び出しのみを受け取る。

```ts cordis-catalog
/**
 * Allow, deny, cancel, or ask before dispatch. `next()` delegates to allow;
 * `cancel` selects the canonical pre-dispatch cancellation result, and missing
 * approval support turns `ask` into denial. Async gates must observe
 * `exec.signal`; the registry rechecks cancellation after they settle but
 * never abandons their promise.
 * Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): agent-scoped listeners receive only that agent's calls.
 * @param exec - the pending call (name, parsed arguments, caller agent).
 * @mode waterfall
 */
'tools/pre-execute'(this: Scoped<ToolsService>, exec: ToolExecution, next: () => Promise<PreToolDecision>): Promise<PreToolDecision>
```

型: [Scoped](scope.md)

ソース: [`packages/core/tools/src/events.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/tools/src/events.ts)

<a id="toolsptc-dispatch-log--waterfall"></a>

#### `tools/ptc-dispatch-log` — waterfall

ブリッジが `tool/ptc-dispatch` イベントを追記する前に、リスナーが 1 つの `run_code` サブディスパッチ結果の永続ログ・コピー内のコンテンツを置換できるようにする。`next()` はコンテンツを変更せずに保つ。リスナーは置換ブロックを返してよい（例: 大きすぎるテキスト結果に対する spill ポリシーのプレビュー + ロケーター）。影響を受けるのはログに記録されたコピーだけである。プログラムはすでに完全な値を受け取っており、モデルはどちらも見ない。throw するリスナーは封じ込められる。ブリッジは元の決着済みコンテンツのログ記録にフォールバックする。スコープ・フィルター付きディスパッチ（`@deepseek-ai/dsh-scope`）: agent スコープのリスナーはその agent のディスパッチのみを受け取る。

```ts cordis-catalog
/**
 * Allow a listener to replace content in the DURABLE LOG COPY of one
 * `run_code` sub-dispatch outcome before the bridge appends its
 * `tool/ptc-dispatch` event. `next()` keeps the
 * content unchanged; a listener may return replacement blocks (e.g. the
 * spill policy's preview + locator for an oversized text result). Only the
 * logged copy is affected — the program already received the complete
 * value, and the model sees neither. A throwing listener is contained:
 * the bridge falls back to logging the original settled content.
 * Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): agent-scoped listeners receive only that agent's dispatches.
 * @param dispatch - the parent execution, sub-call identity, and the settled content to log.
 * @mode waterfall
 */
'tools/ptc-dispatch-log'(this: Scoped<ToolsService>, dispatch: PtcDispatchLog, next: () => Promise<ContentBlock[]>): Promise<ContentBlock[]>
```

型: [ContentBlock](llm-streaming.md) · [Scoped](scope.md)

ソース: [`packages/core/tool-executor-ptc/src/runtime-types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/tool-executor-ptc/src/runtime-types.ts)

<a id="toolsresult--emit"></a>

#### `tools/result` — emit

凍結されたロスレス JSON の最終結果を観察する。リスナーの失敗は封じ込められる。スコープ・フィルター付きディスパッチ（`@deepseek-ai/dsh-scope`）: `exec.agent` をキーとする。

```ts cordis-catalog
/**
 * Observe the frozen, lossless-JSON final outcome. Listener failures are contained.
 * Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): keyed by `exec.agent`.
 * @param exec - the execution object that traversed the pipeline.
 * @param result - a deep-frozen snapshot of the final returned result.
 * @mode emit
 */
'tools/result'(this: Scoped<ToolsService>, exec: Readonly<ToolExecution>, result: Readonly<ToolExecutionResult>): undefined
```

型: [Scoped](scope.md)

ソース: [`packages/core/tools/src/events.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/tools/src/events.ts)
<!-- END GENERATED cordis-surface -->
