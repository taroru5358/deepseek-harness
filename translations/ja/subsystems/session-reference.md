# Session 参照

日本語 | [English](../../../docs/subsystems/session-reference.md) | [中文](../../../docs/subsystems/session-reference.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/session-reference.md](../../../docs/subsystems/session-reference.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

ホスト裏付けのファイル発見と、構造化されたクロスセッション参照リクエストおよび準備済みメッセージコンテキスト。[ファイル参照契約](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/context/file-reference)はパスのみの補完レコードと文法を所有する。[セッション参照契約](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/context/session-reference)は正規 URI、現在サーフェスプロジェクション、タグセーフな JSON とバイト保持、安定したエラー、信頼されないモデルプロンプトを定義する。host アダプターは UI のメンション構文を agent コアに渡す代わりにこれらの型を使う。

ソース: [`packages/context/file-reference/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/context/file-reference/src/types.ts) · [`packages/context/session-reference/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/context/session-reference/src/types.ts)

## ファイル候補

`FileReferenceCandidate` はパスのみの発見結果である。宛先の agent が作業ディレクトリのスコープを供給し、provider はファイル内容を読まずにランキングと名前空間アクセスを決める。

```ts type-equiv
/** One path-only completion candidate inside the target session cwd. */
interface FileReferenceCandidate {
  /** User-facing path accepted by normal prompts and filesystem tools. */
  path: string
  /** Directories keep completion open; files finish the mention. */
  kind: 'file' | 'directory'
}
```

## 入力と候補

`SessionReferenceInput` は host 非依存の選択である。id が権威であり、label はスナップショットに運ばれる表示メタデータである。

```ts type-equiv
/** One source session selected by a host. */
interface SessionReferenceInput {
  /** Opaque source session identity. */
  sessionId: SessionId
  /** Optional user-facing mention label. */
  label?: string
}
```

`SessionReferenceCandidate` は host 向けの発見出力である。その `label` は存在すれば最新の Session タイトルを使い、任意の表示テキストはサブエージェントの永続的な作成ラベルを優先する。フィルタリングは Session id と cwd と並べて両方を検索し、トランスクリプトテキストは決して検索しない。Remote 候補の正規メンションは存在すれば表示テキストを使う。

```ts type-equiv
/** One host-facing candidate from exact session metadata. */
interface SessionReferenceCandidate {
  /** Opaque source session identity. */
  sessionId: SessionId
  /** Latest log-backed title, falling back to the opaque session id. */
  label: string
  /** Display and canonical-mention text, preferring a subagent's durable creation label over {@link label}. */
  displayTitle?: string
  /** Source session working directory, when recorded. */
  cwd?: string
  /**
   * True when {@link SessionReferenceCandidate.cwd} is recorded and equals the
   * requesting agent's. Hosts that only surface a distinguishing location
   * read this instead of comparing paths they never received.
   */
  sameWorkspace: boolean
  /** Source session creation time in Unix epoch milliseconds. */
  createdAt: number
}
```

`sessionReferenceResolver/candidates` Remote メソッドは同じ発見をブラウザコンシューマーに提供し、各候補の正規プロンプトメンションを付加する。

```ts type-equiv
/** One discovery candidate carrying its canonical prompt mention. */
interface SessionReferenceMentionCandidate extends SessionReferenceCandidate {
  /** Canonical `@[label](dsh-session:…)` mention serialized into the prompt draft. */
  mention: string
}
```

## 準備済みメッセージ

準備は読み取り可能な現在のメッセージ内容を保持し、最大 1 つの集約されたコンテキストを返す。その永続ソースレコードは `capturedThroughSeq` を参照先 Session の元の生成における座標として保持し、包含する Session の seq として再解釈することは決してない。`capturedFormatVersion` はその生成を記録し、不在はリリースされたフォーマット v0 を意味する。

```ts type-equiv
/** Durable source session, cited event seqs, and snapshot facts for prepared cross-session context. */
interface SessionReferenceSource {
  kind: 'session-reference'
  /** Material lifted out of another session's log (`recall` context form). */
  form: 'recall'
  version: 1
  references: {
    sessionId: string
    label: string
    /** Source Session format generation; absence identifies version 0. */
    capturedFormatVersion?: number
    capturedThroughSeq: OptionalSessionSeq
    compacted: boolean
    originalMessages: number
    retainedMessages: number
    omittedMessages: number
    omittedBytes: number
    truncated: boolean
    inputIndex: number
  }[]
}
```

```ts type-equiv
/** Direct message content and optional referenced-session context. */
interface PreparedReferencedMessage {
  /** Readable message content after host mention tokens are removed. */
  content: ContentBlock[]
  /** Aggregated untrusted snapshot, absent when the message has no references. */
  additionalContext?: UserMessage
}
```

## エラー

`SessionReferenceError.code` は無効な設定または入力、自己参照、件数制限、ソース読み取り失敗、予算失敗、キャンセルを分離する。host プロトコルはプロンプトバイトを検査せずに、これらのコードを自身のエラーエンベロープにマッピングする。

```ts type-equiv
/** Stable failure codes exposed to host adapters. */
type SessionReferenceErrorCode =
  | 'SESSION_REFERENCE_INVALID_CONFIG'
  | 'SESSION_REFERENCE_INVALID_REFERENCE'
  | 'SESSION_REFERENCE_SELF_REFERENCE'
  | 'SESSION_REFERENCE_TOO_MANY'
  | 'SESSION_REFERENCE_READ_FAILED'
  | 'SESSION_REFERENCE_BUDGET_EXCEEDED'
  | 'SESSION_REFERENCE_CANCELLED'
```

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxfilereferences--filereferenceservice-abstract-seam"></a>

### `ctx.fileReferences` — `FileReferenceService`（抽象 seam）

キャンセル可能なファイル参照発見のための host capability。

```ts cordis-catalog
/**
 * List file and directory candidates for one agent's working directory.
 * @param agent - target agent whose session cwd bounds discovery.
 * @param query - path text following `@` or `@"`.
 * @param signal - caller cancellation.
 * @returns deterministic path-only candidates.
 */
abstract list( agent: Agent, query: string, signal: AbortSignal, ): Promise<FileReferenceCandidate[]>
```

型: [Agent](core.md)

ソース: [`packages/context/file-reference/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/context/file-reference/src/index.ts)

<a id="ctxsessionfilereferences--sessionfilereferences"></a>

### `ctx.sessionFileReferences` — `SessionFileReferences`

合成されたファイル参照 provider 上の host Remote アダプター。

```ts cordis-catalog
/**
 * List file and directory candidates for one Agent's working directory.
 * @param agent - target Agent resolved from the Session identity on the wire.
 * @param query - path text following `@` or `@"`.
 * @param signal - caller cancellation.
 * @returns deterministic path-only candidates from the composed provider.
 */
@Remote list( agent: Agent, query: string, signal: AbortSignal, ): Promise<FileReferenceCandidate[]>
```

型: [Agent](core.md)

ソース: [`packages/api/session-controller/src/file-references.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/session-controller/src/file-references.ts)

<a id="ctxsessionreferenceresolver--sessionreferenceresolver"></a>

### `ctx.sessionReferenceResolver` — `SessionReferenceResolver`

不変のクロスセッション・メッセージコンテキストを準備する厳密読み取りコンシューマー。

```ts cordis-catalog
/**
 * List reference candidates, ranked by working-directory affinity.
 *
 * Discovery runs at keystroke rate, so titles and subagent labels only ever
 * come from projection reads; sessions without either fall back to their id.
 * @param agent - target agent; self is excluded and its cwd drives ranking.
 * @param query - optional case-insensitive session-id/cwd/title/display-title substring.
 * @param limit - optional positive result cap.
 * @param signal - optional cancellation boundary for host autocomplete teardown.
 * @returns candidates with canonical mention labels and presentation titles.
 */
async listCandidates( agent: Agent, query: string = '', limit: number = this.config.candidateLimit, signal?: AbortSignal, ): Promise<SessionReferenceCandidate[]>

/**
 * Remote face of {@link listCandidates}: the configured candidate limit
 * applies, and every candidate carries the canonical mention a host inserts
 * into the prompt draft.
 * @param agent - target agent; self is excluded and its cwd drives ranking.
 * @param query - optional case-insensitive session-id/cwd/title substring.
 * @param signal - caller cancellation.
 * @returns mention-carrying candidates in rank order.
 */
@Remote('candidates') async remoteExportCandidates( agent: Agent, query: string, signal: AbortSignal, ): Promise<SessionReferenceMentionCandidate[]>

/**
 * Snapshot all references for one accepted direct message and return one aggregated durable context.
 * Automatic budgets use the context capacity of `route`.
 * Missing model capacity or adapter uses 64 KiB; other metadata lookup failures and cancellation reject preparation.
 * Truncated previews include omission facts and a full-snapshot spill locator, or an explicit unavailable notice.
 * Cancellation prevents context publication, including when storage completes after cancellation.
 * @param agent - target agent; references to it are rejected.
 * @param content - already host-normalized readable message content.
 * @param references - structured source sessions in mention order.
 * @param route - provider and model whose context capacity sizes the previews.
 * @param signal - optional cancellation boundary for the active turn.
 * @returns detached content and optional referenced-session context.
 */
async prepare( agent: Agent, content: ContentBlock[], references: SessionReferenceInput[], route: Readonly<Pick<LlmCallConfig, 'provider' | 'model'>>, signal?: AbortSignal, ): Promise<PreparedReferencedMessage>
```

型: [Agent](core.md) · [ContentBlock](llm-streaming.md) · [LlmCallConfig](llm-streaming.md)

ソース: [`packages/context/session-reference/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/context/session-reference/src/index.ts)
<!-- END GENERATED cordis-surface -->
