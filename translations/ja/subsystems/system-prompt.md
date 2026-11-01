# システムプロンプトの組み立て

日本語 | [English](../../../docs/subsystems/system-prompt.md) | [中文](../../../docs/subsystems/system-prompt.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/system-prompt.md](../../../docs/subsystems/system-prompt.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

[システムプロンプトパッケージ](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/core/system-prompt)は、プロンプトコントリビューターと 1 回の組み立て呼び出しの間で交換されるデータを所有する。パッケージ [README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/system-prompt/README.md) は登録、順序付け、スコープ、レンダリング動作を文書化する。このページは、プラグインが実装または渡す正確なクロスパッケージ型を記録する。

ソース: [`packages/core/system-prompt/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/system-prompt/src/index.ts)。

## 組み立てコンテキスト

`AssembleContext` は 1 回の組み立てが解決するスコープ層を識別し、そのリクエストの明示的な制御シグナルを運んでよい。これはマージ拡張可能である: `dsh-agent` が任意の `agent` と `modelSelection` フィールドを追加する。`assembleContextFor(agent, signal)` は agent、スコープ、シグナルを設定する。ループはステップ用にキャプチャされ、再試行間で保持されるモデル選択を供給し、省略された推論努力はアダプターのデフォルトを使う。素の組み立てにはスコープもシグナルもない。

```ts type-equiv
/** Merge-extensible context for one prompt assembly. */
interface AssembleContext {
  /**
   * Scope whose providers and waterfall listeners participate. When absent,
   * only global providers and subject-less listeners participate.
   */
  scope?: ScopeKey
  /** Explicit control signal for the turn that requested this assembly, when any. */
  signal?: AbortSignal
}
```

`TOOL_APPLY_PATCH` は `TOOL_EDIT`（1300）と `TOOL_GLOB`（1400）の間の順序 1350 を取る。

## ツール provider の結果

`ToolProviderResult.schemas` は現在の組み立てに対してモデル可視な集合である。`knownNames` は provider の制限前の名前宇宙であり、設定された名前のタイポと、このスコープで意図的に隠された既知のツールを区別するために使われる。

```ts type-equiv
/** Tool schemas visible in one assembly and their pre-restriction name set. */
interface ToolProviderResult {
  /** The schemas this provider contributes to THIS assembly. */
  readonly schemas: readonly ToolSchema[]
  /** The pre-restriction name universe for config validation (defaults to `schemas`' names). */
  readonly knownNames?: readonly string[]
}
```

## プロンプトセクション

エクスポートされた `PERSONA_PREFIX_SECTION`（`deployment:persona-prefix`）と `PERSONA_SUFFIX_SECTION`（`deployment:persona-suffix`）は、グローバル設定とスコープ付きコントリビューションで共有されるスロットを名付ける。それらの `PromptSectionOrderName` エントリーは `DEPLOYMENT_PERSONA_PREFIX` と `DEPLOYMENT_PERSONA_SUFFIX` である。パッケージ [README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/system-prompt/README.md#configure-the-prompt) はそれらの配置とテンプレート設定を所有する。

`PromptSection` は読み取り専用の同一プロセス登録契約である。そのテキストは静的にするか、現在の組み立てコンテキストから解決できる。セクションは昇順の order、次にコードユニット名でソートされる。リポジトリのコントリビューターは `getSectionOrder()` を通じてサービス所有の名前付き割り当てを解決する。ランタイムコンテキストのコントリビューターは `getContextOrder()` を通じて独立した割り当てを解決する。協調的組み立ての後、1 つの有効な `complete` セクションが唯一のプロンプトセクションになる。agent-loop は組み立てられたセクションを `renderPrompt` でレンダリングし、そのテキストを `system/message` サーフェスノードとしてコミットする — 最初のステップでサーフェスノード 0 として追記され、その後、レンダリングされたテキストが変わるとインプレースで置換されるか、準備された呼び出しが `systemPromptUpdate: 'in-history'` を宣言している場合は、継続するシリーズの非空の更新に対してキャッシュされた履歴の後に追記される — したがってプロンプトはリクエストフィールドではなく派生履歴のメッセージとしてモデルに届く（[決定](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-09-02-system-prompt-as-surface-node.md)、[決定ルール](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent-loop/README.md#understand-the-implementation)）。

```ts type-equiv
/** One contributed section of the system prompt (registry input). */
interface PromptSection {
  /** Unique name — a duplicate registration throws (see {@link SystemPrompt.section}). */
  readonly name: string
  /**
   * Sections are concatenated in ascending order. Equal orders use code-unit
   * name order.
   */
  readonly order: number
  /**
   * Static text or a provider evaluated at each assembly with that assembly's
   * {@link AssembleContext}. The text may reference `{{variable}}`s — they are
   * interpolated later, by {@link renderPrompt}, unless `interpolate` is false.
   */
  readonly text: string | ((context: AssembleContext) => string)
  /** Whether to interpolate prompt variables. Defaults to true; false preserves literal text. */
  readonly interpolate?: boolean
  /**
   * Treat this contribution as the complete system prompt. Assembly still
   * runs the cooperative waterfall so tools, contexts, and variables can be
   * resolved, then restores this exact section as the sole prompt section.
   * More than one effective complete section makes assembly fail.
   */
  readonly complete?: boolean
}
```

## 動的プロンプトコンテキスト

`PromptContext` は `PromptSection` のキャッシュセーフな対応物である。組み立てはこれらのコントリビューションを解決して順序付け、agent-loop は保持されたモデル履歴の後に、それが変更されたかコンパクションがそれを削除した場合にのみ、完全な現在のスナップショットをログに記録する。

```ts type-equiv
/** Dynamic model context materialized as a durable user-role snapshot. */
interface PromptContext {
  /** Unique name — a duplicate registration throws (see {@link SystemPrompt.context}). */
  readonly name: string
  /** Contexts are joined in ascending order. */
  readonly order: number
  /** Static text or a provider evaluated at assembly and admission refresh. Empty text contributes nothing. */
  readonly text: string | ((context: AssembleContext) => string)
  /** Whether to interpolate prompt variables. Defaults to true; false preserves literal text. */
  readonly interpolate?: boolean
  /** Keep this operational context when optional runtime context is disabled. */
  readonly required?: boolean
}
```

`required: true` は、任意のランタイムコンテキストが抑制されたときに運用上のコントリビューションを保持する。`interpolate: false` は変数のようなディレクトリ名を含め、リテラルテキストを保持する。`AssembledContext` は解決されたテキストとその補間選択をレンダリングに運ぶ。

```ts type-equiv
/** One resolved dynamic context contribution. */
interface AssembledContext {
  /** The contributing context's unique name. */
  name: string
  /** The resolved text before variable interpolation. */
  text: string
  /** Whether to interpolate prompt variables. Defaults to true; false preserves literal text. */
  interpolate?: boolean
}
```

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxsystemprompt--systemprompt"></a>

### `ctx.systemPrompt` — `SystemPrompt`

各モデルステップの前に組み立てられるプロンプト入力のレジストリサービス。

```ts cordis-catalog
/**
 * Register an ordered prompt section in the calling context's scope. A scoped
 * section shadows a global section with the same name; duplicates within one
 * layer and non-finite orders throw. Registration and disposal emit
 * `system-prompt/change`.
 * @param section - the section to register.
 * @returns the exact Cordis effect disposer.
 */
section(section: PromptSection): () => void

/**
 * Resolve the centrally owned placement of a repository prompt section.
 * @param name - stable section placement name.
 * @returns the section's numeric sort order.
 */
getSectionOrder(name: PromptSectionOrderName): number

/**
 * Resolve the centrally owned placement of a repository runtime context.
 * @param name - stable context placement name.
 * @returns the context's numeric sort order.
 */
getContextOrder(name: PromptContextOrderName): number

/**
 * Register ordered dynamic context in the calling context's scope. Scoped
 * entries shadow global entries with the same name.
 * @param context - the context contribution to register.
 * @returns the exact Cordis effect disposer.
 */
context(context: PromptContext): () => void

/**
 * Suppress optional dynamic runtime-context contributions in the calling
 * context's scope without changing the services that own or enforce those
 * facts. Multiple suppressors remain independently disposable.
 * @returns the exact Cordis effect disposer.
 */
suppressRuntimeContext(): () => void

/**
 * Register a tool-schema provider in the calling context's scope. Global and
 * matching scoped providers both contribute; returning the reserved
 * {@link TOOL_ORDER_REST} name makes assembly fail.
 * @param provider - evaluated for each assembly with its context.
 * @returns the exact Cordis effect disposer.
 */
tools(provider: (context: AssembleContext) => ToolProviderResult): () => void

/**
 * Register a prompt variable in the calling context's scope. Scoped values
 * shadow globals; invalid or duplicate names throw. A provider may return
 * `undefined`, but rendering a section that references that value then fails.
 * @param name - the `[a-z][a-z0-9_]*` reference name.
 * @param provider - evaluated for each assembly.
 * @returns the exact Cordis effect disposer.
 */
variable(name: string, provider: (context: AssembleContext) => string | undefined): () => void

/**
 * Refresh accepted registered runtime facts for request admission. Contexts
 * added only by the assembly waterfall retain their accepted values. Current
 * suppression removes optional contexts, and missing required registrations
 * are restored in registry order. Sections, tools, and interpolation variables
 * retain the accepted assembly; their providers and waterfall do not rerun.
 * @param assembly - accepted assembly for this scope and step.
 * @param context - the same scope and current plugin-defined assembly fields.
 * @returns the accepted assembly with current runtime-context provider text.
 */
refreshContext(assembly: PromptAssembly, context: AssembleContext = {}): PromptAssembly

/**
 * Assemble global and scoped providers, detach tool parameters, apply
 * canonical ordering, then run the presentation waterfall. Scoped sections
 * and variables shadow globals. The waterfall's text is authoritative except
 * that an effective complete section is restored afterwards as the sole
 * prompt section; tool declarations stay provider-owned.
 * @param context - the optional scope, step tool set, and plugin-defined assembly fields.
 * @returns the post-waterfall assembly with ordered tool declarations and any complete prompt enforced.
 */
async assemble(context: AssembleContext = {}): Promise<PromptAssembly>
```

ソース: [`packages/core/system-prompt/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/system-prompt/src/index.ts)

<a id="system-prompt-events"></a>

### `system-prompt/*` イベント

<a id="system-promptassemble--waterfall"></a>

#### `system-prompt/assemble` — waterfall

組み立てられたセクション、コンテキスト、変数に対するプレゼンテーション waterfall。スコープフィルタ付きディスパッチ（`@deepseek-ai/dsh-scope`）: スコープ付きリスナーはそのスコープの組み立てだけを受け取る。返されたテキストは権威である。ツール宣言は provider 所有のままである。供給されたシグナルはこの明示的な組み立てリクエストだけを制御し、後の turn を制御するために保持されてはならない。登録された complete セクションはこの waterfall の後に復元されるため、リスナーはそのスコープのシステムプロンプトに追加も置換もできない。

```ts cordis-catalog
/**
 * Presentation waterfall over assembled sections, contexts, and variables.
 * Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): scoped listeners
 * receive only that scope's assemblies. The returned text is authoritative; tool declarations remain provider-owned.
 * A supplied signal controls only this explicit assembly request and must not
 * be retained to control later turns. A registered complete section is
 * restored after this waterfall, so listeners cannot add to or replace
 * that scope's system prompt.
 * @param assembly - the mutable assembly built from registered providers.
 * @param context - the caller's per-assembly context.
 * @mode waterfall
 */
'system-prompt/assemble'(this: Scoped<SystemPrompt>, assembly: PromptPresentation, context: AssembleContext, next: () => Promise<PromptPresentation>): Promise<PromptPresentation>
```

型: [Scoped](scope.md)

ソース: [`packages/core/system-prompt/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/system-prompt/src/index.ts)

<a id="system-promptchange--emit"></a>

#### `system-prompt/change` — emit

任意のプロンプト provider が変更されたときに emit される。グローバルな変更はすべてのスコープに影響するため、このレジストリ通知はフィルタされない。

```ts cordis-catalog
/**
 * Emitted when any prompt provider changes. This registry notification is
 * unfiltered because a global change affects every scope.
 * @mode emit
 */
'system-prompt/change'(): void
```

ソース: [`packages/core/system-prompt/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/system-prompt/src/index.ts)
<!-- END GENERATED cordis-surface -->
