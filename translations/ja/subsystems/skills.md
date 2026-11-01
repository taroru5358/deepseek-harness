# スキル

日本語 | [English](../../../docs/subsystems/skills.md) | [中文](../../../docs/subsystems/skills.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/skills.md](../../../docs/subsystems/skills.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

[スキル機能ファミリー](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/skill)は、Service Definition（[dsh-skill](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/skill/skill)、`ctx.skills`）、ローカルの Service Provider（[dsh-skill-filesystem](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/skill/skill-filesystem)）、任意のパッケージ型プロバイダー（[dsh-skill-badge](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/experimental/skill-badge)、[dsh-skill-office](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/skill/skill-office)、および [dsh-sandbox-windows-acl](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/sandbox/sandbox-windows-acl) の Windows ACL 診断プロバイダー）、そしてコンシューマー（[dsh-tool-skill](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/skill/tool-skill)）を含む。レジストリはプロバイダー・カタログをホスト層とスコープ単位の層にまたがってマージする。プロバイダーはローカルまたはパッケージ型のスキルを提供し、コンシューマーは初期カタログと置換カタログ、およびモデル向けの `skill` ツールを所有する。スキルはオプションの指示であり、Session イベントではない。したがってその語彙は [core.md](core.md) ではなくここに置かれる。

ソース: [`packages/skill/skill/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/skill/skill/src/index.ts)、[`packages/skill/skill-filesystem/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/skill/skill-filesystem/src/index.ts)、[`packages/experimental/skill-badge/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/skill-badge/src/index.ts)、[`packages/skill/skill-office/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/skill/skill-office/src/index.ts)、[`packages/sandbox/sandbox-windows-acl/src/acl-skill.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/sandbox/sandbox-windows-acl/src/acl-skill.ts)、および [`packages/skill/tool-skill/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/skill/tool-skill/src/index.ts)。

## プロバイダー・レジストリ

`ctx.skills` はローカル、組み込み、リモート、その他のプロバイダーを組み合わせる。登録は同期であり、リモートの初期化と検出は await される `list()` に属する。プロバイダー・オブジェクト、オプション、候補は読み取り専用で借用され、セマンティックなフィールドは検証される。

レジストリはホスト層＋スコープ単位の階層構造を持ち、これは [tools レジストリ](tools.md) が [dsh-scope](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/core/scope) 上で確立した形である。登録は呼び出し元コンテキストのスコープの層に格納されるため、ホスト行とリポジトリ・プラグインはグローバル層に、agent プリセットの常設コンポジションによってマウントされたプラグインはそのプリセットの層に着地し、プロバイダー名はプロセス全体ではなく層ごとに一意である。読み取りはグローバル層と閲覧スコープのチェーンをマージする。最も近い層のエントリが重複するスキル名を無条件に勝ち取り、以下のランク順は 1 つの層内での重複だけを決める。検出キャッシュは解決済みのスコープ・チェーンをキーとするため、スコープの再親子付け（空 Session の再コンポーズ）は、レジストリの変更なしに次の読み取りで可視になる。

1 つの層内では、重複名はランク、プロバイダー順、次にローカル順で解決される。サマリーは名前順にソートされる。拒否された `list()` はログに記録され、不完全な観測から除外される。一方、明示的な不完全観測は、結果をキャッシュ可能にすることなく使用可能な候補を提供する。不正な候補は即座に失敗する。各プロバイダー・ファクトリは登録スコープのコントロールを受け取る。その `invalidate()` は、まさにその登録がアクティブな間だけ完了済みカタログをクリアし、シグナルは登録失敗または破棄で abort する。進行中の検出は、プロバイダーの世代が変わると 1 回だけ再試行する。2 回目の変更は最新の候補を不完全かつ非キャッシュとして返す。プロバイダーと runtime の変更は、フィルターされていない `skills/change` 無効化イベントを発行する。差分は運ばないため、コンシューマーは自身のルックアップ・オプションで `snapshot()` を再取得する。

`SkillProvider.list()` が返す配列は、完全検出の省略形である。`SkillProviderObservation` を使うと、プロバイダーは観測が権威的でないと報告しつつ、直接ロード可能なままの候補を公開できる。

```ts type-equiv
/** Provider candidates plus whether the current discovery is authoritative. */
interface SkillProviderObservation {
  /** Candidates available from the current provider discovery. */
  readonly candidates: readonly SkillCandidate[]
  /** Whether discovery completed and these candidates may be cached. */
  readonly complete: boolean
}
```

```ts type-equiv
/** Provider interface for one source of skills, such as local directories or a remote registry. */
interface SkillProvider {
  /** Unique provider name in the `ctx.skills` registry. */
  readonly name: string
  /**
   * List available skill candidates for the current lookup context. Provider
   * plugins register synchronously during `apply()`; remote initialization,
   * authentication, and discovery are awaited inside this method. Implementations
   * should settle promptly when `options.signal` aborts.
   * @param options - lookup options; `cwd` selects workspace-sensitive skills and `signal` cancels work.
   * @returns provider candidates as a complete-array shorthand, or an explicit
   *   observation when usable candidates came from incomplete discovery.
   */
  readonly list: (options: SkillLookupOptions) => Promise<readonly SkillCandidate[] | SkillProviderObservation>
  /**
   * Load a complete skill body for a previously listed candidate.
   * @param candidate - the winning candidate originally returned by this provider.
   * @param options - lookup options; `cwd` selects workspace-sensitive skills and `signal` cancels work.
   * @returns the full skill body, or `undefined` if it is no longer loadable.
   */
  readonly get: (candidate: SkillCandidate, options: SkillLookupOptions) => Promise<SkillDefinition | undefined>
}
```

```ts type-equiv
/** Registration-scoped lifecycle and invalidation capability borrowed by one provider. */
interface SkillProviderControl {
  /** Aborts if registration fails or when the exact provider registration is disposed. */
  readonly signal: AbortSignal
  /** Invalidate completed catalogs and notify consumers only while the exact registration remains active. */
  readonly invalidate: () => void
}
```

## ローカル検出の優先順位

出荷されるローカル・プロバイダーは、ルートをランク順にスキャンする:

| ランク | ソース | ルート |
|---|---|---|
| 100 | `project-dsh` | `<projectRoot>/.dsh/skills` |
| 200 | `project-agents` | `<projectRoot>/.agents/skills` |
| 300 | `custom` | `Config.customSkillDirs` |
| 400 | `user-dsh` | `<dshHome>/skills` |
| 500 | `user-agents` | `<agentsHome>/skills` |
| 600 | `bundled` | 設定されている場合の `Config.bundledSkillDir` |

プロジェクト・ルートは `.git` を含む最も近い祖先である。それがなければ現在の cwd が使われる。`ctx.fs` が利用可能な場合、git ルートの探索はファイルシステム・サービスを通じて `.git` を探るため、リモートやサンドボックス化された workspace がホストのファイルシステム境界にフォールバックすることはない。ユーザー DSH ルートはその `.system` 子をスキップする。ローカル・プロバイダーは組み込みのシステム・スキルを合成しない。デプロイは設定済みのバンドル・ルートまたは専用プロバイダーを通じてパッケージ型スキルを供給する。失敗したルートは候補を一切提供しないため、別のルートにある低優先度の同名スキルが、回復するまでカタログの description とロードされる本文の両方を供給することがある。

`dsh-skill-badge` は `BUNDLED_SKILL_RANK` に 1 つの不変な `bundled` 候補を登録し、パッケージ済みのアセット・ディレクトリを `resourceBase` を通じて公開する。出荷される CLI はこのプラグインを無効と宣言しているため、そのコンポジション行を有効化することは明示的なオプトインである。

Chokidar は既存のルートを監視し、直接のバンドル／フラット・エントリの追加と削除、および直接のスキル・エントリの変更を捉える。存在しないルートは、Chokidar がアタッチできるまで、最も近い既存の祖先から欠落したパス・セグメントを 1 つずつたどる。バンドル配下のリソース・ファイルはカタログ変更ではない。モデル向けの `write` と `edit` の観測は、対象がカタログに関係する場合にプロバイダーを同期的に無効化する。一方、ホストのウォッチャーは IDE、Git、シェル、外部プロセスの変更をカバーする。ウォッチャーの失敗は、直接ロード可能な候補を隠すことなく、現在の観測を不完全にする。プロジェクト・スコープのウォッチャーは、設定された境界付き LRU を使う。

## スキルの同一性

スキル名は kebab-case（`^[a-z0-9]+(?:-[a-z0-9]+)*$`）である。ローカル・プロバイダーはディレクトリ・バンドル（`<name>/SKILL.md`）とフラットな Markdown ファイル（`<name>.md`）を受け付ける。ネストした再帰的な `**/SKILL.md` 検出はサポートされない。

```ts type-equiv
/** Origin bucket for a skill contribution. The value is prompt-visible metadata, not precedence by itself. */
type SkillSource = 'project-dsh' | 'project-agents' | 'runtime' | 'user-dsh' | 'user-agents' | 'custom' | 'bundled' | (string & {})
```

## サマリー、候補、完全な定義

`SkillSummary` はレジストリの呼び出し中立なサマリー形状である。コンシューマーはどのエントリとフィールドを描画するかを選ぶ。モデル Session カタログはモデル呼び出し可能な `name` と `description` だけを使い、本文や絶対ファイルパスは決して使わない。`SkillInvocationPolicy` は 2 つの独立した呼び出し制御を正の真偽値に正規化し、解決済みのすべてのサマリー、候補、定義がそれを運ぶ。任意の frontmatter がドメイン・モデルになることはない。

```ts type-equiv
/** Invocation controls shared by skill discovery consumers. */
interface SkillInvocationPolicy {
  /** Whether model-facing catalogs and loaders include this skill. */
  readonly modelInvocable: boolean
  /** Whether human-facing command catalogs and loaders include this skill. */
  readonly userInvocable: boolean
}
```

```ts type-equiv
/** Invocation-neutral skill metadata returned by `ctx.skills.list()`. */
interface SkillSummary {
  /** Absolute instruction file path when supplied by the provider; absent for virtual skills. */
  readonly path?: string
  /** Kebab-case identifier used to address the skill. */
  readonly name: string
  /** Short routing description shown by discovery consumers. */
  readonly description: string
  /** Optional extra routing guidance. */
  readonly whenToUse?: string
  /** Resolved model and user invocation controls. */
  readonly invocation: SkillInvocationPolicy
  /** Discovery source that produced this winning skill. */
  readonly source: SkillSource
  /** Provider that owns this skill body. */
  readonly provider: string
  /** Provider-specific base for relative resources. */
  readonly resourceBase?: SkillResourceBase
}
```

`ctx.skills.list()` は 4 つのポリシー組み合わせをすべて保持する。`isModelInvocable(skill)` と `isUserInvocable(skill)` は対応する必須フィールドを読む。モデル専用スキルは `{ modelInvocable: true, userInvocable: false }` を設定し、ユーザー専用スキルは `{ modelInvocable: false, userInvocable: true }` を設定する。両方を `false` にすると、信頼された `ctx.skills.get()` 呼び出し元だけがそのスキルを利用できる。ローカル・プロバイダーは kebab-case の frontmatter キー `disable-model-invocation` と `user-invocable` をそのまま読み、省略されたフィールドは `true` に既定し、解析したすべてのスキルをこの正規化ポリシーへ投影する。

`SkillCatalogSnapshot` は、権威的な不在と、一時的なプロバイダー障害または検出中に変化し続けたカタログとを区別する。`skills` はその観測で収集されたソート済みの呼び出し中立サマリーを含み、`complete` は、登録済みのすべてのプロバイダーが同時カタログ改訂なしに完了した場合にのみ true である。不完全なスナップショットはキャッシュされないため、次のルックアップが検出を再試行する。部分的な結果をどう提示するかは各コンシューマーが決める。

```ts type-equiv
/** One catalog observation plus whether discovery completed within a stable catalog revision. */
interface SkillCatalogSnapshot {
  /** Sorted invocation-neutral summaries collected in this observation. */
  readonly skills: SkillSummary[]
  /** Whether every registered provider completed without a concurrent catalog revision. */
  readonly complete: boolean
}
```

`SkillCandidate` はプロバイダーからレジストリへの形状である。`locator` はプロバイダーの不透明な状態であり、レジストリはそれを保管し、勝ったプロバイダーの `get()` に返すだけである。

```ts type-equiv
/** Provider catalog entry used by the registry to merge and later load skills. */
interface SkillCandidate extends SkillSummary {
  /** Lower ranks win duplicate skill names before provider registration order is considered. */
  readonly rank: number
  /** Opaque provider-owned handle passed back to `provider.get()`. */
  readonly locator: unknown
  /** Parsed optional metadata object from provider-specific skill frontmatter. */
  readonly metadata?: Readonly<Record<string, unknown>>
}
```

`SkillDefinition` は `ctx.skills.get()` が返し、`skill` ツールが使う完全な解析結果である。`resourceBase` は、ローカル、URL、プロバイダー管理のスキルに対して相対リソースのガイダンスをどう描画するかをツールに伝える。

```ts type-equiv
/** Optional provider-specific base used by loaded skill bodies to resolve relative resources. */
type SkillResourceBase =
  | { readonly kind: 'directory'; readonly path: string }
  | { readonly kind: 'url'; readonly url: string }
  | { readonly kind: 'opaque'; readonly description: string }
```

```ts type-equiv
/** Complete parsed skill definition, including the body loaded by `ctx.skills.get()`. */
interface SkillDefinition extends SkillSummary {
  /** Markdown instruction body after any provider-specific metadata removal. */
  readonly content: string
  /** Parsed optional metadata object from frontmatter. */
  readonly metadata?: Readonly<Record<string, unknown>>
}
```

runtime スキルの入力は、呼び出し制御とプロバイダー・ラベルを省略できる。レジストリは両方のデフォルトを一度だけ解決し、その後はプロバイダーと同じ完全な定義形状と先勝ちの収集順を使う。返される disposer はコントリビューションを削除し、検出キャッシュを無効化する。

```ts type-equiv
/** Runtime skill contribution accepted by `ctx.skills.register()`. */
type SkillRegistration = Omit<SkillDefinition, 'invocation' | 'provider'> & {
  /** Invocation controls; omission permits both model and user surfaces. */
  readonly invocation?: SkillInvocationPolicy
  /** Provider label; omission uses the registry-owned runtime provider. */
  readonly provider?: string
}
```

## ルックアップと設定

スキル・ルックアップは cwd に依存する。プロバイダーが workspace ローカルのスキルを公開する可能性があるためであり、その任意のシグナルは呼び出し元に代わってプロバイダーの作業をキャンセルする。レジストリの読み取りはさらに閲覧スコープを受け取る。コンシューマーは呼び出し元の agent（それ自体がスコープ・キーである）を `SkillViewOptions` を通じて渡す。レジストリは層の選択のために `scope` を消費し、プロバイダーは同じ借用オプション・オブジェクトから自身の `SkillLookupOptions` 契約だけを読む。キャンセルはカタログ選択の前後（キャッシュ・ヒットを含む）で確認され、検出と完全定義のロードの両方と競争する。git ルートが見つからない場合、ローカル・プロバイダーは供給された cwd 自体をプロジェクト・ルートとして扱う。

完全な定義はレジストリによってキャッシュされない。各 `get()` は勝ったプロバイダーを選択された候補とともに呼ぶため、ローカル・プロバイダーは現在の本文を再読する。名前がその候補と一致しなくなった定義は拒否され、再検出のためにその正確なプロバイダーを無効化する。

```ts type-equiv
/** Caller context used for cwd-sensitive and abortable provider work. */
interface SkillLookupOptions {
  /** Workspace selector for the current lookup. */
  readonly cwd?: string | undefined
  /** Abort discovery or loading work for the current caller. */
  readonly signal?: AbortSignal | undefined
}
```

```ts type-equiv
/**
 * Registry read options: provider lookup context plus the viewing scope.
 * The registry consumes `scope` to select layers; providers receive the same
 * borrowed options object and read only their {@link SkillLookupOptions}
 * contract from it.
 */
interface SkillViewOptions extends SkillLookupOptions {
  /** Viewing scope (the calling agent); omitted reads the global layer alone. */
  readonly scope?: ScopeKey | undefined
}
```

レジストリは自身の検出キャッシュ境界だけを所有する。ローカル・プロバイダーはファイルシステム・ルート（`dshHome`、`agentsHome`、`customSkillDirs`、および任意の `bundledSkillDir`/`DSH_BUNDLED_SKILL_DIR`）と、ウォッチャーの有効化、ポーリング、安定性、シンボリックリンク、プロジェクト容量の制御を所有する。コンシューマーは自身のカタログ description 境界を所有する。正確なデフォルトと検証は生成された [設定カタログ](../../../docs/config-catalog.md) にある。

```ts type-equiv
/** Skill registry configuration. */
interface Config {
  /** Maximum number of completed cwd/provider catalogs kept in memory. */
  readonly collectCacheMaxEntries?: number
}
```

## Session カタログとツール契約

`dsh-tool-skill` は、利用可能なスキルまたは不完全な検出を観測したライブ Session の最初の `agent/pre-step` で、初期の永続的な user ロールの `<system-reminder>` を注入する。カタログに含まれるのはソート済みのスキル `name` と正規化・XML エスケープ済みの `description` だけであり、本文、パス、ソース、プロバイダー、ルーティング・ヒントは含まない。検出はステップの abort シグナルを `SkillLookupOptions` 経由で転送する。`catalogDescriptionMaxLength` は description 境界のコンシューマー設定で、デフォルトは `500`、整数の最小値は `3` である。

以降の各モデル・ステップの前に、コンシューマーは正確なツール可視性を適用し、Session ログ内の最新の認識可能な可視カタログ・メッセージとエントリおよび完全性を比較する。変化した観測は pre-step 決定に永続的な置換を寄与させ、すべてのスキルを削除すると明示的な空の置換を追加する。不完全な検出は、警告と任意の `incomplete: true` ソース・メタデータを伴って利用可能なサブセットを公開する。このフラグがないことは完全を意味する。回復はエントリが変わらなくてもカタログを再公開する。compaction が過去のカタログ・メッセージをすべて隠した場合、次の観測が現在のカタログを再確立する。先行するカタログのない完全な空ビューは何も発行しない。これらのカタログ・メッセージは Session 履歴であり、World State ではない。

モデル向けの `skill({ name })` ツールは kebab-case 名を検証し、呼び出し中立カタログでサマリーを見つけ、`isModelInvocable` がアクセスを許可しない限りロード前に拒否し、呼び出し元 agent の cwd で完全な定義を再読し、コンテンツを返す前にポリシーを再確認する。解決できないスキルは unknown または no longer available として報告し、`<skill_content name="...">`、`<skill_resources>`、`<skill_instructions>` を含むツール結果を返す。`resourceBase` は明示的に参照されたスクリプト、参照、アセットを必要に応じてのみ解決する。ロード結果はスキル・ディレクトリを列挙しない。したがって本文だけの編集は、カタログ・メッセージを生成したり以前のツール結果を書き換えたりすることなく、以降のツール呼び出しを変える。

## ブラウザ Session カタログ

`SkillListRequest` は `sessionId` で 1 つの Session を指定する。`SkillListValue` はユーザー呼び出し可能なエントリを、名前、description、任意の使用ガイダンス、モデル呼び出しの可用性とともに返す。`SessionSkillCatalog` は Agent をアクティブ化せずに Session の cwd と記録済みプリセットを読む。ライブ Agent は自身のスコープ付きレジストリを供給でき、コールドな Session はプリセットの常設スコープを使う。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxsessionskillcatalog--sessionskillcatalog"></a>

### `ctx.sessionSkillCatalog` — `SessionSkillCatalog`

コールドな Agent をアクティブ化せずに `ctx.remote.skills` を支えるホスト・サービス。

```ts cordis-catalog
/**
 * List the user-invocable skills visible to one Session composition.
 * @param request - Session identity whose cwd and preset select the catalog view.
 * @param signal - caller lifetime carried by the Remote transport; admitted catalog reads retain their existing completion semantics.
 * @returns user-invocable skill metadata without loading skill bodies.
 * @throws RemoteError when the Session cannot be inspected or no registry can serve it.
 */
@Remote async list(request: SkillListRequest, signal: AbortSignal): Promise<SkillListValue>
```

ソース: [`packages/api/session-controller/src/skill-catalog.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/session-controller/src/skill-catalog.ts)

<a id="ctxskills--skillregistry"></a>

### `ctx.skills` — `SkillRegistry`

スキル・プロバイダーの階層レジストリであり、tools レジストリが確立したホスト層＋スコープ単位の形である。登録は呼び出し元コンテキストのスコープ（scopeOf）の層に格納される。ホスト行とリポジトリ・プラグインはグローバル層に、agent プリセットの常設コンポジションでマウントされたプラグインはそのプリセットの層に着地する。読み取りはグローバル層と閲覧スコープのチェーンをマージする。最も近い層のエントリが重複名を無条件に勝ち取り、ランク順は 1 つの層内での重複だけを決める。ソート済みの呼び出し中立サマリーを公開し、完全なスキル本文をオンデマンドでロードする。

```ts cordis-catalog
/**
 * Register a borrowed same-process provider synchronously during plugin
 * apply, into the calling context's layer: a scoped context (an agent
 * preset's standing mount) registers for that scope alone, an unscoped
 * context registers globally. Duplicate names within one layer and reserved
 * names throw; remote initialization belongs in `list()`. Fiber disposal
 * unregisters the provider and invalidates catalog caches.
 * @param create - synchronous factory receiving this registration's lifecycle and invalidation control.
 * @returns the exact Cordis effect disposer that unregisters this provider;
 *   composite effects may yield it directly to preserve teardown ordering.
 */
registerProvider(create: (control: SkillProviderControl) => SkillProvider): () => void

/**
 * Register a borrowed readonly runtime skill into the calling context's
 * layer. Project entries outrank runtime entries, which outrank user
 * entries, within one layer. Same-name runtime entries in one layer are
 * first-wins; a duplicate logs a warning and receives a no-op disposer so
 * it cannot remove the winner.
 * @param skill - the skill definition input; omitted invocation and provider fields receive defaults.
 * @returns the exact Cordis effect disposer, preserving composite teardown order and invalidating caches.
 */
register(skill: SkillRegistration): () => void

/**
 * List invocation-neutral skill summaries for a workspace. Consumers apply
 * model or user invocation policy at their operational boundary. Lookup
 * options and provider candidates are readonly same-process values borrowed
 * throughout discovery.
 * @param options - view options; `scope` selects the viewing agent's layers, `cwd` selects project roots, and `signal` cancels discovery.
 * @returns all sorted winning summaries.
 */
async list(options: SkillViewOptions = {}): Promise<SkillSummary[]>

/**
 * Observe the current invocation-neutral catalog and whether discovery completed within a stable revision.
 * Incomplete observations are never cached, allowing consumers to retain last-good state and
 * retry on their next request boundary.
 * @param options - view options; `scope` selects the viewing agent's layers, `cwd` selects project roots, and `signal` cancels discovery.
 * @returns sorted summaries plus discovery-completeness state.
 */
async snapshot(options: SkillViewOptions = {}): Promise<SkillCatalogSnapshot>

/**
 * Load and validate the winning candidate, passing its opaque discovery locator back to the
 * provider. Cancellation is rechecked after selection, including cache hits, and raced against
 * loading so an uncooperative provider cannot hang the caller.
 * @param name - kebab-case skill name.
 * @param options - view options; `scope` selects the viewing agent's layers,
 *   `cwd` selects workspace-sensitive skills, and `signal` cancels work.
 * @returns the full skill, including body content, or `undefined`.
 */
async get(name: string, options: SkillViewOptions = {}): Promise<SkillDefinition | undefined>
```

ソース: [`packages/skill/skill/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/skill/skill/src/index.ts)

<a id="skills-events"></a>

### `skills/*` イベント

<a id="skillschange--emit"></a>

#### `skills/change` — emit

スキル・プロバイダー、runtime コントリビューション、またはプロバイダー裏付けのカタログが変更された可能性がある。これはフィルターされていない無効化通知であり、コンシューマーは自身のルックアップ・オプションでカタログを再取得する。リスナーの失敗はコンテインメントされ、レジストリの変更を拒否できない。

```ts cordis-catalog
/**
 * A skill provider, runtime contribution, or provider-backed catalog may
 * have changed. This is an unfiltered invalidation notification; consumers
 * refetch the catalog for their own lookup options. Listener failures are
 * contained and cannot veto the registry mutation.
 * @mode emit
 */
'skills/change'(): void
```

ソース: [`packages/skill/skill/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/skill/skill/src/index.ts)
<!-- END GENERATED cordis-surface -->
