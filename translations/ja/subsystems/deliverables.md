# 成果物

日本語 | [English](../../../docs/subsystems/deliverables.md) | [中文](../../../docs/subsystems/deliverables.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/deliverables.md](../../../docs/subsystems/deliverables.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

1 つの turn がユーザーに渡すもの。[成果物パッケージ・グループ](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/deliverables/README.md)が所有する。モデルが `present` ツールを通じて宣言したファイル（ログ専用の Session イベントに記録される）と、turn が変更したファイル（turn 開始時と turn 終了時の git 作業ツリー・スナップショット、および git が対象としないパスについて各ファイル・ツール編集の前後で取得したファイル全体のキャプチャから要約され、ログ専用イベントで告知され、Session が存続する間 Host サービスが配信する。各リスト・ファイルの turn 開始時と turn 終了時の比較も添えられる）である。読むのはクライアントだけであり、Web の[成果物プラグイン](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-deliverables/README.md)が turn の終わりに両方をレンダリングする。ツールの挙動、スナップショットの仕組み、設定は [`tool-present`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/deliverables/tool-present/README.md) と [`workspace-changes`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/deliverables/workspace-changes/README.md) のパッケージ README にある。

ソース: [`packages/deliverables/tool-present/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/deliverables/tool-present/src/types.ts), [`packages/deliverables/workspace-changes/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/deliverables/workspace-changes/src/types.ts)

パッチ呼び出しも同じ要約 API と比較 API を使う。Git の対象外のパスでは、移動は削除されたソースと追加された宛先として寄与する。中止されたコミットは実際の正味のファイル変更だけを寄与する。[キャプチャ・ルール](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/deliverables/workspace-changes/README.md#use-this-package)がパス選択と観測の挙動を定義する。

## `PresentedFile` — 宣言された 1 つの成果物

```ts type-equiv
/** A declared filesystem file whose current contents remain at its source path. */
interface PresentedFile {
  /** Original absolute path or path relative to the Session working directory. */
  path: string
  /** Optional description supplied by the model. */
  description?: string
}
```

## `WorkspaceChangedFile` — 変更された 1 ファイル

```ts type-equiv
/** One file changed during a turn, with line counts from git or from the whole-file captures around its file-tool edits. */
interface WorkspaceChangedFile {
  /** Path relative to the Session working directory, or an absolute Host path outside it. */
  path: string
  /**
   * Sort key and label: the relative path inside the working directory, a
   * `../` path for repository files above it, a `~` path under the home
   * directory, otherwise the absolute path. Always slash-separated.
   */
  display: string
  /** Lines added; zero for a binary or oversized file. */
  added: number
  /** Lines deleted; zero for a binary or oversized file. */
  deleted: number
  /** Present when git reported the file as binary, or when a captured side holds a NUL byte. */
  binary?: true
  /** Present when a captured side exceeded the plugin's `maxFileBytes`; the file is listed without counts or comparison. */
  oversized?: true
}
```

## `WorkspaceChangesSummary` — 1 turn の変更サマリー

```ts type-equiv
/** Files changed during one top-level turn, kept on the Host until its Session is disposed. */
interface WorkspaceChangesSummary {
  /** The turn whose file changes this summary describes. */
  turn: number
  /** The Session working directory `path` values are relative to. */
  cwd: string
  /** Changed files in `display` order, capped at the plugin's `maxFiles`. */
  files: WorkspaceChangedFile[]
  /** Complete changed-file count, including files omitted by the cap. */
  total: number
  /** Lines added over every changed file, including files omitted by the cap. */
  added: number
  /** Lines deleted over every changed file, including files omitted by the cap. */
  deleted: number
  /** Git tree ids of the turn-start and turn-end snapshots; absent when no snapshot was taken. */
  snapshot?: { before: string; after: string }
}
```

## `WorkspaceDiffHunk` — 1 つの unified diff ハンク

```ts type-equiv
/** One unified-diff hunk with three context lines; every line keeps its `+`, `-`, or space prefix. */
interface WorkspaceDiffHunk {
  /** First line of the hunk in the turn-start content, 1-based; a side without lines starts at 1 with zero lines. */
  oldStart: number
  /** Lines of the hunk taken from the turn-start content. */
  oldLines: number
  /** First line of the hunk in the turn-end content, 1-based; a side without lines starts at 1 with zero lines. */
  newStart: number
  /** Lines of the hunk taken from the turn-end content. */
  newLines: number
  /** Hunk body in order, each line prefixed with `+`, `-`, or a space. */
  lines: string[]
}
```

## `WorkspaceFileDiff` — 1 ファイルの比較

```ts type-equiv
/** The comparison of one listed file's turn-start and turn-end contents, computed when asked for. */
type WorkspaceFileDiff =
  | {
    kind: 'text'
    /** The listed file's `path`. */
    path: string
    /** The listed file's `display`. */
    display: string
    /** Whether the file existed at turn start. */
    before: boolean
    /** Whether the file existed at turn end. */
    after: boolean
    /** Hunks in file order; empty when both sides hold the same lines. */
    hunks: WorkspaceDiffHunk[]
    /** True when the line comparison exceeded the plugin's `diffTimeoutMs` and every line is shown as replaced. */
    coarse: boolean
  }
  /** A side git reported as binary or that holds a NUL byte; no lines are served. */
  | { kind: 'binary'; path: string; display: string }
  /** A side larger than the plugin's `maxFileBytes`; no lines are served. */
  | { kind: 'oversized'; path: string; display: string }
```

## `WorkspaceChanges` — 要約と比較を配信する Host サービス

```ts type-equiv
/** Serves the summaries and file comparisons the recorder keeps for live Sessions. */
interface WorkspaceChanges {
  /**
   * The summary announced by one `workspace/changes` event.
   * @param sessionId - the Session that appended the event.
   * @param seq - the event's sequence number.
   * @returns the summary, or undefined once its Session was disposed or when this Host never recorded it.
   */
  summary(sessionId: SessionId, seq: number): WorkspaceChangesSummary | undefined
  /**
   * Compare one listed file's contents at turn start and turn end.
   * @param sessionId - the Session that appended the event.
   * @param seq - the event's sequence number.
   * @param index - the file's index in the summary's `files`.
   * @param signal - cancels the reads.
   * @returns the comparison, or undefined once its Session was disposed, when this Host never recorded it, or when no file has that index.
   * @throws when a snapshot read fails for a live Session.
   */
  diff(sessionId: SessionId, seq: number, index: number, signal: AbortSignal): Promise<WorkspaceFileDiff | undefined>
}
```

## 永続イベントと配信されるサマリー

`tool-present` は `deliverables/presented: { turn; callId; files: PresentedFile[] }` を `SessionEventMap` に宣言マージし、成功した最終 `present` 結果ごとに 1 回追記する。`workspace-changes` は `workspace/changes: { turn }` をマージし、トップレベルの turn が停止したときに追記する。そのイベントが告知したサマリーはログにはなく、Session が破棄されるまで、イベントのシーケンスに対して `workspaceChanges.summary(sessionId, seq)` が返す。そのため Host の再起動後に開き直した会話には、以前の turn の変更ファイル・カードがない。`workspaceChanges.diff(sessionId, seq, index, signal)` は同じ条件で 1 つのリスト・ファイルを比較する。同じ turn の後のイベントは前のものを置き換えるので、クライアントは最新だけを保持する。生成された[永続化カタログ](../../../docs/persistence-catalog.md#deliverablespresented--log-only)が両方の宣言箇所を記録する。どちらのイベントもモデルには届かない。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。dispatch モードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxworkspacechanges--workspacechanges"></a>

### `ctx.workspaceChanges` — `WorkspaceChanges`

レコーダーがライブ Session のために保持するサマリーとファイル比較を配信する。

```ts cordis-catalog
/**
 * The summary announced by one `workspace/changes` event.
 * @param sessionId - the Session that appended the event.
 * @param seq - the event's sequence number.
 * @returns the summary, or undefined once its Session was disposed or when this Host never recorded it.
 */
summary(sessionId: SessionId, seq: number): WorkspaceChangesSummary | undefined

/**
 * Compare one listed file's contents at turn start and turn end.
 * @param sessionId - the Session that appended the event.
 * @param seq - the event's sequence number.
 * @param index - the file's index in the summary's `files`.
 * @param signal - cancels the reads.
 * @returns the comparison, or undefined once its Session was disposed, when this Host never recorded it, or when no file has that index.
 * @throws when a snapshot read fails for a live Session.
 */
diff(sessionId: SessionId, seq: number, index: number, signal: AbortSignal): Promise<WorkspaceFileDiff | undefined>
```

型: [SessionId](core.md)

ソース: [`packages/deliverables/workspace-changes/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/deliverables/workspace-changes/src/types.ts)
<!-- END GENERATED cordis-surface -->
