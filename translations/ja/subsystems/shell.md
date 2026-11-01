# シェル・エグゼキューター

日本語 | [English](../../../docs/subsystems/shell.md) | [中文](../../../docs/subsystems/shell.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/shell.md](../../../docs/subsystems/shell.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

シェル実行 seam は [dsh-shell](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/shell/shell) を `ctx.shell` 上の Service Definition として使う。[shell パッケージ・グループ](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/shell/README.md) はその Bash および PowerShell プロバイダーとモデル向けコンシューマーを列挙する。汎用のバックグラウンド・ジョブ id、所有権、制御は [jobs.md](jobs.md) にある。この seam はジョブ登録なしにプロセス・ハンドルを返す。管理レンジの機構は [subprocess seam](subprocess.md) の背後にある。

ソース: [`packages/shell/shell/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/shell/shell/src/types.ts)

## 管理シェル環境名前空間

`DSH_*` 変数は Harness 所有の子プロセス・ファクトである。モデル向け bash ツールはそれらを `ctx.shellEnv` を通じて収集し、`ShellExecRequest.dshEnv` を通じて渡す。サブプロセス・サービスは現在のスナップショットをマージする前に継承された `DSH_*` 名を削除する。`DshEnvironmentKey`/`DshEnvironment` の語彙は [subprocess seam](subprocess.md) が所有し、`dsh-shell` が再エクスポートする。

## リクエスト対 spec: `resolve()` の分割

この seam は、**モデル／プラグイン向けリクエスト**（任意の `workdir`／`timeoutMs`／`stdoutMaxBytes` を持ち、設定またはリクエスト・ポリシーから埋められる）と、エグゼキューターが作用する**完全に解決済みの spec**（これらのフィールドが必須）を分離する。ツール層はその間で `ctx.shell.resolve(request)` を呼ぶ（リポジトリの「パッケージ境界では暗黙より明示」ルール）。`ShellExecSpec` は解決済みの値を運ぶ。

```ts type-equiv
/**
 * A caller's execution REQUEST: `workdir` and `timeoutMs` are optional and
 * filled by {@link ShellExecutor.resolve} from the implementation's config.
 * This is the model-/plugin-facing shape; pass it to `resolve()` to obtain a
 * fully-resolved {@link ShellExecSpec}.
 */
interface ShellExecRequest {
  command: string
  /** Working directory override (default: implementation-configured). */
  workdir?: string | undefined
  /** Timeout override in milliseconds (implementations cap it). */
  timeoutMs?: number | undefined
  /** Deadline policy at `timeoutMs` expiry (default `'kill'`). */
  onExpiry?: ShellExpiryPolicy | undefined
  /**
   * Foreground stdout capture budget in bytes. Absent uses the executor's
   * default output cap. Trusted in-process consumers use this when they must
   * parse complete stdout up to their own bounded limit; the model-facing bash
   * tool does not expose it as a parameter.
   */
  stdoutMaxBytes?: number | undefined
  /** Abort signal — implementations kill the command when it fires, and treat a signal that is already aborted as fired. */
  signal?: AbortSignal | undefined
  /**
   * Bytes to write to the command's stdin, then close it. Absent leaves stdin
   * closed/empty (the default for model-driven tool calls). Set by in-process
   * plugins (e.g. the hooks bridges, which write a hook command's JSON payload
   * to its stdin); the model-facing bash tool does not expose it as a parameter
   * (a model that needs stdin uses shell syntax like a heredoc or a pipe).
   */
  stdin?: string | undefined
  /**
   * Ordinary environment entries for the command, merged after the credential
   * scrub. Managed facts belong in {@link dshEnv}, which merges after this
   * map, so an entry here can never displace one. Set by in-process plugins
   * (the hooks bridges set `CLAUDE_PROJECT_DIR`, `CLAUDE_PLUGIN_ROOT`, …); the
   * model-facing bash tool does not expose it as a parameter.
   */
  env?: Record<string, string> | undefined
  /**
   * Harness-owned `DSH_*` variables for this execution (typed to managed
   * keys). Executors discard ambient `DSH_*` entries before merging this
   * snapshot last, so an unavailable current fact cannot inherit a stale
   * value from the harness process and a caller {@link env} entry cannot
   * displace a managed one.
   */
  dshEnv?: DshEnvironment | undefined
  /** Fully resolved per-call sandbox policy; sandboxing executors default it. */
  sandboxPolicy?: SandboxExecutionPolicy | undefined
}
```

```ts type-equiv
/**
 * A resolved execution spec. {@link ShellExecutor.resolve} fills and caps the
 * required fields; under `onExpiry: 'none'` the resolved `timeoutMs` arms no
 * timer and is only echoed into {@link ShellRunResult.timeoutMs}.
 */
interface ShellExecSpec {
  command: string
  workdir: string
  timeoutMs: number
  /** Deadline policy at `timeoutMs` expiry ({@link ShellExecutor.resolve} defaults it to `'kill'`). */
  onExpiry: ShellExpiryPolicy
  /**
   * Resolved stdout capture budget in bytes, applied to every execution's
   * stdout; stderr keeps the executor's own output cap.
   */
  stdoutMaxBytes: number
  /** Abort signal — implementations kill the command when it fires, and treat a signal that is already aborted as fired. */
  signal?: AbortSignal | undefined
  /** Bytes to write to stdin before closing it; absent means no stdin. */
  stdin?: string | undefined
  /**
   * Ordinary environment entries carried through from
   * {@link ShellExecRequest.env}; {@link dshEnv} still merges after them.
   * OPTIONAL on the spec for the same reason as `stdin`: absent means no
   * ordinary extra environment.
   */
  env?: Record<string, string> | undefined
  /** Managed `DSH_*` snapshot (typed to managed keys); merges after {@link env}. */
  dshEnv?: DshEnvironment | undefined
  /** Resolved sandbox policy; ignored by executors that do not confine. */
  sandboxPolicy: SandboxExecutionPolicy | undefined
}
```

`stdin` と `env` は信頼されたプロセス内プラグイン入力であり、`dsh-tool-bash` によって公開されない。ローカル・エグゼキューターは、呼び出し元が明示的に供給した env をマージする前に周囲のクレデンシャルをスクラブする。

`stdoutMaxBytes` も信頼されたプラグイン専用である。これによりフォアグラウンドのコンシューマーは、stderr、バックグラウンド・ジョブ、モデル向け bash ツールの通常の出力上限を変えることなく、境界付きのパーサー予算まで完全な stdout を要求できる。

## フォアグラウンド実行: `ShellRunResult`

完了した（または終了させられた）1 回のフォアグラウンド実行の結果。直交する結果は**独立に**報告される。プロセスはシグナルを捕まえたためにタイムアウトしつつ exit 0 することもある。したがって `timedOut`、`aborted`、`signal`、`exitCode` はそれぞれ独立したフィールドであり、呼び出し元が途中で切り詰められた実行をクリーンな成功として読むことはない。

```ts type-equiv
/** The outcome of a foreground run, including timeout during preparation. */
interface ShellRunResult {
  /** Exit code; null when preparation expired or the process died from a signal. */
  exitCode: number | null
  /** Terminating signal, or null when none was reported, including preparation expiry. */
  signal: NodeJS.Signals | null
  /**
   * True when the executor's own timeout was the FIRST cause to cut the command
   * short. Mutually exclusive with {@link aborted}: one fused deadline drives
   * both the timeout and the caller's cancellation, so a timeout and an abort
   * racing before process close report the single first-abort cause, not both
   * (see the [timeout-library Agent Note](../../../../.agents/notes/implemented/architecture/2026-07-06-timeout-deadline-library.md)).
   */
  timedOut: boolean
  /**
   * True when the caller's `AbortSignal` was the FIRST cause to kill the command
   * (and it was not the executor's own timeout). Mutually exclusive with
   * {@link timedOut} — see there for the first-cause classification.
   */
  aborted: boolean
  /** The effective timeout applied to this run (after defaulting/capping). */
  timeoutMs: number
  stdout: CollectedOutput
  stderr: CollectedOutput
  /** Sandbox execution facts, absent for an unsandboxed executor. */
  sandbox?: ShellSandboxInfo
}
```

各ストリームは `CollectedOutput` である。切り詰められた可能性のあるテキストと回復情報であり、切り詰められた場合、`text` は**末尾**で、完全なストリームはプライベート・ファイルにスピルされる。フィールドは [subprocess seam](subprocess.md) が所有し、`dsh-shell` が再エクスポートする。

## ファイル・サンドボックス: `ShellSandboxInfo`

サンドボックスを消費するエグゼキューターは、設定済みのモード・フォールバックを `ShellExecutor.sandboxMode` を通じて公開する。ツール層は [`@deepseek-ai/dsh-sandbox-policy`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/sandbox/sandbox-policy/README.md) に依頼して、各呼び出し Session の永続的な `sandbox/mode` オーバーライドと不変の cwd を `ShellExecRequest.sandboxPolicy` に解決する。ユーザーが承認した厳密に広い呼び出しはモードだけを置換する。モード／ルート／エンフォースメントの語彙は [`@deepseek-ai/dsh-sandbox` seam](sandbox.md) が所有し、モードはファイル効果だけを左右する。

サンドボックス化された実行は、そのモード、保守的な拒否分類、エンフォースメントの完全性を報告する。`runnerFailed` はコマンドが実行される前のサンドボックス・ランナー失敗を示す。フォアグラウンド実行は `SANDBOX_UNAVAILABLE` をスローし、決着済みのバックグラウンド・プロセスはファクト・チャネルだけを持つ。

```ts type-equiv
/**
 * Sandbox facts for one run, present iff a sandboxing executor handled it.
 * Facts are reported independently of process exit status so callers can
 * distinguish command failures from policy denials and runner failures.
 */
interface ShellSandboxInfo {
  /** The mode the command actually ran under. */
  mode: SandboxMode
  /** Whether the sandbox denied a file operation. */
  denied: boolean
  /** How completely the selected runner enforced the requested mode. */
  enforcement?: SandboxEnforcement
  /** Whether the sandbox runner failed before the command could run. */
  runnerFailed?: boolean
}
```

`SANDBOX_UNAVAILABLE` エラー・コード（[sandbox seam](sandbox.md) が所有）は、隔離モードに使用可能なバックエンドがないときに `ctx.sandbox` プロバイダーがスローし、エグゼキューターが伝播するものである。選択されたランナーがプロファイルを拒否する場合も、同じフェイルクローズのフォアグラウンド・エラーに到達する。決着済みのバックグラウンド・ジョブは `runnerFailed` を記録する。モデルは結果の中で拒否／ランナーのファクトを受け取り、有効なモードは拒否マーカーがそれを名指したときにのみ知り、`sandbox_permissions` と `justification` を通じて一回限りの厳密に広い再試行を要求できる。何かが実行される前に、`ctx.approval` がその正確な呼び出しを承認しなければならない。完全なポリシーと切り替えの設計は [サンドボックス・リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/sandbox/sandbox-policy/README.md) である。

## バックグラウンド・プロセス: `ShellProcess`

`start()` は非同期の起動準備の後にハンドルで解決する。キャンセルまたは準備の失敗は公開前に拒否する。ハンドルは id も所有者も持たない。`dsh-tool-bash` はそれを `ctx.jobs.start()` フックへ適応させ、その後は汎用ランタイムがジョブの同一性とライフサイクルを所有する。`done` は基盤プロセスが決着したときに解決し、決して拒否しない。サブプロセス・プロバイダーの拒否は、stderr にステージ中立のエラーを持つ `killed` プロセスになる。読み取りは決着後も有効で、サンドボックス・ファクトは `done` が解決する前に刻印される。

```ts type-equiv
/**
 * A background process handle returned by {@link ShellExecutor.start}. It is the
 * only access path; buffered output remains readable after exit. Composition
 * teardown (the subprocess service's disposal) kills running processes and
 * awaits {@link done}; an executor-only reload leaves them running.
 */
interface ShellProcess {
  /** Process lifecycle state (settled exactly once). */
  status: ShellProcessStatus
  /** Exit code once finished (null = killed by signal / still running). */
  exitCode: number | null
  /** Terminating signal name, when signal-killed. */
  signal: NodeJS.Signals | null
  /**
   * Resolves when the underlying process settles (never rejects — provider
   * rejection settles as `killed` with a stage-neutral error on stderr).
   */
  readonly done: Promise<void>
  /** Sandbox facts, stamped once a confined process settles. */
  sandbox?: ShellSandboxInfo
  /**
   * Read output produced since the previous read (consuming — consecutive
   * reads never re-deliver). Reads that lost data flag `lossy` and point at
   * full-stream spill files when available.
   */
  readOutput(): ShellProcessRead
  /**
   * Non-consuming offset readers over the same captured streams the consuming
   * {@link readOutput} cursor drains, including the provider-failure note a
   * rejected spawn leaves on stderr. Independent observers read here at their
   * own offsets without stealing bytes from `readOutput`.
   */
  observed: ShellObservedStreams
  /**
   * Terminate the provider-managed range. Returns false when it had already finished
   * (no-op); idempotent.
   */
  kill(): boolean
}
```

`readOutput()` は増分デルタとスピル回復のファクトを返す:

```ts type-equiv
/** One incremental {@link ShellProcess.readOutput} read. */
interface ShellProcessRead {
  /** Output produced since the previous read (stderr in a marked section). */
  delta: string
  /** True when truncation dropped unread bytes the delta cannot include. */
  lossy: boolean
  /** Full stdout spill file, when stdout truncation occurred and a safe path is available. */
  stdoutSpillPath?: string
  /** Full stderr spill file, when stderr truncation occurred and a safe path is available. */
  stderrSpillPath?: string
}
```

## サービス

`ShellExecutor` は `resolve`、フォアグラウンドの `run`、バックグラウンド・プロセスの `start`、および `sandboxMode` ケイパビリティ・ファクトを所有する。`dsh-bash-local` はコマンドの既定化、タイムアウト／abort の分類、ターミナル環境、バックグラウンド読み取りのマージを所有する。管理レンジの終了、境界付きコレクター、スピル・ファイル、クレデンシャル・スクラブ、破棄時の静止は [subprocess サービス](subprocess.md) のものである。`dsh-tool-bash` はモデル向けの描画を所有し、バックグラウンド・ハンドルを[汎用ジョブ・ランタイム](jobs.md)へ適応させる。`dsh-shell` はシェル・ツールの共有終了ステータス契約を所有する。エクスポートされる `parseExitStatus`/`ParsedExitStatus` は、`dsh-tool-bash` の `renderResult` と `dsh-tool-pwsh` の `renderPwshResult` が追加する `[exit code: N]` / `[killed by signal: X]` マーカーを反転し、両ツールの `presentResult` がそれを使って、描画済みテキストをターミナル・カードの出力本文とその exit-status ピルに分割する。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxshell--shellexecutor-abstract-seam"></a>

### `ctx.shell` — `ShellExecutor`（抽象 seam）

抽象 bash 実行サービス。サブクラス化し、抽象メソッドを実装して、そのサブクラスを plugin としてロードする。`ctx.shell` として登録される（コンテキストごとに 1 実装。2 つ目をロードするとスローする。これは cordis 標準の重複サービス挙動である）。

execute は準備後にプロセス・ハンドルで解決する。「フォアグラウンド」は spawn ではなく、呼び出し元が何を await するかの性質である。ShellExecution.result を await する呼び出し元はコマンドをフォアグラウンドで実行したのであり、ハンドルを保持する呼び出し元はバックグラウンドで実行したのである。しばらくだけ待つ呼び出し元は `onExpiry: 'none'` の下でコマンドを実行し、自身の待機を制限する。呼び出し元が待機をやめた後もハンドルは有効なままである。

実装は次のセマンティクスを守らなければならない。

- ShellExecution.result はインフラストラクチャの失敗に対してのみ拒否する。非ゼロ終了、タイムアウトによる終了、abort による終了は、説明的な結果で解決する。第一原因の `timedOut`/`aborted`、および spec の `timeoutMs` のエコーである。
- ハンドルは準備後に公開される。`done` はプロセスの終了時に決着し、決して拒否しない。spawn の失敗は読み取りパス上のエラーを伴う `killed` として決着し、`result()` は同じ失敗を自身の拒否として運ぶ。
- `onExpiry: 'none'` は期限をアームしない。`'kill'` は期限到来時に終了させる。準備中の期限到来は、出力のない決着済みタイムアウト・ハンドルを返す。
- ShellProcess.readOutput は増分型である。連続する読み取りが出力を繰り返すことはない。ロッシーな読み取りは切り詰めと利用可能なスピル・ファイルを報告する。
- まだ実行中のプロセスは、所有するコンポジションの teardown 時に停止され、await される。subprocess seam との境界ではそれが `ctx.subprocess` の破棄であるため、エグゼキューターだけのリロードではプロセスが生き残る。

```ts cordis-catalog
/**
 * Apply implementation-owned defaults and caps to a request before execution.
 * @param request - the caller's request; omitted fields get this
 *   implementation's defaults, capped fields are clamped.
 * @returns the fully-specified spec to hand to {@link execute}.
 */
abstract resolve(request: ShellExecRequest): ShellExecSpec

/**
 * Prepare and spawn the command under its resolved deadline.
 * @param spec - a resolved spec from {@link resolve}, never a raw request.
 * @returns the prepared handle, including its result projection;
 *   preparation timeout yields an already-settled handle with no output.
 * @throws on preparation failure or caller cancellation before process publication.
 */
abstract execute(spec: ShellExecSpec): Promise<ShellExecution>
```

ソース: [`packages/shell/shell/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/shell/shell/src/index.ts)

<a id="ctxshellenv--shellenvregistry"></a>

### `ctx.shellEnv` — `ShellEnvRegistry`

信頼された実行ごとの `DSH_*` 変数のためのレジストリ（`ctx.shellEnv`）。名前空間はモデルのシェル呼び出しごとに再構築される。エグゼキューターが周囲の `DSH_*` 値を破棄し、その後レジストリの現在のスナップショットが注入される。組み込みのシェル・ファクトはレジストリ自身が所有したままで、プラグインはエフェクト・スコープの破棄を伴う追加の列挙可能なファクトを登録できる。

```ts cordis-catalog
/**
 * Register one environment contributor. Names and keys are unique; built-in
 * keys are reserved. Registration is disposed with the calling plugin fiber.
 * @param contributor - declared key ownership and per-execution resolver.
 * @returns the disposer that unregisters the contribution.
 */
register(contributor: BashEnvContributor): () => void

/**
 * Build the trusted `DSH_*` snapshot for one shell tool execution.
 * @param execution - the current tool execution.
 * @returns an immutable environment overlay containing built-ins and current contributions.
 */
collect(execution: ToolExecution): DshEnvironment

/**
 * Enumerate plugin-contributed variables without executing their resolvers.
 * @returns declarations sorted by environment variable name.
 */
list(): BashEnvVariableInfo[]
```

型: [DshEnvironment](subprocess.md) · [ToolExecution](tools.md)

ソース: [`packages/shell/shell-env/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/shell/shell-env/src/index.ts)
<!-- END GENERATED cordis-surface -->
