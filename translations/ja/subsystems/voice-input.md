# 音声入力

日本語 | [English](../../../docs/subsystems/voice-input.md) | [中文](../../../docs/subsystems/voice-input.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/voice-input.md](../../../docs/subsystems/voice-input.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

実験的な音声認識には 3 つの役割がある。[サービス定義](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/speech-to-text/README.md)は名前付きプロバイダーをルーティングし、[SenseVoice プロバイダー](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/speech-to-text-sensevoice/README.md)はローカル推論を所有し、[Remote コンシューマー](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/api-speech-to-text/README.md)はブラウザーに配信する。[任意の bundle](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/voice-input-bundle/README.md)がそれらをマイク UI と合成する。

## プロバイダー選択

`SpeechProviderId` は登録のアイデンティティをブランド化する。`SpeechProviderInfo` は表示名、受け付ける `languages` ヒント、`host-local` または `cloud` の処理場所を運ぶ。`SpeechRequest` は WAV バイトと任意のプロバイダー/言語選択を含み、`resolve()` は 1 つのキャプチャ済みプロバイダー・インスタンスを持つ `SpeechSpec` を生成する。プロバイダーが見つからない場合や言語がサポートされていない場合は失敗する。置き換えや撤回は解決済みの作業を無効にする。音声が別のプロバイダーにフォールバックすることは決してない。

`SpeechProvider.transcribe()` は 1 つの完全な `SpeechInput` と `AbortSignal` を受け取る。`Transcript` はプレーンな `text`、`audioSeconds`、`inferenceSeconds` を返す。プロバイダーはキャンセルを尊重し、登録解除が完了する前に作業を決着させる。ローカル・プロバイダーは呼び出しを直列化し、キューを制限し、ワーカーのライフタイムを Session とは独立に管理する。

## ブラウザーと Host の所有権

ブラウザーはマイク・トラックと未送信の下書きを所有する。`TranscriptionRequest` は正規の base64 PCM16 WAV を認証済み Remote 経由で送る。`SpeechCatalog` は利用可能なプロバイダー、既定の選択、録音のバイト/時間上限を通知する。API は認識の前にこのプロセス入力を検証する。

入力ファサードは録音の前にリビジョンを持つ選択をキャプチャする。`InputActions.insertText()` は、選択リビジョンが最新で、送信が編集を許す間だけ、1 つの undo 可能なプレーン・テキスト編集を挿入する。拒否された挿入は、トランスクリプトを明示的な挿入のために利用可能なまま残す。Session の切り替えやプラグインの破棄は、遅れて届いた結果を無効にする。認識自体は Session イベントを書き込まない。最終的なモデル可視テキストは通常のユーザー送信が所有する。

## 準備と設定

`SpeechProviderInfo.downloadSources` は準備の取得元を通知する。`SpeechPreparationOptions.downloadSource` はタスクのために 1 つを選択し、省略時はプロバイダーのポリシーを保つ。SenseVoice は通知された選択を検証し、手動のソースをフォールバックなしで固定し、準備の実行中のソース変更を拒否する。UI は現在のカード内でリトライ用の選択を保持するが、それは永続化された認識設定ではない。

Host プロバイダーは、ページと Session の変更をまたいで 1 つの準備タスクを所有する。Client はコンポーザー、セットアップ・プロンプト、bundle 詳細の間で 1 つの `follow()` 購読を共有する。任意の `SpeechSetupEstimate` メタデータは、測定された進捗とは別に、プロバイダー固有の計画ヒントを提供する。`SpeechPreparationStepKind` は順序付きのリソース操作を識別し、`SpeechPreparationStep` は各ステータスと開始時刻を記録する。完全な `SpeechPreparationState` はキャンセルまたは失敗の後もこれらのステップを保持する。折りたたまれた UI は現在の操作を表示し、展開するとすべてのステップを列挙し、実行中のステップについてのみバイト進捗または経過時間を示す。オブザーバーを閉じても準備がキャンセルされることは決してない。検証済みファイルはリトライとアイドル・ワーカーの回収を生き延びる。

失敗した準備は、アセット、取得元のオリジン、分類された理由、任意の診断コードまたは HTTP ステータスを含む `SpeechDownloadFailure` を含みうる。Client は復旧のアドバイスをローカライズし、生のダウンロード原因は Host に留まる。

マイクはモデル・セレクターと Send の間の `conversation.input.activity` を占める。クリックでキャプチャが始まりツールバーが展開する。Stop は文字起こしして下書きに挿入する。このアクティビティはエディターと送信アクションを保持し、ローカル・フィードバックを所有し、アンマウント時に展開を解放する。キャンセル、Escape、またはページを隠すことはキャプチャを破棄する。波形履歴は測定されたマイク振幅を表示する。bundle 詳細には認識設定、準備状態、進捗が含まれる。明示的な有効化は `plugins.bundle.activation` を使い、モデルが不足しているユーザーをセットアップへ導く。リストには bundle の説明とスイッチだけが表示される。

## 設計根拠

[音声入力の決定](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-09-16-experimental-voice-input.md)は、一時的な音声、明示的なプロバイダー選択、遅延ローカル準備を説明する。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。dispatch モードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxspeechcontroller--speechcontroller"></a>

### `ctx.speechController` — `SpeechController`

音声呼び出しは Agent をアクティベートしたり、Agent に送信したりすることは決してない。

```ts cordis-catalog
/**
 * Read provider choices without preparing a recognizer.
 * @returns available providers, resolved default, and recording limits.
 */
@Remote catalog(): SpeechCatalog

/**
 * Follow provider readiness independently of Session and preparation lifetimes.
 * @param signal - Client observation lifetime.
 * @returns initial and subsequent complete readiness snapshots.
 */
@Remote({ mode: 'stream' }) async *follow(signal: AbortSignal): AsyncIterable<SpeechCatalog>

/**
 * Persist the user's recognition preferences.
 * @param patch - changed preference fields.
 * @returns after preferences are saved.
 */
@Remote configure(patch: SpeechSelectionPatch): Promise<void>

/**
 * Start or join one Host-owned preparation task.
 * @param providerId - selected recognizer.
 * @param options - task-local source selection validated by the provider.
 */
@Remote prepare(providerId: SpeechProviderId, options?: SpeechPreparationOptions): void

/**
 * Explicitly cancel resource preparation.
 * @param providerId - selected recognizer.
 * @returns after the preparation task settles.
 */
@Remote cancelPreparation(providerId: SpeechProviderId): Promise<void>

/**
 * Validate and transcribe one recording through the explicit provider selection.
 * @param request - canonical WAV encoded as base64, provider id and language hint.
 * @param signal - Client cancellation or Remote contribution disposal.
 * @returns final transcript without adding a Session event.
 */
@Remote async transcribe(request: TranscriptionRequest, signal: AbortSignal): Promise<Transcript>
```

ソース: [`packages/experimental/api-speech-to-text/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/api-speech-to-text/src/index.ts)

<a id="ctxspeechtotext--speechtotext"></a>

### `ctx.speechToText` — `SpeechToText`

1 つの Host 合成内のすべての文字起こしコンシューマーが共有するレジストリ。

```ts cordis-catalog
/**
 * Register one recognizer; duplicate ids fail without replacing the original.
 * @param provider - recognizer owned by the contributing fiber.
 * @returns idempotent disposer which rejects admission, cancels, and joins accepted work.
 */
register(provider: SpeechProvider): () => Promise<void>

/**
 * Read the current recognizer roster.
 * @returns available provider facts in registration order.
 */
listProviders(): readonly SpeechProviderInfo[]

/**
 * Observe complete readiness snapshots; a slow reader coalesces intermediate progress.
 * @param caller - observer lifetime, independent of any preparation task.
 * @returns an initial snapshot followed by the latest provider states.
 */
async *follow(caller: AbortSignal): AsyncIterable<SpeechSnapshot>

/**
 * Read provider readiness and current user preferences together.
 * @returns one detached complete observation.
 */
snapshot(): SpeechSnapshot

/**
 * Persist changed selection fields into this plugin's profile entry; the resulting language must be accepted by the selected provider.
 * @param patch - explicit provider or language changes.
 * @returns after the profile write and the live update it applies.
 */
async configure(patch: SpeechSelectionPatch): Promise<void>

/**
 * Start or join provider-owned preparation.
 * @param id - exact registered provider identity.
 * @param options - task-local source selection validated by the provider.
 */
prepare(id: SpeechProviderId, options?: SpeechPreparationOptions): void

/**
 * Explicitly cancel provider preparation without tying it to a browser connection.
 * @param id - exact registered provider identity.
 * @returns after the preparation task settles.
 */
async cancelPreparation(id: SpeechProviderId): Promise<void>

/**
 * Apply composition defaults and capture the selected provider. Missing providers and unsupported languages fail explicitly.
 * @param request - complete recording and optional selection.
 * @returns provider-pinned input for transcribe().
 */
resolve(request: SpeechRequest): SpeechSpec

/**
 * Execute exactly the resolved provider; no fallback sends audio elsewhere.
 * @param spec - resolved input; a withdrawn or replaced registration is rejected.
 * @param signal - caller cancellation.
 * @returns final transcript after provider settlement.
 */
async transcribe(spec: SpeechSpec, signal: AbortSignal): Promise<Transcript>
```

ソース: [`packages/experimental/speech-to-text/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/speech-to-text/src/index.ts)
<!-- END GENERATED cordis-surface -->
