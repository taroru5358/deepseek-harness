# メッセージフィードバック

日本語 | [English](../../../docs/subsystems/feedback.md) | [中文](../../../docs/subsystems/feedback.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/feedback.md](../../../docs/subsystems/feedback.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

[`@deepseek-ai/dsh-message-feedback`](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/feedback/message-feedback) は、個々のアシスタントメッセージに対する編集可能なフィードバックを所有する。正規の Session ログは `feedback/message-put` と `feedback/message-delete` を保存する。不変の Session レベルの所見は `feedback/record` のままで、[`@deepseek-ai/dsh-command-feedback`](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/feedback/command-feedback) が、両方の種類のフィードバックが属する `FeedbackCategory` 分類法とともに所有する。3 つともログ専用のイベントであり、モデルのコンテキストには決して入らない。

ソース: [`packages/feedback/message-feedback/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/feedback/message-feedback/src/types.ts)

## 公開型

```ts type-equiv
/** Opaque compare-and-set token for one exact feedback item revision. */
type MessageFeedbackVersion = Branded<'MessageFeedbackVersion'>
```

```ts type-equiv
/** The human's overall judgment of one assistant message. */
type MessageFeedbackRating = 'positive' | 'negative'
```

```ts type-equiv
/** One current feedback value and its opaque mutation token. */
interface MessageFeedbackItem {
  /** Stable identity of the assistant message inside the owning Session. */
  readonly messageId: MessageId
  /** Overall positive or negative judgment. */
  readonly rating: MessageFeedbackRating
  /** Optional explanation, preserved verbatim after validation. */
  readonly note?: string
  /** Category the human filed the judgment under. */
  readonly category?: FeedbackCategory
  /** Equality-only token replaced by every material create or update. */
  readonly version: MessageFeedbackVersion
  /** Host-assigned creation time in Unix epoch milliseconds. */
  readonly createdAt: number
  /** Host-assigned time of the most recent material update. */
  readonly updatedAt: number
}
```

```ts type-equiv
/** A material creation or edit, retaining its complete current value. */
interface MessageFeedbackPut {
  /** Owning Session; inherited feedback in a fork belongs to its parent. */
  readonly sessionId: SessionId
  /** Value after this mutation, including the original creation time. */
  readonly item: MessageFeedbackItem
}
```

```ts type-equiv
/** A material deletion of one current feedback item. */
interface MessageFeedbackDelete {
  /** Session that owns the deleted feedback. */
  readonly sessionId: SessionId
  /** Message whose feedback was removed. */
  readonly messageId: MessageId
}
```

```ts type-equiv
/** Read all message feedback belonging to one persisted Session lifecycle. */
interface MessageFeedbackListRequest {
  /** Session whose feedback events should be read. */
  readonly sessionId: SessionId
}
```

```ts type-equiv
/** Current feedback values for one Session, in first-creation order. */
interface MessageFeedbackListValue {
  /** Fresh immutable item snapshots. */
  readonly items: readonly MessageFeedbackItem[]
}
```

```ts type-equiv
/** Create or replace feedback for one assistant message. */
interface MessageFeedbackPutRequest {
  /** Persisted Session that owns the target message. */
  readonly sessionId: SessionId
  /** Target assistant-message identity. */
  readonly messageId: MessageId
  /** Desired overall judgment. */
  readonly rating: MessageFeedbackRating
  /** Optional non-blank explanation. */
  readonly note?: string
  /** Optional category; absent keeps the item uncategorized. */
  readonly category?: FeedbackCategory
  /** Observed item version, or `null` to require that no item exists. */
  readonly ifVersion: MessageFeedbackVersion | null
}
```

```ts type-equiv
/** Delete feedback for one message after observing its current version. */
interface MessageFeedbackDeleteRequest {
  /** Session that owns the feedback. */
  readonly sessionId: SessionId
  /** Message whose feedback should be absent after this operation. */
  readonly messageId: MessageId
  /** Observed item version; ignored when the item is already absent. */
  readonly ifVersion: MessageFeedbackVersion
}
```

```ts type-equiv
/** Idempotent deletion acknowledgement. */
interface MessageFeedbackDeleteValue {
  /** Stable postcondition shared by the first deletion and every retry. */
  readonly absent: true
}
```

```ts type-equiv
/** No persisted Session header exists for the requested id. */
interface MessageFeedbackSessionNotFound {
  readonly code: 'session-not-found'
  readonly sessionId: SessionId
}
```

```ts type-equiv
/** The id does not name a derived, append-origin assistant message. */
interface MessageFeedbackTargetNotFound {
  readonly code: 'target-not-found'
  readonly sessionId: SessionId
  readonly messageId: MessageId
}
```

```ts type-equiv
/** A material mutation did not match the addressed item's current version. */
interface MessageFeedbackVersionConflict {
  readonly code: 'version-conflict'
  /** Authoritative current item, or `null` when it does not exist. */
  readonly current: MessageFeedbackItem | null
}
```

```ts type-equiv
/** A supplied note contains no non-whitespace character. */
interface MessageFeedbackNoteBlank {
  readonly code: 'note-blank'
}
```

```ts type-equiv
/** A supplied note exceeds the configured UTF-8 byte limit. */
interface MessageFeedbackNoteTooLarge {
  readonly code: 'note-too-large'
  readonly maxBytes: number
  readonly actualBytes: number
}
```

```ts type-equiv
/** Failures shared by the public message-feedback operations. */
type MessageFeedbackFailure =
  | MessageFeedbackSessionNotFound
  | MessageFeedbackTargetNotFound
  | MessageFeedbackVersionConflict
  | MessageFeedbackNoteBlank
  | MessageFeedbackNoteTooLarge
```

```ts type-equiv
/** Successful public operation result. */
interface MessageFeedbackSuccess<T> {
  readonly ok: true
  readonly value: T
}
```

```ts type-equiv
/** Rejected public operation result with a stable business failure. */
interface MessageFeedbackRejected<E extends MessageFeedbackFailure> {
  readonly ok: false
  readonly error: E
}
```

```ts type-equiv
/** Result returned by the message-feedback `list` operation. */
type MessageFeedbackListResult =
  | MessageFeedbackSuccess<MessageFeedbackListValue>
  | MessageFeedbackRejected<MessageFeedbackSessionNotFound>
```

```ts type-equiv
/** Result returned by the message-feedback `put` operation. */
type MessageFeedbackPutResult =
  | MessageFeedbackSuccess<MessageFeedbackItem>
  | MessageFeedbackRejected<
    | MessageFeedbackSessionNotFound
    | MessageFeedbackTargetNotFound
    | MessageFeedbackVersionConflict
    | MessageFeedbackNoteBlank
    | MessageFeedbackNoteTooLarge
  >
```

```ts type-equiv
/** Result returned by the message-feedback `delete` operation. */
type MessageFeedbackDeleteResult =
  | MessageFeedbackSuccess<MessageFeedbackDeleteValue>
  | MessageFeedbackRejected<MessageFeedbackSessionNotFound | MessageFeedbackVersionConflict>
```

## Session フィードバック型

ソース: [`packages/feedback/command-feedback/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/feedback/command-feedback/src/types.ts)

```ts type-equiv
/** One of the fixed feedback categories; the ids are durable log vocabulary. */
type FeedbackCategory =
  | 'task-result'
  | 'instruction-following'
  | 'product-interaction'
  | 'service-stability'
  | 'resource-cost'
  | 'security-privacy-permission'
  | 'other'
```

```ts type-equiv
/**
 * One recorded human remark about a Session. Both members are optional: a
 * submission with neither still records that the human asked for the
 * Session to be reviewed, which is what authorizes log delivery.
 */
interface FeedbackRecord {
  /** Free-text remark with surrounding whitespace removed; never empty when present. */
  readonly text?: string
  /** Category the human filed the remark under. */
  readonly category?: FeedbackCategory
}
```

```ts type-equiv
/** Record one Session-level remark through the Host Remote. */
interface SessionFeedbackRecordRequest {
  /** Live Session the remark describes. */
  readonly sessionId: SessionId
  /** Free-text remark; blank text is recorded as absent. */
  readonly text?: string
  /** Category the human filed the remark under. */
  readonly category?: FeedbackCategory
}
```

```ts type-equiv
/** Stable postcondition of a recorded remark. */
interface SessionFeedbackRecordValue {
  /** The remark is appended to the Session log; flushing follows the Session's own schedule. */
  readonly recorded: true
}
```

```ts type-equiv
/** No live Session carries the requested id. */
interface SessionFeedbackSessionNotFound {
  readonly code: 'session-not-found'
  readonly sessionId: SessionId
}
```

```ts type-equiv
/** Result returned by the `sessionFeedback.record` operation. */
type SessionFeedbackRecordResult =
  | { readonly ok: true; readonly value: SessionFeedbackRecordValue }
  | { readonly ok: false; readonly error: SessionFeedbackSessionNotFound }
```

## データと並行性

現在の項目は、ペイロードの `sessionId` が所有 Session と一致する正規のフィードバックイベントから畳み込まれる。各項目は肯定的または否定的な評価、任意の note、任意のカテゴリ、Host が割り当てた `createdAt`/`updatedAt` タイムスタンプ、自身の不透明なバージョンを持つ。バージョンは等価性のためだけに、また対象のメッセージに対してのみ比較される。呼び出し元はそれらを順序付けたり合成したりしない。

`put` は厳密な楽観的並行性を使う。既存項目へのすべてのリクエストは、no-op（保存済みの rating、note、カテゴリを繰り返す put）を含めて現在の `ifVersion` と一致しなければならない。競合は権威ある現在の項目（または `null`）を返すため、呼び出し元は応答の喪失や同時編集を別の読み取りなしで調整できる。すでに存在しない項目の削除は成功する。Session ごとのキューが読み取りと変更を直列化し、cold の変更は読み取り、比較、append、flush にわたって永続化の write ハンドルを保持する。一致する no-op はイベントを追記しない。

## 対象とライフサイクルの権威

ライブ所有者のインメモリログが対象 Session の観測を直接供給する。cold 読み取りは `SessionPersistence.open(id, 'read')` ハンドルを使い、変更は write ハンドルを使う。どちらの経路も Session や Agent を構築しない。`stat(id)` のプリフライトが明確な不在を分類し、`stat` が確認した Session の読み取り失敗はインフラ障害として伝播する。`put` は要求された `MessageId` を持つ、空でない append 由来の `assistant/message` だけを受け入れる。replacement 由来、usage のみの空、アシスタント以外のレコードはフィードバックの対象ではない。

fork の seed は親のフィードバックイベントを含むことがあるが、そのペイロードは親の `sessionId` を保持するため、子の現在のフィードバックにはならない。項目を削除すると tombstone を追記する。以前の rating と note はログに残る。

## 永続化と Remote 契約

成功したメッセージフィードバックの変更は正規の永続化を待機する。ライブ操作は所有 Session を通じて追記し、参加する `ctx.sessions.flush` リスナーを必要とする。cold 操作は自身の write ハンドルを通じて追記して flush する。永続化の失敗は成功を報告せずに伝播する。`maxNoteBytes` は必須で、note のテキストを UTF-8 バイト数で制限する。Web Host の構成は `8192` を設定する。パッケージは `TypertRemoteService` と `@Remote` を通じて Host の `messageFeedback.list`、`messageFeedback.put`、`messageFeedback.delete` の unary Remote 契約を公開する。`command-feedback` はライブ Session の Session レベル所見のために同じ方法で `sessionFeedback.record` を公開する。以下の生成された Cordis API がメソッドレベルの権威である。

プラグインの破棄は操作の受け入れを閉じ、受け入れた Session ごとのキューの作業をドレインする。

デフォルトでは、[`session-log-deepseek`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-log-deepseek/README.md) が、後続の対象となる DeepSeek リクエストで通常の `dsh_session_log` サフィックスの一部としてフィードバックを運ぶ。構成は `enabled: false` でそれを無効にする。フィードバックの記録は LLM リクエストや別の `dsh_feedback` アップロードをトリガーしない。DeepSeek 以外のルートでは、[OTel バックエンド](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-telemetry-otel/README.md)が記録されたフィードバックを通じて正規のプレフィックスを解放できる。コマンドの確認応答は記録を確認し、Session と匿名ユーザーを特定する。テレメトリポリシーも配信も報告しない。

## Web サーフェス

[`@deepseek-ai/dsh-client-ui-message-feedback`](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/client/ui-message-feedback) はブラウザの consumer である。`@deepseek-ai/dsh-api-remotes` が生成された `messageFeedback` と `sessionFeedback` のコントリビューションをマウントするため、プラグインは `ctx.remote.messageFeedback` と `ctx.remote.sessionFeedback` を呼び、トランスポートには決して触れない。

コントロールは `conversation.chat.assistant-actions` リストスロットの `feedback` エントリ（順序 10）であり、`ui-conversation` が宣言し、確定したアシスタントメッセージの IconActions 行の中でレンダリングする。`AssistantMessageNode` は `assistant/message` イベントからの任意の `messageId` を運ぶ。このフィールドは中断で凍結された部分メッセージには存在せず、レンダリング箇所はそれが存在しないときスロットをスキップする。ストリップは turn ごとに 1 回、締めくくりのアシスタントメッセージにレンダリングされる。Host は append 由来のすべての step メッセージを対象として受け入れるが、複数 step の turn の以前の step は評価可能な本文ではなく tool 行をレンダリングするため、UI は Host 契約が許すより狭い集合を公開する。

Session ごとに 1 つの `MessageFeedbackController` がその Session のすべてのメッセージコントロールを支える。1 回の `list` 読み取りがトランスクリプト全体をシードし、マウント時ではなく最初のホバーまたはフォーカスまで遅延される。各変更はコントローラが最後に観測したバージョンを `ifVersion` として送る。`version-conflict` の応答は権威ある項目を運ぶため、コントローラは再取得せずに応答から調整する。変更は Session ごとに直列化されるため、キューに入った操作はコミット済みのバージョンと比較される。注入された `retract` 操作はそのキュー内でコミット済みの rating を再確認し、同時変更の後では no-op になるため、古い UI が単なる rating を記録してダイアログを迂回することはできない。`connection/reset` はすでに読み取った Session だけを更新する。

未記録の rating のいずれかが、Session のフィードバックダイアログ — `conversation.input.overlay` の `feedback-dialog` エントリ — を開く。7 つのカテゴリチップと詳細ボックスを備えた共有 Modal カードである。送信すると、選択された判断が選んだカテゴリとトリムされた説明を伴って、またはどちらも伴わずに put される。成功するとダイアログが閉じ、確認トーストが表示される。失敗するとダイアログと下書きが開いたままになり、警告トーストが表示される。同じダイアログは Session に対して引数なしの `/feedback` からも開く — `ui-commands` が `action` としてルーティングする装飾である — そして `sessionFeedback.record` を通じて記録する。`/feedback <text>` は Host コマンド経路を保つ。記録済みの rating をクリックすると、ダイアログを開かずにそれを撤回する。

## 境界と制限

- 操作キューはプロセスローカルである。cold の writer 排他は選択された永続化 provider に依存する。
- 削除は現在の項目を削除するのであり、追記専用ログの以前の note テキストやすでに配信されたサフィックスを削除するのではない。
- live のデタッチ後、永続化カタログがヘッダーを実体化する前の狭い間隔でのリクエストは `session-not-found` を受け取ることがある。呼び出し元は退役の実体化後に再試行する。
- cold リクエストは完全なログを読む。サービスには項目数や集約バイト数の上限がない。`maxNoteBytes` は各 note だけを制限する。
- Host 契約は認証済みのアクターや監査 ID を記録しないため、信頼された呼び出し元境界を前提とする。
- Web コントロールはチャットビューにのみ表示される。trajectory ビューと waterfall ビューは、アシスタントノードが同じ `messageId` を運んでいてもフィードバックエントリをレンダリングしない。
- Web コントローラはフィードバックログイベントを消費しないため、2 つ目のタブの rating は即座ではなく、再接続時または次の競合応答時に可視になる。
- ダイアログは `maxNoteBytes` を事前チェックしない。メッセージに対する過大な説明は入力中ではなく送信時に `note-too-large` で失敗する。Session の所見にはサイズ上限がない。`/feedback` コマンドにもなかったからである。
- `sessionFeedback.record` はライブ Session にのみ提供し、それ以外には `session-not-found` を返す。ダイアログは開いている間に Session が退役するとその失敗を報告する。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxmessagefeedback--messagefeedbackservice"></a>

### `ctx.messageFeedback` — `MessageFeedbackService`

Session ログサービス。cold 操作は Session や Agent を決して構築しない。

```ts cordis-catalog
/**
 * Read current feedback from the canonical log.
 * @param request - Session to inspect.
 * @returns immutable items or a definite persistence miss.
 */
@Remote('list') list(request: MessageFeedbackListRequest): Promise<MessageFeedbackListResult>

/**
 * Create or replace feedback after checking its current version.
 * Matching no-ops retain the version and append no event.
 * @param request - Target, desired value, and observed item version.
 * @returns the durable item or an explicit business failure.
 */
@Remote('put') put(request: MessageFeedbackPutRequest): Promise<MessageFeedbackPutResult>

/**
 * Delete one item after checking its version; absence succeeds without an event.
 * @param request - Session, message, and observed item version.
 * @returns the stable absent postcondition or an explicit failure.
 */
@Remote('delete') delete(request: MessageFeedbackDeleteRequest): Promise<MessageFeedbackDeleteResult>
```

ソース: [`packages/feedback/message-feedback/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/feedback/message-feedback/src/index.ts)

<a id="ctxsessionfeedback--sessionfeedbackservice"></a>

### `ctx.sessionFeedback` — `SessionFeedbackService`

プロダクトのサーフェスが Session レベルの所見を記録する Host Remote。

```ts cordis-catalog
/**
 * Record one remark on a live Session.
 * @param request - target Session plus the optional text and category.
 * @returns the recorded postcondition, or `session-not-found` when no live
 * Session carries the id.
 */
@Remote('record') record(request: SessionFeedbackRecordRequest): Promise<SessionFeedbackRecordResult>
```

ソース: [`packages/feedback/command-feedback/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/feedback/command-feedback/src/index.ts)

<a id="feedback-events"></a>

### `feedback/*` イベント

<a id="feedbackcommitted--parallel"></a>

#### `feedback/committed` — parallel

ライブ Session を公開せずに、永続的な cold フィードバック変更を観測する。オブザーバーは write 所有権が解放される前に実行され、この Session の別の message-feedback 操作を待機してはならない。ペイロードは読み取り専用で借用される。所有権を移す前にディープクローンすること（たとえば Session.fromRestore へ）。

```ts cordis-catalog
/**
 * Observe a durable cold feedback mutation without publishing a live Session.
 * Observers run before write ownership is released and must not await
 * another message-feedback operation for this Session. The payload is borrowed
 * read-only; deep-clone it before transferring ownership (for example, to Session.fromRestore).
 * @param inspection - committed canonical prefix, including the feedback as its last event.
 * @mode parallel
 */
'feedback/committed'(inspection: SessionInspection): void
```

型: [SessionInspection](persistence.md)

ソース: [`packages/feedback/message-feedback/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/feedback/message-feedback/src/index.ts)
<!-- END GENERATED cordis-surface -->
