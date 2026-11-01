# plugin とライフサイクル

日本語 | [English](../../../../../docs/user/develop/framework/index.md) | [中文](../../../../../docs/user/develop/framework/index.zh.md)

> これは非公式の日本語訳です。英語版（[docs/user/develop/framework/index.md](../../../../../docs/user/develop/framework/index.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

このページでは Cordis の plugin モデルとライフサイクル状態機械を説明します。

## Fiber 状態機械

読み込まれたすべての plugin は、次の状態を持つ **Fiber** スコープを所有します:

```
PENDING → LOADING → ACTIVE
                 ↘ FAILED
ACTIVE → UNLOADING → DISPOSED
```

| 状態 | 意味 |
|------|------|
| PENDING | 宣言済みだが、必須依存関係が準備できていない |
| LOADING | 依存関係が準備でき、`apply` が実行中 |
| ACTIVE | plugin が実行中 |
| FAILED | `apply` がエラーをスローした |
| UNLOADING | plugin がアンロード中でリソースを破棄している |
| DISPOSED | plugin が完全にアンロードされた |

## 依存関係駆動の読み込み

`inject` を持つ plugin は、読み込む前にすべての必須サービスを待ちます:

```ts ignore-check
export const inject = ['tools', 'llm']

export function apply(ctx: Context) {
  // ctx.tools and ctx.llm are ready here.
}
```

必須サービスが消失した場合、たとえば provider の置き換え中に、plugin は自動的にアンロードされ（ACTIVE → DISPOSED）、サービスが戻ると再び読み込まれます。

## 自動クリーンアップ

`ctx` を通じて行われたすべての登録は、plugin のアンロード時に元に戻されます:

```ts ignore-check
export function apply(ctx: Context) {
  // Event listener: removed automatically on unload.
  ctx.on('some-event', handler)

  // Custom resource: the returned disposer runs on unload.
  ctx.effect(() => {
    const connection = createConnection()
    return () => connection.close()
  })
}
```

フレームワークはこれらの操作をすべて追跡して破棄します:
- `ctx.on(event, handler)` — イベントリスナー
- `ctx.tools.register(tool)` — tool の登録
- `ctx.llm.registerAdapter(names, adapter)` — LLM アダプターの登録
- `ctx.effect(() => cleanup)` — カスタムリソース

アンロード中、disposer の呼び出しは登録の逆順で始まりますが、複数の非同期 disposer は並行して実行され、直列完了の保証はありません。順序に依存するクリーンアップは、単一の `ctx.effect()` から返す1つの disposer にまとめ、そこでそのステップを直列に await してください。

## ネストしたコンテキスト

`ctx.plugin()` は、親コンテキストを継承しつつ独立したライフサイクルを持つ子 Fiber を作成します:

```ts ignore-check
export function apply(ctx: Context) {
  // Register a child plugin.
  ctx.plugin(childPlugin)

  // The child has its own Fiber and unloads with its parent.
}
```

## Dispose のセマンティクス

plugin インスタンスを早期に停止するには:

```ts
import type { Context } from '@deepseek-ai/cordis'

declare const ctx: Context
declare function myPlugin(ctx: Context): void

const fiber = ctx.plugin(myPlugin)

// Dispose it manually later.
await fiber.dispose()
```

`dispose` は次を保証します:
1. plugin が所有するすべての登録が削除される。
2. 子 plugin が再帰的にアンロードされる。
3. 返された promise は、すべての非同期クリーンアップが完了した後に解決される。

## ホットリプレース（HMR）

`cordis.yml` から `@deepseek-ai/dsh-hmr` を読み込むと、plugin のソースファイルを編集することで次が発生します:

1. 古い plugin をアンロードし、その登録をクリーンアップする。
2. 新しいコードを読み込む。
3. 新しい `apply` を実行する。

plugin の登録は自身をクリーンアップするため、ホットリプレースで古いインスタンスの登録が残ることはありません。

## ライフサイクルの例

```ts ignore-check
export function apply(ctx: Context) {
  console.log('plugin loading')

  ctx.effect(() => {
    console.log('effect registered')
    return () => console.log('effect cleaned up')
  })
}
```

読み込み時は次を出力します:
```
plugin loading
effect registered
```

アンロード時は次を出力します:
```
effect cleaned up
```

## 次のステップ

- [サービスと依存関係](service.md) — capability を他の plugin に公開する
- [イベントシステム](events.md) — plugin 間で通信する
- [Cordis チュートリアル](../../../cordis-tutorial/index.md) — 同じライフサイクル、サービス、イベントを Cordis runtime に対して段階的に構築する
