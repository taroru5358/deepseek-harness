# PTC ランタイム

日本語 | [English](../../../docs/subsystems/ptc-runtime.md) | [中文](../../../docs/subsystems/ptc-runtime.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/ptc-runtime.md](../../../docs/subsystems/ptc-runtime.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

PTC 実行 [capability seam](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-06-13-capability-seams.md) は [dsh-ptc-runtime](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/ptc-runtime/ptc-runtime) を通じて `ctx.ptcRuntime` を提供する。1 つのプログラムを host バインディングに対して実行し、出力、失敗、該当する sandbox の事実を報告する。PTC 実行は [agent-loop の背骨](core.md) の一部ではなく任意である。[PTC foundation](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/feature/2026-06-15-ptc.md) はレジストリ表示を所有し、[typed-return 契約](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/feature/2026-07-20-ptc-typed-tool-returns.md) はバインディング値を所有し、[sandboxed Node の決定](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-09-11-sandboxed-node-ptc-runtime.md) は出荷される実行 provider を所有する。

ソース: [`packages/ptc-runtime/ptc-runtime/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/ptc-runtime/ptc-runtime/src/types.ts)

## 実行: リクエスト入力、結果出力

`PtcRunRequest` はプログラム、バインディング、キャンセル、任意の実行選択を含む。provider の `resolve` はサポートされる選択を検証し、デプロイメントのデフォルトを適用する。`run` は明示的なディレクトリと期限の選択を持つ `PtcRunSpec` を受け取る。省略されたタイムアウトは provider のデフォルトを使い、数値は上限付きの経過予算を要求し、`null` は経過期限なしを要求する。provider は実行前にサポートされない選択を拒否する:

```ts type-equiv
/**
 * Caller inputs for one program. The provider's resolve method validates supported
 * options and supplies directory, deadline, and authority before execution.
 */
interface PtcRunRequest {
  /**
   * The program source, in the runtime's {@link ../index.ts | language}. It
   * follows the provider's programFormat: an async function body supports
   * top-level `await` and `return`; REPL and Script programs support top-level
   * declarations, `await`, and a final expression, but not top-level `return`.
   * The Script provider lowers top-level await; its executionInstructions and
   * documentation define retained-binding semantics. The completion value
   * becomes {@link PtcRunResult.value}.
   */
  program: string
  /**
   * Object-identity owner for optional in-memory state. Omission requests a fresh
   * environment. Consumers must await runtime.release(owner) when the owner ends.
   * Stateless providers ignore this field; retained state is never durable.
   */
  stateOwner?: object
  /** Host functions exposed to the program, one global object per namespace. */
  bindings: PtcBindingNamespace[]
  /** Working directory in the mounted filesystem and subprocess execution world. */
  cwd?: string
  /**
   * Elapsed execution budget in milliseconds. Omission uses provider defaults;
   * null requests no deadline. Providers validate and cap numeric budgets or reject unsupported choices.
   */
  timeoutMs?: number | null
  /** Resolved authority for this execution. Providers without confinement reject an explicit policy. */
  sandboxPolicy?: SandboxExecutionPolicy
  /**
   * Abort the run: the runtime stops the program (hard, even mid-loop) and
   * resolves with a {@link PtcRunFailure} of kind `'abort'`. In-flight
   * binding calls are the CALLER's to settle — the runtime only stops asking.
   */
  signal?: AbortSignal
}
```

```ts type-equiv
/** Fully resolved execution inputs; run never supplies a missing directory or deadline choice. */
interface PtcRunSpec extends PtcRunRequest {
  /** Absolute directory in the provider's execution world. */
  cwd: string
  /** Positive finite elapsed budget in milliseconds after provider capping, or null for no deadline. */
  timeoutMs: number | null
}
```

```ts type-equiv
/** File confinement applied to a program, independently of its terminal outcome. */
interface PtcRunSandbox {
  /** File-effect mode used for this execution. */
  mode: SandboxMode
  /** Program failure text matched backend diagnostics; not enforcement proof or an exhaustive denial record. */
  denied: boolean
  /** Completeness reported by the selected confining backend; absent for full access. */
  enforcement?: SandboxEnforcement
}
```

プログラムの失敗は `PtcRunResult.error` で解決する。無効な呼び出し元入力は実行前に拒否されることがある。sandbox モード、観察された拒否、強制の完全性は別々の事実であるため、プログラムの成功だけでは要求されたすべての制限が強制されたことを証明しない:

```ts type-equiv
/**
 * The outcome of one run. An error is a FIELD on a resolved result, never a
 * rejection of `run()` — reporting a failed program is the caller's job, not
 * an exception path.
 */
interface PtcRunResult {
  /** Applied file policy and observed denial, when the provider enforces file policy. */
  sandbox?: PtcRunSandbox
  /**
   * The program's completion value (a return or final REPL/Script expression), when it ran to
   * completion and the value crossed the runtime's lossless-JSON boundary.
   * Invalid or over-limit completions fail the run instead of substituting a
   * rendered string; a failed or value-less run leaves this absent.
   */
  value?: PtcJsonValue
  /**
   * Captured text. Each source channel preserves emission order; interleaving
   * across independent channels is backend-dependent. Bounded only as part of
   * the outer result.
   */
  logs: string[]
  /** Present iff the run failed; see {@link PtcRunFailure} for the taxonomy. */
  error?: PtcRunFailure
}
```

```ts type-equiv
/** Source evaluation form, independent of retained program state. */
type PtcProgramFormat = 'function-body' | 'repl' | 'script'
```

`programFormat` は async 関数本体、REPL 文、スクリプトを区別する。REPL と Script プログラムはトップレベルの `await`、宣言、最終式を受け入れる。トップレベルの `return` はサポートされない。Node Script provider はトップレベル await を lower し、完了値を await する。その[宣言の制限](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/ptc-runtime/ptc-runtime-node/README.md#known-limitations-and-deferred-work)は変換されたプログラムに適用される。`retainsState` はソース構文とは独立にメモリ保持を報告する。`stateOwner` は任意の保持メモリのためのインプロセスオブジェクト同一性であり、異なるオブジェクトは分離されたままである。コンシューマーはその所有者が終了したときに `release(owner)` を待つ。クリーンアップ失敗はその呼び出しを拒否する。省略された所有者はワンショットであり、ステートレス provider は所有権を無視する。[永続 Node provider](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/ptc-runtime/ptc-runtime-node/README.md#persistent-repl) はリセットと復旧の動作を定義する。保持されたオブジェクトが永続 Session フォーマットに入ることはない。

## バインディング: プログラムグローバルとしての host 関数

各 `PtcBindingNamespace` は async 呼び出し可能のグローバルオブジェクトになる。PTC は `tools` を渡す。引数と解決はロスレス JSON でなければならない。provider は自身のトランスポート上限を強制する。seam は一律のバインディングバイト上限を設定しない。任意のエラークラス記述子は、runtime 内でコンシューマーを名指しすることなく、プログラム可視の型付き拒否を生成する。バインディング名は own プロパティなので、`__proto__` がプロトタイプをたどることはできない:

```ts type-equiv
/**
 * Program-visible typed rejection for one binding namespace. The runtime
 * injects a real error constructor under `name`; rejected member calls become
 * its instances and expose the exact member name through
 * `memberNameProperty`. Both strings are runtime data rather than knowledge
 * of a particular consumer such as PTC mode.
 */
interface PtcBindingErrorClass {
  /** Constructor global and resulting `Error.name`; same portable identifier rule as {@link PtcBindingNamespace.global}. */
  name: string
  /**
   * Non-empty own property for the member name. The portable exclusion set is
   * `RESERVED_ERROR_MEMBERS` plus dunder-form names (`__x__`, non-empty
   * middle), enforced identically by every backend; any other name —
   * identifiers or not — is accepted everywhere.
   */
  memberNameProperty: string
}
```

```ts type-equiv
/**
 * A named group of {@link PtcBindingFunction}s the runtime exposes to the
 * program as one global object (e.g. `tools`). Function names are arbitrary
 * strings — a runtime must treat names like `__proto__` or `constructor` as
 * ordinary own properties (null-prototype construction), never as prototype
 * collisions.
 */
interface PtcBindingNamespace {
  /**
   * The global identifier the program sees. Must match the LANGUAGE-PORTABLE
   * identifier subset `[A-Za-z_][A-Za-z0-9_]*` and no language's reserved
   * words, so the same namespace list works against every backend regardless
   * of `language` — a JS-only spelling like `$tools` is rejected by design,
   * not just by the Python backend. Names that satisfy the identifier rule but
   * name a backend-owned slot (`RESERVED_BINDING_GLOBALS`, e.g. `console`,
   * `__dsh_main__`) are also refused everywhere; see its declaration for the
   * exact set and why each entry is reserved.
   */
  global: string
  /** The callable members, keyed by the exact name the program calls. */
  functions: Record<string, PtcBindingFunction>
  /** Optional program-visible typed rejection contract for this namespace. */
  errorClass?: PtcBindingErrorClass
}
```

```ts type-equiv
/** A lossless JSON value transferable through the dependency-light Service Definition. */
type PtcJsonValue = null | boolean | number | string | PtcJsonValue[] | { [key: string]: PtcJsonValue }
```

```ts type-equiv
/**
 * One host-side function exposed to the program as an async callable. The
 * runtime bridges calls to it (possibly across a serialization boundary), so
 * `args` and the resolution value MUST be lossless JSON. A runtime rejects a
 * lossy or non-cloneable value with a descriptive error rather than corrupting
 * the run. No seam-level byte cap applies to a binding resolution. A rejection
 * of this function surfaces inside the program as a rejection of the
 * corresponding call.
 */
type PtcBindingFunction = (args: unknown) => Promise<PtcJsonValue>
```

## キャプチャされた出力と失敗の分類

ログは平文の文字列である。各ソースチャネルは emit 順序を保持するが、独立チャネル間のインターリーブはバックエンド依存である。チャネルメタデータは seam の一部ではないからである。runtime はプログラムの console とストリーム出力をキャプチャし、コンシューマーはテキストだけをレンダリングする。実装はシリアライズされた外側のログ配列と完了値または失敗メッセージのペイロードに上限を課す。固定の結果エンベロープ構文とコンシューマー表示の空白は、その可変ペイロード台帳の一部ではない。オーバーフローはインバンドの値置換ではなく明示的な失敗である。

失敗の種類は**独立に報告される直交する結果**である（[defensive-patterns](../defensive-patterns/general.md) に従う）: 予算の期限切れは例外ではなく、abort はタイムアウトではなく、基盤の終了（例: OOM）はどちらでもない:

```ts type-equiv
/**
 * Why a run failed. The kinds are orthogonal outcomes reported independently
 * (per docs/defensive-patterns/general.md): a budget expiry is not an exception, an
 * abort is not a timeout, and a substrate death is neither.
 *
 * - `'exception'` — the program threw or failed to parse/transform.
 * - `'timeout'` — an implementation-owned budget expired; the message says which.
 * - `'abort'` — {@link PtcRunRequest.signal} fired.
 * - `'worker-exit'` — the execution substrate died without settling (e.g. OOM).
 * - `'invalid-output'` — the completion value was not lossless JSON.
 * - `'output-limit'` — the serialized outer logs/value/diagnostic exceeded the configured cap.
 * - `'protocol'` — the program sent invalid or over-budget control traffic.
 * - `'sandbox-unavailable'` — required confinement could not be established.
 */
interface PtcRunFailure {
  /** The failure class (see the interface doc for each kind's meaning). */
  kind: 'exception' | 'timeout' | 'abort' | 'worker-exit' | 'invalid-output' | 'output-limit' | 'protocol' | 'sandbox-unavailable'
  /** Human-readable detail, suitable for feeding back to a model to self-correct. */
  message: string
}
```

## サービス

`PtcRuntime` は [`src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/ptc-runtime/ptc-runtime/src/index.ts) で定義される。`resolve(request)` は完全な実行入力を受け取り、`run(spec)` がそれらを実行する。`executionInstructions` はコンシューマー表示のための provider 所有の使用ガイダンスを提供する。`timeout` は呼び出しごとの上書きがサポートされるときに、設定された経過時間のデフォルトと最大値を報告する。`resolve` は依然として各リクエストを検証し上限を課す。`language` はサポートされるプログラム表示を選択する。`isolation` はセキュリティを主張せずに基盤を記述する。`sandboxMode` はファイルポリシーのサポートを通知し、閉じ込めを提供しない provider では `undefined` である。各実装は実行間でプログラム状態を分離し、破棄中にアクティブな実行を終了して待機する。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxptcruntime--ptcruntime-abstract-seam"></a>

### `ctx.ptcRuntime` — `PtcRuntime`（抽象 seam）

`ctx.ptcRuntime` の実装を 1 つ登録する。プログラム、予算、abort、基盤の失敗は PtcRunResult で解決する。拒否するのは Service Definition 契約の誤用だけである。実装は構造化クローン可能なバインディングをブリッジし、宣言された各名前空間の拒否クラスを実体化し、プログラムを敵対的ピアとして扱い、異なる状態所有者を分離し、破棄中に進行中の実行を終了して待機する。

```ts cordis-catalog
/**
 * Release an owner's retained environment and await its execution resources.
 * Stateless providers have no retained resources to release.
 * @param owner - The exact object supplied as the request's state owner.
 * @returns Completion after all owned execution resources have been released.
 * @throws When managed resource cleanup fails.
 */
release(owner: object): Promise<void>

/**
 * Resolve supported options and provider defaults before execution.
 * @param request - Program, bindings, cancellation and optional execution choices.
 * @returns Complete directory, deadline and supported authority for run.
 * @throws When an explicit choice is invalid or unsupported by this provider.
 */
abstract resolve(request: PtcRunRequest): PtcRunSpec

/**
 * Execute resolved inputs; program outcomes resolve as result fields.
 * @param spec - directory, deadline, program, bindings, cancellation and supported policy.
 * @returns Captured output and the execution outcome.
 */
abstract run(spec: PtcRunSpec): Promise<PtcRunResult>
```

ソース: [`packages/ptc-runtime/ptc-runtime/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/ptc-runtime/ptc-runtime/src/index.ts)
<!-- END GENERATED cordis-surface -->
