# LLM ストリーミング

日本語 | [English](../../../docs/subsystems/llm-streaming.md) | [中文](../../../docs/subsystems/llm-streaming.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/llm-streaming.md](../../../docs/subsystems/llm-streaming.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

[`packages/llm`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/README.md) の会話とストリーミングの型: 永続的な `Message` 値、リクエスト専用のユーザー入力、共有された `ContentBlock` バリアント、完全に組み立てられたモデルリクエスト、生の `StreamChunk` プロトコル、すべてのアダプターが実装しなければならないアダプター契約、および共有アセンブラー。[core パッケージ](core.md)は毎ターンこれらの値を保持してログに記録する。このページはそれらを宣言する。

ソース: [`packages/llm/llm/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/llm/src/types.ts)

<a id="content-blocks-and-messages"></a>

## コンテンツブロックとメッセージ

会話は `Message` の集まりであり、メッセージは型付きの**コンテンツブロック**の配列である。ブロックのユニオンは `ContentBlockMap` から派生する。

ソース: [`packages/llm/llm/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/llm/src/types.ts)

```ts type-equiv
/**
 * Merge-extensible content blocks keyed by `type`. New core blocks must land
 * with adapter, UI, and compaction support. Tool-change blocks belong to
 * developer messages; `projectToolUpdates` selects what each route receives.
 */
interface ContentBlockMap {
  'text': TextBlock
  'reasoning': ReasoningBlock
  'image': ImageBlock
  'file': FileBlock
  'tool-call': ToolCallBlock
  'tool-addition': ToolAdditionBlock
  'tool-removal': ToolRemovalBlock
}
```

ブロックのインターフェース（完全なフィールドはソースを参照）: `TextBlock`（`text`）、`ReasoningBlock`（可視テキストとは区別される思考）、`ImageBlock`（永続的な[画像添付](attachment.md)）、`FileBlock`（リクエスト組み立て時にすべてのルートでハンドルテキストに投影される、永続的な逐語的[ファイル添付](attachment.md)）、`ToolCallBlock`（`id: ToolCallId`、`name`、生の JSON の `arguments`）。ツール結果は `toolCallId`、結果の `content`、任意の `isError` を持つ一級の `ToolResultMessage` 値であり、コンテンツブロックではない。`ContentBlock = ContentBlockMap[ContentBlockType]`。新しいモダリティがマージ拡張可能なマップに属するのは、そのアダプター、UI、コンパクション、永続リプレイの各経路がそれを尊重する場合だけである。開発者向けのツール変更ブロックは、解決されたルート機能に従って投影される。

画像アクセスは、永続的な添付や決定論的なリクエスト画像バージョンではなく、リクエストのシリアライゼーションに属する。`resolveImageAttachmentAccess()` は、添付プロバイダーの任意のホストオブジェクトパスと、現在のツール実行ファイルシステムに対してコンシューマーが提供するマッピングを組み合わせる。結果はそのリクエストでのみ利用可能で、`variantId` には参加しない。

ソース: [`packages/llm/llm/src/content.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/llm/src/content.ts)

```ts type-equiv
/** Execution-world path that model tools can use to read one normalized attachment. */
interface ImageAttachmentAccess {
  /** Absolute path to immutable normalized bytes; callers must treat it as read-only. */
  readonlyPath: string
}
```

ソース: [`packages/llm/llm/src/message.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/llm/src/message.ts)

`Message` は識別子を持ち不変な role/source/content の値である。モデルが生成した assistant メッセージは、生成したプロバイダーとモデルを名指しし、ソースに任意のアダプター専用リプレイデータを持つ:

```ts type-equiv
/** Provider/model identity and adapter-private replay data for an assistant message. */
interface AssistantProviderMetadata {
  /** Provider route that produced the message. */
  provider: string
  /** Provider model id that produced the message. */
  model: string
  /**
   * Lossless-JSON adapter state needed to replay the provider response.
   * `LlmRuntime` exposes it to a target adapter only when that adapter instance
   * currently owns both this historical provider and the target provider.
   */
  replayState?: unknown
}
```

```ts type-equiv
/** Any persisted conversation message, discriminated by its `role`. */
type Message = MessageRoleMap[keyof MessageRoleMap]
```

`DeveloperMessage` は `developer` ロールで、会話順に agent セッションの増分変更を記録する。`ToolAdditionBlock.toolName` は、それを含む Session イベントの履歴ヘッダー参照によって選択された定義を有効化する。`ToolRemovalBlock.toolName` は有効な定義を削除する。どちらのブロックも他のメッセージロールでは拒否される。`deferLoading` は追加レコードを必要とせずに、定義の遅延ロードを独立して要求する。ヘッダー束縛については [Session](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/session/README.md)、プロバイダーサポートの制限については [LLM パッケージ](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/llm/README.md#known-limitations-and-deferred-work) を参照。`ToolUpdate` は、解決済みおよび準備済みのモデルメタデータ上で `in-history` または `addition-only` である。`GenerateOptions.toolHistory` は `ToolHistory` を運ぶ。初期の `tools` と順序付きの `updates` であり、各 update は開発者 `messageId` を歴史的に解決された `additions` に束縛する。ランタイムはこの状態をプロバイダー宣言に投影するが、ログに記録されたヘッダーの有効なツール一覧は変更しない。

メッセージの出所自体がマージ拡張可能な合併型である:

```ts type-equiv
/**
 * Where a message (or injected content) came from, in the harness's own
 * vocabulary. Merge-extensible sum type — each producer declares its own
 * `kind` in its own module; there is no shared catch-all `plugin` kind.
 * Model and tool sources answer their role messages; user messages carry any
 * producer's kind, and consumers fall through unknown kinds.
 */
interface MessageSourceMap {
  user: { kind: 'user' }
  model: ModelMessageSource
  tool: ToolMessageSource
  'system-prompt': SystemPromptMessageSource
}
```

生成元の同一性と提示形式は独立している。`kind` は*誰がこれを生成したか*に答え、任意の `form` は*これがどの種類の情報か*に答える。提示方法はコンシューマーが決める。複数の生成元が 1 つの form を共有してよく、1 つの生成元がセッション中に複数の form を出してよい。値は意味論的であり、1 つずつ増える。値がないか認識できない場合は、文書化されたデフォルトを使い、不透明なコンテンツとして提示する:

```ts type-equiv
/**
 * The kind of information in producer-supplied context, declared by the
 * producer in the same `MessageSource`.
 *
 * `MessageSource.kind` answers *who produced this*; `form` answers *what kind
 * of thing it is*, and the two axes are deliberately independent — several
 * producers share one form, and one producer may emit more than one form over
 * a session.
 *
 * The vocabulary is SEMANTIC, never visual: a value states that the content is
 * a file's instructions or a catalog of available items, and a consumer decides
 * what that looks like. Colors, icons, ordering, and collapse defaults are the
 * consumer's business and must not enter this union. It grows one value at a
 * time as producers gain the structured fields their form needs; an absent or
 * unknown value is the documented default, presented as opaque content.
 */
type ContextForm =
  /** Instructions read out of workspace files the model is expected to follow. */
  | 'instructions'
  /** A catalog of items available in this session, republished as it changes. */
  | 'catalog'
  /** Current state, where a later snapshot from the same producer supersedes an earlier one. */
  | 'snapshot'
  /** A one-off account of something that just happened; it supersedes nothing. */
  | 'notice'
  /** A message another agent addressed to this one. */
  | 'relay'
  /** Material lifted out of another session's log, possibly reduced on the way in. */
  | 'recall'
```

```ts type-equiv
/** One named contribution to a `snapshot`-form context, in assembly order. */
interface ContextSnapshotSection {
  /** The contributing subsystem's name. */
  readonly name: string
  /** That contribution's model-facing text, exactly as assembled. */
  readonly text: string
}
```

```ts type-equiv
/**
 * Producer-declared {@link ContextForm} and the fields that form requires,
 * mixed into the source types that carry one.
 *
 * Discriminated by `form` so a producer cannot select a form without the
 * fields needed to present it: a `notice` must record its one-line
 * account, a `snapshot` its sections. Omitting `form` stays valid — an
 * undeclared context is the documented default.
 */
type ContextFormed =
  | { readonly form?: never }
  | { readonly form: 'instructions' }
  | { readonly form: 'catalog' }
  | {
    readonly form: 'snapshot'
    /** The named contributions this snapshot assembled, in order. */
    readonly sections: readonly ContextSnapshotSection[]
  }
  | {
    readonly form: 'notice'
    /** One-line account of what happened, shown without expanding the row. */
    readonly summary: string
  }
  | { readonly form: 'relay' }
  | { readonly form: 'recall' }
```

<a id="streamchunk--the-raw-protocol"></a>

## `StreamChunk` — 生のプロトコル

ストリーミング応答は複数の型付きブロック（テキスト、推論、複数のツール呼び出し）をインターリーブする。`index` は各デルタをそのブロックに結びつけ、`block-end` は完全に組み立てられた `ContentBlock` を運ぶので、コンシューマーがデルタを自前で再組み立てする必要はない。これは**閉じた**判別可能合併型である。`type` に対する `switch` は `assertNever` で終わるため、バリアントを追加すると、それを処理する必要のあるすべてのコンシューマーでコンパイルが壊れる。

```ts type-equiv
/**
 * Adapter-private lossless-JSON state for replaying a successful response,
 * carried by a terminal `finish` chunk and stored on the assembled assistant
 * message's model source. Both halves stay opaque to the harness; only the
 * split is shared vocabulary, so assembly can keep stored metadata aligned
 * with stored content without reading either half.
 */
interface ReplayEnvelope {
  /** Response-level adapter-private metadata (ids, native stop reason). */
  response: unknown
  /**
   * Per-block adapter-private metadata, one entry per emitted block in
   * first-seen stream order. When assembly drops a block it drops the entry at
   * the same position; entries whose length does not match the emitted block
   * count discard the whole envelope. An adapter whose metadata is independent
   * of block structure omits this field and the envelope passes through
   * assembly unchanged.
   */
  blocks?: readonly unknown[]
}
```

```ts type-equiv
/**
 * Raw streaming protocol emitted by adapters.
 * Block indexes correlate interleaved deltas, and `block-end` carries the
 * assembled block. Adapters emit usage before the terminal finish and nothing
 * afterward; tool arguments remain raw JSON strings. An adapter implementation
 * may throw, but `LlmRuntime.stream()` normalizes that failure to a terminal
 * `error` or `aborted` finish before exposing it to consumers.
 */
type StreamChunk =
  | { type: 'block-start'; index: number; blockType: ContentBlockType }
  | { type: 'text-delta'; index: number; text: string }
  | { type: 'reasoning-delta'; index: number; text: string }
  | { type: 'tool-call-delta'; index: number; id: ToolCallId; name?: string; argumentsDelta: string }
  | { type: 'block-end'; index: number; block: ContentBlock }
  | { type: 'usage'; usage: TokenUsage }
  | {
    type: 'finish'
    reason: FinishReason
    /** Replay metadata for a successful response; see {@link ReplayEnvelope}. */
    replayState?: ReplayEnvelope
  }
```

<a id="compact-assistant-streams"></a>

## コンパクトな Assistant ストリーム

`AssistantStreamAccumulator` は各 `StreamChunk` を元の安全な整数のタイムスタンプと対にし、`AssistantStreamRecord[]` を生成する。同じブロックに対する連続したテキスト、推論、ツール引数のデルタは、`time0`、正確なタイムスタンプの間隔、元のデルタごとの 1 つの配列エントリーを持つ 1 つのレコードになる。それ以外のチャンクはタイムスタンプ付きの生レコードのままである。この表現は、トークン境界を連結したり、終端・usage・ブロック・失敗・リプレイの事実を落としたりすることなく、繰り返されるイベントエンベロープを取り除く。

`snapshot()` はデタッチされた不変のストリームを返す。`expandAssistantStream()` は、正確なタイミング付きチャンク列を再作成する前に、レコードのキー、メンバー数、インデックス、タイムスタンプ、ツール呼び出しの同一性、ロスレス JSON を厳密に検査する。Session ログは、サーフェス結果に対しては `assistant/message` に、サーフェスメッセージを持たない試行に対しては `assistant/attempt` に、このストリームを埋め込む。

プロセスローカルの `agent/assistant-stream` フレームはライブの提示を運ぶ。永続リプレイと復元検証は依然として埋め込まれた確定結果を展開する。テレメトリー、トークン会計、Host のフォールドはコンパクトなレコードを直接読む。レコードレベルのリーダー（`assistantStreamFirstTokenTime`、`assistantStreamHasVisibleContent`、`assistantStreamHasVisibleText`、`lastAssistantStreamChunk`、`assistantStreamChunks`、`joinAssistantStreamText`、`assembleAssistantStream`、および実行ごとの `runFirstTokenTime` と `runFirstVisibleTime`）は、レコードを 1 回走査して早期終了しながらコンシューマーの問いに答えるため、大きな履歴でも確定ごとのコストは O(members) の展開ではなく O(records) になる（[フォールドの参考](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/llm/README.md)）。`expandAssistantStream()` は、永続境界で読み取るレコードと、すべてのメンバーを必要とするコンシューマーにとって、検証を行う経路であり続ける。

## `LlmFailure`

スローされた、または帯域内の最終アダプター障害はすべて、シリアライズ可能なプロバイダー中立の 1 つのペイロードに正規化される。`providerRetryAfterMs` はプロバイダーが要求した検証済みの正の遅延であり、リトライの決定ではない。`ProviderRequestId` は診断用の不透明なブランド付き文字列である。

```ts type-equiv
/** Serializable provider or transport failure facts; policy decides whether they are retryable. */
interface LlmFailure {
  /** Human-readable provider or transport failure. */
  readonly message: string
  /** Stable provider-neutral machine-routing code. */
  readonly code: string
  /** HTTP status returned by the provider, when available. */
  readonly status?: number
  /** Provider-requested delay in milliseconds, when valid and available. */
  readonly providerRetryAfterMs?: number
  /** Opaque provider-issued request identifier for diagnostics. */
  readonly requestId?: ProviderRequestId
  /**
   * With code `IMAGE_OFFLOAD_REQUIRED`: how many more of the oldest retained
   * image occurrences the route needs offloaded before the same request fits
   * its exact byte accounting. `dsh-compaction-image-offload` records the
   * selected occurrences in an `image/offload` event and retries the step.
   */
  readonly offloadImages?: number
}
```

## リクエスト画像の価格設定

プロバイダーがリクエスト画像に視覚トークンを課金するアダプターは、`LlmAdapter.imageRequestPricing` をオーバーライドしてルートごとの価格を宣言し、`ctx.llm.imageRequestPricing(provider, model)` がコンシューマーのために同期的に解決する。トークンメーターは測定のたびにルーティングされたモデルの価格を解決するため、コンパクションの圧力、保持、範囲選択は、ルーティングされたリクエストが実際に送信する形で画像履歴を価格計算する。DeepSeek アダプターは、保持された各出現を、公開された視覚会計に基づくモデルごとのリクエストターゲットで価格計算し、ログに記録された画像オフロード判断によって選択された各出現をそのプレースホルダーテキストとして価格計算する。一方、完了したリクエストではプロバイダーの usage が権威ある基準であり続ける。

```ts type-equiv
/**
 * Request price of one ordered image occurrence under one exact model route's
 * request projection. Every occurrence resolves to the pair the wire actually
 * carries: provider visual tokens for a retained image, plus the model-visible
 * text sent with or instead of it (request-preview handle, offload placeholder,
 * or text-only substitution). The caller prices `text` with its own text
 * estimator so provider pricing never fixes a text tokenization.
 */
interface LlmImageRequestPrice {
  /** Provider visual tokens for the retained request image; 0 when only text represents this occurrence. */
  visualTokens: number
  /** Model-visible text sent for this occurrence, to be priced by the caller's text estimator. */
  text: string
}
```

```ts type-equiv
/**
 * Provider-side request-image pricing for one exact model route. Implemented
 * by adapters whose provider charges visual tokens; consumers (the token
 * meter) resolve it synchronously per measurement, so implementations must not
 * perform I/O.
 */
interface LlmImageRequestPricing {
  /**
   * Price every image occurrence of one request projection.
   * @param images - surface image blocks in request order, one entry per occurrence; an `offloaded` block
   *   is priced as its placeholder text.
   * @returns one price per occurrence, aligned by index with `images`.
   */
  priceImages(images: readonly ImageBlock[]): readonly LlmImageRequestPrice[]
}
```

## アダプター契約

すべてのアダプターは以下に従わなければならず、すべてのコンシューマーはそれらに依拠してよい:

- **`finish` より前に `usage`、`finish` の後には何もない。** 両方をプロバイダーの end-of-stream マーカーまで遅延させ、末尾の usage のみのチャンクが順序に違反しないようにする。
- **ツール呼び出しの `arguments` はエンドツーエンドで生の JSON 文字列のまま。** 部分的な断片は `argumentsDelta` でストリームされ、解析済みオブジェクトを返すプロバイダーは `block-end` で再文字列化する。
- **認められた 2 つのエラー経路、1 つの `LlmFailure` 型。** 障害は `stream()` から THROW する（トランスポート/プロトコルエラー）**か**、`finish {kind:'error'|'aborted', failure}` でストリームを終了する（プロバイダーの帯域内エラー、ストリーム途中でスローできないアダプター向け）。`LlmError.failure` は同じ `LlmFailure` を運ぶ。呼び出しがアダプターを選択した後、ストリームはスローされた正確な `Error` オブジェクトを保持し、不変の事実と、サービス中の登録の不変リトライポリシーをその呼び出しに関連づける。agent ループは試行ストリームを `assistant/attempt` としてコミットし、失敗したステップを閉じ、エラー、事実、不変の以前のリトライ事実、サービス提供ポリシー、ターンシグナルを `agent/request-error` に提供する。ハンドリングリスナーは、待機した修復の後に `{ kind: 'retry' }` を返す。回復がない場合、構造化された障害がターンエラーになり、その試行に対してサーフェスの Assistant メッセージやツールの副作用はコミットされない。
- **1 回のアダプター呼び出しは 1 回のプロバイダー試行。** アダプターはライブラリのリトライを無効にする。agent レベルの回復は、別の永続的な番号付きターンを開く。直接の `ctx.llm.stream()` 呼び出し元は単一試行のままである。
- **プロバイダーのストールはトランスポートで制限される。** 出荷される両方のリモートアダプターは、5 分のデフォルトを持つ正の有限 `streamIdleTimeoutMs` を公開する。ウォッチドッグはイテレーターの `next()` が未完了の間だけ作動し、リクエスト全体で 1 つの安定したシグナルを使い、自身の期限切れを `TIMEOUT` にマップし、先行する呼び出し元の中断を `ABORTED` のまま保つ。
- **コンテキストオーバーフローには 1 つの正規コードがある。** 両方の DeepSeek アダプターは、`isContextWindowExceededError()` を通じて明示的なプロバイダー詳細を分類し、障害がスローされた HTTP `LlmError` として届いても、帯域内の finish エラーとして届いても `CONTEXT_WINDOW_EXCEEDED` を表面化する。コンシューマーはプロバイダーのテキストではなくコードでルーティングする。
- **空の完了はサイレントな成功ではなく、リトライ可能なエラーである。** 両方のアダプターは、コンテンツブロックを運ばなかった終端の `stop` finish を、正規の `EMPTY_RESPONSE` コードを持つ `finish {kind:'error'}` にマップし、`dsh-llm-retry` はデフォルトでそれをリトライする。
- **すべてのプロバイダー HTTP リクエストはアプリ帰属ヘッダーを運ぶ。** アダプターは `attributionHeaders()`（下記）を送信し、それは `User-Agent` のベースラインであり、ワイヤレベルのテストで証明する。
- **リプレイ状態はアダプター所有であり、その分割は共有される。** 成功した `finish` は `ReplayEnvelope` を運んでよい。不透明な応答レベルのメタデータと、出力されたブロック列に整列した任意のブロックごとのエントリーである。この整列は harness の語彙である。組み立てがブロックを落とすと、同じ位置のエントリーも落とすため、保存されたメタデータは常に保存されたコンテンツを記述する。ループは剪定済みのエンベロープを、組み立てられた assistant メッセージとともに保存する。後のリクエストで、`LlmRuntime` は履歴プロバイダーとターゲットプロバイダーが現在まったく同じアダプターインスタンスに登録されている場合にのみ、その状態を渡す。そのアダプターは状態を検証し、クロスモデルまたはクロスプロバイダーの変換を所有する。他のアダプターは、プライベート状態なしでプロバイダー中立のコンテンツとプロバイダー/モデルフィールドを受け取る。永続コンテンツが権威であり続ける。読み取りアダプターが使えない保存状態は、リクエストを失敗させるのではなく、その 1 つのメッセージを診断付きのプロバイダー中立変換に縮退させる。

## `ResolvedRetryPolicy`

リトライ設定は、ルート登録の前に不変の判別可能合併型へ解決される。normal モードは `mode: 'normal'`、有限の `maxRetries`、`retryableCodes`、および必須の `initialDelayMs`、`maxDelayMs`、`jitterRatio` を持つ。always モードは `mode: 'always'` と同じ必須のバックオフフィールドを持ち、有限の最大値を持たない。プロバイダーポリシーを省略すると、5 回のリトライという normal のデフォルトを使う。階層化された設定は、always モードに切り替えた後も normal 専用の `maxRetries` や `retryableCodes` を保持してよい。リゾルバーはそれらの非アクティブなフィールドを無視し、純粋な always ポリシーを捕捉する。`LlmRuntime.providerRetryPolicy(provider)` は登録済みの値を返し、`llmRetryPolicyOf(stream)` は呼び出しが選択した後にサービス中の登録から捕捉した値を返すため、後のルート破棄や置換が進行中の障害の回復ポリシーを変更することはできない。生成された[設定カタログ](../../../docs/config-catalog.md)は任意の入力フィールドを列挙する。

## `AppIdentity` — アプリの帰属情報

すべてのアダプターがプロバイダーに送信する静的な公開アプリケーション同一性（[`packages/llm/llm/src/attribution.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/llm/src/attribution.ts)）。`attributionHeaders(identity?)` はそれを標準の `User-Agent` ヘッダーにのみマップする。OpenRouter 固有のアプリ帰属ヘッダーは、この契約では意図的にサポートされない。デフォルトの `APP_IDENTITY` はパッケージマニフェストからバージョンを取得する。すべてのフィールドは公開された製品の事実であり、シークレット、パス、セッション ID、ユーザーごとの識別子を含まず、リクエストごとの何かが値に影響してはならない。

```ts type-equiv
/**
 * Static public application identity sent to LLM providers.
 *
 * Every field is a public product fact, safe on every request: no secrets,
 * local paths, session ids, prompt text, or per-user identifiers belong here,
 * and nothing per-request may influence the values.
 */
interface AppIdentity {
  /** `User-Agent` product token (lowercase, hyphenated). */
  product: string
  /** Product version; sourced from package metadata, never hand-copied. */
  version: string
  /** Repository home URL of the app, used as the `User-Agent` comment. */
  url: string
}
```

## `TokenUsage`

呼び出しごとのトークン会計。カウントは**互いに素**である。`inputTokens` は非キャッシュの入力のみであり、キャッシュされた入力は別に報告され、課金対象の入力は 3 つの合計である。プロバイダーがキャッシュヒットを単一のプロンプト合計（DeepSeek の `prompt_tokens`）に折り込むアダプターは、それらを差し引いて戻す。任意の `totalTokens` は、プロバイダーから保持されるか、権威ある集計カウンターから再構成される、プロンプトと出力の正確な合計である。利用できないか矛盾している場合、アダプターはそれを省略する。`reasoningTokens` が存在する場合、それはすでに `outputTokens` に含まれる情報提供の詳細であり、合計に再度加えてはならない。

```ts type-equiv
/**
 * Token accounting for one model call (cache fields are optional).
 *
 * Counts are DISJOINT: `inputTokens` is uncached input only; cached input is
 * reported separately as `cacheReadTokens`/`cacheWriteTokens` (billed input =
 * sum of the three). Adapters whose providers fold cache hits into a total
 * prompt count (DeepSeek's `prompt_tokens`) subtract them out.
 */
interface TokenUsage {
  inputTokens: number
  outputTokens: number
  /**
   * Exact full-call total including aggregate prompt and output tokens.
   *
   * Adapters preserve a provider total or derive it from authoritative
   * aggregate prompt/output counters; they omit it when unavailable or
   * inconsistent.
   */
  totalTokens?: number
  cacheReadTokens?: number
  cacheWriteTokens?: number
  reasoningTokens?: number
}
```

## `BlockAssembler`

`BlockAssembler`（[`packages/llm/llm/src/assembler.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/llm/src/assembler.ts)）は、`StreamChunk` ストリームを `ContentBlock`、usage、finish reason、リプレイ状態に折り戻す単一の共有実装である。ループは生のチャンクをログに記録しながら、同じチャンクをアセンブラーに通し、組み立てられた assistant コンテンツを、それを生成したプロバイダーとモデルとともに保存する。折り込みを再実装せずに組み立て結果を必要とするコンシューマーはこれを使う。

1 つの keep/drop 判断がコンテンツとメタデータをまとめて扱う。`max-tokens` の finish は、切り詰められた呼び出しが実行に安全でないため、すべてのツール呼び出しを落とし、同じ判断が落とされた各位置でリプレイエンベロープのブロックごとのエントリーを剪定する。したがって、組み立てが何を取り除いても、`blocks()` と `replayState` が食い違うことはない。

```ts public-api
/**
 * Incrementally assembles raw {@link StreamChunk}s into complete
 * {@link ContentBlock}s and a final assistant {@link Message}.
 *
 * The agent loop feeds it while logging raw chunks for replay fidelity, then
 * reads `blocks()` / `message()` / `usage` / `finish` once the stream ends,
 * or `interruptedBlocks()` when cancellation cut the stream short.
 *
 * Tolerant of delta-only protocols (no block-start/end); deltas arriving for
 * an index already closed by `block-end` are ignored (malformed stream) so a
 * misbehaving adapter cannot grow memory or corrupt a completed block.
 */
declare class BlockAssembler {
  /**
   * Feed one chunk into the assembly state.
   * @param chunk - the next raw chunk, in stream order.
   */
  push(chunk: StreamChunk): void;
  /**
   * Assemble all blocks seen so far, in stream order.
   * @returns one block per seen index, except that max-token truncation drops
   *   tool calls that cannot be executed safely; an open block assembles from
   *   its accumulated deltas (an unknown block type never closed by `block-end` throws).
   */
  blocks(): ContentBlock[];
  /**
   * Assemble the prefix an interrupted stream can safely finalize: closed and
   * open text/reasoning blocks with non-whitespace content, in stream order.
   * Tool calls are omitted because interruption precedes dispatch; retaining
   * one would require a fabricated result. Open unknown blocks are also omitted.
   * @returns the kept blocks; empty when nothing streamed before the interruption.
   */
  interruptedBlocks(): ContentBlock[];
  /** Usage from the `usage` chunk; undefined until one arrives. */
  get usage(): TokenUsage | undefined;
  /** Finish reason from the `finish` chunk; `{kind: 'stop'}` when the stream ended without one. */
  get finish(): FinishReason;
  /**
   * Replay metadata from the terminal finish chunk, if any, with per-block
   * entries pruned in step with {@link blocks}. Undefined when the envelope's
   * entries do not align with the emitted blocks.
   */
  get replayState(): ReplayEnvelope | undefined;
  /**
   * The assembled assistant message.
   * @param source - provider/model attribution (without the `kind` tag) for the assembled message.
   * @returns a frozen assistant-role message over `blocks()` (same open-block assembly rules).
   */
  message(source: Omit<ModelMessageSource, 'kind'>): AssistantMessage;
}
```

<a id="the-model-request-and-result"></a>

## モデルリクエスト

1 回のモデル呼び出しは、完全に組み立てられた `GenerateOptions` である。アダプターは生の [`StreamChunk`](#streamchunk--the-raw-protocol) ストリームで応答し、コンシューマーはそれを [`BlockAssembler`](#blockassembler) で組み立てる。

ソース: [`packages/llm/llm/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/llm/src/types.ts)

プロバイダーとモデルの検出には、小さなプロバイダー中立の記述子を使う。モデルカタログは助言的なものであり、ルーティングは依然として登録済みプロバイダーをキーにする。

アダプターを登録するとハンドルが返る。ディスポーザーに加えて、ルート集合がユーザー設定可能なプラグインが必要とするアトミックなルート置換である。

```ts type-equiv
/**
 * What {@link LlmRuntime.registerAdapter} returns: the disposer, plus an
 * atomic route replacement for the same adapter instance.
 */
interface AdapterRegistrationHandle {
  /** Release every route this registration currently holds. */
  (): void
  /**
   * Replace this registration's routes with `providers`, keeping the same
   * adapter instance. The candidate set is validated in full first — a
   * conflict with another adapter, an invalid name, or bad provider metadata
   * throws and leaves the current routes untouched — and the swap itself is
   * one synchronous section, so no request can observe a gap. An empty array
   * is legal here (a settings section that emptied holds zero routes while
   * staying registered), unlike an empty initial registration.
   *
   * Throws `LlmError` with code `REGISTRATION_DISPOSED` once the registration
   * has been released: its routes are gone and its disposer has already run,
   * so anything registered afterwards would have no owner left to release it.
   * @param providers - the complete next route set for this registration.
   */
  replace(providers: string[]): void
}
```

```ts type-equiv
/** Display metadata for one registered provider route. */
interface LlmProviderInfo {
  /** Provider route key used by {@link GenerateOptions.provider}. */
  id: string
  /** Human-readable provider name for selectors and diagnostics. */
  name: string
}
```

アダプタープラグインはさらに、`registerConfigurableProviders()` を通じて*実行される可能性がある*ルートを宣言し、各ルートのユーザー設定セクションを指定する。これにより、設定サーフェスはルートが登録される前に休止中のプロバイダーを提示できる。

```ts type-equiv
/**
 * One provider route an adapter plugin can activate through configuration,
 * whether or not the route is currently registered. Configuration surfaces
 * merge this directory with `listProviders()` to offer every configurable
 * provider alongside its live/dormant state.
 */
interface LlmConfigurableProvider {
  /** Provider route key this entry activates when configured. */
  provider: string
  /** Human-readable provider name for configuration surfaces. */
  displayName: string
  /** User-settings namespace whose section configures this provider. */
  settingsNs: string
  /**
   * Path from that namespace's section root to this provider's profile
   * object; empty when the whole section is the profile.
   */
  settingsPath: readonly string[]
  /**
   * Whether the owning adapter knows this route only because configuration
   * declared it — a gateway or self-hosted server it ships nothing about.
   * Absent means the adapter draws no such distinction; false means it does
   * and this route is one of its own. Only the adapter can answer: a stored
   * profile is how a user-added route AND a corrected shipped one both look
   * from outside.
   */
  declared?: boolean
  /**
   * Route endpoint the adapter uses when the profile sets no `baseURL`; the
   * adapter may still send an individual model to that model's own endpoint.
   * Absent when the route has no fixed endpoint.
   */
  defaultBaseURL?: string
  /** Environment variables used to construct a provider endpoint; absent when none are reported. */
  baseURLEnv?: readonly string[]
  /** Authentication the adapter performs when the profile names no credential reference; absent when it reports none. */
  nativeAuth?: LlmNativeAuth
  /** Configuration diagnostic for repair; unaffected models may remain serviceable. */
  error?: string
}
```

設定エディターは、エンドポイントや資格情報参照を未設定のままにするプロファイルに対して、`defaultBaseURL`、`baseURLEnv` からのエンドポイント設定変数、`nativeAuth` を表示する。どちらの変数リストも名前だけを含み、値を含まない。

```ts type-equiv
/** Authentication a provider route performs without a credential reference in its profile. */
interface LlmNativeAuth {
  /** Environment variables the route's authentication reads, in lookup order; empty when it reads none. */
  env: readonly string[]
  /** Whether a stored OAuth sign-in can authenticate the route. */
  oauth: boolean
}
```

カタログエントリーは、セレクターがモデル名の横に表示する、プロバイダー作成の `badge` を運んでよい。これは表示データである。`resolveModelInfo` はそれを省略し、どのモデルリクエストもそれを運ばない。

```ts type-equiv
/**
 * Emphasis of a model badge: `neutral` states a plain fact, `info` highlights
 * the model, `success` marks a benefit, and `warning` asks for attention.
 */
type LlmModelBadgeTone = 'neutral' | 'info' | 'success' | 'warning'
```

```ts type-equiv
/** Short provider-authored label a selector shows beside one advertised model name. */
interface LlmModelBadge {
  /** One literal, or translations keyed by lowercase language id with a required English entry. */
  text: LocalizedText
  /** Emphasis of the label. */
  tone: LlmModelBadgeTone
}
```

```ts type-equiv
/** One adapter-discovered model; catalog membership is advisory, not request validation. */
interface LlmModelInfo {
  /** Provider route that owns this model entry. */
  provider: string
  /** Model id passed to {@link GenerateOptions.model}. */
  id: string
  /** Human-readable model name for selectors. */
  name: string
  /** Optional user-facing distinction from otherwise similar models. */
  description?: string
  /** Accepted request modalities; absent means unknown, while an explicit omission is negative capability. */
  inputModalities?: readonly ModelModality[]
  /**
   * Selector label advertised by catalog discovery; exact-route resolution
   * omits it, and no model request or Session event carries it.
   */
  badge?: LlmModelBadge
}
```

正確性重視のメタデータは、助言的なカタログとは別に解決され、正確なルートにサービスするアダプターが所有する。コンテキスト容量、アダプター呼び出しのデフォルト、推論の選択肢、システムプロンプト更新モードは 1 つの正確なモデル結果を共有するため、コンシューマーは権威あるモデル解決を繰り返さない。`SystemPromptUpdate` は単一の値 `'in-history'` を持つ。モデルは `messages` の任意の位置にある最新の `system` メッセージを完全な有効システムプロンプトとして読むため、agent ループはメッセージ 0 を書き換える代わりに、キャッシュされた履歴の後に変更されたプロンプトを追記できる（[決定ルール](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent-loop/README.md#understand-the-implementation)）。モードがない場合は先頭のシステムメッセージだけが読まれ、`normalizeModelInfo` は他の値を `INVALID_MODEL_INFO` で拒否する。

```ts type-equiv
/** Provider-owned context capacity for one exact provider/model route. */
interface LlmModelContext {
  /** Maximum combined request and response context in tokens. */
  contextWindow: number
}
```

推論エフォートも別の正確なルート機能である。core は識別子にブランドを付けるが、その値を列挙しない。各アダプターが、エフォートの昇順の選択肢、表示名、任意のデプロイデフォルトを所有する。順序は選択可能なコントロールを比較するものであり、予測されるトークン使用量、コスト、レイテンシーではない。

```ts type-equiv
/** Adapter-owned identifier for one model's selectable reasoning effort. */
type ReasoningEffortId = Branded<'ReasoningEffortId'>
```

```ts type-equiv
/** Display metadata for one adapter-owned reasoning effort. */
interface LlmReasoningEffortInfo {
  /** Opaque stable value accepted by {@link GenerateOptions.reasoningEffort}. */
  id: ReasoningEffortId
  /** Human-readable effort name for selectors and diagnostics. */
  name: string
  /** Optional user-facing distinction from otherwise similar efforts. */
  description?: string
}
```

```ts type-equiv
/** Selectable reasoning efforts for one exact provider/model route. */
interface LlmModelReasoningInfo {
  /** Supported efforts from least to greatest selectable reasoning effort, not predicted token use or latency. */
  efforts: readonly LlmReasoningEffortInfo[]
  /**
   * Adapter-configured default materialized into requests when callers omit
   * an effort. Absence preserves the provider's own default.
   */
  defaultEffort?: ReasoningEffortId
}
```

```ts type-equiv
/** Exact-route model metadata resolved by its owning adapter. */
interface LlmResolvedModelInfo extends LlmModelInfo {
  /** Provider-owned context capacity when known. */
  context?: LlmModelContext
  /** Adapter-configured per-request output cap materialized when callers omit one. */
  defaultMaxTokens?: number
  /** Adapter-owned selectable reasoning levels when exposed. */
  reasoning?: LlmModelReasoningInfo
  /** Declared mid-conversation system prompt handling; absent means only a leading system message is read. */
  systemPromptUpdate?: SystemPromptUpdate
  /** Declared mid-conversation tool declaration handling; absent means every request declares the complete tool list. */
  toolUpdate?: ToolUpdate
}
```

補助的な呼び出し元は、永続的な同一性やソースなしでユーザーコンテンツを提供できる。既存の `Message[]` 履歴は有効なリクエスト入力のままである。Session の書き込み、Agent の配信、記録されるタイトルリクエストは、引き続き永続メッセージを必要とする。

```ts type-equiv
/** User input for one LLM request; it has no durable Session identity or source. */
interface RequestUserInput {
  readonly role: 'user'
  readonly content: UserMessage['content']
  readonly id?: never
  readonly source?: never
}
```

```ts type-equiv
/** A durable conversation message or a user input used only for one request. */
type RequestMessage = Message | RequestUserInput
```

```ts type-equiv
/** Auxiliary request purposes declared by their producer packages. */
interface LlmRequestPurposeMap {}
```

```ts type-equiv
/** Declared auxiliary request purpose; ordinary conversation requests leave it absent. */
type LlmRequestPurpose = Extract<keyof LlmRequestPurposeMap, string>
```

```ts type-equiv
/**
 * A single model request, fully assembled.
 * @typeParam Purpose - declared purpose keys for callers; forwarding adapters may accept `string`.
 */
interface GenerateOptions<Purpose extends string = LlmRequestPurpose> {
  /** Registered provider route selecting the adapter instance. */
  provider: string
  model: string
  /** Adapter-owned reasoning effort selected for this exact model. */
  reasoningEffort?: ReasoningEffortId
  /**
   * Ordered conversation messages, exactly as the provider sees them. A
   * loop-built request passes the derived history (dsh-agent-loop), whose
   * leading system-role message carries the system prompt; a hand-built
   * one-shot may include identity-free user inputs.
   */
  messages: RequestMessage[]
  /**
   * System prompt text for one-shot callers; adapters map it to the provider's
   * system slot ahead of `messages`. Loop-built requests leave it undefined.
   */
  system?: string
  /** Tool schemas (adapters map to the provider's `tools` field). */
  tools?: ToolSchema[]
  /** Session-folded tool history used for route projection; omission sends complete declarations without tool updates. */
  toolHistory?: ToolHistory
  temperature?: number
  maxTokens?: number
  /**
   * Stop sequences: generation halts as soon as the model produces any one of
   * these strings (adapters map to the provider's stop field, e.g. OpenAI
   * `stop`). The stop string itself is not included in the output.
   */
  stop?: string[]
  signal?: AbortSignal
  /**
   * Session identity stamped by the loop for request routing. Replay uses it
   * to separate cursors; adapters may map it to model-hidden transport metadata.
   */
  sessionId?: Branded<'SessionId'>
  /**
   * Provider-neutral classification for an auxiliary model call. Adapters may
   * map the purpose to model-hidden transport metadata. Ordinary conversation
   * requests leave it unset.
   */
  purpose?: Purpose
}
```

モデル応答が停止した理由は、マージ拡張可能な理由である。終端のプロバイダー障害は、ストリーミング契約の [`LlmFailure`](#llmfailure) を運ぶ:

```ts type-equiv
/**
 * Why a model response stopped.
 * Merge-extensible so adapters can surface provider-specific reasons.
 */
interface FinishReasonMap {
  'stop': { kind: 'stop' }
  'tool-calls': { kind: 'tool-calls' }
  'max-tokens': { kind: 'max-tokens' }
  'aborted': { kind: 'aborted'; failure: LlmFailure }
  'error': { kind: 'error'; failure: LlmFailure }
}
```

`FinishReason = FinishReasonMap[keyof FinishReasonMap]`。`TokenUsage`（互いに素なキャッシュフィールドを持つ呼び出しごとの会計）は[下記](#tokenusage)で詳述する。

`GenerateOptions.tools` は `ToolSchema` を運ぶ。これはモデルに送信されるツールの JSON スキーマ記述である。これが dsh-tools ではなく dsh-llm で宣言されているのは、まさにループが毎ステップ組み立てるリクエストの一部だからである:

```ts type-equiv
/**
 * JSON-schema description of a tool, as sent to the model.
 *
 * Declared here (not in dsh-tools) because it is part of {@link GenerateOptions};
 * dsh-tools' ToolDefinition and dsh-system-prompt's PromptAssembly both import
 * it from this package.
 */
interface ToolSchema {
  /**
   * Requests deferred loading of the tool definition into model context,
   * independently of whether a tool-addition block records the tool.
   * Uses Anthropic's defer_loading terminology.
   */
  deferLoading?: true
  name: string
  description: string
  /** JSON Schema object for the arguments. */
  parameters: Record<string, unknown>
}
```

モデル向けの `ToolSchema` はワイヤ型であり、それを生成する登録済みの `ToolDefinition`（スキーマ + `execute`）は [tools.md](tools.md) にある。

サーフェスがまだ下書き中のプロバイダーにはルートもカタログもないため、問い合わせは別に記述される。リクエストはユーザーが編集中の下書きを運び、応答はサーフェスが提供しなければならないカタログではなく、サーフェスが採用してよい候補である。

```ts type-equiv
/**
 * One interrogation of a provider endpoint that configuration has not stored
 * yet. Configuration surfaces send the draft a user is still editing, so the
 * request carries the endpoint and credential directly instead of naming a
 * route: a provider being added has no route to name.
 */
interface LlmModelDiscoveryRequest {
  /**
   * Route the draft is editing, when it edits an existing one. A route whose
   * adapter already knows its models answers from that knowledge instead of
   * asking the endpoint — the adapter's own registry is the better answer, and
   * it costs no network call.
   */
  provider?: string
  /**
   * Endpoint to interrogate. Optional because a route the adapter already
   * describes needs none; a route it does not must supply one.
   */
  baseURL?: string
  /** Wire protocol the endpoint speaks, when the draft names one. */
  api?: string
  /** Credential for this interrogation alone; the harness never stores it. */
  apiKey?: string
}
```

```ts type-equiv
/**
 * One model an endpoint reports about itself. Every field but the id is
 * optional because most provider listings disclose an id and nothing else;
 * a surface adopting one of these still owes the capacities its adapter needs.
 */
interface LlmDiscoveredModel {
  /** Model id the endpoint accepts. */
  id: string
  /** Human-readable name when the endpoint supplies one. */
  name?: string
  /** Maximum combined request and response context, when disclosed. */
  contextWindow?: number
  /** Maximum output tokens, when disclosed. */
  maxTokens?: number
  /** Accepted input types when disclosed by the catalog or endpoint; absent means unknown. */
  inputModalities?: readonly ModelModality[]
}
```

### リクエストエンベロープ: `LlmCallConfig` とログに記録されるヘッダー

ループはログに記録された状態から各リクエストを構築する。`EpochHeader` は呼び出し設定を記録し、アダプターのデフォルトによって供給されたフィールドをマークし、完全な `request/header` スナップショットを通じて権威ある返却ツール順序（`toolOrder` で設定されるか、未設定時は辞書順）を記録する。レンダリングされたプロンプトは派生履歴であり、サーフェスノード 0 の `system/message` と、`in-history` ルートが追記したそれ以降のシステムノードである。したがって、ヘッダーと派生履歴が合わさって、リクエストをセッションログから再構成可能にする。[session.md](session.md#the-request-header-event-requestheader) と[再構成可能性の Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-07-05-reconstructable-requests.md) を参照。

非推奨の `agent/request` hook は既存のリスナーが引き続き利用できる。新しいコードは [Session のモデル選択](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent/README.md#use-this-package)を通じてモデルと推論エフォートを選択する。hook は、ステップが捕捉したモデル選択に基づく凍結された呼び出し設定シードを受け取り、サンプリングなどのコントロールを置き換えてよい。置き換えはプロバイダー、モデル、任意の明示的推論エフォートを保持しなければならない。それらを変更するには、次のステップの前に新しい Session 選択を記録する。waterfall の前に、ループはアダプターのデフォルトとしてマークされた値を取り除き、正確なモデルの準備が選択されたルートの現在の値を実体化する。マークされていない明示設定は提案に残る。waterfall の後、準備はサポートされない明示的エフォート ID をクランプせずに拒否し、有効な設定とアダプターのデフォルトが供給したフィールドをターンシグナルの下でログに記録する。ステップの受付では、この waterfall と準備は組み立てと `step/start` の後、システムプロンプトと受け入れられたユーザーバッチがコミットされる前に実行される。どちらかの間のキャンセルは両方ともコミットしない。準備された機能がプロンプトの再調整を統治し、呼び出しはディスパッチを通じて 1 つのアダプター登録を維持する。`llm/stream` に到達するリクエストは深く凍結されるため変更はスローされ、プロセスローカルのループ同一性を運ぶので、オブザーバーは別々にログに記録された凍結済みの補助呼び出しを会話リクエストと混同しない。

ワイヤ上では、ループが構築したリクエストは派生履歴のみである。レンダリングされたプロンプトは先頭の `system` ロールメッセージ（サーフェスノード 0、`system/message` イベント）として渡り、準備された呼び出しが `systemPromptUpdate: 'in-history'` を宣言している場合、空でない変更済みプロンプトがキャッシュされた履歴の後に後続の `system` ロールメッセージとして続き、モデルはそれを有効なプロンプトとして読む。リクエストの `system` フィールドは未設定である。`GenerateOptions.system` はタイトルプロバイダーのような直接のワンショット呼び出し元にサービスする。空のレンダリングは、以前のリクエストが複数のプロンプトバージョンを保持していたとしても、派生履歴にシステムメッセージを残さない。ログに記録されたリクエストは、ターンの最初のステップでは最新の `user/message` で終わり、後のステップでは前のステップのツール結果で終わる。

FIXME(call-config-shape): 残りのどのフィールドがキャッシュ目的で本当にエポックレベルなのか再検討する（`model` とモデル所有の推論エフォートは明示的であり、サンプリングのスカラーは慎重を期してここに置いている）。

```ts type-equiv
/** Concrete generation settings; omitted controls use the selected route's defaults. */
interface LlmCallControls {
  reasoningEffort?: ReasoningEffortId
  temperature?: number
  maxTokens?: number
  stop?: string[]
}
```

```ts type-equiv
/**
 * Provider, model, reasoning effort, and sampling scalars of one conversation's
 * requests. Every field maps 1:1 onto the same-named `GenerateOptions` field;
 * the loop builds requests from the logged header rather than accepting these
 * per call.
 */
interface LlmCallConfig extends LlmCallControls {
  provider: string
  model: string
}
```

```ts type-equiv
/**
 * Synchronous, pure configuration of one call before defaults and validation.
 * Compose functions in the desired order; later writes replace earlier ones.
 * The returned controls cannot change the captured route. Errors reject
 * preparation before dispatch; only the resolved configuration is recordable.
 * @param controls - detached, deeply frozen proposed controls, without defaults.
 * @param model - detached, deeply frozen metadata from the captured adapter generation.
 * @returns concrete controls; omitted fields receive the route's defaults.
 */
type ConfigureCall = (
  controls: Readonly<LlmCallControls>,
  model: Readonly<LlmResolvedModelInfo>,
) => LlmCallControls
```

```ts type-equiv
/**
 * Effective config fields supplied by exact-model adapter resolution rather
 * than by the caller's request proposal.
 */
interface LlmCallConfigAdapterDefaults {
  reasoningEffort?: true
  maxTokens?: true
}
```

## 公式 DeepSeek リクエスト拡張

`ctx.deepseekLlmApiExtensions` は、`deepseek-official` リクエストの追加的なトップレベルフィールドのためのプロバイダー固有レジストリである。コントリビュータープラグインは `register(field, provider)` を使って 1 つのフィールドを要求する。アダプターはベースボディをシリアライズした後に `prepare(request)` を呼び、返されたフィールドを HTTP の前にマージする。準備された `accept()` トランザクションは 2xx の後に実行されるため、コントリビューターはトランスポートやプロバイダーの拒否を受理とみなすことなく、配信状態をコミットできる。準備がスローするコントリビューターはログに記録されてそのリクエストから省略され、受理の失敗はリクエストを失敗させずにログに記録される。ベースフィールドの衝突は `REQUEST_EXTENSION` を使い、モデルリクエストを失敗させる。マージされたボディのシリアライズに失敗した場合、拡張フィールドなしで送信され、受理はスキップされ、プロバイダープラグインが省略されたフィールド名をログに記録する。

[ワイヤのリファレンス](../deepseek-llm-api-wire-extensions.md)は、正確なリクエストヘッダー、拡張トランザクション、フィールドバージョン、受信側の義務を定義する。出荷されるコンポジションは、[`dsh_session_log`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-log-deepseek/README.md) をロスレスな増分の正規ログサフィックスとして、[`dsh_plugin_packages`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/plugin-package-inventory-deepseek/README.md) を完全なアクティブな Loader ベースのパッケージ集合として登録する。これらのフィールドはモデルメッセージの外にあり、pi-ai アダプター経路には存在しない。

## Pi-ai リクエストヘッダー

`PiAiRequestHeaderContext` は、準備された呼び出しのプロバイダー、モデル、解決済みモデルの `baseURL`、任意の Session ID と補助目的を含む。それは不変であり、メッセージや認証フィールドを含まず、同期の `llm-pi-ai/request-headers` waterfall でリクエスト所有の `Headers` オブジェクトに同伴する。[アダプターのリクエストヘッダーのドキュメント](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/llm-pi-ai/README.md#request-headers)は、コントリビューションの所有権、帰属の優先順位、失敗時の振る舞いを定義する。この拡張ポイントは pi-ai 推論リクエストに固有である。

## サービスとプロバイダーの契約

`LlmAdapter` はプロバイダー契約である。サブクラス化し、`stream()` を実装し、1 つのアダプターインスタンスを `ctx.llm.registerAdapter(providers, adapter)` で登録する。`GenerateOptions.provider` は登録済みアダプターを選択し、`GenerateOptions.model` はそのアダプターに渡され、ライフサイクル開始時に登録されている必要はない。重複するプロバイダールートはアトミックに失敗する。任意の `providerRetryPolicy()` はルートごとに通常のデフォルトで捕捉され、`providerInfo()` と非同期の `listModels()` はデタッチされたセレクターメタデータを `LlmRuntime.listProviders()` / `listModels()` に供給する。このカタログはリクエストのホワイトリストではなく助言的なものであり、アダプターが引き続き権威であり、未掲載のモデル ID を受け入れてよい。1 回の非同期 `resolveModel()` クエリは、正確なモデル同一性に加えて、任意の正確性重視のコンテキスト容量、アダプター設定の `defaultMaxTokens`、および任意のデプロイデフォルトを持つ順序付きのモデル所有推論 ID を返す。欠落したフィールドは、無効なカタログメンバーシップではなく、利用できないメタデータまたはプロバイダー所有の振る舞いを意味する。リゾルバーは任意のキャンセルを受け取り、中断後は速やかに決着しなければならない。`LlmRuntime.resolveModelInfo()` は集約を検証してデタッチする。最終アダプター境界では、`resolveCallConfig()` が `maxTokens` が欠落している場合にのみ出力デフォルトを実体化し、推論を検証して実体化するため、直接呼び出しがどちらの設定済み振る舞いも迂回できない。直接ディスパッチはその解決を待つ前に 1 つの登録を捕捉する。一方、agent ループは `prepareCall()` を使い、モデル解決、永続ヘッダーのログ記録、ディスパッチにわたって同じ登録を維持し、その正確なルックアップからデタッチされたコンテキストメタデータを保持し、アダプターがデフォルト化した設定フィールドを報告する。アダプターのルックアップは `llm/stream` waterfall の終端継続で行われるため、リスナーはルックアップ前に呼び出しをショートサーキットしたり、可変のワンショットリクエストをルーティングしたりできる。AgentLoop は、外側の waterfall がストリームハンドルを返した時点でリクエスト試行を観察する。その限定的な境界は、遅延終端アダプターが構築されたことやプロバイダー I/O を開始したことを証明しない。`block-start` / `block-end` の `index` 相関とアセンブラーが相まって、アダプターは整形されたチャンクを出力するだけでよい。ブロックの再組み立ては各アダプターの責務ではない。[architecture.md](../architecture.md#turn-flow) は、1 ターンの中で `ctx.llm.stream()` と `llm/stream` waterfall がどこに位置するかを示す。

```ts type-equiv
/** One model call whose config and adapter registration were resolved together. */
interface PreparedLlmCall {
  /** Detached, deep-frozen config with any adapter-owned default materialized. */
  readonly config: LlmCallConfig
  /** Immutable retry policy captured with the adapter registration. */
  readonly retryPolicy: ResolvedRetryPolicy
  /** Detached context metadata resolved with the registration-bound call. */
  readonly context?: LlmModelContext
  /** Exact model modalities captured with the adapter dispatch generation. */
  readonly inputModalities?: readonly ModelModality[]
  /** Exact model system prompt update mode captured with the adapter dispatch generation. */
  readonly systemPromptUpdate?: SystemPromptUpdate
  /** Exact model tool update mode captured with the adapter dispatch generation. */
  readonly toolUpdate?: ToolUpdate
  /** Config fields materialized by the captured adapter rather than proposed by the caller. */
  readonly adapterDefaults: LlmCallConfigAdapterDefaults
  /**
   * Dispatch this call once through the registration captured during
   * preparation. The request's call-config fields must match {@link config};
   * reuse or mismatch fails with `INVALID_PREPARED_CALL`.
   * @param options - fully assembled request carrying the prepared config.
   * @returns the chunk stream, including the `llm/stream` waterfall.
   */
  stream(options: GenerateOptions): AsyncIterable<StreamChunk>
}
```

```ts public-api
/**
 * Provider-wire adapter for the harness message and stream vocabulary. Register implementations
 * with `ctx.llm.registerAdapter(providers, adapter)`. Every provider HTTP request must include
 * `attributionHeaders()`; prove the headers are added in the wire request or library header hook. The direct-fetch
 * DeepSeek and library-backed pi-ai adapters meet this contract through different internals.
 */
declare abstract class LlmAdapter {
  /**
   * Describe one provider route owned by this adapter.
   * @param provider - a route passed to `registerAdapter()` for this instance.
   * @returns detached display metadata whose id must equal `provider`.
   */
  providerInfo(provider: string): LlmProviderInfo;
  /**
   * Return the provider-owned retry policy captured with this route.
   * @param _provider - a route passed to `registerAdapter()` for this instance.
   * @returns a resolved policy, or `undefined` to use the normal defaults.
   */
  providerRetryPolicy(_provider: string): ResolvedRetryPolicy | undefined;
  /**
   * Resolve provider-side request-image pricing for one exact model route.
   * The default declares none, so consumers fall back to their own neutral
   * estimate. Implementations must answer synchronously without I/O; the
   * token meter resolves this per measurement.
   * @param _provider - a route passed to `registerAdapter()` for this instance.
   * @param _model - exact model id passed to {@link GenerateOptions.model}.
   * @returns route-owned image pricing, or `undefined` when the route declares none.
   */
  imageRequestPricing(_provider: string, _model: string): LlmImageRequestPricing | undefined;
  /**
   * Resolve the image dimensions and byte target this adapter sends to the model.
   * The default preserves normalized images; adapters that resize must override it.
   * @param _provider - registered provider route.
   * @param _model - exact model id.
   * @param image - normalized image whose request projection is being resolved.
   * @returns the same target used by request serialization, before provider-internal processing.
   */
  imageRequestTarget(_provider: string, _model: string, image: ImageAttachmentRef): ImageRequestTarget;
  /**
   * List models this adapter can currently advertise for one owned provider.
   * Core routing accepts unlisted model ids; catalog-driven entry points such
   * as the GUI may require membership. Adapters used there must advertise
   * their available models; the base empty catalog offers no GUI selection.
   * @param _provider - one provider route owned by this adapter.
   * @returns discoverable models in adapter-preferred order.
   */
  listModels(_provider: string): Promise<readonly LlmModelInfo[]>;
  /**
   * Resolve all metadata available for one exact model. This query is
   * independent of the advisory catalog and does not validate request routing.
   * @param provider - one provider route owned by this adapter.
   * @param model - exact model id passed to {@link GenerateOptions.model}.
   * @param _signal - cancellation for this exact-model lookup; asynchronous
   *   implementations must settle promptly after it aborts.
   * @returns provider/model identity plus any context, call-default, and reasoning metadata.
   */
  resolveModel(
    provider: string,
    model: string,
    _signal?: AbortSignal,
  ): Promise<LlmResolvedModelInfo>;
  /**
   * Bind exact model metadata and the eventual request dispatch to one adapter generation.
   * Dynamic adapters override this so settings changes between preparation and
   * dispatch cannot combine one generation's capabilities with another's endpoint.
   * @param provider - registered provider route.
   * @param model - exact model id.
   * @param signal - cancellation for model resolution.
   * @returns model metadata and a one-generation stream entry point.
   */
  async prepareCall(provider: string, model: string, signal?: AbortSignal): Promise<PreparedAdapterCall>;
  /**
   * Stream one model call as raw chunks. The only required method.
   * @param options - the fully-assembled request; implementations must honor `options.signal`.
   * @returns the chunk stream, obeying the adapter contract documented on `StreamChunk`.
   */
  abstract stream(options: GenerateOptions): AsyncIterable<StreamChunk>;
}
```

`ContentBlockType`（`index` で相関するブロックが運ぶキー集合）は、上記の [`ContentBlockMap`](#content-blocks-and-messages) から派生する。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxdeepseekllmapiextensions--deepseekllmapiextensionregistry"></a>

### `ctx.deepseekLlmApiExtensions` — `DeepSeekLlmApiExtensionRegistry`

公式 DeepSeek リクエスト向けの、独立して所有されるトップレベルフィールドのレジストリ。

```ts cordis-catalog
/**
 * Register the sole provider of one top-level request field. Registration is effect-scoped.
 * @param field - declaration-merged field owned by the provider.
 * @param provider - request-time field preparation and optional acceptance behavior.
 * @returns disposer that releases the field.
 */
register<K extends keyof DeepSeekLlmApiExtensionMap>( field: K, provider: DeepSeekLlmApiExtensionProvider<DeepSeekLlmApiExtensionMap[K]>, ): () => Promise<void>

/**
 * Prepare every currently registered field from one immutable base request.
 * A provider whose preparation throws, or whose value cannot be cloned, is omitted
 * from this request; the first such failure per field is logged. Only cancellation rejects. Field values are cloned
 * and frozen; providers retain no mutable alias to the outgoing request.
 * @param request - exact serialized request facts before extension fields.
 * @returns detached fields and their idempotent joint acceptance transaction.
 */
async prepare(request: DeepSeekLlmApiExtensionRequest): Promise<PreparedDeepSeekLlmApiExtensions>
```

ソース: [`packages/llm/deepseek-llm-api-extensions/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/deepseek-llm-api-extensions/src/index.ts)

<a id="ctxllm--llmruntime"></a>

### `ctx.llm` — `LlmRuntime`

抽象的な `llm` サービス: アダプターレジストリとストリーミングのモデル呼び出し API であり、`llm/stream` waterfall で傍受できる。

```ts cordis-catalog
/**
 * Register an adapter for the given provider routes. Throws `LlmError` with code
 * `DUPLICATE_ADAPTER` if any provider already has an adapter (all-or-nothing).
 * Disposed with the fiber.
 * @param providers - every provider route this adapter should serve.
 * @param adapter - the adapter that streams calls for those providers.
 * @returns the disposer, carrying {@link AdapterRegistrationHandle.replace}.
 */
registerAdapter(providers: string[], adapter: LlmAdapter): AdapterRegistrationHandle

/**
 * Describe provider routes with a registered adapter.
 * @returns detached provider metadata in registration order.
 */
@Remote listProviders(): LlmProviderInfo[]

/**
 * Declare provider routes an adapter plugin can activate through
 * configuration. Registration is all-or-nothing: an empty list, invalid
 * entry, or a provider already declared by any registration throws
 * `LlmError` without registering the rest. Disposed with the fiber.
 * @param entries - every configurable provider this plugin owns.
 * @returns a handle that withdraws all of them, and can atomically replace them.
 */
registerConfigurableProviders(entries: readonly LlmConfigurableProvider[]): DirectoryRegistrationHandle

/**
 * List every declared configurable provider, registered or dormant.
 * @returns detached directory entries in declaration order.
 */
@Remote listConfigurableProviders(): LlmConfigurableProvider[]

/**
 * Offer to interrogate provider endpoints on behalf of the settings
 * namespace this plugin owns. The namespace is the key because that is what
 * a configuration surface already holds from the configurable-provider
 * directory, and because a provider being *added* has no route to name yet.
 * Disposed with the fiber.
 * @param settingsNs - the namespace whose profiles this discovery serves.
 * @param discover - interrogates one endpoint and must honor the supplied signal.
 * @returns the disposer that withdraws the offer.
 */
registerModelDiscovery( settingsNs: string, discover: ( request: LlmModelDiscoveryRequest, signal?: AbortSignal, ) => Promise<readonly LlmDiscoveredModel[]>, ): () => void

/**
 * Interrogate one provider endpoint for the models it advertises. The
 * request describes a draft, not a stored route, so nothing here reads or
 * writes settings or credentials — the caller owns both, and the reply is
 * candidate metadata a surface may offer for adoption.
 * @param settingsNs - namespace whose registered discovery serves this draft.
 * @param request - the endpoint, protocol, and one-shot credential to use.
 * @param signal - caller cancellation.
 * @returns the advertised models, deduplicated in endpoint order.
 */
async discoverModels( settingsNs: string, request: LlmModelDiscoveryRequest, signal?: AbortSignal, ): Promise<LlmDiscoveredModel[]>

/**
 * Remote adapter for one draft provider interrogation.
 * @param settingsNs - namespace whose registered discovery serves this draft.
 * @param request - endpoint, protocol, and one-shot credential to use.
 * @param signal - caller cancellation supplied by the Remote carrier.
 * @returns advertised models in endpoint order.
 * @throws RemoteError with `llm/model-discovery-rejected` when discovery refuses or fails.
 */
@Remote('discoverModels') async remoteDiscoverModels( settingsNs: string, request: LlmModelDiscoveryRequest, signal: AbortSignal, ): Promise<LlmDiscoveredModel[]>

/**
 * Resolve the retry policy captured when one provider route was registered.
 * @param provider - registered provider route to inspect.
 * @returns the provider-owned policy, with normal defaults already resolved.
 */
providerRetryPolicy(provider: string): ResolvedRetryPolicy

/**
 * Resolve provider-side request-image pricing for one exact route, or
 * `undefined` when the provider is unregistered or declares none. Unknown
 * providers degrade to `undefined` rather than throwing because callers
 * price durable history whose route may no longer be mounted.
 * @param provider - provider route named by a request header.
 * @param model - exact model id named by the same header.
 * @returns the owning adapter's image pricing for the route, when declared.
 */
imageRequestPricing(provider: string, model: string): LlmImageRequestPricing | undefined

/**
 * Resolve the image projection for an exact registered model route.
 * @param provider - registered provider route.
 * @param model - exact model id.
 * @param image - normalized image reference.
 * @returns dimensions and byte target used by the adapter's request serializer.
 */
imageRequestTarget(provider: string, model: string, image: ImageAttachmentRef): ImageRequestTarget

/**
 * Resolve the exact text one durable file occurrence contributes to every
 * provider request in the current execution environment.
 * @param ref - durable verbatim file reference from model history.
 * @returns the same deterministic handle text used at adapter dispatch.
 */
fileRequestText(ref: FileAttachmentRef): string

/**
 * Discover models advertised by one registered provider. Catalog membership
 * does not constrain core routing. Catalog-driven entry points may restrict
 * selection and submission to the advertised models.
 * @param provider - registered provider route to inspect.
 * @returns detached model metadata in adapter-preferred order.
 */
async listModels(provider: string): Promise<LlmModelInfo[]>

/**
 * Resolve and validate all metadata from the adapter that owns one exact
 * route. The result is detached from adapter-owned objects; catalog
 * membership remains advisory and does not control request routing.
 * @param provider - registered provider route to inspect.
 * @param model - exact model id passed to the adapter.
 * @param signal - optional cancellation for adapter-owned asynchronous lookup.
 * @returns exact model identity plus available context and reasoning metadata.
 */
async resolveModelInfo( provider: string, model: string, signal?: AbortSignal, ): Promise<LlmResolvedModelInfo>

/**
 * Validate a conversation call config against its exact model capability and
 * materialize adapter-configured defaults. Unsupported explicit efforts
 * reject before provider I/O; no clamping or aliasing is performed. This
 * standalone query does not bind a later dispatch; use {@link prepareCall}
 * when logging and streaming must share one adapter registration.
 * @param config - provider/model route and optional request controls.
 * @param signal - optional cancellation for adapter-owned capability lookup.
 * @returns a detached config only when a default must be materialized.
 */
async resolveCallConfig(config: LlmCallConfig, signal?: AbortSignal): Promise<LlmCallConfig>

/**
 * Resolve one call under its current adapter registration. The returned
 * one-shot handle keeps that registration across header logging and dispatch,
 * so HMR cannot combine one adapter's capability result with another adapter.
 * An optional synchronous callback selects concrete controls using captured
 * model metadata. Defaults and validation apply to its result. Callback
 * failures and cancellation reject preparation before dispatch.
 * @param config - provider/model route and optional concrete request controls.
 * @param signal - optional cancellation for adapter-owned capability lookup.
 * @param configure - pure control selection, called once before defaults and validation.
 * @returns a prepared config and its registration-bound stream entry point.
 */
async prepareCall( config: LlmCallConfig, signal?: AbortSignal, configure?: ConfigureCall, ): Promise<PreparedLlmCall>

/**
 * Stream one model call as raw chunks (token-level deltas). Replay state is
 * retained only when the same adapter instance owns its historical provider
 * and the target provider. Final adapter selection remains fixed through
 * asynchronous exact-model resolution and dispatch. Adapter selection,
 * dispatch, and iteration failures become terminal `error` or `aborted`
 * finish chunks; middleware, nested-call, cleanup, and consumer failures
 * remain thrown.
 * @param options - the full request; `options.provider` selects the adapter.
 * @returns the chunk stream, possibly wrapped by `llm/stream` listeners.
 */
stream(options: GenerateOptions): AsyncIterable<StreamChunk>
```

型: [FileAttachmentRef](attachment.md) · [ImageAttachmentRef](attachment.md) · [ImageRequestTarget](attachment.md)

ソース: [`packages/llm/llm/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/llm/src/index.ts)

<a id="llm-events"></a>

### `llm/*` イベント

<a id="llmadapters-updated--emit"></a>

#### `llm/adapters-updated` — emit

プロバイダーのトポロジーが変わった。アダプターがルートを登録または登録解除したか、設定可能プロバイダーディレクトリがエントリーを追加または失った。このペイロードのないレジストリ通知は各コミットポイント（登録の破棄を含む）で発火し、コンシューマーは新しい状態のために `listProviders()`、`listModels()`、または `listConfigurableProviders()` を再読み取りする。オブザーバーの失敗は封じ込められ、レジストリの変更を拒否できない。

```ts cordis-catalog
/**
 * The provider topology changed: an adapter registered or unregistered
 * routes, or the configurable-provider directory gained or lost entries.
 * This payload-free registry notification fires at each commit point
 * (including registration disposal); consumers re-read `listProviders()`,
 * `listModels()`, or `listConfigurableProviders()` for the new state.
 * Observer failures are contained and cannot veto the registry mutation.
 * @mode emit
 */
'llm/adapters-updated'(): void
```

ソース: [`packages/llm/llm/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/llm/src/types.ts)

<a id="llmmodels-updated--emit"></a>

#### `llm/models-updated` — emit

プロバイダーの公開モデルまたは表示メタデータが変わった。コンシューマーは `listModels()` を再読み込みする。

```ts cordis-catalog
/**
 * A provider's advertised models or display metadata changed; consumers reload `listModels()`.
 * @mode emit
 * @param provider - route whose model catalog changed.
 */
'llm/models-updated'(provider: string): void
```

ソース: [`packages/llm/llm/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/llm/src/types.ts)

<a id="llmstream--waterfall"></a>

#### `llm/stream` — waterfall

すべてのストリーミングモデル呼び出しを囲む waterfall（リトライ、リプレイ、ルーティング）。LlmRuntime に束縛される。`next()` を呼んで解決されたアダプターのストリームに到達するか、自前のチャンクを yield してショートサーキットする。

```ts cordis-catalog
/**
 * Waterfall around every streaming model call (retry, replay, routing).
 * Bound to the {@link LlmRuntime}; call `next()` to reach the resolved
 * adapter's stream, or yield your own chunks to short-circuit.
 * @param options - the full request. A LOOP-built request carries the
 *   process-local {@link markAgentLoopRequest} identity and arrives deep-frozen
 *   (mutation throws): its content is a pure function of the session log (the
 *   reconstructability Agent Note), so listeners read it, never rewrite it.
 *   Hand-built calls do not carry that marker; callers own their request
 *   inputs and must keep them unchanged until the stream settles.
 * @mode waterfall
 */
'llm/stream'(this: LlmRuntime, options: GenerateOptions, next: () => AsyncIterable<StreamChunk>): AsyncIterable<StreamChunk>
```

ソース: [`packages/llm/llm/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/llm/src/index.ts)

<a id="llm-pi-ai-events"></a>

### `llm-pi-ai/*` イベント

<a id="llm-pi-airequest-headers--waterfall"></a>

#### `llm-pi-ai/request-headers` — waterfall

モデル検出を除く 1 回の pi-ai 推論呼び出しのヘッダーを準備する。リスナーは同期的に実行され、`next()` を呼ばなければならない。リクエスト所有のヘッダーのみを編集でき、それらを保持してはならない。スローは推論 I/O の前に `REQUEST_HEADERS` で呼び出しを失敗させる。Harness の帰属情報がその後で優先される。TypeError 以外のメッセージはユーザーに表示され、シークレットを含んではならない。TypeError の詳細と Error 以外の値は安全なガイダンスに置き換えられる。

```ts cordis-catalog
/**
 * Prepare headers for one pi-ai inference call, excluding model discovery.
 * Listeners run synchronously and must call next(); they may edit only the
 * request-owned headers and must not retain them. Throws fail the call with
 * REQUEST_HEADERS before inference I/O. Harness attribution wins afterward.
 * Non-TypeError messages are user-visible and must exclude secrets. TypeError
 * details and non-Error values are replaced with safe guidance.
 * @mode waterfall
 * @param request - immutable routing metadata from the exact prepared call.
 * @param headers - fresh profile headers, shared only within this dispatch.
 * @param next - continue header preparation through the remaining listeners.
 * @returns headers passed to pi-ai after Harness attribution is applied.
 */
'llm-pi-ai/request-headers'(request: PiAiRequestHeaderContext, headers: Headers, next: () => Headers): Headers
```

ソース: [`packages/llm/llm-pi-ai/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/llm-pi-ai/src/index.ts)
<!-- END GENERATED cordis-surface -->
