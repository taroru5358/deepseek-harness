# ファイルシステム

日本語 | [English](../../../docs/subsystems/filesystem.md) | [中文](../../../docs/subsystems/filesystem.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/filesystem.md](../../../docs/subsystems/filesystem.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

オプションのファイルシステム機能は次のロールを組み合わせる: [dsh-fs](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/fs/fs) は `ctx.fs` とガード付きのファイル変更を所有し、[dsh-fs-local](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/fs/fs-local) はローカルディスクを実装し、[dsh-fs-observation-policy](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/fs/fs-observation-policy) は観測された存在または不在を記録し、サービスではなくイベントを通じて鮮度ルールを追加し、[dsh-tool-fs](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/fs/tool-fs) はモデル向けの read/write/edit 呼び出しを直接実行してウィンドウをレンダリングする。これは agent-loop の背骨の外側にある。代替バックエンドはポリシーやツール・スキーマを変えない。純粋な [dsh-fs-patch](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/fs/fs-patch/README.md) ライブラリはパッチの解析とテキスト適用を提供する。

`dsh-fs-observation-policy` はオプションである。これがない場合、`FileSystem` Service Definition、プロバイダー、`dsh-tool-fs` Consumer が完全で制約のないファイルシステム seam を形成する。`write` は無条件に作成または上書きし、`edit` は無条件にリテラルテキストを置換する。ポリシー・プラグインは `fs/*` ウォーターフォールを決定することでこれらの操作を変更する。これを削除してもツールは壊れない。ツールは `ctx.fs` を呼び出してイベントをディスパッチし、ポリシーのメソッドを呼び出さないためである。`dsh-tool-fs` をロードするデプロイは `dsh-fs-observation-policy` もロードすることが期待される。そうすることで既定の動作が read-before-write/edit になる。

プロバイダーのソース: [`packages/fs/fs/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/fs/fs/src/types.ts) と [`packages/fs/fs/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/fs/fs/src/index.ts)。ポリシーのソース: [`packages/fs/fs-observation-policy/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/fs/fs-observation-policy/src/types.ts)。読み取りレンダリングのソース: [`packages/fs/tool-fs/src/read-render.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/fs/tool-fs/src/read-render.ts)。

## ターゲットの同一性とメタデータ（プロバイダー契約）

あらゆる操作は、まずユーザー指定のパスを不透明なバックエンド・ターゲットへ解決する。Consumer は `displayPath` を表示してよいが、`targetKey`（ブランド付きの不透明 ID）を解析したり、ローカル絶対パスと仮定したりしてはならない。

ファイルシステムの実行世界を共有する Consumer は、その同一性を解釈する代わりにプロバイダーを通じて cross-capability の座標を得る: `processPath(target)` はサブプロセスが開ける正規絶対パスを返し、`processPathFromHostPath(hostPath)` は実行世界が共有する場合に限り harness ホストの絶対ファイルをマップし、`fileUrl(target)` はプロバイダー・プラットフォームの `file:` URI を返し、`contains(parent, child)` は正規の同一性または子孫の包含をテストする。

```ts type-equiv
/**
 * A path resolved by a backend into a stable identity. `resolve()` produces
 * this; every other operation takes it.
 */
interface FsTarget {
  /** Opaque key for stale guards and target lookup. */
  targetKey: FsTargetKey
  /**
   * Path for model/UI-facing output. May be a local absolute path,
   * workspace-relative path, or remote URI depending on the backend.
   */
  displayPath: string
}
```

バックエンドはファイル・バージョン・トークン（変更がガードする鮮度トークン）を所有する。ポリシー・プラグインは陳腐化チェックのためにそれらを保存し、Consumer はそれらを解釈しない。どちらの ID もブランド付きの不透明文字列である。

```ts type-equiv
/**
 * Opaque key for stale guards and target lookup. The local backend uses a
 * realpath-like string; a remote backend might use a workspace URI or file id.
 * Consumers MUST NOT parse it or assume it is a local absolute path.
 */
type FsTargetKey = Branded<'FsTargetKey'>
```

```ts type-equiv
/**
 * Opaque file-version token — the freshness token a write/edit guards against.
 * The local backend derives it from high-resolution stat identity and freshness
 * fields; a remote backend might use a revision id. The policy layer records it
 * for stale checks; consumers may display related metadata but MUST NOT
 * interpret this token.
 */
type FsVersion = Branded<'FsVersion'>
```

`stat` はメタデータ（コンテンツは決して返さない）を返し、ターゲットが存在しない場合は `undefined` を返す。`type` により Consumer は読み取り前にディレクトリや特殊ファイルを拒否でき、`size` によりテキスト Consumer は失敗で探ることなく `readText` と `streamText` を選べる。テキスト Consumer は `streamText` を消費しながら独自の保持上限を適用する。生バイトの Consumer は `readBytes(target, signal, maxBytes)` を使う。その必須の完全コンテンツ上限により、既知または発見されたオーバーフローは、切り詰めたり無制限にバッファリングしたりする代わりに `FS_TOO_LARGE` で失敗する。

```ts type-equiv
/**
 * Metadata about a target — what {@link FileSystem.stat} returns. Lets the
 * policy layer reject directories/special files before reading and choose
 * `readText` vs `streamText` from `size` without probing by failure. `version`
 * is the freshness token. `undefined` from `stat` means the target is absent.
 */
interface FsInfo {
  /** Opaque freshness token of the target right now. */
  version: FsVersion
  /** Whether the target is a regular file, a directory, or something else. */
  type: 'file' | 'directory' | 'other'
  /** Byte size of a regular file, when the backend can report it. */
  size?: number
}
```

`lstat` はパスレベルの no-follow メタデータ・プリミティブである。`resolve` は安定した同一性を生成するために意図的にシンボリックリンクをたどるので、これは `FsTarget` ではなくパスを受け取る。信頼境界のチェックが必要な Consumer は、先に `lstat` を呼び出し、解決前に `symlink` を拒否できる。

```ts type-equiv
/**
 * Metadata about a path without following the final path component when it is a
 * symbolic link. Unlike {@link FsInfo}, this path-level probe can report
 * `symlink` so consumers with trust-boundary rules can reject repository-owned
 * links before resolving a target.
 */
interface FsPathInfo {
  /** Opaque freshness token of the path entry right now. */
  version: FsVersion
  /** Whether the path entry is a regular file, directory, symlink, or other. */
  type: 'file' | 'directory' | 'symlink' | 'other'
  /** Byte size of the path entry, when the backend can report it. */
  size?: number
}
```

`listDir` は直接の子エントリーを安定した名前順で返す。各エントリーは子のベース名、型、解決済みターゲット、およびバックエンドが報告できる場合は安価なメタデータを持つ。ファイルの内容を読んではならないため、`size` は通常ファイルのみに用い、`version` はメタデータ由来である。壊れた、または消えた子はメタデータなしの `other` として返されることがある。子メタデータの一覧取得や解決中の権限またはバックエンド I/O の失敗は、`FS_PERMISSION_DENIED` または `FS_IO_ERROR` で一覧全体を失敗させる。

```ts type-equiv
/**
 * One direct child returned by {@link FileSystem.listDir}. Listing returns
 * metadata and resolved targets only; it must not read file contents.
 */
interface FsDirEntry {
  /** Basename of the child inside the listed directory. */
  name: string
  /** Whether the child is a regular file, a directory, or something else. */
  type: 'file' | 'directory' | 'other'
  /** Resolved child target for follow-up operations. */
  target: FsTarget
  /** Opaque freshness token when the backend can report metadata cheaply. */
  version?: FsVersion
  /** Byte size of a regular file, when the backend can report it. */
  size?: number
}
```

<a id="patch-syntax-records"></a>
## パッチ構文のレコード

[dsh-fs-patch](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/fs/fs-patch/README.md) は完全なパッチテキストを、パスが未解決の文字列であるシリアライズ可能な操作へ解析し、`applyHunks` は呼び出し側が提供する LF 正規化済みテキストにハンクを適用して、完全な新テキストまたは構造化されたマッチ失敗を返す。パッケージ README が固定された解析・マッチング規則を所有する。

```ts type-equiv
/** One contiguous update. */
interface PatchHunk {
  /** One-based input line containing the marker or first implicit hunk line. */
  lineNumber: number
  /** Text following `@@ `, used as a search anchor by the text matcher. */
  context?: string
  /** Context and removed lines, without patch prefixes or line endings. */
  oldLines: string[]
  /** Context and added lines, without patch prefixes or line endings. */
  newLines: string[]
  /** Whether `*** End of File` follows this hunk. */
  isEndOfFile: boolean
}

```

```ts type-equiv
/** Ordered file operation; a move is an update with a destination path. */
type PatchOperation =
  | {
    kind: 'add'
    path: string
    lineNumber: number
    /** Added lines joined with LF, including a final LF for every added line. */
    content: string
  }
  | {
    kind: 'delete'
    path: string
    lineNumber: number
  }
  | {
    kind: 'update'
    path: string
    lineNumber: number
    moveTo?: string
    hunks: PatchHunk[]
  }
```

```ts type-equiv
/** The first hunk whose anchor or old lines could not be located in the original text. */
interface HunkApplyFailure {
  /** One-based line of the failing hunk in the patch input. */
  hunkLineNumber: number
  /** The failed anchor or old-content search. */
  stage: 'context' | 'oldLines'
  /** Zero-based sequential search cursor; EOF searches override it with the tail candidate. */
  searchStart: number
  /** Original anchor or old lines, including any trailing empty line retried without its sentinel. */
  pattern: string[]
}

```

```ts type-equiv
/** Complete new text or a match failure; failures contain no partially updated text. */
type ApplyHunksResult =
  | { ok: true; after: string }
  | { ok: false; failure: HunkApplyFailure }
```

## 書き込みと編集のガード（プロバイダー契約）

同じオプションのガード規則がパッチ、削除、移動をカバーする。プロバイダーは観測記録を行わない。呼び出し側が intent をディスパッチし、完了した結果を記録する。

| 操作 | Intent | 成功時の観測 |
|---|---|---|
| `patchText` | `fs/edit-intent` | 返されたバージョンでターゲットが存在する。 |
| `deleteFile` | `fs/delete-intent` | ソースが存在しない。 |
| `moveFile` ソース | `fs/edit-intent` | `sourceDeleted` が true の場合に限りソースが存在しない。 |
| `moveFile` 宛先 | `fs/write-intent` | `destination.version` で宛先が存在する。 |

パッチ、削除、移動のソースは通常ファイルでなければならない。存在しない、または陳腐化したソースは `FS_STALE_VERSION` を報告する。パッチのマッチングは変更ロック内の LF 正規化済みテキストで実行され、不一致はパッチのハンク行とステージを伴う `FS_PATCH_MISMATCH` を報告する。移動は同一のターゲット・キーを `FS_SAME_TARGET` で拒否し、宛先には書き込みガードを使い、ソースの行末とモードを保持する。削除、移動元、上書きされる宛先の基底は、上限付きの nullable な LF テキストである。移動は 2 つの効果からなる。ソースの削除が失敗しても宛先は公開されることがある。キャンセルはローカル宛先のステージング前にチェックされる。その後、プロバイダーは両方の効果を完了し、その結果を報告する。

ターゲットはすでにシンボリックリンクをたどるため、シンボリックリンクのソースを拒否する呼び出し側は `resolve` の前に `lstat` でパスを検査する。

```ts type-equiv
/** Outcome of a guarded delete; the bounded basis follows FsWriteOutcome.before. */
interface FsDeleteOutcome {
  /** LF-normalized removed content, or null for an unavailable, binary, or over-limit basis. */
  before: string | null
}
```

```ts type-equiv
/** Serializable failure after a move has published its destination but retained its source. */
interface FsMoveSourceFailure {
  /** Filesystem failure code for source removal. */
  code: FsErrorCode
  /** Diagnostic identifying the source and the failed removal. */
  message: string
}
```

```ts type-equiv
/** Published destination and removal status; sourceBefore is the bounded LF source basis captured under the move lock. */
type FsMoveOutcome =
  | { sourceBefore: string | null; destination: FsWriteOutcome; sourceDeleted: true }
  | { sourceBefore: string | null; destination: FsWriteOutcome; sourceDeleted: false; sourceFailure: FsMoveSourceFailure }
```

`writeText` と `editText` はどちらもバージョン・ガードを任意で受け取る。無条件の（ベア・プロバイダーによる）変更では省略し、ガードする場合は指定する。`writeText` のガードは `FsWriteIntent` である。`createIfAbsent` は存在しないターゲットを作成し、既存のターゲットを `FS_NOT_OBSERVED` で拒否する。これには、公開自体が no-replace でなければならないため、プロバイダーの初期プローブ後に現れたターゲットも含まれる。`replaceIfVersion` は、ターゲットが観測されたバージョンで存在する場合にのみ置換し、そうでなければ `FS_STALE_VERSION` を返す。`expected` を省略すると無条件に作成または上書きする。ユニオン自体は 2 つのガード付き intent のみを持ち、「ガードなし」は省略で表現されるため、write と edit は同じ任意の `expected` フィールドを使う。

```ts type-equiv
/**
 * Guarded write intent. `createIfAbsent` rejects an existing target with
 * `FS_NOT_OBSERVED`; `replaceIfVersion` rejects absence or mismatch with
 * `FS_STALE_VERSION`. Omitting the intent from `writeText` means unconditional
 * create-or-overwrite, not a third union arm.
 */
type FsWriteIntent =
  | { kind: 'createIfAbsent' }
  | { kind: 'replaceIfVersion'; version: FsVersion }
```

```ts type-equiv
/** Outcome of a full-file write. */
interface FsWriteOutcome {
  /** Whether the write created a new file or replaced an existing one. */
  operation: 'create' | 'update'
  /** Opaque version of the file after the write. */
  version: FsVersion
  /**
   * The file's content BEFORE the write, or `null` when the file did not exist
   * (a create) or the backend declined a contextual basis (for example, a
   * binary/non-UTF-8 prior file or either overwrite side reaching its exclusive limit).
   * LF-normalized storage text (the diff basis), never a diff — a consumer
   * computes the result-time contextual diff from `before`/`after` when
   * `before` is present, else falls back to a whole-file diff.
   */
  before: string | null
  /** The file's content AFTER the write, LF-normalized to share `before`'s diff basis. */
  after: string
}
```

`editText` はプロバイダー・レベルの変更であり、他所で `read` と `write` を組み合わせたものではない。ガード付きの場合、リテラル・マッチングの前に期待バージョンを検証する（したがって陳腐化した編集は、より新しいコンテンツに対するマッチ失敗ではなく `FS_STALE_VERSION` を報告する）。ガードなしの場合は現在のコンテンツを編集する。いずれの場合も置換を適用してアトミックに書き込み、マッチング、行末処理、陳腐化チェック、アトミック置換を 1 つの変更クリティカル・セクション内に保つ。存在しないターゲットはどちらのパスでも `FS_STALE_VERSION` を報告する。

```ts type-equiv
/** A literal-replacement edit request. */
interface FsEditRequest {
  /** Literal non-empty text to replace. Must match exactly (after line-ending normalization). */
  oldString: string
  /** Literal replacement text. An empty string deletes the matched text. */
  newString: string
  /** Replace every match instead of requiring exactly one. */
  replaceAll: boolean
}
```

```ts type-equiv
/** Outcome of a literal edit. */
interface FsEditOutcome {
  /** Opaque version of the file after the edit. */
  version: FsVersion
  /**
   * The file's content BEFORE the edit. Raw storage text (LF-normalized by the
   * backend), never a diff — a consumer computes the result-time contextual diff
   * (the applied hunk with context) from `before`/`after`.
   */
  before: string
  /** The file's content AFTER the edit. */
  after: string
}
```

## fs ポリシー・イベント（プロバイダー契約の語彙）

`dsh-fs` は変更 Consumer と観測ポリシーのために 4 つのイベントを所有する。Consumer は操作の前に intent をディスパッチし、完了した効果を記録する。ポリシー・プラグインには依存しない。イベントは `dsh-fs` の語彙と不透明な `object` アクターのみを運び、モデル向けの概念も agent/session の所有者構造も持たない。

`fs/write-intent`、`fs/edit-intent`、`fs/delete-intent` は**単一スロットの決定ウォーターフォール**である。ツールはそれぞれを `undefined` を返す既定のサンク（ベア・プロバイダー）付きでディスパッチし、リスナーは `next()` を呼ばずに決定を完全に下す。スロットは登録順の first-wins である。これを所有するポリシー・プラグインはデプロイの慣習であり、強制される不変条件ではない。`fs/observed` は `FsObservation`（あるバージョンでの存在、または不在の確認）を運ぶ fire-and-forget の記録イベントである。これは素の `ctx.emit` でディスパッチされる。そのリスナーは同期的で副作用のみでなければならない。ツールは emit をガードしないため、例外を投げるリスナーは読み取りエラーを置き換えたり、変更がすでに成功した後にツールの `isError` 結果として表面化したりする可能性がある。下記の生成された [cordis サーフェス](#cordis-surface) は正確なシグネチャを示す。

```ts type-equiv
/**
 * One authoritative observation of a target. A present observation carries the
 * version used by guarded replacement; an absent observation authorizes only a
 * guarded create, never an edit.
 */
type FsObservation =
  | { readonly kind: 'present'; readonly version: FsVersion }
  | { readonly kind: 'absent' }
```

## 実行コンテキスト（ポリシー・プラグイン）

ポリシー・プラグインは、`fs/*` イベントが運ぶ不透明な `object` アクターを絞り込むことで、観測状態の所有者を導出するのに十分な実行コンテキストだけを必要とする。`ToolExecution` は必要なフィールドを持つため、`dsh-tool-fs` は実行オブジェクトをアクターとしてそのまま渡し、`dsh-fs-observation-policy` がツール、agent、session の各パッケージをインポートする必要はない。

```ts type-equiv
/**
 * Minimal structural view of a tool execution the policy plugin needs to derive
 * an observed-state owner. `@deepseek-ai/dsh-tools`' `ToolExecution` contains
 * these fields, so the tool passes its `exec` straight through as the opaque
 * `object` actor on the `fs/*` events; this plugin narrows that actor to
 * `FsObservationActor` without importing `dsh-tools`, `dsh-agent`, or `dsh-session`.
 *
 * The owner is `agent.session` when present. It is treated as an opaque object
 * identity (a `WeakMap` key); this package never reads any of its fields.
 */
interface FsObservationActor {
  /** The agent on whose behalf the call runs, when there is one. */
  agent?: {
    /** The session that owns observed-file state, used as an opaque key. */
    session?: object
  }
}
```

## 読み取り結果（Consumer / 読み取りレンダリング）

テキスト読み取りは行ウィンドウ、バイト上限、バックエンドの制限で制限される。バイト上限に達した後も、`totalLines` が正確なままになるよう、より多くの行を保持せずにスキャンを続ける。モデル向けの `read` ツールがレンダリングする結果は純粋に表示用である。`full`/`partial` のビューは存在しない。認可は鮮度ベースである（ツールは stat のバージョンを持つ present な `fs/observed` を直接 emit する）ため、ファイルが変更されていなければ、任意のウィンドウ付き読み取りが後続の write/edit を認可できる。メタデータのミスは、ツールが `FS_NOT_FOUND` を返す前に absent な観測を emit し、後続のガード付き書き込みが、編集を認可せずに外部で削除されたターゲットを再作成できるようにする。読み取りを所有する実行器である `dsh-tool-fs` が読み取りウィンドウ処理を実装してこの結果を構築する。ポリシー・プラグインはそうしない。

```ts type-equiv
/** Outcome of a bounded text read — what {@link formatReadOutput} renders. */
interface FileReadOutcome {
  /** 1-based first line requested. */
  offset: number
  /** Returned lines, already numbered. */
  lines: FileTextLine[]
  /** Exact total line count in the file. */
  totalLines: number
  /** Whether selected output hit the byte cap. */
  truncatedByBytes?: true
}
```

## 観測ファイル状態（ポリシー・プラグイン）

観測状態は `dsh-fs-observation-policy` プラグイン内部に保持される `WeakMap<owner, Map<targetKey, FsObservation>>` である。マップ・エントリーが存在しない場合は未観測を意味する。`{ kind: 'absent' }` は、`read` または `str_replace_editor` の `view`、`str_replace`、`insert` のメタデータ・ミスが不在を確認したことを意味する。`{ kind: 'present', version }` は、読み取り、書き込み、編集がそのバージョンを観測したことを意味する。書き込みの決定は未観測と不在を `createIfAbsent` にマップし、存在は `replaceIfVersion` にマップする。編集／削除の決定は未観測を `FS_NOT_OBSERVED`、不在を `FS_NOT_FOUND`、存在をそのバージョン・ガードにマップする。所有者はイベント・アクター（通常は `exec.agent.session`）から導出され、不透明として扱われ、決して読まれない。破棄はすべてを落とす（HMR 安全性）。またポリシーはファイルシステム I/O を行わない。

## パッチ実行（Consumer）

内部の [tool-fs パッチ実行器](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/fs/tool-fs/README.md#understand-the-implementation) は、プリフライトで観測を emit せずに intent の決定をディスパッチし、先行する操作をシミュレートし、各コミットでガードを再取得する。サンドボックスの封じ込めは `write` と `edit` と同様、コミット時にプロバイダーに残る。どの効果が記録されるかはプロバイダーの結果で決まる。実行器の詳細と失敗時の保証はパッケージ README に属する。

| 操作 | 最初のターゲット使用時のプリフライト intent | コミット intent | 成功時の観測 |
|---|---|---|---|
| add | `fs/write-intent` | `fs/write-intent` | ターゲットが存在(version) |
| update | `fs/edit-intent` | `fs/edit-intent` | ターゲットが存在(version) |
| delete | `fs/delete-intent` | `fs/delete-intent` | ターゲットが不在 |
| move | ソース `fs/edit-intent`、宛先 `fs/write-intent` | 同じ 2 つの intent を新たにディスパッチ | 宛先が存在(version)。削除された場合に限りソースが不在 |

`apply_patch` ツールは、操作ごとの診断とレンダリング用の全文変更入力を持つ厳密なコミット値を返す。永続化されたメタデータは、プロバイダーの上限付き基底からの差分ハンクと stopped フラグを保持する。構文とプリフライトの拒否は `isError: true` を設定する。コミットの報告は stopped であっても `isError: false` を維持し、既知の部分差分が Session の表示に届くようにする。ただし、キャンセルされた呼び出しは他のすべてのツールと同じくツール・レジストリのアボート結果を受け取る。フォーマットとリカバリーは [ツール README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/fs/tool-fs/README.md#use-this-package) が所有する。

## エラー分類（プロバイダー契約）

ファイルシステムの失敗は、`FsError`（`HarnessError`）が運ぶ安定した `FsErrorCode` 文字列を使う。ツール・レジストリはエラー結果の `{ name, code }` を保持するため、リトライ、権限、UI の各レイヤーはテキストを解析せずに分岐できる。

`FS_PATCH_MISMATCH` はプロバイダーのコミット内でハンクが失敗したことを意味し、そのメッセージはパッチ行と失敗した検索ステージを示す。`FS_SAME_TARGET` は移動のソースと宛先が解決済みの同一性を共有することを意味する。削除 intent は編集 intent に従う。未観測は `FS_NOT_OBSERVED`、観測された不在は `FS_NOT_FOUND`、存在状態はバージョンを提供する。プロバイダー側で見つからないパッチ／削除／移動のソースは、ガードがなくても `FS_STALE_VERSION` を使う。

```ts
import { FS_ERROR_CODES } from '@deepseek-ai/dsh-fs'
```

```ts type-equiv
/** One machine-routable filesystem failure code from FS_ERROR_CODES. */
type FsErrorCode = typeof FS_ERROR_CODES[number]
```

`FS_NOT_DIRECTORY`、`FS_PERMISSION_DENIED`、`FS_IO_ERROR` は、ディレクトリ一覧で、既存の非ディレクトリ・ターゲット、拒否された一覧、予期しないバックエンド I/O 失敗を区別するために使われる。`FS_SANDBOX_DENIED` はサンドボックスを強制するバックエンド（`dsh-fs-sandbox`）からの POLICY 拒否（モードのフェンスが変更を拒否した）であり、`FS_PERMISSION_DENIED`（ホスト・カーネルによる拒否）とは異なる。`FS_NOT_OBSERVED` は、ポリシー・プラグインにこの所有者の事前観測記録がないこと（または `createIfAbsent` が既存ファイルに当たったこと）を意味する。`FS_NOT_FOUND` は、確認された不在によって拒否された編集も表す。`FS_STALE_VERSION` は、バックエンドのバージョンが観測されたものと一致しなくなったこと（またはプロバイダー自体が存在しないターゲットの編集を受け取ったこと）を意味する。鮮度の認可に partial/full の区別はないため、`FS_PARTIAL_OBSERVATION` は存在しない。

## ファイル IO にタイムアウトはない

`read`/`write`/`edit` は `timeoutMs` を**受け取らず**、プロバイダー契約も期限を設けない。bash と web（[`@deepseek-ai/dsh-timeout`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/util/timeout/README.md) を消費する）や、サブプロセスに支えられた `glob`/`grep`（宣言された `timeoutMs` は `@deepseek-ai/dsh-tool-call-timeout-policy` によって強制される）とは異なる。それらはプロセスに支えられており、期限が実際に作業を止められる。ローカル syscall はせいぜい best-effort でアボートできるだけである。タイムアウトは進行中の `fsync`/`rename` を強制的に停止できないため、ここでの `timeoutMs` は seam が強制できない期限になり、explicit-over-implicit が禁じるまさにその場所に暗黙の既定値が入ることになる。キャンセルは引き続きツール実行シグナルを通じて伝播し、syscall 境界での best-effort アボートを可能にする。

## サービスとプラグイン

`FileSystem`（`ctx.fs`、抽象）はプロバイダー・プリミティブを所有する: `resolve`、`processPath`、`processPathFromHostPath`、`fileUrl`、`contains`、`stat`、`lstat`、`readText`、`streamText`、`readBytes`、`listDir`、`writeText`、`editText`、`patchText`、`deleteFile`、`moveFile`。`dsh-fs-observation-policy` は**サービスを登録しない**。これは `fs/*` イベント・ゲートを通じてポリシーを追加するプラグインであり、未観測／不在／存在の状態から write/edit/delete の intent ウォーターフォールを決定し、`FsObservation` 値を記録する。実行器は `dsh-tool-fs` である。これは `ctx.fs` を通じて読み取り／書き込み／編集を行い、ウォーターフォールをディスパッチし、記録イベントを emit する。下記の生成された [`ctx.fs` セクション](#ctxfs--filesystem-abstract-seam) は正確なシグネチャを示す。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxfs--filesystem-abstract-seam"></a>

### `ctx.fs` — `FileSystem`（抽象 seam）

抽象ファイルシステム・プロバイダー。ターゲットはエイリアス間で同一性を保たなければならず、読み取りは通常の UTF-8 テキストまたは型付きエラーを公開し、一覧は安定してコンテンツを含まず、単一ファイルの公開はアトミックである。移動は宛先を公開した後にソースを保持することがある。任意のガードは、ガードなしのプロバイダー契約を変えずに陳腐化保護を追加する。

```ts cordis-catalog
/**
 * Observe one file or a directory's direct entries in this provider's execution world.
 * @param target - resolved file or directory, including an absent path to observe for creation.
 * @param changed - invalidation callback; errors can be reported during or after initialization.
 * @param signal - cancels watcher initialization; the caller closes an initialized watcher.
 * @returns a promise resolving once observation is active, with an asynchronous close function.
 * @throws when the provider does not support watching or cannot initialize the watcher.
 */
watch(target: FsTarget, changed: (error?: Error) => void, signal: AbortSignal): Promise<() => Promise<void>>

/**
 * Resolve a model/plugin-supplied path into a stable {@link FsTarget}. May perform I/O (a
 * remote/sandboxed backend may need a round-trip to map a path to a stable identity), hence
 * async even though the local backend only normalizes + realpaths.
 *
 * @param path - the path to resolve; relative paths resolve against `opts.cwd`.
 * @param opts - optional cwd override and cancellation signal.
 * @returns the stable target; the same file yields the same `targetKey`.
 */
abstract resolve(path: string, opts?: { cwd?: string; signal?: AbortSignal }): Promise<FsTarget>

/**
 * Return the canonical absolute path a subprocess in this filesystem's
 * execution world can open. The path is deliberately separate from
 * {@link FsTarget.targetKey}: consumers may pass this value to another OS
 * capability, but must continue treating the target key as opaque.
 * @param target - the resolved target whose process path is required.
 * @returns an absolute path in the backend's execution world.
 */
abstract processPath(target: FsTarget): string

/**
 * Map an absolute path from the harness host into this filesystem's
 * execution world when both paths identify the same file. The base provider
 * exposes no mapping; host-backed or explicitly shared backends override it.
 * @param hostPath - absolute path in the harness host filesystem.
 * @returns the process path for the same file, or undefined when this
 *   execution world cannot read that host file.
 */
processPathFromHostPath(hostPath: string): string | undefined

/**
 * Return the canonical `file:` URI for a target in this filesystem's
 * execution world. Backends own URI encoding because the host platform may
 * differ from the execution platform.
 * @param target - the resolved target to encode.
 * @returns the target's canonical file URI.
 */
abstract fileUrl(target: FsTarget): string

/**
 * Test canonical containment without exposing or parsing backend target
 * keys. Both targets must come from this provider.
 * @param parent - canonical directory target.
 * @param child - canonical candidate target.
 * @returns true when `child` is `parent` or a descendant of it.
 */
abstract contains(parent: FsTarget, child: FsTarget): boolean

/**
 * Return target metadata, or `undefined` when the target does not exist.
 * @param target - the resolved target to stat.
 * @param signal - aborts the metadata round-trip.
 * @returns metadata only, never content; undefined for an absent target.
 */
abstract stat(target: FsTarget, signal?: AbortSignal): Promise<FsInfo | undefined>

/**
 * Return path metadata without following the final path component when it is a
 * symbolic link. This is intentionally path-shaped, not target-shaped:
 * {@link resolve} follows symlinks to produce the stable identity used by
 * normal reads/writes, while `lstat` lets a consumer reject the path itself
 * before that follow happens.
 *
 * `opts.cwd` follows {@link resolve}'s cwd rules. `undefined` means the path is
 * absent.
 * @param path - the path to inspect; relative paths resolve against `opts.cwd`.
 * @param opts - `cwd` overrides the backend's default base for relative paths.
 * @param signal - aborts the metadata round-trip.
 * @returns metadata only, never content; undefined for an absent path.
 */
abstract lstat(path: string, opts?: { cwd?: string }, signal?: AbortSignal): Promise<FsPathInfo | undefined>

/**
 * Read the whole regular text file as a single decoded string.
 * @param target - the resolved target to read.
 * @param signal - aborts the read.
 * @returns the full decoded UTF-8 content.
 */
abstract readText(target: FsTarget, signal?: AbortSignal): Promise<string>

/**
 * Stream the whole regular text file as decoded text chunks (same text
 * semantics as {@link readText}, for large files). The backend owns
 * cross-chunk UTF-8 decoding and binary rejection so the policy layer never
 * touches raw bytes.
 * @param target - the resolved target to read.
 * @param signal - aborts the stream, including between chunks.
 * @returns the chunk iterable, decoded and validated like {@link readText}.
 */
abstract streamText(target: FsTarget, signal?: AbortSignal): Promise<AsyncIterable<string>>

/**
 * Read the whole regular file as raw bytes with no decoding or binary
 * rejection. The bound lives at this seam so a backend can never buffer an
 * unbounded file: a target known or discovered to exceed `maxBytes` fails
 * with `FS_TOO_LARGE` instead of returning a truncated result.
 * @param target - the resolved target to read.
 * @param signal - aborts the read.
 * @param maxBytes - inclusive byte cap on the complete content.
 * @returns the full raw content, at most `maxBytes` long.
 */
abstract readBytes(target: FsTarget, signal: AbortSignal | undefined, maxBytes: number): Promise<Uint8Array>

/**
 * Read one byte window of the regular file as raw bytes with no decoding or
 * binary rejection: the bytes at `[offset, offset + length)`, shorter when
 * the file ends inside the window and empty when `offset` lies at or past
 * its end. The window is the bound here, not the file: a backend transfers
 * at most `length` bytes of content beyond the prefix it skips to reach
 * `offset` and never buffers the whole file, so the caller's cap on `length`
 * is the guard against unbounded buffering.
 * @param target - the resolved target to read.
 * @param range - `offset`, the 0-based first byte, and `length`, the largest byte count; both non-negative integers.
 * @param signal - aborts the read.
 * @returns the window's bytes, at most `length` long.
 */
abstract readByteRange(target: FsTarget, range: { offset: number; length: number }, signal?: AbortSignal): Promise<Uint8Array>

/**
 * List direct children of a directory in stable name order. Returns resolved
 * child targets plus cheap metadata only; never reads file contents.
 * @param target - the resolved directory target.
 * @param signal - aborts the listing.
 * @returns one entry per direct child, in stable name order.
 */
abstract listDir(target: FsTarget, signal?: AbortSignal): Promise<FsDirEntry[]>

/**
 * Atomically create or replace UTF-8 text. `expected` guards intent and
 * staleness; omission allows unconditional overwrite.
 * @param target - the resolved target to write.
 * @param content - the full new file content.
 * @param expected - the write intent guarding the write; omit for unconditional.
 * @param signal - aborts before atomic publication takes effect.
 * @param sandboxPolicy - the per-call mode and workspace root this write
 *   runs under; a sandboxing backend fences the write by it, the bare backend
 *   ignores it. Omit to leave the backend its own default.
 * @returns the outcome, including the version the write produced.
 */
abstract writeText( target: FsTarget, content: string, expected?: FsWriteIntent, signal?: AbortSignal, sandboxPolicy?: SandboxExecutionPolicy, ): Promise<FsWriteOutcome>

/**
 * Atomically edit literal text. When supplied, the version guard is checked
 * before matching so stale content reports `FS_STALE_VERSION`; omission edits
 * the current content without a freshness precondition.
 * @param target - the resolved target to edit.
 * @param edit - the literal search/replace request.
 * @param expected - the version guard; omit for an unconditional edit.
 * @param signal - aborts before atomic publication takes effect.
 * @param sandboxPolicy - the per-call mode and workspace root this edit runs
 *   under; a sandboxing backend fences the edit by it, the bare backend
 *   ignores it. Omit to leave the backend its own default.
 * @returns the outcome, including the version the edit produced.
 */
abstract editText( target: FsTarget, edit: FsEditRequest, expected?: { version: FsVersion }, signal?: AbortSignal, sandboxPolicy?: SandboxExecutionPolicy, ): Promise<FsEditOutcome>

/**
 * Atomically apply parsed hunks to the LF-normalized text and restore the
 * stored line-ending style. A missing or stale target reports
 * `FS_STALE_VERSION`; an unmatched hunk reports `FS_PATCH_MISMATCH`.
 * @param target - the resolved regular file to patch.
 * @param hunks - ordered hunks from `parsePatch`.
 * @param expected - the version guard; omit for an unconditional patch.
 * @param signal - aborts before atomic publication takes effect.
 * @param sandboxPolicy - the per-call mode and workspace root this patch runs
 *   under; a sandboxing backend fences the patch by it, the bare backend
 *   ignores it. Omit to leave the backend its own default.
 * @returns the outcome, including the version the patch produced.
 */
abstract patchText( target: FsTarget, hunks: readonly PatchHunk[], expected?: { version: FsVersion }, signal?: AbortSignal, sandboxPolicy?: SandboxExecutionPolicy, ): Promise<FsEditOutcome>

/**
 * Remove a regular file. A missing or stale target reports `FS_STALE_VERSION`;
 * a directory or other non-file reports `FS_NOT_REGULAR_FILE`. Resolution has
 * already followed symlinks, so the target is the real file.
 * @param target - the resolved file to remove.
 * @param expected - the version guard; omit for an unconditional removal.
 * @param signal - aborts before removal.
 * @param sandboxPolicy - the per-call mode and workspace root this removal
 *   runs under; a sandboxing backend fences it, the bare backend ignores it.
 *   Omit to leave the backend its own default.
 * @returns the bounded LF-normalized removed text, or null when no basis is available.
 */
abstract deleteFile( target: FsTarget, expected?: { version: FsVersion }, signal?: AbortSignal, sandboxPolicy?: SandboxExecutionPolicy, ): Promise<FsDeleteOutcome>

/**
 * Publish the source text, optionally patched, at the destination, then remove
 * the source. Both targets stay locked in canonical order. Identical targets
 * report `FS_SAME_TARGET`; the source guard follows `patchText` and the
 * destination guard follows `writeText`. The two effects are separate: a
 * published destination with a retained source is a final outcome reported
 * through `sourceDeleted: false`.
 * @param source - the resolved regular source; its line-ending style and mode carry over.
 * @param destination - the resolved destination within the same provider.
 * @param options - optional hunks and the source/destination guards; an omitted guard is unconditional.
 * @param signal - checked before destination staging; the commit then completes both effects.
 * @param sandboxPolicy - the per-call mode and workspace root both paths run
 *   under; a sandboxing backend fences them, the bare backend ignores it.
 *   Omit to leave the backend its own default.
 * @returns the bounded LF source basis, the destination outcome, and the source-removal status.
 */
abstract moveFile( source: FsTarget, destination: FsTarget, options: { hunks?: readonly PatchHunk[] expectedSource?: { version: FsVersion } expectedDestination?: FsWriteIntent }, signal?: AbortSignal, sandboxPolicy?: SandboxExecutionPolicy, ): Promise<FsMoveOutcome>
```

型: [SandboxExecutionPolicy](sandbox.md)

ソース: [`packages/fs/fs/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/fs/fs/src/index.ts)

<a id="fs-events"></a>

### `fs/*` イベント

<a id="fsdelete-intent--waterfall"></a>

#### `fs/delete-intent` — waterfall

`FileSystem.deleteFile` の単一スロット決定。最初に返されたガードが勝つ。`next()` を呼ぶと無条件の削除に委譲する。

```ts cordis-catalog
/**
 * Single-slot decision for FileSystem.deleteFile; the first returned guard wins.
 * Calling next() delegates to an unconditional deletion.
 * @param target - the resolved source about to be removed.
 * @param actor - the opaque tool-execution context used to select observed state.
 * @mode waterfall
 */
'fs/delete-intent'(target: FsTarget, actor: object | undefined, next: () => { version: FsVersion } | undefined | Promise<{ version: FsVersion } | undefined>): Promise<{ version: FsVersion } | undefined>
```

ソース: [`packages/fs/fs/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/fs/fs/src/index.ts)

<a id="fsedit-intent--waterfall"></a>

#### `fs/edit-intent` — waterfall

次の `FileSystem.editText` の単一スロット決定。`next()` を呼ぶと無条件の編集になる。最初に返されたガードが勝つ。

```ts cordis-catalog
/**
 * Single-slot decision for the next {@link FileSystem.editText}. Calling
 * `next()` yields an unconditional edit; the first returned guard wins.
 * @param target - the resolved target about to be edited.
 * @param actor - the opaque tool-execution context the decider keys off.
 * @mode waterfall
 */
'fs/edit-intent'(target: FsTarget, actor: object | undefined, next: () => { version: FsVersion } | undefined | Promise<{ version: FsVersion } | undefined>): Promise<{ version: FsVersion } | undefined>
```

ソース: [`packages/fs/fs/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/fs/fs/src/index.ts)

<a id="fsobserved--emit"></a>

#### `fs/observed` — emit

権威ある肯定または否定の観測を記録する。リスナーは同期の記録者でなければならない。例外はツール呼び出しを失敗させ、返された promise は待機されない。

```ts cordis-catalog
/**
 * Record an authoritative positive or negative observation. Listeners must
 * be synchronous recorders: throws fail the tool call and returned promises
 * are not awaited.
 * @param target - the target whose presence or absence was observed.
 * @param observation - present with its version, or confirmed absent.
 * @param actor - the observing tool-execution context; undefined records nothing useful.
 * @mode emit
 */
'fs/observed'(target: FsTarget, observation: FsObservation, actor: object | undefined): void
```

ソース: [`packages/fs/fs/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/fs/fs/src/index.ts)

<a id="fswrite-intent--waterfall"></a>

#### `fs/write-intent` — waterfall

次の `FileSystem.writeText` の単一スロット決定。`next()` を呼ぶとベア・プロバイダーの無条件書き込みになる。intent を返した最初のリスナーが、ピアと合成するのではなく決定を所有する。

```ts cordis-catalog
/**
 * Single-slot decision for the next {@link FileSystem.writeText}. Calling
 * `next()` yields the bare provider's unconditional write; the first listener
 * that returns an intent owns the decision rather than composing with peers.
 * @param target - the resolved target about to be written.
 * @param actor - the opaque tool-execution context the decider keys off.
 * @mode waterfall
 */
'fs/write-intent'(target: FsTarget, actor: object | undefined, next: () => FsWriteIntent | undefined | Promise<FsWriteIntent | undefined>): Promise<FsWriteIntent | undefined>
```

ソース: [`packages/fs/fs/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/fs/fs/src/index.ts)
<!-- END GENERATED cordis-surface -->
