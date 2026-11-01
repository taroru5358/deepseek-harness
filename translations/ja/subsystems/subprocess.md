# サブプロセス

日本語 | [English](../../../docs/subsystems/subprocess.md) | [中文](../../../docs/subsystems/subprocess.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/subprocess.md](../../../docs/subsystems/subprocess.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

サブプロセス seam は Service Definition（[dsh-subprocess](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/subprocess/subprocess)、`ctx.subprocess`）と Service Provider（[dsh-subprocess-local](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/subprocess/subprocess-local)）に分かれている。そのコンシューマーは他のケイパビリティ seam とプロセス外バックエンドである。[bash エグゼキューター・ファミリー](shell.md) は収集済みバッチ出力を使い、LSP は生のプロトコル・パイプを使い、PTY バックエンドはターミナル・プリミティブを使い、ACP サブエージェント・バックエンドはパイプされた ndjson と継承された stderr を使う。この seam は管理された `DSH_*` 環境名前空間、共有のクレデンシャル・スクラブ（`scrubbedParentEnv`）、および `CollectedOutput` 形状を所有する。[dsh-shell](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/shell/shell) は bash コンシューマーが 1 つのインポート・ルートを保てるよう、その語彙を再エクスポートする。

ソース: [`packages/subprocess/subprocess/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/subprocess/subprocess/src/types.ts) および [`packages/subprocess/subprocess/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/subprocess/subprocess/src/index.ts)

## 実行ファイルのルックアップ

1 つのプロバイダーの spawn 作業ディレクトリ、実行ファイル・パス、通常のプロセス、ターミナル Session は、マウントされたファイルシステム・プロバイダーと同じパスおよびプロセスの名前空間に属する。`resolveExecutable(command, env?, signal?)` は絶対実行ファイル・パスを検証するか、プロバイダーのスクラブ済み `PATH` と意図的なオーバーライドを通じてベア名を解決する。

## 管理環境名前空間とキャプチャ出力

`DSH_*` 変数は Harness 所有の子プロセス・ファクトである。実装は呼び出し元の明示的な `env` がマージされる前に周囲の `DSH_*` 名を破棄するため、現在のファクトは意図的な文字列エントリとしてのみ届く。一方、明示的な `undefined` の墓標は通常の周囲の値を削除する。収集された各ストリームは、その切り詰めとスピル回復の状態を `CollectedOutput` を通じて報告する。

```ts type-equiv
/** One environment key inside the managed {@link DSH_ENV_PREFIX} namespace. */
type DshEnvironmentKey = `${typeof DSH_ENV_PREFIX}${string}`
```

```ts type-equiv
/** Trusted DeepSeek Harness variables for one child-process execution. */
type DshEnvironment = Readonly<Record<DshEnvironmentKey, string>>
```

```ts type-equiv
/** One captured stream: the (possibly truncated) text plus recovery info. */
interface CollectedOutput {
  /** Collected text — the TAIL of the stream when truncated. */
  text: string
  /** True when bytes were dropped from `text`. */
  truncated: boolean
  /** Path to a file holding the COMPLETE stream, when truncated and available. */
  spillPath?: string
}
```

## Node 形式の stdio 配置

各ストリームの配置は明示的で、コンシューマーごとに選ばれる。プロトコル・フレーミング（LSP JSON-RPC、ACP ndjson）には生のパイプ、パススルー診断には inherit、境界付きバッチ出力には collect モードを使い、スピル・ファイルは任意である。したがって診断の末尾（言語サーバーの stderr）は、ファイルを残さずにバッファリングされる。

```ts type-equiv
/**
 * stdin disposition. `'ignore'` leaves fd 0 on `/dev/null`; `'pipe'` exposes
 * {@link SubprocessHandle.stdin} for the caller's ongoing protocol writes;
 * `{ data }` writes the bytes and closes (the batch shape).
 */
type SubprocessStdinMode = 'ignore' | 'pipe' | { readonly data: string }
```

```ts type-equiv
/**
 * Bounded in-memory collection for one output stream, with an optional
 * full-stream spill file. Omitting `spill` keeps only the in-memory tail —
 * the diagnostic-tail shape (a language server's stderr); including it makes
 * the complete stream recoverable up to its cap (the bash tool shape).
 */
interface SubprocessCollect {
  /** In-memory cap in bytes; overflow keeps the TAIL. */
  maxBytes: number
  /** Full-stream spill file; absent disables spilling entirely. */
  spill?: {
    /** Whole-stream byte cap; a larger stream discards its now-incomplete spill. */
    maxBytes: number
  }
}
```

```ts type-equiv
/**
 * stdout/stderr disposition. `'pipe'` exposes the raw `Readable` for the
 * caller's protocol decoding; `'inherit'` passes the parent's descriptor
 * through (child diagnostics land on the harness's own stream); a
 * {@link SubprocessCollect} object buffers boundedly with offset-based reads.
 */
type SubprocessOutputMode = 'pipe' | 'inherit' | SubprocessCollect
```

```ts type-equiv
/** Per-stream stdio dispositions, all explicit — this seam applies no defaults. */
interface SubprocessStdio {
  stdin: SubprocessStdinMode
  stdout: SubprocessOutputMode
  stderr: SubprocessOutputMode
  /** Request a separate byte-mode duplex channel; omission creates none. */
  control?: 'pipe'
}
```

## 完全に明示的な spawn 仕様

この seam はデフォルトを一切適用しない。すべての配置、制限、ディレクトリが spec 上で明示されるため、隠れたサブプロセス・サービスのデフォルトではなく、呼び出し元自身の設定がそれらを決める。`argv` がシェル解釈されることは決してない。

```ts type-equiv
/**
 * A fully-specified spawn request. This seam applies no defaults: every
 * disposition, limit, and directory is explicit, so the caller's own config —
 * not a hidden subprocess-service default — decides them (the `dsh-shell`
 * request/spec split is the owning template).
 */
interface SubprocessSpawnSpec {
  /** Executable and arguments; `argv[0]` is the program. Never shell-interpreted here. */
  argv: readonly string[]
  /** Working directory for the child. */
  cwd: string
  /** Per-stream stdio dispositions. */
  stdio: SubprocessStdio
  /**
   * Positive finite grace period in milliseconds, no greater than
   * `MAX_TIMER_DELAY_MS`, available to the provider's termination procedure
   * and used for draining still-open collected pipes after the process exits
   * (an inherited descriptor held by a survivor cannot hold the outcome open
   * indefinitely). Providers document whether range termination is staged or
   * immediate.
   */
  graceMs: number
  /**
   * Abort signal — starts the terminate escalation on the managed range when
   * it fires. The caller owns deadlines and cause classification; this seam
   * only reacts to the abort.
   */
  signal?: AbortSignal | undefined
  /**
   * Explicit environment entries merged onto the implementation's scrubbed
   * parent base (see `scrubbedParentEnv`), with no namespace validation. A
   * string is a deliberate caller opt-in, so a forwarded credential-shaped
   * entry or current `DSH_*` fact survives the scrub; `undefined` is a
   * tombstone that removes an ordinary ambient entry from the child.
   */
  env?: NodeJS.ProcessEnv | undefined
}
```

## ハンドル: ストリーム、リーダー、管理レンジの終了

spawn はライブ・ハンドルを同期的に返し、対象と管理レンジの同一性はプロバイダー・プライベートのままである。collect モードのリーダーはストリーム全体のバイト・オフセットを取り、決して消費しないため、独立したリーダーが互いの差分を奪うことはない。パイプされたストリームは呼び出し元に属する。`terminate()` はプロバイダーが文書化した手順を開始し、`waitForExit()` は同じプロバイダー管理レンジを観測する。段階的なプロバイダーは `graceMs` を使ってよく、即時プロバイダーは遅延しない。コンシューマーはこれら 2 つの操作の上に独自の teardown ラダーを構築できる（ACP バックエンドの stdin-EOF 優先の `disposeAcpChild` がテンプレートである）。

```ts type-equiv
/**
 * A live subprocess and its provider-managed process range. Collected output
 * remains readable after exit; piped streams belong to the caller.
 *
 * Termination and {@link SubprocessHandle.waitForExit} use the same managed
 * range. Each provider documents the range it can observe and its signalling
 * and observation limits.
 */
interface SubprocessHandle {
  /** The child's stdin, present iff spawned with `stdin: 'pipe'`. */
  readonly stdin: Writable | undefined
  /** The child's raw stdout, present iff spawned with `stdout: 'pipe'`. */
  readonly stdout: Readable | undefined
  /** The child's raw stderr, present iff spawned with `stderr: 'pipe'`. */
  readonly stderr: Readable | undefined
  /** Separate caller-owned byte channel when requested; native startup failure may leave it absent. */
  readonly control: Duplex | undefined
  /** Offset-based readers for collect-mode streams (also readable after exit). */
  readonly collected: SubprocessCollectedOutputs
  /** Resolves with spawned-command exit facts; rejects for spawn or provider failures. */
  readonly done: Promise<SubprocessOutcome>
  /**
   * Begin the provider's documented termination procedure on the managed range
   * — the seam's only termination verb. Idempotent, a no-op once that range is
   * gone, and also triggered by the spec's abort signal.
   */
  terminate(): void
  /**
   * Wait until the same managed range is empty — not just until the spawned
   * command reports its outcome, so surviving work remains observable.
   * @param signal - optional bound for the wait.
   * @returns `true` when the managed range is empty, `false` when the signal aborted first.
   * @throws when the selected provider can no longer observe its managed range.
   */
  waitForExit(signal?: AbortSignal): Promise<boolean>
}
```

```ts type-equiv
/**
 * Cursor-free incremental access to one collected output stream. Offsets are
 * whole-stream byte coordinates owned by the caller, so independent readers
 * cannot consume one another's output; `readFrom(0)` after settlement is the
 * batch result (`lossy` then means the in-memory tail lost its head — the
 * {@link CollectedOutput.truncated} fact).
 */
interface SubprocessOutputReader {
  /**
   * Read everything captured since `fromByte`. When that offset has slid out
   * of the in-memory tail window the read is `lossy` — it returns the whole
   * retained tail and the gap is only recoverable from the spill file.
   * @param fromByte - whole-stream offset to resume from (a prior read's `nextOffset`; 0 for the first read).
   * @returns the delta text, the next offset, the `lossy` flag, and the spill path when one exists.
   */
  readFrom(fromByte: number): SubprocessOutputRead
}
```

```ts type-equiv
/** One incremental {@link SubprocessOutputReader.readFrom} read. */
interface SubprocessOutputRead {
  /** Stream text from the requested offset (the whole retained tail when lossy). */
  text: string
  /** Whole-stream offset to resume from on the next read. */
  nextOffset: number
  /** True when the requested offset slid out of the in-memory tail window. */
  lossy: boolean
  /** Path to the full-stream spill file, when one was created and remains intact. */
  spillPath?: string
}
```

```ts type-equiv
/** Offset-based readers for the streams spawned in collect mode. */
interface SubprocessCollectedOutputs {
  /** Present iff stdout is a {@link SubprocessCollect}. */
  readonly stdout?: SubprocessOutputReader
  /** Present iff stderr is a {@link SubprocessCollect}. */
  readonly stderr?: SubprocessOutputReader
}
```


## 結果は終了ファクトのみを運ぶ

`done` は Node の close イベント語彙を報告し、原因分類は行わない。サービスは abort で終了させるが、その理由を決めることはない（呼び出し元が自身の所有する期限シグナルを読む。たとえば bash エグゼキューターの `timedOut`/`aborted` の分割）。収集済み出力は決着後も `handle.collected` を通じて読めるため、バッチとストリーミングの呼び出し元は 1 つのアクセス・パスを共有する。

```ts type-equiv
/**
 * Exit facts of one closed process — Node's `close`-event vocabulary.
 * Deliberately carries NO timeout or cancellation classification (the caller
 * reads the signal it owns to classify causes) and NO output: collected
 * streams stay readable through {@link SubprocessHandle.collected} after
 * settlement, so batch and streaming callers share one access path.
 */
interface SubprocessOutcome {
  /** Exit code; null when the process died from a signal. */
  exitCode: number | null
  /** Terminating signal (e.g. 'SIGTERM'); null on normal exit. */
  signal: NodeJS.Signals | null
}
```

## ターミナル・プロセス・プリミティブ

`spawnTerminal(spec)` はパイプを使わないプロセス・プリミティブである。プロバイダーは制御端末を割り当て、UTF-8 テキスト転送、フォアグラウンド・プロセス・グループの検査とシグナリング、そしてプロバイダーがまだ観測できるすべての Session メンバーが静止するまで到達する 1 回の await される TERM→KILL 操作を所有する。プロバイダーは基盤固有の観測限界を文書化する。PTY バックエンドは引き続き、プロンプト検出、準備完了の推論、スクロールバック、サンドボックス・ポリシー、永続 Session の所有を担う。通常の `spawn()` では制御端末のセマンティクスを再構築できない。

ターミナル spec は argv、cwd、環境オーバーライド、ターミナル・タイプ、次元、クリーンアップ・グレース、および任意の割り当てキャンセルとシェル活動観測を完全に指定する。そのハンドルは `pid`、順序付き出力、`done`、`write`、`resize`、`inspectForeground`、`inspectActivity`、`signalForeground`、await される `terminate` を公開する。[`SubprocessTerminalSpawnSpec` と `SubprocessTerminalHandle`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/subprocess/subprocess/src/types.ts) がこれらのフィールドと操作を定義する。`resize(cols, rows)` はライブ PTY の次元を更新し、プロセス終了後は拒否する。

`inspectActivity()` は `SubprocessTerminalActivity` を返す。`state` は `idle`、`busy`、`unknown` のいずれかで、`revision` はプロバイダーが観測した活動または入力に伴って変化する。ターミナル・リクエストは `shellActivity` でサポートされるシェル・ライフサイクル観測をオプトインする。プロバイダー固有のサポートと保守的な unknown ケースは [subprocess-local](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/subprocess/subprocess-local/README.md#running-terminal-sessions) が文書化している。

`terminalEnvironment(signal?)` は `SubprocessTerminalEnvironment` を返す。実行環境のプラットフォーム（`posix` または `windows`）と任意の `defaultShell` である。これらのファクトは Web サーバーやブラウザではなくプロバイダーから来る。`resolveExecutable` はシェル候補を検証し、`SubprocessExecutableNotFoundError` は実行ファイルの欠落を識別する。プロバイダーとトランスポートの失敗はエラーのままである。

## サービス挙動

抽象的な [`SubprocessRuntime`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/subprocess/subprocess/src/index.ts) Service Definition は、実行世界の座標、実行ファイル・ルックアップ、通常の `spawn`、および `spawnTerminal` を規定する。[`LocalSubprocessRuntime`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/subprocess/subprocess-local/src/index.ts) は、プラットフォームが選択する管理レンジ、配置ごとの配線、クレデンシャル・スクラブ、`node-pty`、プラットフォームのプロセス検査、および terminate-and-join の破棄でそれらを提供する。Service Definition の契約は [`dsh-subprocess`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/subprocess/subprocess/README.md)、ローカル機構は [`dsh-subprocess-local`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/subprocess/subprocess-local/README.md) を参照。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxsubprocess--subprocessruntime-abstract-seam"></a>

### `ctx.subprocess` — `SubprocessRuntime`（抽象 seam）

抽象サブプロセス・サービス。サブクラス化して spawn を実装し、そのサブクラスを plugin としてロードする。`ctx.subprocess` として登録される（コンテキストごとに 1 実装。2 つ目をロードするとスローする。これは cordis 標準の重複サービス挙動である）。

実装は次のセマンティクスを守らなければならない。

- 実行ファイル・パスは、マウントされたファイルシステム・プロバイダーと共有される 1 つの実行世界に属する。
- spawn はライブ・ハンドルを同期的に返す。対象の同一性はプロバイダー・プライベートのままである。`done` は spawn されたコマンドの終了ファクトで解決し、spawn またはプロバイダーの失敗で拒否することがある。
- collect モードのリーダーはオフセット・ベースで非消費型であるため、独立したリーダーが互いの出力を消費することはない。ロッシーな読み取りは切り詰めと、存在する場合は完全なストリームを保持するスピル・ファイルを報告する。パイプされたストリームは生のまま呼び出し元に渡され、ここでバッファリングされることはない。
- SubprocessHandle.terminate（および spec の abort シグナル）は、プロバイダーが文書化した手順をその管理レンジに対して開始する。SubprocessHandle.waitForExit は同じレンジを観測するため、コンシューマー所有の teardown ラダーが各段を実際の静止に保持できる。各プロバイダーは自身のシグナリングと観測限界を文書化する。
- サービスの破棄は、まだ実行中の管理プロセスをすべて終了し、それらの終了を待つ。
- spawnTerminal は、ターミナル割り当て、テキスト転送、フォアグラウンド・グループ、シグナリング、Session 全体の静止を、1 つの await される終了メソッドの背後で所有する。準備完了と永続シェルのポリシーは PTY コンシューマーに残る。その出力ストリームは、トップレベル・プロセスが終了したときに、キュー済みのターミナル出力の後に終わる。

```ts cordis-catalog
/**
 * Resolve one configured executable in this provider's execution world.
 * Absolute paths are verified; bare names use the provider's scrubbed PATH
 * plus explicit environment overrides. Relative paths containing separators
 * are rejected: the resolution base is undefined, so providers fail loud
 * instead of guessing.
 * @param command - absolute executable path or bare PATH name.
 * @param env - explicit environment entries used for lookup.
 * @param signal - aborts remote or local lookup.
 * @returns a canonical executable path.
 */
abstract resolveExecutable( command: string, env?: Readonly<Record<string, string>>, signal?: AbortSignal, ): Promise<string>

/**
 * Inspect shell-selection facts in the provider's execution environment.
 * @param signal - cancellation of remote environment inspection.
 * @returns platform and preferred shell; executable lookup and allocation remain separate operations.
 */
abstract terminalEnvironment(signal?: AbortSignal): Promise<SubprocessTerminalEnvironment>

/**
 * Start one managed child process from a fully-specified spec; this seam
 * applies no defaults.
 * @param spec - argv, directory, stdio dispositions, grace, cancellation, and environment.
 * @returns the live process handle (streams/readers, signalling, outcome promise).
 * @throws synchronously when pre-aborted or when argv, cwd, environment, or grace is invalid before handle creation.
 */
abstract spawn(spec: SubprocessSpawnSpec): SubprocessHandle

/**
 * Allocate a real terminal and start one owned process session. This is the
 * only non-pipe process primitive: implementations own terminal byte I/O,
 * foreground groups, signals, and whole-session quiescence.
 * @param spec - fully specified argv, cwd, environment, dimensions, grace, and allocation cancellation.
 * @returns the live terminal handle after allocation succeeds.
 */
abstract spawnTerminal(spec: SubprocessTerminalSpawnSpec): Promise<SubprocessTerminalHandle>
```

ソース: [`packages/subprocess/subprocess/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/subprocess/subprocess/src/index.ts)
<!-- END GENERATED cordis-surface -->
