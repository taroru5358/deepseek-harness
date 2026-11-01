# はじめての plugin

日本語 | [English](../../../../../docs/user/develop/basic/index.md) | [中文](../../../../../docs/user/develop/basic/index.zh.md)

> これは非公式の日本語訳です。英語版（[docs/user/develop/basic/index.md](../../../../../docs/user/develop/basic/index.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

このチュートリアルでは、最小限の Harness plugin を作成し、Web UI に読み込みます。[ソースからの実行パス](../../../overview.md#run-from-source) を完了したリポジトリのチェックアウトから始めます。

## ローカルプロジェクトを作成する

リポジトリルートから、チュートリアル用のスクラッチプロジェクトを作成します:

```sh
mkdir -p scratch-plugin/src
```

## plugin とは何か

Harness では、plugin は `apply` 関数をエクスポートする TypeScript モジュールです。フレームワークは plugin を読み込むときに `apply` を呼び出し、plugin が capability を登録する `ctx` コンテキストオブジェクトを渡します:

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'my-plugin'

export function apply(ctx: Context) {
  // Register capabilities here.
}
```

これが完全な設定です。

## plugin ファイルを作成する

`scratch-plugin/src/my-plugin.ts` を作成します:

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'hello-plugin'

export function apply(ctx: Context) {
  // Required dependencies are ready before apply runs.
  console.log('[hello-plugin] plugin loaded!')
}
```

## cordis.yml に登録する

リポジトリルートで `pwd` を実行し、ローカル plugin を挿入する Web オーバーレイとして `scratch-plugin/cordis.yml` を作成します。下の `/absolute/path/to/deepseek-harness` を表示されたパスに置き換えます:

```yaml
- insert:
    - id: hello
      name: '/absolute/path/to/deepseek-harness/scratch-plugin/src/my-plugin.ts'
```

plugin のパスは絶対パスでなければなりません。patch ファイルは設定を提供しますが、loader がモジュールパスを解決する profile ディレクトリは変更しません。

そのオーバーレイで Web UI を起動します:

```sh
pnpm dsh web --patch ./scratch-plugin/cordis.yml
```

`http://127.0.0.1:3080` を開きます。起動中にターミナルが `[hello-plugin] plugin loaded!` を出力します。

## 自動クリーンアップ

`ctx` を通じて登録されたもの（イベントリスナー、tool、タイマー）は、plugin のアンロード時にクリーンアップされます。removeListener や clearInterval を手動で呼ぶ必要はありません。

ネットワーク接続など明示的なクリーンアップが必要なリソースには、`ctx.effect()` を使ってその disposer を提供します:

```ts
import type { Context } from '@deepseek-ai/cordis'

export function apply(ctx: Context) {
  ctx.effect(() => {
    const timer = setInterval(() => {
      console.log('heartbeat')
    }, 5000)

    // The returned function runs when the plugin unloads.
    return () => clearInterval(timer)
  })
}
```

## 依存関係を宣言する

plugin が `tools` や `llm` などの別のサービスを利用する場合は、`inject` で宣言します:

```ts ignore-check
import type { Context } from '@deepseek-ai/cordis'

export const name = 'my-tool-plugin'
export const inject = ['tools']

export function apply(ctx: Context) {
  // ctx.tools is ready here.
  ctx.tools.register(/* ... */)
}
```

フレームワークは plugin を読み込む前に、すべての必須サービスを待ちます。

## 3つの plugin 形式

関数モジュールに加えて、plugin はオブジェクト形式またはクラス形式を使えます。

### オブジェクト形式

```ts
import type { Context } from '@deepseek-ai/cordis'

export default {
  name: 'my-plugin',
  inject: ['tools'],
  apply(ctx: Context) {
    // ...
  },
}
```

### クラス形式

```ts
import { Service, type Context } from '@deepseek-ai/cordis'

export default class MyService extends Service {
  static inject = ['tools']

  constructor(ctx: Context) {
    super(ctx, 'myService')
    // Perform synchronous initialization in the constructor.
  }
}
```

ほとんどの場合、関数形式で十分です。plugin が他の plugin にサービスを提供する場合はクラス形式を使います。[サービスと依存関係](../framework/service.md) を参照してください。

## 次のステップ

- [tool を構築する](tool.md) — tool 定義 DSL を学ぶ
- [plugin の設定](config.md) — ユーザー設定を受け付ける
- [Cordis チュートリアル](../../../cordis-tutorial/index.md) — その下にある plugin フレームワーク。API キーなしのスクラッチディレクトリから構築する
