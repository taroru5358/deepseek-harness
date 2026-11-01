# イベントシステム

日本語 | [English](../../../../../docs/user/develop/framework/events.md) | [中文](../../../../../docs/user/develop/framework/events.zh.md)

> これは非公式の日本語訳です。英語版（[docs/user/develop/framework/events.md](../../../../../docs/user/develop/framework/events.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

イベントは Cordis plugin 間の主要な通信機構です。Harness は疎結合な拡張ポイントのためにイベントを広く使います。

## 基本的な使い方

### イベントを待ち受ける

```ts ignore-check
ctx.on('event-name', (payload) => {
  // Handle the event.
})
```

### イベントを発火する

```ts ignore-check
ctx.emit('event-name', payload)
```

## イベントモード

Cordis は異なる相互作用契約のためにいくつかのイベントモードを提供します。

### emit — ブロードキャスト

すべてのリスナーが同期的に実行され、戻り値は無視されます:

```ts ignore-check
// Emit
ctx.emit('my-plugin/ready', { id: 'worker-1' })

// Listen
ctx.on('my-plugin/ready', ({ id }) => {
  console.log(`${id} is ready`)
})
```

### bail — ショートサーキット

リスナーは順番に実行され、`null`、`false`、`undefined` 以外の最初の結果が最終結果になります:

```ts ignore-check
// Dispatch
const result = ctx.bail('some-check', input)

// Listen: a returned value stops later listeners.
ctx.on('some-check', (input) => {
  if (shouldBlock(input)) return 'blocked'
  // Return null, false, or undefined to continue to the next listener.
})
```

### serial — 順序付き実行

リスナーは登録順に実行され、非同期結果は待機されます。`null`、`false`、`undefined` 以外の最初の結果が以降の実行を停止します:

```ts ignore-check
await ctx.serial('setup-phase', context)
```

### waterfall — パイプライン

各リスナーは下流の結果をラップして処理チェーンを形成できます。リスナーは**下流に委譲するために `next()` を呼び出す必要があります**。呼び出しを省略するとパイプラインがショートサーキットされます:

```ts ignore-check
// Dispatch
const output = await ctx.waterfall('my-plugin/transform', input, async () => input)

// Listen: next() is mandatory.
ctx.on('my-plugin/transform', async (_input, next) => {
  const downstream = await next()
  return downstream.trim()
})
```

::: warning
waterfall のリスナーは**必ず `next()` を呼び出す必要があります**。省略すると設計上パイプラインがショートサーキットされ、インターセプトやゲートウェイの動作が可能になります。
:::

## 型付きイベント

Harness は型安全なイベントのために TypeScript の宣言マージを使います:

```ts
import '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' {
  interface Events {
    'my-plugin/ready': (payload: { id: string }) => void
    'my-plugin/check': (input: string) => boolean | undefined
    'my-plugin/transform': (input: string, next: () => Promise<string>) => Promise<string>
  }
}

// ctx.on('my-plugin/ready', ...) and ctx.emit('my-plugin/ready', ...)
// are now inferred correctly.
```

## Cordis イベントと session レコード

Harness の Cordis イベントは `namespace/action` 名を使います。`agent/pre-step`、`agent/request-error`、`tools/result`、`session/event` などです。[サブシステムページ](../../../subsystems/core.md) の生成された `cordis-surface` 領域が、完全なシグネチャとモードを記録します。

`turn/*`、`step/*`、`tool/call`、`tool/result`、`compaction/*` は永続的な session イベント型であり、同名の Cordis イベントではありません。それらを観察するには、`session/event` を待ち受けて `event.type` を調べます。

## イベントリスナーは effect

`ctx.on()` で登録されたリスナーは、その plugin のアンロード時に自動的に削除されます:

```ts ignore-check
export function apply(ctx: Context) {
  // This listener is removed when the plugin disposes.
  ctx.on('tools/result', handler)
}
```

## 例: ロギング plugin

この plugin は tool 呼び出しと結果をログに記録します:

```ts
import type { Context } from '@deepseek-ai/cordis'
import '@deepseek-ai/dsh-tools'

export const name = 'tool-logger'

export function apply(ctx: Context) {
  ctx.on('tools/result', (exec, result) => {
    console.log(`[tool] ${exec.name}(${JSON.stringify(exec.arguments)})`)
    const text = result.content
      .map(block => block.type === 'text' ? block.text : '')
      .join('')
    console.log(`[tool result] ${text.slice(0, 100)}`)
  })
}
```

## 次のステップ

- [Capability のレイヤー化](../practice/index.md) — capability インターフェース内のイベントを理解する
- [LLM アダプター](../practice/llm-adapter.md) — 完全な LLM バックエンドを実装する
