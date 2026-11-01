# ワークフロー

日本語 | [English](../../../docs/subsystems/workflow.md) | [中文](../../../docs/subsystems/workflow.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/workflow.md](../../../docs/subsystems/workflow.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

ワークフロー seam は、subagent を起動するモデル作成のオーケストレーションスクリプトを agent が実行できるようにする。[subagent](subagent.md) と同様に、これは**1 つの任意 capability** であり、agent ループの一部ではない。したがってその型と操作は [core.md](core.md) ではなくここにある。bash と同様に、コンテキストごとに 1 つのエンジン実装だけが `ctx.workflowEngine` を提供できる。名前付き provider レジストリは存在しない（2 つ目のエンジンは、並んで実行されるのではなく、プラグイン設定によって最初のエンジンを置き換える）。

Service Definition: [dsh-workflow](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/workflow/workflow)（`ctx.workflowEngine` と以下の語彙）。[dsh-workflow-ptc](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/workflow/workflow-ptc) は、呼び出し元 Session のファイルポリシーの下で、共有 Node PTC プロセス runtime を通じて VM とヘルパーを実行する。コンシューマーは [dsh-tool-workflow](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/workflow/tool-workflow) とオプトインの [dsh-tool-ralph](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/experimental/tool-ralph) である。[Workflow sandbox reuse](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-09-13-workflow-ptc-sandbox-reuse.md) は実行上の選択を所有し、[dynamic-workflows リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/workflow/workflow/README.md) はスクリプト意味論を所有する。

ソース: ブラウザセーフな語彙は [`packages/workflow/workflow/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/workflow/workflow/src/types.ts) に、Host リクエストとライブ実行用ハンドルは [`runtime-types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/workflow/workflow/src/runtime-types.ts) にある。

## 開始リクエスト

呼び出し元が実行を開始するときに要求するもの。通常の workflow ツールはモデルの `{ script, meta, args }` 呼び出しと呼び出し元 agent からこれを構築する。専門のコンシューマーは実行のためにエンジン全体の `subagentProvider` を 1 つ選択し、`maxTotalAgents` を下げることもできるが、スクリプトはどちらのポリシーも観察も置換もできない。`meta` と `args` は平文の JSON データである（エンジンは `meta` をそのスキーマに対して検証し、何かが実行される前に大声で拒否する — スクリプトテキストが評価されて取得されることは決してない）。`parent` は必須である — スクリプトが起動するすべての子はそれに帰属し、cwd、系譜、深さは [subagent seam](subagent.md) を通る。

```ts type-equiv
/**
 * What a caller asks for when starting a workflow run. `meta` and `args` are
 * plain JSON data by the seam contract. `parent` is required because every
 * `agent()` spawned by the script is attributed to that live Agent.
 */
interface WorkflowStartRequest {
  /** The plain-JS script body (top-level await allowed; ends with `return <json-value>`). */
  script: string
  /** The workflow's identity block, as plain JSON data (shape-validated by the engine). */
  meta: WorkflowMeta
  /** Optional input exposed verbatim to the script as the `args` global. */
  args?: unknown
  /** Optional engine-wide child-provider override for this run. */
  subagentProvider?: string
  /** Optional per-run total-child ceiling. */
  maxTotalAgents?: number
  /** The agent on whose behalf the run executes (parent of every child). */
  parent: Agent
  /** Cancels the run when aborted. */
  signal?: AbortSignal
}
```

## ワークフローの同一性: `WorkflowMeta`

開始リクエストのデータとして運ばれる同一性ブロック（ツールの `meta` パラメーター。フィールド語彙は Claude Code の dynamic-workflows メタブロックに一致する）。`phases` は進捗の語彙にすぎない: `phase()` 呼び出しはオブザーバー向けにタイトルを照合する。実行構造は含意されない。

```ts type-equiv
/**
 * The script's identity block, provided as plain JSON data alongside the
 * script body (the model-facing tool carries it as its `meta` parameter) and
 * validated by the engine before the body runs. `name`/`description` are
 * required; the rest is optional annotation. The field vocabulary matches the
 * Claude Code dynamic-workflows meta block.
 */
interface WorkflowMeta {
  /** Short kebab-case workflow name (display + persistence key). */
  name: string
  /** One-line description of what the workflow does. */
  description: string
  /** Optional guidance on when this workflow applies (shown in listings). */
  whenToUse?: string
  /** Optional phase declarations matched by `phase()` calls. */
  phases?: WorkflowPhase[]
}
```

## 終端結果: `WorkflowResult`

1 回の実行の結果であり、`WorkflowRun.result` によって解決される。`value` はスクリプトの実体化された戻り値 — 平文の host レルム JSON データ（スクリプトが何も返さなかった場合は `null`） — であり、`completed` の場合にのみ意味を持つ。`stopReason` は閉じた union である（エンジンが所有し、コンシューマーは網羅してよい）: `completed` | `cancelled` | `error`。`completed` 以外の理由は失敗を `error` に運び、コンシューマーは部分的な出力を成功として報告するのではなく、それを `isError` のツール結果にマッピングする。

```ts type-equiv
/**
 * The outcome resolved by a live workflow run. `value` is
 * the script's materialized return value (plain host-realm JSON data; `null`
 * when the script returned `undefined`) — meaningful only for `completed`.
 * A non-`completed` reason carries the failure in `error`; the consumer maps
 * it to an `isError` tool result rather than reporting partial output.
 */
interface WorkflowResult {
  /** The script's return value (host JSON data; `null` for no return). */
  value: unknown
  /** Why the run settled. */
  stopReason: WorkflowStopReason
  /** The failure message (present iff `stopReason` is not `completed`). */
  error?: string
  /**
   * How many `agent()` calls the run accepted over its whole lifetime. On a
   * graceful settlement this is the script-side count (calls still queued for
   * a concurrency slot included); on a termination path (cancellation or
   * process failure) it degrades to the host-observed count — calls queued
   * inside a terminated script are unknowable then.
   */
  agentsStarted: number
}
```

## ライブ実行: `WorkflowRun`

コンシューマーは `result` を待機し、実行中に `cancel` してよく、すべての経路で `dispose` しなければならない。`result` は決して拒否しない: スクリプトの失敗は `stopReason: 'error'` で解決し、キャンセルは `'cancelled'` で解決する。PTC エンジンには全体的な経過期限はない。キャンセルされると管理対象プロセスを直ちに中断する。破棄は provider 契約の下でプロセスと子のクリーンアップを待機し、独立したワークフローのクリーンアップ期限は持たない。

```ts type-equiv
/**
 * Holder-owned live workflow. `result` never rejects; consumers may cancel
 * and must call idempotent `dispose()` to await script and child quiescence.
 */
interface WorkflowRun {
  readonly id: WorkflowRunId
  /** The validated meta block available before the script body runs. */
  readonly meta: WorkflowMeta
  readonly result: Promise<WorkflowResult>
  /** Cancel the run and its children. */
  cancel(reason?: string): void
  /** Cancel if needed and await script and child cleanup. */
  dispose(): Promise<void>
}
```

## 失敗の規律: `WorkflowError.fatal`

スクリプト内での hook の誤用 — 不正な引数、未知または遅延された `agent()` オプション、[構造化出力サブセット](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/tools/README.md) の外側のスキーマ、上限の超過、seam の開始失敗、キャンセル — は `fatal: true` を持つ `WorkflowError` を投げる。`parallel()`/`pipeline()` コンビネーターは、項目を `null` にマッピングするのではなく、致命的エラーを再スローする: タイプミスのオプションはスクリプトを大声で終了させなければならず、通常の子の失敗として読めるものに溶け込んではならない。項目ごとの `null` は、子実行の失敗（`completed` 以外の停止理由）と通常のステージ内スクリプトエラー用に予約されている。

## イベント

`workflow/*` イベント（`workflow/start`、`workflow/phase`、`workflow/log`、`workflow/agent-start`、`workflow/agent-end`、`workflow/end` — [イベントカタログ](#cordis-surface) を参照）は、データスナップショットを運ぶ**観察専用**の emit である: すべてのペイロードは `WorkflowRunInfo`（id + meta）で始まり、ライブの `WorkflowRun` ではないため、サブスクライバーは `cancel`/`dispose` を得られない。また `workflow/end` は結果値を意図的に省略する（結果を観察するリスナーが呼び出し元の結果の可変エイリアスを受け取ってはならない）。すべての emit はリスナーごとに封じ込められる — スローするサブスクライバーはログに記録され、伝播せず、その後に登録されたリスナーを飢えさせない — そしてすべてのリスナーは自身のペイロードクローンを受け取るため、それを変更してもエンジンも他のリスナーも壊れない。この封じ込めは `subagent/start`/`subagent/end` を鏡写しにしている。

## 永続 Chat レコード

トップレベルの `dsh-tool-workflow` コンシューマーは、実行所有権を変えずに表示事実を呼び出し元の親 Session に投影する。実行が受け入れられた後に `tool-workflow/run-start` を書き、メンバーの開始と終了を `runId + seq` でペアにし、結果が判明し破棄が静止に達した後にのみ `tool-workflow/run-end` を書く。ネストされたトランスポート呼び出しはレコードを書かない。最初の追記失敗はその実行の後続の書き込みを無効化するため、ログは空または合法な連続プレフィックスのままであり、ツール結果は変わらない。

レコードプロトコルは、実行ごとに 1 つの開始、正の一意なメンバーシーケンス、ペアになったメンバー終了、開いたメンバーを持つ実行終了の不在、実行終了後の更新の不在を許す。ログテールでのメンバー終了または実行終了の欠落は、破損ではなく有効な中断の証拠である。

`dsh-client-ui-workflow-run` は 4 つのイベントを Conversation Node エンジンを通じて、元の workflow ツールノードの後に、run-start シーケンスにアンカーされた 1 つの `workflow-run` Chat ノードに折り畳む。フェーズグループは実際のメンバー開始からのみ来て、正確な文字列を保持する。省略されたフェーズと `''` の区別も含む。閉じた Locations は欠落した終端事実を中断された表示に変える。[UI パッケージ README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-workflow-run/README.md) は開示、ステータス、同一親内のローカルナビゲーション動作を所有する。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxworkflowengine--workflowengine-abstract-seam"></a>

### `ctx.workflowEngine` — `WorkflowEngine`（抽象 seam）

Workflow Service Definition の契約。無効なリクエストは公開前にスローする。ライブ実行はホルダー所有であり、その結果は決して拒否せず、破棄はスクリプトと子のクリーンアップを待つ。ライフサイクルリスナーの失敗は封じ込められ、`workflow/end` は結果の解決時にちょうど 1 回発火する。

```ts cordis-catalog
/**
 * Parse and execute a workflow script.
 * @param request - the script, its `args`, the parent agent, and an
 *   optional cancel signal.
 * @returns the live run; its `result` resolves when the script settles.
 */
abstract start(request: WorkflowStartRequest): WorkflowRun
```

ソース: [`packages/workflow/workflow/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/workflow/workflow/src/index.ts)

<a id="workflow-events"></a>

### `workflow/*` イベント

<a id="workflowagent-end--emit"></a>

#### `workflow/agent-end` — emit

1 つの `agent()` 呼び出しが解決した（クリーンな結果、子の失敗、または実行のキャンセル）。`agent.seq` によって `Events['workflow/agent-start']` とペアになり、すべての停止経路で開始された呼び出しごとにちょうど 1 回発生する — エンジンの終了経路では、終了は outcome `'cancelled'` でエンジンが合成する。

```ts cordis-catalog
/**
 * One `agent()` call settled (clean result, child failure, or run
 * cancellation). Paired with {@link Events['workflow/agent-start']} by
 * `agent.seq`, exactly once per started call on every stop path — on an
 * engine termination path the end is
 * engine-synthesized with outcome `'cancelled'`.
 * @param info - the run's identity snapshot.
 * @param agent - the call identity plus its outcome.
 * @mode emit
 */
'workflow/agent-end'(info: WorkflowRunInfo, agent: WorkflowAgentEndInfo): void
```

ソース: [`packages/workflow/workflow/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/workflow/workflow/src/index.ts)

<a id="workflowagent-start--emit"></a>

#### `workflow/agent-start` — emit

1 つの `agent()` 呼び出しが公開済みの子実行を確立した。`agent.seq` によって `Events['workflow/agent-end']` とペアになる。provider から公開済みの実行を決して受け取らない呼び出しは、このペアのどちらのイベントも emit しない。

```ts cordis-catalog
/**
 * One `agent()` call established a published child run. Paired with
 * {@link Events['workflow/agent-end']} by `agent.seq`. A call that never
 * receives a published run from the provider emits neither
 * event in this pair.
 * @param info - the run's identity snapshot.
 * @param agent - the call's sequence number, label, phase, and child id.
 * @mode emit
 */
'workflow/agent-start'(info: WorkflowRunInfo, agent: WorkflowAgentInfo): void
```

ソース: [`packages/workflow/workflow/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/workflow/workflow/src/index.ts)

<a id="workflowend--emit"></a>

#### `workflow/end` — emit

ワークフロー実行が解決した（任意の停止理由）。`WorkflowRun.result` が解決したときに発火する。`Events['workflow/start']` とペアになる。

```ts cordis-catalog
/**
 * A workflow run settled (any stop reason). Fired when
 * {@link WorkflowRun.result} resolves. Paired with
 * {@link Events['workflow/start']}.
 * @param info - the run's identity snapshot.
 * @param result - the outcome data (stop reason, error, agent count) —
 *   deliberately WITHOUT the result value (see {@link WorkflowResultInfo}).
 * @mode emit
 */
'workflow/end'(info: WorkflowRunInfo, result: WorkflowResultInfo): void
```

ソース: [`packages/workflow/workflow/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/workflow/workflow/src/index.ts)

<a id="workflowlog--emit"></a>

#### `workflow/log` — emit

スクリプトがナレーション行（`log(message)` 呼び出し）を emit した。

```ts cordis-catalog
/**
 * The script emitted a narration line (a `log(message)` call).
 * @param info - the run's identity snapshot.
 * @param message - the logged message, verbatim.
 * @mode emit
 */
'workflow/log'(info: WorkflowRunInfo, message: string): void
```

ソース: [`packages/workflow/workflow/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/workflow/workflow/src/index.ts)

<a id="workflowphase--emit"></a>

#### `workflow/phase` — emit

スクリプトがフェーズ（`phase(title)` 呼び出し）に入った — オブザーバー向けの進捗グループ化であり、実行意味論はない。

```ts cordis-catalog
/**
 * The script entered a phase (a `phase(title)` call) — progress grouping
 * for observers; no execution semantics.
 * @param info - the run's identity snapshot.
 * @param title - the phase title, verbatim.
 * @mode emit
 */
'workflow/phase'(info: WorkflowRunInfo, title: string): void
```

ソース: [`packages/workflow/workflow/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/workflow/workflow/src/index.ts)

<a id="workflowstart--emit"></a>

#### `workflow/start` — emit

ワークフロー実行が開始した — スクリプトのメタブロックが検証され、本体がまさに実行されようとしている。`Events['workflow/end']` とペアになる。

```ts cordis-catalog
/**
 * A workflow run started — the script's meta block validated, the body
 * about to execute. Paired with {@link Events['workflow/end']}.
 * @param info - the run's identity snapshot (id + meta).
 * @mode emit
 */
'workflow/start'(info: WorkflowRunInfo): void
```

ソース: [`packages/workflow/workflow/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/workflow/workflow/src/index.ts)
<!-- END GENERATED cordis-surface -->
