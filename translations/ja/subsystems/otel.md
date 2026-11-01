# OTel レポート

日本語 | [English](../../../docs/subsystems/otel.md) | [中文](../../../docs/subsystems/otel.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/otel.md](../../../docs/subsystems/otel.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

[OTel プラグイン](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/telemetry/otel/README.md)は `ctx.otel` を登録する。これは独立した通常イベント・チャネルと Session ログ・チャネルのための共有ファクトリーである。トランスポートと SDK バッチ処理の実装を所有する。ビジネス・コンシューマーはサービスを注入し、明示的なトランスポート／リソース／スコープのオプションを渡し、返された各チャネルのシャットダウンを所有する。

UI のフィードバックは、既存の Remote API を通じて Session フィードバック・サービスに届く。フィードバックの記録は、OTel チャネルを通じて報告する前に、Session アダプターの正規接頭辞キャプチャとリダクトを認可する。通常の製品コンシューマーは製品アダプターを通じて選択されたフィールドを送信する。共有サービスには自動のアイデンティティやキャプチャ・ポリシーがなく、チャネルはキューや要求を決して共有しない。

[製品アダプター](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/host/product-telemetry-otel/README.md)と [Session アダプター](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-telemetry-otel/README.md)は、自身のデプロイ設定と破棄デッドラインを保持する。コンシューマーはシャットダウン後にチャネルを再利用してはならない。OTel サービスのみをマウントしても、プロバイダーやトランスポートは割り当てられない。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxotel--otel"></a>

### `ctx.otel` — `OTel`

共有トランスポート・プロバイダー。マウントしてもキュー、アイデンティティ、ネットワーク接続は作成されない。

```ts cordis-catalog
/**
 * Create an independent ordinary-event channel with count-based batching.
 * The injected consumer must drain it during its fiber disposal.
 * @param options - transport, scope, resource, queue, and diagnostic settings selected by the consumer.
 * @returns the caller-owned channel; no state is shared with other channels.
 */
createEventReporter(options: EventLogOptions): EventLogReporter

/**
 * Create an independent byte-bounded Session-log channel.
 * Authorization and redaction precede reporting; the consumer owns shutdown and its outer deadline.
 * @param options - transport, scope, resource, queue, and diagnostic settings selected by the consumer.
 * @returns the caller-owned channel, preserving complete accepted events within the request byte ceiling.
 */
createSessionLogReporter(options: SessionLogOptions): SessionLogReporter
```

ソース: [`packages/telemetry/otel/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/telemetry/otel/src/index.ts)
<!-- END GENERATED cordis-surface -->
