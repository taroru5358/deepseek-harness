# コンパクション

日本語 | [English](../../../docs/subsystems/compaction.md) | [中文](../../../docs/subsystems/compaction.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/compaction.md](../../../docs/subsystems/compaction.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

コンパクション seam — bash のように分割された [capability seam](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-06-13-capability-seams.md): Service Definition（[dsh-compaction](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/compaction/compaction)、`ctx.compaction`）、Service Provider（[dsh-compaction-basic](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/compaction/compaction-basic) などのバックエンド）、そして人間の Consumer（[dsh-command-compact](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/compaction/command-compact)）である。コンパクションは**1 つの任意 capability** であり、agent-loop の背骨の一部ではない。したがってその語彙は [core.md](core.md) ではなくここにある。トークナイザーまたはテンプレートに基づくバックエンドは、同じインターフェースを実装する兄弟パッケージである。bash と異なり、このインターフェースは必然的に `dsh-session` と `dsh-llm` に依存する。その動詞は agent が所有する `Session` に作用し、永続サマリーイベントは `ContentBlock` 語彙を使う（[compaction capability-seam リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/compaction/compaction/README.md) を参照）。

ソース: [`packages/compaction/compaction/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/compaction/compaction/src/types.ts)

## `compaction/*` セッションイベント

コンパクションは宣言マージによって 3 つのイベント型で [`SessionEventMap`](session.md) を拡張する。3 つとも**ログ専用**である — ロック、サマリー、選択範囲、シャドーされたイベント seq、トークン数、モデル呼び出しを記録するが、サーフェスには参加しない。`SurfaceEventType` は意図的に拡張されない（メッセージを生成するイベントだけがモデルに届く）。そのためサマリー自体は、`surfaceOp: { op: 'replace', startSeq, endSeq }` を持つ別の `user/message` に乗る — サマリーコンパクションが行う唯一のサーフェス変更である。[歴史的 Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/archived/feature/2026-06-18-compaction-capability-seam.md) は `user/message` を再利用する根拠を記録している。

| イベント | ペイロード | 役割 |
|---|---|---|
| `compaction/start` | `{ turn }` | ログに記録されたロックを取得する。数値は開いている自動 turn を識別し、`null` は単独の手動試行を識別する |
| `compaction/summary` | `{ summary, rawOutput?, llmStreamCall?, shadowedRange, shadowedSeqs, shadowedTokenCount, provider, model, maxTokens?, usage? }` | 安全なサマリープロジェクション、任意の完全な provider 出力と usage、結果の生成がこのコンテキストの `ctx.llm.stream()` を通じたちょうど 1 回の呼び出しを消費したときの `llmStreamCall: true` マーカー（完全な `rawOutput` を要求する）、シャドーされたサーフェス境界ペア（`start`/`end` seq — 数値区間ではなく位置スパン）、サーフェス順のシャドーされた seq、推定トークン数、および要約呼び出しのエンベロープ（`provider`、`model`、適用された場合はその生成上限） — ワンショットリクエストがログ + コードから再構築できるようにログに記録される（再構築可能性の Agent Note）。マークのない `rawOutput` は呼び出し経路を識別しない |
| `compaction/end` | `{ turn, error? }` | 同じ数値または null の所有者でロックを解放する（`error` は失敗した試行を記録する） |

ロックは**操作全体**を囲む: 最初に `compaction/start` が追記され、次に要約、`compaction/summary` レコード、`user/message` の置換がすべて着地し、その後にだけ `compaction/end` が来る。最後にロックを解放することで、操作途中のクラッシュは検出可能な孤児ロック（対応する `compaction/end` のない `compaction/start`）になり、コンパクションが完了したと偽って主張する `compaction/end` にはならない。

マーカーはロックの時点であり、排他的なコンテナではない。要約が保留中の間に、無関係なアイドル注入が単独の手動開始と終了の間に現れることがある。手動パスは選択した位置スパンだけを再検証するため、その注入されたコンテキストは置換チェックポイント後も残る。ライブの未対応 start はすべてのエントリーポイントをブロックする。より新しい `session/end-seed` より前の未対応 start は、以前のライフサイクルからの古い証拠であり、無視される。

これらのバリアントは `declare module '@deepseek-ai/dsh-session/types'` ブロック内でマージされるため、他のサブシステムページのトップレベル型とは異なり、ドリフトチェック付きの ` ```ts type-equiv ` ブロックとしては貼り付けられない（`verify-type-equiv` エクストラクターは名前でトップレベル宣言だけに一致する）。上のペイロード表がカタログエントリーであり、権威あるフィールドについてはソースリンクを参照。

<a id="image-offload"></a>
## 画像オフロード

`compaction-image-offload` は `image/offload` 宣言とその純粋なメッセージプロジェクションを所有する。各ターゲットは現在の入力ノードと正確な深さ優先の画像出現を識別する。イベントはノードとメッセージの同一性を保持し、`surfaceOp` を持たない。[パッケージ README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/compaction/compaction-image-offload/README.md) は復旧ポリシー、登録、デタッチされたリプレイを所有する。

```ts type-equiv
/** Exact input-image occurrences selected by one durable offload decision. */
interface ImageOffloadTarget {
  /** Current message-producing event containing these occurrences. */
  seq: SessionSeq
  /** Zero-based depth-first image indexes within the immutable message. */
  imageIndexes: number[]
}
```

## `CompactionResult`

成功したコンパクションが呼び出し元に返すもの: 簿記イベントの seq、安全なサマリープロジェクション、シャドーされた範囲と seq、推定トークン数。

```ts type-equiv
/** Result of a successful compaction operation. */
interface CompactionResult {
  /** Stable identity shared by this compaction's complete durable lifecycle. */
  compactionId: CompactionId
  /** Human command that initiated this compaction, when it was manual. */
  sourceCommandId?: CommandId
  /** The seq of the appended `compaction/start` event. */
  startSeq: SessionSeq
  /** The seq of the appended `compaction/summary` event. */
  summarySeq: SessionSeq
  /** The seq of the appended `compaction/end` event. */
  endSeq: SessionSeq
  /** The summary content blocks produced by the backend. */
  summary: ContentBlock[]
  /**
   * The surface-boundary pair that was shadowed: the seqs of the first
   * (`start`) and last (`end`) surface nodes of the replaced range. A
   * surface-POSITION span, not a numeric seq interval — after a prior replace
   * lands a fresh high-seq summary node at an older range's position, `start`
   * can be GREATER than `end`. {@link CompactionResult.shadowedSeqs} is the
   * authoritative set of shadowed nodes, in surface order.
   */
  shadowedRange: { start: SessionSeq; end: SessionSeq }
  /** The seqs of all shadowed surface nodes, in surface order. */
  shadowedSeqs: SessionSeq[]
  /** Estimated token count of the shadowed content. */
  shadowedTokenCount: number
}
```

## サービス

自動の呼び出し元はポリシーが実行される理由を述べる。実装は確認されたオーバーフローを通常のプレッシャーよりも積極的に扱ってよい。

```ts type-equiv
/** Why automatic policy is asking a backend to consider compaction. */
type CompactionTrigger = 'pressure' | 'context-overflow'
```

`CompactionEngine` は、自動の `pressure` または `context-overflow` ポリシー用の `compactIfNeeded(agent, trigger, signal)`、プレッシャー未満でも 1 回の有用なアイドルセッション削減を行う `compactNow(agent, signal)`、明示的な包含的サーフェス範囲用の `compactRegion(...)` を公開する。`compactNow()` は turn 間の agent メンテナンスとして実行され、有用な範囲が存在しないときは書き込まずに `null` を返し、要約前に単独の `turn: null` ブラケットを記録し、後続のキュー済みプロンプトが新しいサーフェスから派生しうる前にクローズされた試行をフラッシュする。すべてのバックエンドは `compactCheckpointSource(compactionId, sourceCommandId?)` で置換 `user/message` ソースを生成する。クライアントと wire のコンシューマーは、cordis に依存しない `@deepseek-ai/dsh-compaction/checkpoint` サブパスからそのコンストラクター、`CompactionCheckpointSource`、`isCompactCheckpointSource()` をインポートし、パッケージルートは host コンシューマー向けにそれらを再エクスポートする。必須のトランザクション同一性が置換チェックポイントを相関させ、述語は認識を特定のバックエンドから独立させる。実装は供給されたシグナルを要約に転送しなければならない。この seam は価格 API を所有しない。シングルトン [`ctx.tokenMeter`](token-meter.md) が推定とリプレイを直接所有し、`dsh-compaction-basic` が保持、イベント順序付け、ルーティングされた要約呼び出し、およびそれらの設定を所有する。

想定される手動の失敗は `ManualCompactionErrorCode` を使う:

```ts type-equiv
/** Expected failure classes for an explicit idle-session compaction request. */
type ManualCompactionErrorCode =
  | 'busy'
  | 'cancelled'
  | 'changed'
  | 'summary'
  | 'commit'
  | 'persistence'
```

`changed` と `summary` は失敗した試行をサマリー置換なしでクローズして永続化する。復旧中に記録された画像の省略は有効なまま残る。`commit` は部分的な変更に続くことがある。`persistence` はインメモリのブラケットが閉じたがフラッシュが失敗したことを意味する。キャンセルは別であり、必要なクリーンアップ後に正確な abort 理由を投げる。

プレッシャーコンパクションは、コミット済みリクエストヘッダーに対して `agent/pre-step` で、またヘッダーがログに記録される前に変更された宛先に対して `agent/request` で実行される。モデル切り替えのサマリーは、明示的なサマリーターゲットが設定されていない限り以前のルートを保持し、失敗した切り替えはそのヘッダーと完了した削減を保持する。プレッシャーまたは正規のオーバーフローが条件を満たすと、compaction-basic は範囲選択の前に任意の [`ctx.toolResultPruner`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/compaction/compaction-tool-result-pruner/README.md) を呼び出し、`ctx.tokenMeter` を通じて再測定し、サマリーなしでサーフェスを進めることができる。失敗したリクエストの復旧は、失敗したステップが閉じた後に `agent/request-error` を通じて実行され、たとえ後続のサマリー処理がプルーニング後にスローしても、サーフェス置換の世代が進む場合にのみ retry アクションを返す。キャンセルは依然として優先される。リージョン境界はツール呼び出し/結果のペアリングを保持するが、turn 全体は保持しないため、1 つの巨大な turn の早期に閉じたステップをコンパクションできる。`dsh-compaction-basic` はしきい値、保持テールポリシー、オーバーフロー上限、失敗処理を所有する。

Service Definition は、seq の前後でのツール呼び出し/結果ペアリングチェック用に `toolPairingBalancedBefore(session, seq)` と `toolPairingBalancedAfter(session, seq)` をエクスポートする。どちらも現在のサーフェスメンバーシップを検証し、欠落した seq と孤児の結果を拒否する。[パッケージ契約](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/compaction/compaction/README.md#tool-pairing-boundaries) がそれらのキャッシュ動作を定義する。

## ツール結果プルーニングの結果

任意のツール結果プルーニングサービスは、各永続コンテンツ置換と集約された Unicode コードポイント削減を報告する。その公開結果型は [`compaction-tool-result-pruner/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/compaction/compaction-tool-result-pruner/src/types.ts) にある。

```ts type-equiv
/** Cited source event and size accounting for one landed surface replacement. */
interface PrunedEntry {
  /** Full-fidelity tool-result event shadowed by the replacement. */
  readonly originalSeq: SessionSeq
  /** Newly appended pruned tool-result event. */
  readonly replacementSeq: SessionSeq
  /** Tool call shared by the original and replacement. */
  readonly callId: ToolCallId
  /** Original text size in Unicode code points. */
  readonly charsBefore: number
  /** Replacement text size in Unicode code points. */
  readonly charsAfter: number
}
```

```ts type-equiv
/** Aggregate outcome of one stable-surface pruning pass. */
interface PruneResult {
  /** Replacements in the snapshotted surface order. */
  readonly pruned: readonly PrunedEntry[]
  /** Total Unicode code points removed across replacements. */
  readonly charsRemoved: number
}
```

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxcompaction--compactionengine-abstract-seam"></a>

### `ctx.compaction` — `CompactionEngine`（抽象 seam）

抽象コンパクションサービス。実装はトリガーポリシー、保持、要約を所有し、別の測定サービスを消費してよい。成功した実行は選択されたサーフェススパンを 1 つのサマリーノードで置換し、同じセッションの並行コンパクションを防ぐ。置換される user メッセージはトランザクション同一性を持つ compactCheckpointSource を使うため、コンシューマーはバックエンドから独立してそれを認識し相関させられる。コンテキストごとに 1 つの実装を `ctx.compaction` としてロードする。

```ts cordis-catalog
/**
 * Consider automatic compaction for one explicit trigger. Pressure policy
 * uses the latest durable routed request, while context-overflow policy may
 * force a useful balanced reduction even below the normal threshold. Return
 * `null` when no safe range can be compacted. A single oversized retained
 * unit or request envelope cannot be repaired through surface compaction.
 *
 * @param agent - agent context owning the session surface and routing options.
 * @param trigger - normal pressure or provider-confirmed context overflow.
 * @param signal - cancellation signal; model-backed implementations must forward it.
 * @returns the compaction result, or `null` if no compaction was needed.
 */
abstract compactIfNeeded( agent: CompactionAgentContext, trigger: CompactionTrigger, signal: AbortSignal, ): Promise<CompactionResult | null>

/**
 * Explicitly compact useful history even below automatic pressure thresholds.
 * Implementations synchronously start an idle task before any asynchronous
 * work, select a useful range without writing on a no-op, then
 * append a standalone `compaction/start` before summarization. That durable
 * marker is the compaction lock until one `compaction/end` attempt. Later waking
 * prompts remain accepted in FIFO order and start only after the optional
 * durability checkpoint and idle-task settlement. Context injected while the
 * summary runs may sit between the marker pair; only the selected span must
 * remain stable.
 *
 * @param agent - idle agent whose durable history should be compacted.
 * @param signal - cancellation scoped to this compaction request.
 * @param sourceCommandId - initiating command identity for a manual compaction.
 * @returns the compaction result, or `null` when no safe useful range exists.
 * @throws {@link ManualCompactionError} for expected busy, agent-cancellation,
 * changed-span, summarization/shrink, commit-stage, or persistence failures;
 * an aborted request preserves its exact abort reason. Failed attempts remain
 * visible in the log.
 */
abstract compactNow( agent: ManualCompactAgentContext, signal: AbortSignal, sourceCommandId?: CommandId, ): Promise<CompactionResult | null>

/**
 * Forcibly compact a range of surface nodes into a single summary node.
 * `start` and `end` name an inclusive span by surface position, not numeric seq
 * order; replacements can make visible seqs non-monotonic. Both edges must be
 * balanced so assistant tool calls remain paired with their results. A model-
 * backed implementation forwards cancellation and rejects active, missing,
 * reversed, or unbalanced ranges. The target session is `agent.session`.
 * Its replacement user message must use {@link compactCheckpointSource} with
 * the transaction's `CompactionId`.
 * Use {@link toolPairingBalancedBefore} and {@link toolPairingBalancedAfter}
 * for the edge checks.
 *
 * @param start - first surface seq, inclusive.
 * @param end - last surface seq, inclusive.
 * @param agent - context whose session is mutated and whose routing options guide summarization.
 * @param signal - optional cancellation; model-backed implementations must forward it.
 * @throws when compaction is active or the range is missing, reversed, or unbalanced.
 * @returns the appended event seqs, summary, replaced range, and token accounting.
 */
abstract compactRegion( start: SessionSeq, end: SessionSeq, agent: CompactionAgentContext, signal?: AbortSignal, ): Promise<CompactionResult>
```

型: [CommandId](commands.md) · [SessionSeq](session.md)

ソース: [`packages/compaction/compaction/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/compaction/compaction/src/index.ts)

<a id="ctxtoolresultpruner--toolresultpruner"></a>

### `ctx.toolResultPruner` — `ToolResultPruner`

現在のツール結果サーフェスノードに対する決定論的な先頭/中間/末尾プルーニング。

```ts cordis-catalog
/**
 * Measure text content in Unicode code points; non-text blocks cost zero.
 * @param blocks - tool-result content to measure.
 * @returns total Unicode code points across text blocks.
 */
measureContent(blocks: readonly ContentBlock[]): number

/**
 * Replace an over-budget text middle while retaining rich-block order.
 * Text slicing is by Unicode code point, not UTF-16 code unit, so a retained
 * boundary cannot split a surrogate pair. Grapheme clusters may still split.
 * @param blocks - original tool-result content.
 * @returns pruned content, or `null` when the text is within budget.
 */
pruneContent(blocks: readonly ContentBlock[]): ContentBlock[] | null

/**
 * Prune every over-budget tool result from one stable current-surface snapshot.
 * Each replacement preserves the complete event data except for `content`,
 * cites the shadowed node so replay can recover the replacement input, and is
 * immediately preceded by a `compaction/prune` shadow-price event pricing the
 * shadowed node through the injected token meter, so pure consumers can
 * subtract it without per-node state.
 * @param session - session whose current surface is rewritten.
 * @returns landed replacements and aggregate Unicode-code-point savings.
 * @throws when the session rejects a replacement; replacements committed
 * earlier in the pass remain durable.
 */
pruneSession(session: Session): PruneResult
```

型: [ContentBlock](llm-streaming.md) · [Session](session.md)

ソース: [`packages/compaction/compaction-tool-result-pruner/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/compaction/compaction-tool-result-pruner/src/index.ts)

<a id="compaction-events"></a>

### `compaction/*` イベント

<a id="compactionsummary-error--waterfall"></a>

#### `compaction/summary-error` — waterfall

選択された入力への永続的な変更を同期的に記録することで、失敗したサマリーリクエストを復旧する。進展を遂げた後にのみ true を返す。provider は再試行前に選択を再導出し再価格付けする。失敗が復旧できない場合は next() を呼ぶ。決定は後続のサマリー失敗やキャンセルを生き残る。

```ts cordis-catalog
/**
 * Recover a failed summary request by synchronously recording a durable
 * change to its selected input. Return true only after making progress;
 * the provider re-derives and re-prices the selection before retrying.
 * Call next() when the failure cannot be recovered. Decisions survive a
 * later summary failure or cancellation.
 * @param payload.session - session containing the selected input.
 * @param payload.sourceEventSeqs - selected message events in request order.
 * @param payload.error - failure thrown by the summarizer.
 * @param payload.signal - optional compaction cancellation signal.
 * @param next - delegate to the next recovery listener.
 * @mode waterfall
 */
'compaction/summary-error'(payload: { session: Session; sourceEventSeqs: readonly SessionSeq[]; error: unknown; signal?: AbortSignal }, next: () => boolean): boolean
```

型: [Session](session.md) · [SessionSeq](session.md)

ソース: [`packages/compaction/compaction/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/compaction/compaction/src/index.ts)
<!-- END GENERATED cordis-surface -->
