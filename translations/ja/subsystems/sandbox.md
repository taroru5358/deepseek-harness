# プロセス sandbox

日本語 | [English](../../../docs/subsystems/sandbox.md) | [中文](../../../docs/subsystems/sandbox.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/sandbox.md](../../../docs/subsystems/sandbox.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

[dsh-sandbox](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/sandbox/sandbox) のプロセス sandbox seam は、同じ世界のサブプロセス argv をファイル効果ポリシーでラップし、コンシューマーをプラットフォームランナーに結合しない。[dsh-sandbox-local](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/sandbox/sandbox-local) は Linux bwrap/Landlock、macOS Seatbelt、Windows ACL 制限トークンバックエンドを提供する。[dsh-bash-sandbox](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/shell/bash-sandbox) と [dsh-pwsh-sandbox](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/shell/pwsh-sandbox) がそれを消費する。[dsh-sandbox-ssh](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/ssh/sandbox-ssh/README.md) は SSH ファイルシステムとサブプロセス provider とペアになったリモートバックエンドを通じて同じポリシーを適用する。

ソース: [`packages/sandbox/sandbox/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/sandbox/sandbox/src/index.ts)

## モードと強制

`SandboxMode` はファイルシステム効果だけを統治する。`read-only` はバックエンドに書き込みを拒否させる — POSIX ランナーはさらにシェルが必要とする `/dev/null` シンクを付与し、Windows ACL ランナーは明示的な書き込み可能ルートを付与せず、その環境 ACL ギャップに対して部分的な強制を報告する。`workspace-write` は workspace ルート、明示的な追加ディレクトリ、バックエンドが約束する temp 領域の下での書き込みを許可する。`danger-full-access` は閉じ込めをバイパスする。ネットワークとプロセスの可視性はこの語彙の外である。

```ts type-equiv
/**
 * File-effect policy for confined processes. `read-only` permits only required
 * sinks such as `/dev/null`; `workspace-write` also permits the workspace,
 * explicit extra roots, and backend-defined temp areas; `danger-full-access` bypasses confinement. Network
 * and process visibility are outside this vocabulary.
 */
type SandboxMode = 'read-only' | 'workspace-write' | 'danger-full-access'
```

最初の 2 つのモードだけが provider に送信できる。`danger-full-access` のコンシューマーは元の argv を spawn し、`ctx.sandbox` を呼ばない。

```ts type-equiv
/** A confining (non-`danger-full-access`) mode — the modes a {@link SandboxPolicy} can carry. */
type ConfinedSandboxMode = Exclude<SandboxMode, 'danger-full-access'>
```

強制は報告された事実である。`full` はバックエンドがモードによって約束されたすべてのファイル効果を統治することを意味する。`partial` はアクティブなバックエンドまたは古いカーネル ABI が一部だけを統治することを意味するため、絶対的な約束を必要とするコンシューマーはその区別を拒否するか表面化しなければならない。古い Landlock ABI と Windows ACL ランナーのハードリンク、非閉じ込め読み取り、AppContainer-ACL の境界が現在の partial ケースである。

```ts type-equiv
/**
 * Enforcement completeness for this host. `partial` means an active backend or
 * older kernel ABI cannot govern every promised file effect; callers requiring
 * an absolute boundary must not treat it as `full`.
 */
type SandboxEnforcement = 'full' | 'partial'
```

## 呼び出しごとのポリシー

完全な実行ポリシーは capability 呼び出しごとに解決され運ばれる。これには `danger-full-access` が含まれるため、コンシューマーは閉じ込めをバイパスするかどうかを決める前にポリシーを 1 回解決できる。通常のツール呼び出しは呼び出し元セッションの不変 cwd から `workspaceRoot` を導出する。デプロイメント設定は agentless のフォールバックと明示的な追加書き込み許可を供給する。リゾルバーは絶対的な実行世界の綴りを保持する。強制する provider はファイルが存在する場所でルートを正規化するため、`symlink/..` を含む cwd は、ペアになったサブプロセス provider が実際に実行するディレクトリを識別する。

```ts type-equiv
/**
 * The complete file-effect policy resolved for one capability call. The root
 * is carried even under modes that do not consume it so callers can resolve
 * policy once before choosing the enforcement path.
 */
interface SandboxExecutionPolicy {
  /** The file-effect mode this execution runs under. */
  mode: SandboxMode
  /** Absolute root directory `workspace-write` may write under. */
  workspaceRoot: string
  /**
   * Extra physical absolute directories `workspace-write` may also write under.
   * Absence means none. The enforcing host rejects missing, non-directory, and
   * symlinked entries on each call. Remote paths are checked on the remote host.
   * These grant nothing under `read-only` or `danger-full-access`.
   */
  additionalWritableRoots?: readonly string[]
  /**
   * Opaque identity of the calling session (the branded `dsh-session`
   * SessionId). Backends key per-session state off it (e.g. windows-acl gives
   * each live session/workspace pair a random private temp directory and SID,
   * while the workspace SID and standing grant remain per-workspace); absent
   * for agentless calls, which fall back to per-call backend state.
   */
  sessionId?: SessionId
}
```

`ctx.sandboxPolicy.resolve()` はアクティブなセッションと、承認された再試行のために明示的なモードを受け入れる。サービスが優先順位とルートフォールバックを所有するため、bash と fs はそれを繰り返さない。

```ts type-equiv
/** Inputs that select the sandbox policy for one capability call. */
interface SandboxPolicyRequest {
  /** Calling session; its immutable cwd becomes the workspace boundary. */
  session?: Session
  /** Explicit approved mode override, which outranks session policy. */
  mode?: SandboxMode
}
```

閉じ込められた実行だけが `ctx.sandbox` に到達する。その provider ポリシーは同じルートを保持したままモードを狭める。これにより、並行セッション、コンシューマー、ワンショットのエスカレートされた再試行が、provider 状態を変更せずに同じ provider に異なる境界を要求できる。

```ts type-equiv
/**
 * What one confined execution is allowed to touch — carried PER CALL, not
 * fixed on the provider: two consumers may confine under different policies
 * at the same instant (bash under `read-only` while a confined child agent
 * needs its state directory writable), and an approved escalated retry is a
 * new call with a wider policy. Defaulting/resolution is an explicit step at
 * the consumer boundary; the provider treats the policy as fully specified.
 */
interface SandboxPolicy extends SandboxExecutionPolicy {
  /** The file-effect mode this execution runs under. */
  mode: ConfinedSandboxMode
}
```

<a id="wrapped-argv-and-classification-dialects"></a>

## ラップされた argv と分類ダイアレクト

`RunnerFailureRule` は、ランナーがコマンドを実行する前に失敗した証拠を結合する。コンシューマーは非ゼロ終了、任意の許可終了コードゲート、残りの 1 つの stderr 行内の大文字小文字を区別しない致命的シグネチャを要求する。大文字小文字を区別しない完全行の情報的除外が最初に削除されるため、無害なランナー通知だけでは失敗を証明できない。一致した行はエラー詳細として利用可能なまま残り、分類は stderr を書き換えない。

```ts type-equiv
/**
 * Evidence that identifies a sandbox runner failing before it executes the
 * wrapped command. A consumer first applies {@link allowedExitCodes} when
 * present, removes {@link informationalLines} by case-insensitive exact line
 * equality, then matches {@link fatalSignatures} case-insensitively within
 * each remaining stderr line. Exit status alone never proves runner failure.
 */
interface RunnerFailureRule {
  /** Nonzero process exit codes on which this rule may match; omitted permits any nonzero exit. */
  allowedExitCodes?: readonly number[]
  /** Non-empty substrings identifying a fatal runner diagnostic on one stderr line. */
  fatalSignatures: readonly string[]
  /** Benign stderr lines excluded by exact full-line equality before fatal matching. */
  informationalLines?: readonly string[]
}
```

`ConfinedArgv` はコンシューマーが spawn するものである。置換 argv に加えて、バックエンドの強制事実と 2 つの直交する stderr 分類子を運ぶ。`denialSignatures` は sandbox が正しく動作している間に閉じ込められたコマンドがブロックされたことを識別する。`runnerFailureRules` は sandbox ランナーがコマンドを実行する前に拒否または失敗したことを識別する。コンシューマーはこれらを最初に検査し、通常のタスク失敗ではなく sandbox インフラの失敗を表面化する。

```ts type-equiv
/**
 * A {@link SandboxProvider.confine} result: the argv to spawn in place of
 * the caller's own, plus the enforcement completeness the selected backend
 * achieves for it.
 */
interface ConfinedArgv {
  /** The wrapped argv (runner, profile, separator, then the caller's argv). */
  argv: string[]
  /** How completely the selected backend enforces the policy's file effects. */
  enforcement: SandboxEnforcement
  /**
   * The selected backend's denial DIALECT: the case-insensitive stderr
   * substrings a file effect denied by THIS backend produces (EROFS text
   * under bwrap's read-only binds, EACCES under Landlock, EPERM under
   * Seatbelt). A consumer that infers denials from a failed run's stderr
   * matches against exactly these rather than a cross-backend union — the
   * union claims denials a given backend never produces.
   */
  denialSignatures: readonly string[]
  /**
   * Structured runner-failure evidence rules. Consumers require a matching
   * fatal stderr line (after informational exclusions) and any rule-specific
   * exit-code gate before checking denial signatures: runner failure means the
   * command never ran, while denial means confinement worked and blocked it.
   */
  runnerFailureRules: readonly RunnerFailureRule[]
}
```

[ローカル provider](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/sandbox/sandbox-local/README.md) はオペレーター設定を所有し、そのランナーダイアレクトをこれらのルールにマッピングする。[サンドボックス化された bash コンシューマー](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/shell/bash-sandbox/README.md) は spawn と結果の帰属を所有する。

## Provider と fail-closed エラー

`await ctx.sandbox.confine(argv, policy, signal)` はポリシーパスを解決し、実行世界から `ConfinedArgv` を返すか、使用可能なバックエンドがないときに `SANDBOX_UNAVAILABLE` コードで `SandboxUnavailableError` を拒否する。任意のシグナルは起動前に解決をキャンセルする。コンシューマーは返された argv の spawn 中または観察中に失敗を分類してもよい。その帰属はコンシューマー契約に属する。閉じ込められたポリシーに対して静かな非閉じ込めパススルーは決して合法ではない。

provider の選択、プロービング、キャッシング、バックエンド固有の強制レポートは[ローカル provider](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/sandbox/sandbox-local/README.md) に属する。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxsandbox--sandboxprovider-abstract-seam"></a>

### `ctx.sandbox` — `SandboxProvider`（抽象 seam）

抽象プロセス sandbox サービス。confine は強制する argv を返すか、ラップ時またはランナー実行時に fail closed しなければならない。静かな非閉じ込めパススルーは禁止である。機能プローブがマルチランナーチェーンを仲裁し、単一候補ではスキップされることがある。その候補自身の拒否が fail-closed の終端として残る。

```ts cordis-catalog
/**
 * Wrap `argv` so it executes confined under `policy` on this host; the
 * caller spawns the returned argv in place of its own.
 * @param argv - the exact argv the caller is about to spawn (program plus
 *   arguments), NOT a shell string — a shell-shaped consumer passes
 *   `['bash', '-c', command]`.
 * @param policy - the file-effect policy this execution runs under,
 *   carried per call (see {@link SandboxPolicy}).
 * @param signal - cancellation while the provider resolves the policy and runner.
 * @returns the argv to spawn instead, plus the enforcement completeness
 *   the selected backend achieves for it.
 */
abstract confine(argv: readonly string[], policy: SandboxPolicy, signal?: AbortSignal): Promise<ConfinedArgv>
```

ソース: [`packages/sandbox/sandbox/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/sandbox/sandbox/src/index.ts)

<a id="ctxsandboxpolicy--sandboxpolicyservice"></a>

### `ctx.sandboxPolicy` — `SandboxPolicyService`

sandbox ポリシーサービス（`ctx.sandboxPolicy`）。デプロイメントのデフォルトモード、フォールバック workspace ルート、現在のリクエスト時ポリシーセクションを所有する。ツール層は実行ごとに resolve を呼ぶため、セッションのモードログと不変 cwd がすべての強制 capability に一緒に運ばれる。

```ts cordis-catalog
/**
 * Resolve the complete policy for one capability call. An approved explicit
 * mode outranks the session's last `sandbox/mode` event, which outranks the
 * deployment default. A session cwd is its workspace-write boundary; the
 * configured root is the fallback for agentless calls and sessions without a
 * cwd.
 * @param request - optional session and approved mode override.
 * @returns the fully resolved per-call mode and absolute workspace root.
 */
resolve(request: SandboxPolicyRequest = {}): SandboxExecutionPolicy

/**
 * Read the session override without applying the deployment default.
 * @param session - session whose log supplies the override.
 * @returns the last logged mode, or `undefined` without one.
 */
overrideOf(session: Session): SandboxMode | undefined
```

型: [Session](session.md)

ソース: [`packages/sandbox/sandbox-policy/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/sandbox/sandbox-policy/src/index.ts)
<!-- END GENERATED cordis-surface -->
