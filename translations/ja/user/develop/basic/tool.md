# tool を構築する

日本語 | [English](../../../../../docs/user/develop/basic/tool.md) | [中文](../../../../../docs/user/develop/basic/tool.zh.md)

> これは非公式の日本語訳です。英語版（[docs/user/develop/basic/tool.md](../../../../../docs/user/develop/basic/tool.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

このチュートリアルでは、`greet` tool を Web UI に追加します。先に [はじめての plugin](index.md) を完了し、その `scratch-plugin` ディレクトリを保持してください。

## tool plugin を作成する

`scratch-plugin/src/my-plugin.ts` を次のように置き換えます:

```ts
import type { Context } from '@deepseek-ai/cordis'
import { defineTool } from '@deepseek-ai/dsh-tools'

export const name = 'greet-tool'
export const inject = ['tools']

export function apply(ctx: Context) {
  ctx.tools.register(defineTool({
    name: 'greet',
    description: 'Greet someone by name.',
    parameters: {
      name: { type: 'string', required: true, description: 'The name to greet' },
    },
    output: {
      schema: { type: 'string' },
      render: (_args, value) => [{ type: 'text', text: value }],
    },
    async execute(args) {
      return `Hello, ${args.name}!`
    },
  }))
}
```

`inject` は Cordis に tool registry を待たせます。`defineTool` は `parameters` から `args` を推論して検証します。`execute` は `output.schema` で宣言された正規値を返し、`output.render` がその値をモデル向けコンテンツに変換します。

## 実行して tool を呼び出す

開発コマンドが実行されていない場合は再起動します:

```sh
pnpm dsh web --patch ./scratch-plugin/cordis.yml
```

`http://127.0.0.1:3080` を開き、次のように尋ねます: `Use the greet tool to greet Ada.` モデルは `greet` を呼び出し、tool 結果として `Hello, Ada!` を受け取ります。

## 次のステップ

- [plugin の設定](config.md) — あいさつを設定可能にする。
- [tool 作成リファレンス](../../../cookbook/adding-a-tool.md) — ネストしたスキーマ、正規値、バックグラウンド作業、ポリシーフック、PTC モード、UI カードを調べる。
- [Capability のレイヤー化](../practice/index.md) — 交換可能な capability を Service Definition、Service Provider、Consumer パッケージに分割する。
