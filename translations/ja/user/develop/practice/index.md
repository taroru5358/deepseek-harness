# 3ロールの capability 設計

日本語 | [English](../../../../../docs/user/develop/practice/index.md) | [中文](../../../../../docs/user/develop/practice/index.zh.md)

> これは非公式の日本語訳です。英語版（[docs/user/develop/practice/index.md](../../../../../docs/user/develop/practice/index.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

このページは2部構成です。3ロール capability パターンの概念リファレンスの後に、1つの capability を構築する上級チュートリアルが続きます。先に [基本 plugin パス](../basic/index.md) と [service チュートリアル](../framework/service.md) を完了してください。

## 概念リファレンス

capability が、Bash 実行のように交換可能な provider を必要とするほど一般的な場合、Harness は3つのロールを分離します。**Service Definition**、**Service Provider**、**Consumer** です。独立して進化または置き換える必要がある場合は、ロールを別々のパッケージに置きます。そうでなければ、1つのパッケージが複数のロールを所有してもかまいません。完全な capability がその seam です。個々のロールは seam ではありません。

## Bash の例

Bash 実行 capability は次で構成されます:

- **Service Definition**（`dsh-shell`）— Cordis サービスと Bash のリクエスト・結果型を定義する
- **Service Provider**（`dsh-bash-local`）— ローカルマシン上でコマンドを実行する
- **Consumer**（`dsh-tool-bash`）— capability をモデルが呼び出せる tool として公開する

```
┌─────────────┐     ┌──────────────────┐     ┌──────────────┐
│  dsh-shell   │────▶│  dsh-bash-local  │     │ dsh-tool-bash│
│(definition) │     │    (provider)     │     │(consumer/tool)│
└─────────────┘     └──────────────────┘     └──────────────┘
       ▲                                            │
       └────────────────────────────────────────────┘
                    inject: ['shell']
```

## 分割の利点

### provider を置き換える

1つの Service Definition に、`cordis.yml` を通じて選択される複数の provider を持たせられます:

```yaml
# Local execution
- name: '@deepseek-ai/dsh-bash-local'

# Replace this row with another package that provides the same service.
```

provider が変わっても、Service Definition と tool は変わりません。

### 独立して進化させる

- Service Definition は、呼び出し元がその契約に依存した後はほとんど変更されません。
- Service Provider はパフォーマンスとセキュリティを独立して改善できます。
- Consumer は、capability をモデルに提示する方法を変更できます。

### 依存関係を分離する

- Service Provider は Service Definition に依存します。
- Consumer は Service Definition に依存します。
- Service Provider と Consumer は**互いに依存しません**。

[capability-seam リファレンス](../../../capability-seams.md) が現在の組み込みファミリーとパッケージリンクを管理しています。

## チュートリアル: 3ロール capability を開発する

### ステップ1: Service Definition を書く

```ts ignore-check
// packages/my-cap/my-cap/src/index.ts
import { Service, type Context } from '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' {
  interface Context {
    myCap: MyCapService
  }
}

export abstract class MyCapService extends Service {
  constructor(ctx: Context) {
    super(ctx, 'myCap')
  }

  /** Execute the capability. */
  abstract execute(request: MyCapRequest): Promise<MyCapResult>
}

export interface MyCapRequest {
  input: string
}

export interface MyCapResult {
  output: string
}
```

### ステップ2: Service Provider を書く

```ts ignore-check
// packages/my-cap/my-cap-local/src/index.ts
import type { Context } from '@deepseek-ai/cordis'
import { MyCapService, type MyCapRequest, type MyCapResult } from '@deepseek-ai/dsh-my-cap'

class MyCapLocal extends MyCapService {
  async execute(request: MyCapRequest): Promise<MyCapResult> {
    // Local provider behavior.
    return { output: request.input.toUpperCase() }
  }
}

export const name = 'my-cap-local'

export function apply(ctx: Context) {
  ctx.plugin(MyCapLocal)
}
```

### ステップ3: consumer を書く

```ts ignore-check
// packages/my-cap/tool-my-cap/src/index.ts
import type { Context } from '@deepseek-ai/cordis'
import { defineTool } from '@deepseek-ai/dsh-tools'

export const name = 'tool-my-cap'
export const inject = ['tools', 'myCap']

export function apply(ctx: Context) {
  ctx.tools.register(defineTool({
    name: 'my_cap',
    description: 'Execute my capability.',
    parameters: {
      input: { type: 'string', required: true },
    },
    output: {
      schema: { type: 'string' },
      render: (_args, value) => [{ type: 'text', text: value }],
    },
    async execute(args) {
      const result = await ctx.myCap.execute({ input: args.input })
      return result.output
    },
  }))
}
```

### cordis.yml で構成する

```yaml
- name: '@deepseek-ai/dsh-my-cap-local'
- name: '@deepseek-ai/dsh-tool-my-cap'
```

## 設計上のポイント

- **先回りして分割しない** — ロールが独立して進化する必要がある場合にのみ、別々のパッケージを使います。単純な tool plugin では不要です。
- **Service Definition が Request/Result 型を所有する** — Service Provider と Consumer は Service Definition パッケージにのみ依存します。
- **明示 > 暗黙** — `run()` の中に `?? default` 式を隠すのではなく、明示的な `resolve(request): Spec` ステップで既定値を解決します。

## 次のステップ

- [LLM アダプター](llm-adapter.md) — LLM provider を実装する
