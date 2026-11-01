# 7. harness へ

日本語 | [English](../../../docs/cordis-tutorial/07-into-the-harness.md) | [中文](../../../docs/cordis-tutorial/07-into-the-harness.zh.md)

> これは非公式の日本語訳です。英語版（[docs/cordis-tutorial/07-into-the-harness.md](../../../docs/cordis-tutorial/07-into-the-harness.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

この章では、harness の `tools` サービスにモデル呼び出し可能なツールを登録し、harness のツール・パイプラインを通じて実行し、結果イベントを観察します。引き続きキーなしで、モデルは呼び出しません。

## ツール・プラグイン

`tmp/cordis-tutorial` に `greet-tool.ts` を作成します:

```ts
import type { Context } from '@deepseek-ai/cordis'
import { brandString } from '@deepseek-ai/dsh-brand'
import { defineTool } from '@deepseek-ai/dsh-tools'
import type { ToolCallId } from '@deepseek-ai/dsh-llm'

export const name = 'greet-tool'
export const inject = ['tools']

export function apply(ctx: Context) {
  ctx.tools.register(defineTool({
    name: 'greet',
    description: 'Greet the named person.',
    parameters: {
      name: { type: 'string', required: true, description: 'Who to greet' },
    },
    output: {
      schema: { type: 'string' },
      render: (_args, value) => [{ type: 'text', text: value }],
    },
    async execute(args) {
      return `Hello, ${args.name}!`
    },
  }))

  // Drive one call through the real execution pipeline, standing in for
  // the model. ToolCallId brands the correlation id a provider would issue.
  void (async () => {
    const result = await ctx.tools.execute({
      calls: [{
        type: 'tool-call', id: brandString<ToolCallId>('demo-1'),
        name: 'greet', arguments: JSON.stringify({ name: 'Cordis' }),
      }],
      signal: new AbortController().signal,
    })
    console.log('tool replied:', JSON.stringify(result.results[0]?.result.content))
  })()
}
```

ここでのすべてのパターンは以前の章のものです。`inject: ['tools']`（[第3章](03-services.md)）はツール・レジストリが存在するまでプラグインを保持します。`ctx.tools.register(...)` は登録のディスポーザーをプラグインに結び付けるため（[第2章](02-lifecycle-and-effects.md)）、アンロードするとツールが登録解除されます。`defineTool` は `parameters` 仕様をモデルに見せる JSON Schema に変換し、`args` の型を推論し、`execute` が実行される前にモデル供給の引数を検証します。ツールは `output.schema` が宣言する正規値を返し、`output.render` が別途 Native と永続的な結果コンテンツを生成します。

## オブザーバー・プラグイン

`tool-logger.ts` を作成します。harness の `tools/result` イベントを通じて、アプリ内のすべてのツール呼び出しを監視する別のプラグインです:

```ts
import type { Context } from '@deepseek-ai/cordis'
import type {} from '@deepseek-ai/dsh-tools'

export const name = 'tool-logger'
export const inject = ['tools']

export function apply(ctx: Context) {
  ctx.on('tools/result', (exec, result) => {
    const text = result.content
      .map(block => (block.type === 'text' ? block.text : ''))
      .join('')
    console.log(`[tool-logger] ${exec.name} -> ${text}`)
  })
}
```

`import type {} from '@deepseek-ai/dsh-tools'` の行はパッケージの宣言マージを取り込み、`'tools/result'` とそのペイロードを型付けします。第4章の `stats.ts` import と同じ動きを、パッケージ規模で行うものです。

## 構成して実行する

```yaml
- name: '@deepseek-ai/dsh-system-prompt'
- name: '@deepseek-ai/dsh-tools'
- name: './tool-logger.ts'
- name: './greet-tool.ts'
```

`@deepseek-ai/dsh-tools` は `systemPrompt` サービスを inject します。ツールがシステム・プロンプトにスキーマを提供するためで、構成はそのプロバイダーも列挙します。それがなければ、[第6章](06-composition-and-hmr.md)で説明したように tools プラグインは PENDING のままです。

```sh
node --import tsx ../../vendor/cordis/bin.js
```

```
[tool-logger] greet -> Hello, Cordis!
tool replied: [{"type":"text","text":"Hello, Cordis!"}]
```

ロガーが最初に発火しました。`tools/result` は結果の実体化の一部として emit され、`execute` の promise が呼び出し元に解決する前です。あなたの2つのプラグインは互いの存在を知りません。レジストリ・サービスとイベントがそれらを接続します。

## ここから完全な agent へ

実際の agent は、この構成にさらに多くのプラグインを加えたものです。LLM アダプター、agent loop、永続化、アプリケーション・エントリーです。[base プロファイル・レイヤー](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/base/cordis.patch.yml)と[ヘッドレス・レイヤー](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/headless/cordis.patch.yml)を比較してください。今ならそれらのエントリーを読めます。`greet-tool.ts` は小さな `--patch` オーバーレイで追加できます。

次に進む先:

- [ツールを構築する](../user/develop/basic/tool.md) — `defineTool` のさらに詳しい内容。プレゼンテーションとより豊かなスキーマを含みます。
- [3層ケイパビリティ設計](../user/develop/practice/index.md) — harness が交換可能なケイパビリティをどのように構造化するか。
- [サブシステムのページ](../subsystems/core.md)にある生成済みの `cordis-surface` 領域 — inject してリッスンできるすべて。それぞれ所有ページにあります。
- [アーキテクチャ](../architecture.md) — これらのプラグインが住むシステム・マップ。

[![](https://img.shields.io/badge/powered_by-dsh-4D6BFE?style=flat-square&logo=deepseek&logoColor=white)](https://github.com/deepseek-ai/deepseek-harness)
