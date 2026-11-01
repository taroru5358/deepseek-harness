# 2. ライフサイクルと効果

日本語 | [English](../../../docs/cordis-tutorial/02-lifecycle-and-effects.md) | [中文](../../../docs/cordis-tutorial/02-lifecycle-and-effects.zh.md)

> これは非公式の日本語訳です。英語版（[docs/cordis-tutorial/02-lifecycle-and-effects.md](../../../docs/cordis-tutorial/02-lifecycle-and-effects.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

Cordis プラグインは、設定の編集、ホットリロード、明示的な破棄、必須サービスの喪失によってアンロードされます。Cordis API を通じて行われた登録は効果であり、所有するプラグインがアンロードされると巻き戻されます。それらの API の外で管理されるリソースは `ctx.effect()` で包まなければなりません。

## 効果

Cordis がまだ管理していないリソース — タイマー、接続、watcher — は `ctx.effect()` で包み、ディスポーザーを返します:

`tmp/cordis-tutorial` に `lifecycle.ts` を作成します:

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'lifecycle-demo'

function heartbeat(ctx: Context) {
  console.log('heartbeat plugin loading')
  ctx.effect(() => {
    const timer = setInterval(() => console.log('tick'), 200)
    return () => {
      clearInterval(timer)
      console.log('heartbeat cleaned up')
    }
  })
}

export function apply(ctx: Context) {
  // Mount a child plugin and keep its fiber to dispose it later.
  const fiber = ctx.plugin(heartbeat)
  // The demo timer is itself an effect: if THIS plugin is unloaded first,
  // the pending callback is cancelled instead of firing on a dead app.
  ctx.effect(() => {
    const timer = setTimeout(async () => {
      await fiber.dispose()
      console.log('disposed')
      process.exit(0)
    }, 700)
    return () => clearTimeout(timer)
  })
}
```

`cordis.yml` をそれに向けます:

```yaml
- name: './lifecycle.ts'
```

実行（`node --import tsx ../../vendor/cordis/bin.js`）すると、次が得られます:

```
heartbeat plugin loading
tick
tick
tick
heartbeat cleaned up
disposed
```

注目すべき3つの点:

- `ctx.plugin(heartbeat)` は関数を**コードから**プラグインとしてマウントします。YAML ローダーが各設定エントリーに対して行うのと同じ操作です。関数プラグインに `apply` メソッドは不要です。Cordis は関数を直接呼び、その名前は診断にのみ使います。`apply` メソッドが必要なのはオブジェクト形式 `ctx.plugin({ apply(ctx) { /* ... */ } })` だけです。呼び出しは**fiber**、すなわちロードされた1つのプラグイン・インスタンスのランタイム・ハンドルを返します。
- 効果の本体はロード中に実行され、返されたディスポーザーはアンロード中に実行されます。プラグイン寿命のリソースに対して、自分でディスポーザーを呼ぶことはありません。
- `fiber.dispose()` は、プラグインのすべてのクリーンアップ — 非同期ディスポーザーを含む — が完了した後に解決し、マウントした子プラグインを再帰的にアンロードします。

## fiber の状態機械

ロードされた各プラグイン・インスタンスは、次の状態を遷移する fiber を所有します:

```
PENDING → LOADING → ACTIVE → UNLOADING → DISPOSED
                 ↘ FAILED
```

- **PENDING** — 宣言されているが、必須サービス（第3章）がまだ利用できない。
- **LOADING / ACTIVE** — `apply` が実行中／完了している。
- **FAILED** — `apply` または設定検証が throw した。
- **UNLOADING / DISPOSED** — ディスポーザーが実行中／すべてが破棄された。

PENDING は[第6章](06-composition-and-hmr.md)で再び登場します。そこでは「なぜ私のプラグインは何も出力しないのか」への通常の答えになります。

## すでに効果であるもの

組み込みの登録 API がすでに効果であるため、自分で `ctx.effect()` を書くことはほとんどありません:

- `ctx.on(event, listener)` — リスナーはアンロード時に削除されます（[第4章](04-events.md)）。
- `ctx.plugin(child)` — 子は親とともに破棄されます。
- サービスの登録は効果です。`ctx.tools.register(...)` のような harness レジストリも、返されたディスポーザーを呼び出し元プラグインに結び付けるため、自動的に巻き戻されます（[第7章](07-into-the-harness.md)）。

Cordis が管理しないリソースは、`ctx.effect()` 内で取得し、それを解放するディスポーザーを返します。Cordis はアンロード中（ホットリロードを含む）にその解放を呼び出します。

順序に関する注意点が1つあります。ディスポーザーは登録の逆順で開始されますが、複数の**非同期**ディスポーザーは並行して実行されます。teardown のステップを順番に実行する必要がある場合は、1つのディスポーザーに入れてそこで await してください。

次: [サービス](03-services.md) — プラグインがケイパビリティを共有する方法。

[![](https://img.shields.io/badge/powered_by-dsh-4D6BFE?style=flat-square&logo=deepseek&logoColor=white)](https://github.com/deepseek-ai/deepseek-harness)
