# plugin の設定

日本語 | [English](../../../../../docs/user/develop/basic/config.md) | [中文](../../../../../docs/user/develop/basic/config.zh.md)

> これは非公式の日本語訳です。英語版（[docs/user/develop/basic/config.md](../../../../../docs/user/develop/basic/config.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

`cordis.yml` を通じて提供される設定を受け付けます。

## Config 型を定義する

`Config` 型と同じ名前の Schemastery スキーマをエクスポートします。既定値はスキーマのフィールドに直接置きます:

```ts
import type { Context } from '@deepseek-ai/cordis'
import Schema from '@deepseek-ai/schemastery'

export const name = 'my-plugin'

export interface Config {
  greeting: string
  maxRetries: number
  verbose?: boolean
}

export const Config: Schema<Config> = Schema.object({
  greeting: Schema.string().default('Hello'),
  maxRetries: Schema.number().default(3),
  verbose: Schema.boolean().default(false),
})

export function apply(ctx: Context, config: Config) {
  console.log(config.greeting)  // User value or schema default.
}
```

挿入したローカル plugin の行の設定を `scratch-plugin/cordis.yml` に追加します:

```yaml
- insert:
    - id: hello
      name: './src/my-plugin.ts'
      config:
        greeting: 'Hi there'
        maxRetries: 5
```

plugin を読み込むとき、Cordis はエクスポートされたスキーマを使って設定を検証し、既定値を埋めます。`Config` としてプレーンオブジェクトをエクスポートしないでください。Cordis が必要とする Standard Schema インターフェースを実装していません。

## スキーマ検証

より厳密な検証を表現するには Schemastery を使います:

```ts
import type { Context } from '@deepseek-ai/cordis'
import Schema from '@deepseek-ai/schemastery'

export const name = 'validated-plugin'

export interface Config {
  apiKey: string
  timeout: number
  mode: 'fast' | 'accurate'
}

export const Config = Schema.object({
  apiKey: Schema.string().required(),
  timeout: Schema.number().default(30000),
  mode: Schema.union(['fast', 'accurate']).default('fast'),
})

export function apply(ctx: Context, config: Config) {
  // config is validated and type-safe.
}
```

スキーマは plugin の読み込み中に実行されます。無効な設定は、対処可能なエラーで読み込みを失敗させます。

## 設計原則

### 調整可能な値をハードコードしない

Harness は、**2つのデプロイが異なる値を設定したいと思うものはすべて設定フィールドにする**ことを要求します。

```ts
// Wrong: hardcoded timeout.
const TIMEOUT = 30000

// Correct: configurable.
export interface Config {
  timeoutMs: number  // Defaults to 30000.
}
```

テストは、`cordis.yml` がコード編集なしで値を変更できるかどうかです。

### 無効な設定では明確に失敗する

自己完結した制約はスキーマで表現し、無効な設定が plugin の読み込み中に失敗するようにします。サービスや登録済みリソースへの参照には依存性注入が必要です。[サービスチュートリアル](../framework/service.md) がその契約を紹介します。

## HMR と連携する

設定を編集すると plugin がホットリプレースされます。フレームワークは古いインスタンスをアンロードし、新しいものを読み込みます。登録は effect であり自身をクリーンアップするため、置き換えで古いインスタンスの登録が残ることはありません。

## 次のステップ

- [plugin をパッケージ化してインストールする](publish.md) — plugin をインストール可能なパッケージとして配布する
- [plugin とライフサイクル](../framework/index.md) — plugin の完全なライフサイクルを理解する
- [サービスと依存関係](../framework/service.md) — 他の plugin にサービスを提供する
