# Webhook ランタイム

日本語 | [English](../../../docs/subsystems/webhook.md) | [中文](../../../docs/subsystems/webhook.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/webhook.md](../../../docs/subsystems/webhook.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

Webhook サブシステムは、認証済みの外部配信を任意の通常のルート Session に変える。プロバイダー・アダプターは認証と汎用 JSON の受け入れを所有し、信頼されたプログラム的ルールは条件と外部呼び出しを所有し、`ctx.webhookRuntime` はコールバックの寿命と Workspace に支えられた Session の作成を所有する。[過去の実装済み決定](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/archived/feature/2026-08-22-fire-and-forget-webhook-sessions.md)は、ランタイムが配信や完了の状態を保持しない理由を記録している。

## 共有される値

`WebhookRuleId`、`WebhookSourceId`、`WebhookDeliveryId` は不透明な文字列である。配信 id はプロバイダーの識別子だけを記録し、ランタイムはそれを保存も重複排除もしない。

`WebhookEventMap` はプロバイダー種別によってマージ拡張可能である。`WebhookEventOf<K>` は既知のプロバイダー・イベントを選択し、それ以外では汎用のロスレス JSON を受け入れるので、ランタイム・パッケージを変えずにツリー外のアダプターを使える。

`VerifiedWebhookDelivery<K>` は `kind`、設定された `source`、プロバイダーの `deliveryId`、正規化された `event`、非負の安全な整数の `receivedAt` を含む。ランタイムは値を複数のルールにディスパッチする前に、値全体を検証し、切り離し、凍結する。

`WebhookRule<K>` は一意の id、プロバイダー種別、`run(delivery, signal)` を含む。コールバックは任意の信頼されたコードを実行してよい。`null` または 1 つの `WebhookSessionRequest` を返し、登録のアンロード時に停止すべき非同期処理のためにシグナルを監視しなければならない。

`WebhookSessionRequest` は絶対的な `workspacePath`、タイトル、テキスト・プロンプト、agent プリセット、permission プリセットを必要とする。任意の `model` は明示的なプロバイダー／モデル・ルートと、任意の出力トークン上限を指名し、そのアダプターの推論デフォルトを使う。省略した場合は、最初の要求が永続ヘッダーを記録するまで、推論エフォートを含む現在のデプロイ選択全体をスナップショットする。

## ファイア・アンド・フォーゲットのディスパッチ

`dispatch()` は一致するルールをスナップショットし、それぞれを独立にスケジュールし、どのコールバックも落ち着く前に返る。throw と拒否はルールごとに封じ込められる。登録の破棄は、アクティブな呼び出しを中止して排出する前にルールを削除するので、後の配信がアンロード中のコードに入ることはない。

ランタイムにはキュー、再試行、重複排除、実行ステータス、クラッシュ・リプレイ、Agent ステータス・リスナー、完了結果がない。配信の繰り返しは Session の繰り返しを作成しうる。唯一のアクティブ操作テーブルはプライベートな解体の記帳であり、プロセスとともに消える。

## Session の作成

非 null の結果は非同期プリフライトの前にスナップショットされる。ランタイムは permission と agent のプリセットを検証し、正規の Workspace を解決または作成し、Session の cwd が Workspace のパスに等しい Agent を作成し、公開前に選択された agent プリセットをマウントし、permission、タイトル、最初のフォローアップを適用する前に Session を永続的にアタッチする。

フォローアップは `source.kind: "webhook"` と、プロバイダー、ソース、配信、ルールの識別子を持つ、通常の永続的な user ロール・メッセージである。その受け入れられた受信箱への挿入が webhook 操作をコミットする。ランタイムはターンを特別にフラッシュしたり待ったりせず、その後の通常の Session 永続化と Agent ライフサイクルが適用される。

アタッチの失敗は、プロンプトが存在する前に新しい Agent を破棄する。アタッチとプロンプトのアドミッションの間の失敗は、元のエラーを置き換えずに Workspace のデタッチと Agent の破棄を試みる。プリフライト中に自動的に作成された Workspace は、別の並行する呼び出し元がすでに使っている可能性があるため残る。

## GitHub アダプター

`@deepseek-ai/dsh-webhook-github` は、注入された WebServer 上に正確なルートを登録し、要求ごとにその資格情報参照を解決し、パース前に手を加えていない `application/json` ボディを検証し、インメモリのディスパッチ直後に `202` を返す。その正規化されたイベントは、署名済みのロスレス JSON オブジェクトを保証し、ルールは自身が消費するイベント固有のフィールドを検証する。

[GitHub レビュー・ガイド](../user/guide/github-review.md)はこのルートを分離された 2 つ目の WebServer にマウントするので、webhook の入り口を公開してもブラウザー API は公開されない。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxwebhookruntime--webhookruntime"></a>

### `ctx.webhookRuntime` — `WebhookRuntime`

ファイア・アンド・フォーゲットのルール・ランタイム。Session の作成が唯一の組み込みアクションである。

```ts cordis-catalog
/**
 * Register one trusted programmatic rule.
 * @param rule - unique id, provider kind, and arbitrary callback.
 * @returns awaitable effect disposer that aborts and drains this rule's active callbacks.
 */
register<K extends string>(rule: WebhookRule<K>): () => Promise<void>

/**
 * Start every currently matching rule and return before any callback settles.
 * @param delivery - authenticated provider data; snapshotted before dispatch.
 * @throws synchronously when the runtime is closing or the delivery is malformed.
 */
dispatch<K extends string>(delivery: VerifiedWebhookDelivery<K>): void
```

ソース: [`packages/experimental/webhook/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/webhook/src/index.ts)
<!-- END GENERATED cordis-surface -->
