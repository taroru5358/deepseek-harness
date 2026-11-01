# 4. イベント

日本語 | [English](../../../docs/cordis-tutorial/04-events.md) | [中文](../../../docs/cordis-tutorial/04-events.zh.md)

> これは非公式の日本語訳です。英語版（[docs/cordis-tutorial/04-events.md](../../../docs/cordis-tutorial/04-events.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

サービスは直接呼び出しをサポートします。**イベント**により、プラグインは誰がリッスンしているかを知らなくても何かを通知できます。harness はツール結果、モデル・リクエスト、承認の決定などのインタラクションにイベントを使います。

## 宣言、emit、リッスン

`tmp/cordis-tutorial` に `stats.ts` を作成します。何かを数え、各変更を通知するサービスです:

```ts
import { Service, type Context } from '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' {
  interface Context {
    stats: StatsService
  }
  interface Events {
    'stats/report'(name: string, count: number): void
  }
}

export class StatsService extends Service {
  private counts = new Map<string, number>()

  constructor(ctx: Context) {
    super(ctx, 'stats')
  }

  bump(name: string) {
    const next = (this.counts.get(name) ?? 0) + 1
    this.counts.set(name, next)
    this.ctx.emit('stats/report', name, next)
  }
}

export const name = 'stats'

export function apply(ctx: Context) {
  ctx.plugin(StatsService)
}
```

`interface Events` のマージは、第3章の `interface Context` マージのイベント・システム側の双子です。イベント名とリスナー・シグネチャを宣言するため、`ctx.emit` と `ctx.on` が完全に型付けされます。`namespace/action` の命名規約がフラットなイベント名前空間を読みやすく保ちます。

`reporter.ts` を作成します:

```ts ignore-check
import type { Context } from '@deepseek-ai/cordis'
import type {} from './stats.ts'

export const name = 'reporter'
export const inject = ['stats']

export function apply(ctx: Context) {
  ctx.on('stats/report', (name, count) => {
    console.log(`[stats] ${name} -> ${count}`)
  })
  ctx.stats.bump('tool_call')
  ctx.stats.bump('tool_call')
  ctx.stats.bump('prompt')
}
```

`import type {} from './stats.ts'` の行は実行時に何も import しません。TypeScript が宣言マージを見るために存在します。構成して実行します:

```yaml
- name: './stats.ts'
- name: './reporter.ts'
```

```
[stats] tool_call -> 1
[stats] tool_call -> 2
[stats] prompt -> 1
```

`ctx.on()` は効果であるため、リスナーはプラグインとともに消えます。手動の `removeListener` 管理は決して必要ありません。

## ディスパッチ・モード

`emit` は5つのディスパッチ・モードの1つです。イベントがどれを使うかはそのコントラクトの一部であり、リスナーが値を返せるか、並行して実行できるか、互いにショートサーキットできるかを決めます:

| モード | 呼び出し | セマンティクス |
|---|---|---|
| emit | `ctx.emit(name, ...args)` | 同期ブロードキャスト。返された promise と値は待機も収集もされない。 |
| parallel | `await ctx.parallel(name, ...args)` | すべてのリスナーが並行して実行され、まとめて待機される。 |
| serial | `await ctx.serial(name, ...args)` | リスナーは順番に実行され、待機される。最初の `null`/`false`/`undefined` 以外の戻り値が勝ち、残りを停止する。 |
| bail | `ctx.bail(name, ...args)` | serial の同期版。 |
| waterfall | `ctx.waterfall(name, ...args, next)` | アラウンド・ミドルウェア。以下を参照。 |

すべての harness イベントは、所有する[サブシステムのページ](../subsystems/core.md)の生成リファレンスでそのモードを文書化します。

## Waterfall: 変換またはショートサーキット

Waterfall は傍受を支えるモードです。各リスナーは引数と `next()` 継続を受け取り、`next()` が返すものを変換するか、`next()` を呼ばずに戻って残りのチェーンをショートサーキットできます。Cordis のドキュメントではこれを veto と呼びます。`waterfall-demo.ts` を作成します:

```ts
import type { Context } from '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' {
  interface Events {
    'demo/transform'(input: string, next: () => Promise<string>): Promise<string>
  }
}

export const name = 'waterfall-demo'

export function apply(ctx: Context) {
  // Listener 1: wrap the downstream result.
  ctx.on('demo/transform', async (input, next) => {
    const downstream = await next()
    return downstream.toUpperCase()
  })

  // Listener 2: short-circuit when it owns the decision.
  ctx.on('demo/transform', async (input, next) => {
    if (input.includes('blocked')) return '** blocked **'
    return next()
  })

  void (async () => {
    console.log(await ctx.waterfall('demo/transform', 'hello', async () => 'hello'))
    console.log(await ctx.waterfall('demo/transform', 'blocked words', async () => 'blocked words'))
  })()
}
```

`cordis.yml` をこのファイルだけに向けて実行します:

```
HELLO
** BLOCKED **
```

2行目を順に追います。リスナー1が最初に実行され、`next()` を呼び、それがリスナー2を呼び出します。リスナー2は `blocked` を見て、`next()` を呼ばずに戻ります。最内側の既定（`ctx.waterfall` に渡された関数）は決して実行されず、リスナー1が戻る途中で置換メッセージを大文字にします。

そこから導かれる規律: **観察または注釈だけを行う waterfall リスナーは `next()` を呼ばなければなりません**。それを呼ばずに戻るのは意図的なショートサーキットです。ログ用リスナーで `next()` を忘れると、下流のすべてに対して既定の振る舞いを黙って飲み込みます。これはこのリポジトリの常設ルールです（[waterfall セマンティクス](../cordis-primer.md#cordis-waterfall-semantics)）。

harness は、協調するプラグインが包んだり回答したりできる決定に waterfall を使います。[`agent/pre-step`](../subsystems/core.md#agentpre-step--waterfall) はプラグインが提案されたステップを拒否したり入力を置き換えたりできるようにし、[`approval/request`](../subsystems/approval.md#approvalrequest--waterfall) はユーザーの代わりにポリシーが回答できるようにします。

次: [設定](05-config.md) — `cordis.yml` からのプラグイン・オプション。

[![](https://img.shields.io/badge/powered_by-dsh-4D6BFE?style=flat-square&logo=deepseek&logoColor=white)](https://github.com/deepseek-ai/deepseek-harness)
