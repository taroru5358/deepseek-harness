# 作業ディレクトリ

日本語 | [English](../../../docs/subsystems/working-directory.md) | [中文](../../../docs/subsystems/working-directory.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/working-directory.md](../../../docs/subsystems/working-directory.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

Session のアクティブ・ディレクトリは [`dsh-working-directory`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/working-directory/README.md) が所有する。不変のヘッダーが元のプロジェクトを識別し、`working-directory/change` が、その後のディレクトリ・ベースの操作で使われる実効ディレクトリを記録する。イベント宣言は生成された[永続化カタログ](../../../docs/persistence-catalog.md)が所有する。

## ディレクトリの選択

`ctx.workingDirectory.get(session)` は記録された値を読む。`ensure(agent)` はそれを検証し、ディレクトリが消えていた場合は元のプロジェクトを復元する。`set(agent, path)` は要求された変更を検証して記録する。ヘッダー・ディレクトリを持たない Session は設定されたランタイム・フォールバックを使い、フォールバックが利用できない場合はエラーになる。

変更は 1 つの Session にローカルである。相対的なファイルシステム操作、新しく開始されたプロセス、指示の発見、スキル、ファイル補完はアクティブ・ディレクトリを使う。既存のターミナルと永続シェルは自身のディレクトリを保持する。サンドボックスの書き込み可能ルートとプロジェクト・メンバーシップは元のプロジェクトに固定されたままである。

## モデル・コンテキスト

初期値とその後の現在値は、任意のランタイム・コンテキストが無効な場合でも、必須のユーザー・コンテキスト・スナップショットとしてモデルに届く。コミットされた各変更は、通常の Agent 受信箱を通じて通知をキューに入れようとする。キャンセルや破棄は、要求のアドミッション前に通知を破棄しうる。ディレクトリ・イベントは存続し、次にアドミットされた要求が現在値を受け取る。ディレクトリ・コンテキストはシステム・プロンプトに決して入らない。[`working_directory`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/tool-working-directory/README.md)は任意の `cd` 引数を 1 つ公開する。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxworkingdirectory--workingdirectoryservice"></a>

### `ctx.workingDirectory` — `WorkingDirectoryService`

各 Session の実効ディレクトリと、そのモデル可視の変更に対する 1 つの所有者。

```ts cordis-catalog
/**
 * Read the committed directory without filesystem I/O.
 * @param session - Session whose directory is requested.
 * @returns its effective absolute directory.
 */
get(session: Session): string

/**
 * Validate the current directory and restore the original project when it disappeared.
 * @param agent - live or unpublished Agent owning the Session.
 * @param signal - cancellation for filesystem inspection.
 * @returns the existing directory; recovery is committed before fulfillment.
 * A notice failure is warned without reverting the committed state.
 * @throws when the original project is also unavailable.
 */
ensure(agent: Agent, signal?: AbortSignal): Promise<string>

/**
 * Change one Session's directory without changing existing processes or permissions.
 * @param agent - live or unpublished Agent owning the Session.
 * @param path - absolute path or a path relative to its current directory.
 * @param signal - cancellation before the durable change.
 * @returns the canonical absolute directory, committed before fulfillment.
 * A notice failure is warned; the next request still receives the committed directory.
 * @throws when the requested path is not an existing directory.
 */
set(agent: Agent, path: string, signal?: AbortSignal): Promise<string>
```

型: [Agent](core.md) · [Session](session.md)

ソース: [`packages/session/working-directory/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/working-directory/src/index.ts)
<!-- END GENERATED cordis-surface -->
