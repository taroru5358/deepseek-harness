# ブラウザー操作

日本語 | [English](../../../docs/subsystems/browser-use.md) | [中文](../../../docs/subsystems/browser-use.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/browser-use.md](../../../docs/subsystems/browser-use.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

ブラウザー操作により、モデルは設定されたバックエンドを通じて Web ページを検査し操作できる。DSH がタスク・ループを所有し、プロバイダーがブラウザー操作を提供して、1 つのライブ Session のターンをまたいでブラウザー状態を保持する。

## プロバイダーの選択

[`dsh-browser-use`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/browser-use/browser-use/README.md)と 1 つのプロバイダーを同じコンポジションにマウントする。プロバイダーは実験的な公開 npm パッケージであり、明示的な有効化を必要とする。当初のブラウザー・エンジンは Chromium である。

| プロバイダー | 統合 |
|---|---|
| [Playwright MCP](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/browser-use-playwright-mcp/README.md) | Playwright のブラウザー制御 MCP ツール |
| [Chrome DevTools MCP](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/browser-use-chrome-devtools-mcp/README.md) | MCP 経由の Chrome DevTools 検査と制御 |
| [Stagehand](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/browser-use-stagehand-native/README.md) | AI 支援のアクション、観察、抽出を伴うネイティブのブラウザー操作 |

共有サービスは名前だけを登録し、2 つ目のプロバイダーを拒否する。同じ名前の別インスタンスも含まれる。共通のブラウザー操作メソッド、ブラウザー・リソース、モデルが制御するセレクターは持たない。プロファイルまたはプリセット内のプロバイダー設定が、そのアクティブ化のための起動かアタッチかを選択する。

## Session の所有権

起動されたブラウザーは、それを使う正確なライブ Agent と Session に属する。ターンをまたぐ呼び出しはそのブラウザーを再利用する。Session ランタイムを破棄すると起動されたリソースが閉じられる。Session をリロードまたはフォークすると新しいブラウザー状態で始まる。ブラウザー・プロファイルとログイン状態は Session ログから復元されない。

アタッチされたブラウザーは外部所有のままである。プロバイダーはそのプロバイダー・インスタンス内の 1 つの Session のためにそれを予約し、既存のブラウザー状態を保ち、別の Session による同時のアタッチを拒否する。解体は切断し、外部ブラウザーを実行したままにする。別々の DSH プロセスや他のクライアントはこの予約の外に留まる。

プロバイダーのシャットダウンはツールのアドミッションを止め、所有作業とリソースのクリーンアップを待ってから共有プロバイダー登録を解放する。キャンセルは、すでに届けられたブラウザー・アクションを取り消せない。

## MCP の初期化

MCP プロバイダーは、プロバイダーがロードされた後に作成されるライブ Agent ごとに 1 つのクライアントを初期化する。既存の直列な `agent/created` イベントは、作成または再開が完了してキューに入った入力が走る前に、接続と発見を待つ。クライアントはターンをまたいで Session とともに残る。起動の失敗やキャンセルは作成または再開を拒否し、クライアントのクリーンアップをトリガーする。

アタッチがビジーな場合、そのアクティブ化はブラウザーなしで続行し、後のターンで再試行しない。解放後、新しく作成された、または再開されたアクティブ化がそれを獲得できる。プロバイダーのロードやリロードはすでにアクティブな Session を引き継がない。これらの初期化規則は[共有ランタイム](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/browser-use-runtime/README.md)が所有する。

## ツールと記録される結果

プロバイダーのツールは通常の DSH 実行パイプラインと Session ログを使う。プロバイダーは自身のツール・スキーマ、結果のレンダリング、画像サポート、設定、上流の制限を所有し、共有サービスはモデル可視のコンテンツを追加しない。Stagehand の AI 支援操作は明示的に設定されたネイティブ・モデルを使い、DSH はタスク・ループを保持する。DSH のモデル・ルーティング、資格情報の再利用、基盤となる推論要求／レスポンスのキャプチャ、Session の使用量計上への統合は先送りされ、返された SDK データとメタデータは通常の記録済みツール結果のままである。

ブラウザー MCP 接続は[リソースとサーバー指示](mcp.md)も公開する。ブラウザー・サーバー宛てのリソース呼び出しはその Session キューを使い、他の Session を拒否する。サーバー指示はその所有 Session のためにのみ組み立てられる。

[決定記録](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-09-12-browser-use-provider-registration.md)は、登録のみのサービスと Session ごとの所有権について説明する。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxbrowseruse--browseruseregistry"></a>

### `ctx.browserUse` — `BrowserUseRegistry`

共有 browser-use サービス内の、任意の 1 つのプロバイダー登録を所有する。

```ts cordis-catalog
/**
 * Reserve the sole provider slot until the contribution is disposed.
 * A second registration fails even when it repeats the current name. Providers
 * must stop their tools and await owned work before releasing this registration.
 * @param name - provider-owned name used in registration diagnostics.
 * @returns the effect disposer for this exact registration.
 */
register(name: BrowserUseProviderName): () => Promise<void>
```

ソース: [`packages/browser-use/browser-use/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/browser-use/browser-use/src/index.ts)
<!-- END GENERATED cordis-surface -->
