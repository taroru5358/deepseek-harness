# プラグイン設定フォーム

日本語 | [English](../../../docs/subsystems/settings.md) | [中文](../../../docs/subsystems/settings.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/settings.md](../../../docs/subsystems/settings.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

[設定サービス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/settings/settings/README.md)は、アクティブなプロファイル・エントリーから揮発性の Config フィールドを投影する。[設定エディター](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/config-editor/README.md)は Cordis パッチを通じて編集を永続化する。ビジネス・コンシューマーは自身の Config 参照で `.get()` を読む。

## アイデンティティと値

フォーム名前空間は、アクティブなプロファイル内の一意にアドレス指定されたエントリーのローカル id である。複数のプラグイン・インスタンスは、エントリー id が異なれば別々のフォームを持つ。通常のフィールドは除外される。記述子は解決済みの値、継承値、明示的なプロファイル・オーバーライド、楽観的リビジョンを運ぶ。

## 編集

`update` は送信されたフィールドをマージする。`replace` は送信されたフィールドを適用する前に、ライブ・フィールドを継承設定にリセットする。`mutate` は個々のパスを指定し、クライアント応答に存在しないシークレットを保持する。すべての書き込みは完全な Config を検証し、永続化の前に陳腐化したリビジョンを拒否する。

`settings/document-updated` は、Loader 設定の変更後にフォーム記述子を無効化する。これは UI 通知であり、コンシューマーは登録ファクトをリフレッシュする必要があるときだけ `loader/volatile-update` を使う。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。dispatch モードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxsettings--settingsforms"></a>

### `ctx.settings` — `SettingsForms`

Config スキーマをフォームに投影し、任意のインスタンス・レベルの UI ポリシーを所有する。

```ts cordis-catalog
/** Register the calling plugin instance's page policy without changing its Config.
 * @param presentation Automatic-page policy for this instance; `auto` defaults to true.
 * @param owner Plugin instance the policy belongs to; defaults to the calling fiber.
 * @returns Disposer; register it with the calling plugin's effects.
 * @throws If this instance already has a registered policy.
 */
configure(presentation: { auto?: boolean }, owner: Fiber = this.ctx.fiber): () => void

/** Locate the profile patch for native editing.
 * @returns The existing profile patch path.
 */
prepareDocument(): Promise<string>

/** Read active plugin schemas and their live values.
 * @param options Redaction required for remote callers.
 * @returns Forms keyed by unique profile entry ids.
 */
describe(options?: SettingsDescribeOptions): SettingsDescriptor[]

/** Merge editable fields into an entry's config.
 * @param ns Profile entry id.
 * @param patch Fields to merge.
 * @param expectedRevision Revision returned by describe.
 */
async update(ns: string, patch: object, expectedRevision?: number): Promise<void>

/** Reset all live fields, then set the supplied fields; ordinary config is preserved.
 * @param ns Profile entry id.
 * @param section Complete form values.
 * @param expectedRevision Revision returned by describe.
 */
async replace(ns: string, section: object, expectedRevision?: number): Promise<void>

/** Apply field edits without restating redacted secrets; unsetting an array index removes its element.
 * @param ns Profile entry id.
 * @param ops Ordered form edits.
 * @param expectedRevision Revision returned by describe.
 */
async mutate(ns: string, ops: readonly SettingsPathOp[], expectedRevision?: number): Promise<void>
```

ソース: [`packages/settings/settings/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/settings/settings/src/index.ts)

<a id="ctxsettingscontroller--settingscontroller"></a>

### `ctx.settingsController` — `SettingsController`

生成された `ctx.remote.settings` 名前空間を支える Host サービス。すべての Remote 読み取りは `redactSecrets: true` を使うので、`role('secret')` のフィールドが応答に乗ることはない。書き込みは設定サービスのマージ、置き換え、パス指定操作を公開し、すべてのプロバイダー拒否をサービスのメッセージとともに `settings/conflict` または `settings/rejected` に分類する。

```ts cordis-catalog
/**
 * Describe every registered namespace for a configuration page: redacted
 * layered values plus the serialized schema the page renders its form from.
 * @returns provider writability, local-document presence, and one view per namespace.
 * @throws RemoteError when no settings provider is mounted.
 */
@Remote describe(): SettingsDescribeValue

/**
 * Merge a patch into one namespace's stored user section.
 * @param ns - namespace key to write.
 * @param patch - fields to merge into the user section.
 * @param expectedRevision - revision the caller read; `undefined` writes unconditionally.
 * @returns the namespace's redacted view after the write.
 * @throws RemoteError when the request is invalid, no provider is mounted, or the provider refuses the write.
 */
@Remote update( ns: string, patch: Record<string, JsonValue>, expectedRevision: number | undefined, ): Promise<SettingsNamespaceView>

/**
 * Replace one namespace's stored user section wholesale.
 * @param ns - namespace key to write.
 * @param section - complete replacement user section.
 * @param expectedRevision - revision the caller read; `undefined` writes unconditionally.
 * @returns the namespace's redacted view after the write.
 * @throws RemoteError when the request is invalid, no provider is mounted, or the provider refuses the write.
 */
@Remote replace( ns: string, section: Record<string, JsonValue>, expectedRevision: number | undefined, ): Promise<SettingsNamespaceView>

/**
 * Apply path-addressed edits to one namespace's user section, resolved against
 * the section as stored rather than against whatever the caller last read,
 * then answer with that namespace's new redacted view.
 * @param ns - namespace key to write.
 * @param ops - the edits to apply, in order.
 * @param expectedRevision - revision the caller read; `undefined` writes unconditionally.
 * @returns the namespace's redacted view after the write.
 * @throws RemoteError when the request is invalid, no provider is mounted, or the provider refuses the write.
 */
@Remote async mutate( ns: string, ops: SettingsPathOpView[], expectedRevision: number | undefined, ): Promise<SettingsNamespaceView>

/**
 * Materialize the provider-owned settings document and open it in a native text editor.
 * @param signal - caller lifetime; abort terminates preparation or the native command.
 * @returns confirmation after the native opener accepts the document.
 * @throws RemoteError when no document exists, preparation fails, or opening fails.
 */
@Remote async openSettingsDocument(signal: AbortSignal): Promise<SettingsDocumentOpenValue>
```

ソース: [`packages/api/settings-controller/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/settings-controller/src/index.ts)

<a id="settings-events"></a>

### `settings/*` イベント

<a id="settingsdocument-updated--emit"></a>

#### `settings/document-updated` — emit

1 つのプロファイル・エントリーのフォーム値、可用性、またはページ・ポリシーが変わった。フォーム・クライアントはそのスキーマ、解決済みの値、リビジョンを読み直す。

```ts cordis-catalog
/**
 * One profile entry's form values, availability, or page policy changed.
 * Form clients re-read its schema, resolved values, and revision.
 * @param ns Profile entry id.
 * @param revision The entry's new revision.
 * @mode emit
 */
'settings/document-updated'(ns: SettingsNamespace, revision: number): void
```

ソース: [`packages/settings/settings/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/settings/settings/src/types.ts)
<!-- END GENERATED cordis-surface -->
