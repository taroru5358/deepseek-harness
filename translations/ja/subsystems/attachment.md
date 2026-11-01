# 永続的な添付ファイル

日本語 | [English](../../../docs/subsystems/attachment.md) | [中文](../../../docs/subsystems/attachment.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/attachment.md](../../../docs/subsystems/attachment.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

添付ファイル seam は、バイナリ画像と汎用ファイルの所有権をセッションログから分離する。プロデューサーはバイトを [`ctx.attachments`](#ctxattachments--attachmentstore-abstract-seam) に渡し、サービスはオブジェクトが永続化された後にのみ、不変のコンテンツアドレス方式の参照を公開する。Session イベントとモデル可視の添付ブロックはその参照とメタデータを含み、ブラウザのオブジェクト URL、host の一時パス、provider URL、base64 ペイロードは決して含まない。独立した [`ctx.fileUploads`](#ctxfileuploads--fileuploads) サービスが、ブラウザのファイル転送とステージング済みレシートを受け取る Agent に束縛する。

未送信のブラウザ下書きはメモリに留まってよく、ネイティブクライアントは OS の一時ストレージにステージングしてよい。ブラウザの汎用ファイルは、ステージングされた prompt レシートを受け取る前に永続化される。host がユーザーメッセージを受け入れると、その画像はユーザーイベントが追記される前に `<DSH_HOME>/attachments/v1` の下へ移動する。構造化されたモデル画像出力も同じ「イベント前に永続化」の規則に従う。

ソース: [`packages/attachment/attachment/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/attachment/attachment/src/types.ts)

## アイデンティティと検証済みメタデータ

`AttachmentId` はブランド付きの不透明な文字列である。ローカルバックエンドは現在 `sha256:<digest>` を出力するが、consumer はその表現を解析したり、そこからファイルシステムパスを導出したりしてはならない。consumer は `imageHostPath()` を通じて添付 provider にオブジェクトの場所を問い合わせてよく、その後、モデルの tool がその host パスを読めるかどうかを現在の実行環境のファイルシステムで判断しなければならない。

```ts type-equiv
/** Raster image formats accepted by the version-one attachment path. */
type ImageMediaType = 'image/png' | 'image/jpeg' | 'image/webp' | 'image/gif'
```

```ts type-equiv
/** Durable, serializable reference to one immutable normalized image. */
interface ImageAttachmentRef {
  /** Opaque storage identifier; never a filesystem path or bearer URL. */
  attachmentId: AttachmentId
  /** Media type verified from the stored bytes. */
  mediaType: ImageMediaType
  /** Exact encoded byte length. */
  bytes: number
  /** Intrinsic encoded width in pixels. */
  width: number
  /** Intrinsic encoded height in pixels. */
  height: number
  /** Optional display name stripped of local path information. */
  name?: string
  /**
   * Input dimensions after applying EXIF orientation and before normalization
   * scaling. Present only when normalization reduced the image.
   */
  originalDimensions?: {
    width: number
    height: number
  }
}
```

```ts type-equiv
/** Deployment-resolved limits used by upload admission and request buffering. */
interface ImageAttachmentLimits {
  maxImageBytes: number
  maxImagesPerMessage: number
  maxMessageImageBytes: number
  maxImagePixels: number
  /** Maximum intrinsic width and maximum intrinsic height in pixels for one image. */
  maxImageDimension: number
  mediaTypes: readonly ImageMediaType[]
}
```

ローカルバックエンドはメッセージごとに最大 20 枚の画像と 200 MiB のエンコード済みソースデータを受け入れる。1 つのソースは最大 50 MiB、64,000,000 ピクセル、どちらかの辺が最大 65,535 ピクセルまで使える。これらのソース制限は独立した正規化段階より前のものであり、正規化はデフォルトで長辺を 2048 ピクセル、エンコード済みデータを 4 MiB に制限する。

参照は本来の寸法とエンコード済み長を記録するため、クライアントは先にデコードせずに履歴をレイアウトできる。読み取りは保存されたオブジェクトを、渡された参照の ID、フォーマット、バイト長、幅、高さに対して検証する。任意の表示名と正規化前の寸法は、呼び出し元が提供した表示情報として、保存バイトに対して検証せずにそのまま返される。

## コミットと検証済み読み取りのペイロード

```ts type-equiv
/**
 * Browser-submitted prompt content accepted by Host prompt endpoints; the
 * accepting Host promotes image parts to durable references through
 * `ctx.attachments.admitPromptContent()` before any message is created, so a wire caller can
 * never cite an attachment it did not upload.
 */
type PromptContentPart =
  | { readonly type: 'text'; readonly text: string }
  | {
    readonly type: 'image'
    readonly mediaType: ImageMediaType
    readonly data: string
    readonly name?: string
  }
```

```ts type-equiv
/**
 * Host prompt content whose upload receipts are resolved. An image part
 * carries encoded bytes awaiting admission, or references an image this
 * service already stored for an upload the same Session staged, with
 * `encodedBytes` holding the byte length of that upload.
 */
type AttachmentAdmissionPart =
  | PromptContentPart
  | { readonly type: 'file'; readonly attachment: FileAttachmentRef }
  | { readonly type: 'image'; readonly attachment: ImageAttachmentRef; readonly encodedBytes: number }
```

```ts type-equiv
/** Host-admitted prompt content with every attachment represented by its durable reference. */
type AdmittedPromptContentPart =
  | { readonly type: 'text'; readonly text: string }
  | { readonly type: 'image'; readonly attachment: ImageAttachmentRef }
  | { readonly type: 'file'; readonly attachment: FileAttachmentRef }
```

```ts type-equiv
/** Base64-encoded image upload accompanying one wire request. */
interface EncodedImageAttachment {
  /** Declared media type, verified against the decoded bytes during admission. */
  mediaType: ImageMediaType
  /** Canonical base64 encoding of the image bytes. */
  data: string
  /** Optional display name; it is never interpreted as a path. */
  name?: string
}
```

```ts type-equiv
/** Request to validate and durably commit one image. */
interface SaveImageAttachment {
  data: Uint8Array
  /** Caller-declared media type, checked against fully decoded bytes. */
  mediaType: ImageMediaType
  /** Optional browser/provider display name; it is never interpreted as a path. */
  name?: string
}
```

```ts type-equiv
/** Stored image bytes returned after reference and digest verification. */
interface StoredImageAttachment {
  ref: ImageAttachmentRef
  data: Uint8Array
}
```

```ts type-equiv
/** Deterministic request-image target selected by one exact model route for one attachment. */
interface ImageRequestTarget {
  /** Target width in pixels; a target above the source keeps the source width. */
  width: number
  /** Target height in pixels; a target above the source keeps the source height. */
  height: number
  /** Encoded-byte target before base64 expansion or Files API upload; the smallest quality-ladder output is kept when no quality fits. */
  maxBytes: number
}
```

```ts type-equiv
/** Integer width and height of one projected image. */
interface ProjectedDimensions {
  width: number
  height: number
}
```

```ts type-equiv
/** Cached request version derived from one provider-independent normalized attachment. */
interface RequestImageAttachment {
  /** Cache and upload-index key over the attachment id, policy, and fixed encoder parameters. */
  variantId: ImageVariantId
  /** Durable normalized attachment from which this request version was derived. */
  attachment: ImageAttachmentRef
  /** Encoded request bytes. */
  data: Uint8Array
  mediaType: ImageMediaType
  bytes: number
  width: number
  height: number
  /** Provider-compatible sample depth proven after request encoding. */
  depth: 'uchar'
  /** Provider-compatible color space proven after request encoding. */
  space: 'srgb'
  /** Whether the encoded request version retains an alpha channel. */
  hasAlpha: boolean
}
```

`saveImage()` は provider 非依存の正規化済み添付を準備してアトミックにコミットしてから、その `ImageAttachmentRef` を返す。`saveImages()` は検証済みのすべての添付をバッチ公開前に一度だけ準備するため、検証拒否で部分的なオブジェクトが残らず、公開がデコードや品質選択を繰り返さない。`admitPromptContent()` はアップロードレシート解決後の完全で順序付けられた Host prompt を受け入れ、base64 画像アップロードを永続参照に置き換え、永続ファイル参照とすでに保存された画像をそのまま通し、すべての画像をメッセージごとの画像数と集約バイト制限に数える。保存済み画像はそのアップロードのバイトで数える。`admitEncodedImages()` は他の wire エントリをサポートし、数、集約バイト、順序付きバッチ受け入れを `saveImages()` に委譲する。`admitEncodedFile()` はエンコードされたプロトコルアダプタに同じサービス所有の正規の base64 受け入れを与え、`isAttachmentError()` はそれらのアダプタが実装ヘルパーをインポートせずに安定した添付失敗を認識できるようにする。`readImage()` は Session 履歴を参照せずに、正規化済みオブジェクトを渡された完全な参照に対して検証し、その参照をバイトとともに返す。アクセスポリシーは呼び出し元 API に属する。Session Controller はストアレベルの [画像アクセスルール](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-10-23-store-scoped-image-reads.md)を使う。`imageHostPath()` は provider 所有の host オブジェクトの場所だけを公開し、現在の tool 実行ワールドがそれを読めるかどうかは判断しない。`readImageRequest()` はルートが選んだ正確なターゲットサイズとエンコード済みバイトターゲットで、1 つの決定的なリクエストバージョンを導出してキャッシュする。そのバージョンはエンコード済みバイトとメタデータを含むが、実行ワールドのパスは含まない。新しいエントリは公開前に完全にデコードされ、キャッシュヒットは有界のメタデータプローブを使う。呼び出し元は順序付きバッチが必要なとき、単数メソッドに `Promise.all` を使う。ローカル実装は優先候補を遅延エンコードし、等しいリクエスト ID を singleflight し、各待機者が独立してキャンセルできるようにし、待機者がいなくなったら共有作業を停止し、すべての変換をインスタンスレベルのリミッタで制限する。リミッタはデフォルトで同時 2 変換である。サービスは保持について中立である。再開されたセッションと fork されたセッションはオブジェクトを共有できるため、参照を意識したガベージコレクションは 1 つのセッションの削除に結び付けず延期される。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxattachments--attachmentstore-abstract-seam"></a>

### `ctx.attachments` — `AttachmentStore`（抽象 seam）

不変のバイナリ添付サービス。実装は参照を公開する前にバイトを検証する。

```ts cordis-catalog
/**
 * Validate one image without persisting it.
 * Batch callers validate every member before saving any member.
 * @param input - encoded bytes, declared media type, and optional display name.
 * @returns completion after the encoded raster has been fully decoded.
 */
abstract validateImage(input: SaveImageAttachment): Promise<void>

/**
 * Validate and durably commit one ordered image batch.
 * @param inputs - encoded images in owning-message order.
 * @returns durable normalized attachment references in the same order after every member succeeds.
 */
async saveImages(inputs: readonly SaveImageAttachment[]): Promise<readonly ImageAttachmentRef[]>

/**
 * Admit one Host prompt and replace each encoded image with its durable reference.
 * Text, durable file references, and already stored images pass through unchanged.
 * Every image part counts toward `maxImagesPerMessage` and `maxMessageImageBytes`;
 * a stored image counts the bytes of the upload that stored it. Refused content
 * starts no writes. A prompt without encoded images performs no storage operation.
 * @param content - prompt parts in message order after upload receipt resolution.
 * @returns admitted prompt parts in the same order as `content`.
 * @throws AttachmentError when the image batch is refused.
 */
async admitPromptContent( content: readonly AttachmentAdmissionPart[], ): Promise<AdmittedPromptContentPart[]>

/**
 * Decode and durably commit one canonical base64 file upload.
 * @param input - canonical base64 bytes and optional display name.
 * @returns the durable content-addressed file reference.
 * @throws AttachmentError when the encoding or storage operation is refused.
 */
admitEncodedFile(input: EncodedFileAttachment): Promise<FileAttachmentRef>

/**
 * Identify a failure emitted by this attachment capability by its stable code.
 * @param error - value caught from an attachment operation.
 * @returns whether the value is an attachment failure.
 */
isAttachmentError(error: unknown): error is AttachmentError

/**
 * Validate and durably commit one image before its owning session event is appended.
 * The returned reference describes the persisted normalized image. When
 * normalization reduces the raster, its `originalDimensions` records the
 * orientation-applied input dimensions.
 * @param input - encoded bytes, declared media type, and optional display name.
 * @returns the durable content-addressed normalized image reference.
 */
abstract saveImage(input: SaveImageAttachment): Promise<ImageAttachmentRef>

/**
 * Read one image and verify its ID, format, byte length, and dimensions against the supplied reference.
 * @param ref - supplied image reference; name and originalDimensions are preserved without file verification.
 * @param signal - optional cancellation for backend read and verification work.
 * @returns the verified bytes and normalized attachment reference.
 * @throws the signal reason when aborted, or a storage error when verification fails.
 */
abstract readImage(ref: ImageAttachmentRef, signal?: AbortSignal): Promise<StoredImageAttachment>

/**
 * Locate the provider-owned normalized object in the harness host filesystem.
 * @param ref - durable normalized attachment reference.
 * @returns an absolute host path, or undefined when this backend is not host-file-backed.
 * @throws an AttachmentError when the durable reference is invalid.
 */
imageHostPath(ref: ImageAttachmentRef): string | undefined

/**
 * Durably commit one file byte-for-byte before its owning session event is
 * appended. Files carry no admission limits: any byte content and length is
 * accepted, and the stored object is the exact submitted bytes. Backends
 * without verbatim file storage keep this default rejection.
 * @param input - exact bytes and optional display name.
 * @returns the durable content-addressed file reference.
 */
saveFile(input: SaveFileAttachment): Promise<FileAttachmentRef>

/**
 * Durably commit one file byte-for-byte from bounded chunks. Providers must
 * apply backpressure and must not collect the complete file in memory.
 * Backends without streamed verbatim storage keep this default rejection.
 * @param input - ordered exact bytes, optional cancellation, and display name.
 * @returns the durable content-addressed file reference.
 */
saveFileStream(input: SaveFileStreamAttachment): Promise<FileAttachmentRef>

/**
 * Read and verify one verbatim stored file as bounded chunks. Providers must
 * not collect the complete file in memory. Backends without verbatim file
 * reads keep this default rejection.
 * @param ref - durable file reference from the session log.
 * @param signal - optional cancellation for backend reads and verification work.
 * @returns exact file bytes in order; integrity failures reject the iteration.
 */
async *readFileStream( ref: FileAttachmentRef, signal?: AbortSignal, ): AsyncIterable<Uint8Array>

/**
 * Locate the verbatim stored file object in the harness host filesystem.
 * @param ref - durable file reference.
 * @returns an absolute host path, or undefined when this backend is not host-file-backed.
 * @throws an AttachmentError when the durable reference is invalid.
 */
fileHostPath(ref: FileAttachmentRef): string | undefined

/**
 * Generate or read one deterministic model-request version from the stored normalized image.
 * @param ref - durable provider-independent normalized attachment reference.
 * @param target - route-chosen dimensions and byte target; an unmet byte target yields the smallest ladder output.
 * @param signal - optional cancellation.
 * @returns request bytes and the cache/upload identity covering every transform input.
 */
readImageRequest( ref: ImageAttachmentRef, target: ImageRequestTarget, signal?: AbortSignal, ): Promise<RequestImageAttachment>
```

ソース: [`packages/attachment/attachment/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/attachment/attachment/src/index.ts)

<a id="ctxfileuploads--fileuploads"></a>

### `ctx.fileUploads` — `FileUploads`

アップロードストレージと、受け取る各 Agent の正確な Session でキー付けられたレシートを所有する Host サービス。

```ts cordis-catalog
/**
 * Register the ordinary-Session resolver used when a raw upload addresses a cold Session.
 * @param resolve - resolver that returns the exact live Agent or throws a Remote error.
 * @returns disposer removing this resolver.
 */
registerAgentResolver(resolve: AgentResolver): () => void

/**
 * Persist one encoded upload and stage it under the Agent receiver selected by Typert.
 * @param agent - receiving Agent resolved from the Remote Agent scope.
 * @param request - canonical base64 bytes and optional display name.
 * @param signal - caller cancellation before storage begins.
 * @returns the staged receipt and durable file reference.
 */
@Remote('upload') async upload(agent: Agent, request: EncodedFileUploadRequest, signal: AbortSignal): Promise<FileUploadValue>

/**
 * Persist raw chunks for one Session without aggregating the upload.
 * @param request - Session identity, ordered bytes, cancellation, and optional display name.
 * @returns the staged receipt and durable file reference.
 */
async uploadStream(request: { readonly sessionId: SessionId readonly data: AsyncIterable<Uint8Array> readonly signal?: AbortSignal readonly name?: string }): Promise<FileUploadValue>

/**
 * Persist one prompt image for a Session: read at most the smaller of the
 * deployment's per-image and per-message byte limits, then validate,
 * normalize, and store the image the way prompt admission does, so a later
 * prompt cites it by receipt.
 * Cancellation before storage begins stores nothing; a cancelled upload
 * never receives a receipt, although an image whose storage already began
 * stays in the content-addressed store.
 * @param request - Session identity, ordered bytes, the declared media type that this deployment
 * must accept, cancellation, and optional display name.
 * @returns the staged receipt and durable normalized image reference.
 */
async uploadImageStream(request: { readonly sessionId: SessionId readonly data: AsyncIterable<Uint8Array> readonly mediaType: string readonly signal?: AbortSignal readonly name?: string }): Promise<ImageUploadValue>

/**
 * Resolve one staged receipt for the receiving Agent's exact Session.
 * @param agent - receiving Agent.
 * @param receiptId - opaque receipt minted for one completed upload.
 * @returns the staged file or image, or `undefined` for an unknown or foreign receipt.
 */
resolve(agent: Agent, receiptId: FileUploadReceiptId): StagedAttachment | undefined

/**
 * Bind receipts while one prompt enters an Agent inbox.
 * Disposal restores every prior binding unless the caller commits successful delivery.
 * @param agent - receiving Agent.
 * @param receiptIds - distinct staged receipts referenced by the prompt.
 * @param requestId - prompt identity later observed in queue or history.
 * @returns binding kept after commit until queue or history observation retires its receipts.
 */
bindPrompt( agent: Agent, receiptIds: readonly FileUploadReceiptId[], requestId: string, ): PromptFileBinding

/**
 * Retire every receipt accepted by one removed queue occurrence.
 * @param agent - receiving Agent.
 * @param requestId - prompt identity carried by the queue occurrence.
 */
retirePrompt(agent: Agent, requestId: string): void
```

型: [Agent](core.md) · [SessionId](core.md)

ソース: [`packages/client/file-upload/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/file-upload/src/index.ts)
<!-- END GENERATED cordis-surface -->
