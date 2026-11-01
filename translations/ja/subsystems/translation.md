# テキスト翻訳

日本語 | [English](../../../docs/subsystems/translation.md) | [中文](../../../docs/subsystems/translation.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/translation.md](../../../docs/subsystems/translation.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

実験的な[翻訳器](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/translator/README.md)は、匿名のブラウザ・エンドポイント、または明示的に選択された有料のネイティブ Flash ルートを通じてテキストを翻訳する。表示、スケジューリング、Session のアクティブ化はコンシューマーが所有する。Session に束縛された要求は、アクティブな Session ライターを通じて再利用可能な結果を保持する。

## 要求と結果

`TranslationProvider` は `bing`、`google`、`deepseek-account`、`deepseek-official` のいずれかを選択する。`TranslationRequest` は `text`、`targetLanguage`、および任意の `sourceLanguage`、`provider`、ブランド付き `sessionId` を提供する。`resolve()` は明示的なルーティングを `TranslationSpec` に取り込む。設定されたプロバイダーはデフォルトで Bing になり、ソース言語を省略すると自動検出を使う。一般的な中国語ロケール・タグは、選択されたプロバイダーの言語コードにマップされる。`translate()` は解決されたプロバイダーからのプレーン・テキストを返す。

サービスは設定された `maxTextChars` を公開する。これは UTF-16 コード単位で測られる。コンシューマーは長い入力を翻訳要求の前に分割する。各呼び出しはレスポンスのバイト数を制限し、レスポンス・ボディの読み取りをそのデッドラインに含める。呼び出し元のキャンセルとサービスの破棄は受け入れ済みの要求を中止し、破棄はそれらが落ち着くのを待つ。

## 失敗とデータの扱い

`TranslationError` は `TranslationErrorCode` を運び、テキスト制限、HTTP 失敗、無効なレスポンス、レスポンス制限、所有デッドライン、要求失敗を区別する。`TRANSLATION_SESSION_REQUIRED` は永続化バックエンドまたは Session の欠如を拒否する。`TRANSLATION_SESSION_INACTIVE` はキャッシュされていない非アクティブな Session をプロバイダーへのディスパッチ前に拒否する。`TRANSLATION_STORAGE_ERROR` は無効な保存レコードまたは永続化の失敗を拒否する。プロバイダーの診断は送信されたテキストとレスポンス・ボディを省略する。失敗した呼び出しは別のプロバイダーを選択しない。レスポンスは外部 JSON として検証され、リダイレクトは拒否される。

`TRANSLATION_UNAVAILABLE` は、アクティブな組み込み owner または登録済みルートの欠如、資格情報の未設定、正確な `deepseek-flash` カタログ・メンバーシップの欠如、owner、モデル設定（Official の `models` または Account の `modelOverrides`）、またはディスパッチ前の Official 資格情報参照の変更、プロバイダーの撤退、あるいは推論エフォートが `off` でない準備済み呼び出しを拒否する。reason が `error` のネイティブ `finish` は `TRANSLATION_REQUEST_FAILED` にマップされる。finish が欠如しているか、`stop` 以外の値である場合は `TRANSLATION_INVALID_RESPONSE` にマップされる。

Google はソース・テキストをフォーム POST ボディで受け取り、Bing は JSON POST ボディで受け取る。どちらもリモート・サービスである。それらの匿名エンドポイントは、サポートされる開発者向けの可用性やクォータを保証しない。地域ごとの接続性はユーザーのネットワークで検証する必要がある。

## 永続的な翻訳

Session に束縛された呼び出しは、すべてのプロバイダーで `plugin:translator/request` と `plugin:translator/result` を使う。要求は正確なソース、言語タグ、選択されたプロバイダー、レシピのアイデンティティを保持する。成功した結果はその要求シーケンスを参照する。任意の要求メタデータは、これらのレコード型を変えずにプロバイダー固有の詳細を保持する。翻訳器は外部ディスパッチの前に要求を、返す前に完了した結果をチェックポイントする。読み取り専用のキャッシュ検索はプロバイダーの処理に先行し、ライブな Session を必要としない。失敗や未完了の試行は再利用可能なテキストを提供しない。新しいレコードはアクティブな Session の既存のライターを使う。翻訳器は書き込みハンドルを開かず、Session をアクティブ化しない。翻訳レコードは元のイベントとモデル入力を保持する。Session のない匿名呼び出しはステートレスなままであり、有料ルートは Session を必要とする。実験的なレコードの将来の形式移行にわたる保持はベストエフォートである。

有料ルートのキャッシュ・ミスには、アクティブな組み込み Account または Official の owner、設定済みの資格情報、正確な `deepseek-flash` カタログ・メンバーシップが必要である。名前やバージョン・ラベルは無関係であり、pi-ai は除外される。各ミスは思考をオフにした、テキストのみの別個のクエリを準備する。その正確な設定、システム、固有のユーザー・フラグメントは、同じ要求レコード内の任意の `metadata.modelRequest` を占める。保存済みの結果はこのアドミッションの前に読まれるため、資格情報の欠如が既存の翻訳を無効化することはない。ネイティブ要求の拡張はメイン Session のアイデンティティを受け取らず、その完全なログをアップロードできない。

## 推論の表示

[任意のバンドル](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/cot-translation-bundle/README.md)は、翻訳器を [GUI コンシューマー](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/client-ui-cot-translation/README.md)と組み合わせる。その `cotTranslation` Remote は認証済みのブラウザ呼び出しを提供する。Session が非アクティブで保存済みの結果がない場合、Host は通常の Session コントローラーのアクティブ化に参加し、1 回再試行する。アクティブ化は通常のプリセット・フック、再開マーカー、中断されたターンの修復を保持する。キャンセルは呼び出し元の待機を止め、共有のアクティブ化をキャンセルせずに、後の翻訳呼び出しを防ぐ。バンドルの設定は翻訳サービスとターゲット言語を選択し、`auto` はアクティブなインターフェース言語に従う。

`CotTranslationPreferences` は受け入れられたプロバイダーとターゲット言語を含む。`CotTranslationSnapshot` はこれらの設定を、翻訳器の現在の `maxTextChars` と対象となるプロバイダー ID と組み合わせる。ブラウザは翻訳コントロールを登録する前にこの Host スナップショットを読み、ローカルの設定ストレージが利用できない場合も同様である。受け入れられた設定の更新と再接続は両方の値をリフレッシュする。要求制限が変わると、拡張されたテキストをキャンセルして再度分割する。

GUI は session スコープの `conversation.chat.reasoning.body` 単一 Slot に登録し、Chat のデフォルト・エントリーを置き換える。両方のエントリーは公式の `conversation.chat.reasoning.content` Factory を再利用する。翻訳ラッパーは表示テキストを選択し、そのツールバーとビュー状態を所有し、Markdown のレンダリングを重複させずにローカライズされたラベルを提供する。有効な間、GUI は完了した段落と、長いストリーミング段落の要求サイズの接頭辞を、展開後のみ翻訳し、完了した結果を現在の Session から復元する。そのローカル・キャッシュはその開示を最適化するだけである。未完了の末尾は元のまま残る。元のテキストはアクセス可能なままである。開示を閉じること、プロバイダーや言語を切り替えること、アンロードは未完了の呼び出しをキャンセルし、遅れた結果を除外する。プロバイダーの失敗は元のテキストを保持し、明示的な再試行を必要とする。翻訳されたテキストが元の Session イベントやモデル入力を置き換えることは決してない。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxcottranslation--cottranslationcontroller"></a>

### `ctx.cotTranslation` — `CotTranslationController`

任意の推論翻訳は、Session に束縛されたストレージを翻訳器に委譲する。

```ts cordis-catalog
/**
 * Read accepted translation preferences and the current request limit without sending text.
 * @param signal - browser query cancellation or Remote contribution withdrawal.
 * @returns authoritative preferences, eligible routes, and maximum UTF-16 text length per request.
 */
@Remote async limits(signal: AbortSignal): Promise<CotTranslationSnapshot>

/**
 * Translate one displayed fragment through the reader's selected provider.
 * @param request - original text and Session identity supplied by the Client; provider and explicit
 * target language match accepted preferences, while auto uses the browser locale.
 * @param signal - browser cancellation or Remote contribution withdrawal.
 * @returns translated text; an uncached inactive Session joins ordinary GUI activation before retry.
 * Failures omit the source and provider response. Cancellation stops this caller's wait, not shared activation.
 */
@Remote async translate(request: TranslationRequest, signal: AbortSignal): Promise<string>
```

ソース: [`packages/experimental/client-ui-cot-translation/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/client-ui-cot-translation/src/index.ts)

<a id="ctxtranslator--translator"></a>

### `ctx.translator` — `Translator`

明示的なルーティング、キャンセル、静かなアンロードを備えた 1 つの Host サービス。

```ts cordis-catalog
/**
 * Inspect eligible native routes without inference; saved results remain readable for unavailable routes.
 * @param signal - optional caller cancellation, combined with service disposal.
 * @returns anonymous choices followed by eligible explicitly selected paid routes.
 */
async availableProviders(signal?: AbortSignal): Promise<readonly TranslationProvider[]>

/**
 * Resolve provider and source-language defaults without sending text.
 * @param request - consumer text, destination and optional routing choices.
 * @returns a complete specification; exceeding `maxTextChars` throws `TRANSLATION_TEXT_LIMIT`.
 */
resolve(request: TranslationRequest): TranslationSpec

/**
 * Translate one resolved specification; the selected provider receives its text.
 * @param spec - complete routing and language choices from `resolve()`.
 * @param signal - optional caller cancellation, combined with service disposal.
 * @returns translated plain text, durably retained before return when a Session is supplied.
 * An uncached supplied Session must be active; otherwise rejects with `TRANSLATION_SESSION_INACTIVE` before dispatch.
 * Rejects provider/storage/limit failures and preserves cancellation reasons.
 */
async translate(spec: TranslationSpec, signal?: AbortSignal): Promise<string>
```

ソース: [`packages/experimental/translator/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/translator/src/index.ts)
<!-- END GENERATED cordis-surface -->
