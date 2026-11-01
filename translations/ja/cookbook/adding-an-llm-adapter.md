# クックブック: LLM アダプターの追加

日本語 | [English](../../../docs/cookbook/adding-an-llm-adapter.md) | [中文](../../../docs/cookbook/adding-an-llm-adapter.zh.md)

> これは非公式の日本語訳です。英語版（[docs/cookbook/adding-an-llm-adapter.md](../../../docs/cookbook/adding-an-llm-adapter.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

新しいモデル・プロバイダーを接続する方法です。参照実装: `packages/llm/llm-deepseek`（直接 HTTP、`eventsource-parser` でフレーム化する SSE）と `packages/llm/llm-pi-ai`（LLM ライブラリのラップ）。まず `packages/llm/llm/src/types.ts` の `StreamChunk` ドキュメントを読んでください。両アダプターが検証されたプロトコル規約が記録されています。

## 形

```ts ignore-check
class MyAdapter extends LlmAdapter {
  async * stream(options: GenerateOptions): AsyncIterable<StreamChunk> { … }
}

export const name = 'llm-myprovider'
export const inject = ['llm']
export const Config: z<Config> = z.object({ apiKey: z.string(), … })

export function apply(ctx: Context, config: Config) {
  ctx.llm.registerAdapter(['my-provider'], new MyAdapter(…))
}
```

登録は効果ベース（HMR 安全）です。プロバイダー・ルートごとに1つのアダプターで、重複は throw し、複数ルート登録はオール・オア・ナッシングです。`options.provider` がアダプターを選択し、`options.model` がプロバイダーのモデル id です。そのため動的カタログ・アダプターはライフサイクル再設定なしで新しいモデルにサービスできます。シークレットは cordis ネイティブです。環境フォールバック付きの schemastery Config を使い、cordis.yml から `!!js process.env.MY_KEY` で供給します。コード内でアドホックなキー・ファイルを読んではいけません。

## プロトコル上の義務（2つの実装が検証したコントラクト）

- `finish` の前に `usage` を emit し、`finish` の後には何も emit しません。堅牢な方法は、プロバイダーの end-of-stream マーカーまで finish/usage をバッファし、その後フラッシュすることです（末尾に usage のみのチャンクを送るプロバイダーを処理できます）。
- ツール呼び出しの `arguments` はエンドツーエンドで生の JSON 文字列です。フラグメントは `argumentsDelta` としてストリームします。プロバイダーが解析済みオブジェクトを返す場合は、`block-end` で再文字列化します。
- ブロックの `index` はストリームで最初に現れた順に割り当て、同じブロックのすべてのデルタでその index を再利用します。
- エラーには正確に2つの認められた経路があります。`stream()` から THROW する（トランスポートとプロトコルの失敗 — 安定したコードを持つ `LlmError` を使う）か、`finish {kind: 'error' | 'aborted'}` でストリームを終える（プロバイダーの帯域内失敗）かです。コンシューマーは両方を処理します。失敗クラスごとに選び、文書化します。
- `options.signal` を尊重します（fetch / SDK に渡します）。
- プロバイダーが尊重できない `GenerateOptions` フィールド（例: stop シーケンスを持たないプロバイダーの `stop` リスト）は、黙って落とすのではなく `LlmError(..., 'UNSUPPORTED_OPTION')` を throw します。
- プロバイダーがフォローアップ呼び出しでレスポンス id、署名、その他のネイティブ・メタデータを必要とする場合は、最小のロスレス JSON projection を `finish.replayState` として emit します。履歴を再構築するときにそれを検証します。`LlmRuntime` は、履歴のプロバイダー・ルートとターゲット・プロバイダー・ルートが現在まったく同じアダプター・インスタンスに所有されている場合にのみそれを渡します。同一モデル、クロスモデル、クロスプロバイダーの復元が合法かどうかはアダプターが決めます。状態がない場合に、プロバイダー名／モデル名だけからネイティブ・リプレイを推論してはいけません。

プロバイダー固有の thinking-mode トグルはアダプターの Config に残ります。正確なモデル・メタデータは1つのプロバイダー中立なケイパビリティ・シームを使います。`resolveModel()` をプロバイダー／モデル同一性と任意の `context`、`reasoning` フィールドで実装し、設定された `defaultEffort` は存在する場合にのみ宣言し、リゾルバーの任意の `AbortSignal` を尊重します。reasoning effort は、アダプターがプロバイダー・リクエストにマップする順序付きの不透明 id です。アダプターの権威ある選択可能リストを保持し、サポートされる場合はアダプター定義の `off` を含め、最終的なワイヤー表記を公開したりサポートされない値をクランプしたりしません。id はワイヤー表現と等しい必要はありません。

## 実装構造

ワイヤー型、リクエスト・シリアライゼーション、トランスポート解析、チャンク変換、アダプター・クラスを別々の責務として保ちます。[`llm-deepseek`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/llm-deepseek/README.md) が参照レイアウトです。

## 検証

[リポジトリのテスト方針](../testing.md)に従ってください。アダプターのカバレッジ、実際のプロバイダーでのチェック、公開エントリーの要件を所有します。
