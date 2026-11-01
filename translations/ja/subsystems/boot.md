# プロファイル管理

日本語 | [English](../../../docs/subsystems/boot.md) | [中文](../../../docs/subsystems/boot.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/boot.md](../../../docs/subsystems/boot.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

[boot パッケージ・グループ](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/README.md)は、ランチャーが提供するプロファイル・アクセスと plugin マネージャーを所有する。[Plugin Manager](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/plugin-manager/README.md)は、永続化、リロード、パッケージ操作の挙動を記述する。

## 管理レコード

`PluginEntryId` は 1 つの Loader エントリーを識別する。呼び出し元は patch id を自前で構築するのではなく、`listPlugins` から取得する。

`PluginInfo` はモジュールの同一性、実効的な有効化状態、fiber フェーズ、任意の表示用 `meta` を運び、さらに一意な `patchId` または `readOnlyReason` を持つ。

`BundleInfo.official` は、プロジェクトが保守する出荷済みのオプション bundle とオンデマンド・カタログ・エントリーを、インストールとは独立に識別する。`availability` は、読み取り可能なインストール提供パッケージまたはプロファイル・パッケージ、あるいは `missing` を識別する。オンデマンド・カタログ・エントリーは宣言されていない推移的コピーを除外する。`installed` はプロファイルの依存宣言を記録する。オンデマンドの `installTarget` は、ホストが導出した正確な spec とバージョンを運ぶ。`BundleInfo` はさらに、パッケージ名、任意のインストール済みバージョン、選択された有効化状態、削除の可否、更新の適格性（`updatable`）、任意の解決エラーまたは未サポート・プラットフォーム・エラー、そしてプロファイル自身の依存が供給しインストールは供給しない bundle については `source`（`pnpm add` が受け入れる spec としてのその依存）を運ぶ。任意の `meta` と各 `BundleRowInfo.meta` は表示テキストまたはメタデータ診断を含む。Client は描画時に言語を選択する。

`InstallBundleOptions.saveExact` は共有パッケージ・インストーラーに `--save-exact` を渡す。`InstallBundleOptions.enabled` のデフォルトは true である。false の場合、bundle レイヤーを選択せずにインストールする。`approvedBuilds` は、供給された保留中のパッケージ名に対して、インストール前に永続的なスクリプト許可を与える。`registry` は最初に問い合わせるレジストリを名指す。省略時は設定済みのものを指す。

`BundleDetail` はプロファイル bundle の長い説明である。`{ status: 'found' }` とその `origin`（`locale` または `readme`）、パッケージ内の `path`、および `markdown`、あるいは `{ status: 'none' }` を持つ。[`readBundleDetail`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/plugin-manager/README.md#bundle-detail-documents) が順序と画像ルートを記述する。`NpmFileSource` は公開 npm ファイル CDN である `jsdelivr`、`unpkg`、`npmmirror` のいずれかを名指す。読み取りの `npmFileSources` は最初に問い合わせるものを列挙する。

`PluginRegistries` は、設定済みの最初のレジストリ、pnpm 自身の設定が名指すものに対する `null`、その後ろに問い合わせるフォールバック、および pnpm 自身の設定が名指す URL または未読み取りの間は `null` である `resolved` を運ぶ。`InspectOptions.registry` は検索が最初に問い合わせるレジストリを名指す。

`ChangeResult.changed` はディスク編集を `application` とは独立に報告する。`applied`、`restart-required`、`overridden`、`failed` である。任意の `error` はローカライズ可能なコードと外部診断を運ぶ。`packageResult` は実行の終了ステータス、境界付きの出力、切り詰めフラグ、完全な診断ログのパスを記録し、さらにマネージャーが出力を止めた実行を終了した場合や GitHub 接続チェックが期限に達した場合には `timedOut` を記録する。終了された実行は、プロセスがステータスを返さなかったため、1 に強制されたステータスを記録する。終了された実行は、シグナルが残した終了ステータスにかかわらず `timeout` に分類される。したがってインストールと削除は成功ではなく失敗を報告し、それ以上のレジストリは問い合わせられない。`pendingBuilds` はプロファイル全体で未決定のパッケージを列挙し、`approvedBuilds` はこの操作が許可を与えた名前を記録し、`registries` はインストールが問い合わせたレジストリを順に列挙する。`bundle` と `version` は、完了したインストールが追加したパッケージとそのマニフェスト・バージョンを名指す。`failedAt` は、最後に失敗した実行が、問い合わせたレジストリに到達できなかったのか、git または tarball の spec を取得するホストに到達できなかったのかを示す。

## マーケット・レコード

[Plugin Market](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/plugin-market/README.md)はインデックス形式 1 を読み取り、[`dsh-plugin-market-index`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/util/plugin-market-index/README.md) によって検証される。

`MarketIndexSummary` は、設定済みの 1 つのインデックス URL と、最後に成功した読み取りの `title` および `description`、そして最後の読み取りが失敗した場合にはその `error` を運ぶ。

`MarketQuery` はインデックスの 1 ページを選択する。あいまい検索の `text`、`category`、`labels`（すべて存在する必要がある）、`MarketSort`、`offset`、最大 100 の `limit`、キャッシュ済みドキュメントを使わず再取得する `refresh`、そして 1 つのドキュメントのページを続ける `fetchedAt` である。`MarketSearchResult` はインデックス URL、`title`、任意の `description`、マーケットの `metadata`、ドキュメントの `fetchedAt`、ページの `entries`、一致件数の `total`、`nextOffset` を運ぶ。各 `MarketListingEntry` は、デフォルトでインストールされるバージョンである `defaultVersion` と、パッケージ名に対するプロファイルの状態である `installation` を追加する。その各 `versions` は、`installBundle` 用の正確な spec と任意の最初のレジストリである `install`、およびバージョン免除を含む `engines.dsh` の評価である `compatibility` を追加する。

`PluginDetail` はマーケット・エントリーの長い説明である。`{ status: 'found' }` とその `origin`（`locale`、`readme`、`repository`）、`markdown`、および `document` id（その画像は `GET /api/plugin-market/detail-file` が提供する）、あるいは `{ status: 'none' }` を持つ。[plugin market はそのソースを読む順序](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/plugin-market/README.md#plugin-detail-documents)を記述する。

## プロファイルの調整

<a id="profile-reconciliation"></a>

```ts type-equiv
/** Outcome of {@link Hmr.reconcileProfile}. */
interface ProfileReconciliation {
  /** Diagnostics for Loader entries left inactive. */
  warnings: string[]
  /** Loaded modules whose files changed on disk and whose previous code keeps running until a restart. */
  pending: PendingModule[]
  /**
   * Why every change waits for a restart: `module-comparison` when module caches cannot be inspected or their files
   * compared, even when no URLs are known; `package-moved` when the runtime resolution cannot publish a moved package.
   */
  restartReason?: 'module-comparison' | 'package-moved'
}
```

```ts type-equiv
/** A changed module that loads only after a restart. */
interface PendingModule {
  url: string
  /**
   * `host-module`: the process entry point's static graph imports it. `unmanaged-plugin`: it is imported by a plugin
   * that runs outside a traversable Loader entry. `module-comparison`: loaded modules could not be compared with disk.
   */
  reason: 'host-module' | 'unmanaged-plugin' | 'module-comparison'
}
```

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxactioncards--actioncards"></a>

### `ctx.actionCards` — `ActionCards`

カード・ツール、kind レジストリ、Session 投影、ユーザー操作、ステータス通知。

```ts cordis-catalog
/**
 * Register one action kind for the caller's lifetime.
 * @param kind - the kind's skill, details schema, verbs and handler.
 * @returns disposer removing the kind; actions on its cards are then unavailable.
 * @throws when another kind already registered the same action.
 */
register<Details>(kind: ActionCardKind<Details>): () => void

/**
 * Record a card decision in the Session log; the next model request reports it.
 * @param agent - agent whose Session owns the card.
 * @param decision - the card's new state.
 * @throws when no kind ever registered the decision's action with this service, or its phase is one the projection derives.
 */
record(agent: Agent, decision: ActionCardDecision): void

/**
 * Read the cards of one Session.
 * @param session - a Session whose projections are prepared.
 * @returns proposals and decisions by call id.
 */
view(session: Session): ActionCardView

/**
 * Apply a user action to a logged card. Shared checks run first, then the kind reconciles the card; `dismiss` is
 * recorded here and every other verb goes to the card's kind. `dismiss` also works on an unstarted card whose kind
 * is no longer registered. Concurrent actions on one card share one operation.
 * @param agent - live root resolved from the displayed Session.
 * @param callId - the card's `propose_action` call id.
 * @param verb - `dismiss` or one of the kind's verbs.
 * @param payload - kind-defined fields such as a chosen location.
 * @returns acceptance of background work, the decision in effect, or a refusal.
 */
@Remote act(agent: Agent, callId: ToolCallId, verb: string, payload?: Record<string, JsonValue>): Promise<ActionCardResult>
```

型: [Agent](core.md) · [Session](session.md) · [ToolCallId](core.md)

ソース: [`packages/experimental/action-cards/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/action-cards/src/index.ts)

<a id="ctxconfigeditor--configeditor"></a>

### `ctx.configEditor` — `ConfigEditor`

完全な生の設定を永続化し、通常の Loader 経路を通じて適用する。

```ts cordis-catalog
/** Addressable profile rows; nested Includes have independent configuration ownership.
 * @returns Active entries with unique profile patch ids.
 */
entries(): Entry[]

/** Read inherited and explicit profile values for the active entries.
 * @returns Detached layer values alongside their Loader entries.
 */
configuration(): Array<{ entry: Entry; inherited: Record<string, unknown>; override: Record<string, unknown> }>

/** Validate, persist, and reconcile a plugin's next config; ordinary fields keep normal lifecycle rules.
 * @param entry Current Loader entry, also used to detect replacement during the write.
 * @param change Derive a raw config from the current entry and its inherited layer.
 * @returns Fulfillment after Loader reconciliation completes.
 */
async edit( entry: Entry, change: (current: Record<string, unknown>, inherited: Record<string, unknown>) => Record<string, unknown>, ): Promise<void>
```

ソース: [`packages/boot/config-editor/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/config-editor/src/index.ts)

<a id="ctxheadlessstartup--headlessstartupvalues"></a>

### `ctx.headlessStartup` — `HeadlessStartupValues`

ランナーの行が HEADLESS_STARTUP_SERVICE から読み取るもの。

ソース: [`packages/bundle/headless/src/startup.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/headless/src/startup.ts)

<a id="ctxhmr--hmr"></a>

### `ctx.hmr` — `Hmr`

Cordis 互換のモジュール設定とイベントを備えたホット・リロード・サービス。

```ts cordis-catalog
/** Serialize a caller-owned mutation with all automatic reload paths.
 * @param operation Work that must not overlap module or configuration replacement.
 * @returns The operation result after its asynchronous work completes.
 */
runExclusive<T>(operation: () => Promise<T>): Promise<T>

/** Watch a configuration path through the same queue as module replacement.
 * @param filename Absolute path, which may not exist yet.
 * @param refresh Rebuilds configuration from its current files and awaits Loader completion.
 * @returns Disposer closing this registration and waiting for its pending refresh.
 */
async watchConfig(filename: string, refresh: () => Promise<void>): Promise<() => Promise<void>>

/** Read direct module dependency URLs from the active Node loader.
 * @param url Module URL.
 * @returns Linked module URLs, or an empty list for an uncached module.
 */
async getLinked(url: string): Promise<string[]>

/**
 * Recompose the profile from its manifest and patch files, replacing running plugins whose loaded files below the
 * profile directory or its linked package targets changed on disk, and publishing its runtime resolution. Changed
 * modules are evicted and the Loader entries running them stop; app-boot's {@link recomposeProfile} then publishes the
 * resolution, applies the generation, starts the stopped entries with the current files and configuration,
 * imports entries whose import failed again, and judges activation. Afterwards the modules below the mappings the
 * recomposition dropped are evicted, so a removed or deselected package is imported afresh when it returns. Callers run
 * inside {@link runExclusive}.
 * @param requiredIds Entry ids whose inactive state is reported as a failure even when it predates this call.
 * @returns Inactive-entry diagnostics, the changed modules that load after a restart, and a profile-wide restart reason.
 * @throws The patch read, publication, preparation, apply, or activation failure of the recomposition, including a
 * replacement plugin that fails to load or start, whose previous code no longer runs.
 */
async reconcileProfile(requiredIds: readonly string[] = []): Promise<ProfileReconciliation>
```

ソース: [`packages/boot/hmr/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/hmr/src/index.ts)

<a id="ctxpluginmanager--pluginmanager"></a>

### `ctx.pluginManager` — `PluginManager`

プロファイル・ファイルを管理し、宣言されたリロード・ライフサイクルを適用する。

```ts cordis-catalog
/**
 * Read a bundle's long description: the Markdown file its locale `meta.detail` names in `language`, otherwise its
 * README. An installed bundle is read from its directory without network access; an official bundle the installation
 * offers before it is installed is read from the npm file CDNs at its offered version. Nothing is cached.
 * @param name - the bundle's package name, as `listBundles()` lists it.
 * @param language - the display language, such as `zh-CN`; documents fall back through its prefixes to English.
 * @param options - `npmFileSources` names npm file CDNs to ask first for an offered bundle, in order, such as the one
 *   a Client measured as fastest; configured CDNs it omits follow, and CDNs that are not configured are not asked.
 * @returns the document with its origin and path, or `none`.
 * @throws {RemoteError} `plugin-manager/unknown-bundle`, `plugin-manager/invalid-language`, or
 *   `plugin-manager/detail-unavailable`.
 */
@Remote async readBundleDetail(name: string, language: string, options?: { npmFileSources?: NpmFileSource[] }): Promise<BundleDetail>

/** Read exact plugin-version exemptions saved in this profile.
 * @returns Accepted package-name@version keys with the runtime versions they may run on, and any
 * record or file problem the reader rejected, which the caller reports instead of failing.
 */
@Remote listVersionExemptions(): { exemptions: Record<string, string[]>; warnings: string[] }

/** Grant or revoke one exact plugin/runtime exemption and reevaluate live plugins.
 * @param packageVersion Exact manifest package name followed by @ and its version; never an installation spec or alias.
 * @param runtimeVersion Exact current DSH version for grants; revocation may name a previous runtime.
 * @param enabled Whether to grant rather than revoke the exemption.
 * @param acceptRisk Required true for grants after the user accepts possible crashes and data loss.
 * @returns Saved and runtime outcomes. Startup-only profiles require restart.
 */
@Remote setVersionExemption(packageVersion: string, runtimeVersion: string, enabled: boolean, acceptRisk?: boolean): Promise<ChangeResult>

/** Read current plugins, including why a row cannot be changed through the profile patch.
 * @returns Current runtime entries with persistent patch targets.
 */
@Remote async listPlugins(): Promise<PluginInfo[]>

/** Read installed, installation-provided, and offline Official catalog bundles, plus selected non-bundle names.
 * A dependency without a bundle patch is listed, as a `not-bundle` problem, only while it is selected.
 * @returns Package versions, manifest descriptions, the installable spec of profile dependencies, rows, optional
 * display metadata, activation selections, whether the installation offers the bundle, and removal availability.
 */
@Remote listBundles(): Promise<BundleInfo[]>

/** Read the registries this manager asks: the configured first one, its fallbacks in order, and what pnpm's own configuration names.
 * @returns The registries in pnpm's comparison form; null is the one pnpm's own configuration names, `resolved` as pnpm reads it now.
 */
@Remote async registries(): Promise<PluginRegistries>

/** Read what a spec names before installing it.
 * @param spec One package spec: a registry name, an absolute path, a git address, or a tarball.
 * @param options The registry asked first.
 * @param signal Ends a registry lookup early.
 * @returns The package the spec names, or why it is refused.
 */
@Remote async inspect(spec: string, options?: InspectOptions, signal?: AbortSignal): Promise<PluginSpecInspection>

/** Read the published versions newer than an installed registry package's version, from the first registry that answers.
 * Official bundles follow the DSH version and are not listed.
 * @param name Installed profile dependency name.
 * @param options The registry asked first; absent, the one the package was installed from in this process, else the configured one.
 * @param signal Ends the lookups early.
 * @returns At most `versionListLimit` newer versions, newest first, with their tags, deprecation, bundle declaration,
 * and DSH compatibility, and the registry that answered; or why no list could be read, with every registry asked.
 */
@Remote async listVersions(name: string, options?: ListVersionsOptions, signal?: AbortSignal): Promise<PluginVersionList>

/**
 * Replace an installed registry bundle with a newer published version. The target is checked on the registry first:
 * it must exist, declare a bundle, and accept the running DSH version, or nothing changes. With HMR the installed
 * version is removed, so its plugins stop, and the target is installed with the same selection; when that
 * installation fails or is cancelled, the removed version is installed again from the registry it came from.
 * Without HMR the target is installed over the running version and the result reports the restart.
 * @param name Installed profile dependency name.
 * @param version Exact published version newer than the installed one.
 * @param options The registry asked first, as for `listVersions`, and the request id naming the progress and the
 * target installation's logs and cancellation.
 * @returns The versions, the last step that ran with its result, and the reinstallation when one ran.
 */
@Remote async updateBundle(name: string, version: string, options?: UpdateBundleOptions): Promise<PluginUpdateResult>

/** Persist a plugin entry's desired enablement and apply it on live profiles.
 * @param id Loader entry identity returned by listPlugins.
 * @param enabled Whether the plugin should run.
 * @returns Saved and runtime outcomes, including higher-priority overrides.
 */
@Remote setPluginEnabled(id: PluginEntryId, enabled: boolean): Promise<ChangeResult>

/** Select or remove a bundle layer while retaining installed dependencies.
 * @param name Bundle package name.
 * @param enabled Whether the bundle contributes its patch layer.
 * @returns Persisted and runtime outcomes.
 */
@Remote setBundleEnabled(name: string, enabled: boolean): Promise<ChangeResult>

/**
 * Install a package using the same pnpm implementation as dsh plugin. GitHub
 * repositories get a connection check bounded by githubConnectionTimeoutMs before pnpm starts;
 * only network failures or timeouts stop installation, and a stop streams the check's diagnostic
 * as the installation's install-log run under its request id, while pnpm owns authentication and transport fallback. An
 * HTTP(S) Git URL on another host is installed over SSH instead when access probes bounded by gitTransportTimeoutMs
 * find that only SSH can read it. A run
 * that fails, is cancelled, or adds a package without a bundle patch restores
 * `package.json` and `pnpm-lock.yaml` as they were; downloaded files can stay.
 * @param spec One package spec, including local paths relative to the invocation directory.
 * @param options Whether to activate the installed bundle (defaults to true), the request id a cancellation names,
 * the pending build scripts to allow, whether to save an exact dependency, and the registry asked first.
 * @returns Package-manager diagnostics, the registries asked, and the observed activation outcome.
 */
@Remote installBundle(spec: string, options?: InstallBundleOptions): Promise<ChangeResult>

/** Recover the result of an active installation without cancelling it.
 * @param requestId The id supplied when installation started.
 * @returns The installation's outcome after it settles, or null if no active request has that id.
 * Completed results are not retained; null establishes neither success nor cancellation.
 */
@Remote async waitForInstall(requestId: PluginInstallRequestId): Promise<ChangeResult | null>

/** Stop an installation this manager owns and wait until its files are back.
 * @param requestId The id the installation was started with.
 * @returns `cancelled` once the Git check or pnpm exited and the files are restored, `too-late` once the bundle is being
 * applied, `not-running` for any other id.
 */
@Remote async cancelInstall(requestId: PluginInstallRequestId): Promise<PluginInstallCancellation>

/** Unload and remove a profile-owned bundle dependency through dsh plugin's pnpm path; a selected name no
 * dependency holds is only deselected.
 * @param name Installed dependency or selected bundle name.
 * @returns Removal diagnostics and the remaining profile state.
 */
@Remote removeBundle(name: string): Promise<ChangeResult>
```

ソース: [`packages/boot/plugin-manager/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/plugin-manager/src/index.ts)

<a id="ctxpluginmarket--pluginmarket"></a>

### `ctx.pluginMarket` — `PluginMarket`

設定済みのマーケット・インデックスを取得、検証、キャッシュし、エントリーにプロファイルのインストール情報を注釈する。

```ts cordis-catalog
/**
 * List the configured index sources without fetching them.
 * @returns every source in order, with the title of its last successful read and, when the last read failed, its error.
 */
@Remote listIndexes(): MarketIndexSummary[]

/**
 * Replace the configured index sources through Settings, without remounting this service.
 * `plugin-market/changed` announces the committed list. A URL already configured is kept as written, so a client can
 * add or remove sources beside one that `listIndexes` reports as unusable.
 * @param urls - index URLs in display order.
 * @throws {RemoteError} `plugin-market/invalid-index-url` for an invalid new URL or a repeated URL;
 * `plugin-market/settings-unavailable` without Settings or a profile entry.
 */
@Remote async setIndexes(urls: string[]): Promise<void>

/**
 * Return one page of the plugins of a configured index that match a query. Concurrent reads of a URL share one
 * fetch; a successful document is reused for `cacheTtlMs` unless `refresh` is set. Matching and ordering read only
 * the document; installation status and compatibility are computed for the returned page on every call.
 * @param url - a configured index URL.
 * @param query - text, filters, order, page, and whether to bypass the cached document.
 * @returns the index header, the page's plugins with each plugin's default version and installation status and each
 *   version's compatibility and install arguments, the number of matches, and the next page's offset.
 * @throws {RemoteError} `plugin-market/invalid-query`, `plugin-market/unknown-index`, `plugin-market/invalid-index-url`,
 *   `plugin-market/fetch-failed`, or `plugin-market/invalid-index`.
 */
@Remote async search(url: string, query: MarketQuery): Promise<MarketSearchResult>

/**
 * Read one plugin of a configured index, using the cached document as `search` does.
 * @param url - a configured index URL.
 * @param name - the plugin's package name.
 * @returns the plugin with its default version, installation status, and annotated versions; undefined when the
 *   index does not list it.
 * @throws {RemoteError} `plugin-market/unknown-index`, `plugin-market/invalid-index-url`, `plugin-market/fetch-failed`,
 *   or `plugin-market/invalid-index`.
 */
async entry(url: string, name: string): Promise<MarketListingEntry | undefined>

/**
 * Read the long description of one listed version of a plugin market entry, from the version's source: the npm file
 * CDNs for a public npm version, or its pinned Git commit. The Markdown file the package's locale `meta.detail` names
 * in `language` comes first, then the package's README, then the README of the entry's `repository`, or for an npm
 * version without one, of its `package.json` `repository`, at the default branch. An npm version from another
 * registry has no readable package files, so only the entry's `repository` is read.
 * @param url - a configured index URL.
 * @param name - the plugin's package name.
 * @param version - one listed version.
 * @param language - the display language, such as `zh-CN`; documents fall back through its prefixes to English.
 * @param options - `refresh` reads again instead of reusing a result cached for `cacheTtlMs`; `npmFileSources` names
 *   npm file CDNs to ask first for an npm version, in order; configured CDNs it omits follow, and CDNs that are not
 *   configured are not asked.
 * @returns the document with its origin and image document id, or `none`.
 * @throws {RemoteError} the failures of `search`, `plugin-market/unknown-plugin` for an unlisted plugin or version,
 *   `plugin-market/invalid-query`, or `plugin-market/detail-unavailable`.
 */
@Remote async readEntryDetail( url: string, name: string, version: string, language: string, options?: { refresh?: boolean; npmFileSources?: NpmFileSource[] }, ): Promise<PluginDetail>
```

ソース: [`packages/boot/plugin-market/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/plugin-market/src/index.ts)

<a id="ctxpluginrecommendations--pluginrecommendations"></a>

### `ctx.pluginRecommendations` — `PluginRecommendations`

カード・テキスト用のカタログ検索と、`enable_official_plugin` カードの背後にある有効化。

```ts cordis-catalog
/**
 * Read the current catalog text for a logged card.
 * @param agent - Exact live root resolved by the Gateway from the Session id.
 * @param callId - The card's `propose_action` call id, never a package supplied by the browser.
 * @returns the plugin's title, description and selection, or a refusal when the card names no available official plugin.
 */
@Remote async describe(agent: Agent, callId: ToolCallId): Promise<RecommendationDetails>
```

型: [Agent](core.md) · [ToolCallId](core.md)

ソース: [`packages/experimental/plugin-recommendations/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/plugin-recommendations/src/index.ts)

<a id="ctxpluginregistryprobe--pluginregistryprobe"></a>

### `ctx.pluginRegistryProbe` — `PluginRegistryProbe`

Host 上で公開レジストリの応答を比較する。最初の選択は Client が所有する。

```ts cordis-catalog
/**
 * Race npm and npmmirror HTTPS ping responses through the Host's fetch proxy.
 * Concurrent readers share a probe; a winner cancels and awaits the other request.
 * @returns the first registry with a successful response, or null when disabled or neither responds successfully; results are cached.
 * @throws rejects when the service has been unloaded.
 */
@Remote async fastest(): Promise<string | null>
```

ソース: [`packages/client/ui-plugin-manager/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-plugin-manager/src/index.ts)

<a id="ctxprofilecontext--profilecontext"></a>

### `ctx.profileContext` — `ProfileContext`

現在のプロファイルの事実。スケジューリングと変更は呼び出し元に属する。

ソース: [`packages/boot/app-boot/src/profile-context.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/app-boot/src/profile-context.ts)

<a id="app-boot-events"></a>

### `app-boot/*` イベント

<a id="app-bootconfig-reload--emit"></a>

#### `app-boot/config-reload` — emit

プロファイル・パッチが実行中の Loader ツリーに調整された。すべてのエントリー更新が決着し、新しい非アクティブ・エントリーは導入されなかった。差分は運ばない。リスナーは Loader エントリーを再読み取りする。

```ts cordis-catalog
/**
 * Profile patches were reconciled into the running Loader tree: every entry update settled and no new
 * inactive entry was introduced. Carries no diff; listeners re-read Loader entries.
 * @mode emit
 */
'app-boot/config-reload'(): void
```

ソース: [`packages/boot/app-boot/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/app-boot/src/index.ts)

<a id="hmr-events"></a>

### `hmr/*` イベント

<a id="hmrchange--emit"></a>

#### `hmr/change` — emit

監視対象のファイルにモジュール・ハンドラーも設定ハンドラーもない。

```ts cordis-catalog
/** A watched file has no module or configuration handler.
 * @mode emit
 * @param url Canonical file URL.
 */
'hmr/change'(url: string): void
```

ソース: [`packages/boot/hmr/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/hmr/src/index.ts)

<a id="hmrreload--emit"></a>

#### `hmr/reload` — emit

モジュールの置き換えがロードを終えた。

```ts cordis-catalog
/** Module replacements have finished loading.
 * @mode emit
 * @param reloads Replaced plugins and their module locations.
 */
'hmr/reload'(reloads: Map<Plugin, Reload>): void
```

ソース: [`packages/boot/hmr/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/hmr/src/index.ts)

<a id="plugin-manager-events"></a>

### `plugin-manager/*` イベント

<a id="plugin-managerchanged--emit"></a>

#### `plugin-manager/changed` — emit

プロファイルの plugin、bundle、または構成が変わった。マネージャーの操作が完了した。CLI や手編集の後に HMR のウォッチャーがマネージャー外で適用したパッチ生成は、ここでは何も通知しない。

```ts cordis-catalog
/**
 * The profile's plugins, bundles, or composition changed: a manager
 * operation completed. A patch generation applied outside the manager,
 * by HMR's watcher after a CLI or hand edit, announces nothing here.
 * @mode emit
 * @param change - what changed.
 */
'plugin-manager/changed'(change: PluginChange): void
```

ソース: [`packages/boot/plugin-manager/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/plugin-manager/src/types.ts)

<a id="plugin-managerinstall-log--emit"></a>

#### `plugin-manager/install-log` — emit

インストール実行の出力の 1 チャンク。実行が生成するそばからストリームされる。

```ts cordis-catalog
/**
 * One chunk of an installation run's output, streamed as the run produces it.
 * @mode emit
 * @param chunk - the chunk and the run it belongs to.
 */
'plugin-manager/install-log'(chunk: PluginInstallLogChunk): void
```

ソース: [`packages/boot/plugin-manager/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/plugin-manager/src/types.ts)

<a id="plugin-managerinstall-state--emit"></a>

#### `plugin-manager/install-state` — emit

インストールが Host のフェーズ間を移動した。`installing` はインストールが問い合わせるレジストリごとに 1 回、その試行のレジストリと位置とともに通知される。`cancelling` と `applying` は 1 回である。

```ts cordis-catalog
/**
 * An installation moved between its Host phases. `installing` is announced once per registry the
 * installation asks, with the attempt's registry and position; `cancelling` and `applying` once.
 * @mode emit
 * @param progress - the installation's request id and phase, with the attempt while installing.
 */
'plugin-manager/install-state'(progress: PluginInstallProgress): void
```

ソース: [`packages/boot/plugin-manager/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/plugin-manager/src/types.ts)

<a id="plugin-manageroperation--parallel"></a>

#### `plugin-manager/operation` — parallel

マネージャーの呼び出しが 1 回開始または終了した。サブスクライバーは管理を遅らせずに独立して実行される。

```ts cordis-catalog
/**
 * One manager invocation started or ended; subscribers run independently without delaying management.
 * @mode parallel
 * @param operation - Correlated operation facts; a start without an end has an unknown outcome.
 */
'plugin-manager/operation'(operation: PluginOperation): void | Promise<void>
```

ソース: [`packages/boot/plugin-manager/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/plugin-manager/src/types.ts)

<a id="plugin-managerupdate-state--emit"></a>

#### `plugin-manager/update-state` — emit

更新が次のステップに進んだ。対象の確認、インストール済みバージョンの削除、対象のインストール、または対象が失敗した後の削除済みバージョンの再インストールである。ステップごとに 1 回通知される。

```ts cordis-catalog
/**
 * An update moved to its next step: checking the target, removing the installed version, installing the target,
 * or reinstalling the removed version after the target failed. Announced once per step.
 * @mode emit
 * @param progress - the update's request id and step.
 */
'plugin-manager/update-state'(progress: PluginUpdateProgress): void
```

ソース: [`packages/boot/plugin-manager/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/plugin-manager/src/types.ts)

<a id="plugin-market-events"></a>

### `plugin-market/*` イベント

<a id="plugin-marketchanged--emit"></a>

#### `plugin-market/changed` — emit

設定済みのインデックス・ソースが変わった。

```ts cordis-catalog
/**
 * The configured index sources changed.
 * @mode emit
 */
'plugin-market/changed'(): void
```

ソース: [`packages/boot/plugin-market/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/plugin-market/src/types.ts)
<!-- END GENERATED cordis-surface -->
