# SSH

日本語 | [English](../../../docs/subsystems/ssh.md) | [中文](../../../docs/subsystems/ssh.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/ssh.md](../../../docs/subsystems/ssh.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

[SSH プロバイダー・ファミリー](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/ssh/README.md)は、デプロイが所有する OpenSSH 接続を通じて 1 つのリモート・ファイルシステム／プロセスの世界を提供する。Harness、モデル・トランスポート、Session ストレージはホスト上に残る。このファミリーは既存のファイルシステム、サブプロセス、サンドボックスの API を実装し、SSH 固有のモデル・ツールは導入しない。

## 実行座標

ファイルシステムのアイデンティティ、実行ファイルの探索、プロセスの cwd、サンドボックスのワークスペース・ルート、言語サーバーのファイル URL は SSH ホストを指す。プロバイダーはファイルが存在する場所でパスを正規化し、`symlink/..` のファイルシステム解釈を保つ。ポリシー・リゾルバーは、Harness ホスト上でリモート・パスを解決しようとせず、絶対的な実行世界の綴りを運ぶ。

`processPath()` は、対になるサブプロセス・プロバイダーが使えるパスを提供する。`processPathFromHostPath()` は SSH では利用できないままである。リモート・アーティファクトをインストールしても、任意のホスト・パスがポータブルになるわけではない。したがって [`NodePtcRuntime`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/ptc-runtime/ptc-runtime-node/README.md) は、検証済みのリモート起動構成、すなわちインストール済みスクリプトのブートストラップか、[ヘルパー実行ファイルに埋め込まれたワーカー](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/ssh/ssh-helper-runtime/README.md)を受け取る。

## トランスポートと信頼

管理 RPC はヘルパーの SSH exec ストリームを使う。通常の stdin、stdout、stderr、ターミナル出力、および任意の fd 7 制御トラフィックは、別々に認証された転送 Unix ソケットを使う。転送される各ストリームは独自の SSH チャネル・ウィンドウを持ち、一時停止したプログラム出力が制御や管理のウィンドウを共有することはない。ただしすべてのチャネルは接続帯域とトランスポート障害を共有する。

デプロイの認証、インストール済みアーティファクトの検証、ストリームごとの TLS 認証は [`dsh-ssh`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/ssh/ssh/README.md) に属する。ヘルパーはリモート・マシン上の信頼されたローカル・プロバイダーでファイルシステムとプロセスの要求を実行する。SSH はトランスポートであり、選択されたリモート・サンドボックス・プロバイダーがファイルへの効果を強制する。

## プロセスの寿命とキャンセル

プロセスはストリームが接続される前に予約され、起動は最大 1 回だけ受け入れられる。`done` は直接の結果を報告し、`waitForExit` はリモートの管理レンジを観測する。ターミナル操作は非同期の共有 API を保つ。準備のキャンセル、起動済みプロセスの終了、プロバイダーの破棄は、ヘルパーを通じて所有リソースを解放する。

管理上のデッドラインは個々の RPC 観測を制限するが、Bash や ptc-runtime のコンシューマーが選ぶ実行デッドラインを置き換えるものではない。他の要求が進む間、リモートの待機は保留のままになりうる。SSH の喪失は保留中の操作を無効化する。ヘルパーの EOF、シグナル、リースの期限切れがリモートのクリーンアップを開始する。クライアントは未確認の結果を正直に報告し、実行されたかもしれないアクションを再生するために再接続することは決してない。

## コンポジションの範囲

Headless はマウントされたファイルシステム・プロバイダーを通じて Session の cwd を記録し検証する。したがってリモートの FS、Bash、ターミナル、LSP、PTC のコンシューマーはこれらの座標を共有できる。ホストのファイルシステム・アクセスを前提とする Web ワークスペース・ビューには別の統合が必要である。プロバイダーを置き換えるだけでは、それらのビューがリモート対応になるわけではない。

選択肢と検証義務については[決定記録](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-09-11-posix-ssh-runtime.md)を参照。

## 接続 API

```ts type-equiv
/** Installed helper invocation; script deployments may also install a PTC bootstrap. */
type HelperLaunch = {
  /** Run the installed script with a separately installed Node executable. */
  kind: 'node-script'
  /** Absolute remote Node executable. */
  node: string
  /** Absolute remote PTC bootstrap; requires bootstrapHash. */
  bootstrapPath?: string
  /** Lowercase SHA-256 of bootstrapPath; requires that path. */
  bootstrapHash?: string
} | {
  /** Run the helper executable with its embedded Node and PTC worker. */
  kind: 'executable'
}
```

```ts type-equiv
/** Deployment-owned SSH identity and installed helper; no model argument selects these values. */
interface Config {
  /** OpenSSH host alias, including its existing user, key and known-host configuration. */
  host: string
  /** Explicit script or self-contained executable invocation. */
  launch: HelperLaunch
  /** Absolute path to the installed, bundled helper entry. */
  helper: string
  /** SHA-256 of that bundled helper; mismatches refuse the connection. */
  helperHash: string
  /** Absolute remote default workspace. */
  workspace: string
  /** Connection and administrative-request deadline, at most 2,147,483,647 milliseconds. */
  requestTimeoutMs?: number
  /** Maximum JSON payload bytes per helper request or response. */
  maxFrameBytes?: number
  /** Maximum ordinary requests; heartbeat and bounded resource cleanup have reserved capacity. */
  maxPending?: number
  /** Remote helper lease; loss of heartbeats starts remote managed cleanup. */
  leaseMs?: number
}
```

```ts public-api
/** One non-reconnecting SSH session; loss invalidates all active operations. */
declare class SshConnection extends Service {
  static Config: schema<Config>;
  /** Verified remote helper coordinates; callers must await this before launch. */
  readonly ready: Promise<Hello>;
  constructor(ctx: Context, config: Config);
  /** Hold plugin readiness until the remote identity and helper digest are verified. */
  async [Service.init](): Promise<void>;
  /** Verified remote PTC launch configuration; script deployments require a verified bootstrap. */
  get ptcLaunch(): { kind: 'embedded'; executable: string } | { kind: 'node-script'; executable: string; bootstrapPath: string };
  /**
     * Send a helper operation; cancellation never replays an ambiguous mutation.
     * @param method - the private helper operation.
     * @param params - JSON request fields validated by the helper.
     * @param result - response validation before returning provider-visible data.
     * @param signal - cancellation, which does not undo completed remote effects.
     * @param wait - allow a process observation to outlast the administrative deadline.
     * @returns the validated remote result.
     */
  async request<T>(method: string, params: unknown, result: z.ZodType<T>, signal?: AbortSignal, wait: boolean = false): Promise<T>;
  /**
     * Forward one authenticated stream through an independent SSH channel.
     * @param endpoint - private coordinates issued by this connection's helper.
     * @param signal - cancellation of allocation and the resulting socket.
     * @returns a paused socket; attach a consumer before resuming it.
     */
  async connectStream(endpoint: SshStreamEndpoint, signal?: AbortSignal): Promise<Socket>;
  /** Tear down the helper's remote managed ranges before releasing the SSH master when reachable. */
  dispose(): Promise<void>;
}
```

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxssh--sshconnection"></a>

### `ctx.ssh` — `SshConnection`

再接続しない 1 本の SSH セッション。喪失はすべてのアクティブな操作を無効化する。

```ts cordis-catalog
/**
 * Send a helper operation; cancellation never replays an ambiguous mutation.
 * @param method - the private helper operation.
 * @param params - JSON request fields validated by the helper.
 * @param result - response validation before returning provider-visible data.
 * @param signal - cancellation, which does not undo completed remote effects.
 * @param wait - allow a process observation to outlast the administrative deadline.
 * @returns the validated remote result.
 */
async request<T>(method: string, params: unknown, result: z.ZodType<T>, signal?: AbortSignal, wait: boolean = false): Promise<T>

/**
 * Forward one authenticated stream through an independent SSH channel.
 * @param endpoint - private coordinates issued by this connection's helper.
 * @param signal - cancellation of allocation and the resulting socket.
 * @returns a paused socket; attach a consumer before resuming it.
 */
async connectStream(endpoint: SshStreamEndpoint, signal?: AbortSignal): Promise<Socket>

/** Tear down the helper's remote managed ranges before releasing the SSH master when reachable. */
dispose(): Promise<void>
```

ソース: [`packages/ssh/ssh/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/ssh/ssh/src/index.ts)
<!-- END GENERATED cordis-surface -->
