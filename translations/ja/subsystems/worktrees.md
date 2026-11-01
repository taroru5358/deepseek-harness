# Git worktree

日本語 | [English](../../../docs/subsystems/worktrees.md) | [中文](../../../docs/subsystems/worktrees.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/worktrees.md](../../../docs/subsystems/worktrees.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

実験的な [`dsh-experimental-worktree`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/worktree/README.md) サービスが `ctx.worktrees` を提供する。その [`create_worktree`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/tool-worktree/README.md) コンシューマーは新しい Git ブランチとチェックアウトを作成し、[working-directory サービス](working-directory.md)を通じてそれを選択する。[Git Worktrees オプション・バンドル](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/tool-worktree/README.md#use-this-package)は GUI プラグイン・マネージャーを通じて両方のパッケージをロードする。同梱時はオフであり、明示的なコンポジションでも 2 つのパッケージをマウントできる。

## 作成と保持

呼び出し元 Session の現在のディレクトリがソース・リポジトリを選択する。作成はチェックアウトを割り当てる前に、要求されたローカルのコミット、ブランチ、またはタグを固定する。省略すると `HEAD` が選択される。各名前は新しいブランチと未使用のチェックアウト・パスを識別しなければならない。この操作は不足しているオブジェクトを取得せず、ソース・チェックアウトからステージ済み・未ステージ・未追跡・無視されたファイルをコピーしない。

デフォルトの宛先は `<current checkout root>/.agents/worktrees/<name>` である。`repositoryRoot` は呼び出し元の現在のチェックアウトの最上位ディレクトリであり、したがってリンクされたチェックアウトは自身のプールを持つ。サービスは、その割り当てが設定されたプール・ディレクトリを作成するときに限り、`*` と末尾の改行を含む `.gitignore` を書き込む。既存のファイルと無視規則は保持される。呼び出し元 Session の既存のサンドボックス・ポリシーは、宛先と共有 Git 管理ディレクトリへの書き込みを許可していなければならない。Git のセットアップが成功してから working-directory の変更が行われる。

`working_directory({ cd: path })` を通じて離れても、チェックアウトとブランチは保持される。キャンセルや失敗も部分的な成果物を保持しうる。エラーはその場所を示す。サービスは破棄時に保留中の操作を中止して排出する。設定と Git の要件は[パッケージ README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/worktree/README.md)が所有する。

## 要求と結果

要求は新しい名前とローカル・リビジョンだけを選択する。結果は作成されたチェックアウトと正確なコミットを識別し、Session の元のプロジェクトや書き込み権限は変更しない。任意のコールバックは、作成の各段階が始まる前にそれを受け取る。復元は、削除されたチェックアウトを記録されたディレクトリに再びチェックアウトする。ブランチから、ブランチが失われている場合はそのベース・コミットから行う。

```ts type-equiv
/** A new checkout from a locally available Git commit, branch, or tag. */
interface CreateWorktreeRequest {
  /** New branch name, also used as the relative checkout directory; omission generates a name. */
  name?: string
  /** Local revision to resolve before creation; omission selects HEAD. */
  from?: string
}
```

```ts type-equiv
/** Committed Git baseline and the calling Session's resulting working directory. */
interface CreatedWorktree {
  /** Canonical absolute checkout directory in the filesystem provider's execution world. */
  path: string
  /** Newly created branch name. */
  branch: string
  /** Resolved commit object name; repository-local replacement refs can change its checked-out content. */
  baseCommit: string
  /** Canonical root of the source checkout. */
  repositoryRoot: string
}
```

```ts type-equiv
/**
 * Stage of one creation or restoration, reported before it begins: `prepare`
 * validates the request, allocates the checkout directory, and creates the
 * branch, which a restoration creates again only when it is gone; `checkout`
 * writes the files and, for a creation, enters the checkout.
 */
type CreateWorktreePhase = 'prepare' | 'checkout'
```

```ts type-equiv
/** A removed checkout to check out again at its recorded directory. */
interface RestoreWorktreeRequest {
  /** Canonical checkout directory the worktree had. */
  path: string
  /** Branch the worktree had checked out. */
  branch: string
  /** Commit the branch started from; a branch that no longer exists is created again here. */
  baseCommit: string
  /** Canonical root of the source checkout. */
  repositoryRoot: string
}
```

```ts type-equiv
/** Outcome of one restoration. */
interface RestoredWorktree {
  /** Checkout directory, present again. */
  path: string
  /** Whether the branch no longer existed and was created again at the base commit, without the commits it had gained. */
  branchRecreated: boolean
}
```

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxworktrees--worktreeservice"></a>

### `ctx.worktrees` — `WorktreeService`

マウントされたファイルシステムとサンドボックス・プロバイダーの下で、保持される Git worktree を作成する。

```ts cordis-catalog
/**
 * Create a fresh branch and checkout at a pinned local revision, then enter it.
 * Existing branches or checkout paths fail. Uncommitted files stay in the source checkout.
 * Checkout disables configured clean, smudge, and process filters without changing Git config.
 * Repository-local replacement refs still apply; baseCommit reports the resolved object name.
 * The new checkout becomes current only after Git setup succeeds. Failures may retain newly
 * allocated Git/filesystem artifacts; no branch or checkout is removed automatically.
 * @param agent - caller whose current directory selects the source repository and file policy.
 * @param request - optional new name and local revision; defaults are generated name and HEAD.
 * @param signal - cancellation of lookup, creation, and working-directory publication.
 * @param onPhase - receives each stage before it begins; it must not throw.
 * @returns canonical checkout path, branch name, pinned commit, and source repository root.
 */
async create( agent: Agent, request: CreateWorktreeRequest = {}, signal?: AbortSignal, onPhase?: (phase: CreateWorktreePhase) => void, ): Promise<CreatedWorktree>

/**
 * Check out a removed worktree again at its recorded directory: its branch
 * when the branch still exists, otherwise a new branch of the same name at
 * `baseCommit`. Only the missing target's unlocked registration is removed, and an
 * existing directory fails. Changes that were never committed are not
 * recovered. Checkout disables configured clean, smudge, and process filters,
 * as creation does. The calling Session's working directory does not change.
 * Failures may retain newly allocated Git/filesystem artifacts.
 * @param agent - caller whose sandbox policy confines the commands.
 * @param request - recorded directory, branch, base commit, and source repository root.
 * @param signal - cancellation of the restoration.
 * @param onPhase - receives each stage before it begins; it must not throw.
 * @returns the directory and whether the branch was created again.
 */
async restore( agent: Agent, request: RestoreWorktreeRequest, signal?: AbortSignal, onPhase?: (phase: CreateWorktreePhase) => void, ): Promise<RestoredWorktree>
```

型: [Agent](core.md)

ソース: [`packages/experimental/worktree/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/worktree/src/index.ts)
<!-- END GENERATED cordis-surface -->
