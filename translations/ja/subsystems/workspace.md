# ワークスペース

日本語 | [English](../../../docs/subsystems/workspace.md) | [中文](../../../docs/subsystems/workspace.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/workspace.md](../../../docs/subsystems/workspace.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

ワークスペースとは、ユーザーが作業するディレクトリの永続的な記録である。正規化されたパスに対する安定した ID、表示タイトル、そしてそこに属するセッションの順序付きの台帳からなる。このサブシステムは 1 つのパッケージ（[dsh-workspace](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/workspace/workspace)、`ctx.workspaceRegistry`）であり、ホスト側のオプションのケイパビリティであって agent-loop の背骨の一部ではなく、モデルからは見えない（ツールも、プロンプト・テキストも、セッション・イベントもない）。記録は [storage の domain 形式](storage.md) を通じて保存し、セッションの所属は [`SessionHeader.cwd`](persistence.md#sessionheader--metadata-beside-the-log) に対して検証する。したがって `storageDomain` と `sessionPersistence` は必須の起動時依存である。永続化のピアが利用できない場合、plugin は空の履歴と誤認されるのではなく pending のままになる。設計記録: [domain KV storage Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/proposed/architecture/2026-07-24-domain-kv-storage-and-workspace.md)、ブートストラップと GUI の順序: [Workspace UI product-flow Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/archived/feature/2026-07-25-workspace-ui-product-flow.md)。

ソース: [`packages/workspace/workspace/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/workspace/workspace/src/types.ts)

## 同一性

```ts type-equiv
/**
 * Identifies one workspace record. A generated uuid, never the path: path
 * normalization rewrites paths, and a reference anchor must stay stable.
 */
type WorkspaceId = Branded<'WorkspaceId'>
```

`WorkspaceId` は[ブランド付き ID](core.md#branded-ids) である。パスの同一性は別であり、`realpathNormalize`（`fs.realpath`。末尾スラッシュ、`..`、シンボリックリンクを解決する）が唯一の一意性の規範である。ワークスペースのパスは正規化して保存され、一意性は正規化済みパスの文字列一致であり（所有済みディレクトリへのシンボリックリンクは衝突する）、アタッチ時のセッション cwd チェックも同じ規範を通る。

## ワークスペース・エンティティ

コンシューマーが見るのは `Workspace` インターフェースだけであり、実装はパッケージ・プライベートのままである。

```ts type-equiv
/**
 * One workspace: a stable id over an existing directory, a display title, and
 * an ordered candidate account of sessions. Membership validates header cwd
 * against the workspace path, but preserves an existing account when its
 * fully qualified cwd is missing. Consumers only see this interface; the
 * implementation stays private.
 */
interface Workspace {
  /** Stable record id (generated uuid). */
  readonly id: WorkspaceId

  /**
   * Canonical directory path: the `fs.realpath` of the path given at create
   * time (trailing slashes, `..`, and symlinks all resolved). Never rewritten
   * afterwards, even when the directory disappears (see {@link status}).
   */
  readonly path: string

  /** Display title. Defaults to the final path segment, or a filesystem root's own spelling; duplicates are allowed. */
  readonly title: string

  /** ISO-8601 creation instant, stamped at create and never rewritten. */
  readonly createdAt: string

  /** ISO-8601 instant of the last durable mutation (create counts as one). */
  readonly updatedAt: string

  /**
   * Header-validated sessions in manually owned order: a new session is
   * prepended at attach, explicit reordering goes through
   * `insertSessionBefore`, and activity never reorders. The durable candidate
   * account is filtered synchronously: missing headers, invalid cwd values,
   * and canonical cwd mismatches are never returned. A fully qualified cwd
   * whose realpath or directory stat fails with ENOENT preserves an existing
   * candidate, including a dangling symlink; other filesystem failures do not.
   * A subsequent mutation prunes filtered candidates, not missing-directory
   * candidates. This does not admit new sessions or recreate directories.
   */
  readonly sessionIds: readonly SessionId[]

  /**
   * Replace the display title durably.
   * @param title - New title; any string, duplicates across workspaces allowed.
   * @returns resolution after durability.
   */
  setTitle(title: string): Promise<void>

  /**
   * Prepend a session to this workspace's candidate account. An already
   * accounted id resolves without writing, aside from the durable
   * filtered-candidate prune every accepted mutation performs. A new id's
   * live or persisted
   * header cwd must resolve to an existing directory equal to {@link path};
   * unknown ids, missing or invalid cwd values, and mismatches reject without
   * writing.
   * @param sessionId - The session to record.
   * @returns resolution after durability.
   */
  attachSession(sessionId: SessionId): Promise<void>

  /**
   * Move an accounted session within the manual order, DOM-insertBefore-like:
   * with an anchor the session lands before it, without one it appends to the
   * end. Only the moved id changes position. A session or anchor absent from
   * the account rejects without writing; a move to the current position
   * resolves without writing, aside from the durable filtered-candidate
   * prune every accepted mutation performs; decided on the domain write
   * chain.
   * @param sessionId - The accounted session to move.
   * @param beforeSessionId - Accounted anchor to insert before; omitted appends.
   * @returns resolution after durability.
   */
  insertSessionBefore(sessionId: SessionId, beforeSessionId?: SessionId): Promise<void>

  /**
   * Remove a session from this workspace's account. Idempotent: an id not on
   * the account resolves without writing, aside from the durable
   * filtered-candidate prune every accepted mutation performs; decided on
   * the domain write chain like attach. Never touches the session's own stored log.
   * @param sessionId - The session to remove.
   * @returns resolution after durability.
   */
  detachSession(sessionId: SessionId): Promise<void>

  /**
   * Live directory check, uncached: whether {@link path} currently exists and
   * is a directory. A missing directory never mutates the record — the
   * directory may only be temporarily moved.
   * @returns `'ok'` when the directory exists, `'missing-dir'` otherwise.
   */
  status(): Promise<'ok' | 'missing-dir'>
}
```

所有の真実はレコードの順序付き `sessionIds` であり、セッションの cwd から導出されることはない。所属は記録済みのヘッダー cwd をワークスペースのパスに対して検証する。既存の候補は、その完全修飾 cwd の解決またはディレクトリの stat が `ENOENT` で失敗しても所有されたまま残る。これにより、ディレクトリ欠落やダングリング・シンボリックリンクの台帳を、新しいセッションを認めることなく保持する。失敗した書き込みは拒否される（`insertSessionBefore` の台帳エラーは `WorkspaceMoveInvalidError`、ストレージ障害は素のエラー）。受け入れられたすべての変更は `updatedAt` を刻み、所属チェックを通過しなくなった候補を永続的に刈り込む。

## レジストリ: `ctx.workspaceRegistry`

`WorkspaceRegistry`（[シグネチャ](#ctxworkspaceregistry--workspaceregistry)）は登録と解決を所有する。`create(path, title?)` は完全修飾パスを要求し、それを正規化し、存在しないパス（元の `ENOENT`）やディレクトリでないパスを拒否する。正規化済みパスがすでに所有されている場合は既存のエンティティを変更せずに返し、そうでなければ `title ?? defaultWorkspaceTitle(path)` を持つレコードを永続レジストリ順序の先頭に作成する（異なる正規化パスが同じ表示タイトルを共有してよく、最終セグメントを持たないパスはそのルートの綴りを使う）。`get(id)` と順序付きの `list()` は同期的なキャッシュ読み取りである。`resolveByPath(path)` は作成せずに同じ完全修飾 realpath 規範を適用する。`delete(id)` は登録、順序エントリー、セッション台帳だけを削除する。ディレクトリ、ユーザー・ファイル、ライブ・セッション、永続化ログには決して触れないため、それらのセッションは Ungrouped になる（[リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/workspace/workspace/README.md)）。未知の ID は `false` を返す。作成と削除は、2 つの書き込み（レコード + 順序）が食い違う前に保留中の変更マーカーを永続化する。起動時はマークされた変更を正確に解決する。すなわちマークされたテーブル行を削除することで、中断された削除を完了し、中断された作成をロールバックする（登録は再作成可能なので、ロールバックが安全な方向である）。マークのない順序／テーブルの不一致は破損として明示的に失敗する。

セッションの cwd は作成時に、このレジストリではなくセッションを作成する側から与えられる。API gateway は新しいセッションの cwd を、選択されたワークスペースの `path` から解決し（明示的またはデフォルトの cwd にフォールバックする）、その cwd が不変の [`SessionHeader`](persistence.md#sessionheader--metadata-beside-the-log) に入るようにセッションを作成し、その後 `attachSession` を呼ぶ。`attachSession` は保存済みのヘッダー cwd をワークスペースのパスに対して再検証する。最初の起動が成功すると、レジストリは永続化済みのヘッダーだけ（`id`、`cwd`、`createdAt`。イベント本体は決して使わない）から履歴をブートストラップし、有効な正規化 cwd を持つセッションをディレクトリごとのワークスペースに、新しい順でグループ化する。初期化済みマーカーは最後に書き込まれるため、中断されたブートストラップは安全に再開できる。ブートストラップは 1 回限りである。cwd を持たないレガシー・セッションは Ungrouped のままで、その後作成されたセッションは `attachSession` を通じてのみワークスペースに参加する。

## デフォルト・ワークスペースの初期化

コントローラーの `initializeDefault` はリクエストを受け取らない。固定の `default-workspace` ディレクトリ名を所有し、Documents の場所を解決し、レジストリに 1 回だけ初期化を要求する。レジストリはディレクトリ・リゾルバーを受け取り、初期タイトルを正規化後のセグメントではなく要求されたディレクトリの最終セグメントから導出し、登録をその永続的な同一性とともにコミットする。言語は Host に届かない。ブラウザのコンシューマーは、その自動タイトルをまだ持つワークスペースをコントローラーの `workspaceDisplayTitle` を通じてラベル付けするため、画面上の名前だけが読者の言語に従う。[初回利用時の挙動と設定](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/workspace-controller/README.md#first-use-workspace)が、再利用と障害時の扱いを説明する。

## セッションのピン留め

コントローラーの[トランスポート型](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/workspace-controller/src/types.ts)は `WorkspacePinSessionRequest` と `WorkspaceUnpinSessionRequest` を定義し、それぞれが 1 つの `sessionId` を運ぶ。どちらの操作も `WorkspacePinValue` を返す。これは Session ID の完全な `pinnedSessionIds` 配列で、最後にピン留めしたものが先頭になる。ピン留めには既知でアーカイブされていない Session が必要である。ピン留めされていない ID のピン解除は、集合を変えずに成功する。アーカイブは同じ永続書き込みで Session のピンを削除し、アンアーカイブはそれを復元しない。

## アーカイブの受け入れ

アーカイブはレジストリ全体で共有される永続的な集合であり、レジストリはその背後で実行中の作業を隠すことを拒否する。このルールは、パッケージが宣言・ディスパッチする 2 つの Host イベント（[イベント](#workspace-events)）に対するケイパビリティ seam である。`workspace/session-activity`（waterfall）は構成済みのプロバイダーに、Session のためにまだ何が動いているかを尋ね、`workspace/session-stop`（parallel）はそれを停止するよう求める。各プロバイダーは他のリスナーと同様にルートに登録するため、パッケージは agent、job、schedule といった語彙を知らない。ファミリーはマージ拡張可能なマップのキーである。

```ts type-equiv
/**
 * Activity families a `workspace/session-activity` listener may report. This
 * package declares none: each provider merges its own key from a module both
 * its Host and Client faces import, so a consumer that renders the families
 * sees exactly the keys its program compiled and falls through to a generic
 * description for any other. The shipped providers merge `turn` (the Agent
 * registry), `job` (the job registry seam), `subagent` (the Subagent
 * runtime), and `schedule` (the Schedule plugin).
 */
interface SessionActivityKindMap {}
```

`SessionActivityKind` は `keyof SessionActivityKindMap` であるため、プロバイダーを 1 つもコンパイルしていないプログラムはキーをまったく見ない。出荷されるキーはクライアントからインポート可能な型モジュールにある。`turn` は Agent レジストリの `types.ts`、`job` は job レジストリ seam の `view.ts`、`subagent` は Subagent ランタイムの `control-types.ts`、`schedule` は Schedule plugin の `types.ts` である。ファミリーを描画するコンシューマーは、自らのケースのためにこれらのモジュールをインポートし、その他のキー用に汎用の行を保つ。プロバイダーは `next()` の結果の先頭に自身の `SessionActivity` エントリーを追加して waterfall に答える。レジストリの最も内側のコールバックは空のリストを返すため、プロバイダーのない構成では自由にアーカイブできる。

```ts type-equiv
/**
 * One reason a session counts as active for archive admission. Families with
 * per-item identity list their items so a caller can name what must stop.
 */
interface SessionActivity {
  readonly kind: SessionActivityKind
  /** Active items of the family; absent for a family without per-item identity (`turn`). */
  readonly items?: readonly SessionActivityItem[]
}
```

`SessionActivityItem` はファミリー固有の `id`（session、job、schedule の ID）と、任意の表示用 `label` を運ぶ。`archiveSession(sessionId)` は存在チェックの後に waterfall を 1 回だけ尋ね、空でない答えに対しては何も書き込まずに `WorkspaceActiveSessionError`（`sessionId`、`activity`）で拒否する。コントローラーはそれを `workspace/session-active` エラーにマップし、その詳細は同じ 2 つのフィールドを運ぶ。`archiveSession(sessionId, { stopActivity: true })`（トランスポート・リクエストが `stopActivity` として公開する `ArchiveSessionOptions` のフィールド）はチェックを飛ばしてアーカイブを書き込み、その後 `workspace/session-stop` をディスパッチする。拒否するプロバイダーはログに記録され、アーカイブは残り、停止された作業の決着は待たれない。すでにアーカイブ済みの ID は尋ねも停止もしない。出荷されるプロバイダー、それらが停止するもの、アーカイブ済み Session がモデルの step を実行しないようにする `agent/pre-step` ゲートは、[レジストリ・パッケージ](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/workspace/workspace/README.md#api-behavior)に記述されている。決定記録は [archive-stops-running-work Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/feature/2026-09-21-archive-stops-running-session-work.md) である。

## コンシューマー

[`dsh-workspace-controller`](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/api/workspace-controller) は `ctx.workspaceRegistry` を介して GUI クライアントにワークスペースの CRUD を提供し、[`dsh-session-controller`](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/api/session-controller) は上記の「セッションを作成してからアタッチする」フローを実行する。[dsh-agent-instructions](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/context/agent-instructions) は名前にもかかわらずコンシューマーでは**ない**。agent 自身の cwd の下にある AGENTS.md 形式の指示ファイルを発見するだけで、`ctx.workspaceRegistry` には決して触れない。共有されている語は、このレジストリのエンティティではなく、ユーザーの作業ディレクトリを指している。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxdirectorypicker--directorypicker-abstract-seam"></a>

### `ctx.directoryPicker` — `DirectoryPicker`（抽象 seam）

抽象的なディレクトリ選択サービス。サブクラス化して `capability()` を実装し、そのサブクラスを plugin としてロードする。`ctx.directoryPicker` として登録される（コンテキストごとに 1 実装。2 つ目をロードすると、cordis 標準の重複サービス挙動でスローする）。ケイパビリティ・オブジェクトはサービスの生存期間中安定していなければならない。コンシューマーは呼び出しをまたいでそれを捕捉してよい。

```ts cordis-catalog
/**
 * The backend's interaction capability.
 * @returns the discriminated capability consumers switch on.
 */
abstract capability(): DirectoryPickerCapability
```

ソース: [`packages/host/directory-picker/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/host/directory-picker/src/index.ts)

<a id="ctxdirectorypickercontroller--directorypickercontroller"></a>

### `ctx.directoryPickerController` — `DirectoryPickerController`

生成された `ctx.remote.directoryPicker` 名前空間を支える Host サービス。エクスポートする seam は抽象的であり、それ自体が Loader エントリーになることはないため、このコントローラーがワイヤー上の verb を担う。構成済みの 1 つのバックエンドがネイティブ・チューザーまたはブラウズ・プリミティブのどちらかを提供し、構成が提供できない verb は近似されるのではなく拒否される。

```ts cordis-catalog
/**
 * Open the host's OS chooser for a Remote caller.
 * @param signal - caller lifetime; abort terminates the chooser.
 * @returns the chosen absolute path, or null when the operator cancels.
 */
@Remote('pick') async pick(signal: AbortSignal): Promise<string | null>

/**
 * List one directory level for a Remote caller's in-app browser.
 * @param path - absolute directory to list; absent lists the home directory.
 * @param signal - caller lifetime; abort stops the backend's scan instead of
 *   letting it outlive a disconnected caller.
 * @returns the level's listing with its ancestry.
 */
@Remote('list') async list(path: string | undefined, signal: AbortSignal): Promise<DirectoryListing>

/**
 * Create one child directory for a Remote caller's in-app browser.
 * @param path - absolute existing parent directory.
 * @param name - single non-blank path segment.
 * @returns the created directory's absolute path.
 */
@Remote('createDirectory') async createDirectory(path: string, name: string): Promise<string>
```

ソース: [`packages/api/workspace-controller/src/directory-picker.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/workspace-controller/src/directory-picker.ts)

<a id="ctxterminalcontroller--terminalcontroller"></a>

### `ctx.terminalController` — `TerminalController`

Session が所有する一時的なターミナル・プロセスの型付き Remote 制御。

```ts cordis-catalog
/**
 * Read the Session's current directory and terminal limits without filesystem validation or shell lookup.
 * @param agent - Session owner supplied by the Gateway.
 * @param signal - request cancellation.
 * @returns the logged current directory and terminal limits; retained terminals keep their own process directories.
 */
@Remote environment(agent: Agent, signal: AbortSignal): TerminalEnvironment

/**
 * Discover installed shells in the Session's execution environment.
 * @param agent - Session owner supplied by the Gateway.
 * @param signal - request cancellation.
 * @returns verified profiles, with the configured or system default first.
 */
@Remote shells(agent: Agent, signal: AbortSignal): Promise<TerminalShell[]>

/**
 * List retained terminals without resolving or activating an Agent.
 * @param sessionId - displayed Session identity, including offline history.
 * @returns terminals retained for this Host lifetime.
 */
@Remote list(sessionId: SessionId): WebTerminalInfo[]

/**
 * Allocate a user shell once for a caller-generated identity, without Agent sandbox or approval restrictions.
 * @param agent - Session owner supplied by the Gateway.
 * @param request - initial dimensions and idempotency identity.
 * @param signal - allocation cancellation; committed terminals survive disconnection.
 * @returns the existing or newly committed terminal.
 */
@Remote async create(agent: Agent, request: TerminalCreateRequest, signal: AbortSignal): Promise<WebTerminalInfo>

/**
 * Retain an existing terminal for a window without activating its Agent or taking input control.
 * @param sessionId - owning Session identity, including an inactive saved layout.
 * @param id - retained Host terminal identity.
 * @param signal - physical Remote stream cancellation.
 * @returns a hold acknowledgement followed by an open lifetime stream.
 */
@Remote({ mode: 'stream' }) retain(sessionId: SessionId, id: WebTerminalId, signal: AbortSignal): AsyncIterable<TerminalRetentionFrame>

/**
 * Attach terminal input and output to one physical Remote stream.
 * @param agent - Session owner supplied by the Gateway.
 * @param id - terminal identity.
 * @param signal - attachment cancellation; the process remains alive and input is never replayed.
 * @returns screen recovery, output, writable state and provider-acceptance acknowledgements.
 */
@Remote({ mode: 'stream' }) attach(agent: Agent, id: WebTerminalId, signal: AbortSignal): RemoteStream<TerminalFrame, TerminalInput>

/**
 * Rename a terminal without changing its shell.
 * @param agent - Session owner supplied by the Gateway.
 * @param id - terminal identity.
 * @param title - nonempty display title, at most 120 characters.
 */
@Remote rename(agent: Agent, id: WebTerminalId, title: string): void

/**
 * Close an identity to future creation and kill its process range; repeated closes succeed.
 * @param agent - Session owner supplied by the Gateway.
 * @param id - terminal identity.
 * @returns after provider cleanup succeeds. A failure retains the terminal for retry.
 */
@Remote async close(agent: Agent, id: WebTerminalId): Promise<void>
```

型: [Agent](core.md) · [RemoteStream](typert.md) · [SessionId](core.md)

ソース: [`packages/api/terminal-controller/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/terminal-controller/src/index.ts)

<a id="ctxworkspacecontroller--workspacecontroller"></a>

### `ctx.workspaceController` — `WorkspaceController`

生成された `ctx.remote.workspace` 名前空間を支える Host サービス。

```ts cordis-catalog
/**
 * Create or idempotently resolve one Workspace over an existing directory.
 * @param request - directory path to register.
 * @returns the Workspace and whether this call created it.
 */
@Remote('create') create(request: WorkspaceCreateRequest): Promise<WorkspaceCreateValue>

/**
 * Initialize or reuse the default Workspace during first-use startup. The
 * directory name is fixed, so the Host never renames or relocates an
 * existing default; its initial title is that same name, which browser
 * consumers label in the reader's language.
 * @param signal - caller lifetime; cancels native directory lookup.
 * @returns the durable Workspace, or undefined when first-use initialization is ineligible; creates no Session or message.
 */
@Remote('initializeDefault') async initializeDefault(signal: AbortSignal): Promise<WorkspaceValue | undefined>

/**
 * Rename one Workspace to a unique non-blank title.
 * @param request - Workspace identity and proposed title.
 * @returns the updated Workspace projection.
 */
@Remote('rename') rename(request: WorkspaceRenameRequest): Promise<WorkspaceValue>

/**
 * Remove one Workspace registration while retaining files and Sessions.
 * @param request - Workspace identity to remove.
 * @returns deletion confirmation.
 */
@Remote('delete') delete(request: WorkspaceDeleteRequest): Promise<WorkspaceDeleteValue>

/**
 * Move one Workspace within the registry display order.
 * @param request - moved Workspace and optional anchor.
 * @returns the complete resulting Workspace order.
 */
@Remote('insertBefore') insertBefore(request: WorkspaceInsertBeforeRequest): Promise<WorkspaceOrderValue>

/**
 * Move one accounted Session within a Workspace.
 * @param request - Workspace, Session, and optional anchor identities.
 * @returns the updated Workspace projection.
 */
@Remote('insertSessionBefore') insertSessionBefore(request: WorkspaceInsertSessionBeforeRequest): Promise<WorkspaceValue>

/**
 * Hide one known Session from Workspace grouping surfaces.
 * @param request - Session identity to archive.
 * @returns the complete resulting archive set.
 */
@Remote('archiveSession') archiveSession(request: WorkspaceArchiveSessionRequest): Promise<WorkspaceArchiveValue>

/**
 * Restore one archived Session to Workspace grouping surfaces.
 * @param request - Session identity to unarchive.
 * @returns the complete resulting archive set.
 */
@Remote('unarchiveSession') unarchiveSession(request: WorkspaceUnarchiveSessionRequest): Promise<WorkspaceArchiveValue>

/**
 * Surface one known unarchived Session ahead of unpinned Sessions.
 * @param request - Session identity to pin.
 * @returns the complete resulting pin set, most recently pinned first.
 */
@Remote('pinSession') pinSession(request: WorkspacePinSessionRequest): Promise<WorkspacePinValue>

/**
 * Remove one Session's pin without changing its saved Session order.
 * @param request - Session identity to unpin.
 * @returns the complete resulting pin set, most recently pinned first.
 */
@Remote('unpinSession') unpinSession(request: WorkspaceUnpinSessionRequest): Promise<WorkspacePinValue>

/**
 * Stream a complete Workspace baseline followed by ordered increments. A
 * non-owner receives only the Workspaces and Sessions `allows()` admits.
 * When a delivered Workspace is no longer admitted at its next change or
 * the next order change, the stream sends a `replace` frame with the
 * complete filtered state instead of a `remove`, because a Client never
 * resurrects a removed Workspace; a `remove` frame reports a deletion and
 * reaches only a Client that received that Workspace.
 * @param signal - generation cancellation.
 * @returns baseline followed by ordered Workspace increments.
 */
@Remote({ mode: 'stream' }) async *follow(signal: AbortSignal): AsyncIterable<WorkspaceFollowFrame>
```

ソース: [`packages/api/workspace-controller/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/workspace-controller/src/index.ts)

<a id="ctxworkspacefiles--workspacefiles"></a>

### `ctx.workspaceFiles` — `WorkspaceFiles`

構成済みファイルシステムを介した Host の Remote ファイル読み取りとワークスペース・ディレクトリの観測。

```ts cordis-catalog
/**
 * Read one page of lines from a UTF-8 file readable by the filesystem backend.
 * @param workspaceFileScope - header-derived workspace root for the Session identity on the wire.
 * @param path - absolute path or path relative to the workspace root; files outside it are allowed.
 * @param range - the line window; omitted fields take the page defaults.
 * @param signal - caller cancellation.
 * @returns the page, the file's version at the stat before it, and whether it reaches the last line.
 */
@Remote async read( workspaceFileScope: WorkspaceFileScope, path: string, range: WorkspaceFileRange, signal: AbortSignal, ): Promise<WorkspaceFileText>

/**
 * Read a complete regular file or one byte range without text decoding.
 * @param workspaceFileScope - header-derived workspace root for the Session identity on the wire.
 * @param path - target path, absolute or workspace-relative; relative to the base file's directory when provided.
 * @param options - optional base file and range; without a range the complete-file cap applies.
 * @param signal - caller cancellation.
 * @returns native bytes with the file's version and size at the preceding stat, byte offset, and EOF marker.
 */
@Remote async readBytes( workspaceFileScope: WorkspaceFileScope, path: string, options: WorkspaceByteReadOptions, signal: AbortSignal, ): Promise<WorkspaceFileBytes>

/**
 * Report one regular file's identity, version, and size without its content.
 * @param workspaceFileScope - header-derived workspace root for the Session identity on the wire.
 * @param path - absolute path or path relative to the workspace root; files outside it are allowed.
 * @param signal - caller cancellation.
 * @returns the file's absolute path, current version, and byte size.
 */
@Remote async stat(workspaceFileScope: WorkspaceFileScope, path: string, signal: AbortSignal): Promise<WorkspaceFileStat>

/**
 * List the direct children of one directory inside the Session's workspace.
 * @param workspaceFileScope - header-derived workspace root for the Session identity on the wire.
 * @param path - workspace path, absolute or relative to the workspace root.
 * @param signal - caller cancellation.
 * @returns the directory's children in the backend's stable name order, bounded by the entry cap.
 */
@Remote async list(workspaceFileScope: WorkspaceFileScope, path: string, signal: AbortSignal): Promise<WorkspaceDirectoryListing>

/**
 * Watch one file or a directory's direct entries in the Session's filesystem.
 * Files use the backend's read authority; directories remain workspace-scoped.
 * @param workspaceFileScope - header-derived workspace root for the Session identity on the wire.
 * @param path - target path; the Host determines its type and confines directories to the workspace.
 * @param signal - generation cancellation.
 * @returns `ready` once the target watch is active, then current metadata for queued and live invalidations.
 * @throws RemoteError when watching is unavailable or a directory is outside the workspace.
 */
@Remote({ mode: 'stream' }) changes(workspaceFileScope: WorkspaceFileScope, path: string, signal: AbortSignal): AsyncIterable<WorkspaceFileWatchFrame>
```

ソース: [`packages/api/workspace-files/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/workspace-files/src/index.ts)

<a id="ctxworkspaceregistry--workspaceregistry"></a>

### `ctx.workspaceRegistry` — `WorkspaceRegistry`

永続的なワークスペース・レジストリ。起動時は `sessionPersistence` を待ち、正規化 cwd のヘッダー索引を 1 つ構築し、サービスがアクティブになる前に 1 回限りの履歴ブートストラップを完了する。永続化の依存は必須であり、利用できないピアが空の履歴と誤認されて初期化済みマーカーをコミットすることは決してない。

```ts cordis-catalog
/**
 * Create or reuse a workspace for an existing directory. The fully qualified
 * path is canonicalized through `fs.realpath`; a relative, nonexistent, or
 * non-directory path rejects. Repeated calls for the same canonical path
 * return the existing entity without changing its title.
 * A newly created workspace is prepended to the durable registry order.
 * Different canonical paths may share a display title.
 * @param path - Existing directory to own, in a fully qualified path spelling.
 * @param title - Display title used only when a new record is created.
 * @returns the existing or newly durable workspace.
 */
async create(path: string, title?: string): Promise<Workspace>

/**
 * Initialize the default Workspace only while both the registry and Session
 * history are empty. Repeated requests reuse its durable identity; deleting
 * that registration permanently disables automatic creation.
 * @param resolveDirectory - resolve the absolute directory; called only for
 * eligible creation, inside the registry mutation queue. Missing directories
 * are created recursively before registration, and the initial title is the
 * requested directory's own final segment — not the canonical one, so a
 * symlink at that path does not retitle the Workspace after its target.
 * After resolution, caller cancellation does not roll back creation or registration.
 * @returns the initialized Workspace, or undefined when automatic creation is ineligible.
 */
initializeDefault(resolveDirectory: () => Promise<string>): Promise<Workspace | undefined>

/**
 * Look up a workspace by id.
 * @param id - Workspace id.
 * @returns the workspace, or `undefined` when unknown.
 */
get(id: WorkspaceId): Workspace | undefined

/**
 * Synchronous workspace projection in durable registry order. Every
 * entity's `sessionIds` getter is already filtered by the startup/live
 * canonical-cwd header index; this method performs no persistence reads.
 * @returns a fresh ordered array of workspace entities.
 */
list(): Workspace[]

/**
 * Delete one workspace registration while retaining its directory and every
 * session log. The durable order is updated before the table deletion; a
 * failed table write restores the prior order and keeps the entity
 * published. Unknown ids are an idempotent no-op for domain callers.
 * @param id - Workspace registration to remove.
 * @returns `true` when a record was deleted, `false` when it was unknown.
 */
delete(id: WorkspaceId): Promise<boolean>

/**
 * Move one workspace within the durable display order, DOM-insertBefore-like.
 * With an anchor it lands before that workspace; without one it appends.
 * @param id - Workspace to move.
 * @param beforeId - Workspace anchor; omitted appends.
 * @returns the complete committed workspace order.
 */
insertBefore(id: WorkspaceId, beforeId?: WorkspaceId): Promise<readonly WorkspaceId[]>

/**
 * Archive one session durably. The session must exist (live or in session
 * persistence); its workspace accounting — or lack of one — is irrelevant.
 * Without `stopActivity` the session must also be inactive: the
 * `workspace/session-activity` waterfall is asked once, and any reported
 * activity rejects with {@link WorkspaceActiveSessionError} before anything
 * is written. With `stopActivity` the archive is written without an
 * activity check, and the `workspace/session-stop` providers are then asked
 * to stop the session's work: the durable archive set is what a provider's
 * `agent/pre-step` gate reads, so every wake the stops induce is already
 * blocked. Archiving drops the session's pin in the same durable write
 * (pinning and archival are mutually exclusive). An already archived id
 * resolves without writing, asking, or stopping.
 * @param sessionId - The session to archive.
 * @param options - Whether running work is stopped instead of refusing.
 * @returns resolution after durability and, with `stopActivity`, after every stop request was issued.
 */
archiveSession(sessionId: SessionId, options: ArchiveSessionOptions = {}): Promise<void>

/**
 * Unarchive one session durably by dropping it from the registry-global
 * archive set; the accounting slot was never touched, so the session
 * returns to its recorded position. Unarchiving runs no session-existence
 * check because removing an id cannot introduce an unknown one, so an
 * entry whose session is gone still resolves. An id that is not archived
 * resolves without writing.
 * @param sessionId - The session to unarchive.
 * @returns resolution after durability.
 */
unarchiveSession(sessionId: SessionId): Promise<void>

/**
 * Pin one session durably, prepending it to the registry-global pin set.
 * The session must exist (live or in session persistence) and must not be
 * archived. An already pinned id resolves without writing or reordering.
 * @param sessionId - The session to pin.
 * @returns resolution after durability.
 */
pinSession(sessionId: SessionId): Promise<void>

/**
 * Unpin one session durably by dropping it from the registry-global pin
 * set. Unpinning runs no session-existence check because removing an id
 * cannot introduce an unknown one, so an entry whose session is gone still
 * resolves. An id that is not pinned resolves without writing.
 * @param sessionId - The session to unpin.
 * @returns resolution after durability.
 */
unpinSession(sessionId: SessionId): Promise<void>

/**
 * Resolve by canonical directory path without creating or mutating a
 * workspace. A missing path rejects during `realpath`; an existing unowned
 * directory returns `undefined`.
 * @param path - Existing directory path in a fully qualified spelling.
 * @returns the workspace owning the canonical path, when one exists.
 */
async resolveByPath(path: string): Promise<Workspace | undefined>
```

型: [SessionId](core.md)

ソース: [`packages/workspace/workspace/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/workspace/workspace/src/index.ts)

<a id="workspace-events"></a>

### `workspace/*` イベント

<a id="workspacesession-activity--waterfall"></a>

#### `workspace/session-activity` — waterfall

アーカイブされる前に、セッションのためにまだ何が動いているかを構成済みのプロバイダーに尋ねる。リスナーは自身の SessionActivity エントリーを `next()` の結果の先頭に追加する。レジストリの最も内側のコールバックは空のリストを返すため、プロバイダーのない構成では自由にアーカイブできる。空でない結果は書き込みなしでアーカイブを拒否する。

```ts cordis-catalog
/**
 * Ask the composed providers what still runs for a session before it is
 * archived. A listener prepends its own {@link SessionActivity} entries to
 * the result of `next()`; the registry's innermost callback returns an
 * empty list, so a composition without providers archives freely. Any
 * non-empty result refuses the archive without a write.
 * @param request - the session about to be archived.
 * @param next - delegate to the remaining providers.
 * @mode waterfall
 */
'workspace/session-activity'( request: SessionActivityRequest, next: () => Promise<readonly SessionActivity[]>, ): Promise<readonly SessionActivity[]>
```

ソース: [`packages/workspace/workspace/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/workspace/workspace/src/index.ts)

<a id="workspacesession-stop--parallel"></a>

#### `workspace/session-stop` — parallel

呼び出し元が `stopActivity` 付きでアーカイブしたため、セッションの実行中の作業を停止する。これがディスパッチされる時点でアーカイブ集合は永続化されている。各プロバイダーは自身のファミリー（turn、その subagent の子孫、所有する job、アクティブな schedule）を、ユーザー自身の停止操作が使うのと同じキャンセル経路を通じて停止する。したがってセッション・ログは開いているすべての turn を正常に終了し、後のアンアーカイブで会話を続けられる。リスナーは実行中の作業の決着を待たずに停止要求を発行する。リスナーは自身の永続化バリアを待ってもよい。拒否はレジストリによってログに記録され、アーカイブを取り消さない。

```ts cordis-catalog
/**
 * Stop a session's running work because the caller archived it with
 * `stopActivity`; the archive set is durable when this dispatches. Each
 * provider stops its own families — cancelling a turn, its subagent
 * descendants, owned jobs, or active schedules — through the same cancel
 * paths the user's own stop actions use, so the session log ends every
 * open turn regularly and a later unarchive can continue the
 * conversation. Listeners issue their stop requests without waiting for
 * running work to settle; a listener may await its own durability
 * barrier. A rejection is logged by the registry and does not undo the
 * archive.
 * @param request - the session being archived.
 * @mode parallel
 */
'workspace/session-stop'(request: SessionActivityRequest): Promise<void> | void
```

ソース: [`packages/workspace/workspace/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/workspace/workspace/src/index.ts)
<!-- END GENERATED cordis-surface -->
