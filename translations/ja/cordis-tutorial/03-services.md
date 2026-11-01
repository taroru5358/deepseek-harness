# 3. サービス

日本語 | [English](../../../docs/cordis-tutorial/03-services.md) | [中文](../../../docs/cordis-tutorial/03-services.zh.md)

> これは非公式の日本語訳です。英語版（[docs/cordis-tutorial/03-services.md](../../../docs/cordis-tutorial/03-services.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

**サービス**は、1つのプラグインが提供し、他のプラグインが `ctx` を通じて消費する名前付きのケイパビリティです。harness では `ctx.tools`、`ctx.llm`、`ctx.agents` がサービスです。コンシューマーはプロバイダーを import するのではなく、`'tools'` のようなケイパビリティを名指しするため、コンシューマーを変更せずに設定でプロバイダーを選択できます。

## サービスを提供する

`tmp/cordis-tutorial` に `greeter.ts` を作成します:

```ts
import { Service, type Context } from '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' {
  interface Context {
    greeter: GreeterService
  }
}

export class GreeterService extends Service {
  constructor(ctx: Context) {
    super(ctx, 'greeter')
  }

  greet(who: string) {
    return `Hello, ${who}!`
  }
}

export const name = 'greeter'

export function apply(ctx: Context) {
  ctx.plugin(GreeterService)
}
```

2つの部分が協調します:

- **実行時**: `super(ctx, 'greeter')` はインスタンスを `greeter` という名前で登録します。以降、どのプラグインも `ctx.greeter` としてそれに到達できます。登録は効果であり、プロバイダーをアンロードするとサービスが削除されます。
- **コンパイル時**: `declare module '@deepseek-ai/cordis'` ブロックは TypeScript の宣言マージです。`Context` インターフェースに `greeter` を追加するため、`ctx.greeter` がどこでも型チェックされます。コードは生成しません。それがなくてもサービスは実行時に動作しますが、コンシューマーは型安全性を失います。

`Service` サブクラスはそれ自体がプラグイン（第1章のクラス形式）であるため、`ctx.plugin(GreeterService)` は他のプラグインと同様にそれをマウントします。

## `inject` でサービスを消費する

`consumer.ts` を作成します:

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'consumer'
export const inject = ['greeter']

export function apply(ctx: Context) {
  console.log(ctx.greeter.greet('world'))
}
```

`inject` はこのプラグインが要求するサービスを列挙します。Cordis は列挙されたすべてのサービスが存在するまでプラグインを PENDING に保つため、`apply` 内では `ctx.greeter` が準備完了であることが保証されます。`cordis.yml` のロード順は重要ではありません。プラグインがいつ開始するかを決めるのはファイル順ではなく依存関係です。

構成して実行します:

```yaml
- name: './greeter.ts'
- name: './consumer.ts'
```

```
Hello, world!
```

`cordis.yml` の2行を入れ替えて再実行すると、同じ出力になります。`./greeter.ts` を完全に削除してみてください。コンシューマーは PENDING のままで何も出力しません。クラッシュも部分実行もありません。PENDING の fiber は Node のイベント・ループも生かし続けないため、他に何も実行していない構成は静かに終了コード 0 で終了します。[第6章](06-composition-and-hmr.md)でその状態を診断する方法を示します。

## 依存関係はロード後も追跡される

`inject` は一度限りの起動チェックではありません。アプリの実行中に必須サービスが消えると — そのプロバイダーがアンロードまたはホット置換されると — 依存するすべてのプラグインもアンロードされ、サービスが戻ると再びロードされます。効果（[第2章](02-lifecycle-and-effects.md)）と組み合わせることで、実行中のコンシューマーが利用できないサービスへの参照を保持することを防ぎます。依存が消えると、自身の登録が巻き戻されます。

これが設定でサービスの置き換えが機能する理由でもあります。`dsh-bash-local` エントリーをアンロードし、別の `shell` プロバイダーをマウントすると、`'shell'` を inject するすべてのプラグインが新しい実装に対してクリーンに再起動します。

## 任意の依存関係

`inject` は厳格な要件のためのものです。プラグインがなくても生きていけるケイパビリティでは、`inject` を省略し、使用箇所で調べます:

```ts ignore-check
export function apply(ctx: Context) {
  // undefined when no provider is loaded; the plugin still runs.
  const greeter = ctx.get('greeter')
  console.log(greeter?.greet('maybe') ?? 'no greeter available')
}
```

## 命名

サービス名はアプリケーションごとに1つのフラットな名前空間に存在します。自分のサービスには特徴的なプレフィックスまたは名前空間を付けてください（harness は `tools` や `llm` のような平易な名前をすでに使用しています）。[サブシステムのページ](../subsystems/core.md)にある生成済みの `cordis-surface` 領域が、harness が登録するすべての名前を列挙します。

次: [イベント](04-events.md) — 共有サービスなしの通信。

[![](https://img.shields.io/badge/powered_by-dsh-4D6BFE?style=flat-square&logo=deepseek&logoColor=white)](https://github.com/deepseek-ai/deepseek-harness)
