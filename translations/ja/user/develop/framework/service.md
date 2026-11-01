# サービスと依存関係

日本語 | [English](../../../../../docs/user/develop/framework/service.md) | [中文](../../../../../docs/user/develop/framework/service.zh.md)

> これは非公式の日本語訳です。英語版（[docs/user/develop/framework/service.md](../../../../../docs/user/develop/framework/service.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

サービスとは、1つの plugin が他の plugin に公開する capability です。`inject` は plugin が必要とするサービスを宣言します。

## サービスとは何か

Harness では、`tools`、`llm`、`agents` はサービスです。それぞれが `ctx` にマウントされた名前付き capability です:

```ts ignore-check
ctx.tools    // ToolRuntime service
ctx.llm      // LLM service
ctx.agents   // Agent service
```

どの plugin も、他の plugin が利用するサービスを提供できます。

## サービスを利用する

既存のサービスを使うには `inject` を宣言します:

```ts ignore-check
export const inject = ['tools']

export function apply(ctx: Context) {
  // ctx.tools exists and is ready here.
  ctx.tools.register(/* ... */)
}
```

`apply` が実行されるとき、`inject` で宣言されたすべてのサービスは準備完了です。サービスが準備できていない場合、plugin は実行せずに待機します。

## サービスを提供する

### Service を拡張する

```ts
import { Service, type Context } from '@deepseek-ai/cordis'

export default class MetricsService extends Service {
  static inject = ['llm']  // A service may depend on other services.

  constructor(ctx: Context) {
    super(ctx, 'metrics')  // 'metrics' is the service name.
  }

  // Public service method.
  record(event: string, value: number) {
    // ...
  }
}
```

この plugin を読み込んだ後、consumer は `ctx.metrics` としてサービスにアクセスします:

```ts ignore-check
export const inject = ['metrics']

export function apply(ctx: Context) {
  ctx.metrics.record('tool_call', 1)
}
```

### 型を宣言する

TypeScript の宣言マージを使って `ctx.metrics` に型を付けます:

```ts
import { Service, type Context } from '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' {
  interface Context {
    metrics: MetricsService
  }
}

export default class MetricsService extends Service {
  constructor(ctx: Context) {
    super(ctx, 'metrics')
  }

  record(event: string, value: number) { /* ... */ }
}
```

## 依存関係の動作

### 必須依存とオプション依存

```ts ignore-check
// Required: the plugin does not load while the service is absent.
export const inject = ['tools']

// Optional: omit inject and query with ctx.get() at the use site.
export function apply(ctx: Context) {
  const metrics = ctx.get('metrics')
  metrics?.record('plugin_loaded', 1)
}
```

### サービスが消失したとき

必須サービスがアプリケーション実行中に消失した場合、たとえばその provider がアンロードされた場合:

1. 依存する plugin は自動的に dispose されます。
2. サービスが戻ると再び読み込まれます。

これは、存在しなくなったサービスを plugin が呼び出すことを防ぎます。

## サービスの分離

`cordis.yml` はサービスを分離でき、別々の plugin グループが同じサービスの別々のインスタンスを見るようにできます:

```yaml
- id: group-a
  name: '@deepseek-ai/cordis-plugin-group'
  group: true
  isolate:
    shell: true
  config:
    - name: '@deepseek-ai/dsh-bash-local'
      config:
        timeoutMs: 5000
    - name: './src/plugin-a.ts'

- id: group-b
  name: '@deepseek-ai/cordis-plugin-group'
  group: true
  isolate:
    shell: true
  config:
    - name: '@deepseek-ai/dsh-bash-local'
      config:
        timeoutMs: 60000
    - name: './src/plugin-b.ts'
```

`plugin-a` と `plugin-b` はそれぞれ自分のグループ内の Bash インスタンスを見て、グループ間の影響はありません。

## Harness の組み込みサービス

リポジトリは、サービス名、公開メソッド、ソース位置を各サービスの [サブシステムページ](../../../subsystems/core.md) に生成します。plugin を開発するときは、それらの生成領域とサービスの TypeScript インターフェースを使ってください。2つ目の静的リストを維持しないでください。

## 次のステップ

- [イベントシステム](events.md) — 密結合なしで plugin 間を通信する
- [Capability のレイヤー化](../practice/index.md) — サービスを capability インターフェースとして使う
