# LLM アダプター

日本語 | [English](../../../../../docs/user/develop/practice/llm-adapter.md) | [中文](../../../../../docs/user/develop/practice/llm-adapter.zh.md)

> これは非公式の日本語訳です。英語版（[docs/user/develop/practice/llm-adapter.md](../../../../../docs/user/develop/practice/llm-adapter.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

このガイドでは、新しい LLM provider を Harness に接続します。

## 概要

LLM アダプターは `LlmAdapter` を拡張して `stream()` を実装し、Harness の provider 中立なリクエストを provider API 呼び出しに変換し、応答を Harness のチャンクに変換して戻します。

## 最小実装

```ts
import type { Context } from '@deepseek-ai/cordis'
import Schema from '@deepseek-ai/schemastery'
import { LlmAdapter, type GenerateOptions, type StreamChunk } from '@deepseek-ai/dsh-llm'

class MyAdapter extends LlmAdapter {
  private apiKey: string

  constructor(apiKey: string) {
    super()
    this.apiKey = apiKey
  }

  async *stream(options: GenerateOptions): AsyncIterable<StreamChunk> {
    // 1. Convert options.messages to the provider format.
    // 2. Call the streaming API.
    // 3. Convert the response into StreamChunk values.
  }
}

export interface Config {
  apiKey: string
  providers: string[]
}

export const Config: Schema<Config> = Schema.object({
  apiKey: Schema.string().required(),
  providers: Schema.array(Schema.string()).required(),
})

export const name = 'my-llm-adapter'
export const inject = ['llm']

export function apply(ctx: Context, config: Config) {
  const adapter = new MyAdapter(config.apiKey)
  ctx.llm.registerAdapter(config.providers, adapter)
}
```

## StreamChunk プロトコル

`stream()` はこのプロトコルでチャンクを yield します:

```ts
import { brandString } from '@deepseek-ai/dsh-brand'
import type { StreamChunk, ToolCallId } from '@deepseek-ai/dsh-llm'

async function* exampleChunks(): AsyncIterable<StreamChunk> {
  // 1. Start each content block with block-start.
  yield { type: 'block-start', index: 0, blockType: 'text' }

  // 2. Stream text through text-delta.
  yield { type: 'text-delta', index: 0, text: 'Hello' }
  yield { type: 'text-delta', index: 0, text: ' world' }

  // 3. End each content block with block-end and the complete block.
  yield {
    type: 'block-end',
    index: 0,
    block: { type: 'text', text: 'Hello world' },
  }

  // 4. Tool-call block.
  yield { type: 'block-start', index: 1, blockType: 'tool-call' }
  yield {
    type: 'tool-call-delta',
    index: 1,
    id: brandString<ToolCallId>('call-123'),
    name: 'bash',
    argumentsDelta: '{"command":"ls"}',
  }
  yield {
    type: 'block-end',
    index: 1,
    block: {
      type: 'tool-call',
      id: brandString<ToolCallId>('call-123'),
      name: 'bash',
      arguments: '{"command":"ls"}',
    },
  }

  // 5. Token usage.
  yield { type: 'usage', usage: { inputTokens: 100, outputTokens: 50 } }

  // 6. Finish reason.
  yield { type: 'finish', reason: { kind: 'stop' } }
  // Alternatively, { kind: 'tool-calls' } requests tool execution.
}
```

### 主なルール

- すべての `block-start` には対応する `block-end` があります。
- `index` は 0 から増加し、コンテンツブロックの順序を識別します。
- `tool-call-delta` は `argumentsDelta` に生の JSON テキストを運びます。一度にすべてでも、複数のチャンクに分けてもかまいません。
- `finish` は最後のチャンクです。
- `finish` の前に `usage` を出力します。

## GenerateOptions

`stream()` はエクスポートされた `GenerateOptions` 型を受け取ります。これにはモデル、アダプター所有の reasoning-effort id、会話履歴、システムプロンプト、tool スキーマ、生成パラメーター、停止シーケンス、abort signal が含まれます。`@deepseek-ai/dsh-llm` がエクスポートする TypeScript 型を正とみなしてください。サポートするフィールドを provider API にマップします。provider がフィールドを尊重できない場合は、黙って破棄するのではなく、安定したコードを持つ `LlmError` をスローします。

`resolveModel(provider, model, signal?)` をオーバーライドして、正確な provider/model の同一性と、オプションの `context` および `reasoning` メタデータを1回の lookup で返します。reasoning メタデータには、順序付きの不透明な id と表示名、およびオプションの設定済み既定値が含まれます。上流の capability API が `off` を返す場合を含め、アダプターの正となる選択可能リストを保持し、それらの値をコアの enum に昇格させないでください。非同期 lookup ではオプションの signal を尊重し、キャンセルと破棄が静止状態に達するようにします。サービスは集約を検証し、サポートされない明示的な effort を `stream()` の前に拒否します。`reasoning` を省略すると、そのモデルに選択可能な reasoning-effort capability がないことを意味します。

## アダプターを登録する

```ts ignore-check
ctx.llm.registerAdapter(['my-provider'], adapter)
```

最初の引数は、アダプターが処理する provider ルートのリストです。`GenerateOptions.provider` は登録済みアダプターを選択し、`GenerateOptions.model` はライフサイクル登録なしでアダプター所有のモデル id を渡します。アダプターがセレクターにモデル選択肢を通知できる場合は `listModels()` をオーバーライドします。

## cordis.yml から使う

```yaml
- id: my-llm
  name: './src/my-llm-adapter.ts'
  config:
    apiKey: !!js process.env.MY_API_KEY
    providers:
      - my-provider

- id: agent-loop
  name: '@deepseek-ai/dsh-agent-loop'
  config:
    agents:
      - id: main
        provider: my-provider
        model: my-model-v1
```

## リファレンス実装

リポジトリには完全な実装があります:

- `packages/llm/llm-deepseek/` — Messages API を使う DeepSeek アダプター
- `packages/llm/llm-pi-ai/` — 異なる API 形式を使う Pi AI アダプター

同梱の2つのアダプターを比較すると、同じ harness 契約が異なる provider SDK 上で実装されていることがわかります。

## エラーハンドリング

アダプターはトランスポートとプロトコルの失敗を、安定したコードを持つ `LlmError` 値としてスローします。agent ループは診断とポリシーのためにエラーとコードを保持し、通常の `Error` を自動変換しません。すべての provider HTTP リクエストは `attributionHeaders()` をマージし、`options.signal` を転送する必要もあります。

```ts
import {
  attributionHeaders,
  LlmAdapter,
  LlmError,
  type GenerateOptions,
  type StreamChunk,
} from '@deepseek-ai/dsh-llm'

class HttpAdapter extends LlmAdapter {
  constructor(private readonly endpoint: string) {
    super()
  }

  async *stream(options: GenerateOptions): AsyncIterable<StreamChunk> {
    const response = await fetch(this.endpoint, {
      method: 'POST',
      headers: {
        'content-type': 'application/json',
        ...attributionHeaders(),
      },
      body: JSON.stringify({ model: options.model, messages: options.messages }),
      ...options.signal ? { signal: options.signal } : {},
    })
    if (!response.ok) {
      throw new LlmError(`Provider API error: ${response.status}`, 'PROVIDER_HTTP_ERROR')
    }
    // A real adapter parses the response and emits the complete chunk sequence.
    yield { type: 'finish', reason: { kind: 'stop' } }
  }
}
```
